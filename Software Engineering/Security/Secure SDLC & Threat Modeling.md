---
title: "Secure SDLC & Threat Modeling"
tags: [security, security/sdlc]
aliases: [Secure SDLC, SSDLC, Threat Modeling, STRIDE, DevSecOps, SAST DAST SCA, SBOM, SLSA, Cyber Resilience Act, CRA]
updated: 2026-09-27
---
# Secure SDLC & Threat Modeling

<Note>
**TL;DR**

A secure SDLC builds security into every phase instead of testing it in at the end: requirements, **threat modeling** during design (STRIDE is the classic method), secure coding standards and **security-focused code review**, automated **SAST, SCA, secret scanning and DAST** in CI, **SBOMs and signed provenance (SLSA)** for what you ship, and vulnerability response after release. Regulation now requires parts of this: the EU **Cyber Resilience Act** makes vulnerability and incident reporting mandatory from **11 September 2026** and full product requirements from **11 December 2027**.
</Note>

<Info>
**🆕 2025–2026**

- **2024-12-10**: EU Cyber Resilience Act (Regulation (EU) 2024/2847) entered into force.
- **2025-04**: SLSA v1.1 approved; **2025-11-12**: **SLSA v1.2** approved, adding the **Source Track** (source-code integrity levels) next to the Build Track.
- **2025-05**: OWASP ASVS 5.0 released.
- **2025-08**: CISA published a draft **2025 Minimum Elements for an SBOM** (adds component hash, license, tool name, generation context; comments closed 2025-10-03).
- **2025-11 / 2026-01**: OWASP Top 10:2025 puts **Software Supply Chain Failures** at #3 ([OWASP Top 10](/Software%20Engineering/Security/OWASP%20Top%2010)).
- **2026-07-27**: European Commission published CRA implementation guidance. **2026-09-11**: CRA reporting obligations apply (24 h early warning, 72 h notification, final report) for actively exploited vulnerabilities and severe incidents, **including products already on the market**.
- **2027-12-11**: CRA main obligations apply (secure-by-design requirements, vulnerability handling, SBOM, CE marking).
</Info>

## The secure SDLC at a glance

{/* diagram source: /diagrams/diagram-385.mmd */}
![diagram](/diagrams/diagram-385.svg)

Frameworks that describe this: NIST **SSDF** (SP 800-218), OWASP **SAMM** (maturity model), Microsoft **SDL**, BSIMM.

