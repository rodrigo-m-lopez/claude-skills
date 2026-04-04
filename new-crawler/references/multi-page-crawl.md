# Multi-Page Crawl Reference

Deep-dive implementations for URL discovery, canonicalization, sitemap parsing,
robots.txt handling, and the BFS crawl loop. Referenced from SKILL.md Step 2.5.

---

## 1. Full URL Canonicalization

Use this before adding any URL to the `visited` set or the crawl queue. Two URLs that
look different but refer to the same resource must map to the same canonical string.

```java
import java.net.URI;
import java.net.URISyntaxException;
import java.util.TreeMap;
import java.util.stream.Collectors;

public static String canonicalize(String rawUrl, String baseUrl) {
    try {
        URI base = new URI(baseUrl);
        URI uri = base.resolve(rawUrl.trim()); // resolve relative URLs

        String scheme = "https";              // always normalize to https
        String host = uri.getHost().toLowerCase()
                .replaceFirst("^www\\.", ""); // strip www.
        int port = uri.getPort();
        String path = uri.getPath()
                .replaceFirst("/+$", "");     // strip trailing slash
        if (path.isEmpty()) path = "";

        // Sort query parameters alphabetically; strip tracking params
        String query = null;
        if (uri.getQuery() != null) {
            TreeMap<String, String> params = new TreeMap<>();
            for (String pair : uri.getQuery().split("&")) {
                String[] kv = pair.split("=", 2);
                String key = kv[0];
                // drop common tracking params
                if (key.matches("utm_.*|fbclid|gclid|ref|source")) continue;
                params.put(key, kv.length > 1 ? kv[1] : "");
            }
            if (!params.isEmpty()) {
                query = params.entrySet().stream()
                        .map(e -> e.getKey() + "=" + e.getValue())
                        .collect(Collectors.joining("&"));
            }
        }

        // fragment (#...) is never part of canonical form
        URI canonical = new URI(scheme, null, host,
                port == 443 || port == -1 ? -1 : port,
                path, query, null);
        return canonical.toString();
    } catch (URISyntaxException e) {
        return rawUrl; // fallback: return as-is
    }
}
```

---

## 2. Sitemap Parsing

### 2a. Standard sitemap.xml

```java
import org.w3c.dom.*;
import javax.xml.parsers.*;
import java.io.*;
import java.util.*;

public static List<String> parseStandardSitemap(String xml) throws Exception {
    List<String> urls = new ArrayList<>();
    DocumentBuilder builder = DocumentBuilderFactory.newInstance().newDocumentBuilder();
    Document doc = builder.parse(new ByteArrayInputStream(xml.getBytes("UTF-8")));
    NodeList locs = doc.getElementsByTagNameNS("*", "loc");
    for (int i = 0; i < locs.getLength(); i++) {
        String loc = locs.item(i).getTextContent().trim();
        if (!loc.isEmpty()) urls.add(loc);
    }
    return urls;
}
```

Note: sitemaps use the namespace `http://www.sitemaps.org/schemas/sitemap/0.9`. The
`getElementsByTagNameNS("*", "loc")` wildcard handles both namespaced and non-namespaced docs.

### 2b. Sitemap index (recursive)

A sitemap index contains `<sitemap><loc>…</loc></sitemap>` entries instead of `<url>`.
Detect it by the presence of `<sitemapindex>` as the root element.

```java
public static List<String> parseSitemapIndex(String xml) throws Exception {
    // Same parsing as standard, but <loc> entries are child sitemap URLs, not page URLs
    return parseStandardSitemap(xml); // returns the child sitemap URLs
}

public static boolean isSitemapIndex(String xml) {
    return xml.contains("<sitemapindex");
}

// Recursive discovery
public static List<String> discoverAllUrls(String sitemapUrl, HttpClient http) throws Exception {
    String xml = http.get(sitemapUrl);
    if (xml == null) return List.of();

    // Handle gzip
    if (sitemapUrl.endsWith(".gz")) {
        xml = decompress(http.getBytes(sitemapUrl));
    }

    if (isSitemapIndex(xml)) {
        List<String> childUrls = parseSitemapIndex(xml);
        List<String> allPageUrls = new ArrayList<>();
        for (String child : childUrls) {
            allPageUrls.addAll(discoverAllUrls(child, http)); // recurse
        }
        return allPageUrls;
    } else {
        return parseStandardSitemap(xml);
    }
}
```

### 2c. Gzip sitemap

```java
import java.util.zip.GZIPInputStream;

public static String decompress(byte[] compressed) throws IOException {
    try (GZIPInputStream gzip = new GZIPInputStream(new ByteArrayInputStream(compressed));
         ByteArrayOutputStream out = new ByteArrayOutputStream()) {
        byte[] buf = new byte[4096];
        int n;
        while ((n = gzip.read(buf)) != -1) out.write(buf, 0, n);
        return out.toString("UTF-8");
    }
}
```

---

## 3. robots.txt Parsing

