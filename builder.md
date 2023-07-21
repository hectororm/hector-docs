```index
breadcrumb: Builder
summary-order: 3
```

# Builder

To do simple queries on your entities, the builder is accessible with static methods.

To access to the builder, call the static method `Entity::query()` or creates new builder object `new Hector\Orm\Query\Builder(MyEntity::class)`. 

Entity builder is based on QueryBuilder of library `hectororm/query`. 

## Find by primary key

On "find" methods, you can pass multiple primary keys to get one entity or a collection of entities. If your entity has
more than one column in primary key, pass an associative array to the argument.

### Find

Pass the primary key to the first argument to get the entity. If not exists, returns null.

Note: methods `Entity::find*()` are shortcuts of `Entity::query()->find*()`.

```php
$entity = MyEntity::find(1);
$collection = MyEntity::find(1, 2);
```

### Find all

It's same comportment than `Entity::find()` method but in all cases, a collection is returned.

```php
$collection = MyEntity::findAll(1);
```

### Find or fail

It's same comportment than `Entity::find()` method but if entity does not exist, an
exception `Hector\Orm\Exception\NotFoundException` is thrown.

```php
try {
    $entity = MyEntity::findOrFail(1);
} catch (Hector\Orm\Exception\NotFoundException $exception) {
    // ... Do something
}
```

### Find or new

It's same comportment than `Entity::find()` method but if entity does not exist, a new entity is created with passed
data in second argument.

```php
$entity = MyEntity::findOrNew(1, ['foo' => 'value', 'bar' => 'value']);
print $entity->foo; // Print the default value 'value' if entity does not exist 
```

## Get by offset

### Get

Pass the offset to the first argument to get the entity. If not exists, returns null.

Note: methods `Entity::get*()` are shortcuts of `Entity::query()->get*()`.

```php
$entity = MyEntity::get(1);
$collection = MyEntity::get(1, 2);
```

### Get or fail

It's same comportment than `Entity::get()` method but if entity does not exist, an
exception `Hector\Orm\Exception\NotFoundException` is thrown.

```php
try {
    $entity = MyEntity::get(1);
} catch (Hector\Orm\Exception\NotFoundException $exception) {
    // ... Do something
}
```

### Get or new

It's same comportment than `Entity::get()` method but if entity does not exist, a new entity is created with passed data
in second argument.

```php
$entity = MyEntity::getOrNew(1, ['foo' => 'value', 'bar' => 'value']);
print $entity->foo; // Print the default value 'value' if entity does not exist 
```

## All

You can retrieve all entities in a collection with method `Entity::all()`.

```php
$collection = MyEntity::all();
$collection = MyEntity::query()->all();
```

### Chunk

In certain case, it's not judicious to retrieve all entities in one time, for memory consumption for example. So it's
possible to chunk in X results with method `Entity::chunk()`.

*It's a good practice to not retrieve all entities in one time and have a high memory consumption.*

```php
use Hector\Orm\Collection\Collection;

$collection = MyEntity::chunk(
    100,
    function(Collection $collection) {
        // ...
    }
);
$collection = MyEntity::query()->chunk(
    100,
    function(Collection $collection) {
        // ...
    }
);
```

### Yield

You can retrieve entities with a [Generator](https://www.php.net/manual/class.generator.php).

*It's a good practice to not retrieve all entities in one time and have a high memory consumption.*

```php
$generator = MyEntity::yield();
$generator = MyEntity::query()->yield();

foreach ($generator as $entity) {
    // ...
}
```

### Count

In some case, you need to retrieve the total number of results, `Entity::count()` helps you for that.

```php
$count = MyEntity::count();
$count = MyEntity::query()->count();
```

This method works also with conditions and remove limit restrictions:

```php
$query = MyEntity::query()->limit(10);

$count = $query->count(); // Result 100 (in case of 100 rows in table)
$collection = $query->all(); // Collection of 10 entities
```

### Limit

Results can be limited with `limit` method of builder.

```php
$collection = MyEntity::query()->limit(10)->all(); // Limit collection to 10 results
$collection = MyEntity::query()->limit(10, 5)->all(); // Limit collection to 10 results at offset 5
$collection = MyEntity::query()->offset(5)->all(); // Limit collection to all results at offset 5 and more
```


## Conditions

`where*` methods are also available with `having*` ; where conditions are done before results return and having are done on the result list, so the second has bad performances.

### Where

Example of usage:

```php
$builder = MyEntity::query();
$builder
    ->where('column_name', '=', $value)
    ->where('column_name', '<>', $value)
    ->where('column_name', '>=', $value)
    ->where('column_name', '>', $value)
    ->where('column_name', '<=', $value)
    ->where('column_name', '<', $value)
    ->where('column_name', $value)
    ->andWhere('column_name', $value)
    ->orWhere('column_name', $value);
````

If operator is not given, it's the equals used.

### Where in / Where not in

Builder allow doing `IN` condition on an array of value:

```php
MyEntity::query()->whereIn('column_name', ['foo', 'bar']);
````

The "not" method is also available:

```php
MyEntity::query()->whereNotIn('column_name', ['foo', 'bar']);
```

### Between / Not between

Do a condition on column value between 2 parameters values.

```php
MyEntity::query()->whereBetween('column_name', $value1, $value2);
```

The "not" method is also available:

```php
MyEntity::query()->whereNotBetween('column_name', $value1, $value2);
```

### Greater than / Greater than or equal

Do a condition on column value greater than parameter value.

```php
MyEntity::query()->whereGreaterThan('column_name', $value);
```

The method greater than or equal is also available:

```php
MyEntity::query()->whereGreaterThanOrEqual('column_name', $value);
```

### Less than / Less than or equal

Do a condition on column value less than parameter value.

```php
MyEntity::query()->whereLessThan('column_name', $value);
```

The method less than or equal is also available:

```php
MyEntity::query()->whereLessThanOrEqual('column_name', $value);
```

### Exists / Not exists

Do a condition `EXISTS` with sub statement:

```php
MyEntity::query()->whereExists($statement);
```

The "not" method is also available:

```php
MyEntity::query()->whereNotExists($statement);
```

### Where equals

Method `whereEquals` accept an array in parameter and guess the operator to used.

```php
MyEntity::query()->whereEquals([
    'column1_name' => 'value', // Use equals comparator
    'column2_name' => ['foo', 'bar'], // Use where in
    'FUNCTION(column3_name)', // Use raw
]);
```
