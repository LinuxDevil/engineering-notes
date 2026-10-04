---
title: "Kubernetes Operations - Logs, Debugging & Scheduling"
tags: [devops, devops/kubernetes]
aliases: [Kubernetes Troubleshooting, kubectl debugging, Day-2 Kubernetes, K8s Scheduling, CrashLoopBackOff, Pod Pending, kubectl logs, Kubernetes Operator Handbook]
updated: 2026-09-27
---
# Kubernetes Operations - Logs, Debugging & Scheduling

<Note>
**TL;DR**

Day-2 Kubernetes is three loops. **Read** what happened (`kubectl logs`, `kubectl events`, Grafana/Loki), **diagnose** by comparing `spec` with `status` (`describe`, `get -o yaml`, `debug`), and **understand placement** (the scheduler filters then scores nodes; affinity, topology spread, taints and priorities steer it). Every object is kept in shape by a controller's reconciliation loop, and in a GitOps cluster Argo CD is one more reconciler, so fixes go into Git, not `kubectl edit`. This note is the operator's handbook for Acme Shop; production manifests are in [Kubernetes Workloads in Production](/Software%20Engineering/DevOps/Kubernetes/Kubernetes%20Workloads%20in%20Production).
</Note>

## 1. Logs

### 1.1 `kubectl logs` in practice

```bash
# One pod (first container if you don't say which)
kubectl -n shop-prod logs orders-api-7c9d8f6b5d-x2kqp

# Follow, only the last 100 lines, only the last 15 minutes, with timestamps
kubectl -n shop-prod logs -f --tail=100 --since=15m --timestamps orders-api-7c9d8f6b5d-x2kqp

# The PREVIOUS container instance: the one that just crashed (CrashLoopBackOff, OOMKilled)
kubectl -n shop-prod logs --previous orders-api-7c9d8f6b5d-x2kqp

# A specific container (sidecar, init container)
kubectl -n shop-prod logs orders-api-7c9d8f6b5d-x2kqp -c db-migrate

# Every container in the pod, prefixed with [pod/.../container]
kubectl -n shop-prod logs orders-api-7c9d8f6b5d-x2kqp --all-containers --prefix

# All pods behind a label (defaults to --tail=10 per pod, max 5 concurrent streams)
kubectl -n shop-prod logs -l app=orders-api --prefix --tail=50 -f --max-log-requests=10

# Via the owner: picks ONE pod of the Deployment...
kubectl -n shop-prod logs deployment/orders-api --since=10m
# ...or all of them (newer kubectl; implies --prefix)
kubectl -n shop-prod logs deployment/orders-api --all-pods --since=10m

# Jobs and CronJob runs
kubectl -n shop-prod logs job/orders-db-migrate-9f1c2ab

# Pretty-print pino JSON on the fly
kubectl -n shop-prod logs deploy/orders-api --since=5m | npx pino-pretty
kubectl -n shop-prod logs deploy/orders-api --since=5m | jq -c 'select(.level=="error") | {time, msg, trace_id}'
```

**stern** (maintained at `stern/stern`) is the tool you want during incidents: tails many pods by regex, survives pod restarts and new pods appearing, colours by pod, and understands JSON.

```bash
stern -n shop-prod orders-api --since 10m                     # regex on pod name
stern -n shop-prod -l app=orders-api -c api --tail 20          # label selector, one container
stern -n shop-prod 'catalog|checkout|shell' --include '"level":"error"'
stern -n shop-prod orders-api -o json | jq -r '.message'       # raw line for piping
stern -A --field-selector spec.nodeName=ip-10-0-3-17.eu-west-1.compute.internal   # everything on one node
```

**kubetail**: the name covers the old bash script (`kubetail orders-api`) and the newer `kubetail-org/kubetail` project, a browser-based log dashboard that runs in the cluster. Both are convenience layers over the same kubelet log API.

### 1.2 Where logs actually live

{/* diagram source: /diagrams/diagram-157.mmd */}
![diagram](/diagrams/diagram-157.svg)

- The container runtime writes each line in the **CRI log format** (`2026-09-27T10:15:02.123Z stdout F {"level":"info",...}`) to `/var/log/pods/<namespace>_<pod>_<uid>/<container>/<restartCount>.log`.
- The **kubelet rotates** these files: `containerLogMaxSize` (default 10Mi) and `containerLogMaxFiles` (default 5) in the kubelet config. `kubectl logs` only sees what is still on disk; `--previous` only reaches one restart back.
- When a Pod is **deleted or its node dies, its logs go with it**. That is why central collection (Alloy → Loki) is non-negotiable in production.
- Why **stdout JSON**: the platform collects stdout for free (no sidecar, no log files inside the container, works with `readOnlyRootFilesystem`), and JSON makes every field queryable in Loki without regexes. Log to files inside the container and you lose `kubectl logs`, rotation and collection in one go.
- Multi-line stack traces break line-based collection. With JSON, the stack is one escaped field in one line, so the problem disappears.

### 1.3 kubectl vs Argo CD UI vs Grafana

| | `kubectl logs` / stern | Argo CD UI (pod → Logs tab) | Grafana (Loki) |
|---|---|---|---|
| Source | kubelet, live files on the node | kubelet via the Argo CD API server | Loki, collected copy |
| History | current + one previous container, rotation-limited | same as kubectl | retention period (e.g. 31 days), survives pod deletion |
| Across pods/services | label selector, stern regex | one pod or container at a time | any query across all apps, correlated with traces |
| Access control | Kubernetes RBAC (`pods/log` get) | Argo CD RBAC: **since Argo CD 3.0 logs RBAC is enforced by default**, users need an explicit `logs, get` grant | Grafana data source permissions |
| Best for | live debugging of one workload | app teams without kubectl access, "what just happened after this sync" | incidents, history, aggregation, alerting |

Rule of thumb: kubectl for **now**, Grafana for **what happened** and **how often**. LogQL recipes are in [Grafana & the LGTM Stack](/Software%20Engineering/DevOps/Delivery%20Platform/Grafana%20%26%20the%20LGTM%20Stack).

## 2. The debugging toolbox

### 2.1 Read state

