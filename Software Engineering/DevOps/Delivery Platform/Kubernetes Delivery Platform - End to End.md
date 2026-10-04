---
title: "Kubernetes Delivery Platform - End to End"
tags: [devops, devops/kubernetes, devops/gitops, index]
aliases: [Delivery Platform, K8s Delivery Pipeline, Jenkins Argo Grafana, End to End Deployment]
updated: 2026-09-27
---
# Kubernetes Delivery Platform - End to End

<Note>
**TL;DR**

This note connects the delivery-platform notes into one story. A developer pushes code, and the change travels:
1. **Jenkins** builds, tests, scans and pushes an image, then bumps the image tag in a **GitOps repo**.
2. **Argo CD** notices the change and syncs it to Kubernetes.
3. **Argo Rollouts** shifts traffic gradually and checks error rates in **Prometheus**.
4. **Grafana** shows the metrics, logs (Loki) and traces (Tempo) for every pod.

Every note in this folder uses the same example system, *Acme Shop*: three Next.js micro-frontends plus a Node.js API.
</Note>

## The example system: Acme Shop

| Component | What it is | Route | Deep dive |
|---|---|---|---|
| `shell` | Next.js 16 host zone (layout, nav, home) | `/` | [Deploying Next.js Micro-frontends to Kubernetes](/Software%20Engineering/DevOps/Delivery%20Platform/Deploying%20Next.js%20Micro-frontends%20to%20Kubernetes) |
| `catalog` | Next.js 16 micro-frontend zone | `/catalog/*` | [Deploying Next.js Micro-frontends to Kubernetes](/Software%20Engineering/DevOps/Delivery%20Platform/Deploying%20Next.js%20Micro-frontends%20to%20Kubernetes) |
| `checkout` | Next.js 16 micro-frontend zone | `/checkout/*` | [Deploying Next.js Micro-frontends to Kubernetes](/Software%20Engineering/DevOps/Delivery%20Platform/Deploying%20Next.js%20Micro-frontends%20to%20Kubernetes) |
| `orders-api` | Node.js 24 LTS + Fastify + Postgres | `/api/orders/*` | [Deploying Node.js Apps to Kubernetes](/Software%20Engineering/DevOps/Delivery%20Platform/Deploying%20Node.js%20Apps%20to%20Kubernetes) |

Environments are namespaces: `shop-dev`, `shop-staging` and `shop-prod`. Platform tools run in `jenkins`, `argocd` and `monitoring`.

## The whole flow

{/* diagram source: /diagrams/diagram-156.mmd */}
![diagram](/diagrams/diagram-156.svg)

**Two principles hold the design together:**
1. **CI and CD are separate.** Jenkins never runs `kubectl apply` against production. Its last step is a Git commit. Argo CD is the only thing with write access to the cluster, so Git is the audit log and `git revert` is the rollback. This is called *pull-based deployment*; see [GitOps - Argo CD & Flux](/Software%20Engineering/DevOps/GitOps%20-%20Argo%20CD%20%26%20Flux).
2. **Everything is observable by default.** Every app:
   - logs JSON to stdout with a `trace_id`,
   - exposes `/metrics`,
   - exports OpenTelemetry traces.

   The platform collects all of it without any per-app setup.

## The GitOps repo

```text
acme/platform-gitops
├── applicationsets/
│   └── shop.yaml                 # generates 4 apps × 3 envs = 12 Argo CD Applications
├── apps/
│   ├── orders-api/
│   │   ├── base/                 # Deployment/Rollout, Service, HTTPRoute, HPA, PDB
│   │   └── overlays/{dev,staging,prod}/kustomization.yaml   # image tag lives here
│   ├── shell/  catalog/  checkout/   (same shape)
└── platform/
    ├── argocd/  gateway/
    └── monitoring/               # kube-prometheus-stack, loki, tempo, alloy values
```

The only line CI changes per release is the image tag in an overlay:

```yaml
# apps/orders-api/overlays/dev/kustomization.yaml
resources: [../../base]
namespace: shop-dev
images:
  - name: ghcr.io/acme/orders-api
    newTag: 3f9c2e1        # ← Jenkins commits this (git SHA of the build)
```

To promote dev → staging → prod, open a PR that copies the same tag into the next overlay. The PR is the approval gate.

## Reading order

