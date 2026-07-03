# zer0space

> **This is a sanitized public showcase.** Real IPs, Tailscale addresses, domain names, and credentials have been replaced with placeholders (`<NODE_IP>`, `<TAILNET>`, `<DOMAIN>`, etc.). The production configuration lives in a private repository.

Self-hosted homelab on ThinkCentre M910q nodes — Docker Swarm, GitOps, zero exposed origin.

[![Nodes](https://img.shields.io/badge/nodes-7-blue)](#node-inventory)
[![OS](https://img.shields.io/badge/OS-Debian%2012-informational)](#node-inventory)
[![Orchestration](https://img.shields.io/badge/orchestration-Docker%20Swarm-blue)](#architecture)
[![Deployed by](https://img.shields.io/badge/deployed%20by-Portainer%20GitOps-blueviolet)](#gitops-workflow)
[![Tunnel](https://img.shields.io/badge/tunnel-Cloudflare-orange)](#network-security-model)
[![VPN](https://img.shields.io/badge/VPN-Tailscale-5B5BFF)](#network-security-model)

---

## What is this?

**zer0space** is a self-hosted homelab running on bare-metal Debian 12 across 7 ThinkCentre M910q mini PCs. The entire setup is orchestrated with Docker Swarm, managed as code in a private repository, and deployed via Portainer GitOps — making the repo the single source of truth for the whole infrastructure.

Public services are exposed exclusively through a **Cloudflare Tunnel** (direct routing, no reverse proxy), so the origin IP is never visible to the internet. Remote management happens over **Tailscale**. A custom **Dashboard** (Node.js + SQLite) gives a unified view of the cluster with live per-node metrics.

---

## Architecture

```
                        ┌────────────────────────────────────────┐
                        │              INTERNET                   │
                        └──────────────────┬─────────────────────┘
                                           │ HTTPS only
                                           │ (origin IP never exposed)
                        ┌──────────────────▼─────────────────────┐
                        │           Cloudflare Edge               │
                        │    DNS · Proxy · Zero Trust Tunnel      │
                        │    Access (GitHub login: <TAILNET>)     │
                        └──────────────────┬─────────────────────┘
                                           │
                        ┌──────────────────▼─────────────────────┐
                        │   cloudflared  (Tunnel daemon in Swarm) │
                        │   direct routing per subdomain →        │
                        │   http://<stack>_<service>:<port>       │
                        └──┬──────────────────────────────────┬───┘
                           │                                  │
          ┌────────────────▼────────┐        ┌───────────────▼────────────────┐
          │   Manager Nodes (×3)    │        │   Worker Nodes (×4)             │
          │                         │        │                                 │
          │  zs-node-01  10.0.0.1   │        │  zs-worker-01  10.0.0.4 (32 GB)│
          │  zs-node-02  10.0.0.2   │        │  zs-worker-02  10.0.0.5  (8 GB)│
          │  zs-node-03  10.0.0.3   │        │  zs-worker-03  10.0.0.6  (8 GB)│
          │                         │        │  zs-worker-04  10.0.0.7  (8 GB)│
          │  Swarm quorum           │        │  Workloads + NFS storage        │
          └────────────────┬────────┘        └─────────────────────────────────┘
                           │
          ┌────────────────▼────────────────────────────────────────┐
          │                  Management Plane                        │
          │                                                          │
          │   Tailscale mesh VPN  ──  SSH  ──  Portainer GitOps     │
          │   (no open ports, NAT traversal)                         │
          └──────────────────────────────────────────────────────────┘
```

> **Note on Traefik:** Traefik was evaluated but rejected — Traefik v3.x sends Docker API version 1.24 despite the socket-proxy being configured for 1.44+, causing the proxy to reject all requests. `cloudflared` routes directly to service DNS names in the overlay network instead. See [docs/journey.md](docs/journey.md) for the full story.

---

## Node Inventory

| Hostname | Role | IP (placeholder) | RAM | Disk | Function |
|---|---|---|---|---|---|
| zs-node-01 | Manager | 10.0.0.1 | 8 GB | 256 GB | Swarm control · Dashboard |
| zs-node-02 | Manager | 10.0.0.2 | 8 GB | 256 GB | Swarm control · DNS (Technitium) |
| zs-node-03 | Manager | 10.0.0.3 | 8 GB | 256 GB | Swarm control · cloudflared |
| zs-worker-01 | Worker | 10.0.0.4 | 32 GB | 1 TB | Jellyfin · NFS storage |
| zs-worker-02 | Worker | 10.0.0.5 | 8 GB | 256 GB | General workloads |
| zs-worker-03 | Worker | 10.0.0.6 | 8 GB | 256 GB | General workloads (Stirling-PDF) |
| zs-worker-04 | Worker | 10.0.0.7 | 8 GB | 256 GB | General workloads |

All nodes run **Debian 12** bare-metal (no Proxmox, no VMs). Docker Engine installed via `get.docker.com`.

---

## Tech Stack

| Layer | Component | Purpose |
|---|---|---|
| Orchestration | Docker Swarm | Cluster management, 3 managers + 4 workers |
| GitOps | Portainer CE | Deploys stacks from the private repo automatically |
| Public Tunnel | Cloudflare Tunnel (`cloudflared`) | Hides origin IP, direct service routing, no port forwarding |
| Auth | Cloudflare Access | Outer auth layer — GitHub login protects all public services |
| DNS / DHCP | Technitium | Self-hosted DNS + DHCP on zs-node-02, standalone, host networking |
| VPN | Tailscale | Management access, no open ports, NAT traversal |
| Dashboard | Custom (Node.js + SQLite) | Unified service overview, live node metrics via Glances |
| Media | Jellyfin (planned) | Streaming — VPN only, never via Cloudflare (ToS §2.8) |
| Storage | NFS (planned) | Shared volume from zs-worker-01 |
| Passwords | Vault (built into Dashboard) | Native AES-256-GCM password manager, no separate stack |
| PDF Tools | Stirling-PDF | Stateless PDF toolkit (merge / convert / edit) |
| CI/CD | GitHub Actions | Builds dashboard Docker image on every push |

---

## Dashboard

The custom dashboard is the primary operations interface, built with Node.js/Express + SQLite. It is the only service in this cluster built from source — everything else uses public images.

**Feature overview:**

| Feature | Details |
|---|---|
| Views | Home · AI · Cloud · Media · Vault · Settings (SPA, no page reload) |
| Live metrics | CPU · RAM · disk · network per node via [Glances](https://github.com/nicolargo/glances) REST API |
| Swarm status | Node count · service count · task health via `docker-socket-proxy` (read-only) |
| Service list | Configurable service grid stored in SQLite |
| Authentication | bcrypt (cost 12) + session cookie · brute-force protection (5 attempts / 10 min → 15 min lockout) |
| Roles | `admin` (full access) · `viewer` (read-only + own theme) |
| Vault | Per-user AES-256-GCM password manager · key derived from login password via PBKDF2 (never stored) · CSRF-protected · rate-limited |
| Theme system | 8 presets + free color picker, stored per user in SQLite |
| Background | Generative canvas animation or custom image upload (JPG/PNG/WebP, max 10 MB) |
| Security headers | Helmet · strict CSP · no `unsafe-inline` on scripts · `frame-ancestors: none` |

Image: `ghcr.io/sige0/zer0space-dashboard:latest` — built automatically by GitHub Actions.

---

## Network Security Model

```
PUBLIC INTERNET
──────────────────────────────────────────────────────────────────
Browser  ──►  Cloudflare Edge (TLS)  ──►  Cloudflare Access
                                               │  (GitHub login: <TAILNET>)
                                        cloudflared daemon
                                               │  (overlay network)
                                        Service container

✓  Origin IP is never exposed to the internet
✓  All inbound TLS terminates at Cloudflare edge
✓  No router port forwarding required
✓  No request reaches a service without passing Cloudflare Access first

MANAGEMENT (private, VPN-only)
──────────────────────────────────────────────────────────────────
Admin  ──►  Tailscale mesh  ──►  Manager node  ──►  Swarm API

✓  SSH, Docker API, and Portainer reachable only inside Tailscale
✓  Jellyfin served over Tailscale only (Cloudflare ToS §2.8)
✓  No WireGuard/UDP through the Cloudflare Tunnel (HTTP/HTTPS only)
```

---

## GitOps Workflow

```
┌─────────────────┐    commit + push    ┌──────────────────┐
│  Edit config    │ ──────────────────► │   GitHub Repo    │
│  (local / AI)   │                     │  (private, prod) │
└─────────────────┘                     └────────┬─────────┘
                                                 │
                                        Portainer watches repo
                                                 │
                                        ┌────────▼─────────┐
                                        │  Docker Swarm     │
                                        │  Cluster (×7)     │
                                        └──────────────────┘
```

Changes are made by editing files in the private repo, committing, and pushing. Portainer picks up the changes and runs `docker stack deploy` automatically. No manual SSH sessions, no config drift.

The dashboard image is built automatically by a GitHub Actions workflow on every push to `stacks/dashboard/**` and pushed to GHCR.

---

## Repository Structure (private repo layout)

```
zer0space/
├── .github/workflows/
│   └── dashboard.yml          Builds + pushes dashboard image to GHCR
├── docs/
│   ├── dashboard.md           Dashboard architecture + API reference
│   ├── roadmap.md             Phase plan with status
│   └── security.md            Security review log
├── stacks/
│   ├── cloudflared/           Cloudflare Tunnel daemon (Swarm stack)
│   ├── dashboard/             Custom dashboard (Dockerfile + Compose + src/)
│   ├── homepage/              Homepage dashboard (gethomepage/homepage)
│   ├── jellyfin/              Jellyfin media server (zs-worker-01)
│   ├── portainer/             Portainer CE + agent
│   ├── technitium/            Technitium DNS+DHCP (standalone, host networking)
│   ├── traefik/               Traefik (evaluated, rejected — not deployed)
│   └── stirling-pdf.yml       Stirling-PDF toolkit (stateless, zs-worker-03)
└── CLAUDE.md                  Project context for Claude Code
```

---

## Roadmap

| Phase | Name | Status |
|---|---|---|
| 0 | Preparation | ✅ Done |
| 1 | Base OS | ✅ Done |
| 2 | Swarm + Portainer | ✅ Done |
| 3 | Reverse Proxy + Public Access | ✅ Done (cloudflared) |
| 4 | Dashboard + Auth | ✅ Done (v2.3 + security hardening) |
| 5 | DNS in Docker | ✅ Done |
| 6 | Streaming + Storage | ⬜ Planned |
| 7 | Polish | ⬜ Planned |

---

## Further Reading

- [docs/journey.md](docs/journey.md) — build story: what went wrong, what was learned, and why specific decisions were made
- [examples/](examples/) — sanitized, generic Docker Compose templates

---

Built and maintained with [Claude Code](https://claude.ai/code).
