---
title: "Authentication & Token Storage"
tags: [frontend, frontend/security, frontend/auth]
aliases: [Authentication, Auth, Token Storage, OAuth, PKCE, JWT storage, Passkeys]
updated: 2026-09-27
---
# Authentication & Token Storage

<Note>
**TL;DR**

**Authentication** proves *who* you are; **authorization** decides *what* you may do. Web apps authenticate with **server sessions** (cookie holds an opaque ID), **tokens** (usually JWTs), or delegate to an identity provider with **OAuth 2.0 / OpenID Connect** (Authorization Code + **PKCE**). Where you keep the credential is a security decision: **HttpOnly, Secure, SameSite cookies** (ideally via a **BFF**) resist XSS token theft; `localStorage` does not. **Passkeys (WebAuthn)** are replacing passwords.
</Note>

## Authentication vs authorization

<Tip>
**💡**

- **Authentication (authn)** → verifying identity ("Are you really Sara?").
- **Authorization (authz)** → verifying role and determining the actions and permissions ("Can Sara cancel this booking?").
</Tip>

- Authn happens once per session; authz happens on **every request/action**.
- The frontend can *hide* UI based on roles, but **authorization must be enforced on the server** — a hidden button is not access control.
- HTTP 401 = not authenticated; 403 = authenticated but not allowed.

More: [Authentication vs Authorization](/Software%20Engineering/APIs%20%26%20Auth/Authentication%20vs%20Authorization).

## HTTP is stateless
HTTP doesn't remember previous requests, so various state management mechanisms carry identity between requests:
- **Cookies** – small pieces of data the server stores in the browser (`Set-Cookie`); the browser sends them back automatically with every matching request (same domain/path, respecting `SameSite`, `Secure`). Used for session management and preferences.
- **Local Storage** – key-value storage in the browser (~5–10 MB per origin), used for preferences, caching and app state. Only readable by JS; never sent automatically.
- **Session Storage** – same API as Local Storage but scoped to one **tab** and cleared when that tab's page session ends (closing the tab). Used for temporary data.
- **IndexedDB** – a low-level, async API for storing structured data, files and blobs.

Full comparison: [Browser Storage - Cookies, Web Storage & IndexedDB](/Software%20Engineering/Frontend/03%20Browser%20%26%20Web%20Platform/Browser%20Storage%20-%20Cookies%2C%20Web%20Storage%20%26%20IndexedDB).

---

## Types of authentication

### 1. Session-based

A session is created on the server for each user after they log in; the server sends a unique **session ID** (stored as a cookie) to the client.

{/* diagram source: /diagrams/diagram-323.mmd */}
![diagram](/diagrams/diagram-323.svg)

**Workflow**
- When a user logs in, the server generates a unique session ID and stores session data (user ID, expiration time, etc.) server-side.
- The server sets a cookie in the response containing the session ID.
- For subsequent requests, the browser sends the cookie automatically.
- The server verifies the session ID against the stored session data.

| Pros | Cons |
|---|---|
| **Secure**: data stays on the server; the cookie is an opaque random ID, useless if decoded | **Scalability**: every request needs a session lookup; in distributed systems you need a shared store (Redis) or sticky sessions |
| **Easy**: built into most frameworks | **Server-side storage**: memory/DB load |
| **Automatic expiration** and **instant revocation** (delete the session → logged out everywhere) | **CSRF**: cookies are sent automatically, so add `SameSite` + CSRF tokens |

### 2. Token-based (JWT)

A token containing user information (claims) is generated at login and returned to the client **in the response body** (or set as a cookie).

A **JWT** = `base64url(header).base64url(payload).signature`:
```json
// header
{ "alg": "RS256", "typ": "JWT", "kid": "2026-key-1" }
// payload (claims) — readable by anyone!
{ "sub": "42", "role": "agent", "iat": 1790000000, "exp": 1790000900, "iss": "https://auth.example.com", "aud": "bookings-api" }
```

**Workflow**
- After successful authentication, the server creates a JWT with the user's claims and **signs** it (HMAC secret or private key).
- The server sends the JWT to the client (response body, header or cookie).
- The client sends it with each request, typically `Authorization: Bearer <token>`.
- The server verifies the signature, `exp`, `iss`, `aud` — no DB lookup needed.

