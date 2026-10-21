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
- Understanding how a Laravel app is wired: which classes a route, job, event, scheduled task, policy, notification, or mailable ends up building
- Change impact analysis: what depends on a class or interface (outbound or inbound)
- Finding hidden dependencies: facades, `app()`/`resolve()` calls, global helpers, and `new` instantiation
- Contributing missing wiring edges the graph couldn't detect automatically
- Adding Neo4j MCP tools to a Laravel 12 or 13 application
- Configuring Cursor MCP with `php artisan boost:mcp` (single-server pattern)
- Starting local Neo4j via `neo4j-boost:start-neo4j`
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

## Step 1 — Install packages

```bash
composer require neo4j/laravel-boost
```

`laravel/boost` and `laudis/neo4j-php-client` are pulled automatically. `laravel/mcp` is a dev dependency of the package; you only need to install it explicitly if you're not already using it:

```bash
composer require laravel/mcp  # only if not already installed
```

If `composer require` fails with dependency errors → verify PHP 8.2+ and Laravel 12/13. **Stop and report.**

---

## Step 2 — Run interactive setup

```bash
php artisan neo4j-boost:setup
```

Setup defaults to the **driver transport** (in-process Bolt via `laudis/neo4j-php-client`). No `neo4j-mcp` binary is required unless you set `NEO4J_MCP_TRANSPORT=stdio`. Options:

- `--install-mcp` — download binary for STDIO transport
- `--skip-mcp` — skip binary step
- `--no-cursor-config` — skip Cursor `.cursor/mcp.json` generation

Non-interactive (CI / Composer hook):
```bash
php artisan neo4j-boost:setup --no-interaction
```

Optional — auto-run after `composer update`:
```json
{
  "scripts": {
    "post-update-cmd": [
      "@php artisan neo4j-boost:setup --no-interaction"
    ]
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

For Aura, use the `neo4j+s://` console URI — driver transport works with it directly:
```env
NEO4J_URI=neo4j+s://xxxx.databases.neo4j.io
NEO4J_USERNAME=neo4j
NEO4J_PASSWORD=your-aura-password
```

Verify `.env` is in `.gitignore`. Never hardcode credentials.

---

## Step 4 — Start local Neo4j

```bash
php artisan neo4j-boost:start-neo4j
```

Starts Docker container on:
- `bolt://localhost:7687`
- `http://localhost:7474`

APOC enabled by default — `get-schema` uses `apoc.meta.schema` when available, falls back to `db.labels`/`db.relationshipTypes` without it.

Recreate with fresh data:
```bash
php artisan neo4j-boost:start-neo4j --recreate
```

---

## Step 5 — Configure Cursor MCP

Auto-generate `.cursor/mcp.json`:
```bash
php artisan neo4j-boost:cursor-config
```

Merges with existing servers in `.cursor/mcp.json`. Result:

```json
{
  "mcpServers": {
    "laravel-boost": {
      "command": "php",
      "args": ["artisan", "boost:mcp"],
      "env": {
        "APP_ENV": "local"
      }
    }
  }
}
```

> `APP_ENV=local` required — Laravel Boost only registers commands when `APP_ENV=local` or `APP_DEBUG=true`. Without it: `"There are no commands defined in the 'boost' namespace"`.

If `neo4j-boost:cursor-config` fails → manually create `.cursor/mcp.json` using the template above. Ensure the `env` block includes `APP_ENV=local`.

**After config change**: reload Cursor or open MCP settings for pickup.

---

## Step 6 — Verify

```bash
php artisan neo4j-boost:doctor
```

Diagnoses transport, binary, password, and readiness. If doctor passes → MCP ready.

If doctor fails → check reported transport, re-run `php artisan neo4j-boost:setup`, verify `.env` values, then re-run doctor. **Stop and report** if it still fails after setup re-run.

End-to-end STDIO test:
```bash
php artisan neo4j-boost:test-stdio --tool=get-schema
```

---

## App Knowledge Graph (core feature)

Boost exports your Laravel app's complete wiring into Neo4j so the assistant can answer architecture questions from a graph instead of grepping source files.

### What the graph covers

Routes, middleware, events, jobs, scheduled tasks, auth guards, providers, password brokers, policies and gate abilities, notifications and their channels, mailables and mailers, broadcast channels, and all container bindings. Both **declared** (constructor injection) and **hidden** dependencies (facades, `app()`/`resolve()`, global helpers like `cache()`, `auth()`, `config()`, and `new ClassName()`).

### 1 — Export (re-run after code changes or upgrades)

