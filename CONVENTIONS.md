# Conventions and gotchas

Patterns and gotchas that hold across every reference-architecture repo in this portfolio. These are the lessons captured from shipping `influxdb3-ref-bess` (the pilot), `influxdb3-ref-iiot` (the second), and `influxdb3-ref-network-telemetry` (the multi-node trailblazer). Read this before starting a new repo or copy-pasting a pattern from one of the existing ones.

For the higher-level portfolio design, see [`docs/superpowers/specs/2026-04-23-reference-architectures-portfolio-design.md`](docs/superpowers/specs/2026-04-23-reference-architectures-portfolio-design.md).

## Where things live

- **Per-repo specs:** in the per-repo `docs/superpowers/specs/`. Each vertical's design document ships in that vertical's repo.
- **Implementation plans:** in this meta-repo's `docs/superpowers/plans/`. Plans are the controller's contract for execution; they don't ride along with the user-facing repo.
- **Portfolio-level design:** in this meta-repo's `docs/superpowers/specs/`.

## Processing Engine plugins

### `LineBuilder` is INJECTED, not imported

The Processing Engine sets `LineBuilder` in the plugin module's globals before exec'ing the file. Do **not**:

```python
from influxdb3_local import LineBuilder              # ❌ wrong
try:
    from influxdb3_local import LineBuilder
except ImportError:
    LineBuilder = None                               # ❌ even more wrong
```

The except clause runs after the engine's injection and clobbers the real `LineBuilder` with `None`, breaking the plugin silently. Plugin files should reference `LineBuilder` as a free name and rely on injection.

For unit tests, monkeypatch a fake into the plugin module's namespace:

```python
monkeypatch.setattr(mod, "LineBuilder", FakeLineBuilder, raising=False)
```

### Schedule trigger format: `every:` vs `cron:`

Two formats supported. Pick whichever matches the cadence.

- **`every:<duration>`** — interval string (`every:5s`, `every:1m`, `every:1h`). Use for short, regular cadences (heartbeats, live rollups). No alignment to wall-clock time. Example: `influxdb3-ref-network-telemetry` uses `every:5s` for both schedule plugins.
- **`cron:<sec> <min> <hour> <dom> <mon> <dow>`** — 6-field cron (NOT the 5-field Unix format). Use when the cadence must align to time-of-day (shift boundaries, daily rollups). Example: `influxdb3-ref-iiot` uses `cron:0 0 6,14,22 * * *` for shift summaries; `influxdb3-ref-bess` uses `cron:0 5 0 * * *` for daily SoH.

Cron-format mistakes:

| Goal | Correct | Wrong |
|---|---|---|
| Daily at 00:05 UTC | `cron:0 5 0 * * *` | `cron:5 0 * * *` |
| 06:00, 14:00, 22:00 UTC daily | `cron:0 0 6,14,22 * * *` | `cron:0 6,14,22 * * *` |

### Schedule trigger `--node-spec` (multi-node only)

In multi-node clusters every node validates the catalog at startup. By default a trigger has `node_spec: All`, meaning every node with `--plugin-dir` tries to **execute** it. For schedule plugins this means the plugin fires N times per tick (once per node) — and any node missing a plugin dependency (e.g. `httpx`) crashes on every tick. Pin schedule and request triggers to the node that should run them:

```bash
influxdb3 create trigger fabric_health \
    --database nt \
    --trigger-spec "every:5s" \
    --path "schedule_fabric_health.py" \
    --node-spec "nodes:nt-process" \    # schedule → process node
    fabric_health

influxdb3 create trigger top_talkers \
    --database nt \
    --trigger-spec "request:top_talkers" \
    --path "request_top_talkers.py" \
    --node-spec "nodes:nt-query" \      # request → query node
    top_talkers
```

