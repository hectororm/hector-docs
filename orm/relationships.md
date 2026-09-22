---
breadcrumb:
  - ORM
  - Relationships
summary-order: ;5
keywords:
  - relationships
  - has-one
  - has-many
  - belongs-to
  - many-to-many
  - eager-loading
---

# 🔗 Relationships

This guide presents all relationship types in **Hector ORM** using named parameters, including filtering options,
polymorphic handling, and advanced querying via builders. It also provides contextual explanations to help you
understand when and how to use each feature.

## Overview of attributes

**Hector ORM** provides the following attributes to declare entity relationships. These attributes are declared as PHP attributes
and support named parameters.

| Attribute       | Purpose                                  |
|-----------------|------------------------------------------|
| `HasOne`        | References one entity through a local foreign key |
| `HasOneChild`   | Defines a single child with the foreign key on the target (unreleased) |
| `HasMany`       | Defines a one-to-many relationship       |
| `BelongsToMany` | Defines a many-to-many link via pivot    |
| `BelongsTo`     | Derives the inverse of a declared relationship |

### Parameters

| Parameter     | Type          | Required | Description                                                   |
|---------------|---------------|----------|---------------------------------------------------------------|
| `target`      | string (FQCN) | Yes      | The target entity class                                       |
| `name`        | string        | Yes      | The public name used to access the relation                   |
| `columns`     | array         | No       | Associative mapping of local ↔ foreign keys                   |
| `pivotTable`  | string        | No       | Name of the pivot table (for `BelongsToMany` only)            |
| `columnsFrom` | array         | No       | Mapping of source → pivot table columns (for `BelongsToMany`) |
| `columnsTo`   | array         | No       | Mapping of pivot → target table columns (for `BelongsToMany`) |

### Additional parameters

These optional named parameters can be used to filter or shape the relationship:

| Parameter | Type            | Description                                   |
|-----------|-----------------|-----------------------------------------------|
| `where`   | array           | Static WHERE conditions on the related entity |
| `orderBy` | string or array | ORDER BY clause                               |
| `groupBy` | string or array | GROUP BY clause                               |
| `having`  | array           | HAVING conditions for grouped queries         |
| `limit`   | int             | Maximum number of results returned            |

---

## One-to-One / Many-to-One

