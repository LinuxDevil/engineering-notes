---
title: "Nginx"
tags: [backend, backend/infrastructure]
aliases: [NGINX, Reverse Proxy, nginx.conf, Nginx Load Balancing, Web Server]
updated: 2026-09-27
---
# Nginx

<Note>
**TL;DR**

**Nginx** is an event-driven web server and **reverse proxy**: it sits in front of your application servers and handles TLS, static files, load balancing, caching, compression, rate limiting and WebSocket/gRPC proxying, so your app only does app logic. Config is a tree of `http → server → location` blocks. Know `proxy_pass` and its headers, upstreams, `try_files`, timeouts and buffering, and always `nginx -t` before `reload`. In 2026 the stable line is **1.30.x** (1.30.0 on 2026-04-14), and Kubernetes' community **ingress-nginx** controller was **retired in March 2026**.
</Note>

## What is a reverse proxy

A reverse proxy is **the application that sits in front of back-end applications and forwards client (e.g. browser) requests to those applications**.

![Untitled 32](/Attachments/Untitled%2032.png)
![Untitled 33](/Attachments/Untitled%2033.png)

- A **forward proxy** acts for *clients* (corporate proxy, VPN-like): the server sees the proxy.
- A **reverse proxy** acts for *servers*: the client talks to the proxy and doesn't know which backend served it.

### How it works

When a client sends a request, the reverse proxy intercepts it, picks an appropriate backend, forwards the request, receives the response and relays it back to the client (possibly transformed, compressed or cached).

```text
                   +--------------+
                   |   Client     |
                   +--------------+
                          |
                          v
                   +--------------+
                   |    Nginx     |   TLS, routing, caching, limits
                   +--------------+
                          |
               +----------+----------+
               |                     |
               v                     v
        +--------------+     +--------------+
        |   Server 1   |     |   Server 2   |
        +--------------+     +--------------+
```

### Benefits

- **Load balancing**: distributes requests across servers so no single one is overloaded; removes failed servers.
- **Caching**: stores frequently requested responses, reducing backend load and latency.
- **TLS termination**: handles HTTPS encryption/decryption (historically "SSL") so app servers don't have to.
- **Security**: hides backend topology, blocks malicious clients, limits request rates and sizes, absorbs slow clients (Slowloris) so app workers aren't tied up. It helps against DDoS but is not a full DDoS solution on its own (that's a CDN/scrubbing service).
- **Compression, HTTP/2 and HTTP/3** towards clients, even if the app only speaks HTTP/1.1.

## Why Nginx is fast: the architecture

- One **master** process (reads config, manages workers) and N **worker** processes (usually one per CPU core: `worker_processes auto;`).
- Each worker runs an **event loop** with non-blocking I/O (epoll/kqueue) and handles thousands of connections, unlike the old thread-per-connection model (Apache prefork). Same idea as the Node.js event loop ([Node.js Index](/Software%20Engineering/Backend/The%20Hard%20Parts%20of%20Servers%20%26%20Node.js/Node.js%20Index)).
- **Reload without downtime**: `nginx -s reload` starts new workers with the new config; old workers finish in-flight requests and exit.

## Getting started

```bash
# Debian/Ubuntu
sudo apt update && sudo apt install nginx
# RHEL/CentOS/Fedora
sudo dnf install nginx        # (yum on older systems)

sudo systemctl start nginx
sudo systemctl enable nginx    # start at boot

sudo nginx -t                  # test config syntax: ALWAYS before reload
sudo systemctl reload nginx    # apply config without dropping connections
sudo systemctl status nginx
sudo nginx -T                  # dump the full effective config
```

- Main config: `/etc/nginx/nginx.conf`. Site configs: `/etc/nginx/conf.d/*.conf` (or `sites-available` + symlinks in `sites-enabled` on Debian).
- Default web root is often `/var/www/html`. Logs: `/var/log/nginx/access.log` and `error.log`.

## Configuration structure

