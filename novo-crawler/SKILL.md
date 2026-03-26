---
name: novo-crawler
description: >
  Guia para implementar um novo crawler que coleta dados de um site externo via HTTP
  ou browser headless. Use quando o usuário pedir para criar, adicionar ou integrar
  um novo site/fonte de dados ao sistema de crawling. Carregue automaticamente ao
  iniciar qualquer tarefa de novo crawler ou integração com site externo.
user-invocable: true
disable-model-invocation: false
---

Você está implementando um novo crawler para coletar dados de um site externo.
Siga rigorosamente este processo antes de escrever qualquer linha de código de produção.

---

## Etapa 1 — Reconhecimento (obrigatório antes de qualquer código)

Peça ao usuário, ou investigue com DevTools/curl, as seguintes informações:

1. **URL do site-alvo** — qual é o site a ser crawleado?
2. **Tipo de dado desejado** — o que deve ser extraído? (eventos, preços, produtos, artigos…)
3. **Identificação da fonte de dados** — abrir DevTools > Network > XHR/Fetch no site e responder:
   - Existe uma URL de API que retorna JSON diretamente?
   - Essa URL funciona com `curl` simples, sem cookies?
   - Quais headers a requisição usa? (`Referer`, `Origin`, `x-api-key`, cookies…)
   - O conteúdo está no HTML renderizado (SSR) ou carregado por JS (SPA)?

Com base nessas respostas, classifique o site em um dos níveis abaixo antes de continuar.

---

## Etapa 2 — Classificação por Nível de Complexidade

Use o nível **mais baixo que funcione**. Nunca pule para um nível maior sem confirmar que o anterior falha.

### Nível 1 — HTTP simples + JSON API pública
**Quando:** A URL de API retorna JSON sem autenticação e funciona com `curl` sem headers extras.

**Diagnóstico:**
```bash
curl -s "https://api.exemplo.com/dados" | head -c 500
# Deve retornar JSON com dados reais, não página de erro nem HTML
```

**Implementação-modelo (Java):**
```java
// Apenas uma URL → JsonObject ou biblioteca HTTP simples
JsonObject response = new JsonObject("https://api.exemplo.com/dados");
```

**Riscos:** Contrato da API pode mudar sem aviso. Parâmetros de versão/token podem rotacionar.

---

### Nível 2 — HTTP com headers obrigatórios
**Quando:** A API existe, mas retorna 403/429/vazio sem headers específicos (`Referer`, `Origin`, `User-Agent`, `x-api-key`, cookies de sessão estáticos).

**Diagnóstico:**
```bash
curl -s -H "Referer: https://www.exemplo.com/" \
     -H "Origin: https://www.exemplo.com" \
     -H "User-Agent: Mozilla/5.0 ..." \
     "https://api.exemplo.com/dados" | head -c 500
# Se retornar dados: é Nível 2
```

**Implementação-modelo (Java):**
```java
Map<String, String> headers = Map.of(
    "User-Agent", "Mozilla/5.0 (Windows NT 10.0; Win64; x64) ...",
    "Referer",    "https://www.exemplo.com/",
    "Origin",     "https://www.exemplo.com",
    "Accept",     "application/json, */*",
    "sec-fetch-site", "same-origin",
    "sec-fetch-mode", "cors",
    "sec-fetch-dest", "empty"
);
JsonObject response = JsonObject.fromUrl(url, headers);
```

**Riscos:** Tokens de autenticação rotativos exigem lógica de renovação. Revisar headers periodicamente.

---

### Nível 3 — Playwright headless com warm-up de sessão
**Quando:** Curl com headers falha, mas um browser headless que visita a homepage antes consegue sessão válida. Proteção leve: Cloudflare JS Challenge, cookies gerados via JS.

**Diagnóstico:**
```
1. Nível 2 falhou mesmo com todos os headers copiados do DevTools
2. Playwright sem stealth, após navegar para a homepage, consegue chamar a API
```

**Implementação-modelo (Java + Playwright):**
```java
Browser browser = playwright.chromium().launch(new LaunchOptions().setHeadless(true));
BrowserContext context = browser.newContext();

// warm-up: estabelece sessão navegando até a homepage
Page page = context.newPage();
page.navigate("https://www.exemplo.com");
page.close();

// agora usa os cookies da sessão para chamar a API
APIResponse resp = context.request().get("https://www.exemplo.com/api/dados");
String json = resp.text();
```