```bash
php artisan container:graph            # export to Neo4j
php artisan container:graph --dry-run  # preview only
php artisan container:graph --print-cypher  # show generated Cypher
```

For hidden dependencies, set the static scan paths **before** running:
```env
NEO4J_CONTAINER_GRAPH_STATIC_SCAN_PATHS=/abs/path/to/app/Services
NEO4J_CONTAINER_GRAPH_STATIC_SCAN_PROVIDER_PATHS=/abs/path/to/app/Providers
```

Without scan paths, the graph only captures constructor-injection dependencies.

Export is idempotent — `MERGE`-based, safe to re-run.

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

Key edge properties:
- `DEPENDS_ON.type`: `constructor_injection`, `method_injection`, `global_helper`, or `instantiation`
- `DEPENDS_ON.source`: `di`, `facade`, `global_helper`, or `service_location`
- `RESOLVES_TO.lifetime`: `singleton` or `bind`

### 3 — Query the graph

Use `get-class-dependency-graph` for a single class. Use `read-cypher` for cross-cutting questions:

```cypher
// What does a route build? Full dependency chain
MATCH (r:Route)-[:HANDLED_BY]->(:Abstract)-[:RESOLVES_TO]->(root:Instance)
OPTIONAL MATCH path = (root)-[:DEPENDS_ON|IDENTIFIED_AS|RESOLVES_TO*0..8]->(n)
RETURN r, root, path LIMIT 50;

// What depends on PaymentGateway? (inbound)
MATCH (dep)-[:DEPENDS_ON]->(:Dependency)-[:IDENTIFIED_AS]->(:Abstract {fqcn: 'App\\Services\\PaymentGateway'})
RETURN dep LIMIT 50;
```

Always check `graph_completeness` in `get-class-dependency-graph` responses before claiming "nothing depends on X".

### 4 — Contribute missing edges

When you find wiring the graph lacks, call `contribute-graph-knowledge`:

- `confidence: medium` or `low` → tool returns `confirmation_required` and writes nothing. Ask the user, then call again with `confirmed: true`. Edge stored with `source: user`.
- `confidence: high` → written immediately with `source: agent`.
- Contributed `DEPENDS_ON` edges default to `type: service_location`.

