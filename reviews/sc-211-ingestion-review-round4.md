---
title: SC-211 - Ingestion Review Round 4
type: review-record
permalink: tahirs-list-kb/reviews/sc-211-ingestion-review-round4
description: 'Rounds 4-5 of the security/code review of PR #82 (SC-211 ingestion workflow):
  APPROVE at 9d2c995 (B5/B6 fixed, five non-blocking findings) and the final APPROVE
  at 5e3ebab, where reattachClaim recomputes and mirrors both endpoints of a claim
  move, closing finding 1. Includes the source==destination idempotency and later-write
  ordering analysis, the remaining residuals, the deferred items and the build evidence.'
tags:
- tahirs-list
- review
- security
- ingestion
- sc-211
- hand-cut
status: active
generated: 2026-11-08
generated_by: omar
verified: true
sources:
- resource: https://github.com/Losers-Lab/tahirs-list/pull/82
  title: Implementation PR (head 5e3ebab, open)
- resource: https://github.com/Losers-Lab/tahirs-list/pull/82#issuecomment-6727915072
  title: Round 4 review comment
- resource: https://github.com/Losers-Lab/tahirs-list/pull/82#issuecomment-6728153844
  title: Round 5 final review comment (record of record)
---

# SC-211 - Ingestion Review Round 4

