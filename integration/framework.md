---
breadcrumb:
  - Integration
  - Framework integration
summary-order: 4;1
keywords:
  - framework
  - integration
  - service
  - container
  - bootstrap
---

# 🔌 Framework integration

This guide is intended for **framework authors** or developers who want to integrate Hector ORM into a framework or a
custom application bootstrap. It describes the key entry points, the services to expose, and the lifecycle to manage.

> 💡 **Tip**: If you are a framework **user** looking to get started quickly, see [Getting started](../index.md).

---

## Overview

Integrating Hector ORM into a framework requires wiring a few core services:

| Service          | Class                                          | Role                                                   |
|------------------|------------------------------------------------|--------------------------------------------------------|
| Connection       | `Hector\Connection\Connection`                 | PDO wrapper for database access                        |
| ORM              | `Hector\Orm\Orm`                               | Central ORM instance (entity storage, mappers, events) |
| Schema container | `Hector\Schema\SchemaContainer`                | Cached schema metadata                                 |
| Event dispatcher | `Psr\EventDispatcher\EventDispatcherInterface` | Entity lifecycle events (optional)                     |
| Cache            | `Psr\SimpleCache\CacheInterface`               | Schema metadata caching (optional but recommended)     |

The simplest way to bootstrap everything is through `OrmFactory::orm()`, which handles connection creation, schema
introspection, and caching in a single call.

---

## Bootstrapping the ORM

### Using `OrmFactory` (recommended)

`OrmFactory::orm()` is the main entry point. It creates a `Connection`, introspects the database schema, and returns a
fully configured `Orm` instance:

```php
use Hector\Orm\Orm;
use Hector\Orm\OrmFactory;
use Psr\EventDispatcher\EventDispatcherInterface;
use Psr\SimpleCache\CacheInterface;

// In your framework's service container definition:

/** @var CacheInterface $cache — from your framework's cache system */
/** @var EventDispatcherInterface $eventDispatcher — from your framework's event system */

$orm = OrmFactory::orm(
    options: [
        'schemas' => ['my_database'],         // Database name(s) to introspect
        'dsn' => 'mysql:host=localhost;dbname=my_database',
        'username' => 'root',
        'password' => 'secret',
        'read_dsn' => null,                   // Optional: read replica DSN
        'name' => 'default',                  // Optional: connection name
        'log' => false,                       // Optional: enable query logging
    ],
    connection: null,                         // null = auto-created from options
    eventDispatcher: $eventDispatcher,        // Optional
    cache: $cache,                            // Optional but recommended
);
```

When `connection` is `null`, the factory creates a `Connection` internally using the `dsn`, `username`, `password`,
`read_dsn`, `name`, and `log` keys from the options array.

### Using an existing connection

If your framework already manages database connections, pass the `Connection` (or `ConnectionSet`) directly:

```php
use Hector\Connection\Connection;
use Hector\Orm\OrmFactory;

$connection = new Connection(
    dsn: 'mysql:host=localhost;dbname=my_database',
    username: 'root',
    password: 'secret',
);

$orm = OrmFactory::orm(
    options: ['schemas' => ['my_database']],
    connection: $connection,
    cache: $cache,
);
```

### Manual bootstrapping

For full control, you can skip `OrmFactory` and wire the `Orm` class directly:

```php
use Hector\Connection\Connection;
use Hector\Orm\Orm;
use Hector\Schema\Generator\MySQL;

$connection = new Connection('mysql:host=localhost;dbname=my_database', 'root', 'secret');
$generator = new MySQL($connection);
$schemaContainer = $generator->generateSchemas('my_database');

$orm = new Orm($connection, $schemaContainer, $eventDispatcher);
```

> ⚠️ **Warning**: With manual bootstrapping, schema introspection happens every time. You should implement your own
> caching strategy around the `SchemaContainer` (it is serializable).

---

## Services to expose

When building a framework integration (ServiceProvider, Bundle, etc.), consider exposing these services in the
dependency injection container:

