---
tags: [devops, devops/docker]
aliases: [Compose, docker-compose, compose.yaml]
updated: 2026-09-27
---
# Docker Compose

> [!tldr] TL;DR
> Docker Compose describes a **multi-container application** in one `compose.yaml` (services, networks, volumes, env, healthchecks) and runs it with `docker compose up`. Services on the same Compose network reach each other **by service name** via built-in DNS. It is the standard tool for local development and CI integration tests, and fine for single-host deployments; for multi-node production use [[Kubernetes]] or a managed container platform.

## The idea

Without Compose you would type long `docker run` commands with `--network`, `-v`, `-e` and `-p` for every container, in the right order. Compose makes that declarative and versionable.

```mermaid
flowchart LR
  subgraph compose network "myapp_default"
    web[web :3000] -->|http://api:8080| api[api :8080]
    api -->|postgres://db:5432| db[(db: postgres)]
    api -->|redis://cache:6379| cache[(cache: redis)]
  end
  browser((Browser)) -->|localhost:3000| web
```

## A realistic `compose.yaml`

```yaml
# compose.yaml  (the top-level "version:" key is obsolete; omit it)
name: shop

services:
  api:
    build:
      context: ./api
      target: runtime            # multi-stage target
    image: ghcr.io/acme/shop-api:dev
    ports:
      - "8080:8080"               # host:container
    environment:
      DATABASE_URL: postgres://shop:${DB_PASSWORD}@db:5432/shop
      REDIS_URL: redis://cache:6379
    env_file: .env                # values for ${...} interpolation also come from .env
    depends_on:
      db:
        condition: service_healthy
      cache:
        condition: service_started
    restart: unless-stopped
    develop:
      watch:                      # `docker compose watch`: sync or rebuild on file changes
        - action: sync
          path: ./api/src
          target: /app/src
        - action: rebuild
          path: ./api/package-lock.json

  db:
    image: postgres:17
    environment:
      POSTGRES_USER: shop
      POSTGRES_PASSWORD: ${DB_PASSWORD}
      POSTGRES_DB: shop
    volumes:
      - db-data:/var/lib/postgresql/data
    healthcheck:
      test: ["CMD-SHELL", "pg_isready -U shop"]
      interval: 5s
      timeout: 3s
      retries: 10

  cache:
    image: redis:8-alpine

  migrate:
    build: ./api
    command: ["npm", "run", "migrate"]
    depends_on:
      db: { condition: service_healthy }
    profiles: ["tools"]           # only runs with --profile tools

volumes:
  db-data:
```

## Everyday commands

```bash
docker compose up -d                 # build if needed, create network/volumes, start in background
docker compose up --build            # force rebuild
docker compose watch                 # hot sync/rebuild using the develop.watch rules
docker compose ps                    # status (incl. health)
docker compose logs -f api           # follow logs of one service
docker compose exec api sh           # shell into a running service
docker compose run --rm migrate      # one-off task container
docker compose --profile tools up    # include profiled services
docker compose down                  # stop and remove containers + network
docker compose down -v               # ... and named volumes (destroys data!)
docker compose config                # print the fully resolved config (great for debugging)
```

## How it works

- **Project**: the directory name (or `name:`) prefixes resources: `shop-api-1`, network `shop_default`, volume `shop_db-data`.
- **Networking**: Compose creates a user-defined bridge network; each service gets a DNS name equal to its service name. Inside containers use `db:5432`, never `localhost:5432` (localhost is the container itself).
- **`depends_on`** controls start order only. Without `condition: service_healthy` it does not wait for the database to be ready. Apps should still retry connections.
- **Overrides**: `compose.override.yaml` is merged automatically; use `-f compose.yaml -f compose.prod.yaml` for environment variants, or `include:` to compose multiple files.
- **Scaling**: `docker compose up --scale worker=3` (no fixed host ports on scaled services).

## Compose vs Kubernetes

| | Compose | Kubernetes |
|---|---|---|
| Scope | Single host | Cluster of nodes |
| Self-healing | `restart:` policy only | Controllers reschedule Pods across nodes |
| Rolling updates, autoscaling | No | Yes |
| Learning curve | Low | High |
| Best for | Local dev, CI integration tests, small single-VM apps | Production at scale |

Converters such as Kompose exist, but most teams keep Compose for dev and write Kubernetes manifests or Helm charts for prod. Skaffold or Tilt give a Compose-like inner loop on Kubernetes (see [[Kubernetes Basics]]).

> [!info]+ 🆕 2025–2026
> - **Compose v5.0.0 "Mont Blanc" (2025-12-02)**: the internal builder was removed and **builds are delegated to Docker Bake**, and Compose can now be used officially as a **Go SDK** embedded in other tools. The team skipped v3 and v4 to avoid confusion with the obsolete `version: "3.x"` file formats. Also added: OCI and Git remote resources, `--wait` for `start`.
> - Since **Compose v2.38 (2025)** a top-level `models:` element declares AI models (e.g. `llm: { model: ai/smollm2 }`) served by Docker Model Runner, so an app and its local LLM start together.

## Pitfalls

> [!warning] Common pitfalls
> - Using `localhost` to reach another service.
> - Relying on `depends_on` without a healthcheck and getting "connection refused" on first boot.
> - `docker compose down -v` in the wrong directory wiping a local database.
> - Committing `.env` with real secrets. Commit a `.env.example` instead.
> - Bind-mounting `node_modules` from a macOS/Windows host into a Linux container (native modules break). Use a named volume for `node_modules` or `develop.watch`.

## Senior perspective

- Keep the dev environment reproducible: a new engineer should run `docker compose up` and have a working stack in minutes. That is a real onboarding metric.
- Use the same images in CI integration tests (or Testcontainers) as in dev to avoid "works on my machine".
- Compose on a single VM is a legitimate production choice for small internal tools; be explicit about what you give up (HA, rolling deploys, autoscaling).

## Interview questions

> [!question]- How do containers in the same Compose project find each other?
> Compose creates a user-defined network with embedded DNS. Each service is resolvable by its service name (and aliases), so `api` connects to `db:5432`.

> [!question]- Does `depends_on` wait until the database is ready?
> Not by default: it only orders container start. With `condition: service_healthy` and a healthcheck on the dependency, Compose waits for it to be healthy. Apps should still implement retries.

> [!question]- Named volume vs bind mount?
> A named volume is managed by Docker (stored in Docker's data dir) and is ideal for database data. A bind mount maps a host path into the container, ideal for live-editing source code in development.

> [!question]- When would you not use Compose in production?
> When you need multiple hosts, automatic failover, rolling or canary deploys, autoscaling or fine-grained RBAC. Then use Kubernetes, ECS, Cloud Run or similar.

## Related
- [[Docker]] · [[Docker Basics]] · [[Dockerfile Best Practices & Image Security]] · [[DevOps Index]]
- [[Kubernetes]] · [[Containers vs VMs vs Serverless]] · [[Microservices Architecture]]

## Sources
- [Docker docs: Compose overview](https://docs.docker.com/compose/)
- [Compose file reference (Compose Specification)](https://docs.docker.com/reference/compose-file/)
- [Docker docs: Compose Watch](https://docs.docker.com/compose/how-tos/file-watch/)
- [Docker docs: Control startup order](https://docs.docker.com/compose/how-tos/startup-order/)
- [docker/compose release v5.0.0 "Mont Blanc"](https://github.com/docker/compose/releases/tag/v5.0.0)
- [Docker docs: Use AI models in Compose](https://docs.docker.com/ai/compose/models-and-compose/)
