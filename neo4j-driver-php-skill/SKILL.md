---
name: neo4j-driver-php-skill
description: Neo4j PHP Driver (laudis/neo4j-php-client ^3.x) — Client > Driver > Session >
  Transaction architecture, ClientBuilder, driver lifecycle, session-level executeQuery and
  bookmarks, auto-committed queries, transaction functions (writeTransaction/readTransaction),
  unmanaged transactions (beginTransaction/commit/rollback), result access via CypherList
  and CypherMap, Cypher-to-PHP type mapping, ParameterHelper, URL schemes, SessionConfiguration,
  TransactionConfiguration, Aura and cluster support, causal consistency. Use when writing
  PHP code that connects to Neo4j via Bolt or routed protocol. Package is laudis/neo4j-php-client.
  Does NOT handle Cypher query authoring — use neo4j-cypher-skill.
  Does NOT cover Laravel MCP integration — use neo4j-laravel-boost-skill.
  Does NOT cover driver version migration — use neo4j-migration-skill.
version: 1.0.0
allowed-tools: Bash
---

## When to Use
- Writing PHP code that connects to Neo4j
- Setting up ClientBuilder, sessions, transaction functions, or unmanaged transactions
- Maintaining causal consistency (read-your-own-writes) across multiple queries on a cluster
- Debugging result handling, type mapping, or Aura/cluster connectivity
- Reviewing Neo4j driver usage in PHP code

## When NOT to Use
- **Writing/optimizing Cypher** → `neo4j-cypher-skill`
- **Laravel + MCP integration** → `neo4j-laravel-boost-skill`
- **Driver version upgrades** → `neo4j-migration-skill`

---

## Installation

```bash
composer require laudis/neo4j-php-client
```

If fails → verify PHP ^8.1, `ext-json`, `ext-mbstring`. **Stop and report.**

> HTTP/HTTPS removed in v3.3.0. Use `bolt://`, `bolt+s://`, `bolt+ssc://`, `neo4j://`, `neo4j+s://`, or `neo4j+ssc://` only.

---

## Environment Variables

Never hardcode credentials — load from environment:

```php
$uri      = getenv('NEO4J_URI')      ?: 'bolt://localhost:7687';
$user     = getenv('NEO4J_USERNAME') ?: 'neo4j';
$password = getenv('NEO4J_PASSWORD') ?: '';
$database = getenv('NEO4J_DATABASE') ?: 'neo4j';
```

`.env` file format:
```
NEO4J_URI=neo4j+s://xxxx.databases.neo4j.io
NEO4J_USERNAME=neo4j
NEO4J_PASSWORD=secret
NEO4J_DATABASE=neo4j
```

