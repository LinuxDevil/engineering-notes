---
tags: [devops, devops/whats-new]
aliases: [DevOps 2025-2026, DevOps Changelog, What's New in DevOps]
updated: 2026-09-27
---
# What's New in DevOps 2025-2026

> [!tldr] TL;DR
> The big themes of 2025–2026: **Kubernetes matured** (sidecars, in-place resize, user namespaces, HPA scale-to-zero), **ingress-nginx was retired** and the ecosystem moved to the **Gateway API**, **supply-chain security** became mandatory after the tj-actions compromise and free hardened images, **IaC diverged** (Terraform under IBM vs OpenTofu in CNCF), **OpenTelemetry** added profiling, and **AI agents** arrived in CI and platform tooling. Every item links to the note with details; all dates were verified against official sources on 2026-09-27.

## Timeline

| Date | Change | Note |
|---|---|---|
| 2025-02-27 | IBM completes acquisition of HashiCorp (Terraform owner) | [[Infrastructure as Code]] |
| 2025-03-14/15 | `tj-actions/changed-files` GitHub Action compromised (CVE-2025-30066) | [[CI-CD Pipelines]] |
| 2025-03-24 | IngressNightmare: critical ingress-nginx CVE-2025-1974 | [[Kubernetes Networking - Services, Ingress & Gateway API]] |
| 2025-04-23 | Kubernetes v1.33: native sidecar containers GA | [[Kubernetes Workloads in Production]] |
| 2025-04-23 | OpenTofu accepted into CNCF (Sandbox) | [[Infrastructure as Code]] |
| 2025-05-06 | Argo CD 3.0 GA | [[GitOps - Argo CD & Flux]] |
| 2025-06-24 | OpenTofu 1.10: OCI registries, native S3 locking | [[Infrastructure as Code]] |
| 2025-08-15 | GitHub Actions policies can enforce SHA pinning and block actions | [[CI-CD Pipelines]] |
| 2025-08-27 | Kubernetes v1.34: DRA core GA, swap GA, tracing GA, no removals | [[Kubernetes Workloads in Production]] |
| 2025-08-28 → 09-29 | Bitnami free image catalog reduced; versioned tags to `bitnamilegacy` | [[Dockerfile Best Practices & Image Security]] |
| 2025-09-25 | GitHub Copilot coding agent GA (runs on GitHub Actions) | [[CI-CD Pipelines]] |
| 2025-10-06 | Gateway API v1.4 (BackendTLSPolicy to Standard) | [[Kubernetes Networking - Services, Ingress & Gateway API]] |
| 2025-10 | Ubuntu 25.10: sudo-rs and Rust coreutils by default | [[Linux & Shell Essentials]] |
| 2025-10-19/20 | AWS us-east-1 outage (DynamoDB DNS race condition, ~14.5 h) | [[Site Reliability Engineering & Incident Management]] |
| 2025-10-22 | Rails 8.1 | [[Rails Project Structure]] |
| 2025-11-10 | Docker Engine 29: containerd image store default | [[Docker Basics]] |
| 2025-11-11 | ingress-nginx retirement announced | [[Kubernetes Networking - Services, Ingress & Gateway API]] |
| 2025-11-12 | Helm 4.0 (server-side apply, Wasm plugins) | [[Helm & Kustomize]] |
| 2025-11-18 | Cloudflare global outage (oversized feature file) | [[Site Reliability Engineering & Incident Management]] |
| 2025-11-19 | Terraform 1.14: `terraform query`, `action` blocks | [[Infrastructure as Code]] |
| 2025-12-02 | Docker Compose v5 "Mont Blanc" (Bake builds, SDK) | [[Docker Compose]] |
| 2025-12-02 | Prometheus 3.8: native histograms stable | [[Observability - Logs, Metrics & Traces]] |
| 2025-12-09 | OpenTofu 1.11: ephemeral resources, write-only attributes | [[Infrastructure as Code]] |
| 2025-12-17 | Kubernetes v1.35: in-place Pod resize GA; kubelet fails on cgroup v1 by default; last containerd 1.x; ipvs deprecated | [[Kubernetes Workloads in Production]] |
| 2025-12-17 | Docker Hardened Images free and open source (Apache 2.0) | [[Dockerfile Best Practices & Image Security]] |
| 2026-01-01 | GitHub-hosted runner prices cut up to 39%; self-hosted fee later postponed | [[CI-CD Pipelines]] |
| 2026-01-05 | DORA metrics guide: five metrics incl. deployment rework rate | [[CI-CD Pipelines]] |
| 2026-01-29 | Kubernetes Steering + SRC statement urging ingress-nginx migration | [[Kubernetes Networking - Services, Ingress & Gateway API]] |
| 2026-02-25 | GitHub Copilot CLI GA | [[CI-CD Pipelines]] |
| 2026-02-27 | Gateway API v1.5 (largest release, features to Standard) | [[Kubernetes Networking - Services, Ingress & Gateway API]] |
| 2026-03 | OpenTelemetry Profiles signal enters public alpha | [[Observability - Logs, Metrics & Traces]] |
| 2026-03-20 | ingress2gateway v1.0 (30+ ingress-nginx annotations) | [[Kubernetes Networking - Services, Ingress & Gateway API]] |
| 2026-03-24 | ingress-nginx retired | [[Kubernetes Networking - Services, Ingress & Gateway API]] |
| 2026-03-24 | CNCF + SlashData platform engineering report | [[Platform Engineering & Internal Developer Platforms]] |
| 2026-04-22 | Kubernetes v1.36: user namespaces GA, mutating admission policies GA, gitRepo volume removed | [[Kubernetes Workloads in Production]] |
| 2026-05-14 | OpenTofu 1.12 | [[Infrastructure as Code]] |
| 2026-07-08 / 11-11 | Helm 3 end of bug fixes / security fixes | [[Helm & Kustomize]] |
| 2026-08-26 | Kubernetes v1.37: HPA scale-to-zero beta (on by default), Pod certificates GA, kube-dns deprecated | [[Kubernetes Workloads in Production]] |
| 2026-08-26 | Terraform 1.16.0 | [[Infrastructure as Code]] |
| pending | Git 3.0 (SHA-256, reftable, `main`, Rust) announced, no date | [[Git Essentials & Workflows]] |

## What to do about it (checklist)

- [ ] Inventory ingress-nginx usage and plan the Gateway API migration.
- [ ] Nodes on cgroup v2 and containerd 2.x before upgrading past Kubernetes v1.35.
- [ ] Pin GitHub Actions to SHAs and enforce it by policy; minimise `GITHUB_TOKEN` permissions.
- [ ] Mirror third-party images and charts you depend on (Bitnami lesson).
- [ ] Migrate from Helm 3 to Helm 4 before Helm 3 security fixes end (2026-11-11).
- [ ] Decide Terraform vs OpenTofu deliberately and pin versions.
- [ ] Map hidden single-region and control-plane dependencies (AWS Oct 2025 lesson).
- [ ] Add guardrails for AI agents in CI: scoped tokens, required reviews, sandboxed runners.

## Related
- [[DevOps Index]] · [[Software Engineering]]
