# Elasticsearch Photo Indexing for Gallery 2 — Design

**Status:** Approved 2026-11-17 (user approval). Ready for planning.
**Date:** 2026-11-17
**Spec:** `docs/features/elasticsearch/requirements.md` (refined 2026-11-17)
**Approach chosen:** B (core-first vertical slice); dedicated view, admin screen, and highlighting polish deferred to a follow-up pass.

## 1. Intent and why

Add an **optional, additive** Elasticsearch backend to Gallery 2 so items can be
full-text searched with ranking and highlighting, while preserving Gallery's
permission model and legacy architecture. Elasticsearch is a second search
provider alongside the existing database `LIKE` search (`GalleryCoreSearch`); it
never replaces it. The module must degrade gracefully: an Elasticsearch outage
must not break item editing or gallery browsing.

Why now: Gallery's `LIKE` search is coarse and unranked; Elasticsearch gives
ranking, highlighting, and a scalable index, with no schema changes (settings
live in plugin parameters).

## 2. Goals, non-goals, anti-goals

**Goals (v1, Approach B core-first slice)**
- Reusable HTTP transport (`ElasticsearchClient`) covering index/add/delete/update/query/create-index/delete-index, testable via platform `fsockopen` mocking.
- Indexing helpers (`ElasticsearchHelper`) that build documents from items and reindex everything with pagination.
- Event-driven indexing (`ElasticsearchIndexer`) keeping the index in sync on item save/delete, without blocking the user action.
- Search provider (`ElasticsearchSearch`) implementing `GallerySearchInterface_1_0`, permission-filtered at query time, usable through `SearchScan`/`SearchShowAll`.
- Plugin parameters for configuration (enabled, url, index, username, password), seeded with sensible defaults on install.
- Unit/integration tests for client, helper, indexer, and permission filtering.

**Non-goals (v1)**
- Dedicated search view/template and pagination UI (deferred; `SearchScan`/`SearchShowAll` cover browsing).
- Site-admin configuration screen, ping/create-index/reindex buttons (deferred; config via plugin params / defaults for v1).
- Highlight rendering polish (client returns highlights; surfacing them in UI deferred with the dedicated view).
- Indexing comments, tags, or derivative metadata.
- Cluster tuning, ILM, custom analyzers beyond a default mapping.
- Replacing the database `LIKE` search.

**Anti-goals (would make this a failure even if it "works")**
- Returning item ids a guest cannot view (permission bypass).
- Blocking item save/delete when Elasticsearch is down.
- Throwing from the transport layer instead of returning status arrays.
- Hand-editing generated `Entities.inc`/`Maps.inc` or adding schema changes (none required).

## 3. Constraints

- PHP 8.1 runtime (no Composer/autoload); module-local code; follow `modules/AGENTS.md`.
- New code: modern PHP 8.1-compatible style + `declare(strict_types=1)` (visibility keywords, typed props/params/returns, `[]`, constructor promotion, `match`). **No namespaces** — `modules/AGENTS.md` forbids them and the framework loads module/controller/view/search classes by bare global name. Existing/legacy files keep legacy style. 8.2+ features (`readonly`, DNF, standalone types) deferred until a runtime bump. Never mix styles within one file.
- No external Elasticsearch client; HTTP via `GalleryCoreApi::requestWebPage()` (method/headers/body), JSON via `json_encode`/`json_decode`.
- Tests via the web runner; HTTP mockable through the platform's `setReply('fsockopen', ...)`.
- No schema/entity changes (settings as plugin parameters) — avoids "Ask First" schema items.
- Target Elasticsearch 8.x (`/{index}/_doc/{id}`, `/_update/{id}`, `multi_match`, highlighting).
- AGENTS.md "Ask First" items otherwise: module registration, factory registrations, permissions. None of these require schema changes here.
- Shared Docker volume (`gallery2-data`) mutates the dev DB during tests; one schema-touching feature at a time (this feature touches no schema).

## 4. Approaches considered

