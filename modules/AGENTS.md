<!-- FOR AI AGENTS - Human readability is a side effect, not a goal -->
<!-- Managed by agent: keep sections and order; edit content, not structure -->
<!-- Last updated: 2026-11-16 | Last verified: 2026-11-16 -->

# AGENTS.md — Modules

## Overview
`modules/core` is the framework; every other directory is a plugin module that adds entities, maps, controllers/views, toolkits, permissions, admin screens or event listeners. There is no autoloader: `module.inc` declares what it registers, and `GalleryCoreApi::registerFactoryImplementation()` binds implementations at runtime.

Anatomy (`modules/comment` is the reference): `module.inc` (metadata, activation, `upgrade()`, factory registrations), `*.inc` (controllers and views), `classes/*.class` (entities, helpers), `classes/Maps.xml` plus generated `Maps.inc`/`Entities.inc`, `templates/*.tpl`, `po/messages.po`, `test/phpunit/`, `MANIFEST`.

Data has two shapes:
- **Entities** — one row per object. Persisted members are declared as `@g2 <member>` XML in the class PHPDoc and compiled to `modules/<module>/classes/Entities.inc` by `lib/tools/bin/generate-entities.php`. `GalleryCoreApi::describeEntity()` reads the serialized schema stored per plugin in the `Schema` table, populated by `GalleryStorage::configureStore()`.
- **Maps** — row sets declared in `Maps.xml`, compiled to `Maps.inc` by `lib/tools/bin/generate-maps.php`. Access via `GalleryCoreApi::getMapEntry()`, `addMapEntry()`, `updateMapEntry()`, `removeMapEntry()`.

Storage logic lives in helpers under `modules/core/classes/helpers/` (`GalleryItemHelper_advanced`, `GalleryPermissionHelper_simple`) behind thin `GalleryCoreApi` wrappers.

### Entities
Under `modules/core/classes/` unless noted:
- `GalleryPersistent` tracks modified members; `GalleryEntity` is the stored base: `id`, `creationTimestamp`/`modificationTimestamp`, `serialNumber` (optimistic concurrency), `isLinkable`/`linkId`, `entityType`, `onLoadHandlers`.
- `GalleryChildEntity` adds `parentId`; `GalleryFileSystemEntity` adds `pathComponent` (on-disk name). `GalleryDerivative` and `modules/comment/classes/GalleryComment.class` extend `GalleryChildEntity`.
- `GalleryItem`: `title`, `summary`, `description`, `keywords`, `ownerId`, `originationTimestamp`, `viewedSinceTimestamp`, `canContainChildren`, `renderer`.
  - `GalleryAlbumItem`: `theme`, `orderBy`, `orderDirection`.
  - `GalleryDataItem`: `mimeType`/`size` → `GalleryPhotoItem` (`width`/`height`), `GalleryMovieItem`, `GalleryAnimationItem`, `GalleryUnknownItem`.
  - `GalleryDynamicAlbum`: behaves like an album, never stored; extra properties must be declared (PHP 8 rejects undeclared dynamic properties).
- `GalleryDerivative`: `derivativeSourceId`, `derivativeOperations` (e.g. `scale|640`), `derivativeType` (`DERIVATIVE_TYPE_IMAGE_THUMBNAIL`/`_RESIZE`/`_PREFERRED` in `GalleryConstants.class`), `isBroken`; `GalleryDerivativeImage` adds `width`/`height`.
- Standalone: `GalleryUser` (`userName`, `hashedPassword`, `email`, `language`, `locked`), `GalleryGroup`.

Create: register with `GalleryCoreApi::registerFactoryImplementation('GalleryEntity', …)`, list the type in `getModuleEntityTypes()` (see `modules/thumbnail/module.inc`), then `newFactoryInstance('GalleryEntity', 'GalleryComment')` + `$entity->create()` (see `modules/comment/AddComment.inc`). Load with `loadEntitiesById()`, delete with `deleteEntityById()`. `save()`/`delete()` raise `GalleryEntity::save`/`GalleryEntity::delete` events for `GalleryEventListener` implementations. Wrap tree- or permission-affecting changes in `acquireReadLock()`/`acquireWriteLock()` and release with `releaseLocks()`.

