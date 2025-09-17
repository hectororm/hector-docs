---
breadcrumb:
  - ORM
  - Builder
summary-order: ;3
---

# Builder

The `Builder` provides a high-level, entity-oriented way to perform queries on your data models. It is built on top of
the lower-level `QueryBuilder` and integrates deeply with **Hector ORM** entity management.

## Accessing the Builder

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

## Finding Entities

### Find by Primary Key

```php
$entity = MyEntity::find(1);
$collection = MyEntity::find(1, 2);
```

If the entity does not exist, `null` is returned.

### Find All

Returns a collection even for a single ID:

```php
$collection = MyEntity::findAll(1);
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

## Get by Offset (non-PK access)

### Get / Get or Fail / Get or New

Same behavior as `find*` methods, but works on offset instead of primary key:

```php
$entity = MyEntity::get(1);
$entity = MyEntity::getOrNew(1, ['foo' => 'value']);
```

## Retrieving All Entities

```php
$collection = MyEntity::all();
```

Also available via the builder:

```php
$collection = MyEntity::query()->all();
```

## Chunking and Yielding

Use `chunk()` for memory-friendly batch processing:

```php
MyEntity::query()->chunk(100, function (Collection $entities) {
    // ...
});
```

With lazy mode disabled:

```php
MyEntity::query()->chunk(100, function ($collection) {
    // ...
}, lazy: false);
```

Use `yield()` to iterate using a generator:

```php
foreach (MyEntity::query()->yield() as $entity) {
    // ...
}
```

## Counting

```php
$count = MyEntity::query()->count();
```

This will ignore any `limit()` that was previously applied.

## Limiting and Offsetting Results

```php
MyEntity::query()->limit(10)->offset(5)->all();
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

### Where Equals (Smart Inference)

```php
MyEntity::query()->whereEquals([
    'column1' => 'value',
    'column2' => ['a', 'b'], // auto uses whereIn
]);
```

## Compatibility with QueryBuilder

All filtering, ordering, joining, and limiting operations are passed to the underlying `QueryBuilder`, which can be used
directly for low-level control.

If you need full SQL flexibility or want to decouple from ORM, refer to the
standalone [QueryBuilder documentation](../components/query.md).
