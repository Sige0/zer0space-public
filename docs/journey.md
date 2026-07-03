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

## Lessons

**Host networking solves a whole class of Swarm problems.** If you're running a per-node agent that other services need to address by node, stop trying to make the overlay work and just bind to the host NIC. Use the Swarm API's `Status.Addr` to get each node's address dynamically. It's simpler and more reliable.

**Cloudflare Tunnel is easier than a reverse proxy for a homelab.** No TLS certificates to manage, no port forwarding, no DynDNS. The trade-off is a dependency on Cloudflare, but for a personal homelab that's an acceptable trade.

**The socket-proxy pattern is worth the complexity.** Giving any service direct access to the Docker socket means a compromised container can control the entire host. A socket proxy that exposes only the API endpoints a service actually needs (in this case: `SERVICES`, `NODES`, `TASKS`) limits the blast radius significantly.

**Gitops from day one.** Every service configuration is in Git. Every deployment happens via Portainer watching the repo. There has never been a "I made a manual change and forgot to document it" incident because manual changes aren't the workflow.

**Ship the security review before going public.** The security hardening pass before public launch found several real issues: missing HTTP security headers, error responses that leaked internal error messages, a JSON body size limit that didn't exist, and input validation gaps on the service management API. None were catastrophic, but all were worth fixing before real users hit them.
