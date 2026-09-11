---
breadcrumb:
  - Components
  - Schema
  - Plan
summary-order: ;5;2
keywords:
  - plan
  - ddl
  - migration
  - create-table
  - alter-table
  - compiler
---

# 📐 Plan

> 🆕 **Info**: *Since version 1.3*

> ℹ️ **Note**: Plans are part of the [`hectororm/schema`](https://github.com/hectororm/schema) package.
> You can find it on [Packagist](https://packagist.org/packages/hectororm/schema).
> See also: [Schema introspection](schema.md) and [Migration](migration.md).

The **Plan** system is the schema migration module of **Hector ORM**. It provides a fluent, object-oriented API to
describe
DDL operations (CREATE TABLE, ALTER TABLE, DROP TABLE, ...) and compile them into executable SQL statements for
MySQL/MariaDB and SQLite.

A `Plan` does **not** execute SQL — it only produces statements. You decide when and how to execute them through your
own `Connection`.

---

## Quick start

```php
use Hector\Connection\Connection;
use Hector\Schema\Index;
use Hector\Schema\Plan\Plan;
use Hector\Schema\Plan\Compiler\AutoCompiler;

$connection = new Connection('mysql:host=localhost;dbname=mydb', 'root', 'secret');
$compiler = new AutoCompiler($connection);

$plan = new Plan();

$plan->create('users', function ($table) {
    $table->addColumn('id', 'INT', autoIncrement: true);
    $table->addColumn('email', 'VARCHAR(255)');
    $table->addColumn('name', 'VARCHAR(100)', nullable: true);
    $table->addIndex('PRIMARY', ['id'], Index::PRIMARY);
    $table->addIndex('idx_email', ['email'], Index::UNIQUE);
});

foreach ($plan->getStatements($compiler) as $sql) {
    $connection->execute($sql);
}
```

> 💡 **Tip**: `getStatements()` returns an `iterable` (generator). Statements are produced lazily — you can iterate
> and execute them one by one without loading everything into memory.

---

## Building a plan

A `Plan` is an ordered collection of entries. Each entry implements `OperationInterface`. Table operations (
`CreateTable`, `AlterTable`) group sub-operations (columns, indexes, foreign keys, triggers) for a single table; view
operations (`CreateView`, `AlterView`, `DropView`), trigger operations (`CreateTrigger`, `DropTrigger`), atomic table
operations (`DropTable`, `MigrateData`), foreign key check operations (`DisableForeignKeyChecks`,
`EnableForeignKeyChecks`) and raw statements (`RawStatement`) are standalone entries.

`Plan` implements `Countable` and `IteratorAggregate`.

> When a callback is provided to `create()` or `alter()`, the method returns `$this` (the `Plan`) for chaining.
> Without a callback, it returns the `CreateTable` or `AlterTable` instance for direct manipulation.

### Creating a table

Use `create()` with a callback to define columns, indexes and foreign keys in one go:

```php
use Hector\Schema\ForeignKey;
use Hector\Schema\Index;
use Hector\Schema\Plan\Raw;

$plan->create('posts', function ($table) {
    $table->addColumn('id', 'INT', autoIncrement: true);
    $table->addColumn('title', 'VARCHAR(255)');
    $table->addColumn('body', 'TEXT', nullable: true);
    $table->addColumn('user_id', 'INT');
    $table->addColumn('created_at', 'DATETIME', default: new Raw('CURRENT_TIMESTAMP()'));

    $table->addIndex('PRIMARY', ['id'], Index::PRIMARY);
    $table->addIndex('idx_user', ['user_id']);

    $table->addForeignKey(
        name: 'fk_posts_user',
        columns: ['user_id'],
        referencedTable: 'users',
        referencedColumns: ['id'],
        onDelete: ForeignKey::RULE_CASCADE,
    );
});
```

Without a callback, `create()` returns the `CreateTable` instance directly for chaining:

```php
$table = $plan->create('tags');
$table->addColumn('id', 'INT', autoIncrement: true);
$table->addColumn('label', 'VARCHAR(50)');
$table->addIndex('PRIMARY', ['id'], Index::PRIMARY);
```

#### IF NOT EXISTS

```php
$plan->create('users', callback: function ($table) {
    $table->addColumn('id', 'INT', autoIncrement: true);
    // ...
}, ifNotExists: true);
```

### Altering a table

Use `alter()` to add, modify, drop or rename columns, indexes and foreign keys on an existing table:

```php
$plan->alter('users', function ($table) {
    $table->addColumn('avatar', 'VARCHAR(255)', nullable: true);
    $table->modifyColumn('name', 'VARCHAR(200)');
    $table->renameColumn('email', 'email_address');
    $table->dropColumn('legacy_field');

    $table->addIndex('idx_name', ['name']);
    $table->dropIndex('idx_old');

    $table->addForeignKey(
        name: 'fk_users_team',
        columns: ['team_id'],
        referencedTable: 'teams',
        referencedColumns: ['id'],
    );
    $table->dropForeignKey('fk_users_old');
});
```

#### Column placement (MySQL only)

`addColumn()` and `modifyColumn()` accept `after` and `first` parameters for column positioning:

```php
$plan->alter('users', function ($table) {
    $table->addColumn('middle_name', 'VARCHAR(100)', nullable: true, after: 'first_name');
    $table->addColumn('row_id', 'INT', first: true);
});
```

#### Column defaults

> 🆕 **Info**: *Since version 1.4*

A plain `default` value is emitted as a **quoted string literal** (e.g. `DEFAULT 'active'`). To emit a value
**verbatim** — for SQL functions or keywords such as `CURRENT_TIMESTAMP()`, `NOW()` or `UUID()` — wrap it in
`Hector\Schema\Plan\Raw`:

```php
use Hector\Schema\Plan\Raw;

$plan->create('posts', function ($table) {
    // Literal string default: DEFAULT 'draft'
    $table->addColumn('status', 'VARCHAR(20)', default: 'draft');

    // Raw SQL expression: DEFAULT CURRENT_TIMESTAMP()
    $table->addColumn('created_at', 'DATETIME', default: new Raw('CURRENT_TIMESTAMP()'));
});
```

The `hasDefault` parameter of `addColumn()` / `modifyColumn()` is now `?bool` and defaults to `null` (auto):
the `DEFAULT` clause is enabled automatically when a `Raw` expression or any non-`null` value is provided.
Pass an explicit boolean to force the behaviour — in particular, use `hasDefault: true` together with a `null`
value to emit `DEFAULT NULL`.

> ℹ️ **Note**: `Raw` is meant for actual SQL expressions only; keep using a plain string for literal defaults.
> It is not intended to express a `NULL` default (use a `null` value with `hasDefault: true` instead).

#### Automatic update timestamps (MySQL / MariaDB)

> 🆕 **Info**: *Unreleased*

`addColumn()` and `modifyColumn()` accept `useCurrentOnUpdate: bool = false`:

```php
use Hector\Schema\Plan\Raw;

$plan->create('events', function ($table) {
    $table->addColumn('id', 'INTEGER', autoIncrement: true);
    $table->addIndex('PRIMARY', ['id'], \Hector\Schema\Index::PRIMARY);
    $table->addColumn(
        name: 'updated_at',
        type: 'TIMESTAMP',
        default: new Raw('CURRENT_TIMESTAMP'),
        useCurrentOnUpdate: true,
    );
});
```

On **MySQL / MariaDB**, this emits `DEFAULT CURRENT_TIMESTAMP ON UPDATE CURRENT_TIMESTAMP`.
The database updates the timestamp when another column actually changes, including changes made outside Hector.
An explicit assignment to the timestamp takes precedence. The option does not implicitly add a `DEFAULT` clause.

Only `TIMESTAMP` and `DATETIME`, with optional fractional seconds precision from **0 to 6**, are supported.
For example, `DATETIME(6)` emits `ON UPDATE CURRENT_TIMESTAMP(6)`; use a matching
`default: new Raw('CURRENT_TIMESTAMP(6)')` if a current-time default is also needed on MySQL/MariaDB.
An incompatible type or `autoIncrement: true` together with this option raises a `PlanException` on MySQL/MariaDB.
This is a current-time option, not a general-purpose update expression or a foreign-key `ON UPDATE` rule.

On **SQLite**, the option is **ignored without an exception or an automatically created trigger**.
The first example can therefore be used with MySQL in production and SQLite in tests: the insertion default works
on both, but SQLite does not automatically update the timestamp. Test the database-side update behaviour against
MySQL/MariaDB. Other SQL expressions still need to be compatible with the target engine; for example,
`CURRENT_TIMESTAMP(6)` is not a SQLite expression.

When modifying a column, repeat `useCurrentOnUpdate: true` to retain the behaviour. Omitting the option or setting
it to `false` removes the clause on MySQL/MariaDB:

```php
$plan->alter('events')->modifyColumn(
    'updated_at',
    'TIMESTAMP',
    default: new Raw('CURRENT_TIMESTAMP'),
    useCurrentOnUpdate: false,
);
```

#### Rename column compatibility

`renameColumn()` uses the modern `RENAME COLUMN` syntax by default. When the compiler detects an older server
(MySQL < 8.0, MariaDB < 10.5.2) via `AutoCompiler`, it automatically falls back to `CHANGE COLUMN` using the
current column definition from the schema.

> ℹ️ **Note**: The `CHANGE COLUMN` fallback requires a `Schema` to introspect the current column definition.
> A `PlanException` is thrown if no schema is provided and the server does not support `RENAME COLUMN`.

### Dropping a table

```php
$plan->drop('old_table');
$plan->drop('maybe_exists', ifExists: true);
```

### Renaming a table

```php
$plan->rename('old_name', 'new_name');
```

You can also rename a table as part of an `alter()` call, combined with other operations:

```php
$plan->alter('users', function ($table) {
    $table->addColumn('avatar', 'VARCHAR(255)', nullable: true);
    $table->renameTable('members');
});
```

> ℹ️ **Note**: The rename is always emitted as a separate `ALTER TABLE ... RENAME TO ...` statement **after** the
> other operations, regardless of the order in which you call `renameTable()`.

### Migrating data

Copy data from one table to another, optionally mapping columns:

```php
// Copy all columns (SELECT *)
$plan->migrate('source_table', 'target_table');

// Map specific columns
$plan->migrate('source_table', 'target_table', [
    'old_name' => 'new_name',
    'old_email' => 'email_address',
]);
```

> ℹ️ **Note**: `migrate()` generates an `INSERT INTO ... SELECT ...` statement. Both tables must exist at the time
> of execution.

### Views

Plans can also manage views with `createView()`, `dropView()` and `alterView()`:

```php
// Create a view
$plan->createView('active_users', 'SELECT * FROM users WHERE active = 1');

// Create or replace
$plan->createView('active_users', 'SELECT * FROM users WHERE active = 1', orReplace: true);

// With MySQL-specific algorithm
$plan->createView('active_users', 'SELECT * FROM users WHERE active = 1', algorithm: 'MERGE');

// Alter a view (replace its SELECT statement)
$plan->alterView('active_users', 'SELECT * FROM users WHERE active = 1 AND verified = 1');

// Drop a view
$plan->dropView('old_view', ifExists: true);
```

> ℹ️ **Note**: On SQLite, `OR REPLACE` is emulated with `DROP VIEW IF EXISTS` + `CREATE VIEW`, and `ALTER VIEW` is
> emulated with `DROP VIEW IF EXISTS` + `CREATE VIEW`. The `algorithm` parameter is silently ignored.

### Triggers

Plans can create and drop triggers with `createTrigger()` and `dropTrigger()`:

```php
use Hector\Schema\Plan\CreateTrigger;

// Create a trigger (standalone)
$plan->createTrigger(
    name: 'trg_users_audit',
    table: 'users',
    timing: CreateTrigger::AFTER,
    event: CreateTrigger::INSERT,
    body: 'INSERT INTO audit_log (table_name, action) VALUES (\'users\', \'insert\')',
);

// Drop a trigger
$plan->dropTrigger('trg_users_audit', 'users');
```

Triggers can also be created inside `create()` or `alter()` calls:

```php
$plan->create('users', function ($table) {
    $table->addColumn('id', 'INT', autoIncrement: true);
    $table->addColumn('name', 'VARCHAR(255)');
    $table->addIndex('PRIMARY', ['id'], Index::PRIMARY);

    $table->createTrigger(
        'trg_users_audit',
        CreateTrigger::AFTER,
        CreateTrigger::INSERT,
        'INSERT INTO audit_log (table_name, action) VALUES (\'users\', \'insert\')',
    );
});

$plan->alter('users', function ($table) {
    $table->dropTrigger('trg_users_audit');
});
```

**Available constants on `CreateTrigger`:**

| Timing constants            | Event constants         |
|-----------------------------|-------------------------|
| `CreateTrigger::BEFORE`     | `CreateTrigger::INSERT` |
| `CreateTrigger::AFTER`      | `CreateTrigger::UPDATE` |
| `CreateTrigger::INSTEAD_OF` | `CreateTrigger::DELETE` |

**WHEN condition (SQLite only):**

SQLite supports an optional `WHEN` condition on triggers. On MySQL, this parameter is ignored:

```php
$plan->createTrigger(
    name: 'trg_status_change',
    table: 'users',
    timing: CreateTrigger::BEFORE,
    event: CreateTrigger::UPDATE,
    body: 'INSERT INTO audit_log (action) VALUES (\'status_changed\')',
    when: 'NEW.status != OLD.status',
);
```

> ℹ️ **Note**: On MySQL, the trigger body is emitted directly after `FOR EACH ROW`. On SQLite, it is wrapped in
> `BEGIN ... END` and `IF NOT EXISTS` is added to the `CREATE TRIGGER` statement.

**Trigger ordering:**

- `CREATE TRIGGER` implements `PostOperationInterface` — it is emitted **after** structure operations
  (columns, indexes, etc.), just like `ADD FOREIGN KEY`.
- `DROP TRIGGER` implements `PreOperationInterface` — it is emitted **before** structure operations,
  just like `DROP FOREIGN KEY`.

### Foreign key checks

When performing operations that would violate foreign key constraints — such as dropping a table referenced by another,
handling circular foreign keys, or performing large data migrations — you can explicitly disable and re-enable
foreign key checks:

```php
use Hector\Schema\Plan\DisableForeignKeyChecks;
use Hector\Schema\Plan\EnableForeignKeyChecks;

$plan = new Plan();

$plan->add(new DisableForeignKeyChecks());

$plan->drop('users');          // Would fail if referenced by other tables
$plan->drop('orders');
$plan->migrate('old_data', 'new_data', ['col_a' => 'col_b']);

$plan->add(new EnableForeignKeyChecks());
```

These operations leverage the compiler's automatic ordering:

- `DisableForeignKeyChecks` implements `PreOperationInterface` — it is emitted **first**, before any other operation
  (including `DROP FOREIGN KEY` and `DROP TRIGGER`).
- `EnableForeignKeyChecks` implements `PostOperationInterface` — it is emitted **last**, after all other operations
  (including `ADD FOREIGN KEY` and `CREATE TRIGGER`).

This guarantees that FK checks are disabled for the entire plan execution, regardless of where you place the
`add()` calls in your code.

**Generated SQL per driver:**

| Driver | Disable                      | Enable                       |
|--------|------------------------------|------------------------------|
| MySQL  | `SET FOREIGN_KEY_CHECKS = 0` | `SET FOREIGN_KEY_CHECKS = 1` |
| SQLite | `PRAGMA foreign_keys = OFF`  | `PRAGMA foreign_keys = ON`   |

> ⚠️ **Warning**: Disabling foreign key checks silences constraint violations. Use this only when you are certain that
> the data will be consistent after all operations complete. In most cases, you should let FK checks fail early to
> detect real data integrity issues.

> 💡 **Tip**: On SQLite, the table rebuild mechanism normally emits its own `PRAGMA foreign_keys = OFF/ON` pair.
> When the plan contains `DisableForeignKeyChecks`/`EnableForeignKeyChecks`, the rebuild skips these statements
> automatically to avoid duplication and premature re-enabling.

### Raw SQL statements

For SQL features not covered by the Plan API (fulltext indexes, engine changes, spatial indexes, partitions, stored
procedures, etc.),
you can inject raw SQL statements that will be executed as-is:

```php
use Hector\Schema\Index;

$plan->create('articles', function ($table) {
    $table->addColumn('id', 'INT', autoIncrement: true);
    $table->addColumn('title', 'VARCHAR(255)');
    $table->addColumn('body', 'TEXT');
    $table->addIndex('PRIMARY', ['id'], Index::PRIMARY);
});

// Fulltext index — not supported by the Plan API
$plan->raw('CREATE FULLTEXT INDEX ft_search ON articles (title, body)');

// Engine change
$plan->raw('ALTER TABLE articles ENGINE = InnoDB');
```

Raw statements bypass the compiler and are emitted **verbatim** during the structure pass, preserving their position
relative to other plan entries.

#### Driver filter

Raw statements can be scoped to specific database drivers using the `drivers` parameter. When the compiler does not
match any of the specified drivers, the statement is silently skipped:

```php
// Only executed on MySQL / MariaDB / Vitess
$plan->raw('ALTER TABLE articles ENGINE = InnoDB', drivers: ['mysql']);

// Only executed on SQLite
$plan->raw('PRAGMA journal_mode = WAL', drivers: ['sqlite']);

// Executed on all drivers (default behavior)
$plan->raw('INSERT INTO migrations (name) VALUES (\'v1.3\')');
```

This is especially useful for **mixed-environment migrations** where development uses SQLite and production uses MySQL:

```php
$plan->create('articles', function ($table) {
    $table->addColumn('id', 'INT', autoIncrement: true);
    $table->addColumn('title', 'VARCHAR(255)');
    $table->addColumn('body', 'TEXT');
    $table->addIndex('PRIMARY', ['id'], Index::PRIMARY);
});

// MySQL-only: fulltext index and engine
$plan->raw('CREATE FULLTEXT INDEX ft_search ON articles (title, body)', drivers: ['mysql']);
$plan->raw('ALTER TABLE articles ENGINE = InnoDB', drivers: ['mysql']);

// SQLite-only: WAL mode for better concurrency
$plan->raw('PRAGMA journal_mode = WAL', drivers: ['sqlite']);
```

Available driver names (matching `Connection\Driver\DriverInfo::getDriver()`):

| Driver    | Description | Compiler used    |
|-----------|-------------|------------------|
| `mysql`   | MySQL       | `MySQLCompiler`  |
| `mariadb` | MariaDB     | `MySQLCompiler`  |
| `vitess`  | Vitess      | `MySQLCompiler`  |
| `sqlite`  | SQLite      | `SqliteCompiler` |

> ℹ️ **Note**: When `drivers` is `null` (the default), the statement is emitted unconditionally — this preserves
> full backward compatibility.

**Execution order with raw statements:**

1. **Pre-operations** (global): `DisableForeignKeyChecks`, all `DROP FOREIGN KEY`, `DROP TRIGGER` from all entries
2. **Structure + Raw** (in declaration order): compiled structure operations interleaved with raw statements
3. **Post-operations** (global): all `ADD FOREIGN KEY`, `CREATE TRIGGER`, `EnableForeignKeyChecks` from all entries

This means raw statements are always executed **after** foreign key drops and **before** foreign key additions,
which is the safest position for structural changes.

> ⚠️ **Warning**: Raw SQL without a driver filter is database-specific and not portable across drivers.
> Use `drivers:` to safely scope vendor-specific statements in multi-environment projects.

### TableOperation operations

`TableOperation` is the abstract base class for `CreateTable` and `AlterTable`. It groups sub-operations
(columns, indexes, foreign keys) for a single table. It implements `Countable` and `IteratorAggregate`.

**Column operations (shared — CreateTable and AlterTable):**

| Method                                              | Description  |
|-----------------------------------------------------|--------------|
| `addColumn($name, $type, $nullable, $default, ...)` | Add a column |

> 🆕 **Info**: *Since version 1.4* — `$default` accepts a `Hector\Schema\Plan\Raw` expression (emitted
> verbatim), and `$hasDefault` is now `?bool` (defaults to `null` = auto-detected). See
> [Column defaults](#column-defaults).

**Column operations (ALTER only):**

| Method                                                   | Description     |
|----------------------------------------------------------|-----------------|
| `dropColumn($column)`                                    | Drop a column   |
| `modifyColumn($column, $type, $nullable, $default, ...)` | Modify a column |
| `renameColumn($column, $newName)`                        | Rename a column |

**Index operations:**

| Method                             | Description   |
|------------------------------------|---------------|
| `addIndex($name, $columns, $type)` | Add an index  |
| `dropIndex($index)`                | Drop an index |

**Foreign key operations:**

| Method                                                                      | Description        |
|-----------------------------------------------------------------------------|--------------------|
| `addForeignKey($name, $columns, $referencedTable, $referencedColumns, ...)` | Add a foreign key  |
| `dropForeignKey($foreignKey)`                                               | Drop a foreign key |

**Trigger operations:**

| Method                                                | Description                                                    |
|-------------------------------------------------------|----------------------------------------------------------------|
| `createTrigger($name, $timing, $event, $body, $when)` | Create a trigger on this table (on CreateTable and AlterTable) |
| `dropTrigger($name)`                                  | Drop a trigger on this table (ALTER only)                      |

**Table-level operations (ALTER only):**

| Method                                | Description                                               |
|---------------------------------------|-----------------------------------------------------------|
| `renameTable($newName)`               | Rename the table (emitted as a separate statement)        |
| `modifyCharset($charset, $collation)` | Modify character set and collation (MySQL only, on ALTER) |

**Utility methods:**

| Method                                 | Description                         |
|----------------------------------------|-------------------------------------|
| `getObjectName(): string`              | The table name                      |
| `isEmpty(): bool`                      | Check if the plan has no operations |
| `count(): int`                         | Number of operations                |
| `getArrayCopy(): OperationInterface[]` | Get all operations as an array      |

---

## Compilers

A `Plan` is compiled into SQL statements by a `CompilerInterface`. Three implementations are provided:

| Compiler         | Description                                          |
|------------------|------------------------------------------------------|
| `MySQLCompiler`  | Generates MySQL / MariaDB DDL                        |
| `SqliteCompiler` | Generates SQLite DDL, with automatic table rebuild   |
| `AutoCompiler`   | Detects the driver from a `Connection` and delegates |

All compilers implement `CompilerInterface`:

| Method                                      | Description                        |
|---------------------------------------------|------------------------------------|
| `compile($plan, $schema): iterable<string>` | Compile a plan into SQL statements |

### AutoCompiler

The simplest approach — pass your `Connection` and the correct compiler is resolved automatically:

```php
use Hector\Schema\Plan\Compiler\AutoCompiler;

$compiler = new AutoCompiler($connection);

foreach ($plan->getStatements($compiler) as $sql) {
    $connection->execute($sql);
}
```

| Constructor Parameter | Type         | Description                        |
|-----------------------|--------------|------------------------------------|
| `$connection`         | `Connection` | Used to detect the database driver |

Supported drivers: `mysql`, `mariadb`, `vitess`, `sqlite`.

### Using a specific compiler

```php
use Hector\Schema\Plan\Compiler\MySQLCompiler;
use Hector\Schema\Plan\Compiler\SqliteCompiler;

$compiler = new MySQLCompiler();
// or
$compiler = new SqliteCompiler();

foreach ($plan->getStatements($compiler) as $sql) {
    echo $sql . ";\n";
}
```

> 💡 **Tip**: Without a `Connection`, you can use compilers directly to generate SQL strings — useful for dry-run
> migrations, SQL export scripts, or testing.

---

## Schema introspection

When a `Schema` object is provided to `getStatements()`, the compiler can introspect the current database state and
adapt its strategy:

```php
use Hector\Schema\Generator\MySQL as MySQLGenerator;

$generator = new MySQLGenerator($connection);
$schema = $generator->generateSchema('mydb');

foreach ($plan->getStatements($compiler, $schema) as $sql) {
    $connection->execute($sql);
}
```

With a schema, the compiler will:

- **Check index existence** before adding — if an index already exists, it emits a `DROP INDEX` + `CREATE INDEX` pair
  instead of failing
- **Use table rebuild** for SQLite when the operation requires it (e.g., `MODIFY COLUMN`, `ADD/DROP FOREIGN KEY`)

Without a schema, all operations are compiled optimistically — the compiler assumes the database supports them natively.

> ⚠️ **Warning**: The schema represents the database state **before** the plan is executed. It may become stale
> after execution. If you need to run multiple plans sequentially, regenerate the schema between each one.

---

## Operation ordering

`Plan` automatically reorders operations to avoid constraint violations and dependency issues:

1. **Pre-operations** (global): `DisableForeignKeyChecks`, `DROP FOREIGN KEY`, `DROP TRIGGER` — from all entries
2. **Structure + Raw** (in declaration order): `CREATE/ALTER/DROP TABLE`, columns, indexes, data migrations, views, and
   raw SQL statements
3. **Post-operations** (global): `ADD FOREIGN KEY`, `CREATE TRIGGER`, `EnableForeignKeyChecks` — from all entries

This happens transparently — you don't need to worry about the order in which you add operations to the plan.
Raw statements are interleaved with structure operations in the order you declared them.

```php
use Hector\Schema\Index;

$plan = new Plan();

// These can be added in any order — FK ordering is automatic
$plan->create('orders', function ($table) {
    $table->addColumn('id', 'INT', autoIncrement: true);
    $table->addColumn('user_id', 'INT');
    $table->addIndex('PRIMARY', ['id'], Index::PRIMARY);
    $table->addForeignKey('fk_order_user', ['user_id'], 'users', ['id']);
});

$plan->raw('CREATE FULLTEXT INDEX ft_order_ref ON orders (reference)');

$plan->create('users', function ($table) {
    $table->addColumn('id', 'INT', autoIncrement: true);
    $table->addColumn('name', 'VARCHAR(100)');
    $table->addIndex('PRIMARY', ['id'], Index::PRIMARY);
});

// Execution order:
// 1. CREATE TABLE orders (structure)
// 2. CREATE FULLTEXT INDEX ft_order_ref ON orders (raw — in position)
// 3. CREATE TABLE users (structure)
// 4. ALTER TABLE orders ADD CONSTRAINT fk_order_user ... (post — last)
foreach ($plan->getStatements($compiler) as $sql) {
    $connection->execute($sql);
}
```

---

## SQLite: table rebuild

SQLite has limited `ALTER TABLE` support — it cannot modify columns, add or drop foreign keys. When these operations are
detected and a `Schema` is available, the `SqliteCompiler` automatically generates a **table rebuild sequence**:

1. `PRAGMA foreign_keys = OFF`
2. `CREATE TABLE` a temporary table with the new schema
3. `INSERT INTO ... SELECT ...` to migrate data from the original table
4. `DROP TABLE` the original table
5. `ALTER TABLE ... RENAME TO ...` the temporary table to the original name
6. Recreate non-primary indexes
7. `PRAGMA foreign_keys = ON`

This is fully transparent — you write the same `Plan` for both MySQL and SQLite.

```php
// Works on both MySQL and SQLite
$plan = new Plan();
$plan->alter('users', function ($table) {
    $table->modifyColumn('name', 'VARCHAR(200)');
    $table->addForeignKey('fk_team', ['team_id'], 'teams', ['id']);
});

// On MySQL: ALTER TABLE users MODIFY COLUMN ..., ADD CONSTRAINT ...
// On SQLite: Full table rebuild sequence (7 statements)
foreach ($plan->getStatements($compiler, $schema) as $sql) {
    $connection->execute($sql);
}
```

> ⚠️ **Warning**: Table rebuild requires a `Schema`. Without it, the SQLite compiler will attempt native `ALTER TABLE`
> statements that may fail.

---

## Complete example: migration script

This example demonstrates how operations can be declared in **any order** — the compiler automatically reorders
them to avoid constraint violations:

```php
use Hector\Connection\Connection;
use Hector\Schema\ForeignKey;
use Hector\Schema\Index;
use Hector\Schema\Generator\MySQL as MySQLGenerator;
use Hector\Schema\Plan\Compiler\AutoCompiler;
use Hector\Schema\Plan\CreateTrigger;
use Hector\Schema\Plan\Plan;

$connection = new Connection('mysql:host=localhost;dbname=mydb', 'root', 'secret');
$generator = new MySQLGenerator($connection);
$schema = $generator->generateSchema('mydb');
$compiler = new AutoCompiler($connection);

$plan = new Plan();

// 1. Alter posts — drop old FK, add new FK and a trigger
$plan->alter('posts', function ($table) {
    $table->dropForeignKey('fk_posts_author');
    $table->addColumn('category_id', 'INT', nullable: true);
    $table->addForeignKey(
        name: 'fk_posts_category',
        columns: ['category_id'],
        referencedTable: 'categories',
        referencedColumns: ['id'],
        onDelete: ForeignKey::RULE_SET_NULL,
    );
    $table->createTrigger(
        'trg_posts_audit',
        CreateTrigger::AFTER,
        CreateTrigger::INSERT,
        'INSERT INTO audit_log (table_name, action) VALUES (\'posts\', \'insert\')',
    );
});

// 2. Create the categories table (referenced by posts FK above)
$plan->create('categories', function ($table) {
    $table->addColumn('id', 'INT', autoIncrement: true);
    $table->addColumn('name', 'VARCHAR(100)');
    $table->addIndex('PRIMARY', ['id'], Index::PRIMARY);
    $table->addIndex('idx_name', ['name'], Index::UNIQUE);
});

// 3. MySQL-only: fulltext index on posts
$plan->raw('CREATE FULLTEXT INDEX ft_search ON posts (title, body)', drivers: ['mysql']);

// 4. Drop an old table
$plan->drop('legacy_posts', ifExists: true);

// Execute
foreach ($plan->getStatements($compiler, $schema) as $sql) {
    $connection->execute($sql);
}
```

The compiler produces SQL in this order:

<details open name="compiler-output">
<summary>MySQL</summary>

```mysql
-- Pass 1 — Pre-operations: FK drops first
ALTER TABLE `posts` DROP FOREIGN KEY `fk_posts_author`;

-- Pass 2 — Structure (in declaration order)
ALTER TABLE `posts` ADD COLUMN `category_id` INT NULL DEFAULT NULL;

CREATE TABLE `categories` (
  `id` INT NOT NULL AUTO_INCREMENT,
  `name` VARCHAR(100) NOT NULL,
  PRIMARY KEY (`id`),
  UNIQUE INDEX `idx_name` (`name`)
);

CREATE FULLTEXT INDEX ft_search ON posts (title, body);

DROP TABLE IF EXISTS `legacy_posts`;

-- Pass 3 — Post-operations: FK additions and trigger creations last
ALTER TABLE `posts` ADD CONSTRAINT `fk_posts_category` FOREIGN KEY (`category_id`)
  REFERENCES `categories` (`id`) ON DELETE SET NULL;

CREATE TRIGGER `trg_posts_audit` AFTER INSERT ON `posts`
  FOR EACH ROW INSERT INTO audit_log (table_name, action) VALUES ('posts', 'insert');
```

</details>

<details name="compiler-output">
<summary>SQLite (with schema)</summary>

```sqlite
-- Pass 2 — Structure (in declaration order)
-- The ALTER on posts triggers a full table rebuild (drop FK + add FK):
PRAGMA foreign_keys = OFF;

CREATE TABLE "__htemp_xxx_posts" (
  "id" INT NOT NULL PRIMARY KEY AUTOINCREMENT,
  "title" VARCHAR(255) NOT NULL,
  "body" TEXT NULL,
  "author_id" INT NOT NULL,
  "category_id" INT NULL DEFAULT NULL,
  CONSTRAINT "fk_posts_category" FOREIGN KEY ("category_id")
    REFERENCES "categories" ("id") ON DELETE SET NULL
);

INSERT INTO "__htemp_xxx_posts" ("id", "title", "body", "author_id")
  SELECT "id", "title", "body", "author_id" FROM "posts";

DROP TABLE "posts";

ALTER TABLE "__htemp_xxx_posts" RENAME TO "posts";

PRAGMA foreign_keys = ON;

CREATE TABLE "categories" (
  "id" INT NOT NULL PRIMARY KEY AUTOINCREMENT,
  "name" VARCHAR(100) NOT NULL
);

CREATE UNIQUE INDEX "idx_name" ON "categories" ("name");

-- raw('CREATE FULLTEXT INDEX ...', drivers: ['mysql']) → skipped

DROP TABLE IF EXISTS "legacy_posts";

-- Pass 3 — Post-operations: trigger creation last
CREATE TRIGGER IF NOT EXISTS "trg_posts_audit" AFTER INSERT ON "posts"
  FOR EACH ROW BEGIN
    INSERT INTO audit_log (table_name, action) VALUES ('posts', 'insert');
  END
```

</details>

> 💡 **Tip**: Notice how the `categories` table is created **before** the FK that references it, even though
> `alter('posts')` was declared first. The 3-pass ordering ensures drops happen first, then structure,
> then additions — regardless of the declaration order in the Plan.
> The MySQL-only `raw()` statement is silently skipped on SQLite thanks to the `drivers:` filter.
