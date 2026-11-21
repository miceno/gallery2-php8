<!-- FOR AI AGENTS - Human readability is a side effect, not a goal -->
<!-- Managed by agent: keep sections and order; edit content, not structure -->
<!-- Last updated: 2026-11-16 | Last verified: 2026-11-16 -->

# AGENTS.md

**Precedence:** the **closest `AGENTS.md`** to the files you're changing wins; explicit user instructions override both.

Gallery 2.3.2 fork ported to PHP 8.1. Deliberately legacy: no project-level Composer manifest or autoloader, no CI, no static analysis, shared globals. Preserve the surrounding Gallery patterns; do not modernize broadly (no PSR-12 sweep, no `strict_types`, no framework swap).

## Commands
| Task | Command | ~Time |
|------|---------|-------|
| Start stack | `docker compose up -d` | ~20s |
| Rebuild PHP | `PHP_TARGET=debug docker compose build php` | ~2min |
| Syntax check | `docker compose exec php php -l modules/core/UserLogin.inc` | ~1s |
| Open installer | `docker compose exec php chmod a+w /gallery2-data` | ~1s |
| Run module tests | `/opt/homebrew/bin/bash auto-test.sh http://host.docker.internal:10000/ comment` | minutes |
| Run one test | `/opt/homebrew/bin/bash auto-test.sh http://host.docker.internal:10000/ AddCommentControllerTest.testAddComment` | ~10s |
| Regenerate entities | `docker compose exec -w /app/modules/comment/classes php php ../../../lib/tools/bin/generate-entities.php` | ~2s |

- Shell: macOS ships Bash 3.2 at `/bin/bash`; Homebrew Bash 5.3.20 is at `/opt/homebrew/bin/bash`. Use the Homebrew path for scripts that need Bash 4+ (for example the agent-rules `validate-structure.sh`) and for the test commands above.
- Install at `http://host.docker.internal:10000/install/`; `PHP_TARGET=debug` adds Xdebug (`base`/unset does not), and switching requires rebuilding and restarting `php`.
- PHP exists only in the `php` container. Test runner UI: `http://host.docker.internal:10000/lib/tools/phpunit/index.php` — type a filter and press Enter (no submit button).

## Workflow
1. Read the closest `AGENTS.md` and the target module's `module.inc` before editing.
2. Copy a golden sample named in that scoped file; modules repeat patterns instead of abstracting.
3. Change the narrowest layer: `.inc` controller/view → helper class → entity/map → `schema.tpl` → regenerated files.
4. Syntax-check changed files in the container, then run the narrowest test scope.
5. Before claiming done, paste the command and its output. "Should work" is not evidence.
6. If research reveals a reusable structure, contract, or rule that is not already documented, update the closest `AGENTS.md` in the same change.

## Repository map
- `main.php` is the front controller; `bootstrap.inc` creates the Gallery instance and loads `config.php`; `init.inc` initializes the framework. `index.php`/`embed.php` are entry variants.
- `modules/` plugins with `modules/core` as framework; `themes/` Smarty themes (`theme.inc`, `templates/`, `theme.css`); `lib/` shared libraries and tools.
- `install/`, `upgrade/` browser flows; `images/`, `locale/` assets; `docker/` nginx + PHP config; setup in `DEVELOPMENT.md`.

## Tech stack
- **Language:** PHP 8.1 (port of Gallery 2.3.2; legacy PHP 4/5 idioms remain by design).
- **Framework:** Gallery 2 (modular plugin system, `modules/core` as framework, Smarty templating in `themes/`).
- **Database:** MySQL/MariaDB via `mysqli` + ADOdb (`lib/adodb`).
- **Web server:** nginx + PHP-FPM (Docker).
- **Frontend/libs:** Smarty, YUI, BBCode, ImageMagick, GD, phpass, JSON.
- **Runtime/dev:** Docker Compose (nginx + php-fpm + mariadb); web-driven PHPUnit test runner.
- **No project-level Composer manifest or autoloader**; third-party code is vendored under `lib/`.