| Service           | Type                              | Singleton | Notes                                   |
|-------------------|-----------------------------------|:---------:|-----------------------------------------|
| `Orm`             | `Hector\Orm\Orm`                  |     ✔     | Core ORM instance — must be a singleton |
| `Connection`      | `Hector\Connection\Connection`    |     ✔     | Database connection                     |
| `ConnectionSet`   | `Hector\Connection\ConnectionSet` |     ✔     | Only if using multiple connections      |
| `SchemaContainer` | `Hector\Schema\SchemaContainer`   |     ✔     | Cached schema metadata                  |
| `QueryBuilder`    | `Hector\Query\QueryBuilder`       |     —     | Can be created per-request              |

> ℹ️ The `Orm` instance is a **singleton** internally (accessed via `Orm::get()`). The constructor of `Orm` registers
> itself as the global instance. You should only create one `Orm` per application.

---

## Cache integration

Schema introspection is expensive (it queries `INFORMATION_SCHEMA`). In production, you should always provide a PSR-16
`CacheInterface` implementation to persist the schema metadata across requests.

```php
use Hector\Orm\OrmFactory;

// Plug your framework's PSR-16 cache adapter
$orm = OrmFactory::orm(
    options: ['schemas' => ['my_database']],
    connection: $connection,
    cache: $frameworkCache, // Any PSR-16 CacheInterface
);
```

The cache key used is `OrmFactory::CACHE_ORM_KEY` (`'HECTOR_ORM'`).

> ⚠️ **Warning**: Clear this cache key after database migrations or schema changes.

See [Cache documentation](../orm/cache.md) for more details.

---

## Event dispatcher integration

Hector ORM dispatches [PSR-14 events](../orm/events.md) during the entity lifecycle. To integrate with your framework's
event system, pass its `EventDispatcherInterface` to the factory:

```php
$orm = OrmFactory::orm(
    options: ['schemas' => ['my_database']],
    connection: $connection,
    eventDispatcher: $frameworkEventDispatcher,
);
```

If no dispatcher is provided, events are silently ignored with zero overhead.

See [Events documentation](../orm/events.md) for the list of dispatched events.

---

## Migration CLI integration

The [Migration package](../components/migration.md) provides a `MigrationRunner` that can be wrapped in your
framework's CLI command system. Here is a typical pattern:

```php
use Hector\Migration\MigrationRunner;
use Hector\Migration\Provider\DirectoryProvider;
use Hector\Migration\Tracker\DbTracker;
use Hector\Schema\Plan\Compiler\AutoCompiler;

// These would be resolved from the framework's container
$compiler = new AutoCompiler($connection);
$provider = new DirectoryProvider($migrationsDirectory);
$tracker = new DbTracker($connection);

$runner = new MigrationRunner(
    provider: $provider,
    tracker: $tracker,
    compiler: $compiler,
    connection: $connection,
    logger: $frameworkLogger,           // Optional: PSR-3
    eventDispatcher: $frameworkEvents,  // Optional: PSR-14
);

// In your "migrate" command:
$applied = $runner->up();

// In your "rollback" command:
$reverted = $runner->down(steps: 1);

// In your "status" command:
$status = $runner->getStatus();
```

---

## Example: skeleton ServiceProvider

Below is a pseudo-code example of what a framework integration might look like. Adapt it to your framework's conventions
(Symfony Bundle, Laravel ServiceProvider, Slim middleware, etc.):

```php
use Hector\Connection\Connection;
use Hector\Orm\Orm;
use Hector\Orm\OrmFactory;

class HectorOrmServiceProvider
{
    public function register(ContainerInterface $container): void
    {
        // Connection (singleton)
        $container->singleton(Connection::class, function () use ($container) {
            return new Connection(
                dsn: $container->get('config.database.dsn'),
                username: $container->get('config.database.username'),
                password: $container->get('config.database.password'),
                readDsn: $container->get('config.database.read_dsn'),
            );
        });

        // ORM (singleton)
        $container->singleton(Orm::class, function () use ($container) {
            return OrmFactory::orm(
                options: [
                    'schemas' => $container->get('config.database.schemas'),
                ],
                connection: $container->get(Connection::class),
                eventDispatcher: $container->get(EventDispatcherInterface::class),
                cache: $container->get(CacheInterface::class),
            );
        });
    }

    public function boot(ContainerInterface $container): void
    {
        // Ensure the ORM is initialized at boot time
        $container->get(Orm::class);
    }
}
```

> ℹ️ The `boot()` method ensures the ORM singleton is created early. Entity classes use `Orm::get()` internally, so
> the instance must exist before any entity operation.
