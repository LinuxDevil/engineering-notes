---
title: "Helm & Kustomize"
tags: [devops, devops/kubernetes]
aliases: [Helm, Kustomize, Helm Charts, Kubernetes Packaging]
updated: 2026-09-27
---
# Helm & Kustomize

<Note>
**TL;DR**

Raw YAML does not scale to many services and environments. **Helm** is a package manager: a **chart** is templated YAML plus `values.yaml`, installed as a versioned **release** you can upgrade and roll back. **Kustomize** (built into `kubectl -k`) is template-free: a **base** plus per-environment **overlays** that patch it. Many teams use Helm for third-party software and Kustomize (or Helm) for their own apps, applied by a GitOps controller. **Helm 4** shipped on 2025-11-12 with server-side apply and WebAssembly plugins.
</Note>

## Why package manifests at all?

A single service in three environments needs a Deployment, Service, HPA, PDB, ConfigMap, ServiceAccount, HTTPRoute... times three, with small differences (replicas, image tag, hostnames, resources). Copy-paste drifts. You want one source with parameters.

## Helm

```text
orders-api/
  Chart.yaml          # name, version (chart), appVersion, dependencies
  values.yaml         # defaults
  values-prod.yaml    # environment overrides (often kept in a separate repo)
  templates/
    deployment.yaml
    service.yaml
    hpa.yaml
    _helpers.tpl      # named templates (labels, names)
    NOTES.txt
```

```yaml
# templates/deployment.yaml (excerpt)
apiVersion: apps/v1
kind: Deployment
metadata:
  name: {{ include "orders.fullname" . }}
  labels: {{- include "orders.labels" . | nindent 4 }}
spec:
  replicas: {{ .Values.replicaCount }}
  template:
    metadata:
      annotations:
        checksum/config: {{ include (print $.Template.BasePath "/configmap.yaml") . | sha256sum }}
    spec:
      containers:
        - name: api
          image: "{{ .Values.image.repository }}:{{ .Values.image.tag | default .Chart.AppVersion }}"
          resources: {{- toYaml .Values.resources | nindent 12 }}
```

```bash
helm repo add bitnami https://charts.bitnami.com/bitnami        # classic repo
helm install pg oci://registry-1.docker.io/bitnamicharts/postgresql  # charts are OCI artifacts now
helm upgrade --install orders ./orders-api -f values-prod.yaml --set image.tag=1.8.3 -n orders --atomic
helm list -n orders
helm history orders -n orders
helm rollback orders 3 -n orders
helm template ./orders-api -f values-prod.yaml | kubectl diff -f -   # render locally and diff
helm lint ./orders-api
```

Strengths: huge ecosystem of charts, releases with history and rollback, hooks (pre-install migrations), dependencies. Weaknesses: Go templating of YAML is error-prone (indentation!), charts become "every field is a value", and logic hides in templates.

## Kustomize

```text
k8s/
  base/
    kustomization.yaml
    deployment.yaml
    service.yaml
  overlays/
    staging/kustomization.yaml
    prod/
      kustomization.yaml
      replicas-patch.yaml
```

```yaml
# overlays/prod/kustomization.yaml
apiVersion: kustomize.config.k8s.io/v1beta1
kind: Kustomization
namespace: orders-prod
resources:
  - ../../base
images:
  - name: ghcr.io/acme/orders-api
    newTag: 1.8.3
patches:
  - path: replicas-patch.yaml
configMapGenerator:
  - name: orders-config
    literals: [LOG_LEVEL=warn]      # hashed name → Pods roll on change
```

```bash
kubectl kustomize overlays/prod     # render
kubectl apply -k overlays/prod      # apply
```

## Helm vs Kustomize

| | Helm | Kustomize |
|---|---|---|
| Approach | Templates + values | Base + patches (plain YAML) |
| Distribution | Charts in repos/OCI registries | Git directories |
| Release tracking / rollback | Yes (release history in Secrets) | No (Git history / GitOps) |
| Learning curve | Go templates, Sprig functions | Patching semantics |
| Best for | Third-party software, reusable internal "golden" charts | Your own apps with a few environment variations |

They combine: Kustomize can inflate a Helm chart (`helmCharts:`), and Argo CD/Flux support both. See [GitOps - Argo CD & Flux](/Software%20Engineering/DevOps/GitOps%20-%20Argo%20CD%20%26%20Flux).

<Info>
**🆕 2025–2026**

