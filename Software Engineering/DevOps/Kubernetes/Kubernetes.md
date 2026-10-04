---
title: "Kubernetes"
tags: [devops, devops/kubernetes, index]
aliases: [K8s, Kubernetes Home, Kubernetes Index, Kubernetes MOC]
updated: 2026-09-27
---
# Kubernetes

<Note>
**TL;DR**

Hub note for Kubernetes. Learn the objects and the course walkthrough in [Kubernetes Basics](/Software%20Engineering/DevOps/Kubernetes/Kubernetes%20Basics), then make workloads production-ready ([Kubernetes Workloads in Production](/Software%20Engineering/DevOps/Kubernetes/Kubernetes%20Workloads%20in%20Production)), understand traffic ([Kubernetes Networking - Services, Ingress & Gateway API](/Software%20Engineering/DevOps/Kubernetes/Kubernetes%20Networking%20-%20Services%2C%20Ingress%20%26%20Gateway%20API)), package them ([Helm & Kustomize](/Software%20Engineering/DevOps/Kubernetes/Helm%20%26%20Kustomize)) and deliver them with [GitOps - Argo CD & Flux](/Software%20Engineering/DevOps/GitOps%20-%20Argo%20CD%20%26%20Flux).
</Note>

{/* diagram source: /diagrams/diagram-176.mmd */}
![diagram](/diagrams/diagram-176.svg)

## Notes in order
| Level | Note | What you learn |
|---|---|---|
| Beginner | [Kubernetes Basics](/Software%20Engineering/DevOps/Kubernetes/Kubernetes%20Basics) | Architecture, Pods, Deployments, Services, Ingress, Skaffold (course notes) |
| Intermediate | [Kubernetes Networking - Services, Ingress & Gateway API](/Software%20Engineering/DevOps/Kubernetes/Kubernetes%20Networking%20-%20Services%2C%20Ingress%20%26%20Gateway%20API) | CNI, DNS, kube-proxy, Ingress limits, Gateway API, ingress-nginx migration |
| Intermediate | [Helm & Kustomize](/Software%20Engineering/DevOps/Kubernetes/Helm%20%26%20Kustomize) | Charts, releases, overlays, Helm 4 |
| Senior | [Kubernetes Workloads in Production](/Software%20Engineering/DevOps/Kubernetes/Kubernetes%20Workloads%20in%20Production) | Probes, requests/limits, HPA/VPA, ConfigMaps/Secrets, PDBs, sidecars, RBAC, release changes |
| Senior | [GitOps - Argo CD & Flux](/Software%20Engineering/DevOps/GitOps%20-%20Argo%20CD%20%26%20Flux) | Git as source of truth, reconciliation, promotion |

## kubectl cheat sheet
```bash
kubectl config get-contexts / use-context <ctx>
kubectl get pods -A -o wide
kubectl describe pod <p>                  # events explain Pending / CrashLoopBackOff
kubectl logs <p> -f --previous
kubectl exec -it <p> -- sh
kubectl debug -it <p> --image=busybox --target=<container>   # ephemeral debug container
kubectl apply -f k8s/ ; kubectl diff -f k8s/
kubectl rollout status|history|undo deploy/<d>
kubectl top pods                          # needs metrics-server
kubectl auth can-i <verb> <resource> --as=<user>
```

<Info>
**🆕 2025–2026 in Kubernetes**

- 2025-04-23 · v1.33: native sidecars GA → [Kubernetes Workloads in Production](/Software%20Engineering/DevOps/Kubernetes/Kubernetes%20Workloads%20in%20Production)
- 2025-08-27 · v1.34: no deprecations/removals, DRA stable → [Kubernetes Workloads in Production](/Software%20Engineering/DevOps/Kubernetes/Kubernetes%20Workloads%20in%20Production)
- 2025-11-11 · ingress-nginx retirement announced; retired 2026-03-24 → [Kubernetes Networking - Services, Ingress & Gateway API](/Software%20Engineering/DevOps/Kubernetes/Kubernetes%20Networking%20-%20Services%2C%20Ingress%20%26%20Gateway%20API)
- 2025-11-12 · Helm 4 → [Helm & Kustomize](/Software%20Engineering/DevOps/Kubernetes/Helm%20%26%20Kustomize)
- 2025-12-17 · v1.35: in-place Pod resize GA, kubelet fails on cgroup v1 by default, last containerd 1.x, ipvs deprecated → [Kubernetes Workloads in Production](/Software%20Engineering/DevOps/Kubernetes/Kubernetes%20Workloads%20in%20Production)
- 2026-02-27 · Gateway API v1.5 → [Kubernetes Networking - Services, Ingress & Gateway API](/Software%20Engineering/DevOps/Kubernetes/Kubernetes%20Networking%20-%20Services%2C%20Ingress%20%26%20Gateway%20API)
- 2026-04-22 · v1.36: user namespaces GA, mutating admission policies GA, gitRepo volume removed → [Kubernetes Workloads in Production](/Software%20Engineering/DevOps/Kubernetes/Kubernetes%20Workloads%20in%20Production)
- 2026-08-26 · v1.37: HPA scale-to-zero beta, Pod certificates GA, kube-dns deprecated → [Kubernetes Workloads in Production](/Software%20Engineering/DevOps/Kubernetes/Kubernetes%20Workloads%20in%20Production)
</Info>

## Related
- [DevOps Index](/Software%20Engineering/DevOps/DevOps%20Index) · [Docker](/Software%20Engineering/DevOps/Docker/Docker) · [Microservices Architecture](/Software%20Engineering/DevOps/Microservices%20Architecture) · [Containers vs VMs vs Serverless](/Software%20Engineering/DevOps/Containers%20vs%20VMs%20vs%20Serverless)
- Hands-on delivery: [Kubernetes Delivery Platform - End to End](/Software%20Engineering/DevOps/Delivery%20Platform/Kubernetes%20Delivery%20Platform%20-%20End%20to%20End) · [Kubernetes Operations - Logs, Debugging & Scheduling](/Software%20Engineering/DevOps/Delivery%20Platform/Kubernetes%20Operations%20-%20Logs%2C%20Debugging%20%26%20Scheduling) · [Argo CD in Practice](/Software%20Engineering/DevOps/Delivery%20Platform/Argo%20CD%20in%20Practice) · [Grafana & the LGTM Stack](/Software%20Engineering/DevOps/Delivery%20Platform/Grafana%20%26%20the%20LGTM%20Stack)
- Original course study plan: [Microservices Study Plan (original)](/Software%20Engineering/DevOps/_Archive/Microservices%20Study%20Plan%20(original))
