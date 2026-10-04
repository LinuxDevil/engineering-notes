---
title: "Kubernetes Basics"
tags: [devops, devops/kubernetes]
aliases: [K8s Basics, Kubernetes Fundamentals, kubectl basics, Pods Deployments Services]
updated: 2026-09-27
---
# Kubernetes Basics

<Note>
**TL;DR**

Kubernetes (K8s) runs containers across a cluster of machines and keeps them in the **desired state** you declare in YAML. You describe objects (**Pods**, **Deployments**, **Services**, **Ingress/Gateway**) and apply them with `kubectl apply -f`; controllers continuously reconcile reality toward that spec (restart crashed Pods, roll out new images). These notes follow Stephen Grider's *Microservices with Node JS and React* course (a blog app with `posts`, `comments`, `query`, `moderation`, `event-bus` and a React `client`). Production topics are in [Kubernetes Workloads in Production](/Software%20Engineering/DevOps/Kubernetes/Kubernetes%20Workloads%20in%20Production), networking in [Kubernetes Networking - Services, Ingress & Gateway API](/Software%20Engineering/DevOps/Kubernetes/Kubernetes%20Networking%20-%20Services%2C%20Ingress%20%26%20Gateway%20API), packaging in [Helm & Kustomize](/Software%20Engineering/DevOps/Kubernetes/Helm%20%26%20Kustomize).
</Note>

## 1. What Kubernetes is

Kubernetes is a system for running many different containers. We give it configuration describing how our containers should run and interact, and it makes it happen across one or more nodes.

{/* diagram source: /diagrams/diagram-172.mmd */}
![diagram](/diagrams/diagram-172.svg)

| Component | Role |
|---|---|
| **kube-apiserver** | The only entry point; every client (including `kubectl` and controllers) talks to it |
| **etcd** | Key-value store holding the cluster state |
| **scheduler** | Picks a node for each new Pod |
| **controller-manager** | Runs control loops (Deployment, ReplicaSet, Node, Job controllers...) |
| **kubelet** | Agent on each node; starts containers via the container runtime (containerd/CRI-O) and reports status |
| **kube-proxy** | Programs Service routing on each node (iptables/nftables) |

We interact with a cluster using:

```bash
kubectl
```

Locally you can use Docker Desktop's built-in Kubernetes, kind, minikube or k3d.

## 2. Pods

A **Pod** is the smallest deployable unit in Kubernetes. It is a *logical host* for one or more containers that are always co-located and co-scheduled on the same node, share the same network namespace (they reach each other on `localhost`) and can share volumes. In practice a Pod usually runs **one main container**, sometimes with helper **sidecar** containers.

![Screenshot_2023-04-12_at_3.20.09_AM](/Attachments/Screenshot_2023-04-12_at_3.20.09_AM.png)

![Screenshot_2023-04-12_at_3.26.48_AM](/Attachments/Screenshot_2023-04-12_at_3.26.48_AM.png)

### Config files

We create config files (YAML) to create **objects**. Keep them in version control; they are the documentation of what runs.

![Screenshot_2023-04-12_at_11.56.36_PM](/Attachments/Screenshot_2023-04-12_at_11.56.36_PM.png)

`posts.yaml`:

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: posts
spec:
  containers:
    - name: posts
      image: aligmohammad/posts:0.0.1
