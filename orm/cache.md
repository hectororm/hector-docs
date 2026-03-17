---
breadcrumb:
  - ORM
  - Cache
summary-order: ;2
keywords:
  - cache
  - psr-16
  - schema
  - performance
---

# 💾 Cache

**Hector ORM** includes an internal caching strategy designed to improve performance by avoiding redundant operations
during runtime. It automatically stores and reuses:

* Schema metadata (table structure, columns, primary keys, etc.)
* Data type mappings
* Reflection data for entities

This caching mechanism is built upon the [PSR-16 Simple Cache](https://www.php-fig.org/psr/psr-16/) specification and is
abstracted internally to allow flexibility and extensibility.

To manage and persist cached data, Hector ORM provides a factory class: `Hector\Orm\OrmFactory`. It handles cache
initialization and usage automatically but also allows you to customize or plug in your own PSR-16 cache implementation
if needed.

> 💡 **Tip**: You can inject your own `Psr\SimpleCache\CacheInterface` into the factory to take full control over cache
> persistence (e.g. file-based, memory, Redis, etc.).

Example:

```php
use Hector\Orm\OrmFactory;
use Symfony\Component\Cache\Adapter\FilesystemAdapter;
use Symfony\Component\Cache\Psr16Cache;

$psr6Adapter = new FilesystemAdapter();
$cache = new Psr16Cache($psr6Adapter);

$orm = OrmFactory::orm(
    options: [...],
    connection: $connection,
    cache: $cache,
);
```

> 💡 **Tip**: See [Advanced configuration](configuration.md) for available `options`.

By default, if no cache is provided, Hector ORM falls back to a lightweight in-memory cache.

---

## Cache invalidation

The cache should be invalidated whenever your database schema changes (e.g., after running migrations).

With PSR-16 implementation:

```php
$cache->clear();
```

> ⚠️ **Warning**: Integrate cache clearing into your deployment pipeline, right after database migrations.

---

## Using in-memory cache explicitly

If you want to disable persistent caching entirely (useful for testing):

```php
use Hector\Orm\OrmFactory;

$orm = OrmFactory::orm(
    options: [...],
    connection: $connection,
    cache: null,
);
```

This cache lives only for the duration of the request and is not persisted.
