---
breadcrumb:
  - ORM
  - Builder
summary-order: ;3
keywords:
  - builder
  - query
  - find
  - where
  - pagination
  - orm
---

# 🔧 Builder

The `Builder` provides a high-level, entity-oriented way to perform queries on your data models. It is built on top of
the lower-level `QueryBuilder` and integrates deeply with **Hector ORM** entity management.

## Accessing the builder

You can access the builder using the static `Entity::query()` method, or instantiate it directly:

```php
use Hector\Orm\Query\Builder;

$builder = MyEntity::query();
// or
$builder = new Builder(MyEntity::class);
```

> The builder internally uses `hectororm/query`:
> see [QueryBuilder documentation](../components/query.md).
> You can also use `QueryBuilder` independently of the ORM:
> see [`hectororm/query`](https://github.com/hectororm/query) on GitHub
> or [Packagist](https://packagist.org/packages/hectororm/query).

## Finding entities

The builder provides several methods to retrieve entities. Here is a quick overview:

| Method                         | Lookup by       | Returns      | If not found               |
|--------------------------------|-----------------|--------------|----------------------------|
| `find($pk)`                    | Primary key     | `?Entity`    | `null`                     |
| `findOrFail($pk)`              | Primary key     | `Entity`     | Throws `NotFoundException` |
| `findOrNew($pk, $defaults)`    | Primary key     | `Entity`     | New entity with defaults   |
| `findAll($pk, ...)`            | Primary key(s)  | `Collection` | Empty collection           |
| `get($offset)`                 | Result offset   | `?Entity`    | `null`                     |
| `getOrFail($offset)`           | Result offset   | `Entity`     | Throws `NotFoundException` |
| `getOrNew($offset, $defaults)` | Result offset   | `Entity`     | New entity with defaults   |
| `all()`                        | *(all results)* | `Collection` | Empty collection           |

### Find by primary key

```php
$entity = MyEntity::find(1);
```

Returns the entity or `null` if not found.

> ⚠️ **Warning**: Passing multiple IDs to `find()` is deprecated. Use `findAll()` instead for multiple entities.

---

### Find all

Returns a collection of entities matching the given primary key(s):

```php
$collection = MyEntity::findAll(1);
$collection = MyEntity::findAll(1, 2, 3);
```

### Find or fail

Throws `NotFoundException` if not found:

```php
use Hector\Orm\Exception\NotFoundException;

try {
    $entity = MyEntity::findOrFail(1);
} catch (NotFoundException $e) {
    // Handle exception
}
```

### Find or new

Returns existing entity or creates a new one with default values:

```php
$entity = MyEntity::findOrNew(1, ['foo' => 'value']);
```

## Get by offset (non-PK access)

These methods retrieve entities by their position in the result set (zero-based offset), not by primary key. Useful when
you need the Nth result of a query.

### Get / get or fail / get or new

```php
// Get first result (offset 0)
$entity = MyEntity::query()->get();

// Get third result (offset 2)
$entity = MyEntity::query()->where('active', true)->get(2);

// Throw NotFoundException if no result at offset
$entity = MyEntity::query()->getOrFail(0);

// Return new entity with default values if no result at offset
$entity = MyEntity::query()->getOrNew(0, ['status' => 'draft']);
```

---

## Retrieving all entities

```php
$collection = MyEntity::all();
```

Also available via the builder:

```php
$collection = MyEntity::query()->all();
```

## Chunking and yielding

Use `chunk()` for memory-friendly batch processing:

```php
MyEntity::query()->chunk(100, function (Collection $entities) {
    foreach ($entities as $entity) {
        // Process each entity
    }
});
```

With lazy mode disabled (eager fetch per chunk):

```php
MyEntity::query()->chunk(100, function (Collection $collection) {
    // Each chunk is fetched eagerly from DB
}, lazy: false);
```

Use `yield()` to iterate using a lazy generator. Entities are hydrated one by one as you iterate, minimizing memory
usage:

```php
foreach (MyEntity::query()->yield() as $entity) {
    // Each entity is loaded on-demand
}
```

> 💡 **Tip**: `yield()` returns a `LazyCollection`. The underlying query is executed once, but entities are hydrated
> lazily during iteration.

---

## Counting

```php
$count = MyEntity::query()->count();
```

This will ignore any `limit()` that was previously applied.

## Limiting and offsetting results

```php
MyEntity::query()->limit(10)->offset(5)->all();
```

## Ordering results

```php
MyEntity::query()->orderBy('created_at', 'DESC')->all();
MyEntity::query()->orderBy('name')->all(); // ASC by default
```

## Conditions

You can filter entities using a fluent API similar to the `QueryBuilder`.

### Where / Having

```php
MyEntity::query()->where('column', 'value')->orWhere('column', '<', 10);
```

### Where In / Not In

```php
MyEntity::query()->whereIn('column', ['foo', 'bar']);
MyEntity::query()->whereNotIn('column', ['foo', 'bar']);
```

### Between / Not Between

```php
MyEntity::query()->whereBetween('column', 1, 10);
MyEntity::query()->whereNotBetween('column', 1, 10);
```

### Greater / Less

```php
MyEntity::query()->whereGreaterThan('column', 5);
MyEntity::query()->whereLessThan('column', 100);
```

### Exists / Not Exists

```php
MyEntity::query()->whereExists($subQuery);
MyEntity::query()->whereNotExists($subQuery);
```

### Where equals (smart inference)

```php
MyEntity::query()->whereEquals([
    'status' => 'active',           // WHERE status = 'active'
    'category_id' => [1, 2, 3],     // AND category_id IN (1, 2, 3)
]);
```

Automatically uses `=` for scalar values and `IN` for arrays.

### Grouped conditions

Pass a `Closure` to `where()` or `having()` to create grouped (parenthesized) conditions:

```php
use Hector\Query\Statement\Conditions;

MyEntity::query()
    ->where(function (Conditions $conditions): void {
        $conditions->where('status', 'active');
        $conditions->orWhere('role', 'admin');
    })
    ->where('verified', true)
    ->all();
// WHERE ( status = ? OR role = ? ) AND verified = ?
```

> 💡 **Tip**: Type-hinting the closure parameter as `Conditions` enables full autocompletion in your IDE.

---

## Pagination

> 🆕 **Info**: *Since version 1.3*

The `Builder` provides a `paginate()` method that returns paginated entity collections. It integrates with
the [Pagination](../components/pagination.md) component.

### Basic usage

```php
use Hector\Pagination\Request\OffsetPaginationRequest;

$request = new OffsetPaginationRequest(page: 2, perPage: 15);

$pagination = User::query()
    ->where('active', true)
    ->orderBy('created_at', 'DESC')
    ->paginate($request);

// Iterate over User entities
foreach ($pagination as $user) {
    echo $user->name;
}

// Pagination metadata
$pagination->getCurrentPage();  // 2
$pagination->hasMore();         // true/false
```

### With total count

```php
$pagination = User::query()
    ->where('active', true)
    ->paginate($request, withTotal: true);

$pagination->getTotal();       // 150
$pagination->getTotalPages();  // 10
```

### Cursor pagination

For large datasets, cursor pagination is more efficient:

```php
use Hector\Pagination\Request\CursorPaginationRequest;

$request = new CursorPaginationRequest(
    perPage: 20,
    position: ['id' => 100],
);

$pagination = Post::query()
    ->orderBy('id')
    ->paginate($request);

foreach ($pagination as $post) {
    // Process Post entities
}

$pagination->getNextPosition();     // ['id' => 120]
$pagination->getPreviousPosition(); // ['id' => 100]
```

### Supported request types

| Request Type              | Returns            | Best For                        |
|---------------------------|--------------------|---------------------------------|
| `OffsetPaginationRequest` | `OffsetPagination` | Traditional page navigation     |
| `CursorPaginationRequest` | `CursorPagination` | Large datasets, infinite scroll |
| `RangePaginationRequest`  | `RangePagination`  | RFC 7233 style APIs             |

> 💡 **Tip**: Unlike the raw `QueryBuilder`, the ORM `Builder` returns hydrated entity collections, not raw arrays.

### Optimized pagination

When using JOINs (especially `*ToMany` relationships), a single row per entity can be multiplied into several rows. This
causes `LIMIT` to return fewer entities than expected. Pass `optimized: true` to fix this:

```php
$pagination = User::query()
    ->where('orders.status', 'paid')       // auto-joins "orders" (1-to-many)
    ->orderBy('created_at', 'DESC')
    ->paginate($request, optimized: true);
```

Under the hood, the paginator executes two queries:

1. `SELECT DISTINCT pk FROM … JOIN … WHERE … ORDER BY … LIMIT …` — fetches only the paginated primary key values
2. `SELECT * FROM … WHERE pk IN (…)` — loads full entities for those IDs

Results are automatically reordered to match the original ORDER BY.

This works with all three pagination types (offset, cursor, range) and is fully compatible with `withTotal: true`:

```php
$pagination = User::query()
    ->where('orders.status', 'paid')
    ->paginate($request, withTotal: true, optimized: true);
```

> ℹ️ **Note**: Optimized pagination requires the entity to have a primary key defined.

---

## Compatibility with QueryBuilder

All filtering, ordering, joining, and limiting operations are passed to the underlying `QueryBuilder`, which can be used
directly for low-level control.

If you need full SQL flexibility or want to decouple from ORM, refer to the
standalone [QueryBuilder documentation](../components/query.md).
