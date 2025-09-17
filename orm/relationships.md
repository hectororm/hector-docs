---
breadcrumb:
  - ORM
  - Relationships
summary-order: ;5
---

# Relationships

This guide presents all relationship types in **Hector ORM** using named parameters, including filtering options,
polymorphic handling, and advanced querying via builders. It also provides contextual explanations to help you
understand when and how to use each feature.

## Overview of attributes

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

## One-to-One / Many-to-One

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

## Polymorphic Relationships

Polymorphic relations allow one entity to reference several types of targets through a shared interface. These are
defined by using multiple `#[BelongsTo(...)]` attributes with the same `name`, but filtered by custom logic based on
your entity data.

There is no explicit `type` parameter supported. Instead, **Hector ORM** resolves which relation to load by comparing
declared relation keys (`columns`) and applying them based on your implementation (e.g. via discriminator column).

### Example: A Comment is linked to either an Article or a Video entity

```php
#[BelongsToMany(
    target: Article::class,
    name: 'articles',
    pivotTable: 'comment_link',
    columnsFrom: ['id' => 'comment_id'],
    columnsTo: ['entity_id' => 'article_id'],
    where: ['entity_type' => 'article']
)]
#[BelongsToMany(
    target: Video::class,
    name: 'videos',
    pivotTable: 'comment_link',
    columnsFrom: ['id' => 'comment_id'],
    columnsTo: ['entity_id' => 'video_id'],
    where: ['entity_type' => 'video']
)]
class Comment extends MagicEntity {}
```

## Accessing Relations

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

## Eager Loading

Use `with()` to preload relationships and avoid the N+1 query problem.

```php
$users = User::builder()->with(['profile', ['posts' => ['user']]])->all();
$comments = Comment::builder()->with(['target'])->all();
```

## Persisting Relations

Related objects are not automatically persisted. Save them explicitly before assigning.

```php
$role = Role::create(['name' => 'Editor']);
// manually assign role to relation and persist pivot manually if needed
$user->save();
```

## Without Foreign Keys

If your database does not enforce foreign keys, always declare column mappings manually.

```php
#[HasOne(
    target: Company::class,
    name: 'company',
    columns: ['company_id' => 'id']
)]
class Employee extends MagicEntity {}
```

## Using Relationship Builders

Relationship builders allow dynamic filtering or querying of related records.

```php
$builder = $user->getRelated()->getBuilder('posts');
$posts = $builder->where('status', 'published')->limit(5)->all();
```

## Utilities & Debugging

The `getRelated()` object gives access to utilities for managing relationships manually.

```php
$user->getRelated()->get('profile');
$user->getRelated()->set('profile', $profile);
$user->getRelated()->isset('profile');
$user->getRelated()->exists('profile');
```

Inspect loaded relations:

```php
print_r($user->getRelated()->all());
```

## Best Practices

* Use named parameters for clarity and readability
* Define `columns` explicitly in absence of foreign keys
* Declare filtering options (`where`, `orderBy`, `groupBy`, `having`, `limit`) as named parameters directly in the
  relationship attribute
* Use `with()` to improve performance via eager loading
* Prefer `getBuilder()` for dynamic queries
* For polymorphic relationships, rely on multiple relation declarations with shared `name`, and control resolution using
  application logic (e.g., a `target_type` discriminator)
