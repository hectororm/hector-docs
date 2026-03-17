---
breadcrumb:
  - Getting started
summary-order: 1
keywords:
  - getting-started
  - installation
  - quick-start
  - orm
---

# 🚀 Getting started

## Introduction

**Hector ORM** is a lightweight, framework-agnostic PHP ORM — designed to be modular, fast, and expressive. It draws
inspiration from existing ORM concepts, while promoting freedom of structure and strong typing.

### Requirements

- PHP 8.0+
- PDO extension
- Database driver (e.g., `pdo_mysql`, `pdo_sqlite`)

### What is an ORM?

> Object-relational mapping (ORM, O/RM, and O/R mapping tool!) in computer science is a programming technique for
> converting data between incompatible type systems using object-oriented programming languages. This creates, in
> effect,
> a "virtual object database" that can be used from within the programming language.
>
> — [Wikipedia](https://en.wikipedia.org/wiki/Object%E2%80%93relational_mapping)

### Choose your style

You can manage entities in multiple ways:

* [Classic entities](./orm/entity.md): define PHP properties explicitly and **Hector ORM** handles mapping.
* [Magic entities](./orm/entity.md): rely on **Hector ORM** dynamic behavior using PHP’s magic methods.
* Roll your own 🧪: create a custom Mapper if you want total control over mapping logic.

### Why Hector ORM?

| Feature                  | Hector ORM | Doctrine | Eloquent |
|--------------------------|:----------:|:--------:|:--------:|
| Zero config              |     ✅      |    ❌     |    ⚠️    |
| Framework-agnostic       |     ✅      |    ✅     |    ❌     |
| Magic + Classic entities |     ✅      |    ❌     |    ✅     |
| Schema introspection     |     ✅      |    ❌     |    ❌     |

### DBMS compatibility

| DBMS    |   Version   | Compatibility |
|---------|:-----------:|:-------------:|
| MySQL   |  5.7 - 9.6  |       ✔       |
| MariaDB | 10.5 - 12.2 |       ✔       |
| Vitess  |      -      |       ✔       |
| SQLite  |     3.x     |       ✔       |

> ℹ️ **Note**: Versions listed are actively tested in CI. Older versions may work but are not officially supported.

## Quick start

### 1. Installation

Install with [Composer](https://getcomposer.org/):

```bash
composer require hectororm/hectororm
```

### 2. Create a database connection

```php
use Hector\Connection\Connection;

$connection = new Connection(
    dsn: 'mysql:host=localhost;dbname=my_database',
    username: 'user',
    password: 'pass',
);
```

> 💡 **Tip**: See the [Connection documentation](components/connection.md) for read/write separation, multiple
> connections, and logging.

### 3. Boot the ORM

```php
use Hector\Orm\OrmFactory;

$orm = OrmFactory::orm(
    options: [
        'schemas' => ['my_database'], // Your database name(s)
    ],
    connection: $connection,
);
```

The `schemas` option tells Hector ORM which database(s) to introspect at boot time. It will read the table structure
and cache the metadata for entity mapping.

> 💡 **Tip**: See [Cache](orm/cache.md) to persist schema metadata across requests in production, and
> [Advanced configuration](orm/configuration.md) for all available options.

### 4. Define entities

Entity classes map to database tables. By default, the class name is converted to **snake_case** to find the table:

```php
use Hector\Orm\Attributes as Orm;
use Hector\Orm\Entity\MagicEntity;

// Maps to table "foo"
#[Orm\HasOne(Bar::class, 'bar')]
class Foo extends MagicEntity {}

// Maps to table "bar"
#[Orm\BelongsTo(Foo::class, 'foo')]
class Bar extends MagicEntity {}
```

> 💡 **Tip**: You can also use [Classic entities](orm/entity.md) with explicitly declared properties for better IDE
> support. See [Entities](orm/entity.md) for details.

### 5. Use the ORM

```php
$foo = Foo::findOrFail(1); // Find a Foo entity by primary key
$bar = $foo->bar;          // Access the related Bar entity (lazy loaded)

echo $bar->field;          // Access a field from the related Bar
```

---

## Explore the ecosystem

You're now ready to build with **Hector ORM**. Here is a guide to the documentation:

### ORM

* [Entities](orm/entity.md) — Define your data models as PHP classes (Magic or Classic)
* [Relationships](orm/relationships.md) — HasOne, HasMany, BelongsTo, ManyToMany
* [Builder](orm/builder.md) — Query, filter and paginate entities
* [Events](orm/events.md) — Hook into the entity lifecycle (save, delete)
* [Advanced configuration](orm/configuration.md) — Table mapping, column types, hidden fields
* [Cache](orm/cache.md) — Schema caching for production

### Standalone components

Each component can be used independently of the ORM:

* [Collection](components/collection.md) — Typed and lazy collections for data manipulation
* [Connection](components/connection.md) — PDO wrapper with read/write separation and logging
* [Data Types](components/data-types.md) — Type casting between database and PHP (DateTime, Enum, JSON, UUID...)
* [Query Builder](components/query.md) — Fluent SQL query building
* [Schema](components/schema.md) — Database introspection (tables, columns, indexes, foreign keys)
* [Plan](components/plan.md) — DDL operations builder (CREATE/ALTER/DROP TABLE)
* [Migration](components/migration.md) — Database migration runner with providers and trackers
* [Pagination](components/pagination.md) — Offset, cursor and range pagination with PSR-7

### Going further

* [Architecture](architecture.md) — Package overview, dependency graph, and design philosophy
* [Berlioz Framework](integration/berlioz.md) — First-class integration with Berlioz Framework
* [Framework integration](integration/framework.md) — How to integrate Hector ORM into your own framework
