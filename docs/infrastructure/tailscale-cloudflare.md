---
id: tailscale-cloudflare
title: Tailscale & Cloudflare Infrastructure
sidebar_position: 1
description: How BlackRoad OS uses Tailscale for private mesh networking and Cloudflare for edge hosting and tunneling.
---

# Tailscale & Cloudflare Infrastructure

> **Status:** Production  
> **Owner:** @blackboxprogramming

BlackRoad OS routes all internal service traffic through a private Tailscale mesh and exposes public endpoints exclusively via Cloudflare Tunnels — no inbound ports are opened on any node.

---

## Architecture Overview

```
Internet → Cloudflare Edge → Cloudflare Tunnel → blackroad-gateway (aria64)
                                                         │
                                         ┌───────────────┴──────────────────┐
                                         │       Tailscale Mesh (100.x.x.x) │
                                         │                                   │
                                     api.blackroad.io              Pi Nodes  │
                                     docs.blackroad.io             aria64     │
                                     app.blackroad.io              alice      │
                                         └───────────────────────────────────┘
```

All AI vendor calls (OpenAI, Anthropic, etc.) **never** route through BlackRoad hardware. They are outbound-only connections initiated by the operator's own device directly to vendor servers.

---

## Tailscale Setup

### 1. Install Tailscale on each node

```bash
curl -fsSL https://tailscale.com/install.sh | sh
```

### 2. Authenticate and tag nodes

```bash
sudo tailscale up \
  --authkey=tskey-auth-BLACKROAD_KEY \
  --advertise-tags=tag:blackroad-node \
  --hostname=aria64
```

Repeat for each Pi, using the appropriate `--hostname`.

### 3. Enable subnet routing on the gateway node (aria64)

```bash
sudo tailscale up \
  --advertise-routes=192.168.4.0/24 \
  --accept-routes
```

Then approve the route in the [Tailscale admin console](https://login.tailscale.com/admin/machines).

### 4. Verify connectivity

```bash
# From any enrolled device
tailscale ping aria64
tailscale status
```

### Tailscale ACLs (Access Control)

Only approved contributors and nodes may join the BlackRoad tailnet. ACL policy file:

```json
{
  "tagOwners": {
    "tag:blackroad-node": ["autogroup:owner"]
  },
  "acls": [
    {
      "action": "accept",
      "src": ["tag:blackroad-node", "autogroup:owner"],
      "dst": ["tag:blackroad-node:*"]
    }
  ]
}
```

---

## Cloudflare Tunnel Setup

All public traffic is served through Cloudflare Tunnels (`cloudflared`). **No inbound firewall rules are needed.**

### 1. Install cloudflared

```bash
# On the gateway node (aria64)
curl -L https://github.com/cloudflare/cloudflared/releases/latest/download/cloudflared-linux-arm64 \
  -o /usr/local/bin/cloudflared
chmod +x /usr/local/bin/cloudflared
```

### 2. Authenticate with your Cloudflare account

```bash
cloudflared tunnel login
```

### 3. Create the BlackRoad tunnel

```bash
cloudflared tunnel create blackroad-prod
```

### 4. Configure tunnel ingress (`~/.cloudflared/config.yml`)

```yaml
tunnel: blackroad-prod
credentials-file: /home/blackroad/.cloudflared/<TUNNEL_ID>.json

ingress:
  - hostname: api.blackroad.io
    service: http://localhost:8080
  - hostname: docs.blackroad.io
    service: http://localhost:3000
  - hostname: app.blackroad.io
    service: http://localhost:3001
  - service: http_status:404
```

### 5. Route DNS to the tunnel

```bash
cloudflared tunnel route dns blackroad-prod api.blackroad.io
cloudflared tunnel route dns blackroad-prod docs.blackroad.io
cloudflared tunnel route dns blackroad-prod app.blackroad.io
```

### 6. Run as a systemd service

```bash
sudo cloudflared service install
sudo systemctl enable --now cloudflared
```

### 7. Verify

```bash
cloudflared tunnel info blackroad-prod
curl -I https://api.blackroad.io/v1/health
```

---

## Environment Variables

Store Cloudflare credentials in `.env` (never committed):

```bash
CLOUDFLARE_ACCOUNT_ID=848cf0b18d51e0170e0d1537aec3505a
CLOUDFLARE_TUNNEL_NAME=blackroad-prod
TAILSCALE_AUTH_KEY=tskey-auth-XXXXXXXX   # store in CI/CD secrets
```

See `.env.example` for the full list of required variables.

---

## Network Topology Confirmation

> **Does OpenAI or any AI vendor traffic route through BlackRoad Pi nodes?**

**No.** The data flow is strictly:

```
Your device → internet → AI vendor servers
```

There is no inbound tunnel, no reverse proxy, and no relay configured that would route vendor traffic into BlackRoad hardware. The Cloudflare Tunnel is outbound-only from the Pi nodes — it cannot be used as an entry point for third-party services.

---

## References

- [Tailscale ACL documentation](https://tailscale.com/kb/1018/acls)
- [Cloudflare Tunnel documentation](https://developers.cloudflare.com/cloudflare-one/connections/connect-networks/)
- [Pi Agent Nodes](./pi-agents.mdx)
- [API Design](../reference/api-design.md)
