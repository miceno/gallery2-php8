# Requirements: Elasticsearch Photo Indexing for Gallery 2

## 1. Purpose and scope

Add an optional **Elasticsearch** backend that indexes Gallery items (photos and other items) and provides full-text search with ranking and highlighting, while preserving Gallery's permission model and legacy architecture.

**In scope**

- A reusable Elasticsearch storage/transport interface (add, delete, update, query).
- Index lifecycle hooks so item create/update/delete keeps the index in sync.
- A search form and result templates (dedicated view plus integration with the existing search framework).
- Site-admin configuration and a bulk reindex action.
- Permission-filtered result sets.

**Out of scope (initial)**

- Indexing comments, tags from other modules, or derivative metadata beyond the item itself.
- Distributed/cluster tuning, ILM, or analyzers beyond a sensible default mapping.
- Replacing the existing database `LIKE` search; Elasticsearch is additive.

## 2. Constraints (must respect)

- PHP 8.1 only, no Composer/autoload; module-local code; follow `modules/AGENTS.md`.
- No external PHP Elasticsearch client library; HTTP via `GalleryCoreApi::requestWebPage()` / platform `fsockopen`, JSON via `json_encode`/`json_decode`.
- Keep legacy style: tabs, same-line braces, `array()`, explicit properties, no `strict_types`.
- Tests run through the web runner; HTTP must be mockable via the platform's `setReply('fsockopen', ...)`.
- No schema/entity changes required (settings stored as plugin parameters).

## 3. Module identity

New module `modules/elasticsearch/`.

- Class: `ElasticsearchModule extends GalleryModule`
- `setId('elasticsearch')`
- `setName($gallery->i18n('Elasticsearch Search'))`
- `setVersion('1.0.0')`
- `setGroup('gallery', $gallery->i18n('Gallery'))`
- `setRequiredCoreApi(array(7, 54))`
- `setRequiredModuleApi(array(3, 9))`
- `setCallbacks('getSiteAdminViews')`

## 4. Components

### 4.1 `modules/elasticsearch/classes/ElasticsearchClient.class`

Transport/storage interface. Stateless utility class (like `AkismetApi`) configured from plugin params or constructor args.

**Public methods**

- `setConfig($url, $index, $username='', $password='')`
- `ping()` -> `array($status, bool)`
- `indexDocument($id, $document)` -> `array($status)` (PUT/POST `/{index}/_doc/{id}`)
- `deleteDocument($id)` -> `array($status)` (DELETE `/{index}/_doc/{id}`; treat 404 as success)
- `updateDocument($id, $partial)` -> `array($status)` (POST `/{index}/_update/{id}` with `{"doc":...}`)
- `search($query, $fields, $offset, $limit)` -> `array($status, $result)` where `$result = array('total'=>int, 'hits'=>array(array('id'=>int, 'score'=>float, 'source'=>array, 'highlight'=>array)))`
- `createIndex($mapping)` -> `array($status)` (PUT `/{index}` with mapping)
- `deleteIndex()` -> `array($status)`

**Implementation notes**

- Use `GalleryCoreApi::requestWebPage()` with method/headers/body; parse JSON.
- Basic auth header when credentials present.
- All HTTP errors converted to `GalleryCoreApi::error(ERROR_UNKNOWN, ...)` with the Elasticsearch error message.
- Must not throw; return status arrays consistently.
- Testable by mocking `fsockopen` via the platform.

### 4.2 `modules/elasticsearch/classes/ElasticsearchHelper.class`

Indexing helpers.

- `buildDocument($item)` -> associative array:
  - `id` (item id), `title`, `summary`, `description`, `keywords`
  - `entityType` (class name), `mimeType` (if data item)
  - `ownerId`, `ownerName` (resolve user; empty on failure)
  - `parentId`
  - `creationTimestamp`, `modificationTimestamp`
- `indexItem($itemId)` -> load item via `GalleryCoreApi::loadEntitiesById`, build document, client index.
- `deleteItem($itemId)` -> client delete.
- `reindexAll()` -> iterate all `GalleryItem` ids via storage search with pagination (`SELECT [GalleryItem::id] FROM [GalleryItem]`), index each, call `guaranteeTimeLimit(30)` per batch.

