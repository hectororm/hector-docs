```index
breadcrumb: Entities; Magic entity
summary-order: ;1
```

# Magic entity

Magic entities relies on PHP magic methods. So it's not necessary to declare properties and getter/setter inside entity
class.

## Usage

Your entities must extend  `Hector\Orm\Entity\MagicEntity` class.

## Example

In this example, we use the representation of foo table:

```
foo {
  foo_id: integer
  create_time: DateTime
  update_time: DateTime
  title: varchar(64)
  content: text
}
```

The entity class of the representation is very simple:

```php
class Foo extends Hector\Orm\Entity\MagicEntity {
}
```

So the usage of entity with magic methods is easy:

```php
$foo = Foo::findOrFail(1); // Get row with primary key "1"

print $foo->title; // Show title column of row
```