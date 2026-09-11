---
breadcrumb:
  - Components
  - Schema
  - Migration
summary-order: ;5;3
keywords:
  - migration
  - runner
  - provider
  - tracker
  - rollback
  - plan
---

# 🔄 Migration

> 🆕 **Info**: *Since version 1.3*

> ℹ️ **Note**: Migrations are part of the [`hectororm/migration`](https://github.com/hectororm/migration) package.
> You can find it on [Packagist](https://packagist.org/packages/hectororm/migration).
> See also: [Schema introspection](schema.md) and [Plan](plan.md).

The **Migration** package orchestrates database migrations for **Hector ORM**. It builds on top of the
[Plan system](plan.md) to provide a complete workflow: define migrations as PHP classes, track which ones have been
applied, and run them forward or backward with transaction safety.

---

## Quick start

```php
use Hector\Connection\Connection;
use Hector\Migration\MigrationRunner;
use Hector\Migration\Provider\DirectoryProvider;
use Hector\Migration\Tracker\DbTracker;
use Hector\Schema\Plan\Compiler\AutoCompiler;

$connection = new Connection('mysql:host=localhost;dbname=mydb', 'root', 'secret');
$compiler = new AutoCompiler($connection);

$provider = new DirectoryProvider(__DIR__ . '/migrations');
$tracker = new DbTracker($connection);

$runner = new MigrationRunner($provider, $tracker, $compiler, $connection);

// Apply all pending migrations
$applied = $runner->up();
// Returns: ['20260101000000_CreateUsers', '20260302143000_AddPosts', ...]
```

---

## Writing migrations

A migration is a PHP class that implements `MigrationInterface`. It receives a `Plan` object and describes the DDL
operations to apply.

### Basic migration (one-way)

```php
use Hector\Migration\MigrationInterface;
use Hector\Schema\Plan\Plan;

class AddAvatarColumn implements MigrationInterface
{
    public function up(Plan $plan): void
    {
        $plan->alter('users', function ($table) {
            $table->addColumn('avatar', 'VARCHAR(255)', nullable: true);
        });
    }
}
```

### Clearing data before a schema change

> 🆕 **Info**: *Since version 1.5*

When existing data may be discarded, use `Plan::purge()` before an alteration that would otherwise conflict with
that data. For example, clear a table containing `NULL` references before making its reference column mandatory:

```php
use Hector\Migration\MigrationInterface;
use Hector\Schema\Plan\Plan;

class RequireArticleReference implements MigrationInterface
{
    public function up(Plan $plan): void
    {
        $plan->purge('articles', resetIncrement: true);

        $plan->alter('articles')
            ->modifyColumn('reference', 'VARCHAR(255)', nullable: false);
    }
}
```

Omit `resetIncrement`, or pass `false`, to avoid an explicit counter reset. On SQLite, requesting a reset requires
`sqlite_sequence` to exist. Purge compilation itself does not require schema metadata; an alteration requiring a
SQLite table rebuild still requires the runner's `Schema`, as described in [Schema introspection](plan.md#schema-introspection).

The runner logs the purge SQL and includes it in `dryRun` without executing it or recording the migration as applied.
A purge failure stops subsequent statements. Database-native transaction rules still apply: on MySQL/MariaDB,
an implicit commit can leave the table empty even if a later statement fails. The migration remains pending and
will execute the purge again on retry. A `down()` method cannot automatically restore deleted rows.

See [Purging a table](plan.md#purging-a-table) for database-specific SQL, counter reset prerequisites, foreign keys,
triggers, and transaction boundaries.

### Reversible migration

Implement `ReversibleMigrationInterface` to support rollback:

```php
use Hector\Migration\ReversibleMigrationInterface;
use Hector\Schema\Index;
use Hector\Schema\Plan\Plan;

class CreateUsersTable implements ReversibleMigrationInterface
{
    public function up(Plan $plan): void
    {
        $plan->create('users', function ($table) {
            $table->addColumn('id', 'INT', autoIncrement: true);
            $table->addColumn('email', 'VARCHAR(255)');
            $table->addColumn('name', 'VARCHAR(100)', nullable: true);
            $table->addIndex('PRIMARY', ['id'], Index::PRIMARY);
            $table->addIndex('idx_email', ['email'], Index::UNIQUE);
        });
    }

    public function down(Plan $plan): void
    {
        $plan->drop('users');
    }
}
```

> ⚠️ **Warning**: The runner will throw a `MigrationException` if you try to revert a migration that does not
> implement `ReversibleMigrationInterface`.

### Using raw SQL

For SQL features not covered by the Plan API, use `$plan->raw()` to inject raw SQL statements.
See the [Plan documentation](plan.md#-raw-sql-statements) for details.

```php
class AddFulltextSearch implements MigrationInterface
{
    public function up(Plan $plan): void
    {
        $plan->alter('articles', function ($table) {
            $table->addColumn('title', 'VARCHAR(255)');
            $table->addColumn('body', 'TEXT');
        });

        // Fulltext index — not supported by the Plan API
        $plan->raw('CREATE FULLTEXT INDEX ft_search ON articles (title, body)');
    }
}
```

### The `#[Migration]` Attribute

You can annotate migration classes with the `#[Migration]` attribute to add a human-readable description:

```php
use Hector\Migration\Attributes\Migration;
use Hector\Migration\ReversibleMigrationInterface;
use Hector\Schema\Index;
use Hector\Schema\Plan\Plan;

#[Migration(description: 'Create the users table')]
class CreateUsersTable implements ReversibleMigrationInterface
{
    public function up(Plan $plan): void
    {
        $plan->create('users', function ($table) {
            $table->addColumn('id', 'INT', autoIncrement: true);
            $table->addColumn('email', 'VARCHAR(255)');
            $table->addColumn('name', 'VARCHAR(100)', nullable: true);
            $table->addIndex('PRIMARY', ['id'], Index::PRIMARY);
            $table->addIndex('idx_email', ['email'], Index::UNIQUE);
        });
    }

    public function down(Plan $plan): void
    {
        $plan->drop('users');
    }
}
```

| Parameter      | Type      | Default | Description                                      |
|----------------|-----------|---------|--------------------------------------------------|
| `$description` | `?string` | `null`  | Human-readable description, used in log messages |

The attribute is **optional**. Migrations without it work normally. When present, the description is included in
log messages: `Applying migration "CreateUsers" (Create the users table)...`

---

## Providers

Providers supply the list of migrations. All providers implement `MigrationProviderInterface` (`IteratorAggregate`,
`Countable`, `getArrayCopy()`).

### ArrayProvider

Register migrations manually:

```php
use Hector\Migration\Provider\ArrayProvider;

$provider = new ArrayProvider([
    'create_users' => new CreateUsersTable(),
    'add_posts' => new AddPostsTable(),
]);

// Or add incrementally (migration first, optional ID second)
$provider = new ArrayProvider();
$provider->add(new CreateUsersTable(), 'create_users');
$provider->add(new AddPostsTable());  // ID defaults to FQCN
```

Duplicate migration identifiers throw a `MigrationException`.

### DirectoryProvider

Scan a directory for migration files. Each file must return a `MigrationInterface` instance or a FQCN string.
The migration identifier is the **relative file path without extension**, with directory separators normalized
to `/`:

```php
use Hector\Migration\Provider\DirectoryProvider;

$provider = new DirectoryProvider(
    directory: __DIR__ . '/migrations',
    pattern: '*.php',  // glob pattern (default)
    depth: 0,          // 0 = flat (default), -1 = unlimited, N = max levels
);
```

| Parameter    | Type                  | Default   | Description                        |
|--------------|-----------------------|-----------|------------------------------------|
| `$directory` | `string`              | *(req.)*  | Path to the migrations directory   |
| `$pattern`   | `string`              | `'*.php'` | Glob pattern for matching files    |
| `$depth`     | `int`                 | `0`       | Recursive scanning depth           |
| `$container` | `?ContainerInterface` | `null`    | PSR-11 container for instantiation |

The `$depth` parameter controls recursive directory scanning:

- `0` — scan only the top-level directory (default)
- `-1` — scan all subdirectories recursively (unlimited)
- `N` — scan up to N levels of subdirectories

Migration files must return either an instance or a fully qualified class name:

```php
// migrations/20260101000000_CreateUsers.php

// Option 1: return an instance (supports anonymous classes)
return new class implements ReversibleMigrationInterface {
    public function up(Plan $plan): void { /* ... */ }
    public function down(Plan $plan): void { /* ... */ }
};

// Option 2: return a FQCN string
return CreateUsersTable::class;
```

Files that don't match the glob pattern are silently ignored. Files that match but return an invalid value
(neither a `MigrationInterface` instance nor a valid FQCN string) throw a `MigrationException`. Duplicate
identifiers also throw a `MigrationException`.

#### Dependency injection via container

When a file returns a class name, the `DirectoryProvider` can resolve it through a PSR-11 container:

```php
use Hector\Migration\Provider\DirectoryProvider;

$provider = new DirectoryProvider(
    directory: __DIR__ . '/migrations',
    container: $container, // ?ContainerInterface (PSR-11)
);
```

If the container has the class, it is resolved from the container (enabling constructor injection). Otherwise, the
class is instantiated directly with `new`.

> ℹ️ **Note**: The `psr/container` package is suggested, not required.

### Psr4Provider

Load migration classes using PSR-4 autoloading conventions. Migration classes are discovered by scanning a directory
and mapping file paths to fully qualified class names:

```php
use Hector\Migration\Provider\Psr4Provider;

$provider = new Psr4Provider(
    namespace: 'App\\Migration',
    directory: __DIR__ . '/src/Migration',
    depth: -1,  // unlimited (default for PSR-4)
);
```

| Parameter    | Type                  | Default   | Description                          |
|--------------|-----------------------|-----------|--------------------------------------|
| `$namespace` | `string`              | *(req.)*  | Base PSR-4 namespace                 |
| `$directory` | `string`              | *(req.)*  | Directory mapped to the namespace    |
| `$pattern`   | `string`              | `'*.php'` | Glob pattern for matching files      |
| `$depth`     | `int`                 | `-1`      | Recursive scanning depth (unlimited) |
| `$container` | `?ContainerInterface` | `null`    | PSR-11 container for instantiation   |

**How it works:**

1. Scans `$directory` for files matching `$pattern` (respecting `$depth`)
2. Deduces the FQCN using PSR-4 convention: `$namespace` + relative path
3. Skips files whose class does not implement `MigrationInterface`
4. Uses the **FQCN** as the migration identifier (e.g. `App\Migration\CreateUsers`)
5. Migrations are sorted alphabetically by FQCN

#### Example directory structure

```
src/Migration/
├── CreateUsers.php             → App\Migration\CreateUsers
├── AddPosts.php                → App\Migration\AddPosts
└── V2/
    └── AddComments.php         → App\Migration\V2\AddComments
```

> ℹ️ **Note**: Unlike `DirectoryProvider`, the `Psr4Provider` relies on Composer autoloading — classes are not
> `require`d manually. Make sure the namespace is registered in your `composer.json` `autoload` section.

---

## Trackers

Trackers record which migrations have been applied. All trackers implement `MigrationTrackerInterface`
(`IteratorAggregate`, `Countable`, `getArrayCopy()`).

| Method                                                       | Description                           |
|--------------------------------------------------------------|---------------------------------------|
| `getArrayCopy(): string[]`                                   | Get all applied migration identifiers |
| `isApplied(string $migrationId): bool`                       | Check if a migration has been applied |
| `markApplied(string $migrationId, ?float $durationMs): void` | Mark a migration as applied           |
| `markReverted(string $migrationId): void`                    | Mark a migration as reverted          |

The `$durationMs` parameter on `markApplied()` records the execution duration in milliseconds. It is passed
automatically by the runner.

### DbTracker

Stores applied migrations in a database table (created automatically). Uses the `QueryBuilder` internally for
SELECT, INSERT and DELETE operations:

```php
use Hector\Migration\Tracker\DbTracker;

$tracker = new DbTracker(
    connection: $connection,
    tableName: 'hector_migrations', // default
);
```

The table has four columns:

- `id` — auto-increment primary key (records the exact application order)
- `migration_id` — VARCHAR(255), `NOT NULL`, unique
- `applied_at` — DATETIME, `NOT NULL`
- `duration_ms` — FLOAT, nullable (execution time in milliseconds)

The table is created with `CREATE TABLE IF NOT EXISTS` on first use. The auto-increment syntax is
driver-specific:

| Driver        | `id` column definition                                     |
|---------------|------------------------------------------------------------|
| MySQL/MariaDB | `id INT AUTO_INCREMENT PRIMARY KEY`                        |
| SQLite        | `id INTEGER PRIMARY KEY AUTOINCREMENT`                     |
| PostgreSQL    | `id INTEGER GENERATED BY DEFAULT AS IDENTITY PRIMARY KEY`  |

> ℹ️ **Note**: `DbTracker` requires the `hectororm/query` package (`composer require hectororm/query`).

#### Upgrading the tracking table

> 🆕 **Info**: *Since version 1.4*

The tracking table schema changed: `migration_id` is no longer the primary key. A dedicated
auto-increment `id` column was introduced to record the exact application order (`applied_at` only
has second precision and cannot disambiguate migrations applied within the same second), and
`migration_id` became a `NOT NULL UNIQUE` column.

A fresh install needs nothing — the table is created with the new schema automatically. An **existing**
tracking table created by an earlier version is *not* migrated automatically: `DbTracker` raises a clear
`MigrationException` when it reads an outdated table, asking you to upgrade it. Run the matching
statements once, against your tracking table (default name `hector_migrations`):

> ⚠️ **Warning**: Back up your tracking table before running these statements, and adapt the table name
> if you customised `tableName`.

**MySQL / MariaDB**

```sql
ALTER TABLE hector_migrations DROP PRIMARY KEY;
ALTER TABLE hector_migrations ADD UNIQUE (migration_id);
ALTER TABLE hector_migrations ADD COLUMN id INT AUTO_INCREMENT PRIMARY KEY FIRST;
```

**PostgreSQL**

```sql
ALTER TABLE hector_migrations DROP CONSTRAINT hector_migrations_pkey;
ALTER TABLE hector_migrations ADD UNIQUE (migration_id);
ALTER TABLE hector_migrations ADD COLUMN id INTEGER GENERATED BY DEFAULT AS IDENTITY PRIMARY KEY;
```

**SQLite** (no `DROP PRIMARY KEY` / `ADD ... AUTOINCREMENT`, so rebuild the table):

```sql
CREATE TABLE hector_migrations_new (
    id INTEGER PRIMARY KEY AUTOINCREMENT,
    migration_id VARCHAR(255) NOT NULL UNIQUE,
    applied_at DATETIME NOT NULL,
    duration_ms FLOAT NULL
);
INSERT INTO hector_migrations_new (migration_id, applied_at, duration_ms)
    SELECT migration_id, applied_at, duration_ms FROM hector_migrations ORDER BY applied_at;
DROP TABLE hector_migrations;
ALTER TABLE hector_migrations_new RENAME TO hector_migrations;
```

> 💡 **Tip**: Existing rows get an `id` reflecting their order (ascending `applied_at` for the SQLite
> rebuild), preserving the recorded application order on replay/rollback.

### FileTracker

Stores applied migrations in a local JSON file:

```php
use Hector\Migration\Tracker\FileTracker;

$tracker = new FileTracker(
    filePath: __DIR__ . '/.hector.migrations.json',
);
```

| Parameter   | Type     | Default  | Description                                       |
|-------------|----------|----------|---------------------------------------------------|
| `$filePath` | `string` | *(req.)* | Path to the JSON tracking file                    |
| `$lock`     | `bool`   | `true`   | Use exclusive lock when writing (disable for NFS) |

The file is created on first write and contains a JSON object indexed by migration identifier:

```json
{
    "20260101000000_CreateUsers": {
        "applied_at": "2026-01-15T10:30:00+00:00",
        "duration_ms": 42.57
    }
}
```

### FlysystemTracker

Same as `FileTracker` but uses [League Flysystem](https://flysystem.thephpleague.com/) for storage — useful for
remote or cloud-based tracking:

```php
use Hector\Migration\Tracker\FlysystemTracker;
use League\Flysystem\Filesystem;
use League\Flysystem\Local\LocalFilesystemAdapter;

$filesystem = new Filesystem(new LocalFilesystemAdapter('/path'));
$tracker = new FlysystemTracker(
    filesystem: $filesystem,
    filePath: '.hector.migrations.json', // default
);
```

| Parameter     | Type                 | Default                     | Description                   |
|---------------|----------------------|-----------------------------|-------------------------------|
| `$filesystem` | `FilesystemOperator` | *(req.)*                    | Flysystem filesystem instance |
| `$filePath`   | `string`             | `'.hector.migrations.json'` | Path within the filesystem    |

> ℹ️ **Note**: The `league/flysystem` package is suggested, not required.

### ChainTracker

Aggregate multiple trackers with a configurable strategy:

```php
use Hector\Migration\Tracker\ChainTracker;
use Hector\Migration\Tracker\ChainStrategy;

$tracker = new ChainTracker(
    [$databaseTracker, $fileTracker],
    ChainStrategy::ANY,
);
```

**Write operations** (`markApplied`, `markReverted`) are always propagated to **all** trackers.

**Read operations** (`isApplied`, `getArrayCopy`) depend on the strategy:

| Strategy               | `isApplied()` returns `true` if...       | `getArrayCopy()` returns...   |
|------------------------|------------------------------------------|-------------------------------|
| `ChainStrategy::ANY`   | **any** tracker reports it applied       | Union of all trackers         |
| `ChainStrategy::ALL`   | **all** trackers report it applied       | Intersection of all trackers  |
| `ChainStrategy::FIRST` | the **first** tracker reports it applied | Only the first tracker's data |

---

## MigrationRunner

The `MigrationRunner` orchestrates the full migration lifecycle:

```php
use Hector\Migration\MigrationRunner;

$runner = new MigrationRunner(
    provider: $provider,
    tracker: $tracker,
    compiler: $compiler,
    connection: $connection,
    schema: $schema,            // optional — enables schema introspection in Plan compilers
    logger: $logger,            // optional — PSR-3 LoggerInterface
    eventDispatcher: $dispatcher, // optional — PSR-14 EventDispatcherInterface
);
```

### Constructor parameters

| Parameter          | Type                         | Default      | Description                                    |
|--------------------|------------------------------|--------------|------------------------------------------------|
| `$provider`        | `MigrationProviderInterface` | *(required)* | Migration source                               |
| `$tracker`         | `MigrationTrackerInterface`  | *(required)* | Tracks applied migrations                      |
| `$compiler`        | `CompilerInterface`          | *(required)* | Compiles Plan operations into SQL              |
| `$connection`      | `Connection`                 | *(required)* | Database connection for executing statements   |
| `$schema`          | `?Schema`                    | `null`       | Enables schema introspection in Plan compilers |
| `$logger`          | `?LoggerInterface`           | `null`       | PSR-3 logger for migration lifecycle           |
| `$eventDispatcher` | `?EventDispatcherInterface`  | `null`       | PSR-14 event dispatcher                        |

### Runner methods

| Method                                                   | Description                           |
|----------------------------------------------------------|---------------------------------------|
| `up(?int $steps = null, bool $dryRun = false): string[]` | Apply pending migrations (null = all) |
| `down(int $steps = 1, bool $dryRun = false): string[]`   | Revert applied migrations             |
| `getPending(): array<string, MigrationInterface>`        | Get migrations not yet applied        |
| `getApplied(): array<string, MigrationInterface>`        | Get migrations already applied        |
| `getStatus(): array<string, bool>`                       | Get status of all migrations          |

### Running migrations

```php
// Apply ALL pending migrations
$applied = $runner->up();

// Apply at most 3 pending migrations
$applied = $runner->up(steps: 3);

// Revert the last applied migration
$reverted = $runner->down();

// Revert the last 5 applied migrations
$reverted = $runner->down(steps: 5);
```

Both `up()` and `down()` return an array of migration identifiers that were applied or reverted.

### Querying status

```php
// Migrations not yet applied
$pending = $runner->getPending();
// Returns: ['20260302143000_AddPosts' => AddPostsTable, ...]

// Migrations already applied
$applied = $runner->getApplied();
// Returns: ['20260101000000_CreateUsers' => CreateUsersTable, ...]

// Full status map
$status = $runner->getStatus();
// Returns: ['20260101000000_CreateUsers' => true, '20260302143000_AddPosts' => false, ...]
```

### Transaction safety

Each migration is executed within a database transaction:

1. `beginTransaction()`
2. Execute all Plan statements
3. `commit()`
4. Mark applied/reverted in the tracker

If any statement fails, the transaction is rolled back and a `MigrationException` is thrown with the original
error as `previous`. The tracker is **not** updated on failure.

> ℹ️ **Note**: Migrations that produce an empty Plan (no DDL operations) are still tracked — they are simply marked
> as applied/reverted without opening a transaction.

### Concurrency

Tracking is **atomic per migration**: the `DbTracker` records applied migrations using the primary key of its
tracking table, so two runners cannot apply or record the *same* migration twice (a concurrent duplicate is detected
and treated as already applied).

The runner does **not** provide a global run lock, however. It does not prevent two processes from running
*different* pending migrations at the same time, which would break the guaranteed migration order. Ensuring that a
**single migration run executes at a time** is the responsibility of the calling application or orchestrator, for
example:

- a single, serialized deployment/CI-CD step that runs migrations;
- an application-level or infrastructure lock (e.g. a distributed lock, a job queue with concurrency 1);
- a database-level advisory lock acquired by your application around the run (e.g. MySQL `GET_LOCK()`,
  PostgreSQL `pg_advisory_lock()`).

> ⚠️ **Note**: Running multiple migration processes concurrently without such an external lock is not supported and
> may apply different migrations out of order.

### Dry-run mode

Both `up()` and `down()` accept a `dryRun` parameter. In dry-run mode, the runner goes through the full migration
flow — building the plan, compiling SQL, dispatching events — but **does not execute** the statements, open
transactions, or update the tracker:

```php
// Preview which SQL would be generated
$applied = $runner->up(dryRun: true);

// Also works with down
$reverted = $runner->down(steps: 2, dryRun: true);
```

In dry-run mode:

- SQL statements are logged with a `[DRY-RUN]` prefix
- Events are dispatched normally (listeners can inspect the plan)
- No database changes are made
- No tracker updates are recorded
- The return value is the list of migration identifiers that **would** be applied/reverted

---

## Logging (PSR-3)

The runner accepts an optional PSR-3 `LoggerInterface` to log the migration lifecycle:

| Level   | When                                | Message example                                                |
|---------|-------------------------------------|----------------------------------------------------------------|
| `info`  | Before executing a migration        | `Applying migration "CreateUsers" (Create the users table)...` |
| `info`  | After a successful migration        | `Migration "CreateUsers" applied successfully (42.57ms)`       |
| `debug` | For each SQL statement              | `[CreateUsers] CREATE TABLE ...`                               |
| `debug` | Migration skipped by event listener | `Migration "..." skipped by event listener`                    |
| `error` | On migration failure                | `Migration "..." failed during up: ...`                        |

When a migration class has a `#[Migration(description: '...')]` attribute, the description is included in log
messages. Without the attribute, only the migration identifier is shown.

In dry-run mode, all log messages are prefixed with `[DRY-RUN]`.

```php
use Psr\Log\LoggerInterface;

$runner = new MigrationRunner(
    provider: $provider,
    tracker: $tracker,
    compiler: $compiler,
    connection: $connection,
    logger: $logger, // any PSR-3 logger
);
```

---

## Events (PSR-14)

The runner dispatches PSR-14 events around each migration. If no event dispatcher is provided, events are
simply not dispatched (zero overhead thanks to the null-safe `?->` operator).

### Event classes

All event classes live in `Hector\Migration\Event\`.

| Event                  | When dispatched                 | Stoppable? | Extra data                      |
|------------------------|---------------------------------|------------|---------------------------------|
| `MigrationBeforeEvent` | Before executing a migration    | **Yes**    | migration ID, direction         |
| `MigrationAfterEvent`  | After a successful migration    | No         | + `durationMs` (execution time) |
| `MigrationFailedEvent` | After a failure (post-rollback) | No         | + exception + `durationMs`      |

All events carry: `getMigrationId()`, `getMigration()`, `getDirection()`, `getTime()`, `isDryRun()`.

The `direction` is a `string` value from the `Direction` class constants: `Direction::UP` or `Direction::DOWN`.

### Stoppable before events

If a listener stops a `MigrationBeforeEvent`, the migration is **skipped** entirely — not executed and not tracked:

```php
use Hector\Migration\Event\MigrationBeforeEvent;

// Example: skip a specific migration
$listener = function (MigrationBeforeEvent $event): void {
    if ($event->getMigrationId() === '20260101000000_CreateUsers') {
        $event->stopPropagation();
    }
};
```

This enables use cases like: **conditional filtering** or **interactive confirmation**.

### After and failed events

The `MigrationAfterEvent` carries the execution duration:

```php
use Hector\Migration\Event\MigrationAfterEvent;

$listener = function (MigrationAfterEvent $event): void {
    $durationMs = $event->getDurationMs(); // float|null
    // Audit, metrics, etc.
};
```

The `MigrationFailedEvent` carries the original exception and the duration until failure:

```php
use Hector\Migration\Event\MigrationFailedEvent;

$listener = function (MigrationFailedEvent $event): void {
    $exception = $event->getException();  // Throwable
    $durationMs = $event->getDurationMs(); // float|null
};
```

---

## Complete example

```php
use Hector\Connection\Connection;
use Hector\Migration\MigrationRunner;
use Hector\Migration\Provider\DirectoryProvider;
use Hector\Migration\Tracker\DbTracker;
use Hector\Schema\Generator\MySQL as MySQLGenerator;
use Hector\Schema\Plan\Compiler\AutoCompiler;

// Setup
$connection = new Connection('mysql:host=localhost;dbname=mydb', 'root', 'secret');
$compiler = new AutoCompiler($connection);
$generator = new MySQLGenerator($connection);
$schema = $generator->generateSchema('mydb');

// Provider: scan migrations directory
$provider = new DirectoryProvider(
    directory: __DIR__ . '/migrations',
);

// Tracker: store status in the database
$tracker = new DbTracker($connection);

// Runner
$runner = new MigrationRunner($provider, $tracker, $compiler, $connection, $schema);

// Check what needs to be done
echo "Pending: " . count($runner->getPending()) . "\n";
echo "Applied: " . count($runner->getApplied()) . "\n";

foreach ($runner->getStatus() as $id => $applied) {
    echo sprintf("  [%s] %s\n", $applied ? 'x' : ' ', $id);
}

// Preview with dry-run
$runner->up(dryRun: true);

// Apply all pending migrations
$applied = $runner->up();
echo "Applied " . count($applied) . " migration(s)\n";

// Revert the last one
$reverted = $runner->down();
echo "Reverted: " . implode(', ', $reverted) . "\n";
```

### Example migration file

```php
// migrations/20260101000000_CreateUsers.php

use Hector\Migration\ReversibleMigrationInterface;
use Hector\Schema\Index;
use Hector\Schema\Plan\Plan;
use Hector\Schema\Plan\Raw;

return new class implements ReversibleMigrationInterface {
    public function up(Plan $plan): void
    {
        $plan->create('users', function ($table) {
            $table->addColumn('id', 'INT', autoIncrement: true);
            $table->addColumn('email', 'VARCHAR(255)');
            $table->addColumn('name', 'VARCHAR(100)', nullable: true);
            $table->addColumn('created_at', 'DATETIME', default: new Raw('CURRENT_TIMESTAMP()'));
            $table->addIndex('PRIMARY', ['id'], Index::PRIMARY);
            $table->addIndex('idx_email', ['email'], Index::UNIQUE);
        });
    }

    public function down(Plan $plan): void
    {
        $plan->drop('users');
    }
};
```

> 💡 **Tip**: Anonymous classes (`new class implements ...`) work great for migration files — they keep things
> self-contained and avoid polluting the autoloader.

---

## Installation

```bash
composer require hectororm/migration
```

Optional dependencies:

```bash
# For DbTracker (database-backed tracking)
composer require hectororm/query

# For FlysystemTracker
composer require league/flysystem

# For container-based migration instantiation
composer require psr/container

# For migration lifecycle logging
composer require psr/log

# For migration lifecycle events
composer require psr/event-dispatcher
```

---

## See also

- [Schema](schema.md) — database introspection and metadata
- [Plan](plan.md) — DDL operations builder and SQL compiler