- **Helm 4.0.0 (2025-11-12, KubeCon NA; Helm's 10th anniversary)**: first major release in six years. **Server-side apply** replaces the client-side three-way merge (field ownership tracked by the API server), redesigned plugin system with optional **WebAssembly** plugins, post-renderers as plugins, kstatus-based waiting, content-based chart caching, reproducible chart archives, `slog` logging for SDK users. **Helm 3 support**: bug fixes until **2026-07-08**, security fixes until **2026-11-11**, so plan the migration now.
- **Bitnami catalog change (Aug–Sep 2025)**: many charts depended on free Bitnami images that moved to `bitnamilegacy`; charts broke unless images were overridden. Mirror dependencies you rely on. Details in [Dockerfile Best Practices & Image Security](/Software%20Engineering/DevOps/Docker/Dockerfile%20Best%20Practices%20%26%20Image%20Security).
</Info>

## Pitfalls

<Warning>
**Common pitfalls**

- Secrets in `values.yaml` committed to Git. Use External Secrets/SOPS.
- `helm upgrade` without `--atomic`/`--wait`: a failed release leaves half-applied resources.
- Charts that template every field: the chart becomes a worse copy of the Kubernetes API. Prefer opinionated defaults with a few knobs.
- Drift: someone runs `kubectl edit` on a Helm-managed object; the next upgrade may fight it (SSA in Helm 4 makes ownership conflicts explicit).
- Changing a Deployment's `selector` labels in a chart: selectors are immutable, upgrade fails.
</Warning>

## Senior perspective

- Always **render and diff** in CI (`helm template | kubeconform`, `helm diff`, Argo CD diff) so reviewers see real YAML, not template changes.
- Maintain a small number of internal golden charts or Kustomize components owned by the platform team; app teams supply values. That is the "paved road" of [Platform Engineering & Internal Developer Platforms](/Software%20Engineering/DevOps/Platform%20Engineering%20%26%20Internal%20Developer%20Platforms).
- Pin chart versions and verify provenance (OCI registries support signing with cosign).

## Interview questions

<Accordion title="What is a Helm release?">
An installed instance of a chart with a name, namespace and revision history (stored as Secrets in the namespace). Each upgrade creates a new revision you can roll back to.
</Accordion>

<Accordion title="Helm or Kustomize for our microservices?">
Kustomize if configs differ modestly between environments and you want plain YAML and GitOps. Helm if you need to distribute a reusable package, have many parameters, or need hooks and release history. Many teams use a shared internal Helm chart for all services.
</Accordion>

<Accordion title="How do you make Pods restart when a ConfigMap changes?">
Add a checksum of the ConfigMap as a Pod template annotation (Helm), or use Kustomize's `configMapGenerator` which appends a content hash to the name, so the Pod template changes and a rollout happens.
</Accordion>

<Accordion title="What did Helm 4 change about applying resources?">
It uses Kubernetes server-side apply instead of Helm's client-side three-way merge, so the API server tracks field managers and reports conflicts when another tool owns a field.
</Accordion>

## Related
- [Kubernetes](/Software%20Engineering/DevOps/Kubernetes/Kubernetes) · [Kubernetes Basics](/Software%20Engineering/DevOps/Kubernetes/Kubernetes%20Basics) · [Kubernetes Workloads in Production](/Software%20Engineering/DevOps/Kubernetes/Kubernetes%20Workloads%20in%20Production) · [GitOps - Argo CD & Flux](/Software%20Engineering/DevOps/GitOps%20-%20Argo%20CD%20%26%20Flux) · [DevOps Index](/Software%20Engineering/DevOps/DevOps%20Index)
- [Infrastructure as Code](/Software%20Engineering/DevOps/Infrastructure%20as%20Code) · [CI-CD Pipelines](/Software%20Engineering/DevOps/CI-CD%20Pipelines)

## Sources
- [Helm docs](https://helm.sh/docs/)
- [Helm blog: Helm 4 Released](https://helm.sh/blog/helm-4-released/)
- [CNCF: Helm marks 10 years with release of version 4 (2025-11-12)](https://www.cncf.io/announcements/2025/11/12/helm-marks-10-years-with-release-of-version-4/)
- [Kubernetes docs: Declarative management with Kustomize](https://kubernetes.io/docs/tasks/manage-kubernetes-objects/kustomization/)
- [Kustomize docs](https://kubectl.docs.kubernetes.io/references/kustomize/)
