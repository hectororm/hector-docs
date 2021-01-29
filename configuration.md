# Advanced configuration

## Specify the table or schema name

In some case, your entity has not the same name than the datable table.

With attribute `Hector\Orm\Attributes\Table` you can specify names. Specify the name of schema is only necessary when
you use multiple databases with tables with same name.

Example:

```php
use Hector\Orm\Attributes as Orm;
use Hector\Orm\Entity\MagicEntity;

#[Orm\Table('table_name', 'schema_name')]
class Foo extends MagicEntity {
}
```

In this example, Foo entity used the database "**schema_name**" and table "**table_name**".