```

![Screenshot_2023-04-13_at_1.12.21_AM](/Attachments/Screenshot_2023-04-13_at_1.12.21_AM.png)

```bash
kubectl apply -f posts.yaml
```

`-f` points `kubectl apply` at a file (or directory, or URL) containing manifests in YAML or JSON.

![Screenshot_2023-04-13_at_1.20.27_AM](/Attachments/Screenshot_2023-04-13_at_1.20.27_AM.png)

![Screenshot_2023-04-13_at_1.10.22_AM](/Attachments/Screenshot_2023-04-13_at_1.10.22_AM.png)

```bash
kubectl get pods
```

![Screenshot_2023-04-13_at_1.11.17_AM](/Attachments/Screenshot_2023-04-13_at_1.11.17_AM.png)

### Common kubectl commands

![Screenshot_2023-04-13_at_1.17.46_AM](/Attachments/Screenshot_2023-04-13_at_1.17.46_AM.png)

| Docker world | Kubernetes world |
|---|---|
| `docker ps` | `kubectl get pods` (`-o wide`, `-A` for all namespaces) |
| `docker exec -it <c> sh` | `kubectl exec -it <pod> -- sh` |
| `docker logs <c>` | `kubectl logs <pod>` (`-f`, `--previous`, `-c <container>`) |
| | `kubectl delete pod <pod>` |
| | `kubectl apply -f <file>` |
| | `kubectl describe pod <pod>` (events: why is it Pending/CrashLoopBackOff?) |
| | `kubectl get events --sort-by=.lastTimestamp` |
| | `kubectl port-forward pod/<pod> 8080:4000` |

Reference: https://kubernetes.io/docs/reference/kubectl/

## 3. Deployments

People rarely create bare Pods. A bare Pod that dies (or whose node dies) is gone. Instead, create a **Deployment**. A Deployment does not run code itself; it manages Pods (through a ReplicaSet).

![Screenshot_2023-04-13_at_1.30.54_AM](/Attachments/Screenshot_2023-04-13_at_1.30.54_AM.png)

It has two main jobs:
- If a Pod crashes or is deleted, recreate it so the replica count matches.
- Versioning and updates: roll out a new image gradually (rolling update) and roll back if needed.

![Screenshot_2023-04-13_at_1.31.52_AM](/Attachments/Screenshot_2023-04-13_at_1.31.52_AM.png)

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: posts-depl
spec:
  replicas: 1
  selector:
    matchLabels:
      app: posts
  template:
    metadata:
      labels:
        app: posts
    spec:
      containers:
        - name: posts
          image: aligmohammad/posts:0.0.1
```

- `apiVersion`: the API group and version for this kind (`apps/v1` for Deployments).
- `kind`: the object type.
- `metadata.name`: the object's name (unique per kind per namespace).
- `spec`: the desired state.
- `replicas`: how many Pods should run.
- `selector.matchLabels`: which Pods this Deployment owns: those labelled `app: posts`.
- `template`: the Pod template used to create new Pods; its `labels` **must match** the selector.
- `template.spec.containers`: the containers in each Pod (here one, `posts`, from image `aligmohammad/posts:0.0.1`).

![Screenshot_2023-04-13_at_1.58.50_AM](/Attachments/Screenshot_2023-04-13_at_1.58.50_AM.png)

### Deployment commands

![Screenshot_2023-04-13_at_2.20.48_AM](/Attachments/Screenshot_2023-04-13_at_2.20.48_AM.png)

![Screenshot_2023-04-13_at_2.21.04_AM](/Attachments/Screenshot_2023-04-13_at_2.21.04_AM.png)

```bash
kubectl get deployments
kubectl describe deployment posts-depl
kubectl rollout status deployment/posts-depl
kubectl rollout history deployment/posts-depl
kubectl rollout undo deployment/posts-depl
kubectl delete deployment posts-depl     # also deletes its ReplicaSets and Pods
```

`READY 1/1` means: 1 Pod is ready out of 1 desired.

![Screenshot_2023-04-13_at_2.22.09_AM](/Attachments/Screenshot_2023-04-13_at_2.22.09_AM.png)

If you delete a Pod managed by a Deployment, a new one is created automatically:

![Screenshot_2023-04-13_at_2.23.14_AM](/Attachments/Screenshot_2023-04-13_at_2.23.14_AM.png)

### Updating a Deployment

**Method 1: change the image tag in the manifest and re-apply.** Not used much in the course because it means editing a version in YAML for each change, but it is actually **the production way** (usually automated by CI or GitOps, see [GitOps - Argo CD & Flux](/Software%20Engineering/DevOps/GitOps%20-%20Argo%20CD%20%26%20Flux)).

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: posts-depl
spec:
  replicas: 1
  selector:
    matchLabels:
      app: posts
  template:
    metadata:
      labels:
        app: posts
    spec:
      containers:
        - name: posts
          image: aligmohammad/posts:0.0.2
```

![Screenshot_2023-04-13_at_2.26.22_AM](/Attachments/Screenshot_2023-04-13_at_2.26.22_AM.png)

![Screenshot_2023-04-13_at_2.29.46_AM](/Attachments/Screenshot_2023-04-13_at_2.29.46_AM.png)

**Method 2 (the course's dev workflow): use `latest` and restart.**

![Screenshot_2023-04-13_at_2.35.15_AM](/Attachments/Screenshot_2023-04-13_at_2.35.15_AM.png)

1. The Deployment uses the `latest` tag (or no tag):

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: posts-depl
spec:
  replicas: 1
  selector:
    matchLabels:
      app: posts
  template:
    metadata:
      labels:
        app: posts
    spec:
      containers:
        - name: posts
          image: aligmohammad/posts
```

