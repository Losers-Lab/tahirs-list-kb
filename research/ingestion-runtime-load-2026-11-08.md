---
type: note
title: "SC-211 ingestion load — claims, venue identity and the V22 schema"
description: "Verified runtime-load model for certifier ingestion: cited claims, deterministic venue identity, the V22 schema and its Flyway constraints, and the replay-idempotency rules the load must follow."
tags:
  - tahirs-list
  - ingestion
  - sc-211
  - schema
  - postgres
  - flyway
  - halal-verification
status: active
date: 2026-11-08
author: hamza
sources:
  - resource: https://github.com/Losers-Lab/tahirs-list/pull/79
    title: "PR #79 — ingestion claims schema and venue key"
  - resource: https://github.com/Losers-Lab/tahirs-list/pull/75
    title: "PR #75 — certifier libraries, translators and verified hand-cut rule"
  - resource: backend/adapters/persistence/src/main/resources/db/migration/V22__sc_211_ingestion_schema.sql
    title: "V22 ingestion schema (implementation repo)"
permalink: tahirs-list-kb/research/ingestion-runtime-load-2026-11-08
---

# SC-211 ingestion load — claims, venue identity and the V22 schema

Cross-cutting facts produced while building the SC-211 runtime ingestion load. The
feature design package stays in the implementation repo under
`docs/features/SC-211/`; this note records the durable model, the schema
constraints, and the traps a later agent must not re-learn.

## Status (2026-11-08)

| Piece | State |
|---|---|
| Design package (D-11…D-18) | merged — PR #72 |
| Certifier libraries, translators, pre-flight, artifact guard, `VenueIdentity` | merged — PR #75 (`6589218`) |
| Runtime-load increment 1: domain claims + V22 schema | PR #79 open (head `ddf9c50`), rebased on `main` `b029a2b` |
| Whole backend | **808 tests, 0 failures / 0 errors, 110 suites** in `eclipse-temurin:21-jdk` with Testcontainers PostGIS |

Still to land: `RestaurantListingRepository.applyBatch`, `LoadIngestionBatch.preview/execute`,
`BrandRepository`, the pipeline wiring (`RunSampleIngestion`, `CompileManualBatch`,
`IngestReport`) and the bootstrap CLIs. See “Remaining work”.

## Domain model

| Type | What it is |
|---|---|
| `CertificationClaim` | One cited observation: `source`, `sourceRecordRef`, `sourceUrl`, `certifier`, `tier`, `declaredHandCut`, `expiresOn`, `firstSeen`, `lastSeen`. Identity is `"$source::$sourceRecordRef"`. |
| `CertificationTier` | `TIER_1` / `TIER_2` / `TIER_3` with `rank`; only `TIER_1.confersRegistryVerification == true`. |
| `HandCutDeclaration` | `HAND_CUT` / `NOT_HAND_CUT` / `UNSPECIFIED`; `isHandCut` is true only for `HAND_CUT`. This is the certifier's own policy, not its tier. |
| `VerificationBasis` | Why a listing is `VERIFIED`: `COMMITTEE_REVIEW` (owner claim) or `CERTIFIER_REGISTRY` (Tier-1 evidence). |
| `RestaurantListing.imported(...)` | Ingested-row factory: no owner/cuisine required, provenance required, `venueKey` required, `VERIFIED` only with `CERTIFIER_REGISTRY` **and** a certifier-registry provenance. |

`Provenance` is a closed vocabulary. V22 widened it with `certifier-hms`,
`certifier-hfsaa`, `certifier-hma` and `manual-zabihah`; `Provenance.CertifierRegistry`
is the subset that may back `VERIFIED`. `manual-zabihah` is cited but never verifies
(D-13).

### Invariants worth not re-deriving

- A claim **source** is a lowercase slug (`^[a-z0-9-]+$`, ≤64); the record ref is
  trimmed and ≤255; the citation URL is `https://` + a non-empty remainder, no
  whitespace, ≤1024; the certifier name is ≤255. The caps mirror the V22 widths so
  persistence can never truncate silently.
- `venueKey` is a **lowercase 64-char SHA-256 hex** (normalized name + street +
  city + region + country; coordinates excluded). It is required on new imported
  rows and intentionally immutable on owner edit — it is venue identity, not content.
- An owner edit (`withUpdatedFields`) can never change `id`, `ownerId`, `brandId`,
  `provenance`, `venueKey`, `verificationStatus` or `createdAt`.
- The domain `HandCutDeclaration` deliberately duplicates the library enum in
  `:certifier-core`: `:domain` must not depend on a certifier library. The
  translator is the only crossing point (D-12).

## V22 schema

`V22__sc_211_ingestion_schema.sql` is schema only — there is **no generated data
migration**; the load writes through the runtime repository (D-15, “V23 retired”).

**`external_certification_claims`**

| Column | Notes |
|---|---|
| `id` | UUID PK, DB default `gen_random_uuid()` |
| `listing_id` | FK → `restaurant_listings(id)`, **`ON DELETE RESTRICT`** |
| `source`, `source_record_ref` | global unique index `uq_external_claims_source_record` — the idempotency key |
| `source_tier` | SMALLINT, CHECK 1–3 |
| `source_url` | CHECK `^https://[^[:space:]]+$` |
| `certifier` | VARCHAR(255) |
| `expires_on` | DATE, nullable (HMS publishes it; HFSAA does not) |
| `hand_cut_declaration` | VARCHAR(32), CHECK `(HAND_CUT, NOT_HAND_CUT, UNSPECIFIED)` |
| `first_seen`, `last_seen` | nullable; CHECK `last_seen >= first_seen` |
| `source` CHECK | lowercase slug, so the key cannot drift by case |