Add `.env` to `.gitignore`. Use [vlucas/phpdotenv](https://github.com/vlucas/phpdotenv) if no built-in loader (Laravel and Symfony load `.env` automatically):

```bash
composer require vlucas/phpdotenv
```
```php
$dotenv = Dotenv\Dotenv::createImmutable(__DIR__);
$dotenv->load();
```

---

## Architecture: Client > Driver > Session > Transaction

| Layer | Get it with | Lifetime | Role |
|---|---|---|---|
| Client | `ClientBuilder::create()->...->build()` | one per app | Maps aliases to drivers; failover by priority; convenience `run`/`*Transaction` (opens a **new session per call**) |
| Driver | `$client->getDriver('alias')` or `Basic\Driver::create($uri, authenticate: ...)` | one per app | Connection pool, routing table, `verifyConnectivity()`, `getServerInfo()`, `closeConnections()` |
| Session | `$driver->createSession(SessionConfiguration::default()->withDatabase('neo4j'))` | one per unit of work / request | Target database + access mode, chains bookmarks for causal consistency, `executeQuery()`, `close()` |
| Transaction | `$session->writeTransaction(fn)` / `readTransaction(fn)` / `beginTransaction()` | one atomic unit | Runs Cypher; managed (retried) or unmanaged |

**Single Session required when later queries must observe earlier writes (read-your-own-writes on a cluster).** Separate `$client->...` calls open new sessions and do not share bookmarks.

---

## Client Lifecycle

One client per application — create at startup, reuse across requests.

```php
use Laudis\Neo4j\Authentication\Authenticate;
use Laudis\Neo4j\ClientBuilder;

$client = ClientBuilder::create()
    ->withDriver('default', $uri, Authenticate::basic($user, $password))
    ->withDefaultDriver('default')
    ->build();
```

For Aura/cluster with routing — same pattern, `neo4j+s://` URI:

```php
$client = ClientBuilder::create()
    ->withDriver('aura', $uri, Authenticate::basic($user, $password))
    ->withDefaultDriver('aura')
    ->build();
```

Create a Driver directly (no Client alias mapping needed):

```php
use Laudis\Neo4j\Basic\Driver;
use Laudis\Neo4j\Authentication\Authenticate;

$driver = Driver::create($uri, authenticate: Authenticate::basic($user, $password));
$driver->verifyConnectivity();   // throws on failure
$driver->getServerInfo();        // Neo4j version, address
```

URI schemes:

| Driver | Scheme | TLS valid cert | TLS self-signed |
|---|---|---|---|
| neo4j (routing) | `neo4j` | `neo4j+s` | `neo4j+ssc` |
| bolt (single) | `bolt` | `bolt+s` | `bolt+ssc` |

Default if no scheme given: `bolt://localhost:7687?database=neo4j`.

❌ New client per query → `$client = ClientBuilder::...->build()` inside a loop/request handler.
✅ Build once at startup; inject or pass as singleton.

---

## Choosing the Right Approach

| Approach | Method | Auto-retry | Use when |
|---|---|---|---|
| Transaction functions (Client) | `$client->writeTransaction()` / `readTransaction()` | ✅ TransientError, NotALeader, connection errors | **Default** — production, Aura, clusters, HA |
| Session-level query | `$session->executeQuery($cypher, $params)` | ✅ same retry; full result loaded | Causal consistency / bookmarks across calls |
| Auto-committed | `$client->run()` / `runStatement()` | ⚠️ connection/routing errors only (not TransientError) | Simple scripts, non-critical reads |
| Multiple auto-committed | `$client->runStatements([...])` | ⚠️ each is its own auto-commit tx — **not atomic** | Bulk independent statements |
| Unmanaged | `$session->beginTransaction()` | ❌ | Explicit commit/rollback control |

---

## Transaction Functions (Default)

Re-executes callback on TransientError (up to 3 attempts, 100 ms pause), clears routing table on `NotALeader`, retries on connection errors. Routes writes to leader, reads to followers.

```php
use Laudis\Neo4j\Contracts\TransactionInterface;

$result = $client->writeTransaction(static function (TransactionInterface $tsx) {
    return $tsx->run(
        'MERGE (p:Person {email: $email}) RETURN p.name AS name',
        ['email' => 'alice@example.com']
    )->first()->get('name');
});

$names = $client->readTransaction(static function (TransactionInterface $tsx) {
    return $tsx->run('MATCH (p:Person) RETURN p.name AS name')
               ->pluck('name')
               ->toArray();
});
```

> Callback **must be idempotent** — driver re-runs it on retry. Move side effects outside.

```php
// ❌ counter incremented on every retry
$client->writeTransaction(function (TransactionInterface $tsx) use ($counter) {
    $counter->increment();
    $tsx->run('MERGE (x:Node {id: $id})', ['id' => Uuid::v4()]);
});

// ✅ ID generated before callback; counter incremented once after success
$id = Uuid::v4();
$client->writeTransaction(static function (TransactionInterface $tsx) use ($id) {
    $tsx->run('MERGE (x:Node {id: $id})', ['id' => $id]);
});
$counter->increment();
```

---

## Session-level Queries (Causal Consistency)

`executeQuery()` retries on errors and loads the full result. Use one session when queries must see each other's writes.

```php
use Laudis\Neo4j\Databags\SessionConfiguration;

$driver  = $client->getDriver('default');
$session = $driver->createSession(
    SessionConfiguration::default()->withDatabase('neo4j')
);

$session->executeQuery(
    'MERGE (p:Person {email: $email}) RETURN p',
    ['email' => 'alice@example.com']
);

// Same session → bookmarks ensure read-your-own-writes
$people = $session->executeQuery('MATCH (p:Person) RETURN p.name AS name');

// Propagate bookmark across requests
$bookmark = $session->getLastBookmark();
$session->close();

$session2 = $driver->createSession(
    SessionConfiguration::default()
        ->withDatabase('neo4j')
        ->withBookmarks($bookmark)   // experimental in 3.6.x
);
```

---

## Auto-committed Queries

Retries on connection/routing errors only — **not** on `TransientError`.

```php
$client->run(
    'MERGE (user:User {email: $email})',
    ['email' => 'bob@example.com'],
    'default'           // connection alias (optional)
);

use Laudis\Neo4j\Databags\Statement;

$client->runStatement(new Statement(
    'MERGE (user:User {email: $email})',
    ['email' => 'bob@example.com']
));

// NOT atomic — each runs in its own auto-commit transaction
$client->runStatements([
    Statement::create('MERGE (a:A {id: $id})', ['id' => 1]),
    Statement::create('MERGE (b:B {id: $id})', ['id' => 2]),
]);
// ❌ runStatements for atomicity  ✅ Use beginTransaction() instead
```

---

## Unmanaged Transactions

```php
use Laudis\Neo4j\Databags\Statement;

$tsx = $client->beginTransaction(
    [Statement::create('MERGE (x:Person {email: $e})', ['e' => 'a@b.com'])],
    'default'   // connection alias
);
// Results of initial statements discarded

$result  = $tsx->run('MATCH (x) RETURN x LIMIT 100');
$results = $tsx->runStatements([Statement::create('MATCH (x) RETURN x LIMIT 100')]);

$tsx->commit();
// or: $tsx->rollback();
```

❌ Omit `commit()` or `rollback()` → transaction leak.
✅ Always wrap in try/catch; call `rollback()` in catch before re-throwing.

---

## Result Access

```php
$results = $client->run('MATCH (p:Person) RETURN p.name AS name, p.age AS age');

foreach ($results as $row) {
    echo $row->get('name');   // throws if absent
    echo $row->get('age', 0); // with default
}

$first = $results->first();                  // throws on empty result
$names = $results->pluck('name')->toArray();
$count = $results->count();
```

Node and relationship access:

```php
$results = $client->run('MATCH (p:Person)-[r:KNOWS]->(f) RETURN p, r, f');

foreach ($results as $row) {
    $node      = $row->get('p');              // Laudis\Neo4j\Types\Node
    $intId     = $node->getId();              // integer ID (legacy)
    $elementId = $node->getElementId();       // string element ID (Neo4j 5+)

    $rel       = $row->get('r');              // Laudis\Neo4j\Types\Relationship
    $type      = $rel->getType();             // 'KNOWS'
    $startEId  = $rel->getStartNodeElementId();
    $endEId    = $rel->getEndNodeElementId();
}
```

Full type mapping → [references/type-mapping.md](references/type-mapping.md)

---

## ParameterHelper — List vs Map Disambiguation

Empty `[]` is ambiguous — use `ParameterHelper`:

```php
use Laudis\Neo4j\ParameterHelper;

$client->run('MATCH (x) WHERE x.slug IN $slugs RETURN x',
    ['slugs' => ParameterHelper::asList([])]);   // → empty Cypher List

$client->run('CREATE (x:Node $props)',
    ['props' => ParameterHelper::asMap([])]);    // → empty Cypher Map
```

---

## Batch Writes with UNWIND

One transaction for the whole batch — never one tx per item.

```php
$people = [
    ['name' => 'Alice', 'age' => 30],
    ['name' => 'Bob',   'age' => 25],
];

$client->writeTransaction(static function (TransactionInterface $tsx) use ($people) {
    $tsx->run(
        'UNWIND $rows AS row MERGE (p:Person {name: row.name}) SET p.age = row.age',
        ['rows' => $people]
    );
});
```

---

## Common Errors

| Mistake | Fix |
|---|---|
| Multiple clients per script | Build once at startup; share |
| String interpolation in Cypher | Use `$param` placeholders |
| Side effects inside `writeTransaction` callback | Move outside — callback may retry |
| Non-idempotent callback (random ID inside) | Generate ID before callback; pass via `use` |
| Empty array for list parameter | `ParameterHelper::asList([])` |
| Empty array for map parameter | `ParameterHelper::asMap([])` |
| `http://` or `https://` URI | Removed in v3.3.0 — use `bolt://` or `neo4j://` |
| `beginTransaction` without `commit` | Transaction leak — always `commit()` or `rollback()` |
| `first()` on empty result | Throws — check `count()` first |
| Credentials hardcoded | Load from env / `.env` |
| Multiple `$client->run()` expecting causal consistency | Single session + `executeQuery()` |
| `runStatements([...])` expecting atomicity | Not atomic — use `beginTransaction()` |
| `$node->getId()` expecting element ID | `$node->getElementId()` (string, Neo4j 5+) |

---

## References
- [references/type-mapping.md](references/type-mapping.md) — Cypher-to-PHP type table, temporal types, Point subtypes, Vector
- [references/configuration.md](references/configuration.md) — DriverConfiguration, SessionConfiguration, TransactionConfiguration, result summary counters

Docs:
- https://github.com/neo4j-php/neo4j-php-client
- https://packagist.org/packages/laudis/neo4j-php-client

---

## Checklist
- [ ] `laudis/neo4j-php-client` installed via Composer
- [ ] PHP ^8.1, `ext-json`, `ext-mbstring` present; `ext-bcmath` optional
- [ ] One client (or driver) built at startup; shared across requests
- [ ] Credentials from env vars; `.env` in `.gitignore`
- [ ] `writeTransaction` / `readTransaction` for production code
- [ ] Callback idempotent — side effects moved outside
- [ ] `$param` placeholders in all Cypher — no string interpolation
- [ ] `ParameterHelper::asList()` / `asMap()` for empty array params
- [ ] `bolt://` or `neo4j://` URI scheme — no `http://`
- [ ] `beginTransaction()` always paired with `commit()` or `rollback()`
- [ ] Batch writes use UNWIND (not one tx per item)
- [ ] Single session used when causal consistency required
- [ ] `getElementId()` used for Neo4j 5+ element IDs
