# PHP Extended SQL

PHP Extended SQL is an alternative to the also-known DQL (Doctrine Query Language). It combines the flexibility of SQL with the powerful Doctrine metadata to give you more control over queries.

```php
<?php
use App\Car;
use App\Model;
use Soyuka\ESQL\Bridge\Doctrine\ESQL;
use Soyuka\ESQL\Bridge\Doctrine\ESQLMapper;

$connection = $managerRegistry->getConnection();
$esql = new ESQL($managerRegistry)
[
  'table' => $table,
  'identifier' => $identifier,
  'columns' => $columns,
  'join' => $join
] = $esql(Car::class);
['table' => $modelTable, 'columns' => $modelColumns] = $esql(Model::class);

$query = <<<SQL
SELECT {$columns()}, {$modelColumns()} FROM {$table} 
INNER JOIN {$modelTable} ON {$join(Model::class)}
WHERE {$identifier()}
SQL;

$stmt = $connection->prepare($query);
$stmt->execute(['id' => 1]);
$data = $stmt->fetch();

// Use the ESQLMapper to transform this array to objects:
$mapper = new ESQLMapper($autoMapper, $managerRegistry);
dump($mapper->map($stmt->fetch(), Car::class));
```

## API Platform bridge

This package comes with an API Platform bridge that supports filters and pagination. To use our bridge, use the `esql` attribute:


```php
<?php

use ApiPlatform\Core\Annotation\ApiResource;

/**
 * @ApiResource(attributes={"esql"=true})
 */
 class Car {}
```

This will automatically enable the use of:
  - a `CollectionDataProvider` using raw SQL. 
  - an `ItemDataProvider` using raw SQL. 
  - compose-able filters built using [Postgrest](https://postgrest.org/en/v7.0.0/api.html#horizontal-filtering-rows) specification
  - a powerful sort extension also following [Postgrest](https://postgrest.org/en/v7.0.0/api.html#ordering) specification
  - our own `DataPaginator` that you can extend to your will

You can find examples of [Sorting](./tests/Api/SortExtensionTest.php) and [Filtering](./tests/Api/FilterExtensionTest.php).

## FAQ

### Wait did you just re-create DQL?

No. This library offers shortcuts to redundant operations when writing SQL using Doctrine's metadata. We still benefit from Doctrine's metadata and you can still use it to manage your Schema or fixtures. 

### What about Eloquent or another ORM?

It's planned to add support for Eloquent or other ORM systems once the API is stable enough.

### Which Database Management Systems are supported?

With this library you write native SQL. All our helpers will output strings that are useable in the standard SQL specification and therefore should be supported by every relational DBMSusing SQL. The API Platform bridge is tested with SQLite and Postgres. It's only a matter of time to add tests for MariaDB and Mysql.

### Are there any limitations or caveats?

You'll still write SQL so I guess not? The only thing noticeable is that binded parameters will take the name of the fields prefixed by `:`. For example `identifier()` will output `alias.identifier_column = :identifier_fieldname`. Our [`FilterParser`](./src/Filter/FilterParser.php) uses unique parameters names. 

### What is the Mapper all about?

The Mapper maps arrays received via the [PHP Data Objects (PDO) statement](https://www.php.net/manual/en/book.pdo.php) to plain PHP objects also known as Entities. This is why Object Relation Mapping is all about. Internally we're using [JanePHP](https://github.com/janephp/janephp/)'s automapper or Symfony's serializer (TBD). 

### What about writes on the API Platform bridge?

Write support, extended to how Doctrine does is is rather complex especially if you want to support embed writes (write relation at the same time as the main entity). It is possible but there's not much benefits in adding this on our bridge. However you can use some of our helpers to do updates and inserts.

A simple update:

```php
<?php
$car = new Car();
['table' => $table, 'predicates' => $predicates, 'identifier' => $identifier] = $esql($car);
$binding = $this->automapper->map($car, 'array'); // map your object to an array somehow

$query = <<<SQL
UPDATE {$table} SET {$predicates()}
WHERE {$identifier()}
SQL;

$connection->beginTransaction();
$stmt = $connection->prepare($query);
$stmt->execute($binding);
$connection->commit();
```

Same goes for inserting value:

```php
<?php
$car = new Car();
$binding = $this->automapper->map($car, 'array'); // map your object to an array somehow
['table' => $table, 'columns' => $columns, 'parameters' => $parameters] = $esql($car);
$query = <<<SQL
INSERT INTO {$table} ({$columns()}) VALUES ({$parameters($binding)});
SQL;

$connection->beginTransaction();
$stmt = $connection->prepare($query);
$stmt->execute($binding);
$connection->commit();
```

Note that if you used a sequence you'd need to handle that yourself.

## Documentation

### With Doctrine

An ESQL instance offers a few methods to help you write SQL with the help of Doctrine's metadata. To ease there use inside [HEREDOC](https://www.php.net/manual/en/language.types.string.php#language.types.string.syntax.heredoc) calling `__invoke($classOrObject)` on the `ESQL` class will return an array with the following closure:

```php
<?php
use Soyuka\ESQL\Bridge\Doctrine\ESQL;
use App\Entity\Car;
use App\Entity\Model;

// Doctrine's ManagerRegistry
$esql = new ESQL($managerRegistry);

[
  // The only variable here, the Table name
  // outputs "car"
  'table' => $table,
  // Get columns: columns(?array $fields = null, string $glue = ', '): string
  // columns() outputs "car.id, car.name"
  'columns' => $columns,
  // Get a single column: column(string $fieldName): string
  // column('id') outputs "car.id"
  'column' => $column,
  // Get an identifier predicate: identifier(): string
  // identifier() outputs "car.id = :id"
  'identifier' => $identifier,
  // Get a join predicate: join(string $relationClass): string
  // join(Model::class) outputs "car.model_id = model.id"
  'join' => $join,
  // All kinds of predicates: predicates(?array $fields = null, string $glue = ', '): string
  // predicates() outputs "car.id = :id, car.name = :name"
  'predicates' => $predicate,
] = $esql->__invoke(Car::class);
```

More advanced utilities are available as:

```php
<?php
use Soyuka\ESQL\Bridge\Doctrine\ESQL;

// Doctrine's ManagerRegistry
$esql = new ESQL($managerRegistry);

[
  // Relation field name: relationFieldName(string $relationClass): string
  // relationFieldName(Model::class) outputs "model"
  'relationFieldName' => $relationFieldName,
  // Get a normalized value for SQL, sometimes booleans are in fact integer: toSQLValue(string $fieldName, $value)
  // toSQLValue('sold', true) output "1" on sqlite but "true" on postgresql
  'toSQLValue' => $toSQLValue,
  // Given an array of bindings, will output keys prefixed by `:`: parameters(array $bindings): string
  // parameters(['id' => 1, 'color' => 'blue']) will output ":id, :color"
  'parameters' => $parameters,
] = $esql->__invoke(Car::class);
```

This are useful to build filters, write systems or even a custom mapper.

The full interface is available as [ESQLInterface](./src/ESQLInterface.php), shortcuts are defined in [ESQL](./src/ESQL.php).

## To do list
  - ESQLMapper with symfony serializer
