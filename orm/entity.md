---
breadcrumb:
  - ORM
  - Entities
summary-order: ;2
---

# 🧱 Entities

**Hector ORM** offers two main approaches to manage entities within your project, each with its own benefits and
trade-offs.

> 💡 **Tip**: For advanced entity configuration (table mapping, column types, hidden fields, custom mappers), see [Advanced configuration](configuration.md).

## 🪄 Magic Entity

Magic entities use PHP's `__get` and `__set` magic methods to handle property access dynamically. This allows for
concise classes without explicitly declaring properties.

### Example

```php
use Hector\Orm\Entity\MagicEntity;
use Hector\Orm\Collection\Collection;
use DateTimeInterface;

/**
 * @property int $id
 * @property string $firstname
 * @property string $lastname
 * @property string|null $email
 * @property DateTimeInterface $created_at
 * @property-read Collection<Post> $posts
 */
class User extends MagicEntity {}

$user = new User();
$user->firstname = 'Alice';
$user->lastname = 'Dupont';
echo $user->firstname; // Outputs "Alice"
```

### Pros

* Minimal boilerplate
* Very flexible
* Easy to prototype quickly

### Cons

* No native IDE autocompletion (partially mitigated with @property PHPDoc)
* No static analysis
* Harder to debug or refactor

---

## 🏛️ Classic Entity

Classic entities use explicitly declared class properties, giving better integration with IDEs and static analysis
tools.

### Example

```php
use Hector\Orm\Entity\Entity;

class User extends Entity
{
    public string $firstname;
    public string $lastname;
}

$user = new User();
$user->firstname = 'Alice';
$user->lastname = 'Dupont';
echo $user->firstname; // Outputs "Alice"
```

### Pros

* IDE support (autocompletion, inspections)
* Better for long-term maintenance
* Easier to read and understand

### Cons

* More verbose
* Less flexible for dynamic schemas

---

## 💾 Persisting Entities

### Creating and Saving

To persist a new entity in the database, instantiate it, set its properties, and call the `save()` method:

```php
$user = new User();
$user->firstname = 'Alice';
$user->lastname = 'Dupont';
$user->email = 'alice@example.com';

$user->save();
```

#### Cascade Save

> 🆕 **Info**: *Since version 1.1*

When your entity has relationships, you can persist them all at once using the `cascade` parameter. This will automatically save any related entities that have been modified or created:

```php
$user->save(cascade: true);
```

### Updating

```php
$user = User::find(1);
$user->email = 'new-email@example.com';
$user->save();

// Check if entity has been modified
if ($user->isAltered()) {
    $user->save();
}

// Check specific column
if ($user->isAltered('email')) {
    // ...
}
```

### Deleting

```php
$user = User::find(1);
$user->delete();
```

### Refreshing from Database

```php
$user = User::find(1);
$user->firstname = 'Modified';
$user->refresh(); // Reloads original data from DB
```

### Loading Relations On-Demand

Use `load()` to eagerly load relations on an existing entity instance. This is useful when you need to load relations after the entity has been fetched, avoiding N+1 queries.

```php
$user = User::find(1);

// Load single relation
$user->load(['posts']);

// Load multiple relations
$user->load(['posts', 'profile']);

// Load nested relations
$user->load(['posts' => ['comments', 'author']]);
```

> 💡 **Tip**: For bulk loading on collections, prefer `with()` on the query builder. Use `load()` when you need to load relations on an already-fetched entity.

### Comparing Entities

Use `isEqualTo()` to compare two entities by their primary key values:

```php
$user1 = User::find(1);
$user2 = User::find(1);
$user3 = User::find(2);

$user1->isEqualTo($user2); // true (same primary key)
$user1->isEqualTo($user3); // false (different primary key)

// Also works with pivot data for ManyToMany relations
$role1 = $user->roles[0];
$role2 = $anotherUser->roles[0];
$role1->isEqualTo($role2); // Compares both PK and pivot keys
```

> 💡 **Tip**: This method compares primary key values, not object identity. Two different instances representing the same database row are considered equal.

### Bulk Operations on Collections

Collections returned by the ORM support bulk operations, allowing you to apply actions to multiple entities at once:

```php
$users = User::query()->where('active', false)->all();

// Save all entities in collection
$users->save();

// Delete all entities in collection
$users->delete();

// Refresh all entities from database
$users->refresh();
```

#### Cascade Save on Collections

> 🆕 **Info**: *Since version 1.1*

Like individual entities, collections also support cascade saving. This is useful when you need to persist a batch of entities along with their relationships:

```php
$users->save(cascade: true);
```

---

## 🧩 Custom Mapper

You can also implement your own mapper to take full control over how entities are hydrated and managed.

See [Advanced configuration](configuration.md#specify-a-custom-mapper) for implementation details.

### Use case

* When you need full customization for hydration or entity behavior
* When neither magic nor classic entities suit your needs
