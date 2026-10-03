---
tags: [devops, devops/ci-cd]
aliases: [Jenkins, Jenkinsfile, Jenkins Pipeline, Declarative Pipeline, JCasC, Jenkins Shared Library, Jenkins on Kubernetes]
updated: 2026-09-27
---
# Jenkins - Pipelines as Code

> [!tldr] TL;DR
> **Jenkins** is the self-hosted, plugin-driven automation server that still runs a large share of enterprise CI. You describe the pipeline in a **`Jenkinsfile`** (Declarative syntax) stored with the code, run the **controller** on Kubernetes with the official Helm chart, configure it with **JCasC** YAML, and run every build in an **ephemeral agent pod** via the Kubernetes plugin. In Acme Shop, Jenkins builds, tests, scans and pushes `ghcr.io/acme/<app>:<git-sha>`, then commits the new tag to `acme/platform-gitops`; from there **Argo CD** takes over. Its costs are plugin maintenance and a stateful controller; as of the April 2026 LTS it needs **Java 21+**.

This note is the hands-on Jenkins deep dive. General CI/CD concepts (CI vs CD, deployment strategies, DORA, GitHub Actions) live in [[CI-CD Pipelines]]; the deploy side lives in [[Argo CD in Practice]] and [[Argo Rollouts - Canary & Blue-Green]].

## 1. What Jenkins is, and where it fits in 2026

Jenkins is an open-source (MIT) automation server written in Java. Out of the box it does very little; almost everything (Git, Pipeline, Kubernetes agents, credentials, UI views) comes from **plugins** (roughly 2,000 in the update center). A **Pipeline** is a Groovy-based DSL that describes stages and steps; "pipeline as code" means that DSL lives in a `Jenkinsfile` in the repo and is reviewed like any other code.

| | **Jenkins** | **GitHub Actions** | **GitLab CI** | **Tekton** | **Buildkite** |
|---|---|---|---|---|---|
| Hosting | Self-hosted only (you run the controller) | SaaS (+ self-hosted runners, GHES) | SaaS or self-managed | Self-hosted, Kubernetes-native CRDs | SaaS control plane, your agents |
| Pipeline format | Groovy `Jenkinsfile` (Declarative or Scripted) | YAML workflows | `.gitlab-ci.yml` | YAML `Task`/`Pipeline` CRDs | YAML, dynamic pipelines via script |
| Extensibility | ~2,000 plugins, shared libraries, arbitrary Groovy | Marketplace actions, reusable workflows | Components/templates, includes | Tasks from Tekton Hub/catalog, anything in a container | Plugins, dynamic pipeline generation |
| State | Stateful controller (`JENKINS_HOME`) you back up and upgrade | None for you | None (SaaS) or GitLab instance | State in etcd (PipelineRuns) | None for you |
| Best when | On-prem/air-gapped, complex enterprise flows, many SCMs, heavy existing investment | Code is on GitHub, you want zero ops | Code is on GitLab, one integrated platform | Building your own platform on Kubernetes | Large monorepos, huge parallelism on your own compute |
| Main costs | Plugin upgrades and CVEs, controller ops, Groovy sandbox quirks, dated UX | Vendor lock-in, runner minutes, action supply chain | Lock-in to GitLab | Low-level, verbose, you build the UX | Per-seat SaaS cost, agent ops |

**When Jenkins still wins**
- Air-gapped or regulated on-prem environments where SaaS CI is not allowed.
- Complex, long-lived enterprise pipelines: many SCMs (GitHub, Bitbucket, Gerrit, Perforce), manual gates, integration with legacy tools that only have a Jenkins plugin.
- Organisations with years of shared libraries and trained staff: migration cost is real.
- Full control of compute (GPUs, big build machines, special hardware, mainframes).

**What it costs you**
- **Plugin maintenance**: every plugin is a dependency with its own release cadence and CVEs. Most Jenkins security advisories are plugin advisories.
- **Controller state**: the controller is a pet. It holds config, credentials, build history and queue state on disk. You must back it up, upgrade it, and plan for its downtime.
- **Groovy CPS**: pipelines are run by a continuation-passing interpreter so they survive restarts; some normal Groovy (closures in `each`, non-serialisable objects) behaves surprisingly.

## 2. Architecture

```mermaid
flowchart LR
  subgraph K8s["Kubernetes cluster, namespace jenkins"]
    C["Controller pod<br/>UI, scheduler, queue, plugins<br/>JENKINS_HOME on a PVC<br/>0 executors"]
    subgraph P1["Agent pod: orders-api #42"]
      J1[jnlp container<br/>agent.jar]
      N1[node]
      B1[buildkit]
    end
    subgraph P2["Agent pod: catalog #17"]
      J2[jnlp]
      N2[node]
    end
  end
  GH[GitHub webhook] --> C
  C -- "creates pod via K8s API" --> P1
  J1 -- "WebSocket / TCP 50000" --> C
  C -- "creates pod" --> P2
  J2 --> C
```

| Concept | What it is |
|---|---|
| **Controller** | The Jenkins server: web UI, REST API, job config, build queue, scheduling, plugin host. Formerly called "master". |
| **Agent** | A machine or container that runs builds. Connects to the controller (inbound/JNLP over TCP or WebSocket, or outbound SSH). Formerly "slave". |
| **Executor** | A slot on a node that runs one build (or one branch of a `parallel`) at a time. |
| **Label** | A tag on agents; pipelines request `agent { label 'linux' }` and the scheduler picks a matching node. |
| **Workspace** | The per-job directory on the agent where source is checked out and built. On Kubernetes agents it is an `emptyDir` shared by all containers in the pod. |
| **Plugin** | A `.hpi/.jpi` extension. Pipeline itself, Git, Kubernetes, Credentials and JCasC are all plugins. |
| **`JENKINS_HOME`** | The controller's data directory: `config.xml`, `jobs/`, `plugins/`, `secrets/`, `credentials.xml`, build logs. |

**Why builds must never run on the controller**: a build on the built-in node runs as the Jenkins process user, so it can read `JENKINS_HOME`, including `secrets/master.key` and every stored credential, and it can rewrite job config. A malicious PR (or a compromised npm package in `npm ci`) then owns your CI. It also competes with the UI and scheduler for CPU and memory. Set the built-in node to **0 executors** (the Helm chart does by default) and run all work on agents. The Jenkins docs call this "controller isolation".

## 3. Installing on Kubernetes (Helm + JCasC)

### 3.1 Helm chart