## Core Rules
- Two data shapes: **entities** (`@g2` PHPDoc → generated `Entities.inc`) and **maps** (`Maps.xml` → generated `Maps.inc`); never hand-edit generated files. Storage goes through `GalleryStorage`; queries use `[Class::member]`, never hard-coded prefixes.
- Match the file you edit (tabs, same-line braces, `array()`), keep PHP 8.1 compatibility, and declare properties explicitly — PHP 8 rejects undeclared dynamic properties.
- User-visible text: `$gallery->i18n()` in PHP, `{g->text}` in templates; update `modules/<module>/po/messages.po`. Forms: use `{g->hiddenFormVars}` and `{g->formVar}` instead of hand-written `g2_` names.
- Permissions: register during module upgrade; check server-side with `GalleryCoreApi::hasItemPermission()`, `assertHasItemPermission()` or `fetchPermissionsForItems()`.
- Schema changes: `@g2`/`Maps.xml` → `schema.tpl` → `configureStore()` → regenerate; full recipe in `modules/AGENTS.md`.

## Tests
- Web-driven runner, not a PHPUnit CLI: tests live in `modules/<module>/test/phpunit/`, helpers in `lib/tools/phpunit/`; `auto-test.sh` needs `USERNAME`/`PASSWORD` and takes a regex scope (`:1by1` stops at the first failure).
- Locale files for es/ca/de/fr/pt_BR are prerequisites for some core tests: `lib/tools/phpunit/create-locale.sh`, see `DEVELOPMENT.md`.
- Development installations only: tests write data and the runner commits its transaction. Keep `tmp-cookies.txt` and generated `*.html` reports out of commits.

## Boundaries

### Always Do
- Read the target module's code and tests before changing behavior; keep changes module-local.
- Add a test alongside new behavior; verify with the narrowest command and show the output.
- Use Conventional Commits: `type(scope): subject`, e.g. `fix(comment): keep author on preview`.
- **Changelog:** there is no CHANGELOG file; each PR records user-visible changes in its commit subject/body (types `feat:`, `fix:`, `docs:`, `chore:`), and release history is the git log.

### Ask First
- Schema changes that affect existing installs (`schema.tpl`, upgrade path, `Schema` rows).
- Module dependencies, module registration changes, or regenerating shared `.inc` files.
- Repo-wide modernization, Docker/PHP bumps, or deleting/renaming public classes, permissions or config keys.

### Never Do
- Commit `config.php`, credentials, `tmp-cookies.txt`, generated test HTML or `gallery2-data`.
- Hand-edit generated `Entities.inc`/`Maps.inc`.
- Bypass permission or form auth-token checks, or put passwords/tokens in redirect URLs or logs.
- Mix broad style rewrites into functional changes.

## Codebase State
- PHP 8.1 compatibility port; 2.3.2-era code (PHP 4 idioms, `var` declarations, `array()`) remains by design; Docker plus the web test runner are the only feedback loop. Composer trees under `lib/adodb` are vendored third-party code, not the project build.
- `DEVELOPMENT.md` drifts from the tree: it documents `/install.php` (actual: `install/index.php`; `main.php` redirects to `install/`) and `lib/tools/release` (absent). Trust the files.

## Related Skills
`php-modernization`, `testing-strategy`, `persistence-patterns`, `architecture-migration`, `security-architecture` — load when the task matches. They explain general practice; this file wins on repo specifics.

## Scoped AGENTS.md (MUST read when working in these directories)
- `./modules/AGENTS.md` — plugin anatomy, golden samples, entity/map hierarchy, terminology, permissions, schema recipe, form/view patterns.
- `./modules/search/AGENTS.md` — search contract, aggregate vs paged flows, provider registration, permission filtering, search tests.

## When instructions conflict
The closest `AGENTS.md` wins. Explicit user instructions override both.
