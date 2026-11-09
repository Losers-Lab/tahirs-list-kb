---
title: SC-211 - Ingestion Review Round 2
type: review-record
permalink: tahirs-list-kb/reviews/sc-211-ingestion-review-round2
description: 'Round 2 security/code review of PR #82 (SC-211 ingestion workflow) at
  head 53c7b16: verdict CHANGES REQUESTED, two blocking trust-projection defects (hand-cut
  cannot follow a certifier withdrawal; a same-venue merge discards the retired row
  trust), non-blocking findings, what was verified sound and the containerised build
  evidence.'
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
  title: Implementation PR (head 53c7b16, open)
- resource: https://github.com/Losers-Lab/tahirs-list/pull/82#issuecomment-6726268633
  title: Round 2 review comment (record of record)
---

# SC-211 - Ingestion Review Round 2

**Round 2: `Verdict: CHANGES REQUESTED` on head `53c7b16`** (PR comment
https://github.com/Losers-Lab/tahirs-list/pull/82#issuecomment-6726268633).
Delta reviewed: `2c348f1..53c7b16` (commits 1edb101 "Fix ingestion review
blockers", 53c7b16 "Configure metros as data, not code"). Round 1 record:
[SC-211 ingestion review](sc-211-ingestion-review.md). Model:
[ingestion runtime load](../research/ingestion-runtime-load-2026-11-08.md).

Round 1's two blockers (B1 already-loaded run applied, B2 Tier-3 upgrade
demoting a VERIFIED listing) are both genuinely fixed. The two new blockers are
the same harm class as B2, reached through code added in this delta.

## Blocking findings

| # | Defect | Where | Why it blocks |
|---|---|---|---|
| B3 | `TrustMerge.handCut` is a TRUE-wins lattice, so a certifier's own newer declaration can never lower a stored TRUE | `TrustMerge.kt:30-34` used at `ExposedRestaurantListingRepository.kt:453-456`; `writeClaim` updates `hand_cut_declaration` at `:610-629` | On the ordinary refresh path (same claim key re-fetched, venue matched by key ⇒ Update), a certifier changing from hand-cut to not-hand-cut/silent leaves `restaurant_listings.is_hand_cut = TRUE` (and the mirror) while its own Tier-1 claim row says `NOT_HAND_CUT`/`UNSPECIFIED`. Violates the AGENTS hand-cut rule ("TRUE only when the certifier declares hand-cut") and D-16; overstates a religious dietary attribute, silently. Round 1 only required that a *Tier-3 manual* row must not clear a known value; a real FALSE from the certifier is not "unknown". |
| B4 | `mergeListings` moves claims/children but never reconciles the keeper's trust | `ExposedRestaurantListingRepository.kt:344-431` (deletes the retired mirror row at `:421-430`) | `same-venue` on a claim conflict may keep the Tier-3 manual row (`LoadIngestionBatch.kt:279-295` allows either side). The keeper stays UNVERIFIED + UNCERTAIN (never index-qualified), the verified row's `listing_search` row is deleted, and its Tier-1 claim now hangs off the keeper: the venue silently disappears from the public catalogue. `diffSummary()` shows only `merged: 1` and `same-venue → <id>`. Also leaves an APPROVED review on an UNVERIFIED listing. |

Fix direction for both: make the listing's trust columns a *projection of the
claim set* (recompute after the claims are written/moved) instead of a lattice
merge of a stale column. That keeps B2 fixed — a manual candidate carries no
Tier-1 claim, so it cannot clear a known value — while letting a certifier
withdraw or change its own declaration, and makes a merge keep the strongest
evidence. Surface before/after trust in the reviewed diff.

## Non-blocking findings

1. **Cross-candidate plan divergence.** Merge retirement is resolved only inside
   one candidate's plan (`LoadIngestionBatch.kt:299-310`); a merge decided for
   candidate A can retire a listing candidate B targets with `Update(retireId)`,
   and execute then fails at `ExposedRestaurantListingRepository.kt:448`
   ("vanished between preview and apply") after preview reported a resolved,
   loadable diff. Fail-closed (one transaction) but the reviewed diff was wrong.
2. **Merge can detach an owner.** `:350-355` carries the retired owner only when
   the keeper has none; two owned rows ⇒ one RO/VRO silently loses the venue.
3. **`mergeListings` is a hand-maintained child-table list.** All six current FKs
   are handled, in the right order (RESTRICT/NO ACTION moved first, CASCADE
   children copied and PK-deduped). A future child table is either silent CASCADE
   evidence loss or an aborting NO ACTION; nothing fails at build time.
4. **`CandidateValidator` does not check the geometry it claims to guard.** It
   only requires a non-blank metro (`CandidateValidator.kt:35`); it never
   recomputes `VenueIdentity` nor checks the metro radius, so a hand-edited
   artifact can attach a forged Tier-1 claim to any venue key and mislabel a
   metro. `report.md` now lists candidates and claims, which is the review aid
   the round-1 F8 finding asked for; recomputing the venue key in preview would
   close it.
5. **`--execute` is check-then-act.** `LoadIngestBatchMain.kt:66` + DB unique
   indexes: the loser of a concurrent execute fails closed or converges, replay is
   now monotonic, so bounded — advisory lock on the run id would make it
   deterministic. `markLoaded` after commit can leave a committed-but-unloaded
   run; the replay is now idempotent, so acceptable.
6. **`resolutions.json` integrity judged sound.** In-process the preview-time
   answers are applied; across processes decisions are re-validated against live
   state (stale, id, option validation), so an edit can only pick an offered
   option. `writeResolutions` refuses a foreign run and a loaded run;
   `loaded.json` records `resolutionsSha256`. Document that the in-run copy is
   the record of record when `--resolutions=<path>` points outside.
7. **Metro file bounds.** `radiusKm` only `> 0` (`MetroFilter.kt:37`), so a typo
   or hostile `--metros-file` can cover the world; duplicate aliases resolve
   first-wins (`:153`) and the loser's rows are dropped by the radius check.
   Suggest a maximum radius and a duplicate-alias check.
8. **`updateImported` rewrites presentation fields from a lower-tier row**
   (`:466-491`); a Tier-3 row sharing a venue key can move a verified venue's
   pin (coordinates are not part of the venue key).
9. `CandidateBatchJson.decode` still aborts the whole file on one malformed row
   (`CandidateBatch.kt:50-58`) — round-1 F5, unchanged.
10. `report.md` escapes `|` and newlines only (`IngestReport.kt:123`).

## Verified sound at this head

- Round-1 **B1** fixed: `refuseIfLoaded` before read and before execute
  (`LoadIngestBatchMain.kt:66`, `FileRunArtifacts.kt:180-200`), `markLoaded`
  only after the commit returns; pinned by `FileRunArtifactsTest`.
- Round-1 **B2** fixed for the reported path: `updateImported` goes through
  `TrustMerge` for status / hand-cut / cross-contamination / provenance, the
  mirror is rewritten from the merged row, and `ExposedIngestionBatchTest` pins
  "a Tier-3 manual update never demotes a verified Tier-1 listing".
  `insertImported` now uses `RestaurantListing.imported`.
- Decisions are a closed vocabulary with validation: a decided entry needs a
  reason; `bad-citation` may not name a listing; `same-venue`/`attach-to` must
  name one of the two listings actually in the conflict; run mismatch and unknown
  decision words are hard errors.
- Stale decisions cannot be applied or silently dropped: `decidedKeys() -
  consumed` throws `StaleResolutionException` before the review is returned.
- Merge FK/order effects are correct at the current schema, and reattachments are
  re-validated against the current claim owner at execute time.
- SQL fully bound on every new statement; `findByNormalizedLocation` now carries
  `provenance IS NOT NULL`, matching the partial unique index exactly.
- Metro membership is the confirmed geocoded point, never the directory city
  text; aliases are a whole-word prefilter; an unknown metro is a reject; a
  mistyped `--metros` selection fails the run; no metro names remain in code.
- Round-1 minors all landed: transitive `mergeCandidates`, `dedupeWrites`,
  case-insensitive brand resolution, index-usable location lookup, `report.md`
  candidate table, whole-token geocoder matching, `RUN_ID` validation and
  `--env-file` credential handling.

## Evidence

Containerised `./gradlew build` on a copy of `backend/` at `53c7b16`
(`eclipse-temurin:21-jdk`, Testcontainers sibling PostGIS): **BUILD SUCCESSFUL in
8m 37s**, **125 suites / 895 tests / 0 failures / 0 errors / 1 skipped** (gated
live Photon). Round 1 at `2c348f1` was 857 tests / 120 suites; the delta adds 38
tests and 5 suites.

## Relations

- relates_to [ingestion runtime load model](../research/ingestion-runtime-load-2026-11-08.md)
- relates_to [SC-211 ingestion review, round 1](sc-211-ingestion-review.md)
- relates_to [review order](../process/review-order.md)