```bash
# Wide listing: node, IP, restarts, age
kubectl -n shop-prod get pods -o wide -l app=orders-api

# Describe: spec + status + conditions + the last events (the first thing to read)
kubectl -n shop-prod describe pod orders-api-7c9d8f6b5d-x2kqp

# Events for one object, sorted, watchable (GA since v1.28)
kubectl -n shop-prod events --for pod/orders-api-7c9d8f6b5d-x2kqp
kubectl -n shop-prod events --for deployment/orders-api --watch
kubectl events -A --types=Warning                                  # everything going wrong right now

# The full object, including status and managedFields
kubectl -n shop-prod get pod orders-api-7c9d8f6b5d-x2kqp -o yaml

# jsonpath / custom-columns for targeted questions
kubectl -n shop-prod get pod orders-api-7c9d8f6b5d-x2kqp \
  -o jsonpath='{range .status.containerStatuses[*]}{.name}{"\t"}{.restartCount}{"\t"}{.lastState.terminated.reason}{"\t"}{.lastState.terminated.exitCode}{"\n"}{end}'

kubectl -n shop-prod get pods -o custom-columns='POD:.metadata.name,NODE:.spec.nodeName,PHASE:.status.phase,IMAGE:.spec.containers[0].image'

# Which pods are not Running?
kubectl get pods -A --field-selector=status.phase!=Running,status.phase!=Succeeded
```

Events are kept only for about an hour by default (API server `--event-ttl`), so capture them early or rely on an event exporter shipping them to Loki.

### 2.2 Get inside

```bash
# Shell in a running container (only if the image has a shell)
kubectl -n shop-prod exec -it orders-api-7c9d8f6b5d-x2kqp -c api -- sh
kubectl -n shop-prod exec deploy/orders-api -- node -e "fetch('http://localhost:8080/ready').then(r=>console.log(r.status))"

# Talk to it from your laptop
kubectl -n shop-prod port-forward svc/orders-api 8080:80
kubectl -n shop-prod port-forward pod/orders-api-7c9d8f6b5d-x2kqp 9229:9229   # Node inspector, if enabled
curl -s localhost:8080/metrics | grep http_request_duration_seconds_count
```

### 2.3 `kubectl debug`

Distroless and minimal images have no shell. Ephemeral containers add a debug container to a **running** Pod without restarting it.

```bash
# Ephemeral container sharing the process namespace of the "api" container
kubectl -n shop-prod debug -it orders-api-7c9d8f6b5d-x2kqp \
  --image=nicolaka/netshoot --target=api -- bash
#   ps aux                       -> see the node process
#   ls /proc/1/root/app          -> the app container's filesystem
#   dig orders-db.shop-prod.svc  -> DNS
#   curl -sv orders-api:80/ready

# Copy the pod with changes (e.g. to debug a CrashLoop with a different command)
kubectl -n shop-prod debug orders-api-7c9d8f6b5d-x2kqp -it \
  --copy-to=orders-api-debug --container=api -- sh

# Node debugging: a pod on the node, host filesystem at /host
kubectl debug node/ip-10-0-3-17.eu-west-1.compute.internal -it \
  --image=busybox:1.37 --profile=sysadmin
#   chroot /host
#   crictl ps ; crictl logs <id> ; journalctl -u kubelet --since "10 min ago"
#   ls /var/log/pods/shop-prod_*

kubectl -n shop-prod delete pod orders-api-debug        # clean up copies; node debug pods too
```

`--profile` (`general`, `baseline`, `restricted`, `netadmin`, `sysadmin`) sets the security context of the debug container. Pod Security Admission `restricted` namespaces may reject privileged profiles; that is intended.

### 2.4 Resources and permissions

```bash
kubectl top nodes
kubectl -n shop-prod top pods -l app=orders-api --containers --sort-by=memory   # needs metrics-server

kubectl auth can-i get pods/log -n shop-prod                               # me
kubectl auth can-i create pods/exec -n shop-prod --as=jane@acme.dev
kubectl auth can-i list secrets -n shop-prod --as=system:serviceaccount:shop-prod:orders-api
kubectl auth can-i --list -n shop-prod --as=system:serviceaccount:argocd:argocd-application-controller
kubectl auth whoami
```

`kubectl top` is a point-in-time sample; use Grafana for trends (queries in [Grafana & the LGTM Stack](/Software%20Engineering/DevOps/Delivery%20Platform/Grafana%20%26%20the%20LGTM%20Stack)).

### 2.5 Triage decision tree

{/* diagram source: /diagrams/diagram-158.mmd */}
![diagram](/diagrams/diagram-158.svg)

### 2.6 Symptom playbook

