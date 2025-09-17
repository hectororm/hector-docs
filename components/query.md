---
breadcrumb:
- Components
- Query Builder
summary-order: ;1
---

# 🔨 Query

> **Note**: While query builder are part of the **Hector ORM** ecosystem, they are available as a standalone package:
> [`hectororm/query`](https://github.com/hectororm/query).
> You can find it on
> [Packagist](https://packagist.org/packages/hectororm/query).
> You can use them independently of the ORM, in any PHP application. 🎉

The `QueryBuilder` in **Hector ORM** provides a fluent, object-oriented API to construct and execute SQL queries. It is
designed to streamline the building of `SELECT`, `INSERT`, `UPDATE`, `DELETE`, and `UNION` queries, while maintaining
control and readability in your codebase.

## 🚀 Initialization

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

## 🧱 Query Types

The library provides specific classes for each query type. Each of these classes implements the `StatementInterface`,
enabling you to manually build the query and bind parameters.

* `Hector\Query\Select`
* `Hector\Query\Insert`
* `Hector\Query\Update`
* `Hector\Query\Delete`
* `Hector\Query\Union`

### 🧪 Example

```php
use Hector\Query\Select;

$select = new Select();
$select
    ->from('table')
    ->where('field', 'value');

use Hector\Connection\Bind\BindParamList;
use Hector\Connection\Connection;

$binds = new BindParamList();
$statement = $select->getStatement($binds);

$connection = new Connection('...');
$result = $connection->fetchAll($statement, $binds);
```

## 🧮 Conditions

Both `WHERE` and `HAVING` clauses are supported using the same API. Simply switch the method prefix.

### Where / Having

```php
use Hector\Query\QueryBuilder;

$queryBuilder
    ->from('table', 'alias')
    ->where('field', '=', 'value')
    ->orWhere('field', '=', 'value2');
```

### Condition Shortcuts

Several convenience methods are provided:

```php
$queryBuilder->whereIn('id', [1, 2, 3]);
$queryBuilder->whereNotBetween('age', 18, 30);
$queryBuilder->whereGreaterThan('score', 50);
$queryBuilder->whereExists(new Select(...));
$queryBuilder->whereContains('substring');
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
* `whereContains($string)`
* `whereStartsWith($string)`
* `whereEndsWith($string)`

## 📋 Selecting Columns

You can customize the columns returned by your query:

```php
$queryBuilder
    ->column('name')
    ->column('email', 'user_email');

$queryBuilder->resetColumns();
$queryBuilder->columns(['id', 'name', 'created_at']);
```

## 🧑‍🤝‍🧑 Grouping Results

To group query results:

```php
$queryBuilder
    ->groupBy('category_id')
    ->groupBy('status');

$queryBuilder->resetGroups();
$queryBuilder->groupByWithRollup();
```

## 🔢 Ordering Results

Sort your result set with `orderBy()`:

```php
$queryBuilder
    ->orderBy('created_at', 'DESC')
    ->orderBy('name', 'ASC');

$queryBuilder->resetOrder();
$queryBuilder->random(); // ORDER BY RAND()
```

## 📦 Limiting Results

Control pagination using `limit()` and `offset()`:

```php
$queryBuilder->limit(10);
$queryBuilder->offset(20);
$queryBuilder->resetLimit();
```

## 📝 Assigning Values

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

## 🔗 Joins

Join tables with the following methods:

```php
$queryBuilder->innerJoin('users', 'users.id = posts.user_id', 'u');
$queryBuilder->leftJoin('categories', 'categories.id = posts.category_id');
$queryBuilder->rightJoin('tags', 'tags.id = posts.tag_id');

$queryBuilder->resetJoin();
```

## 🔀 Unions

The `Union` class allows combining multiple `SELECT` queries:

```php
use Hector\Query\Select;
use Hector\Query\Union;

$select1 = (new Select())->from('table1');
$select2 = (new Select())->from('table2');

$union = new Union();
$union->addSelect($select1, $select2);
```

`Union` also implements `StatementInterface`, allowing you to bind and execute it like any other query.

## 📤 Fetching Results

Use the following methods to retrieve data:

```php
$queryBuilder->fetchOne();      // ?array - first row
$queryBuilder->fetchAll();      // Generator - all rows
$queryBuilder->fetchColumn();   // Generator - specific column
```

> `fetchAll()` and `fetchColumn()` return a `Generator`. Refer to PHP
> documentation: [https://www.php.net/manual/en/class.generator.php](https://www.php.net/manual/en/class.generator.php)

## 🔢 Counting Results

Quickly count the results of a query:

```php
$count = $queryBuilder
    ->from('table')
    ->where('status', 'active')
    ->count();

$rows = $queryBuilder->fetchAll();
```

This method resets column selection, limit, and order, but does not mutate the original query builder.

## 🔁 Distinct Values

Use `distinct()` to eliminate duplicates:

```php
$queryBuilder
    ->from('users')
    ->distinct()
    ->fetchAll();
```

## ❓ Existence Check

Determine if any row matches the conditions:

```php
$exists = $queryBuilder
    ->from('users')
    ->where('email', 'alice@example.com')
    ->exists();
```

Does not alter the query builder instance.

## ✍️ Shortcuts for Insert / Update / Delete

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

These shortcut methods do not affect the `QueryBuilder` instance, so it remains reusable for further operations.