```bash
helm repo add jenkins https://charts.jenkins.io
helm repo update
helm upgrade --install jenkins jenkins/jenkins \
  --namespace jenkins --create-namespace \
  -f values-jenkins.yaml
# also published as OCI: oci://ghcr.io/jenkinsci/helm-charts/jenkins
```

In Acme Shop this chart is itself deployed by Argo CD from `platform-gitops/platform/jenkins/` (Helm for third-party platform charts), so Jenkins' own config is GitOps-managed too.

```yaml
# values-jenkins.yaml
controller:
  numExecutors: 0                  # chart default; never build on the controller
  image:
    repository: ghcr.io/acme/jenkins   # custom image with plugins baked in (see below)
    tag: "2.568.3-jdk21"
  installPlugins: false            # plugins come from the image, not downloaded at boot
  resources:
    requests: { cpu: "1", memory: 4Gi }
    limits:   { memory: 4Gi }
  javaOpts: "-XX:MaxRAMPercentage=75"
  JCasC:
    defaultConfig: true            # chart generates the Kubernetes cloud config
    authorizationStrategy: |-
      globalMatrix:
        entries:
          - group:
              name: "authenticated"
              permissions: ["Overall/Read", "Job/Read"]
          - group:
              name: "platform-team"
              permissions: ["Overall/Administer"]
          - group:
              name: "developers"
              permissions: ["Job/Build", "Job/Cancel", "Run/Replay"]
    configScripts:
      acme: |
        jenkins:
          systemMessage: "Acme Shop CI. Config is managed in platform-gitops; UI edits are overwritten."
        unclassified:
          location:
            url: "https://ci.acme.dev/"
          globalLibraries:
            libraries:
              - name: "acme-pipeline"
                defaultVersion: "v1"
                allowVersionOverride: true
                retriever:
                  modernSCM:
                    scm:
                      git:
                        remote: "https://github.com/acme/jenkins-shared-library.git"
                        credentialsId: "github-app"
persistence:
  enabled: true
  size: 50Gi                       # chart default is 8Gi
agent:
  enabled: true
```

- **Custom image with pinned plugins** beats `installPlugins` at boot: startup does not depend on the update center, and the plugin set is reviewed and reproducible.
  ```dockerfile
  FROM jenkins/jenkins:2.568.3-jdk21
  COPY plugins.txt /usr/share/jenkins/ref/plugins.txt
  RUN jenkins-plugin-cli --plugin-file /usr/share/jenkins/ref/plugins.txt
  ```
  `plugins.txt` lists `kubernetes`, `workflow-aggregator`, `git`, `github-branch-source`, `configuration-as-code`, `matrix-auth`, `credentials-binding`, `pipeline-graph-view`, `junit`, `timestamper`, `pipeline-utility-steps`, each pinned to a version (`name:version`) that Renovate/Dependabot bumps via PR.
- Expose the UI through the chart's **Gateway API HTTPRoute** option (or your own `HTTPRoute`), see [[Kubernetes Networking - Services, Ingress & Gateway API]]. Put SSO (OIDC) in front; the chart's default `local` realm with one admin is for bootstrap only.

### 3.2 Configuration as Code (JCasC)

