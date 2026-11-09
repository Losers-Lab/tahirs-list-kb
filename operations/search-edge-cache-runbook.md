---
title: Search edge cache — runbook and accepted staleness
type: operation
permalink: tahirs-list-kb/operations/search-edge-cache-runbook
description: How the public location-search response is cached at the edge (SC-199),
  what must be true before a Cloudflare Cache Rule is applied, and the staleness the
  founder accepted on 2026-11-09.
tags:
- tahirs-list
- operations
- cloudflare
- caching
- sc-199
status: planned
---

# Search edge cache — runbook and accepted staleness

> **Status: rule not applied yet.** SC-199 ships the origin header; the Cloudflare
> Cache Rule is WI-8 and is still an operator action. Nothing here claims edge
> caching is live.

## What the origin sends

`GET /v1/listings/search` returns, on a **200 only**:

```text
Cache-Control: public, max-age=60, s-maxage=300, stale-while-revalidate=60
```

Every non-200 is `no-store`. Both channels are bounded so that
`max-age + swr <= 600` and `s-maxage + swr <= 600`; the process refuses to start
with a larger window (`APP_SEARCH_*` variables).

## Before applying the Cache Rule

1. **Key on host + the full query string.** Cloudflare does not cache JSON by
   default; a Cache Rule is required ("Cache eligibility: eligible for cache").
   Ignoring any query parameter would cross-serve different filter combinations.
2. **Do not add query-param normalisation.** The key is the client's URL by design
   (SC-199 D-3). Rounding happens at the origin and does not merge edge keys.
3. **Absolute URLs in the body.** Each card's image URLs are built from the
   request's host/scheme. Cloudflare's default key includes the host, so one
   host's body is not served under another. Do not disable that part of the key.
4. **Verify after applying:** two requests to the identical URL, second one
   `cf-cache-status: HIT`; a different precision in `center` must MISS.

## Accepted staleness (founder, 2026-11-09)

- **Verified badge on search cards may be stale for up to the window** (≤ 360 s
  with defaults). SC-199 D-6, option A. There is no strict SLA.
- **The certificate and image proxies stay at `public, max-age=86400`** (24 h).
  Accepted as-is; no follow-up story (SC-199 D-7).

This note is an explicit, dated exception to §11 of
[Scalability and Reliability Goals](../architecture/scalability-and-reliability-goals.md)
("invalidate on verification transitions") and to the §5.2 five-second target for
**these two surfaces only**. The detail page remains uncached and read-live.