```nginx
user  nginx;
worker_processes auto;

events { worker_connections 4096; }

http {
  include       mime.types;
  sendfile      on;
  gzip          on;
  log_format main '$remote_addr "$request" $status $body_bytes_sent rt=$request_time urt=$upstream_response_time';

  server {                       # a virtual host
    listen 80;
    server_name example.com;
    root /var/www/example.com;

    location / {                 # URL matching
      try_files $uri $uri/ =404;
    }
  }
}
```

The server block listens on port 80 for `example.com`; `root` points to the site's files; `location` blocks decide how each URL is handled. `try_files` tries the file, then a directory, else 404.

**Location matching order**: exact `=` → longest prefix with `^~` → regex `~`/`~*` in file order → longest plain prefix.

## Worked example: production reverse proxy for an app

```nginx
upstream app {
  least_conn;                              # or round-robin (default), ip_hash, hash $key
  server 10.0.0.11:3000 max_fails=3 fail_timeout=10s;
  server 10.0.0.12:3000 max_fails=3 fail_timeout=10s;
  keepalive 64;                            # reuse connections to backends
}

server {                                   # redirect HTTP → HTTPS
  listen 80;
  server_name api.example.com;
  return 301 https://$host$request_uri;
}

server {
  listen 443 ssl;
  listen 443 quic reuseport;               # HTTP/3
  http2 on;
  server_name api.example.com;

  ssl_certificate     /etc/letsencrypt/live/api.example.com/fullchain.pem;
  ssl_certificate_key /etc/letsencrypt/live/api.example.com/privkey.pem;
  ssl_protocols TLSv1.2 TLSv1.3;
  add_header Alt-Svc 'h3=":443"; ma=86400' always;
  add_header Strict-Transport-Security "max-age=63072000" always;

  client_max_body_size 10m;                # upload limit (default 1m → 413)

  location /static/ {
    root /var/www/app;
    expires 30d;
    add_header Cache-Control "public, immutable";
  }

  location / {
    proxy_pass http://app;
    proxy_http_version 1.1;
    proxy_set_header Connection "";        # needed for upstream keepalive
    proxy_set_header Host $host;
    proxy_set_header X-Real-IP $remote_addr;
    proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
    proxy_set_header X-Forwarded-Proto $scheme;
    proxy_set_header X-Request-Id $request_id;
    proxy_connect_timeout 2s;
    proxy_read_timeout 30s;
    proxy_next_upstream error timeout http_502 http_503;   # retry only safe failures
  }
}
```

Other common blocks:

```nginx
# gRPC
location /orders.v1.OrderService/ { grpc_pass grpc://orders:50051; }

# Rate limiting (see Rate Limiting & Throttling)
limit_req_zone $binary_remote_addr zone=perip:10m rate=10r/s;
location /login { limit_req zone=perip burst=5 nodelay; proxy_pass http://app; }

# Micro-caching an API response for 1s absorbs spikes
proxy_cache_path /var/cache/nginx keys_zone=api:50m max_size=1g inactive=10m;
location /api/public/ { proxy_cache api; proxy_cache_valid 200 1s; proxy_cache_use_stale updating error timeout; proxy_pass http://app; }

# SPA fallback
location / { try_files $uri /index.html; }
```

WebSocket and SSE proxying: see [Webhooks, Polling, SSE & WebSockets](/Software%20Engineering/APIs%20%26%20Auth/Webhooks%2C%20Polling%2C%20SSE%20%26%20WebSockets).

## Common pitfalls

<Warning>
**Things that bite**

- **Trailing slash in `proxy_pass`**: `location /api/ { proxy_pass http://app/; }` strips `/api/`; without the slash the full URI is passed. Know which you want.
- Forgetting `proxy_set_header Host` and `X-Forwarded-*`: the app builds wrong redirect URLs or sees every client as the proxy IP.
- `add_header` in a `location` **replaces** all `add_header`s inherited from the parent (security headers vanish). Repeat them or use `include`.
- `if` inside `location` is famously surprising ("if is evil"); prefer `map`, `try_files`, `return`.
- Default `proxy_read_timeout 60s` cuts long requests, SSE and WebSockets.
- `client_max_body_size` default of 1 MB → `413` on uploads.
- Response buffering on (default) breaks streaming; set `proxy_buffering off` or `X-Accel-Buffering: no` for SSE/LLM streams.
- Reloading without `nginx -t`.
</Warning>

