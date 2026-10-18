---
breadcrumb:
  - ORM
  - Optimistic locking
summary-order: ;7
keywords:
  - optimistic locking
  - version
  - concurrency
  - conflict
  - stale data
  - transactions
---

# 🔒 Optimistic locking

> 🆕 **Info**: *Since version 1.7*

Transactions and pessimistic locks (`SELECT ... FOR UPDATE`) protect work **within** a transaction. They do not protect
an edit that spans several requests: two users can load the same entity, edit it independently, and the last save
silently overwrites the other one.

Optimistic locking detects these conflicts with an integer **version column**. Every write is conditioned on the version
that was read, and increments it:

```sql
UPDATE articles
SET title = :title, version = version + 1
WHERE id = :id AND version = :expected_version
```

If no row matches, the entity was modified or deleted by someone else since it was read, and the ORM throws an
`OptimisticLockException` instead of overwriting the changes.

Optimistic locking is **opt-in**: entities without a version column behave exactly as before.

---

## Configuration

Add an integer column to the table:

```sql
ALTER TABLE articles ADD version INT NOT NULL DEFAULT 1;
```

Then declare it with the `Hector\Orm\Attributes\Version` attribute:

```php
use Hector\Orm\Attributes as Orm;
use Hector\Orm\Entity\MagicEntity;

#[Orm\Version('version')]
class Article extends MagicEntity {}
```

With a classic entity, declare the property as a nullable integer:

```php
use Hector\Orm\Attributes as Orm;
use Hector\Orm\Entity\Entity;

#[Orm\Version('version')]
class Article extends Entity
{
    public ?int $id = null;
    public ?string $title = null;
    public ?int $version = null;
}
```

The attribute is inherited by subclasses; the nearest declaration wins.

Version checks are performed by the built-in `GenericMapper` and `MagicMapper` (through `AbstractMapper`). A custom
mapper implementing the `Mapper` interface directly does not check versions.

### Requirements

The version column is validated against the schema on first use (write, `expectVersion()`, assignment on a
`MagicEntity`). An `OrmException` is thrown if it:

- does not exist in the table;
- is not an integer column;
- is a generated column;
- is part of the primary key, or the entity has no primary key (composite primary keys are supported).

---

## Lifecycle of the version

| Operation           | Behavior                                                                                    |
|---------------------|---------------------------------------------------------------------------------------------|
| Insert              | The ORM writes version `1`. A version set by the user is rejected with a `MapperException`. |
| Update              | Conditioned on the loaded version and incremented atomically. Conflict if no row matches.   |
| Save without change | No `UPDATE` and no increment, but the version is still checked. Conflict if the row changed. |
| Delete              | Conditioned on the loaded version. Conflict if no row matches.                              |
| `refresh()`         | Reloads the latest version, see [expected version](#validating-a-version-from-a-previous-request). |

After a successful write, the entity and its original data are refreshed from the database, so `isAltered()` returns
`false`. If someone else changed the row between the write and this refresh, the entity reflects their row, like an
explicit `refresh()`: it has no pending change, so nothing is lost.

```php
$article = Article::find(1);   // version 1
$article->title = 'New title';
$article->save();              // UPDATE ... WHERE id = 1 AND version = 1

echo $article->version;        // 2
```

### Rejected writes

The version is managed by the ORM:

- A `MagicEntity` rejects the assignment of its version column with an `OrmException`.
- With a classic entity, a modified version property is rejected on save with a `MapperException`.

A write is also rejected with a `MapperException` when the loaded version is unavailable: entity loaded with a partial
column selection, or `NULL` version in the database. Initialize the column for existing rows (e.g. with a default
value) before enabling optimistic locking.

---

## Handling conflicts

```php
use Hector\Orm\Exception\OptimisticLockException;

$article = Article::find(1);
$article->title = 'My edit';

try {
    $article->save();
} catch (OptimisticLockException $exception) {
    $exception->getEntity();          // The entity in conflict
    $exception->getExpectedVersion(); // The version the write was conditioned on

    // Inform the user, reload and merge, etc.
}
```

`OptimisticLockException` extends `OrmException`. It is thrown as is by `Entity::save()`, `Entity::delete()` and
`Orm::persist()`.

On an update or deletion conflict, the in-memory state of the entity is left untouched: its pending changes and its
version are kept. To retry, reload the entity with `refresh()`, reapply the changes and save again.

On a save without change, the version is checked by the refresh that follows the save: when the conflict is reported,
the entity has already been refreshed, and its related entities may have been linked. Within a
[relationship lifecycle](relationship-lifecycle.md) operation, everything is rolled back and restored.

---

## Validating a version from a previous request

In web applications, the entity is usually loaded again when the form is submitted. Reloading it gives the **latest**
version, which would silently accept a stale edit. Send the version with the form and check it with
`expectVersion()`:

```php
// Request 1: display the form with the current version
$article = Article::find($id);
// <input type="hidden" name="version" value="<?= $article->version ?>">

// Request 2: handle the submitted form
$article = Article::find($id);
$article->expectVersion((int)$_POST['version']); // Throws OptimisticLockException if stale
$article->title = $_POST['title'];
$article->save();
```

`expectVersion()`:

- throws an `OptimisticLockException` immediately if the loaded version differs;
- otherwise conditions the next write of the entity on this version, **even if the entity is refreshed in between**:
  a `refresh()` does not discard the client version;
- is released after the next successful write or deletion;
- throws an `OrmException` if the entity is not versioned, or a `MapperException` if its version is not loaded (e.g. a
  new entity).

---

## Transactions

Optimistic conflicts are **not** transient failures: `Orm::transaction()` and `Connection::transaction()` never retry
them, whatever the number of `attempts`. Retrying would hit the same conflict.

With a [relationship lifecycle](relationship-lifecycle.md) operation, such as `save(cascade: true)` on an entity with
lifecycle policies, a conflict on any entity of the graph rolls back the whole operation and restores the in-memory
state, including the versions of entities already written. The operation can be retried without false conflicts.

> ⚠️ **Warning**: Outside lifecycle operations (`Orm::transaction()`, `Orm::persist()` without lifecycle policies),
> only the database is rolled back. Entities already written in the failed transaction keep their incremented
> version in memory, so their next write is reported as a conflict. Reload them before reusing them.

---

## Limitations

- **Bulk operations are not checked.** `update()` and `delete()` on a [Builder](builder.md) or a `QueryBuilder` are
  executed as is: they neither check nor increment the version. Increment it explicitly with an
  [atomic increment](../components/query.md#atomic-increment--decrement) to invalidate loaded entities:

  ```php
  use Hector\Query\Statement\Increment;

  Article::query()
      ->whereEquals(['published' => false])
      ->update(['archived' => true, 'version' => new Increment()]);
  ```

- **The version is not a history.** The column holds a single counter per row: previous states are overwritten.
  Use a history table (e.g. fed by [events](events.md)) or database features to keep previous versions.
- The version is an integer counter; timestamp versions are not supported.
