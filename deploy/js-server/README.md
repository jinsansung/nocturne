# Nocturne on `noc.whalesound.net` (js-server shared edge)

This directory holds a **shared-edge, single-tenant** deployment of Nocturne that
runs on a **subdomain** of an apex owned by someone else (`whalesound.net`),
behind the shared **js-server Caddy** — not the bundled Caddy from
[`../docker-compose/`](../docker-compose/).

## Why this exists — the earlier subdomain-install failure

The stock bundle (`../docker-compose/docker-compose.yaml`) is built for a
**dedicated apex**:

- it ships its **own Caddy** that binds host ports **80/443**, and
- it mints **wildcard on-demand TLS** for `*.{BASE_DOMAIN}` (one cert per tenant
  subdomain), and
- its `.env.example` says *"Root domain only … subdomains are generated per
  tenant."*

On the js-server droplet none of that holds: the **shared Caddy already owns
80/443** and all public routing (the "6-rule contract"), so a second Caddy can't
start — and a subdomain host has no room for the per-tenant wildcard model. That
collision is why the previous attempt to install on a subdomain failed.

**The application code was never the problem.** `BASE_DOMAIN` is used *relative*
to whatever host it's set to:

- `rpId = BASE_DOMAIN` (minus port) → WebAuthn/passkeys scope to
  `noc.whalesound.net` and its subdomains
  (`ServiceRegistrationExtensions.cs`).
- `SubdomainParser.Extract(host, BASE_DOMAIN)` resolves tenants relative to the
  base domain — it's unit-tested with a multi-label base domain
  (`nocturne.theconen.de`).
- With exactly one tenant, `TenantResolutionMiddleware` auto-resolves it on the
  apex host, so the whole app can live at a single host with **no subdomains**.

So the fix is purely at the **edge**: drop the bundled Caddy, publish only the
YARP **gateway** onto the shared `js-server_edge` network by a fixed container
name, and let the shared Caddy reverse-proxy `noc.whalesound.net` to it. We run
**single-tenant** so `noc.whalesound.net` is the whole app and **no wildcard
DNS/TLS is required**.

## What this stack does / doesn't do

| | |
|---|---|
| Public app | `https://noc.whalesound.net` (single tenant, apex-resolved) |
| Bundled Caddy | **removed** — the shared js-server Caddy terminates TLS |
| Published host ports | **none** — the gateway is reachable only over `js-server_edge` |
| Edge upstream | `nocturne-gateway:5000` (fixed `container_name`) |
| Persistence | external named volume `nocturne-noc-postgres-data` |
| Per-tenant subdomains (`*.noc…`) | **off** (single-tenant) — would need a wildcard route |
| Public share links (`{token}.share.noc…`) | **off** — see "Enabling public shares" below |

## Deploy (on the droplet, in `/opt/js-noc/`)

```bash
# 1. join the shared edge network (owned by js-server; we only join)
docker network create js-server_edge 2>/dev/null || true

# 2. create the persistent data volume once (survives compose down / re-create)
docker volume create nocturne-noc-postgres-data

# 3. secrets
cp .env.example .env
#   set BASE_DOMAIN=noc.whalesound.net and fill INSTANCE_KEY + the 4 POSTGRES_* .
#   generate each with:  openssl rand -base64 32

# 4. up
docker compose up -d
```

Then hand the routing request below to the **js-server ops session**. Until the
shared Caddy has the route, the gateway is only reachable inside `js-server_edge`
(by design — no ports are published).

First load of `https://noc.whalesound.net` returns `503 setup_required` and the
UI bounces to `/setup`; complete setup to create the single tenant + owner
passkey.

## Routing request — hand this to the js-server ops session

> Filling in the "신규 스택 → 서버운영" template:
>
> 1. **Subdomain slug:** `noc` → `noc.whalesound.net`  *(noc was dropped, reusable)*
> 2. **Upstream:** `nocturne-gateway:5000`
> 3. **Upstream protocol:** `http` (plaintext; TLS terminates at the shared Caddy)
> 4. **Proxy specifics:**
>    - **WebSocket + SSE required** — SignalR hubs (alerts, live glucose widget)
>      use WebSocket; live streams use SSE. Enable `flush_interval -1` and
>      WebSocket upgrade. A single 5-minute activity timeout is already set on the
>      gateway's web cluster.
>    - Forward the original **Host** and **X-Forwarded-*** (Proto/Host) — the app
>      trusts them (`ASPNETCORE_FORWARDEDHEADERS_ENABLED=true`, SvelteKit
>      `ORIGIN=https://noc.whalesound.net`).
>    - No large-upload path; default body limits are fine.
> 5. **Visibility:** public (Caddy route).
> 6. **Edge join confirmed:** yes — `nocturne-gateway` joins `js-server_edge`.
> 7. **Memory (mem_limit, for the manifest):** postgres 512m · api 640m ·
>    web 512m · gateway 256m  (≈ **1.9 GB** ceiling total).
> 8. **Stack nature:** code stack (own repo: `jinsansung/nocturne`), images from
>    `ghcr.io/nightscout/nocturne/*`.

A minimal shared-Caddy route looks like:

```caddy
noc.whalesound.net {
    reverse_proxy nocturne-gateway:5000
}
```

(Caddy proxies WebSocket automatically; add `flush_interval -1` if SSE buffers.)

## Enabling public share links later (optional)

Share links are served at `{token}.share.noc.whalesound.net` — an unbounded set
of hostnames. To turn them on you need, on the **shared Caddy**:

- a wildcard route for `*.share.noc.whalesound.net` (and `*.noc.whalesound.net`
  if you also want multi-tenant subdomains) → `nocturne-gateway:5000`, and
- on-demand TLS gated by Nocturne's authorizer so certs only mint for real
  hosts: `on_demand_tls { ask http://nocturne-api:8080/api/v4/platform/tls-authorize }`
  (the `nocturne-noc-api` container must then also join `js-server_edge`), and
- Cloudflare DNS `*.share.noc` (and `*.noc`) as **gray-cloud** A records.

Because HTTP-01 can't issue wildcards, this relies on Caddy's **per-host**
on-demand issuance, not a single wildcard cert. This is a shared-Caddy config
change — request it from the ops session; it is out of scope for this stack.

## Updating

Images are pinned to `:latest`. To update:

```bash
docker compose pull && docker compose up -d
```

(No Watchtower here — leave image lifecycle to whatever js-server standardizes on.)
