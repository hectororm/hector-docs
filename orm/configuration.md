---
breadcrumb:
  - ORM
  - Advanced configuration
summary-order: ;1
---

# Advanced configuration

**Hector ORM** allows you to configure advanced entity behaviors using PHP attributes. These attributes can be applied
to your entity classes to control relationships, column visibility, table mapping, type casting, and other behaviors.

## Relationships

See the [Relationships](relationships.md) section for how to declare `OneToOne`, `OneToMany`, `ManyToOne`, and
`ManyToMany` associations using dedicated attributes.

## Specify Column Types

Use the `Hector\Orm\Attributes\Type` attribute on the class to define the type of a column and pass optional arguments.
The `type` must implement `TypeInterface`.

See the [Column Types](../components/data-types.md) section for full details and examples.

**Example:**

```php
use Hector\Orm\Attributes as Orm;
use Hector\Orm\Entity\Entity;
use App\Orm\Type\PriceType;

#[Orm\Type('price', PriceType::class, 'EUR')]
class Product extends Entity {
    public float $price;
}
```

## Hide Columns from Output

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

## Define a Primary Column

Use the `Hector\Orm\Attributes\Primary` attribute on the class to designate one or more columns as the primary key.

This attribute is only necessary when the schema does not already declare the primary key, such as with views or
incomplete database introspection. It helps the ORM support methods like `find()` and `findOne()`.

**Example:**

```php
use Hector\Orm\Attributes as Orm;
use Hector\Orm\Entity\MagicEntity;

#[Orm\Primary('id')]
class User extends MagicEntity {}
```

## Set Custom Table and Schema Names

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

## Specify a Custom Mapper

Use the `Hector\Orm\Attributes\Mapper` attribute on the class to associate a custom mapper. The given class must
implement `MapperInterface`.

**Example:**

```php
use Hector\Orm\Attributes as Orm;
use App\Mapper\CustomUserMapper;

#[Orm\Mapper(CustomUserMapper::class)]
class User {
    // ...
}
```

This allows custom logic to be injected for mapping between the entity and database representation.
