---
title: SC-211 - Ingestion Review Round 3
type: review-record
permalink: tahirs-list-kb/reviews/sc-211-ingestion-review-round3
description: 'Round 3 security/code review of PR #82 (SC-211 ingestion workflow) at
  head 715b4d3: verdict CHANGES REQUESTED, round-2 blockers B3/B4 verified fixed,
  two remaining trust-projection defects (attach-to never recomputes the source listing;
  a curated cross-contamination value is never persisted on update), non-blocking
  findings, what was verified sound and the containerised build evidence.'
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
  title: Implementation PR (head 715b4d3, open)
- resource: https://github.com/Losers-Lab/tahirs-list/pull/82#issuecomment-6727386921
  title: Round 3 review comment (record of record)
---

# SC-211 - Ingestion Review Round 3

**Round 3: `Verdict: CHANGES REQUESTED` on head `715b4d3`** (PR comment
https://github.com/Losers-Lab/tahirs-list/pull/82#issuecomment-6727386921).
Delta reviewed: `53c7b16..715b4d3` (commits `9d9c6a1` "Drop metro city aliases;
the radius decides", `715b4d3` "Project trust from claims after every write").
Records: [round 1](sc-211-ingestion-review.md),
[round 2](sc-211-ingestion-review-round2.md). Model:
[ingestion runtime load](../research/ingestion-runtime-load-2026-11-08.md).

## Round-2 blockers: both fixed

- **B3 (hand-cut cannot follow a certifier withdrawal).** `TrustMerge` is deleted;
  after every write the adapter re-derives status, hand-cut, provenance and index
  qualification from the claim set (`recomputeCertifierTrust`). Hand-cut is TRUE
  only for `HAND_CUT`, FALSE for declared-not **and** for silence. Pinned by
  "a certifier withdrawing hand-cut on a refresh is reflected, not frozen at TRUE"
  and "a certifier going silent is FALSE, not a retained TRUE".
- **B4 (merge into a lower-tier keeper discards Tier-1 trust).** `mergeListings`
  moves the claims, then recomputes the keeper and refreshes its mirror before
  deleting the retire row; pinned by a test that asserts status, hand-cut,
  provenance and the `listing_search` row on the keeper.

## Blocking findings

| # | Defect | Where | Why it blocks |
|---|---|---|---|
| B5 | `attach-to` re-points the claim but never recomputes the listing it was taken from (nor, independently, the destination) | `ExposedRestaurantListingRepository.kt:306-332`, called at `:265`; recompute only at `:289` (write target) and `:423` (merge keeper) | A listing whose only claim was Tier-1 keeps VERIFIED + basis CERTIFIER_REGISTRY + hand-cut TRUE + certifier provenance + NO_CROSS_CONTAMINATION and stays in the index with **zero claims** — the B3 harm through a different decision. The existing test moves the claim and asserts nothing about the source row. The feature doc (07:107) states the rule as "after claims are written **or moved**"; reattachment is the only "moved" case not covered |
| B6 | `updateImported` no longer writes `cross_contamination`, and `recomputeCertifierTrust` never derives it without a Tier-1 winner | `ExposedRestaurantListingRepository.kt:456-485`, `:500-576`; `IngestionModel.kt:163-168`; BR-9 (`02-use-cases.md:69`) | The curator's explicit cross-contamination evidence — required by BR-9 and documented as the candidate's value — now reaches the DB only on INSERT. A correction of an existing venue is dropped while preview reports `upgrade: 1`: `UNCERTAIN → NO_CROSS_CONTAMINATION` fails to index a qualified venue; `NO_CROSS_CONTAMINATION → PRESENT` keeps a venue with known cross-contamination indexed. Pre-delta `NO → PRESENT` was already blocked by the NO-wins lattice (QA round 2's product question); this widens the loss to every transition |

## Non-blocking findings

1. **Committee basis is re-armed.** With a Tier-1 claim present, the winner branch
   writes `verification_basis = 'CERTIFIER_REGISTRY'` onto a row the committee
   verified (basis NULL via `updateVerificationStatus`). A later claim move then
   demotes it. Doc 07:138's "never touched" only holds without a Tier-1 claim.
   Storing `COMMITTEE_REVIEW` needs V23's CHECK widened (it allows only NULL or
   `CERTIFIER_REGISTRY`, while `VerificationBasis` has two values).
2. **`verification_basis` is write-only.** No read path selects it
   (`LISTING_COLUMNS`), so a VERIFIED row cannot report why; the DB vocabulary and
   the domain enum disagree. Fine for this story, a gap for verified display.
3. **Merge still loses a committee verification** (approved review moves, status
   does not), leaving an APPROVED review on an UNVERIFIED listing.
4. **Merge chains are not applied in dependency order**: `A→B` plus `B→C` in one
   batch passes preview and fails closed at apply ("merge keeper … does not exist").
5. **Metro selection geocodes before selecting**: every coordinate-less row in every
   feed is geocoded even when its metro is not selected (~122 Photon calls for a
   Dallas-only run with HMA 76 + HMS 46); `PhotonGeocoder` has no rate limit. The
   removed alias prefilter used to absorb this cost.
6. **A geocoder miss is double-counted** as both a NO_GEOCODER_MATCH reject and an
   "outside every configured metro" gap.
7. **Provenance fallback** labels an unrecognised claim source `certifier-hms`
   instead of failing closed (defensive only: `CandidateValidator` rejects unknown
   sources at preview).
8. `redirectWrites` is invisible in `diffSummary()`; the report shows the
   candidate's intended trust, not the stored row, so a TRUE→FALSE hand-cut flip
   still cannot be seen in the reviewed diff.
9. Round-2 leftovers unchanged: `CandidateValidator` never checks the metro
   geometry it guards; a metro radius has no upper bound.
10. Product question, not this PR: a certifier *removing* a venue from its directory
    cannot lower trust — claims are never deleted and `expires_on` is never
    consulted (PRD's unresolved expiry/revocation scope).

## Verified sound at this head

- Order inside `applyBatch`: reattachments → merges → identity update → claims →
  recompute → mirror, one transaction, full rollback on failure.
- All new SQL is bound; no new interpolation, path or injection surface.
- **V23 seed path**: the restored `uq_restaurant_listings_seed_location` reproduces
  V6's predicate verbatim, so V7's `ON CONFLICT (location_lat_6, location_lng_6)
  WHERE provenance = 'research-seed / photon-geocode'` still has an arbiter on a
  fresh install; `findByNormalizedLocation` gained the `provenance IS NOT NULL`
  predicate so it still uses the plain index; the broad uniqueness is gone on
  purpose and the multiple-rows case is handled (`atLocation` is a list) — QA
  round 2's F1 is fixed.
- `verification_basis` is used safely: every write is NULL or CERTIFIER_REGISTRY,
  `updateVerificationStatus` clears it on committee promotion, the backfill
  satisfies the new CHECK, and no read path needs it yet.
- **Metro radius-only** fails closed: the accepted point is the geocoded one and is
  re-checked against the circle with the distance, so wrong directory coordinates
  cannot place a listing in the wrong metro; a place name can neither include nor
  exclude a row; overlapping circles resolve to the nearest centre; a row is
  geocoded once and reused (cache keyed by the query text), and rows with
  coordinates never reach the geocoder.
- **Redirect** for the single-merge case: writes aimed at a retired listing are
  rewritten to the keeper before dedupe, claim sets are unioned, one write per
  target.

## Evidence

Containerised `./gradlew build` on a copy of `backend/` at `715b4d3` with all
`build/`, `.gradle` and `.kotlin` outputs removed first (`eclipse-temurin:21-jdk`,
Testcontainers sibling PostGIS, unique Gradle home volume): **BUILD SUCCESSFUL in
6m 8s**, 71 actionable tasks (63 executed, 8 from cache — compile only; every
module test task executed). **124 suites / 899 tests / 0 failures / 0 errors /
1 skipped** (`PhotonGeocoderLiveTest`, `LIVE_TESTS` gated), all 124 XMLs written
by this run. Round 2 was 125 suites / 895 tests: `TrustMergeTest` (4 tests) deleted,
8 tests added.

## Observations

- [omar] A claim that moves between listings makes the listing's trust columns stale unless the source is recomputed too; reattachment is a trust-changing write #sc-211 #trust #defect
- [omar] Deriving trust from the claim set does not cover fields the claim set cannot express: a curated cross-contamination value has no home in the projection and is lost on update #sc-211 #trust #defect
- [omar] Recording only the certifier basis makes committee trust indistinguishable from "no basis", so a certifier refresh can re-label it and then demote it #sc-211 #schema #risk
- [omar] Radius-only metro membership is correct at the accepted point, but selecting by circle requires geocoding coordinate-less rows before the selection is known #sc-211 #cost #geocoder

## Relations

- relates_to [ingestion runtime load model](../research/ingestion-runtime-load-2026-11-08.md)
- relates_to [SC-211 ingestion review, round 1](sc-211-ingestion-review.md)
- relates_to [SC-211 ingestion review, round 2](sc-211-ingestion-review-round2.md)
- relates_to [SC-211 ingestion review, round 4](sc-211-ingestion-review-round4.md)
- relates_to [review order](../process/review-order.md)