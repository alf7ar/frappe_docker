# Hosting behind Cloudflare (Tunnel)

This page explains how Cloudflare fits into an ERPNext deployment and how to
publish your site through a **Cloudflare Tunnel** with no open inbound ports.

## Can ERPNext run "on Cloudflare"?

**No — not on Cloudflare Workers or Pages.** ERPNext/Frappe is a stateful,
multi-container Python application:

- a **gunicorn** (Python) web backend,
- a **MariaDB** database,
- **Redis** (cache + queue),
- background **queue workers** and a **scheduler**,
- a **Node.js socket.io** realtime server,
- an **nginx** frontend.

Cloudflare Workers run short-lived JavaScript/WASM isolates with a per-request
CPU cap, no persistent filesystem, and no long-running processes. Pages is
static hosting plus Workers-based Functions. Neither can run Python, host a
database, or keep worker processes alive. There is no adapter that changes
this.

**What Cloudflare _does_ do** is sit *in front of* a real Linux host that runs
these containers:

| Cloudflare feature | Role |
| --- | --- |
| DNS | Points your domain at the tunnel/host |
| Proxy / CDN (orange cloud) | TLS termination, caching of static assets, DDoS protection |
| **Tunnel (`cloudflared`)** | Reaches your host with **zero open inbound ports** |
| WAF / Rate limiting / Access | Security & auth in front of the Desk |
| R2 | Optional S3-compatible bucket for off-site backups |

The containers themselves still run on a server you control — any Linux VM
with Docker (Hetzner, DigitalOcean, AWS Lightsail/EC2, GCP, Oracle Cloud, or
on-prem hardware). A practical minimum for ERPNext is **2 vCPU / 4 GB RAM**;
8 GB is more comfortable.

## Why a Tunnel

A Cloudflare Tunnel runs a small `cloudflared` agent next to your stack that
dials **outbound** to Cloudflare and receives proxied traffic over that
connection. The result:

- **No firewall ports open** to the internet (not even 80/443) — smaller attack
  surface.
- **TLS handled by Cloudflare**, so you can skip Let's Encrypt/Traefik if you
  want.
- Works behind NAT / on machines without a public IP.

## Setup

### 1. Create the tunnel in Cloudflare

1. Cloudflare **Zero Trust** dashboard → **Networks → Tunnels → Create a
   tunnel** → choose **Cloudflared**.
2. Name it (e.g. `erpnext`) and copy the **tunnel token** it displays.
3. Under the tunnel's **Public Hostnames**, add a route:
   - **Subdomain / Domain:** `erp.yourdomain.com`
   - **Service:** `HTTP` → `frontend:8080`

   The service target is the Docker **service name and port**. `cloudflared`
   runs on the same `frappe_network`, so it reaches `frontend:8080` directly —
   no host ports required.

### 2. Provide the token

Add the token to your environment (or `.env` file). Keep it secret.

```bash
CLOUDFLARE_TUNNEL_TOKEN=eyJ...
```

### 3. Start the stack with the overlay

```bash
docker compose \
  -f pwd.yml \
  -f overrides/compose.cloudflared.yaml \
  up -d
```

That's it — `https://erp.yourdomain.com` now serves your ERPNext site through
Cloudflare.

### 4. Set the site's host name (recommended)

`pwd.yml` pins `FRAPPE_SITE_NAME_HEADER=frontend`, so the site is reachable
regardless of the Host header. For correct absolute URLs in emails and links,
tell the site its public address:

```bash
docker compose exec backend \
  bench --site frontend set-config host_name https://erp.yourdomain.com
```

## Hardening options

- **Close the direct port.** Once the tunnel works, remove the published
  `ports: ["8080:8080"]` from the `frontend` service (or bind it to
  `127.0.0.1`) so the host has no public web port at all.
- **Full (strict) TLS.** If you keep Traefik/nginx-proxy for local TLS, point
  the tunnel's public hostname at the proxy service instead of `frontend` and
  set Cloudflare's SSL mode to **Full (strict)** with an origin certificate.
- **Cloudflare Access.** Put the Desk behind Cloudflare Access to require SSO
  before the login page is even reachable.
- **Backups to R2.** Cloudflare R2 is S3-compatible; you can push Frappe
  backups to an R2 bucket for off-site storage. See
  [Backup strategy](02-backup-strategy.md).

## Combining with the existing production overlays

This overlay replaces the *public entry point*, so use it **instead of** the
Traefik/nginx TLS overlays (Cloudflare terminates TLS). The monitoring and
resource-limit overlays are independent and can still be layered on:

```bash
docker compose \
  -f pwd.yml \
  -f overrides/compose.cloudflared.yaml \
  -f overrides/compose.monitoring.yaml \
  -f overrides/compose.resource-limits.yaml \
  up -d
```

## Rebranding note

Publishing through Cloudflare does not change the ERPNext UI. Renaming/reskinning
ERPNext to "NextERP" (logo, app title, login page, theme, favicon in the Desk)
is done inside a Frappe **app's hooks** (e.g. `app_logo_url`, `app_title`,
`website_context`, a Website Theme) — not in this deployment repo. This repo
controls which app is baked into the image (`build/apps.json`) and how it is
served; the branding lives in the app itself.
