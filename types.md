```index
breadcrumb: Data types
summary-order: 4
```

# Data types

## Declaration with PHP attribute

Data types must be declared with attribute `Hector\Orm\Attributes\Type`.

Arguments of attribute:

1. **column**: name of database column
2. **type**: class name of type
3. *additional arguments specific to the data type*


## List of default types

All default types are in namespace: `Hector\DataTypes\Type`.

| Type           | Description                                     | Additional arguments                                                                                                               |
|----------------|-------------------------------------------------|------------------------------------------------------------------------------------------------------------------------------------|
| `BooleanType`  | Integer from database to PHP boolean            | *None*                                                                                                                             |
| `DateTimeType` | Date/time from database to PHP DateTime         | `string $format = 'Y-m-d H:i:s'` format of date time ; `string $class = DateTime::class` class name of DateTimeInterface generated |
| `EnumType`     | String/integer from database to PHP Enum        | `string $enum` enum class name                                                                                                     |
| `JsonType`     | JSON string from database to PHP array/stdClass | `bool $associative = true` to convert to associative array or `stdClass`                                                           |
| `NumericType`  | Numeric from database to PHP numeric int/float  | `string $type = 'int'` PHP type expected                                                                                           |
| `SetType`      | Set of value from database to PHP array         | *None*                                                                                                                             |
| `StringType`   | Data from database to PHP string                | *None*                                                                                                                             |

Example:

```php
use Hector\DataTypes\Type\EnumType;
use Hector\Orm\Attributes as Orm;
use Hector\Orm\Entity\MagicEntity;

#[Orm\Type(column: 'column', type: EnumType::class, enum: MyEnum::class)]
class Foo extends MagicEntity {
}

enum MyEnum: string {
    case FOO = 'foo';
    case BAR = 'bar';
    // ...
}
```

## Usage

After declaration of types on entities, the type of variables are automatically converted. No action needed.

## Create of your own type

To create your own type, you need to implement interface `Hector\DataTypes\Type\TypeInterface`.
The abstract class `Hector\DataTypes\Type\AbstractType` can help you to be faster.

### Example

Expected type:

```php
class Foo {
    public function __construct(
        private string $foo,
    ) {
    }
    
    public function getValue(): string
    {
        return $this->foo;
    }
}
```

Hector type class:

```php
use Hector\DataTypes\Type\AbstractType;

class FooType extends AbstractType {
    /**
     * @inheritDoc
     */
    public function fromSchema(mixed $value, ?ExpectedType $expected = null): mixed
    {
        return new Foo($value);
    }

    /**
     * @inheritDoc
     */
    public function toSchema(mixed $value, ?ExpectedType $expected = null): mixed
    {
        return $value->getValue();
    }
}
```
