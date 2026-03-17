---
breadcrumb:
  - ORM
  - Events
summary-order: ;4
keywords:
  - events
  - lifecycle
  - psr-14
  - save
  - delete
  - listener
---

# 📡 Events

**Hector ORM** dispatches [PSR-14](https://www.php-fig.org/psr/psr-14/) events during the entity lifecycle. This allows
you to hook into save and delete operations — for example, to add audit logging, enforce validation rules, or prevent
an operation from executing.

---

## Setup

Pass a PSR-14 `EventDispatcherInterface` when bootstrapping the ORM:

```php
use Hector\Orm\OrmFactory;
use Psr\EventDispatcher\EventDispatcherInterface;

/** @var EventDispatcherInterface $eventDispatcher */
$orm = OrmFactory::orm(
    options: ['schemas' => ['my_database']],
    connection: $connection,
    eventDispatcher: $eventDispatcher,
);
```

If no event dispatcher is provided, events are silently ignored (zero overhead).

---

## Event reference

| Event                     | Dispatched                          | Stoppable | Extra data   |
|---------------------------|-------------------------------------|:---------:|--------------|
| `EntityBeforeSaveEvent`   | Before an insert or update          |     ✔     | `isUpdate()` |
| `EntityAfterSaveEvent`    | After a successful insert or update |     —     | `isUpdate()` |
| `EntityBeforeDeleteEvent` | Before a delete                     |     ✔     | —            |
| `EntityAfterDeleteEvent`  | After a successful delete           |     —     | —            |

All events extend `EntityEvent` and carry:

| Method        | Return type         | Description                       |
|---------------|---------------------|-----------------------------------|
| `getEntity()` | `Entity`            | The entity being saved or deleted |
| `getTime()`   | `DateTimeImmutable` | Timestamp of the event            |

Save events (`EntityBeforeSaveEvent` and `EntityAfterSaveEvent`) also carry:

| Method       | Return type | Description                                                           |
|--------------|-------------|-----------------------------------------------------------------------|
| `isUpdate()` | `bool`      | `true` if updating an existing entity, `false` if inserting a new one |

---

## Stoppable events

`EntityBeforeSaveEvent` and `EntityBeforeDeleteEvent` implement `StoppableEventInterface`. If a listener calls
`stopPropagation()`, the operation is **cancelled** — the entity is not saved or deleted.

```php
use Hector\Orm\Event\EntityBeforeSaveEvent;

$listener = function (EntityBeforeSaveEvent $event): void {
    $entity = $event->getEntity();

    // Prevent saving if validation fails
    if (empty($entity->email)) {
        $event->stopPropagation();
    }
};
```

> ⚠️ **Warning**: When propagation is stopped, the entity remains in its current state in the storage. No database
> operation is performed, and no "after" event is dispatched.

---

## Examples

### Audit logging

```php
use Hector\Orm\Event\EntityAfterSaveEvent;
use Hector\Orm\Event\EntityAfterDeleteEvent;

$onSave = function (EntityAfterSaveEvent $event): void {
    $action = $event->isUpdate() ? 'updated' : 'created';
    $class = get_class($event->getEntity());

    $logger->info("Entity {$class} was {$action}");
};

$onDelete = function (EntityAfterDeleteEvent $event): void {
    $class = get_class($event->getEntity());

    $logger->info("Entity {$class} was deleted");
};
```

### Automatically setting timestamps

```php
use Hector\Orm\Event\EntityBeforeSaveEvent;

$listener = function (EntityBeforeSaveEvent $event): void {
    $entity = $event->getEntity();

    if (!$event->isUpdate() && property_exists($entity, 'createdAt')) {
        $entity->createdAt = new DateTimeImmutable();
    }

    if (property_exists($entity, 'updatedAt')) {
        $entity->updatedAt = new DateTimeImmutable();
    }
};
```

### Preventing deletion

```php
use Hector\Orm\Event\EntityBeforeDeleteEvent;

$listener = function (EntityBeforeDeleteEvent $event): void {
    $entity = $event->getEntity();

    // Soft-delete: mark as deleted instead of actually deleting
    if (property_exists($entity, 'deletedAt')) {
        $entity->deletedAt = new DateTimeImmutable();
        $entity->save();
        $event->stopPropagation();
    }
};
```

---

## Class hierarchy

```
AbstractEvent
├── EntityEvent
│   ├── EntitySaveEvent
│   │   ├── EntityBeforeSaveEvent    (stoppable)
│   │   └── EntityAfterSaveEvent
│   └── EntityDeleteEvent
│       ├── EntityBeforeDeleteEvent  (stoppable)
│       └── EntityAfterDeleteEvent
```

All classes live in the `Hector\Orm\Event` namespace.