**Round 4: `Verdict: APPROVE` on head `9d2c995`** (PR comment
https://github.com/Losers-Lab/tahirs-list/pull/82#issuecomment-6727915072).
Delta reviewed: `715b4d3..9d2c995` (one commit, "Fix attach-to trust and curated cross
value"): `ExposedRestaurantListingRepository.kt`, `ExposedIngestionBatchTest.kt`,
`RunSampleIngestion.kt`, `RunSampleIngestionTest.kt`. **No doc change in the delta, and
none was needed**: the code now matches `07-class-diagram-complete.md:107/138` and BR-9
(`02-use-cases.md:69`), which already stated the rule. Records:
[round 1](sc-211-ingestion-review.md), [round 2](sc-211-ingestion-review-round2.md),
[round 3](sc-211-ingestion-review-round3.md). Model:
[ingestion runtime load](../research/ingestion-runtime-load-2026-11-08.md).

## Round 5 (final): `Verdict: APPROVE` on head `5e3ebab`

(PR comment https://github.com/Losers-Lab/tahirs-list/pull/82#issuecomment-6728153844.)
Delta `9d2c995..5e3ebab` (one commit, "Recompute both sides of a claim move"):
`ExposedRestaurantListingRepository.kt` (+6/-2), `ExposedIngestionBatchTest.kt` (+19/-4).
Closes **non-blocking finding 1** below and Yusuf's round-4 destination half of B5.

- **Both endpoints now project**: `reattachClaim` (`:313-348`) moves the claim, then
  recomputes + mirrors the source (`:344-345`) **and** the destination (`:346-347`), so
  the invariant lives in the port instead of depending on the caller. The destination
  call passes no manual cross value, so a curated row that gains no Tier-1 claim keeps
  its explicit value (early return at `:600`), while a moved Tier-1 claim applies D-16
  (`VERIFIED`, `CERTIFIER_REGISTRY`, hand-cut from that declaration,
  `NO_CROSS_CONTAMINATION`) and re-enters `listing_search` through the gate (`:122`).
- **Source == destination is unreachable and would be idempotent**: `ClaimReattachment.init`
  refuses it (`Findings.kt:114-116`); if it were reachable both calls are the same
  projection, and the withdrawal branch cannot oscillate (basis becomes `NULL`, so the
  second pass stops at `:600`). Chained moves and two moves into one destination are safe
  because every move recomputes both of its endpoints.
- **Destination later in the same batch**: order is `reattachments -> merges -> writes`
  (`:266-267`); the write's recompute (`:290-297`) re-reads the claim set from the
  database, so it supersedes the reattachment projection instead of compounding it, and
  re-mirrors post-write identity (`:297`). A write that would give a moved claim back to
  its old owner fails closed in `assertClaimsBelongTo` (`:681-688`). A merge in either
  role stays correct (`:439-451`).
- **No new reachable outcome**: `attach-to` must name the listing the venue resolves to
  (`LoadIngestionBatch.kt:266-271`), which is the batch's write target, so the destination
  was already recomputed there; this is defense in depth plus correctness for a
  repository-level reattachment — the shape the test exercises (`writes = emptyList()`).
- **Evidence**: targeted `:persistence:test --tests ExposedIngestionBatchTest` on a copy
  of `backend/` at `5e3ebab` (container `omar85`, cold volume
  `tahirs-list-gradle-omar85`, Testcontainers sibling PostGIS, docker.sock +
  `host.docker.internal:host-gateway`): **BUILD SUCCESSFUL in 3m**, **18 tests / 0
  failures / 0 errors / 0 skipped / 21.028s** (22:39:16Z), 18 `PASSED` including the
  reattachment-only destination assertions (`:550`, `:571-578`). The delta touches no
  other module and adds no test; round 4's full `build` (124 suites / 903 tests at
  `9d2c995`) remains the whole-suite evidence, and this run compiled `domain`,
  `application` and `persistence` from the `5e3ebab` copy.
- **Residual, non-blocking**: the PR body still does not name SC-225 — the round-4
  condition is met by hamza's close-out comment (22:34:14Z), but the link belongs in the
  body; findings 2, 3 and 5 stay open by design; `09-decisions-and-metros.md:57` no longer
  states that the re-point itself refreshes both rows.

## Blockers: none — B5 and B6 fixed in the adapter

- **B5 (a claim moved away left the source `VERIFIED` with zero claims).**
  `reattachClaim` recomputes the *from* listing and refreshes its mirror immediately after
  the claim UPDATE (`ExposedRestaurantListingRepository.kt:340-343`). With basis
  `CERTIFIER_REGISTRY` and no winning claim the withdrawal branch (`:576-590`) sets
  `UNVERIFIED` / basis `NULL` / hand-cut `NULL` / cross `UNCERTAIN`, and the mirror gate
  (`:167-174`) deletes the `listing_search` row. The test now asserts the emptied source
  and its missing index row (`ExposedIngestionBatchTest.kt:514-565`), and a replayed
  reattachment fails the batch (`:652`) instead of re-pointing other evidence.
- **B6 (a curated cross-contamination value was dropped on update).** `applyBatch` passes
  the candidate's explicit value to the recompute when the candidate has no Tier-1 winner
  (`:290-297`) and the no-winner path writes it (`:592-605`). All four combinations hold:
  Tier-1 winner forces `VERIFIED` + `CERTIFIER_REGISTRY` + hand-cut from that declaration +
  `NO_CROSS_CONTAMINATION` and never reads the manual value (D-16); no winner + curated
  value persists and drives the index; no winner + withdrawal + no value falls back to
  `UNCERTAIN` and leaves the index; committee trust (basis `NULL`) is not demoted because
  the status/basis/hand-cut write is gated on basis `CERTIFIER_REGISTRY`. Pinned by
  `ExposedIngestionBatchTest.kt:567-612`, which walks `UNCERTAIN -> NO` (indexed),
  `NO -> PRESENT` (delisted) on a curated row.

## Verified sound

- **Ordering**: `reattachments -> merges -> writes -> recompute -> mirror` with a mirror
  refresh on every trust-changing path (write loop `:290-297`, reattached source
  `:342-343`, merge keeper `:435-436` before the retire mirror row and row are deleted
  `:438-447`). No path leaves a stale mirror row or stale trust after the commit. The one residual
  here (the destination not refreshed on a write-less reattachment) failed closed at
  `9d2c995` and is closed in round 5.
- Reattachment then merge composes in either role: the claim lands on the keeper and the
  keeper is recomputed from the union.
- **Provenance fallback** (`:553-556`, bound `:569`): an unrecognised source no longer
  writes a false `certifier-hms` label, and `provenance` is nullable (`Tables.kt:106`) so
  the null bind is legal.
- **Delta hygiene**: all SQL bound, no new external input or I/O surface, one transaction,
  full rollback; the `RunSampleIngestion` change removes a double count (an unlocated row
  is a reject, not also an "outside every metro" gap) without hiding real gaps, and the new
  unselected-metro gap is derived from configured metros and feed ids, not page content.
- No test was weakened; the old combined `attach-to` test was split into two stronger ones.

## Non-blocking findings

1. **Closed in round 5 (`5e3ebab`).** `reattachClaim` refreshed only the source (`:340-343`). The destination gains a Tier-1
   claim but is recomputed only because the use case always writes that listing
   (`09-decisions-and-metros.md:57`); a repository-level reattachment with no write — the
   shape of the new test at `ExposedIngestionBatchTest.kt:543-553` — leaves it `UNVERIFIED`
   and unindexed. Fail-closed; a second recompute + mirror is idempotent and closes it.
2. `IngestionCandidate.manualCrossContamination` defaults to `UNCERTAIN`
   (`IngestionModel.kt:137`), so an update rewrites the stored cross value even on a row
   whose trust the batch does not own; a re-curated row that omits the field delists a
   previously qualified venue. Spec-conformant (BR-9's default is authoritative) but
   invisible in `diffSummary()`; make the curated field nullable to distinguish "not
   restated", and show the stored->candidate cross change in the reviewed diff.
3. `mergeListings` recomputes the keeper with no manual value (`:435`), so a merge drops a
   curated `NO_CROSS_CONTAMINATION` when the keeper is `UNCERTAIN`: claims union, non-claim
   evidence does not.
4. **Structurally prevented, not by the adapter.** `mergeListings` (`:356-448`) has no
   local guard, but `ListingMerge.init` already refuses `keepId == retireId`
   (`Findings.kt:102-104`, present at `9d2c995` too), so no equal-id merge value can exist;
   the adapter relies on its input type here.
5. The unknown-source fallback can write `VERIFIED` while keeping a `NULL` or
   non-certifier provenance (`:555-569`), bypassing the domain invariant that `VERIFIED`
   implies a certifier-registry provenance. Unreachable today (`CandidateValidator.kt:41-49`
   rejects unknown sources at preview, and `candidate.provenance` would throw first);
   failing the batch on a Tier-1 winner with no provenance stamp would fail closed instead.

## The four deferred items

Accepted as deferrable, with one condition. Merge cycles failing closed, the unused
`verification_basis` vocabulary, and the ~122-row geocoding cost are acceptable (fail-closed
by design, no reader yet, bounded and cached). Items 1 and 3 are the same root: the DB
cannot say "the committee approved this once" — basis `NULL` means both "committee" and
"no basis", and `:576-590` depends on that. Deferring is fine only as a **tracked story**
referenced on the PR before merge; otherwise the next certifier refresh demotes a
committee-verified venue with no record of why.

## Evidence

Containerised `./gradlew build` on a **copy of `backend/` outside the repo worktree** at
`9d2c995` with all `build/`, `.gradle` and `.kotlin` outputs removed first
(`eclipse-temurin:21-jdk`, container `omar84`, `--memory=3g --cpus=2`, Testcontainers
sibling PostGIS, docker.sock as in earlier rounds, **cold** Gradle home volume
`tahirs-list-gradle-omar84`): **BUILD SUCCESSFUL in 9m 2s**, 71 actionable tasks
(**71 executed, none from cache**). **124 suites / 903 tests / 0 failures / 0 errors /
1 skipped** (`PhotonGeocoderLiveTest`, `LIVE_TESTS=0`), all 124 XMLs written by this run
(17:10:44–17:16:12 local). Round 3 was 124 / 899; the +4 are exactly the four added tests
(`ExposedIngestionBatchTest` 16→18, `RunSampleIngestionTest` 8→10). The run's sibling
containers were cleaned up and no container from this run remains.

## Observations

- [omar] Trust projection is only complete when every listing whose trust columns change also gets its mirror refreshed; a repository method that moves evidence between two rows must recompute both or the invariant depends on its caller #sc-211 #trust #review
- [omar] A candidate field with a non-null default cannot express "the source did not restate this", so a re-load silently overwrites stored evidence with the default; make such fields nullable when "absent" and "default" mean different things #sc-211 #data-model
- [omar] A merge unions claims but not non-claim evidence, so cross-contamination and committee verification are still lost on same-venue — the same class of defect each round #sc-211 #merge #trust
- [omar] Basis NULL carries two meanings (committee approved / no basis), which is why a committee verification cannot be kept across a certifier refresh; that conflation is the root of the deferred basis story #sc-211 #schema #risk
- [omar] A defensive fallback that preserves a row's existing provenance can still leave a VERIFIED row whose provenance does not confer verification; failing the batch is the fail-closed option #sc-211 #trust #review
- [omar] A recompute that projects from stored state is idempotent, so making a move refresh both endpoints costs one duplicate write and removes an ordering dependency on the caller; an accumulator recompute would not be safe to call twice #sc-211 #trust #idempotency
- [omar] An invariant that only holds because the caller writes the other row is not an invariant; the test that drives the port alone (writes = emptyList()) is what proves the port owns it #sc-211 #testing #review

## Relations

- relates_to [SC-211 ingestion review, round 3](sc-211-ingestion-review-round3.md)
- relates_to [SC-211 ingestion review, round 2](sc-211-ingestion-review-round2.md)
- relates_to [SC-211 ingestion review, round 1](sc-211-ingestion-review.md)
- relates_to [ingestion runtime load model](../research/ingestion-runtime-load-2026-11-08.md)
- relates_to [review order](../process/review-order.md)