2. Build and push the image:

```bash
docker build -t aligmohammad/posts .
docker push aligmohammad/posts
```

3. Restart the Deployment so new Pods pull the new image (`latest` defaults to `imagePullPolicy: Always`):

```bash
kubectl rollout restart deployment posts-depl
```

![Screenshot_2023-04-13_at_2.40.54_AM](/Attachments/Screenshot_2023-04-13_at_2.40.54_AM.png)

<Warning>
**`latest` in production**

Fine for a local course loop. In production it makes rollbacks and audits impossible (you cannot tell which code runs). Deploy immutable tags or digests.
</Warning>

## 4. Services

Pods are ephemeral and get new IPs whenever they are recreated. A **Service** gives a stable virtual IP and DNS name in front of a set of Pods (selected by labels) and load-balances between them. We use Services for Pod-to-Pod communication and for access from outside the cluster.

This is a **ClusterIP** Service (internal):

![Screenshot_2023-04-13_at_2.49.44_AM](/Attachments/Screenshot_2023-04-13_at_2.49.44_AM.png)

This is a **LoadBalancer** or **NodePort** Service (external):

![Screenshot_2023-04-13_at_2.49.52_AM](/Attachments/Screenshot_2023-04-13_at_2.49.52_AM.png)

### Service types

![Screenshot_2023-04-13_at_2.50.20_AM](/Attachments/Screenshot_2023-04-13_at_2.50.20_AM.png)

1. **ClusterIP** (default): a virtual IP reachable **only from inside the cluster**. Other Pods call it by DNS name (`posts-clusterip-srv`, or fully `posts-clusterip-srv.default.svc.cluster.local`). Traffic is load-balanced across matching Pods.

![Screenshot_2023-04-13_at_3.03.15_AM](/Attachments/Screenshot_2023-04-13_at_3.03.15_AM.png)

2. **NodePort**: also gets a ClusterIP, plus opens the same static port (30000–32767) on **every node**; traffic to `<nodeIP>:<nodePort>` is forwarded to the Service. Mostly for development and testing.

![Screenshot_2023-04-13_at_3.04.00_AM](/Attachments/Screenshot_2023-04-13_at_3.04.00_AM.png)

3. **LoadBalancer**: also a NodePort + ClusterIP, and asks the cloud provider (AWS, GCP, Azure...) to provision an external load balancer that forwards to it. The usual way to expose one Service to the internet, or to expose the Ingress/Gateway controller.

![Screenshot_2023-04-13_at_3.03.25_AM](/Attachments/Screenshot_2023-04-13_at_3.03.25_AM.png)

4. **ExternalName**: no proxying, just a DNS CNAME from a cluster name to an external hostname (e.g. `db` → `mydb.example.com`).

### NodePort Service for posts

![Screenshot_2023-04-13_at_3.09.57_AM](/Attachments/Screenshot_2023-04-13_at_3.09.57_AM.png)

```yaml
apiVersion: v1
kind: Service
metadata:
  name: posts-srv
spec:
  type: NodePort
  selector:
    app: posts
  ports:
    - name: posts
      protocol: TCP
      port: 4000        # the Service's own port
      targetPort: 4000  # the container port traffic is sent to
```

- `type: NodePort`: expose on a static port on each node.
- `selector`: send traffic to Pods labelled `app: posts`.
- `ports.port`: port of the Service (what clients inside the cluster use).
- `ports.targetPort`: port the container listens on.
- `ports.nodePort`: optional; if omitted, Kubernetes picks one from 30000–32767.

```bash
kubectl apply -f posts-srv.yaml
```

![Screenshot_2023-04-13_at_3.16.51_AM](/Attachments/Screenshot_2023-04-13_at_3.16.51_AM.png)

The randomly assigned port (31406 here) is the port we use to reach the Service from our machine.

![Screenshot_2023-04-13_at_3.19.00_AM](/Attachments/Screenshot_2023-04-13_at_3.19.00_AM.png)

![Screenshot_2023-04-13_at_3.20.09_AM](/Attachments/Screenshot_2023-04-13_at_3.20.09_AM.png)

```bash
kubectl get services
kubectl describe service posts-srv
```

