---
breadcrumb:
- Components
- Data Types
summary-order: ;3
---

# 🎯 Data types

> ℹ️ **Note**: While data types are part of the **Hector ORM** ecosystem, they are available as a standalone package:
> [`hectororm/data-types`](https://github.com/hectororm/data-types).
> You can find it on
> [Packagist](https://packagist.org/packages/hectororm/data-types).
> You can use them independently of the ORM, in any PHP application. 🎉

Data types allow automatic conversion between database values and PHP objects or primitives. This abstraction
facilitates cleaner entity code and ensures consistent handling of complex or custom types, such as enums, UUIDs, or
JSON structures.

Each type supports **bidirectional conversion**:

* From **database → PHP**: during hydration
* From **PHP → database**: during insertion/update

---

## 🔍 Overview

| Type             | Description                              | Key Arguments           |
|------------------|------------------------------------------|-------------------------|
| `BooleanType`    | Integer to PHP boolean                   | –                       |
| `DateTimeType`   | Date string to `DateTimeInterface`       | `format`, `class`       |
| `EnumType`       | DB value to PHP `enum`                   | `enum`, `try`           |
| `JsonType`       | JSON string to array/stdClass/scalar     | `associative`           |
| `NumericType`    | DB number to PHP int/float               | `type`                  |
| `SetType`        | Comma-separated values to array          | –                       |
| `StringType`     | String with optional truncation/encoding | `maxlength`, `encoding` |
| `UuidType`       | UUID as string                           | `storage`               |
| `RamseyUuidType` | UUID as `Ramsey\Uuid\UuidInterface`      | `storage`               |

---

## 🧩 Built-in types

### BooleanType

`#[Orm\Type(column: 'is_active', type: BooleanType::class)]`

Converts `0`/`1` values from the database to PHP booleans and vice versa.

*No arguments.*

#### Conversion

| Database value | PHP value |
|----------------|-----------|
| `0`            | `false`   |
| `1`            | `true`    |

### DateTimeType

`#[Orm\Type(column: 'created_at', type: DateTimeType::class, format: 'Y-m-d H:i:s', class: DateTimeImmutable::class)]`

Converts formatted date/time strings to `DateTimeInterface` instances and back.

#### Arguments

| Name   | Description                            | Required | Default           |
|--------|----------------------------------------|----------|-------------------|
| format | Format of datetime string              | No       | `'Y-m-d H:i:s'`   |
| class  | `DateTimeInterface`-implementing class | No       | `DateTime::class` |

#### Conversion

| Database value          | PHP value                                  |
|-------------------------|--------------------------------------------|
| `'2025-07-04 09:00:00'` | `DateTimeImmutable('2025-07-04 09:00:00')` |
| `'invalid-date'`        | `Exception`                                |

### EnumType

`#[Orm\Type(column: 'status', type: EnumType::class, enum: Status::class, try: true)]`

Maps string or integer values from the database to backed PHP enums.

> ⚠️ **Warning**: Requires PHP 8.1+ and a [backed enum](https://www.php.net/manual/en/language.enumerations.backed.php) (with `string` or `int` backing type).

#### Arguments

| Name | Description                         | Required | Default |
|------|-------------------------------------|----------|---------|
| enum | Backed enum class name              | Yes      | –       |
| try  | Use `tryFrom()` instead of `from()` | No       | `false` |

> 💡 **Tip**: Use `VARCHAR`, `TEXT`, or `INT` columns depending on your enum backing type

> ⚠️ **Warning**: With `try = false`, invalid DB values will throw a `ValueError`

#### Conversion

| Database value | PHP value       | `try = false` behavior | `try = true` behavior |
|----------------|-----------------|------------------------|-----------------------|
| `'draft'`      | `Status::DRAFT` | OK                     | OK                    |
| `'invalid'`    | Exception       | `ValueError`           | `null`                |

### JsonType

`#[Orm\Type(column: 'meta', type: JsonType::class, associative: true)]`

Converts JSON strings into PHP arrays, objects (`stdClass`), or scalars.

#### Arguments

| Name        | Description                                  | Required | Default |
|-------------|----------------------------------------------|----------|---------|
| associative | Decode as array (`true`) or object (`false`) | No       | `true`  |

#### Conversion

| Database JSON     | `associative = true` | `associative = false`      |
|-------------------|----------------------|----------------------------|
| `"hello"`         | `'hello'`            | `'hello'`                  |
| `'123'`           | `123`                | `123`                      |
| `'null'`          | `null`               | `null`                     |
| `'[1,2,3]'`       | `[1,2,3]`            | `[1,2,3]`                  |
| `'{"foo":"bar"}'` | `['foo' => 'bar']`   | `(object)['foo' => 'bar']` |

### NumericType

`#[Orm\Type(column: 'price', type: NumericType::class, numericType: 'float')]`

Casts numeric string values from the database into PHP `int` or `float`.

#### Arguments

| Name        | Description     | Required | Default |
|-------------|-----------------|----------|---------|
| numericType | Target PHP type | No       | `'int'` |

#### Conversion

| Database value | `numericType = 'int'` | `numericType = 'float'` |
|----------------|-----------------------|-------------------------|
| `'42'`         | `42`                  | `42.0`                  |
| `'3.14'`       | `3`                   | `3.14`                  |

### SetType

`#[Orm\Type(column: 'tags', type: SetType::class)]`

Converts comma-separated strings into PHP arrays.

*No arguments.*

#### Conversion

| Database value      | PHP value          |
|---------------------|--------------------|
| `'php,mysql'`       | `['php', 'mysql']` |
| `''` (empty string) | `[]`               |

### StringType

`#[Orm\Type(column: 'title', type: StringType::class, maxlength: 255, encoding: 'UTF-8')]`

Converts database text fields to PHP strings, with optional truncation or encoding.

#### Arguments

| Name      | Description                     | Required | Default |
|-----------|---------------------------------|----------|---------|
| maxlength | Max length for truncation       | No       | `null`  |
| encoding  | Encoding used during truncation | No       | `null`  |

#### Conversion

| Database value    | `maxlength = 5` | PHP value |
|-------------------|-----------------|-----------|
| `'abcdef'`        | `'abcde'`       | `'abcde'` |
| `'✓✓✓✓✓'` (UTF-8) | `'✓✓✓✓✓'`       | `'✓✓✓✓✓'` |

### UuidType

`#[Orm\Type(column: 'uuid', type: UuidType::class, storage: 'binary')]`

Converts UUIDs from and to PHP strings.

#### Arguments

| Name    | Description                   | Required | Default    |
|---------|-------------------------------|----------|------------|
| storage | Format in DB (`binary`, etc.) | No       | `'string'` |

#### Storage guide

| Storage       | Column type  | Length | Notes                      |
|---------------|--------------|--------|----------------------------|
| `binary`      | `BINARY(16)` | 16     | Compact, non-readable      |
| `string`      | `CHAR(36)`   | 36     | RFC 4122 with hyphens      |
| `hexadecimal` | `CHAR(32)`   | 32     | No hyphens, hex characters |

#### Conversion

| Storage       | Database value                           | PHP string UUID                          |
|---------------|------------------------------------------|------------------------------------------|
| `binary`      | `\xB4...` (16 bytes)                     | `'b40c5c6a-e2a4-4b5c-990b-5feacdbdbe68'` |
| `string`      | `'b40c5c6a-e2a4-4b5c-990b-5feacdbdbe68'` | `'b40c5c6a-e2a4-4b5c-990b-5feacdbdbe68'` |
| `hexadecimal` | `'b40c5c6ae2a44b5c990b5feacdbdbe68'`     | `'b40c5c6a-e2a4-4b5c-990b-5feacdbdbe68'` |

### RamseyUuidType

`#[Orm\Type(column: 'uuid', type: RamseyUuidType::class, storage: 'hexadecimal')]`

Converts UUIDs from and to `Ramsey\Uuid\UuidInterface` instances.

> ⚠️ **Warning**: Requires the [`ramsey/uuid`](https://github.com/ramsey/uuid) package.

#### Arguments

| Name    | Description                   | Required | Default    |
|---------|-------------------------------|----------|------------|
| storage | Format in DB (`binary`, etc.) | No       | `'string'` |

#### Storage guide

(see UuidType)

#### Conversion

| DB (hexadecimal)                     | PHP value               |
| ------------------------------------ | ----------------------- |
| `'b40c5c6ae2a44b5c990b5feacdbdbe68'` | `Uuid::fromString(...)` |

---

## 🧑‍💻 Custom type

Define your own data type by implementing `Hector\DataTypes\Type\TypeInterface`, or extending
`Hector\DataTypes\Type\AbstractType`.

`#[Orm\Type(column: 'foo_column', type: FooType::class, prefix: 'bar_')]`

#### Arguments

| Name   | Description                                     | Required | Default |
|--------|-------------------------------------------------|----------|---------|
| column | Name of the database column                     | Yes      | –       |
| type   | Type class implementing the logic               | Yes      | –       |
| ...    | Additional arguments passed to type constructor | No       | –       |

#### Example implementation

```php
class Foo {
    public function __construct(private string $foo) {}
    public function getValue(): string { return $this->foo; }
}
```

```php
use Hector\DataTypes\Type\AbstractType;
use Hector\DataTypes\ExpectedType;

class FooType extends AbstractType {
    public function __construct(private string $prefix = '') {}

    public function fromSchema(mixed $value, ?ExpectedType $expected = null): mixed {
        return new Foo($this->prefix . $value);
    }

    public function toSchema(mixed $value, ?ExpectedType $expected = null): mixed {
        $raw = $value->getValue();
        // Remove prefix if present
        if (str_starts_with($raw, $this->prefix)) {
            $raw = substr($raw, strlen($this->prefix));
        }
        return $raw;
    }
}
```

Usage:

```php
use Hector\Orm\Attributes as Orm;
use Hector\Orm\Entity\Entity;

#[Orm\Type(column: 'code', type: FooType::class, prefix: 'CODE_')]
class Product extends Entity {
    public Foo $code;
}
```

---

## 🛠️ Global best practices

* ✅ Always match DB column names to declared types.
* ✅ Validate that your database contains values compatible with the declared type.
* ⚠️ Attribute must be declared at the **class level**, not on properties.
