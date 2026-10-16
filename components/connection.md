---
breadcrumb:
  - Components
  - Connection
summary-order: ;4
keywords:
  - connection
  - pdo
  - database
  - transaction
  - read-write
  - logging
---

# ⚡️ Connection

> ℹ️ **Note**: While the Connection component is part of the **Hector ORM** ecosystem, it is available as a standalone
> package: [`hectororm/connection`](https://github.com/hectororm/connection).
> You can find it on [Packagist](https://packagist.org/packages/hectororm/connection).
> You can use it independently of the ORM, in any PHP application. 🎉

The `hectororm/connection` package provides a lightweight and highly flexible abstraction layer for managing PDO-based
database connections in PHP. It enables clean handling of basic SQL execution, read/write DSN separation, transactions,
and driver introspection, making it suitable for both simple scripts and advanced web applications.

This guide covers how to create and configure connections, run queries, manage transactions, use multiple connections,
introspect driver capabilities, and enable query logging for development and debugging purposes. 🧰

## Creating a connection

The `Connection` class allows you to establish a connection using a DSN (Data Source Name). You can optionally provide
credentials, define a connection name, set up a read-only replica, or inject a logger instance.

### Constructor parameters

| Parameter  | Type     | Default     | Description                                          |
|------------|----------|-------------|------------------------------------------------------|
| `dsn`      | `string` | —           | DSN string for write operations (required)           |
| `username` | `string` | `null`      | Optional database username                           |
| `password` | `string` | `null`      | Optional database password                           |
| `readDsn`  | `string` | `null`      | Optional DSN string for read operations              |
| `name`     | `string` | `'default'` | Optional connection name                             |
| `logger`   | `Logger` | `null`      | Optional logger instance to capture executed queries |
| `options`  | `array`  | `[]`        | Driver-specific PDO options (since 1.5)              |

### Simple connection

```php
use Hector\Connection\Connection;

$connection = new Connection(
    dsn: 'mysql:host=localhost;dbname=mydb',
    username: 'root',
    password: 'secret'
);
```

### Using secrets in container environments

```php
$dsn = 'mysql:host=db;dbname=app';
$username = trim(file_get_contents('/run/secrets/db_user'));
$password = trim(file_get_contents('/run/secrets/db_pass'));

$connection = new Connection(
    dsn: $dsn,
    username: $username,
    password: $password
);
```

### Read/write separation

```php
$connection = new Connection(
    dsn: 'mysql:host=master.db;dbname=mydb;user=write_user;password=secret',
    readDsn: 'mysql:host=replica.db;dbname=mydb;user=read_user;password=secret'
);
```

Read operations will use the read DSN until a write or transaction begins, after which the write DSN is used
exclusively.

### PDO options

> 🆕 **Info**: *Since version 1.5*

`options` is passed as the `$options` argument of the PDO constructor. The same options apply to the read/write and
read-only connections, and are preserved when the connection is serialized.

```php
$connection = new Connection(
    dsn: 'mysql:host=localhost;dbname=mydb',
    username: 'root',
    password: 'secret',
    options: [
        PDO::ATTR_TIMEOUT => 5,
        // Driver-specific attributes are accepted as well (e.g. SSL options of pdo_mysql)
    ],
);
```

> ⚠️ **Warning**: Options such as `PDO::ATTR_AUTOCOMMIT` or `PDO::ATTR_ERRMODE` may interfere with the internal
> transaction handling and should not be altered lightly.

---

## Executing queries

The `Connection` class provides simple and expressive methods for executing SQL queries and retrieving data.

### Query methods

| Method                                                          | Description                                     |
|-----------------------------------------------------------------|-------------------------------------------------|
| `execute(string $sql, array $params = [])`                      | Execute a write query (e.g. `INSERT`, `UPDATE`) |
| `fetchOne(string $sql, array $params = [])`                     | Fetch a single row (or `null` if none found)    |
| `fetchAll(string $sql, array $params = [])`                     | Return a generator of all result rows           |
| `fetchColumn(string $sql, array $params = [], int $column = 0)` | Generator over a single column from result set  |
| `yieldAll(string $sql, array $params = [])`                     | Alias of `fetchAll(...)` using a generator      |
| `yieldColumn(string $sql, array $params = [], int $column = 0)` | Alias of `fetchColumn(...)` using a generator   |

### Usage example

```php
$affected = $connection->execute(
    'UPDATE users SET name = ? WHERE id = ?',
    ['Alice', 1]
);

$user = $connection->fetchOne('SELECT * FROM users WHERE id = ?', [1]);

foreach ($connection->fetchAll('SELECT * FROM users') as $row) {
    // Process each row
}

foreach ($connection->fetchColumn('SELECT email FROM users') as $email) {
    // Process each email address
}
```

### Last insert ID

```php
$connection->execute('INSERT INTO posts (title) VALUES (?)', ['Hello']);
$id = $connection->getLastInsertId();
```

---

## Transactions

Ensure atomic operations using transactions.

### Transactional callback

> 🆕 **Info**: *Since version 1.6*

`transaction()` runs a callback inside a transaction and returns its result. The callback receives the connection.
On success, the transaction is committed. On any exception, it is rolled back and the original exception is rethrown.

```php
$userId = $connection->transaction(function (Connection $connection): string {
    $connection->execute('INSERT INTO users (name) VALUES (?)', ['Bob']);
    $userId = $connection->getLastInsertId();
    $connection->execute('INSERT INTO profiles (user_id) VALUES (?)', [$userId]);

    return $userId;
});
```

#### Nested transactions

When a transaction is already active, `transaction()` uses a savepoint. A failing nested callback only discards its
own work: the outer transaction stays active and can still be committed.

```php
$connection->transaction(function (Connection $connection): void {
    $connection->execute('INSERT INTO orders (ref) VALUES (?)', ['A-1']);

    try {
        $connection->transaction(function (Connection $connection): void {
            $connection->execute('INSERT INTO order_notes (ref) VALUES (?)', ['A-1']);
            throw new RuntimeException('Optional step failed');
        });
    } catch (RuntimeException) {
        // Only the note is rolled back
    }
});
// The order is committed
```

This also applies to a transaction opened with `beginTransaction()` or directly on the PDO object.

> ⚠️ **Warning**: DDL statements (`CREATE`, `ALTER`, `DROP`, ...) trigger an implicit `COMMIT` on MySQL/MariaDB, which
> also discards savepoints. Work done before the DDL statement is committed and cannot be rolled back.

#### Retrying on concurrency failures

The `attempts` parameter retries the whole transaction when it fails with a transient concurrency error: deadlock,
lock wait timeout, serialization failure or busy/locked SQLite database. Other exceptions are rethrown immediately,
and the last exception is rethrown when all attempts fail.

```php
$connection->transaction(function (Connection $connection): void {
    $connection->execute('UPDATE accounts SET balance = balance - 100 WHERE id = ?', [1]);
    $connection->execute('UPDATE accounts SET balance = balance + 100 WHERE id = ?', [2]);
}, attempts: 3);
```

Retries happen without delay. Nested transactions are never retried: these failures usually abort the outer
transaction, which is the one to retry.

> ⚠️ **Warning**: The callback is executed again on each attempt. Keep side effects outside the database (emails, HTTP
> calls, files...) out of the callback, or make them idempotent.

To build your own retry policy (delay, logging...), use `DriverInfo::isRetryable()`, which detects these errors for the
current driver, including when the PDO exception is wrapped by another exception:

```php
try {
    $connection->transaction($work);
} catch (\Throwable $exception) {
    if (false === $connection->getDriverInfo()->isRetryable($exception)) {
        throw $exception;
    }
    // Schedule a new attempt
}
```

| Driver          | Retryable errors                                                     |
|-----------------|----------------------------------------------------------------------|
| MySQL / MariaDB | Deadlock (`1213`), lock wait timeout (`1205`), SQLSTATE `40001`      |
| PostgreSQL      | Serialization failure (`40001`), deadlock (`40P01`)                  |
| SQLite          | `SQLITE_BUSY` (`5`), `SQLITE_LOCKED` (`6`), including extended codes |
| Others          | SQLSTATE `40001`                                                     |

### Manual transactions

```php
$connection->beginTransaction();
try {
    $connection->execute('INSERT INTO users (name) VALUES (?)', ['Bob']);
    $connection->execute('INSERT INTO profiles (user_id) VALUES (?)', [$connection->getLastInsertId()]);
    $connection->commit();
} catch (\Throwable $e) {
    $connection->rollBack(allTransactions: false);
    throw $e;
}
```

Since 1.6, nested `beginTransaction()` calls create savepoints, and `commit()` releases the current savepoint.
Only the outermost Hector-owned transaction is actually committed. `transaction()` uses these same methods.

Choose the rollback scope explicitly:

```php
$connection->rollBack(allTransactions: false); // Roll back only the current level
$connection->rollBack(allTransactions: true);  // Roll back all Hector-managed levels
```

With only one Hector-managed level, both options have the same effect. If the transaction was opened directly on PDO,
`beginTransaction()` creates a savepoint even at the first Hector level: committing or rolling back Hector's levels
preserves the caller's PDO transaction.

After an implicit DDL commit, `commit()` and `rollBack()` clear the obsolete nesting state without issuing transaction
commands; the next `beginTransaction()` opens a real transaction.

> **Migration to 2.0:** `$allTransactions = null` (the default) means `true` in 1.x and will mean `false` in 2.0.
> Omitting it while several Hector transaction levels are active emits `E_USER_DEPRECATED` since 1.6.
> Pass `allTransactions: true` to preserve full rollback, or `false` to isolate the current level.
> `transaction()` already passes `false` explicitly.

### Transaction API

| Method                                           | Description                                               |
|--------------------------------------------------|-----------------------------------------------------------|
| `transaction(callable $work, int $attempts = 1)` | Run the callback in a transaction (savepoint when nested) |
| `beginTransaction()`                             | Start a new transaction                                   |
| `commit()`                                       | Commit the current transaction                            |
| `rollBack(?bool $allTransactions = null)`          | Roll back all levels or only the current one               |
| `inTransaction()`                                | Returns `true` if inside a transaction                    |

Match each `beginTransaction()` with a `commit()` or `rollBack(allTransactions: false)`.
`rollBack(allTransactions: true)` closes all Hector-managed levels at once.

---

## Managing multiple connections

Use the `ConnectionSet` class to register and retrieve multiple named `Connection` instances.

### ConnectionSet API

| Method                                           | Description                                                       |
|--------------------------------------------------|-------------------------------------------------------------------|
| `addConnection(Connection)`                      | Register a `Connection` instance                                  |
| `hasConnection(string)`                          | Check if a connection with the given name exists                  |
| `getConnection(string $name = DEFAULT)`          | Retrieve a named connection or the default one                    |
| `transaction(callable $work, int $attempts = 1)` | Run the callback in a transaction on every connection (since 1.6) |
| `rollBack(?bool $allTransactions = null)`          | Forward the rollback scope to every connection                    |

> ⚠️ **Warning**: `ConnectionSet::transaction()` is not a distributed transaction. Connections are committed one after
> the other: if a commit fails, the connections already committed are not rolled back.

With `attempts`, the transaction is retried on every connection when any of them reports a retryable failure. No
retry happens if a transaction was already active on one of the connections.

### Usage example

```php
use Hector\Connection\Connection;
use Hector\Connection\ConnectionSet;

$set = new ConnectionSet();
$set->addConnection(new Connection('mysql:host=localhost;dbname=app', name: 'default'));
$set->addConnection(new Connection('mysql:host=replica;dbname=app', name: 'replica'));

// Get default connection (name = 'default')
$main = $set->getConnection();

// Get named connection
$replica = $set->getConnection('replica');

// Check existence
$set->hasConnection('replica'); // true
$set->hasConnection('unknown'); // false
```

---

## Query logging

To help debug and optimize queries, you can enable logging using the built-in `Logger` class. This logger collects
detailed information for each executed SQL statement, including execution time and stack trace.

> ⚠️ **Warning**: Logging should be disabled in production environments to avoid performance penalties and potential
> data exposure.

### Logger API

| Class      | Method                   | Description                        |
|------------|--------------------------|------------------------------------|
| `Logger`   | `getLogs(): LogEntry[]`  | Retrieve captured query logs       |
| `LogEntry` | `getStatement(): string` | Executed SQL query                 |
|            | `getParameters(): array` | Parameters bound to the query      |
|            | `getDuration(): float`   | Execution time in milliseconds     |
|            | `getTrace(): array`      | PHP stack trace of query execution |

### Usage example

```php
use Hector\Connection\Log\Logger;

$logger = new Logger();
$connection = new Connection('mysql:host=localhost;dbname=app', logger: $logger);
$connection->fetchOne('SELECT * FROM users WHERE id = ?', [1]);

foreach ($logger->getLogs() as $log) {
    echo $log->getStatement();
    print_r($log->getParameters());
    echo $log->getDuration() . "ms\n";
}
```

---

## Driver information

The `Connection` class can return introspective metadata about the current PDO driver in use via `getDriverInfo()`.

```php
$driverInfo = $connection->getDriverInfo();

echo $driverInfo->getDriver();   // e.g. 'mysql'
echo $driverInfo->getVersion();  // e.g. '8.0.32'

$capabilities = $driverInfo->getCapabilities();
$capabilities->hasJson();        // true
```

### DriverInfo API

| Method                                | Description                                                      |
|---------------------------------------|------------------------------------------------------------------|
| `getDriver(): string`                 | Name of the underlying PDO driver                                |
| `getVersion(): string`                | Version string of the driver                                     |
| `getCapabilities()`                   | Returns `DriverCapabilities` object                              |
| `isRetryable(Throwable $error): bool` | Whether the error is a transient concurrency failure (since 1.6) |

### DriverCapabilities API

| Method                       | Description                                 |
|------------------------------|---------------------------------------------|
| `hasLock(): bool`            | Whether SQL `FOR UPDATE` is supported       |
| `hasLockAndSkip(): bool`     | Whether `SKIP LOCKED` is supported          |
| `hasWindowFunctions(): bool` | Whether SQL window functions are supported  |
| `hasJson(): bool`            | Whether native JSON functions are supported |
| `hasStrictMode(): bool`      | Whether strict SQL mode is enforced         |
| `hasCheckConstraints(): bool` | Whether the engine/version enforces CHECK constraints (since 1.5) |
| `hasDropCheck(): bool` | Whether native `ALTER TABLE ... DROP CHECK` syntax is supported (since 1.5) |

**Interface change:** third-party `DriverCapabilities` implementations must implement both CHECK methods.
Return `false` when support cannot be established.

MySQL **8.0.16+** returns `true` for both. MariaDB **10.2.1+**, SQLite and PostgreSQL return `true` for CHECK
support and `false` for `DROP CHECK`: MariaDB/PostgreSQL use `DROP CONSTRAINT`, while SQLite requires a rebuild.
Older MySQL/MariaDB versions, Vitess and unknown engines return `false` for both.
These capabilities describe engine support, not session overrides. PostgreSQL has native support but no Hector
schema-plan compiler. See [CHECK compilation](plan.md#check-constraints) for online/offline usage.

Since **1.5**, the concrete `Hector\Connection\Driver\SQLiteCapabilities` class also exposes
`hasGeneratedColumns(): bool`, which returns `true` for SQLite **3.31.0 or newer**. This method is specific to SQLite
capabilities and is not part of the common `DriverCapabilities` interface.

---

## Installation

```bash
composer require hectororm/connection
```
