---
title: SC-209 - MCP Server Review Record
type: review-record
permalink: tahirs-list-kb/reviews/sc-209-mcp-review
description: How the MCP server PR was reviewed, the blocking defects each round found,
  what was fixed versus accepted, and the evidence behind the final approval.
tags:
- tahirs-list
- review
- security
- qa
- mcp
- sc-209
status: active
generated: 2026-11-08
generated_by: hamza
verified: true
sources:
- resource: https://github.com/Losers-Lab/tahirs-list/pull/76
  title: Implementation PR (merged as f6b82d3; approved head c0e7813)
- resource: docs/features/SC-209/10-traceability.md
  title: Round-by-round traceability
- resource: docs/features/SC-209/00-story.md
  title: Acceptance criteria
---

# SC-209 - MCP Server Review Record

**Converged 2026-11-08.** Omar (security, static) and Yusuf (QA, dynamic) each
posted `Verdict: APPROVE` on head `c0e7813`; the PR merged as `f6b82d3`. This is
the second PR reviewed under the corrected order in
[Review order](../process/review-order.md).

## Rounds

| Round | Reviewer | Blocking findings | Resolution |
|---|---|---|---|
| 1 | Omar | Plain `IllegalArgumentException` from the shared parser escaped as a generic failure and logged a stack trace; JSON-RPC batches bypassed the per-client budget; SDK parse errors echoed parser text | Typed catch scoped to parsing; per-call budget token so a batch cannot amplify; fixed parse-error envelope |
| 1 | Yusuf | The parse-error sanitizer matched the wrong envelope shape and was dead code; its test passed for the wrong reason | Match the real `-32700` envelope; assert absence of parser internals |
| 2 | Omar | Batches of non-tool methods could still materialise ~430 MB from one POST | 16 KiB body cap (bounds it to ~1.6 MB) and recorded acceptance |
| 3 | Omar | Residual amplification still ~6.4 MB at 64 KiB; unanchored matcher; missing regression test for the parse/execute split | Cap lowered to 16 KiB; anchored + 400-status-gated matcher; masked-IAE test |
| 4 | Omar | The new spoof test was vacuous | Direct unit tests on the matcher plus an end-to-end test that a listing containing `-32700` stays a 200 |
| 2 | Yusuf | Validation failures stopped being logged after the parse/execute split; the search schema omitted four address fields and typed nullable fields as non-null | Parse back inside the timed block; full schema with `["T","null"]` unions; `center.lat` label |
| final | both | Address fields still declared non-nullable; the anchor had no discriminating test | Nullable union types via one shared helper; anchor pinned and proven by mutation (deleting `^` fails the suite) |

## Accepted, not fixed

- A 16 KiB POST can expand to ~1.6 MB of `tools/list` replies — constant, not
  attacker-scaled; the largest legitimate body is ~1.7 KB.
- Non-representable JSON numbers (`1e400`) are refused by the SDK as `-32602`
  before the tool runs; AC-4's "tool error" applies to representable values.
- No automated log-capture test; call logging (`tool`, `outcome`, `durationMs`,
  hashed client key) was verified with a throwaway appender harness.
- `CF-Connecting-IP` trust is deployment-coupled: it is honoured only under
  `APP_RATELIMIT_TRUST_PROXY=true`, and the origin must then be tunnel-only.
- The refusal-log key hash is a salted 32-bit `hashCode` — a debug breadcrumb.
- Exact distance ties are not order-stable (pre-existing query behaviour).
- Q-3, Fatima's sign-off on the LLM-facing wording, has no recorded decision; the
  shipped text matches the approved design literals.

## Evidence at the approved head

- Full backend suite: 668 tests, 667 pass, 1 skipped (the gated live Photon test).
- `:mcp` 55 tests, `:bootstrap` 196 tests.
- Live Photon test passes with `LIVE_TESTS=1`; with `--network none` the same test
  fails, proving it really reaches the external service.
- Post-merge dev check: initialize, `tools/list`, a live Toronto search, detail,
  sanitized malformed body, `405` on GET, `403` on a disallowed Origin — all on
  `https://mcp-dev.tahirslist.com/mcp`.