![Screenshot_2023-04-13_at_3.18.35_AM](/Attachments/Screenshot_2023-04-13_at_3.18.35_AM.png)

### ClusterIP Services: wiring posts and the event bus

![Screenshot_2023-04-13_at_3.48.22_AM](/Attachments/Screenshot_2023-04-13_at_3.48.22_AM.png)

![Screenshot_2023-04-13_at_3.45.41_AM](/Attachments/Screenshot_2023-04-13_at_3.45.41_AM.png)

The `posts` Pod sends events to `event-bus-srv`, and the event bus sends events back to `posts-clusterip-srv`.

1. Build the event-bus image

![Screenshot_2023-04-13_at_3.53.34_AM](/Attachments/Screenshot_2023-04-13_at_3.53.34_AM.png)

2. Push it

![Screenshot_2023-04-13_at_3.54.04_AM](/Attachments/Screenshot_2023-04-13_at_3.54.04_AM.png)

3. Create the Deployment for the event bus

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: event-bus-depl
spec:
  replicas: 1
  selector:
    matchLabels:
      app: event-bus
  template:
    metadata:
      labels:
        app: event-bus
    spec:
      containers:
        - name: event-bus
          image: aligmohammad/event-bus
```

4. Apply it

![Screenshot_2023-04-13_at_3.56.34_AM](/Attachments/Screenshot_2023-04-13_at_3.56.34_AM.png)

5. Add a ClusterIP Service in the same file, separated by `---`

`event-bus-depl.yaml`:

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: event-bus-depl
spec:
  replicas: 1
  selector:
    matchLabels:
      app: event-bus
  template:
    metadata:
      labels:
        app: event-bus
    spec:
      containers:
        - name: event-bus
          image: aligmohammad/event-bus
---
apiVersion: v1
kind: Service
metadata:
  name: event-bus-srv
spec:
  type: ClusterIP # the default, can be omitted
  selector:
    app: event-bus
  ports:
    - name: event-bus
      protocol: TCP
      port: 4005
      targetPort: 4005
```

![Screenshot_2023-04-13_at_4.04.06_AM](/Attachments/Screenshot_2023-04-13_at_4.04.06_AM.png)

`posts-depl.yaml`:

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: posts-depl
spec:
  replicas: 1
  selector:
    matchLabels:
      app: posts
  template:
    metadata:
      labels:
        app: posts
    spec:
      containers:
        - name: posts
          image: aligmohammad/posts
---
apiVersion: v1
kind: Service
metadata:
  name: posts-clusterip-srv
spec:
  type: ClusterIP
  selector:
    app: posts
  ports:
    - name: posts
      protocol: TCP
      port: 4000
      targetPort: 4000
```

![Screenshot_2023-04-13_at_4.09.59_AM](/Attachments/Screenshot_2023-04-13_at_4.09.59_AM.png)

![Screenshot_2023-04-13_at_4.10.22_AM](/Attachments/Screenshot_2023-04-13_at_4.10.22_AM.png)

6. Wire them up: change the URLs in the code from `localhost:4005` to the Service names.

`posts/index.js` (calls `http://event-bus-srv:4005/events`):

![Screenshot_2023-04-13_at_4.20.19_AM](/Attachments/Screenshot_2023-04-13_at_4.20.19_AM.png)

`event-bus/index.js` (calls `http://posts-clusterip-srv:4000/events`):

![Screenshot_2023-04-13_at_4.21.34_AM](/Attachments/Screenshot_2023-04-13_at_4.21.34_AM.png)

Rebuild, push and restart:

```bash
cd posts && docker build -t aligmohammad/posts . && docker push aligmohammad/posts
cd ../event-bus && docker build -t aligmohammad/event-bus . && docker push aligmohammad/event-bus

kubectl get deployments
kubectl rollout restart deployment posts-depl
kubectl rollout restart deployment event-bus-depl
```

![Screenshot_2023-04-13_at_4.29.11_AM](/Attachments/Screenshot_2023-04-13_at_4.29.11_AM.png)

7. Verify: find the NodePort and send a request.

```bash
kubectl get services
```

![Screenshot_2023-04-13_at_4.31.44_AM](/Attachments/Screenshot_2023-04-13_at_4.31.44_AM.png)

Send a request to port 31406 with Postman:

![Screenshot_2023-04-13_at_4.32.29_AM](/Attachments/Screenshot_2023-04-13_at_4.32.29_AM.png)

