---
title: "Frontend Security - XSS, CSRF, CSP & Trusted Types"
tags: [frontend, frontend/security]
aliases: [Frontend Security, XSS, CSRF, CSP, Trusted Types, Web Security]
updated: 2026-09-27
---
# Frontend Security - XSS, CSRF, CSP & Trusted Types

<Note>
**TL;DR**

The browser's security model rests on the **same-origin policy**. Most frontend attacks try to get around it: **XSS** runs the attacker's script *inside* your origin, **CSRF** makes the victim's browser send authenticated requests *to* your origin, and **clickjacking** frames your origin to trick clicks. Defences are layered: output encoding (framework escaping), sanitization, **CSP** with nonces + `strict-dynamic`, **Trusted Types**, **SameSite/HttpOnly cookies** plus CSRF tokens, `frame-ancestors`, **SRI**, and never shipping secrets to the client.
</Note>

## Threat map

{/* diagram source: /diagrams/diagram-326.mmd */}
![diagram](/diagrams/diagram-326.svg)

## Same-origin policy (SOP) in one paragraph
An **origin** = scheme + host + port. Scripts from one origin can't read data (DOM, responses, storage) from another origin. SOP doesn't stop a page from *sending* requests to other origins (forms, images, scripts), it stops it from *reading* the responses. That's why CSRF works (sending is allowed) and why CORS exists (to *relax* reading).

---

## XSS (Cross-Site Scripting)

The attacker gets their JavaScript to execute in your page. Once it runs, it can do anything your code can: read the DOM, read `localStorage` tokens, call your APIs with the user's cookies, log keystrokes, change the page.

| Type | Where the payload lives | Example |
|---|---|---|
| **Stored** | Saved in your DB, served to other users | Review text `<img src=x onerror=steal()>` shown to all visitors |
| **Reflected** | In the request (URL/query), echoed in the response | `/search?q=<script>…</script>` rendered by the server |
| **DOM-based** | Never touches the server; client JS moves data from a *source* to a *sink* | `el.innerHTML = location.hash.slice(1)` |

**Sources** (attacker-controlled): `location.*`, `document.referrer`, `postMessage` data, URL params, API data containing user content, `localStorage`.
**Sinks** (execute or parse HTML): `innerHTML`, `outerHTML`, `insertAdjacentHTML`, `document.write`, `eval`, `new Function`, `setTimeout(string)`, `<a href="javascript:…">`, `<iframe srcdoc>`, jQuery `$(html)`.

### Defence 1: framework escaping (output encoding)
React, Angular, Vue and Svelte **escape text interpolation by default** — the value becomes a text node, never HTML:

```jsx
const comment = '<img src=x onerror="alert(1)">';
<p>{comment}</p>   // renders the literal text, safe
```

The escape hatches are where bugs live:

| Framework | Dangerous API |
|---|---|
| React | `dangerouslySetInnerHTML={{ __html }}`, `href={userUrl}` with `javascript:` URLs (React warns/blocks `javascript:` URLs in recent versions, but validate anyway), refs + `innerHTML` |
| Angular | `bypassSecurityTrustHtml/Url/ResourceUrl/Script/Style`, `ElementRef.nativeElement.innerHTML` |
| Vue | `v-html` |
| Svelte | `{@html}` |
| Vanilla | `innerHTML`, `insertAdjacentHTML`, `document.write` |

<Info>
**Safe vs unsafe rendering of user content**

```jsx
// ❌ XSS if bio contains HTML
<div dangerouslySetInnerHTML={{ __html: user.bio }} />

// ✅ Plain text: just interpolate
<div>{user.bio}</div>

// ✅ Rich text you must render as HTML: sanitize first
import DOMPurify from 'dompurify';
<div dangerouslySetInnerHTML={{ __html: DOMPurify.sanitize(user.bioHtml) }} />

// ✅ URLs: allow-list the scheme
const safeHref = (url) => {
  try { return ['http:', 'https:', 'mailto:'].includes(new URL(url, location.href).protocol) ? url : '#'; }
  catch { return '#'; }
};
<a href={safeHref(user.website)}>Website</a>
```
</Info>

