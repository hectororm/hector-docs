```index
breadcrumb: Relationships
summary-order: 5
```

# Relationships

## Declaration with PHP attributes

4 attributes are available to define relationships.

| Attribute                             | Description                                      |
|---------------------------------------|--------------------------------------------------|
| `Hector\Orm\Attributes\HasOne`        | To define a relation many-to-one or one-to-one   |
| `Hector\Orm\Attributes\HasMany`       | To define a relation one-to-many                 |
| `Hector\Orm\Attributes\BelongsToMany` | To define a relation many-to-many                |
| `Hector\Orm\Attributes\BelongsTo`     | To inverse a relation already declared in target |

Example:

```php
use Hector\Orm\Attributes as Orm;
use Hector\Orm\Entity\MagicEntity;

#[Orm\HasOne(Bar::class, 'bar')]
#[Orm\HasMany(Baz::class, 'baz')]
#[Orm\BelongsToMany(Qux::class, 'qux')]
class Foo extends MagicEntity {
}

#[Orm\BelongsTo(Foo::class, 'foo')]
class Bar extends MagicEntity {
}
```

## Usage

After declaration of relations on entities, the relations are accessibles with method `MyEntity::getRelated(): Related`.

`Hector\Orm\Entity\Related` class have some interesting methods:

- `Related::getBuilder('MY_RELATION_NAME'): Builder`: get builder to get relation(s)
- `Related::get('MY_RELATION_NAME'): Collection|Entity|null`: get final value
- `Related::exists('MY_RELATION_NAME'): bool`: known if relation name exists

### Magic entity

```php
use Hector\Orm\Attributes as Orm;
use Hector\Orm\Entity\MagicEntity;

#[Orm\HasOne(Bar::class, 'bar')]
#[Orm\HasMany(Baz::class, 'baz')]
class Foo extends MagicEntity {
}
```

```php
$entity = Foo::get();
$entity->bar; // Return entity of type `Bar`
$entity->baz; // Return entity of type `Collection<Bar>`
$entity->getRelated()->get('baz'); // Same as `$entity->baz`
```

### Classic entity

```php
use Hector\Orm\Attributes as Orm;
use Hector\Orm\Collection\Collection;
use Hector\Orm\Entity\Entity;

#[Orm\HasOne(Bar::class, 'bar')]
#[Orm\HasMany(Baz::class, 'baz')]
class Foo extends Entity {
    public function getBar(): ?Bar
    {
        return $this->getRelated()->get('bar');
    }

    public function getBar(): Collection
    {
        return $this->getRelated()->get('baz');
    }

    public function getFirstBar(): ?Baz
    {
        return $this->getRelated()->getBuilder('baz')->get();
    }
}
```

```php
$entity = Foo::get();
$entity->getBar(); // Return entity of type `Bar`
$entity->getBaz(); // Return entity of type `Collection<Bar>`
$entity->getRelated()->get('baz'); // Same as `$entity->getBaz()`
```

## Type of relations

In all relationship types, **Hector ORM** tries to guess the column names based on:

- The entity names
- The defined foreign keys in DBMS

So the columns' parameter is therefore always optional.

### One to one / Many to one

`Hector\Orm\Attributes\HasOne` classes attribute wait some arguments:

1. Class name of target entity (**required**)
2. Name of property (**required**)
3. An associative array of columns between two tables (*optional*)

### One to many

`Hector\Orm\Attributes\HasMany` class attribute wait same arguments as `Hector\Orm\Attributes\HasOne` class.

### Many to many

`Hector\Orm\Attributes\BelongsToMany` class attribute wait some arguments:

1. Class name of target entity (**required**)
2. Name of property (**required**)
3. Pivot table name (*optional*)
4. An associative array of columns between source table and pivot table (*optional*)
5. An associative array of columns between pivot table and target table (*optional*)

**Hector ORM** tries to guess the pivot table name with entity names.

### Inverse

`Hector\Orm\Attributes\BelongsTo` class attribute wait some arguments:

1. Class name of target entity (**required**)
2. Name of property (**required**)
3. Name of target relationship (*optional*)

**Hector ORM** tries to guess the target relationship name based on declared relationships.