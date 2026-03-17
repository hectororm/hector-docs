---
breadcrumb:
  - Integration
  - Berlioz Framework
summary-order: 4;2
keywords:
  - berlioz
  - framework
  - integration
  - package
  - configuration
  - events
  - cli
---

# 🫐 Berlioz Framework

[Berlioz Framework](https://getberlioz.com) provides first-class integration with Hector ORM through the
[`berlioz/hector-package`](https://github.com/BerliozFramework/HectorPackage) package. It handles configuration, cache,
events, debugging and CLI commands automatically.

---

## Installation

```bash
composer require berlioz/hector-package
```

This installs `hectororm/hectororm` (the full ORM distribution) as a dependency.

> ℹ️ For general information about Berlioz packages, see the
> [Berlioz packages documentation](https://getberlioz.com/docs/3.x/guides/packages).

---

## Configuration

Create a `hector.json` file in your
[configuration directory](https://getberlioz.com/docs/3.x/getting-started/directories):

```json
{
    "hector": {
        "dsn": "mysql:dbname=mydb;host=127.0.0.1;port=3306;charset=UTF8",
        "username": "root",
        "password": "secret",
        "schemas": [
            "mydb"
        ]
    }
}
```

> ⚠️ **Warning**: This file contains credentials. Add it to your `.gitignore` file.

### Configuration reference

| Key | Type | Default | Description |
|---|---|---|---|
| `dsn` | `string\|null` | `null` | PDO connection DSN |
| `read_dsn` | `string\|null` | `null` | Read replica DSN (optional) |
| `username` | `string\|null` | `null` | Database username |
| `password` | `string\|null` | `null` | Database password |
| `schemas` | `string[]` | `[]` | Database name(s) to introspect |
| `dynamic_events` | `bool` | `true` | Enable magic event methods on entities |
| `types` | `object` | `{}` | Custom [Data Types](../components/data-types.md) mapping |

---

## What the package provides

Once installed, the package automatically:

- **Registers the `Orm` service** in the container (accessible as `hector` or by type-hint `Hector\Orm\Orm`)
- **Connects the ORM event dispatcher** to Berlioz's event manager
- **Uses the framework cache** for schema metadata (no manual cache setup needed)
- **Converts `NotFoundException`** from entity lookups into HTTP 404 errors
- **Adds a debug page** in the Berlioz debug console (visible in development)
- **Registers CLI commands** (see [below](#-cli-commands))

---

## Usage

Once the package is installed and configured, use Hector ORM as usual — the framework handles bootstrapping:

```php
use Hector\Orm\Entity\MagicEntity;

class User extends MagicEntity {}

// In a controller:
$user = User::findOrFail($id); // Throws NotFoundException → HTTP 404
$user->email = $request->getParsedBody()['email'];
$user->save();
```

For dependency injection, type-hint `Orm` or use the `HectorAwareInterface`:

```php
use Berlioz\HectorPackage\HectorAwareInterface;
use Berlioz\HectorPackage\HectorAwareTrait;
use Hector\Orm\Orm;

class MyService implements HectorAwareInterface
{
    use HectorAwareTrait;

    public function doSomething(): void
    {
        $orm = $this->getHector(); // Returns the Orm instance
    }
}
```

> 💡 **Tip**: For detailed ORM usage (entities, relationships, builder, etc.), see the
> [Hector ORM documentation](../index.md).

---

## Events magic methods

When `dynamic_events` is enabled (default), you can define magic methods directly on your entities to react to
lifecycle events. These methods are resolved through the service container, so **dependency injection is available**.

### Save events

| Method | Called |
|---|---|
| `onBeforeSave(): void` | Before insert or update |
| `onSave(): void` | After insert or update |
| `onAfterSave(): void` | After insert or update |

### Insert events

| Method | Called |
|---|---|
| `onBeforeInsert(): void` | Before insert |
| `onInsert(): void` | After insert |
| `onAfterInsert(): void` | After insert |

### Update events

| Method | Called |
|---|---|
| `onBeforeUpdate(): void` | Before update |
| `onUpdate(): void` | After update |
| `onAfterUpdate(): void` | After update |

### Delete events

| Method | Called |
|---|---|
| `onBeforeDelete(): void` | Before delete |
| `onDelete(): void` | After delete |
| `onAfterDelete(): void` | After delete |

### Example

```php
use Hector\Orm\Entity\MagicEntity;
use Psr\Log\LoggerInterface;

class User extends MagicEntity
{
    // Dependency injection works here!
    public function onBeforeSave(LoggerInterface $logger): void
    {
        $logger->info('About to save user: ' . $this->email);
    }

    public function onAfterDelete(): void
    {
        // Clean up related resources...
    }
}
```

> ℹ️ These magic methods are specific to the Berlioz integration. The underlying ORM uses
> [PSR-14 events](../orm/events.md) — the package bridges both systems automatically.

---

## CLI commands

The package registers the following commands in the Berlioz console:

### `hector:cache-clear`

Clear the Hector ORM schema cache:

```bash
vendor/bin/berlioz hector:cache-clear
```

Use this after database migrations or schema changes in production.

### `hector:generate-schema`

Initialize the ORM and trigger schema generation:

```bash
vendor/bin/berlioz hector:generate-schema
```

This is useful to warm the cache after deployment.

---

## See also

- [Getting started](../index.md) — Quick start with Hector ORM
- [Events](../orm/events.md) — PSR-14 entity lifecycle events
- [Cache](../orm/cache.md) — Schema caching details
- [Framework integration guide](framework.md) — How to build your own integration for another framework
- [Berlioz documentation](https://getberlioz.com/docs/3.x/guides/orm/hector) — Official Berlioz guide for Hector
