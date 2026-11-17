# Elasticsearch Module — Core Slice (v1) Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** Implement the Approach B core-first slice of the Elasticsearch module: transport, indexing helper, event listener, search provider, module registration with defaults, and tests — without the dedicated view, admin screen, templates, or highlighting polish.

**Architecture:** Single new module `modules/elasticsearch/` modeled on `modules/comment` (golden sample). Stateless HTTP client via `GalleryCoreApi::requestWebPage()` (platform `fsockopen`, mockable). Event-driven indexing through `GalleryEventListener`. Search provider implements `GallerySearchInterface_1_0` with server-side `fetchPermissionsForItems` filtering. Configuration in plugin parameters seeded on install. No schema/entity changes.

**Tech Stack:** PHP 8.1 (new code: modern 8.1-compatible style + `declare(strict_types=1)`, visibility keywords, typed props/params/returns, `[]`, constructor promotion, `match`; no namespaces per `modules/AGENTS.md`); Gallery 2.3.2 framework; Docker web test runner (`auto-test.sh`).

**Spec:** `docs/features/elasticsearch/specs/design.md` (approved 2026-11-17)

## Global Constraints

- Syntax check: `docker compose exec php php -l <file>` (must pass for every changed file).
- Tests: `/opt/homebrew/bin/bash auto-test.sh http://host.docker.internal:10000/ <scope>` (dev install only; tests commit their transaction).
- Regen (not needed this plan — no entities): `docker compose exec -w /app/modules/<m>/classes php php ../../../lib/tools/bin/generate-entities.php`.
- Style (NEW code): modern PHP 8.1-compatible + `declare(strict_types=1)`; visibility keywords instead of `var`; typed props/params/returns; `[]` not `array()`; constructor promotion; `match`; **no namespaces** (`modules/AGENTS.md`); `/** ... */` + `/* ... */` comments. Existing/legacy files touched only in legacy style. Never mix styles within one file.
- No schema/entity changes: no `configureStore()`, no `@g2` bumps, no generated `.inc` edits.
- AGENTS.md "Ask First" items flagged in design doc: module registration, factory registrations, permissions (none registered in v1 — no item permissions; site admin only). No schema/PHP/Docker bumps.
- Shared Docker volume (`gallery2-data`) mutates the dev DB during tests; worktrees share state. One schema-touching feature at a time (this feature touches no schema).
- `testing-strategy` + `test-driven-development`: unit tests for client/helper; integration-style (real test DB) for indexer/search where fixtures needed. Mock only at HTTP boundary.
- Cleanup: mark entities with `_markForCleanup()` and plugin params with `_markPluginParametersForCleanup()`.
- i18n: `$gallery->i18n()` in PHP, `{g->text}` in templates; update `modules/elasticsearch/po/messages.po`. (Templates deferred; strings still go in po for any PHP-side messages.)

## Review Focus

Input classes / failure modes the spec implies but no task's tests directly exercise, most likely first:

1. **ES returns a malformed/empty JSON body** — client must not fatal; return error status. (Client task owns this.)
2. **Permission filtering with zero hits or all-private result sets** — `fetchPermissionsForItems` on empty array; must not error. (Search task owns this.)
3. **Event listener invoked for a non-Item entity (e.g. GalleryUser)** — must be a no-op, not an index attempt. (Indexer task owns this.)
4. **Reindex over a large item set with a slow ES** — `guaranteeTimeLimit` cadence; no test at scale, but batch size must be bounded. (Helper task owns this.)
5. **Module activated on an install where ES is unreachable** — activation must succeed; indexing failures are swallowed. (Module scaffold + indexer tasks own this.)

---
### Task 1: Module scaffold

**Files:**
- Create: `modules/elasticsearch/module.inc`
- Create: `modules/elasticsearch/MANIFEST`
- Create: `modules/elasticsearch/po/messages.po` (stub)
- Create: `modules/elasticsearch/classes/` (dir)
- Create: `modules/elasticsearch/test/phpunit/` (dir)

**Interfaces:**
- Produces: `ElasticsearchModule` class with identity, `upgrade()` seeding defaults, `performFactoryRegistrations()` registering `GalleryEventListener` → `ElasticsearchIndexer` and `GallerySearchInterface_1_0` → `ElasticsearchSearch`, `getModuleEntityTypes()` returning `array()`.

- [ ] **Step 1: Create directories.**

Run: `mkdir -p modules/elasticsearch/classes modules/elasticsearch/test/phpunit modules/elasticsearch/templates`
Expected: dirs exist.