| Pros | Cons |
|---|---|
| **Stateless**: all needed information is in the token, no server-side session storage | **Revocation is hard**: a valid token works until `exp` → keep access tokens short-lived (5–15 min) + refresh tokens, or keep a deny-list (which reintroduces state) |
| **Scalable**: any server/service with the public key can verify it (good for microservices, edge) | **Size**: bigger than a session ID, sent on every request |
| **Tamper-proof**: the signature detects modification | **Readable**: the payload is only base64url-encoded, **not encrypted** — never put secrets/PII in it (encryption requires JWE, rarely used) |
| Works across domains and for mobile/native clients | **Theft**: whoever holds it is the user (bearer token). HTTPS is mandatory, and storage location matters (below) |

<Warning>
**JWT pitfalls**

- Accepting `alg: none` or letting the token choose the algorithm — pin the expected algorithm.
- Not validating `exp`, `aud`, `iss`.
- Long-lived access tokens with no revocation.
- Decoding a JWT in the browser (`jwt-decode`) is fine for *reading* claims for UI — but the frontend can't *verify* it and must never make security decisions on it.
</Warning>

Deep dive: [JWT vs Cookies](/Software%20Engineering/APIs%20%26%20Auth/JWT%20vs%20Cookies) — and note that "JWT vs cookies" is a false dichotomy: a JWT is a *format*, a cookie is a *transport/storage*. You can put a JWT in an HttpOnly cookie.

### 3. OAuth 2.0 and OpenID Connect

**OAuth 2.0** is a protocol for **delegated authorization**: it lets a third-party app access a user's resources **without seeing the user's credentials** ("Allow TravelApp to read your Google Calendar"). **OpenID Connect (OIDC)** is a thin identity layer on top that adds an **ID token** (a JWT about *who* logged in) — "Sign in with Google" is OIDC, not plain OAuth.

**Roles:** Resource Owner (user), Client (your app), Authorization Server (Google, Auth0, Keycloak, Entra ID), Resource Server (the API).