## Threat modeling
Threat modeling answers four questions (Shostack's framework):
1. **What are we working on?** (diagram: components, data flows, trust boundaries)
2. **What can go wrong?** (STRIDE, attack trees, abuse cases)
3. **What are we going to do about it?** (mitigate, eliminate, transfer, accept)
4. **Did we do a good job?** (review, test the mitigations)

### STRIDE
| Threat | Violates | Example | Typical mitigation |
|---|---|---|---|
| **S**poofing | Authentication | Forged JWT, stolen session | Strong auth, MFA, signature validation |
| **T**ampering | Integrity | Modifying a price in a request, altering a build artifact | Server-side validation, signatures, checksums |
| **R**epudiation | Non-repudiation | "I never made that transfer" | Audit logs with identities, signed logs |
| **I**nformation disclosure | Confidentiality | IDOR, verbose errors, unencrypted backups | Authz checks, encryption, minimal responses |
| **D**enial of service | Availability | Unbounded queries, ReDoS, huge uploads | Rate limits, quotas, timeouts, size limits |
| **E**levation of privilege | Authorization | User calls an admin endpoint, container escape | Least privilege, deny by default, sandboxing |

### Worked example: "upload profile picture" feature

{/* diagram source: /diagrams/diagram-386.mmd */}
![diagram](/diagrams/diagram-386.svg)
Trust boundaries: browser → API, API → storage, queue → worker (worker parses untrusted files).

| STRIDE | Threat | Mitigation |
|---|---|---|
| S | Upload on behalf of another user | Auth required; owner taken from the token, not the form |
| T | Replace another user's image by guessing the key | Random object keys; write via API only; bucket not public-writable |
| R | User denies uploading offensive content | Log uploader id, hash, timestamp |
| I | EXIF GPS data leaks home location | Strip metadata on resize |
| D | 2 GB upload or decompression bomb | Size limit at gateway, pixel limit in the resizer, timeouts |
| E | Malicious image exploits the image library (e.g. ImageTragick-style RCE) | Sandboxed worker, patched library, no shell calls, least-privilege IAM role |
| I/E | SVG with script served from main domain → XSS | Disallow SVG or serve from a separate sandbox domain with `Content-Disposition` and CSP |

Keep it light: a 45-minute session with the team, a diagram and a table like this, stored next to the design doc ([Technical Design Docs & RFCs](/Software%20Engineering/Career/Technical%20Design%20Docs%20%26%20RFCs)). Other methods: PASTA (risk-centric), LINDDUN (privacy), attack trees; tools: OWASP Threat Dragon, Microsoft Threat Modeling Tool.

## Security reviews
- **Design review**: triggered by new data types (PII, payments), new external integrations, auth changes, new public endpoints. Uses the threat model.
- **Code review with a security lens**: where does input enter, what interprets it, where is authz checked, what's logged, which dependencies were added? See [Code Review Best Practices](/Software%20Engineering/Career/Code%20Review%20Best%20Practices).
- **Penetration tests**: before big launches and periodically; by people independent of the team.
- **Bug bounty / vulnerability disclosure policy**: a `security.txt` (RFC 9116) and a clear reporting channel. Under the CRA manufacturers must have a coordinated vulnerability disclosure policy.

## Automated testing in the pipeline

| Tool type | What it does | When | Examples | Limits |
|---|---|---|---|---|
| **SAST** (static analysis) | Scans source for risky patterns and data flows | Every PR | Semgrep, CodeQL, SonarQube | False positives; misses logic/authz bugs |
| **SCA** (software composition analysis) | Finds known-vulnerable or malicious dependencies, licenses | Every PR + daily | Dependabot, Renovate, Snyk, OSV-Scanner, npm audit | Only known CVEs; reachability matters |
| **Secret scanning** | Detects committed credentials | Pre-commit + push + history | Gitleaks, TruffleHog, GitHub push protection | Only detects; you must still rotate |
| **IaC / container scanning** | Misconfig in Terraform/K8s, vulnerable image packages | Every PR / build | Checkov, Trivy, Grype | Policy tuning needed |
| **DAST** (dynamic) | Attacks the running app from outside | Staging, nightly | OWASP ZAP, Burp Suite | Needs auth setup; limited coverage |
| **IAST / RASP** | Instrumented runtime analysis/protection | Test / prod | Contrast, Datadog ASM | Agent overhead, vendor lock-in |
| **Fuzzing** | Random/structured inputs to find crashes | Parsers, native code | OSS-Fuzz, Jazzer, AFL++ | Setup effort |

<Tip>
**Make findings actionable**

Block merges only on high-confidence, high-severity rules; route the rest to a backlog with SLAs (e.g. critical in 7 days, high in 30). Prioritise vulnerabilities that are **reachable** and **exploited** (CISA KEV catalogue, EPSS score), not only by CVSS.
</Tip>

## Dependency scanning and updates
- Lockfiles committed; CI installs with `npm ci` / `pip install --require-hashes` / Gradle dependency locking.
- Automated update PRs (Renovate/Dependabot) grouped and auto-merged for patch versions when tests pass; a **cooldown** (e.g. wait a few days after publish) reduces exposure to freshly published malicious versions.
- Remove unused dependencies; fewer dependencies = smaller attack surface.
- Frontend specifics (npm worms, install scripts, provenance): [Supply Chain Security](/Software%20Engineering/Frontend/11%20Security/Supply%20Chain%20Security).

## SBOM
A **Software Bill of Materials** lists every component in what you ship (name, version, supplier, hashes, licenses, dependency relationships).
- Formats: **CycloneDX** (OWASP) and **SPDX** (Linux Foundation, ISO/IEC 5962).
- Generate at build time from the real artifact: `syft`, `cdxgen`, `trivy sbom`, build-tool plugins.
```bash
syft packages dir:. -o cyclonedx-json > sbom.cdx.json
grype sbom:sbom.cdx.json      # scan the SBOM for known vulnerabilities
```
- Uses: answer "are we affected by CVE-X?" in minutes, license compliance, customer and regulatory requirements (US federal procurement; the CRA requires manufacturers to draw up an SBOM, at least for top-level dependencies).
- Pair with **VEX** (Vulnerability Exploitability eXchange) to state "affected / not affected" per CVE.

## SLSA (Supply-chain Levels for Software Artifacts)
A framework of levels for **build integrity**: can a consumer verify that an artifact was built from the expected source by the expected build process?

| Build level | Requirement (simplified) |
|---|---|
| L0 | Nothing |
| L1 | Provenance exists (how it was built) |
| L2 | Hosted build platform; provenance is **signed** by it |
| L3 | Hardened build platform; builds isolated from each other; signing keys not accessible to build steps |

SLSA v1.2 (Nov 2025) adds a **Source Track** with levels for source integrity (version control history, enforced review/branch protection, continuous enforcement). Practical path: GitHub Actions with artifact attestations or the SLSA GitHub generator, Sigstore/cosign signing, npm `--provenance`, and verifying provenance on deploy (policy controllers in Kubernetes).

## EU Cyber Resilience Act (CRA) in brief
| Date | What happens |
|---|---|
| 2024-12-10 | Regulation entered into force |
| 2026-06-11 | Rules on conformity assessment bodies apply |
| **2026-09-11** | **Reporting obligations**: manufacturers report actively exploited vulnerabilities and severe incidents via ENISA's single reporting platform: early warning within **24 h**, notification within **72 h**, final report (14 days after a fix for vulnerabilities; one month for incidents) |
| **2027-12-11** | Full application: essential cybersecurity requirements, vulnerability handling for the support period, SBOM, CE marking |

Scope: "products with digital elements" placed on the EU market (software and hardware). Pure SaaS is mostly covered by NIS2 instead. Non-commercial open source is excluded; "open-source stewards" (foundations supporting OSS used commercially) have lighter obligations. Check the official guidance for your case; this is not legal advice.

## Pitfalls
<Warning>
**Common mistakes**

- Buying tools and turning on every rule: alert fatigue, ignored results.
- Threat modeling once, as a big document, never updated.
- Security team as a gate at the end instead of a partner in design.
- SBOM generated from the manifest rather than the built artifact (misses vendored or transitive code).
- Treating SCA findings as all equal instead of checking reachability and exploitation.
</Warning>

## Senior perspective
- Integrate security into existing rituals: design doc template has a "Security & privacy" section; the PR template has a security checklist; incidents have blameless postmortems.
- Measure: time to remediate criticals, % repos with SAST/SCA/secret scanning, % services with an up-to-date threat model, SLSA level of release artifacts.
- Security champions in each team scale better than a central team reviewing everything.
- Know your regulatory exposure (CRA, NIS2, PCI DSS 4.0, SOC 2) early; they drive requirements like SBOMs, logging and disclosure timelines.

## Interview questions
<Accordion title="What is STRIDE?">
A threat-classification mnemonic from Microsoft: Spoofing, Tampering, Repudiation, Information disclosure, Denial of service, Elevation of privilege. You apply it to each element and data flow of a diagram to find threats, then choose mitigations.
</Accordion>

<Accordion title="SAST vs DAST vs SCA?">
SAST analyses source code without running it (finds injection patterns early, false positives). DAST attacks the running app from outside (finds runtime/config issues, needs a deployed environment). SCA checks third-party dependencies for known vulnerabilities and licenses. You need all three; none finds business-logic authorization bugs well.
</Accordion>

<Accordion title="What is an SBOM and why does it matter?">
A machine-readable inventory of components in a software artifact (CycloneDX or SPDX). It lets you answer quickly whether you're affected by a new vulnerability, supports license compliance, and is increasingly required by customers and regulation (US federal, EU CRA).
</Accordion>

<Accordion title="What is SLSA?">
A framework of levels for supply-chain integrity. Build levels require signed, non-forgeable provenance from a hardened build platform so consumers can verify how and from what source an artifact was built. v1.2 (2025) added a Source Track.
</Accordion>

<Accordion title="When should a team do threat modeling?">
During design of new features that change trust boundaries, data sensitivity, authentication or external integrations, and again when the architecture changes significantly. Keep it lightweight and repeatable.
</Accordion>

<Accordion title="What does the EU Cyber Resilience Act require of software makers?">
From 11 September 2026, reporting actively exploited vulnerabilities and severe incidents (24 h / 72 h / final report). From 11 December 2027, secure-by-design requirements, vulnerability handling during the support period, an SBOM, security updates and conformity assessment/CE marking for products with digital elements sold in the EU.
</Accordion>

## Related
- [Security Index](/Software%20Engineering/Security/Security%20Index)
- [OWASP Top 10](/Software%20Engineering/Security/OWASP%20Top%2010) · [Backend Security Essentials](/Software%20Engineering/Security/Backend%20Security%20Essentials) · [API Security](/Software%20Engineering/Security/API%20Security) · [Secrets Management](/Software%20Engineering/Security/Secrets%20Management)
- [Supply Chain Security](/Software%20Engineering/Frontend/11%20Security/Supply%20Chain%20Security) · [Frontend Security - XSS, CSRF, CSP & Trusted Types](/Software%20Engineering/Frontend/11%20Security/Frontend%20Security%20-%20XSS%2C%20CSRF%2C%20CSP%20%26%20Trusted%20Types)
- [Technical Design Docs & RFCs](/Software%20Engineering/Career/Technical%20Design%20Docs%20%26%20RFCs) · [Code Review Best Practices](/Software%20Engineering/Career/Code%20Review%20Best%20Practices)

## Sources
- [European Commission: Cyber Resilience Act](https://digital-strategy.ec.europa.eu/en/policies/cyber-resilience-act) and [CRA reporting obligations](https://digital-strategy.ec.europa.eu/en/policies/cra-reporting)
- [Regulation (EU) 2024/2847](https://eur-lex.europa.eu/eli/reg/2024/2847/oj)
- [cyberresilienceact.eu: CRA explained (deadlines)](https://www.cyberresilienceact.eu/explained.html)
- [SLSA: Announcing SLSA v1.2 (2025-11)](https://slsa.dev/blog/2025/11/announce-slsa-v1.2), [SLSA v1.2 spec](https://slsa.dev/spec/v1.2/)
- [CISA: 2025 Minimum Elements for an SBOM (draft)](https://www.cisa.gov/resources-tools/resources/2025-minimum-elements-software-bill-materials-sbom)
- [NIST SP 800-218 SSDF](https://csrc.nist.gov/pubs/sp/800/218/final)
- [OWASP SAMM](https://owaspsamm.org/), [OWASP ASVS](https://github.com/OWASP/ASVS)
- Adam Shostack, *Threat Modeling: Designing for Security* (Wiley, 2014), [Threat Modeling Manifesto](https://www.threatmodelingmanifesto.org/)
- [CycloneDX](https://cyclonedx.org/), [SPDX](https://spdx.dev/)
- [CISA Known Exploited Vulnerabilities catalog](https://www.cisa.gov/known-exploited-vulnerabilities-catalog), [FIRST EPSS](https://www.first.org/epss/)