Vanilla DOM equivalents: use `textContent` (not `innerHTML`), `setAttribute` for known-safe attributes, and `createElement` to build markup.

### Defence 2: sanitization
When you must render user HTML (rich text, markdown output, CMS content), strip dangerous elements/attributes with an allow-list sanitizer:
- **DOMPurify** — the standard library; works everywhere today.
- **Sanitizer API** — the browser-native version: `el.setHTML(untrusted)` parses and sanitizes in one step (strips scripts and event handlers by default), with an optional config. **Status: Firefox 148 and Chrome 146 (early 2026), not Safari — Baseline Limited**, so feature-detect and fall back to DOMPurify.
- Sanitize on output (at render time), close to the sink. Server-side sanitization of stored content is a good extra layer.

```js
if ('setHTML' in Element.prototype) {
  el.setHTML(untrustedHtml);                 // native sanitizer
} else {
  el.innerHTML = DOMPurify.sanitize(untrustedHtml);
}
```

`Document.parseHTMLUnsafe()` (Baseline 2025-09) is the explicitly *unsafe* counterpart — the name tells reviewers no sanitization happens.

### Defence 3: Content Security Policy (CSP)
A response header telling the browser which scripts/styles/connections are allowed. It's a **second line of defence**: if an injection slips through, the injected script still won't run.

Modern best practice is a **strict CSP** based on **nonces** (or hashes) + `strict-dynamic`, not domain allow-lists (which are easily bypassed via JSONP endpoints or script gadgets on allowed CDNs).

<Info>
**Strict CSP with nonce + strict-dynamic**

```http
Content-Security-Policy:
  script-src 'nonce-R4nd0mP3rR3qu3st' 'strict-dynamic' https: 'unsafe-inline';
  object-src 'none';
  base-uri 'none';
  frame-ancestors 'self';
  require-trusted-types-for 'script';
  report-to csp-endpoint
```
```html
<script nonce="R4nd0mP3rR3qu3st" src="/assets/app.js"></script>
<!-- app.js can load more scripts: strict-dynamic trusts scripts created by trusted scripts -->
```
- **Nonce**: a fresh random value per response; only `<script>` tags carrying it run. Injected `<script>` or inline `onclick` handlers don't have it → blocked.
- **`'strict-dynamic'`**: scripts loaded *by* a nonced script are trusted too, so bundlers' dynamic chunks work; host allow-lists are ignored in supporting browsers.
- `https:` and `'unsafe-inline'` are **fallbacks for old browsers only** — they're ignored when a nonce/`strict-dynamic` is present.
- `object-src 'none'` blocks plugins; `base-uri 'none'` stops `<base>` tag hijacking of relative URLs.
- Roll out with `Content-Security-Policy-Report-Only` first and collect reports.
</Info>

Framework support: Next.js can generate a nonce per request in middleware/`proxy.ts`; Angular supports `ngCspNonce` / `CSP_NONCE`; Astro 6 made CSP support stable. Static sites use **hashes** (`'sha256-…'`) since there's no per-request server.

Other useful directives: `connect-src` (fetch/XHR/WebSocket targets), `img-src`, `style-src`, `upgrade-insecure-requests`, `form-action`.

### Defence 4: Trusted Types
DOM XSS happens when a **string** reaches a sink. Trusted Types make dangerous sinks **refuse plain strings** and only accept typed objects (`TrustedHTML`, `TrustedScript`, `TrustedScriptURL`) created by **policies** you define and review.

```http
Content-Security-Policy: require-trusted-types-for 'script'; trusted-types app-sanitizer default
```