**Pending**
- Symptom: `0/6 nodes are available: 3 Insufficient cpu, 3 node(s) had untolerated taint {dedicated: batch}`.
- Commands: `kubectl -n shop-prod describe pod <p>` (read the `FailedScheduling` event), `kubectl get nodes -L topology.kubernetes.io/zone,karpenter.sh/nodepool`, `kubectl describe node <n>` (Allocated resources), `kubectl get pvc -n shop-prod` (unbound PVCs also cause Pending).
- Fixes: lower or right-size requests; let the autoscaler add nodes (check its logs if it doesn't); fix nodeSelector/affinity typos; add the missing toleration; relax `topologySpreadConstraints` from `DoNotSchedule` if a zone has no capacity; fix the StorageClass/zone of the PVC.

**ContainerCreating (stuck)**
- Commands: `kubectl -n shop-prod events --for pod/<p>`.
- Typical events and fixes: `FailedMount ... secret "orders-db" not found` (create it, or check External Secrets sync); `MountVolume.SetUp failed ... volume is already exclusively attached` (RWO volume still attached to the old node: wait, or fix the stuck VolumeAttachment); `failed to setup network for sandbox` (CNI issue, IP exhaustion in the subnet); a large image still pulling.

**ImagePullBackOff / ErrImagePull**
- Commands: `kubectl -n shop-prod describe pod <p> | grep -A5 Events`, `kubectl -n shop-prod get pod <p> -o jsonpath='{.spec.containers[*].image}'`, `kubectl -n shop-prod get sa orders-api -o yaml` (imagePullSecrets).
- Causes and fixes: wrong tag (the GitOps commit points to a SHA that CI never pushed: check Jenkins, see [Jenkins - Pipelines as Code](/Software%20Engineering/DevOps/Delivery%20Platform/Jenkins%20-%20Pipelines%20as%20Code)); `ghcr.io` credentials missing or expired (`kubectl create secret docker-registry ghcr-pull ...` and attach to the ServiceAccount); `manifest unknown` for the node's architecture (build multi-arch); registry rate limits.

**CrashLoopBackOff**
- The container starts, exits, and the kubelet restarts it with an exponential back-off (10 s doubling up to 5 min).
- Commands: `kubectl -n shop-prod logs <p> --previous`, the jsonpath above for `lastState.terminated.reason/exitCode`, `kubectl -n shop-prod debug <p> --copy-to=dbg -it --container=api -- sh` to start it by hand.
- Exit codes: `1` app error (bad config, DB unreachable at boot, missing env var); `137` SIGKILL (OOM or liveness kill); `143` SIGTERM not handled in time; `127` command not found; `0` process finished (a server that doesn't block, or a Job-like image in a Deployment).
- Fixes: fix config/Secret; make startup tolerant of dependencies (retry instead of exit); add a `startupProbe` if liveness kills a slow boot.

**OOMKilled**
- Symptom: `Last State: Terminated, Reason: OOMKilled, Exit Code: 137`; also visible as `kube_pod_container_status_last_terminated_reason{reason="OOMKilled"}`.
- Commands: `kubectl -n shop-prod top pod <p> --containers`, the Grafana "memory working set vs limit" panel, `kubectl -n shop-prod get pod <p> -o jsonpath='{.spec.containers[0].resources}'`.
- Fixes: raise the memory limit (and request) to fit real peak; for Node, align `--max-old-space-size` to about 75% of the container limit so V8 GCs before the kernel kills it; find leaks with heap snapshots ([Node.js Performance & Debugging](/Software%20Engineering/Backend/The%20Hard%20Parts%20of%20Servers%20%26%20Node.js/Node.js%20Performance%20%26%20Debugging)). Since v1.35, **in-place resize** can change memory on a running Pod, but in GitOps the permanent fix is still a commit.

**Readiness failing (Running, 0/1 READY)**
- Symptom: events `Readiness probe failed: HTTP probe failed with statuscode: 503`; the Pod gets no traffic, rollout stalls (`Waiting for deployment "orders-api" rollout to finish: 1 of 3 updated replicas are available`).
- Commands: `kubectl -n shop-prod describe pod <p>`, `kubectl -n shop-prod exec <p> -- wget -qO- localhost:8080/ready` or port-forward + curl, check the probe's `port` name matches the container port.
- Fixes: wrong path/port; app listening on `127.0.0.1` instead of `0.0.0.0`; readiness checking a dependency that is down (decide whether that should really remove all pods); timeouts too short under load.

**Evicted**
- Symptom: `Status: Failed, Reason: Evicted, Message: The node was low on resource: memory` (or `ephemeral-storage`).
- Commands: `kubectl describe node <n>` (Conditions: MemoryPressure/DiskPressure), `kubectl get pods -A --field-selector=status.phase=Failed`.
- Fixes: set requests so BestEffort/Burstable pods aren't over-packed; limit `ephemeral-storage` for apps writing temp files; stop logging to files; clean evicted pods: `kubectl -n shop-prod delete pods --field-selector=status.phase=Failed`.

**Terminating (stuck)**
- Commands: `kubectl -n shop-prod get pod <p> -o jsonpath='{.metadata.finalizers} {.metadata.deletionTimestamp}'`, `kubectl get node <n>` (NotReady?).
- Causes and fixes: the node is gone, so the kubelet can't confirm (Pods are cleaned up after the node object is removed; for StatefulSets check first that the node is really dead, then `kubectl delete pod <p> --grace-period=0 --force`); a finalizer whose controller is not running (see section 4); a `preStop` hook or app ignoring SIGTERM until `terminationGracePeriodSeconds`. Force deletion only removes the API object: the container may still run on a partitioned node.

**Service with no endpoints**
- Symptom: `503`/`no healthy upstream` from the Gateway, `curl orders-api.shop-prod.svc` refuses.
- Commands:
  ```bash
  kubectl -n shop-prod get endpointslices -l kubernetes.io/service-name=orders-api -o wide
  kubectl -n shop-prod get svc orders-api -o jsonpath='{.spec.selector}{"\n"}{.spec.ports}{"\n"}'
  kubectl -n shop-prod get pods -l app=orders-api --show-labels
  ```
- Fixes: selector doesn't match Pod labels (a Kustomize `commonLabels` change is a classic); `targetPort` name/number wrong; all Pods unready (the slice lists them with `ready: false`); for the Gateway path, check `HTTPRoute` status conditions ([Kubernetes Networking - Services, Ingress & Gateway API](/Software%20Engineering/DevOps/Kubernetes/Kubernetes%20Networking%20-%20Services%2C%20Ingress%20%26%20Gateway%20API)). Use **EndpointSlices**, not `kubectl get endpoints`: the v1 Endpoints API is deprecated since v1.33.

## 3. Pod orchestration and scheduling

### 3.1 How the scheduler picks a node

The scheduler watches for Pods with no `spec.nodeName`, runs each through a plugin pipeline, and writes a **binding**. The kubelet on that node then does the rest.

{/* diagram source: /diagrams/diagram-159.mmd */}
![diagram](/diagrams/diagram-159.svg)

- **Filtering** (hard rules): enough allocatable CPU/memory for the **requests** (`NodeResourcesFit`), nodeSelector and required node affinity, taints without tolerations, required pod (anti-)affinity, `DoNotSchedule` topology spread, volume zone and attach limits, host ports.
- **Scoring** (preferences): balanced/least-allocated resources, preferred affinities, `ScheduleAnyway` spread, image already on the node, fewer intolerable `PreferNoSchedule` taints.
- The scheduler only looks at **requests**, never actual usage. A node full of Pods using 5% of their requests is "full".

### 3.2 Pod lifecycle

{/* diagram source: /diagrams/diagram-160.mmd */}
![diagram](/diagrams/diagram-160.svg)

`Terminating`, `CrashLoopBackOff`, `ContainerCreating` and `ImagePullBackOff` are not phases: they are what kubectl shows from deletion timestamps and **container states** (`waiting.reason`). Pod **conditions** (`PodScheduled`, `Initialized`, `ContainersReady`, `Ready`) tell you which step is stuck.

### 3.3 Placement controls for `orders-api`

```yaml
# platform-gitops/apps/orders-api/overlays/prod/placement-patch.yaml
apiVersion: apps/v1
kind: Deployment
metadata: { name: orders-api }
spec:
  template:
    spec:
      priorityClassName: shop-critical
      nodeSelector:
        kubernetes.io/os: linux
      affinity:
        nodeAffinity:
          requiredDuringSchedulingIgnoredDuringExecution:
            nodeSelectorTerms:
              - matchExpressions:
                  - { key: kubernetes.io/arch, operator: In, values: [amd64, arm64] }   # multi-arch image
                  - { key: karpenter.sh/capacity-type, operator: In, values: [on-demand] }
          preferredDuringSchedulingIgnoredDuringExecution:
            - weight: 50
              preference:
                matchExpressions:
                  - { key: node.acme.dev/pool, operator: In, values: [general] }
        podAntiAffinity:                        # prefer not two replicas on one node
          preferredDuringSchedulingIgnoredDuringExecution:
            - weight: 100
              podAffinityTerm:
                topologyKey: kubernetes.io/hostname
                labelSelector: { matchLabels: { app: orders-api } }
        podAffinity:                            # prefer being in the same zone as its pgbouncer
          preferredDuringSchedulingIgnoredDuringExecution:
            - weight: 20
              podAffinityTerm:
                topologyKey: topology.kubernetes.io/zone
                labelSelector: { matchLabels: { app: orders-pgbouncer } }
      topologySpreadConstraints:
        - maxSkew: 1                            # zones may differ by at most 1 replica
          topologyKey: topology.kubernetes.io/zone
          whenUnsatisfiable: DoNotSchedule      # hard: survive a zone outage
          labelSelector: { matchLabels: { app: orders-api } }
          matchLabelKeys: [pod-template-hash]   # compute skew per ReplicaSet, not across old+new during rollout
          minDomains: 3
        - maxSkew: 1
          topologyKey: kubernetes.io/hostname
          whenUnsatisfiable: ScheduleAnyway     # soft: spread over nodes when possible
          labelSelector: { matchLabels: { app: orders-api } }
      tolerations:
        - key: node.acme.dev/dedicated          # allowed onto nodes tainted for shop workloads
          operator: Equal
          value: shop
          effect: NoSchedule
```

| Control | Hard or soft | Use for |
|---|---|---|
| `nodeSelector` | hard | simple label match (OS, pool) |
| Node affinity | required (hard) / preferred (soft) | expressions: `In`, `NotIn`, `Exists`, `Gt` |
| Pod affinity / anti-affinity | required / preferred | co-locate with or keep away from other Pods, per topology key |
| `topologySpreadConstraints` | `DoNotSchedule` / `ScheduleAnyway` | even spread over zones/nodes (cheaper and more precise than anti-affinity at scale) |
| Taints + tolerations | taint repels, toleration permits | dedicated nodes (GPU, batch), node problems (`node.kubernetes.io/not-ready:NoExecute`) |
| PriorityClass | ordering + preemption | critical services win capacity over batch |

"IgnoredDuringExecution" means rules are checked only at scheduling; changing node labels later doesn't move running Pods (that is the descheduler's job). Taints with `NoExecute` are the exception: they evict running Pods without a matching toleration.

```bash
kubectl taint nodes ip-10-0-9-4.eu-west-1.compute.internal node.acme.dev/dedicated=shop:NoSchedule
kubectl taint nodes ip-10-0-9-4.eu-west-1.compute.internal node.acme.dev/dedicated=shop:NoSchedule-   # remove
kubectl label nodes ip-10-0-9-4.eu-west-1.compute.internal node.acme.dev/pool=general
```

**PriorityClasses and preemption**:

```yaml
apiVersion: scheduling.k8s.io/v1
kind: PriorityClass
metadata: { name: shop-critical }
value: 100000
globalDefault: false
preemptionPolicy: PreemptLowerPriority
description: "Customer-facing Acme Shop services"
---
apiVersion: scheduling.k8s.io/v1
kind: PriorityClass
metadata: { name: batch-low }
value: 1000
preemptionPolicy: Never          # waits instead of evicting others
description: "Reports, backfills, CronJobs"
```

When a `shop-critical` Pod can't fit, PostFilter looks for a node where evicting lower-priority Pods makes room, respecting PDBs where it can (best effort). Keep the number of classes small; `system-cluster-critical` and `system-node-critical` are reserved for cluster add-ons.

### 3.4 Requests, limits and bin packing

Requests are the scheduler's currency; limits are runtime enforcement (details and QoS in [Kubernetes Workloads in Production](/Software%20Engineering/DevOps/Kubernetes/Kubernetes%20Workloads%20in%20Production)). Operational consequences:

- **Over-requesting** wastes nodes (low utilisation, high bill). **Under-requesting** packs too tightly: CPU contention, memory evictions, OOM kills.
- The default scoring spreads load (`LeastAllocated`). A cost-focused cluster can configure `NodeResourcesFit` with `MostAllocated` to **bin-pack** so the autoscaler can remove empty nodes.
- Right-size from data: p95 CPU and max memory working set per container over 2 weeks (Grafana), or VPA in recommendation mode.
- **Descheduler** (kubernetes-sigs/descheduler) evicts Pods that are now badly placed so the scheduler can place them again: `RemoveDuplicates`, `LowNodeUtilization`, `RemovePodsViolatingTopologySpreadConstraint`, `RemovePodsViolatingNodeAffinity`. It respects PDBs; run it as a CronJob or Deployment.

```yaml
apiVersion: "descheduler/v1alpha2"
kind: "DeschedulerPolicy"
profiles:
  - name: acme
    pluginConfig:
      - name: DefaultEvictor
        args: { evictLocalStoragePods: false, nodeFit: true }
      - name: RemovePodsViolatingTopologySpreadConstraint
        args: { constraints: [DoNotSchedule, ScheduleAnyway] }
      - name: LowNodeUtilization
        args:
          thresholds: { cpu: 20, memory: 20, pods: 20 }
          targetThresholds: { cpu: 60, memory: 60, pods: 60 }
    plugins:
      balance: { enabled: [RemovePodsViolatingTopologySpreadConstraint, LowNodeUtilization] }
```

**Cluster Autoscaler vs Karpenter**

| | Cluster Autoscaler | Karpenter |
|---|---|---|
| Model | scales pre-defined node groups (ASGs, MIGs) up/down | provisions individual nodes directly from pending Pods' needs |
| Instance choice | fixed per node group | picks from many instance types per `NodePool` |
| Consolidation | scale down empty/underused nodes | active consolidation: replaces nodes with cheaper/fewer ones |
| Clouds | most clouds | AWS (core), Azure (AKS NAP), others via providers |
| Operational feel | predictable, slower | fast, cost-efficient, needs PDBs and disruption budgets tuned |

```yaml
apiVersion: karpenter.sh/v1
kind: NodePool
metadata: { name: general }
spec:
  template:
    metadata:
      labels: { node.acme.dev/pool: general }
    spec:
      nodeClassRef: { group: karpenter.k8s.aws, kind: EC2NodeClass, name: default }
      requirements:
        - { key: kubernetes.io/arch, operator: In, values: [amd64, arm64] }
        - { key: karpenter.sh/capacity-type, operator: In, values: [on-demand, spot] }
        - { key: karpenter.k8s.aws/instance-category, operator: In, values: [c, m, r] }
      expireAfter: 720h
  disruption:
    consolidationPolicy: WhenEmptyOrUnderutilized
    consolidateAfter: 5m
    budgets:
      - nodes: "10%"
  limits: { cpu: "200", memory: 800Gi }
```

Both only react to **Pending Pods** (based on requests). If requests are wrong, autoscaling is wrong.

### 3.5 Controllers: which one for which job

| Controller | Guarantees | Acme Shop example |
|---|---|---|
| **Deployment → ReplicaSet** | N identical, interchangeable Pods; rolling updates; history | `shell`, `catalog`, `checkout`, `orders-api` |
| **StatefulSet** | stable names (`pg-0`), stable per-Pod PVCs, ordered start/stop | Postgres (usually via an operator such as CloudNativePG), Kafka |
| **DaemonSet** | one Pod per (matching) node | Alloy, node-exporter, CNI agents |
| **Job** | run to completion, retries (`backoffLimit`), parallelism | DB migration before a release |
| **CronJob** | Jobs on a schedule | nightly abandoned-cart cleanup |

```yaml
apiVersion: batch/v1
kind: CronJob
metadata: { name: orders-cleanup, namespace: shop-prod }
spec:
  schedule: "30 2 * * *"
  timeZone: Europe/Berlin
  concurrencyPolicy: Forbid           # don't start if last run still going
  startingDeadlineSeconds: 600
  successfulJobsHistoryLimit: 3
  failedJobsHistoryLimit: 5
  jobTemplate:
    spec:
      backoffLimit: 2
      activeDeadlineSeconds: 1800
      ttlSecondsAfterFinished: 86400  # garbage-collect finished Jobs and their Pods
      template:
        spec:
          priorityClassName: batch-low
          restartPolicy: Never
          containers:
            - name: cleanup
              image: ghcr.io/acme/orders-api:3f9c2e1     # same image, different entrypoint
              command: ["node", "dist/jobs/cleanup-abandoned-carts.js"]
              resources:
                requests: { cpu: 100m, memory: 128Mi }
                limits: { memory: 256Mi }
```

```bash
kubectl -n shop-prod create job --from=cronjob/orders-cleanup orders-cleanup-manual-1   # run now
kubectl -n shop-prod get jobs,pods -l job-name=orders-cleanup-manual-1
```

### 3.6 Rollouts and node maintenance

```bash
kubectl -n shop-prod rollout status deployment/orders-api --timeout=5m     # CI gate; non-zero exit on failure
kubectl -n shop-prod rollout history deployment/orders-api
kubectl -n shop-prod rollout history deployment/orders-api --revision=42  # pod template of rev 42
kubectl -n shop-prod rollout undo deployment/orders-api --to-revision=41  # see GitOps caveat below
kubectl -n shop-prod rollout restart deployment/orders-api                # new pods, same spec (reload secrets)
kubectl -n shop-prod rollout pause deployment/orders-api                  # batch several changes...
kubectl -n shop-prod rollout resume deployment/orders-api                 # ...into one rollout
```

`maxSurge` / `maxUnavailable` (on 3 replicas):

| Setting | Behaviour | Trade-off |
|---|---|---|
| `maxSurge: 25%, maxUnavailable: 0` | adds 1 new Pod, waits for Ready, removes 1 old | never below capacity; needs spare room |
| `maxSurge: 0, maxUnavailable: 1` | removes 1 old first | no extra capacity needed; runs at 2/3 during rollout |
| `maxSurge: 100%, maxUnavailable: 0` | full new set, then drop old | fastest, doubles resources briefly |

A rollout that never gets Ready stalls; `progressDeadlineSeconds` (default 600) marks it `Progressing=False` (`ProgressDeadlineExceeded`) but does **not** roll back by itself. Canary and automated rollback are Argo Rollouts' job ([Argo Rollouts - Canary & Blue-Green](/Software%20Engineering/DevOps/Delivery%20Platform/Argo%20Rollouts%20-%20Canary%20%26%20Blue-Green)).

**Draining a node** (kernel patch, node upgrade):

```bash
kubectl cordon ip-10-0-3-17.eu-west-1.compute.internal                     # no new pods
kubectl drain ip-10-0-3-17.eu-west-1.compute.internal \
  --ignore-daemonsets --delete-emptydir-data --timeout=10m                  # evicts via the Eviction API
kubectl get pdb -A                                                          # ALLOWED DISRUPTIONS = 0 blocks the drain
kubectl uncordon ip-10-0-3-17.eu-west-1.compute.internal
```

`drain` uses the **Eviction API**, which honours PodDisruptionBudgets: with `orders-api` at 3 replicas and `minAvailable: 2`, pods are evicted one at a time, each waiting for its replacement to be Ready elsewhere. A drain that hangs forever usually means a PDB that can never be satisfied (1 replica + `minAvailable: 1`), or a Pod with no controller.

## 4. Syncing and desired state

### 4.1 The reconciliation loop

Every controller runs the same loop: **watch** objects through informers, **compare** desired (`spec`) with observed (`status` and the real world), **act** to close the gap, **update status**, repeat. It is level-triggered: it reacts to the current state, not to a missed event, which is why Kubernetes self-heals after crashes.

{/* diagram source: /diagrams/diagram-161.mmd */}
![diagram](/diagrams/diagram-161.svg)

### 4.2 Owner references and garbage collection

Each ReplicaSet has an `ownerReference` to its Deployment; each Pod to its ReplicaSet. The **garbage collector** deletes dependents whose owner is gone.

```bash
kubectl -n shop-prod get pod orders-api-7c9d8f6b5d-x2kqp -o jsonpath='{.metadata.ownerReferences[*].kind}/{.metadata.ownerReferences[*].name}{"\n"}'
# ReplicaSet/orders-api-7c9d8f6b5d

kubectl -n shop-prod delete deployment orders-api                          # background: owner goes, GC removes the rest
kubectl -n shop-prod delete deployment orders-api --cascade=foreground     # dependents first, then owner
kubectl -n shop-prod delete deployment orders-api --cascade=orphan         # keep ReplicaSets/Pods running
```

### 4.3 Finalizers

A **finalizer** is a key in `metadata.finalizers` that blocks deletion until a controller has done its cleanup (release a cloud load balancer, delete a PVC's volume, remove Argo CD-managed resources). Deletion sets `deletionTimestamp`; the object disappears only when the list is empty.

```bash
kubectl get namespace shop-staging -o jsonpath='{.spec.finalizers}{" "}{.status.conditions[*].message}{"\n"}'
kubectl -n argocd get application orders-api-staging -o jsonpath='{.metadata.finalizers}'
# ["resources-finalizer.argocd.argoproj.io"]  -> Argo CD deletes the app's resources first
```

Stuck `Terminating` namespace: find which resources remain (`kubectl api-resources --verbs=list --namespaced -o name | xargs -n1 kubectl get -n shop-staging --ignore-not-found --show-kind`), usually CRs whose controller was uninstalled first, or an unavailable aggregated API (`kubectl get apiservices | grep False`). Fix the controller or API. Removing a finalizer by hand (`kubectl patch ... --type=merge -p '{"metadata":{"finalizers":null}}'`) is a last resort: it skips the cleanup and can leak cloud resources.

### 4.4 GitOps: Argo CD is one more reconciler

Argo CD applies the loop one level up: **desired state = Git**, live state = cluster. With `automated: { prune: true, selfHeal: true }`:

- **`kubectl edit` / `kubectl set image` / `kubectl scale` in `shop-*` namespaces get reverted** within seconds to minutes (selfHeal re-applies Git for fields Git manages). Worse, your fix is lost silently. Change the overlay in `platform-gitops` and let Argo CD sync.
- **`kubectl rollout undo` fights Git** the same way. Roll back with `git revert` of the tag bump (or abort the Argo Rollouts canary); Argo CD's own rollback also requires disabling auto-sync.
- **HPA vs `replicas` in Git**: omit `spec.replicas` from the Deployment in Git (or `ignoreDifferences` on `/spec/replicas`) so Argo CD doesn't reset the autoscaler.
- **Allowed imperative actions**: read-only commands, `logs`, `exec`/`debug` for investigation, `rollout restart` (Argo CD also exposes a Restart action; the annotation it adds is not a Git-managed field), `cordon`/`drain` (nodes are not in Git). For an emergency manual change, **disable auto-sync on that Application first**, then write the change back to Git.

Details: [Argo CD in Practice](/Software%20Engineering/DevOps/Delivery%20Platform/Argo%20CD%20in%20Practice) and the overview in [GitOps - Argo CD & Flux](/Software%20Engineering/DevOps/GitOps%20-%20Argo%20CD%20%26%20Flux).

## 5. What changed recently

<Info>
**🆕 2025–2026 for operators**

- **`kubectl events`** has been GA since v1.28 (alpha 1.23, beta 1.26): prefer it over `kubectl get events` for sorting, `--for` and `--watch`.
- **v1.33 (2025-04-23)**: **native sidecars GA** (init container with `restartPolicy: Always`: log with `-c <sidecar>`, they start before and stop after the app); **v1 Endpoints API deprecated** in favour of EndpointSlices; **kuberc** introduced as alpha.
- **v1.34 (2025-08-27)**: **kuberc beta**: `~/.kube/kuberc` (`kind: Preference`, `apiVersion: kubectl.config.k8s.io/v1beta1`) holds kubectl aliases and default flags separately from kubeconfig; **Dynamic Resource Allocation (DRA) GA** for GPUs and other devices (ResourceClaims instead of opaque extended resources); `preStop` sleep action GA.
- **v1.35 (2025-12-17)**: **in-place Pod resize GA**: change CPU/memory of a running Pod through the `resize` subresource, often without a restart; kuberc gains an allowlist for credential exec plugins (blog 2026-01-09); **gang scheduling** arrives as alpha through the new **Workload API** (`scheduling.k8s.io/v1alpha1`, PodGroups with `minCount`), feature gates `GenericWorkload` and `GangScheduling`.
- **v1.37 (2026-08-26)**: scheduler **preemption to make room for in-place resize** (alpha); HPA scale-to-zero beta (see [Kubernetes Workloads in Production](/Software%20Engineering/DevOps/Kubernetes/Kubernetes%20Workloads%20in%20Production)).
</Info>

kuberc example:

```yaml
# ~/.kube/kuberc
apiVersion: kubectl.config.k8s.io/v1beta1
kind: Preference
defaults:
  - command: delete
    options:
      - { name: interactive, default: "true" }      # confirm before deleting
  - command: apply
    options:
      - { name: server-side, default: "true" }
aliases:
  - name: shoplogs
    command: logs
    options:
      - { name: namespace, default: shop-prod }
      - { name: prefix, default: "true" }
```

In-place resize (emergency headroom while the Git change goes through):

```bash
kubectl -n shop-prod patch pod orders-api-7c9d8f6b5d-x2kqp --subresource resize --type=merge \
  -p '{"spec":{"containers":[{"name":"api","resources":{"requests":{"memory":"768Mi"},"limits":{"memory":"768Mi"}}}]}}'
kubectl -n shop-prod get pod orders-api-7c9d8f6b5d-x2kqp -o jsonpath='{.status.containerStatuses[0].resources}'
```

The resize applies to that Pod only; the next rollout uses the Deployment template again, so commit the new values to Git too.

## Pitfalls

<Warning>
**Common pitfalls**

- Reading `kubectl logs` instead of `--previous` on a crash-looping Pod: you see the new, empty container.
- Relying on `kubectl logs` for incidents: rotation, deleted Pods and dead nodes lose the evidence. Ship to Loki.
- Apps logging to files inside the container: no `kubectl logs`, disk-pressure evictions, broken read-only filesystems.
- `kubectl edit` in a GitOps namespace: reverted by selfHeal, and no one knows why the "fix" vanished.
- Force-deleting StatefulSet Pods on a partitioned node: two Pods with the same identity writing to the same volume.
- Removing finalizers blindly: orphaned load balancers, volumes, DNS records.
- Hard anti-affinity (`required...`) with more replicas than nodes: Pods stuck Pending forever. Use topology spread or preferred rules.
- `topologySpreadConstraints` with `DoNotSchedule` and no `matchLabelKeys: [pod-template-hash]`: rollouts get stuck because old and new ReplicaSets count together.
- PDB with `minAvailable` equal to replicas (or `maxUnavailable: 0`): node drains and cluster upgrades hang.
- Requests copied from a template and never revisited: the autoscaler adds nodes for capacity nobody uses.
- Events expire after about an hour: capture them before they disappear.
</Warning>

## Senior perspective

- **Debug by diffing spec and status.** Every Kubernetes problem is "desired != actual"; find which controller is failing to converge and read its events and logs (kube-controller-manager, scheduler, Argo CD application-controller, Karpenter).
- **Make the fast path read-only.** Give developers `get/list/watch`, `pods/log` and ephemeral-container debug in their namespaces, Grafana for history, and Argo CD for deploys. Write access in prod goes through Git and a break-glass role that is audited.
- **Capacity is a scheduling problem first.** Accurate requests, priority classes, PDBs and topology spread determine whether autoscaling, spot instances and zone failures are survivable.
- **Rehearse failure**: drain a node during business hours in staging, kill a zone's nodes, fill a node's disk. Check that PDBs, spread and alerts behave.
- **Runbooks per symptom** (the playbook in section 2.6) linked from alerts turn a 30-minute investigation into a 3-minute one.
- Review questions: What happens to `orders-api` if one zone disappears? Can we drain any node right now (`kubectl get pdb -A`)? Which Pods would be preempted first? How would an on-call engineer without cluster-admin see the logs from a Pod that died an hour ago?

## Interview questions

<Accordion title="A Pod is stuck in Pending. How do you debug it?">
`kubectl describe pod` and read the FailedScheduling event: it lists why each node was rejected (insufficient CPU/memory, untolerated taint, affinity mismatch, topology spread, unbound PVC). Then check node allocatable vs requested, labels and taints, and whether the cluster autoscaler or Karpenter tried to add a node and why it didn't. Fix requests, selectors, tolerations or capacity.
</Accordion>

<Accordion title="CrashLoopBackOff: what does it mean and what do you run?">
The container keeps exiting and the kubelet restarts it with exponential back-off up to 5 minutes. Run `kubectl logs --previous` for the crashed instance, check `lastState.terminated.reason` and exit code (137 OOM or kill, 1 app error, 0 exits normally), read events for probe failures, and if needed start a copy with `kubectl debug --copy-to` and a shell to reproduce.
</Accordion>

<Accordion title="Explain how the scheduler chooses a node.">
For each unscheduled Pod it runs filter plugins to find feasible nodes (resources by requests, selectors/affinity, taints, spread, volumes), then score plugins to rank them (resource balance, preferred affinity, spread, image locality), then reserves and binds the best node. If no node fits, PostFilter may preempt lower-priority Pods. Actual usage is not considered, only requests.
</Accordion>

<Accordion title="Taints and tolerations vs node affinity?">
Taints are on nodes and repel Pods that don't tolerate them (keep general Pods off GPU nodes). Node affinity is on Pods and attracts them to nodes with certain labels. A toleration only allows a Pod onto a tainted node, it doesn't force it there, so dedicated pools use both.
</Accordion>

<Accordion title="How do you spread replicas across zones, and why topology spread over anti-affinity?">
`topologySpreadConstraints` with `topologyKey: topology.kubernetes.io/zone`, `maxSkew: 1` and `whenUnsatisfiable: DoNotSchedule` (plus `matchLabelKeys: [pod-template-hash]`). It supports more than one replica per zone with an even skew, whereas required anti-affinity allows at most one per domain and is expensive to compute in large clusters.
</Accordion>

<Accordion title="What are finalizers and owner references?">
Owner references link dependents to owners (Pod → ReplicaSet → Deployment) so the garbage collector deletes dependents when owners go, in background or foreground mode, or orphans them if asked. Finalizers are keys that block an object's deletion until a controller finishes cleanup; the object stays Terminating until the list is empty.
</Accordion>

<Accordion title="Why shouldn't you `kubectl edit` a Deployment managed by Argo CD?">
Argo CD continuously compares live state with Git. With selfHeal enabled it re-applies Git and reverts your edit; without it, the app shows OutOfSync and the next sync reverts it anyway. Either way Git and cluster drift. Change the manifest in Git (or disable auto-sync for a documented emergency change and commit it afterwards).
</Accordion>

<Accordion title="How does `kubectl drain` interact with PodDisruptionBudgets?">
Drain cordons the node and evicts Pods through the Eviction API, which refuses evictions that would violate a PDB. Pods are evicted as their budget allows, waiting for replacements to become Ready. A PDB that allows zero disruptions blocks the drain until timeout.
</Accordion>

## Related
- [DevOps Index](/Software%20Engineering/DevOps/DevOps%20Index) · [Kubernetes Delivery Platform - End to End](/Software%20Engineering/DevOps/Delivery%20Platform/Kubernetes%20Delivery%20Platform%20-%20End%20to%20End) · [Kubernetes](/Software%20Engineering/DevOps/Kubernetes/Kubernetes) · [Kubernetes Basics](/Software%20Engineering/DevOps/Kubernetes/Kubernetes%20Basics)
- [Kubernetes Workloads in Production](/Software%20Engineering/DevOps/Kubernetes/Kubernetes%20Workloads%20in%20Production) (probes, limits, HPA, PDB manifests) · [Kubernetes Networking - Services, Ingress & Gateway API](/Software%20Engineering/DevOps/Kubernetes/Kubernetes%20Networking%20-%20Services%2C%20Ingress%20%26%20Gateway%20API)
- [Grafana & the LGTM Stack](/Software%20Engineering/DevOps/Delivery%20Platform/Grafana%20%26%20the%20LGTM%20Stack) (logs and metrics history) · [Observability - Logs, Metrics & Traces](/Software%20Engineering/DevOps/Observability%20-%20Logs%2C%20Metrics%20%26%20Traces)
- [Argo CD in Practice](/Software%20Engineering/DevOps/Delivery%20Platform/Argo%20CD%20in%20Practice) · [Argo Rollouts - Canary & Blue-Green](/Software%20Engineering/DevOps/Delivery%20Platform/Argo%20Rollouts%20-%20Canary%20%26%20Blue-Green) · [GitOps - Argo CD & Flux](/Software%20Engineering/DevOps/GitOps%20-%20Argo%20CD%20%26%20Flux) · [Helm & Kustomize](/Software%20Engineering/DevOps/Kubernetes/Helm%20%26%20Kustomize)
- [Deploying Node.js Apps to Kubernetes](/Software%20Engineering/DevOps/Delivery%20Platform/Deploying%20Node.js%20Apps%20to%20Kubernetes) · [Deploying Next.js Micro-frontends to Kubernetes](/Software%20Engineering/DevOps/Delivery%20Platform/Deploying%20Next.js%20Micro-frontends%20to%20Kubernetes) · [Site Reliability Engineering & Incident Management](/Software%20Engineering/DevOps/Site%20Reliability%20Engineering%20%26%20Incident%20Management)

## Sources
- [kubectl logs reference](https://kubernetes.io/docs/reference/kubectl/generated/kubectl_logs/) · [kubectl events reference](https://kubernetes.io/docs/reference/kubectl/generated/kubectl_events/) · [KEP-1440 kubectl events (stages)](https://github.com/kubernetes/enhancements/issues/1440)
- [Logging architecture (node-level logging, rotation)](https://kubernetes.io/docs/concepts/cluster-administration/logging/)
- [Debug running Pods, ephemeral containers, node debugging](https://kubernetes.io/docs/tasks/debug/debug-application/debug-running-pod/) · [Debugging Kubernetes nodes with kubectl](https://kubernetes.io/docs/tasks/debug/debug-cluster/kubectl-node-debug/)
- [Debug Services](https://kubernetes.io/docs/tasks/debug/debug-application/debug-service/) · [EndpointSlices](https://kubernetes.io/docs/concepts/services-networking/endpoint-slices/) · [Continuing the transition from Endpoints to EndpointSlices (2025-04-24)](https://kubernetes.io/blog/2025/04/24/endpoints-deprecation/)
- [Pod lifecycle](https://kubernetes.io/docs/concepts/workloads/pods/pod-lifecycle/) · [Scheduling framework](https://kubernetes.io/docs/concepts/scheduling-eviction/scheduling-framework/) · [Assigning Pods to nodes](https://kubernetes.io/docs/concepts/scheduling-eviction/assign-pod-node/) · [Topology spread constraints](https://kubernetes.io/docs/concepts/scheduling-eviction/topology-spread-constraints/) · [Taints and tolerations](https://kubernetes.io/docs/concepts/scheduling-eviction/taint-and-toleration/) · [Pod priority and preemption](https://kubernetes.io/docs/concepts/scheduling-eviction/pod-priority-preemption/) · [Resource bin packing](https://kubernetes.io/docs/concepts/scheduling-eviction/resource-bin-packing/)
- [Safely drain a node](https://kubernetes.io/docs/tasks/administer-cluster/safely-drain-node/) · [Deployments (rolling update, progress deadline)](https://kubernetes.io/docs/concepts/workloads/controllers/deployment/) · [CronJob](https://kubernetes.io/docs/concepts/workloads/controllers/cron-jobs/)
- [Garbage collection and owner references](https://kubernetes.io/docs/concepts/architecture/garbage-collection/) · [Finalizers](https://kubernetes.io/docs/concepts/overview/working-with-objects/finalizers/) · [Controllers](https://kubernetes.io/docs/concepts/architecture/controller/)
- [kuberc docs](https://kubernetes.io/docs/reference/kubectl/kuberc/) · [kuberc beta in v1.34 (2025-08-28)](https://kubernetes.io/blog/2025/08/28/kubernetes-v1-34-kubectl-kuberc-beta/) · [kuberc credential plugin allowlist (2026-01-09)](https://kubernetes.io/blog/2026/01/09/kubernetes-v1-35-kuberc-credential-plugin-allowlist/)
- [In-place Pod resize GA (2025-12-19)](https://kubernetes.io/blog/2025/12/19/kubernetes-v1-35-in-place-pod-resize-ga) · [Resize container resources](https://kubernetes.io/docs/tasks/configure-pod-container/resize-container-resources/) · [Workload aware scheduling in v1.35 (2025-12-29)](https://kubernetes.io/blog/2025/12/29/kubernetes-v1-35-introducing-workload-aware-scheduling/)
- Release blogs: [v1.33](https://kubernetes.io/blog/2025/04/23/kubernetes-v1-33-release/) · [v1.34](https://kubernetes.io/blog/2025/08/27/kubernetes-v1-34-release/) · [v1.35](https://kubernetes.io/blog/2025/12/17/kubernetes-v1-35-release/) · [v1.37](https://kubernetes.io/blog/2026/08/26/kubernetes-v1-37-release/)
- [Argo CD 2.14 to 3.0 upgrade notes (logs RBAC)](https://argo-cd.readthedocs.io/en/stable/operator-manual/upgrading/2.14-3.0/) · [Argo CD automated sync and self-heal](https://argo-cd.readthedocs.io/en/stable/user-guide/auto_sync/)
- [stern](https://github.com/stern/stern) · [kubetail](https://github.com/kubetail-org/kubetail) · [Descheduler](https://github.com/kubernetes-sigs/descheduler) · [Karpenter docs](https://karpenter.sh/docs/) · [Cluster Autoscaler](https://github.com/kubernetes/autoscaler/tree/master/cluster-autoscaler)