- [ ] **Step 2: Write `module.inc`** with identity from requirements §3, `upgrade()` seeding defaults (`enabled=0`, `url=http://localhost:9200`, `index=gallery2`, empty credentials) via `setParameter`, no schema/`configureStore()`, `performFactoryRegistrations()` registering the event listener and search interface, `getModuleEntityTypes()` returning `array()`. Callbacks: none in v1 (admin deferred). Style: legacy (this is an existing-pattern file; match `modules/comment/module.inc`).
- [ ] **Step 3: Write `MANIFEST`** listing `module.inc`, class files (added as created), and `po/messages.po`.
- [ ] **Step 4: Write `po/messages.po` stub** (header only, UTF-8).
- [ ] **Step 5: Verify syntax.**

Run: `docker compose exec php php -l modules/elasticsearch/module.inc`
Expected: `No syntax errors detected`.

- [ ] **Step 6: Commit.**

```bash
git add modules/elasticsearch/module.inc modules/elasticsearch/MANIFEST modules/elasticsearch/po/messages.po
git commit -m "feat(elasticsearch): module scaffold and registration"
```

### Task 2: ElasticsearchClient (transport)

**Files:**
- Create: `modules/elasticsearch/classes/ElasticsearchClient.class`
- Create: `modules/elasticsearch/test/phpunit/ElasticsearchClientTest.class`

**Interfaces:**
- Consumes: `GalleryCoreApi::requestWebPage()`, `GalleryCoreApi::error()`.
- Produces: `ElasticsearchClient` with methods `setConfig($url, $index, $username = '', $password = ''): void`, `ping(): array`, `indexDocument(int $id, array $document): array`, `deleteDocument(int $id): array`, `updateDocument(int $id, array $partial): array`, `search(string $query, array $fields, int $offset, int $limit): array`, `createIndex(array $mapping): array`, `deleteIndex(): array`. Each returns `array($status, ...)` and never throws.