### 4.3 Event listener `modules/elasticsearch/classes/ElasticsearchIndexer.class`

Registered as `GalleryEventListener` for `GalleryEntity::save` and `GalleryEntity::delete`.

`handleEvent($event)`:

- `$entity = $event->getEntity()`; if empty -> error.
- If `!GalleryUtilities::isA($entity, 'GalleryItem')` -> return success (no-op).
- On `GalleryEntity::save`: call `ElasticsearchHelper::indexItem($entity->getId())`.
- On `GalleryEntity::delete`: call `ElasticsearchHelper::deleteItem($entity->getId())`.
- Indexing failure must not block the user action; log via `$gallery->debug()` and return `null`.

### 4.4 Search interface `modules/elasticsearch/classes/ElasticsearchSearch.class`

Extends `GallerySearchInterface_1_0` so the existing `SearchScan`/`SearchShowAll` aggregates it.

- `getSearchModuleInfo()`: name/description; options `titles`, `summaries`, `descriptions`, `keywords` (all enabled by default).
- `search($options, $criteria, $offset, $count)`:
  - If module disabled or unconfigured -> return empty result set (`count 0`), status `null`.
  - Map selected options to Elasticsearch fields.
  - Build a `multi_match` query on selected fields; execute client search with `from=$offset`, `size=$count`, highlighting enabled.
  - Collect hit ids; call `GalleryCoreApi::fetchPermissionsForItems($ids, $activeUserId)`; keep hits where `core.view` is present.
  - Format each hit as `array('itemId'=>..., 'fields'=>array(...))` with highlighted snippet values, plus owner.
  - Return `array(null, array('start'=>..., 'end'=>..., 'count'=>filteredCount, 'results'=>...))`.
  - On client error: return empty result set and store the last error in a module parameter for admin display.

### 4.5 Dedicated search view

File `modules/elasticsearch/ElasticsearchSearch.inc` containing controller and view.

- Controller `ElasticsearchSearchController` (extends `GalleryController`):
  - `handleRequest($form, &$template)` validates `form[searchCriteria]`, calls `ElasticsearchSearch::search(...)`, assigns results and thumbnails (via `GalleryCoreApi::fetchThumbnailsByItemIds`).
  - Auth token check enabled (default).
- View `ElasticsearchSearchView` (extends `GalleryView`):
  - `doLoadTemplate()` loads `modules/elasticsearch/templates/ElasticsearchSearch.tpl`.
  - Initializes defaults only for a new form (`formName`).

### 4.6 Admin settings

File `modules/elasticsearch/ElasticsearchSiteAdmin.inc`.

- Controller and view for site admin:
  - Fields: `enabled` (checkbox), `url`, `index`, `username`, `password` (password field, never refill after submit; follow `UserLogin` pattern), `verify` (ping button), `createIndex` (button), `reindex` (button).
  - Permission: `GalleryCoreApi::assertUserIsSiteAdministrator()`.
  - Save params via `GalleryCoreApi::setPluginParameter('module', 'elasticsearch', ...)`.
  - Reindex triggers `ElasticsearchHelper::reindexAll()`; show count/time.
- Registered via `getSiteAdminViews()`.

### 4.7 Templates

- `modules/elasticsearch/templates/ElasticsearchSearch.tpl`
  - Search form using `{g->hiddenFormVars}`, `{g->formVar}`.
  - Results list with thumbnail, title link to `core.ShowItem`, highlighted snippet, owner, pagination.
  - Empty state and error state messages.
- `modules/elasticsearch/templates/ElasticsearchSiteAdmin.tpl`
  - Settings form and action buttons.

### 4.8 Module registration

`performFactoryRegistrations()`:

- Register `GalleryEventListener` -> `ElasticsearchIndexer` for hints `array('GalleryEntity::save', 'GalleryEntity::delete')`.
- Register `GallerySearchInterface_1_0` -> `ElasticsearchSearch` id `elasticsearch`.
- Ensure `ElasticsearchSearch.inc` and `ElasticsearchSiteAdmin.inc` are listed in `MANIFEST`.

`upgrade($currentVersion)`:

- On initial install set defaults: `enabled=0`, `url=http://localhost:9200`, `index=gallery2`, empty credentials.
- Bump version strings only; no schema.

