---
breadcrumb:
  - ORM
  - Entities
summary-order: ;2
---

# Entities

**Hector ORM** offers two main approaches to manage entities within your project, each with its own benefits and
trade-offs:

## Magic Entity

Magic entities use PHP's `__get` and `__set` magic methods to handle property access dynamically. This allows for
concise classes without explicitly declaring properties.

### Example

```php
use Hector\Orm\Entity\MagicEntity;

/**
 * @property string $firstname
 * @property string $lastname
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

## Classic Entity

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

## Custom Mapper

You can also implement your own mapper to take full control over how entities are hydrated and managed. This approach is
more complex and requires deeper integration with the ORM's internals.

### Use case

* When you need full customization for hydration or entity behavior
* When neither magic nor classic entities suit your needs