```bash
kubectl get pods
kubectl logs event-bus-depl-848d9bd744-ccvpt
kubectl logs posts-depl-9b9559c5c-tfg6f
```

![Screenshot_2023-04-13_at_4.33.52_AM](/Attachments/Screenshot_2023-04-13_at_4.33.52_AM.png)

![Screenshot_2023-04-13_at_4.34.21_AM](/Attachments/Screenshot_2023-04-13_at_4.34.21_AM.png)

Apply every config file in a directory:

```bash
kubectl apply -f .
```

## 5. Exposing the React client: LoadBalancer vs Ingress

The React Pod only **serves** HTML/CSS/JS to the browser. The browser then sends API requests to the other services, so those requests come from **outside** the cluster.

![Screenshot_2023-04-16_at_3.20.57_AM](/Attachments/Screenshot_2023-04-16_at_3.20.57_AM.png)

![Screenshot_2023-04-16_at_3.21.09_AM](/Attachments/Screenshot_2023-04-16_at_3.21.09_AM.png)

How should the browser reach the services?
- **A NodePort per service** (bad): random high ports that can change, one port per service, and every service exposed directly.

![Screenshot_2023-04-16_at_3.22.39_AM](/Attachments/Screenshot_2023-04-16_at_3.22.39_AM.png)

- **One LoadBalancer + a router** (good): a single entry point into the cluster that routes by host and path.

![Screenshot_2023-04-16_at_3.23.48_AM](/Attachments/Screenshot_2023-04-16_at_3.23.48_AM.png)

### LoadBalancer Service

![Screenshot_2023-04-16_at_3.29.32_AM](/Attachments/Screenshot_2023-04-16_at_3.29.32_AM.png)

A **load balancer** distributes incoming traffic across backends to improve availability and scalability. A `type: LoadBalancer` Service asks the cloud provider to provision one and point it at the Service. On its own it forwards to **one** Service; it does not route by path.

![Screenshot_2023-04-16_at_3.33.34_AM](/Attachments/Screenshot_2023-04-16_at_3.33.34_AM.png)

### Ingress and Ingress controllers

- **Ingress**: an API object with HTTP(S) routing rules (host + path → Service), TLS termination and name-based virtual hosting, consolidated in one resource.
- **Ingress controller**: the software (a Pod) that watches Ingress objects and actually routes traffic (NGINX, HAProxy, Traefik, cloud load balancers...). Ingress objects do nothing without a controller.

![Screenshot_2023-04-16_at_3.35.46_AM](/Attachments/Screenshot_2023-04-16_at_3.35.46_AM.png)

