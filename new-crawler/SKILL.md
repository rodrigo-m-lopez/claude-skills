---
name: new-crawler
description: >
  Guide for implementing a new crawler that collects data from an external website via
  HTTP or headless browser. Use this skill whenever adding a new data source, integrating
  a new website, or implementing any scraping/crawling functionality. Load it even when
  the user just mentions a site name as a target — the reconnaissance and level
  classification steps will save significant debugging time later.
---

# New Crawler

Before writing any production code, go through the reconnaissance and classification steps below. Choosing the wrong approach means either unnecessary complexity (if you skip to a higher level) or time lost debugging bot protection (if you underestimate it).

---

## Step 1 — Reconnaissance

Gather the following before touching any code. Ask the user or investigate with DevTools/curl:

1. **Target URL** — what site needs to be crawled?
2. **Data of interest** — what should be extracted? (events, prices, products, articles…)
3. **Data source identification** — open DevTools > Network > XHR/Fetch on the site and answer:
   - Is there an API URL that returns JSON directly?
   - Does that URL work with a plain `curl`, no cookies?
   - What headers does the request use? (`Referer`, `Origin`, `x-api-key`, cookies…)
   - Is the content in server-rendered HTML or loaded by JS (SPA)?

Use these answers to classify the site into one of the levels below before continuing.

---

## Step 2 — Complexity Level Classification

Start at the lowest level that works. Never jump to a higher level without confirming the previous one fails — each level adds meaningful overhead in complexity, memory, and maintenance.

### Level 1 — Plain HTTP + public JSON API
**When:** The API URL returns JSON without authentication and works with plain `curl`.

**Diagnosis:**
```bash
curl -s "https://api.example.com/data" | head -c 500
# Should return real JSON, not an error page or HTML
```

**Implementation pattern (Java):**
```java
JsonObject response = new JsonObject("https://api.example.com/data");
```

**Watch out for:** API contract changes without notice; versioned or rotating tokens.

---

### Level 2 — HTTP with required headers
**When:** The API exists but returns 403/429/empty without specific headers (`Referer`, `Origin`, `User-Agent`, `x-api-key`, static session cookies).

**Diagnosis:**
```bash
curl -s -H "Referer: https://www.example.com/" \
     -H "Origin: https://www.example.com" \
     -H "User-Agent: Mozilla/5.0 ..." \
     "https://api.example.com/data" | head -c 500
# If data comes back: this is Level 2
```

**Implementation pattern (Java):**
```java
Map<String, String> headers = Map.of(
    "User-Agent",     "Mozilla/5.0 (Windows NT 10.0; Win64; x64) ...",
    "Referer",        "https://www.example.com/",
    "Origin",         "https://www.example.com",
    "Accept",         "application/json, */*",
    "sec-fetch-site", "same-origin",
    "sec-fetch-mode", "cors",
    "sec-fetch-dest", "empty"
);
JsonObject response = JsonObject.fromUrl(url, headers);
```

**Watch out for:** Rotating auth tokens require renewal logic; headers need periodic review.

---

### Level 3 — Headless Playwright with session warm-up
**When:** curl with headers fails, but a headless browser that visits the homepage first establishes a valid session. Typical for: Cloudflare JS Challenge, JS-generated cookies.

**Diagnosis:**
```
1. Level 2 failed even with all headers copied from DevTools
2. Playwright (no stealth), after navigating to the homepage, can call the API
```

**Implementation pattern (Java + Playwright):**
```java
Browser browser = playwright.chromium().launch(new LaunchOptions().setHeadless(true));
BrowserContext context = browser.newContext();

// warm-up: establish session by loading the homepage
Page page = context.newPage();
page.navigate("https://www.example.com");
page.close();

// now use session cookies to call the API
APIResponse resp = context.request().get("https://www.example.com/api/data");
String json = resp.text();
```

**Watch out for:** 1–3 s per session startup; high memory with many concurrent sites; Chromium must be installed.

---

### Level 4 — Playwright with stealth + DOM extraction
**When:** Simple warm-up isn't enough. Content is in the DOM (no JSON API) or protection is more aggressive (Cloudflare Managed Challenge, advanced fingerprinting).

**Diagnosis:**
```
1. Level 3 returns an error page, CAPTCHA, or empty HTML
2. Content loads normally in a real browser
```

**Implementation pattern (Java + Playwright + Jsoup):**
```java
BrowserType.LaunchOptions opts = new BrowserType.LaunchOptions()
    .setHeadless(false)   // or Xvfb on server
    .setArgs(List.of("--disable-blink-features=AutomationControlled"));
Browser browser = playwright.firefox().launch(opts); // Firefox has a smaller fingerprint

Page page = browser.newPage();
page.navigate("https://www.example.com/list");
page.waitForLoadState(LoadState.NETWORKIDLE);

Document doc = Jsoup.parse(page.content());
Elements items = doc.select(".item-selector");
```

**Additional tools:** playwright-stealth, camoufox, nodriver.

**Watch out for:** 5–15 s per page; CSS selectors break on redesigns.

---

### Level 5 — Playwright + residential proxy + simulated behavior
**When:** Serious protection: DataDome, PerimeterX, Imperva, behavioral reCAPTCHA v3. Detects headless even with stealth.

**Diagnosis:**
```
Level 4 is still blocked after a few requests, or requires interactive CAPTCHA.
```

**Implementation pattern (Java + Playwright + proxy):**
```java
BrowserType.LaunchOptions opts = new BrowserType.LaunchOptions()
    .setProxy(new Proxy("http://user:pass@residential-proxy:8080"));

// Simulate human behavior
page.mouse().move(randomX, randomY);
page.keyboard().type(text, new Keyboard.TypeOptions().setDelay(120));
Thread.sleep(randomBetween(800, 2400));
page.evaluate("window.scrollBy(0, " + randomScroll + ")");
```

**Tools:** Rotating residential proxies (Brightdata, Oxylabs, Smartproxy), CAPTCHA solvers, Browserless.io.

**Cost:** High (~$2–15/GB of proxy). Evaluate cost-benefit before committing to this level.

---

## Step 3 — Implementation (TDD)

Follow the `tdd` skill before writing any production code.

For crawlers specifically, start here:
- Capture a real JSON response from the site and save it to `src/test/resources/json/SiteName/response.json`
- Write the mapper test using that fixture (assert: fields are mapped correctly)
- Only then implement `CrawlerSiteName` and `SiteNameJsonMapper`

Recommended structure:
```
crawler/
  CrawlerSiteName.java       ← implements ICrawler (or shared abstract base)
json/
  SiteNameJsonMapper.java    ← converts raw JSON → domain entities
src/test/resources/json/SiteName/
  response.json              ← real response captured from the site
```

---

## Step 4 — Quality checklist

- [ ] Crawler returns an empty list (does not throw) when the site is unavailable
- [ ] Errors are logged with `log.warn(...)`, not propagated up
- [ ] API URL and headers are named constants, not inline literals
- [ ] At least one test uses a real JSON fixture
- [ ] New crawler is registered in the crawler container/active list
- [ ] Site is seeded in the database (or created as transient if not persisted)

---

## Project level reference

| Level | Project example | Base class |
|---|---|---|
| 1 | Goldebet, Bateu, Esportiva… | `AbstractAltenarCrawler` |
| 2 | Betano | `ICrawler` direct with `JsonObject.fromUrl(url, headers)` |
| 3 | SportingBet, Betway | `PlaywrightHeadlessBrowserClient` |
| 4–5 | (not implemented) | Playwright + stealth/proxy |