<Info>
**🆕 2025–2026**

- **2025-03:** "IngressNightmare" (CVE-2025-1974 and related CVEs) in the Kubernetes **ingress-nginx** controller allowed unauthenticated remote code execution from inside the cluster network.
- **2025-11-11:** Kubernetes SIG Network announced the **retirement of ingress-nginx**; **2026-01-29** Steering and Security Response Committees reiterated it; **March 2026:** retired, with no further releases or security fixes. It ran in roughly half of cloud-native environments. Migrate to **Gateway API** implementations (the `ingress2gateway` tool helps) or another controller. Note: this is the *community* `kubernetes/ingress-nginx` project, not NGINX itself; F5's separate NGINX Ingress Controller continues.
- **2026-04-14:** **nginx 1.30.0 stable** released, bringing the 1.29.x mainline features: **103 Early Hints**, **HTTP/2 to upstreams**, **Encrypted Client Hello (ECH)**, **sticky sessions for upstreams** in open source, **Multipath TCP**, and `proxy_http_version` defaulting to **1.1 with keep-alive**. Patch releases through 1.30.5 (2026-09-15) include security fixes, so keep up to date.
- **HTTP/3 adoption**: Cloudflare's 2025 Year in Review reports ~21% of requests over HTTP/3, ~50% HTTP/2, ~29% HTTP/1.x, roughly flat versus 2024. Nginx has supported QUIC/HTTP/3 since 1.25.0 (2023).
</Info>

## Senior perspective

- **Nginx vs alternatives**: Envoy (dynamic xDS config, service mesh, first-class gRPC/observability), HAProxy (L4/L7 load balancing performance), Caddy (automatic HTTPS, simple config), Traefik (container-native), cloud LBs (ALB, Cloud Load Balancing). Nginx remains a great default for a fixed topology; Envoy wins when config must change dynamically. See [API Gateways & Reverse Proxies](/Software%20Engineering/Backend/API%20Gateways%20%26%20Reverse%20Proxies).
- Tune: `worker_connections`, file descriptor limits, upstream `keepalive`, buffer sizes; watch `$upstream_response_time` vs `$request_time` to tell app slowness from client slowness.
- Treat config as code: templated, reviewed, tested (`nginx -t` in CI), deployed with reload, with access logs in structured JSON.
- Security: keep it patched (edge software is attacked first), minimal modules, `server_tokens off`, TLS config from Mozilla's generator.
- Review questions: *What are the timeouts at each hop? Is the client IP trustworthy (`set_real_ip_from`)? Which requests can be retried on the next upstream? What's cached, and is `Vary`/auth respected?*

## Interview questions

<Accordion title="What is a reverse proxy and why use one?">
A server in front of your backends that receives client requests and forwards them. It centralises TLS termination, load balancing, caching, compression, rate limiting and security filtering, hides backend topology and lets you scale or change backends without clients noticing.
</Accordion>

<Accordion title="Forward vs reverse proxy?">
A forward proxy acts on behalf of clients to reach many servers (the server sees the proxy). A reverse proxy acts on behalf of servers to receive traffic from many clients (the client sees the proxy).
</Accordion>

<Accordion title="Why is Nginx able to handle many concurrent connections?">
An event-driven, non-blocking architecture: a few worker processes each run an event loop over epoll/kqueue and multiplex thousands of connections, instead of a thread or process per connection.
</Accordion>

<Accordion title="What load-balancing methods does Nginx support?">
Round robin (default, optionally weighted), least_conn, ip_hash, generic hash (e.g. consistent hashing on a key), random with two choices, and (since 1.30 in open source) sticky sessions; plus passive health checks via max_fails/fail_timeout.
</Accordion>