### Maps
Core maps in `modules/core/classes/Maps.xml`: `GalleryAccessMap` (bitmask per accessListId × userOrGroupId), `GalleryAccessSubscriberMap` (item → access-list cache), `GalleryPermissionSetMap`, `GalleryItemAttributesMap` (`orderWeight`, `parentSequence`, `viewCount`), `GalleryUserGroupMap`, `GalleryPluginMap`, `GalleryPluginParameterMap`, `GalleryFactoryMap`, the `GalleryToolkit*` maps, plus session/cache/maintenance tables.

### Permissions
Register item permissions on first install with `GalleryCoreApi::registerPermission()` in `module.inc`'s `upgrade()` (pattern: `modules/comment/module.inc`); composites set `GALLERY_PERMISSION_COMPOSITE` and list child permission names. Check with `hasItemPermission()`, `assertHasItemPermission()` or `fetchPermissionsForItems()`. Well-known groups: `GROUP_ALL_USERS`, `GROUP_SITE_ADMINS`, `GROUP_EVERYBODY` (`GalleryConstants.class`).

### Schema changes
1. Declare members in `@g2` markup (or `Maps.xml`) and bump `@g2 <schema-minor>`.
2. Update the end-state `CREATE TABLE` and add an `# A_<Class>_<major>.<minor>` block in `modules/<module>/classes/GalleryStorage/schema.tpl` under every platform (`## mysql`, `## postgres`, `## oracle`, `## db2`, `## mssql`, `## sqlite`); keep the matching `Schema` row in step. Block kinds: `#` create, `A_` alter, `R_` remove, `T_` fixtures.
3. Call `$storage->configureStore($this->getId(), array('<Class>:<maj.min>'))` from the matching `upgrade()` case and bump `setVersion()`.
4. Regenerate `Entities.inc`/`Maps.inc` and add per-version records under `modules/<module>/classes/GalleryStorage/xml-src/`.

## Setup
PHP 8.1 runs only in the Docker `php` service; the host may have no PHP CLI. No Composer, build step or static analysis. Generators run from the module's `classes/` directory (they glob `*.class` and write in place), e.g. `docker compose exec -w /app/modules/comment/classes php php ../../../lib/tools/bin/generate-entities.php`. The `GNUmakefile` in each `classes/` dir wraps the same scripts when `make` is installed.

## Build & tests
- Tests live in `modules/<module>/test/phpunit/` and run through the web runner at `/lib/tools/phpunit/index.php` or `bash auto-test.sh <SERVER_URL> <SCOPE>` from the repo root.
- Extend `GalleryTestCase`, `GalleryControllerTestCase`, `GalleryViewTestCase` or `GalleryImmediateViewTestCase` (`lib/tools/phpunit/`).
- Follow `modules/comment/test/phpunit/AddCommentControllerTest.class`: `setUp()` creates fixtures with `_createRandomDataItem()`, marks entities with `_markForCleanup()` and plugin parameters with `_markPluginParametersForCleanup()`.
- The runner commits its transaction and creates data; never point it at a production site.

## Code style
- Imitate the surrounding file: tabs, same-line braces, `array()`, no `declare(strict_types=1)`, no namespaces. Keep PHP 8.1 compatible without starting a repo-wide modernization.
- Declare object properties explicitly; undeclared dynamic properties are deprecated in PHP 8.2 (tests already declare `$_saveMarkup`, `$_startTime`).
- Prefer `GalleryCoreApi` wrappers over direct storage calls; keep controllers thin and put reusable logic in helpers.
- Translate user-visible strings with `$gallery->i18n()` (PHP) or `{g->text}` (templates), and update `modules/<module>/po/messages.po`.
- Templates use `{g->hiddenFormVars}`, `{g->formVar var="form[field]"}`, `{g->url}`; pass user HTML through the `|markup` filter, otherwise escape with `|escape`.
- Forms: `main.php` collects `form[...]` from POST/FILES/GET and calls `handleRequest()`; `GalleryView::doLoadTemplate()` reloads the form, and `formName` distinguishes a new form from a delegated redisplay (initialize defaults only for a new form).

