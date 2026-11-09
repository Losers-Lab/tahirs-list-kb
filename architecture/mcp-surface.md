---
title: MCP Server Surface (Agent Access)
type: architecture-note
permalink: tahirs-list-kb/architecture/mcp-surface
description: The embedded, public, stateless MCP endpoint that lets AI agents search
  Tahir's List and read one listing, plus the admission, rate-limit, Origin, Host
  and body-size controls that keep an anonymous write-free surface safe.
tags:
- tahirs-list
- mcp
- agents
- architecture
- security
- ktor
- sc-209
status: active
generated: 2026-11-08
generated_by: hamza
verified: true
sources:
- resource: docs/features/SC-209/00-story.md
  title: SC-209 story and acceptance criteria
- resource: docs/features/SC-209/07-class-diagram-complete.md
  title: Complete class diagram, config table and traceability
- resource: backend/adapters/mcp
  title: MCP adapter source (McpServer.kt, tools, converter)
- resource: https://github.com/Losers-Lab/tahirs-list/pull/76
  title: Implementation PR (merged as f6b82d3)
---

# MCP Server Surface (Agent Access)

**Established.** SC-209 shipped an embedded MCP server so AI agents can find halal
restaurants and read one listing. It is not a separate service: the `:mcp` Gradle
module is a **driving adapter** inside the same Ktor application, mounted at
`POST /mcp`, and it reuses the application/domain read paths.

## Shape

| Concern | Decision |
|---|---|
| Protocol | MCP Streamable HTTP, **stateless** (no session id), JSON responses |
| Methods | `POST` only; `GET`/`DELETE` answer `405` with `Allow: POST, OPTIONS`; `OPTIONS` is CORS preflight |
| SDK | `io.modelcontextprotocol:kotlin-sdk` 0.16.0 |
| Tools | exactly `search_restaurants`, `get_restaurant` |
| Auth | none — public and anonymous by design; read-only |
| Serialization | MCP payloads use the SDK's `McpJson` (no explicit nulls); every other response keeps the REST `Json` (explicit nulls) via `McpAwareJsonConverter` |
| Module | `backend/adapters/mcp`, wired in `bootstrap` |

## Tool contract

- `search_restaurants`: exactly one of `place` (free text) or `center` (lat/lng);
  `radiusMiles` required; optional `handCutOnly`, `deliveryOnly`, `cuisines`
  (AND/OR via `cuisineLogic`), `minPrice`/`maxPrice`, `minRating`, `limit`
  (1–100, default 20) and `offset` (≤ 10 000). Results are ordered by distance and
  carry `nextOffset` for paging.
- `get_restaurant`: a listing id; returns address parts, cuisine, tri-state
  `isHandCut`/`isDelivery`/`rating`, `halalScope`, per-item `halalItems`,
  `crossContamination`, `verificationStatus`, image and website URLs.
- **Certificate facts only when the listing is `VERIFIED`, the review is approved,
  and an archived certificate key exists** — an unapproved or unarchived
  certificate is never advertised.
- Tool errors are actionable `isError: true` results (validation, unknown id,
  geocoder outage). Internal detail is never reflected: only parse failures are
  echoed, and those are caller-safe.
- Listing text is community-sourced; both tool descriptions instruct the model to
  treat it as data, never as instructions.

The card/detail field names are the public agent contract; the JSON Schemas declare
the genuinely nullable fields as `["<type>", "null"]`.

## Controls on an anonymous endpoint

| Risk | Control |
|---|---|
| Abuse / cost | Per-client Bucket4j budget (default 60 requests / 60 s; capacity, refill and key count configurable). Refusal happens before any DB or geocoder work and returns `429` + `Retry-After`. |
| Batch amplification | One POST token covers the first tool call; every further call in a JSON-RPC batch must consume another token, so a batch cannot multiply work. |
| Oversized bodies | 16 KiB request cap on the transport (`413`), which also bounds cheap-method batch expansion to ~1.6 MB. |
| Cross-origin browsers | Exact-match `Origin` allow-list (`APP_MCP_ALLOWED_ORIGINS`); a present but unlisted Origin gets `403`, requests with no Origin are allowed for non-browser agents; CORS headers only echo an allowed origin. |
| DNS rebinding | SDK `DnsRebindingProtection` with allowed hosts = loopback + `APP_MCP_ALLOWED_HOSTS` + hosts parsed from the public base URLs. |
| Parse-error leakage | Malformed bodies are replaced by a fixed `-32700` envelope, matched only at the start of the response and only when the outgoing status is `400`. |
| Client-key spoofing | `CF-Connecting-IP`/`X-Forwarded-For` are trusted **only** when `APP_RATELIMIT_TRUST_PROXY=true`; otherwise the socket peer keys the bucket. Refusal logs carry a salted hash, never the IP. |
| Error hygiene | No exception text, SQL, path or stack trace reaches the caller; unexpected failures log server-side and return a fixed message. |

## Configuration

| Variable | Default | Purpose |
|---|---|---|
| `APP_MCP_ENABLED` | `true` | `false` leaves `/mcp` unmounted (404, not public) |
| `APP_MCP_PUBLIC_WEB_BASE_URL` | `http://localhost:5173` | `websiteUrl` in results |
| `APP_MCP_PUBLIC_API_BASE_URL` | `http://localhost:8080` | `imageUrl` / certificate URL |
| `APP_MCP_ALLOWED_ORIGINS` | empty | exact browser origins |
| `APP_MCP_ALLOWED_HOSTS` | empty | extra Host names for DNS-rebinding protection |
| `APP_GEOCODER_BASE_URL` | `https://photon.komoot.io/api/` | Photon endpoint |
| `APP_MCP_RATELIMIT_CAPACITY` / `_REFILL_TOKENS` / `_WINDOW_SECONDS` / `_MAX_KEYS` | 60 / 60 / 60 / 100 000 | budget shape |
| `APP_RATELIMIT_TRUST_PROXY` | `false` | honour proxy client-IP headers |

The `prod` profile refuses to start when MCP is enabled and the three public URLs
are not absolute `https` or still point at loopback.

## Verification

- `:mcp:test` — 55 tests: discovery, transport, Origin/CORS, rate-limit and batch
  gating, sanitized errors, both tools, config wiring.
- `McpSearchPostgisTest` — 7 integration tests against real PostGIS: ordering,
  tri-state hand-cut, delivery, cuisine AND/OR, price/rating exclusion, paging,
  cross-contamination index gate.
- `McpPublicAccessTest` / `McpRegressionTest` — no JWT needed; REST read behaviour
  unchanged.
- One gated live test (`PhotonGeocoderLiveTest`, `LIVE_TESTS=1`) hits the real
  Photon API; the default suite skips it so no test depends on a third party.
- Full backend suite at merge: 668 tests, 667 pass, 1 skipped (the live test).

## Open items and residuals

See [SC-209 review record](../reviews/sc-209-mcp-review.md) and
[MCP deployment](../operations/mcp-deployment.md). Known, accepted limits: a 16 KiB
body can still expand to ~1.6 MB of `tools/list` replies; non-representable JSON
numbers are refused by the SDK as `-32602` before the tool runs; the per-node
limiter is not shared across replicas; and per-client budgeting behind the tunnel
requires `APP_RATELIMIT_TRUST_PROXY=true` with an origin reachable only through the
tunnel.