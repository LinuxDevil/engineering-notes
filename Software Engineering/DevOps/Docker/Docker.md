---
title: "Docker"
tags: [devops, devops/docker, index]
aliases: [Docker Home, Docker Index, Docker MOC]
updated: 2026-09-27
---
# Docker

<Note>
**TL;DR**

Hub note for everything Docker and containers. Start with [Docker Basics](/Software%20Engineering/DevOps/Docker/Docker%20Basics) (images, containers, CLI, first Dockerfile), then [Dockerfile Best Practices & Image Security](/Software%20Engineering/DevOps/Docker/Dockerfile%20Best%20Practices%20%26%20Image%20Security) (multi-stage, caching, non-root, scanning), then [Docker Compose](/Software%20Engineering/DevOps/Docker/Docker%20Compose) (multi-container apps). Orchestration continues in [Kubernetes](/Software%20Engineering/DevOps/Kubernetes/Kubernetes).
</Note>

{/* diagram source: /diagrams/diagram-165.mmd */}
![diagram](/diagrams/diagram-165.svg)

## Notes in order
| Level | Note | What you learn |
|---|---|---|
| Beginner | [Docker Basics](/Software%20Engineering/DevOps/Docker/Docker%20Basics) | Image vs container, lifecycle, `run/exec/logs/stop`, Dockerfile, cache, tags, ports |
| Beginner | [Containers vs VMs vs Serverless](/Software%20Engineering/DevOps/Containers%20vs%20VMs%20vs%20Serverless) | Where containers fit among compute models |
| Intermediate | [Docker Compose](/Software%20Engineering/DevOps/Docker/Docker%20Compose) | `compose.yaml`, service DNS, volumes, healthchecks, watch |
| Intermediate → Senior | [Dockerfile Best Practices & Image Security](/Software%20Engineering/DevOps/Docker/Dockerfile%20Best%20Practices%20%26%20Image%20Security) | Multi-stage builds, BuildKit cache/secret mounts, `.dockerignore`, non-root, SBOM, signing, scanning |

## Cheat sheet
```bash
docker build -t app:1.0 .            # build
docker run -d -p 8080:8080 app:1.0   # run detached, publish port
docker ps / docker ps -a             # list
docker logs -f <id>                  # logs
docker exec -it <id> sh              # shell
docker stop <id> && docker rm <id>   # stop + remove
docker image ls / docker rmi <img>   # images
docker system df / docker system prune
docker compose up -d / down          # multi-container
```

<Info>
**🆕 2025–2026 in Docker**

- 2025-11-10 · Docker Engine 29: containerd image store default → [Docker Basics](/Software%20Engineering/DevOps/Docker/Docker%20Basics)
- 2025-12-02 · Compose v5 "Mont Blanc": builds via Bake, Compose SDK → [Docker Compose](/Software%20Engineering/DevOps/Docker/Docker%20Compose)
- 2025-12-17 · Docker Hardened Images free and open source → [Dockerfile Best Practices & Image Security](/Software%20Engineering/DevOps/Docker/Dockerfile%20Best%20Practices%20%26%20Image%20Security)
- 2025-08/09 · Bitnami public image catalog reduced (`bitnamilegacy`) → [Dockerfile Best Practices & Image Security](/Software%20Engineering/DevOps/Docker/Dockerfile%20Best%20Practices%20%26%20Image%20Security)
</Info>

## Related
- [DevOps Index](/Software%20Engineering/DevOps/DevOps%20Index) · [Kubernetes](/Software%20Engineering/DevOps/Kubernetes/Kubernetes) · [CI-CD Pipelines](/Software%20Engineering/DevOps/CI-CD%20Pipelines) · [Microservices Architecture](/Software%20Engineering/DevOps/Microservices%20Architecture)
- Original course study plan: [Microservices Study Plan (original)](/Software%20Engineering/DevOps/_Archive/Microservices%20Study%20Plan%20(original))
