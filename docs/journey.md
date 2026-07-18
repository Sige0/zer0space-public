# Building zer0space: A Homelab Journey

A blog-style write-up of the decisions, dead ends, and breakthroughs that shaped this setup. If you're building something similar, this is the article I wish I'd found first.

---

## Starting Point

Seven ThinkCentre M910q mini PCs, Debian 12, and a goal: a self-hosted cluster where every service configuration lives in Git, deploys automatically, and exposes nothing directly to the internet.

The hardware choice was deliberate — M910qs are cheap, power-efficient, silent, and take up almost no space. At 35 W TDP each, all seven together draw less than a single mid-range gaming PC. The 32 GB / 1 TB unit for media storage was an eBay find.

The architecture goal: **zero open ports on the router**. No NAT, no DynDNS, no "just forward 443 and hope for the best." The public internet should never see the origin IP.

---

## Why Cloudflare Tunnel Instead of a Reverse Proxy

The initial plan was the standard homelab stack: Traefik as the reverse proxy, Let's Encrypt for TLS, a DDNS updater for the dynamic IP, and port 443 forwarded through the router.

The problem: that still exposes your home IP. Anyone can look up the A record, or catch it in a certificate transparency log. That felt like an unnecessary risk.

**Cloudflare Tunnel** solves this cleanly. `cloudflared` runs as a Docker Swarm service inside the cluster. It opens an outbound connection to Cloudflare's edge — no inbound ports, no port forwarding. Cloudflare terminates TLS at the edge, and the tunnel carries the decrypted request to whichever service is mapped to a given hostname.

The routing is direct: `cloudflared` maps each public hostname to a Docker Swarm service DNS name like `http://dashboard_dashboard:3000`. Inside the overlay network, Docker's built-in DNS resolves that to the right container. No intermediate proxy hop needed.

The added bonus: **Cloudflare Access** sits in front of everything. Before any request reaches a service, it has to pass a GitHub login check. That's MFA enforced at the network edge, before a single packet reaches the cluster.

---

## The Traefik Dead End (Swarm API v1.24 Rejection)

Despite the Cloudflare Tunnel approach for public traffic, Traefik was still evaluated as an internal routing layer — it can do useful things like TLS between services and more flexible routing rules.

The setup: `tecnativa/docker-socket-proxy` configured to allow API version 1.44+, with Traefik pointed at the proxy.

The problem appeared immediately in the Traefik logs:

```
Error while fetching server list: received error 400: client version 1.24 is too old
```

Traefik v3.x negotiates the Docker API version by sending `1.24` as the client version — regardless of what the server supports. The socket-proxy sees an API version below its configured minimum and returns a 400. Traefik never gets a list of running services.

This is not a configuration error on either side. It's a version mismatch between what Traefik negotiates and what a locked-down socket-proxy allows. The socket-proxy is doing exactly what it's supposed to do.

The workaround options are all bad: give Traefik direct socket access (defeats the point of the proxy), patch the proxy config to allow 1.24 (weakens the security posture), or use Traefik v2 (goes against the grain on a new setup).

**Decision: drop Traefik entirely.** Since `cloudflared` routes directly to service DNS names in the overlay network, there's no need for a reverse proxy layer at all. Traefik's stack is kept in the repo for reference but not deployed.

---

## Overlay Network Debugging: Why Glances Runs on Host Networking