```js
const policy = trustedTypes.createPolicy('app-sanitizer', {
  createHTML: (input) => DOMPurify.sanitize(input),
  createScriptURL: (url) => {
    const u = new URL(url, location.href);
    if (u.origin === location.origin) return u.href;
    throw new TypeError('Untrusted script URL: ' + url);
  },
});

el.innerHTML = policy.createHTML(userHtml); // ✅ allowed
el.innerHTML = userHtml;                    // ❌ TypeError: sink requires TrustedHTML
```

- A policy named **`default`** is called automatically for plain strings — handy for migrating legacy code and third-party libraries.
- It turns "audit every `innerHTML` in the codebase" into "audit the few policies".
- Angular has built-in Trusted Types support; **React 19.3 added Trusted Types support in React DOM**.

### XSS impact on tokens
If XSS runs, anything readable by JS (`localStorage`, `sessionStorage`, non-HttpOnly cookies, in-memory variables) can be stolen, and the attacker can call your APIs *as the user* even with HttpOnly cookies. **HttpOnly reduces token theft; it doesn't prevent XSS.** See [Authentication & Token Storage](/Software%20Engineering/Frontend/11%20Security/Authentication%20%26%20Token%20Storage).

---

## CSRF (Cross-Site Request Forgery)

The victim is logged in to `bank.com` (session cookie). They visit `evil.com`, which auto-submits a form to `bank.com/transfer`. The browser **attaches the bank cookie automatically**, so the request looks legitimate. The attacker can't read the response, but the side effect (transfer, email change) happens.

{/* diagram source: /diagrams/diagram-327.mmd */}
![diagram](/diagrams/diagram-327.svg)

### Defences
1. **`SameSite` cookies**
   - `Strict` – never sent on cross-site requests (even top-level link clicks → user appears logged out when arriving from another site).
   - `Lax` – sent on top-level **GET** navigations, not on cross-site POSTs, iframes or fetches. Good default for session cookies.
   - `None` – sent everywhere; requires `Secure`. Needed for real cross-site use (embedded widgets, some SSO).
   - Chrome treats cookies without a `SameSite` attribute as `Lax`, but **don't rely on browser defaults** — set it explicitly.
   - "Site" = registrable domain (eTLD+1): `a.example.com` and `b.example.com` are **same-site** (but cross-origin), so SameSite doesn't protect against a compromised sibling subdomain.
