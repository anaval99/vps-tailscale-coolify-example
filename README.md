# VPS + Tailscale + Coolify

Expose self-hosted apps to the internet without a static IP or port forwarding.

## Architecture

```
Internet --> VPS (Caddy + Tailscale) --tailnet--> Home Server (Tailscale + Coolify/Traefik) --> apps
```

- **VPS**: Cheap cloud server with a public IP. Caddy handles HTTPS and forwards traffic through the Tailscale tunnel.
- **Home Server**: Runs Coolify, which manages app deployments, builds, and routing via Traefik.
- **Tailscale**: Creates a secure private network between VPS and home server. No port forwarding needed on your router.

## Setup

### 1. Home Server

Install Coolify:

```bash
curl -fsSL https://cdn.coollabs.io/coolify/install.sh | sudo bash
```

Start the Tailscale sidecar so your home server joins the tailnet:

```bash
cd homeserver
cp .env .env.local  # edit with your Tailscale auth key
docker compose up -d
```

After Tailscale connects, note your home server's Tailscale IP (`tailscale status`).

### 2. VPS

```bash
cd vps
cp .env .env.local  # edit with your values
docker compose up -d
```

Required `.env` values:
- `TS_AUTHKEY` — Tailscale auth key ([generate here](https://login.tailscale.com/admin/settings/keys))
- `COOLIFY_TAILSCALE_IP` — your home server's Tailscale IP (e.g. `100.x.x.x`)
- `ACME_EMAIL` — email for Let's Encrypt certificates

Then edit `vps/Caddyfile` and replace the example domains with your own.

### 3. DNS

Point your domains to the VPS public IP:
- `example.com` → VPS IP (A record)
- `www.example.com` → VPS IP (A record)
- `coolify.example.com` → VPS IP (A or CNAME record)

### 4. Coolify Configuration

In the Coolify dashboard:
- **Important: Set app domains to `http://` (not `https://`)** — Caddy on the VPS handles TLS termination. If you use `https://`, Traefik will try to redirect HTTP to HTTPS, causing an infinite redirect loop.
- Coolify's Traefik routes requests to the correct app by domain on port 80

#### Traefik Forwarded Headers

By default, Traefik doesn't trust `X-Forwarded-Proto` from upstream proxies. This can cause redirect loops in apps that check the protocol. To fix this, add the following to Coolify's Traefik config via `docker-compose.custom.yml` on your home server (at `/data/coolify/source/docker-compose.custom.yml`):

```yaml
services:
  proxy:
    command:
      - --entrypoints.http.forwardedheaders.trustedips=100.64.0.0/10
```

The `100.64.0.0/10` range covers all Tailscale IPs, so Traefik will trust headers from your VPS.

### 5. GitHub Auto-Deploy

Coolify needs to receive webhooks from GitHub. Two options:

**Option A: Forward webhooks through the VPS**
Add a webhook route in the Caddyfile that forwards to Coolify's webhook endpoint on the home server.

**Option B: GitHub Actions + Coolify API**
Trigger deployments via Coolify's API from a GitHub Action — no inbound webhook needed. See [Coolify docs](https://coolify.io/docs/applications/ci-cd/github/actions).

## Adding More Apps

1. Deploy the app in Coolify with its domain set to HTTP
2. Add a new block in `vps/Caddyfile`:
   ```
   app2.example.com {
     reverse_proxy {$COOLIFY_TAILSCALE_IP}:80 {
       header_up X-Forwarded-Proto {scheme}
       header_up X-Forwarded-Port {port}
       header_up X-Real-IP {remote_host}
     }
   }
   ```
3. Point `app2.example.com` to your VPS IP in DNS
4. Restart Caddy: `docker compose restart caddy`

## Ports Reference

| Port | Where | Purpose |
|------|-------|---------|
| 80, 443 | VPS | Public HTTP/HTTPS (Caddy) |
| 80 | Home Server | Coolify's Traefik (app routing) |
| 8000 | Home Server | Coolify dashboard |
| 6001 | Home Server | Coolify realtime (WebSocket) |
| 6002 | Home Server | Coolify terminal (WebSocket) |
