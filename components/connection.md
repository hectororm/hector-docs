---
breadcrumb:
  - Components
  - Connection
summary-order: ;4
---

# ⚡️ Connection

> **Note**: While `Connection` are part of the **Hector ORM** ecosystem, they are available as a standalone package:
> [`hectororm/connection`](https://github.com/hectororm/connection).
> You can find it on
> [Packagist](https://packagist.org/packages/hectororm/connection).
> You can use them independently of the ORM, in any PHP application. 🎉

The `hectororm/connection` package provides a lightweight and highly flexible abstraction layer for managing PDO-based
database connections in PHP. It enables clean handling of basic SQL execution, read/write DSN separation, transactions,
and driver introspection, making it suitable for both simple scripts and advanced web applications.

This guide covers how to create and configure connections, run queries, manage transactions, use multiple connections,
introspect driver capabilities, and enable query logging for development and debugging purposes. 🧰

## 🧱 Creating a Connection

The `Connection` class allows you to establish a connection using a DSN (Data Source Name). You can optionally provide
credentials, define a connection name, set up a read-only replica, or inject a logger instance.

### 🏗️ Constructor Parameters

| Parameter  | Type     | Default                    | Description                                          |
|------------|----------|----------------------------|------------------------------------------------------|
| `dsn`      | `string` | —                          | DSN string for write operations (required)           |
| `username` | `string` | `null`                     | Optional database username                           |
| `password` | `string` | `null`                     | Optional database password                           |
| `readDsn`  | `string` | `null`                     | Optional DSN string for read operations              |
| `name`     | `string` | `Connection::DEFAULT_NAME` | Optional connection name                             |
| `logger`   | `Logger` | `null`                     | Optional logger instance to capture executed queries |

### 🔌 Simple Connection

```php
use Hector\Connection\Connection;

$connection = new Connection('mysql:host=localhost;dbname=mydb;user=root;password=secret');
```

You may also pass credentials explicitly as constructor arguments:

```php
$connection = new Connection(
    dsn: 'mysql:host=localhost;dbname=mydb',
    username: 'root',
    password: 'secret'
);
```

### 🐳 Using Secrets in Container Environments

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

### 🧭 Read/Write Separation

```php
$connection = new Connection(
    dsn: 'mysql:host=master.db;dbname=mydb;user=write_user;password=secret',
    readDsn: 'mysql:host=replica.db;dbname=mydb;user=read_user;password=secret'
);
```

Read operations will use the read DSN until a write or transaction begins, after which the write DSN is used
exclusively.

## 🔎 Executing Queries

The `Connection` class provides simple and expressive methods for executing SQL queries and retrieving data.

### Query Methods

| Method                                                          | Description                                     |
|-----------------------------------------------------------------|-------------------------------------------------|
| `execute(string $sql, array $params = [])`                      | Execute a write query (e.g. `INSERT`, `UPDATE`) |
| `fetchOne(string $sql, array $params = [])`                     | Fetch a single row (or `null` if none found)    |
| `fetchAll(string $sql, array $params = [])`                     | Return a generator of all result rows           |
| `fetchColumn(string $sql, array $params = [], int $column = 0)` | Generator over a single column from result set  |
| `yieldAll(string $sql, array $params = [])`                     | Alias of `fetchAll(...)` using a generator      |
| `yieldColumn(string $sql, array $params = [], int $column = 0)` | Alias of `fetchColumn(...)` using a generator   |

### Usage Example

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

### 🆔 Last Insert ID

```php
$connection->execute('INSERT INTO posts (title) VALUES (?)', ['Hello']);
$id = $connection->getLastInsertId();
```

## 🔁 Transactions

Ensure atomic operations using transactions.

### Usage Example

```php
$connection->beginTransaction();
try {
    $connection->execute('INSERT INTO users (name) VALUES (?)', ['Bob']);
    $connection->execute('INSERT INTO profiles (user_id) VALUES (?)', [$connection->getLastInsertId()]);
    $connection->commit();
} catch (\Throwable $e) {
    $connection->rollBack();
    throw $e;
}
```

### Transaction API

| Method               | Description                            |
|----------------------|----------------------------------------|
| `beginTransaction()` | Start a new transaction                |
| `commit()`           | Commit the current transaction         |
| `rollBack()`         | Roll back the current transaction      |
| `inTransaction()`    | Returns `true` if inside a transaction |

> ⚠️ **Note**: Nested calls to `beginTransaction()` are ignored. Each transaction must be matched with a `commit()` or
`rollBack()`.

## 🧩 Managing Multiple Connections

Use the `ConnectionSet` class to register and retrieve multiple named `Connection` instances.

### ConnectionSet API

| Method                                  | Description                                      |
|-----------------------------------------|--------------------------------------------------|
| `addConnection(Connection)`             | Register a `Connection` instance                 |
| `hasConnection(string)`                 | Check if a connection with the given name exists |
| `getConnection(string $name = DEFAULT)` | Retrieve a named connection or the default one   |

### Usage Example

```php
use Hector\Connection\Connection;
use Hector\Connection\ConnectionSet;

$set = new ConnectionSet();
$set->addConnection(new Connection('mysql:host=localhost;dbname=app', name: 'main'));
$set->addConnection(new Connection('mysql:host=replica;dbname=app', name: 'replica'));

$main = $set->getConnection();
$replica = $set->getConnection('replica');
```

## 🪵 Query Logging

To help debug and optimize queries, you can enable logging using the built-in `Logger` class. This logger collects
detailed information for each executed SQL statement, including execution time and stack trace.

> ⚠️ Logging should be disabled in production environments to avoid performance penalties and potential data exposure.

### Logger API

| Class      | Method                   | Description                        |
|------------|--------------------------|------------------------------------|
| `Logger`   | `getLogs(): LogEntry[]`  | Retrieve captured query logs       |
| `LogEntry` | `getStatement(): string` | Executed SQL query                 |
|            | `getParameters(): array` | Parameters bound to the query      |
|            | `getDuration(): float`   | Execution time in milliseconds     |
|            | `getTrace(): array`      | PHP stack trace of query execution |

### Usage Example

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

## 🔍 Driver Information

The `Connection` class can return introspective metadata about the current PDO driver in use.

### DriverInfo API

| Method                 | Description                         |
|------------------------|-------------------------------------|
| `getDriver(): string`  | Name of the underlying PDO driver   |
| `getVersion(): string` | Version string of the driver        |
| `getCapabilities()`    | Returns `DriverCapabilities` object |

### DriverCapabilities API

| Method                       | Description                                 |
|------------------------------|---------------------------------------------|
| `hasLock(): bool`            | Whether SQL `FOR UPDATE` is supported       |
| `hasLockAndSkip(): bool`     | Whether `SKIP LOCKED` is supported          |
| `hasWindowFunctions(): bool` | Whether SQL window functions are supported  |
| `hasJson(): bool`            | Whether native JSON functions are supported |
| `hasStrictMode(): bool`      | Whether strict SQL mode is enforced         |
