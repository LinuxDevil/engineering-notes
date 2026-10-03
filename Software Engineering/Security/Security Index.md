---
tags: [security, index]
aliases: [Security Home, Security MOC, AppSec Index]
updated: 2026-09-27
---
# Security Index

> [!tldr] Start here
> Application security for engineers, from the common vulnerability classes (OWASP Top 10) to backend and API defences, secrets, and building security into the development process. Browser-side security (XSS, CSRF, CSP, token storage, npm supply chain) lives in the Frontend notes and is linked here rather than repeated. Identity notes (OAuth/OIDC, passkeys) and cryptography are linked from their own notes.

```mermaid
mindmap
  root((Security))
    Risks
      OWASP Top 10 2025
      API Security Top 10
    Build it in
      Threat modeling
      SAST DAST SCA
      SBOM & SLSA
      Cyber Resilience Act
    Backend
      Injection
      Access control
      SSRF
      Secrets
    Identity
      OAuth & OIDC
      Passkeys & MFA
      Token storage
    Frontend
      XSS CSRF CSP
      Supply chain
    Foundations
      Cryptography
```

## 1. Beginner: know the risks
- [[OWASP Top 10]]: the 2025 list, examples, fixes, and the 2021 → 2025 mapping
- [[Backend Security Essentials]]: SQL injection and parameterised queries, IDOR, SSRF, mass assignment, validation, headers, logging
- [[Frontend Security - XSS, CSRF, CSP & Trusted Types]]: browser-side attacks and defences
- [[Cryptography Basics for Engineers]]: hashing, encryption, signatures, TLS

## 2. Intermediate: identity, APIs and secrets
- [[API Security]]: OWASP API Security Top 10 (2023), RBAC vs ABAC vs ReBAC
- [[OAuth 2.x & OpenID Connect]] · [[Passkeys, WebAuthn & MFA]] · [[Authentication & Token Storage]]
- [[Secrets Management]]: managers, workload identity, rotation, scanning, leak response

## 3. Senior: security as a process
- [[Secure SDLC & Threat Modeling]]: STRIDE, security reviews, SAST/DAST/SCA, SBOM, SLSA, EU Cyber Resilience Act
- [[Supply Chain Security]]: npm incidents of 2025, provenance, trusted publishing

> [!example]- 🧭 Suggested study path
> 1. [[OWASP Top 10]] → [[Backend Security Essentials]] → [[Frontend Security - XSS, CSRF, CSP & Trusted Types]]
> 2. [[Cryptography Basics for Engineers]] → [[OAuth 2.x & OpenID Connect]] → [[Passkeys, WebAuthn & MFA]] → [[Authentication & Token Storage]]
> 3. [[API Security]] → [[Secrets Management]]
> 4. [[Secure SDLC & Threat Modeling]] → [[Supply Chain Security]]

> [!example]- 🎯 Security questions in interviews
> Backend: [[Backend Security Essentials]] and [[API Security]] interview sections, plus [[Backend Interview Questions]] (security part). Frontend: [[Frontend Security - XSS, CSRF, CSP & Trusted Types]] and [[Authentication & Token Storage]].

## 🆕 2025–2026 in this area
- 2025-04 | SLSA v1.1 approved → [[Secure SDLC & Threat Modeling]]
- 2025-05 | OWASP ASVS 5.0 released → [[OWASP Top 10]]
- 2025-07-31 | NIST SP 800-63-4 final (password length over composition rules, no forced periodic resets) → [[OWASP Top 10]]
- 2025-08 | CISA draft 2025 Minimum Elements for an SBOM → [[Secure SDLC & Threat Modeling]]
- 2025-09 | npm worm and maintainer-phishing incidents (chalk/debug, Shai-Hulud) → [[Supply Chain Security]], [[Secrets Management]]
- 2025-11-06 | OWASP Top 10:2025 release candidate (final early 2026): Supply Chain Failures #3, Mishandling of Exceptional Conditions #10, SSRF merged into A01 → [[OWASP Top 10]]
- 2025-11-12 | SLSA v1.2 with the Source Track → [[Secure SDLC & Threat Modeling]]
- 2026-03 | GitGuardian: ~29M secrets leaked on public GitHub in 2025, AI-service secrets +81% → [[Secrets Management]]
- 2026-09-11 | EU Cyber Resilience Act reporting obligations apply; full application 2027-12-11 → [[Secure SDLC & Threat Modeling]]

## Related
- [[Software Engineering]], the vault root
- [[Frontend Home]] · [[Interviews Index]] · [[Career Index]]