A `HasOne` relationship references one instance of another entity, with the foreign key stored on the **current entity**.
For a single child whose foreign key is stored on the target, use [HasOneChild](#single-child-hasonechild).

### Example: A User has one related Profile entity

```php
use Hector\Orm\Attributes\BelongsTo;
use Hector\Orm\Attributes\HasOne;
use Hector\Orm\Attributes\Table;
use Hector\Orm\Entity\MagicEntity;

#[Table('users')]
#[HasOne(
    target: Profile::class,
    name: 'profile',
    columns: ['profile_id' => 'id'],
)]
class User extends MagicEntity {}

#[Table('profiles')]
#[BelongsTo(
    target: User::class,
    name: 'users',
    foreignName: 'profile',
)]
class Profile extends MagicEntity {}
```

Here, `users.profile_id` references `profiles.id`. With the existing `HasOne` implementation (`ManyToOne`), the inverse
`$profile->users` is a collection. The child-owned foreign key example below has scalar navigation in both directions.

### With filters

Use filtering parameters directly to restrict results statically (e.g. only active profiles).

```php
#[HasOne(
    target: Profile::class,
    name: 'profile',
    where: ['active' => '1']
)]
```

---

## Single child: HasOneChild

> **Unreleased:** This API is introduced by [issue #135](https://github.com/hectororm/hectororm/issues/135).

Use `HasOneChild` when the current entity is the parent and the foreign key belongs to its single child:

```text
users.id ← profiles.user_id (FOREIGN KEY + UNIQUE)
```

```php
use Hector\Orm\Attributes as Orm;
use Hector\Orm\Entity\MagicEntity;

#[Orm\Table('users')]
#[Orm\HasOneChild(
    target: Profile::class,
    name: 'profile',
    columns: ['id' => 'user_id'],
    orphanRemoval: true,
)]
class User extends MagicEntity {}

#[Orm\Table('profiles')]
#[Orm\BelongsTo(
    target: User::class,
    name: 'user',
    foreignName: 'profile',
)]
class Profile extends MagicEntity {}
```

Both `$user->profile` and `$profile->user` return an entity or `null`. On creation:

```php
$user = new User();
$user->name = 'Example';
$user->profile = new Profile();
$user->profile->bio = 'Hello';
$user->save();
```

Hector saves the user, propagates its generated identifier into `profile.user_id`, then saves the profile. A bidirectional
graph may also be saved from the child side. Resolved one-to-one directions participate in lifecycle transactions on
either side; an invalid child write also rolls back a new parent. Deferred batches prioritize materialized parent roots
even when a child was queued first. Existing related non-link field changes still use `save(cascade: true)`.

The relation accepts the target type, its subclasses or `null`; it does not accept a collection. Declare a `UNIQUE`
constraint on the child FK, or use a shared primary key, to enforce at most one child per parent in the database. Multiple
children returned by a parent-side read are rejected rather than silently selecting one.

### Detachment, replacement and filters

`HasOneChild` defaults to `orphanRemoval: false`; declaring a child does not implicitly authorize deleting it.

| Configuration | Assigning `null` or replacing the child |
| --- | --- |
| `orphanRemoval: false` | Detach the old child if all linking columns can be cleared; otherwise throw `RelationException`. |
| `orphanRemoval: true` | Delete the old child before attaching the replacement. |

Replacements are atomic through the shared `Lifecycle` service. Reassigning the same persisted child does not delete it.
A new instance with an assigned/shared primary key is still a new entity requiring an INSERT; replacement must follow the
declared policy. Failed writes restore generated keys, the previous entity state and the pending scalar assignment.

An explicit assignment works even if the previous child was not loaded. Hector resolves that previous child through the
relation view. Query hydration is not an assignment and does not overwrite an already-pending assignment.

`where`, `orderBy`, `limit`, `groupBy` and `having` configure the read view, as with other relationships. A child hidden by
that view is never implicitly deleted. If it already owns the unique FK, attempting to insert another child will fail the
database constraint and roll back. Read filters are not default values for new entities.

`getRelated()->unset('profile')` discards the cache and pending assignment; it does not change the database. Parent deletion
remains governed by SQL FK actions or explicit ORM deletes. See [Relationship lifecycle](relationship-lifecycle.md).

### Advanced direction configuration

For custom mapper declarations, `Relationships::oneToOne()` exposes the direction explicitly:

```php
$relationships->oneToOne(
    target: Profile::class,
    name: 'profile',
    columns: ['id' => 'user_id'],
    isParent: true,
    orphanRemoval: true,
);
```

`isParent` belongs to the existing `Relationship\OneToOne` constructor and the programmatic declaration:

| Value | Meaning |
| --- | --- |
| `true` | Source is the parent; save it before the target child. `HasOneChild` fixes this role. |
| `false` | Source is the child; save the target parent as needed before propagating its keys locally. |
| `null` | Infer a unique matching FK direction from schema metadata; otherwise keep the historical child-side strategy before v2. |

When `columns` is omitted, Hector can infer one FK mapping. Several distinct matching mappings require explicit columns.
Without a matching FK, conventional column names come from the source PK for an explicit parent role, or from the target
PK for the child/historical role. PK membership or column names alone never establish the dependency direction.

The relation exposes `isParent()` (resolved `true`, `false`, or `null` for historical fallback) and
`getConfiguredIsParent()` (the originally supplied option). `reverse()` inverts a resolved direction and the mapping,
but does not copy orphan-removal policies. An unresolved historical relation keeps its historical inverse before v2.

**Compatibility:** `HasOne` still creates `ManyToOne`. Direct users of `OneToOne` may observe corrected write ordering when
the schema reveals a FK on the target, and corrected ordering in its inverse. Specify the direction explicitly when the
schema cannot establish it. Removal of the historical fallback is tracked for v2.

---

## One-to-Many

A `HasMany` relationship allows a single entity to reference multiple target entities. This is typically used for
collections.

### Example: A User is linked to multiple Posts

```php
#[HasMany(
    target: Post::class,
    name: 'posts'
)]
class User extends MagicEntity {}
```

### With sorting and limit

```php
#[HasMany(
    target: Post::class,
    name: 'posts',
    orderBy: ['created_at' => 'DESC'],
    limit: 5
)]
```

---

## Many-to-Many

A `BelongsToMany` relationship is used when an entity is related to many others, and vice versa, through a pivot table.

### Example: A User can be assigned multiple Roles

```php
#[BelongsToMany(
    target: Role::class,
    name: 'roles'
)]
class User extends MagicEntity {}
```

### Customizing the pivot

```php
#[BelongsToMany(
    target: Role::class,
    name: 'roles',
    pivotTable: 'user_role',
    columnsFrom: ['user_id' => 'id'],
    columnsTo: ['role_id' => 'id'],
    where: ['enabled' => '1'],
    orderBy: ['name' => 'ASC']
)]
```

---

## Polymorphic relationships

Polymorphic relations allow one entity to reference several types of targets. These are
defined by using multiple relationship attributes with different names, filtered by a discriminator column.

### Example

A Comment can be linked to either Articles or Videos:

```php
#[HasMany(
    target: Comment::class,
    name: 'comments',
    columns: ['id' => 'commentable_id'],
    where: ['commentable_type' => 'article']
)]
class Article extends MagicEntity {}

#[HasMany(
    target: Comment::class,
    name: 'comments',
    columns: ['id' => 'commentable_id'],
    where: ['commentable_type' => 'video']
)]
class Video extends MagicEntity {}

#[BelongsTo(
    target: Article::class,
    name: 'article',
    columns: ['commentable_id' => 'id'],
    where: ['commentable_type' => 'article']
)]
#[BelongsTo(
    target: Video::class,
    name: 'video',
    columns: ['commentable_id' => 'id'],
    where: ['commentable_type' => 'video']
)]
class Comment extends MagicEntity {
    public int $commentable_id;
    public string $commentable_type; // 'article' or 'video'
}
```

Accessing the related entity:

```php
$comment = Comment::find(1);

// Access based on type
if ($comment->commentable_type === 'article') {
    $target = $comment->article;
} else {
    $target = $comment->video;
}
```

---

## Accessing relations

After declaration, relationships are directly accessible as properties (if using `MagicEntity`).

```php
$user->profile; // Profile entity
$comment->target; // Either Article or Video instance
```

For collections:

```php
foreach ($user->posts as $post) {
    echo $post->title;
}
```

If you're not using `MagicEntity`, you can expose relationships through explicit methods:

```php
class User extends Entity {
    public function getProfile(): ?Profile
    {
        return $this->getRelated()->get('profile');
    }
    
    public function getPosts(): Collection
    {
        return $this->getRelated()->get('posts');
    }
}
```

---

## Eager loading

Use `with()` to preload relationships and avoid the N+1 query problem.

```php
$users = User::query()->with(['profile', 'posts'])->all();

// Nested eager loading
$users = User::query()->with(['posts' => ['comments', 'author']])->all();
```

> **See also**: You can filter entities through their relationships using dot notation in conditions (e.g.
> `where('relation.column', value)` or `where('relation1.relation2.column', value)`).
> See [Filtering through relationships](builder.md#filtering-through-relationships) in the Builder documentation.

---

## Persisting relations

### Assigning relations

With `MagicEntity`, you can assign relations directly as properties:

```php
$user = User::find(1);

// One-to-One / Many-to-One
$profile = new Profile();
$profile->bio = 'Hello world';
$user->profile = $profile;

// One-to-Many / Many-to-Many (assign collection)
$user->roles = new Collection([$role1, $role2]);
```

With classic `Entity`, use `getRelated()->set()`:

```php
$user->getRelated()->set('profile', $profile);
```

### Saving new relations

When assigning **new** (not yet persisted) entities, the ORM automatically detects they need to be saved:

```php
$user = User::find(1);

$profile = new Profile();
$profile->bio = 'New profile';
$user->profile = $profile;

$user->save(); // Profile is automatically saved (new entity detected)
```

### Saving modified relations

For **existing** (already persisted) relations that have been modified, use `save(cascade: true)`:

```php
$user = User::find(1);
$user->profile->bio = 'Updated bio'; // Modify existing relation

$user->save(cascade: true); // Required to persist changes on existing relations
```

> 💡 **Tip**: `save(cascade: true)` is only necessary when modifying already-persisted related entities. For new
> relations, a simple `save()` is sufficient.

### Working with collections

```php
$user = User::find(1);

// Append to existing collection
$user->posts[] = $newPost;
$user->save(); // New post is saved automatically

// Modify existing post in collection
$user->posts[0]->title = 'Updated title';
$user->save(cascade: true); // Required for existing entity
```

### Removing relations

```php
// Clear a One-to-One / Many-to-One relation
$user->profile = null;
$user->save();

// Remove from collection
$roles = $user->roles;
unset($roles[0]);
$user->save();

// Clear relation cache to force reload from DB
$user->getRelated()->unset('roles');
```

For `BelongsToMany`, removing an entity from the collection removes its pivot link, not the target entity.
For `HasMany`, the historical behavior deletes explicitly removed children when the parent is saved.
Clearing the relation cache does not detach or delete anything in the database.

The upcoming explicit `orphanRemoval` policy distinguishes detachment from deletion. See
[Relationship lifecycle](relationship-lifecycle.md) for collection replacement, filtered views, the `Lifecycle` service,
transaction boundaries and the planned v2 migration.

---

## Pivot Data (Many-to-Many)

When working with `BelongsToMany` relationships, you often need to access additional columns from the pivot table (e.g.,
timestamps, quantities, or status flags). Use `getPivot()` to retrieve this data.

### Accessing pivot data

```php
#[BelongsToMany(
    target: Role::class,
    name: 'roles',
    pivotTable: 'user_role',
)]
class User extends MagicEntity {}
```

Given a pivot table `user_role` with additional columns:

```sql
CREATE TABLE user_role (
    user_id INT,
    role_id INT,
    assigned_at DATETIME,
    assigned_by INT,
    PRIMARY KEY (user_id, role_id)
);
```

Access pivot data on related entities:

```php
$user = User::find(1);

foreach ($user->roles as $role) {
    $pivot = $role->getPivot();
    
    // Get pivot keys (foreign keys)
    $pivot->getKeys();
    // ['user_id' => 1, 'role_id' => 5]
    
    // Get additional pivot data
    $pivot->getData();
    // ['assigned_at' => '2025-01-15 10:30:00', 'assigned_by' => 42]
}
```

### PivotData API

| Method                 | Return Type | Description                                     |
|------------------------|-------------|-------------------------------------------------|
| `getKeys()`            | `array`     | Foreign key columns linking the two entities    |
| `getData()`            | `array`     | Additional columns from the pivot table         |
| `setData(array, bool)` | `void`      | Set pivot data (second param: replace or merge) |

### Modifying pivot data

```php
$role = $user->roles[0];
$pivot = $role->getPivot();

// Replace all pivot data
$pivot->setData(['assigned_at' => date('Y-m-d H:i:s')]);

// Merge with existing data
$pivot->setData(['notes' => 'Promoted'], replace: false);

$user->save(cascade: true);
```

> ⚠️ **Warning**: `getPivot()` returns `null` for entities not loaded through a ManyToMany relationship.

---

## Without foreign keys

If your database does not enforce foreign keys, always declare column mappings manually.

```php
#[HasOne(
    target: Company::class,
    name: 'company',
    columns: ['company_id' => 'id']
)]
class Employee extends MagicEntity {}
```

---

## Using relationship builders

Relationship builders allow dynamic filtering or querying of related records:

```php
$builder = $user->getRelated()->getBuilder('posts');
$recentPosts = $builder
    ->where('status', 'published')
    ->where('created_at', '>=', '2025-01-01')
    ->orderBy('created_at', 'DESC')
    ->limit(5)
    ->all();
```

> 💡 **Tip**: The builder is independent of the cached relation. It always queries the database.

---

## Utilities & debugging

The `getRelated()` object gives access to utilities for managing relationships.

### API Reference

| Method                     | Description                                        |
|----------------------------|----------------------------------------------------|
| `get(string $name)`        | Get related entity or collection (lazy-loads)      |
| `set(string $name, $val)`  | Assign a related entity or collection              |
| `isset(string $name)`      | Check if relation is already loaded                |
| `unset(string $name)`      | Clear cached relation (will reload on next access) |
| `exists(string $name)`     | Check if relation is declared on entity            |
| `getBuilder(string $name)` | Get a query builder for the relation               |
| `save(bool $cascade)`      | Save all loaded relations                          |

### Examples

```php
// Get relation (lazy-loads if not cached)
$profile = $user->getRelated()->get('profile');

// Assign relation
$user->getRelated()->set('profile', $newProfile);

// Check if already loaded (no DB query)
if ($user->getRelated()->isset('posts')) {
    // ...
}

// Check if relation exists on entity
if ($user->getRelated()->exists('comments')) {
    // ...
}

// Clear cache to force reload
$user->getRelated()->unset('posts');
$freshPosts = $user->posts; // Reloads from DB
```

---

## Best practices

* Use named parameters for clarity and readability
* Define `columns` explicitly in absence of foreign keys
* Declare filtering options (`where`, `orderBy`, `groupBy`, `having`, `limit`) as named parameters directly in the
  relationship attribute
* Use `with()` to improve performance via eager loading
* Prefer `getBuilder()` for dynamic queries
* For polymorphic relationships, rely on multiple relation declarations with shared `name`, and control resolution using
  application logic (e.g., a `target_type` discriminator)