Single-node repos can omit `--node-spec` (default `all` is fine when there's only one node).

### Plugin filename → trigger type

`init.sh` registers triggers based on the filename prefix:

| Prefix | Trigger type | Trigger name |
|---|---|---|
| `wal_<name>.py` | WAL | `<name>` |
| `schedule_<name>.py` | Schedule | `<name>` |
| `request_<name>.py` | Request | `<name>` (also the URL path: `/api/v3/engine/<name>`) |

Don't include the trigger type in the URL path; the bare `<name>` is the path.

### Request-trigger response shape

Return the body dict directly. The `{"status": ..., "body": ...}` wrapper that some Processing Engine docs show is **not** auto-unwrapped by the current Enterprise version — the wrapper makes it to the HTTP client as-is, and `data.body.lines` instead of `data.lines` will surprise the caller.

```python
def process_request(...):
    return {"lines": [...], "generated_at": "..."}    # ✅ body dict directly

# Not this:
def process_request(...):
    return {"status": 200, "body": {"lines": [...]}}  # ❌ client gets the wrapper
```

If you need a non-200 response code, this is the spot where the convention may need to evolve — flag it in your repo's spec.

### Plugin in-process state survives across batches

The engine keeps the plugin module loaded between WAL batches, so module-level dicts/deques persist and are usable for transition-detect or windowed-derivative WAL plugins. State **does** reset on engine restart — document this caveat in `ARCHITECTURE.md` if your plugin relies on it. Production deployments that need cross-restart continuity should externalize state.

Examples:
- `iiot/plugins/wal_downtime_detector.py` — `_prev_state: dict[str, str]` for transition-detect.
- `iiot/plugins/wal_quality_excursion.py` — `_windows: dict[str, deque]` and `_above: dict[str, bool]` for windowed/derivative + edge-detect.

## SQL / DataFusion

### Read the LVC via `last_cache(table, cache_name)` TVF

The Last Value Cache is read as a **table-valued function**, not by adding WHERE clauses on cache-key columns. A plain `SELECT … FROM <table> WHERE <key columns>` will scan the table.

```sql
-- ✅ Hits the LVC, returns one row per cache-key combination
SELECT machine_id, state
FROM last_cache('machine_state', 'machine_state_last')
WHERE site = 'acme-main'

-- ❌ Full table scan, returns one row per tick
SELECT machine_id, state
FROM machine_state
WHERE site = 'acme-main' AND ...
```

### Multiple unaliased `COUNT(*)` scalar subqueries fail to plan

DataFusion's `scalar_subquery_to_join` optimizer rule fails with "Ambiguous reference to unqualified field 'count(*)'" when a SELECT has more than one unaliased `COUNT(*)` correlated subquery:

```sql
-- ❌ Optimizer error
SELECT
  ms.line_id,
  (SELECT COUNT(*) FROM ... WHERE line_id = ms.line_id) AS running,
  (SELECT COUNT(*) FROM ... WHERE line_id = ms.line_id) AS planned
FROM machine_state ms
GROUP BY ms.line_id
```

Workaround: split into separate `GROUP BY` queries, merge the results in Python (or in the plugin). One query per fact source is also more readable.

### `date_bin()` returns a nanosecond integer string

Bucketed time columns come back over the wire as strings of nanoseconds since epoch (e.g., `"1777217040000000000"`), **not** ISO-8601 datetimes. Browser code that tries `new Date(bucket).getTime()` will get `NaN` and silently render nothing. Parsers must handle:

- Nanosecond integer strings (16+ digits)
- Millisecond integer strings (13 digits)
- Second integer strings (10 digits)
- ISO-8601 datetime strings (in case future engine versions format these on the server)

See `iiot/ui/static/app.js` `bucketToSeconds()` for the reference parser.

### Time-window filters use `INTERVAL`

`WHERE time > now() - INTERVAL '1 hour'` works. `WHERE time > now() - 3600` does not.

### `count(distinct …)` and the Distinct Value Cache

`SELECT COUNT(DISTINCT <col>) FROM <table> WHERE time > <window>` will use the DVC implicitly when the column has a configured cache. No hint required.

## UI

### Browser-facing endpoints need a separate URL

The compose-internal hostname (`http://influxdb3:8181`) is unreachable from the browser. Server-side calls (FastAPI partial routes) use the internal hostname; browser-direct calls (the Processing Engine direct-fetch pattern) need a separate `INFLUX_PUBLIC_URL` env var that defaults to `http://localhost:8181` for local-compose users.

```python
# ui/app.py
INFLUX_URL = os.environ.get("INFLUX_URL", "http://influxdb3:8181")
INFLUX_PUBLIC_URL = os.environ.get("INFLUX_PUBLIC_URL", "http://localhost:8181")
```

The browser-direct URL is then passed to the page as a `data-…` attribute or template variable.

### Direct-fetch + latency badge is the default Processing Engine UI integration

For panels backed by a Request trigger, fetch the endpoint **directly from the browser** with a "served by Processing Engine: N ms" badge measuring the round-trip. This makes the Processing-Engine-as-application-backend pattern visible — the most teachable part of the trigger.

The plugin response should be shaped to drive the entire panel (current state + history if the panel includes a chart) so a single fetch is enough. See `iiot/plugins/request_andon_board.py` and `iiot/ui/static/app.js` for the reference implementation. Avoid mixing direct-fetch with separate FastAPI partial routes for the same panel.

### Starlette `TemplateResponse` new-API signature

Modern FastAPI/Starlette requires the `Request` as the **first positional argument**, not as a `"request"` key in the context dict.

```python
return TEMPLATES.TemplateResponse(request, "name.html", {"foo": ...})    # ✅
return TEMPLATES.TemplateResponse("name.html", {"request": request, …})  # ❌ TypeError: unhashable type: 'dict'
```

The old form raises `TypeError: unhashable type: 'dict'` deep in the Jinja2 cache lookup — the error message doesn't point at the real cause.

## Compose / init.sh

### Create tables explicitly via `/api/v3/configure/table` (preferred)

bess and iiot use the **sentinel-row** pattern: `init.sh` writes one `__init` row at timestamp 1ns to make each table exist before caches and triggers reference it. The cost is that those rows leak into LVC reads, so consumers must filter `WHERE site <> '__init'` everywhere.

`influxdb3-ref-network-telemetry` introduced the **explicit-create** pattern via the configure API (or the equivalent `influxdb3 create table` CLI). Tables get their full schema (and per-table retention) declared up front. No sentinel rows; no `__init` filter needed downstream.

```bash
influxdb3 create table fabric_health \
    --database nt \
    --tags site,layer \
    --fields "status:utf8,spines_up:int64,bgp_up:int64,ingress_bps:float64,ecn_pct:float64" \
    --retention-period 24h
```

CLI field-type names are `int64`, `uint64`, `float64`, `utf8`, `bool` — **not** the line-protocol shorthand `i64`/`string`. Use the explicit-create pattern for all new repos. bess and iiot can be migrated as a follow-up.

### Token bootstrap pattern

A one-shot `token-bootstrap` compose service generates the offline admin token before the main `influxdb3` server starts, writing it to a file in the shared `influxdb-data` named volume. Other services (simulator, UI) mount the volume read-only at `/tokens` and read the JSON file. Lifecycle of the token is decoupled from server lifecycle; restart the server without losing the token.

### `INFLUXDB3_UNSET_VARS=LOG_FILTER`

The `influxdb:3-enterprise` base image's defaults set the deprecated `LOG_FILTER` env var; if it leaks in via the compose env, the engine chokes on its own log output format. Always set `INFLUXDB3_UNSET_VARS: LOG_FILTER` (and a fresh `INFLUXDB3_LOG_FILTER: info`) on every `influxdb3`-image service in compose.

### Healthchecks must include the admin token

In recent Enterprise versions `/health` requires authentication and there's no longer a public unauthenticated probe. The bess/iiot pattern of `curl -s -o /dev/null http://.../health` (relying on "any HTTP roundtrip including 401 = server is up") doesn't reliably mark the container healthy any more.

`token-bootstrap` writes a plain-text token alongside the JSON admin token file so healthchecks can `cat` it directly. Every InfluxDB node's healthcheck then passes it as a Bearer token and uses `curl --fail` (so non-2xx responses fail the check):

```yaml
healthcheck:
  test: ["CMD-SHELL", "test -s /var/lib/influxdb3/.<db>-token-plain && curl -s --max-time 2 -o /dev/null --fail -H \"Authorization: Bearer $$(cat /var/lib/influxdb3/.<db>-token-plain)\" http://127.0.0.1:8181/health"]
  interval: 5s
  timeout: 3s
  retries: 120
  start_period: 5s
```

Note `$$(cat …)` — compose interprets a single `$` as variable substitution; the doubled `$$` becomes `$` at runtime so the shell inside the container evaluates `$(cat …)`.

The bess and iiot single-node compose files are also worth back-porting this fix to.

## Multi-node compose pattern (clustered repos)

Reference: `influxdb3-ref-network-telemetry` is the multi-node trailblazer. Subsequent clustered repos (Fleet, Data Center, etc.) should follow this shape unless their domain genuinely requires a different topology.

### Shared `influxdb-data` named volume across every InfluxDB node

The volume holds the object store, catalog, and admin token. Every node mounts it at `/var/lib/influxdb3` and uses `--object-store file --data-dir /var/lib/influxdb3`. **Sharing the disk is what makes the cluster a cluster** — every node sees the same database list, table schemas, caches, and triggers, and writes from any ingest node are immediately readable from query and process nodes. No coordination protocol needed.

### Mode flags

- `--mode ingest` — accepts writes
- `--mode query` — serves reads + hosts request plugins
- `--mode compact` — compaction-only; no HTTP traffic
- `--mode process,query` — process node combo: query engine available locally for plugins to use `influxdb3_local.query()`, plus plugin runtime
- Setting `--plugin-dir` automatically adds `process` mode

### All InfluxDB nodes must mount the plugin directory

Even nodes that don't execute plugins (ingest, compact) **read the catalog at startup and validate every registered trigger's plugin path**. If a trigger references `/plugins/foo.py` and the node doesn't have `/plugins` mounted, it panics with `Plugin not found` exit 101 on second-boot. Mount `./plugins:/plugins:ro` on **every** InfluxDB node.

### Per-node `--virtual-env-location`

The plugin runtime initializes a Python venv. The default location is `/plugins/.venv` which is read-only. Each InfluxDB node needs its own venv path inside the data volume so they don't fight:

```
--plugin-dir /plugins
--virtual-env-location /var/lib/influxdb3/plugin-venv-<role>
```

Where `<role>` is `query`, `process`, `ingest-1`, `ingest-2`, `compact`, etc.

### Plugin packages installed via `influxdb3 install package`

The plugin venvs ship empty. Schedule plugins that depend on `httpx` (for cross-node write-back) need it installed on each node that runs them:

```bash
influxdb3 install package httpx --host http://nt-process:8181 --token "$TOKEN"
```

Run from `init.sh` after the cluster is up. With `--node-spec` pinning triggers to specific nodes, you only need the package installed on those targeted nodes.

### Schedule plugin write-back via httpx (cross-node)

A schedule plugin running on a process-only (or `process,query`) node has no obvious local ingest target — `LineBuilder` + `influxdb3_local.write()` aren't the right tool when the local node doesn't accept writes. Instead, schedule plugins import from a shared `plugins/_writeback.py` module that:

1. Reads the admin token from the shared volume at module-import time.
2. POSTs line protocol to a list of ingest URLs (compose-DNS names like `http://nt-ingest-1:8181`), round-robining across them.
3. Falls back to the next URL on connection error.

The required env vars on the process node:

- `<DB>_INGEST_URLS` (comma-separated)
- `<DB>_DB`
- `<DB>_TOKEN_FILE`

(replace `<DB>` with the per-repo prefix, e.g. `NT_INGEST_URLS`)

`LineBuilder` is **not used** by schedule plugins on a process-only node.

### Bring up the query node first to seed the license

Bringing up all InfluxDB nodes simultaneously fires N parallel license-validation requests at the InfluxData license server, which intermittently 500s. Add `depends_on: influxdb3-query: service_healthy` to the other InfluxDB nodes so query validates the trial license alone first, writes it to the shared volume, and the others read it from cache:

```yaml
influxdb3-ingest-1:
  depends_on:
    token-bootstrap:
      condition: service_completed_successfully
    influxdb3-query:
      condition: service_healthy
```

Process is implicitly ordered after query through its existing `ingest-1` dep. This adds maybe ~10 s to total bring-up time but eliminates flaky boots.

### Demo wait-for-healthy uses `docker inspect`, not `docker compose ps`

`docker compose ps SERVICE` filters by **service** names (`influxdb3-ingest-1`); the rest of the demo script uses **container** names (`nt-ingest-1`). Use `docker inspect --format '{{.State.Health.Status}}' <container>` for healthcheck polling — it accepts container names directly:

```bash
spin_until "${name} healthy" \
    "docker inspect --format '{{.State.Health.Status}}' ${name} 2>/dev/null | grep -q '^healthy$'" 180
```

### Trigger registration on second-boot

`init.sh`'s idempotent `create trigger` calls treat "already exists" as success — but if you change a trigger's flags (e.g. add `--node-spec`), the existing trigger **doesn't** get updated. Delete-and-recreate at the top of `ensure_triggers()`:

```bash
for t in fabric_health anomaly_detector top_talkers src_ip_detail; do
    cli delete trigger "${t}" --database "${INFLUX_DB}" --force 2>/dev/null || true
done
# then create with current flags …
```

A no-op on first boot; corrects stale catalog entries on subsequent boots.

## Testing

### Three tiers, one structure

Every repo ships:

- **Tier 1 — Plugin unit tests** (`tests/test_plugins/`). Pure-function tests with a recording fake `influxdb3_local`. No Docker. Fast.
- **Tier 2 — Scenario tests** (`tests/test_scenarios/`). `testcontainers-python` boots the real image; tests write synthetic data and assert on outcomes. Skip if Docker unavailable.
- **Tier 3 — Smoke test** (`tests/test_smoke.py`). Runs the actual `make up` flow against a license-validated `influxdb-data` volume artifact (see each repo's `FOR_MAINTAINERS.md`).

### Substring-match test fakes need disambiguation when SQL queries overlap

A naïve `if key in sql: return rows` fake breaks down once a plugin issues two queries against the same table with different aggregation windows. Use either:

1. **Longest-key-wins**: pick the longest matching key. Works when the more-specific query has a strict-superset substring (e.g., `"FROM machine_state WHERE state"` beats `"FROM machine_state"`).
2. **Tuple-AND keys**: each route's key is a tuple of substrings that must all be present; total matched-character count is the score. Works when no single substring uniquely identifies a query.

The iiot test fakes use the tuple-AND pattern. See `iiot/tests/test_plugins/test_request_andon_board.py`.

## Repo conventions

- **Spec lives in the per-repo `docs/superpowers/specs/`.** Plans live in the meta-repo. Both are committed.
- **Single GitHub repo per vertical** under the `influxdata` org, public, Apache 2.0.
- **README structure:** quickstart, "what's in this repo" table, headline features, scaling-to-production pointer. The architecture diagram (`diagrams/architecture.png`) is the first thing the README shows.
- **Demo script (`scripts/demo.sh`) is narrative.** Banner, "what this demo shows", actor key (`[script]`/`[ui]`/`[sim]`/`[db]`), numbered "what's about to happen". Step through prereqs → bring-up → license validation → scenario → query results → request endpoint → LVC demo → summary. The bess and iiot scripts are mutually-templatable; bess is the seed.
- **`Makefile` surface is identical** across repos: `up`, `down`, `clean`, `demo`, `demo-fresh`, `cli`, `query`, `cli-example`, `scenario`, `scenario-list`, `test`, `test-unit`, `test-scenarios`, `test-smoke`, `lint`, `format`.

## When in doubt, look at the most recent reference

- **Single-node:** look at `influxdb3-ref-iiot`. It's the canonical single-node template — Python signals, two WAL plugin patterns, schedule + request triggers, the andon-board direct-fetch UI pattern.
- **Multi-node:** look at `influxdb3-ref-network-telemetry`. It's the canonical multi-node template — 5-node compose, shared volume, plugin write-back via httpx, three UI patterns side-by-side, per-table retention, `every:` schedule format.

When a new repo's design has to make a choice the portfolio spec doesn't cover, default to whichever of those matches your topology, unless there's a domain reason not to.