2. **CSRF tokens** (synchronizer token or double-submit cookie) for state-changing requests; frameworks (Angular `HttpClient` XSRF support, Rails, Django, Next.js Server Actions' origin checks) do this for you.
3. **Check `Origin` / `Sec-Fetch-Site` headers** on the server for state-changing requests.
4. **Never change state on GET.**
5. Custom headers (`X-Requested-With`) or `Content-Type: application/json` trigger a CORS preflight for cross-origin requests, which adds protection — but only if your CORS config is strict.

Token-in-`Authorization`-header auth is **not** CSRF-prone (the browser doesn't attach it automatically) — but the token is then readable by JS, so XSS becomes the risk. That's the core trade-off in [JWT vs Cookies](/Software%20Engineering/APIs%20%26%20Auth/JWT%20vs%20Cookies).

---

## Clickjacking
The attacker loads your site in an invisible `<iframe>` over a decoy page ("Click to win!") so the user clicks your "Delete account" button.

```http
Content-Security-Policy: frame-ancestors 'self' https://partner.example.com
X-Frame-Options: DENY            # legacy fallback (DENY or SAMEORIGIN only)
```

`frame-ancestors` supersedes `X-Frame-Options` and can't be set via `<meta>` — it must be a header.

---

## CORS: common misconceptions

CORS (Cross-Origin Resource Sharing) lets a server **opt in** to letting other origins **read** its responses (`Access-Control-Allow-Origin`, `-Credentials`, `-Methods`, `-Headers`, preflight `OPTIONS`).

| Misconception | Reality |
|---|---|
| "CORS protects my API." | CORS protects **users' browsers** from other sites reading responses. `curl`, Postman and servers ignore it. Authenticate and authorise every request. |
| "CORS prevents CSRF." | Simple requests (form POST, `text/plain`) are **sent** without preflight; CORS only blocks reading the response. The side effect still happens. |
| "Just set `*` to fix the error." | `*` can't be used with credentials, and reflecting any `Origin` with `Allow-Credentials: true` lets any site read authenticated data. Allow-list exact origins. |
| "A CORS error is a frontend bug." | It's a server config issue (or you should proxy through your own backend / BFF). |
| "Preflight happens for every request." | Only for non-simple requests (custom headers, JSON content type, PUT/DELETE…), and it can be cached with `Access-Control-Max-Age`. |

---

## Subresource Integrity (SRI)
Pin a hash of third-party scripts/styles so a compromised CDN can't swap the file:

```html
<script src="https://cdn.example.com/lib@4.2.0/lib.min.js"
        integrity="sha384-oqVuAfXRKap7fdgcCY5uykM6+R9GqQ8K/uxy9rx7HNQlGYl1kPzQho1wx4JwY8wC"
        crossorigin="anonymous"></script>
```
If the hash doesn't match, the browser refuses to run it. Only works for **versioned, immutable** URLs. Better still: self-host and bundle. See [Supply Chain Security](/Software%20Engineering/Frontend/11%20Security/Supply%20Chain%20Security).

---

## Secrets in client bundles
**Everything shipped to the browser is public.** Minification is not protection.
- Build tools inline env vars with public prefixes into the bundle: `NEXT_PUBLIC_*` (Next.js), `VITE_*` (Vite), `REACT_APP_*` (CRA). Anything with that prefix is visible in DevTools.
- Never put API secrets, DB credentials, private signing keys or third-party secret keys in frontend code. Call third-party APIs through your server/BFF.
- Publishable keys (Stripe `pk_…`, Maps keys restricted by referrer) are designed to be public — restrict them by domain and scope.
- Source maps in production can expose original source; upload them to your error tracker privately instead of serving them publicly.
- With RSC/Server Components, keep server-only modules out of the client with `import 'server-only'`, and remember Server Function arguments are **untrusted input**.

---

## Angular's security model (brief)
- Treats **all values as untrusted by default** and sanitizes them per context: HTML (`[innerHTML]`), style, URL (`href`, `src`), resource URL (`<script src>`, iframe `src`).
- Unsafe values are stripped with a console warning; `DomSanitizer.bypassSecurityTrust*` explicitly opts out — treat every call as a security review item.
- **AOT template compilation** prevents template injection (never build templates from user input).
- Built-in **XSRF** support in `HttpClient` (reads the `XSRF-TOKEN` cookie, sends `X-XSRF-TOKEN` header).
- Supports **Trusted Types** and **CSP nonces**.

See [Angular Fundamentals](/Software%20Engineering/Frontend/08%20Angular/Angular%20Fundamentals).

---

## Other headers worth knowing

| Header | Purpose |
|---|---|
| `Strict-Transport-Security: max-age=63072000; includeSubDomains; preload` | Force HTTPS (HSTS) |
| `X-Content-Type-Options: nosniff` | Stop MIME sniffing (e.g. treating an upload as script) |
| `Referrer-Policy: strict-origin-when-cross-origin` | Don't leak full URLs (tokens in query strings!) to other sites |
| `Permissions-Policy: camera=(), geolocation=()` | Disable powerful features you don't use |
| `Cross-Origin-Opener-Policy: same-origin` | Isolate your window from cross-origin popups (tab-nabbing, Spectre) |

Also: `rel="noopener"` is implied for `target="_blank"` links in modern browsers; validate `postMessage` `event.origin`; avoid open redirects (`?next=https://evil.com`).

<Warning>
**Common pitfalls**

- Thinking React makes you XSS-proof — `dangerouslySetInnerHTML`, `javascript:` URLs, SSR-injected JSON (`<script>window.__DATA__ = ${JSON.stringify(data)}</script>` without escaping `</script>`), and third-party scripts still bite.
- CSP with `'unsafe-inline'` and no nonce → nearly useless against XSS.
- Storing long-lived tokens in `localStorage` "because cookies have CSRF" — trading one risk for a worse one.
- `Access-Control-Allow-Origin` reflecting any origin with credentials.
- Secrets behind a `NEXT_PUBLIC_`/`VITE_` prefix.
- Treating Server Actions/Functions as private — they're public HTTP endpoints; validate and authorise inside them.
</Warning>

<Info>
**🆕 2025–2026**

- **Trusted Types are Baseline Newly available (2026-02-24)** with Firefox 148 — now usable across all major engines.
- **Sanitizer API `setHTML()`** shipped in **Firefox 148 and Chrome 146** (early 2026); no Safari → Baseline Limited, feature-detect.
- **`Document.parseHTMLUnsafe()`** is Baseline (2025-09).
- **React 19.3 (2026-09-09)** added Trusted Types support in React DOM.
- **CVE-2025-55182** (2025-12-03): CVSS 10.0 unauthenticated **RCE** via React Server Components' Server Function payload deserialization (`react-server-dom-*` 19.0–19.2.0; fixed in 19.0.1 / 19.1.2 / 19.2.1). Apps supporting RSC were affected even without explicit Server Functions; it hit Next.js (CVE-2025-66478), React Router, Waku, Parcel and Vite RSC. Follow-up advisory on 2025-12-11 for DoS and Server Function source exposure (fixed in 19.0.4 / 19.1.5 / 19.2.4). Community nickname "React2Shell" (not used in React's post).
- **Next.js** (2026-09-22): critical RCE in Node `ImageResponse` (`next/og`) via Satori and upstream deps, affecting `>=16.2.0 <16.3.6`; fixed in 16.3.6.
- **Astro 6** made its CSP support stable.
- Lesson: the "frontend" framework now includes a server attack surface — patch framework versions quickly.
</Info>

## Interview questions

<Accordion title="What are the types of XSS and how do you prevent them?">
Stored, reflected and DOM-based. Prevent by encoding output (framework escaping, `textContent`), avoiding HTML sinks or sanitizing with DOMPurify / `setHTML`, validating URL schemes, and adding defence in depth with a strict nonce-based CSP and Trusted Types. HttpOnly cookies limit token theft if XSS happens.
</Accordion>

<Accordion title="Is React safe from XSS by default?">
Mostly for text interpolation, which is escaped. But `dangerouslySetInnerHTML`, user-controlled `href`/`src` (`javascript:`), direct DOM manipulation via refs, unsafely serialized SSR state and third-party scripts can still introduce XSS.
</Accordion>

<Accordion title="Explain CSRF and how SameSite cookies help.">
The attacker's site makes the victim's browser send a request to your site with the victim's cookies attached. `SameSite=Lax/Strict` stops cookies being sent on cross-site subrequests and POSTs. Add CSRF tokens or Origin/`Sec-Fetch-Site` checks for defence in depth, and never change state on GET.
</Accordion>

<Accordion title="What is a strict CSP and why nonces over allow-lists?">
A CSP where scripts run only if they carry a per-response nonce (or match a hash), plus `strict-dynamic` so trusted scripts can load their dependencies, and `object-src 'none'`, `base-uri 'none'`. Host allow-lists are bypassable through JSONP endpoints and gadgets on allowed domains; nonces tie trust to your own markup.
</Accordion>

<Accordion title="What are Trusted Types?">
A browser feature (enabled via CSP `require-trusted-types-for 'script'`) that makes DOM XSS sinks like `innerHTML` reject strings and accept only typed values produced by reviewed policies. It shrinks the audit surface to the policies. Baseline since Feb 2026.
</Accordion>

<Accordion title="Does CORS protect against CSRF?">
No. CORS controls whether a cross-origin page can *read* a response. Simple cross-site requests are still sent (and have side effects). Use SameSite cookies, CSRF tokens and Origin checks.
</Accordion>

<Accordion title="How do you prevent clickjacking?">
Send `Content-Security-Policy: frame-ancestors 'self'` (or specific partners), with `X-Frame-Options: DENY/SAMEORIGIN` as a legacy fallback.
</Accordion>

<Accordion title="What is SRI?">
Subresource Integrity: an `integrity="sha384-…"` attribute on `<script>`/`<link>` so the browser refuses a third-party file whose hash doesn't match — protecting against a compromised CDN.
</Accordion>

## Related
- [Authentication & Token Storage](/Software%20Engineering/Frontend/11%20Security/Authentication%20%26%20Token%20Storage)
- [Supply Chain Security](/Software%20Engineering/Frontend/11%20Security/Supply%20Chain%20Security)
- [JWT vs Cookies](/Software%20Engineering/APIs%20%26%20Auth/JWT%20vs%20Cookies)
- [Authentication vs Authorization](/Software%20Engineering/APIs%20%26%20Auth/Authentication%20vs%20Authorization)
- [Browser Storage - Cookies, Web Storage & IndexedDB](/Software%20Engineering/Frontend/03%20Browser%20%26%20Web%20Platform/Browser%20Storage%20-%20Cookies%2C%20Web%20Storage%20%26%20IndexedDB)
- [Networking - HTTP, Caching & Real-time](/Software%20Engineering/Frontend/03%20Browser%20%26%20Web%20Platform/Networking%20-%20HTTP%2C%20Caching%20%26%20Real-time)
- [React Server Components & Server Functions](/Software%20Engineering/Frontend/05%20React/React%20Server%20Components%20%26%20Server%20Functions)
- [Angular Fundamentals](/Software%20Engineering/Frontend/08%20Angular/Angular%20Fundamentals)
- [Modern Web Platform APIs 2025-2026](/Software%20Engineering/Frontend/03%20Browser%20%26%20Web%20Platform/Modern%20Web%20Platform%20APIs%202025-2026)

## Sources
- [OWASP – XSS Prevention Cheat Sheet](https://cheatsheetseries.owasp.org/cheatsheets/Cross_Site_Scripting_Prevention_Cheat_Sheet.html)
- [OWASP – CSRF Prevention Cheat Sheet](https://cheatsheetseries.owasp.org/cheatsheets/Cross-Site_Request_Forgery_Prevention_Cheat_Sheet.html)
- [web.dev – Strict CSP](https://web.dev/articles/strict-csp)
- [web.dev – Trusted Types](https://web.dev/articles/trusted-types)
- [MDN – Content Security Policy](https://developer.mozilla.org/en-US/docs/Web/HTTP/CSP)
- [MDN – Trusted Types API](https://developer.mozilla.org/en-US/docs/Web/API/Trusted_Types_API)
- [MDN – HTML Sanitizer API](https://developer.mozilla.org/en-US/docs/Web/API/HTML_Sanitizer_API)
- [MDN – CORS](https://developer.mozilla.org/en-US/docs/Web/HTTP/CORS)
- [MDN – Subresource Integrity](https://developer.mozilla.org/en-US/docs/Web/Security/Subresource_Integrity)
- [angular.dev – Security](https://angular.dev/best-practices/security)
- [React – Critical Security Vulnerability in RSC (2025-12-03)](https://react.dev/blog/2025/12/03/critical-security-vulnerability-in-react-server-components)
- [React – DoS and Source Code Exposure in RSC (2025-12-11)](https://react.dev/blog/2025/12/11/denial-of-service-and-source-code-exposure-in-react-server-components)
- [React 19.3 release](https://react.dev/blog/2026/09/09/react-19-3)
- [Next.js security update, Sept 22 2026](https://nextjs.org/blog/nextjs-security-update-september-22-2026)
