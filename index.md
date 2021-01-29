```index
breadcrumb: Getting started
summary-order: 1
```

# Getting started

## Introduction

**Hector ORM** is a PHP ORM independent of any framework and inspired by others ORM functionalities.

### ORM definition

> Object-relational mapping (ORM, O/RM, and O/R mapping tool!) in computer science is a programming technique for converting data between incompatible type systems using object-oriented programming languages. This creates, in effect, a "virtual object database" that can be used from within the programming language. There are both free and commercial packages available that perform object-relational mapping, although some programmers opt to construct their own ORM tools.
> 
> Source: [Wikipedia](https://en.wikipedia.org/wiki/Object%E2%80%93relational_mapping)

### Ways

2 ways to manage entities:

- [Classic entity](./entity/classic.md): you need to declare properties on your entities, and **Hector ORM** based on to do relation with DBMS 
- [Magic entities](./entity/magic.md): all is managed by **Hector ORM** and uses magic properties of PHP

Another way is possible, it's your way, create yourself Mapper to imagine your entity management.


## Quick start

### Installation

Installation of ORM is easy with [Composer](https://getcomposer.org/):

```bash
composer install hectororm/orm
```

### Usage

1. Create connection
   
   ```php
   $connection = new Hector\Connection\Connection('dsn');
   ```

2. Create ORM object

   ```php
   $orm = Hector\Orm\OrmFactory::orm(['schemas' => 'my-schema'], $connection);
   ```

3. Creates your entities
   
   ```php
   use Hector\Orm\Attributes as Orm;
   use Hector\Orm\Entity\MagicEntity;
   
   #[Orm\HasOne(Bar::class, 'bar')]
   class Foo extends MagicEntity {
   }
   
   #[Orm\BelongsTo(Foo::class, 'foo')]
   class Bar {
   }
   ```

4. Uses your entities
   
   ```php
   $entity = Foo::findOrFail(1);
   $foreignEntity = $entity->bar;
   
   print $foreignEntity->field;
   ```