**Workflow (owner's summary)**
- Register your application with the OAuth provider and obtain client credentials (client ID, and a client secret for confidential server-side clients only).
- Redirect users to the provider's authorization endpoint.
- After authentication and consent, the provider redirects back to your app with an **authorization code**.
- Exchange the code for an **access token** (and optionally a **refresh token**, and with OIDC an **ID token**).
- Use the access token to call the resource API on behalf of the user.

| Pros | Cons |
|---|---|
| **SSO**: one identity across many services (via OIDC) | **Complexity**: many flows, parameters and edge cases — use a certified library |
| **Granular permissions** via scopes (`calendar.read`) | **Trust dependency** on the provider; privacy concerns |
| **Widely supported** standard | **Token management**: access + refresh tokens, rotation, expiry |

#### Authorization Code flow with PKCE (the one to use for SPAs and mobile)
Browsers and mobile apps are **public clients**: they can't keep a client secret. **PKCE** (Proof Key for Code Exchange) replaces the secret with a one-time random **code verifier**, so a stolen authorization code is useless. The old **Implicit flow** (token in the URL fragment) is deprecated — the OAuth 2.0 Security Best Current Practice (RFC 9700) and the OAuth 2.1 draft require PKCE for all clients.

{/* diagram source: /diagrams/diagram-324.mmd */}
![diagram](/diagrams/diagram-324.svg)

<Info>
**Generating the PKCE pair in the browser**

```js
function base64url(bytes) {
  return btoa(String.fromCharCode(...new Uint8Array(bytes)))
    .replace(/\+/g, '-').replace(/\//g, '_').replace(/=+$/, '');
}

async function createPkce() {
  const verifier = base64url(crypto.getRandomValues(new Uint8Array(32)));
  const digest = await crypto.subtle.digest('SHA-256', new TextEncoder().encode(verifier));
  return { verifier, challenge: base64url(digest) };
}

const { verifier, challenge } = await createPkce();
const state = crypto.randomUUID();
sessionStorage.setItem('pkce', JSON.stringify({ verifier, state }));

location.assign('https://auth.example.com/authorize?' + new URLSearchParams({
  response_type: 'code', client_id: 'travel-spa', redirect_uri: location.origin + '/callback',
  scope: 'openid profile bookings.read', state,
  code_challenge: challenge, code_challenge_method: 'S256',
}));
```
In practice use a vetted library (oidc-client-ts, Auth.js, the provider's SDK) — or better, do the whole exchange server-side in a BFF.
</Info>

### Refresh tokens and rotation
- **Access token**: short-lived (minutes), sent to APIs.
- **Refresh token**: long-lived, used *only* with the authorization server to get new access tokens.
- **Rotation**: every refresh returns a **new** refresh token and invalidates the old one. If an old refresh token is ever reused, the server assumes theft and **revokes the whole token family** (reuse detection).
- Refresh silently before expiry, and handle concurrent 401s with a single in-flight refresh:

```js
let refreshing = null;

async function apiFetch(url, opts = {}) {
  let res = await fetch(url, { ...opts, credentials: 'include' });
  if (res.status !== 401) return res;
  refreshing ??= fetch('/auth/refresh', { method: 'POST', credentials: 'include' })
    .finally(() => { refreshing = null; });
  const ok = (await refreshing).ok;
  if (!ok) { location.assign('/login'); throw new Error('Session expired'); }
  return fetch(url, { ...opts, credentials: 'include' }); // retry once
}
```
(This is the "interceptor" pattern — Axios interceptors or Angular `HttpInterceptor` do the same.)

### BFF (Backend for Frontend) pattern
The current best practice for browser apps: **tokens never reach the browser**.

{/* diagram source: /diagrams/diagram-325.mmd */}
![diagram](/diagrams/diagram-325.svg)

- The BFF is a confidential OAuth client, holds access/refresh tokens server-side, and gives the browser only an **HttpOnly session cookie**.
- XSS can't exfiltrate tokens (it can still act as the user while the page is open — XSS is always bad).
- Protect the cookie against CSRF with `SameSite` + CSRF token/Origin check.
- Next.js route handlers/Server Actions, Remix loaders and Angular SSR servers are natural BFFs.

### Passkeys (WebAuthn / FIDO2)
Passwordless login using **public-key cryptography**: the device creates a key pair per site; the private key never leaves the device (or the user's synced password manager), unlocked with biometrics/PIN.
- **Phishing-resistant**: credentials are bound to the site's origin (RP ID), so a fake `bank-login.com` can't use them.
- Nothing reusable is stored on the server (only public keys) → breaches leak nothing useful.
- API: `navigator.credentials.create({ publicKey })` to register, `navigator.credentials.get({ publicKey })` to sign in; the server sends a random **challenge**, the authenticator signs it.
- Conditional UI (`mediation: 'conditional'` + `autocomplete="username webauthn"`) shows passkeys in the username field's autofill.

```js
// Sign in (options come from your server; challenge is random per attempt)
const options = await (await fetch('/webauthn/login-options')).json();
const credential = await navigator.credentials.get({
  publicKey: { ...options, challenge: base64urlToBytes(options.challenge) },
  mediation: 'conditional',
});
await fetch('/webauthn/verify', { method: 'POST', body: JSON.stringify(serialize(credential)) });
```
Use a library (SimpleWebAuthn, your IdP) for the encoding and server-side verification.

---

## Where to store tokens

The question isn't "cookie vs JWT", it's **"can JavaScript read the credential?"** and **"does the browser send it automatically?"**

| Storage | Readable by JS (XSS theft)? | Sent automatically (CSRF)? | Verdict |
|---|---|---|---|
| **HttpOnly + Secure + SameSite cookie** | No | Yes (mitigated by SameSite + CSRF token) | ✅ Best default for browser apps (with BFF) |
| **In-memory variable** + `Authorization` header | Yes, but only while the page runs; lost on reload | No | ✅ OK for short-lived access tokens, refresh via HttpOnly cookie |
| **localStorage** | Yes, persistently | No | ⚠️ Avoid for long-lived tokens |
| **sessionStorage** | Yes (per tab) | No | ⚠️ Slightly smaller window, same XSS risk |
| Non-HttpOnly cookie | Yes | Yes | ❌ Worst of both |

### Cookies
| Pros | Cons |
|---|---|
| Automatically sent with each matching request | Vulnerable to **CSRF** unless `SameSite` + CSRF protection |
| Built-in security flags: **`HttpOnly`** (JS can't read it), **`Secure`** (HTTPS only), **`SameSite`**, `Domain`/`Path` scoping, `Max-Age`/`Expires`, `__Host-` prefix | Scoped to a **domain/path** (not the same-origin policy) — cross-site APIs need `SameSite=None` and CORS with credentials |
| | ~4 KB per cookie; sent on every request (overhead) |

<Info>
**A well-configured session cookie**

```http
Set-Cookie: __Host-sid=8f3a…; Path=/; Secure; HttpOnly; SameSite=Lax; Max-Age=3600
```
`__Host-` prefix forces `Secure`, `Path=/` and no `Domain` → can't be set or overridden by subdomains.
</Info>

### Authorization header (Bearer tokens)
The header is a *transport*; the question is where the token lives before you attach it.

| Pros | Cons |
|---|---|
| Not sent automatically → **no CSRF** | Must be stored somewhere JS can read → **XSS** can steal it |
| Works across domains, APIs and mobile clients | Requires managing the token life cycle (expiry, refresh, revocation) |
| A JWT can carry claims like roles (for the *server* to use) | Token theft if not sent over HTTPS |

### Local Storage
| Pros | Cons |
|---|---|
| Larger capacity (~5–10 MB), simple JS API | **Any XSS can read it** — including a compromised npm dependency |
| Persists across tabs and restarts | Not sent automatically (you attach it manually) |
| | Synchronous API, no expiry |

### Session Storage
| Pros | Cons |
|---|---|
| Cleared automatically when the **tab** closes | Same XSS exposure as localStorage |
| JS API for temporary data | Per tab: opening a new tab requires re-authentication; not sent automatically |

<Tip>
**Senior answer**

"For a browser app I'd use a BFF: the OAuth flow runs server-side, tokens stay on the server, and the browser gets an `__Host-` HttpOnly, Secure, SameSite=Lax session cookie. I'd add CSRF protection for state-changing requests and a strict CSP to reduce XSS. If there's no BFF, I'd keep a short-lived access token in memory and the refresh token in an HttpOnly cookie with rotation — never long-lived tokens in localStorage."
</Tip>

<Warning>
**Common pitfalls**

- Treating client-side route guards as security — the API must check permissions.
- Putting tokens in URLs (they leak via logs, history and `Referer`).
- Forgetting logout must revoke server-side (session delete / refresh-token revoke), not just clear client state.
- Not rotating the session ID after login (session fixation).
- Using the Implicit flow or a client secret inside a SPA.
</Warning>

<Info>
**🆕 2025–2026**

- **CVE-2025-55182** (2025-12-03, CVSS 10) showed that auth code running in React Server Components/Server Functions is server code: patch RSC frameworks promptly and validate/authorize inside every Server Function. See [Frontend Security - XSS, CSRF, CSP & Trusted Types](/Software%20Engineering/Frontend/11%20Security/Frontend%20Security%20-%20XSS%2C%20CSRF%2C%20CSP%20%26%20Trusted%20Types).
- **Next.js 16** replaced `middleware.ts` with **`proxy.ts`** (Node runtime) — the place where many apps check auth cookies and redirect. `cookies()`/`headers()` must now be awaited. See [Next.js 15 to 16.3](/Software%20Engineering/Frontend/07%20Next.js/Next.js%2015%20to%2016.3).
- npm moved maintainers toward 2FA/FIDO and short-lived tokens after the 2025 supply-chain attacks — the same phishing-resistant direction as passkeys. See [Supply Chain Security](/Software%20Engineering/Frontend/11%20Security/Supply%20Chain%20Security).
</Info>

## Interview questions

<Accordion title="Authentication vs authorization?">
Authentication verifies identity (login, passkey, OIDC). Authorization decides what that identity may do (roles, scopes, ownership checks) and must be enforced server-side on every request.
</Accordion>

<Accordion title="Session-based vs JWT-based auth — trade-offs?">
Sessions: opaque ID in a cookie, state on the server; easy revocation, needs a shared store to scale, CSRF considerations. JWT: self-contained signed claims; stateless and easy to verify across services, but hard to revoke, larger, and readable. Common hybrid: short-lived JWT access tokens + rotating refresh tokens, or sessions at the edge (BFF) with JWTs between services.
</Accordion>

<Accordion title="Where should a SPA store its tokens?">
Ideally nowhere: use a BFF with an HttpOnly, Secure, SameSite cookie. Otherwise keep the access token in memory and the refresh token in an HttpOnly cookie with rotation. Avoid localStorage for long-lived tokens because any XSS (including a malicious dependency) can read it.
</Accordion>

<Accordion title="What is PKCE and why is it needed?">
Proof Key for Code Exchange. The client sends a hash (`code_challenge`) of a random secret when starting the flow and the secret (`code_verifier`) when exchanging the code. An attacker who intercepts the authorization code can't redeem it. Required for public clients (SPAs, mobile) that can't hold a client secret; recommended for all clients.
</Accordion>

<Accordion title="OAuth vs OpenID Connect?">
OAuth 2.0 is delegated authorization (access tokens for APIs). OIDC adds authentication: an ID token (JWT) with identity claims, a UserInfo endpoint and standard scopes (`openid`, `profile`). "Sign in with Google" is OIDC.
</Accordion>

<Accordion title="What is refresh token rotation?">
Each refresh returns a new refresh token and invalidates the previous one. If an old one is reused, the server detects theft and revokes the whole family, forcing re-login.
</Accordion>

<Accordion title="What is the BFF pattern and why is it recommended?">
A server component, owned by the frontend team, that runs the OAuth flow as a confidential client, stores tokens server-side and gives the browser only an HttpOnly session cookie. It removes tokens from JS reach and simplifies CORS and token refresh.
</Accordion>

<Accordion title="Why are passkeys phishing-resistant?">
The credential is bound to the site's origin; the browser only uses it for the matching RP ID, and authentication signs a server challenge with a private key that never leaves the device. There's no shared secret to type into a fake site or leak in a breach.
</Accordion>

## Related
- [JWT vs Cookies](/Software%20Engineering/APIs%20%26%20Auth/JWT%20vs%20Cookies)
- [Authentication vs Authorization](/Software%20Engineering/APIs%20%26%20Auth/Authentication%20vs%20Authorization)
- [Frontend Security - XSS, CSRF, CSP & Trusted Types](/Software%20Engineering/Frontend/11%20Security/Frontend%20Security%20-%20XSS%2C%20CSRF%2C%20CSP%20%26%20Trusted%20Types)
- [Browser Storage - Cookies, Web Storage & IndexedDB](/Software%20Engineering/Frontend/03%20Browser%20%26%20Web%20Platform/Browser%20Storage%20-%20Cookies%2C%20Web%20Storage%20%26%20IndexedDB)
- [Networking - HTTP, Caching & Real-time](/Software%20Engineering/Frontend/03%20Browser%20%26%20Web%20Platform/Networking%20-%20HTTP%2C%20Caching%20%26%20Real-time)
- [Next.js App Router Fundamentals](/Software%20Engineering/Frontend/07%20Next.js/Next.js%20App%20Router%20Fundamentals)
- [Supply Chain Security](/Software%20Engineering/Frontend/11%20Security/Supply%20Chain%20Security)
- [Ultimate NextJs Course](/Software%20Engineering/Frontend/Courses%20%26%20Books/FE%20Masters/Ultimate%20NextJs%20Course)

## Sources
- Migrated from the owner's [Ultimate NextJs Course](/Software%20Engineering/Frontend/Courses%20%26%20Books/FE%20Masters/Ultimate%20NextJs%20Course) note (Authentication / Authorization, types of authentication, HTTP state management, storing session tokens)
- [OAuth 2.0 Security Best Current Practice – RFC 9700](https://www.rfc-editor.org/rfc/rfc9700)
- [RFC 7636 – PKCE](https://www.rfc-editor.org/rfc/rfc7636)
- [OAuth 2.0 for Browser-Based Apps (IETF draft)](https://datatracker.ietf.org/doc/draft-ietf-oauth-browser-based-apps/)
- [OpenID Connect Core](https://openid.net/specs/openid-connect-core-1_0.html)
- [OWASP – Session Management Cheat Sheet](https://cheatsheetseries.owasp.org/cheatsheets/Session_Management_Cheat_Sheet.html)
- [OWASP – JWT Cheat Sheet](https://cheatsheetseries.owasp.org/cheatsheets/JSON_Web_Token_for_Java_Cheat_Sheet.html)
- [MDN – Using HTTP cookies](https://developer.mozilla.org/en-US/docs/Web/HTTP/Cookies)
- [MDN – Web Authentication API](https://developer.mozilla.org/en-US/docs/Web/API/Web_Authentication_API)
- [web.dev – Passkeys](https://web.dev/articles/passkey-registration)
- [Next.js – Authentication guide](https://nextjs.org/docs/app/guides/authentication)
- [React – RSC security advisory (2025-12-03)](https://react.dev/blog/2025/12/03/critical-security-vulnerability-in-react-server-components)
