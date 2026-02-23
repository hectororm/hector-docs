---
breadcrumb:
  - ORM
  - Builder
summary-order: ;3
---

# 🔧 Builder

The `Builder` provides a high-level, entity-oriented way to perform queries on your data models. It is built on top of
the lower-level `QueryBuilder` and integrates deeply with **Hector ORM** entity management.

## 🔍 Accessing the Builder

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

## 🎯 Finding Entities

### Find by Primary Key

```php
$entity = MyEntity::find(1);
```

Returns the entity or `null` if not found.

> ⚠️ **Warning**: Passing multiple IDs to `find()` is deprecated. Use `findAll()` instead for multiple entities.

---

### Find All

Returns a collection of entities matching the given primary key(s):

```php
$collection = MyEntity::findAll(1);
$collection = MyEntity::findAll(1, 2, 3);
```

### Find or Fail

Throws `NotFoundException` if not found:

```php
use Hector\Orm\Exception\NotFoundException;

try {
    $entity = MyEntity::findOrFail(1);
} catch (NotFoundException $e) {
    // Handle exception
}
```

### Find or New

Returns existing entity or creates a new one with default values:

```php
$entity = MyEntity::findOrNew(1, ['foo' => 'value']);
```

## 📊 Get by Offset (non-PK access)

These methods retrieve entities by their position in the result set (zero-based offset), not by primary key. Useful when you need the Nth result of a query.

### Get / Get or Fail / Get or New

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

## 📋 Retrieving All Entities

```php
$collection = MyEntity::all();
```

Also available via the builder:

```php
$collection = MyEntity::query()->all();
```

## 🔄 Chunking and Yielding

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

Use `yield()` to iterate using a lazy generator. Entities are hydrated one by one as you iterate, minimizing memory usage:

```php
foreach (MyEntity::query()->yield() as $entity) {
    // Each entity is loaded on-demand
}
```

> 💡 **Tip**: `yield()` returns a `LazyCollection`. The underlying query is executed once, but entities are hydrated lazily during iteration.

---

## 🔢 Counting

```php
$count = MyEntity::query()->count();
```

This will ignore any `limit()` that was previously applied.

## 📦 Limiting and Offsetting Results

```php
MyEntity::query()->limit(10)->offset(5)->all();
```

## 🔢 Ordering Results

```php
MyEntity::query()->orderBy('created_at', 'DESC')->all();
MyEntity::query()->orderBy('name')->all(); // ASC by default
```

## 🧮 Conditions

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

### Where Equals (Smart Inference)

```php
MyEntity::query()->whereEquals([
    'status' => 'active',           // WHERE status = 'active'
    'category_id' => [1, 2, 3],     // AND category_id IN (1, 2, 3)
]);
```

Automatically uses `=` for scalar values and `IN` for arrays.

### Grouped Conditions

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

## 📄 Pagination

> 🆕 **Info**: *Since version 1.3*

The `Builder` provides a `paginate()` method that returns paginated entity collections. It integrates with the [Pagination](../components/pagination.md) component.

### Basic Usage

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

### With Total Count

```php
$pagination = User::query()
    ->where('active', true)
    ->paginate($request, withTotal: true);

$pagination->getTotal();       // 150
$pagination->getTotalPages();  // 10
```

### Cursor Pagination

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

### Supported Request Types

| Request Type                | Returns              | Best For                    |
|-----------------------------|----------------------|-----------------------------|
| `OffsetPaginationRequest`   | `OffsetPagination`   | Traditional page navigation |
| `CursorPaginationRequest`   | `CursorPagination`   | Large datasets, infinite scroll |
| `RangePaginationRequest`    | `RangePagination`    | RFC 7233 style APIs         |

> 💡 **Tip**: Unlike the raw `QueryBuilder`, the ORM `Builder` returns hydrated entity collections, not raw arrays.

---

## 🔗 Compatibility with QueryBuilder

All filtering, ordering, joining, and limiting operations are passed to the underlying `QueryBuilder`, which can be used
directly for low-level control.

If you need full SQL flexibility or want to decouple from ORM, refer to the
standalone [QueryBuilder documentation](../components/query.md).
