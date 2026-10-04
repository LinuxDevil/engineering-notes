---
title: "OAuth 2.x & OpenID Connect"
tags: [apis, apis/auth]
aliases: [OAuth, OAuth 2.0, OAuth 2.1, OpenID Connect, OIDC, PKCE, DPoP, Authorization Code Flow, Client Credentials, Device Flow, SSO]
updated: 2026-09-27
---
# OAuth 2.x & OpenID Connect

<Note>
**TL;DR**

**OAuth 2.0** lets a *client* app get a limited **access token** to call an API on behalf of a user (or itself) without ever seeing the user's password. **OpenID Connect (OIDC)** is a thin identity layer on top that adds an **ID token** so the client also learns *who* the user is (login / SSO). Today there is essentially **one flow for users (Authorization Code + PKCE)**, **one for machines (Client Credentials)**, and **Device Flow** for TVs/CLIs. Implicit and Password grants are deprecated. **OAuth 2.1** (still a draft in 2026) consolidates these rules; **RFC 9700** (Jan 2025) is the security baseline.
</Note>

## Roles and vocabulary

| Term | Meaning | Example |
|---|---|---|
| **Resource owner** | the user who owns the data | you |
| **Client** | the app wanting access | a photo-printing web app |
| **Authorization server (AS)** | issues tokens after authenticating the user and getting consent | Google, Okta, Auth0, Keycloak, Entra ID |
| **Resource server (RS)** | the API that accepts access tokens | Google Photos API |
| **Scope** | what access is requested | `photos.read` |
| **Confidential client** | can keep a secret (server-side app) | a Rails backend / BFF |
| **Public client** | cannot keep a secret | SPA, mobile app, CLI |

## Token types

| Token | Audience | Format | Lifetime | Purpose |
|---|---|---|---|---|
| **Access token** | the API (RS) | opaque or JWT (RFC 9068 `at+jwt`) | minutes | call APIs; **never** use it to "log the user in" |
| **Refresh token** | the AS only | opaque | hours–days, rotating | get new access tokens without user interaction |
| **ID token** (OIDC) | the client | always a signed JWT | minutes | tells the client who authenticated, when, how (`sub`, `auth_time`, `acr`, `amr`, `nonce`) |
| Authorization code | the AS | opaque, one-time | ~seconds–1 min | exchanged at the token endpoint |

Access tokens are **bearer** tokens by default: whoever holds them can use them. **Sender-constrained** tokens (DPoP, mTLS) fix that.

## Authorization Code flow + PKCE (the default for everything with a user)

{/* diagram source: /diagrams/diagram-032.mmd */}
![diagram](/diagrams/diagram-032.svg)

**PKCE** (RFC 7636) binds the code to the client that started the flow: a stolen authorization code is useless without the `code_verifier`. Originally for mobile apps, it is now required for **all** clients (OAuth 2.1, RFC 9700).

```ts
// PKCE in Node
import crypto from 'node:crypto';
const verifier = crypto.randomBytes(32).toString('base64url');
const challenge = crypto.createHash('sha256').update(verifier).digest('base64url');

const url = new URL('https://as.example.com/authorize');
url.search = new URLSearchParams({
  response_type: 'code', client_id: 'web-bff', redirect_uri: 'https://app.example.com/callback',
  scope: 'openid profile orders.read', state: crypto.randomUUID(), nonce: crypto.randomUUID(),
  code_challenge: challenge, code_challenge_method: 'S256',
}).toString();
// store verifier/state/nonce in the server session, redirect the user to url
```

```http
POST /token HTTP/1.1
Host: as.example.com
Content-Type: application/x-www-form-urlencoded
Authorization: Basic d2ViLWJmZjpzZWNyZXQ=

grant_type=authorization_code&code=abc&redirect_uri=https%3A%2F%2Fapp.example.com%2Fcallback&code_verifier=dBjftJeZ4CVP-mB92K27uhbUJU1p1r_wW1gFWFOEjXk
```

In real code use a certified library (openid-client, Spring Security, Authlib, etc.), not hand-rolled flows.

## Other grants

