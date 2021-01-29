```index
breadcrumb: Entities; Classic entity
summary-order: ;2
```

# Classic entity

Classic entities relies on declared properties.

## Usage

Your entities must extend  `Hector\Orm\Entity\Entity` class.

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

The entity class of the representation with declaration of properties:

```php
class Foo extends Hector\Orm\Entity\Entity {
    private int $foo_id;
    private DateTime $create_time;
    private ?DateTime $update_time;
    private string $title;
    private string $content;
    
    // ... getter/setter
    public function getTitle(): string
    {
        return $this->title;
    }
}
```

So the usage of entity:

```php
$foo = Foo::findOrFail(1); // Get row with primary key "1"

print $foo->getTitle(); // Show title column of row
```