| Step | Note | You'll learn |
|---|---|---|
| 0 | [Kubernetes Basics](/Software%20Engineering/DevOps/Kubernetes/Kubernetes%20Basics) → [Kubernetes Workloads in Production](/Software%20Engineering/DevOps/Kubernetes/Kubernetes%20Workloads%20in%20Production) | Pods, Deployments, probes, limits, HPA |
| 1 | [Kubernetes Operations - Logs, Debugging & Scheduling](/Software%20Engineering/DevOps/Delivery%20Platform/Kubernetes%20Operations%20-%20Logs%2C%20Debugging%20%26%20Scheduling) | `kubectl logs`, debugging broken pods, scheduling, rollouts |
| 2 | [Jenkins - Pipelines as Code](/Software%20Engineering/DevOps/Delivery%20Platform/Jenkins%20-%20Pipelines%20as%20Code) | Jenkinsfile, K8s agents, image builds, updating the GitOps repo |
| 3 | [Argo CD in Practice](/Software%20Engineering/DevOps/Delivery%20Platform/Argo%20CD%20in%20Practice) | Applications, sync policies, waves and hooks, ApplicationSets, promotion |
| 4 | [Argo Rollouts - Canary & Blue-Green](/Software%20Engineering/DevOps/Delivery%20Platform/Argo%20Rollouts%20-%20Canary%20%26%20Blue-Green) | Progressive delivery with automated analysis |
| 5 | [Grafana & the LGTM Stack](/Software%20Engineering/DevOps/Delivery%20Platform/Grafana%20%26%20the%20LGTM%20Stack) | Loki/LogQL, PromQL dashboards, Tempo traces, alerting |
| 6 | [Deploying Node.js Apps to Kubernetes](/Software%20Engineering/DevOps/Delivery%20Platform/Deploying%20Node.js%20Apps%20to%20Kubernetes) | `orders-api` from code to prod |
| 7 | [Deploying Next.js Micro-frontends to Kubernetes](/Software%20Engineering/DevOps/Delivery%20Platform/Deploying%20Next.js%20Micro-frontends%20to%20Kubernetes) | Multi-Zones, routing, caching, independent deploys |

## Hands-on lab: run it on your laptop

This lab recreates the platform locally with `kind`. Allow about 2 hours.

```bash
# 1. Cluster
kind create cluster --name acme
kubectl create namespace shop-dev

# 2. Argo CD
helm repo add argo https://argoproj.github.io/argo-helm
helm install argocd argo/argo-cd -n argocd --create-namespace
kubectl -n argocd get secret argocd-initial-admin-secret -o jsonpath='{.data.password}' | base64 -d; echo
kubectl -n argocd port-forward svc/argocd-server 8080:443   # UI on https://localhost:8080

# 3. Monitoring (metrics + Grafana)
helm repo add prometheus-community https://prometheus-community.github.io/helm-charts
helm repo add grafana https://grafana.github.io/helm-charts
helm install kps prometheus-community/kube-prometheus-stack -n monitoring --create-namespace
helm install loki grafana/loki -n monitoring -f loki-values.yaml      # single-binary for a lab
helm install alloy grafana/alloy -n monitoring -f alloy-values.yaml   # ships pod logs to Loki
kubectl -n monitoring port-forward svc/kps-grafana 3000:80

# 4. Point Argo CD at your GitOps repo (a fork of the layout above)
kubectl apply -n argocd -f applicationsets/shop.yaml

# 5. Watch it work
kubectl get applications -n argocd
kubectl -n shop-dev get pods -w
kubectl -n shop-dev logs deploy/orders-api -f
```

The Helm values and the ApplicationSet are in [Grafana & the LGTM Stack](/Software%20Engineering/DevOps/Delivery%20Platform/Grafana%20%26%20the%20LGTM%20Stack) and [Argo CD in Practice](/Software%20Engineering/DevOps/Delivery%20Platform/Argo%20CD%20in%20Practice). Jenkins is optional for the lab: bump the tag in Git by hand and watch Argo CD sync it.

<Info>
**Exercises**

1. **Deploy by Git commit.** Change `newTag` in the dev overlay and push. Time how long Argo CD takes to sync, then add a webhook to make it instant.
2. **Break it on purpose.** Set a wrong image tag and watch the pod go to `ImagePullBackOff`. Diagnose it with [Kubernetes Operations - Logs, Debugging & Scheduling](/Software%20Engineering/DevOps/Delivery%20Platform/Kubernetes%20Operations%20-%20Logs%2C%20Debugging%20%26%20Scheduling).
3. **Drift.** Run `kubectl scale deploy/orders-api --replicas=5` by hand. With `selfHeal: true`, Argo CD reverts it. Understand why that's a feature.
4. **Logs.** In Grafana Explore, find every 5xx from `orders-api` in the last hour with LogQL, then jump from one log line to its trace.
5. **Canary.** Deploy a version that returns 500 on 20% of requests. The Rollouts analysis should abort and roll back automatically.
6. **Independent deploys.** Release a new `catalog` without touching `shell` or `checkout`, then roll back only `catalog`.
</Info>

## Day-2 questions and where they're answered