**Riscos:** 1–3 s por sessão; alto consumo de memória com muitos sites simultâneos. Chromium precisa estar instalado.

---

### Nível 4 — Playwright com stealth + extração de DOM
**Quando:** Warm-up simples não é suficiente. O conteúdo está no DOM (não em API JSON) ou há proteção mais sofisticada (Cloudflare Managed Challenge, fingerprinting avançado).

**Diagnóstico:**
```
1. Nível 3 retorna página de erro, CAPTCHA ou HTML vazio
2. O conteúdo aparece normalmente em um browser real
```

**Implementação-modelo (Java + Playwright + Jsoup):**
```java
BrowserType.LaunchOptions opts = new BrowserType.LaunchOptions()
    .setHeadless(false)                           // ou Xvfb em servidor
    .setArgs(List.of("--disable-blink-features=AutomationControlled"));
Browser browser = playwright.firefox().launch(opts); // Firefox tem menor fingerprint

Page page = browser.newPage();
page.navigate("https://www.exemplo.com/lista");
page.waitForLoadState(LoadState.NETWORKIDLE);    // aguarda JS terminar

// extrai HTML renderizado e parseia com Jsoup
Document doc = Jsoup.parse(page.content());
Elements itens = doc.select(".item-seletor");
```

**Ferramentas adicionais:** playwright-stealth, camoufox, nodriver.

**Riscos:** 5–15 s por página; seletores CSS são frágeis e quebram com redesign.

---

### Nível 5 — Playwright + proxy residencial + comportamento simulado
**Quando:** Proteções sérias: DataDome, PerimeterX, Imperva, reCAPTCHA v3 com score comportamental. Detectam headless mesmo com stealth.

**Diagnóstico:**
```
Nível 4 ainda é bloqueado após poucas requisições ou exige CAPTCHA interativo.
```

**Implementação-modelo (Java + Playwright + proxy):**
```java
BrowserType.LaunchOptions opts = new BrowserType.LaunchOptions()
    .setProxy(new Proxy("http://user:pass@proxy-residencial:8080"));

// Simular comportamento humano
page.mouse().move(randomX, randomY);
page.keyboard().type(texto, new Keyboard.TypeOptions().setDelay(120));
Thread.sleep(randomBetween(800, 2400));
page.evaluate("window.scrollBy(0, " + randomScroll + ")");
```

**Ferramentas:** Proxies residenciais rotativos (Brightdata, Oxylabs, Smartproxy), resolvedores de CAPTCHA, Browserless.io.

**Custo:** Alto (~$2–15/GB de proxy). Só usar se os benefícios justificarem. Avaliar custo-benefício antes de implementar.

---

## Etapa 3 — Implementação (TDD obrigatório)

Siga o processo da skill `tdd` antes de escrever qualquer código de produção.

Para crawlers, o ponto de partida do teste deve ser:
- Capturar uma resposta JSON real do site e salvar em `src/test/resources/json/NomeSite/resposta.json`
- Escrever o teste do mapper usando essa fixture (comportamento esperado: campos mapeados corretamente)
- Só então implementar `CrawlerNomeSite` e `NomeSiteJsonMapper`

Estrutura recomendada:
```
crawler/
  CrawlerNomeSite.java       ← implementa ICrawler (ou subclasse da abstração compartilhada)
json/
  NomeSiteJsonMapper.java    ← converte JSON bruto → entidades do domínio
src/test/resources/json/NomeSite/
  resposta.json              ← fixture real capturada do site
```

---

## Etapa 4 — Checklist de qualidade antes de finalizar

- [ ] O crawler retorna lista vazia (não lança exceção) quando o site está indisponível
- [ ] Erros são logados com `log.warn(...)`, não propagados para cima
- [ ] A URL da API e os headers estão em constantes nomeadas, não inline
- [ ] Existe pelo menos um teste com fixture de JSON real
- [ ] O novo crawler está registrado no container/lista de crawlers ativos
- [ ] O site está cadastrado na base de dados (ou criado como transient se não persistente)

---

## Referências de nível no projeto

| Nível | Exemplo no projeto | Classe-base |
|---|---|---|
| 1 | Goldebet, Bateu, Esportiva… | `AbstractAltenarCrawler` |
| 2 | Betano | `ICrawler` direto com `JsonObject.fromUrl(url, headers)` |
| 3 | SportingBet, Betway | `PlaywrightHeadlessBrowserClient` |
| 4–5 | (não implementado) | Playwright + stealth/proxy |