- **A — Full module as specified** (client, helper, indexer, search, dedicated view, admin, templates, tests). Meets all §8 acceptance criteria in one pass, but largest blast radius and more to review.
- **B — Core-first vertical slice** (client, helper, indexer, search interface, tests; dedicated view/admin/highlighting deferred). Smaller, reviewable, proves the indexing + permission-filtered search path early; UI iterates on real data. Does not fully satisfy §8 in v1 (admin config, dedicated view, reindex UI deferred).
- **C — Client + admin only.** Dropped: delivers no search, defeats the purpose.

**Chosen: B.** The user prefers a smaller reviewable slice that proves the hard parts (transport, indexing, permission filtering) before investing in UI. The deferred items are tracked as a follow-up pass reusing `docs/features/elasticsearch/` with a new plan file.

## 5. Architecture

### 5.1 Module identity
`modules/elasticsearch/`, class `ElasticsearchModule extends GalleryModule`, id `elasticsearch`, version `1.0.0`, group `gallery`, required core API `array(7, 54)`, required module API `array(3, 9)`, callbacks `getSiteAdminViews`.

### 5.2 Components (v1)

- **`ElasticsearchClient`** (stateless utility, like `AkismetApi`): `setConfig($url, $index, $username='', $password='')`, `ping()`, `indexDocument($id, $document)`, `deleteDocument($id)`, `updateDocument($id, $partial)`, `search($query, $fields, $offset, $limit)`, `createIndex($mapping)`, `deleteIndex()`. All return `array($status, ...)`; never throw; HTTP errors become `GalleryCoreApi::error(ERROR_UNKNOWN, ...)` with the ES error message. Basic auth header when credentials present. 404 on delete treated as success.
- **`ElasticsearchHelper`**: `buildDocument($item)` (id, title, summary, description, keywords, entityType, mimeType, ownerId, ownerName, parentId, creationTimestamp, modificationTimestamp), `indexItem($itemId)`, `deleteItem($itemId)`, `reindexAll()` (paginated `SELECT [GalleryItem::id] FROM [GalleryItem]`, `guaranteeTimeLimit(30)` per batch).
- **`ElasticsearchIndexer`** (`GalleryEventListener` for `GalleryEntity::save`/`delete`): guard with `GalleryUtilities::isA($entity, 'GalleryItem')`; on save index, on delete remove; indexing failures logged via `$gallery->debug()` and swallowed (return `null`).
- **`ElasticsearchSearch`** (`GallerySearchInterface_1_0`): `getSearchModuleInfo()` (name/description; options titles/summaries/descriptions/keywords, all enabled by default); `search($options, $criteria, $offset, $count)`:
  - disabled/unconfigured → empty result set + `null` status.
  - empty criteria → empty result set + `null` status (no error status; `SearchScan`/`SearchShowAll` must not crash). (Minor call: requirements allowed either; chose the non-crashing option.)
  - otherwise `multi_match` on selected fields, `from`/`size`, highlighting; collect hit ids; `fetchPermissionsForItems($ids, $activeUserId)`; keep hits where `core.view` present; format `array('itemId', 'fields' => array('key','field','value'))` plus owner; return `array(null, array('start','end','count','results'))` where `count` = number of permission-filtered hits.
  - on client error: empty result set + `null` status; store the ES error message in the `lastError` plugin parameter.

### 5.3 Registration (`performFactoryRegistrations()`)
- Register `GalleryEventListener` → `ElasticsearchIndexer` for hints `array('GalleryEntity::save', 'GalleryEntity::delete')`.
- Register `GallerySearchInterface_1_0` → `ElasticsearchSearch` id `elasticsearch`.
- List `ElasticsearchSearch.inc` (and future admin) in `MANIFEST`.

### 5.4 Upgrade (`upgrade($currentVersion)`)
- Initial install: set defaults `enabled=0`, `url=http://localhost:9200`, `index=gallery2`, empty credentials.
- Bump version strings only; no schema, no `configureStore()`.

