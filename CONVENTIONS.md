# Conventions and gotchas

Patterns and gotchas that hold across every reference-architecture repo in this portfolio. These are the lessons captured from shipping `influxdb3-ref-bess` (the pilot) and `influxdb3-ref-iiot` (the second). Read this before starting a new repo or copy-pasting a pattern from one of the existing ones.

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

### Cron strings are SIX fields

`<sec> <min> <hour> <dom> <mon> <dow>`. Not the five-field Unix format.

| Goal | Correct | Wrong |
|---|---|---|
| Daily at 00:05 UTC | `cron:0 5 0 * * *` | `cron:5 0 * * *` |
| 06:00, 14:00, 22:00 UTC daily | `cron:0 0 6,14,22 * * *` | `cron:0 6,14,22 * * *` |

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

### Don't seed sentinel rows into tables that have an LVC

`init.sh` writes `__init` sentinel rows to make tables exist before the simulator boots. These rows leak into the LVC: `last_cache('table', 'cache_name')` returns the sentinel row alongside the real entities until the simulator overwrites every cache-key combination.

Two options:

1. **Filter `WHERE site <> '__init'`** in every LVC consumer (plugin SQL, UI queries). Cheap, defensive, but the filter has to be repeated everywhere.
2. **Don't seed rows that match the LVC key shape.** Rely on the simulator to write the first row before any consumer reads. Cleaner but requires every UI/plugin to handle the "table empty" case gracefully (which they should anyway).

The reference repos currently use option 1.

### Token bootstrap pattern

A one-shot `token-bootstrap` compose service generates the offline admin token before the main `influxdb3` server starts, writing it to a file in the shared `influxdb-data` named volume. Other services (simulator, UI) mount the volume read-only at `/tokens` and read the JSON file. Lifecycle of the token is decoupled from server lifecycle; restart the server without losing the token.

### `INFLUXDB3_UNSET_VARS=LOG_FILTER`

The `influxdb:3-enterprise` base image's defaults set the deprecated `LOG_FILTER` env var; if it leaks in via the compose env, the engine chokes on its own log output format. Always set `INFLUXDB3_UNSET_VARS: LOG_FILTER` (and a fresh `INFLUXDB3_LOG_FILTER: info`) on every `influxdb3`-image service in compose.

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

## When in doubt, look at iiot

`influxdb3-ref-iiot` is the most recent reference and was the source of most of the gotchas above. When a new repo's design has to make a choice the portfolio spec doesn't cover, default to the iiot pattern unless there's a domain reason not to.
