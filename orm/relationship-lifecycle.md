---
breadcrumb:
  - ORM
  - Relationship lifecycle
summary-order: ;6
keywords:
  - relationships
  - lifecycle
  - orphan-removal
  - transactions
  - rollback
  - migration
---

# Relationship lifecycle

> **Unreleased:** The explicit lifecycle policies and service described here accompany
> [HectorORM PR #148](https://github.com/hectororm/hectororm/pull/148).

See [Relationships](relationships.md) for relationship declarations, loading and query options.

Cardinality, dependency direction and lifecycle are separate concerns. A child relationship does not, by its name alone,
authorize deleting the target entity.

## Parent-child policy

Configure `orphanRemoval` on `HasMany` or `HasOneChild`, or on their programmatic declarations. Both parent-side scalar
and collection relationships use the same `Lifecycle` service:

```php
use Hector\Orm\Attributes as Orm;

#[Orm\HasMany(
    target: OrderLine::class,
    name: 'lines',
    columns: ['id' => 'order_id'],
    orphanRemoval: true,
)]
```

| Policy | Explicitly removed child |
| --- | --- |
| `false` | Clear its linking columns and save it. Required linking columns or primary-key columns prevent detachment and produce a `RelationException`. |
| `true` | Delete it through the ORM. |
| Omitted / `null` on `HasMany` (before v2) | Preserve historical deletion of detached children. |
| Omitted / `null` on `HasOneChild` | Default to `false`: detach rather than delete. |

The omitted-policy default on `HasMany` is a **deprecated compatibility behavior**. Set `orphanRemoval: true` explicitly to
preserve deletion when upgrading to v2. The v2 target default is `false` for both parent-side relation types.

See [HasOneChild](relationships.md#single-child-hasonechild) for the scalar relation introduced by
[issue #135](https://github.com/hectororm/hectororm/issues/135).

## Explicit scalar changes

```php
$user->profile = null;          // Detach or delete the previous profile according to its policy.
$user->save();

$user->profile = new Profile(); // A replacement releases the previous unique link first.
$user->profile->bio = 'New';
$user->save();
```

An explicit scalar assignment resolves the previous child when needed, even if it was never loaded. This differs from
collection replacement: a scalar relation describes at most one child, whereas an unloaded collection may contain
arbitrarily many unseen members.

The lookup respects the declared relation view. A hidden child is preserved; its unique FK may then prevent insertion of
a replacement. Reading an absent/filtered-out child alone never schedules removal. Reassigning the same persisted child
does not delete it, and replacing an intermediate unsaved assignment does not delete an unrelated persisted entity that
was never linked by that assignment.

Failed persistence restores the old child state and pending assignment for retry. Invalidating the cache explicitly with
`getRelated()->unset('profile')` also discards that pending assignment without changing database rows.

## Explicit collection changes

```php
$lines = $order->lines;
unset($lines[0]);            // Explicit removal, processed when saving the order.
$order->save();
```

Replacing an offset also removes its previous entity. Removing and then reattaching the same entity before saving does
not delete it. A transient child removed before persistence produces no SQL DELETE; a pending insert for that removed
child is cancelled when it has not already been persisted.

When the policy is **explicitly configured**, assigning another collection replaces the previously materialized members:

```php
use Hector\Orm\Collection\Collection;

$order->lines;                         // Materialize the relation.
$order->lines = new Collection([$kept]);
$order->save();                        // Process known members absent from the new collection.
```

- Only previously materialized members and explicitly tracked removals are candidates for removal.
- Assigning an empty collection to an **unloaded** relation does not clear unseen database rows.
- Replacing a filtered or limited collection never removes rows outside the loaded view.
- Assigning `null` with an explicit policy is equivalent to assigning an empty collection, with the same rules.
- Without an explicit policy, pre-v2 whole-collection assignment keeps its historical behavior (no inferred difference).
- Query hydration is not user replacement and never creates removal records.
- `getRelated()->unset('lines')` only invalidates the cache. Save pending removals before discarding their collection.

This is deliberately not a database-wide collection synchronization API. Query filters, grouping and limits do not define
ownership of all matching or missing rows. To remove children, load the intended entities and explicitly remove them.

For existing linked children whose non-link fields change, continue to use `save(cascade: true)`.

## Other relation types

A parent reference (`ManyToOne`, currently exposed by `HasOne`) cannot apply `orphanRemoval` to its parent. A many-to-many
relationship removes pivot links, not the shared target entities. Passing a non-null `orphanRemoval` option to unsupported
relationships throws a `RelationException`; it is not silently ignored.

Inverting a relationship does not copy its deletion policy to the opposite direction.

## Atomic writes and rollback

`Orm::lifecycle()` exposes the `Hector\Orm\Lifecycle` service associated with that ORM instance. It orchestrates child
detachment/removal, change tracking and the active transaction context. Normal entity saves invoke it automatically.
The underlying `Storage\LifecycleTransaction` handles snapshots and database transaction/savepoint mechanics.

### Transaction snapshots and PHP serialization

`Related`, ORM `Collection` and `EntityData` implement the internal
`Hector\Orm\Storage\LifecycleSnapshotInterface` contract:

- `lifecycleSnapshot()` captures the state owned by the participant, including pending changes needed for rollback.
- `restoreLifecycleSnapshot()` restores that state on the same instance without issuing SQL or scheduling new mutations.

The transaction uses this contract rather than calling `__serialize()` or `__unserialize()`. Snapshot arrays may retain
object references; they are process-local rollback state, not a transport or cache format. Mapped entity properties and
storage statuses are also captured by the transaction.

PHP serialization of `Related` keeps its existing `related` payload. The new scalar assignment journal (`assignments`)
is deliberately excluded and is empty after unserialization; serializing an entity does not clear the journal on the
original live object. A native serialization round-trip preserves cached relation values, but does not transfer a complete
ORM unit of work. Reload entities in the receiving ORM and explicitly reapply intended assignments before persisting them.

### Explicit lifecycle transactions

An explicit operation can use the same service:

```php
$orm->lifecycle()->transaction($order, function () use ($order): void {
    $order->save();
    // Additional ORM writes on the same connection share the lifecycle context.
});
```

The callback result is returned. Nested calls join the active context; their exceptions must propagate to its boundary
to roll back the complete operation. The service resets its active context on both success and failure. Its `track()`,
`linkChild()`, `removeChild()`, `cancelPendingInsert()` and `persistBatch()` methods are internal integration points for the ORM
and relationships. A `persist()` called inside this service joins the active lifecycle transaction and tracks its
pending entities, even when they have no loaded child relationship themselves.

Saving a materialized graph containing a parent-child lifecycle relation runs its linking/removal writes and the parent
save in one transaction. Direct `OneToMany::linkNative()` calls also protect their child operations. Removed links are
released before new children are saved, allowing replacements under unique constraints.
For `save(cascade: true)`, related saves remain within that same lifecycle transaction: an invalid field on an existing
child rolls back earlier orphan removals as well. Batch preparation cancels never-persisted removed children before any
queued insertion, including when the child was scheduled before its parent.

On failure, the transaction restores the captured mapped properties (including generated identifiers and uninitialized
typed properties), original ORM data, entity statuses, relation caches and collection removal tracking. Pending user
edits and the desired replacement remain available for correction and retry. Custom non-mapped state or external side
effects of event listeners are not undone. Snapshots preserve property values/references; they do not deep-copy arbitrary
mutable user objects. After-save/delete events are not after-commit notifications. A vetoed child removal or save aborts
the lifecycle operation rather than silently discarding removal tracking. A listener that restores a detached FK or
redirects a newly attached child to another parent also aborts the operation.

The transaction scope is **one connection**. Cross-connection materialized graphs are rejected before their captured
entities are written; this feature does not provide distributed transactions. `persist()` batches containing lifecycle
relations likewise require a single connection and retain snapshots until the complete batch succeeds. SQL writes
performed outside the ORM cannot be tracked in memory.

When a caller already owns a transaction, a savepoint isolates a failing lifecycle operation. A successful savepoint
release is not a commit of the outer transaction. If the caller subsequently rolls back that outer transaction, discard
or reload the affected entity graph before reuse, as with other external database changes. Use a transactional storage
engine with savepoint support (for example InnoDB or SQLite); do not execute implicitly committing DDL inside callbacks.

Transactions do not implement optimistic versioning of relationship assignments. Applications performing concurrent
reparenting must coordinate their updates/locking. Keep FK and uniqueness constraints in the database.

## Parent deletion

`orphanRemoval` handles removal from an association, not deletion of its parent. Parent deletion remains governed by SQL
FK actions (`CASCADE`, `SET NULL`, `RESTRICT`) or explicit ORM deletes. SQL cascades do not dispatch child ORM events.

## Migration checklist

1. Declare `orphanRemoval: true` on existing `HasMany` relations that intentionally delete removed children.
2. Use `false` for detachable children; ensure all linking columns are nullable and are not primary-key columns.
3. Review whole-collection assignments when opting in: known missing members now follow the declared policy.
4. Keep many-to-many target deletion separate from pivot removal.
5. Retain explicit FK actions for parent deletion and uniqueness constraints for one-to-one associations.

Follow [#146](https://github.com/hectororm/hectororm/issues/146) for this foundation and
[#147](https://github.com/hectororm/hectororm/issues/147) for the v2 transition.
