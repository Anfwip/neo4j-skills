---
name: neo4j-laravel-boost-skill
description: neo4j/laravel-boost — x-ray vision into a Laravel app's wiring: routes, middleware,
  events, jobs, scheduled tasks, policies, notifications, mailables, and container bindings exported
  to a Neo4j knowledge graph. Use to answer architecture questions (what depends on X, what breaks
  if I change Y, which classes build when POST /checkout is hit), find hidden dependencies (facades,
  app()/resolve(), global helpers, new instantiation), contribute missing edges, or install and
  configure the MCP server (driver transport default, STDIO optional, HTTP optional). Covers
  neo4j-boost:setup, Cursor MCP config (.cursor/mcp.json), neo4j-boost:start-neo4j Docker helper,
  neo4j-boost:doctor diagnostics, and all six MCP tools including contribute-graph-knowledge.
  Does NOT handle raw PHP driver code — use neo4j-driver-php-skill.
  Does NOT handle Cypher query authoring — use neo4j-cypher-skill.
  Does NOT handle standalone MCP server setup (without Laravel) — use neo4j-mcp-skill.
version: 1.0.0
allowed-tools: Bash
---

## When to Use
- Answering how a Laravel app is wired: which classes a route, job, event, scheduled task, policy, notification, or mailable builds
- Change impact: what depends on a class or interface (outbound or inbound)
- Finding hidden dependencies: facades, `app()`/`resolve()`, global helpers, `new` instantiation
- Contributing missing wiring edges the graph couldn't detect
- Adding Neo4j MCP tools to a Laravel 12 or 13 application
- Configuring Cursor MCP with `php artisan boost:mcp`
- Choosing between driver (default), STDIO, or HTTP transport
- Diagnosing Neo4j MCP issues with `neo4j-boost:doctor`

## When NOT to Use
- **Raw PHP driver code (ClientBuilder, transactions)** → `neo4j-driver-php-skill`
- **Writing/optimizing Cypher** → `neo4j-cypher-skill`
- **Standalone MCP server (no Laravel)** → `neo4j-mcp-skill`
- **Provisioning Aura instances** → `neo4j-aura-provisioning-skill`

---

## Requirements

| Requirement | Version |
|---|---|
| PHP | ^8.2 |
| Laravel | ^12.0 or ^13.0 |
| Laravel Boost | ^2.0 |
| Laravel MCP | ^0.5.1 (dev dependency) |

`laudis/neo4j-php-client ^3.4` pulled automatically.

---

## Step 1 — Install

```bash
composer require neo4j/laravel-boost
```

`laravel/boost` and `laudis/neo4j-php-client` pulled automatically. Install `laravel/mcp` separately only if not already present:

```bash
composer require laravel/mcp
```

If fails → verify PHP 8.2+ and Laravel 12/13. **Stop and report.**

---

## Step 2 — Run setup

```bash
php artisan neo4j-boost:setup
```

Defaults to **driver transport** (in-process Bolt — no binary needed). Flags:

- `--install-mcp` — download binary for STDIO transport
- `--skip-mcp` — skip binary step
- `--no-cursor-config` — skip `.cursor/mcp.json` generation

Non-interactive:
```bash
php artisan neo4j-boost:setup --no-interaction
```

Auto-run after `composer update` (optional):
```json
{
  "scripts": {
    "post-update-cmd": ["@php artisan neo4j-boost:setup --no-interaction"]
  }
}
```

---

## Step 3 — Configure .env

```env
NEO4J_URI=bolt://localhost:7687
NEO4J_USERNAME=neo4j
NEO4J_PASSWORD=your-password
```

For Aura — driver transport works with `neo4j+s://` directly:
```env
NEO4J_URI=neo4j+s://xxxx.databases.neo4j.io
NEO4J_USERNAME=neo4j
NEO4J_PASSWORD=your-aura-password
```

`.env` must be in `.gitignore`. Never hardcode credentials.

---

## Step 4 — Start local Neo4j

