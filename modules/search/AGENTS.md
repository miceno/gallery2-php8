<!-- FOR AI AGENTS - Human readability is a side effect, not a goal -->
<!-- Managed by agent: keep sections and order; edit content, not structure -->
<!-- Last updated: 2026-11-16 | Last verified: 2026-11-16 -->

# AGENTS.md — Search

## Overview
`modules/search` owns the cross-module search contract and the user-facing search flows. Other modules contribute results by implementing `GallerySearchInterface_1_0`; this module discovers them at runtime and renders the aggregate UI. Read `modules/AGENTS.md` first for plugin anatomy, entities, maps, permissions and form/view patterns; this file covers only search-specific rules.

## Key Files
| File | Role |
|------|------|
| `module.inc` | Thin module metadata: id `search`, version `1.0.8`, no `upgrade()`, no factory registrations, empty callbacks. |
| `classes/GallerySearchInterface_1_0.class` | The contract every searchable module implements. |
| `classes/SearchUtilities.class` | `sanitizeSearchCriteria()`; currently only escapes `%` to `%%`. |
| `SearchScan.inc` | Aggregate search across all registered providers (scan size 3); handles slideshow and block form names. |
| `SearchShowAll.inc` | Paged search for one provider (`pageSize = 20`), with next/previous and `lastSearchCriteria`. |
| `templates/SearchScan.tpl`, `templates/SearchShowAll.tpl` | Result markup, pagination and empty/error states. |
| `templates/blocks/SearchBlock.tpl`, `SearchBlock.js`, `Preloads.inc` | Sidebar/block search form and its preload registration. |
| `HighlightResults.js` | Highlights matching text in results. |
| `test/phpunit/SearchUtilitiesTest.class` | Existing unit coverage; extend it for new search behavior. |

## Search contract
`GallerySearchInterface_1_0` has two methods; both return `array($ret, $data)`:

- `getSearchModuleInfo()` returns `array('name' => ..., 'description' => ..., 'options' => array($optionKey => array('description' => ..., 'enabled' => 0|1)))`.
- `search($options, $criteria, $offset=0, $count=-1)` returns `array('start' => ..., 'end' => ..., 'count' => ..., 'results' => array(array('itemId' => int, 'fields' => array(array('key' => ..., 'field' => ..., 'value' => ...)))))`.
- `$offset` is zero-based; `$count = -1` means no limit. `$options` is the subset the user selected, keyed by the option keys from `getSearchModuleInfo()`.
- `SearchScan` calls providers with `$count = 3`; `SearchShowAll` computes `$offset = ($page - 1) * 20` and `$count = 20`.
- Providers receive criteria already passed through `SearchUtilities::sanitizeSearchCriteria()`. They must still escape wildcards/query syntax for their own backend.

## Golden Samples (follow these patterns)
| For | Reference |
|-----|-----------|
| Database-backed core search | `modules/core/classes/GalleryCoreSearch.class` — `LIKE` on `[GalleryItem::title|summary|keywords|description]`, joins `[GalleryAccessSubscriberMap]`, filters by `core.view` access-list ids, returns owner + field rows. |
| Module-specific search | `modules/comment/classes/GalleryCommentSearch.class` — searches comments and maps hits back to parent item ids. |
| Field variants | `modules/multilang/classes/MultiLangSearch.class`, `modules/customfield/classes/CustomFieldSearch.class`, `modules/mapv3/classes/mapv3Search.class`. |
| Registration | `modules/core/CoreModuleExtras.inc` (core search) and `performFactoryRegistrations()` in `modules/comment/module.inc`, `modules/multilang/module.inc`, `modules/customfield/module.inc`, `modules/mapv3/module.inc`. |

## Setup & environment
- PHP 8.1 runs only in the Docker `php` service; no host PHP CLI, no Composer, no static analysis.
- Use `/opt/homebrew/bin/bash` for `auto-test.sh`; the system `/bin/bash` is 3.2.
- Start the stack with `docker compose up -d`; install at `http://host.docker.internal:10000/install/`.
- Tests need `USERNAME`/`PASSWORD` and, for some core tests, the locale packages described in `DEVELOPMENT.md`.

