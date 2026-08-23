# Performance & Scalability Improvement Plan

Analysis of the Packagist Symfony app (Doctrine ORM/DBAL, MySQL, Redis) covering entities,
repositories, controllers, models, listeners, workers, and commands.

Scale context: ~400k+ packages, millions of version rows, download counters incremented per
install, append-only audit log. Issues are ordered by expected impact.

Project conventions to respect when fixing (from CLAUDE.md):

- No Doctrine migrations system → provide raw SQL for schema changes.
- Verify with `composer phpstan` and `composer test -- --filter <suite>`.
- Avoid `composer` vendor name in test fixtures.
- Prefer constructor property promotion, virtual properties w/ accessors.

---

## Summary checklist

### High severity

- [ ] **H1.** Download counts stored as ever-growing JSON blobs — prune/roll up old date keys (`src/Entity/Download.php:41`, `src/Model/DownloadManager.php`)
- [ ] **H2.** N+1 full-blob download queries on downloads endpoints — batch into single query + MGET (`src/Controller/PackageController.php:752-765`)
- [ ] **H3.** HTML package page lazy-hydrates ALL versions incl. JSON columns — use existing partial loader (`src/Controller/PackageController.php:557`, `:1251`)
- [ ] **H4.** Per-package sequential Redis round trips in listings — pipeline ZCARD/ZSCORE loops (`src/Controller/Controller.php:52-64`)
- [ ] **H5.** Non-sargable `LIKE '%/name'` scans on submit path — indexed suffix lookup + LIMIT (`src/Controller/PackageController.php:256-257`, `src/Validator/TypoSquattersValidator.php:65-66`)
- [ ] **H6.** Audit log stores multi-KB payloads per version event; no retention anywhere — slim payload + prune command (`src/EventListener/VersionListener.php:39-44`)

### Medium severity

- [ ] **M1.** Webhook handler hydrates every maintained package — targeted query by repo URL/remoteId (`src/Controller/ApiController.php:640-654`)
- [ ] **M2.** Unbounded vendor page — paginate + shared max age (`src/Controller/PackageController.php:299-315`)
- [ ] **M3.** Job table: no purge + weak index coverage — composite indexes + retention job (`src/Entity/Job.php:22-28`, `src/Entity/JobRepository.php`)
- [ ] **M4.** Provider-set rebuild stampede + SMEMBERS of registry — single-flight lock; serve list.json from dump (`src/Model/ProviderManager.php:57-107`)
- [ ] **M5.** PhpStat JSON churn + heavy JSON_TABLE aggregation — roll-ups, batched flushes (`src/Entity/PhpStatRepository.php:189-268`)
- [ ] **M6.** Blocking sleeps / external HTTP in request & worker paths — timeouts, caching, move to jobs
- [ ] **M7.** Nightly commands do per-item round trips over whole corpus — batch queries + pipelines (`CompileStatsCommand`, `PopulateDependentsSuggestersCommand`)
- [ ] **M8.** Missing indexes vs actual query patterns — raw SQL migrations for `audit_log.packageId/userId`, `filter_list_entry.publicId`, `job` composites

### Low severity

- [ ] **L1.** Hourly full-table GROUP BY aggregates on stats page — materialized summary table
- [ ] **L2.** Metadata-changes endpoint builds ≤100k arrays before size check — ZCOUNT guard first
- [ ] **L3.** Security voter evaluated per soft-deleted version rendered in Twig loop — hoist out
- [ ] **L4.** Uncached `SELECT max(id)` on homepage random packages — result cache alongside neighbors
- [ ] **L5.** Advisory sync double self-join on sources before OR filter — restructure joins
- [ ] **L6.** 10 sequential DELETE statements per version removal — FK cascades or combined statements
- [ ] **L7.** REGEXP on normalizedVersion + full-blob loads for major-version stats — prefix compare + cache

---

## HIGH severity

### H1. Download counts stored as ever-growing JSON blobs

- **Files:** `src/Entity/Download.php:41`, `src/Model/DownloadManager.php:60-79`, `251-289`
- **Problem:** `Download::$data` is JSON keyed `YYYYMMDD => count`, one key per day since the
  package was created. Nothing prunes old keys anywhere in the codebase.
  `transferDownloadsToDb()` rewrites the whole blob every transfer cycle;
  `getDownloads()` loads + json-decodes the entire history only to sum 30 days.
