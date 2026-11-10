# Agent Guide

This repository is a Gallery 2.3.2-based fork with PHP 8 compatibility changes. Use this guide to locate relevant code and follow existing project patterns; consult the linked project documentation for detailed setup instructions.

## Project map

- `main.php` handles Gallery page requests. `bootstrap.inc` creates and initializes the Gallery instance and loads `config.php`; `init.inc` contains framework initialization.
- `modules/` contains Gallery modules, `themes/` contains theme implementations, and `lib/` contains shared libraries and tools.
- `images/` and `locale/` hold assets and localization data; `install/` and `upgrade/` contain installation and upgrade flows.
- `README.md` describes the fork; `DEVELOPMENT.md` documents the development environment.

## Implementation conventions

Use the existing Gallery patterns in the surrounding module; avoid broad style or API modernization. `modules/comment/module.inc` shows module metadata and callback registration. Request handling and view preparation live in `.inc` files such as `modules/comment/AddComment.inc` (controller and view) and `modules/comment/ShowComments.inc`; reusable comment entities and helpers are in `modules/comment/classes/`.

Views select Smarty templates from `modules/<module>/templates/` (for example, `modules/comment/templates/AddComment.tpl`). Translation sources are in `modules/<module>/po/`, including `messages.po`, with compiled catalogs alongside them. The Comment module is a useful starting example, not a requirement that every module have identical structure.

For framework contracts, inspect the related classes under `modules/core/classes/`, including `GalleryModule.class`, `GalleryController.class`, and `GalleryView.class`. Themes are implemented separately under `themes/<theme>/` with a `theme.inc`, templates, styles, and often their own translations; check a nearby theme when changing theme-facing behavior.

### Gallery forms

In templates, use `{g->hiddenFormVars}` for the return/form URLs and session auth token, and `{g->formVar var="form[field]"}` to name form inputs. Gallery prefixes request variables with `GALLERY_FORM_VARIABLE_PREFIX` (default `g2_`); avoid hard-coding the prefix.

`main.php` selects a controller from the request, collects its `$form` with `GalleryUtilities::getFormVariables('form')`, and passes it to `handleRequest()`. The collector merges matching arrays from `$_POST`, `$_FILES`, and `$_GET`; form inputs conventionally use names such as `form[field]` and `form[action][submit]`. Before handling the request, `main.php` checks its session auth token unless the controller opts out; the check accepts non-persistent guest sessions and is bypassed in maintenance mode.

Controllers branch on `$form['action']` keys and validate field values, then return a redirect, a same-request delegation, or a return result. A delegation preserves request values and can pass error flags such as `form[error][field][missing]` back to the view; `main.php` adds those flags to the request. Redirect after a data change to avoid reposts, but do not put passwords, tokens, or other sensitive values in a redirect URL. `modules/core/UserLogin.inc` shows the pattern: failed login validation delegates to the admin/login view with error flags, while success or cancel returns to the saved destination.

`GalleryView::doLoadTemplate()` reloads the form array and exposes it to Smarty. Views use `formName` to distinguish a new form from a delegated redisplay: initialize defaults for a new form, preserve values intended for redisplay, and render `$form.error` flags next to relevant fields. `UserLoginView` retains the username but does not refill the password; `GalleryCommentHelper::getAddComment()` initializes or normalizes AddComment fields.

For shared forms, `UserAdminView` passes the same form array to its selected subview, while `modules/core/templates/UserAdmin.tpl` owns the form wrapper and common hidden fields and includes the subview body. Keep subview templates such as `modules/core/templates/UserLogin.tpl` to their fields and actions rather than nesting another form. By contrast, `modules/comment/templates/AddComment.tpl` is a standalone-form example: it provides its own wrapper, hidden fields, inputs, errors, and actions.

## Integration examples

### ImageMagick

Use `modules/imagemagick/module.inc` to trace registration: it exposes `imagemagick.AdminImageMagick` in site administration, registers `ImageMagickToolkit` as a `GalleryToolkit`, and advertises supported operations/properties during activation. `modules/imagemagick/AdminImageMagick.inc` handles administrator authorization, path validation, and saving module parameters; `modules/imagemagick/templates/AdminImageMagick.tpl` renders the path, settings, test results, and warnings (including when PHP disables `exec()`).

For binary behavior, start in `modules/imagemagick/classes/ImageMagickToolkitHelper.class`: `testBinaries()` checks the required executables against sample data, while version and binary discovery support configuration and compatibility checks. The module also attempts automatic configuration using `PATH` and common platform directories. `modules/imagemagick/classes/ImageMagickToolkit.class` uses the stored binary path when performing toolkit operations, so changes to configuration or command execution may cross all of these files.