The FK is `RESTRICT`, not `CASCADE`: deleting a listing must never erase the audit
trail or free a global `(source, source_record_ref)` key for reuse.

**`restaurant_listings`**

- `venue_key CHAR(64) NULL` + CHECK `^[0-9a-f]{64}$` + partial unique index
  `uq_restaurant_listings_venue_key WHERE venue_key IS NOT NULL`.
- `uq_restaurant_listings_research_location (location_lat_6, location_lng_6) WHERE
  provenance IS NOT NULL` — the D-18 database backstop: one ingested venue per 6-dp
  coordinate cell. Ordinary user rows (`provenance NULL`) are unaffected.

## Why the old seed-location index was dropped

V6 created `uq_restaurant_listings_seed_location` on the same columns with the
narrower predicate `provenance = 'research-seed / photon-geocode'`. V22 drops it and
creates the research-wide index instead.

That is safe because PostgreSQL infers a **broader** partial index for a narrower
`ON CONFLICT` predicate: `provenance = 'research-seed / photon-geocode'` implies
`provenance IS NOT NULL`. V7's seed replay therefore still resolves and still
`DO NOTHING`s — verified by `SeedRestaurantsMigrationTest` (14/14) running after the
full 22-migration chain. Keeping both indexes is harmless but redundant.

**Flyway trap:** an applied migration is frozen by its checksum. V7 cannot be
edited to change its conflict target; fix the shape in a new migration (V22) or
prove the new index is inferred. Do not “correct” history.

## Identity and replay rules (D-18)

The load must match in this order, per candidate:

1. **Claim** `(source, source_record_ref)` → the listing that already owns this
   source record. Same record never attaches to a second listing, even if its
   coordinates drifted.
2. **Venue key** → the same normalized venue.
3. **Normalized location** → a *report-only* fallback. A location-only match is
   held as `SUSPECTED_DUPLICATE` and **not written** until a curator resolves it.

A claim already pointing at a different listing aborts the batch (E10) instead of
duplicating. Replaying an older/archived batch updates the newer listing.

## Verification and hand-cut (D-11, D-16)

- Verification follows the **tier**: Tier-1 registry evidence may set `VERIFIED`
  with basis `CERTIFIER_REGISTRY`; manual/web rows stay `UNVERIFIED`.
- Hand-cut follows the **certifier's own declaration**: `TRUE` only on a declared
  hand-cut; `FALSE` for declared-not **and** for silence; `NULL` only when there is
  no certifier claim at all. Never derive hand-cut from the tier.
- Tier-1 certification implies index qualification:
  `cross_contamination = NO_CROSS_CONTAMINATION`. Manual/web rows keep their
  explicit value (default `UNCERTAIN`). No directory publishes cross-contamination
  evidence, so this is a policy decision, not a scraped field.
- There is exactly one cutting attribute — hand-cut. No counter-category exists in
  code, data, tests, docs or copy.

## Test evidence

| Suite | What it pins |
|---|---|
| `CertificationClaimTest` (10) | identity, slug/trim normalization, length caps, https citation, tier ranks, hand-cut derivation, seen order |
| `RestaurantListingImportedTest` (13) | imported invariants, certifier-vs-manual `VERIFIED`, malformed venue key, governance immutability |
| `IngestionSchemaMigrationTest` (10) | claims columns, global source-record uniqueness, venue-key uniqueness/format, location backstop, source/tier/url/hand-cut/seen CHECKs, `RESTRICT` delete, index swap |
| `SeedRestaurantsMigrationTest` (14) | V7 replay idempotency through the research-wide index |
| Whole backend | 808 tests, 0 failures, 110 suites |

```bash
./scripts/backend-test.sh            # full build + tests in eclipse-temurin:21-jdk
./scripts/backend-test.sh test-only  # tests only
```

Persistence specs share one PostGIS container per spec, so claim fixtures must use
unique `(source, source_record_ref)` pairs — the global unique index is real, and a
reused default fixture will fail the *next* test, not the one that wrote it.

## Gotchas

- `Tables.ExternalCertificationClaims` names the column property `claimSource`
  because `source` collides with Exposed's `ColumnSet.source`.
- `venue_key` is `CHAR(64)`: always exactly 64 characters, so no padding shows up,
  but a future shorter key would read back padded.
- The claim table's `source` is an identifier, not a display name; the certifier
  name lives in `certifier`.
- `ON DELETE RESTRICT` means any future listing-delete feature must handle claimed
  listings explicitly (soft-delete or a committee decision), not cascade them away.

## Remaining work

1. `RestaurantListingRepository` matching: `findClaim(source, ref)`,
   `findByVenueKey`, `findByNormalizedLocation`.
2. `applyBatch(writes)` in **one transaction**: brands, listings, cuisines, halal
   items, claims, search mirror; claim-first ordering; location-only matches held.
3. `LoadIngestionBatch.preview/execute`, `CandidateValidator`, `CatalogUpdateReview`,
   `BatchLoadResult`; `BrandRepository.resolve/resolveOrCreate`.
4. Pipeline wiring: `RunSampleIngestion`, `CompileManualBatch`, `IngestReport`,
   `SampleIngestMain` / `LoadIngestBatchMain`, `scripts/ingest-sample.sh`,
   `scripts/load-ingest-batch.sh`.
5. Shortcut AC wording sync for AC-4/AC-5/AC-10 (token expired during this work).

## Relations

- builds_on [[certifier-libraries-2026-11-08]]
- relates_to [[sc-119-partial-halal]]
- relates_to [[sc-46-owner-claim]]