The dashboard needs live metrics from every node — CPU, RAM, disk, network. The plan was a [Glances](https://github.com/nicolargo/glances) container in global mode (one per node) on a shared overlay network, with the dashboard backend polling each agent over that network.

It didn't work.

The symptom: Glances containers would start and show as healthy, but requests from the dashboard to any Glances agent returned `ECONNREFUSED`. The overlay DNS would resolve correctly, but the connection would fail. Restarting containers would sometimes help briefly, then fail again.

**What was actually happening:** In Docker Swarm, overlay networking routes traffic through VXLAN tunnels managed by the Swarm control plane. When a global service runs on many nodes and the dashboard tries to address a specific node's instance by hostname, the overlay routing gets complicated — you're essentially trying to pin a request to a specific node inside a distributed network that wasn't designed for that use case.

The root cause: overlay networks work well for service-to-service calls where any replica is equivalent. They work poorly when you need to address a *specific node's* container — which is exactly what a per-node metrics agent requires.

**The fix was to get off the overlay entirely.**

Glances now runs with `endpoint_mode: host`. The container shares the host's network namespace — port 61208 is bound directly to the host's physical NIC, not through any VXLAN tunnel. The dashboard backend gets each node's LAN IP from the Swarm API (`Status.Addr` on the Node object), and hits `http://<node-ip>:61208/api/4/cpu` directly.

The result: rock-solid. No more intermittent failures, no overlay routing complexity. The Swarm API is the source of truth for which nodes are online and what their addresses are. If a node goes down, the Swarm API reflects that and the dashboard shows it as offline — no stale state.

One thing that doesn't work with `endpoint_mode: host` in Swarm: `pid: host`. That mount is silently ignored. It turns out not to matter — Glances gets correct CPU and RAM values through the container's `/proc` view, and disk/network metrics come from bind-mounts of `/sys` and `/var/run/docker.sock`. The metrics are accurate without `pid: host`.

---

## The HTTPS / CSP Chain

Getting the dashboard behind HTTPS surfaced a subtle but important ordering constraint.

The dashboard runs HTTP internally (port 3000). Cloudflare terminates TLS at the edge and proxies plaintext to the container. This means:

- **`COOKIE_SECURE=true`** must only be set when the connection between browser and Cloudflare is HTTPS — which it always is once the domain is proxied. But the flag only makes sense once the tunnel is live.
- **`FORCE_HTTPS=true`** enables `Strict-Transport-Security` headers and adds `upgrade-insecure-requests` to the CSP. This must never be sent over plain HTTP — a browser that receives HSTS over HTTP will remember it and refuse plain HTTP connections forever for that domain.

The safe order: get the Cloudflare Tunnel confirmed working first with both flags off, verify the dashboard is reachable over HTTPS, then flip both flags and redeploy.

One trap with Helmet (the Node.js security header library): Helmet v7+ uses `strictTransportSecurity` as the config key. The older `hsts` key is silently ignored — HSTS would then be **always active**, including during HTTP-only testing. The server config uses `strictTransportSecurity` explicitly and gates it behind `FORCE_HTTPS`.

**CSP design decisions:**

The Content Security Policy has `unsafe-inline` for `style-src` but not for `script-src`. This is intentional:

- Metric progress bars (`style="width: 42%"`) are set by the frontend via `innerHTML` — they need inline styles to work.
- Scripts are served as external files (`/app.js`, `/login.js`). There are no inline `<script>` blocks. `unsafe-inline` on `script-src` would add no capability but would meaningfully weaken XSS protection.

The dashboard also loads [Tabler Icons](https://tabler.io/icons) from jsDelivr CDN for the icon font. That required adding `https://cdn.jsdelivr.net` to both `style-src` and `font-src`.

---

## MFA via Cloudflare Access

Cloudflare Access acts as the outermost authentication layer. Before a request reaches any service, the user must authenticate through an Access policy — in this setup, a GitHub OAuth login.

This means:

1. The browser hits `dashboard.<DOMAIN>`.
2. Cloudflare checks for a valid Access session cookie.
3. If missing or expired, it redirects to the GitHub OAuth flow.
4. After a successful GitHub login (and the GitHub account matching the allowed identity), Cloudflare issues a JWT and sets the Access session cookie.
5. Only then does the request travel through the tunnel to the dashboard container.

The dashboard itself also has its own login (bcrypt + session cookie). That's intentional — the dashboard has roles (admin vs. viewer) that Cloudflare Access can't model, and the built-in Vault requires authentication to derive the encryption key. Cloudflare Access is the perimeter; the dashboard's own auth is the application layer.

For services that don't have their own authentication (like Stirling-PDF), Cloudflare Access provides the only gate. That's sufficient for a single-user homelab.

---

## The Stateless Swarm Refactor: Why Nodes Became Disposable

For the first several months, the Swarm nodes weren't actually stateless. The dashboard's SQLite file lived in a bind-mount on whichever manager node it happened to be pinned to. Portainer's own data volume lived inside the Swarm it was managing. A few smaller services wrote to local paths on specific workers "because that's where the disk was."

This worked, right up until it didn't: losing a node meant losing whatever bind-mounted data lived on it, and re-provisioning a node meant remembering — by hand — which paths needed to exist before a stack would come back healthy. That's the opposite of the GitOps promise. The repo was supposed to be the single source of truth; in practice, a chunk of the truth was sitting on individual disks that weren't in Git at all.

The fix was to draw a hard line: **the Swarm holds no state.** Every Swarm node — manager or worker — had to become genuinely interchangeable: wipe it, reinstall Debian, `docker swarm join`, done. No data recovery step, because there's no data to recover.

That line only works if persistent data has somewhere else to live, so the cluster grew two new dedicated machines that sit deliberately outside the Swarm:

- **`zs-store-01`**, an NFS server. Every Swarm service that needs a data volume mounts it from here instead of a local path. Docker's `local` volume driver with `type: nfs` options makes this a drop-in replacement for a bind-mount from the service's point of view — see [examples/nfs-volume-service.yml](../examples/nfs-volume-service.yml).
- **`zs-state-01`**, a plain-Docker host (deliberately *not* a Swarm member) for the handful of things that genuinely want local, low-latency disk instead of a network filesystem — PostgreSQL being the main one. More on why Postgres didn't just go on NFS below.

The result: the Swarm went from 6 nodes to 7 (a fourth worker, `zs-worker-04`, was added around the same time for extra headroom), and the homelab as a whole went from 7 devices to 9. But the meaningful change isn't the node count — it's that all 7 Swarm nodes are now genuinely disposable, and only two machines in the whole fleet actually need to be treated carefully.

---

## The Portainer Migration Pain

Portainer had a bootstrapping problem that took an outage to notice: it was deployed *as a Swarm stack*, managing the very Swarm it was running on, with its own data volume bind-mounted to whichever manager node it landed on.

That's fine until the node holding Portainer's data goes down. At that point the tool you'd normally reach for to fix a broken Swarm — Portainer — is the thing that's broken. Recovering meant SSHing into a manager by hand and running `docker service` commands directly, exactly the workflow GitOps was supposed to eliminate. The control plane depended on the infrastructure it was supposed to control.

The fix: pull Portainer out of the Swarm entirely and run it as a standalone Docker container on `zs-state-01`, talking to the Swarm as a **remote** Docker environment over TLS instead of the local socket.

The migration itself was the genuinely painful part:

- Portainer's environment registration had to be redone against a remote TCP endpoint, which meant generating and distributing new TLS client certificates for the Docker daemon on each manager node (`dockerd` doesn't expose the remote API by default — that's an intentional security posture, not an oversight, so enabling it required care).
- Every existing GitOps webhook URL pointed at the old Portainer instance's internal address. Each one had to be regenerated and re-pasted into the corresponding GitHub Actions workflow / repo webhook config — there's no bulk-migrate for this.
- Portainer's own stack definitions (the list of what to deploy from the repo) live in Portainer's database, not in the Git repo itself. That state had to be manually re-created after the move, because the old Portainer instance's database wasn't something worth carrying forward — it was mixed up with the very bind-mount-on-a-random-node problem this migration was fixing.
- Swarm-side, only a lightweight **Portainer agent** remains, running as a global stateless service. It has no data of its own — if it disappears, redeploying it is a no-op operationally.

The downtime was measured in a very tense hour, not days, but it was a self-inflicted hour: the outage that triggered this whole refactor was a manager node going down with Portainer's data volume on it, at a moment when nobody had planned for it.

**The lesson that stuck:** never let your control plane live inside the infrastructure it controls. If Portainer manages the Swarm, Portainer's own state cannot be a Swarm service.

---

## The Backup Restore That Saved the Vault

The other half of the outage above: the dashboard's SQLite database — which holds the built-in Vault, the AES-256-GCM password manager — was on the same manager node that went down, in a bind-mount that had not yet been migrated to NFS.

The node came back up, but the disk didn't. Whatever caused the failure took the local filesystem with it, and the dashboard's data file was gone.

This is the point where a homelab project either has a backup story or doesn't. It turned out there was a nightly cron job on `zs-worker-01` that had been rsyncing the dashboard's data directory for weeks — set up early on, mostly forgotten about, never actually tested end-to-end. The backup from roughly 20 hours before the failure was intact.

Restoring it wasn't just "copy the file back." The Vault's encryption key is derived from each user's login password via PBKDF2 and is never stored anywhere — which is exactly the property that makes the Vault worth using, but it also means there's no way to verify a restored database is *actually* intact without logging in and confirming that stored entries decrypt correctly. A corrupted or partially-written backup would have looked fine until someone tried to open a password entry and got garbage back. That verification step — log in, open a few Vault entries, confirm they decrypt — became a mandatory part of the restore procedure, not an afterthought.

**The lesson:** an untested backup is a hope, not a backup. The rsync job had been running for weeks without anyone confirming a restore actually worked. It happened to work. That's the point where "nightly backup exists" got upgraded to "nightly backup exists *and* the restore path has been exercised for real," and it's also what pushed the storage NFS work higher up the priority list — the dashboard's data now lives on `zs-store-01`, not on whichever node it's scheduled on, which removes this entire failure class going forward.

---

## Choosing NFS Over Alternatives for Shared Storage

Once "Swarm nodes must be disposable" became the rule, something had to hold the data that used to live in per-node bind-mounts. The options considered:

- **GlusterFS / Ceph** — distributed, replicated, resilient to losing a storage node. Also a meaningful operational burden for a single-user homelab: more moving parts, more failure modes to understand, more things that can go subtly wrong at 2 AM. Overkill for the actual requirement here.
- **A per-node local-storage-plus-sync setup** — essentially rebuilding the same bind-mount problem with extra steps.
- **Plain NFS** from a dedicated server — boring, decades-old, well-understood, and Docker's `local` volume driver supports NFS mount options natively, so services just declare a volume with `driver_opts` instead of a bind-mount path. No extra sidecar container needed on the Swarm side.

NFS won on simplicity. The honest trade-off: `zs-store-01` is a single point of failure for Swarm service data. That's accepted for a personal homelab, and mitigated the same way the Vault incident got fixed — nightly backups off of NFS to `zs-worker-01`, with a Cloudflare R2 offsite copy planned as the next layer of protection.

One deliberate exception: **PostgreSQL did not move to NFS.** Databases and network filesystems have a long, well-documented history of not getting along — NFS's caching and locking semantics aren't what a database engine expects from local disk, and the failure modes (silent corruption under contention) are worse than the failure modes NFS was adopted to solve. PostgreSQL instead runs directly on `zs-state-01`'s local disk, which is also why that host exists as a separate category from "pure NFS-backed storage" — some state genuinely needs a real, dedicated, non-Swarm machine underneath it.

---

## Lessons

**Host networking solves a whole class of Swarm problems.** If you're running a per-node agent that other services need to address by node, stop trying to make the overlay work and just bind to the host NIC. Use the Swarm API's `Status.Addr` to get each node's address dynamically. It's simpler and more reliable.

**Cloudflare Tunnel is easier than a reverse proxy for a homelab.** No TLS certificates to manage, no port forwarding, no DynDNS. The trade-off is a dependency on Cloudflare, but for a personal homelab that's an acceptable trade.

**The socket-proxy pattern is worth the complexity.** Giving any service direct access to the Docker socket means a compromised container can control the entire host. A socket proxy that exposes only the API endpoints a service actually needs (in this case: `SERVICES`, `NODES`, `TASKS`) limits the blast radius significantly.

**Gitops from day one.** Every service configuration is in Git. Every deployment happens via Portainer watching the repo. There has never been a "I made a manual change and forgot to document it" incident because manual changes aren't the workflow.

**Ship the security review before going public.** The security hardening pass before public launch found several real issues: missing HTTP security headers, error responses that leaked internal error messages, a JSON body size limit that didn't exist, and input validation gaps on the service management API. None were catastrophic, but all were worth fixing before real users hit them.

**A control plane cannot live inside the infrastructure it controls.** Portainer managing a Swarm while running as a stateful service *on* that Swarm is a bootstrapping trap — the day you need Portainer most is the day a node holding its data went down. Pulling it out to a dedicated, non-Swarm host fixed this permanently.

**Untested backups are a hope, not a backup.** The rsync job that saved the Vault had been running unverified for weeks. It happened to work. Now every restore includes an explicit verification step — log in, open a few entries, confirm they decrypt — because "the file copied successfully" and "the data is actually usable" are different claims.

**Stateless is a design decision, not a default.** Docker Swarm doesn't make your services stateless for you; it just makes it easy to *forget* where state actually lives until a node dies. Deciding upfront that the Swarm holds zero persistent data — and giving that data exactly two homes (NFS for volumes, a dedicated host for anything that needs real local disk) — turned "which node is my data on?" from a live incident question into a non-question.