| Grant | Who | Notes |
|---|---|---|
| **Client Credentials** | service → service, no user | `grant_type=client_credentials`; authenticate with secret, `private_key_jwt` or mTLS |
| **Device Authorization** (RFC 8628) | TVs, CLIs, IoT | device shows a code; user approves on a phone; device polls the token endpoint |
| **Refresh Token** | any | rotate for public clients |
| **Token Exchange** (RFC 8693) | service acting on behalf of a user downstream | swap a token for one with a different audience/scope |
| **JWT Bearer** (RFC 7523) | server with a signed assertion | used by Google service accounts |
| ~~Implicit~~ | SPAs (historically) | **deprecated**: tokens in the URL fragment leak; removed in OAuth 2.1 |
| ~~Resource Owner Password Credentials~~ | legacy first-party apps | **deprecated**: the client sees the password, breaks MFA/passkeys; removed in OAuth 2.1 |

## OpenID Connect in one page

- Request scope `openid` (+ `profile`, `email`), receive an **ID token**.
- **Discovery**: `https://issuer/.well-known/openid-configuration` lists endpoints, supported algorithms, `jwks_uri`.
- **UserInfo endpoint**: more claims with the access token.
- `nonce` binds the ID token to the login request (replay protection); `acr`/`amr` say how the user authenticated (e.g. `amr: ["hwk","pin"]` for a passkey), which matters for step-up.
- Logout: RP-initiated logout, back-channel logout (server-to-server notification) for SSO sessions.
- SAML 2.0 is the older XML-based enterprise SSO equivalent; still common in B2B.

## Sender-constrained tokens: DPoP

**DPoP** (RFC 9449, 2023) makes a stolen access token useless without the client's private key. For each request the client signs a small JWT proof containing the method, URL and a hash of the access token:

```http
GET /orders HTTP/1.1
Host: api.example.com
Authorization: DPoP eyJhbGciOiJFUzI1NiIs...       ← access token bound to the key (cnf.jkt)
DPoP: eyJ0eXAiOiJkcG9wK2p3dCIsImFsZyI6IkVTMjU2IiwiandrIjp7Imt0eSI6IkVDIi...
      (payload: {"htm":"GET","htu":"https://api.example.com/orders","iat":1790001234,"jti":"…","ath":"<sha256 of access token>"})
```

The resource server checks the proof signature, that its key matches the token's `cnf.jkt`, `htm`/`htu`, freshness and `jti` replay. **mTLS-bound tokens** (RFC 8705) do the same with client certificates for server-side clients.

## Hardening extras worth knowing

- **PAR** (Pushed Authorization Requests, RFC 9126): the client POSTs authorization parameters to the AS first and gets a `request_uri`, so nothing sensitive or tamperable rides in the browser URL.
- **RAR** (Rich Authorization Requests, RFC 9396): structured `authorization_details` (e.g. "pay €50 to X") instead of coarse scopes.
- **`iss` in the authorization response** (RFC 9207) prevents mix-up attacks when a client talks to several ASes.
- **Resource Indicators** (RFC 8707): `resource=https://api.example.com` so the token's audience is exactly one API.
- **Metadata**: AS metadata (RFC 8414) and **Protected Resource Metadata** (RFC 9728, April 2025), where an API advertises which AS protects it; MCP uses this for agent auth discovery.

## What changed: OAuth 2.0 → 2.1

| OAuth 2.0 (RFC 6749, 2012) | OAuth 2.1 (draft) / RFC 9700 BCP |
|---|---|
| PKCE optional, for mobile | **PKCE required** for every authorization code client |
| Implicit grant allowed | **removed** |
| Password grant allowed | **removed** |
| Redirect URI pattern matching tolerated | **exact string matching** |
| Bearer token in query string allowed | **forbidden** (header or body only) |
| Refresh tokens unconstrained | public clients: **rotate** or **sender-constrain** |

<Info>
**🆕 2025–2026**

