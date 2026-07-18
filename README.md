# zer0space

> **This is a sanitized public showcase.** Real IPs, Tailscale addresses, domain names, and credentials have been replaced with placeholders (`<NODE_IP>`, `<TAILNET>`, `<DOMAIN>`, etc.). The production configuration lives in a private repository.

Self-hosted homelab — Docker Swarm, GitOps, zero exposed origin.

[![Nodes](https://img.shields.io/badge/nodes-9-blue)](#node-inventory)
[![OS](https://img.shields.io/badge/OS-Debian%2012-informational)](#node-inventory)
[![Orchestration](https://img.shields.io/badge/orchestration-Docker%20Swarm-blue)](#architecture)
[![Deployed by](https://img.shields.io/badge/deployed%20by-Portainer%20GitOps-blueviolet)](#gitops-workflow)
[![Tunnel](https://img.shields.io/badge/tunnel-Cloudflare-orange)](#network-security-model)
[![VPN](https://img.shields.io/badge/VPN-Tailscale-5B5BFF)](#network-security-model)

---

## What is this?

**zer0space** is a self-hosted homelab running on bare-metal Debian 12 across 9 ThinkCentre M910q mini PCs. Seven of them form a **Docker Swarm** cluster (3 managers + 4 workers); the other two sit deliberately outside the Swarm as a dedicated stateful host and a central storage server. The whole setup is managed as code in a private repository and deployed via Portainer GitOps — making the repo the single source of truth for the whole infrastructure.

Public services are exposed exclusively through a **Cloudflare Tunnel** (direct routing, no reverse proxy), so the origin IP is never visible to the internet. Remote management happens over **Tailscale**. A custom **Dashboard** (Node.js + SQLite) gives a unified view of the cluster with live per-node metrics.

### Architecture principle: a stateless Swarm

Docker Swarm here holds **no persistent data at all**. Every service running inside the Swarm is disposable — if a Swarm node dies, the fix is to reinstall Debian, join it back to the Swarm, and let Portainer redeploy the stacks. No backup restore, no data recovery, no drama.

That's possible because persistent state lives in exactly two places, both outside the Swarm:

- **`zs-store-01`** — a dedicated NFS server. All Swarm service volumes (`/srv/nfs/swarm-data`) and the media library (`/srv/nfs/media`) live here.
- **`zs-state-01`** — a dedicated stateful host running plain Docker (not Swarm) for services that need genuinely local, low-latency disk: PostgreSQL and a standalone Portainer CE instance.

This split came out of a Portainer outage that doubled as a lesson in not letting your control plane depend on the infrastructure it controls — the full story is in [docs/journey.md](docs/journey.md).

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
          ┌────────────────▼────────┐        ┌───────────────▼─────────────────┐
          │   Manager Nodes (×3)    │        │   Worker Nodes (×4)              │
          │                         │        │                                  │
          │  zs-node-01  <NODE_IP>  │        │  zs-worker-01  <NODE_IP> (32 GB) │
          │  zs-node-02  <NODE_IP>  │        │  zs-worker-02  <NODE_IP>  (8 GB) │
          │  zs-node-03  <NODE_IP>  │        │  zs-worker-03  <NODE_IP>  (8 GB) │
          │                         │        │  zs-worker-04  <NODE_IP> (16 GB) │
          │  Swarm quorum           │        │  Stateless workloads              │
          └────────────────┬────────┘        └────────────────┬─────────────────┘
                           │                                   │
                           └─────────────────┬─────────────────┘
                                             │ NFS mounts — the only place
                                             │ Swarm workloads keep data
                        ┌────────────────────▼──────────────────────┐
                        │   zs-store-01 — NFS Storage Server          │
                        │   (bare Debian, NOT in Swarm)               │
                        │   /srv/nfs/swarm-data · /srv/nfs/media      │
                        └──────────────────────────────────────────────┘

          ┌───────────────────────────────────────────────────────────┐
          │   zs-state-01 — Stateful Host (bare Debian, NOT in Swarm)  │
          │   24 GB RAM · 450 GB disk                                  │
          │   PostgreSQL · Portainer CE (standalone, plain Docker)     │
          │   planned: Technitium DNS, Jellyfin                        │
          └────────────────────────────┬────────────────────────────────┘
                                       │ manages
                        ┌──────────────▼───────────────────────────────┐
                        │                  Management Plane              │
                        │                                                │
                        │   Tailscale mesh VPN  ──  SSH  ──  Portainer  │
                        │   GitOps  (no open ports, NAT traversal)      │
                        └────────────────────────────────────────────────┘
```

> **Note on Traefik:** Traefik was evaluated but rejected — Traefik v3.x sends Docker API version 1.24 despite the socket-proxy being configured for 1.44+, causing the proxy to reject all requests. `cloudflared` routes directly to service DNS names in the overlay network instead. See [docs/journey.md](docs/journey.md) for the full story.

---

## Node Inventory

| Hostname | Role | IP (placeholder) | RAM | Disk | Function |
|---|---|---|---|---|---|
| zs-node-01 | Swarm Manager | 10.0.0.1 | 8 GB | 256 GB | Swarm control |
| zs-node-02 | Swarm Manager | 10.0.0.2 | 8 GB | 256 GB | Swarm control |
| zs-node-03 | Swarm Manager | 10.0.0.3 | 8 GB | 256 GB | Swarm control |
| zs-worker-01 | Swarm Worker | 10.0.0.4 | 32 GB | 1 TB | Heavy workloads · nightly backup target |
| zs-worker-02 | Swarm Worker | 10.0.0.5 | 8 GB | 256 GB | General workloads |
| zs-worker-03 | Swarm Worker | 10.0.0.6 | 8 GB | 256 GB | General workloads (Stirling-PDF) |
| zs-worker-04 | Swarm Worker | 10.0.0.7 | 16 GB | 256 GB | General workloads |
| zs-state-01 | Stateful Host (**not** in Swarm) | 10.0.0.8 | 24 GB | 450 GB | PostgreSQL · Portainer CE · planned: Technitium, Jellyfin |
| zs-store-01 | Storage Server (**not** in Swarm) | 10.0.0.9 | — | — | NFS server for Swarm volumes (`/srv/nfs/swarm-data`, `/srv/nfs/media`) |

All nodes run **Debian 12** bare-metal (no Proxmox, no VMs). Docker Engine installed via `get.docker.com`. `zs-node-01/02/03` and `zs-worker-01..04` are Swarm members (7 nodes total); `zs-state-01` and `zs-store-01` run plain Docker / NFS and are intentionally kept outside the Swarm.

---

## Tech Stack

| Layer | Component | Purpose |
|---|---|---|
| Orchestration | Docker Swarm | Cluster management, 3 managers + 4 workers — stateless workloads only |
| GitOps | Portainer CE | Standalone on `zs-state-01` (**not** in Swarm), deploys stacks from the private repo automatically; lightweight Swarm agents remain stateless inside the cluster |
| Public Tunnel | Cloudflare Tunnel (`cloudflared`) | Hides origin IP, direct service routing, no port forwarding |
| Auth | Cloudflare Access | Outer auth layer — GitHub login protects all public services |
| DNS / DHCP | Technitium | Self-hosted DNS + DHCP; planned move to `zs-state-01` |
| VPN | Tailscale | Management access, no open ports, NAT traversal |
| Dashboard | Custom (Node.js + SQLite) | Unified service overview, live node metrics via Glances; data volume on NFS — fully stateless container |
| Database | PostgreSQL | Runs on `zs-state-01` (local disk, not NFS — see [docs/journey.md](docs/journey.md) for why) |
| Media | Jellyfin (planned) | Streaming on `zs-state-01` — VPN only, never via Cloudflare (ToS §2.8) |
| Streaming Frontend | Crimson (self-hosted as `crimson-zer0space`) | Metadata/discovery backend for the streaming site — backend built on [Crimson Haven's `crimson-backend`](https://github.com/crimsonhaven-to/crimson-backend), sits in front of the media library, brings no content of its own |
| Storage | NFS (`zs-store-01`) | Central storage server — backs every Swarm service volume and the media library |
| Backup | Nightly rsync/tar → NFS (`zs-worker-01`) | Cloudflare R2 offsite copy planned |
| Passwords | Vault (built into Dashboard) | Native AES-256-GCM password manager, no separate stack |
| PDF Tools | Stirling-PDF | Stateless PDF toolkit (merge / convert / edit) |
| CI/CD | GitHub Actions | Builds dashboard Docker image on every push |

---

## Dashboard

The custom dashboard is the primary operations interface, built with Node.js/Express + SQLite. It is the only service in this cluster built from source — everything else uses public images. Its SQLite data file lives on an NFS-backed volume (`zs-store-01`), not a local bind-mount — the container (and its Vault data) survives the Swarm node it happens to be scheduled on going away entirely.

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
Admin  ──►  Tailscale mesh  ──►  Manager node / zs-state-01  ──►  Swarm API

✓  SSH, Docker API, and Portainer reachable only inside Tailscale
✓  Jellyfin served over Tailscale only (Cloudflare ToS §2.8)
✓  No WireGuard/UDP through the Cloudflare Tunnel (HTTP/HTTPS only)

STORAGE (internal LAN only, never routed)
──────────────────────────────────────────────────────────────────
Swarm node  ──►  NFS mount  ──►  zs-store-01 (/srv/nfs/swarm-data, /srv/nfs/media)

✓  NFS traffic never leaves the local network
✓  zs-worker-01 pulls nightly backups from NFS — R2 offsite copy planned
```

---

## GitOps Workflow

```
┌─────────────────┐    commit + push    ┌──────────────────┐
│  Edit config    │ ──────────────────► │   GitHub Repo    │
│  (local / AI)   │                     │  (private, prod) │
└─────────────────┘                     └────────┬─────────┘
                                                 │
                                Portainer (standalone, zs-state-01)
                                        watches repo
                                                 │
                                        ┌────────▼──────────┐
                                        │  Docker Swarm      │
                                        │  Cluster (×7 nodes)│
                                        └────────┬───────────┘
                                                 │ volumes
                                        ┌────────▼──────────┐
                                        │  zs-store-01 (NFS) │
                                        └────────────────────┘
```

Changes are made by editing files in the private repo, committing, and pushing. Portainer — now running standalone on `zs-state-01`, outside the Swarm it manages — picks up the changes and runs `docker stack deploy` against the Swarm's remote API. No manual SSH sessions, no config drift.

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
├── stacks/                     Docker Swarm stacks (stateless, deployed via Portainer GitOps)
│   ├── backup/                 Nightly backup job → zs-worker-01 → NFS (R2 offsite planned)
│   ├── cloudflared/            Cloudflare Tunnel daemon (Swarm stack)
│   ├── dashboard/               Custom dashboard (Dockerfile + Compose + src/), NFS-backed volume
│   ├── homepage/                Homepage dashboard (gethomepage/homepage)
│   ├── jellyfin/                 Jellyfin media server (planned, moving to zs-state-01)
│   ├── portainer-agent/        Stateless Swarm-side Portainer agent (server lives on zs-state-01)
│   ├── technitium/               Technitium DNS+DHCP (planned, moving to zs-state-01)
│   ├── traefik/                  Traefik (evaluated, rejected — not deployed)
│   └── stirling-pdf.yml        Stirling-PDF toolkit (stateless, zs-worker-03)
├── state-host/                  Plain Docker Compose files for zs-state-01 (NOT Swarm-managed)
│   ├── postgres/                 PostgreSQL, local disk
│   └── portainer/                Portainer CE server (standalone)
└── CLAUDE.md                   Project context for Claude Code
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
| 6 | Stateless Swarm Refactor | ✅ Done (NFS storage server + dedicated stateful host, 7 → 9 devices) |
| 7 | Streaming | ⬜ Planned (Jellyfin on zs-state-01) |
| 8 | Offsite Backup | ⬜ Planned (Cloudflare R2) |
| 9 | Polish | ⬜ Planned |

---

## Further Reading

- [docs/journey.md](docs/journey.md) — build story: what went wrong, what was learned, and why specific decisions were made
- [examples/](examples/) — sanitized, generic Docker Compose templates

---

## Acknowledgments

The streaming site's backend is built on [**Crimson**](https://github.com/crimsonhaven-to) by Crimson Haven — specifically [`crimson-backend`](https://github.com/crimsonhaven-to/crimson-backend), a FastAPI metadata/discovery engine. It ships with no content sources of its own; this instance (`crimson-zer0space`) points it at this homelab's own media library. Props to Crimson Haven for the framework.

---

Built and maintained with [Claude Code](https://claude.ai/code).