## 5. Permission handling

- Index contains item ids and metadata for **all** items (including private) so reindex is complete.
- At query time, filter hit ids by `core.view` using `GalleryCoreApi::fetchPermissionsForItems($ids, $activeUserId)`.
- Admin reindex requires site admin.
- Do not store passwords/tokens in URLs or logs; mask them in debug output.

## 6. Failure modes and degradation

- If Elasticsearch is unreachable or returns an error: the dedicated search view shows a user-visible error; `ElasticsearchSearch::search` (interface) returns empty results plus `null` status so `SearchScan` does not crash.
- Indexing errors during save/delete are logged and swallowed (the user action succeeds).
- `ping()` is used by the admin screen to validate configuration before enabling.

## 7. Testing requirements

New tests under `modules/elasticsearch/test/phpunit/`:

1. `ElasticsearchClientTest`
   - Mock `fsockopen` replies to test:
     - successful index (201/200) returns success
     - delete 404 treated as success
     - search parses hits/total/highlight
     - HTTP 5xx returns error status
     - basic auth header present when credentials are set
2. `ElasticsearchHelperTest`
   - `buildDocument` maps item fields correctly (use `_createRandomDataItem()`).
3. `ElasticsearchIndexerTest`
   - On `GalleryEntity::save` for a data item, client index is called with the item id.
   - On delete, client delete is called.
   - Non-item entities (for example, a user) are a no-op.
4. `ElasticsearchSearchControllerTest` / view test
   - Search with criteria returns results; permission filtering excludes private items for a guest.
   - Empty criteria -> validation error.
5. Cleanup: mark entities and plugin params with `_markForCleanup()` / `_markPluginParametersForCleanup()`.

Test commands:

- `bash auto-test.sh http://host.docker.internal:10000/ elasticsearch`
- `bash auto-test.sh http://host.docker.internal:10000/ ElasticsearchClientTest`

## 8. Acceptance criteria

- Activating the module registers the event listener and search interface without errors.
- Admin can configure the Elasticsearch URL/index/credentials and ping successfully.
- Creating an index and running reindex indexes all existing items.
- Creating/updating/deleting an item updates the Elasticsearch index (verified by mocked HTTP in tests and, when Elasticsearch is available, by direct query).
- The search form returns ranked results with thumbnails and highlights.
- Results respect `core.view` permissions (a guest cannot see private items).
- An Elasticsearch outage does not break item editing or gallery browsing.

## 9. Files to create

```
modules/elasticsearch/module.inc
modules/elasticsearch/ElasticsearchSearch.inc
modules/elasticsearch/ElasticsearchSiteAdmin.inc
modules/elasticsearch/classes/ElasticsearchClient.class
modules/elasticsearch/classes/ElasticsearchHelper.class
modules/elasticsearch/classes/ElasticsearchIndexer.class
modules/elasticsearch/classes/ElasticsearchSearch.class
modules/elasticsearch/templates/ElasticsearchSearch.tpl
modules/elasticsearch/templates/ElasticsearchSiteAdmin.tpl
modules/elasticsearch/po/messages.po
modules/elasticsearch/test/phpunit/ElasticsearchClientTest.class
modules/elasticsearch/test/phpunit/ElasticsearchHelperTest.class
modules/elasticsearch/test/phpunit/ElasticsearchIndexerTest.class
modules/elasticsearch/test/phpunit/ElasticsearchSearchControllerTest.class
modules/elasticsearch/MANIFEST
```

## 10. AGENTS.md update (search findings)

The verified search integration points are documented in `modules/AGENTS.md` under **Search integration**:

- Implement `GallerySearchInterface_1_0` (`modules/search/classes/GallerySearchInterface_1_0.class`) and register it in `performFactoryRegistrations()` to appear in `SearchScan`/`SearchShowAll`.
- Event-driven indexing: register a `GalleryEventListener` for `GalleryEntity::save`/`GalleryEntity::delete`; guard with `GalleryUtilities::isA($entity, 'GalleryItem')`.
- Filter results at query time with `GalleryCoreApi::fetchPermissionsForItems()`.
- HTTP integrations follow the `AkismetApi` pattern: `GalleryCoreApi::requestWebPage()` plus JSON; mock via platform `fsockopen` replies in tests.