- **Impact:** Unbounded row growth on the hottest rows; read-modify-write amplification
  (binlog, buffer pool churn); memory + CPU cost per stats request grows forever.
- **Fix sketch:**
  1. Add pruning during `transferDownloadsToDb()` / `createDbRecordsForKeys()`: drop keys older
     than ~13 months (or roll up to monthly buckets beyond 90 days: key format `202608`).
  2. One-time backfill command to strip old keys from existing rows (batched by primary key,
     chunked UPDATEs).
- **Verification:** unit test for prune/rollup logic; measure row size before/after on fixture data.

### H2. N+1 full-blob queries on downloads endpoints

- **File:** `src/Controller/PackageController.php:752-765` (`viewPackageDownloadsAction`)
- **Problem:** loops versions calling `DownloadManager::getDownloads($package, $version)` —
  per iteration: 1 SQL `findOneBy(['id','type'])` + full JSON decode + Redis MGET
  (`src/Model/DownloadManager.php:40-100`). ~200 SQL queries for big packages per cache miss
  (`setSharedMaxAge(3600)` only helps hits).
- **Fix sketch:** add `DownloadRepository::findByTypes(array idTypePairs)` fetching all rows in
  one query; add `DownloadManager::getDownloadsForVersions(Package, Version[])` doing one Redis
  MGET for `[dl:<pid>-<vid>:<today|yesterday>]` keys; compute monthly sums in PHP.
- **Also applies to:** the major-version variant of the same endpoint and
  `DownloadRepository::findDataByMajorVersions` (see L7).

### H3. HTML package page lazy-hydrates ALL versions incl. JSON columns

- **File:** `src/Controller/PackageController.php:557` (also `:1251` in `statsAction`)
- **Problem:** `$package->getVersions()->toArray()` after loading via
  `getPartialPackageWithVersions()/getPackageByName()` triggers lazy load of every full Version
  row (10 JSON columns each: autoload, source, dist, extra, authors…). The repo already has
  `PackageRepository::getPartialPackageByNameWithVersions()` (`src/Entity/PackageRepository.php:349-390`)
  built for exactly this ("helps for packages like ccxt/ccxt") but it isn't used here.
- **Impact:** multi-MB hydration + GC pressure on every uncached view of large packages; no
  shared-max-age on the HTML response either.
- **Fix sketch:** switch HTML path to the partial loader; hydrate only what templates need
  (id/version/normalizedVersion/releasedAt/development/extra/isDefaultBranch + softDeletedAt);
  consider short `setSharedMaxAge` where auth-independent.
- **Care:** soft-deleted versions must still render grayed out in aside list; check
  `version_list.html.twig` fields used before trimming columns.

### H4. Per-package sequential Redis round trips in listing pages

- **Files:** `src/Controller/Controller.php:52-64` (`getPackagesMetadata`),
  `src/Model/FavoriteManager.php:75-89` (TODO comment exists), `src/Controller/PackageController.php:361-364`
- **Problem:** one ZCARD per package (favorites) and one ZSCORE per package (trending sort) —
  15–100 sequential RTTs/page across explore/vendor/profile/spam/providers pages.
- **Fix sketch:** batch via Predis pipeline for both ZCARD and ZSCORE loops (or ZMSCORE);
  reuse existing batched path `FavoriteManager::getFaverCounts()` where possible.
- **Note:** keep GitHub-stars special case (skip refetch when stars already on entity).

### H5. Non-sargable leading-wildcard scans on submit path

- **Files:** `src/Controller/PackageController.php:256-257` (public fetch-info endpoint),
  `src/Validator/TypoSquattersValidator.php:65-66`
- **Problem:** `SELECT name FROM package WHERE name LIKE '%/<pkgName>'` — no index possible,
  full scan of ~400k rows, unbounded result set, synchronous inside form validation; validator
  additionally runs `levenshtein()` + lazy-loads maintainers per match.
- **Fix sketch options (pick one):**
  a) MySQL generated column `name_suffix` = substring after `/` + index; equality instead of LIKE.
  b) Reverse-name lookup table populated on insert/update (raw SQL migration + listener).
  c) For TypoSquatters specifically: reuse Levenshtein via search index (Algolia) if acceptable.
