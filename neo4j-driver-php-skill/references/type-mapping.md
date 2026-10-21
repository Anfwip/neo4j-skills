# Cypher-to-PHP Type Mapping

## Full Type Table

| Cypher | PHP |
|---|---|
| `null` | `null` |
| `string` | `string` |
| `integer` | `int` |
| `float` | `float` |
| `boolean` | `bool` |
| `Map` | `\Laudis\Neo4j\Types\CypherMap` |
| `List` | `\Laudis\Neo4j\Types\CypherList` |
| `Point` | `\Laudis\Neo4j\Contracts\PointInterface` |
| `Date` | `\Laudis\Neo4j\Types\Date` |
| `Time` | `\Laudis\Neo4j\Types\Time` |
| `LocalTime` | `\Laudis\Neo4j\Types\LocalTime` |
| `DateTime` | `\Laudis\Neo4j\Types\DateTime` |
| `DateTimeZoneId` | `\Laudis\Neo4j\Types\DateTimeZoneId` |
| `LocalDateTime` | `\Laudis\Neo4j\Types\LocalDateTime` |
| `Duration` | `\Laudis\Neo4j\Types\Duration` |
| `Node` | `\Laudis\Neo4j\Types\Node` |
| `Relationship` | `\Laudis\Neo4j\Types\Relationship` |
| `Path` | `\Laudis\Neo4j\Types\Path` |
| `Vector` | `\Laudis\Neo4j\Types\Vector` |

`Vector` is produced when decoding results from the server only — not supported as a query parameter.

## PHP-to-Cypher Parameter Conversion

| PHP | Cypher |
|---|---|
| `null` | `null` |
| `string` | `string` |
| `int` | `integer` |
| `float` | `float` |
| `bool` | `boolean` |
| Indexed/empty `array` | `List` |
| Associative `array` | `Map` |
| `\DateTimeInterface` | `DateTimeZoneId` |

Empty `[]` is ambiguous — could be list or map. Use `ParameterHelper`:

```php
use Laudis\Neo4j\ParameterHelper;

ParameterHelper::asList([])  // → empty Cypher List
ParameterHelper::asMap([])   // → empty Cypher Map
```

## Point Subtypes

`PointInterface` is implemented by four classes:

| Class | SRID | Use |
|---|---|---|
| `CartesianPoint` | 7203 | 2D Cartesian |
| `Cartesian3DPoint` | 9157 | 3D Cartesian |
| `WGS84Point` | 4326 | 2D geographic (latitude/longitude) |
| `WGS843DPoint` | 4979 | 3D geographic |

```php
$point = $row->get('location');  // PointInterface

if ($point instanceof \Laudis\Neo4j\Types\WGS84Point) {
    echo $point->getX();  // longitude
    echo $point->getY();  // latitude
}
```

## Temporal Types

Temporal types have **no calendar-field getters** (`getYear`, `getMonth`, etc. do not exist). Use `toDateTime()` to convert to `\DateTimeImmutable`, or access raw epoch accessors:

```php
$dt = $row->get('created');     // Laudis\Neo4j\Types\DateTime

// Convert to PHP DateTimeImmutable (recommended)
$php = $dt->toDateTime();       // \DateTimeImmutable

// Raw epoch accessors
$dt->getSeconds();              // epoch seconds
$dt->getNanoseconds();
$dt->getTimeZoneOffsetSeconds();
```

`DateTimeZoneId`, `LocalDateTime`, `Date`, `Time`, `LocalTime` also expose `toDateTime()` or raw accessors — none have year/month/day/hour/minute/second getters.

Pass `\DateTimeInterface` as a parameter — driver maps it to `DateTimeZoneId`:

```php
$client->run(
    'CREATE (e:Event {at: $ts})',
    ['ts' => new \DateTime('2024-06-01 12:00:00')]
);
```

## Node and Relationship Access

```php
// Node
$node      = $row->get('p');                    // Laudis\Neo4j\Types\Node
$intId     = $node->getId();                    // internal integer ID (legacy; always integer)
$elementId = $node->getElementId();             // string element ID (use for Neo4j 5+)
$labels    = $node->getLabels();                // CypherList of strings
$props     = $node->getProperties();            // CypherMap
$name      = $node->getProperty('name');        // single property value
$all       = $node->getProperties()->toArray(); // all as PHP array

// Relationship
$rel        = $row->get('r');                         // Laudis\Neo4j\Types\Relationship
$type       = $rel->getType();                        // e.g. 'KNOWS'
$startIntId = $rel->getStartNodeId();                 // integer (legacy)
$endIntId   = $rel->getEndNodeId();                   // integer (legacy)
$startEId   = $rel->getStartNodeElementId();          // string element ID
$endEId     = $rel->getEndNodeElementId();            // string element ID
$since      = $rel->getProperty('since');

// Path
$path  = $row->get('p');                   // Laudis\Neo4j\Types\Path
$nodes = $path->getNodes();                // CypherList of Node
$rels  = $path->getRelationships();        // CypherList of Relationship
```
