---
breadcrumb:
  - ORM
  - Advanced configuration
summary-order: ;1
---

# ⚙️ Advanced configuration

**Hector ORM** allows you to configure advanced entity behaviors using PHP attributes. These attributes can be applied
to your entity classes to control relationships, column visibility, table mapping, type casting, and other behaviors.

## 🔧 OrmFactory Options

When initializing the ORM via `OrmFactory::orm()`, you can pass the following options:

| Option     | Type     | Required | Description                                             |
|------------|----------|----------|---------------------------------------------------------|
| `schemas`  | `array`  | Yes      | List of database schema names to introspect             |
| `aliases`  | `array`  | No       | Map of schema names to aliases (e.g., for multi-tenant) |
| `dsn`      | `string` | No*      | DSN string (required if no connection provided)         |
| `username` | `string` | No       | Database username                                       |
| `password` | `string` | No       | Database password                                       |
| `read_dsn` | `string` | No       | DSN for read-only replica                               |
| `name`     | `string` | No       | Connection name (default: `'default'`)                  |
| `log`      | `bool`   | No       | Enable query logging (default: `false`)                 |

**Example with all options:**

```php
use Hector\Orm\OrmFactory;

$orm = OrmFactory::orm(
    options: [
        'dsn' => 'mysql:host=localhost;dbname=myapp',
        'username' => 'root',
        'password' => 'secret',
        'read_dsn' => 'mysql:host=replica;dbname=myapp',
        'name' => 'default',
        'log' => true,
        'schemas' => ['myapp'],
        'aliases' => [
            'myapp' => 'app', // Use 'app' as alias for 'myapp' schema
        ],
    ],
    connection: null, // Created from options
);
```

**Example with existing connection:**

```php
use Hector\Connection\Connection;
use Hector\Orm\OrmFactory;

$connection = new Connection('mysql:host=localhost;dbname=myapp', 'root', 'secret');

$orm = OrmFactory::orm(
    options: [
        'schemas' => ['myapp'],
    ],
    connection: $connection,
);
```

---

## 🔗 Relationships

See the [Relationships](relationships.md) section for how to declare `OneToOne`, `OneToMany`, `ManyToOne`, and
`ManyToMany` associations using dedicated attributes.

## 🎯 Specify Column Types

Use the `Hector\Orm\Attributes\Type` attribute on the class to define the type of a column and pass optional arguments.
The `type` must implement `TypeInterface`.

See the [Column Types](../components/data-types.md) section for full details and examples.

---

### Example

```php
use Hector\Orm\Attributes as Orm;
use Hector\Orm\Entity\Entity;
use App\Orm\Type\PriceType;

#[Orm\Type(column: 'price', type: PriceType::class, currency: 'EUR')]
class Product extends Entity {
    public float $price;
}
```

Arguments after the type class are passed to the type constructor:

```php
// PriceType receives 'EUR' as first constructor argument
class PriceType extends AbstractType {
    public function __construct(private string $currency = 'USD') {}
    
    // ...
}
```

---

## 🙈 Hide Columns from Output

Use the `Hector\Orm\Attributes\Hidden` attribute on the class to hide specific columns from serialized outputs (e.g.
JSON). This attribute is repeatable.

This is especially relevant for `MagicEntity` classes, where hidden columns are completely inaccessible via dynamic
property access.

**Example:**

```php
use Hector\Orm\Attributes as Orm;
use Hector\Orm\Entity\MagicEntity;

#[Orm\Hidden('password')]
#[Orm\Hidden('token')]
class User extends MagicEntity {
}
```

---

## 🔑 Define a Primary Column

Use the `Hector\Orm\Attributes\Primary` attribute on the class to designate one or more columns as the primary key.

This attribute is only necessary when the schema does not already declare the primary key, such as:
- Database views (no intrinsic primary key)
- Tables without explicit `PRIMARY KEY` constraint
- Legacy databases with incomplete schema

**Example with a view:**

```php
use Hector\Orm\Attributes as Orm;
use Hector\Orm\Entity\MagicEntity;

#[Orm\Table('user_summary_view')]
#[Orm\Primary('user_id')]
class UserSummary extends MagicEntity {}
```

**Example with composite primary key:**

```php
use Hector\Orm\Attributes as Orm;
use Hector\Orm\Entity\MagicEntity;

#[Orm\Primary('order_id', 'product_id')]
class OrderLine extends MagicEntity {}
```

---

## 🏷️ Set Custom Table and Schema Names

By default, the entity class name is used as the table name. If your database table name differs from the entity class
name, or if you're working with multiple schemas/databases, you can override this behavior using the
`Hector\Orm\Attributes\Table` attribute on the class.

You can also specify a schema alias defined in your database configuration during schema generation.

**Example:**

```php
use Hector\Orm\Attributes as Orm;
use Hector\Orm\Entity\MagicEntity;

#[Orm\Table('table_name', 'schema_name')]
class Foo extends MagicEntity {
}
```

In this example:

* The entity `Foo` maps to the table named `table_name`
* The table resides in the database schema `schema_name`

If you're using schema aliases, `schema_name` should match the alias defined in your schema configuration.

The schema name is optional and should be used only when needed, such as in multi-database setups with overlapping table
names.

---

## 🧩 Specify a Custom Mapper

Use the `Hector\Orm\Attributes\Mapper` attribute on the class to associate a custom mapper. The given class must
implement `MapperInterface`.

**Example:**

```php
use Hector\Orm\Attributes as Orm;
use Hector\Orm\Entity\Entity;

#[Orm\Mapper(CustomUserMapper::class)]
class User extends Entity {
    public int $id;
    public string $email;
}
```

**Custom mapper implementation:**

```php
use Hector\Orm\Mapper\Mapper;
use Hector\Orm\Mapper\MapperInterface;

class CustomUserMapper extends Mapper implements MapperInterface
{
    public function hydrateEntity(object $entity, array $data): void
    {
        // Custom hydration logic
        parent::hydrateEntity($entity, $data);
        
        // Example: normalize email
        if (isset($data['email'])) {
            $entity->email = strtolower(trim($data['email']));
        }
    }
}
```

This allows custom logic to be injected for mapping between the entity and database representation.
