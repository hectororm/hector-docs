---
breadcrumb:
  - Components
  - Collection
summary-order: ;2
---

# 📂 Collection

> **Note**: While collection are part of the **Hector ORM** ecosystem, they are available as a standalone package:
> [`hectororm/collection`](https://github.com/hectororm/collection).
> You can find it on
> [Packagist](https://packagist.org/packages/hectororm/collection).
> You can use them independently of the ORM, in any PHP application. 🎉

Working with arrays in PHP is simple, but when it comes to chaining operations like filtering, mapping, or sorting, code
can quickly become verbose and harder to read. **Hector ORM** introduces a powerful abstraction: **collections** — iterable
data structures that combine the convenience of arrays with the expressiveness of modern fluent interfaces.

Collections are especially useful for transforming data retrieved from APIs, databases, or user input. They encourage a
functional and declarative approach to manipulating data, resulting in cleaner and more maintainable code.

Two types of collections are available:

* `Collection`: An eager collection that behaves much like a traditional PHP array. It evaluates and stores all its
  values in memory.
* `LazyCollection`: A lazy, generator-based collection built for handling large or potentially infinite datasets
  efficiently. Its values are generated on the fly, which makes it ideal when working with streams or when you want to
  avoid unnecessary memory usage.

## 🎓 Creating Collections

```php
use Hector\Collection\Collection;
use Hector\Collection\LazyCollection;

$collection = new Collection();
$collection = new Collection(['my', 'initial', 'array']);

$lazy = new LazyCollection();
lazy = new LazyCollection(['my', 'initial', 'array']);
```

All collections implement `CollectionInterface`. Only `Collection` is `Countable`.

## 🔍 `CollectionInterface` Methods

### `static::new(iterable|Closure $iterable): static`

Create a new instance of the current collection.

```php
$collection = Collection::new(['foo', 'bar']);
```

### `count(): int`

Count the number of items in the collection.

> For lazy collections, the entire iterator will be traversed.

```php
$collection = Collection::new(['foo', 'bar', 'baz']);
$collection->count(); // 3
count($collection); // 3
```

### `getArrayCopy(): array`

Return the collection as a plain array.

```php
$collection = new Collection();
$collection->getArrayCopy(); // []

$collection = new Collection(['my', 'initial', new Collection(['array'])]);
$collection->getArrayCopy(); // ['my', 'initial', ['array']]
```

### `all(): array`

Get all elements of the collection (preserves inner collections).

```php
$collection = new Collection();
$collection->all(); // []

$collection = new Collection(['my', 'initial', new Collection(['array'])]);
$collection->all(); // ['my', 'initial', new Collection(['array'])]
```

### `isEmpty(): bool`

Determine if the collection is empty.

```php
Collection::new(['foo', 'bar'])->isEmpty(); // false
Collection::new()->isEmpty(); // true
```

### `isList(): bool`

Check if the collection is a list (sequential integer keys).

```php
Collection::new(['foo', 'bar'])->isList(); // true
Collection::new(['foo', 'b' => 'bar'])->isList(); // false
```

### `collect(): self`

Collect all data into a new classic `Collection`. Useful for evaluating a `LazyCollection`.

```php
$lazy = LazyCollection::new(['foo', 'bar']);
$collection = $lazy->collect();
```

### `sort(callable|int|null $callback = null): static`

Sort items using optional callback or sorting flag.

```php
$collection = Collection::new(['foo', 'bar']);
$collection = $collection->sort();
$collection->getArrayCopy(); // ['bar', 'foo']
```

### `multiSort(callable ...$callbacks): static`

Perform multi-level sorting with multiple comparison callbacks.

```php
$collection = Collection::new([
    'l' => ['name' => 'Lemon', 'nb' => 1],
    'o' => ['name' => 'Orange', 'nb' => 1],
    'b1' => ['name' => 'Banana', 'nb' => 5],
    'b2' => ['name' => 'Banana', 'nb' => 1],
    'a1' => ['name' => 'Apple', 'nb' => 10],
    'a2' => ['name' => 'Apple', 'nb' => 1],
]);
$collection = $collection->multiSort(
    fn($a, $b) => $a['name'] <=> $b['name'],
    fn($a, $b) => $a['nb'] <=> $b['nb']
);
$collection->getArrayCopy();
// [
//   'a2' => ['name' => 'Apple', 'nb' => 1],
//   'a1' => ['name' => 'Apple', 'nb' => 10],
//   'b2' => ['name' => 'Banana', 'nb' => 1],
//   'b1' => ['name' => 'Banana', 'nb' => 5],
//   'l'  => ['name' => 'Lemon', 'nb' => 1],
//   'o'  => ['name' => 'Orange', 'nb' => 1]
// ]
```

### `filter(?callable $callback = null): static`

Filter items using a callback.

```php
$collection = Collection::new([1, 10, 20, 100]);
$filtered = $collection->filter(fn($v) => $v >= 20);
$filtered->getArrayCopy(); // [20, 100]
```

### `filterInstanceOf(string|object ...$classes): static`

Filter items that are instances of given class(es).

```php
$collection = Collection::new([new stdClass(), new SimpleXMLElement('<root/>')]);
$collection = $collection->filterInstanceOf(stdClass::class);
$collection->getArrayCopy(); // [object(stdClass)]
```

### `map(callable $callback): static`

Map callback to each item.

### `each(callable $callback): static`

Apply callback to each item and return original collection.

### `search(callable|mixed $needle, bool $strict = false): int|string|false`

Search for a value or use callback to find matching item.

```php
$collection = Collection::new(['foo', 'bar', '1', 1, 'quxx']);
$collection->search(1); // 2
$collection->search(1, true); // 3
$collection->search(fn($v) => str_starts_with($v, 'ba')); // 1
```

### `get(int $index = 0): mixed`

Get item at given index (negative indexes allowed).

```php
$collection = Collection::new(['foo', 'bar', 'baz']);
$collection->get();    // 'foo'
$collection->get(1);   // 'bar'
$collection->get(-1);  // 'baz'
```

### `first(?callable $callback = null): mixed`

Return the first item (optionally filtered).

```php
$collection = Collection::new(['foo', 'bar', 'baz']);
$collection->first(); // 'foo'
$collection->first(fn($v) => str_starts_with($v, 'ba')); // 'bar'
```

### `last(?callable $callback = null): mixed`

Return the last item (optionally filtered).

```php
$collection->last(); // 'baz'
$collection->last(fn($v) => str_starts_with($v, 'ba')); // 'baz'
```

### `slice(int $offset, ?int $length = null): static`

Get a slice of the collection.

```php
$collection = Collection::new(['foo', 'bar', 'baz']);
$collection->slice(0, 2)->getArrayCopy(); // ['foo', 'bar']
$collection->slice(-2, 2)->getArrayCopy(); // ['bar', 'baz']
```

### `contains(mixed $value, bool $strict = false): bool`

Check if collection contains value.

```php
$collection = Collection::new(['foo', 'bar', '2', 'baz']);
$collection->contains(2); // true
$collection->contains(2, true); // false
```

### `chunk(int $length): static`

Split collection into chunks.

```php
$collection = Collection::new(['foo', 'bar', 'baz']);
$chunks = $collection->chunk(2)->getArrayCopy();
// [['foo', 'bar'], ['baz']]
```

### `keys(): static`

Return all keys.

### `values(): static`

Return all values.

### `unique(): static`

Remove duplicate values.

```php
$collection = Collection::new(['k1' => 'foo', 1 => 'foo', 'bar', 'k2' => 'baz']);
$collection->unique()->getArrayCopy(); // ['k1' => 'foo', 'bar', 'k2' => 'baz']
```

### `flip(): static`

Flip keys and values.

### `reverse(bool $preserve_keys = false): static`

Reverse the order of items.

```php
$collection = Collection::new(['k1' => 'foo', 'foo', 'bar', 'k2' => 'baz']);
$collection->reverse()->getArrayCopy();
$collection->reverse(true)->getArrayCopy();
```

### `column(string|int|Closure|null $column_key, string|int|Closure|null $index_key = null): static`

Extract a column or reindex collection.

```php
$collection = Collection::new([
    ['k1' => 'foo', 'value' => 'foo value'],
    ['k1' => 'bar', 'value' => 'bar value'],
    ['k1' => 'baz', 'value' => 'baz value'],
]);
$collection->column('k1')->getArrayCopy();
$collection->column('value', 'k1')->getArrayCopy();
```

### `rand(int $length = 1): static`

Pick one or more random items.

### `sum(): int|float`

Sum of values.

### `avg(): int|float`

Average of values.

### `median(): int|float`

Median of values.

### `variance(): int|float`

Population variance.

### `deviation(): int|float`

Standard deviation.

### `reduce(callable $callback, mixed $initial = null): mixed`

Reduce to a single value.

```php
Collection::new([1, 2, 3])->reduce(fn($c, $i) => $c + $i, 10); // 16
```

## 🔍 `Collection` Additional Methods

### `append(mixed ...$values): static`

Append values to collection.

```php
Collection::new(['foo'])->append('bar')->getArrayCopy(); // ['foo', 'bar']
```

### `prepend(mixed ...$values): static`

Prepend values to collection.

```php
Collection::new(['foo'])->prepend('bar')->getArrayCopy(); // ['bar', 'foo']
```

### `lazy(): LazyCollection`

Convert to lazy collection.

## 🧵 `LazyCollection`

`LazyCollection` uses PHP generators to stream data efficiently, deferring evaluation until it's needed. Lazy
collections are ideal when working with large datasets or expensive operations.

> ⚠️ Once a lazy collection is consumed (e.g., after a `foreach`, `toArray()` or `count()`), it **cannot be reused**. To
> reuse its content, convert it to a standard collection using `collect()`.

### Create a LazyCollection

```php
use Hector\Collection\LazyCollection;

$lazy = new LazyCollection(function () {
    for ($i = 1; $i <= 5; $i++) {
        yield $i;
    }
});
```

### Apply lazy operations

```php
$filtered = $lazy->filter(fn($x) => $x % 2 === 0);
$mapped = $filtered->map(fn($x) => $x ** 2);

foreach ($mapped as $val) {
    echo $val . PHP_EOL; // 4, 16
}
```

### Convert between lazy and standard collections

```php
$standard = $mapped->collect();     // => Hector\Collection\Collection
$lazyAgain = $standard->lazy();     // => LazyCollection
```

### 💡 Common Use Cases

Lazy collections are especially useful for:

* **Large datasets**: Reading a huge file (e.g., CSV, logs) without loading everything into memory.

  ```php
  $lazyCsv = new LazyCollection(function () {
      $handle = fopen('data.csv', 'r');
      while (($row = fgetcsv($handle)) !== false) {
          yield $row;
      }
      fclose($handle);
  });

  $names = $lazyCsv->map(fn($row) => $row[1]);
  ```

* **Streaming data from APIs**:

  ```php
  $apiResults = new LazyCollection(function () {
      for ($page = 1; $page <= 100; $page++) {
          $data = fetchApiPage($page);
          foreach ($data as $item) {
              yield $item;
          }
      }
  });
  ```

* **Applying expensive transformations only when needed**:

  ```php
  $users = new LazyCollection($dbRows);
  $emails = $users->filter(fn($u) => $u['active'])->map(fn($u) => strtolower($u['email']));
  ```

### Limitations and Caveats

* A `LazyCollection` is **not reusable** once iterated. If you need to use the same data twice, call `collect()` first.
* Side effects in callbacks (`map`, `filter`, etc.) are only triggered when actually iterated.
* `isEmpty()`, `getArrayCopy()`, `count()` will **consume** the generator.
* Lazy collections are mostly **read-only**: you can't push new elements after creation.
* Not all methods are optimized for laziness; some require full materialization internally (e.g. `reverse()`,
  `unique()`).

---

## 💡 Usage Examples

### Filtering user input

```php
use Hector\Collection\Collection;

$input = Collection::new($_POST);
$filtered = $input->filter(fn($value) => $value !== '');
```

### Transforming API results

```php
$response = Collection::new($api->fetchUsers());
$usernames = $response->map(fn($user) => $user['username']);
```

### Sorting a list of products by price

```php
$products = Collection::new($repository->all());
$sorted = $products->sort(fn($a, $b) => $a['price'] <=> $b['price']);
```

### Extracting specific values

```php
$orders = Collection::new($ordersData);
$orderIds = $orders->column('id');
```

### Using LazyCollection for large data sets

```php
use Hector\Collection\LazyCollection;

$lazy = new LazyCollection(function () {
    foreach (range(1, 1000000) as $i) {
        yield $i;
    }
});

$even = $lazy->filter(fn($n) => $n % 2 === 0)->collect();
```

### Aggregating numerical data

Suppose you have a list of invoices and want to compute the total and average amount:

```php
$invoices = Collection::new($invoicingService->all());
$total = $invoices->column('amount')->sum();
$average = $invoices->column('amount')->avg();
```