- **2025-01:** **RFC 9700** "Best Current Practice for OAuth 2.0 Security" (BCP 240) published: PKCE for all, no implicit/password grants, exact redirect matching, sender-constrained or rotating refresh tokens, mix-up defences.
- **2025-04:** **RFC 9728** OAuth 2.0 Protected Resource Metadata published (APIs advertise their authorization servers; used by MCP).
- **2025-09:** OpenAPI 3.2 can describe the **device authorization flow** and OAuth metadata URLs.
- **2026-03-02:** **OAuth 2.1** reaches `draft-ietf-oauth-v2-1-15`. It is **still an Internet-Draft**, not an RFC, but major IdPs already implement its rules.
- **2026-07-28:** the **MCP** spec hardens its OAuth profile for AI agents: authorization servers must return `iss` (RFC 9207) and clients must validate it, client credentials are bound to their issuer, **Dynamic Client Registration is deprecated in favour of Client ID Metadata Documents (CIMD)**. See [AI in Frontend - MCP, WebMCP & Built-in AI](/Software%20Engineering/Frontend/16%20AI%20in%20Frontend/AI%20in%20Frontend%20-%20MCP%2C%20WebMCP%20%26%20Built-in%20AI).
- **2026-08:** **RFC 10017** "OAuth 2.0 for Browser-Based Applications" (BCP 212) published; it ranks **BFF** as the most secure SPA architecture, then token-mediating backend, then a pure browser client.
</Info>

## Pitfalls

<Warning>
**Common mistakes**

- Using an **access token as proof of login** in the client, or accepting ID tokens at APIs. ID token → client; access token → API.
- Not validating `aud`: a token minted for API A accepted by API B.
- Missing or unchecked `state` (login CSRF) and `nonce`.
- Wildcard or prefix redirect URI matching → open redirect → code theft.
- Long-lived bearer tokens in `localStorage` in SPAs. Prefer a BFF ([JWT vs Cookies](/Software%20Engineering/APIs%20%26%20Auth/JWT%20vs%20Cookies)).
- Over-broad scopes (`admin`, `*`) and never reviewing consent grants.
- Rolling your own OAuth server. Use a proven product or library.
</Warning>

## Senior perspective

- Architecture choice for web apps: **BFF** (confidential client, tokens server-side, session cookie to the browser) is the 2026 default for anything sensitive (RFC 10017).
- Internal microservices: propagate identity with **token exchange** or a gateway-minted internal token with a narrow audience, not the user's original token everywhere.
- Plan **key rotation** (JWKS with multiple `kid`s), token lifetimes, refresh-token rotation and revocation, and **logout across SSO sessions**.
- Machine identities are growing fastest (CI, agents, MCP clients): prefer `private_key_jwt`/mTLS/workload identity federation over static client secrets.
- Review questions: *Which grant and why? Where do tokens live? How are they revoked? What audience/scope does each service accept? Are refresh tokens rotated with reuse detection?*

## Interview questions

<Accordion title="OAuth vs OpenID Connect?">
OAuth 2.x is delegated authorization: it gives a client an access token to call an API with limited scope. OIDC is an identity layer on OAuth that adds the ID token (a signed JWT about the authentication event), UserInfo and discovery, so clients can log users in and do SSO.
</Accordion>

<Accordion title="Explain Authorization Code + PKCE and why PKCE exists.">
The client redirects the user to the AS with a code_challenge (hash of a random verifier). After login the AS redirects back with a one-time code. The client exchanges the code plus the original verifier at the token endpoint; the AS checks the hash. PKCE prevents an attacker who intercepts the code (malicious app, logs, referrer) from redeeming it, and also protects against CSRF-style code injection.
</Accordion>

<Accordion title="Why were the Implicit and Password grants deprecated?">
Implicit returns tokens in the URL fragment where they leak via history, referrers and scripts, and can't be sender-constrained or refreshed safely. Password grant exposes user credentials to the client, trains phishing and can't support MFA, passkeys or federated login. Auth Code + PKCE replaces both.
</Accordion>

<Accordion title="What is DPoP?">
Demonstrating Proof of Possession (RFC 9449): the client holds a key pair, the AS binds the access token to its public key, and each API request carries a signed proof of method, URL and token hash. A stolen token can't be replayed without the private key.
</Accordion>

<Accordion title="Which flow for a CLI tool? For a cron job calling an API?">
CLI: Device Authorization Grant (or Auth Code + PKCE with a loopback redirect). Cron/service with no user: Client Credentials, ideally with private_key_jwt or mTLS instead of a shared secret.
</Accordion>

