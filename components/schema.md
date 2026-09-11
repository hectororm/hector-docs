---
breadcrumb:
  - Components
  - Schema
  - Overview
summary-order: ;5;1
keywords:
  - schema
  - introspection
  - table
  - column
  - index
  - foreign-key
---

# 📏 Schema

> ℹ️ **Note**: While the Schema component is part of the **Hector ORM** ecosystem, it is available as a standalone
> package: [`hectororm/schema`](https://github.com/hectororm/schema).
> You can find it on [Packagist](https://packagist.org/packages/hectororm/schema).
> You can use it independently of the ORM, in any PHP application. 🎉

**Hector Schema** is the schema introspection and metadata module of **Hector ORM**. It provides a simple and consistent
API to explore and manage your database structure. While tightly integrated with **Hector ORM**, this package is
entirely independent and can be used on its own.

## DBMS compatibility

| DBMS    |   Version   | Compatibility |
|---------|:-----------:|:-------------:|
| MySQL   |  5.7 - 9.6  |       ✔       |
| MariaDB | 10.5 - 12.2 |       ✔       |
| Vitess  |      -      |       ✔       |
| SQLite  |     3.x     |       ✔       |

> ℹ️ **Note**: Versions listed are actively tested. Older versions may work but are not officially supported.
> Know of a DBMS version not listed here but works fine? Contributions are welcome — open a PR!

## Usage

### Generate a schema

The schema generator reads your DB structure and returns rich PHP objects that describe it:

```php
use Hector\Connection\Connection;
use Hector\Schema\Generator\MySQL;
use Hector\Schema\Generator\Sqlite;

// MySQL / MariaDB
$connection = new Connection('mysql:host=localhost;dbname=mydb', 'user', 'pass');
$generator = new MySQL($connection);

// SQLite
$connection = new Connection('sqlite:/path/to/database.db');
$generator = new Sqlite($connection);

$schema = $generator->generateSchema('schema_name');
// Returns a `Hector\Schema\Schema` object

$container = $generator->generateSchemas('schema1_name', 'schema2_name');
// Returns a `Hector\Schema\SchemaContainer` object
```

Available generators:

* `Hector\Schema\Generator\MySQL` — for MySQL and MariaDB
* `Hector\Schema\Generator\Sqlite` — for SQLite

> 💡 **Tip**: You can use different generators for different environments (e.g., dev SQLite, prod MySQL).

### Caching schemas

Schema generation can be expensive for large databases. This library does **not** include built-in caching, as caching
strategies vary widely.

Instead, you can serialize schema objects and restore them later:

```php
$serialized = serialize($schema);
file_put_contents('cache/schema.ser', $serialized);

// Later...
$schema = unserialize(file_get_contents('cache/schema.ser'));
```

Inheritance and references between objects are preserved during (de)serialization. This makes caching simple and
flexible — ideal for integrating into your own caching logic 🌟

> ⚠️ **Warning**: Ensure your cache is invalidated when the database schema changes (e.g., after migrations).

---

## API reference

This section covers the main classes used to explore database schemas.

### `Hector\Schema\SchemaContainer`

A container for multiple schemas. Useful when your application interacts with multiple databases or schemas.

**Key methods:**

* `getSchemas(?string $connection = null): Generator`
* `hasSchema(string $name, ?string $connection = null): bool`
* `getSchema(string $name, ?string $connection = null): Schema`
* `hasTable(string $name, ?string $schemaName = null, ?string $connection = null): bool`
* `getTable(string $name, ?string $schemaName = null, ?string $connection = null): Table`

```php
// Get table from default schema
$usersTable = $container->getTable('users');

// Get table from specific schema
$usersTable = $container->getTable('users', 'my_database');

// Get table from specific schema and connection
$usersTable = $container->getTable('users', 'my_database', 'replica');
```

> This class is iterable: `foreach ($container as $schema)` will yield `Hector\Schema\Schema` objects.

### `Hector\Schema\Schema`

Represents a single schema (i.e., database).

**Key methods:**

* `getName(bool $quoted = false): string`
* `getCharset(): string`
* `getCollation(): string`
* `getTables(): Generator`
* `hasTable(string $name): bool`
* `getTable(string $name): Table`
* `getContainer(): ?SchemaContainer`

> **Deprecated:** The `$quoted` parameter on `getName()` is deprecated. Use `Hector\Query\Statement\Quoted` for
> driver-aware identifier quoting.

> This class is also iterable: `foreach ($schema as $table)` yields `Hector\Schema\Table` objects.

### `Hector\Schema\Table`

Represents a table and its structure.

**Key methods:**

* Identification:

    * `getSchemaName(bool $quoted = false): string`
    * `getName(bool $quoted = false): string`
    * `getFullName(bool $quoted = false): string`

* Metadata:

    * `getType(): string`
    * `getCharset(): ?string`
    * `getCollation(): ?string`

* Columns:

    * `getColumns(): Generator`
    * `getColumnsName(bool $quoted = false, ?string $tableAlias = null): array`
    * `hasColumn(string $name): bool`
    * `getColumn(string $name): Column`
    * `getAutoIncrementColumn(): ?Column`

* Indexes:

    * `getIndexes(?string $type = null): Generator`
    * `hasIndex(string $name): bool`
    * `getIndex(string $name): Index`
    * `getPrimaryIndex(): ?Index`

* Foreign Keys:

    * `getForeignKeys(Table $table = null): Generator`

* Other:

    * `getSchema(): Schema`

> **Deprecated:** The `$quoted` parameter on `getSchemaName()`, `getName()`, `getFullName()`, and `getColumnsName()` is
> deprecated. Use `Hector\Query\Statement\Quoted` for driver-aware identifier quoting.

### `Hector\Schema\Column`

Represents a column in a table.

**Key methods:**

* Identification:

    * `getName(bool $quoted = false, ?string $tableAlias = null): string`
    * `getFullName(bool $quoted = false): string`

* Metadata:

    * `getPosition(): int`
    * `getDefault(): mixed`
    * `isNullable(): bool`
    * `getType(): string`
    * `isAutoIncrement(): bool`
    * `getMaxlength(): ?int`
    * `getNumericPrecision(): ?int`
    * `getNumericScale(): ?int`
    * `getOnUpdate(): ?string` — database-side update expression, e.g. `CURRENT_TIMESTAMP(6)` (unreleased)
    * `getDatetimePrecision(): ?int` — fractional seconds precision from MySQL/MariaDB metadata (unreleased)
    * `isUnsigned(): bool`
    * `getCharset(): ?string`
    * `getCollation(): ?string`

* Other:

    * `getTable(): Table`
    * `isPrimary(): bool`

> **Deprecated:** The `$quoted` parameter on `getName()` and `getFullName()` is deprecated. Use
`Hector\Query\Statement\Quoted` for driver-aware identifier quoting.

For MySQL/MariaDB, `getOnUpdate()` returns a normalized expression (`CURRENT_TIMESTAMP` or
`CURRENT_TIMESTAMP(n)`) when a column has an automatic update clause, otherwise `null`.
`getDatetimePrecision()` preserves the distinction between zero fractional digits (`0`) and absent metadata (`null`).
Both properties survive serialization; caches created before these properties existed deserialize with `null` values.

SQLite introspection returns `null` for both properties. In particular, a migration using
[`useCurrentOnUpdate`](plan.md#automatic-update-timestamps-mysql--mariadb) does not create this property in SQLite:
the introspected schema describes what the database actually stores.

### `Hector\Schema\Index`

Represents a table index.

**Key methods:**

* `getName(): string`
* `getType(): string`
* `getColumnsName(bool $quoted = false, ?string $tableAlias = null): array`
* `getTable(): Table`
* `getColumns(): array`
* `hasColumn(Column $column): bool`

### `Hector\Schema\ForeignKey`

Represents a foreign key constraint.

**Key methods:**

* `getName(): string`
* `getColumnsName(bool $quoted = false, ?string $tableAlias = null): array`
* `getReferencedSchemaName(): string`
* `getReferencedTableName(): string`
* `getReferencedColumnsName(bool $quoted = false, ?string $tableAlias = null): array`
* `getUpdateRule(): string`
* `getDeleteRule(): string`
* `getTable(): Table`
* `getColumns(): Generator`
* `getReferencedTable(): ?Table`
* `getReferencedColumns(): Generator`

> **Deprecated:** The `$quoted` parameter on `getColumnsName()` and `getReferencedColumnsName()` is deprecated. Use
`Hector\Query\Statement\Quoted` for driver-aware identifier quoting.

---

## Example: basic schema introspection

```php
$generator = new MySQL($connection);
$schema = $generator->generateSchema('my_database');

foreach ($schema as $table) {
    echo "Table: " . $table->getName() . "\n";
    
    // Columns
    foreach ($table->getColumns() as $column) {
        echo "  Column: " . $column->getName() . " (" . $column->getType() . ")";
        echo $column->isNullable() ? " NULL" : " NOT NULL";
        echo $column->isPrimary() ? " [PK]" : "";
        echo "\n";
    }
    
    // Indexes
    foreach ($table->getIndexes() as $index) {
        echo "  Index: " . $index->getName() . " (" . $index->getType() . ")";
        echo " on [" . implode(', ', $index->getColumnsName()) . "]\n";
    }
    
    // Foreign keys
    foreach ($table->getForeignKeys() as $fk) {
        echo "  FK: " . $fk->getName();
        echo " [" . implode(', ', $fk->getColumnsName()) . "]";
        echo " -> " . $fk->getReferencedTableName();
        echo " [" . implode(', ', $fk->getReferencedColumnsName()) . "]\n";
    }
    
    echo "\n";
}
```

This example will output the structure of your database with all tables, columns, indexes and foreign keys.
Great for CLI tools, documentation generators or migration scripts! 📊

---

## Installation

```bash
composer require hectororm/schema
```

---

## See also

- [Plan](plan.md) — the DDL migration module included in this package. Build CREATE TABLE, ALTER TABLE, DROP TABLE
  operations and compile them into SQL for MySQL/MariaDB and SQLite.
- [Migration](migration.md) — orchestrate database migrations with providers, trackers and a runner.
