---
title: SC-211 - Ingestion Workflow Review Record
type: review-record
permalink: tahirs-list-kb/reviews/sc-211-ingestion-review
description: 'Round 1 security/code review of PR #82 (SC-211 ingestion workflow):
  verdict, blocking defects (already-loaded run re-applied; upgrade demotes a verified
  listing), non-blocking findings and the evidence at head.'
tags:
- tahirs-list
- review
- security
- ingestion
- sc-211
status: active
generated: 2026-11-08
generated_by: omar
verified: false
sources:
- resource: https://github.com/Losers-Lab/tahirs-list/pull/82
  title: Implementation PR (head 2c348f1, open)
---

# SC-211 - Ingestion Workflow Review Record

**Round 1: `Verdict: CHANGES REQUESTED` on head `2c348f1`** (comment
https://github.com/Losers-Lab/tahirs-list/pull/82#issuecomment-6725426804).
Yusuf's QA round is separate; this note records the static/security round.
Context: [ingestion runtime load model](../research/ingestion-runtime-load-2026-11-08.md).

## Blocking findings

| # | Defect | Why it blocks | Fix direction |
|---|---|---|---|
| B1 | `LoadIngestBatchMain` calls `execute` then `markLoaded`; `FileRunArtifacts.readCandidates` never checks `loaded.json` (only `findPending` filters it) | A second `--execute` re-applies the whole batch (upgrading every listing from a stale artifact) and *then* throws `RunAlreadyLoadedException`: committed write + failure exit. Contradicts UC-03 A2 / BR-18 and the CLI's own "will not be loadable again" | `refuseIfLoaded(runId)` before `execute`; test that a second execute never reaches `applyBatch` |
| B2 | `updateImported` unconditionally writes `verification_status`, `is_hand_cut`, `cross_contamination`, `provenance` on every matched update | A Tier-3 manual candidate that matches an ingested Tier-1 listing by `venue_key` flips it VERIFIED -> UNVERIFIED, hand-cut -> NULL, cross-contamination -> UNCERTAIN, and the mirror deletes its `listing_search` row: a verified venue silently leaves the public index, with no warning in the review diff. Against D-10's no-auto-demotion stance | Never downgrade VERIFIED when the candidate is UNVERIFIED, or surface the demotion explicitly; add the missing test |

Related: `insertImported` uses `RestaurantListing.fromStorage` instead of
`RestaurantListing.imported` (D-15), so the domain guard "VERIFIED requires a
certifier-registry provenance" is not on the write path; `CandidateValidator` in
`preview` is the only guard.

## Non-blocking findings

1. Brand lookup is case-insensitive in `ExposedBrandRepository.resolve`/preview but
   exact-match in `applyBatch.resolveOrCreateBrand`, so a case variant creates a
   second brand row that the exact-name unique index permits.
2. `mergeCandidates` is not transitive (first match only): A~C by venue key plus C~B
   by claim leaves two writes sharing a claim, and the batch aborts on the claim
   conflict. Fail-closed but avoidable.
3. `applyBatch` does not dedupe writes by target listing.
4. `findByNormalizedLocation` cannot use the partial location index (no
   `provenance IS NOT NULL` predicate), so it scans per candidate.
5. `CandidateBatchJson.decode` aborts the whole file on one malformed row instead of
   rejecting that candidate.
6. `MatchConfirmationPolicy` accepts a city-only substring hit (and substring
   region/country), the weakest link in "confirmed location".
7. `load-ingest-batch.sh` passes the DB password via `-e` (visible to
   `docker inspect`) and interpolates `RUN_ID` into `-PingestArgs`, which Gradle
   splits on spaces.
8. Artifact trust boundary: the manifest checksum is integrity, not authenticity,
   and the default run directory is inside the repo. Anyone able to write a run
   directory can have a hand-written Tier-1 claim loaded as VERIFIED; `report.md`
   does not list candidates or claims, so the curator's review cannot spot it.

## Sound at this head

SQL is fully bound; the DB enforces claim/venue/location uniqueness (races fail
closed); `applyBatch` is one transaction with the mirror and claim writes; trust
derivation follows the tier and the certifier's own declaration; manual URLs are
cited and never fetched; run-id and snapshot names cannot escape the artifact root;
`markLoaded` runs only after commit.

## Evidence

Containerised `./gradlew build` on `2c348f1`: BUILD SUCCESSFUL; 857 tests,
0 failures / 0 errors, 1 skipped (gated live Photon), 120 suites.

## Environment gotcha

Two agents building the same worktree through the same named Gradle volume deadlock
`/gradle-home/caches/journal-1/journal-1.lock` (the lock owner is the container's own
PID). Wait for the other container to finish rather than deleting its locks; after
it exits, removing stale `*.lock` files and the journal directory unblocks the next
build.

## Relations

- relates_to [ingestion runtime load model](../research/ingestion-runtime-load-2026-11-08.md)
- relates_to [review order](../process/review-order.md)