The **Configuration as Code** plugin reads YAML (from files, a directory, or the chart's ConfigMaps) and applies it at startup and on reload. It replaces clicking through *Manage Jenkins*: security realm, authorisation, clouds, credentials, global libraries, tool installs. The root keys map to the config pages: `jenkins:` (core), `unclassified:` (most plugin globals), `credentials:`, `security:`, `tool:`.

Rules that keep it sane:
- JCasC is **authoritative**: treat UI edits as drift that the next reload overwrites.
- Secrets are **referenced, not written**: `${GITHUB_APP_KEY}` resolves from environment variables or mounted secret files (e.g. a Kubernetes Secret mounted at `/run/secrets`). See [[Secrets Management]].
- Export the current config from *Manage Jenkins → Configuration as Code → View Configuration* when you need the YAML shape of an unfamiliar plugin.

### 3.3 Kubernetes plugin: ephemeral pod agents

The **Kubernetes plugin** creates one pod per build (or per stage that declares an agent), runs the pipeline in it, and deletes it afterwards. Every pod gets an injected `jnlp` container that runs `agent.jar` and connects back to the controller; your tool containers sit next to it and share the workspace `emptyDir`. Steps run in `jnlp` unless you switch with `container('name') { ... }` or set `defaultContainer`.

The pod template for `orders-api` lives in its repo as `ci/jenkins-pod.yaml`:

```yaml
# acme/orders-api: ci/jenkins-pod.yaml
apiVersion: v1
kind: Pod
spec:
  securityContext:
    runAsUser: 1000              # jnlp, node and buildkit all use uid 1000
    runAsGroup: 1000
    fsGroup: 1000
  containers:
    - name: node
      image: node:24-bookworm-slim
      command: ["cat"]
      tty: true
      resources:
        requests: { cpu: "1", memory: 2Gi }
        limits:   { memory: 2Gi }
    - name: postgres               # sidecar for integration tests, reachable on localhost:5432
      image: postgres:18
      securityContext: { runAsUser: 999 }
      env:
        - { name: POSTGRES_USER, value: test }
        - { name: POSTGRES_PASSWORD, value: test }   # throwaway, pod-local
        - { name: POSTGRES_DB, value: orders_test }
    - name: buildkit
      image: moby/buildkit:rootless  # pin by digest in real life
      command: ["cat"]
      tty: true
      env:
        - { name: BUILDKITD_FLAGS, value: --oci-worker-no-process-sandbox }
      securityContext:
        seccompProfile: { type: Unconfined }
        appArmorProfile: { type: Unconfined }   # field needs Kubernetes 1.30+
      volumeMounts:
        - { name: buildkitd, mountPath: /home/user/.local/share/buildkit }
    - name: trivy
      image: aquasec/trivy@sha256:<pinned-digest>   # pin by digest (see the March 2026 incident)
      command: ["cat"]
      tty: true
      env:
        - { name: TRIVY_CACHE_DIR, value: /tmp/trivy }
    - name: skopeo
      image: quay.io/skopeo/stable:latest           # pin in real life
      command: ["cat"]
      tty: true
    - name: tools                  # internal image: git + kustomize
      image: ghcr.io/acme/ci-tools:2026.09
      command: ["cat"]
      tty: true
  volumes:
    - name: buildkitd
      emptyDir: {}
```

Useful knobs: `podRetention onFailure()` keeps a failed pod around for debugging; `inheritFrom 'base'` reuses a template defined in JCasC; `idleMinutes` keeps a pod warm for back-to-back builds (trading isolation for speed). Give agent pods a ServiceAccount with **no** Kubernetes API permissions; they only need to reach the controller and the registry.

## 4. The `orders-api` Jenkinsfile (Declarative, fully worked)

### 4.1 Flow

```mermaid
flowchart LR
  Dev[git push / PR<br/>acme/orders-api] -->|webhook| J[Jenkins multibranch job]
  subgraph Pod["Ephemeral agent pod"]
    J --> I[install] --> Q{{"lint ∥ test<br/>JUnit"}} --> B[build image<br/>BuildKit rootless] --> S[Trivy scan]
  end
  S -->|main only| R[(ghcr.io/acme/orders-api:&lt;sha&gt;)]
  S -->|main only| G[commit to acme/platform-gitops<br/>apps/orders-api/overlays/dev]
  G --> A[Argo CD sync<br/>shop-dev]
  R -. image pull .-> A
  G -. input gate .-> P[bump overlays/prod]
  P --> A2[Argo CD + Argo Rollouts canary<br/>shop-prod]
```

Jenkins' job ends at "commit to the GitOps repo". It never runs `kubectl apply` against the cluster and holds no cluster credentials; Argo CD pulls. That is the core security win of GitOps (see [[Argo CD in Practice]]).

### 4.2 Jenkinsfile

```groovy
// acme/orders-api: Jenkinsfile

// Helper defined outside the pipeline block (allowed in a Jenkinsfile).
// Bumps the image tag in one overlay of the GitOps repo and pushes.
def bumpGitOps(String overlay, String note) {
  withCredentials([usernamePassword(credentialsId: 'github-gitops-bot',
                                    usernameVariable: 'GIT_USER',
                                    passwordVariable: 'GIT_TOKEN')]) {
    withEnv(["OVERLAY=${overlay}", "NOTE=${note}"]) {
      retry(3) {   // another app may have pushed first: re-clone and retry
        // Single quotes: the SHELL expands $GIT_TOKEN, Groovy never sees the secret.
        sh '''
          set -eu
          git config --global credential.helper '!f() { echo "username=${GIT_USER}"; echo "password=${GIT_TOKEN}"; }; f'
          git config --global user.name  "acme-ci"
          git config --global user.email "ci@acme.dev"
          rm -rf gitops
          git clone --depth 1 https://github.com/acme/platform-gitops.git gitops
          (cd "gitops/apps/orders-api/overlays/${OVERLAY}" && \
            kustomize edit set image "ghcr.io/acme/orders-api=ghcr.io/acme/orders-api:${IMAGE_TAG}")
          cd gitops
          if git diff --quiet; then echo "already at ${IMAGE_TAG}"; exit 0; fi
          git commit -am "orders-api(${OVERLAY}): ${IMAGE_TAG} ${NOTE}"
          git push origin HEAD:main
        '''
      }
    }
  }
}

pipeline {
  agent none                      // each top-level stage picks its own agent

  options {
    buildDiscarder(logRotator(numToKeepStr: '30', artifactNumToKeepStr: '10'))
    disableConcurrentBuilds(abortPrevious: true)   // new push cancels the old build of the same branch
    skipDefaultCheckout()
    timestamps()
    preserveStashes(buildCount: 5)                  // lets "Restart from stage" reuse stashes
  }

  environment {
    APP   = 'orders-api'
    IMAGE = 'ghcr.io/acme/orders-api'
  }

  stages {
    stage('CI') {
      agent {
        kubernetes {
          yamlFile 'ci/jenkins-pod.yaml'
          defaultContainer 'node'
        }
      }
      options {
        timeout(time: 30, unit: 'MINUTES')          // bounds the pod's lifetime, not the prod gate
      }
      stages {
        stage('Checkout') {
          steps {
            checkout scm
            script {
              env.IMAGE_TAG = sh(script: 'git rev-parse HEAD', returnStdout: true).trim()
            }
            echo "Building ${env.APP} at ${env.IMAGE_TAG}"
          }
        }

        stage('Install') {
          steps {
            retry(3) {                               // registry hiccups are the usual flake
              sh 'npm ci --no-audit --no-fund'
            }
          }
        }

        stage('Quality') {
          parallel {
            stage('Lint') {
              steps {
                sh 'npm run lint && npm run typecheck'
              }
            }
            stage('Test') {
              environment {
                DATABASE_URL = 'postgres://test:test@localhost:5432/orders_test'
              }
              steps {
                // package.json: "test": "node --test"
                sh '''
                  mkdir -p reports
                  npm test -- \
                    --test-reporter=spec  --test-reporter-destination=stdout \
                    --test-reporter=junit --test-reporter-destination=reports/junit.xml
                '''
              }
              post {
                always {
                  junit testResults: 'reports/junit.xml', allowEmptyResults: false
                }
              }
            }
          }
        }

        stage('Build image') {
          steps {
            container('buildkit') {
              sh '''
                buildctl-daemonless.sh build \
                  --frontend dockerfile.v0 \
                  --local context=. --local dockerfile=. \
                  --opt build-arg:GIT_SHA="${IMAGE_TAG}" \
                  --output type=docker,name="${IMAGE}:${IMAGE_TAG}",dest=image.tar
              '''
            }
          }
        }

        stage('Scan') {
          steps {
            container('trivy') {
              sh '''
                mkdir -p release
                trivy image --input image.tar \
                  --severity HIGH,CRITICAL --ignore-unfixed --exit-code 1
                trivy image --input image.tar --format cyclonedx --output release/sbom.cdx.json
              '''
            }
          }
        }

        stage('Push') {
          when { branch 'main' }                     // PRs build and scan, but never publish
          environment {
            GHCR = credentials('ghcr-push')          // username/password → GHCR_USR, GHCR_PSW
          }
          steps {
            container('skopeo') {
              sh '''
                printf '%s' "$GHCR_PSW" | skopeo login ghcr.io -u "$GHCR_USR" \
                  --password-stdin --authfile /tmp/auth.json
                skopeo copy --authfile /tmp/auth.json --digestfile release/digest \
                  docker-archive:image.tar "docker://${IMAGE}:${IMAGE_TAG}"
              '''
            }
            sh 'echo "${IMAGE_TAG}" > release/tag'
            stash name: 'release', includes: 'release/**'
          }
        }

        stage('Deploy dev (GitOps)') {
          when { branch 'main' }
          steps {
            container('tools') {
              bumpGitOps('dev', "(build ${env.BUILD_NUMBER})")
            }
          }
        }
      }
      post {
        always {
          archiveArtifacts artifacts: 'release/**', allowEmptyArchive: true
        }
      }
    }

    stage('Promote to prod') {
      when {
        beforeInput true                             // don't even ask on non-main branches
        branch 'main'
      }
      options {
        timeout(time: 24, unit: 'HOURS')             // abort if nobody approves within a day
      }
      input {
        message 'Promote orders-api to production?'
        ok 'Promote'
        submitter 'release-managers'                 // group or user IDs allowed to approve
        parameters {
          string(name: 'CHANGE_TICKET', defaultValue: '', description: 'Change ticket, e.g. CHG-1234')
        }
      }
      agent {                                        // allocated only AFTER approval
        kubernetes {
          yamlFile 'ci/jenkins-pod.yaml'
          defaultContainer 'tools'
        }
      }
      steps {
        unstash 'release'
        sh 'cat release/tag release/digest'
        bumpGitOps('prod', "approved, ${env.CHANGE_TICKET}")
      }
    }
  }

  post {
    success { echo "${env.APP} ${env.IMAGE_TAG} delivered" }
    failure { echo "Notify #team-orders: ${env.JOB_NAME} #${env.BUILD_NUMBER} failed: ${env.BUILD_URL}" }
    // In real life: slackSend(...) or mail(...) from the relevant plugin.
  }
}
```

### 4.3 What each construct is doing

| Construct | Where | Why |
|---|---|---|
| `agent none` + stage agents | top level | The prod gate can wait hours **without** holding a pod or an executor. A top-level agent would keep the pod alive during `input`. |
| Nested `stages` under `CI` | `CI` stage | All CI steps share one pod and one workspace (no re-checkout, no re-install). |
| `when { branch 'main' }` | Push, Deploy, Promote | PR and feature-branch builds verify but never publish. Other conditions: `changeRequest()`, `tag 'v*'`, `changeset 'src/**'`, `expression { ... }`, `allOf/anyOf/not`. |
| `when { beforeInput true }` | Promote | Evaluate `when` before prompting, so feature branches never show an approval dialog. |
| `parallel` | Quality | Lint and tests run concurrently in the same workspace; `failFast true` (or `parallelsAlwaysFailFast()` option) aborts siblings on the first failure. |
| `post { always { junit ... } }` | Test | Publishes test reports even when tests fail (that is when you need them). Conditions: `always`, `success`, `failure`, `unstable`, `changed`, `fixed`, `regression`, `aborted`, `cleanup`. |
| `credentials('ghcr-push')` | Push `environment` | Username/password credentials expose `GHCR_USR` and `GHCR_PSW`; secret-text credentials expose one variable. Values are masked in the log. |
| `withCredentials([...])` | `bumpGitOps` | Scoped binding for exactly the block that needs it. |
| `timeout` | `CI` stage, Promote | A hung `npm ci` cannot hold a pod forever; approvals expire. **Do not** put a 30-minute timeout at pipeline level, or it would also kill the 24-hour prod gate. |
| `retry(3)` | Install, GitOps push | Retry only idempotent, known-flaky steps (network, push races). Never retry tests to hide flakiness. |
| `input` | Promote | Human approval gate with `submitter` restriction and an audit trail on the build page. |
| `stash` / `unstash` | Push → Promote | Moves small files (tag, digest, SBOM) between pods. Not for large artifacts; use the registry or an artifact store. |
| `disableConcurrentBuilds(abortPrevious: true)` | options | Newer push to the same branch cancels the stale build. |

Note on string quoting: `sh '...'` (single quotes) lets the **shell** expand `$GHCR_PSW`; `sh "...${GHCR_PSW}..."` would make **Groovy** interpolate the secret into the command line, where it can leak (process list, `set -x`, Replay). Jenkins warns about this in the build log.

## 5. Building images in Kubernetes without Docker-in-Docker

Mounting `/var/run/docker.sock` or running privileged DinD gives the build **root on the node**. Options that do not:

| Tool | How it works | Status (Sep 2026) | Notes |
|---|---|---|---|
| **BuildKit rootless** (`moby/buildkit:rootless`) | Runs `buildkitd` in a user namespace as uid 1000; `buildctl-daemonless.sh` starts a daemon per build | Active, the engine behind `docker build` | Needs `seccompProfile`/`appArmorProfile: Unconfined` and usually `--oci-worker-no-process-sandbox` (the docs note this weakens isolation between build steps and the daemon). Excellent caching (`--export-cache type=registry`). Can also run as a shared remote `buildkitd` Deployment for a warm cache. |
| **Buildah** | Daemonless OCI builder from the Podman family; `buildah bud` / `buildah build` with `--isolation chroot` | Active | Good fit on OpenShift/RHEL; rootless mode needs user-namespace and storage-driver tuning (check the Buildah docs for your cluster). |
| **Kaniko** | Executes Dockerfile steps in userspace inside a normal container | **Google archived `GoogleContainerTools/kaniko` on 2025-06-03**; read-only | Chainguard announced a maintained fork on 2025-06-05 (security fixes and minor bug fixes, no new features), now at `github.com/chainguard-forks/kaniko`. Fine for existing pipelines; for new ones prefer BuildKit or Buildah. |
| Hosted builders | Docker Build Cloud, cloud build services | Active | Moves the privilege problem off your cluster; costs money and egress. |

Whatever you choose: build **once**, scan the exact artifact you push (the example scans `image.tar`, then pushes that same tarball), and deploy by tag + digest. Image hardening itself is covered in [[Dockerfile Best Practices & Image Security]]; signing and SBOM attestation in [[Supply Chain Security]].

## 6. A second Jenkinsfile: the `catalog` Next.js micro-frontend

`catalog` is a Next.js 16 zone served under `/catalog/*` ([[Deploying Next.js Micro-frontends to Kubernetes]]). The pipeline shape is the same; the interesting part is **build-time environment variables**.

> [!warning] `NEXT_PUBLIC_*` is baked in at build time
> Next.js inlines every `process.env.NEXT_PUBLIC_*` reference into the client JavaScript during `next build`. Changing the variable in the Kubernetes Deployment later does **nothing** for browser code. If you pass environment-specific values (`NEXT_PUBLIC_API_URL=https://dev.shop.acme.dev`) as build args, you need one image per environment, which breaks "build once, promote the same digest".
>
> Acme's rule: only **environment-agnostic** values are `NEXT_PUBLIC_*` (build SHA, feature defaults). The API is called with a **relative** path (`/api/orders`), which works in every environment because the Gateway routes it on the same host. Anything environment-specific is read **at runtime** on the server (server components, route handlers) from normal env vars. See [[Next.js App Router Fundamentals]].

```groovy
// acme/catalog: Jenkinsfile
pipeline {
  agent {
    kubernetes {
      yamlFile 'ci/jenkins-pod.yaml'     // node, buildkit, trivy, skopeo, tools containers
      defaultContainer 'node'
    }
  }

  options {
    disableConcurrentBuilds(abortPrevious: true)
    timeout(time: 30, unit: 'MINUTES')   // fine here: no input gate in this pipeline
    timestamps()
  }

  environment {
    IMAGE = 'ghcr.io/acme/catalog'
    NEXT_TELEMETRY_DISABLED = '1'
  }

  stages {
    stage('Install') {
      steps {
        script {
          env.IMAGE_TAG = sh(script: 'git rev-parse HEAD', returnStdout: true).trim()
        }
        retry(3) { sh 'npm ci --no-audit --no-fund' }
      }
    }

    stage('Verify') {
      parallel {
        stage('Lint & types') { steps { sh 'npm run lint && npx tsc --noEmit' } }
        stage('Unit tests') {
          steps { sh 'npm test -- --reporter=junit --outputFile=reports/junit.xml' }   // Vitest
          post  { always { junit 'reports/junit.xml' } }
        }
      }
    }

    stage('Build image') {
      steps {
        container('buildkit') {
          // Only env-agnostic public values are baked in. No API URLs, no per-env keys.
          sh '''
            buildctl-daemonless.sh build \
              --frontend dockerfile.v0 --local context=. --local dockerfile=. \
              --opt build-arg:NEXT_PUBLIC_BUILD_SHA="${IMAGE_TAG}" \
              --import-cache type=registry,ref=ghcr.io/acme/catalog:buildcache \
              --output type=docker,name="${IMAGE}:${IMAGE_TAG}",dest=image.tar
          '''
        }
      }
    }

    stage('Scan') {
      steps {
        container('trivy') {
          sh 'trivy image --input image.tar --severity HIGH,CRITICAL --ignore-unfixed --exit-code 1'
        }
      }
    }

    stage('Push & deploy dev') {
      when { branch 'main' }
      environment { GHCR = credentials('ghcr-push') }
      steps {
        container('skopeo') {
          sh '''
            printf '%s' "$GHCR_PSW" | skopeo login ghcr.io -u "$GHCR_USR" --password-stdin --authfile /tmp/auth.json
            skopeo copy --authfile /tmp/auth.json docker-archive:image.tar "docker://${IMAGE}:${IMAGE_TAG}"
          '''
        }
        container('tools') {
          echo 'Same GitOps bump as orders-api, for apps/catalog/overlays/dev (shared library in section 7)'
        }
      }
    }
  }

  post {
    always { archiveArtifacts artifacts: 'reports/**', allowEmptyArchive: true }
  }
}
```

Next.js specifics worth knowing in CI: use `output: 'standalone'` for a small runtime image, set `basePath: '/catalog'` (or `assetPrefix`) so zone assets do not collide with `shell`, and cache `.next/cache` through a BuildKit cache mount (`RUN --mount=type=cache,target=/app/.next/cache npm run build`) so rebuilds are fast.

## 7. Multibranch pipelines, webhooks and Shared Libraries

### 7.1 Multibranch + GitHub

- A **Multibranch Pipeline** job scans a repo and creates one child job per branch and per PR that contains a `Jenkinsfile`. An **Organization Folder** does the same for every repo in a GitHub org (`acme/*`).
- Use the **GitHub Branch Source** plugin authenticated as a **GitHub App** (higher rate limits, fine-grained repo permissions, no personal token).
- Configure a **GitHub webhook** (`https://ci.acme.dev/github-webhook/`, events: push, pull request) so builds start in seconds instead of on periodic scans. Keep a slow periodic scan as a safety net.
- **PR builds**: GitHub Branch Source builds the PR merged with its target branch by default, and reports status checks back to the PR. Require that check in branch protection.
- **Fork PRs are untrusted**: by default Jenkins uses the target branch's `Jenkinsfile` for forks from untrusted contributors ("trust" setting), so a fork cannot rewrite the pipeline to steal credentials. Keep it that way.
- Useful env vars: `BRANCH_NAME`, `CHANGE_ID` (PR number), `CHANGE_TARGET`, `TAG_NAME`.

### 7.2 Shared Library: one pipeline for all four apps

Four repos with near-identical Jenkinsfiles drift quickly. A **Shared Library** is a Git repo that pipelines load with `@Library`. Layout:

```
jenkins-shared-library/
  vars/
    buildNodeService.groovy    # global step, callable as buildNodeService(...)
    gitopsBump.groovy
  resources/
    pods/node.yaml             # loaded with libraryResource
  src/acme/ci/...              # optional Groovy classes
```

```groovy
// vars/buildNodeService.groovy
def call(Map config) {
  // config.app: 'orders-api' | 'catalog' | 'checkout' | 'shell'
  // config.kind: 'node-api' | 'nextjs'
  def podYaml = libraryResource("pods/${config.kind}.yaml")

  pipeline {
    agent {
      kubernetes {
        yaml podYaml
        defaultContainer 'node'
      }
    }
    options {
      disableConcurrentBuilds(abortPrevious: true)
      timeout(time: 30, unit: 'MINUTES')
      timestamps()
    }
    environment {
      APP   = "${config.app}"
      IMAGE = "ghcr.io/acme/${config.app}"
    }
    stages {
      stage('Install') {
        steps {
          script { env.IMAGE_TAG = sh(script: 'git rev-parse HEAD', returnStdout: true).trim() }
          retry(3) { sh 'npm ci --no-audit --no-fund' }
        }
      }
      stage('Verify') {
        parallel {
          stage('Lint') { steps { sh 'npm run lint' } }
          stage('Test') {
            steps { sh 'npm test' }
            post  { always { junit testResults: 'reports/*.xml', allowEmptyResults: true } }
          }
        }
      }
      stage('Build image') {
        steps {
          container('buildkit') {
            sh '''
              buildctl-daemonless.sh build --frontend dockerfile.v0 \
                --local context=. --local dockerfile=. \
                --output type=docker,name="${IMAGE}:${IMAGE_TAG}",dest=image.tar
            '''
          }
        }
      }
      stage('Scan') {
        steps {
          container('trivy') {
            sh 'trivy image --input image.tar --severity HIGH,CRITICAL --ignore-unfixed --exit-code 1'
          }
        }
      }
      stage('Push & deploy dev') {
        when { branch 'main' }
        environment { GHCR = credentials('ghcr-push') }
        steps {
          container('skopeo') {
            sh '''
              printf '%s' "$GHCR_PSW" | skopeo login ghcr.io -u "$GHCR_USR" --password-stdin --authfile /tmp/auth.json
              skopeo copy --authfile /tmp/auth.json docker-archive:image.tar "docker://${IMAGE}:${IMAGE_TAG}"
            '''
          }
          container('tools') {
            gitopsBump(app: config.app, overlay: 'dev')   // vars/gitopsBump.groovy
          }
        }
      }
    }
  }
}
```

Each app's Jenkinsfile shrinks to:

```groovy
// acme/checkout: Jenkinsfile
@Library('acme-pipeline@v1') _
buildNodeService(app: 'checkout', kind: 'nextjs')
```

- **Version the library** (`@v1` tag or branch) and release it like any other code; `@main` means every app picks up a broken change at once.
- Libraries configured globally by an admin (as in the JCasC above) are **trusted**: they run outside the Groovy sandbox. Protect that repo with the same branch protection and review as production code.
- Test library changes with a PR-pinned version (`@Library('acme-pipeline@pr-42') _`, allowed by `allowVersionOverride`) or with the JenkinsPipelineUnit framework.
- Monorepo variant: one Jenkinsfile loops over changed apps (`changeset` conditions or a script that diffs paths) and calls the same library step per app. See [[Monorepos]].

## 8. Credentials, secrets and security hardening

**Credentials**
- Store secrets in the **Credentials** plugin (encrypted on disk with keys in `JENKINS_HOME/secrets/`), or better, source them from an external store: Kubernetes Secrets (via the Kubernetes Credentials Provider plugin), HashiCorp Vault, or cloud secret managers. See [[Secrets Management]].
- Scope credentials to **folders** (e.g. `shop/orders-api`) rather than global, so one team's job cannot bind another team's secrets.
- Prefer short-lived credentials: GitHub App installation tokens instead of PATs; cloud OIDC/workload identity where your plugins support it.
- Masking is best-effort: base64-encoding, `set -x`, or splitting a secret defeats it. Never `echo` a secret, and keep secrets out of Groovy string interpolation.

**Hardening checklist**

| Area | What to do |
|---|---|
| Authentication | SSO via OIDC/SAML/LDAP; disable signup; no shared admin account. |
| Authorisation | **Matrix Authorization Strategy** (global + per-folder "project-based" matrix). Developers get Build/Cancel/Read on their folder; very few get `Overall/Administer`. |
| CSRF | CSRF protection (crumb issuer) is always on in modern Jenkins; API clients use an API token, which does not need a crumb. |
| Script security | Pipelines run in the **Groovy sandbox**; unsafe method calls need **In-process Script Approval** by an admin. Approve sparingly: approving `Runtime.exec` or `Jenkins.instance` from a Jenkinsfile gives any committer admin. Disable the Script Console for non-admins (it is admin-only by default; keep it that way). |
| Controller isolation | 0 executors on the built-in node; agents cannot write to controller files (agent-to-controller security is always on). |
| Plugins | Fewer plugins = smaller attack surface. Remove unused ones; update weekly from a tested image; subscribe to the security advisory mailing list. |
| Network | Controller not on the public internet without SSO; agents connect over WebSocket or TCP 50000 only from the cluster; restrict agent pod egress with NetworkPolicies. |
| Logs | No secrets in console output; restrict who can read build logs of sensitive jobs; set retention with `buildDiscarder`. |

**Backup of `JENKINS_HOME`**
- With JCasC, Jenkinsfiles and shared libraries in Git, `JENKINS_HOME` mostly holds **build history, credentials and fingerprints**. Rebuilding config is cheap; losing credentials and audit history is not.
- Back up the PVC with CSI volume snapshots or Velero on a schedule, and test restores.
- Always back up `secrets/` (including `master.key` and `hudson.util.Secret`) **together with** `credentials.xml` and job configs; without the keys the encrypted credentials are unreadable. Protect backups like the secrets they contain.
- Exclude `workspace/` and caches; they are disposable.

## 9. Viewing logs and debugging

| Where | What you get |
|---|---|
| **Console Output** (`/job/.../<n>/console`) | The full raw log with timestamps. First place to look. `consoleText` gives plain text for `curl`/`grep`. |
| **Pipeline Graph View** plugin ("Pipeline Overview" on a build) | Nested stage graph, per-stage logs that stream live, collapsible step list. Recommended replacement for Blue Ocean views. |
| **Pipeline: Stage View** | Classic stage table on the job page. |
| **Blue Ocean** | **Deprecated** on the plugin site: no new features, only selective fixes for significant security issues or defects. Don't adopt it for new setups. |
| Test results | The JUnit plugin adds a "Tests" tab with trends and per-test history; failed tests show on the build page. |
| Pipeline Steps (`/flowGraphTable`) | Every step, its duration and which node ran it. Great for "what is slow". |

**Replay and restart**
- **Replay** re-runs a build with an edited Jenkinsfile (and edited shared library code) without committing: ideal to test a fix, then commit it. Needs the `Run/Replay` permission, which is effectively "run arbitrary code with this job's credentials", so grant it carefully.
- **Restart from Stage** (Declarative) re-runs from a chosen top-level stage with the same commit; `preserveStashes()` keeps stashes available for it.

**Agent pod debugging with kubectl**

```bash
# Watch agent pods come and go while a build runs
kubectl -n jenkins get pods -w

# Pod stuck Pending? Look at events: quota, node selector, image pull
kubectl -n jenkins describe pod <agent-pod>
kubectl -n jenkins get events --sort-by=.lastTimestamp | tail -20

# The jnlp container shows connection problems to the controller
kubectl -n jenkins logs <agent-pod> -c jnlp

# Get a shell in the build container while the build is paused or retained
kubectl -n jenkins exec -it <agent-pod> -c node -- bash

# Controller logs (plugin errors, JCasC failures at startup)
kubectl -n jenkins logs statefulset/jenkins -c jenkins --tail=200
```

Typical failures: pod `Pending` (no capacity, ResourceQuota, missing image pull secret), `jnlp` cannot reach the controller (wrong Jenkins URL/tunnel, NetworkPolicy), `OOMKilled` in the `node` container (raise memory limit, check `NODE_OPTIONS=--max-old-space-size`), and permission errors because containers run with different UIDs on the shared workspace (align `runAsUser`/`fsGroup` as in the pod YAML). Set `podRetention onFailure()` temporarily to inspect a failed pod. More kubectl techniques: [[Kubernetes Operations - Logs, Debugging & Scheduling]].

> [!info]+ 🆕 2025–2026
> - **Java 21 minimum**: Jenkins **LTS 2.555.1 (2026-04-15)** requires **Java 21 or newer**; Java 17 is no longer supported (weekly line since 2.545, Jan 2026). **Java 25** support arrived with weekly **2.545** and **LTS 2.541.1 (2026-01-21)**. The requirement covers controller, agents and CLI, not the JDK your builds use (that comes from your container images). Current LTS at time of writing: **2.568.3 (2026-09-02)**, Java 21 and 25.
> - **UI modernisation**: LTS **2.492.1 (2025-02-05)** introduced the **Command Palette** and more UI refresh; LTS **2.516.1 (2025-07-23)** redesigned the header and navigation; 2.528.x and 2.568.1 (2026-07-08) continued refining dialogs and system pages. LTS **2.541.1** added **Content Security Policy** enforcement support, which can break old plugins that use inline scripts.
> - **Other LTS changes**: 2.541.1 switched to a new GPG signing key for RPM/DEB repositories (update your repo key); 2.555.2 (2026-05-13) ships an MSI signed by "LF Open Source, LLC"; 2.568.1 dropped Windows Server 2019 controller images.
> - **Blue Ocean** is marked deprecated on plugins.jenkins.io (maintenance only); use **Pipeline Graph View** or Stage View.
> - **Kaniko archived**: Google archived `GoogleContainerTools/kaniko` on **2025-06-03**; **Chainguard** announced a maintained fork on **2025-06-05**.
> - **Security advisories**: core advisories on 2025-03-05, 2025-04-02, 2025-09-17, 2025-12-10, 2026-02-18, 2026-03-18, 2026-06-10, 2026-08-05 and 2026-09-02. Frequently hit plugins in 2025–2026: **Script Security** (2026-04-29, 06-24, 09-02, 09-16), **Git client** (2025-09-03, 2025-12-10, 2026-06-24), **Credentials Binding** (2025-07-09, 2026-04-29, 2026-05-27), GitHub Branch Source, LDAP/Active Directory. Plan for at least monthly plugin updates.
> - **Scanner supply chain**: on **2026-03-19** the Trivy release v0.69.4, the `trivy-action` and `setup-trivy` GitHub Actions and Docker Hub tags were compromised with a credential stealer. Pin scanner images **by digest** in pod templates, exactly like any other dependency. See [[Supply Chain Security]].

## Pitfalls

> [!warning] Common pitfalls
> - **Builds on the controller** (executors > 0 on the built-in node): any build can read every credential.
> - **Pipeline-level `timeout` with an `input` gate**: either the approval is killed after 30 minutes or the CI part is unbounded. Put timeouts on stages.
> - **Top-level `agent` plus `input`**: the pod and executor sit idle for hours waiting for approval.
> - **Groovy-interpolated secrets** (`sh "curl -u ${TOKEN}"`) instead of shell variables.
> - **Plugin sprawl and click-ops**: 200 plugins installed through the UI, no one knows which are used; upgrades become scary so nobody does them. Pin plugins in an image, configure with JCasC.
> - **Rebuilding per environment**, especially Next.js with env-specific `NEXT_PUBLIC_*` values: what you tested is not what you ship.
> - **Docker socket mounts / privileged DinD** for image builds: root on the node.
> - **`@Library('...@main')`** in every repo: one bad library commit breaks all pipelines at once.
> - **Heavy logic in Groovy** (parsing JSON, loops over big data) runs on the controller's CPU in the CPS interpreter. Push work into `sh` steps on the agent.
> - **`retry` around tests** hides flakiness instead of fixing it.
> - **`kubectl apply` from Jenkins** with cluster-admin credentials instead of GitOps: CI becomes the most privileged system you own.

## Senior perspective

- **Treat Jenkins as production infrastructure.** It has credentials to the registry and the GitOps repo. Upgrades on a schedule (LTS every ~4 weeks, plugins with it), a staging controller for testing upgrades, alerting on queue length and executor starvation, and restore drills for `JENKINS_HOME`.
- **Make the controller disposable.** JCasC + plugin image + Git-hosted Jenkinsfiles and libraries means a new controller can be rebuilt from Git in minutes; only history and credentials need backup. If you can't do that, you have a pet.
- **Keep Jenkins' blast radius small.** Jenkins writes to Git and the registry; Argo CD, which lives in the cluster, applies. Agents have no Kubernetes RBAC. Credentials are folder-scoped. Replay and script approval are restricted.
- **Standardise via a shared library, not copy-paste.** One versioned `buildNodeService` means security fixes (e.g. pinning Trivy by digest after March 2026) land in all four apps with one PR.
- **Supply chain.** Build once, scan the exact artifact, push, deploy by digest; generate an SBOM; sign images (cosign) and have the cluster verify signatures. See [[Supply Chain Security]].
- **Controller scale limits.** One controller handles a lot, but thousands of jobs and heavy Groovy make it a bottleneck. Options: split controllers per business unit, move heavy logic to agents, or (commercially) CloudBees CI's managed controllers.
- **Know when to leave.** If you are on GitHub, have no on-prem constraint and a small platform team, GitHub Actions or Buildkite may cost less in total. Migrations go stage by stage: keep the same container images and scripts so the CI engine is just an orchestrator.
- Questions a senior asks in review: What happens if this job's credentials leak? Can a PR from a fork run this code with secrets? How long is the PR feedback loop, and where does the time go? Who can approve prod and where is that recorded? When did we last restore `JENKINS_HOME`?

## Interview questions

> [!question]- Controller vs agent vs executor?
> The controller is the Jenkins server: UI, config, queue, scheduling and plugins. Agents are the machines or pods that run builds and connect to the controller. An executor is one build slot on a node. In a secure setup the controller has zero executors and all builds run on agents, typically ephemeral Kubernetes pods.

> [!question]- Declarative vs Scripted pipeline?
> Declarative (`pipeline { ... }`) is a structured, validated subset with fixed sections (`agent`, `stages`, `post`, `when`, `options`), better errors, Restart from Stage and visual editor support. Scripted (`node { ... }`) is plain Groovy with full flexibility. Use Declarative, drop into `script { }` or a shared library for the rare complex logic.

> [!question]- Why should builds not run on the Jenkins controller?
> A build there runs as the Jenkins process and can read `JENKINS_HOME`, including the encryption keys and all credentials, and modify job configs. A malicious PR or compromised dependency would own your CI. It also steals CPU/memory from the scheduler and UI. Set 0 executors and use agents.

> [!question]- How do you pass a secret to a shell step safely?
> Bind it with `credentials()` in `environment` or `withCredentials` around the smallest block, and reference it as a shell variable inside a single-quoted `sh '...'` string so Groovy never interpolates it. Never echo it, never `set -x` around it, and scope the credential to the folder that needs it.

> [!question]- How would you share one pipeline across many repos?
> A versioned Jenkins Shared Library with a `vars/buildNodeService.groovy` global step that contains the whole Declarative pipeline and takes a config map. Each repo's Jenkinsfile is two lines calling it. Pin to a tag, review library changes like production code, and test with PR-pinned versions or JenkinsPipelineUnit.

> [!question]- How do you build container images inside Kubernetes agents without Docker-in-Docker?
> Use a rootless daemonless builder: BuildKit rootless (`buildctl-daemonless.sh`) or Buildah. Both avoid mounting the Docker socket or running privileged containers. Kaniko was a popular option but Google archived it in June 2025; Chainguard maintains a fork. Use registry caching to keep builds fast.

> [!question]- How does Jenkins hand over to Argo CD in a GitOps setup?
> Jenkins pushes the image tagged with the git SHA, then commits the new tag into the environment overlay of the GitOps repo (`kustomize edit set image`) and pushes. Argo CD sees the Git change and syncs the cluster. Jenkins never has cluster credentials; rollbacks are Git reverts. Prod promotion is a separate commit behind an approval (Jenkins `input` or a PR in the GitOps repo).

> [!question]- Your Jenkins controller was lost. How do you recover?
> Redeploy the Helm chart from Git; the plugin image and JCasC restore configuration, and multibranch/organization folders rediscover jobs from GitHub. Restore the `JENKINS_HOME` PVC snapshot for credentials (with `secrets/` keys) and build history. If credentials come from an external store, even that step is optional. Then verify webhooks and agent connectivity.

## Related
- [[DevOps Index]] · [[Kubernetes Delivery Platform - End to End]] · [[CI-CD Pipelines]]
- [[Argo CD in Practice]] · [[Argo Rollouts - Canary & Blue-Green]] · [[GitOps - Argo CD & Flux]] · [[Helm & Kustomize]]
- [[Deploying Node.js Apps to Kubernetes]] · [[Deploying Next.js Micro-frontends to Kubernetes]] · [[Kubernetes Operations - Logs, Debugging & Scheduling]]
- [[Supply Chain Security]] · [[Secrets Management]] · [[Dockerfile Best Practices & Image Security]] · [[Docker Basics]]
- [[Kubernetes]] · [[Kubernetes Networking - Services, Ingress & Gateway API]] · [[Monorepos]] · [[Next.js App Router Fundamentals]]
- [[Platform Engineering & Internal Developer Platforms]] · [[Grafana & the LGTM Stack]]

## Sources
- [Jenkins User Handbook: Pipeline](https://www.jenkins.io/doc/book/pipeline/) · [Pipeline Syntax (Declarative)](https://www.jenkins.io/doc/book/pipeline/syntax/)
- [Jenkins: Using credentials / string interpolation](https://www.jenkins.io/doc/book/pipeline/jenkinsfile/#handling-credentials)
- [Jenkins: Extending with Shared Libraries](https://www.jenkins.io/doc/book/pipeline/shared-libraries/)
- [Jenkins: Branches and Pull Requests (Multibranch)](https://www.jenkins.io/doc/book/pipeline/multibranch/)
- [Jenkins: Controller isolation](https://www.jenkins.io/doc/book/security/controller-isolation/) · [Securing Jenkins](https://www.jenkins.io/doc/book/security/) · [In-process Script Approval](https://www.jenkins.io/doc/book/managing/script-approval/)
- [Jenkins: Installing on Kubernetes](https://www.jenkins.io/doc/book/installing/kubernetes/) · [jenkinsci/helm-charts (README, values.yaml)](https://github.com/jenkinsci/helm-charts/tree/main/charts/jenkins)
- [Configuration as Code plugin](https://plugins.jenkins.io/configuration-as-code/) · [Kubernetes plugin](https://plugins.jenkins.io/kubernetes/) · [Matrix Authorization Strategy](https://plugins.jenkins.io/matrix-auth/)
- [Jenkins LTS changelog](https://www.jenkins.io/changelog-stable/) · [Java support policy](https://www.jenkins.io/doc/book/platform-information/support-policy-java/)
- [Jenkins security advisories](https://www.jenkins.io/security/advisories/)
- [Blue Ocean plugin (deprecation notice)](https://plugins.jenkins.io/blueocean/) · [Pipeline Graph View plugin](https://plugins.jenkins.io/pipeline-graph-view/)
- [BuildKit rootless docs](https://github.com/moby/buildkit/blob/master/docs/rootless.md) · [BuildKit Kubernetes examples](https://github.com/moby/buildkit/tree/master/examples/kubernetes)
- [Buildah](https://buildah.io/) · [skopeo](https://github.com/containers/skopeo) · [Trivy docs](https://trivy.dev/)
- [GoogleContainerTools/kaniko (archived)](https://github.com/GoogleContainerTools/kaniko) · [Chainguard: "Fork Yeah: We're Bringing Kaniko Back" (2025-06-05)](https://www.chainguard.dev/unchained/fork-yeah-were-bringing-kaniko-back) · [chainguard-forks/kaniko](https://github.com/chainguard-forks/kaniko)
- [Trivy GHSA-69fq-xp46-6x23: supply chain compromise (2026-03)](https://github.com/aquasecurity/trivy/security/advisories/GHSA-69fq-xp46-6x23) · [Microsoft Security Blog on the Trivy compromise (2026-03-24)](https://www.microsoft.com/en-us/security/blog/2026/03/24/detecting-investigating-defending-against-trivy-supply-chain-compromise/)
- [Next.js docs: Environment variables (build-time inlining of NEXT_PUBLIC_)](https://nextjs.org/docs/app/guides/environment-variables)
- [Node.js test runner reporters](https://nodejs.org/api/test.html#test-reporters)
- [Kustomize: `kustomize edit set image`](https://kubectl.docs.kubernetes.io/references/kustomize/cmd/edit/)