## Security
- Enforce item permissions server-side in the controller or helper; hidden UI is not a control.
- Keep the auth-token check: only genuinely public controllers override `omitAuthTokenCheck()`. Non-persistent guest sessions skip it, and maintenance mode tolerates a wrong token; do not widen either bypass.
- Redirect after data changes; never put passwords, tokens or session values in redirect URLs.
- Validate and sanitize form/plugin input. The `itemadd` server paths and web fetches are trusted-user features — preserve their access checks and directory validation.
- Escape template output unless content is explicitly markup-filtered.

## Terminology
| Term | Means |
|------|-------|
| Entity | Persisted object class (GalleryItem, GalleryUser, …); one storage row |
| Map | Row set declared in Maps.xml (permissions, attributes, plugin parameters) |
| Item | Album-tree node: album or data item |
| Derivative | Generated image (thumbnail, resize, preferred) |
| Toolkit | Image-operation backend (imagemagick, netpbm, gd) or wrapper (squarethumb) |
| Dynamic album | Virtual album computed per request, never stored |

## Checklist
- [ ] `module.inc` declares every new controller, view, factory implementation and permission.
- [ ] New behavior has a test in `modules/<module>/test/phpunit/` with fixtures marked for cleanup.
- [ ] Schema changes include `schema.tpl` blocks, the `configureStore()` call, regenerated `.inc` files and `xml-src` records.
- [ ] New strings are in `modules/<module>/po/messages.po`; no hard-coded user-visible English.
- [ ] Generated files were regenerated, not edited.

## Examples

### Golden samples
| For | Reference | Key patterns |
|-----|-----------|--------------|
| Module anatomy, permissions, standalone form | `modules/comment/module.inc`, `modules/comment/AddComment.inc`, `modules/comment/classes/GalleryCommentHelper.class`, `modules/comment/templates/AddComment.tpl` | `upgrade()` registrations, controller/view split, own form wrapper |
| Shared form with subviews | `modules/core/UserAdmin.inc` + `modules/core/templates/UserAdmin.tpl` | view owns wrapper/hidden fields, subview holds fields only |
| Controller/view/delegation | `modules/core/UserLogin.inc` | `form[error][field][missing]` flags, saved destination, password never refilled |
| Entity + factory | `modules/thumbnail/module.inc`, `modules/thumbnail/classes/ThumbnailImage.class` | `registerFactoryImplementation`, `getModuleEntityTypes`, `@g2` schema |
| Toolkit adapter | `modules/imagemagick/classes/ImageMagickToolkit.class`, `ImageMagickToolkitHelper.class` | PATH-based binary discovery, `testBinaries`, exec-disabled warning |
| Wrapper toolkit | `modules/squarethumb/classes/SquareThumbToolkit.class` | delegates crop/composite/thumbnail; fit mode needs PPM→JPEG support |
| Import plugin | `modules/itemadd/ItemAddFromWeb.inc`, `ItemAddFromServer.inc` | `ItemAddPlugin`, trusted-user access, warns that web fetches can be abused |

- `UserAdminView` owns the wrapper in `modules/core/templates/UserAdmin.tpl`; `modules/core/templates/UserLogin.tpl` holds only fields/actions and never nests another form.
- `modules/core/UserLogin.inc`: failed validation delegates with `form[error][field][missing]`; success/cancel returns to the saved destination; `UserLoginView` keeps the username but never the password.
- ImageMagick chain: `module.inc` → `AdminImageMagick.inc`/template → `ImageMagickToolkitHelper::testBinaries()` → `ImageMagickToolkit`.
- ItemAdd: `module.inc` registers both plugins; `ItemAddSiteAdmin.inc` sets availability and allowed server directories.

## When stuck
- Read the contracts: `modules/core/classes/GalleryModule.class`, `GalleryController.class`, `GalleryView.class`.
- Every storage/factory entry point is a static method on `GalleryCoreApi` — search there first.
- Setup, Docker, locales, Xdebug: `DEVELOPMENT.md`. Test filter syntax and exclusions: `lib/tools/phpunit/index.php`.
- Ask before changing schema, permissions, generated files or shared registration.