<Accordion title="What happened to ingress-nginx?">
The community Kubernetes ingress-nginx controller was retired in March 2026 after years of maintainer shortage and serious 2025 CVEs (IngressNightmare). No more fixes are released; users should migrate to Gateway API implementations or another maintained controller.
</Accordion>

<Accordion title="Why might SSE or streaming responses arrive late through Nginx?">
Proxy buffering (on by default) collects the upstream response before sending; disable it for that location (`proxy_buffering off`) or send `X-Accel-Buffering: no`, and raise `proxy_read_timeout`.
</Accordion>

## Related
- [Backend Index](/Software%20Engineering/Backend/Backend%20Index) · [APIs & Auth Index](/Software%20Engineering/APIs%20%26%20Auth/APIs%20%26%20Auth%20Index)
- [API Gateways & Reverse Proxies](/Software%20Engineering/Backend/API%20Gateways%20%26%20Reverse%20Proxies) · [Rate Limiting & Throttling](/Software%20Engineering/APIs%20%26%20Auth/Rate%20Limiting%20%26%20Throttling) · [Webhooks, Polling, SSE & WebSockets](/Software%20Engineering/APIs%20%26%20Auth/Webhooks%2C%20Polling%2C%20SSE%20%26%20WebSockets) · [HTTP Headers vs Body - Where Data Belongs](/Software%20Engineering/APIs%20%26%20Auth/HTTP%20Headers%20vs%20Body%20-%20Where%20Data%20Belongs) · [gRPC vs REST](/Software%20Engineering/APIs%20%26%20Auth/gRPC%20vs%20REST)
- [Node.js Index](/Software%20Engineering/Backend/The%20Hard%20Parts%20of%20Servers%20%26%20Node.js/Node.js%20Index) · [Microservices Architecture](/Software%20Engineering/DevOps/Microservices%20Architecture) · [Networking - HTTP, Caching & Real-time](/Software%20Engineering/Frontend/03%20Browser%20%26%20Web%20Platform/Networking%20-%20HTTP%2C%20Caching%20%26%20Real-time)
- Other domains: [Kubernetes Networking - Services, Ingress & Gateway API](/Software%20Engineering/DevOps/Kubernetes/Kubernetes%20Networking%20-%20Services%2C%20Ingress%20%26%20Gateway%20API) · [CDN & Edge Computing](/Software%20Engineering/System%20Design/CDN%20%26%20Edge%20Computing) · [Caching Strategies](/Software%20Engineering/System%20Design/Caching%20Strategies)

## Sources
- [nginx.org documentation](https://nginx.org/en/docs/) · [Beginner's guide](https://nginx.org/en/docs/beginners_guide.html) · [ngx_http_proxy_module](https://nginx.org/en/docs/http/ngx_http_proxy_module.html) · [HTTP/3 (QUIC) support](https://nginx.org/en/docs/quic.html)
- [nginx 1.30 CHANGES](https://nginx.org/en/CHANGES-1.30) · [NGINX Community: 1.30.0 stable released](https://community.nginx.org/t/nginx-1-30-0-stable-version-released/9417)
- [Kubernetes blog: Ingress NGINX Retirement (2025-11-11)](https://www.kubernetes.io/blog/2025/11/11/ingress-nginx-retirement/) · [Statement from Steering and SRC (2026-01-29)](https://www.kubernetes.io/blog/2026/01/29/ingress-nginx-statement/)
- [Wiz: IngressNightmare (CVE-2025-1974)](https://www.wiz.io/blog/ingress-nginx-kubernetes-vulnerabilities)
- [Cloudflare Radar 2025 Year in Review](https://blog.cloudflare.com/radar-2025-year-in-review/)
- [Nginx wiki: If is Evil](https://github.com/nginxinc/nginx-wiki/blob/master/source/start/topics/depth/ifisevil.rst)
- [Mozilla SSL Configuration Generator](https://ssl-config.mozilla.org/)
