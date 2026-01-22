---
breadcrumb:
- Getting started
summary-order: 1
---

# 🚀 Getting Started

## Introduction

**Hector ORM** is a lightweight, framework-agnostic PHP ORM — designed to be modular, fast, and expressive. It draws inspiration from existing ORM concepts, while promoting freedom of structure and strong typing.

### Requirements

- PHP 8.0+
- PDO extension
- Database driver (e.g., `pdo_mysql`, `pdo_sqlite`)

### What is an ORM?

> Object-relational mapping (ORM, O/RM, and O/R mapping tool!) in computer science is a programming technique for converting data between incompatible type systems using object-oriented programming languages. This creates, in effect, a "virtual object database" that can be used from within the programming language.
>
> — [Wikipedia](https://en.wikipedia.org/wiki/Object%E2%80%93relational_mapping)

### ✨ Choose Your Style

You can manage entities in multiple ways:

* [Classic entities](./orm/entity.md): define PHP properties explicitly and **Hector ORM** handles mapping.
* [Magic entities](./orm/entity.md): rely on **Hector ORM** dynamic behavior using PHP’s magic methods.
* Roll your own 🧪: create a custom Mapper if you want total control over mapping logic.

### Why Hector ORM?

| Feature                  | Hector ORM | Doctrine | Eloquent |
|--------------------------|:----------:|:--------:|:--------:|
| Zero config              |     ✅     |    ❌    |    ⚠️   |
| Framework-agnostic       |     ✅     |    ✅    |    ❌   |
| Magic + Classic entities |     ✅     |    ❌    |    ✅   |
| Schema introspection     |     ✅     |    ❌    |    ❌   |

## Quick Start

### 1. 📦 Installation

Install with [Composer](https://getcomposer.org/):

```bash
composer require hectororm/hectororm
```

### 2. Create a Database Connection

```php
use Hector\Connection\Connection;

$connection = new Connection('mysql:host=localhost;dbname=test', 'user', 'pass');
```

### 3. Boot the ORM

```php
use Hector\Orm\OrmFactory;

$orm = OrmFactory::orm([
    'schemas' => ['my-schema'] // Database name(s) to introspect
], $connection);
```

### 4. 🧱 Define Entities

```php
use Hector\Orm\Attributes as Orm;
use Hector\Orm\Entity\MagicEntity;

#[Orm\HasOne(Bar::class, 'bar')]
class Foo extends MagicEntity {}

#[Orm\BelongsTo(Foo::class, 'foo')]
class Bar extends MagicEntity {}
```

### 5. 💡 Use the ORM

```php
$foo = Foo::findOrFail(1); // find a Foo entity by primary key
$bar = $foo->bar; // access related Bar entity

echo $bar->field; // access a field from the related Bar
```

---

You're now ready to build with **Hector ORM**. Keep exploring:

* [Entity definition](./orm/entity.md)
* [Working with relationships](./orm/relationships.md)
* [Advanced configuration](./orm/configuration.md)

Happy mapping 🗺️
