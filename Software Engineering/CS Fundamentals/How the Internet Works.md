---
title: "How the Internet Works"
tags: [cs-fundamentals, cs-fundamentals/networking]
aliases: [Internet Basics, What happens when you type a URL, DNS Resolution, TCP vs UDP, IP Addressing]
updated: 2026-09-27
---
# How the Internet Works

<Note>
**TL;DR**

The internet is a network of networks that move **packets** using **IP** addresses; routers forward each packet hop by hop, and autonomous systems exchange routes with **BGP**. On top of IP, **TCP** gives a reliable ordered byte stream (handshake, retransmits, flow and congestion control) and **UDP** gives fast fire-and-forget datagrams (used by DNS, video calls, and QUIC/HTTP/3). **DNS** translates names into IP addresses through a hierarchy of resolvers and nameservers. Typing a URL triggers DNS → TCP/QUIC connection → TLS → HTTP request → server work → response → browser rendering.
</Note>

## Packets, IP and routing
- Data is split into **packets** with headers (source/destination IP, TTL, protocol).
- **IPv4**: 32-bit (`203.0.113.7`), ~4.3 B addresses, exhausted → **NAT** lets many private devices (`192.168.x.x`, `10.x.x.x`) share one public IP. **IPv6**: 128-bit (`2001:db8::1`), no NAT needed.
- **CIDR** notation: `10.0.0.0/16` = 65,536 addresses sharing the first 16 bits. Subnets are how cloud VPCs are carved up.
- **Routers** forward packets based on routing tables (longest-prefix match). IP is **best effort**: packets can be lost, duplicated, reordered.
- **BGP**: networks (Autonomous Systems like ISPs, clouds) announce which IP prefixes they can reach. Misconfigured BGP can take large services offline (Facebook, Oct 2021).
- **Anycast**: one IP announced from many locations; routing picks the nearest (CDNs, public DNS like 1.1.1.1, 8.8.8.8).

## TCP vs UDP
| | TCP | UDP |
|---|---|---|
| Connection | 3-way handshake | None |
| Reliability | Acks, retransmission | None (app decides) |
| Ordering | In-order byte stream | Datagrams, any order |
| Flow/congestion control | Yes | No (app/QUIC implements) |
| Header | 20+ bytes | 8 bytes |
| Head-of-line blocking | Yes: one lost segment stalls the stream | No |
| Used by | HTTP/1.1, HTTP/2, SSH, databases, SMTP | DNS, VoIP, games, video calls (WebRTC), **QUIC → HTTP/3** |

Ports identify applications on a host (443 HTTPS, 53 DNS, 5432 Postgres). A connection is a 4-tuple: (src IP, src port, dst IP, dst port). Details of the handshake and congestion control: [Networking - OSI, TCP-IP & Protocols](/Software%20Engineering/CS%20Fundamentals/Networking%20-%20OSI%2C%20TCP-IP%20%26%20Protocols).

## DNS resolution
{/* diagram source: /diagrams/diagram-098.mmd */}
![diagram](/diagrams/diagram-098.svg)
| Record | Purpose |
|---|---|
| A / AAAA | Name → IPv4 / IPv6 |
| CNAME | Alias to another name |
| MX | Mail servers |
| TXT | Verification, SPF/DKIM/DMARC |
| NS | Delegation to nameservers |
| SRV / HTTPS / SVCB | Service locations; HTTPS record can advertise HTTP/3 and ECH |

- **TTL** controls caching; low TTLs help failover but increase query load. Some clients ignore TTLs.
- Security: **DNSSEC** signs records; **DoH/DoT** (DNS over HTTPS/TLS) encrypt queries from the client to the resolver.

```bash
dig +trace www.example.com        # walk root → TLD → authoritative
dig AAAA example.com @1.1.1.1     # query a specific resolver
nslookup example.com
```

## What happens when you type a URL
{/* diagram source: /diagrams/diagram-099.mmd */}
![diagram](/diagrams/diagram-099.svg)
1. The browser parses the URL; checks HSTS preload list to upgrade to HTTPS.
2. **DNS**: browser cache → OS cache → recursive resolver → root/TLD/authoritative.
3. **Transport**: TCP handshake (1 RTT), or QUIC which combines transport + TLS (1 RTT, 0-RTT on resumption).
4. **TLS**: server proves identity with a certificate chain; keys are agreed ([Cryptography Basics for Engineers](/Software%20Engineering/CS%20Fundamentals/Cryptography%20Basics%20for%20Engineers)).
5. **HTTP request** goes to the nearest CDN PoP ([CDN & Edge Computing](/Software%20Engineering/System%20Design/CDN%20%26%20Edge%20Computing)), maybe served from cache; otherwise to the origin load balancer, app servers, caches, databases ([Scalability Fundamentals](/Software%20Engineering/System%20Design/Scalability%20Fundamentals)).
6. **Response** streams back; the browser renders it: [How Browsers Render - Critical Rendering Path](/Software%20Engineering/Frontend/03%20Browser%20%26%20Web%20Platform/How%20Browsers%20Render%20-%20Critical%20Rendering%20Path). Keep-alive connections are reused for subsequent requests.

<Info>
**Latency budget for a cold visit (user in Europe, origin in US East, RTT 100 ms)**