- **Regardless of option:** add LIMIT (e.g. 50) as a safety bound.

### H6. Audit log writes huge payloads per version event; no retention anywhere

- **Files:** `src/EventListener/VersionListener.php:39-44`, `src/Entity/AuditRecord.php:353-364`
- **Problem:** every version insert calls `toV2Array([])` → lazy-loads tags + 6 link collections
  (~7 SELECTs) then persists multi-KB JSON into `audit_log`. Combined with zero DELETE/prune
  against `audit_log`, `job`, or download history → unboundedly growing tables.
- **Fix sketch:**
  1. Slim down `AuditRecord::versionCreated` payload (summary: name, version, hash of metadata,
     not full metadata), or gate full payloads behind admin flag.
  2. Retention command `packagist:prune-audit-records` deleting older than N days except types
     worth keeping (deletions/freezes); register as recurring job.
- **Related:** same retention pass should purge finished `job` rows older than N days (see M3).

---

## MEDIUM severity

### M1. Webhook handler hydrates every maintained package

- **File:** `src/Controller/ApiController.php:640-654`
- **Problem:** `$user->getPackages()` lazy-load on GitHub push hot path; org bots maintain
  thousands; regex matching per package in PHP.
- **Fix sketch:** dedicated repo method `findByUserAndRepositoryUrl($userId, $host, $path)`
  using indexed `repository_idx` + maintainers join; try remoteId lookup first.

### M2. Unbounded vendor page

- **File:** `src/Controller/PackageController.php:299-315` (`viewVendorAction`)
- **Problem:** no LIMIT/pagination (`'paginate' => false`), no shared max age; feeds the H4 fan-out.
- **Fix sketch:** paginate like other listings (Pagerfanta), or cap + link to search;
  add `setSharedMaxAge(300)` since the page is public.

### M3. Job table: no purge + weak index coverage

- **Files:** `src/Entity/Job.php:22-28`, `src/Service/QueueWorker.php:82-95`,
  `src/Entity/JobRepository.php:70-101`
- **Problem:** one row (payload+result JSON) per scheduled update, kept forever.
  `markTimedOutJobs`/`getScheduledJobIds` filter on `status+executeAfter` but indexes are
  single-column; `findLatestExecutedJob` (runs on package page views for maintainers,
  `PackageController.php:701`) filters packageId+type+status ORDER BY createdAt DESC with only
  `package_id_idx`.
- **Fix sketch:** raw SQL migrations: composite `(status, executeAfter)` and `(packageId, type,
  status)` indexes; retention deletion of completed/errored jobs > 30 days; consider partitioning later.
- **Note:** `getLastGitHubSyncJob` stores userId in the packageId column — document or add type column.

### M4. Provider-set rebuild stampede + SMEMBERS of registry

- **File:** `src/Model/ProviderManager.php:57-67, 82-107`
- **Problem:** `set:providers` expires hourly → first requests rebuild synchronously from MySQL
  GROUP BY ProvideLink; every concurrent FPM worker repeats it (no lock). `getPackageNames()`
  SMEMBERs the entire package registry + PHP sort for `/packages/list.json`.
- **Fix sketch:** SET NX lock around rebuild (single-flight); serve list.json from pre-generated
  dumps (dumper already writes metadata dumps — piggyback); SSCAN fallback otherwise.

### M5. PhpStat JSON churn + heavy aggregation

- **Files:** `src/Entity/PhpStatRepository.php:189-268`, `src/Entity/PhpStat.php:69`
- **Problem:** rows gain a key per PHP-minor/day and are rewritten whole per cycle;
  `createOrUpdateMainRecord` uses JSON_KEYS + dynamic SUM(JSON path) expressions;
  `createOrUpdateRecord` flushes per record inside the loop.
- **Fix sketch:** monthly roll-up like H1; batch flush outside loop; consider keeping exact-depth
  daily series Redis-only with periodic MySQL snapshots.

### M6. Blocking sleeps / external HTTP in request & worker paths

