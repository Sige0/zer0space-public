# zer0space

> **This is a sanitized public showcase.** Real IPs, Tailscale addresses, domain names, and credentials have been replaced with placeholders (`<NODE_IP>`, `<TAILNET>`, `<DOMAIN>`, etc.). The production configuration lives in a private repository.

Self-hosted homelab — Docker Swarm, GitOps, zero exposed origin.

[![Nodes](https://img.shields.io/badge/nodes-9-blue)](#node-inventory)
[![OS](https://img.shields.io/badge/OS-Debian%2012%20%2F%2013-informational)](#node-inventory)
[![Orchestration](https://img.shields.io/badge/orchestration-Docker%20Swarm-blue)](#architecture)
[![Deployed by](https://img.shields.io/badge/deployed%20by-Portainer%20GitOps-blueviolet)](#gitops-workflow)
[![Tunnel](https://img.shields.io/badge/tunnel-Cloudflare-orange)](#network-security-model)
[![VPN](https://img.shields.io/badge/VPN-Tailscale-5B5BFF)](#network-security-model)

---

## What is this?

**zer0space** is a self-hosted homelab running on bare-metal Debian across 9 mini PCs (ThinkCentre M910q, plus newer replacement hardware). The topology is split into three parts: a **7-node Docker Swarm** (fully stateless) for orchestration, one dedicated **stateful host** (`zs-state-01`, outside the Swarm) carrying every stateful service — PostgreSQL, Portainer — as standalone Docker containers, and one central **storage server** (`zs-store-01`, outside the Swarm) that holds every persistent volume over the network. The Swarm side is managed as code in the private repository and deployed via Portainer GitOps — making that repo the single source of truth for every stateless service.

Public services are exposed exclusively through a **Cloudflare Tunnel** (direct routing, no reverse proxy), so the origin IP is never visible to the internet. Remote management happens over **Tailscale**. A custom **Dashboard** (Node.js, data in PostgreSQL) gives a unified view of the cluster with live per-node metrics.

### Architecture principle: a stateless Swarm

Docker Swarm here holds **no persistent data at all**. Every service running inside the Swarm is disposable — if a Swarm node dies, the fix is to reinstall Debian, join it back to the Swarm, and let Portainer redeploy the stacks. No backup restore, no data recovery, no drama.

That's possible because persistent state lives in exactly two places, both outside the Swarm:

- **`zs-store-01`** — a dedicated storage server. It exports `/srv/nfs/swarm-data` over NFS, mounted on every Swarm node as `/mnt/storage`; every stack's persistent data goes there instead of a local bind-mount.
- **`zs-state-01`** — a dedicated stateful host running plain Docker (not Swarm) for services that need genuinely local, low-latency disk: PostgreSQL and a standalone Portainer CE instance.

This split came out of a Portainer outage that doubled as a lesson in not letting your control plane depend on the infrastructure it controls — the full story, including how `zs-state-01` started life as an ordinary Swarm worker, is in [docs/journey.md](docs/journey.md).

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
                        │     DNS · Proxy · Zero Trust Tunnel     │
                        │   Access (Email OTP + Authenticator)    │
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
          └────────────────┬────────┘        └───────────────────────────────────┘
                           │
   ┌───────────────────────┼────────────────────────┐
   │                       │                         │
┌──▼──────────────────┐ ┌──▼───────────────────────┐ ┌▼──────────────────────────┐
│  Management Plane    │ │ Stateful Host            │ │ Storage Server             │
│                      │ │ (NOT in Swarm)           │ │ (NOT in Swarm)             │
│  Tailscale · SSH ·   │ │ zs-state-01              │ │ zs-store-01                │
│  Portainer GitOps    │ │ <NODE_IP>                │ │ <NODE_IP>                  │
│  (no open ports)     │ │ PostgreSQL · Portainer   │ │ central network storage    │
│                      │ │                          │ │ for every Swarm stack      │
└──────────────────────┘ └──────────────────────────┘ └─────────────────────────────┘
```

> **Note on Traefik:** Traefik was evaluated but rejected — Traefik v3.x sends Docker API version 1.24 despite the socket-proxy being configured for 1.44+, which the proxy rejects. `cloudflared` routes directly to service DNS names in the overlay network instead. The Traefik stack has since been removed from the repo entirely. See [docs/journey.md](docs/journey.md) for the full story.

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
                                        │  zs-store-01       │
                                        │  (network storage) │
                                        └────────────────────┘
```

Changes are made by editing files in the private repo, committing, and pushing. Portainer — running standalone on `zs-state-01`, outside the Swarm it manages — picks up the changes and runs `docker stack deploy` against the Swarm's remote API. No manual SSH sessions, no config drift.

The dashboard image is built automatically by a GitHub Actions workflow (`.github/workflows/dashboard.yml`) on every push to `stacks/dashboard/**` and pushed to `ghcr.io/sige0/zer0space-dashboard:latest`.

---

## Node Inventory

**Swarm cluster (7 nodes, fully stateless):**

| Hostname | Role | IP (placeholder) | RAM | Disk | Function |
|---|---|---|---|---|---|
| zs-node-01 | Manager | `<NODE_IP>` | 8 GB | 256 GB | Swarm control · Dashboard |
| zs-node-02 | Manager | `<NODE_IP>` | 8 GB | 256 GB | Swarm control |
| zs-node-03 | Manager | `<NODE_IP>` | 8 GB | 256 GB | Swarm control · Proxy (`cloudflared`) |
| zs-worker-01 | Worker | `<NODE_IP>` | 32 GB | 1 TB | Heavy workloads · nightly backup target |
| zs-worker-02 | Worker | `<NODE_IP>` | 8 GB | 256 GB | General workloads |
| zs-worker-03 | Worker | `<NODE_IP>` | 8 GB | 256 GB | General workloads (Stirling-PDF) |
| zs-worker-04 | Worker | `<NODE_IP>` | 16 GB | 256 GB | General workloads (replacement hardware — see below) |

**Standalone hosts (NOT in the Swarm, plain Docker):**

| Hostname | Role | IP (placeholder) | RAM | Disk | Function |
|---|---|---|---|---|---|
| zs-state-01 | Stateful Host | `<NODE_IP>` | 24 GB | 450 GB | PostgreSQL · Portainer CE |
| zs-store-01 | Storage Server | `<NODE_IP>` | 16 GB | 256 GB | Central network storage backend for every Swarm stack |

`zs-state-01` was originally a Swarm worker — pulled out of the Swarm, repurposed as the dedicated stateful host, and backfilled with a fresh worker so the cluster stayed at 7 nodes. Its own compose files (PostgreSQL, Portainer) are kept locally on the host, not in the GitOps repo — those services are managed by hand, deliberately, since Portainer itself lives there.

`zs-store-01` exports `/srv/nfs/swarm-data` over NFS, mounted on every Swarm node as `/mnt/storage`. Stacks mount their persistent data from there instead of using local bind-mounts, so no Swarm node holds state and a service can be scheduled anywhere. The rule for new services: **persistent data goes to `/mnt/storage/<service>/`, never to a local path.**

All nodes run bare-metal Debian (no Proxmox, no VMs) — existing nodes on Debian 12, new and replacement nodes on Debian 13. Docker Engine installed via `get.docker.com`. All nodes are enrolled in a private Tailscale network for management access, with SSH access via a dedicated non-root user in the `docker` group.

---

## Tech Stack

| Layer | Component | Purpose |
|---|---|---|
| Orchestration | Docker Swarm | Cluster management, 3 managers + 4 workers — fully stateless |
| Stateful Host | `zs-state-01` | Standalone Docker (**not** in Swarm) — carries every stateful service |
| Storage Host | `zs-store-01` | Standalone data server (**not** in Swarm) — central network storage backend for all stacks |
| Database | PostgreSQL | Network database on `zs-state-01` — holds the dashboard's users, services, settings, and vault |
| GitOps | Portainer CE | Standalone on `zs-state-01` (**not** in Swarm), manages the Swarm via lightweight, stateless agents |
| Public Tunnel | Cloudflare Tunnel (`cloudflared`) | Hides origin IP, direct service routing, no port forwarding |
| Auth | Cloudflare Access | Outer auth layer — Email OTP + Authenticator (TOTP) MFA protects every public service |
| Backup (offsite) | Cloudflare R2 (planned) | S3-compatible external backup copy, completing the 3-2-1 rule |
| VPN | Tailscale | Management access, no open ports, NAT traversal |
| Dashboard | Custom (Node.js) | Unified service overview, live node metrics via Glances — stateless container, all data in PostgreSQL |
| Streaming Frontend | Crimson (self-hosted as `crimson-zer0space`) | Metadata/discovery backend for the streaming site — built on [Crimson Haven's `crimson-backend`](https://github.com/crimsonhaven-to/crimson-backend), brings no content of its own |
| Storage | NFS (`zs-store-01`) | `/srv/nfs/swarm-data`, mounted on every node as `/mnt/storage` — every stack's persistent volume lives there |
| Passwords | Vault (built into Dashboard) | Native AES-256-GCM password manager, no separate stack |
| Tools | Stirling-PDF | Stateless PDF toolkit (merge / convert / edit), `zs-worker-03` |
| CI/CD | GitHub Actions | Builds the dashboard image and pushes it to GHCR |

Traefik was evaluated and rejected (Swarm API version mismatch — see above) and its stack removed from the repo. Authentik was evaluated as a self-hosted identity provider and is no longer prioritized — Cloudflare Access already covers the perimeter MFA requirement without adding another stateful service to operate.

---

## Dashboard

The custom dashboard is the primary operations interface, built with Node.js/Express. It is the only service in this cluster built from source — everything else uses public images. Its data — users, services, settings, Vault entries — lives in PostgreSQL on `zs-state-01`, not in a local file: the dashboard container itself is fully stateless and can be scheduled on (or rescheduled to) any Swarm node without carrying any state along with it. It was migrated off an earlier SQLite-backed design via a one-off migration script; see [docs/journey.md](docs/journey.md) for why.

**Feature overview:**

| Feature | Details |
|---|---|
| Views | Home · AI · Cloud · Vault · Settings (SPA, no page reload) |
| Live metrics | CPU · RAM · disk · network per node via [Glances](https://github.com/nicolargo/glances) REST API |
| Swarm status | Node count · service count · task health via `docker-socket-proxy` (read-only) |
| Service list | Configurable service grid stored in PostgreSQL |
| Authentication | bcrypt (cost 12) + session cookie · brute-force protection (5 attempts / 10 min → 15 min lockout) |
| Roles | `admin` (full access) · `viewer` (read-only + own theme) |
| Vault | Per-user AES-256-GCM password manager · key derived from login password via PBKDF2 (never stored) · CSRF-protected · rate-limited |
| Theme system | 8 presets + free color picker, stored per user in PostgreSQL |
| Background | Generative canvas animation or custom image upload (JPG/PNG/WebP, max 10 MB) |
| Security headers | Helmet · strict CSP · no `unsafe-inline` on scripts · `frame-ancestors: none` |

Image: `ghcr.io/sige0/zer0space-dashboard:latest` — built automatically by GitHub Actions.

See `docs/dashboard.md` and `docs/security.md` in the private repo for the full architecture, API reference, and security hardening log.

---

## Network Security Model

```
PUBLIC INTERNET
──────────────────────────────────────────────────────────────────
Browser  ──►  Cloudflare Edge (TLS)  ──►  Cloudflare Access
                                               │  (Email OTP + Authenticator MFA)
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
✓  No WireGuard/UDP through the Cloudflare Tunnel (HTTP/HTTPS only)

STORAGE (internal LAN only, never routed)
──────────────────────────────────────────────────────────────────
Swarm node  ──►  network mount  ──►  zs-store-01

✓  Storage traffic never leaves the local network
✓  zs-worker-01 pulls nightly backups from storage — R2 offsite copy planned
```

---

## Repository Structure (private repo layout)

```
zer0space/
├── .github/
│   └── workflows/
│       ├── dashboard.yml          Builds + pushes dashboard image to GHCR on push
│       └── dashboard-image.yml    (legacy — superseded by dashboard.yml)
├── CLAUDE.md                      Project context (auto-read by Claude Code)
├── README.md                      This file
├── .gitignore                     Ignores stacks/**/.env and OS noise
├── docs/
│   ├── dashboard.md               Dashboard architecture + API reference
│   ├── deployment.md              Portainer stack setup guide
│   ├── node-setup.md              Setting up a new/replacement node
│   ├── roadmap.md                 Phase plan with status
│   └── security.md                Security review log
├── stacks/                        One folder per Swarm service (stateless, deployed via Portainer GitOps)
│   ├── cloudflared/
│   │   └── docker-compose.yml     Cloudflare Tunnel daemon (Swarm stack)
│   ├── dashboard/
│   │   ├── Dockerfile             Multi-stage build: node:20-alpine
│   │   ├── docker-compose.yml     Dashboard + socketproxy + glances (Swarm stack)
│   │   ├── package.json           Node.js dependencies
│   │   ├── .env.example           Template for local dev (no secrets)
│   │   ├── backup.sh              Node backup script (cron on manager nodes)
│   │   ├── scripts/
│   │   │   └── migrate-sqlite-to-pg.js   One-off SQLite → PostgreSQL data migration
│   │   └── src/
│   │       ├── server.js          Express backend (API, auth, metrics)
│   │       ├── db.js              PostgreSQL pool, schema init, query helpers
│   │       ├── vault-crypto.js    AES-256-GCM + PBKDF2 helpers for the vault
│   │       ├── routes/
│   │       │   └── vault.js       Vault CRUD routes (mounted at /api/vault)
│   │       └── public/            SPA shell, login page, frontend JS/CSS
│   └── portainer/
│       └── docker-compose.yml     Portainer agent (Swarm) — Portainer server itself runs on zs-state-01
└── stacks/stirling-pdf.yml        Stirling-PDF toolkit (flat file, zs-worker-03, stateless)
```

Convention: each Swarm service lives in `stacks/<service>/docker-compose.yml`. Compose files deployed via Portainer have no top-level `name:` field (Portainer assigns the stack name). No secrets in the repo — passwords and tokens go into Docker Secrets or a `.env` covered by `.gitignore`. Stateful services (PostgreSQL, Portainer) are **not** in this repo — their compose files live locally on `zs-state-01`.

---

## Roadmap

| Phase | Name | Status |
|---|---|---|
| 0 | Preparation | ✅ Done |
| 1 | Base OS | ✅ Done |
| 2 | Swarm + Portainer | ✅ Done |
| 3 | Reverse Proxy + Public Access | ✅ Done (cloudflared) |
| 4 | Dashboard + Auth | ✅ Done (security hardening) |
| 5 | DNS in Docker | ✅ Done |
| 6 | Stateless Swarm Refactor | ✅ Done (dedicated stateful host + storage server, 7 → 9 devices) |
| 7 | Dashboard Data Migration | ✅ Done (SQLite → PostgreSQL on `zs-state-01`) |
| 8 | Auth Hardening | ✅ Done (Cloudflare Access: Email OTP + Authenticator MFA) |
| 9 | Offsite Backup | ⬜ Planned (Cloudflare R2, 3-2-1 rule) |
| 10 | Polish | ⬜ Planned |

---

## Further Reading

- [docs/journey.md](docs/journey.md) — build story: what went wrong, what was learned, and why specific decisions were made
- [examples/](examples/) — sanitized, generic Docker Compose templates

---

## Acknowledgments

The streaming site's backend is built on [**Crimson**](https://github.com/crimsonhaven-to) by Crimson Haven — specifically [`crimson-backend`](https://github.com/crimsonhaven-to/crimson-backend), a FastAPI metadata/discovery engine. It ships with no content sources of its own; this instance (`crimson-zer0space`) points it at this homelab's own media library. Props to Crimson Haven for the framework.

---

Built and maintained with [Claude Code](https://claude.ai/code).