The course installs the community **ingress-nginx** controller (optionally via the Krew kubectl plugin manager: https://krew.sigs.k8s.io/docs/user-guide/setup/install/).

![Screenshot_2023-04-16_at_3.44.12_AM](/Attachments/Screenshot_2023-04-16_at_3.44.12_AM.png)

Docs: https://kubernetes.github.io/ingress-nginx/. That controller renders an `nginx.conf` from all Ingress objects and reloads NGINX when it changes.

<Warning>
**🆕 ingress-nginx is retired (2026-03-24)**

Kubernetes SIG Network announced on **2025-11-11** that **ingress-nginx** would get only best-effort maintenance until **March 2026**; it was retired on **2026-03-24** with no further releases or security fixes (existing installs keep working). The recommended path is the **Gateway API** (or another maintained Ingress controller). The commercial F5 NGINX Ingress Controller is a different project and is not affected. The concepts below still apply; see [Kubernetes Networking - Services, Ingress & Gateway API](/Software%20Engineering/DevOps/Kubernetes/Kubernetes%20Networking%20-%20Services%2C%20Ingress%20%26%20Gateway%20API) for the modern equivalent and a migration example.
</Warning>

`ingress-srv.yaml` (modern form: `ingressClassName` replaced the deprecated `kubernetes.io/ingress.class` annotation):

```yaml
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: ingress-srv
spec:
  ingressClassName: nginx
  rules:
    - host: posts.com
      http:
        paths:
          - path: /posts
            pathType: Prefix
            backend:
              service:
                name: posts-clusterip-srv
                port:
                  number: 4000
```

### Testing locally with a fake domain

Check nothing else is using port 80:

```bash
sudo lsof -i tcp:80
```

![Screenshot_2023-04-19_at_3.40.44_AM](/Attachments/Screenshot_2023-04-19_at_3.40.44_AM.png)

One cluster can host many apps and many domains. To make `posts.com` resolve to our machine instead of the real site, edit the hosts file (`/etc/hosts` on macOS/Linux, `C:\Windows\System32\drivers\etc\hosts` on Windows):

![Screenshot_2023-04-19_at_3.42.50_AM](/Attachments/Screenshot_2023-04-19_at_3.42.50_AM.png)

```text
127.0.0.1   localhost
255.255.255.255 broadcasthost
::1         localhost
# Added by Docker Desktop
127.0.0.1   kubernetes.docker.internal
# End of section

127.0.0.1   posts.com
```

Now `posts.com` goes to 127.0.0.1, the Ingress controller sees `Host: posts.com` and applies our routing rules.

![Screenshot_2023-04-19_at_3.46.41_AM](/Attachments/Screenshot_2023-04-19_at_3.46.41_AM.png)

### Running the React client in the cluster

Create React App needs two env vars to run inside Docker (CRA is deprecated since 2025; use Vite for new apps):

```dockerfile
FROM node:alpine

ENV CI=true
ENV WDS_SOCKET_PORT=0

WORKDIR /app
COPY package.json ./
RUN npm install
COPY ./ ./

CMD ["npm", "start"]
```

Update the axios calls to use `http://posts.com/...`, then build and push:

```bash
docker build -t aligmohammad/client .
docker push aligmohammad/client
```

`client-depl.yaml`:

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: client-depl
spec:
  replicas: 1
  selector:
    matchLabels:
      app: client
  template:
    metadata:
      labels:
        app: client
    spec:
      containers:
        - name: client
          image: aligmohammad/client
---
apiVersion: v1
kind: Service
metadata:
  name: client-clusterip-srv
spec:
  type: ClusterIP
  selector:
    app: client
  ports:
    - name: client
      protocol: TCP
      port: 3000
      targetPort: 3000
```

```bash
kubectl apply -f client-depl.yaml
```

### Routing rules for all services

Ingress routes on **host and path only**; it cannot distinguish HTTP methods (GET vs POST). So routes that differ only by method need distinct paths (the course changes `POST /posts` to `POST /posts/create`).

![Screenshot_2023-04-19_at_5.19.05_PM](/Attachments/Screenshot_2023-04-19_at_5.19.05_PM.png)

Steps: update the client routes, rebuild and push the client; update the posts service route, rebuild and push; restart both Deployments:

```bash
kubectl rollout restart deployment client-depl
kubectl rollout restart deployment posts-depl
```

Ingress paths do not understand route params like `:id`, so with ingress-nginx we enable regex matching via the `nginx.ingress.kubernetes.io/use-regex: "true"` annotation (controller-specific, and one reason Gateway API was created: portable matching without vendor annotations). Regex paths use `pathType: ImplementationSpecific`, and order matters: the catch-all goes last.

```yaml
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: ingress-srv
  annotations:
    nginx.ingress.kubernetes.io/use-regex: "true"
spec:
  ingressClassName: nginx
  rules:
    - host: posts.com
      http:
        paths:
          - path: /posts/create
            pathType: ImplementationSpecific
            backend:
              service:
                name: posts-clusterip-srv
                port:
                  number: 4000
          - path: /posts
            pathType: ImplementationSpecific
            backend:
              service:
                name: query-srv
                port:
                  number: 4002
          - path: /posts/?(.*)/comments
            pathType: ImplementationSpecific
            backend:
              service:
                name: comments-srv
                port:
                  number: 4001
          - path: /?(.*)
            pathType: ImplementationSpecific
            backend:
              service:
                name: client-clusterip-srv
                port:
                  number: 3000
```

```bash
kubectl apply -f ingress-srv.yaml
```

![Screenshot_2023-04-20_at_10.06.43_PM](/Attachments/Screenshot_2023-04-20_at_10.06.43_PM.png)

Troubleshooting:
- **503**: the Ingress points at a Service name that does not exist or has no ready endpoints. Check `kubectl get services` and `kubectl get endpointslices`.
- **502**: the backend is failing. Check `kubectl logs <pod>`. In the course it was CORS in the query service, fixed with:

```js
app.use(cors({
  origin: ['http://client-clusterip-srv:3000', 'http://event-bus-srv:4005'],
}));
```

(Browsers send the page's origin, e.g. `http://posts.com`, so in general allow the public origin, not internal service names.)

## 6. Skaffold: automating build, push and restart

Rebuilding, pushing and restarting by hand is slow. **Skaffold** (https://skaffold.dev/) runs **outside** the cluster, watches your code and does it for you.

![Screenshot_2023-04-20_at_10.29.08_PM](/Attachments/Screenshot_2023-04-20_at_10.29.08_PM.png)

`skaffold.yaml` at the project root:

```yaml
apiVersion: skaffold/v4beta4
kind: Config
manifests:
  rawYaml:
    - ./infra/k8s/*.yaml
build:
  local:
    push: false
  artifacts:
    - image: aligmohammad/client
      context: client
      docker:
        dockerfile: Dockerfile
      sync:
        manual:
          - dest: "."
            src: "src/**/*.jsx"
    - image: aligmohammad/comments
      context: comments
      docker:
        dockerfile: Dockerfile
      sync:
        manual:
          - dest: "."
            src: "*.js"
    - image: aligmohammad/event-bus
      context: event-bus
      docker:
        dockerfile: Dockerfile
      sync:
        manual:
          - dest: "."
            src: "*.js"
    - image: aligmohammad/moderation
      context: moderation
      docker:
        dockerfile: Dockerfile
      sync:
        manual:
          - dest: "."
            src: "*.js"
    - image: aligmohammad/posts
      context: posts
      docker:
        dockerfile: Dockerfile
      sync:
        manual:
          - dest: "."
            src: "*.js"
    - image: aligmohammad/query
      context: query
      docker:
        dockerfile: Dockerfile
      sync:
        manual:
          - dest: "."
            src: "*.js"
```

What it does:
1. `skaffold dev` applies all manifests in `infra/k8s`.
2. On stop (Ctrl+C) it deletes the objects it created.
3. On code changes it redeploys automatically:
   1. Files matching a `sync` rule (e.g. `.jsx`, `.js`) are copied straight into the running Pod.
   2. Any other change (e.g. `package.json`) triggers a full image rebuild and redeploy.
   3. `push: false`: images stay local (fine for Docker Desktop/kind), no Docker Hub push needed.

```bash
skaffold schema list   # schema versions supported
skaffold fix           # upgrade an old skaffold.yaml
skaffold dev           # start the dev loop
```

Two levels of restart: Skaffold syncs the file into the Pod, then nodemon / the React dev server inside the container notices and restarts the process.

![Screenshot_2023-04-20_at_11.06.39_PM](/Attachments/Screenshot_2023-04-20_at_11.06.39_PM.png)

Alternatives: Tilt, DevSpace, Telepresence, or `docker compose watch` for non-K8s dev.

## Pitfalls

<Warning>
**Common pitfalls**

- Selector labels not matching template labels: the Deployment is rejected (or a Service finds no Pods, leading to 503s).
- Tabs in YAML: YAML allows spaces only.
- Calling other services on `localhost`: inside a Pod, `localhost` is the Pod itself. Use the Service name.
- Forgetting that Services pick Pods **by label**, not by Deployment name.
- `CrashLoopBackOff`: read `kubectl logs --previous` and `kubectl describe pod` events.
- `ImagePullBackOff`: wrong image name/tag or missing registry credentials (`imagePullSecrets`).
</Warning>

## Senior perspective

- Everything in K8s is a **reconciliation loop**: declare desired state, controllers converge. Debugging means comparing `spec` with `status` and reading events.
- This course setup (single replicas, `latest`, no probes, no resource requests) is a dev setup. Before production you need probes, requests/limits, PodDisruptionBudgets, multiple replicas, securityContext and RBAC: see [Kubernetes Workloads in Production](/Software%20Engineering/DevOps/Kubernetes/Kubernetes%20Workloads%20in%20Production).
- Manifests should live in Git and be applied by a pipeline or GitOps controller, not by `kubectl apply` from laptops.
- Most teams should use managed Kubernetes (EKS, GKE, AKS) rather than running the control plane themselves.

<Info>
**🆕 2025–2026**

- Kubernetes ships three minor releases a year: **v1.33 "Octarine" (2025-04-23)**, **v1.34 (2025-08-27)**, **v1.35 "Timbernetes" (2025-12-17)**, **v1.36 "Haru" (2026-04-22)**, **v1.37 "Garhwal" (2026-08-26)**. Details and what matters for workloads: [Kubernetes Workloads in Production](/Software%20Engineering/DevOps/Kubernetes/Kubernetes%20Workloads%20in%20Production).
- **ingress-nginx retired on 2026-03-24**; migrate to Gateway API ([Kubernetes Networking - Services, Ingress & Gateway API](/Software%20Engineering/DevOps/Kubernetes/Kubernetes%20Networking%20-%20Services%2C%20Ingress%20%26%20Gateway%20API)).
</Info>

## Interview questions

<Accordion title="What is a Pod and why not just run containers directly?">
The smallest schedulable unit: one or more containers sharing a network namespace, IP and volumes, always scheduled together. It lets tightly coupled helpers (sidecars) live with the main container and gives Kubernetes one unit to schedule, probe and restart.
</Accordion>

<Accordion title="Deployment vs ReplicaSet vs Pod?">
A Pod runs containers. A ReplicaSet keeps N identical Pods running. A Deployment manages ReplicaSets to provide declarative rolling updates and rollbacks: a new image creates a new ReplicaSet that is scaled up while the old one is scaled down.
</Accordion>

<Accordion title="Explain the Service types.">
ClusterIP: internal virtual IP + DNS (default). NodePort: ClusterIP plus a port on every node. LoadBalancer: NodePort plus a cloud load balancer. ExternalName: DNS CNAME to an external host. Headless (`clusterIP: None`) returns Pod IPs directly, used by StatefulSets.
</Accordion>

<Accordion title="How does one Pod find another?">
Through a Service: cluster DNS (CoreDNS) resolves `<service>.<namespace>.svc.cluster.local` to the Service's ClusterIP, and kube-proxy routes to a ready Pod endpoint.
</Accordion>

<Accordion title="What is the difference between Ingress and a LoadBalancer Service?">
A LoadBalancer Service exposes one Service at L4 via a cloud LB. Ingress is an L7 routing object (host/path rules, TLS) implemented by an Ingress controller, which itself is typically exposed with one LoadBalancer Service; one entry point fronts many Services.
</Accordion>

<Accordion title="What happens when you run `kubectl apply -f deployment.yaml`?">
kubectl sends the object to the API server, which authenticates, authorises, runs admission (mutating/validating), and stores it in etcd. The Deployment controller creates/updates a ReplicaSet, which creates Pods. The scheduler assigns nodes; each kubelet pulls images and starts containers; readiness gates adding Pods to Service endpoints.
</Accordion>

## Related
- [Kubernetes](/Software%20Engineering/DevOps/Kubernetes/Kubernetes) (hub) · [DevOps Index](/Software%20Engineering/DevOps/DevOps%20Index)
- [Kubernetes Workloads in Production](/Software%20Engineering/DevOps/Kubernetes/Kubernetes%20Workloads%20in%20Production) · [Kubernetes Networking - Services, Ingress & Gateway API](/Software%20Engineering/DevOps/Kubernetes/Kubernetes%20Networking%20-%20Services%2C%20Ingress%20%26%20Gateway%20API) · [Helm & Kustomize](/Software%20Engineering/DevOps/Kubernetes/Helm%20%26%20Kustomize) · [GitOps - Argo CD & Flux](/Software%20Engineering/DevOps/GitOps%20-%20Argo%20CD%20%26%20Flux)
- [Docker Basics](/Software%20Engineering/DevOps/Docker/Docker%20Basics) · [Microservices Architecture](/Software%20Engineering/DevOps/Microservices%20Architecture) · [Nginx](/Software%20Engineering/Backend/Nginx)

## Sources
- [Kubernetes docs: Concepts overview](https://kubernetes.io/docs/concepts/overview/)
- [Kubernetes docs: Deployments](https://kubernetes.io/docs/concepts/workloads/controllers/deployment/)
- [Kubernetes docs: Service](https://kubernetes.io/docs/concepts/services-networking/service/)
- [Kubernetes docs: Ingress](https://kubernetes.io/docs/concepts/services-networking/ingress/)
- [Kubernetes blog: Ingress NGINX Retirement (2025-11-11)](https://kubernetes.io/blog/2025/11/11/ingress-nginx-retirement/)
- [Kubernetes releases](https://kubernetes.io/releases/)
- [Skaffold docs](https://skaffold.dev/docs/)
- Stephen Grider, *Microservices with Node JS and React* (Udemy)
