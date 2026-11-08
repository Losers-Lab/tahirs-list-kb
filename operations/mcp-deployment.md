---
type: operations-note
title: MCP Endpoint Deployment and Verification
description: Where the public MCP endpoint is hosted, how the Cloudflare tunnel routes
  it, the environment it needs, how to verify it, and the two deployment traps that
  have already bitten (stale Gradle task, second-level hostname TLS).
tags:
- tahirs-list
- mcp
- deployment
- cloudflare
- tunnel
- operations
- sc-209
status: active
generated: 2026-11-08
generated_by: hamza
verified: true
sources:
- resource: docs/handoff/05-operations-and-infrastructure.md
  title: Deploy path and dev-stack command
- resource: scripts/backend-live.sh
  title: The supported dev-stack launcher (:bootstrap:run)
- resource: https://github.com/Losers-Lab/tahirs-list/pull/76#issuecomment-6723362837
  title: Post-merge dev verification evidence
permalink: tahirs-list-kb/operations/mcp-deployment
---

# MCP Endpoint Deployment and Verification

**Established 2026-11-08.** The MCP endpoint runs inside the normal Ktor backend;
there is no separate MCP container. What it needs from the platform is a hostname
route to the backend and three environment values.

## Endpoints

| Environment | URL | Routes to |
|---|---|---|
| Dev | `https://mcp-dev.tahirslist.com/mcp` | Cloudflare tunnel → `http://tahirs-list-backend:8080` |
| Prod (planned) | `https://mcp.tahirslist.com/mcp` | same shape, prod tunnel |

`dev.tahirslist.com` is the **frontend**; its proxy forwards only `/v1/*`, so
`https://dev.tahirslist.com/mcp` returns 404 by design. The MCP surface has its own
hostname and does not pass through the SvelteKit proxy.

## TLS constraint that chose the name

`tahirslist.com` is on the Cloudflare **Free** plan. Universal SSL covers the apex
and `*.tahirslist.com` only — one level. A second-level name such as
`mcp.dev.tahirslist.com` (or `dev.mcp.tahirslist.com`) has no certificate and the
handshake fails (`SEC_E_ILLEGAL_MESSAGE`). `mcp-dev.tahirslist.com` and
`mcp.tahirslist.com` are first level and work. Keeping `mcp.dev` would require
Total TLS / Advanced Certificate Manager.

## Dev-stack environment

The backend container must carry (in addition to the usual DB/S3 variables):

```text
APP_PROFILE=live
APP_MCP_PUBLIC_WEB_BASE_URL=https://dev.tahirslist.com
APP_MCP_PUBLIC_API_BASE_URL=https://dev.tahirslist.com
APP_MCP_ALLOWED_HOSTS=mcp-dev.tahirslist.com
```

The campaign base URLs stay on the **frontend** host because `imageUrl` and
`websiteUrl` are served there. Only `APP_MCP_ALLOWED_HOSTS` names the MCP host.

## Two traps already observed

1. **Wrong Gradle task.** The deployed backend was started with
   `./gradlew --no-daemon :bootstrap:bootRun` — a Spring-era task that does not
   exist in the Ktor build. The container crash-looped and the preview returned
   502. The correct task, used by `scripts/backend-live.sh` and the handoff doc, is
   **`:bootstrap:run`**. The redeploy worker lives outside this repository; until it
   is corrected, every redeploy breaks the backend again.
2. **Shared rate-limit bucket behind the tunnel.** With
   `APP_RATELIMIT_TRUST_PROXY=false` (the default) the limiter keys on the socket
   peer, which behind the tunnel is cloudflared — so every agent shares one 60/min
   budget. Per-client budgeting needs `true`, and that is only safe when the origin
   is reachable **only** through the tunnel (currently the backend port is also
   published on the host, so this must be tightened first).

## Verification

```bash
curl -s https://mcp-dev.tahirslist.com/v1/health
# {"status":"ok"}

curl -s -X POST https://mcp-dev.tahirslist.com/mcp \
  -H 'Content-Type: application/json' \
  -H 'Accept: application/json, text/event-stream' \
  -d '{"jsonrpc":"2.0","id":1,"method":"initialize","params":{"protocolVersion":"2025-06-18","capabilities":{},"clientInfo":{"name":"check","version":"1"}}}'
```

Then `tools/list` (expect exactly two tools) and `tools/call` with
`{"name":"search_restaurants","arguments":{"place":"Toronto","radiusMiles":10}}`.
A malformed body must return
`{"jsonrpc":"2.0","id":null,"error":{"code":-32700,"message":"Parse error"}}`
with no parser text. `GET /mcp` must answer `405` with `Allow: POST, OPTIONS`.

## Cloudflare tunnel facts

The dev tunnel is **token-managed** (remote configuration), not a local
`config.yml`. Ingress is read and written through
`/accounts/{account_id}/cfd_tunnel/{tunnel_id}/configurations`; the current dev
ingress is `mcp-dev.tahirslist.com → backend:8080`, then
`dev.tahirslist.com → frontend:3000`, then the `http_status:404` catch-all. A
public hostname needs both the ingress rule and a proxied CNAME to
`{tunnel_id}.cfargotunnel.com`; the API token used for the route did not carry
`DNS: Edit`, so the record was added in the dashboard.
