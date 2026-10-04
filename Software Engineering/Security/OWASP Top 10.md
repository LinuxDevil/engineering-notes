---
title: "OWASP Top 10"
tags: [security, security/owasp]
aliases: [OWASP Top Ten, OWASP 2025, OWASP Top 10 2025, Top 10 Web Risks]
updated: 2026-09-27
---
# OWASP Top 10

<Note>
**TL;DR**

The OWASP Top 10 is an awareness list of the most critical web application security risks, built from vulnerability data and a community survey. The current edition is **OWASP Top 10:2025** (release candidate November 2025, final early 2026). Broken Access Control stays #1, Security Misconfiguration jumps to #2, **Software Supply Chain Failures** (#3) and **Mishandling of Exceptional Conditions** (#10) are new, and SSRF is folded into Broken Access Control. It is a starting point for awareness, not a complete standard: use OWASP ASVS for verification.
</Note>

<Info>
**🆕 2025–2026**

- **2025-11-06**: OWASP Top 10:2025 release candidate announced at OWASP Global AppSec USA (Washington, D.C.), open for comments until 2025-11-20; the final version followed (reported as January 2026).
- Data: 589 CWEs analysed, 248 CWEs mapped into the 10 categories.
- **2025-05**: OWASP **ASVS 5.0** released (about 350 requirements in 17 chapters), the verification standard to use alongside the Top 10.
- Sources: [top10.owasp.org/2025](https://top10.owasp.org/2025/), [SecurityWeek](https://www.securityweek.com/two-new-web-application-risk-categories-added-to-owasp-top-10/), [OWASP ASVS](https://github.com/OWASP/ASVS).
</Info>

## The 2025 list

| # | Category | In one line | Typical fix |
|---|---|---|---|
| **A01** | Broken Access Control | Users act outside their permissions: IDOR, missing function-level checks, CORS misconfig, **SSRF** (now here) | Deny by default; check ownership on every object server-side; central authz |
| **A02** | Security Misconfiguration | Default creds, verbose errors, open cloud buckets, missing headers, unneeded features | Hardened, automated configuration (IaC), config scanning |
| **A03** | Software Supply Chain Failures | Vulnerable or malicious dependencies, compromised build/CI, unsigned artifacts | SCA, lockfiles, SBOM, signed provenance (SLSA), pinned CI actions |
| **A04** | Cryptographic Failures | Plaintext data, weak algorithms, bad key management, no TLS | TLS everywhere, modern algorithms, KMS, don't roll your own crypto |
| **A05** | Injection | SQL, NoSQL, OS command, LDAP injection, **XSS** | Parameterised queries, safe APIs, output encoding, input validation |
| **A06** | Insecure Design | Missing security controls by design (no rate limit on reset, weak business rules) | Threat modeling, secure design patterns, abuse cases |
| **A07** | Authentication Failures | Credential stuffing, weak passwords, broken session handling | MFA/passkeys, rate limiting, breached-password checks, secure sessions |
| **A08** | Software or Data Integrity Failures | Trusting unsigned updates, insecure deserialisation, unsafe CI inputs | Signatures, integrity checks, no untrusted deserialisation |
| **A09** | Security Logging & Alerting Failures | Attacks not logged, logs not monitored, no alerts | Log auth/authz events, centralise, alert on them, test detection |
| **A10** | Mishandling of Exceptional Conditions | Failing open, leaking errors, inconsistent state after errors, unchecked return values | Fail closed, central error handling, transactions/rollback, don't leak internals |

## Mapping from 2021 to 2025

{/* diagram source: /diagrams/diagram-382.mmd */}
![diagram](/diagrams/diagram-382.svg)

| 2021 | 2025 | Change |
|---|---|---|
| A01 Broken Access Control | A01 | Same; SSRF merged in |
| A02 Cryptographic Failures | A04 | Down 2 |
| A03 Injection | A05 | Down 2 |
| A04 Insecure Design | A06 | Down 2 |
| A05 Security Misconfiguration | A02 | Up 3 |
| A06 Vulnerable & Outdated Components | A03 Software Supply Chain Failures | Expanded to build systems and distribution |
| A07 Identification & Authentication Failures | A07 Authentication Failures | Renamed |
| A08 Software & Data Integrity Failures | A08 | Same |
| A09 Security Logging & Monitoring Failures | A09 Security Logging & Alerting Failures | Renamed (focus on alerting) |
| A10 SSRF | (merged into A01) | Removed as standalone |
| (new) | A10 Mishandling of Exceptional Conditions | New |

## Examples and fixes per category

### A01 Broken Access Control
```http
GET /api/invoices/1043      ← my invoice
GET /api/invoices/1044      ← someone else's: returned because only authentication was checked
```
```js
// fix: scope the query by the caller
const invoice = await db.invoice.findFirst({ where: { id, ownerId: req.user.id } });
if (!invoice) return res.status(404).end(); // 404 avoids confirming the id exists
```
SSRF example: an "import from URL" feature fetches `http://169.254.169.254/latest/meta-data/` and returns cloud credentials. Fix: allow-list destinations, block private/link-local ranges after DNS resolution, use IMDSv2, and egress firewalls. Details: [Backend Security Essentials](/Software%20Engineering/Security/Backend%20Security%20Essentials), [API Security](/Software%20Engineering/Security/API%20Security).

### A02 Security Misconfiguration
Stack traces in production responses, the admin console exposed on the internet, an S3 bucket with public read, default passwords, XML parsers with external entities enabled (XXE also sits in this category). Fix: repeatable hardened builds, IaC with policy checks (e.g. Checkov, OPA), disable what you don't use, security headers.

### A03 Software Supply Chain Failures
A typosquatted npm package, a compromised maintainer account pushing a malicious version, a GitHub Action tag re-pointed to malicious code. Fix: lockfiles and `npm ci`, pin actions to commit SHAs, SCA scanning, SBOMs, provenance and signing (Sigstore, SLSA), least-privilege CI tokens. See [Supply Chain Security](/Software%20Engineering/Frontend/11%20Security/Supply%20Chain%20Security) and [Secure SDLC & Threat Modeling](/Software%20Engineering/Security/Secure%20SDLC%20%26%20Threat%20Modeling).

### A04 Cryptographic Failures
Passwords hashed with MD5/SHA-1, sensitive data over HTTP, hard-coded keys, ECB mode. Fix: Argon2id/bcrypt for passwords, TLS 1.2+ (prefer 1.3), AES-GCM or libsodium, keys in a KMS. See [Cryptography Basics for Engineers](/Software%20Engineering/CS%20Fundamentals/Cryptography%20Basics%20for%20Engineers).

### A05 Injection
```sql
-- vulnerable: string concatenation
SELECT * FROM users WHERE email = '" + email + "';   -- email = ' OR '1'='1
-- safe: parameterised
SELECT * FROM users WHERE email = $1;
```
XSS is part of Injection: see [Frontend Security - XSS, CSRF, CSP & Trusted Types](/Software%20Engineering/Frontend/11%20Security/Frontend%20Security%20-%20XSS%2C%20CSRF%2C%20CSP%20%26%20Trusted%20Types). Command injection: never build shell strings from input; use `execFile` with an argument array.

### A06 Insecure Design
A password reset with a 4-digit code and no attempt limit; a checkout that trusts the client's price; a "gift card" flow that allows negative quantities. No amount of perfect code fixes a design without the control. Fix: threat modeling and abuse cases during design ([Secure SDLC & Threat Modeling](/Software%20Engineering/Security/Secure%20SDLC%20%26%20Threat%20Modeling)).

### A07 Authentication Failures
Credential stuffing with no rate limit, session ids in URLs, sessions not invalidated on logout or password change, weak password rules. Fix: MFA or passkeys ([Passkeys, WebAuthn & MFA](/Software%20Engineering/APIs%20%26%20Auth/Passkeys%2C%20WebAuthn%20%26%20MFA)), breached-password checks, rate limiting, secure cookies. NIST SP 800-63B-4 (final 2025-07-31) favours length (≥15 characters when password-only) over composition rules and forbids periodic forced resets.

### A08 Software or Data Integrity Failures
Deserialising untrusted data (Java `ObjectInputStream`, Python `pickle`), auto-update without signature checks, CI pipelines that run code from forked PRs with secrets. Fix: signed artifacts, safe data formats (JSON with schema validation), isolate untrusted CI.

### A09 Security Logging & Alerting Failures
A brute-force attack for days with no alert; logs that capture passwords. Fix: log authentication, authorization failures and admin actions with user/request ids, never log secrets, centralise logs, and alert on patterns.

### A10 Mishandling of Exceptional Conditions
```js
// fails OPEN: an error in the permission service lets everyone through
async function canAccess(user, doc) {
  try { return await authz.check(user, doc); }
  catch { return true; }            // ❌
}
// fix: fail closed and log
catch (err) { logger.error({ err }, 'authz check failed'); return false; }
```
Other examples: a half-completed transfer when an exception interrupts two writes (use transactions), error messages revealing SQL or paths, unhandled promise rejections crashing a service (DoS).

## Pitfalls
<Warning>
**Common misunderstandings**

- The Top 10 is **not** a checklist or a standard; it's awareness. Use ASVS for requirements and testing.
- "We passed a scanner" doesn't cover A01, A06 or A10; those need design review and targeted tests.
- Ranking is by data prevalence and impact, not by what's most dangerous for *your* app. Threat-model your own system.
</Warning>

## Senior perspective
- Map your security controls and tests to the Top 10 for reporting, but prioritise by your threat model.
- Broken access control is the top risk because it is **per-endpoint logic**; fix it structurally (central policy, query scoping, tests per role) rather than reviewing endpoints one by one.
- Ask in design review: "What happens if this dependency fails? Do we fail open or closed?" (A10).
- Supply chain moving to #3 matches 2025 reality: npm worm and account-takeover incidents, CI compromises. Budget for it.

## Interview questions
<Accordion title="What is the OWASP Top 10 and what changed in 2025?">
An awareness list of the most critical web app risks. 2025: Broken Access Control still #1, Security Misconfiguration up to #2, new Software Supply Chain Failures (#3, expanding "Vulnerable and Outdated Components") and Mishandling of Exceptional Conditions (#10), SSRF merged into Broken Access Control, and two renames (Authentication Failures; Security Logging & Alerting Failures).
</Accordion>

<Accordion title="Why is Broken Access Control #1?">
It's found in most tested applications, it's application-specific logic that scanners can't infer, and its impact is data exposure. Every endpoint and every object needs an ownership/permission check.
</Accordion>

<Accordion title="Why was SSRF merged into Broken Access Control?">
SSRF is a case of the server being tricked into accessing resources the attacker shouldn't reach (internal services, metadata endpoints), i.e. an access control failure at the network level. Its own data volume was small.
</Accordion>

<Accordion title="What does &quot;fail closed&quot; mean? Give an example.">
When a check can't complete, deny rather than allow. If the authorization service times out, return 503/403 rather than letting the request through.
</Accordion>

<Accordion title="How do you use the Top 10 in a team?">
Training and shared vocabulary, a checklist in code review and design review, mapping SAST/DAST findings to categories, and ASVS for concrete verifiable requirements.
</Accordion>

## Related
- [Security Index](/Software%20Engineering/Security/Security%20Index)
- [Backend Security Essentials](/Software%20Engineering/Security/Backend%20Security%20Essentials) · [API Security](/Software%20Engineering/Security/API%20Security) · [Secure SDLC & Threat Modeling](/Software%20Engineering/Security/Secure%20SDLC%20%26%20Threat%20Modeling) · [Secrets Management](/Software%20Engineering/Security/Secrets%20Management)
- [Frontend Security - XSS, CSRF, CSP & Trusted Types](/Software%20Engineering/Frontend/11%20Security/Frontend%20Security%20-%20XSS%2C%20CSRF%2C%20CSP%20%26%20Trusted%20Types) · [Supply Chain Security](/Software%20Engineering/Frontend/11%20Security/Supply%20Chain%20Security) · [Authentication & Token Storage](/Software%20Engineering/Frontend/11%20Security/Authentication%20%26%20Token%20Storage)
- [OAuth 2.x & OpenID Connect](/Software%20Engineering/APIs%20%26%20Auth/OAuth%202.x%20%26%20OpenID%20Connect) · [Passkeys, WebAuthn & MFA](/Software%20Engineering/APIs%20%26%20Auth/Passkeys%2C%20WebAuthn%20%26%20MFA) · [Cryptography Basics for Engineers](/Software%20Engineering/CS%20Fundamentals/Cryptography%20Basics%20for%20Engineers)

## Sources
- [OWASP Top 10:2025](https://top10.owasp.org/2025/) and [Introduction: what changed](https://top10.owasp.org/2025/0x00_2025-Introduction/)
- [OWASP Top 10:2021](https://owasp.org/Top10/2021/)
- [SecurityWeek: Two new web application risk categories added to OWASP Top 10 (Nov 2025)](https://www.securityweek.com/two-new-web-application-risk-categories-added-to-owasp-top-10/)
- [OWASP ASVS 5.0](https://github.com/OWASP/ASVS)
- [NIST SP 800-63B-4](https://pages.nist.gov/800-63-4/sp800-63b.html)
- [OWASP Cheat Sheet Series](https://cheatsheetseries.owasp.org/)
