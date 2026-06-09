---
breadcrumb:
  - Components
  - Query Builder
summary-order: 3;1
keywords:
  - query-builder
  - select
  - insert
  - update
  - delete
  - where
  - join
  - sort
---

# 🔨 Query

> ℹ️ **Note**: While the Query Builder component is part of the **Hector ORM** ecosystem, it is available as a
> standalone package: [`hectororm/query`](https://github.com/hectororm/query).
> You can find it on [Packagist](https://packagist.org/packages/hectororm/query).
> You can use it independently of the ORM, in any PHP application. 🎉

The `QueryBuilder` in **Hector ORM** provides a fluent, object-oriented API to construct and execute SQL queries. It is
designed to streamline the building of `SELECT`, `INSERT`, `UPDATE`, `DELETE`, and `UNION` queries, while maintaining
control and readability in your codebase.

## Initialization

You can initialize the `QueryBuilder` using a `Connection` object. This setup allows you to immediately start crafting
your queries.

```php
use Hector\Connection\Connection;
use Hector\Query\QueryBuilder;

$connection = new Connection('...');
$queryBuilder = new QueryBuilder($connection);

$result = $queryBuilder
    ->select('table')
    ->where('field1', 'foo')
    ->where('field2', '>=', 2)
    ->fetchAll();
```

## Query types

The library provides specific classes for each query type. Each of these classes implements the `StatementInterface`,
enabling you to manually build the query and bind parameters.

* `Hector\Query\Select`
* `Hector\Query\Insert`
* `Hector\Query\Update`
* `Hector\Query\Delete`
* `Hector\Query\Union`

### Specifying the table

Use `from()` to define the target table for your query:

```php
$queryBuilder->from('users');
// SELECT * FROM users

$queryBuilder->from('users', 'u');
// SELECT * FROM users AS u

// Multiple tables (implicit join)
$queryBuilder
    ->from('users', 'u')
    ->from('profiles', 'p')
    ->where('u.id', '=', 'p.user_id');
```

### Example

```php
use Hector\Connection\Bind\BindParamList;
use Hector\Connection\Connection;
use Hector\Query\Select;

$select = new Select();
$select
    ->from('table')
    ->where('field', 'value');

$binds = new BindParamList();
$statement = $select->getStatement($binds);

$connection = new Connection('...');
$result = $connection->fetchAll($statement, $binds);
```

---

## Conditions

Both `WHERE` and `HAVING` clauses are supported using the same API. Simply switch the method prefix.

### Where / Having

Two-argument form: column, value (implicit "=" operator):

```php
use Hector\Query\QueryBuilder;

$queryBuilder
    ->from('table', 'alias')
    ->where('field', 'value');
```

Three-argument form (column, operator, value) :

```php
use Hector\Query\QueryBuilder;

$queryBuilder
    ->from('table', 'alias')
    ->where('field', '=', 'value')
    ->orWhere('field', '>=', 10);
```

### Grouped conditions

You can pass a `Closure` to `where()` or `having()` to create grouped conditions wrapped in parentheses. The closure
receives a `Conditions` object (`Hector\Query\Statement\Conditions`) as its first argument, which exposes the same
`where()` / `having()` methods:

```php
use Hector\Query\Statement\Conditions;

$queryBuilder
    ->from('users')
    ->where(function (Conditions $conditions): void {
        $conditions->where('age', '>=', 18);
        $conditions->orWhere('role', 'admin');
    })
    ->where('active', true);
// WHERE ( age >= ? OR role = ? ) AND active = ?
```

This is useful for combining `OR` conditions without affecting the rest of the query:

```php
$queryBuilder
    ->from('products')
    ->where('in_stock', true)
    ->where(function (Conditions $conditions): void {
        $conditions->where('category', 'electronics');
        $conditions->orWhere('price', '<', 10);
    });
// WHERE in_stock = ? AND ( category = ? OR price < ? )
```

The same pattern works with `orWhere()`, `having()`, and `orHaving()`:

```php
$queryBuilder
    ->from('orders')
    ->column('customer_id')
    ->column('SUM(amount)', 'total')
    ->groupBy('customer_id')
    ->having(function (Conditions $conditions): void {
        $conditions->having('total', '>', 1000);
        $conditions->orHaving('total', '<', 100);
    });
// HAVING ( total > ? OR total < ? )
```

> 💡 **Tip**: Type-hinting the closure parameter as `Conditions` enables full autocompletion in your IDE.

### Condition shortcuts

Several convenience methods are provided:

```php
$queryBuilder->whereIn('id', [1, 2, 3]);
$queryBuilder->whereNotBetween('age', 18, 30);
$queryBuilder->whereGreaterThan('score', 50);
$queryBuilder->whereExists(new Select(...));
$queryBuilder->whereContains('name', 'john');
```

Full list:

* `whereIn($column, array $values)`
* `whereNotIn($column, array $values)`
* `whereBetween($column, $value1, $value2)`
* `whereNotBetween($column, $value1, $value2)`
* `whereGreaterThan($column, $value)`
* `whereGreaterThanOrEqual($column, $value)`
* `whereLessThan($column, $value)`
* `whereLessThanOrEqual($column, $value)`
* `whereExists($statement)`
* `whereNotExists($statement)`
* `whereContains($column, $string)`
* `whereStartsWith($column, $string)`
* `whereEndsWith($column, $string)`

All `where*` methods have their `having*` counterparts for filtering grouped results:

```php
$queryBuilder
    ->from('orders')
    ->column('customer_id')
    ->column('SUM(amount)', 'total')
    ->groupBy('customer_id')
    ->having('total', '>', 1000)
    ->orHaving('total', '<', 100);
```

Available:

* `having()`
* `orHaving()`
* `havingIn()`
* `havingNotIn()`
* `havingBetween()`
* `havingNotBetween()`,
* `havingGreaterThan()`
* `havingGreaterThanOrEqual()`
* `havingLessThan()`
* `havingLessThanOrEqual()`,
* `havingExists()`
* `havingNotExists()`
* `havingContains()`
* `havingStartsWith()`
* `havingEndsWith()`.

---

## Selecting columns

You can customize the columns returned by your query:

```php
$queryBuilder
    ->from('users')
    ->column('name')
    ->column('email', 'user_email');
// SELECT name, email AS user_email FROM users

$queryBuilder->resetColumns();
$queryBuilder->columns(['id', 'name', 'created_at']);
// SELECT id, name, created_at FROM users
```

## Grouping results

To group query results:

```php
$queryBuilder
    ->from('products')
    ->column('category_id')
    ->column('COUNT(*)', 'count')
    ->groupBy('category_id')
    ->groupBy('status');
// SELECT category_id, COUNT(*) AS count FROM products GROUP BY category_id, status

$queryBuilder->resetGroups();
$queryBuilder->groupByWithRollup();
// Adds WITH ROLLUP modifier (MySQL)
```

## Ordering results

Sort your result set with `orderBy()`:

```php
$queryBuilder
    ->from('posts')
    ->orderBy('created_at', 'DESC')
    ->orderBy('name', 'ASC');
// SELECT * FROM posts ORDER BY created_at DESC, name ASC

$queryBuilder->resetOrder();
$queryBuilder->random();
// SELECT * FROM posts ORDER BY RAND()
```

## Sorting

> 🆕 **Info**: *Since version 1.3*

The `Sort` namespace provides type-safe, composable sorting objects. This is especially useful when sort parameters come
from user input (e.g. `?sort=title:desc`) and need to be validated before being applied to a query.

### Sort objects

The `SortInterface` defines a single method: `apply(QueryBuilder $builder): void`. Two implementations are provided:

```php
use Hector\Query\Sort\Sort;
use Hector\Query\Sort\MultiSort;

// Single sort
$sort = new Sort('title', 'ASC');
$sort->apply($builder);

// Multiple sorts
$sort = new MultiSort(
    new Sort('title', 'ASC'),
    new Sort('id', 'DESC'),
);
$sort->apply($builder);
```

The `QueryBuilder` also provides a fluent shortcut:

```php
$builder->applySort($sort);
```

> ℹ️ `applySort()` is **additive**: it appends the sort to the existing order. To replace the existing order, call
`resetOrder()` first:
> ```php
> $builder->resetOrder()->applySort($sort);
> ```

### SortConfig

`SortConfig` parses and validates sort parameters from HTTP query strings. It lets you control which columns are
sortable, map public parameter names to real column names, and optionally lock sort directions.

```php
use Hector\Query\Sort\SortConfig;

$sortConfig = new SortConfig(
    allowed: ['title', 'created_at', 'id'],
    default: ['title'],
);

// Resolve from query params: ?sort=created_at:desc
$sort = $sortConfig->resolve($request->getQueryParams());
$builder->resetOrder()->applySort($sort);
```

#### Constructor parameters

| Parameter    | Type     | Default  | Description                                                    |
|--------------|----------|----------|----------------------------------------------------------------|
| `allowed`    | `array`  | required | Allowed sort columns (see [Allowed formats](#allowed-formats)) |
| `default`    | `array`  | required | Default sort in `column:dir` format (e.g. `['title:desc']`)    |
| `defaultDir` | `string` | `ASC`    | Direction used when not specified by the user or the mapping   |
| `sortParam`  | `string` | `sort`   | Query parameter name to read from                              |

#### Allowed formats

The `allowed` parameter accepts several formats:

```php
// Simple: column name as-is, user can choose direction
$config = new SortConfig(
    allowed: ['title', 'created_at', 'id'],
    default: ['title'],
);

// Column mapping: expose a public name, map to real column
$config = new SortConfig(
    allowed: ['name' => 'user_name', 'date' => 'created_at'],
    default: ['name'],
);

// Locked direction: the mapping includes :dir, user cannot override it
$config = new SortConfig(
    allowed: ['status' => 'create_time:desc'],
    default: ['status'],
);

// Multi-column mapping: one public name maps to multiple ORDER BY columns
$config = new SortConfig(
    allowed: ['status' => ['status', 'create_time:desc']],
    default: ['status'],
);
```

#### Direction control: free vs. locked

The direction is determined by the **presence of `:dir` in the mapping value**:

| Mapping value              | Direction  | Description                                                                          |
|----------------------------|------------|--------------------------------------------------------------------------------------|
| `'column'` (no dir)        | **Free**   | User chooses with `?sort=col:asc` or `:desc`. Falls back to `defaultDir` if omitted. |
| `'column:desc'` (with dir) | **Locked** | Direction is fixed. User's requested direction is ignored.                           |

For multi-column mappings, each column follows its own rule:

```php
$config = new SortConfig(
    allowed: [
        'create_time' => 'review_id',                     // free direction
        'status' => ['status', 'create_time:desc'],        // status=free, create_time=locked DESC
    ],
    default: ['create_time:desc'],
    defaultDir: 'asc',
);
```

| User request            | Generated ORDER BY                       | Explanation                                    |
|-------------------------|------------------------------------------|------------------------------------------------|
| `?sort=create_time:asc` | `ORDER BY review_id ASC`                 | Free: user direction applied                   |
| `?sort=create_time`     | `ORDER BY review_id ASC`                 | Free: `defaultDir` (asc) applied               |
| `?sort=status:asc`      | `ORDER BY status ASC, create_time DESC`  | status=free (asc), create_time=locked (desc)   |
| `?sort=status:desc`     | `ORDER BY status DESC, create_time DESC` | status=free (desc), create_time=locked (desc)  |
| `?sort=status`          | `ORDER BY status ASC, create_time DESC`  | status=free (`defaultDir`), create_time=locked |
| `?sort=unknown`         | `ORDER BY review_id DESC`                | Invalid: falls back to default                 |
| *(no param)*            | `ORDER BY review_id DESC`                | No param: falls back to default                |

#### Default sort format

Default sort items use the `column:dir` string format. Each item must match a key in `allowed`:

```php
// Single default
$config = new SortConfig(allowed: ['title', 'id'], default: ['title']);

// With explicit direction
$config = new SortConfig(allowed: ['title', 'id'], default: ['title:desc']);

// Multi-column default
$config = new SortConfig(allowed: ['title', 'id'], default: ['title', 'id:desc']);
```

#### Supported URL formats

- Single: `?sort=title:asc`
- Multiple: `?sort[]=title:asc&sort[]=id:desc`
- Without direction (uses `defaultDir`): `?sort=title`

### Custom sort implementations

Implement `SortInterface` for custom sorting logic:

```php
use Hector\Query\Sort\SortInterface;
use Hector\Query\QueryBuilder;

class RandomSort implements SortInterface
{
    public function apply(QueryBuilder $builder): void
    {
        $builder->orderBy('RAND()');
    }
}

class NullsLastSort implements SortInterface
{
    public function __construct(
        private string $column,
        private string $dir = 'ASC',
    ) {}

    public function apply(QueryBuilder $builder): void
    {
        $builder->orderBy(sprintf('%s IS NULL', $this->column), 'ASC');
        $builder->orderBy($this->column, $this->dir);
    }
}
```

## Limiting results

Control pagination using `limit()` and `offset()`:

```php
$queryBuilder->limit(10);
$queryBuilder->offset(20);
$queryBuilder->resetLimit();
```

## Assigning values

When building `INSERT` or `UPDATE` queries, use:

```php
$queryBuilder
    ->assign('name', 'Alice')
    ->assign('email', 'alice@example.com');

$queryBuilder->resetAssignments();
```

Or in bulk:

```php
$queryBuilder->assigns([
    'name' => 'Alice',
    'email' => 'alice@example.com'
]);
```

You can also pass a `StatementInterface` to `assigns()` for more advanced use cases.

## Joins

Join tables with the following methods:

```php
$queryBuilder->innerJoin('users', 'users.id = posts.user_id', 'u');
$queryBuilder->leftJoin('categories', 'categories.id = posts.category_id');
$queryBuilder->rightJoin('tags', 'tags.id = posts.tag_id');

$queryBuilder->resetJoin();
```

---

## Unions

The `Union` class allows combining multiple `SELECT` queries:

```php
use Hector\Query\Select;
use Hector\Query\Union;

$select1 = (new Select())->from('customers')->column('name');
$select2 = (new Select())->from('suppliers')->column('name');

$union = new Union();
$union->addSelect($select1, $select2);
```

`Union` also implements `StatementInterface`, allowing you to bind and execute it like any other query.

### Executing a union

```php
use Hector\Connection\Bind\BindParamList;

$binds = new BindParamList();
$statement = $union->getStatement($binds);

// Execute via connection
foreach ($connection->fetchAll($statement, $binds) as $row) {
    echo $row['name'];
}
```

### Union All (with duplicates)

```php
$union = new Union();
$union->addSelect($select1, $select2);
$union->all(); // UNION ALL instead of UNION
```

---

## Fetching results

Use the following methods to retrieve data:

```php
$queryBuilder->fetchOne();      // ?array - first row
$queryBuilder->fetchAll();      // Generator - all rows
$queryBuilder->fetchColumn();   // Generator - specific column
```

> 💡 **Tip**: `fetchAll()` and `fetchColumn()` return a `Generator`. Refer to PHP
> documentation: [https://www.php.net/manual/en/class.generator.php](https://www.php.net/manual/en/class.generator.php)

## Counting results

Quickly count the results of a query:

```php
$count = $queryBuilder
    ->from('table')
    ->where('status', 'active')
    ->count();

$rows = $queryBuilder->fetchAll();
```

This method resets column selection, limit, and order, but does not mutate the original query builder.

## Distinct values

Use `distinct()` to eliminate duplicates:

```php
$queryBuilder
    ->from('users')
    ->distinct()
    ->fetchAll();
```

## Existence check

Determine if any row matches the conditions:

```php
$exists = $queryBuilder
    ->from('users')
    ->where('email', 'alice@example.com')
    ->exists();
```

Does not alter the query builder instance.

---

## Shortcuts for Insert / Update / Delete

You can execute data manipulation directly:

```php
$queryBuilder->insert([
    'name' => 'Alice',
    'email' => 'alice@example.com'
]);

$queryBuilder->update([
    'status' => 'inactive'
]);

$queryBuilder->delete();
```

You may also use a subquery for insert:

```php
$queryBuilder->insert((new Select())->from('source_table'));
```

### Ignoring duplicates

Use `ignore()` to skip rows that would cause duplicate key violations:

```php
$queryBuilder
    ->from('users')
    ->ignore()
    ->insert([
        'email' => 'alice@example.com',
        'name' => 'Alice'
    ]);
// INSERT IGNORE INTO users ...
```

These shortcut methods do not affect the `QueryBuilder` instance, so it remains reusable for further operations.

---

## Locking rows

Use the `$lock` parameter on fetch methods to acquire a `FOR UPDATE` lock on selected rows. This is useful for
preventing concurrent modifications in transactional contexts.

```php
$connection->beginTransaction();

// Lock the row for update
$user = $queryBuilder
    ->from('users')
    ->where('id', 1)
    ->fetchOne(lock: true);

// Modify and save
$queryBuilder->from('users')->where('id', 1)->update(['balance' => $user['balance'] - 100]);

$connection->commit();
```

Available on:

* `fetchOne(bool $lock = false)`
* `fetchAll(bool $lock = false)`
* `fetchColumn(int $column = 0, bool $lock = false)`

> ⚠️ **Warning**: Locking requires an active transaction. The lock is released when the transaction is committed or
> rolled back.

> 💡 **Tip**: On databases that support it (MySQL 8+, PostgreSQL), `SKIP LOCKED` is automatically added to avoid blocking
> on already-locked rows.

---

## Integrated pagination

> 🆕 **Info**: *Since version 1.3*

The `QueryBuilder` provides a `paginate()` method that integrates directly with the [Pagination](pagination.md)
component. It automatically handles limit/offset and returns a pagination object.

```php
use Hector\Pagination\Request\OffsetPaginationRequest;

$request = new OffsetPaginationRequest(page: 3, perPage: 20);

$pagination = $queryBuilder
    ->from('users')
    ->where('active', true)
    ->orderBy('created_at', 'DESC')
    ->paginate($request);

// Access results
foreach ($pagination as $row) {
    echo $row['name'];
}

// Pagination metadata
$pagination->getCurrentPage();  // 3
$pagination->hasMore();         // true/false
$pagination->getTotal();        // null (not computed by default)
```

### With total count

Pass `withTotal: true` to compute the total count (requires an additional query):

```php
$pagination = $queryBuilder
    ->from('users')
    ->paginate($request, withTotal: true);

$pagination->getTotal();       // 1523
$pagination->getTotalPages();  // 77
```

### Supported request types

| Request Type              | Returns            |
|---------------------------|--------------------|
| `OffsetPaginationRequest` | `OffsetPagination` |
| `CursorPaginationRequest` | `CursorPagination` |
| `RangePaginationRequest`  | `RangePagination`  |

```php
use Hector\Pagination\Request\CursorPaginationRequest;

$request = new CursorPaginationRequest(
    perPage: 20,
    position: ['id' => 42],
);

$pagination = $queryBuilder
    ->from('users')
    ->orderBy('id')
    ->paginate($request);

$pagination->getNextPosition();  // ['id' => 62]
```

> ⚠️ **Cursor pagination requirements**
>
> Cursor (keyset) pagination seeks pages with `WHERE` conditions built from the `ORDER BY` columns, so the ordering must
> meet a few requirements:
>
> - **Total order.** The `ORDER BY` columns must be unique *together*: either a unique column, or a non-unique column
>   followed by a unique tie-breaker (typically the primary key). Ordering by a non-unique column alone — e.g.
>   `ORDER BY status` — **silently skips or duplicates rows** across pages. Use `ORDER BY status, id` instead.
> - **No `NULL` values** in the ordered columns.
> - **No columns whose sort order differs from their bound-parameter comparison.** The most common pitfall is a MySQL
>   `ENUM`: `ORDER BY enum_col` sorts by the ENUM's *declaration index*, while the cursor's `WHERE enum_col > :value`
>   (bound as a string) compares *lexicographically*. The two disagree, so rows are skipped. Order by a plain unique
>   column (such as the primary key) instead.
> - **Plain columns only.** Ordering expressions (e.g. `ORDER BY RAND()`) cannot be used as cursor keys.

### Chunk paginate

Use `chunkPaginate()` to iterate through all pages automatically. The callback receives two arguments — the page items
(plain array of rows) and the `PaginationInterface` for metadata. Return `false` to stop early.

```php
use Hector\Pagination\Request\CursorPaginationRequest;

$queryBuilder
    ->from('users')
    ->orderBy('id')
    ->chunkPaginate(
        new CursorPaginationRequest(perPage: 100),
        function (array $items, PaginationInterface $pagination) {
            foreach ($items as $row) {
                // Process each row...
            }
        },
    );
```

If the builder has a `limit()` set, `chunkPaginate()` honors it as a global maximum across all pages — it adjusts the
last page's per-page size so that the total number of processed rows never exceeds the limit.

This is more efficient than LIMIT/OFFSET for large datasets, especially with cursor pagination, because each page
starts where the previous one left off.

> 💡 **Tip**: See the [Pagination documentation](pagination.md) for details on pagination types, navigators, and response
> preparation.

---

## Installation

```bash
composer require hectororm/query
```