```java
public static class RobotsResult {
    public final List<String> sitemapUrls;
    public final List<String> disallowedPaths; // for User-agent: * and our agent

    public RobotsResult(List<String> sitemapUrls, List<String> disallowedPaths) {
        this.sitemapUrls = sitemapUrls;
        this.disallowedPaths = disallowedPaths;
    }
}

public static RobotsResult parseRobotsTxt(String content, String ourUserAgent) {
    List<String> sitemaps = new ArrayList<>();
    List<String> disallowed = new ArrayList<>();
    boolean applicableSection = false;

    for (String line : content.split("\\n")) {
        line = line.trim();
        if (line.startsWith("#") || line.isEmpty()) continue;

        if (line.toLowerCase().startsWith("user-agent:")) {
            String agent = line.substring("user-agent:".length()).trim();
            applicableSection = agent.equals("*") || 
                                agent.equalsIgnoreCase(ourUserAgent);
        } else if (line.toLowerCase().startsWith("sitemap:")) {
            sitemaps.add(line.substring("sitemap:".length()).trim());
        } else if (line.toLowerCase().startsWith("disallow:") && applicableSection) {
            String path = line.substring("disallow:".length()).trim();
            if (!path.isEmpty()) disallowed.add(path);
        }
    }
    return new RobotsResult(sitemaps, disallowed);
}

// Check before crawling a URL
public static boolean isAllowed(String urlPath, List<String> disallowedPaths) {
    for (String rule : disallowedPaths) {
        if (urlPath.startsWith(rule)) return false;
    }
    return true;
}
```

---

## 4. BFS Crawl Loop Skeleton

Ties together all patterns above. Adapt field types to your project's HTTP/browser client.

```java
import java.util.*;
import java.util.regex.Pattern;

public class MultiPageCrawler {

    private final String seedUrl;
    private final int maxDepth;
    private final int maxPages;
    private final Pattern[] includePaths;     // null means allow all
    private final Pattern[] excludePaths;
    private final boolean allowBackwardCrawling;
    private final String userAgent = "CrawlerBot/1.0";

    public List<PageData> crawl() throws Exception {
        String domain = extractDomain(seedUrl);
        String basePath = extractPath(seedUrl);

        // Step 1: robots.txt
        String robotsTxt = fetchText("https://" + domain + "/robots.txt");
        RobotsResult robots = parseRobotsTxt(robotsTxt, userAgent);

        // Step 2: seed queue from sitemap if available
        Deque<CrawlNode> queue = new ArrayDeque<>();
        Set<String> visited = new HashSet<>();

        if (!robots.sitemapUrls.isEmpty()) {
            for (String sitemapUrl : robots.sitemapUrls) {
                List<String> sitemapUrls = discoverAllUrls(sitemapUrl, httpClient);
                for (String url : sitemapUrls) {
                    if (shouldCrawl(url, domain, basePath, robots, visited)) {
                        queue.add(new CrawlNode(url, 0));
                    }
                }
            }
        } else {
            queue.add(new CrawlNode(seedUrl, 0)); // start from seed
        }

        List<PageData> results = new ArrayList<>();

        // Step 3: BFS loop
        while (!queue.isEmpty() && results.size() < maxPages) {
            CrawlNode node = queue.poll();
            String canonical = canonicalize(node.url, seedUrl);

            if (visited.contains(canonical)) continue;
            visited.add(canonical);

            // Fetch using the complexity level determined in Step 2
            String html = fetch(node.url);

            // Content validation
            if (!isValidContent(html)) {
                log.warn("Invalid content at {}, skipping", node.url);
                continue;
            }

            // Extract and store data
            results.add(extractData(html, node.url));

            // Discover links for next depth level
            if (node.depth < maxDepth) {
                Document doc = Jsoup.parse(html, node.url);
                doc.select("a[href]").forEach(el -> {
                    String href = el.absUrl("href");
                    if (shouldCrawl(href, domain, basePath, robots, visited)) {
                        queue.add(new CrawlNode(href, node.depth + 1));
                    }
                });
            }
        }
        return results;
    }

    private boolean shouldCrawl(String url, String domain, String basePath,
                                  RobotsResult robots, Set<String> visited) {
        try {
            URI uri = new URI(url);
            // Must be same domain
            if (!uri.getHost().replaceFirst("^www\\.", "").equals(domain)) return false;
            // Must not be in visited
            if (visited.contains(canonicalize(url, seedUrl))) return false;
            // Respect robots.txt Disallow
            if (!isAllowed(uri.getPath(), robots.disallowedPaths)) return false;
            // Backward crawling check
            if (!allowBackwardCrawling && !uri.getPath().startsWith(basePath)) return false;
            // Include/exclude patterns
            if (includePaths != null) {
                boolean matched = Arrays.stream(includePaths)
                        .anyMatch(p -> p.matcher(uri.getPath()).find());
                if (!matched) return false;
            }
            if (excludePaths != null) {
                boolean excluded = Arrays.stream(excludePaths)
                        .anyMatch(p -> p.matcher(uri.getPath()).find());
                if (excluded) return false;
            }
            // Skip non-HTML resources
            if (url.matches(".*\\.(jpg|jpeg|png|gif|pdf|zip|css|js|woff2?)$")) return false;
            return true;
        } catch (Exception e) {
            return false;
        }
    }

    record CrawlNode(String url, int depth) {}
}
```