## Build & tests
- Run the module scope: `/opt/homebrew/bin/bash auto-test.sh http://host.docker.internal:10000/ search`
- Run one test: `/opt/homebrew/bin/bash auto-test.sh http://host.docker.internal:10000/ SearchUtilitiesTest`
- Web runner: `http://host.docker.internal:10000/lib/tools/phpunit/index.php` — type a filter and press Enter.
- New search behavior needs a test in `modules/search/test/phpunit/`; use `GalleryControllerTestCase`/`GalleryViewTestCase` for the flows and mark fixtures with `_markForCleanup()`.
- Tests commit their transaction and create data; use a development installation only.

## Code style & conventions
- Match the surrounding legacy style: tabs, same-line braces, `array()`, no `declare(strict_types=1)`, no namespaces.
- Keep controllers thin; put reusable result shaping in `classes/`.
- Translate user-visible text with `$gallery->i18n()` / `$module->translate()` in PHP and `{g->text}` in templates.
- Templates use `{g->hiddenFormVars}`, `{g->formVar var="form[field]"}`, `{g->url}`; escape output unless content is intentionally markup.
- Do not add module dependencies or change `GallerySearchInterface_1_0` without asking; other modules implement it.
- `module.inc` intentionally has no `upgrade()` and no factory registrations. Core registers `GalleryCoreSearch`; other providers register themselves.

## Security & safety
- Providers are responsible for permission filtering. `SearchScan`/`SearchShowAll` load and render whatever item ids a provider returns; they do not re-check permissions.
- Filter with `GalleryCoreApi::fetchAccessListIds('core.view', $activeUserId)` (core pattern) or `GalleryCoreApi::fetchPermissionsForItems()` before returning ids.
- Never return item ids the active user cannot view, even if the underlying backend indexed them.
- Keep the default auth-token check; only genuinely public controllers override `omitAuthTokenCheck()`.
- Validate `form[searchCriteria]` as a string, trim it, and let `SearchUtilities::sanitizeSearchCriteria()` run before provider queries.
- Do not put credentials, tokens or session values in redirect URLs or logs.

## PR/commit checklist
- [ ] Provider implements both `getSearchModuleInfo()` and `search()` with the documented return shape.
- [ ] Results are permission-filtered server-side.
- [ ] `SearchScan` and `SearchShowAll` still work (scan limit, paging, `lastSearchCriteria`).
- [ ] New strings are in `modules/search/po/messages.po` or the provider's own `po/messages.po`.
- [ ] New behavior has a test and the narrowest test command was run.
- [ ] No generated files were hand-edited; no broad style rewrite mixed in.

## Patterns to Follow
- `SearchScanController` delegates to `search.SearchShowAll` for “show all” and to the slideshow interface for slideshows.
- `SearchScanView` collects `getSearchModuleInfo()` for every provider, then calls `search()` with `$count = 3`; it also accepts `formName` values `SearchScan`, `SearchShowAll` and `search_SearchBlock`.
- `SearchShowAllView` keeps `lastSearchCriteria` when paging so next/previous does not change the query, and resets `page` on a new search.
- `SearchUtilitiesTest` is the minimal test shape: `GalleryCoreApi::requireOnce(...)` then assert on the static helper.
- Copy a real provider rather than inventing a new result shape.

## When stuck
- Read the contract: `modules/search/classes/GallerySearchInterface_1_0.class`.
- Read a working provider: `modules/core/classes/GalleryCoreSearch.class` (DB) or `modules/comment/classes/GalleryCommentSearch.class` (module-specific).
- Registration entry points: `GalleryCoreApi::getAllFactoryImplementationIds()`, `newFactoryInstance()`, `newFactoryInstanceById()`.
- Permission helpers: `fetchAccessListIds()`, `fetchPermissionsForItems()`, `hasItemPermission()`.
- Project-wide setup, Docker, locales and Xdebug: root `AGENTS.md` and `DEVELOPMENT.md`.
