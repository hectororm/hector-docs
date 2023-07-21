```index
breadcrumb: Entities; Collection
summary-order: ;3
```

# Collection

A collection is a set of entities.

Default collection class: `Hector\Orm\Collection\Collection`.

## Personalized collection

You can personalize a collection for a specific entity. Uses attribute `Hector\Orm\Attributes\Collection` to do define
the collection class.

```php
use Hector\Orm\Attributes as Orm;
use Hector\Orm\Collection\Collection;
use Hector\Orm\Entity\MagicEntity;

#[Orm\Collection(MyEntityCollection::class)]
class MyEntity extends MagicEntity{
}

class MyEntityCollection extends Collection {
}
```

## Edge loading

You can load specified foreign values of entities in one time with `Collection::load()` method.

```php
// Do optimized SQL queries to retrieve foreigners
$collection->load(['foreign' => ['sub_foreign']]);

foreach($collection as $entity) {
    print $entity->foreign; // Do not SQL query
}
```


## Refresh

In some cases, you need to refresh entities from collection with fresh data from database, `Collection::refresh()` do
that with optimized requests.

```php
$entity = $collection->first();
print $entity->field; // Output: 'bar'

$entity->field = 'foo';
print $entity->field; // Output: 'foo'

$collection->refresh();
print $entity->field; // Output: 'bar'
```


## Save

You can save all entities modifications into database with method `Collection::save()`.


## Delete

You can delete all entities from database with method `Collection::delete()`.


## Json / Array

`Hector\Orm\Collection\Collection` class is a sub class
of [`ArrayObject`](https://www.php.net/manual/class.arrayobject.php). So you can use methods of this parent class.

By default, the JSON result of serialization of a collection, is an array of entities. It's possible to extends the
collection to define a personalized comportment.

To retrieve an array of the entities of collection, use the method: `Collection::getArrayCopy()`.


## Helpers

### Filter

Collection allows filter entities with method `Collection::filter(callable $callback): Generator`, the result of method
is a [Generator](https://www.php.net/manual/class.generator.php).

```php
$generator = $collection->filter(fn(Entity $entity) => $entity->isExpired());

foreach ($generator as $entity) {
    // ...
}
```

### Search

Collection allows to search an entity with method `Collection::search(callable $callback): ?Entity`.

```php
$entity = $collection->search(fn(Entity $entity) => !$entity->isExpired());

if (null !== $entity) {
    // ...
}
```

### Map

Collection allows to apply a function on each entity with method `Collection::map(callable $callback): void`.

```php
$collection->map(fn(Entity $entity) => $entity->setViewed(true));
```

### First

Collection allows to retrieve the first entity with method `Collection::first(): ?Entity`.

```php
if (null !== ($entity = $collection->first())) {
    // ...
}
```

### Contains

You can test if a collection contains an entity object with method `Collection::contains(Entity $entity): bool`. The
comparison is strict, so the comparison two entities whose represent the same row is false.