| Question | Answer in |
|---|---|
| "Where are the logs for the pod that crashed 10 minutes ago?" | `kubectl logs --previous` (short-term) or Loki (long-term): [Kubernetes Operations - Logs, Debugging & Scheduling](/Software%20Engineering/DevOps/Delivery%20Platform/Kubernetes%20Operations%20-%20Logs%2C%20Debugging%20%26%20Scheduling), [Grafana & the LGTM Stack](/Software%20Engineering/DevOps/Delivery%20Platform/Grafana%20%26%20the%20LGTM%20Stack) |
| "Why is prod different from Git?" | Argo CD diff and OutOfSync status: [Argo CD in Practice](/Software%20Engineering/DevOps/Delivery%20Platform/Argo%20CD%20in%20Practice) |
| "How do I roll back?" | `git revert` in the GitOps repo, or `argocd app rollback`, or abort the Rollout: [Argo CD in Practice](/Software%20Engineering/DevOps/Delivery%20Platform/Argo%20CD%20in%20Practice), [Argo Rollouts - Canary & Blue-Green](/Software%20Engineering/DevOps/Delivery%20Platform/Argo%20Rollouts%20-%20Canary%20%26%20Blue-Green) |
| "Why is my pod Pending?" | Scheduling constraints and resources: [Kubernetes Operations - Logs, Debugging & Scheduling](/Software%20Engineering/DevOps/Delivery%20Platform/Kubernetes%20Operations%20-%20Logs%2C%20Debugging%20%26%20Scheduling) |
| "How do I run DB migrations safely?" | PreSync hook Job plus expand/contract migrations: [Deploying Node.js Apps to Kubernetes](/Software%20Engineering/DevOps/Delivery%20Platform/Deploying%20Node.js%20Apps%20to%20Kubernetes), [Database Migrations & Schema Evolution](/Software%20Engineering/Databases/Core%20Concepts/Database%20Migrations%20%26%20Schema%20Evolution) |
| "Why do users get 404s on JS files after a Next.js deploy?" | assetPrefix and deployment skew: [Deploying Next.js Micro-frontends to Kubernetes](/Software%20Engineering/DevOps/Delivery%20Platform/Deploying%20Next.js%20Micro-frontends%20to%20Kubernetes) |
| "What's our error budget?" | SLO burn-rate alerts: [Grafana & the LGTM Stack](/Software%20Engineering/DevOps/Delivery%20Platform/Grafana%20%26%20the%20LGTM%20Stack), [Observability - Logs, Metrics & Traces](/Software%20Engineering/DevOps/Observability%20-%20Logs%2C%20Metrics%20%26%20Traces) |
| "Where do secrets come from?" | External Secrets Operator: [Argo CD in Practice](/Software%20Engineering/DevOps/Delivery%20Platform/Argo%20CD%20in%20Practice), [Secrets Management](/Software%20Engineering/Security/Secrets%20Management) |

## Senior perspective
- **Pick fewer tools.** Jenkins + Argo CD + Rollouts + the Grafana stack is a lot to operate. A small team might use GitHub Actions + Argo CD + Grafana Cloud instead. The *architecture* (CI builds, Git declares, a controller reconciles, telemetry flows to one place) matters more than the brand names. See [Platform Engineering & Internal Developer Platforms](/Software%20Engineering/DevOps/Platform%20Engineering%20%26%20Internal%20Developer%20Platforms).
- **Treat the platform as a product.** Give teams golden-path templates (a shared Jenkins library, a base Kustomize component, a default dashboard), so a new service gets CI/CD and observability in minutes.
- **Security lives in the pipeline.** Pin actions and plugins, sign images, generate SBOMs, and give Argo CD least-privilege `AppProject`s. See [Supply Chain Security](/Software%20Engineering/Frontend/11%20Security/Supply%20Chain%20Security) and [Secure SDLC & Threat Modeling](/Software%20Engineering/Security/Secure%20SDLC%20%26%20Threat%20Modeling).
- **Measure the platform** with DORA metrics: deploy frequency, lead time, change failure rate, time to restore, and rework rate. See [CI-CD Pipelines](/Software%20Engineering/DevOps/CI-CD%20Pipelines).

## Related
- [DevOps Index](/Software%20Engineering/DevOps/DevOps%20Index) · [Kubernetes](/Software%20Engineering/DevOps/Kubernetes/Kubernetes) · [CI-CD Pipelines](/Software%20Engineering/DevOps/CI-CD%20Pipelines) · [GitOps - Argo CD & Flux](/Software%20Engineering/DevOps/GitOps%20-%20Argo%20CD%20%26%20Flux) · [Observability - Logs, Metrics & Traces](/Software%20Engineering/DevOps/Observability%20-%20Logs%2C%20Metrics%20%26%20Traces)
- [Micro-frontends & Module Federation](/Software%20Engineering/Frontend/09%20Architecture/Micro-frontends%20%26%20Module%20Federation) · [Node.js Index](/Software%20Engineering/Backend/The%20Hard%20Parts%20of%20Servers%20%26%20Node.js/Node.js%20Index) · [Microservices Architecture](/Software%20Engineering/DevOps/Microservices%20Architecture)
- [Site Reliability Engineering & Incident Management](/Software%20Engineering/DevOps/Site%20Reliability%20Engineering%20%26%20Incident%20Management)

## Sources
- [Argo CD documentation](https://argo-cd.readthedocs.io/)
- [Argo Rollouts documentation](https://argoproj.github.io/argo-rollouts/)
- [Jenkins User Handbook: Pipeline](https://www.jenkins.io/doc/book/pipeline/)
- [Grafana documentation](https://grafana.com/docs/)
- [OpenGitOps principles (CNCF)](https://opengitops.dev/)
- [kind: Quick start](https://kind.sigs.k8s.io/docs/user/quick-start/)
- [Next.js: Multi-Zones guide](https://nextjs.org/docs/app/guides/multi-zones)
