# ezmvc CLI

Scaffolding tool for the [ezmvc](https://github.com/patrik-gacsal/ezmvc) PHP framework. Generates controllers, entities with DTOs, views, request/response classes, and builds route + Swagger files.

> **Pre-release.** The CLI is at 0.x: commands and flags may change between minor versions until 1.0.0.

## Installation

```bash
composer global require ezmvc/cli
```

Make sure Composer's global `bin` directory is in your `PATH` (`~/.composer/vendor/bin` on Linux/macOS, `%APPDATA%\Composer\vendor\bin` on Windows).

## Usage

Run commands from your project root (the directory containing `index.php` and `Config.php`).

### Deploy via FTPS

> **Experimental.** Deployment to a local path is tested; deployment over FTPS, including `--zip`, has not yet been verified against a real host. Try it on a throwaway target with `--dry-run` first.

Copy `deploy.json.sample` to `deploy.json` in your project root (gitignored) and fill in your FTP credentials, then:

```bash
ezmvc deploy                   # upload all non-excluded files to the server
ezmvc deploy --changed         # only files modified since the last successful deploy
ezmvc deploy --dry-run         # preview what would be uploaded without uploading
ezmvc deploy --full            # upload vendor/ even when composer.lock is unchanged
ezmvc deploy --no-maintenance  # upload in place, without the 503 window
ezmvc deploy --zip             # one archive, unpacked server-side (https only)
```

Before uploading, the tool probes the server's PHP version and extensions and aborts if the server is older than the local PHP. The probe is an authorized request to the maintenance stub rather than a separate file — the framework's catch-all rewrite sends every non-public path to `index.php`, so a standalone probe script never executes. Leftover probe files and deploy archives from previous interrupted deploys are swept at the start of each run. `Config.php`, `deploy.json`, `logs/`, and `private/` are always excluded.

**Maintenance window.** Files go up one at a time, so mid-upload the server holds a mix of old and new code. The deploy therefore replaces `index.php` with a 503 stub first and restores the real one last — every non-public path routes through `index.php`, so that one file takes the app offline and brings it back. A `_ezmvc_maintenance` marker on the server makes an interrupted deploy detectable: the next run reports it and restores `index.php` at the end regardless of whether it changed. Skipped automatically when `index.php` is excluded (nothing would restore it), or on `--no-maintenance`.

**vendor/ is keyed to composer.lock, not timestamps.** A dependency tree is thousands of files at one FTP round trip each, and every `composer install` rewrites their mtimes — so `--changed` alone would re-upload the whole tree after any Composer command. The deploy stores the `composer.lock` hash instead and skips `vendor/` while it matches; `--full` overrides. Run `composer install --no-dev --optimize-autoloader` before deploying to keep the tree small.

**Failed uploads are not recorded.** The deploy state (what `--changed` compares against) is written only when every file succeeded, and the command exits non-zero otherwise — so a partial deploy is retried in full next time instead of leaving files that never landed marked as deployed.

**`--zip` uploads one archive and has the server unpack it**, turning thousands of FTP round trips into one. The unpacking is done by the maintenance stub, which is the only thing on the server guaranteed to be both reachable (the catch-all routes to `index.php`) and temporary (the real `index.php` replaces it at the end). It accepts commands only with a per-run 128-bit token — SHA-256 in the file, compared with `hash_equals`, sent in a header rather than the URL — only over https, and only before a hard 30-minute expiry, after which the file is an inert 503 page. Everything unauthorized gets that same 503, with no 401/403 to reveal the endpoint. Archive entries are validated before anything is written (no `..`, absolute paths, backslashes, drive letters, or `index.php`), and the payload is deleted by the server, again by the CLI, and swept on the next run. Needs the `zip` extension at both ends; falls back to file-by-file if the server lacks it.

If your host gives you SSH or git, prefer a real deployment tool (Deployer, rsync, `git pull`) — those switch releases atomically, which FTP fundamentally cannot.

### Generate routes and Swagger spec

Must be run after every change to route attributes (`#[Get]`, `#[Post]`, etc.):

```bash
ezmvc generate
```

Writes `generated/routes.php` and `generated/swagger.json`. If the project's `composer.json` has an `autoload.classmap` section (see below), this also runs `composer dump-autoload` to pick up any new entity/dto/service/etc. classes.

It also rebuilds `generated/app.css` from every `templates/**.css` — the co-located stylesheets `ezmvc new view` generates. Installed addons (`vendor/ezmvc/*/templates/`) are concatenated first and the project's own `templates/` last, so a project rule wins over an addon's at equal specificity. Rules are joined verbatim; the only transformation is that CSS comments are stripped, so the served file describes neither the project's directory layout nor the framework that built it — a comment whose opener ends in `!` is kept, the usual licence-header convention, and the source `.css` files keep all their comments since they are never served. Each file confines itself by nesting its rules inside the view's own class, the wrapper the scaffolder emits. It lands in `generated/` rather than `public/` because it is a build artifact; the framework's `.htaccess` whitelists it by exact name, the same way it does `generated/swagger.json`.

### Scaffold Entity/Dto from an existing database

```bash
ezmvc scaffold                    # every table in the project's configured DB
ezmvc scaffold --table=users      # specific table(s), comma-separated
ezmvc scaffold --overwrite        # replace existing generated files
ezmvc scaffold --db-class=ShopDb  # name the per-database class (default: from the DB name)
ezmvc scaffold --no-db-class      # skip the per-database class
```

By default it connects using the project's own `Config.php` (same credentials `app.php` uses). For a table that lives in a different database, pass an explicit connection instead:

```bash
ezmvc scaffold --host=localhost --db=other_db --user=root --pass=secret
```

`Dto` is a single class per entity — used for both API input and output on simple, single-entity endpoints, in `dtos/`. Reserve hand-written `requests/`/`responses/` classes (`ezmvc new request`/`ezmvc new response`) for payloads that span more than one entity or are shaped around a specific action rather than a single resource.

Generated Entity and Dto properties are all camelCased and consistent with each other (`numeric_value` → `$numericValue`), typed from the actual column types (nullable columns get `?type`, `tinyint(1)` becomes `bool`). Where a property's name differs from the real column, the Entity property gets `#[Column('numeric_value')]`; the class itself gets `#[Table('table_name')]` instead of a `setTableName()` call, and `#[Id]`/`#[ReadOnlyField]` mark the primary key and read-only columns instead of `setIdField()`/`setReadOnlyFields()` calls. Each Entity also gets a `const` per column (e.g. `TestEntity::NUMERIC_VALUE === 'numeric_value'`, always the real column name) so `SearchCriteria::eq(TestEntity::NUMERIC_VALUE, 5)` gets autocomplete instead of a bare string. The Entity gets a `toDto()` method and a static `fromDto()` factory — since property names match across Entity/Dto, these are just 1:1 assignments. Server-managed fields (id, `create_date`, `delete_date`) are nullable in the Dto even when the column is `NOT NULL`, so a client can omit them on create; `fromDto()` only ever reads the writable fields, silently ignoring anything else. A `delete_date` column adds a `deleteLogical()` method. Existing files are skipped unless `--overwrite` is given.

Scaffolding also writes **one class per database** to `databases/` — `{DbName}Db` (so `orm_test` → `OrmTestDb`), holding a `Database` as a public `$db`, with an accessor per table returning an entity already bound to that object's connection. Inject it and you get entities, raw SQL and transactions on one connection without touching a global:

```php
class CheckoutService {
    public function __construct(private ShopDb $shop) {}
    public function run(int $id): Order {
        return $this->shop->db->runTransaction(fn() => $this->shop->order()->get($id)->…);
    }
}
```

Accessor names are the camelCase of the table name, with no pluralisation guessing. **Any table name works**, including `transaction`, `commit`, `execute` and even `db` — the class inherits nothing, and PHP keeps properties and methods in separate symbol tables, so an accessor can never clash with `$db` or with a framework method. (Holding a `Database` rather than extending one is exactly what buys that: inheritance would put your table names and the framework's API in one namespace, in both directions.) The only rename is for two tables whose camelCased names coincide, such as `user_account` and `userAccount`.

This class is the intended home for your own cross-table queries, so it is written once and skipped afterwards: **`--overwrite` regenerates it and discards hand-written methods.** Add accessors for new tables by hand, or use `--db-class=` to generate a fresh one under another name and merge.

### Verify a scaffolded Entity against the real database

```bash
ezmvc verify TestEntity
```

Smoke-tests a scaffolded Entity+Dto pair end to end: builds a dummy Dto, then exercises `fromDto()` → `store()` → `getById()` → `find()` → `update()` → `toDto()` → `deleteLogical()` (if the table has a `delete_date` column), reporting pass/fail per step. Cleans up the test row afterward regardless of outcome. Only supports the generated Entity+Dto convention — a hand-written Entity without a matching `{Name}Dto` class isn't covered.

### Mirror the Swagger UI assets

```bash
ezmvc swagger:mirror                 # fetch the pinned release into swagger/
ezmvc swagger:mirror --version=5.18.2 # override the pinned version
```

The Swagger UI front-end bundles are the only third-party code the framework ships, and they aren't committed — `swagger/` keeps just the framework's own files plus a one-line `swagger/swagger-ui.version`. The docs page (`/swagger/`) loads the UI from a pinned jsDelivr CDN by default, so a fresh clone renders with no setup. Run `swagger:mirror` to download the referenced assets (the `swagger-ui.css`/`swagger-ui-bundle.js`/`swagger-ui-standalone-preset.js` bundles plus the two favicons) into `swagger/` and self-host them instead — the page then serves the local copies, and since they're real files (not in `deploy`'s exclude list) they ship with the next `ezmvc deploy`. The version comes from `swagger/swagger-ui.version` unless `--version=` overrides it; bump that file and re-run to upgrade.

### Scaffold new files

```bash
ezmvc new controller      <Name>         # ApiController with CRUD route stubs
ezmvc new page-controller <Name>         # HTML page Controller
ezmvc new view            <type> <Name>  # View + template; type is single, multi, leaf, or page
ezmvc new service         <Name>         # Plain Service class, no base class
ezmvc new exception       <Name>         # New Exception class, no base class
```

`<type>` picks what the generated View extends and how its template is wired:
- `single` — `SingleChildView`, holds one nested `View|string $child`, template calls `$this->renderChild()`.
- `multi` — `MultiChildView`, holds `array $children`, template calls `$this->renderChildren()`.
- `leaf` — plain `View`, no nested content at all.
- `page` — `PageView` (what used to be its own `new page-view` command), wraps a page action's output in a full HTML document; template calls `$this->renderChild()`.

Each type writes three files: the View class in `views/`, its template in `templates/`, and a co-located `templates/<slug>.css` holding an empty rule nested under the view's own CSS class (`PaddingView` → `.padding-view`, matching `View::cssClass()` at runtime). `ezmvc generate` concatenates those stylesheets into `generated/app.css`; the `page` type's generated template links it with `<?= $this->stylesheetTag() ?>`, which `PageView` cache-busts by mtime.

`ezmvc new service <Name>` creates `services/{Name}Service.php` — a plain, unrelated class (same pattern as `Dto`/`Request`/`Response`, no shared base class), meant to hold business logic that spans more than one Entity or coordinates side effects, callable from either a `PageController` or an `ApiController`. See `CLAUDE.md`'s "Services (optional)" section for the full convention.

`ezmvc new exception <Name>` creates `exceptions/{Name}Exception.php` extending `\Exception` with a basic message/code/previous constructor — for a Service to throw on an expected business-rule failure (validation, "already exists", etc.) that a Controller should catch and present gracefully rather than as a 500. Not a framework base class — each project defines its own.

### Generate request/response classes

```bash
ezmvc make:request  <Name>   # Request validation class
ezmvc make:response <Name>   # Response DTO class
```

### Set up a new project

```bash
mkdir my-project && cd my-project
ezmvc setup
```

Creates the standard directory structure, a `Config.php` template, and (if missing) a `composer.json` with a classmap autoloader covering `core/` (with every subfolder), `entities/`, `dtos/`, `requests/`, `responses/`, `services/`, `exceptions/`, `views/`, and `controllers/` — run `composer install` afterward to generate `vendor/autoload.php`. Entirely optional: manual `require_once` works fine too, and if `composer.json` already exists it's left untouched.

## Project detection

The CLI looks for a project in the current directory or a parent directory. When ezmvc is installed as a Composer dependency (`vendor/ezmvc/ezmvc/`), the CLI detects and uses that automatically.

## License

MIT