### SquareThumb and ItemAdd

#### SquareThumb

`modules/squarethumb/module.inc` registers the `SquareThumbSiteAdmin` view, a deactivation event listener, and `SquareThumbToolkit`. Activation checks for an underlying toolkit operation that can convert PPM to JPEG, then registers SquareThumb’s high-priority thumbnail operation for supported MIME types. `modules/squarethumb/SquareThumbSiteAdmin.inc` and `modules/squarethumb/templates/SquareThumbSiteAdmin.tpl` manage crop/fit mode and background color; fit mode depends on the PPM conversion capability. `modules/squarethumb/classes/SquareThumbToolkit.class` obtains dimensions and delegates crop, composite, and thumbnail work to other toolkits, so trace both wrapper and underlying toolkit behavior when changing image processing.

#### ItemAdd

`modules/itemadd/module.inc` registers `ItemAddFromWeb` and `ItemAddFromServer` as `ItemAddPlugin` implementations and exposes `ItemAddSiteAdmin` in site administration. `modules/itemadd/ItemAddSiteAdmin.inc` and `modules/itemadd/templates/ItemAddSiteAdmin.tpl` control whether each source is available to all item-add users, site admins only, or disabled, and manage the allowed local-server directories. The server and web plugin behavior is in `modules/itemadd/ItemAddFromServer.inc` and `modules/itemadd/ItemAddFromWeb.inc`, with corresponding templates under `modules/itemadd/templates/`.

When changing these paths, preserve the access checks and directory validation: the server plugin restricts browsing to configured paths, and the admin UI warns that those paths are available to users with item-add permission. The web-import UI also warns that Gallery fetches can be abused to target other websites, so access should be limited to trusted users.

## Development setup

Read `DEVELOPMENT.md` for prerequisites and detailed steps. `docker-compose.yaml` defines Nginx, PHP-FPM, and MariaDB; `docker/PHP.Dockerfile` uses PHP 8.1-FPM, and the optional `PHP_TARGET=debug` target enables Xdebug. The documented `docker compose up -d` command requires Docker Compose; the Gallery installation and writable `/gallery2-data` storage must be completed before using the app or browser tests. Do not copy database or admin credentials into this guide.

Check setup instructions against the current files: `DEVELOPMENT.md` points to `/install.php`, but the repository has `install/index.php`, `main.php` redirects to `install/`, and `docker/nginx.conf` contains no rewrite for `/install.php`. Treat that URL as unverified until the setup documentation or running environment is checked.

## Tests

Tests live in each module’s `test/phpunit/` directory and use Gallery’s custom test cases: for example, `modules/comment/test/phpunit/AddCommentControllerTest.class` extends `GalleryControllerTestCase`, toolkit tests such as `modules/imagemagick/test/phpunit/ImageMagickToolkitTest.class` and `modules/squarethumb/test/phpunit/SquareThumbToolkitTest.class` use `GalleryTestCase`, and `modules/itemadd/test/phpunit/ItemAddFromWebTest.class` extends `ItemAddPluginTestCase`. Test setup typically creates fixtures and marks data or plugin parameters for cleanup; follow the affected module’s examples and test helpers under `lib/tools/phpunit/`.

This is a web-driven runner, not a standalone PHPUnit CLI invocation. Follow `DEVELOPMENT.md` for test prerequisites, including locale setup, then use the browser at `/lib/tools/phpunit/index.php` or the helper syntax `auto-test.sh <SERVER_URL> <SCOPE>`. The scope may select a module, class, or test method; the documented syntax also supports exclusions, test-number ranges, and `:1by1`. The helper requires `USERNAME` and `PASSWORD` in its environment; `DEVELOPMENT.md` says the password is used for both the Gallery admin login and the support-area `setup.password`. Never put actual credentials in this guide or commit them.

## Operational cautions

Run the web test suite only against a development Gallery installation: tests create application data, and the runner commits its transaction. Do not expose `lib/tools/phpunit/index.php` to untrusted users; it is intended for an authenticated site administrator and also requires the support-area password.

`auto-test.sh` clears and writes `tmp-cookies.txt` next to the script and writes a scope-named HTML response in the current directory. The web runner also stores timestamped HTML reports in the configured Gallery data directory under `test/`; treat the cookie jar as an authenticated session artifact and keep generated files out of commits.