| Location | Issue |
|---|---|
| `src/Service/UpdaterWorker.php:158-177` | `file_get_contents` GitHub API, no timeout, in worker loop |
| `src/Controller/ProfileController.php:147-157` | sync GitHub API call on admin profile view |
| `src/Controller/UserController.php:79` | literal `sleep(5)` |
| `src/Model/PackageManager.php:168` | `usleep(500000)` in DELETE flow |

- **Fix sketch:** explicit timeouts (e.g. 5s connect / 10s total); cache token-valid results
  longer; move GitHub lookups into jobs; replace sleep with client-side polling redirect pattern
  used elsewhere in the app.

### M7. Nightly commands do per-item round trips over whole corpus

- **Files:** `src/Command/CompileStatsCommand.php:69-87`,
  `src/Command/PopulateDependentsSuggestersCommand.php:52-76`
- **Problem:** ~800k sequential Redis GETs + SQL SELECTs nightly; dependents command requeues ID
  on lock contention (busy-spin/livelock risk).
- **Fix sketch:** single `SELECT ... WHERE lastUpdated >= X` batch; pipeline redis gets; sleep +
  retry limit on contention instead of tail-requeue.

### M8. Missing/unhelpful indexes vs actual queries

Raw SQL migrations needed (no Doctrine migrations system in this project):

- `audit_log`: add `packageId`, `userId` indexes (currently only type/datetime/vendor/organizationId
  at `src/Entity/AuditRecord.php:33-36`). Any direct join/filter (e.g. backfill subqueries like the
  `frozenAt` one) full-scans an append-only table without them.
- `filter_list_entry`: index `publicId` (`FilterListEntryRepository::findOneByPublicId` :134-137 full-scans).
- `job`: composites per M3.

---

## LOW severity

- **L1.** Stats aggregates `GROUP BY YEAR(), MONTH()` hourly over millions of rows
  (`src/Entity/VersionRepository.php:356-371`, `src/Entity/PackageRepository.php:737-752`) — cached
  but stampede-prone on expiry; cost grows linearly with table size. Consider materialized summary
  table updated by CompileStats.
- **L2.** Metadata-changes endpoint builds ≤100k action arrays before checking size — add cheap
  ZCOUNT guard first (`src/Controller/PackageController.php:180-199`).
- **L3.** Security voter evaluated per soft-deleted version rendered
  (`templates/package/version_list.html.twig:7`, `src/Twig/PackagistExtension.php:188-193`) — hoist
  decision out of the loop.
- **L4.** Homepage random-packages: uncached `SELECT max(id)` per render
  (`src/Controller/ExploreController.php:41-45`) — cache alongside neighbors (60s result cache).
- **L5.** Advisory sync double self-join of `sources` before OR filter
  (`src/Entity/SecurityAdvisoryRepository.php:44-53`) — restructure join order.
- **L6.** `VersionRepository::remove()` issues 10 sequential DELETE statements per version
  (`src/Entity/VersionRepository.php:76-85`) — FK ON DELETE CASCADE or combined statements.
- **L7.** `DownloadRepository::findDataByMajorVersions` REGEXP on normalizedVersion + full-blob
  loads (`src/Entity/DownloadRepository.php:70-92`) — prefix compare instead of REGEXP; cache the
  aggregate.

---

## Good patterns to preserve (do not regress)

- MGET batching in `DownloadManager::getPackagesDownloads` (:121-138)
- Lua-scripted version-id resolution (`src/Model/VersionIdCache`)
- `GROUP_CONCAT` tag batching (`PackageRepository::getTagsByPackageIds` :552)
- `audit_log_search` denormalization replacing JSON_EXTRACT scans
- Redis-cached advisory searches (`searchSecurityAdvisories`, 7-day TTL + listener busting)
- V2 dumper batching with EM clears (`Package::toArray` slices of 100, `Package.php:326-335`)
- Result-cache profiles (`QueryCacheProfile`) on global COUNT queries

## Suggested implementation order

1. Quick wins, low risk: H4 (pipeline batching), L2–L4, M6 (timeouts/sleep removal)
2. Query fixes: H3 (partial loader on HTML page), H2 (batch downloads endpoint), M1, M2
3. Schema work (raw SQL + backfill commands): H5, M8, M3 indexes
4. Data-model changes: H1 + H6 + M3 retention/rollups (needs product sign-off on retention windows)