### 5.5 Permission handling
- Index contains metadata for all items (including private) so reindex is complete.
- At query time filter hit ids by `core.view` via `fetchPermissionsForItems($ids, $activeUserId)`.
- Never return ids the active user cannot view.

### 5.6 Failure modes and degradation
- Unreachable ES / client error: `ElasticsearchSearch::search` returns empty result set + `null` status so `SearchScan` does not crash; last error stored in a module parameter for later admin display.
- Indexing errors during save/delete logged and swallowed.
- `ping()` available for future admin validation.

### 5.7 Configuration in v1 (no admin UI)
Defaults seeded on install; values can be read/written via `GalleryCoreApi::getPluginParameter`/`setPluginParameter('module', 'elasticsearch', ...)`. Tests set them directly. The admin screen is a follow-up (Approach B deferral).

## 6. Testing (v1 — core only)

Under `modules/elasticsearch/test/phpunit/` (Approach B v1 scope; controller/view/admin tests are deferred to the follow-up pass):
1. `ElasticsearchClientTest` — mock `fsockopen` replies: successful index (201/200), delete 404 → success, search parses hits/total/highlight, HTTP 5xx → error status, basic auth header present when credentials set.
2. `ElasticsearchHelperTest` — `buildDocument` maps item fields (use `_createRandomDataItem()`).
3. `ElasticsearchIndexerTest` — on `GalleryEntity::save` for a data item, client index called with item id; on delete, client delete called; non-item entities (e.g. user) are no-op.
4. `ElasticsearchSearchTest` — search with criteria returns results; permission filtering excludes private items for a guest; empty criteria → empty result + `null` status.

Cleanup: mark entities and plugin params with `_markForCleanup()` / `_markPluginParametersForCleanup()`. Test commands: `bash auto-test.sh http://host.docker.internal:10000/ elasticsearch` and `... ElasticsearchClientTest`.

## 7. Deferred (follow-up pass)

Tracked as a separate feature pass reusing `docs/features/elasticsearch/` with a new plan:
- Dedicated `ElasticsearchSearch` controller/view/template (results list with thumbnail, title link to `core.ShowItem`, highlighted snippet, owner, pagination; empty/error states).
- `ElasticsearchSiteAdmin` controller/view/template (enabled, url, index, username, password (never refill), verify ping, create index, reindex with count/time).
- Highlight rendering in templates.
- Admin reindex UI wiring to `ElasticsearchHelper::reindexAll()`.
- **CLI for reindex/operations** (e.g. `lib/tools/elasticsearch-reindex.php` or `modules/elasticsearch/cli/reindex.php`): bootstrap Gallery and call `ElasticsearchHelper::reindexAll()`; useful for large galleries and automation/cron. Thin wrapper only — `reindexAll()` is already the entry point, so no core change needed. Needs bootstrap/auth/output handling; no precedent in repo, so treat as its own small subsystem.

## 8. What's left to the builder

- Exact default mapping for `createIndex` (sensible text fields with standard analyzer).
- Pagination details in `reindexAll` (batch size, `guaranteeTimeLimit` cadence).
- Error-message extraction from ES JSON responses.
- Whether to expose `enabled` gating in `ElasticsearchSearch::search` (yes — return empty when disabled).
- Test fixture setup following `AddCommentControllerTest.class` golden sample.

## 9. Style note

New files in `modules/elasticsearch/` use modern PHP 8.1-compatible style + `declare(strict_types=1)` (visibility keywords, typed props/params/returns, `[]`, promotion, `match`). **No namespaces** (`modules/AGENTS.md` forbids them; framework loads classes by bare global name). Legacy files touched only minimally and only in legacy style. 8.2+ features (`readonly`, DNF, standalone types) deferred until a runtime bump. See `modules/core/UserLogin.inc`, `GalleryModule.class`, `GalleryUtilities.class` for the legacy reference.

---

*This design is in draft. A builder check (fresh subagent read) will run before approval; then the user approves (HARD-GATE) before any plan or code.*