```bash
php artisan neo4j-boost:start-neo4j
```

Starts Docker on `bolt://localhost:7687` and `http://localhost:7474`. APOC enabled — `get-schema` uses `apoc.meta.schema`, falls back to `db.labels`/`db.relationshipTypes`.

```bash
php artisan neo4j-boost:start-neo4j --recreate   # wipe and restart
```

---

## Step 5 — Configure Cursor MCP

```bash
php artisan neo4j-boost:cursor-config
```

Merges into `.cursor/mcp.json`:

```json
{
  "mcpServers": {
    "laravel-boost": {
      "command": "php",
      "args": ["artisan", "boost:mcp"],
      "env": { "APP_ENV": "local" }
    }
  }
}
```

> `APP_ENV=local` required — Boost registers commands only when `APP_ENV=local` or `APP_DEBUG=true`.

If `neo4j-boost:cursor-config` fails → create `.cursor/mcp.json` manually from the template above.

Reload Cursor after any config change.

---

## Step 6 — Verify

```bash
php artisan neo4j-boost:doctor
```

If fails → check transport, re-run setup, verify `.env`, re-run doctor. **Stop and report** if still failing.

End-to-end STDIO test:
```bash
php artisan neo4j-boost:test-stdio --tool=get-schema
```

---

## App Knowledge Graph (core feature)

Graph covers routes, middleware, events, jobs, scheduled tasks, auth guards, providers, password brokers, policies, gate abilities, notifications and channels, mailables and mailers, broadcast channels, container bindings. Includes **declared** deps (constructor injection) and **hidden** deps (facades, `app()`/`resolve()`, helpers like `cache()`, `auth()`, `config()`, and `new ClassName()`).

### 1 — Export

```bash
php artisan container:graph            # export to Neo4j
php artisan container:graph --dry-run  # preview only
php artisan container:graph --print-cypher
```

Set before running to capture hidden deps:
```env
NEO4J_CONTAINER_GRAPH_STATIC_SCAN_PATHS=/abs/path/to/app/Services
NEO4J_CONTAINER_GRAPH_STATIC_SCAN_PROVIDER_PATHS=/abs/path/to/app/Providers
```

Without scan paths → only constructor-injection deps captured. Export is idempotent (`MERGE`-based).

### 2 — Graph model

```
(:Route)-[:HANDLED_BY]->(:Abstract)-[:RESOLVES_TO {lifetime}]->(:Instance)
  -[:DEPENDS_ON {type,source,confidence,...}]->(:Dependency)-[:IDENTIFIED_AS]->(:Abstract)
(:Route)-[:USES_MIDDLEWARE {order,parameters}]->(:Middleware)-[:IDENTIFIED_AS]->(:Abstract)
(:Event|:Job|:ScheduledTask|:Notification|:Mailable|:BroadcastChannel)-[:HANDLED_BY]->(:Abstract)
(:Job|:Mailable)-[:USES_CONNECTION]->(:QueueConnection)
(:AuthGuard)-[:USES_PROVIDER]->(:AuthProvider)-[:USES_MODEL]->(:Abstract)
(:Policy)-[:FOR_MODEL]->(:Abstract)
(:Abstract)-[:BINDS_TO]->(:Abstract)
(:Instance)-[:CONTEXTUAL_BINDS {needs,reason}]->(:Abstract)
```

Edge properties:
- `DEPENDS_ON.type`: `constructor_injection`, `method_injection`, `global_helper`, `instantiation`
- `DEPENDS_ON.source`: `di`, `facade`, `global_helper`, `service_location`
- `RESOLVES_TO.lifetime`: `singleton`, `bind`

### 3 — Query

`get-class-dependency-graph` for one class. `read-cypher` for cross-cutting questions:

```cypher
// Full dep chain from a route
MATCH (r:Route)-[:HANDLED_BY]->(:Abstract)-[:RESOLVES_TO]->(root:Instance)
OPTIONAL MATCH path = (root)-[:DEPENDS_ON|IDENTIFIED_AS|RESOLVES_TO*0..8]->(n)
RETURN r, root, path LIMIT 50;

// What depends on PaymentGateway? (inbound)
MATCH (dep)-[:DEPENDS_ON]->(:Dependency)-[:IDENTIFIED_AS]->(:Abstract {fqcn: 'App\\Services\\PaymentGateway'})
RETURN dep LIMIT 50;
```

Check `graph_completeness` in every `get-class-dependency-graph` response before asserting "nothing depends on X".

### 4 — Contribute missing edges

Call `contribute-graph-knowledge` when the graph lacks wiring you've identified:

- `confidence: medium` or `low` → returns `confirmation_required`, writes nothing. Ask user, retry with `confirmed: true`. Stored with `source: user`.
- `confidence: high` → written immediately as `source: agent`.
- Contributed `DEPENDS_ON` edges default to `type: service_location`.

Per-label query examples and prompts → [`docs/PROMPTS.md`](https://github.com/neo4j-php/neo4j-boost/blob/main/docs/PROMPTS.md), [`docs/tutorials/container-graph.md`](https://github.com/neo4j-php/neo4j-boost/blob/main/docs/tutorials/container-graph.md).

---

## Transport Modes

| Mode | `NEO4J_MCP_TRANSPORT` | How it works | Binary |
|---|---|---|---|
| Driver **(default)** | `driver` | In-process Bolt via `laudis/neo4j-php-client` | No |
| STDIO | `stdio` | `neo4j-mcp` subprocess | Yes — `neo4j-boost:install-mcp` |
| HTTP | `http` | Remote/Docker MCP server | No |

### STDIO

```env
NEO4J_MCP_TRANSPORT=stdio
```

Install binary:
```bash
php artisan neo4j-boost:install-mcp
```

### HTTP

```env
NEO4J_MCP_TRANSPORT=http
NEO4J_MCP_URL=http://localhost:8080/mcp
```

```bash
docker run --rm -p 8080:8080 \
  -e NEO4J_URI=bolt://host.docker.internal:7687 \
  -e NEO4J_USERNAME=neo4j \
  -e NEO4J_PASSWORD=your-password \
  -e NEO4J_TRANSPORT_MODE=http \
  neo4j/mcp:latest
```

---

## Available MCP Tools

`php artisan boost:mcp` exposes Neo4j tools and all Laravel Boost tools (search-docs, browser-logs, database, etc.):

| Tool | Type | Description |
|---|---|---|
| `get-schema` | read | Labels, relationship types, property keys, indexes |
| `read-cypher` | read | Read-only Cypher (MATCH, RETURN, SHOW) |
| `write-cypher` | write | Write Cypher (MERGE, CREATE, SET, DELETE) |
| `list-gds-procedures` | read | GDS procedures (requires GDS plugin) |
| `get-class-dependency-graph` | read | App knowledge graph for one class |
| `contribute-graph-knowledge` | write | Add DEPENDS_ON or BINDS_TO edges the static scan missed |

### get-class-dependency-graph parameters

```json
{
  "class": "App\\Services\\PaymentGateway",
  "depth": 4,
  "direction": "inbound",
  "include_bindings": true,
  "page": 1,
  "per_page": 50
}
```

- `direction`: `outbound`, `inbound`, or `both`
- `depth`: 1–10

Response includes `declared_dependencies`, `hidden_dependencies` (each with `type`, `source`, `confidence`, `visibility`), binding targets, unresolved types, `graph_completeness`. **Always read `graph_completeness` before stating no dependents exist.**

---

## GDS (Optional)

`list-gds-procedures` requires GDS plugin; other tools work without it.

Docker with GDS + APOC:
```yaml
neo4j:
  image: neo4j:5-community
  environment:
    NEO4J_AUTH: neo4j/your-password
    NEO4J_PLUGINS: '["apoc", "graph-data-science"]'
    NEO4J_dbms_security_procedures_unrestricted: 'apoc.*,gds.*'
    NEO4J_dbms_security_procedures_allowlist: 'apoc.*,gds.*'
  ports:
    - "7474:7474"
    - "7687:7687"
```

GDS algorithms → delegate to `neo4j-gds-skill`.

---

## Configuration Reference

```bash
php artisan vendor:publish --tag=neo4j-boost-config
```

Full reference → [references/configuration.md](references/configuration.md)

---

## Artisan Commands

| Command | Description |
|---|---|
| `neo4j-boost:setup` | Interactive setup (driver default; binary optional for STDIO) |
| `neo4j-boost:install-mcp` | Download `neo4j-mcp` binary (STDIO only) |
| `neo4j-boost:start-neo4j` | Start local Neo4j Docker |
| `neo4j-boost:doctor` | Diagnose transport, binary, password, readiness |
| `neo4j-boost:test-stdio` | End-to-end STDIO handshake/tool test |
| `neo4j-boost:cursor-config` | Create/update `.cursor/mcp.json` |
| `container:graph` | Export Laravel app wiring to Neo4j graph |

---

## Troubleshooting

| Symptom | Cause | Fix |
|---|---|---|
| "Could not open input file: artisan" | Cursor not in Laravel app dir | Open Laravel app folder as workspace |
| "no commands in 'boost' namespace" | `APP_ENV` not `local` | Add `"env": {"APP_ENV": "local"}` to `.cursor/mcp.json` |
| "Neo4j password is required for STDIO mode" | `NEO4J_PASSWORD` missing | Set in `.env`; `php artisan config:clear` |
| STDIO test fails with APOC/meta error | APOC not installed | `neo4j-boost:start-neo4j --recreate` |
| HTTP 404: "only handles requests to /mcp" | Cursor sends GET; MCP needs POST | Use `php artisan boost:mcp` (Boost STDIO) |
| `container:graph` hits wrong host in Docker | `NEO4J_URI=localhost` inside container | Use container network host or `NEO4J_DEFAULT_CONNECTION_DSN` |
| GDS errors ("Unknown function 'gds.version'") | GDS not installed | Install GDS; other tools unaffected |
| `get-class-dependency-graph` misses facade/helper deps | Static scan not set | Set `NEO4J_CONTAINER_GRAPH_STATIC_SCAN_PATHS`, re-run `container:graph` |

---

## References

- [references/configuration.md](references/configuration.md) — env vars, transport config, `config/neo4j-boost.php`
- [App graph prompts](https://github.com/neo4j-php/neo4j-boost/blob/main/docs/PROMPTS.md)
- [Container graph tutorial](https://github.com/neo4j-php/neo4j-boost/blob/main/docs/tutorials/container-graph.md)
- [Neo4j Boost GitHub](https://github.com/neo4j-php/neo4j-boost)
- [Official Neo4j MCP Server](https://github.com/neo4j/mcp)
- [Laravel Boost](https://github.com/laravel/boost)

---

## Checklist
- [ ] `composer require neo4j/laravel-boost` installed
- [ ] `php artisan neo4j-boost:setup` completed
- [ ] `.env` has `NEO4J_URI`, `NEO4J_USERNAME`, `NEO4J_PASSWORD`; `.env` in `.gitignore`
- [ ] Transport: `driver` (default), `stdio` (needs `neo4j-boost:install-mcp`), or `http`
- [ ] Neo4j reachable — local via `neo4j-boost:start-neo4j` or remote URI set
- [ ] `.cursor/mcp.json` has `"env": {"APP_ENV": "local"}`
- [ ] Cursor workspace is the Laravel app folder
- [ ] `php artisan neo4j-boost:doctor` passes
- [ ] Smoke test: `read-cypher: RETURN 'connected' AS status` succeeds
- [ ] `container:graph` run before using `get-class-dependency-graph`
- [ ] `NEO4J_CONTAINER_GRAPH_STATIC_SCAN_PATHS` set if hidden deps needed
- [ ] `graph_completeness` checked before asserting no dependents