DNS (~20–100 ms) + TCP (100) + TLS 1.3 (100) + request/response (100) + server time (50) ≈ **400 ms** before the first byte of HTML. A CDN PoP 10 ms away turns the network part into ~40 ms. This is why CDNs, connection reuse, and HTTP/3 matter.
</Info>

## Senior perspective
- **DNS is a dependency of everything**; bad records or resolver failures take down services even when servers are healthy. Keep sane TTLs, monitor resolution, and understand client-side caching during failovers.
- MTU and fragmentation issues (VPNs, tunnels) cause mysterious hangs on large packets.
- NAT and connection tracking tables fill up under many short-lived connections; use pooling ([Connection Pooling](/Software%20Engineering/Databases/Database%20Round/Connection%20Pooling)).
- Know your tools: `dig`, `traceroute`/`mtr`, `curl -v`, `ss`/`netstat`, `tcpdump`/Wireshark.

<Info>
**🆕 2025–2026**

- DNS failures remain a top outage cause: on **2025-10-20** DynamoDB's regional endpoint in AWS us-east-1 resolved to an **empty DNS record** because of a race in AWS's DNS automation, cascading across AWS and customer apps for hours. ([AWS summary](https://aws.amazon.com/message/101925/))
- Cloudflare Radar's 2025 review: ~21% of requests to Cloudflare used **HTTP/3 (QUIC over UDP)**, and **52%** of human web traffic used post-quantum key exchange by year end. ([Cloudflare](https://blog.cloudflare.com/radar-2025-year-in-review/))
</Info>

## Interview questions
<Accordion title="What happens when you type a URL and press Enter?">
URL parsing and HSTS check, DNS resolution through caches and the resolver hierarchy, TCP (or QUIC) connection, TLS handshake with certificate validation, HTTP request (often to a CDN), server-side processing through LB/app/DB, the response, and the browser's parse/layout/paint plus subresource requests.
</Accordion>

<Accordion title="TCP vs UDP: when would you use UDP?">
When low latency matters more than reliability or when the application implements its own reliability: DNS queries, real-time voice/video, games, and QUIC (which builds reliable, multiplexed streams on UDP to avoid TCP's head-of-line blocking and ossification).
</Accordion>

<Accordion title="How does DNS resolution work?">
The stub resolver asks a recursive resolver, which (on cache miss) asks a root server, gets referred to the TLD servers, then to the domain's authoritative nameservers, which return the record. Results are cached at each level for the TTL.
</Accordion>

<Accordion title="What is NAT and why does it exist?">
Network Address Translation rewrites private source addresses/ports to a shared public IP, letting many devices share one IPv4 address. It exists because of IPv4 exhaustion; it complicates peer-to-peer connectivity (hence STUN/TURN in WebRTC).
</Accordion>

<Accordion title="What is anycast?">
Announcing the same IP prefix from many locations so BGP routes each client to the topologically nearest one. Used by CDNs and DNS for latency and DDoS resilience.
</Accordion>

## Related
- [CS Fundamentals Index](/Software%20Engineering/CS%20Fundamentals/CS%20Fundamentals%20Index)
- [Networking - OSI, TCP-IP & Protocols](/Software%20Engineering/CS%20Fundamentals/Networking%20-%20OSI%2C%20TCP-IP%20%26%20Protocols) · [Cryptography Basics for Engineers](/Software%20Engineering/CS%20Fundamentals/Cryptography%20Basics%20for%20Engineers) · [Networking - HTTP, Caching & Real-time](/Software%20Engineering/Frontend/03%20Browser%20%26%20Web%20Platform/Networking%20-%20HTTP%2C%20Caching%20%26%20Real-time) · [How Browsers Render - Critical Rendering Path](/Software%20Engineering/Frontend/03%20Browser%20%26%20Web%20Platform/How%20Browsers%20Render%20-%20Critical%20Rendering%20Path)
- [CDN & Edge Computing](/Software%20Engineering/System%20Design/CDN%20%26%20Edge%20Computing) · [Scalability Fundamentals](/Software%20Engineering/System%20Design/Scalability%20Fundamentals) · [System Design Index](/Software%20Engineering/System%20Design/System%20Design%20Index)

## Sources
- Kurose & Ross, *Computer Networking: A Top-Down Approach*
- [RFC 791 (IPv4)](https://www.rfc-editor.org/rfc/rfc791) · [RFC 8200 (IPv6)](https://www.rfc-editor.org/rfc/rfc8200) · [RFC 9293 (TCP)](https://www.rfc-editor.org/rfc/rfc9293) · [RFC 768 (UDP)](https://www.rfc-editor.org/rfc/rfc768)
- [RFC 1034/1035 (DNS)](https://www.rfc-editor.org/rfc/rfc1034)
- [Cloudflare Learning Center: What is DNS?](https://www.cloudflare.com/learning/dns/what-is-dns/)
- [alex/what-happens-when (GitHub)](https://github.com/alex/what-happens-when)
- [Meta: More details about the October 4 (2021) outage](https://engineering.fb.com/2021/10/05/networking-traffic/outage-details/)
- [AWS: DynamoDB disruption summary, Oct 2025](https://aws.amazon.com/message/101925/)