- [ ] **Step 1: Write the failing test** `ElasticsearchClientTest` (modern style + strict_types) covering: successful index (201/200) → success; delete 404 → success; search parses hits/total/highlight; HTTP 5xx → error status; basic auth header present when credentials set; malformed JSON body → error status (Review Focus #1).
- [ ] **Step 2: Run test to verify it fails.**

Run: `/opt/homebrew/bin/bash auto-test.sh http://host.docker.internal:10000/ ElasticsearchClientTest`
Expected: FAIL (class not defined).

- [ ] **Step 3: Implement `ElasticsearchClient`** (modern style + strict_types). Constructor-free; `setConfig` stores url/index/creds. Each method builds method/headers/body, calls `GalleryCoreApi::requestWebPage()`, parses JSON, returns status arrays; HTTP errors → `GalleryCoreApi::error(ERROR_UNKNOWN, ...)`. Basic auth header when creds present. Delete 404 → success. `search` returns `array($status, array('total'=>int, 'hits'=>array(...)))`.
- [ ] **Step 4: Run test to verify it passes.**

Run: `/opt/homebrew/bin/bash auto-test.sh http://host.docker.internal:10000/ ElasticsearchClientTest`
Expected: PASS.

- [ ] **Step 5: Commit.**

```bash
git add modules/elasticsearch/classes/ElasticsearchClient.class modules/elasticsearch/test/phpunit/ElasticsearchClientTest.class
git commit -m "feat(elasticsearch): HTTP transport client"
```

### Task 3: ElasticsearchHelper

**Files:**
- Create: `modules/elasticsearch/classes/ElasticsearchHelper.class`
- Create: `modules/elasticsearch/test/phpunit/ElasticsearchHelperTest.class`

**Interfaces:**
- Consumes: `ElasticsearchClient`, `GalleryCoreApi` (loadEntitiesById, fetchThumbnailsByItemIds not needed here), `GalleryUser` (owner name resolution).
- Produces: `ElasticsearchHelper` with `buildDocument(GalleryItem $item): array`, `indexItem(int $itemId): array`, `deleteItem(int $itemId): array`, `reindexAll(): array`.

- [ ] **Step 1: Write the failing test** `ElasticsearchHelperTest`: `buildDocument` maps item fields (use `_createRandomDataItem()`); verify keys id/title/summary/description/keywords/entityType/mimeType/ownerId/ownerName/parentId/creationTimestamp/modificationTimestamp.
- [ ] **Step 2: Run test to verify it fails** (class not defined).
- [ ] **Step 3: Implement `ElasticsearchHelper`**. `buildDocument` loads owner name (empty on failure). `indexItem` loads item, builds doc, calls client `indexDocument`. `deleteItem` calls client `deleteDocument`. `reindexAll` paginates `SELECT [GalleryItem::id] FROM [GalleryItem]`, indexes each, calls `guaranteeTimeLimit(30)` per batch, returns count/time.
- [ ] **Step 4: Run test to verify it passes.**
- [ ] **Step 5: Commit.**

### Task 4: ElasticsearchIndexer (event listener)

**Files:**
- Create: `modules/elasticsearch/classes/ElasticsearchIndexer.class`
- Create: `modules/elasticsearch/test/phpunit/ElasticsearchIndexerTest.class`

**Interfaces:**
- Consumes: `ElasticsearchHelper`, `GalleryUtilities::isA()`.
- Produces: `ElasticsearchIndexer` implementing `GalleryEventListener` with `handleEvent($event): ?GalleryStatus`.

- [ ] **Step 1: Write the failing test** `ElasticsearchIndexerTest`: on `GalleryEntity::save` for a data item, client index called with item id; on delete, client delete called; non-item entity (e.g. `GalleryUser`) is a no-op; indexing failure is swallowed (Review Focus #3).
- [ ] **Step 2: Run test to verify it fails.**
- [ ] **Step 3: Implement `ElasticsearchIndexer`**. `handleEvent`: get entity; if empty → error; if `!isA($entity, 'GalleryItem')` → return success (no-op); on `GalleryEntity::save` → `ElasticsearchHelper::indexItem()`; on `GalleryEntity::delete` → `ElasticsearchHelper::deleteItem()`; failures logged via `$gallery->debug()` and return `null`.
- [ ] **Step 4: Run test to verify it passes.**
- [ ] **Step 5: Commit.**

### Task 5: ElasticsearchSearch (search provider)

**Files:**
- Create: `modules/elasticsearch/classes/ElasticsearchSearch.class`
- Create: `modules/elasticsearch/test/phpunit/ElasticsearchSearchTest.class`

**Interfaces:**
- Consumes: `ElasticsearchClient`, `GalleryCoreApi` (`fetchPermissionsForItems`, `getActiveUserId`/active user), `GallerySearchInterface_1_0`.
- Produces: `ElasticsearchSearch` with `getSearchModuleInfo(): array` and `search(array $options, string $criteria, int $offset = 0, int $count = -1): array`.

- [ ] **Step 1: Write the failing test** `ElasticsearchSearchTest`: search with criteria returns results; permission filtering excludes private items for a guest (Review Focus #2); empty criteria → empty result + `null` status; disabled/unconfigured → empty result + `null` status.
- [ ] **Step 2: Run test to verify it fails.**
- [ ] **Step 3: Implement `ElasticsearchSearch`**. `getSearchModuleInfo`: name/description; options titles/summaries/descriptions/keywords (all enabled). `search`: if disabled/unconfigured or empty criteria → empty result + null. Map selected options to fields. `multi_match` query; client search with `from`/`size`; highlighting. Collect hit ids; `fetchPermissionsForItems($ids, $activeUserId)`; keep hits where `core.view` present. Format results. On client error: empty result + null; store error in `lastError` param.
- [ ] **Step 4: Run test to verify it passes.**
- [ ] **Step 5: Commit.**

### Task 6: Final verification

- [ ] **Step 1: Syntax check all new files.**

Run: `docker compose exec php php -l modules/elasticsearch/module.inc` and `... ElasticsearchClient.class`, `... ElasticsearchHelper.class`, `... ElasticsearchIndexer.class`, `... ElasticsearchSearch.class`, plus test files.
Expected: `No syntax errors detected` for each.

- [ ] **Step 2: Run module test suite.**

Run: `/opt/homebrew/bin/bash auto-test.sh http://host.docker.internal:10000/ elasticsearch`
Expected: all pass, 0 failures.

- [ ] **Step 3: Run narrowest scope for any failing test** and fix before proceeding. Paste output as evidence.

- [ ] **Step 4: Verify no scratch/untracked files** (`.planning`, `.superpowers`, `tmp-cookies`) staged.

Run: `git status --short`
Expected: only intended changes.

- [ ] **Step 5: Commit any final fixes.**

---

## Execution Handoff

Plan saved to `docs/features/elasticsearch/plans/implementation.md` (canonical) with compatibility symlink at `docs/superpowers/plans/2026-11-17-elasticsearch.md`.

Subagent-driven execution is preselected. **Does it capture what you want?** (Confirm-only; no choice prompt needed.) Once you confirm, I'll begin Task 1 via subagent-driven-development.