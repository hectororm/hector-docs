---
breadcrumb:
  - ORM
  - Relationships
summary-order: ;5
---

# 🔗 Relationships

This guide presents all relationship types in **Hector ORM** using named parameters, including filtering options,
polymorphic handling, and advanced querying via builders. It also provides contextual explanations to help you
understand when and how to use each feature.

## 📋 Overview of attributes

**Hector ORM** provides four attributes to declare entity relationships. These attributes are declared as PHP attributes
and support named parameters.

| Attribute       | Purpose                                  |
|-----------------|------------------------------------------|
| `HasOne`        | Defines a one-to-one or many-to-one link |
| `HasMany`       | Defines a one-to-many relationship       |
| `BelongsToMany` | Defines a many-to-many link via pivot    |
| `BelongsTo`     | Inverse of `HasOne`, or polymorphic link |

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

## 1️⃣ One-to-One / Many-to-One

A `HasOne` relationship indicates that the current entity is linked to one instance of another entity. When used with
`BelongsTo`, it defines the inverse side of the relation.

### Example: A User has one related Profile entity

```php
#[HasOne(
    target: Profile::class,
    name: 'profile'
)]
class User extends MagicEntity {}

#[BelongsTo(
    target: User::class,
    name: 'user'
)]
class Profile extends MagicEntity {}
```

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

## 📚 One-to-Many

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

## 🔀 Many-to-Many

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

## 🔷 Polymorphic Relationships

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

## 🔍 Accessing Relations

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

## ⚡ Eager Loading

Use `with()` to preload relationships and avoid the N+1 query problem.

```php
$users = User::query()->with(['profile', 'posts'])->all();

// Nested eager loading
$users = User::query()->with(['posts' => ['comments', 'author']])->all();
```

---

## 💾 Persisting Relations

### Assigning Relations

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

### Saving New Relations

When assigning **new** (not yet persisted) entities, the ORM automatically detects they need to be saved:

```php
$user = User::find(1);

$profile = new Profile();
$profile->bio = 'New profile';
$user->profile = $profile;

$user->save(); // Profile is automatically saved (new entity detected)
```

### Saving Modified Relations

For **existing** (already persisted) relations that have been modified, use `save(cascade: true)`:

```php
$user = User::find(1);
$user->profile->bio = 'Updated bio'; // Modify existing relation

$user->save(cascade: true); // Required to persist changes on existing relations
```

> 💡 **Tip**: `save(cascade: true)` is only necessary when modifying already-persisted related entities. For new relations, a simple `save()` is sufficient.

### Working with Collections

```php
$user = User::find(1);

// Append to existing collection
$user->posts[] = $newPost;
$user->save(); // New post is saved automatically

// Modify existing post in collection
$user->posts[0]->title = 'Updated title';
$user->save(cascade: true); // Required for existing entity
```

### Removing Relations

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

---

## 🚫 Without Foreign Keys

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

## 🔧 Using Relationship Builders

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

## 🛠️ Utilities & Debugging

The `getRelated()` object gives access to utilities for managing relationships.

### API Reference

| Method                    | Description                                      |
|---------------------------|--------------------------------------------------|
| `get(string $name)`       | Get related entity or collection (lazy-loads)    |
| `set(string $name, $val)` | Assign a related entity or collection            |
| `isset(string $name)`     | Check if relation is already loaded              |
| `unset(string $name)`     | Clear cached relation (will reload on next access) |
| `exists(string $name)`    | Check if relation is declared on entity          |
| `getBuilder(string $name)`| Get a query builder for the relation             |
| `save(bool $cascade)`     | Save all loaded relations                        |

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

## ✅ Best Practices

* Use named parameters for clarity and readability
* Define `columns` explicitly in absence of foreign keys
* Declare filtering options (`where`, `orderBy`, `groupBy`, `having`, `limit`) as named parameters directly in the
  relationship attribute
* Use `with()` to improve performance via eager loading
* Prefer `getBuilder()` for dynamic queries
* For polymorphic relationships, rely on multiple relation declarations with shared `name`, and control resolution using
  application logic (e.g., a `target_type` discriminator)