Example prompts and per-label queries (jobs, events, policies, mailables, etc.) → [`docs/PROMPTS.md`](https://github.com/neo4j-php/neo4j-boost/blob/main/docs/PROMPTS.md) and [`docs/tutorials/container-graph.md`](https://github.com/neo4j-php/neo4j-boost/blob/main/docs/tutorials/container-graph.md).

---

## Transport Modes

| Mode | `NEO4J_MCP_TRANSPORT` | How it works | Binary needed |
|---|---|---|---|
| Driver **(default)** | `driver` | In-process Bolt via `laudis/neo4j-php-client` | No |
| STDIO | `stdio` | Runs `neo4j-mcp` binary as subprocess | Yes (install with `neo4j-boost:install-mcp`) |
| HTTP | `http` | Connects to remote/Docker MCP server | No |

### Driver (default — no binary required)

Default when `NEO4J_MCP_TRANSPORT` is not set. Uses `NEO4J_URI`, `NEO4J_USERNAME`, `NEO4J_PASSWORD`, `NEO4J_DATABASE` directly.

### STDIO (optional)

Binary auto-installed by `neo4j-boost:setup --install-mcp` or `neo4j-boost:install-mcp`. Subprocess receives `NEO4J_URI`, `NEO4J_USERNAME`, `NEO4J_PASSWORD`.

```env
NEO4J_MCP_TRANSPORT=stdio
```

### HTTP

```env
NEO4J_MCP_TRANSPORT=http
NEO4J_MCP_URL=http://localhost:8080/mcp
```

Start MCP server separately:
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

Single `php artisan boost:mcp` exposes **both** Laravel Boost tools and Neo4j tools:

| Tool | Type | Description |
|---|---|---|
| `get-schema` | read | Labels, relationship types, property keys, indexes |
| `read-cypher` | read | Read-only Cypher (MATCH, RETURN, SHOW) |
| `write-cypher` | write | Write Cypher (MERGE, CREATE, SET, DELETE) |
| `list-gds-procedures` | read | GDS procedures (requires GDS plugin) |
| `get-class-dependency-graph` | read | App knowledge graph for one class — see parameters below |
| `contribute-graph-knowledge` | write | Add DEPENDS_ON or BINDS_TO edges the static scan missed |

Boost tools (search-docs, browser-logs, database, etc.) also available from the same server.

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

Response includes `declared_dependencies`, `hidden_dependencies` (each with `type`, `source`, `confidence`, `visibility`), binding targets, unresolved types, and a `graph_completeness` block. **Always read `graph_completeness` before stating a class has no dependents.**

---

## GDS (Optional)

`list-gds-procedures` requires GDS plugin. Without it: that tool errors; others still work.

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

For GDS algorithms → delegate to `neo4j-gds-skill`.

---

## Configuration Reference

Publish config (optional):
```bash
php artisan vendor:publish --tag=neo4j-boost-config
```

Full reference → [references/configuration.md](references/configuration.md)

---

## Artisan Commands

| Command | Description |
|---|---|
| `neo4j-boost:setup` | Interactive setup (driver transport default; binary optional for STDIO) |
| `neo4j-boost:install-mcp` | Download/install the official `neo4j-mcp` binary (STDIO only) |
| `neo4j-boost:start-neo4j` | Start local Neo4j Docker |
| `neo4j-boost:doctor` | Diagnose transport, binary, password, and readiness |
| `neo4j-boost:test-stdio` | Verbose end-to-end STDIO handshake/tool test |
| `neo4j-boost:cursor-config` | Create/update `.cursor/mcp.json` for the MCP server |
| `container:graph` | Export Laravel app wiring to Neo4j knowledge graph |

---

## Troubleshooting

| Symptom | Cause | Fix |
|---|---|---|
| "Could not open input file: artisan" | Cursor not running from Laravel app dir | Open **Laravel app folder** as workspace, not package dir |
| "Unexpected token … is not valid JSON" or "no commands in 'boost' namespace" | `APP_ENV` not `local` | Add `"env": {"APP_ENV": "local"}` to `.cursor/mcp.json` server entry |
| "Neo4j password is required for STDIO mode" | `NEO4J_PASSWORD` not set | Set in `.env`; run `php artisan config:clear` |
| STDIO test fails with APOC/meta error | Neo4j missing APOC plugin | `php artisan neo4j-boost:start-neo4j --recreate` |
| HTTP 404: "only handles requests to /mcp" | Cursor sends GET; MCP expects POST | Use Boost STDIO (`php artisan boost:mcp`); don't point Cursor at MCP HTTP URL directly |
| `container:graph` connects to wrong host in Docker | `NEO4J_URI` points to `localhost` inside container | Set `NEO4J_URI` to container network host, or use `NEO4J_DEFAULT_CONNECTION_DSN` |
| GDS errors ("Unknown function 'gds.version'") | GDS plugin not installed | Install GDS; other tools work without it |
| `get-class-dependency-graph` misses facade/helper deps | Static scan not configured | Set `NEO4J_CONTAINER_GRAPH_STATIC_SCAN_PATHS` and re-run `container:graph` |

---

## References

- [references/configuration.md](references/configuration.md) — full config/neo4j-boost.php reference, env vars, transport details
- [Neo4j Boost GitHub](https://github.com/neo4j-php/neo4j-boost)
- [App graph prompts](https://github.com/neo4j-php/neo4j-boost/blob/main/docs/PROMPTS.md)
- [Container graph tutorial](https://github.com/neo4j-php/neo4j-boost/blob/main/docs/tutorials/container-graph.md)
- [Official Neo4j MCP Server](https://github.com/neo4j/mcp)
- [Laravel Boost](https://github.com/laravel/boost)

---

## Checklist
- [ ] `composer require neo4j/laravel-boost` — package installed
- [ ] `php artisan neo4j-boost:setup` run successfully
- [ ] `.env` has `NEO4J_URI`, `NEO4J_USERNAME`, `NEO4J_PASSWORD`; `.env` in `.gitignore`
- [ ] Transport mode: `driver` (default, no binary), `stdio` (binary via `neo4j-boost:install-mcp`), or `http`
- [ ] Local Neo4j running (`neo4j-boost:start-neo4j`) or remote URI configured
- [ ] `.cursor/mcp.json` has `"env": {"APP_ENV": "local"}` in server entry
- [ ] Cursor workspace is the Laravel app folder (not the package dir)
- [ ] `php artisan neo4j-boost:doctor` passes
- [ ] Smoke test: `read-cypher: RETURN 'connected' AS status` returns result
- [ ] `container:graph` run if using `get-class-dependency-graph` tool
- [ ] `NEO4J_CONTAINER_GRAPH_STATIC_SCAN_PATHS` set to include `app/` if hidden deps needed
- [ ] `graph_completeness` checked in `get-class-dependency-graph` before asserting no dependents