<Accordion title="What is the status of OAuth 2.1?">
Still an IETF Internet-Draft (draft-15, March 2026). It consolidates OAuth 2.0 plus later RFCs: mandatory PKCE, no implicit or password grants, exact redirect URI matching, no tokens in query strings, and rotated or sender-constrained refresh tokens for public clients. RFC 9700 already makes most of this best practice.
</Accordion>

<Accordion title="How does an API validate an access token?">
If JWT: verify signature with the AS's JWKS (pinned algorithms), check `iss`, `aud`, `exp`, `nbf`, `typ`, scopes, and for DPoP the proof binding. If opaque: call the introspection endpoint (RFC 7662) and cache the result briefly.
</Accordion>

## Related
- [APIs & Auth Index](/Software%20Engineering/APIs%20%26%20Auth/APIs%20%26%20Auth%20Index) · [Backend Index](/Software%20Engineering/Backend/Backend%20Index)
- [Authentication vs Authorization](/Software%20Engineering/APIs%20%26%20Auth/Authentication%20vs%20Authorization) · [JWT vs Cookies](/Software%20Engineering/APIs%20%26%20Auth/JWT%20vs%20Cookies) · [Passkeys, WebAuthn & MFA](/Software%20Engineering/APIs%20%26%20Auth/Passkeys%2C%20WebAuthn%20%26%20MFA) · [API Gateways & Reverse Proxies](/Software%20Engineering/Backend/API%20Gateways%20%26%20Reverse%20Proxies) · [API Security](/Software%20Engineering/Security/API%20Security)
- Frontend: [Authentication & Token Storage](/Software%20Engineering/Frontend/11%20Security/Authentication%20%26%20Token%20Storage) · [AI in Frontend - MCP, WebMCP & Built-in AI](/Software%20Engineering/Frontend/16%20AI%20in%20Frontend/AI%20in%20Frontend%20-%20MCP%2C%20WebMCP%20%26%20Built-in%20AI)
- Other domains: [Security Index](/Software%20Engineering/Security/Security%20Index) · [Backend Security Essentials](/Software%20Engineering/Security/Backend%20Security%20Essentials)

## Sources
- [RFC 6749: OAuth 2.0](https://www.rfc-editor.org/rfc/rfc6749.html) · [RFC 7636: PKCE](https://www.rfc-editor.org/rfc/rfc7636.html)
- [draft-ietf-oauth-v2-1-15 (Mar 2026)](https://datatracker.ietf.org/doc/html/draft-ietf-oauth-v2-1-15) · [oauth.net/2.1](https://oauth.net/2.1/)
- [RFC 9700: Best Current Practice for OAuth 2.0 Security (Jan 2025)](https://www.rfc-editor.org/info/rfc9700/)
- [RFC 10017: OAuth 2.0 for Browser-Based Applications (Aug 2026)](https://datatracker.ietf.org/doc/html/rfc10017)
- [RFC 9449: DPoP](https://www.rfc-editor.org/rfc/rfc9449.html) · [RFC 8705: mTLS](https://www.rfc-editor.org/rfc/rfc8705.html)
- [RFC 9126: PAR](https://www.rfc-editor.org/rfc/rfc9126.html) · [RFC 9396: RAR](https://www.rfc-editor.org/rfc/rfc9396.html) · [RFC 9207: Issuer Identification](https://www.rfc-editor.org/rfc/rfc9207.html) · [RFC 8707: Resource Indicators](https://www.rfc-editor.org/rfc/rfc8707.html)
- [RFC 8628: Device Authorization Grant](https://www.rfc-editor.org/rfc/rfc8628.html) · [RFC 8693: Token Exchange](https://www.rfc-editor.org/rfc/rfc8693.html)
- [RFC 9728: Protected Resource Metadata (Apr 2025)](https://www.rfc-editor.org/info/rfc9728/) · [RFC 8414: AS Metadata](https://www.rfc-editor.org/rfc/rfc8414.html)
- [OpenID Connect Core 1.0](https://openid.net/specs/openid-connect-core-1_0.html)
- [MCP 2026-07-28 release post](https://blog.modelcontextprotocol.io/posts/2026-07-28/)
- Aaron Parecki, *OAuth 2.0 Simplified* (book and [oauth.com](https://www.oauth.com/))
