# Network Telemetry Implementation Plan — `influxdb3-ref-network-telemetry`

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** Ship a complete, runnable `influxdb3-ref-network-telemetry` reference architecture: a 5-node InfluxDB 3 Enterprise cluster (2 ingest + 1 query + 1 compact + 1 process,query) monitoring a synthetic 8 spines × 16 leaves Clos fabric (~10k pts/sec), with two schedule plugins on the process node, two request plugins on the query node, a typeahead-driven dashboard, and a CONVENTIONS amendment for the multi-node patterns it introduces.

**Architecture:** Mirrors the bess/iiot single-node template wherever possible (Python-first, FastAPI/HTMX/uPlot UI, three-tier testing, four GitHub Actions workflows, narrative `make demo`). Adds: 5-node docker-compose with shared `influxdb-data` volume providing catalog consistency; explicit table creation via the `/api/v3/configure/*` API (no sentinel rows); per-table 24h retention on `fabric_health`; schedule plugins use httpx to write back through ingest nodes (NOT `LineBuilder`); UI demonstrates three teaching patterns side-by-side (SQL via FastAPI, SQL from browser via DVC, request plugin from browser).

**Tech Stack:** Python 3.12+, `influxdb3-python`, FastAPI, HTMX, Jinja2, uPlot (vendored), httpx, Docker / docker-compose, `testcontainers-python`, pytest, ruff, GitHub Actions.

**Spec:** [`influxdb3-ref-network-telemetry/docs/superpowers/specs/2026-04-26-network-telemetry-design.md`](https://github.com/influxdata/influxdb3-ref-network-telemetry/blob/main/docs/superpowers/specs/2026-04-26-network-telemetry-design.md) — read this first; this plan implements that design exactly.

**Reference repos** (for copying patterns + assets — local paths):
- `/Users/pauldix/codez/reference_architectures/influxdb3-ref-iiot/` — most recent reference; copy patterns liberally.
- `/Users/pauldix/codez/reference_architectures/influxdb3-ref-bess/` — earlier reference; signals_base.py and vendored JS originally come from here.
- `/Users/pauldix/codez/reference_architectures/influxdb3-reference-architectures/CONVENTIONS.md` — read before starting.

## End-to-end integration testing — run for real

This plan **executes the cluster end-to-end as part of verification**, not just at the user's review time. The manual checkpoints (Phase 4, Phase 7) and the final ship phase (Phase 11) actually `make up` the 5-node cluster, run both scenarios, hit the live UI, and confirm the three teaching patterns work. Code that compiles isn't enough — the repo must be **handed back fully verified**.

**License:** use `INFLUXDB3_ENTERPRISE_EMAIL=paul+refiiot@influxdata.com`. This address has an existing validated Enterprise trial license — the cluster auto-picks it up from a fresh volume without needing a manual email click. Set this in `.env` before any `make up` step in the plan; it's the implementer's hard-coded email for all integration runs.

**Sandbox note for the implementer:** the iiot CLAUDE.md guidance (referenced in `/Users/pauldix/codez/CLAUDE.md` § "Load Testing") applies here too — `make up` backgrounds docker-compose children, which the default Claude Code sandbox kills. Run docker-compose-touching commands with `dangerouslyDisableSandbox: true` (Bash tool flag). Also kill any stale `nt-*` containers from prior runs before each fresh `make up`:

```bash
docker ps -a --format '{{.Names}}' | grep '^nt-' | xargs -r docker rm -f 2>/dev/null
```

---

## Phase 0 — API surface verification (READ AND VERIFY BEFORE STARTING)

This is a **multi-node trailblazer**. Several things the iiot plan took for granted need explicit verification because:
- Multi-node `mode` flag spelling is documented but not exercised in any prior portfolio repo.
- The configure API endpoint shapes (table create, retention) are referenced by the spec but the docs file you have local does not include their JSON request bodies.
- The `every:` schedule trigger format is used here for the first time in the portfolio (vs `cron:` in bess and iiot).
- Plugin write-back via httpx (rather than `influxdb3_local.write()`) is a new convention.

Run Phase 0 against a single-node `influxdb:3-enterprise` container before committing to the multi-node compose shape. Document what works and what doesn't; amend this plan and the spec inline if reality differs.

### Task 0.1: Verify `/api/v3/configure/*` endpoints

Boot a single-node container, then probe the configure API to confirm endpoint paths and JSON request shapes:

- [ ] **Step 1: Boot a probe container**

```bash
docker run --rm -d \
  --name nt-probe \
  -p 18181:8181 \
  -e INFLUXDB3_ENTERPRISE_LICENSE_EMAIL=probe@example.com \
  -e INFLUXDB3_ENTERPRISE_LICENSE_TYPE=trial \
  -e INFLUXDB3_UNSET_VARS=LOG_FILTER \
  -e INFLUXDB3_LOG_FILTER=info \
  influxdb:3-enterprise \
  serve --node-id probe-0 --cluster-id probe \
        --mode all --object-store memory --without-auth
```

Wait for `/health` to respond (validation may not be needed with `--without-auth`; if it is, you'll need a validated license — re-run with `INFLUXDB3_LICENSE_CHECK=false` or use a maintainer email).

- [ ] **Step 2: Discover endpoint shapes**

Try each of these against `http://localhost:18181`. For each, record (a) HTTP status, (b) accepted JSON keys, (c) response body. Update Phase 3 task code with the actual shapes.

```bash
# Database create
curl -i -X POST http://localhost:18181/api/v3/configure/database \
  -H "Content-Type: application/json" \
  -d '{"db": "nt"}'

# Or possibly:
curl -i -X POST "http://localhost:18181/api/v3/configure/database?db=nt"

# Table create with explicit schema
curl -i -X POST http://localhost:18181/api/v3/configure/table \
  -H "Content-Type: application/json" \
  -d '{
    "db": "nt",
    "table": "interface_counters",
    "tags": ["site", "switch", "interface"],
    "fields": [
      {"name": "in_bytes", "type": "i64"},
      {"name": "out_bytes", "type": "i64"}
    ]
  }'

# Retention
curl -i -X POST http://localhost:18181/api/v3/configure/table \
  -H "Content-Type: application/json" \
  -d '{
    "db": "nt",
    "table": "fabric_health",
    "tags": ["site", "layer"],
    "fields": [{"name": "spines_up", "type": "i64"}],
    "retention_period": "24h"
  }'

# CLI fallback (in case the HTTP API doesn't exist or the JSON shape differs)
docker exec nt-probe influxdb3 create database nt
docker exec nt-probe influxdb3 create table fabric_health --database nt \
  --tags site,layer --fields spines_up:i64 --retention-period 24h
```

Either the HTTP API or the CLI will work. **Phase 3's `init.sh` uses whichever responds correctly.**

- [ ] **Step 3: Verify `every:5s` schedule trigger spec**

```bash
# Drop a tiny noop plugin in /var/lib/influxdb3/plugins/sched_test.py
docker exec nt-probe sh -c 'mkdir -p /var/lib/influxdb3/plugins && cat > /var/lib/influxdb3/plugins/sched_test.py <<EOF
def process_scheduled_call(influxdb3_local, call_time, args=None):
    influxdb3_local.info(f"sched_test fired at {call_time}")
EOF'
docker exec nt-probe influxdb3 create trigger sched_test \
    --database nt \
    --trigger-spec "every:5s" \
    --path "sched_test.py"
docker exec nt-probe influxdb3 enable trigger sched_test --database nt
sleep 12
docker logs nt-probe 2>&1 | grep "sched_test fired" | wc -l
# Expected: ≥ 2 (fires every 5s, so ~2-3 runs in 12s)
```

If `every:5s` is rejected, fall back to `cron:*/5 * * * * *` and amend the spec.

- [ ] **Step 4: Verify multi-node `--mode` flag values**

This is hardest to fully test from a single container, but at least confirm each mode string is accepted by the `serve` command (server starts, doesn't reject the flag):

```bash
for m in ingest query compact "process,query"; do
  docker run --rm \
    -e INFLUXDB3_ENTERPRISE_LICENSE_EMAIL=probe@example.com \
    -e INFLUXDB3_ENTERPRISE_LICENSE_TYPE=trial \
    -e INFLUXDB3_UNSET_VARS=LOG_FILTER \
    influxdb:3-enterprise \
    serve --node-id probe --cluster-id probe \
          --mode "$m" --object-store memory --without-auth \
          --plugin-dir /var/lib/influxdb3/plugins 2>&1 | head -5
  echo "---mode=$m---"
done
```

Look for "starting" or healthcheck-listen lines vs "unrecognized mode" / "invalid argument" errors. If any mode is wrong, amend §4.1 of the spec.

Note: per the docs you read locally, **`--plugin-dir` automatically adds `process` mode**. So the process node may need only `--mode query` plus `--plugin-dir`, and the spec's `process,query` mode-combo wording becomes implementation detail. Verify and amend.

- [ ] **Step 5: Tear down the probe**

```bash
docker rm -f nt-probe
```

- [ ] **Step 6: Record findings**

Write `notes/phase0-findings.md` (NOT committed — these are scratch notes for the implementer). Capture:
1. Working configure API shapes (or "use CLI instead")
2. `every:5s` works / falls back to cron
3. Mode flag spellings that work
4. Any other surprises

Use the findings to amend the relevant Phase 3 / Phase 6 task code blocks in this plan **before executing them**. Commit any plan amendments as `plan: phase 0 amendments`.

### 0.2 Hard rules to internalize before starting

These are stable across the portfolio and gotcha you'd otherwise debug for hours. Read [`CONVENTIONS.md`](https://github.com/influxdata/influxdb3-reference-architectures/blob/main/CONVENTIONS.md) in full, then confirm:

- **`LineBuilder` is INJECTED into plugin globals.** Do NOT import it in any file under `plugins/`. Schedule plugins in this repo don't use `LineBuilder` at all (they use httpx for write-back), but the rule still applies — no `from influxdb3_local import LineBuilder` anywhere.
- **Cron strings are 6-field**, but this repo prefers `every:5s` interval format. Use `every:` for short intervals; `cron:` only when you need time-of-day alignment (this repo has none).
- **Plugin response shape:** request plugins return the body dict directly, NOT `{"status": ..., "body": ...}`. The wrapper is not auto-unwrapped.
- **LVC reads use `last_cache(table, cache_name)` TVF**, not WHERE clauses on key columns.
- **DVC reads use `distinct_cache(table, cache_name)` TVF** for prefix search and similar — same TVF pattern.
- **`COUNT(*)` scalar subqueries don't compose** under DataFusion. Split into separate GROUP BY queries; merge in Python.
- **`date_bin()` returns nanosecond integer strings** on the wire. Browser JS must parse ns/ms/s integer strings AND ISO datetimes.
- **Browser-facing endpoints need `INFLUX_PUBLIC_URL`** (defaults `http://localhost:8181`) separate from the in-compose `INFLUX_URL` (`http://nt-query:8181`).
- **Starlette `TemplateResponse(request, name, ctx)`** — request as first positional arg.
- **Tables created via `/api/v3/configure/table` (or CLI equivalent)** at init time, NOT via sentinel-row writes. This is THIS repo's new convention; no `__init` filtering needed.

### 0.3 Project-specific conventions THIS repo introduces

Codify these in CONVENTIONS amendments at the very end (Phase 11):

- **`every:` schedule format** — preferred over `cron:` for short, regular intervals (heartbeats, live rollups). Use `cron:` only when time-of-day alignment is required.
- **Schedule plugin cross-node write-back** — plugins on a process node write back via httpx to an ingest node's `/api/v3/write_lp`, NOT via `influxdb3_local.write()`. The plugin loads the admin token from `/var/lib/influxdb3/.<db>-operator-token` (mounted from the shared volume) at module-init time. Round-robin between ingest nodes with one fallback hop on connection error.
- **Explicit table creation** — use `/api/v3/configure/table` (or CLI) at init time with the full tag/field schema and any retention. Do NOT seed sentinel rows.
- **Multi-node compose pattern** — shared `influxdb-data` named volume mounted at `/var/lib/influxdb3` on every InfluxDB node provides object-store + catalog consistency. Each node has a single mode (or `process,query` combo for the process node).
- **Per-table retention** — set at table-create time via the configure API's `retention_period` field. Demonstrated on `fabric_health` (24h).

---

## File structure (what this plan produces)

```
influxdb3-ref-network-telemetry/
├── README.md
├── ARCHITECTURE.md
├── SCENARIOS.md
├── CLI_EXAMPLES.md
├── FOR_MAINTAINERS.md
├── LICENSE
├── .env.example
├── .gitignore
├── docker-compose.yml             (10 services — 5 InfluxDB nodes + 5 support)
├── Makefile
├── pyproject.toml
├── diagrams/
│   ├── architecture.mmd            (5-node topology)
│   └── architecture.png
├── influxdb/
│   ├── init.sh
│   └── schema.md
├── plugins/
│   ├── _writeback.py               (NEW — httpx round-robin client for plugin → ingest writes)
│   ├── schedule_fabric_health.py
│   ├── schedule_anomaly_detector.py
│   ├── request_top_talkers.py
│   └── request_src_ip_detail.py
├── simulator/
│   ├── Dockerfile
│   ├── __init__.py
│   ├── main.py
│   ├── config.py
│   ├── writer.py                   (round-robins across two ingest URLs)
│   ├── signals_base.py             (copied from bess)
│   ├── signals.py                  (network domain — see §2)
│   └── scenarios/
│       ├── __init__.py
│       ├── _base.py
│       ├── congestion_hotspot.py
│       └── east_west_burst.py
├── ui/
│   ├── Dockerfile
│   ├── __init__.py
│   ├── app.py
│   ├── queries.py
│   ├── templates/
│   │   ├── base.html
│   │   ├── overview.html
│   │   └── partials/
│   │       ├── _fabric_state.html
│   │       ├── _kpi_row.html
│   │       └── _anomalies.html
│   └── static/
│       ├── app.css
│       ├── app.js
│       ├── htmx.min.js             (copied from bess)
│       ├── uplot.min.js            (copied from bess)
│       └── uplot.min.css           (copied from bess)
├── scripts/
│   ├── setup.sh
│   └── demo.sh
├── tests/
│   ├── __init__.py
│   ├── conftest.py
│   ├── test_signals_base.py
│   ├── test_signals.py
│   ├── test_writer.py
│   ├── test_queries.py
│   ├── test_smoke.py
│   ├── test_plugins/
│   │   ├── __init__.py
│   │   ├── test_writeback.py
│   │   ├── test_schedule_fabric_health.py
│   │   ├── test_schedule_anomaly_detector.py
│   │   ├── test_request_top_talkers.py
│   │   └── test_request_src_ip_detail.py
│   └── test_scenarios/
│       ├── __init__.py
│       ├── test_congestion_hotspot.py
│       └── test_east_west_burst.py
├── .github/workflows/
│   ├── unit.yml
│   ├── scenarios.yml
│   ├── smoke.yml
│   └── lint.yml
└── docs/superpowers/specs/
    └── 2026-04-26-network-telemetry-design.md   (already committed)
```

Working directory throughout: `/Users/pauldix/codez/reference_architectures/influxdb3-ref-network-telemetry/` unless noted. Reference repos at `/Users/pauldix/codez/reference_architectures/influxdb3-ref-{bess,iiot}/`.

---

## Phase 1 — Repo scaffolding

### Task 1: License, `.gitignore`, `.env.example`, `pyproject.toml`, README stub

**Files:**
- Create: `LICENSE` (copied from iiot)
- Modify: `.gitignore` (already exists from spec commit; extend it)
- Create: `.env.example`
- Create: `pyproject.toml`
- Create: `README.md` (stub)

- [ ] **Step 1: Copy LICENSE from iiot**

```bash
cp /Users/pauldix/codez/reference_architectures/influxdb3-ref-iiot/LICENSE \
   /Users/pauldix/codez/reference_architectures/influxdb3-ref-network-telemetry/LICENSE
```

- [ ] **Step 2: Extend `.gitignore`** (existing file has the basics; ensure these are present)

```
.env
.venv/
__pycache__/
*.pyc
.pytest_cache/
.ruff_cache/
.DS_Store
node_modules/
*.egg-info/
dist/
build/
.coverage
htmlcov/
notes/
```

(`notes/` is excluded so Phase 0 scratch findings don't get committed.)

- [ ] **Step 3: Create `.env.example`**

```
# Required for InfluxDB 3 Enterprise license validation.
# On first `make up`, a validation email is sent to this address; click the
# link in the email to activate your trial. The container will remain in
# "waiting for validation" state until you do. Validation happens once for
# the cluster — all five InfluxDB nodes share the same license via the
# influxdb-data named volume.
INFLUXDB3_ENTERPRISE_EMAIL=

# Optional: license type. One of: home, trial, commercial. Default: trial.
# INFLUXDB3_ENTERPRISE_LICENSE_TYPE=trial

# Simulator
SIM_RATE_HZ=1.0           # base tick rate; counters/BGP at this rate, flows are bursty
SIM_SEED=42                # reproducibility for tests
SIM_SPINES=8
SIM_LEAVES=16
SIM_SERVERS_PER_LEAF=48
SIM_FLOW_RECORDS_PER_SEC=5000

# UI
UI_PORT=8080
UI_KPI_POLL_MS=5000
UI_THROUGHPUT_POLL_MS=10000
UI_ANOMALIES_POLL_MS=5000
UI_TOP_TALKERS_POLL_MS=5000

# Browser-facing URL for direct fetches (typeahead SQL + request plugins).
# Defaults to localhost so the bundled docker-compose port mapping (8181 on
# the query node) works for local users out of the box.
INFLUX_PUBLIC_URL=http://localhost:8181
```

- [ ] **Step 4: Create `pyproject.toml`**

```toml
[build-system]
requires = ["setuptools>=68", "wheel"]
build-backend = "setuptools.build_meta"

[project]
name = "influxdb3-ref-network-telemetry"
version = "0.1.0"
description = "Reference architecture: InfluxDB 3 Enterprise multi-node cluster for data-center fabric telemetry"
readme = "README.md"
license = { text = "Apache-2.0" }
requires-python = ">=3.12"
dependencies = [
  "influxdb3-python>=0.7",
  "fastapi>=0.110",
  "uvicorn[standard]>=0.27",
  "jinja2>=3.1",
  "httpx>=0.27",
  "python-dotenv>=1.0",
]

[project.optional-dependencies]
dev = [
  "pytest>=8.0",
  "pytest-asyncio>=0.23",
  "testcontainers>=4.0",
  "ruff>=0.4",
  "freezegun>=1.4",
]

[tool.setuptools.packages.find]
include = ["simulator*", "ui*"]

[tool.pytest.ini_options]
testpaths = ["tests"]
asyncio_mode = "auto"
markers = [
  "scenario: tier-2 scenario tests requiring Docker",
  "smoke: tier-3 smoke test running the full make up flow",
]

[tool.ruff]
line-length = 100
target-version = "py312"

[tool.ruff.lint]
select = ["E", "F", "I", "B", "UP", "SIM"]

[tool.ruff.lint.per-file-ignores]
# Plugins reference LineBuilder and influxdb3_local as injected free names;
# E501 is OK in plugin/UI/test files where SQL strings and announce() lines
# read better unwrapped.
"plugins/*.py" = ["F821", "E501"]
"ui/queries.py" = ["E501"]
"tests/**/*.py" = ["E501", "E741"]
"simulator/scenarios/*.py" = ["E501"]
```

- [ ] **Step 5: Create README stub**

```markdown
# influxdb3-ref-network-telemetry

Reference architecture: **InfluxDB 3 Enterprise multi-node cluster monitoring a data-center Clos fabric.**

🚧 Under construction — see [`docs/superpowers/specs/2026-04-26-network-telemetry-design.md`](docs/superpowers/specs/2026-04-26-network-telemetry-design.md) for the design.

The polished README ships in the docs phase of the implementation plan.
```

- [ ] **Step 6: Commit**

```bash
cd /Users/pauldix/codez/reference_architectures/influxdb3-ref-network-telemetry
git add LICENSE .gitignore .env.example pyproject.toml README.md
git commit -m "chore: scaffold license, env, pyproject, gitignore"
```

---

## Phase 2 — Simulator

The simulator writes ~10k pts/sec across 4 tables to two ingest nodes (round-robin) for ~1024 fabric interfaces, ~128 BGP sessions, ~5000 flows/sec, and 64 latency probe pairs.

### Task 2: Copy `signals_base.py` from bess + tests

**Files:**
- Create: `simulator/__init__.py`, `simulator/signals_base.py`
- Create: `tests/__init__.py`, `tests/test_signals_base.py`

- [ ] **Step 1: Create the package markers + copy signals_base**

```bash
mkdir -p /Users/pauldix/codez/reference_architectures/influxdb3-ref-network-telemetry/simulator
mkdir -p /Users/pauldix/codez/reference_architectures/influxdb3-ref-network-telemetry/tests
touch /Users/pauldix/codez/reference_architectures/influxdb3-ref-network-telemetry/simulator/__init__.py
touch /Users/pauldix/codez/reference_architectures/influxdb3-ref-network-telemetry/tests/__init__.py
cp /Users/pauldix/codez/reference_architectures/influxdb3-ref-bess/simulator/signals_base.py \
   /Users/pauldix/codez/reference_architectures/influxdb3-ref-network-telemetry/simulator/signals_base.py
```

- [ ] **Step 2: Write tests** — same 7 tests iiot uses

Create `tests/test_signals_base.py`:

```python
"""Tests for signal primitives. Same tests bess and iiot use."""

from __future__ import annotations

import pytest

from simulator.signals_base import burst, jitter, random_walk, sinusoid, step


def test_sinusoid_zero_phase_at_t_zero():
    assert sinusoid(0.0, period_s=10.0, amplitude=2.0, offset=5.0) == pytest.approx(5.0)


def test_sinusoid_quarter_period():
    assert sinusoid(2.5, period_s=10.0, amplitude=2.0, offset=5.0) == pytest.approx(7.0)


def test_random_walk_is_deterministic_given_seed():
    rw_a = random_walk(seed=42, step_std=1.0, start=0.0)
    rw_b = random_walk(seed=42, step_std=1.0, start=0.0)
    seq_a = [rw_a() for _ in range(100)]
    seq_b = [rw_b() for _ in range(100)]
    assert seq_a == seq_b


def test_random_walk_respects_bounds():
    rw = random_walk(seed=7, step_std=10.0, start=0.0, min_val=-1.0, max_val=1.0)
    for _ in range(1000):
        v = rw()
        assert -1.0 <= v <= 1.0


def test_step_function():
    s = step(at_t=10.0, before=0.0, after=100.0)
    assert s(0.0) == 0.0
    assert s(9.999) == 0.0
    assert s(10.0) == 100.0
    assert s(50.0) == 100.0


def test_burst_function():
    b = burst(at_t=5.0, duration_s=2.0, magnitude=3.0)
    assert b(4.999) == 0.0
    assert b(5.0) == 3.0
    assert b(6.999) == 3.0
    assert b(7.0) == 0.0


def test_jitter_is_deterministic_per_t():
    j = jitter(seed=11, std=1.0)
    assert j(1.0) == j(1.0)
    assert j(1.0) != j(2.0)
```

- [ ] **Step 3: Set up venv + run tests — expect PASS**

```bash
cd /Users/pauldix/codez/reference_architectures/influxdb3-ref-network-telemetry
python3 -m venv .venv && source .venv/bin/activate
pip install -e ".[dev]"
pytest tests/test_signals_base.py -v
```

Expected: 7 passed.

- [ ] **Step 4: Commit**

```bash
git add simulator/__init__.py simulator/signals_base.py tests/__init__.py tests/test_signals_base.py
git commit -m "feat(simulator): copy signal primitives library from bess"
```

### Task 3: Domain signals (`signals.py`)

**Files:**
- Create: `simulator/signals.py`
- Create: `tests/test_signals.py`

This file implements the four network signal classes plus the `Fabric` aggregate. Per spec §2.1 + §2.2.

- [ ] **Step 1: Write tests for the network signal classes**

Create `tests/test_signals.py`:

```python
"""Tests for network-telemetry domain signals."""

from __future__ import annotations

import ipaddress

from simulator.signals import (
    BGPSession,
    Fabric,
    FlowRecord,
    InterfaceCounters,
    LatencyProbe,
    build_fabric,
)


def _parse_line(lp: str) -> tuple[str, dict[str, str], dict[str, str], int]:
    measurement_and_tags, rest = lp.split(" ", 1)
    fields_and_ts = rest.rsplit(" ", 1)
    fields_str, ts_str = fields_and_ts[0], fields_and_ts[1]
    parts = measurement_and_tags.split(",")
    measurement = parts[0]
    tags = dict(p.split("=", 1) for p in parts[1:])
    fields = dict(p.split("=", 1) for p in fields_str.split(","))
    return measurement, tags, fields, int(ts_str)


def test_interface_counters_emits_one_line_per_tick():
    ic = InterfaceCounters(
        site="dc-east-1", switch="leaf-01", interface="et-0/0/1",
        capacity_bps=100_000_000_000, seed=1,
    )
    lines = ic.tick(t_seconds=0.0, t_ns=1_000_000_000)
    assert len(lines) == 1
    m, tags, fields, ts = _parse_line(lines[0])
    assert m == "interface_counters"
    assert tags["switch"] == "leaf-01"
    assert tags["interface"] == "et-0/0/1"
    assert "in_bytes" in fields and "out_bytes" in fields
    assert "ecn_marked_pkts" in fields and "pfc_pause_frames" in fields
    assert ts == 1_000_000_000


def test_interface_counters_can_be_overdriven_to_high_utilization():
    ic = InterfaceCounters(
        site="dc-east-1", switch="leaf-07", interface="et-0/0/12",
        capacity_bps=100_000_000_000, seed=1,
    )
    ic.set_utilization(0.94)
    samples = []
    for i in range(5):
        lines = ic.tick(t_seconds=float(i), t_ns=i * 1_000_000_000)
        m, tags, fields, _ = _parse_line(lines[0])
        samples.append(int(fields["in_bytes"]))
    avg_bps = (samples[-1] - samples[0]) / 4 * 8
    # 0.94 × 100Gbps = 94Gbps; allow ±5% noise
    assert 89e9 <= avg_bps <= 99e9


def test_bgp_session_emits_state_and_prefix_count():
    s = BGPSession(site="dc-east-1", switch="leaf-01", peer_switch="spine-1", seed=1)
    lines = s.tick(t_seconds=0.0, t_ns=1_000_000_000)
    assert len(lines) == 1
    m, tags, fields, _ = _parse_line(lines[0])
    assert m == "bgp_sessions"
    assert tags["switch"] == "leaf-01" and tags["peer_switch"] == "spine-1"
    assert fields["state"] == '"established"'
    assert int(fields["prefixes_received"]) > 0


def test_bgp_session_can_be_flapped():
    s = BGPSession(site="dc-east-1", switch="leaf-01", peer_switch="spine-1", seed=1)
    s.set_state("active")
    _, _, fields, _ = _parse_line(s.tick(0.0, 1)[0])
    assert fields["state"] == '"active"'


def test_flow_record_emits_realistic_row():
    rng_seed = 42
    fr = FlowRecord(site="dc-east-1", seed=rng_seed)
    lines = fr.emit_one(t_ns=1_000_000_000)
    assert len(lines) == 1
    m, tags, fields, _ = _parse_line(lines[0])
    assert m == "flow_records"
    # Both endpoints must be valid IPs
    ipaddress.ip_address(tags["src_ip"])
    ipaddress.ip_address(tags["dst_ip"])
    assert tags["src_switch"].startswith(("leaf-", "spine-"))
    assert int(fields["bytes"]) > 0


def test_flow_record_can_be_skewed_to_a_hot_source():
    fr = FlowRecord(site="dc-east-1", seed=42)
    fr.set_hot_source("10.4.7.91", weight=10.0)
    src_ips = []
    for i in range(5000):
        m, tags, _, _ = _parse_line(fr.emit_one(t_ns=i)[0])
        src_ips.append(tags["src_ip"])
    hot_count = src_ips.count("10.4.7.91")
    # With weight 10× and the default IP pool, the hot source should be a
    # very large minority.
    assert hot_count > 1000


def test_latency_probe_emits_rtt_in_microseconds():
    lp = LatencyProbe(site="dc-east-1", src_switch="leaf-01", dst_switch="leaf-02", seed=1)
    lines = lp.tick(t_seconds=0.0, t_ns=1_000_000_000)
    assert len(lines) == 1
    m, _, fields, _ = _parse_line(lines[0])
    assert m == "latency_probes"
    rtt = float(fields["rtt_us"])
    # Inside-fabric latencies are sub-millisecond
    assert 1.0 <= rtt <= 200.0


def test_build_fabric_creates_24_switches():
    fabric = build_fabric(site="dc-east-1", spines=8, leaves=16, servers_per_leaf=48, seed=42)
    assert len(fabric.switches) == 24
    spines = [s for s in fabric.switches if s.role == "spine"]
    leaves = [s for s in fabric.switches if s.role == "leaf"]
    assert len(spines) == 8
    assert len(leaves) == 16


def test_fabric_has_full_mesh_bgp_sessions():
    fabric = build_fabric(site="dc-east-1", spines=8, leaves=16, servers_per_leaf=48, seed=42)
    # Each leaf has a session to every spine: 16 × 8 = 128
    assert len(fabric.bgp_sessions) == 128


def test_fabric_has_correct_interface_count():
    fabric = build_fabric(site="dc-east-1", spines=8, leaves=16, servers_per_leaf=48, seed=42)
    # 8 spines × 16 leaf-facing ports = 128
    # 16 leaves × (8 spine-uplinks + 48 server-downlinks) = 16 × 56 = 896
    # Total: 128 + 896 = 1024
    assert len(fabric.interfaces) == 1024


def test_fabric_tick_returns_lines_from_all_signals():
    fabric = build_fabric(site="dc-east-1", spines=8, leaves=16, servers_per_leaf=48, seed=42)
    lines = fabric.tick(t_seconds=0.0, t_ns=1_000_000_000, flow_count=50)
    # 1024 interfaces + 128 BGP sessions + 64 latency pairs + 50 flow records
    assert len(lines) == 1024 + 128 + 64 + 50
```

- [ ] **Step 2: Run tests — expect FAIL (signals.py doesn't exist)**

```bash
pytest tests/test_signals.py -v
```

Expected: ImportError.

- [ ] **Step 3: Implement `simulator/signals.py`**

```python
"""Network-telemetry domain signal generators.

Each class produces line protocol when ticked. The Fabric aggregate
composes the topology (8 spines + 16 leaves with 48 servers each by
default) and ticks every interface, BGP session, and latency probe in
lockstep, plus emits a configurable number of sampled flow records per
tick.

Signals support runtime overrides (e.g., `InterfaceCounters.set_utilization`,
`BGPSession.set_state`, `FlowRecord.set_hot_source`) so scenarios can
inject specific patterns without modifying the simulator main loop.
"""

from __future__ import annotations

import ipaddress
import random
from dataclasses import dataclass, field
from typing import Literal

from simulator.signals_base import jitter, random_walk

Role = Literal["spine", "leaf"]
BGPState = Literal["established", "idle", "active", "connect", "openconfirm"]


def _esc(s: str) -> str:
    return s.replace(",", r"\,").replace(" ", r"\ ").replace("=", r"\=")


def _tag_block(tags: dict[str, str]) -> str:
    return ",".join(f"{_esc(k)}={_esc(v)}" for k, v in tags.items())


# ---------------------------------------------------------------------------
# Interface counters (1 Hz per interface)
# ---------------------------------------------------------------------------

@dataclass
class InterfaceCounters:
    site: str
    switch: str
    interface: str
    capacity_bps: int        # e.g., 100Gbps = 100_000_000_000
    seed: int
    _utilization: float = field(init=False, default=0.30)
    _in_bytes: int = field(init=False, default=0)
    _out_bytes: int = field(init=False, default=0)
    _in_pkts: int = field(init=False, default=0)
    _out_pkts: int = field(init=False, default=0)
    _in_errors: int = field(init=False, default=0)
    _in_discards: int = field(init=False, default=0)
    _ecn_marked: int = field(init=False, default=0)
    _pfc_pause: int = field(init=False, default=0)
    _noise: object = field(init=False)
    _last_t: float = field(init=False, default=0.0)
    _error_rate: float = field(init=False, default=0.0)

    def __post_init__(self) -> None:
        self._noise = jitter(seed=self.seed, std=0.03)

    def set_utilization(self, fraction: float) -> None:
        """0.0..1.0 — what fraction of capacity_bps to drive."""
        self._utilization = max(0.0, min(1.0, fraction))

    def set_error_rate(self, fraction: float) -> None:
        """0.0..1.0 — fraction of packets that increment in_errors."""
        self._error_rate = max(0.0, min(1.0, fraction))

    def tick(self, t_seconds: float, t_ns: int) -> list[str]:
        dt = max(0.0, t_seconds - self._last_t) if self._last_t else 1.0
        self._last_t = t_seconds
        util = max(0.0, min(1.0, self._utilization + self._noise(t_seconds)))
        bps = util * self.capacity_bps
        bytes_this_tick = int(bps * dt / 8)
        # Symmetric for the demo (~50/50 in vs out)
        self._in_bytes += bytes_this_tick // 2
        self._out_bytes += bytes_this_tick // 2
        # Average packet size 1024B
        pkts_this_tick = bytes_this_tick // 1024
        self._in_pkts += pkts_this_tick // 2
        self._out_pkts += pkts_this_tick // 2
        self._in_errors += int(pkts_this_tick * self._error_rate)
        # ECN: increases sharply above 70% util
        if util > 0.70:
            self._ecn_marked += int(pkts_this_tick * (util - 0.70) * 0.5)
        # PFC pause: increases above 90%
        if util > 0.90:
            self._pfc_pause += int(pkts_this_tick * (util - 0.90) * 0.2)

        tags = _tag_block({
            "site": self.site, "switch": self.switch, "interface": self.interface,
        })
        fields = (
            f"in_bytes={self._in_bytes}i,out_bytes={self._out_bytes}i,"
            f"in_pkts={self._in_pkts}i,out_pkts={self._out_pkts}i,"
            f"in_errors={self._in_errors}i,in_discards={self._in_discards}i,"
            f"optical_power_dbm={-2.0 + self._noise(t_seconds + 0.5):.3f},"
            f"ecn_marked_pkts={self._ecn_marked:.1f},"
            f"pfc_pause_frames={self._pfc_pause:.1f}"
        )
        return [f"interface_counters,{tags} {fields} {t_ns}"]


# ---------------------------------------------------------------------------
# BGP sessions (1 Hz per session)
# ---------------------------------------------------------------------------

@dataclass
class BGPSession:
    site: str
    switch: str
    peer_switch: str
    seed: int
    _state: BGPState = field(init=False, default="established")
    _uptime_s: int = field(init=False, default=0)
    _prefixes: int = field(init=False, default=0)
    _rng: random.Random = field(init=False)

    def __post_init__(self) -> None:
        self._rng = random.Random(self.seed)
        # Realistic per-session prefix count: 100-2000
        self._prefixes = 100 + self._rng.randint(0, 1900)

    def set_state(self, state: BGPState) -> None:
        if state == "established" and self._state != "established":
            self._uptime_s = 0
        self._state = state

    def state(self) -> BGPState:
        return self._state

    def tick(self, t_seconds: float, t_ns: int) -> list[str]:
        if self._state == "established":
            self._uptime_s += 1
        else:
            self._uptime_s = 0
        tags = _tag_block({
            "site": self.site, "switch": self.switch, "peer_switch": self.peer_switch,
        })
        fields = (
            f'state="{self._state}",'
            f"prefixes_received={self._prefixes}i,"
            f"uptime_s={self._uptime_s}i"
        )
        return [f"bgp_sessions,{tags} {fields} {t_ns}"]


# ---------------------------------------------------------------------------
# Flow records (event-driven; sampled like sFlow 1:4096)
# ---------------------------------------------------------------------------

@dataclass
class FlowRecord:
    site: str
    seed: int
    _rng: random.Random = field(init=False)
    _hot_src: str | None = field(init=False, default=None)
    _hot_weight: float = field(init=False, default=1.0)
    # IP pools: 10.0.0.0/8 with /16 per leaf (16 leaves × /16)
    _leaf_ips: list[list[str]] = field(init=False)

    def __post_init__(self) -> None:
        self._rng = random.Random(self.seed)
        # Pre-generate ~100 IPs per leaf for reproducibility
        self._leaf_ips = []
        for li in range(16):
            base = ipaddress.ip_network(f"10.{li}.0.0/16")
            ips = [str(base.network_address + 1 + self._rng.randint(0, 65000))
                   for _ in range(100)]
            self._leaf_ips.append(ips)

    def set_hot_source(self, src_ip: str, weight: float = 10.0) -> None:
        self._hot_src = src_ip
        self._hot_weight = max(1.0, weight)

    def emit_one(self, t_ns: int) -> list[str]:
        # Pick source: hot src wins with probability weight/(weight+pool_size)
        if self._hot_src and self._rng.random() < self._hot_weight / (self._hot_weight + 100):
            src_ip = self._hot_src
            src_leaf_idx = 4 if self._hot_src.startswith("10.4.") else 0
        else:
            src_leaf_idx = self._rng.randint(0, 15)
            src_ip = self._rng.choice(self._leaf_ips[src_leaf_idx])
        # Destination on a different leaf for east-west traffic
        dst_leaf_idx = (src_leaf_idx + self._rng.randint(1, 15)) % 16
        dst_ip = self._rng.choice(self._leaf_ips[dst_leaf_idx])
        bytes_ = self._rng.randint(64, 1500) * self._rng.randint(1, 100)
        packets = max(1, bytes_ // 1024)
        duration_ms = self._rng.uniform(0.1, 50.0)
        tags = _tag_block({
            "site": self.site,
            "src_ip": src_ip,
            "dst_ip": dst_ip,
            "src_switch": f"leaf-{src_leaf_idx + 1:02d}",
            "dst_switch": f"leaf-{dst_leaf_idx + 1:02d}",
            "vrf": "default",
        })
        fields = (
            f"bytes={bytes_}i,packets={packets}i,"
            f"duration_ms={duration_ms:.3f}"
        )
        return [f"flow_records,{tags} {fields} {t_ns}"]


# ---------------------------------------------------------------------------
# Latency probes (1 Hz per pair)
# ---------------------------------------------------------------------------

@dataclass
class LatencyProbe:
    site: str
    src_switch: str
    dst_switch: str
    seed: int
    _baseline_us: float = field(init=False, default=0.0)
    _noise: object = field(init=False)

    def __post_init__(self) -> None:
        rng = random.Random(self.seed)
        # Spine-leaf: 5-15 us; leaf-leaf via spine: 10-30 us
        self._baseline_us = rng.uniform(5.0, 30.0)
        self._noise = jitter(seed=self.seed + 1000, std=2.0)

    def tick(self, t_seconds: float, t_ns: int) -> list[str]:
        rtt = max(1.0, self._baseline_us + self._noise(t_seconds))
        tags = _tag_block({
            "site": self.site,
            "src_switch": self.src_switch,
            "dst_switch": self.dst_switch,
        })
        return [f"latency_probes,{tags} rtt_us={rtt:.3f} {t_ns}"]


# ---------------------------------------------------------------------------
# Aggregates
# ---------------------------------------------------------------------------

@dataclass
class Switch:
    role: Role
    name: str  # "spine-1", "leaf-07"


@dataclass
class Fabric:
    site: str
    switches: list[Switch]
    interfaces: list[InterfaceCounters]
    bgp_sessions: list[BGPSession]
    latency_probes: list[LatencyProbe]
    flow_record_emitter: FlowRecord

    def tick(self, t_seconds: float, t_ns: int, flow_count: int) -> list[str]:
        out: list[str] = []
        for ic in self.interfaces:
            out.extend(ic.tick(t_seconds, t_ns))
        for s in self.bgp_sessions:
            out.extend(s.tick(t_seconds, t_ns))
        for lp in self.latency_probes:
            out.extend(lp.tick(t_seconds, t_ns))
        for _ in range(flow_count):
            out.extend(self.flow_record_emitter.emit_one(t_ns))
        return out

    def find_interface(self, switch: str, interface: str) -> InterfaceCounters:
        for ic in self.interfaces:
            if ic.switch == switch and ic.interface == interface:
                return ic
        raise KeyError(f"{switch}/{interface}")

    def find_bgp_session(self, switch: str, peer_switch: str) -> BGPSession:
        for s in self.bgp_sessions:
            if s.switch == switch and s.peer_switch == peer_switch:
                return s
        raise KeyError(f"{switch}↔{peer_switch}")


def build_fabric(
    site: str, spines: int, leaves: int, servers_per_leaf: int, seed: int,
) -> Fabric:
    switches: list[Switch] = []
    interfaces: list[InterfaceCounters] = []
    bgp_sessions: list[BGPSession] = []
    latency_probes: list[LatencyProbe] = []

    for i in range(1, spines + 1):
        switches.append(Switch(role="spine", name=f"spine-{i}"))
    for j in range(1, leaves + 1):
        switches.append(Switch(role="leaf", name=f"leaf-{j:02d}"))

    # Spine-side interfaces (one per leaf)
    for spine in [s for s in switches if s.role == "spine"]:
        for j in range(1, leaves + 1):
            iface = f"et-0/0/{j}"
            interfaces.append(InterfaceCounters(
                site=site, switch=spine.name, interface=iface,
                capacity_bps=100_000_000_000,
                seed=seed + hash((spine.name, iface)) % 100_000,
            ))

    # Leaf-side spine uplinks + server downlinks
    for leaf in [s for s in switches if s.role == "leaf"]:
        for i in range(1, spines + 1):
            iface = f"et-0/0/{i}"  # uplink to spine-i
            interfaces.append(InterfaceCounters(
                site=site, switch=leaf.name, interface=iface,
                capacity_bps=100_000_000_000,
                seed=seed + hash((leaf.name, iface)) % 100_000,
            ))
        for k in range(1, servers_per_leaf + 1):
            iface = f"et-0/1/{k}"  # server downlink
            interfaces.append(InterfaceCounters(
                site=site, switch=leaf.name, interface=iface,
                capacity_bps=25_000_000_000,
                seed=seed + hash((leaf.name, iface)) % 100_000,
            ))

    # Full-mesh BGP: each leaf to every spine
    for leaf in [s for s in switches if s.role == "leaf"]:
        for spine in [s for s in switches if s.role == "spine"]:
            bgp_sessions.append(BGPSession(
                site=site, switch=leaf.name, peer_switch=spine.name,
                seed=seed + hash((leaf.name, spine.name)) % 100_000,
            ))

    # Latency probes: spine pairs (28) + leaf pairs sampled (36) = 64
    spine_names = [s.name for s in switches if s.role == "spine"]
    leaf_names = [s.name for s in switches if s.role == "leaf"]
    pair_count = 0
    for i, a in enumerate(spine_names):
        for b in spine_names[i + 1:]:
            latency_probes.append(LatencyProbe(
                site=site, src_switch=a, dst_switch=b,
                seed=seed + hash((a, b)) % 100_000,
            ))
            pair_count += 1
    leaf_rng = random.Random(seed + 999)
    leaf_pairs_seen: set[tuple[str, str]] = set()
    while pair_count < 64:
        a = leaf_rng.choice(leaf_names)
        b = leaf_rng.choice(leaf_names)
        if a >= b or (a, b) in leaf_pairs_seen:
            continue
        leaf_pairs_seen.add((a, b))
        latency_probes.append(LatencyProbe(
            site=site, src_switch=a, dst_switch=b,
            seed=seed + hash((a, b)) % 100_000,
        ))
        pair_count += 1

    flow_record_emitter = FlowRecord(site=site, seed=seed + 7777)

    return Fabric(
        site=site,
        switches=switches,
        interfaces=interfaces,
        bgp_sessions=bgp_sessions,
        latency_probes=latency_probes,
        flow_record_emitter=flow_record_emitter,
    )
```

- [ ] **Step 4: Run tests — expect PASS**

```bash
pytest tests/test_signals.py -v
```

Expected: 11 passed.

- [ ] **Step 5: Commit**

```bash
git add simulator/signals.py tests/test_signals.py
git commit -m "feat(simulator): network-telemetry domain signals (interfaces, BGP, flows, latency)"
```

### Task 4: Writer with round-robin + config + main loop

**Files:**
- Create: `simulator/writer.py`, `simulator/config.py`, `simulator/main.py`
- Create: `tests/test_writer.py`

The writer round-robins across two ingest URLs, with one fallback hop on connection failure. This is THE pattern that the spec's CONVENTIONS amendment will codify for multi-node repos.

- [ ] **Step 1: Write tests** for the writer

Create `tests/test_writer.py`:

```python
"""Tests for the line-protocol HTTP writer with round-robin + fallback."""

from __future__ import annotations

import pytest

from simulator.writer import InfluxDB3Writer


class _RecordingTransport:
    """Records every (url, content) pair. By default returns 200."""

    def __init__(self, fail_urls: set[str] | None = None) -> None:
        self.calls: list[tuple[str, str]] = []
        self._fail_urls = fail_urls or set()

    def post(self, url: str, content: str, headers: dict[str, str]) -> None:
        self.calls.append((url, content))
        if url in self._fail_urls:
            raise ConnectionError(f"recording transport: refusing {url}")


def test_writer_batches_lines_until_flush():
    t = _RecordingTransport()
    w = InfluxDB3Writer(
        urls=["http://ingest-1:8181", "http://ingest-2:8181"],
        database="nt", token="tok", batch_size=3, transport=t,
    )
    w.write("m1,t=1 v=1 1")
    w.write("m1,t=1 v=2 2")
    assert t.calls == []
    w.write("m1,t=1 v=3 3")
    assert len(t.calls) == 1


def test_writer_round_robins_across_urls():
    t = _RecordingTransport()
    w = InfluxDB3Writer(
        urls=["http://ingest-1:8181", "http://ingest-2:8181"],
        database="nt", token="tok", batch_size=1, transport=t,
    )
    w.write("m1,t=1 v=1 1")
    w.write("m1,t=1 v=2 2")
    w.write("m1,t=1 v=3 3")
    w.write("m1,t=1 v=4 4")
    urls = [c[0] for c in t.calls]
    assert "ingest-1" in urls[0]
    assert "ingest-2" in urls[1]
    assert "ingest-1" in urls[2]
    assert "ingest-2" in urls[3]


def test_writer_falls_back_on_connection_error():
    t = _RecordingTransport(fail_urls={
        "http://ingest-1:8181/api/v3/write_lp?db=nt&precision=nanosecond",
    })
    w = InfluxDB3Writer(
        urls=["http://ingest-1:8181", "http://ingest-2:8181"],
        database="nt", token="tok", batch_size=1, transport=t,
    )
    w.write("m1,t=1 v=1 1")
    # First call to ingest-1 fails, fallback succeeds on ingest-2
    assert len(t.calls) == 2
    assert "ingest-1" in t.calls[0][0]
    assert "ingest-2" in t.calls[1][0]


def test_writer_raises_when_both_targets_fail():
    t = _RecordingTransport(fail_urls={
        "http://ingest-1:8181/api/v3/write_lp?db=nt&precision=nanosecond",
        "http://ingest-2:8181/api/v3/write_lp?db=nt&precision=nanosecond",
    })
    w = InfluxDB3Writer(
        urls=["http://ingest-1:8181", "http://ingest-2:8181"],
        database="nt", token="tok", batch_size=1, transport=t,
    )
    with pytest.raises(ConnectionError):
        w.write("m1,t=1 v=1 1")


def test_writer_flush_is_noop_when_empty():
    t = _RecordingTransport()
    w = InfluxDB3Writer(
        urls=["http://ingest-1:8181"],
        database="nt", token="tok", batch_size=10, transport=t,
    )
    w.flush()
    assert t.calls == []
```

- [ ] **Step 2: Run tests — expect FAIL**

```bash
pytest tests/test_writer.py -v
```

Expected: ImportError.

- [ ] **Step 3: Implement `simulator/writer.py`**

```python
"""HTTP line-protocol writer for InfluxDB 3 with round-robin across N
ingest URLs and one-hop fallback on connection error.

The transport is injectable so unit tests can capture payloads and
simulate per-URL connection failures.
"""

from __future__ import annotations

from typing import Protocol

import httpx


class Transport(Protocol):
    def post(self, url: str, content: str, headers: dict[str, str]) -> None: ...


class _HttpxTransport:
    def __init__(self, timeout_s: float = 10.0) -> None:
        self._client = httpx.Client(timeout=timeout_s)

    def post(self, url: str, content: str, headers: dict[str, str]) -> None:
        r = self._client.post(url, content=content, headers=headers)
        r.raise_for_status()


class InfluxDB3Writer:
    def __init__(
        self,
        urls: list[str],
        database: str,
        token: str,
        batch_size: int = 1000,
        transport: Transport | None = None,
    ) -> None:
        if not urls:
            raise ValueError("urls must contain at least one base URL")
        self._endpoints = [
            f"{u.rstrip('/')}/api/v3/write_lp?db={database}&precision=nanosecond"
            for u in urls
        ]
        self._token = token
        self._batch_size = batch_size
        self._buf: list[str] = []
        self._transport: Transport = transport or _HttpxTransport()
        self._next_idx = 0

    def write(self, line: str) -> None:
        self._buf.append(line)
        if len(self._buf) >= self._batch_size:
            self.flush()

    def flush(self) -> None:
        if not self._buf:
            return
        payload = "\n".join(self._buf)
        headers = {
            "Authorization": f"Bearer {self._token}",
            "Content-Type": "text/plain; charset=utf-8",
        }
        # Try each endpoint starting at the next round-robin slot, advancing
        # by one for the next call (or further on failure). One full sweep
        # over endpoints; raise the last error if all failed.
        n = len(self._endpoints)
        primary = self._next_idx
        last_err: Exception | None = None
        for offset in range(n):
            idx = (primary + offset) % n
            url = self._endpoints[idx]
            try:
                self._transport.post(url, payload, headers)
                self._next_idx = (idx + 1) % n
                self._buf.clear()
                return
            except Exception as e:  # noqa: BLE001
                last_err = e
                continue
        # All endpoints failed
        assert last_err is not None
        raise last_err
```

- [ ] **Step 4: Run tests — expect PASS**

```bash
pytest tests/test_writer.py -v
```

Expected: 5 passed.

- [ ] **Step 5: Implement `simulator/config.py`**

```python
"""Simulator configuration loaded from environment variables.

Multi-node specific: SIM_INGEST_URLS is a comma-separated list of base
URLs (no path); the writer round-robins across them.
"""

from __future__ import annotations

import json
import os
from dataclasses import dataclass


@dataclass(frozen=True)
class Config:
    ingest_urls: list[str]
    database: str
    token: str
    site: str
    rate_hz: float
    seed: int
    spines: int
    leaves: int
    servers_per_leaf: int
    flow_records_per_sec: int
    duration_s: float | None


def _load_token() -> str:
    if "INFLUXDB3_TOKEN" in os.environ:
        return os.environ["INFLUXDB3_TOKEN"]
    path = os.environ.get("INFLUX_TOKEN_FILE", "/tokens/.nt-operator-token")
    with open(path) as f:
        return json.load(f)["token"]


def load() -> Config:
    urls_csv = os.environ.get(
        "SIM_INGEST_URLS",
        "http://nt-ingest-1:8181,http://nt-ingest-2:8181",
    )
    urls = [u.strip() for u in urls_csv.split(",") if u.strip()]
    return Config(
        ingest_urls=urls,
        database=os.environ.get("INFLUX_DB", "nt"),
        token=_load_token(),
        site=os.environ.get("SIM_SITE", "dc-east-1"),
        rate_hz=float(os.environ.get("SIM_RATE_HZ", "1.0")),
        seed=int(os.environ.get("SIM_SEED", "42")),
        spines=int(os.environ.get("SIM_SPINES", "8")),
        leaves=int(os.environ.get("SIM_LEAVES", "16")),
        servers_per_leaf=int(os.environ.get("SIM_SERVERS_PER_LEAF", "48")),
        flow_records_per_sec=int(os.environ.get("SIM_FLOW_RECORDS_PER_SEC", "5000")),
        duration_s=float(os.environ["SIM_DURATION_S"]) if "SIM_DURATION_S" in os.environ else None,
    )
```

- [ ] **Step 6: Implement `simulator/main.py`**

```python
"""Simulator entry point. Ticks the fabric and writes line protocol via
the round-robin writer to two ingest nodes."""

from __future__ import annotations

import logging
import time

from simulator.config import load
from simulator.signals import build_fabric
from simulator.writer import InfluxDB3Writer

logging.basicConfig(level=logging.INFO, format="%(asctime)s %(levelname)s %(message)s")
log = logging.getLogger("simulator")


def main() -> None:
    cfg = load()
    log.info("starting simulator: %s", cfg)
    writer = InfluxDB3Writer(
        urls=cfg.ingest_urls,
        database=cfg.database,
        token=cfg.token,
        batch_size=2000,
    )
    fabric = build_fabric(
        site=cfg.site,
        spines=cfg.spines,
        leaves=cfg.leaves,
        servers_per_leaf=cfg.servers_per_leaf,
        seed=cfg.seed,
    )
    period = 1.0 / cfg.rate_hz
    flows_per_tick = max(1, int(cfg.flow_records_per_sec * period))
    t0 = time.time()
    tick = 0
    try:
        while True:
            now = time.time()
            t_seconds = now - t0
            if cfg.duration_s is not None and t_seconds >= cfg.duration_s:
                break
            t_ns = int(now * 1_000_000_000)
            for line in fabric.tick(t_seconds, t_ns, flow_count=flows_per_tick):
                writer.write(line)
            tick += 1
            if tick % 10 == 0:
                writer.flush()
                log.info("tick=%d t=%.1fs flows/tick=%d", tick, t_seconds, flows_per_tick)
            sleep_for = period - (time.time() - now)
            if sleep_for > 0:
                time.sleep(sleep_for)
    finally:
        writer.flush()
        log.info("simulator exiting after %d ticks", tick)


if __name__ == "__main__":
    main()
```

- [ ] **Step 7: Verify imports**

```bash
python -c "from simulator import signals, writer, config, main; print('ok')"
```

Expected: `ok`.

- [ ] **Step 8: Commit**

```bash
git add simulator/writer.py simulator/config.py simulator/main.py tests/test_writer.py
git commit -m "feat(simulator): writer with round-robin + config + main loop"
```

### Task 5: Simulator Dockerfile

**Files:**
- Create: `simulator/Dockerfile`

- [ ] **Step 1: Create Dockerfile**

```dockerfile
FROM python:3.12-slim

WORKDIR /app

RUN pip install --no-cache-dir \
      "influxdb3-python>=0.7" \
      "httpx>=0.27"

COPY simulator /app/simulator
ENV PYTHONPATH=/app

CMD ["python", "-m", "simulator.main"]
```

- [ ] **Step 2: Build to verify**

```bash
docker build -t nt-simulator-check -f simulator/Dockerfile .
docker rmi nt-simulator-check
```

- [ ] **Step 3: Commit**

```bash
git add simulator/Dockerfile
git commit -m "feat(simulator): Dockerfile"
```

---

## Phase 3 — InfluxDB cluster, init.sh, compose, Makefile

This is where multi-node reality hits. The compose file has **10 services**: token-bootstrap (one-shot) + 5 InfluxDB nodes + influxdb3-init (one-shot) + simulator + ui + scenarios. All five InfluxDB nodes mount the same `influxdb-data` named volume at `/var/lib/influxdb3` — that's what gives the cluster a shared catalog and object store.

**Phase 0 findings drive the exact CLI/JSON in this phase.** If `every:5s` was rejected in Phase 0, swap it to `cron:*/5 * * * * *` everywhere it appears below. If `/api/v3/configure/*` doesn't exist or has a different shape, swap the curl calls in init.sh for the equivalent `influxdb3 create ...` CLI commands. Adapt before running this phase.

### Task 6: `influxdb/init.sh` — DB, tables, caches (triggers added in Task 17)

**Files:**
- Create: `influxdb/init.sh`
- Create: `influxdb/schema.md`

The init script targets the query node for all configure API calls (it's exposed on 8181 internally and the catalog is shared, so any node works — we pick query for consistency with how `make cli` connects).

- [ ] **Step 1: Create `influxdb/init.sh`**

```bash
#!/usr/bin/env bash
# Runs inside the influxdb3-init container after the cluster is up.
# Idempotent: safe to re-run.
#
# Creates the nt database, all six tables (with explicit schemas via the
# configure API so caches/triggers can reference them without a sentinel-
# row trick), the LVC and two DVCs, and (in Task 17) the four Processing
# Engine triggers. Targets the query node for all calls — the catalog is
# shared via the influxdb-data volume so every node sees the same state.

set -euo pipefail

INFLUX_HOST="${INFLUX_HOST:-http://nt-query:8181}"
INFLUX_DB="${INFLUX_DB:-nt}"
TOKEN_FILE="/var/lib/influxdb3/.nt-operator-token"
PLAIN_TOKEN_FILE="/var/lib/influxdb3/.nt-token-plain"

log() { echo "[init] $*"; }

wait_for_api() {
    local url="${INFLUX_HOST}/health"
    for _ in $(seq 1 120); do
        if curl -s --max-time 2 -o /dev/null "$url"; then return 0; fi
        sleep 1
    done
    echo "[init] FATAL: ${INFLUX_HOST} API did not become ready" >&2
    exit 1
}

read_token_json() {
    sed -n 's/.*"token"[[:space:]]*:[[:space:]]*"\([^"]*\)".*/\1/p' "$1" | head -n1
}

ensure_token() {
    if [[ ! -s "${TOKEN_FILE}" ]]; then
        echo "[init] FATAL: ${TOKEN_FILE} missing — token-bootstrap failed" >&2
        exit 1
    fi
    if [[ ! -s "${PLAIN_TOKEN_FILE}" ]]; then
        read_token_json "${TOKEN_FILE}" > "${PLAIN_TOKEN_FILE}"
        chmod 600 "${PLAIN_TOKEN_FILE}"
    fi
    log "admin token present"
}

cli() {
    local token
    token=$(read_token_json "${TOKEN_FILE}")
    influxdb3 "$@" --host "${INFLUX_HOST}" --token "${token}"
}

idempotent() {
    local label="$1"; shift
    local out
    if out=$(cli "$@" 2>&1); then
        log "created ${label}"
    elif echo "$out" | grep -qE "already exists|Conflict|409"; then
        log "${label} already exists"
    else
        echo "[init] FATAL while creating ${label}: $out" >&2
        exit 1
    fi
}

ensure_database() {
    idempotent "database ${INFLUX_DB}" create database "${INFLUX_DB}"
}

# Explicit table creation. No sentinel rows.
# Each `create table` gives the full tag set + field schema. The
# fabric_health table additionally gets a 24h retention period.
ensure_tables() {
    # interface_counters: per-interface telemetry (1 Hz × ~1024 interfaces)
    idempotent "table interface_counters" create table interface_counters \
        --database "${INFLUX_DB}" \
        --tags site,switch,interface \
        --fields "in_bytes:int64,out_bytes:int64,in_pkts:int64,out_pkts:int64,in_errors:int64,in_discards:int64,optical_power_dbm:float64,ecn_marked_pkts:float64,pfc_pause_frames:float64"

    # bgp_sessions: per-session state (1 Hz × 128 sessions)
    idempotent "table bgp_sessions" create table bgp_sessions \
        --database "${INFLUX_DB}" \
        --tags site,switch,peer_switch \
        --fields "state:string,prefixes_received:int64,uptime_s:int64"

    # flow_records: sampled (sFlow-like) per-flow records (~5k/s)
    idempotent "table flow_records" create table flow_records \
        --database "${INFLUX_DB}" \
        --tags site,src_ip,dst_ip,src_switch,dst_switch,vrf \
        --fields "bytes:int64,packets:int64,duration_ms:float64"

    # latency_probes: switch-pair RTT probes (1 Hz × 64 pairs)
    idempotent "table latency_probes" create table latency_probes \
        --database "${INFLUX_DB}" \
        --tags site,src_switch,dst_switch \
        --fields "rtt_us:float64"

    # fabric_health: rollup written by schedule_fabric_health every 5s.
    # 24h retention demonstrates per-table retention.
    idempotent "table fabric_health" create table fabric_health \
        --database "${INFLUX_DB}" \
        --tags site,layer \
        --fields "status:string,spines_up:int64,leaves_up:int64,bgp_up:int64,ingress_bps:float64,egress_bps:float64,p95_latency_us:float64,ecn_pct:float64" \
        --retention-period 24h

    # anomalies: rows written by schedule_anomaly_detector when problems
    # are detected. No retention — historical anomalies stick around.
    idempotent "table anomalies" create table anomalies \
        --database "${INFLUX_DB}" \
        --tags site,severity,kind,switch,interface \
        --fields "reason:string,value:float64,started_at_ns:float64"
}

ensure_caches() {
    # LVC on bgp_sessions (one row per leaf↔spine session pair)
    idempotent "last_cache bgp_session_last" create last_cache bgp_session_last \
        --database "${INFLUX_DB}" --table bgp_sessions

    # DVCs on flow_records src/dst IPs (powers the typeahead)
    idempotent "distinct_cache src_ip_distinct" create distinct_cache src_ip_distinct \
        --database "${INFLUX_DB}" --table flow_records --columns src_ip
    idempotent "distinct_cache dst_ip_distinct" create distinct_cache dst_ip_distinct \
        --database "${INFLUX_DB}" --table flow_records --columns dst_ip
}

ensure_triggers() {
    log "no triggers yet (added in Task 17)"
}

main() {
    wait_for_api
    ensure_token
    ensure_database
    ensure_tables
    ensure_caches
    ensure_triggers
    log "initialization complete"
}

main "$@"
```

- [ ] **Step 2: chmod + sanity-check syntax**

```bash
chmod +x /Users/pauldix/codez/reference_architectures/influxdb3-ref-network-telemetry/influxdb/init.sh
bash -n /Users/pauldix/codez/reference_architectures/influxdb3-ref-network-telemetry/influxdb/init.sh
shellcheck /Users/pauldix/codez/reference_architectures/influxdb3-ref-network-telemetry/influxdb/init.sh || true
```

- [ ] **Step 3: Create `influxdb/schema.md`**

```markdown
# Network Telemetry schema

Reference for the six tables in the `nt` database. Tables are created
explicitly by `init.sh` via the configure API at first boot — no
sentinel rows are written, so caches and triggers can reference the
tables immediately and there is no `__init` row to filter out.

## Tables

| Table | Tags | Fields | Source | Retention |
|-------|------|--------|--------|-----------|
| `interface_counters` | site, switch, interface | in_bytes, out_bytes, in_pkts, out_pkts, in_errors, in_discards (i64), optical_power_dbm, ecn_marked_pkts, pfc_pause_frames (f64) | simulator (1 Hz/interface) | none |
| `bgp_sessions` | site, switch, peer_switch | state (string), prefixes_received, uptime_s (i64) | simulator (1 Hz/session) | none |
| `flow_records` | site, src_ip, dst_ip, src_switch, dst_switch, vrf | bytes, packets (i64), duration_ms (f64) | simulator (~5 k/s sampled) | none |
| `latency_probes` | site, src_switch, dst_switch | rtt_us (f64) | simulator (1 Hz/pair) | none |
| `fabric_health` | site, layer | status (string), spines_up, leaves_up, bgp_up (i64), ingress_bps, egress_bps, p95_latency_us, ecn_pct (f64) | plugin (`schedule_fabric_health`) | **24 hours** |
| `anomalies` | site, severity, kind, switch, interface | reason (string), value, started_at_ns (f64) | plugin (`schedule_anomaly_detector`) | none |

## Caches

- **Last Value Cache** `bgp_session_last` on `bgp_sessions` keyed by
  (site, switch, peer_switch). Powers the BGP up-count in the fabric
  banner; read by `schedule_anomaly_detector` for fast "current session
  state" lookups.
- **Distinct Value Cache** `src_ip_distinct` on `flow_records.src_ip`.
  Powers the source-IP typeahead in the UI — direct SQL fetch from the
  browser uses `distinct_cache('flow_records', 'src_ip_distinct')` for
  sub-millisecond prefix lookup.
- **Distinct Value Cache** `dst_ip_distinct` on `flow_records.dst_ip`.
  Available for future drill-downs (not used by v1 UI).

## Retention

- `fabric_health`: 24 hours. Demonstrates per-table retention. The
  schedule plugin writes one row every 5 s (~17 k rows/day), retention
  drops anything older.
- All other tables: no retention in the demo. `ARCHITECTURE.md` discusses
  production retention guidance (raw counters 7–30 days; flow records
  30–90 days depending on regulatory needs; anomalies indefinite).
```

- [ ] **Step 4: Commit**

```bash
git add influxdb/init.sh influxdb/schema.md
git commit -m "feat(influxdb): init.sh with configure API table creation + schema doc"
```

### Task 7: `docker-compose.yml` (10 services, 5-node cluster)

**Files:**
- Create: `docker-compose.yml`

- [ ] **Step 1: Create `docker-compose.yml`**

The shared `influxdb-data` volume mounted at `/var/lib/influxdb3` on every InfluxDB service is what gives us the shared catalog. All five nodes use `--object-store file --data-dir /var/lib/influxdb3`, so the file backend resolves to the same path on the same volume → consistent.

```yaml
name: influxdb3-ref-network-telemetry

services:
  # One-shot: generate the offline admin token shared by the whole cluster.
  token-bootstrap:
    image: influxdb:3-enterprise
    container_name: nt-token-bootstrap
    environment:
      INFLUXDB3_UNSET_VARS: LOG_FILTER
      INFLUXDB3_LOG_FILTER: info
    volumes:
      - influxdb-data:/var/lib/influxdb3
    entrypoint: ["sh", "-c"]
    command:
      - |
        if [ -s /var/lib/influxdb3/.nt-operator-token ]; then
          echo "[bootstrap] admin token already exists, skipping"
          exit 0
        fi
        echo "[bootstrap] generating offline admin token"
        influxdb3 create token --admin --offline \
          --name nt-admin \
          --output-file /var/lib/influxdb3/.nt-operator-token
        chmod 600 /var/lib/influxdb3/.nt-operator-token
        echo "[bootstrap] admin token written"
    restart: "no"

  # ── Ingest tier: 2 nodes, simulator round-robins, plugin write-back targets here ──

  influxdb3-ingest-1:
    image: influxdb:3-enterprise
    container_name: nt-ingest-1
    depends_on:
      token-bootstrap:
        condition: service_completed_successfully
    environment:
      INFLUXDB3_ENTERPRISE_LICENSE_EMAIL: ${INFLUXDB3_ENTERPRISE_EMAIL:?set INFLUXDB3_ENTERPRISE_EMAIL in .env}
      INFLUXDB3_ENTERPRISE_LICENSE_TYPE: ${INFLUXDB3_ENTERPRISE_LICENSE_TYPE:-trial}
      INFLUXDB3_NODE_IDENTIFIER_PREFIX: nt-node
      INFLUXDB3_OBJECT_STORE: file
      INFLUXDB3_DATA_DIR: /var/lib/influxdb3
      INFLUXDB3_UNSET_VARS: LOG_FILTER
      INFLUXDB3_LOG_FILTER: info
    command: >
      serve
        --node-id nt-ingest-1
        --cluster-id nt
        --mode ingest
        --object-store file
        --data-dir /var/lib/influxdb3
        --admin-token-file /var/lib/influxdb3/.nt-operator-token
    volumes:
      - influxdb-data:/var/lib/influxdb3
    healthcheck:
      test: ["CMD-SHELL", "curl -s --max-time 2 -o /dev/null http://127.0.0.1:8181/health && test -s /var/lib/influxdb3/.nt-operator-token"]
      interval: 5s
      timeout: 3s
      retries: 120
      start_period: 5s

  influxdb3-ingest-2:
    image: influxdb:3-enterprise
    container_name: nt-ingest-2
    depends_on:
      token-bootstrap:
        condition: service_completed_successfully
    environment:
      INFLUXDB3_ENTERPRISE_LICENSE_EMAIL: ${INFLUXDB3_ENTERPRISE_EMAIL:?set INFLUXDB3_ENTERPRISE_EMAIL in .env}
      INFLUXDB3_ENTERPRISE_LICENSE_TYPE: ${INFLUXDB3_ENTERPRISE_LICENSE_TYPE:-trial}
      INFLUXDB3_NODE_IDENTIFIER_PREFIX: nt-node
      INFLUXDB3_OBJECT_STORE: file
      INFLUXDB3_DATA_DIR: /var/lib/influxdb3
      INFLUXDB3_UNSET_VARS: LOG_FILTER
      INFLUXDB3_LOG_FILTER: info
    command: >
      serve
        --node-id nt-ingest-2
        --cluster-id nt
        --mode ingest
        --object-store file
        --data-dir /var/lib/influxdb3
        --admin-token-file /var/lib/influxdb3/.nt-operator-token
    volumes:
      - influxdb-data:/var/lib/influxdb3
    healthcheck:
      test: ["CMD-SHELL", "curl -s --max-time 2 -o /dev/null http://127.0.0.1:8181/health && test -s /var/lib/influxdb3/.nt-operator-token"]
      interval: 5s
      timeout: 3s
      retries: 120
      start_period: 5s

  # ── Query tier: 1 node, exposed to host on 8181 ──

  influxdb3-query:
    image: influxdb:3-enterprise
    container_name: nt-query
    depends_on:
      token-bootstrap:
        condition: service_completed_successfully
    environment:
      INFLUXDB3_ENTERPRISE_LICENSE_EMAIL: ${INFLUXDB3_ENTERPRISE_EMAIL:?set INFLUXDB3_ENTERPRISE_EMAIL in .env}
      INFLUXDB3_ENTERPRISE_LICENSE_TYPE: ${INFLUXDB3_ENTERPRISE_LICENSE_TYPE:-trial}
      INFLUXDB3_PLUGIN_DIR: /plugins
      INFLUXDB3_NODE_IDENTIFIER_PREFIX: nt-node
      INFLUXDB3_OBJECT_STORE: file
      INFLUXDB3_DATA_DIR: /var/lib/influxdb3
      INFLUXDB3_UNSET_VARS: LOG_FILTER
      INFLUXDB3_LOG_FILTER: info
    command: >
      serve
        --node-id nt-query
        --cluster-id nt
        --mode query
        --object-store file
        --data-dir /var/lib/influxdb3
        --plugin-dir /plugins
        --virtual-env-location /var/lib/influxdb3/plugin-venv-query
        --admin-token-file /var/lib/influxdb3/.nt-operator-token
        --admin-token-recovery-http-bind 0.0.0.0:8182
    volumes:
      - influxdb-data:/var/lib/influxdb3
      - ./plugins:/plugins:ro
    ports:
      - "8181:8181"
    healthcheck:
      test: ["CMD-SHELL", "curl -s --max-time 2 -o /dev/null http://127.0.0.1:8181/health && test -s /var/lib/influxdb3/.nt-operator-token"]
      interval: 5s
      timeout: 3s
      retries: 120
      start_period: 5s

  # ── Compact tier: 1 node, internal only ──

  influxdb3-compact:
    image: influxdb:3-enterprise
    container_name: nt-compact
    depends_on:
      token-bootstrap:
        condition: service_completed_successfully
    environment:
      INFLUXDB3_ENTERPRISE_LICENSE_EMAIL: ${INFLUXDB3_ENTERPRISE_EMAIL:?set INFLUXDB3_ENTERPRISE_EMAIL in .env}
      INFLUXDB3_ENTERPRISE_LICENSE_TYPE: ${INFLUXDB3_ENTERPRISE_LICENSE_TYPE:-trial}
      INFLUXDB3_NODE_IDENTIFIER_PREFIX: nt-node
      INFLUXDB3_OBJECT_STORE: file
      INFLUXDB3_DATA_DIR: /var/lib/influxdb3
      INFLUXDB3_UNSET_VARS: LOG_FILTER
      INFLUXDB3_LOG_FILTER: info
    command: >
      serve
        --node-id nt-compact
        --cluster-id nt
        --mode compact
        --object-store file
        --data-dir /var/lib/influxdb3
        --admin-token-file /var/lib/influxdb3/.nt-operator-token
    volumes:
      - influxdb-data:/var/lib/influxdb3
    healthcheck:
      test: ["CMD-SHELL", "curl -s --max-time 2 -o /dev/null http://127.0.0.1:8181/health && test -s /var/lib/influxdb3/.nt-operator-token"]
      interval: 5s
      timeout: 3s
      retries: 120
      start_period: 5s

  # ── Process tier: 1 node, runs schedule plugins. Mode is process,query
  #    so plugin code can call influxdb3_local.query() against the local
  #    engine. Plugin write-back goes via httpx to ingest-1 (with ingest-2
  #    fallback) — see plugins/_writeback.py.

  influxdb3-process:
    image: influxdb:3-enterprise
    container_name: nt-process
    depends_on:
      token-bootstrap:
        condition: service_completed_successfully
      influxdb3-ingest-1:
        condition: service_healthy
    environment:
      INFLUXDB3_ENTERPRISE_LICENSE_EMAIL: ${INFLUXDB3_ENTERPRISE_EMAIL:?set INFLUXDB3_ENTERPRISE_EMAIL in .env}
      INFLUXDB3_ENTERPRISE_LICENSE_TYPE: ${INFLUXDB3_ENTERPRISE_LICENSE_TYPE:-trial}
      INFLUXDB3_PLUGIN_DIR: /plugins
      INFLUXDB3_NODE_IDENTIFIER_PREFIX: nt-node
      INFLUXDB3_OBJECT_STORE: file
      INFLUXDB3_DATA_DIR: /var/lib/influxdb3
      INFLUXDB3_UNSET_VARS: LOG_FILTER
      INFLUXDB3_LOG_FILTER: info
      # Read by schedule plugins for cross-node write-back (see _writeback.py)
      NT_INGEST_URLS: http://nt-ingest-1:8181,http://nt-ingest-2:8181
      NT_DB: nt
      NT_TOKEN_FILE: /var/lib/influxdb3/.nt-operator-token
    command: >
      serve
        --node-id nt-process
        --cluster-id nt
        --mode process,query
        --object-store file
        --data-dir /var/lib/influxdb3
        --plugin-dir /plugins
        --virtual-env-location /var/lib/influxdb3/plugin-venv-process
        --admin-token-file /var/lib/influxdb3/.nt-operator-token
    volumes:
      - influxdb-data:/var/lib/influxdb3
      - ./plugins:/plugins:ro
    healthcheck:
      test: ["CMD-SHELL", "curl -s --max-time 2 -o /dev/null http://127.0.0.1:8181/health && test -s /var/lib/influxdb3/.nt-operator-token"]
      interval: 5s
      timeout: 3s
      retries: 120
      start_period: 5s

  # ── Init: one-shot, talks to query node, creates DB/tables/caches/triggers ──

  influxdb3-init:
    image: influxdb:3-enterprise
    container_name: nt-influxdb3-init
    depends_on:
      influxdb3-ingest-1:
        condition: service_healthy
      influxdb3-ingest-2:
        condition: service_healthy
      influxdb3-query:
        condition: service_healthy
      influxdb3-compact:
        condition: service_healthy
      influxdb3-process:
        condition: service_healthy
    environment:
      INFLUX_HOST: http://nt-query:8181
      INFLUX_DB: nt
      INFLUXDB3_UNSET_VARS: LOG_FILTER
      INFLUXDB3_LOG_FILTER: info
    volumes:
      - influxdb-data:/var/lib/influxdb3
      - ./influxdb/init.sh:/usr/local/bin/nt-init.sh:ro
      - ./plugins:/plugins:ro
    entrypoint: ["/usr/local/bin/nt-init.sh"]
    restart: "no"

  # ── Workload: simulator + UI + scenarios ──

  simulator:
    build:
      context: .
      dockerfile: simulator/Dockerfile
    container_name: nt-simulator
    depends_on:
      influxdb3-ingest-1:
        condition: service_healthy
      influxdb3-ingest-2:
        condition: service_healthy
      influxdb3-init:
        condition: service_completed_successfully
    environment:
      SIM_INGEST_URLS: http://nt-ingest-1:8181,http://nt-ingest-2:8181
      INFLUX_DB: nt
      INFLUX_TOKEN_FILE: /tokens/.nt-operator-token
      SIM_RATE_HZ: ${SIM_RATE_HZ:-1.0}
      SIM_SEED: ${SIM_SEED:-42}
      SIM_SPINES: ${SIM_SPINES:-8}
      SIM_LEAVES: ${SIM_LEAVES:-16}
      SIM_SERVERS_PER_LEAF: ${SIM_SERVERS_PER_LEAF:-48}
      SIM_FLOW_RECORDS_PER_SEC: ${SIM_FLOW_RECORDS_PER_SEC:-5000}
    command: ["python", "-m", "simulator.main"]
    volumes:
      - influxdb-data:/tokens:ro

  ui:
    build:
      context: .
      dockerfile: ui/Dockerfile
    container_name: nt-ui
    depends_on:
      influxdb3-query:
        condition: service_healthy
      influxdb3-init:
        condition: service_completed_successfully
    environment:
      INFLUX_URL: http://nt-query:8181
      INFLUX_PUBLIC_URL: ${INFLUX_PUBLIC_URL:-http://localhost:8181}
      INFLUX_DB: nt
      INFLUX_TOKEN_FILE: /tokens/.nt-operator-token
      UI_KPI_POLL_MS: ${UI_KPI_POLL_MS:-5000}
      UI_THROUGHPUT_POLL_MS: ${UI_THROUGHPUT_POLL_MS:-10000}
      UI_ANOMALIES_POLL_MS: ${UI_ANOMALIES_POLL_MS:-5000}
      UI_TOP_TALKERS_POLL_MS: ${UI_TOP_TALKERS_POLL_MS:-5000}
    command: ["uvicorn", "ui.app:app", "--host", "0.0.0.0", "--port", "8080"]
    ports:
      - "8080:8080"
    volumes:
      - influxdb-data:/tokens:ro

  scenarios:
    build:
      context: .
      dockerfile: simulator/Dockerfile
    container_name: nt-scenarios
    profiles: ["scenarios"]
    depends_on:
      influxdb3-ingest-1:
        condition: service_healthy
      influxdb3-init:
        condition: service_completed_successfully
    environment:
      SIM_INGEST_URLS: http://nt-ingest-1:8181,http://nt-ingest-2:8181
      INFLUX_DB: nt
      INFLUX_TOKEN_FILE: /tokens/.nt-operator-token
      SCENARIO: ${SCENARIO:-}
    volumes:
      - influxdb-data:/tokens:ro
    entrypoint: ["sh", "-c", "test -n \"$$SCENARIO\" || { echo 'SCENARIO env var not set' >&2; exit 2; }; exec python -m simulator.scenarios.$$SCENARIO"]
    restart: "no"

volumes:
  influxdb-data:
```

- [ ] **Step 2: Validate compose**

```bash
cd /Users/pauldix/codez/reference_architectures/influxdb3-ref-network-telemetry
INFLUXDB3_ENTERPRISE_EMAIL=test@example.com docker compose config > /dev/null
```

Expected: no output, exit 0.

- [ ] **Step 3: Commit**

```bash
git add docker-compose.yml
git commit -m "feat: 5-node multi-node compose stack (2 ingest + query + compact + process)"
```

### Task 8: `scripts/setup.sh` and `Makefile`

**Files:**
- Create: `scripts/setup.sh`
- Create: `Makefile`

The Makefile mostly mirrors iiot, with two differences: `make cli` shells into `nt-query` (the natural ad-hoc query node), and the data volume name in `make clean` is `influxdb3-ref-network-telemetry_influxdb-data` (compose's default for the named volume).

- [ ] **Step 1: Create `scripts/setup.sh`** (identical to iiot's, with `nt-` paths)

```bash
#!/usr/bin/env bash
set -euo pipefail
cd "$(dirname "$0")/.."

if [[ ! -f .env ]]; then
    cp .env.example .env
    echo "[setup] created .env from .env.example"
fi

if grep -q '^INFLUXDB3_ENTERPRISE_EMAIL=.\+' .env; then
    EMAIL=$(grep '^INFLUXDB3_ENTERPRISE_EMAIL=' .env | cut -d= -f2-)
    echo "[setup] using existing INFLUXDB3_ENTERPRISE_EMAIL=${EMAIL}"
    exit 0
fi

echo
echo "InfluxDB 3 Enterprise requires email-based license validation."
echo "On first startup, a validation email is sent to the address below."
echo "You must click the validation link before the cluster finishes starting."
echo "(Validation happens once for the whole 5-node cluster.)"
echo
read -rp "Enter email for license validation: " EMAIL

if [[ -z "${EMAIL}" ]]; then
    echo "[setup] empty email; aborting" >&2
    exit 1
fi

if grep -q '^INFLUXDB3_ENTERPRISE_EMAIL=' .env; then
    python3 - "$EMAIL" <<'PY'
import pathlib, re, sys
email = sys.argv[1]
p = pathlib.Path(".env")
p.write_text(re.sub(r'^INFLUXDB3_ENTERPRISE_EMAIL=.*$',
                    f'INFLUXDB3_ENTERPRISE_EMAIL={email}',
                    p.read_text(), flags=re.M))
PY
else
    echo "INFLUXDB3_ENTERPRISE_EMAIL=${EMAIL}" >> .env
fi

echo "[setup] wrote INFLUXDB3_ENTERPRISE_EMAIL=${EMAIL} to .env"
```

```bash
chmod +x /Users/pauldix/codez/reference_architectures/influxdb3-ref-network-telemetry/scripts/setup.sh
```

- [ ] **Step 2: Create `Makefile`**

```makefile
SHELL := /usr/bin/env bash
.DEFAULT_GOAL := help
COMPOSE := docker compose

.PHONY: help up down clean demo demo-fresh cli cli-example query logs ps \
        scenario scenario-list test test-unit test-scenarios test-smoke \
        lint format

help: ## Show targets
	@awk 'BEGIN{FS=":.*##"} /^[a-zA-Z0-9_-]+:.*##/ {printf "  \033[1;36m%-20s\033[0m %s\n",$$1,$$2}' $(MAKEFILE_LIST)

demo: ## End-to-end scripted demo: cluster up → browser → scenarios → results
	@./scripts/demo.sh

demo-fresh: ## Same as demo, but wipes state first (forces license re-validation)
	@./scripts/demo.sh --fresh

up: ## Prompt for email (if needed), write .env, then bring the cluster up
	@./scripts/setup.sh
	@echo
	@echo "============================================================="
	@echo "  InfluxDB 3 Enterprise cluster (5 nodes) is starting."
	@echo "  Check the email you provided and CLICK THE VALIDATION LINK."
	@echo "  The simulator and UI will start automatically once validation"
	@echo "  completes. UI: http://localhost:8080   API: http://localhost:8181"
	@echo "============================================================="
	@$(COMPOSE) up -d

down: ## Stop services (preserves data volume)
	@$(COMPOSE) down

clean: ## Stop services and drop the data volume (requires re-validation next time)
	@$(COMPOSE) down -v

logs: ## Tail all service logs
	@$(COMPOSE) logs -f

ps: ## Show service status
	@$(COMPOSE) ps

cli: ## Shell into the QUERY node; TOKEN exported, `iql <sql>` runs queries
	@$(COMPOSE) exec influxdb3-query bash -c '\
	  export TOKEN=$$(cat /var/lib/influxdb3/.nt-token-plain); \
	  iql() { influxdb3 query --database nt --token "$$TOKEN" "$$1"; }; \
	  export -f iql; \
	  echo ""; \
	  echo "  TOKEN is exported. Try:"; \
	  echo "    iql \"SELECT COUNT(*) FROM interface_counters\""; \
	  echo "    iql \"SELECT * FROM bgp_sessions LIMIT 5\""; \
	  echo ""; \
	  exec bash'

query: ## One-shot query against the query node. Usage: make query sql='SELECT 1'
	@test -n "$(sql)" || (echo "usage: make query sql='<SQL>'"; exit 1)
	@$(COMPOSE) exec -T -e "SQL=$(sql)" influxdb3-query bash -c 'TOKEN=$$(cat /var/lib/influxdb3/.nt-token-plain); influxdb3 query --database nt --token "$$TOKEN" "$$SQL"'

cli-example: ## Run a named CLI example. Usage: make cli-example name=list-databases
	@test -n "$(name)" || (echo "usage: make cli-example name=<example>"; exit 1)
	@grep -A 20 "^## $(name)" CLI_EXAMPLES.md | sed -n '/^```bash/,/^```/p' | sed '1d;$$d' \
	  | while read -r line; do echo "+ $$line"; $(COMPOSE) exec -T influxdb3-query bash -lc "export TOKEN=\$$(cat /var/lib/influxdb3/.nt-token-plain); $$line"; done

scenario: ## Run a scenario. Usage: make scenario name=congestion_hotspot
	@test -n "$(name)" || (echo "usage: make scenario name=<scenario>"; exit 1)
	@SCENARIO=$(name) $(COMPOSE) --profile scenarios run --rm scenarios

scenario-list: ## List available scenarios
	@ls simulator/scenarios/*.py 2>/dev/null | grep -v "_base\|__init__" | xargs -I{} basename {} .py | while read n; do \
	  desc=$$(grep -m1 '^"""' simulator/scenarios/$$n.py | sed 's/"""//g'); \
	  printf "  %-32s %s\n" "$$n" "$$desc"; done

test: test-unit test-scenarios ## Run unit + scenario tests (skip smoke)

test-unit: ## Plugin + signal + query unit tests (no docker)
	@pytest tests -q -m "not scenario and not smoke"

test-scenarios: ## Scenario integration tests (uses testcontainers; multi-node!)
	@pytest tests/test_scenarios -q -m scenario

test-smoke: ## End-to-end smoke against the real 5-node compose (slow)
	@pytest tests/test_smoke.py -q -m smoke

lint: ## Check formatting and lint
	@ruff check .
	@ruff format --check .

format: ## Auto-fix formatting
	@ruff check --fix .
	@ruff format .
```

- [ ] **Step 3: Verify `make help`**

```bash
cd /Users/pauldix/codez/reference_architectures/influxdb3-ref-network-telemetry
make help
```

Expected: colorized list of targets.

- [ ] **Step 4: Commit**

```bash
git add scripts/setup.sh Makefile
git commit -m "feat: setup script and Makefile (5-node aware)"
```

---

## Phase 4 — Integration checkpoint #1 (executed, not manual)

### Task 9: First end-to-end run — cluster boots, simulator writes, retention applied

This is the FIRST chance to verify Phase 0 assumptions held under real conditions. **The implementer actually runs the cluster** (no manual click needed — `paul+refiiot@influxdata.com` auto-validates) and asserts on the outputs. Failures here amend the relevant Phase 3 task.

**All `docker compose` / `make up` commands in this phase need `dangerouslyDisableSandbox: true` on the Bash tool call.** The default sandbox kills backgrounded children.

- [ ] **Step 1: Pre-clean and write `.env`**

```bash
cd /Users/pauldix/codez/reference_architectures/influxdb3-ref-network-telemetry
docker ps -a --format '{{.Names}}' | grep '^nt-' | xargs -r docker rm -f 2>/dev/null
# Wipe any prior cluster state so we start clean (license still picks up)
docker compose down -v 2>/dev/null || true
# Write .env directly (skip the interactive setup.sh prompt)
cat > .env <<'EOF'
INFLUXDB3_ENTERPRISE_EMAIL=paul+refiiot@influxdata.com
INFLUXDB3_ENTERPRISE_LICENSE_TYPE=trial
EOF
```

- [ ] **Step 2: Bring up the cluster** (sandbox-disabled bash)

```bash
docker compose up -d 2>&1 | tail -10
```

- [ ] **Step 3: Wait for all 5 nodes healthy** (up to 5 min — license auto-pickup is fast but cluster init is slower than a single node)

```bash
deadline=$(($(date +%s) + 300))
for n in nt-ingest-1 nt-ingest-2 nt-query nt-compact nt-process; do
    while [ $(date +%s) -lt $deadline ]; do
        if docker compose ps --format json $n 2>/dev/null | grep -q '"Health":"healthy"'; then
            echo "$n healthy"
            break
        fi
        sleep 3
    done
done
docker compose ps
# Expected: nt-ingest-1, nt-ingest-2, nt-query, nt-compact, nt-process all "healthy"
```

If any node fails to become healthy, dump its logs (`docker compose logs <node> | tail -50`) and diagnose. Most common cause: license validation failing because the email isn't recognized — verify `paul+refiiot@influxdata.com` is set in `.env` and visible to compose (`docker compose config | grep INFLUXDB3_ENTERPRISE_EMAIL`).

- [ ] **Step 4: Assert init completed cleanly**

```bash
docker compose logs influxdb3-init 2>&1 | tail -20
# Required lines (some may say "already exists"):
#   [init] admin token present
#   [init] created database nt
#   [init] created table interface_counters
#   [init] created table bgp_sessions
#   [init] created table flow_records
#   [init] created table latency_probes
#   [init] created table fabric_health
#   [init] created table anomalies
#   [init] created last_cache bgp_session_last
#   [init] created distinct_cache src_ip_distinct
#   [init] created distinct_cache dst_ip_distinct
#   [init] initialization complete
```

If "initialization complete" doesn't appear, init.sh failed mid-way. Find the FATAL line, amend the relevant `idempotent` call (likely a configure-API field name or CLI flag mismatch), and re-run from Step 1.

- [ ] **Step 5: Assert retention is set on `fabric_health`**

```bash
docker compose exec -T influxdb3-query bash -lc \
  'TOKEN=$(cat /var/lib/influxdb3/.nt-token-plain); influxdb3 show table fabric_health --database nt --token "$TOKEN"' 2>&1
# Expected: output contains "24h" or "86400" (seconds) for retention.
```

If retention isn't showing, the `--retention-period` flag or its configure-API equivalent isn't being applied — fix init.sh's `ensure_tables` for fabric_health.

- [ ] **Step 6: Assert simulator is writing to BOTH ingest nodes**

Wait 30s for the simulator to ramp up:

```bash
sleep 30
docker compose logs simulator 2>&1 | tail -10
# Expected: lines like "tick=N t=Ns flows/tick=5000"

# Token for SQL queries
TOKEN=$(docker compose exec -T influxdb3-query cat /var/lib/influxdb3/.nt-token-plain | tr -d '\r\n')

_q() {
  curl -s -X POST -H "Authorization: Bearer $TOKEN" \
       -H "Content-Type: application/json" \
       http://localhost:8181/api/v3/query_sql \
       -d "{\"db\":\"nt\",\"q\":\"$1\",\"format\":\"json\"}"
}

_q "SELECT COUNT(*) AS n FROM interface_counters WHERE time > now() - INTERVAL '1 minute'"
# Expected: n > 30000 (~1024 interfaces × ~60 ticks/min)

_q "SELECT COUNT(*) AS n FROM flow_records WHERE time > now() - INTERVAL '1 minute'"
# Expected: n > 100000 (~5k/sec × 60s)

_q "SELECT COUNT(*) AS n FROM bgp_sessions WHERE time > now() - INTERVAL '1 minute'"
# Expected: n > 5000 (128 sessions × ~60 ticks/min)
```

Verify BOTH ingest nodes accepted writes (round-robin proof):

```bash
docker compose logs influxdb3-ingest-1 2>&1 | grep -c "write_lp\|wrote\|points received" || echo 0
docker compose logs influxdb3-ingest-2 2>&1 | grep -c "write_lp\|wrote\|points received" || echo 0
# Both counts should be > 0. If one is 0, the round-robin isn't working —
# check SIM_INGEST_URLS env var on the simulator container.
```

- [ ] **Step 7: Assert LVC + DVC are populated**

```bash
_q "SELECT COUNT(*) AS n FROM last_cache('bgp_sessions', 'bgp_session_last')"
# Expected: n = 128 (one per BGP session)

_q "SELECT COUNT(*) AS n FROM distinct_cache('flow_records', 'src_ip_distinct')"
# Expected: n > 100 (simulator generates ~1600 distinct IPs across 16 leaves)
```

- [ ] **Step 8: Verify license persists across down/up**

```bash
docker compose down  # preserves volume
sleep 5
docker compose up -d 2>&1 | tail -5
sleep 60  # let cluster come back up
docker compose ps --format 'table {{.Name}}\t{{.Status}}' | grep nt-
# Expected: all 5 nodes healthy without re-validation; volume preserved license.
```

- [ ] **Step 9: Outcome**

If every step passed, this checkpoint is green — **leave the cluster running** so Phase 5 (scenarios) and Phase 7 can use it without restart.

If any step failed, fix the underlying issue (likely a Phase 3 task — init.sh, compose, or env var), commit the fix, then re-run this checkpoint from Step 1.

---

## Phase 5 — Scenarios

Two scenarios. Both write through the round-robin writer (so they exercise both ingest nodes by default).

### Task 10: Scenario framework

**Files:**
- Create: `simulator/scenarios/__init__.py`
- Create: `simulator/scenarios/_base.py`

- [ ] **Step 1: Create the scenarios package**

```bash
mkdir -p /Users/pauldix/codez/reference_architectures/influxdb3-ref-network-telemetry/simulator/scenarios
touch /Users/pauldix/codez/reference_architectures/influxdb3-ref-network-telemetry/simulator/scenarios/__init__.py
```

- [ ] **Step 2: Create `simulator/scenarios/_base.py`**

```python
"""Shared helpers for scenario scripts.

Scenarios open the same multi-ingest writer the simulator uses, then
synthesize line protocol directly. They run in parallel with the
running simulator — both write streams land in InfluxDB; the plugins
react to whichever rows match their detection patterns.
"""

from __future__ import annotations

import sys
import time

from simulator.config import load
from simulator.writer import InfluxDB3Writer


def open_writer() -> InfluxDB3Writer:
    cfg = load()
    return InfluxDB3Writer(
        urls=cfg.ingest_urls,
        database=cfg.database,
        token=cfg.token,
        batch_size=200,
    )


def announce(step_no: int, message: str) -> None:
    print(f"[scenario step {step_no}] {message}", flush=True)


def fail(message: str) -> None:
    print(f"[scenario] FAIL: {message}", file=sys.stderr, flush=True)
    sys.exit(1)


def now_ns() -> int:
    return int(time.time() * 1_000_000_000)


def sleep(seconds: float) -> None:
    time.sleep(seconds)
```

- [ ] **Step 3: Commit**

```bash
git add simulator/scenarios/__init__.py simulator/scenarios/_base.py
git commit -m "feat(scenarios): shared scenario helpers"
```

### Task 11: `congestion_hotspot` scenario

**Files:**
- Create: `simulator/scenarios/congestion_hotspot.py`

- [ ] **Step 1: Implement**

```python
"""leaf-07 / et-0/0/12 climbs to 94% utilization; anomaly detector fires; banner flips DEGRADED.

Writes interface_counters rows for L=leaf-07/I=et-0/0/12 with
artificially high in_bytes and out_bytes deltas, simulating saturation.
After ~30 seconds at >90%, schedule_anomaly_detector should fire a
high_util anomaly. Then the scenario backs off and the anomaly
detector emits a cleared row.

Run via: make scenario name=congestion_hotspot
"""

from __future__ import annotations

import math

from simulator.scenarios._base import announce, now_ns, open_writer, sleep

SITE = "dc-east-1"
SWITCH = "leaf-07"
INTERFACE = "et-0/0/12"
CAPACITY_BPS = 100_000_000_000  # 100 Gbps
TICK_HZ = 1.0
RAMP_S = 10
HOLD_S = 30
RECOVERY_S = 5


def _interface_line(in_bytes: int, out_bytes: int, in_pkts: int, out_pkts: int,
                    ecn_marked: float, pfc_pause: float, t_ns: int) -> str:
    return (
        f"interface_counters,site={SITE},switch={SWITCH},interface={INTERFACE} "
        f"in_bytes={in_bytes}i,out_bytes={out_bytes}i,"
        f"in_pkts={in_pkts}i,out_pkts={out_pkts}i,"
        f"in_errors=0i,in_discards=0i,"
        f"optical_power_dbm=-2.0,"
        f"ecn_marked_pkts={ecn_marked:.1f},"
        f"pfc_pause_frames={pfc_pause:.1f} {t_ns}"
    )


def main() -> None:
    writer = open_writer()
    in_bytes = 0
    out_bytes = 0
    in_pkts = 0
    out_pkts = 0
    ecn_marked = 0.0
    pfc_pause = 0.0
    try:
        announce(1, "baseline: 10s of nominal traffic on leaf-07/et-0/0/12 (~30%)")
        for _ in range(10):
            bytes_per_sec = int(CAPACITY_BPS * 0.30 / 8)
            in_bytes += bytes_per_sec // 2
            out_bytes += bytes_per_sec // 2
            in_pkts += (bytes_per_sec // 2) // 1024
            out_pkts += (bytes_per_sec // 2) // 1024
            writer.write(_interface_line(
                in_bytes, out_bytes, in_pkts, out_pkts, ecn_marked, pfc_pause, now_ns()
            ))
            writer.flush()
            sleep(1)

        announce(2, f"ramp utilization 30% → 94% over {RAMP_S}s")
        for i in range(RAMP_S):
            util = 0.30 + (0.94 - 0.30) * (i + 1) / RAMP_S
            bytes_per_sec = int(CAPACITY_BPS * util / 8)
            pkts_per_sec = bytes_per_sec // 1024
            in_bytes += bytes_per_sec // 2
            out_bytes += bytes_per_sec // 2
            in_pkts += pkts_per_sec // 2
            out_pkts += pkts_per_sec // 2
            if util > 0.70:
                ecn_marked += pkts_per_sec * (util - 0.70) * 0.5
            writer.write(_interface_line(
                in_bytes, out_bytes, in_pkts, out_pkts, ecn_marked, pfc_pause, now_ns()
            ))
            writer.flush()
            sleep(1)

        announce(3, f"hold at 94% for {HOLD_S}s — anomaly detector should fire within 5s once 30s sustained")
        for _ in range(HOLD_S):
            util = 0.94
            bytes_per_sec = int(CAPACITY_BPS * util / 8)
            pkts_per_sec = bytes_per_sec // 1024
            in_bytes += bytes_per_sec // 2
            out_bytes += bytes_per_sec // 2
            in_pkts += pkts_per_sec // 2
            out_pkts += pkts_per_sec // 2
            ecn_marked += pkts_per_sec * (util - 0.70) * 0.5
            pfc_pause += pkts_per_sec * (util - 0.90) * 0.2
            writer.write(_interface_line(
                in_bytes, out_bytes, in_pkts, out_pkts, ecn_marked, pfc_pause, now_ns()
            ))
            writer.flush()
            sleep(1)

        announce(4, f"recovery: drop back to baseline; anomaly should clear within 5s")
        for _ in range(RECOVERY_S):
            util = 0.30
            bytes_per_sec = int(CAPACITY_BPS * util / 8)
            in_bytes += bytes_per_sec // 2
            out_bytes += bytes_per_sec // 2
            in_pkts += (bytes_per_sec // 2) // 1024
            out_pkts += (bytes_per_sec // 2) // 1024
            writer.write(_interface_line(
                in_bytes, out_bytes, in_pkts, out_pkts, ecn_marked, pfc_pause, now_ns()
            ))
            writer.flush()
            sleep(1)

        announce(5, "DONE — verify: SELECT * FROM anomalies WHERE switch='leaf-07' AND interface='et-0/0/12' ORDER BY time DESC LIMIT 5")
    finally:
        writer.flush()


if __name__ == "__main__":
    main()
```

- [ ] **Step 2: Smoke-test imports**

```bash
python -c "from simulator.scenarios import congestion_hotspot; print('ok')"
```

- [ ] **Step 3: Commit**

```bash
git add simulator/scenarios/congestion_hotspot.py
git commit -m "feat(scenarios): congestion_hotspot"
```

### Task 12: `east_west_burst` scenario

**Files:**
- Create: `simulator/scenarios/east_west_burst.py`

- [ ] **Step 1: Implement**

```python
"""10x traffic burst from 10.4.7.91; typeahead finds it sub-ms; detail panel shows the spike.

Writes ~10x extra flow_records from src_ip=10.4.7.91 over 30 seconds.
No fabric-level anomaly fires (single-source bursts don't necessarily
breach thresholds); the scenario exists to populate the DVC for the
typeahead and give the detail panel something dramatic to render.

Run via: make scenario name=east_west_burst
"""

from __future__ import annotations

import random

from simulator.scenarios._base import announce, now_ns, open_writer, sleep

SITE = "dc-east-1"
HOT_SRC_IP = "10.4.7.91"
HOT_SRC_LEAF = "leaf-05"  # 10.4.x.x sourced from leaf-05
DURATION_S = 30
EXTRA_FLOWS_PER_SEC = 200  # on top of the simulator's nominal ~5000/s

# A small set of destination IPs across other leaves for variety.
DESTINATIONS = [
    ("10.0.1.5",  "leaf-01"),
    ("10.1.4.13", "leaf-02"),
    ("10.7.2.91", "leaf-08"),
    ("10.12.0.42","leaf-13"),
    ("10.3.9.7",  "leaf-04"),
]


def _flow_line(dst_ip: str, dst_switch: str, bytes_: int, packets: int,
               duration_ms: float, t_ns: int) -> str:
    return (
        f"flow_records,site={SITE},src_ip={HOT_SRC_IP},dst_ip={dst_ip},"
        f"src_switch={HOT_SRC_LEAF},dst_switch={dst_switch},vrf=default "
        f"bytes={bytes_}i,packets={packets}i,duration_ms={duration_ms:.3f} {t_ns}"
    )


def main() -> None:
    writer = open_writer()
    rng = random.Random(2026)
    try:
        announce(1, f"baseline: 10s with no extra traffic from {HOT_SRC_IP}")
        sleep(10)

        announce(2, f"injecting {EXTRA_FLOWS_PER_SEC}/s from {HOT_SRC_IP} for {DURATION_S}s")
        for sec in range(DURATION_S):
            for _ in range(EXTRA_FLOWS_PER_SEC):
                dst_ip, dst_switch = rng.choice(DESTINATIONS)
                bytes_ = rng.randint(1024, 1500) * rng.randint(50, 200)  # large bursts
                packets = max(1, bytes_ // 1024)
                duration_ms = rng.uniform(1.0, 30.0)
                writer.write(_flow_line(
                    dst_ip, dst_switch, bytes_, packets, duration_ms, now_ns()
                ))
            writer.flush()
            if sec % 5 == 0:
                announce(3, f"  t={sec}s — {EXTRA_FLOWS_PER_SEC * (sec + 1)} extra flows from {HOT_SRC_IP} so far")
            sleep(1)

        announce(4, f"DONE — verify: typeahead '10.4' returns {HOT_SRC_IP}; detail panel shows the spike")
    finally:
        writer.flush()


if __name__ == "__main__":
    main()
```

- [ ] **Step 2: Smoke-test imports + commit**

```bash
python -c "from simulator.scenarios import east_west_burst; print('ok')"
git add simulator/scenarios/east_west_burst.py
git commit -m "feat(scenarios): east_west_burst"
```

---

## Phase 6 — Processing Engine plugins

**Read Phase 0.2 again before writing any plugin.** `LineBuilder` is injected; cron strings are 6-field but this repo prefers `every:`; plugin filename → trigger type.

**This phase introduces the cross-node write-back pattern** for schedule plugins: rather than `influxdb3_local.write(LineBuilder)`, schedule plugins use httpx to POST line protocol back through an ingest node. The shared `_writeback.py` module factors out the round-robin + token-loading + retry-once-on-failure logic.

### Task 13: `plugins/_writeback.py` — shared httpx writer for schedule plugins

**Files:**
- Create: `plugins/_writeback.py`
- Create: `tests/test_plugins/__init__.py`
- Create: `tests/test_plugins/test_writeback.py`

The plugin reads `NT_INGEST_URLS`, `NT_DB`, and `NT_TOKEN_FILE` env vars (set on the process node by compose) at module-load time. The token is read from the JSON file in the shared volume.

- [ ] **Step 1: Write tests**

```bash
mkdir -p /Users/pauldix/codez/reference_architectures/influxdb3-ref-network-telemetry/tests/test_plugins
touch /Users/pauldix/codez/reference_architectures/influxdb3-ref-network-telemetry/tests/test_plugins/__init__.py
```

Create `tests/test_plugins/test_writeback.py`:

```python
"""Tests for the shared cross-node write-back client used by schedule plugins."""

from __future__ import annotations

import importlib.util
import json
import sys
from pathlib import Path

import pytest


def _load_writeback():
    p = Path(__file__).resolve().parents[2] / "plugins" / "_writeback.py"
    spec = importlib.util.spec_from_file_location("_writeback", p)
    mod = importlib.util.module_from_spec(spec)
    sys.modules["_writeback"] = mod
    spec.loader.exec_module(mod)
    return mod


class _RecordingPoster:
    def __init__(self, fail_urls: set[str] | None = None) -> None:
        self.calls: list[tuple[str, str]] = []
        self._fail = fail_urls or set()

    def __call__(self, url: str, content: str, headers: dict[str, str]) -> None:
        self.calls.append((url, content))
        if url in self._fail:
            raise ConnectionError(f"refuse {url}")


def _setup(monkeypatch, tmp_path, fail_urls=None):
    token_path = tmp_path / "token.json"
    token_path.write_text(json.dumps({"token": "test-token"}))
    monkeypatch.setenv("NT_INGEST_URLS", "http://ingest-1:8181,http://ingest-2:8181")
    monkeypatch.setenv("NT_DB", "nt")
    monkeypatch.setenv("NT_TOKEN_FILE", str(token_path))
    mod = _load_writeback()
    poster = _RecordingPoster(fail_urls)
    monkeypatch.setattr(mod, "_post", poster, raising=False)
    return mod, poster


def test_write_lines_round_robins_across_ingest_urls(monkeypatch, tmp_path):
    mod, poster = _setup(monkeypatch, tmp_path)
    mod.write_lines(["m1,t=1 v=1 1"])
    mod.write_lines(["m1,t=1 v=2 2"])
    mod.write_lines(["m1,t=1 v=3 3"])
    urls = [c[0] for c in poster.calls]
    assert "ingest-1" in urls[0]
    assert "ingest-2" in urls[1]
    assert "ingest-1" in urls[2]


def test_write_lines_fallback_on_connection_error(monkeypatch, tmp_path):
    mod, poster = _setup(monkeypatch, tmp_path, fail_urls={
        "http://ingest-1:8181/api/v3/write_lp?db=nt&precision=nanosecond",
    })
    mod.write_lines(["m1,t=1 v=1 1"])
    # First call to ingest-1 fails → fallback to ingest-2
    assert len(poster.calls) == 2
    assert "ingest-1" in poster.calls[0][0]
    assert "ingest-2" in poster.calls[1][0]


def test_write_lines_raises_when_all_targets_fail(monkeypatch, tmp_path):
    mod, poster = _setup(monkeypatch, tmp_path, fail_urls={
        "http://ingest-1:8181/api/v3/write_lp?db=nt&precision=nanosecond",
        "http://ingest-2:8181/api/v3/write_lp?db=nt&precision=nanosecond",
    })
    with pytest.raises(ConnectionError):
        mod.write_lines(["m1,t=1 v=1 1"])


def test_write_lines_authorization_header_included(monkeypatch, tmp_path):
    mod, poster = _setup(monkeypatch, tmp_path)
    captured = {}

    def _capture_post(url, content, headers):
        captured["headers"] = headers

    monkeypatch.setattr(mod, "_post", _capture_post, raising=False)
    mod.write_lines(["m1,t=1 v=1 1"])
    assert captured["headers"]["Authorization"] == "Bearer test-token"
    assert captured["headers"]["Content-Type"].startswith("text/plain")
```

- [ ] **Step 2: Run tests — expect FAIL**

```bash
pytest tests/test_plugins/test_writeback.py -v
```

- [ ] **Step 3: Implement `plugins/_writeback.py`**

```python
"""Shared cross-node write-back client used by schedule plugins.

Plugin runtime on a process-only (or process,query) node has no local
ingest path: `LineBuilder` + `influxdb3_local.write()` are not the
right tool when the node doesn't accept writes locally. Instead,
schedule plugins import `write_lines()` from this module and POST line
protocol back through an ingest node's HTTP API.

Configuration is read once from environment variables at module load:
- NT_INGEST_URLS — comma-separated ingest base URLs (e.g.,
  "http://nt-ingest-1:8181,http://nt-ingest-2:8181").
- NT_DB — target database (default "nt").
- NT_TOKEN_FILE — path to the JSON token file on the shared volume
  (default "/var/lib/influxdb3/.nt-operator-token").

Round-robin over the URLs by default; on connection error, try the
next URL once. If every URL fails, raise the last error to the caller
(the engine will log it).
"""

from __future__ import annotations

import json
import os
from typing import Iterable

import httpx

_INGEST_URLS = [
    u.strip() for u in os.environ.get("NT_INGEST_URLS", "").split(",") if u.strip()
] or ["http://nt-ingest-1:8181"]
_DB = os.environ.get("NT_DB", "nt")
_TOKEN_FILE = os.environ.get("NT_TOKEN_FILE", "/var/lib/influxdb3/.nt-operator-token")


def _read_token() -> str:
    with open(_TOKEN_FILE) as f:
        return json.load(f)["token"]


_TOKEN = _read_token() if os.path.exists(_TOKEN_FILE) else ""

_ENDPOINTS = [
    f"{u.rstrip('/')}/api/v3/write_lp?db={_DB}&precision=nanosecond"
    for u in _INGEST_URLS
]

_next_idx = 0
_client = httpx.Client(timeout=10.0)


def _post(url: str, content: str, headers: dict[str, str]) -> None:
    """Indirection point for tests to monkeypatch."""
    r = _client.post(url, content=content, headers=headers)
    r.raise_for_status()


def write_lines(lines: Iterable[str]) -> None:
    """POST a batch of line-protocol lines through the next ingest URL,
    falling back to the others on connection error."""
    global _next_idx, _TOKEN
    if not _TOKEN:
        # Token file may have appeared after module import.
        _TOKEN = _read_token()
    payload = "\n".join(lines)
    if not payload:
        return
    headers = {
        "Authorization": f"Bearer {_TOKEN}",
        "Content-Type": "text/plain; charset=utf-8",
    }
    n = len(_ENDPOINTS)
    primary = _next_idx
    last_err: Exception | None = None
    for offset in range(n):
        idx = (primary + offset) % n
        url = _ENDPOINTS[idx]
        try:
            _post(url, payload, headers)
            _next_idx = (idx + 1) % n
            return
        except Exception as e:  # noqa: BLE001
            last_err = e
            continue
    assert last_err is not None
    raise last_err
```

- [ ] **Step 4: Run tests — expect PASS**

```bash
pytest tests/test_plugins/test_writeback.py -v
```

Expected: 4 passed.

- [ ] **Step 5: Commit**

```bash
git add plugins/_writeback.py tests/test_plugins/__init__.py tests/test_plugins/test_writeback.py
git commit -m "feat(plugins): _writeback.py — cross-node httpx round-robin writer for schedule plugins"
```

### Task 14: `schedule_fabric_health` plugin

**Files:**
- Create: `plugins/schedule_fabric_health.py`
- Create: `tests/test_plugins/test_schedule_fabric_health.py`

- [ ] **Step 1: Write tests**

Create `tests/test_plugins/test_schedule_fabric_health.py`:

```python
"""Unit-test schedule_fabric_health with canned query responses + recording write."""

from __future__ import annotations

import importlib.util
import sys
from datetime import datetime, timezone
from pathlib import Path


def _load_plugin(monkeypatch):
    """Load the plugin and stub out _writeback BEFORE plugin import-time
    code runs (the plugin imports _writeback at module load)."""
    # Provide a fake _writeback module first.
    fake_wb = type(sys)("_writeback")
    fake_wb._writes: list[str] = []
    def _write_lines(lines):
        fake_wb._writes.extend(list(lines))
    fake_wb.write_lines = _write_lines
    sys.modules["_writeback"] = fake_wb
    p = Path(__file__).resolve().parents[2] / "plugins" / "schedule_fabric_health.py"
    spec = importlib.util.spec_from_file_location("schedule_fabric_health", p)
    mod = importlib.util.module_from_spec(spec)
    sys.modules["schedule_fabric_health"] = mod
    spec.loader.exec_module(mod)
    return mod, fake_wb


class CannedInflux:
    def __init__(self, responses):
        self.responses = responses
        self.queries = []

    def query(self, sql):
        self.queries.append(sql)
        for k, v in self.responses.items():
            fragments = k if isinstance(k, tuple) else (k,)
            if all(f in sql for f in fragments):
                return v
        return []

    def info(self, msg):
        pass


def test_writes_one_health_row_when_all_healthy(monkeypatch):
    mod, fake_wb = _load_plugin(monkeypatch)
    fake = CannedInflux({
        "FROM last_cache": [
            {"switch": "leaf-01", "peer_switch": "spine-1", "state": "established"}
        ] * 128,  # 128 sessions, all established
        ("FROM interface_counters", "spine_in_bps"): [
            {"layer": "spine", "ingress_bps": 50e9, "egress_bps": 50e9}
        ],
        ("FROM interface_counters", "leaf_uplink_in_bps"): [
            {"layer": "leaf_uplink", "ingress_bps": 30e9, "egress_bps": 30e9}
        ],
        ("FROM interface_counters", "leaf_downlink_in_bps"): [
            {"layer": "leaf_downlink", "ingress_bps": 20e9, "egress_bps": 20e9}
        ],
        ("FROM interface_counters", "ecn_pct"): [{"ecn_pct": 0.001}],
        ("FROM latency_probes", "p95"): [{"p95_latency_us": 25.0}],
        "FROM anomalies": [],
    })
    mod.process_scheduled_call(fake, datetime.now(timezone.utc))
    # One row per layer (3) + plant-level (1) = 4 rows.
    assert len(fake_wb._writes) >= 1
    # All status="healthy" since no anomalies + all BGP up
    assert all('status="healthy"' in lp for lp in fake_wb._writes)


def test_status_is_degraded_when_active_anomaly(monkeypatch):
    mod, fake_wb = _load_plugin(monkeypatch)
    fake_wb._writes.clear()
    fake = CannedInflux({
        "FROM last_cache": [
            {"switch": "leaf-01", "peer_switch": "spine-1", "state": "established"}
        ] * 128,
        ("FROM interface_counters", "spine"): [
            {"layer": "spine", "ingress_bps": 50e9, "egress_bps": 50e9}
        ],
        ("FROM interface_counters", "leaf_uplink"): [
            {"layer": "leaf_uplink", "ingress_bps": 30e9, "egress_bps": 30e9}
        ],
        ("FROM interface_counters", "leaf_downlink"): [
            {"layer": "leaf_downlink", "ingress_bps": 20e9, "egress_bps": 20e9}
        ],
        ("FROM interface_counters", "ecn_pct"): [{"ecn_pct": 0.001}],
        ("FROM latency_probes", "p95"): [{"p95_latency_us": 25.0}],
        "FROM anomalies": [
            {"kind": "high_util", "severity": "critical"}
        ],
    })
    mod.process_scheduled_call(fake, datetime.now(timezone.utc))
    assert any('status="degraded"' in lp for lp in fake_wb._writes)


def test_status_is_down_when_bgp_majority_down(monkeypatch):
    mod, fake_wb = _load_plugin(monkeypatch)
    fake_wb._writes.clear()
    # Only 50 sessions established (out of expected 128) → < 50% → DOWN
    fake = CannedInflux({
        "FROM last_cache": [
            {"switch": f"leaf-{i:02d}", "peer_switch": "spine-1",
             "state": "established" if i < 50 else "active"}
            for i in range(128)
        ],
        ("FROM interface_counters", "spine"): [
            {"layer": "spine", "ingress_bps": 0, "egress_bps": 0}
        ],
        ("FROM interface_counters", "leaf_uplink"): [
            {"layer": "leaf_uplink", "ingress_bps": 0, "egress_bps": 0}
        ],
        ("FROM interface_counters", "leaf_downlink"): [
            {"layer": "leaf_downlink", "ingress_bps": 0, "egress_bps": 0}
        ],
        ("FROM interface_counters", "ecn_pct"): [{"ecn_pct": 0.0}],
        ("FROM latency_probes", "p95"): [{"p95_latency_us": 0.0}],
        "FROM anomalies": [],
    })
    mod.process_scheduled_call(fake, datetime.now(timezone.utc))
    assert any('status="down"' in lp for lp in fake_wb._writes)
```

- [ ] **Step 2: Run tests — expect FAIL**

```bash
pytest tests/test_plugins/test_schedule_fabric_health.py -v
```

- [ ] **Step 3: Implement `plugins/schedule_fabric_health.py`**

```python
"""Schedule trigger: every 5s, write a fabric_health rollup row.

Binding: every:5s, args={}
Runs: on the process node (mode=process,query).
Side effects: posts line protocol to the ingest tier via _writeback.

Computes:
- BGP up-count: rows in last_cache('bgp_sessions','bgp_session_last') with state='established'
- Per-layer ingress/egress bps (spine, leaf_uplink, leaf_downlink) over last 30s
- ECN-marked % over last 30s
- p95 latency over last 30s
- Active anomalies count (from anomalies in last 60s)

Status: 'healthy' if no anomalies AND >= 95% BGP established; 'degraded' if
anomalies present but BGP majority OK; 'down' if BGP < 50% established or
any spine has all interfaces silent.
"""

from __future__ import annotations

from datetime import datetime

from _writeback import write_lines


def _interface_layer_query(layer_filter: str) -> str:
    return f"""
        SELECT
          '{layer_filter}' AS layer,
          SUM(in_bytes) * 8.0 / 30.0 AS ingress_bps,
          SUM(out_bytes) * 8.0 / 30.0 AS egress_bps
        FROM interface_counters
        WHERE time > now() - INTERVAL '30 seconds'
          AND switch {('LIKE ' + repr('spine-%')) if layer_filter == 'spine' else ('LIKE ' + repr('leaf-%'))}
          AND interface {('LIKE ' + repr('et-0/0/%')) if layer_filter in ('spine','leaf_uplink') else ('LIKE ' + repr('et-0/1/%'))}
    """


def process_scheduled_call(influxdb3_local, call_time, args=None):
    site = "dc-east-1"
    # ── BGP up-count via LVC ──────────────────────────────────────
    bgp_rows = influxdb3_local.query(
        "SELECT switch, peer_switch, state "
        "FROM last_cache('bgp_sessions', 'bgp_session_last') "
    )
    bgp_total = len(bgp_rows)
    bgp_up = sum(1 for r in bgp_rows if str(r.get("state", "")).strip('"') == "established")
    # Spines/leaves "up" derived from sessions: a switch is up if at least one
    # of its sessions is established (rough heuristic for the demo).
    spines_up = len({str(r["peer_switch"]) for r in bgp_rows
                     if str(r.get("state", "")).strip('"') == "established"})
    leaves_up = len({str(r["switch"]) for r in bgp_rows
                     if str(r.get("state", "")).strip('"') == "established"})

    # ── Per-layer bps ─────────────────────────────────────────────
    spine_layer = influxdb3_local.query(_interface_layer_query("spine"))
    leaf_uplink_layer = influxdb3_local.query(_interface_layer_query("leaf_uplink"))
    leaf_downlink_layer = influxdb3_local.query(_interface_layer_query("leaf_downlink"))

    # ── ECN % over 30s ────────────────────────────────────────────
    ecn_rows = influxdb3_local.query("""
        SELECT
          SUM(ecn_marked_pkts) / NULLIF(SUM(in_pkts) + SUM(out_pkts), 0) AS ecn_pct
        FROM interface_counters
        WHERE time > now() - INTERVAL '30 seconds'
    """)
    ecn_pct = float((ecn_rows[0] or {}).get("ecn_pct") or 0.0) if ecn_rows else 0.0

    # ── p95 latency over 30s (approximation: use approx_percentile_cont if available) ─
    p95_rows = influxdb3_local.query("""
        SELECT approx_percentile_cont(rtt_us, 0.95) AS p95_latency_us
        FROM latency_probes
        WHERE time > now() - INTERVAL '30 seconds'
    """)
    p95_us = float((p95_rows[0] or {}).get("p95_latency_us") or 0.0) if p95_rows else 0.0

    # ── Anomaly count ─────────────────────────────────────────────
    anom_rows = influxdb3_local.query("""
        SELECT kind, severity FROM anomalies
        WHERE time > now() - INTERVAL '60 seconds'
          AND severity != 'info'
    """)
    active_anomalies = len(anom_rows)

    # ── Decide status ─────────────────────────────────────────────
    if bgp_total > 0 and (bgp_up / bgp_total) < 0.50:
        status = "down"
    elif active_anomalies > 0:
        status = "degraded"
    else:
        status = "healthy"

    # ── Emit one row per layer + a plant row ──────────────────────
    t_ns = int(call_time.timestamp() * 1_000_000_000)
    layers = [
        ("plant", _sum_bps(spine_layer) + _sum_bps(leaf_uplink_layer) + _sum_bps(leaf_downlink_layer)),
        ("spine", _sum_bps(spine_layer)),
        ("leaf_uplink", _sum_bps(leaf_uplink_layer)),
        ("leaf_downlink", _sum_bps(leaf_downlink_layer)),
    ]
    lines = []
    for layer, (in_bps, out_bps) in layers:
        lines.append(
            f"fabric_health,site={site},layer={layer} "
            f'status="{status}",'
            f"spines_up={spines_up}i,leaves_up={leaves_up}i,bgp_up={bgp_up}i,"
            f"ingress_bps={in_bps:.1f},egress_bps={out_bps:.1f},"
            f"p95_latency_us={p95_us:.3f},ecn_pct={ecn_pct:.6f}"
            f" {t_ns}"
        )
    influxdb3_local.info(
        f"fabric_health: status={status} bgp_up={bgp_up}/{bgp_total} "
        f"anomalies={active_anomalies}"
    )
    write_lines(lines)


def _sum_bps(rows: list) -> tuple[float, float]:
    if not rows:
        return 0.0, 0.0
    r = rows[0]
    return float(r.get("ingress_bps") or 0.0), float(r.get("egress_bps") or 0.0)
```

- [ ] **Step 4: Run tests — expect PASS**

```bash
pytest tests/test_plugins/test_schedule_fabric_health.py -v
```

Expected: 3 passed.

- [ ] **Step 5: Commit**

```bash
git add plugins/schedule_fabric_health.py tests/test_plugins/test_schedule_fabric_health.py
git commit -m "feat(plugins): schedule_fabric_health (every:5s rollup, cross-node write-back)"
```

### Task 15: `schedule_anomaly_detector` plugin

**Files:**
- Create: `plugins/schedule_anomaly_detector.py`
- Create: `tests/test_plugins/test_schedule_anomaly_detector.py`

This plugin keeps in-process state of "currently-firing" anomalies per (kind, switch, interface) so a sustained anomaly produces one row per minute (heartbeat) and a clear-row when it ends.

- [ ] **Step 1: Write tests**

```python
"""Unit-test schedule_anomaly_detector."""

from __future__ import annotations

import importlib.util
import sys
from datetime import datetime, timezone
from pathlib import Path


def _load_plugin():
    fake_wb = type(sys)("_writeback")
    fake_wb._writes: list[str] = []
    def _write_lines(lines):
        fake_wb._writes.extend(list(lines))
    fake_wb.write_lines = _write_lines
    sys.modules["_writeback"] = fake_wb

    p = Path(__file__).resolve().parents[2] / "plugins" / "schedule_anomaly_detector.py"
    spec = importlib.util.spec_from_file_location("schedule_anomaly_detector", p)
    mod = importlib.util.module_from_spec(spec)
    sys.modules["schedule_anomaly_detector"] = mod
    spec.loader.exec_module(mod)
    # Reset module-level state for each test
    mod._active_anomalies = {}
    return mod, fake_wb


class CannedInflux:
    def __init__(self, responses):
        self.responses = responses
        self.queries = []

    def query(self, sql):
        self.queries.append(sql)
        for k, v in self.responses.items():
            fragments = k if isinstance(k, tuple) else (k,)
            if all(f in sql for f in fragments):
                return v
        return []

    def info(self, msg):
        pass


def test_high_util_anomaly_fires(monkeypatch):
    mod, fake_wb = _load_plugin()
    fake = CannedInflux({
        ("FROM interface_counters", "MAX"): [
            {"switch": "leaf-07", "interface": "et-0/0/12", "max_util": 0.94}
        ],
        "FROM bgp_sessions": [],
        ("FROM interface_counters", "z_score"): [],
        "GROUP BY leaf, sibling": [],
    })
    mod.process_scheduled_call(fake, datetime.now(timezone.utc))
    assert any("high_util" in lp for lp in fake_wb._writes)
    assert any("leaf-07" in lp and "et-0/0/12" in lp for lp in fake_wb._writes)


def test_high_util_does_not_re_fire_within_one_minute(monkeypatch):
    mod, fake_wb = _load_plugin()
    fake = CannedInflux({
        ("FROM interface_counters", "MAX"): [
            {"switch": "leaf-07", "interface": "et-0/0/12", "max_util": 0.94}
        ],
        "FROM bgp_sessions": [],
        ("FROM interface_counters", "z_score"): [],
        "GROUP BY leaf, sibling": [],
    })
    t = datetime(2026, 4, 26, 12, 0, 0, tzinfo=timezone.utc)
    mod.process_scheduled_call(fake, t)
    fake_wb._writes.clear()
    from datetime import timedelta
    mod.process_scheduled_call(fake, t + timedelta(seconds=5))
    # No new row in the next 5s — heartbeat is per minute
    assert fake_wb._writes == []


def test_high_util_clears_when_resolved(monkeypatch):
    mod, fake_wb = _load_plugin()
    # First tick: anomaly active
    fake1 = CannedInflux({
        ("FROM interface_counters", "MAX"): [
            {"switch": "leaf-07", "interface": "et-0/0/12", "max_util": 0.94}
        ],
        "FROM bgp_sessions": [],
        ("FROM interface_counters", "z_score"): [],
        "GROUP BY leaf, sibling": [],
    })
    mod.process_scheduled_call(fake1, datetime(2026, 4, 26, 12, 0, 0, tzinfo=timezone.utc))
    fake_wb._writes.clear()
    # Second tick: anomaly cleared (no high-util rows)
    fake2 = CannedInflux({
        ("FROM interface_counters", "MAX"): [],
        "FROM bgp_sessions": [],
        ("FROM interface_counters", "z_score"): [],
        "GROUP BY leaf, sibling": [],
    })
    from datetime import timedelta
    mod.process_scheduled_call(fake2, datetime(2026, 4, 26, 12, 0, 5, tzinfo=timezone.utc))
    # Cleared row written
    assert any('reason="cleared"' in lp for lp in fake_wb._writes)


def test_bgp_flap_anomaly_fires(monkeypatch):
    mod, fake_wb = _load_plugin()
    fake = CannedInflux({
        ("FROM interface_counters", "MAX"): [],
        "FROM bgp_sessions": [
            {"switch": "leaf-12", "peer_switch": "spine-3", "flap_count": 4}
        ],
        ("FROM interface_counters", "z_score"): [],
        "GROUP BY leaf, sibling": [],
    })
    mod.process_scheduled_call(fake, datetime.now(timezone.utc))
    assert any("bgp_flap" in lp for lp in fake_wb._writes)
```

- [ ] **Step 2: Run tests — expect FAIL**

```bash
pytest tests/test_plugins/test_schedule_anomaly_detector.py -v
```

- [ ] **Step 3: Implement `plugins/schedule_anomaly_detector.py`**

```python
"""Schedule trigger: every 5s, scan for fabric anomalies, write detected.

Binding: every:5s, args={}
Runs: on the process node.
Side effects: writes anomaly rows via _writeback. In-process state
keeps "currently active" anomalies so heartbeat rows fire once per
minute (not per 5s tick) and a "cleared" row fires when an anomaly
resolves.

Detected anomaly kinds:
- high_util       — interface > 90% utilization sustained ≥ 30s
- bgp_flap        — > 3 state changes in 1 min for any session
- error_spike     — interface error rate Z-score > 3 vs prior 1h baseline
- ecmp_imbalance  — sibling spine-uplinks on a single leaf differ by > 30%
"""

from __future__ import annotations

from dataclasses import dataclass
from datetime import datetime, timedelta

from _writeback import write_lines

# (kind, switch, interface) -> last-emitted-time (datetime)
_active_anomalies: dict[tuple[str, str, str], datetime] = {}
HEARTBEAT_INTERVAL = timedelta(seconds=60)


@dataclass(frozen=True)
class AnomalyKey:
    kind: str
    switch: str
    interface: str  # may be the peer_switch for bgp_flap


def _emit(t_ns: int, kind: str, severity: str, switch: str, interface: str,
          reason: str, value: float, started_at_ns: int) -> str:
    return (
        f"anomalies,site=dc-east-1,severity={severity},kind={kind},"
        f"switch={switch},interface={interface} "
        f'reason="{reason}",value={value:.3f},started_at_ns={started_at_ns:.0f}'
        f" {t_ns}"
    )


def _detect_high_util(influxdb3_local) -> list[tuple[str, str, float]]:
    rows = influxdb3_local.query("""
        SELECT switch, interface, MAX(in_bytes_rate / capacity_bps) AS max_util
        FROM (
          SELECT switch, interface,
            (in_bytes - LAG(in_bytes) OVER (PARTITION BY switch, interface ORDER BY time)) AS in_bytes_rate,
            CASE WHEN switch LIKE 'spine-%' OR interface LIKE 'et-0/0/%' THEN 100000000000 ELSE 25000000000 END AS capacity_bps
          FROM interface_counters
          WHERE time > now() - INTERVAL '30 seconds'
        ) GROUP BY switch, interface
        HAVING MAX(in_bytes_rate / capacity_bps) > 0.90
    """)
    return [(str(r["switch"]), str(r["interface"]), float(r["max_util"])) for r in rows]


def _detect_bgp_flap(influxdb3_local) -> list[tuple[str, str, int]]:
    # Count state-change events per session in the last minute
    rows = influxdb3_local.query("""
        SELECT switch, peer_switch, COUNT(*) AS flap_count FROM (
          SELECT switch, peer_switch, state,
            LAG(state) OVER (PARTITION BY switch, peer_switch ORDER BY time) AS prev_state
          FROM bgp_sessions
          WHERE time > now() - INTERVAL '1 minute'
        ) WHERE state != prev_state AND prev_state IS NOT NULL
        GROUP BY switch, peer_switch
        HAVING COUNT(*) > 3
    """)
    return [(str(r["switch"]), str(r["peer_switch"]), int(r["flap_count"])) for r in rows]


def _detect_error_spike(influxdb3_local) -> list[tuple[str, str, float]]:
    rows = influxdb3_local.query("""
        SELECT switch, interface, z_score
        FROM (
          SELECT switch, interface, error_rate,
            (error_rate - AVG(error_rate) OVER (PARTITION BY switch, interface)) /
              NULLIF(STDDEV(error_rate) OVER (PARTITION BY switch, interface), 0) AS z_score
          FROM (
            SELECT switch, interface,
              SUM(in_errors) * 1.0 / NULLIF(SUM(in_pkts), 0) AS error_rate
            FROM interface_counters
            WHERE time > now() - INTERVAL '1 hour'
            GROUP BY switch, interface, date_bin(INTERVAL '30 seconds', time)
          )
        ) WHERE z_score > 3.0
    """)
    return [(str(r["switch"]), str(r["interface"]), float(r["z_score"])) for r in rows]


def _detect_ecmp_imbalance(influxdb3_local) -> list[tuple[str, str, float]]:
    rows = influxdb3_local.query("""
        SELECT switch AS leaf, sibling, MAX(diff) AS imbalance
        FROM (
          SELECT switch, interface AS sibling,
            ABS(SUM(in_bytes) - AVG(SUM(in_bytes)) OVER (PARTITION BY switch))
              / NULLIF(AVG(SUM(in_bytes)) OVER (PARTITION BY switch), 0) AS diff
          FROM interface_counters
          WHERE time > now() - INTERVAL '30 seconds'
            AND switch LIKE 'leaf-%'
            AND interface LIKE 'et-0/0/%'
          GROUP BY switch, interface
        )
        GROUP BY leaf, sibling
        HAVING MAX(diff) > 0.30
    """)
    return [(str(r["leaf"]), str(r["sibling"]), float(r["imbalance"])) for r in rows]


def process_scheduled_call(influxdb3_local, call_time, args=None):
    detected: dict[AnomalyKey, tuple[str, float]] = {}

    for switch, interface, util in _detect_high_util(influxdb3_local):
        detected[AnomalyKey("high_util", switch, interface)] = (
            f"util={util:.2f}", util,
        )
    for switch, peer, flaps in _detect_bgp_flap(influxdb3_local):
        detected[AnomalyKey("bgp_flap", switch, peer)] = (
            f"flaps={flaps}/min", float(flaps),
        )
    for switch, interface, z in _detect_error_spike(influxdb3_local):
        detected[AnomalyKey("error_spike", switch, interface)] = (
            f"z_score={z:.2f}", z,
        )
    for leaf, sibling, imb in _detect_ecmp_imbalance(influxdb3_local):
        detected[AnomalyKey("ecmp_imbalance", leaf, sibling)] = (
            f"imbalance={imb:.2f}", imb,
        )

    t_ns = int(call_time.timestamp() * 1_000_000_000)
    lines: list[str] = []

    # New + heartbeat anomalies
    for key, (reason, value) in detected.items():
        last_emit = _active_anomalies.get((key.kind, key.switch, key.interface))
        if last_emit is None or call_time - last_emit >= HEARTBEAT_INTERVAL:
            severity = "critical" if key.kind in ("high_util", "bgp_flap") else "warning"
            started_ns = t_ns if last_emit is None else int(last_emit.timestamp() * 1e9)
            lines.append(_emit(
                t_ns, key.kind, severity, key.switch, key.interface,
                reason, value, started_ns,
            ))
            _active_anomalies[(key.kind, key.switch, key.interface)] = call_time

    # Cleared anomalies
    detected_keys = {(k.kind, k.switch, k.interface) for k in detected}
    cleared = [k for k in _active_anomalies if k not in detected_keys]
    for kind, switch, interface in cleared:
        lines.append(_emit(t_ns, kind, "info", switch, interface, "cleared", 0.0, 0))
        del _active_anomalies[(kind, switch, interface)]

    if lines:
        influxdb3_local.info(f"anomaly_detector: {len(lines)} rows ({len(detected)} active)")
        write_lines(lines)
```

- [ ] **Step 4: Run tests — expect PASS**

```bash
pytest tests/test_plugins/test_schedule_anomaly_detector.py -v
```

Expected: 4 passed.

- [ ] **Step 5: Commit**

```bash
git add plugins/schedule_anomaly_detector.py tests/test_plugins/test_schedule_anomaly_detector.py
git commit -m "feat(plugins): schedule_anomaly_detector (windowed detection + heartbeat)"
```

### Task 16: `request_top_talkers` plugin

**Files:**
- Create: `plugins/request_top_talkers.py`
- Create: `tests/test_plugins/test_request_top_talkers.py`

- [ ] **Step 1: Write tests**

```python
"""Unit-test request_top_talkers."""

from __future__ import annotations

import importlib.util
import json
import sys
from pathlib import Path


def _load_plugin():
    p = Path(__file__).resolve().parents[2] / "plugins" / "request_top_talkers.py"
    spec = importlib.util.spec_from_file_location("request_top_talkers", p)
    mod = importlib.util.module_from_spec(spec)
    sys.modules["request_top_talkers"] = mod
    spec.loader.exec_module(mod)
    return mod


class CannedInflux:
    def __init__(self, rows):
        self._rows = rows
    def query(self, sql):
        return self._rows
    def info(self, m):
        pass


def test_returns_top_n_sorted_by_bytes():
    mod = _load_plugin()
    fake = CannedInflux([
        {"src_ip": "10.4.7.91", "bytes": 5_000_000, "packets": 5000, "top_dst": "10.0.1.5"},
        {"src_ip": "10.7.2.3",  "bytes": 3_000_000, "packets": 3000, "top_dst": "10.4.7.91"},
        {"src_ip": "10.1.4.13", "bytes": 1_000_000, "packets": 1000, "top_dst": "10.4.7.91"},
    ])
    body_bytes = json.dumps({"window_minutes": 5, "limit": 10}).encode()
    resp = mod.process_request(fake, query_parameters={}, request_headers={},
                                request_body=body_bytes)
    assert "talkers" in resp
    assert resp["talkers"][0]["src_ip"] == "10.4.7.91"
    assert resp["talkers"][0]["bytes"] == 5_000_000


def test_default_window_and_limit_when_body_empty():
    mod = _load_plugin()
    fake = CannedInflux([])
    resp = mod.process_request(fake, query_parameters={}, request_headers={},
                                request_body=b"")
    assert resp["talkers"] == []
    assert "generated_at" in resp


def test_limit_clamped_to_max():
    mod = _load_plugin()
    fake = CannedInflux([{"src_ip": f"10.{i}.0.1", "bytes": 100, "packets": 1, "top_dst": "10.0.0.1"} for i in range(50)])
    body = json.dumps({"window_minutes": 5, "limit": 1000}).encode()
    resp = mod.process_request(fake, query_parameters={}, request_headers={},
                                request_body=body)
    # We cap at 100 max
    assert len(resp["talkers"]) <= 100
```

- [ ] **Step 2: Run tests — expect FAIL**

```bash
pytest tests/test_plugins/test_request_top_talkers.py -v
```

- [ ] **Step 3: Implement `plugins/request_top_talkers.py`**

```python
"""Request trigger: POST /api/v3/engine/top_talkers — top-N source IPs by bytes.

Binding: request path="top_talkers", args={}
Runs: on the query node.
Body: {"window_minutes": 5, "limit": 10}
Returns: {
  "talkers": [{"src_ip": ..., "bytes": ..., "packets": ..., "top_dst": ...}, ...],
  "generated_at": "..."
}

Uses the DVC for distinct-src enumeration and aggregates flow_records.
"""

from __future__ import annotations

import json
from datetime import UTC, datetime


MAX_LIMIT = 100


def process_request(influxdb3_local, query_parameters, request_headers, request_body, args=None):
    if request_body:
        try:
            body = json.loads(request_body)
        except (json.JSONDecodeError, ValueError):
            body = {}
    else:
        body = {}
    window_min = int(body.get("window_minutes", 5))
    limit = min(int(body.get("limit", 10)), MAX_LIMIT)

    rows = influxdb3_local.query(f"""
        SELECT src_ip,
               SUM(bytes) AS bytes,
               SUM(packets) AS packets,
               (SELECT dst_ip FROM flow_records f2
                WHERE f2.src_ip = f1.src_ip
                  AND f2.time > now() - INTERVAL '{window_min} minutes'
                GROUP BY dst_ip
                ORDER BY SUM(bytes) DESC LIMIT 1) AS top_dst
        FROM flow_records f1
        WHERE time > now() - INTERVAL '{window_min} minutes'
        GROUP BY src_ip
        ORDER BY SUM(bytes) DESC
        LIMIT {limit}
    """)

    talkers = [{
        "src_ip": str(r.get("src_ip", "")),
        "bytes": int(r.get("bytes") or 0),
        "packets": int(r.get("packets") or 0),
        "top_dst": str(r.get("top_dst") or ""),
    } for r in rows]

    return {
        "talkers": talkers,
        "generated_at": datetime.now(UTC).strftime("%Y-%m-%dT%H:%M:%SZ"),
    }
```

- [ ] **Step 4: Run tests — expect PASS**

```bash
pytest tests/test_plugins/test_request_top_talkers.py -v
```

Expected: 3 passed.

- [ ] **Step 5: Commit**

```bash
git add plugins/request_top_talkers.py tests/test_plugins/test_request_top_talkers.py
git commit -m "feat(plugins): request_top_talkers"
```

### Task 17: `request_src_ip_detail` plugin

**Files:**
- Create: `plugins/request_src_ip_detail.py`
- Create: `tests/test_plugins/test_request_src_ip_detail.py`

Composite payload — 4 sub-queries returned as a single JSON.

- [ ] **Step 1: Write tests**

```python
"""Unit-test request_src_ip_detail."""

from __future__ import annotations

import importlib.util
import json
import sys
from pathlib import Path


def _load_plugin():
    p = Path(__file__).resolve().parents[2] / "plugins" / "request_src_ip_detail.py"
    spec = importlib.util.spec_from_file_location("request_src_ip_detail", p)
    mod = importlib.util.module_from_spec(spec)
    sys.modules["request_src_ip_detail"] = mod
    spec.loader.exec_module(mod)
    return mod


class CannedInflux:
    def __init__(self, responses):
        self.responses = responses
    def query(self, sql):
        for k, v in self.responses.items():
            fragments = k if isinstance(k, tuple) else (k,)
            if all(f in sql for f in fragments):
                return v
        return []
    def info(self, m):
        pass


def test_returns_composite_payload():
    mod = _load_plugin()
    fake = CannedInflux({
        ("date_bin", "src_ip"): [
            {"bucket": "1700000000000000000", "bytes": 1000, "packets": 1},
            {"bucket": "1700000060000000000", "bytes": 2000, "packets": 2},
        ],
        ("dst_ip", "GROUP BY dst_ip"): [
            {"dst_ip": "10.0.1.5", "bytes": 5000},
            {"dst_ip": "10.7.2.3", "bytes": 3000},
        ],
        ("dst_switch", "GROUP BY dst_switch"): [
            {"dst_switch": "leaf-01", "bytes": 5000},
            {"dst_switch": "leaf-08", "bytes": 3000},
        ],
        "ecn_pct": [{"ecn_pct": 0.05}],
    })
    body = json.dumps({"src_ip": "10.4.7.91"}).encode()
    resp = mod.process_request(fake, query_parameters={}, request_headers={},
                                request_body=body)
    assert resp["src_ip"] == "10.4.7.91"
    assert "sparkline" in resp and len(resp["sparkline"]) == 2
    assert "top_destinations" in resp and len(resp["top_destinations"]) == 2
    assert "leaf_distribution" in resp and len(resp["leaf_distribution"]) == 2
    assert resp["ecn_pct"] == 0.05


def test_returns_400_when_src_ip_missing():
    mod = _load_plugin()
    fake = CannedInflux({})
    resp = mod.process_request(fake, query_parameters={}, request_headers={},
                                request_body=b"")
    assert resp.get("error") == "missing required body: src_ip"
```

- [ ] **Step 2: Run tests — expect FAIL**

```bash
pytest tests/test_plugins/test_request_src_ip_detail.py -v
```

- [ ] **Step 3: Implement `plugins/request_src_ip_detail.py`**

```python
"""Request trigger: POST /api/v3/engine/src_ip_detail — composite drill-down for one src_ip.

Binding: request path="src_ip_detail", args={}
Body: {"src_ip": "10.4.7.91"}
Returns: {
  "src_ip": ...,
  "sparkline": [{"bucket": ..., "bytes": ..., "packets": ...}, ...],
  "top_destinations": [{"dst_ip": ..., "bytes": ...}, ...],
  "leaf_distribution": [{"dst_switch": ..., "bytes": ...}, ...],
  "ecn_pct": float,
  "generated_at": "..."
}
"""

from __future__ import annotations

import json
from datetime import UTC, datetime


def process_request(influxdb3_local, query_parameters, request_headers, request_body, args=None):
    if not request_body:
        return {"error": "missing required body: src_ip"}
    try:
        body = json.loads(request_body)
    except (json.JSONDecodeError, ValueError):
        return {"error": "request body must be JSON"}
    src_ip = body.get("src_ip", "").strip()
    if not src_ip:
        return {"error": "missing required body: src_ip"}

    # Sparkline: bytes/packets per minute over last hour
    sparkline_rows = influxdb3_local.query(f"""
        SELECT date_bin(INTERVAL '1 minute', time) AS bucket,
               SUM(bytes) AS bytes,
               SUM(packets) AS packets
        FROM flow_records
        WHERE src_ip = '{src_ip}'
          AND time > now() - INTERVAL '1 hour'
        GROUP BY bucket
        ORDER BY bucket
    """)

    # Top destinations
    dst_rows = influxdb3_local.query(f"""
        SELECT dst_ip, SUM(bytes) AS bytes
        FROM flow_records
        WHERE src_ip = '{src_ip}'
          AND time > now() - INTERVAL '1 hour'
        GROUP BY dst_ip
        ORDER BY SUM(bytes) DESC
        LIMIT 5
    """)

    # Leaf distribution (which dst_switches this IP's traffic flows to)
    leaf_rows = influxdb3_local.query(f"""
        SELECT dst_switch, SUM(bytes) AS bytes
        FROM flow_records
        WHERE src_ip = '{src_ip}'
          AND time > now() - INTERVAL '1 hour'
        GROUP BY dst_switch
        ORDER BY SUM(bytes) DESC
    """)

    # ECN-marked % on the leaf-side interfaces this IP's flows transited
    ecn_rows = influxdb3_local.query(f"""
        SELECT
          SUM(ic.ecn_marked_pkts) / NULLIF(SUM(ic.in_pkts) + SUM(ic.out_pkts), 0) AS ecn_pct
        FROM interface_counters ic
        WHERE ic.switch IN (
          SELECT DISTINCT dst_switch FROM flow_records
          WHERE src_ip = '{src_ip}' AND time > now() - INTERVAL '1 hour'
        )
        AND ic.time > now() - INTERVAL '1 hour'
    """)
    ecn_pct = float((ecn_rows[0] or {}).get("ecn_pct") or 0.0) if ecn_rows else 0.0

    return {
        "src_ip": src_ip,
        "sparkline": [{
            "bucket": str(r.get("bucket", "")),
            "bytes": int(r.get("bytes") or 0),
            "packets": int(r.get("packets") or 0),
        } for r in sparkline_rows],
        "top_destinations": [{
            "dst_ip": str(r.get("dst_ip", "")),
            "bytes": int(r.get("bytes") or 0),
        } for r in dst_rows],
        "leaf_distribution": [{
            "dst_switch": str(r.get("dst_switch", "")),
            "bytes": int(r.get("bytes") or 0),
        } for r in leaf_rows],
        "ecn_pct": ecn_pct,
        "generated_at": datetime.now(UTC).strftime("%Y-%m-%dT%H:%M:%SZ"),
    }
```

- [ ] **Step 4: Run tests — expect PASS**

```bash
pytest tests/test_plugins/test_request_src_ip_detail.py -v
```

Expected: 2 passed.

- [ ] **Step 5: Commit**

```bash
git add plugins/request_src_ip_detail.py tests/test_plugins/test_request_src_ip_detail.py
git commit -m "feat(plugins): request_src_ip_detail (composite drill-down)"
```

### Task 18: Register triggers in `init.sh` (replaces stub)

**Files:**
- Modify: `influxdb/init.sh`

The two schedule triggers register on the **process node** (where they need to run). The two request triggers register on the **query node** (where they need to be reachable for the UI's direct fetches).

In InfluxDB 3 Enterprise's clustered model, triggers are catalog objects shared across nodes (the catalog is shared via the volume), but the engine routes execution based on the node's mode + capabilities. Schedule triggers run on whichever node has `process` mode and `--plugin-dir` set; request triggers run on whichever node has `query` mode + `--plugin-dir`. Since we mount `./plugins:/plugins:ro` on both `nt-query` and `nt-process`, both can host whichever triggers match their mode.

For idempotency we register all four triggers via the init container (which talks to the query node by default), and the engine routes execution appropriately. Phase 0 verification confirms this routing actually works in this version of Enterprise; if it doesn't, the trigger creates need to target a specific node, and the init script needs to be split.

- [ ] **Step 1: Replace `ensure_triggers()` body in `influxdb/init.sh`**

```bash
ensure_triggers() {
    log "registering processing-engine triggers"

    # Schedule triggers — every:5s, run on process node
    idempotent "trigger fabric_health" create trigger \
        --database "${INFLUX_DB}" \
        --trigger-spec "every:5s" \
        --path "schedule_fabric_health.py" \
        fabric_health

    idempotent "trigger anomaly_detector" create trigger \
        --database "${INFLUX_DB}" \
        --trigger-spec "every:5s" \
        --path "schedule_anomaly_detector.py" \
        anomaly_detector

    # Request triggers — run on query node
    idempotent "trigger top_talkers" create trigger \
        --database "${INFLUX_DB}" \
        --trigger-spec "request:top_talkers" \
        --path "request_top_talkers.py" \
        top_talkers

    idempotent "trigger src_ip_detail" create trigger \
        --database "${INFLUX_DB}" \
        --trigger-spec "request:src_ip_detail" \
        --path "request_src_ip_detail.py" \
        src_ip_detail

    for t in fabric_health anomaly_detector top_talkers src_ip_detail; do
        cli enable trigger "${t}" --database "${INFLUX_DB}" 2>/dev/null || \
            log "trigger ${t} enable no-op"
    done
}
```

- [ ] **Step 2: Verify shellcheck-clean**

```bash
shellcheck /Users/pauldix/codez/reference_architectures/influxdb3-ref-network-telemetry/influxdb/init.sh || true
```

- [ ] **Step 3: Commit**

```bash
git add influxdb/init.sh
git commit -m "feat(influxdb): register the four processing-engine triggers in init.sh"
```

---

## Phase 7 — Integration checkpoint #2 (executed)

### Task 19: Verify the full end-to-end data flow with plugins active

The cluster is still running from Phase 4. We need init.sh to re-run with the new trigger registrations from Task 18, so cycle the cluster.

**All `docker compose` commands here need `dangerouslyDisableSandbox: true`.**

- [ ] **Step 1: Recycle the cluster so init.sh re-runs the new triggers**

```bash
cd /Users/pauldix/codez/reference_architectures/influxdb3-ref-network-telemetry
docker compose down
sleep 5
docker compose up -d 2>&1 | tail -5

# Wait for all 5 nodes healthy + init complete
deadline=$(($(date +%s) + 300))
for n in nt-ingest-1 nt-ingest-2 nt-query nt-compact nt-process; do
    while [ $(date +%s) -lt $deadline ]; do
        if docker compose ps --format json $n 2>/dev/null | grep -q '"Health":"healthy"'; then break; fi
        sleep 3
    done
done

# Wait for init container to complete
while [ $(date +%s) -lt $deadline ]; do
    if docker compose ps --format json influxdb3-init 2>/dev/null | grep -q '"State":"exited"'; then break; fi
    sleep 3
done

docker compose logs influxdb3-init 2>&1 | tail -10
# Expected: trigger creation lines + "initialization complete"
```

- [ ] **Step 2: Assert all 4 triggers registered and enabled**

```bash
TOKEN=$(docker compose exec -T influxdb3-query cat /var/lib/influxdb3/.nt-token-plain | tr -d '\r\n')
docker compose exec -T influxdb3-query bash -lc \
  "influxdb3 show trigger --database nt --token $TOKEN" 2>&1 | tee /tmp/triggers.out
# Expected: 4 triggers listed — fabric_health, anomaly_detector, top_talkers, src_ip_detail.
grep -c "fabric_health\|anomaly_detector\|top_talkers\|src_ip_detail" /tmp/triggers.out
# Expected: 4
```

If any trigger is missing or disabled, fix init.sh's `ensure_triggers` and re-run from Step 1.

- [ ] **Step 3: Wait for schedule plugins to start firing, then assert fabric_health rows**

```bash
sleep 60  # 12 ticks at every:5s for both schedule plugins

_q() {
  curl -s -X POST -H "Authorization: Bearer $TOKEN" \
       -H "Content-Type: application/json" \
       http://localhost:8181/api/v3/query_sql \
       -d "{\"db\":\"nt\",\"q\":\"$1\",\"format\":\"json\"}"
}

_q "SELECT COUNT(*) AS n FROM fabric_health WHERE time > now() - INTERVAL '1 minute'"
# Expected: n > 30 (4 layer rows × ~12 ticks/min = 48; allow slack)

# Check the actual status values being written
_q "SELECT layer, status, spines_up, leaves_up, bgp_up FROM fabric_health WHERE layer='plant' ORDER BY time DESC LIMIT 1"
# Expected: status="healthy", bgp_up=128 (or close), spines_up=8, leaves_up=16
```

If fabric_health is empty after 60s, the plugin write-back path is broken:

```bash
docker compose logs influxdb3-process 2>&1 | grep -iE "error|traceback|fabric_health" | tail -20
# Look for: Python tracebacks, "ConnectionError", "Token not found"

# Verify process node can reach ingest-1
docker compose exec -T influxdb3-process curl -s -o /dev/null -w '%{http_code}\n' \
    http://nt-ingest-1:8181/health
# Expected: 401 (means reachable, just needs auth — fine)
```

- [ ] **Step 4: Run congestion_hotspot — assert anomaly fires**

```bash
docker compose --profile scenarios run --rm \
  -e SCENARIO=congestion_hotspot scenarios 2>&1 | tail -10
# Should print "[scenario step 5] DONE" and exit 0

sleep 10  # let anomaly_detector's next 5s tick fire after the 30s sustain

_q "SELECT time, kind, severity, switch, interface, reason FROM anomalies WHERE switch='leaf-07' AND interface='et-0/0/12' ORDER BY time DESC LIMIT 5"
# Expected: at least one row with kind="high_util", severity="critical"
# After scenario recovery: also a row with severity="info", reason="cleared"
```

If the high_util anomaly didn't fire, the detector's window query is wrong — check `_detect_high_util` in `schedule_anomaly_detector.py`.

- [ ] **Step 5: Run east_west_burst — assert src_ip_detail returns the spike**

```bash
docker compose --profile scenarios run --rm \
  -e SCENARIO=east_west_burst scenarios 2>&1 | tail -5
# Should print "[scenario step 4] DONE"

sleep 5

# Typeahead via DVC
_q "SELECT src_ip FROM distinct_cache('flow_records', 'src_ip_distinct') WHERE src_ip LIKE '10.4%' LIMIT 20" \
  | tee /tmp/typeahead.out
grep -q "10.4.7.91" /tmp/typeahead.out && echo "✓ typeahead found 10.4.7.91" || echo "✗ MISSING"

# src_ip_detail plugin
curl -s -H "Authorization: Bearer ${TOKEN}" \
     -X POST http://localhost:8181/api/v3/engine/src_ip_detail \
     -H "Content-Type: application/json" \
     -d '{"src_ip": "10.4.7.91"}' | tee /tmp/detail.json | python3 -m json.tool | head -40
# Expected: JSON with src_ip="10.4.7.91", non-empty sparkline, top_destinations (5 entries),
# leaf_distribution, ecn_pct.

python3 -c '
import json
d = json.load(open("/tmp/detail.json"))
b = d.get("body", d)
assert b.get("src_ip") == "10.4.7.91", f"src_ip mismatch: {b}"
assert len(b.get("sparkline", [])) > 0, "empty sparkline"
assert len(b.get("top_destinations", [])) > 0, "empty top_destinations"
print("✓ src_ip_detail OK")
'
```

- [ ] **Step 6: Hit the top_talkers plugin — assert non-empty payload**

```bash
curl -s -H "Authorization: Bearer ${TOKEN}" \
     -X POST http://localhost:8181/api/v3/engine/top_talkers \
     -H "Content-Type: application/json" \
     -d '{"window_minutes": 5, "limit": 10}' | tee /tmp/talkers.json | python3 -m json.tool | head -30

python3 -c '
import json
d = json.load(open("/tmp/talkers.json"))
b = d.get("body", d)
talkers = b.get("talkers", [])
assert len(talkers) > 0, "empty talkers list"
print(f"✓ top_talkers returned {len(talkers)} entries; top src_ip = {talkers[0][\"src_ip\"]}")
'
```

- [ ] **Step 7: Outcome**

If every assertion passed, the plugin layer is fully integrated. **Leave the cluster running** — Phase 8 (UI) builds against it. The integration test in Phase 11 will exercise the UI end-to-end.

If any step failed, fix the underlying plugin / init.sh / writeback issue, commit, recycle, re-run this checkpoint.

---

## Phase 8 — UI

The UI is smaller than iiot's because it's aggregate-led — no per-machine grid; only a compact andon-style banner + KPIs + throughput chart + top-talkers + typeahead + detail panel + active anomalies.

### Task 20: `ui/queries.py` with unit tests

**Files:**
- Create: `ui/__init__.py`
- Create: `ui/queries.py`
- Create: `tests/test_queries.py`

- [ ] **Step 1: Create the package marker**

```bash
mkdir -p /Users/pauldix/codez/reference_architectures/influxdb3-ref-network-telemetry/ui
touch /Users/pauldix/codez/reference_architectures/influxdb3-ref-network-telemetry/ui/__init__.py
```

- [ ] **Step 2: Write tests**

Create `tests/test_queries.py`:

```python
"""Tests for the named SQL queries."""

from __future__ import annotations

import re

from ui import queries as q


def _normalize(sql: str) -> str:
    return re.sub(r"\s+", " ", sql).strip()


def test_fabric_state_query_uses_fabric_health_rollup():
    sql = _normalize(q.fabric_state_sql())
    assert "FROM fabric_health" in sql
    assert "ORDER BY TIME DESC" in sql.upper()
    assert "LIMIT 1" in sql.upper()


def test_kpi_row_query_aggregates_recent_traffic():
    sql = _normalize(q.kpi_row_sql())
    # Should reference both interface_counters (for ingress/egress) and
    # flow_records (for flows/sec) and latency_probes (for p95 latency)
    assert "FROM" in sql.upper()


def test_throughput_chart_query_buckets_per_layer():
    sql = _normalize(q.throughput_chart_sql(minutes=10))
    assert "date_bin(INTERVAL '5 seconds'" in sql
    assert "FROM interface_counters" in sql
    assert "GROUP BY" in sql.upper()
    assert "INTERVAL '10 minutes'" in sql or "INTERVAL '10 minute'" in sql


def test_anomalies_query_orders_desc():
    sql = _normalize(q.anomalies_sql(window_minutes=5, limit=20))
    assert "FROM anomalies" in sql
    assert "INTERVAL '5 minutes'" in sql
    assert "ORDER BY TIME DESC" in sql.upper()
    assert "LIMIT 20" in sql.upper()
```

- [ ] **Step 3: Run tests — expect FAIL**

```bash
pytest tests/test_queries.py -v
```

- [ ] **Step 4: Implement `ui/queries.py`**

```python
"""Named SQL queries the FastAPI backend runs against the query node.

The browser also runs SQL directly against the query node — but only one
query: the typeahead, which is a single SELECT against
distinct_cache('flow_records', 'src_ip_distinct'). That query is built
in JavaScript (see ui/static/app.js) so users can see the cache speed.
The two request plugins (top_talkers, src_ip_detail) handle anything
that warrants a composite payload.

This file is the documented teaching artifact for the SQL-via-FastAPI
path of the three-pattern dashboard.
"""

from __future__ import annotations


def fabric_state_sql() -> str:
    """Latest fabric_health row → drives the green/yellow/red banner.

    Used by: GET /partials/fabric_state -> _fabric_state.html
    """
    return """
        SELECT site, layer, status, spines_up, leaves_up, bgp_up,
               ingress_bps, egress_bps, p95_latency_us, ecn_pct
        FROM fabric_health
        WHERE layer = 'plant'
        ORDER BY time DESC
        LIMIT 1
    """


def kpi_row_sql() -> str:
    """Aggregate KPIs over the last minute for the 5 KPI tiles."""
    return """
        SELECT
          (SELECT SUM(in_bytes) * 8.0 / 60.0 FROM interface_counters
            WHERE switch LIKE 'spine-%' AND time > now() - INTERVAL '1 minute') AS ingress_bps,
          (SELECT SUM(out_bytes) * 8.0 / 60.0 FROM interface_counters
            WHERE switch LIKE 'spine-%' AND time > now() - INTERVAL '1 minute') AS egress_bps,
          (SELECT approx_percentile_cont(rtt_us, 0.95) FROM latency_probes
            WHERE time > now() - INTERVAL '1 minute') AS p95_latency_us,
          (SELECT COUNT(*) FROM flow_records
            WHERE time > now() - INTERVAL '1 minute') AS flows_last_minute,
          (SELECT
             SUM(ecn_marked_pkts) / NULLIF(SUM(in_pkts) + SUM(out_pkts), 0)
            FROM interface_counters
            WHERE time > now() - INTERVAL '1 minute') AS ecn_pct
    """


def throughput_chart_sql(minutes: int = 10) -> str:
    """Per-layer throughput in 5-second buckets for the stacked-area chart."""
    return f"""
        SELECT
          CASE
            WHEN switch LIKE 'spine-%' THEN 'spine'
            WHEN switch LIKE 'leaf-%' AND interface LIKE 'et-0/0/%' THEN 'leaf_uplink'
            WHEN switch LIKE 'leaf-%' AND interface LIKE 'et-0/1/%' THEN 'leaf_downlink'
            ELSE 'other'
          END AS layer,
          date_bin(INTERVAL '5 seconds', time) AS bucket,
          SUM(in_bytes + out_bytes) * 8.0 / 5.0 AS bps
        FROM interface_counters
        WHERE time > now() - INTERVAL '{minutes} minutes'
        GROUP BY layer, bucket
        ORDER BY bucket
    """


def anomalies_sql(window_minutes: int = 5, limit: int = 20) -> str:
    """Recent anomalies for the active-anomalies panel.

    Filters out severity='info' (the cleared rows) so the panel shows
    only currently-active anomalies. Time-window is short by design —
    the heartbeat cadence in schedule_anomaly_detector is 60s, so a
    5-minute window will only show actively-firing problems.
    """
    return f"""
        SELECT time, kind, severity, switch, interface, reason, value
        FROM anomalies
        WHERE time > now() - INTERVAL '{window_minutes} minutes'
          AND severity != 'info'
        ORDER BY time DESC
        LIMIT {limit}
    """
```

- [ ] **Step 5: Run tests — expect PASS**

```bash
pytest tests/test_queries.py -v
```

Expected: 4 passed.

- [ ] **Step 6: Commit**

```bash
git add ui/__init__.py ui/queries.py tests/test_queries.py
git commit -m "feat(ui): named SQL queries (fabric_state, kpi, throughput, anomalies)"
```

### Task 21: FastAPI app + base + overview templates

**Files:**
- Create: `ui/app.py`
- Create: `ui/templates/base.html`, `ui/templates/overview.html`
- Create: `ui/templates/partials/_fabric_state.html`, `_kpi_row.html`, `_anomalies.html`

- [ ] **Step 1: Create directories**

```bash
mkdir -p /Users/pauldix/codez/reference_architectures/influxdb3-ref-network-telemetry/ui/templates/partials
mkdir -p /Users/pauldix/codez/reference_architectures/influxdb3-ref-network-telemetry/ui/static
```

- [ ] **Step 2: Implement `ui/app.py`**

```python
"""FastAPI app serving the network-telemetry dashboard.

Three teaching patterns split across endpoints:
  - SQL via this backend: /partials/fabric_state, /partials/kpi_row,
    /partials/throughput, /partials/anomalies
  - SQL from browser (DVC typeahead): browser hits /api/v3/query_sql
    on the query node directly — see ui/static/app.js
  - Request plugin from browser: top_talkers and src_ip_detail are
    served by Processing Engine plugins on the query node — see
    ui/static/app.js for the direct fetches with latency badges
"""

from __future__ import annotations

import json
import os
from pathlib import Path

import httpx
from fastapi import FastAPI, Request
from fastapi.responses import HTMLResponse, JSONResponse
from fastapi.staticfiles import StaticFiles
from fastapi.templating import Jinja2Templates

from ui import queries

ROOT = Path(__file__).resolve().parent
TEMPLATES = Jinja2Templates(directory=str(ROOT / "templates"))

INFLUX_URL = os.environ.get("INFLUX_URL", "http://nt-query:8181")
INFLUX_PUBLIC_URL = os.environ.get("INFLUX_PUBLIC_URL", "http://localhost:8181")
INFLUX_DB = os.environ.get("INFLUX_DB", "nt")
TOKEN_FILE = os.environ.get("INFLUX_TOKEN_FILE", "/tokens/.nt-operator-token")


def _load_token() -> str:
    if "INFLUXDB3_TOKEN" in os.environ:
        return os.environ["INFLUXDB3_TOKEN"]
    with open(TOKEN_FILE) as f:
        return json.load(f)["token"]


app = FastAPI()
app.mount("/static", StaticFiles(directory=str(ROOT / "static")), name="static")


def _client() -> httpx.Client:
    return httpx.Client(
        base_url=INFLUX_URL,
        headers={"Authorization": f"Bearer {_load_token()}"},
        timeout=10.0,
    )


def _query(sql: str) -> list[dict]:
    with _client() as c:
        r = c.post(
            "/api/v3/query_sql",
            json={"db": INFLUX_DB, "q": sql, "format": "json"},
        )
        r.raise_for_status()
        return r.json() or []


def _poll_intervals() -> dict[str, int]:
    return {
        "kpi": int(os.environ.get("UI_KPI_POLL_MS", "5000")),
        "throughput": int(os.environ.get("UI_THROUGHPUT_POLL_MS", "10000")),
        "anomalies": int(os.environ.get("UI_ANOMALIES_POLL_MS", "5000")),
        "top_talkers": int(os.environ.get("UI_TOP_TALKERS_POLL_MS", "5000")),
    }


@app.get("/", response_class=HTMLResponse)
def overview(request: Request) -> HTMLResponse:
    return TEMPLATES.TemplateResponse(
        request,
        "overview.html",
        {
            "poll": _poll_intervals(),
            "influx_public_url": INFLUX_PUBLIC_URL,
            "influx_db": INFLUX_DB,
            "influx_token": _load_token(),
        },
    )


@app.get("/partials/fabric_state", response_class=HTMLResponse)
def fabric_state(request: Request) -> HTMLResponse:
    rows = _query(queries.fabric_state_sql())
    if not rows:
        return TEMPLATES.TemplateResponse(request, "partials/_fabric_state.html",
                                           {"status": "unknown", "row": None})
    row = rows[0]
    return TEMPLATES.TemplateResponse(
        request,
        "partials/_fabric_state.html",
        {"status": str(row.get("status", "unknown")).strip('"'), "row": row},
    )


@app.get("/partials/kpi_row", response_class=HTMLResponse)
def kpi_row(request: Request) -> HTMLResponse:
    rows = _query(queries.kpi_row_sql())
    r = rows[0] if rows else {}
    return TEMPLATES.TemplateResponse(
        request,
        "partials/_kpi_row.html",
        {
            "ingress_tbps": (float(r.get("ingress_bps") or 0) / 1e12),
            "egress_tbps": (float(r.get("egress_bps") or 0) / 1e12),
            "p95_latency_us": float(r.get("p95_latency_us") or 0),
            "flows_last_minute": int(r.get("flows_last_minute") or 0),
            "ecn_pct": (float(r.get("ecn_pct") or 0) * 100),
        },
    )


@app.get("/partials/throughput", response_class=JSONResponse)
def throughput() -> JSONResponse:
    rows = _query(queries.throughput_chart_sql(minutes=10))
    return JSONResponse({"rows": rows})


@app.get("/partials/anomalies", response_class=HTMLResponse)
def anomalies(request: Request) -> HTMLResponse:
    rows = _query(queries.anomalies_sql(window_minutes=5, limit=20))
    return TEMPLATES.TemplateResponse(request, "partials/_anomalies.html",
                                       {"anomalies": rows})
```

- [ ] **Step 3: Implement templates**

`ui/templates/base.html`:

```html
<!doctype html>
<html lang="en">
<head>
  <meta charset="utf-8">
  <meta name="viewport" content="width=device-width, initial-scale=1">
  <title>{% block title %}Network Telemetry — InfluxDB 3 Enterprise reference{% endblock %}</title>
  <link rel="stylesheet" href="/static/uplot.min.css">
  <link rel="stylesheet" href="/static/app.css">
  <script src="/static/htmx.min.js"></script>
  <script src="/static/uplot.min.js"></script>
</head>
<body>
  <header class="topbar">
    <div class="brand">
      <span class="brand-mark">influxdb3-ref-network-telemetry</span>
      <span class="brand-tag">data-center fabric · multi-node InfluxDB 3 Enterprise</span>
    </div>
  </header>
  <main>{% block main %}{% endblock %}</main>
  <script src="/static/app.js"></script>
</body>
</html>
```

`ui/templates/overview.html`:

```html
{% extends "base.html" %}
{% block main %}

<div id="fabric-state"
     hx-get="/partials/fabric_state"
     hx-trigger="load, every {{ poll.kpi }}ms"
     hx-swap="innerHTML"></div>

<div id="kpi-row"
     hx-get="/partials/kpi_row"
     hx-trigger="load, every {{ poll.kpi }}ms"
     hx-swap="innerHTML"></div>

<section class="panel" id="throughput-panel"
         data-throughput-poll-ms="{{ poll.throughput }}">
  <header class="panel-header">
    <h2>Fabric throughput (last 10 min)</h2>
    <span class="panel-key">spine layer (solid blue) · leaf uplinks (dashed green) · leaf downlinks (dotted yellow)</span>
  </header>
  <div id="throughput-chart"></div>
</section>

<section class="panel" id="top-talkers-panel"
         data-influx-url="{{ influx_public_url }}"
         data-influx-token="{{ influx_token }}"
         data-poll-ms="{{ poll.top_talkers }}">
  <header class="panel-header">
    <h2>Top-N talkers (last 5 min)</h2>
    <span class="panel-badge">⚡ Processing Engine: <span id="talkers-latency">–</span> ms</span>
  </header>
  <div id="top-talkers-content">loading…</div>
</section>

<section class="panel" id="search-panel"
         data-influx-url="{{ influx_public_url }}"
         data-influx-db="{{ influx_db }}"
         data-influx-token="{{ influx_token }}">
  <header class="panel-header">
    <h2>Source IP search</h2>
    <span class="panel-badge">⚡ DVC SQL: <span id="search-latency">–</span> ms</span>
  </header>
  <input id="src-ip-search" type="text" placeholder="search by src_ip prefix… e.g. 10.4"
         autocomplete="off" />
  <div id="search-results"></div>
</section>

<section class="panel" id="detail-panel"
         data-influx-url="{{ influx_public_url }}"
         data-influx-token="{{ influx_token }}">
  <header class="panel-header">
    <h2>Selected IP detail</h2>
    <span class="panel-badge">⚡ Processing Engine: <span id="detail-latency">–</span> ms</span>
  </header>
  <div id="detail-content" class="detail-empty">
    Select a source IP from the search above to drill in.
  </div>
</section>

<div id="anomalies"
     hx-get="/partials/anomalies"
     hx-trigger="load, every {{ poll.anomalies }}ms"
     hx-swap="innerHTML"></div>
{% endblock %}
```

`ui/templates/partials/_fabric_state.html`:

```html
<div class="banner banner-{{ status }}">
  <span class="banner-label">FABRIC STATE</span>
  <span class="banner-value">{{ status | upper }}</span>
  {% if row %}
  <span class="banner-detail">
    spines {{ row.spines_up }}/8 · leaves {{ row.leaves_up }}/16 · BGP {{ row.bgp_up }}/128 up
  </span>
  {% endif %}
</div>
```

`ui/templates/partials/_kpi_row.html`:

```html
<div class="kpi-row">
  <div class="kpi-tile">
    <div class="kpi-label">Ingress</div>
    <div class="kpi-value">{{ "%.2f"|format(ingress_tbps) }} Tbps</div>
    <div class="kpi-sub">spine layer, last 60 s</div>
  </div>
  <div class="kpi-tile">
    <div class="kpi-label">Egress</div>
    <div class="kpi-value">{{ "%.2f"|format(egress_tbps) }} Tbps</div>
    <div class="kpi-sub">spine layer, last 60 s</div>
  </div>
  <div class="kpi-tile">
    <div class="kpi-label">95p latency</div>
    <div class="kpi-value">{{ "%.1f"|format(p95_latency_us) }} µs</div>
    <div class="kpi-sub">switch-pair RTT</div>
  </div>
  <div class="kpi-tile">
    <div class="kpi-label">Flows / min</div>
    <div class="kpi-value">{{ "{:,}".format(flows_last_minute) }}</div>
    <div class="kpi-sub">last 60 s</div>
  </div>
  <div class="kpi-tile">
    <div class="kpi-label">ECN-marked</div>
    <div class="kpi-value">{{ "%.3f"|format(ecn_pct) }}%</div>
    <div class="kpi-sub">last 60 s</div>
  </div>
</div>
```

`ui/templates/partials/_anomalies.html`:

```html
<section class="panel">
  <header class="panel-header"><h2>Active anomalies</h2></header>
  <table class="anomalies">
    <thead><tr><th>Time</th><th>Kind</th><th>Severity</th><th>Switch</th><th>Interface</th><th>Reason</th><th>Value</th></tr></thead>
    <tbody>
      {% for a in anomalies %}
      <tr class="sev-{{ a.severity }}">
        <td>{{ a.time }}</td>
        <td>{{ a.kind }}</td>
        <td>{{ a.severity }}</td>
        <td>{{ a.switch }}</td>
        <td>{{ a.interface }}</td>
        <td>{{ a.reason }}</td>
        <td>{{ "%.3f"|format(a.value or 0.0) }}</td>
      </tr>
      {% else %}
      <tr><td colspan="7" class="empty">fabric is healthy — no active anomalies</td></tr>
      {% endfor %}
    </tbody>
  </table>
</section>
```

- [ ] **Step 4: Verify imports**

```bash
INFLUXDB3_TOKEN=test-only python -c "from ui.app import app; print('routes:', len(app.routes))"
# Expected: routes count ≥ 6
```

- [ ] **Step 5: Commit**

```bash
git add ui/app.py ui/templates/base.html ui/templates/overview.html ui/templates/partials/
git commit -m "feat(ui): FastAPI + templates (banner, KPIs, throughput, anomalies)"
```

### Task 22: Vendored JS/CSS + app.js + app.css + UI Dockerfile

**Files:**
- Copy from bess: `ui/static/htmx.min.js`, `uplot.min.js`, `uplot.min.css`
- Create: `ui/static/app.js`, `ui/static/app.css`
- Create: `ui/Dockerfile`

- [ ] **Step 1: Copy vendored assets**

```bash
cp /Users/pauldix/codez/reference_architectures/influxdb3-ref-bess/ui/static/htmx.min.js \
   /Users/pauldix/codez/reference_architectures/influxdb3-ref-network-telemetry/ui/static/
cp /Users/pauldix/codez/reference_architectures/influxdb3-ref-bess/ui/static/uplot.min.js \
   /Users/pauldix/codez/reference_architectures/influxdb3-ref-network-telemetry/ui/static/
cp /Users/pauldix/codez/reference_architectures/influxdb3-ref-bess/ui/static/uplot.min.css \
   /Users/pauldix/codez/reference_architectures/influxdb3-ref-network-telemetry/ui/static/
```

- [ ] **Step 2: Create `ui/static/app.js`**

The three direct-fetch patterns: top-talkers (request plugin), search/typeahead (direct SQL), src_ip detail (request plugin on selection).

```javascript
// Network-telemetry dashboard JS:
//   - Throughput chart (FastAPI /partials/throughput JSON, uPlot)
//   - Top-N talkers panel — direct fetch to request_top_talkers plugin
//   - Source IP search — direct SQL fetch to query node, DVC-backed
//   - Selected IP detail — direct fetch to request_src_ip_detail on selection

(function () {
  'use strict';

  function el(tag, attrs, children) {
    var e = document.createElement(tag);
    if (attrs) for (var k in attrs) {
      if (k === 'class') e.className = attrs[k];
      else if (k === 'text') e.textContent = attrs[k];
      else e.setAttribute(k, attrs[k]);
    }
    if (children) children.forEach(function (c) { e.appendChild(c); });
    return e;
  }

  // Bucket parsing: handles ns/ms/s integer strings + ISO datetimes.
  function bucketToSeconds(bucket) {
    if (bucket == null) return NaN;
    if (typeof bucket === 'number') return bucket / 1000;
    var s = String(bucket);
    if (/^\d{16,}$/.test(s)) return Number(s) / 1e9;
    if (/^\d{13}$/.test(s)) return Number(s) / 1e3;
    if (/^\d{10}$/.test(s)) return Number(s);
    var t = Date.parse(s);
    return isFinite(t) ? t / 1000 : NaN;
  }

  // ──────────────────────────────────────────────────────────────────
  // Throughput chart (FastAPI partial)
  // ──────────────────────────────────────────────────────────────────

  function buildThroughputChart() {
    var container = document.getElementById('throughput-chart');
    if (!container) return null;
    return new uPlot({
      width: container.clientWidth || 800,
      height: 220,
      legend: {show: false},
      series: [
        {},
        {label: 'Spine', stroke: '#5ec1ff', width: 2},
        {label: 'Leaf uplinks', stroke: '#a3e635', width: 2, dash: [6, 4]},
        {label: 'Leaf downlinks', stroke: '#fbbf24', width: 2, dash: [2, 3]},
      ],
      axes: [
        {scale: 'x'},
        {scale: 'y', values: function (_, t) {
          return t.map(function (v) {
            if (v >= 1e12) return (v / 1e12).toFixed(1) + ' Tbps';
            if (v >= 1e9)  return (v / 1e9).toFixed(0) + ' Gbps';
            if (v >= 1e6)  return (v / 1e6).toFixed(0) + ' Mbps';
            return v.toFixed(0) + ' bps';
          });
        }},
      ],
    }, [[], [], [], []], container);
  }

  function pollThroughput() {
    var panel = document.getElementById('throughput-panel');
    if (!panel) return;
    var pollMs = parseInt(panel.dataset.throughputPollMs, 10) || 10000;
    var chart = buildThroughputChart();

    function fetchOnce() {
      fetch('/partials/throughput').then(function (r) { return r.json(); })
        .then(function (payload) {
          if (!chart) return;
          var byLayer = {spine: {x: [], y: []}, leaf_uplink: {x: [], y: []}, leaf_downlink: {x: [], y: []}};
          (payload.rows || []).forEach(function (r) {
            var layer = r.layer;
            if (!byLayer[layer]) return;
            var ts = bucketToSeconds(r.bucket);
            if (!isFinite(ts)) return;
            byLayer[layer].x.push(ts);
            byLayer[layer].y.push(Number(r.bps) || 0);
          });
          // Use spine x-axis as authoritative
          var xs = byLayer.spine.x;
          chart.setData([xs, byLayer.spine.y, byLayer.leaf_uplink.y, byLayer.leaf_downlink.y]);
        })
        .catch(function (e) { console.error('throughput fetch failed', e); });
    }
    fetchOnce();
    setInterval(fetchOnce, pollMs);
  }

  // ──────────────────────────────────────────────────────────────────
  // Top-N talkers (request plugin, direct fetch + latency badge)
  // ──────────────────────────────────────────────────────────────────

  function pollTopTalkers() {
    var panel = document.getElementById('top-talkers-panel');
    if (!panel) return;
    var url = panel.dataset.influxUrl + '/api/v3/engine/top_talkers';
    var token = panel.dataset.influxToken;
    var pollMs = parseInt(panel.dataset.pollMs, 10) || 5000;

    function fetchOnce() {
      var t0 = performance.now();
      fetch(url, {
        method: 'POST',
        headers: {'Authorization': 'Bearer ' + token, 'Content-Type': 'application/json'},
        body: JSON.stringify({window_minutes: 5, limit: 10}),
      })
        .then(function (r) { return r.json(); })
        .then(function (raw) {
          var ms = Math.round(performance.now() - t0);
          var label = document.getElementById('talkers-latency');
          if (label) label.textContent = ms;
          var data = (raw && raw.body && raw.body.talkers) ? raw.body : raw;
          renderTopTalkers(data.talkers || []);
        })
        .catch(function (e) {
          var label = document.getElementById('talkers-latency');
          if (label) label.textContent = 'error';
          console.error('top_talkers fetch failed', e);
        });
    }
    fetchOnce();
    setInterval(fetchOnce, pollMs);
  }

  function renderTopTalkers(talkers) {
    var c = document.getElementById('top-talkers-content');
    if (!c) return;
    if (talkers.length === 0) {
      c.innerHTML = '<div class="empty">no flow records in window</div>';
      return;
    }
    var t = el('table', {class: 'talkers'});
    t.appendChild(el('thead', null, [el('tr', null, [
      el('th', {text: 'src_ip'}), el('th', {text: 'bytes'}),
      el('th', {text: 'packets'}), el('th', {text: 'top dst'}),
    ])]));
    var tb = el('tbody');
    talkers.forEach(function (row) {
      tb.appendChild(el('tr', null, [
        el('td', {text: row.src_ip}),
        el('td', {text: formatBytes(row.bytes)}),
        el('td', {text: row.packets.toLocaleString()}),
        el('td', {text: row.top_dst}),
      ]));
    });
    t.appendChild(tb);
    c.innerHTML = '';
    c.appendChild(t);
  }

  function formatBytes(b) {
    if (b >= 1e12) return (b / 1e12).toFixed(2) + ' TB';
    if (b >= 1e9) return (b / 1e9).toFixed(2) + ' GB';
    if (b >= 1e6) return (b / 1e6).toFixed(2) + ' MB';
    if (b >= 1e3) return (b / 1e3).toFixed(2) + ' kB';
    return b + ' B';
  }

  // ──────────────────────────────────────────────────────────────────
  // Source IP search — direct SQL to query node (DVC TVF), latency badge
  // ──────────────────────────────────────────────────────────────────

  function setupSearch() {
    var panel = document.getElementById('search-panel');
    var input = document.getElementById('src-ip-search');
    var results = document.getElementById('search-results');
    if (!panel || !input || !results) return;
    var url = panel.dataset.influxUrl + '/api/v3/query_sql';
    var db = panel.dataset.influxDb;
    var token = panel.dataset.influxToken;
    var debounceTimer = null;

    input.addEventListener('input', function () {
      var prefix = input.value.trim();
      if (debounceTimer) clearTimeout(debounceTimer);
      if (prefix.length < 1) {
        results.innerHTML = '';
        return;
      }
      debounceTimer = setTimeout(function () { runSearch(prefix); }, 200);
    });

    function runSearch(prefix) {
      // Escape: only allow digits, dots, colons, hex (for IPv6 prefixes if ever)
      var safe = prefix.replace(/[^0-9.a-fA-F:]/g, '');
      if (!safe) return;
      var sql = "SELECT src_ip FROM distinct_cache('flow_records', 'src_ip_distinct') " +
                "WHERE src_ip LIKE '" + safe + "%' LIMIT 20";
      var t0 = performance.now();
      fetch(url, {
        method: 'POST',
        headers: {'Authorization': 'Bearer ' + token, 'Content-Type': 'application/json'},
        body: JSON.stringify({db: db, q: sql, format: 'json'}),
      })
        .then(function (r) { return r.json(); })
        .then(function (rows) {
          var ms = Math.round(performance.now() - t0);
          var label = document.getElementById('search-latency');
          if (label) label.textContent = ms;
          renderSearchResults(rows || []);
        })
        .catch(function (e) {
          console.error('search failed', e);
        });
    }

    function renderSearchResults(rows) {
      results.innerHTML = '';
      if (!rows || rows.length === 0) {
        results.innerHTML = '<div class="empty">no matches</div>';
        return;
      }
      rows.forEach(function (row) {
        var b = el('button', {class: 'search-result', text: row.src_ip});
        b.addEventListener('click', function () {
          loadDetail(row.src_ip);
        });
        results.appendChild(b);
      });
    }
  }

  // ──────────────────────────────────────────────────────────────────
  // Selected IP detail — direct fetch to src_ip_detail plugin
  // ──────────────────────────────────────────────────────────────────

  function loadDetail(srcIp) {
    var panel = document.getElementById('detail-panel');
    var content = document.getElementById('detail-content');
    if (!panel || !content) return;
    var url = panel.dataset.influxUrl + '/api/v3/engine/src_ip_detail';
    var token = panel.dataset.influxToken;
    content.textContent = 'loading…';
    var t0 = performance.now();
    fetch(url, {
      method: 'POST',
      headers: {'Authorization': 'Bearer ' + token, 'Content-Type': 'application/json'},
      body: JSON.stringify({src_ip: srcIp}),
    })
      .then(function (r) { return r.json(); })
      .then(function (raw) {
        var ms = Math.round(performance.now() - t0);
        var label = document.getElementById('detail-latency');
        if (label) label.textContent = ms;
        var data = (raw && raw.body && raw.body.src_ip) ? raw.body : raw;
        renderDetail(data);
      })
      .catch(function (e) {
        content.textContent = 'error: ' + e.message;
      });
  }

  function renderDetail(data) {
    var c = document.getElementById('detail-content');
    if (!c) return;
    c.innerHTML = '';
    c.appendChild(el('div', {class: 'detail-header'}, [
      el('strong', {text: data.src_ip}),
      el('span', {class: 'detail-ecn',
                  text: 'ECN-marked: ' + (data.ecn_pct * 100).toFixed(3) + '%'}),
    ]));

    // Sparkline
    var sparkDiv = el('div', {class: 'detail-sparkline'});
    c.appendChild(sparkDiv);
    var xs = [], ys = [];
    (data.sparkline || []).forEach(function (s) {
      var ts = bucketToSeconds(s.bucket);
      if (!isFinite(ts)) return;
      xs.push(ts);
      ys.push(Number(s.bytes) || 0);
    });
    if (xs.length > 0) {
      new uPlot({
        width: sparkDiv.clientWidth || 600,
        height: 80,
        legend: {show: false},
        series: [{}, {label: 'bytes/min', stroke: '#5ec1ff', width: 2}],
        axes: [{scale: 'x'}, {scale: 'y',
          values: function (_, t) { return t.map(function (v) { return formatBytes(v); }); }}],
      }, [xs, ys], sparkDiv);
    }

    // Top destinations
    var dstTable = el('table', {class: 'detail-table'});
    dstTable.appendChild(el('thead', null, [el('tr', null, [
      el('th', {text: 'top dst'}), el('th', {text: 'bytes'})])]));
    var dtb = el('tbody');
    (data.top_destinations || []).forEach(function (d) {
      dtb.appendChild(el('tr', null, [
        el('td', {text: d.dst_ip}), el('td', {text: formatBytes(d.bytes)}),
      ]));
    });
    dstTable.appendChild(dtb);
    c.appendChild(dstTable);

    // Leaf distribution
    var leafTable = el('table', {class: 'detail-table'});
    leafTable.appendChild(el('thead', null, [el('tr', null, [
      el('th', {text: 'dst leaf'}), el('th', {text: 'bytes'})])]));
    var ltb = el('tbody');
    (data.leaf_distribution || []).forEach(function (l) {
      ltb.appendChild(el('tr', null, [
        el('td', {text: l.dst_switch}), el('td', {text: formatBytes(l.bytes)}),
      ]));
    });
    leafTable.appendChild(ltb);
    c.appendChild(leafTable);
  }

  document.addEventListener('DOMContentLoaded', function () {
    pollThroughput();
    pollTopTalkers();
    setupSearch();
  });
})();
```

- [ ] **Step 3: Create `ui/static/app.css`**

```css
:root {
  --bg: #0d1117;
  --panel-bg: #161b22;
  --panel-border: #30363d;
  --text: #c9d1d9;
  --text-dim: #8b949e;
  --accent: #58a6ff;
  --ok: #3fb950;
  --warn: #d29922;
  --down: #f85149;
}

* { box-sizing: border-box; }
body {
  font-family: -apple-system, BlinkMacSystemFont, "Segoe UI", system-ui, sans-serif;
  background: var(--bg); color: var(--text); margin: 0; padding: 0;
}
.topbar { padding: 12px 24px; border-bottom: 1px solid var(--panel-border); display: flex; justify-content: space-between; align-items: center; }
.brand-mark { font-weight: 600; font-size: 1.1em; color: var(--accent); }
.brand-tag { color: var(--text-dim); margin-left: 12px; font-size: 0.9em; }
main { padding: 16px 24px; display: flex; flex-direction: column; gap: 16px; }

.banner { padding: 14px 20px; border-radius: 8px; display: flex; align-items: baseline; gap: 16px;
          border: 1px solid var(--panel-border); background: var(--panel-bg); }
.banner-label { color: var(--text-dim); font-size: 0.85em; letter-spacing: 0.08em; }
.banner-value { font-size: 1.6em; font-weight: 700; }
.banner-detail { color: var(--text-dim); font-size: 0.9em; }
.banner-healthy .banner-value { color: var(--ok); }
.banner-degraded .banner-value { color: var(--warn); }
.banner-down .banner-value { color: var(--down); }
.banner-unknown .banner-value { color: var(--text-dim); }

.kpi-row { display: grid; grid-template-columns: repeat(5, 1fr); gap: 12px; }
.kpi-tile { background: var(--panel-bg); border: 1px solid var(--panel-border); border-radius: 8px; padding: 14px 16px; }
.kpi-label { color: var(--text-dim); font-size: 0.85em; }
.kpi-value { font-size: 1.6em; font-weight: 700; margin-top: 4px; }
.kpi-sub { color: var(--text-dim); font-size: 0.8em; margin-top: 4px; }

.panel { background: var(--panel-bg); border: 1px solid var(--panel-border); border-radius: 8px; padding: 14px 16px; }
.panel-header { display: flex; justify-content: space-between; align-items: baseline; gap: 8px; }
.panel-header h2 { margin: 0 0 8px 0; font-size: 1.05em; font-weight: 600; }
.panel-key { color: var(--text-dim); font-size: 0.8em; }
.panel-badge { font-size: 0.85em; color: var(--accent); background: rgba(88, 166, 255, 0.08);
               padding: 4px 10px; border-radius: 6px; font-family: monospace; }

#throughput-chart { width: 100%; }

table.talkers, table.anomalies, table.detail-table { width: 100%; border-collapse: collapse; font-size: 0.9em; margin-top: 8px; }
table.talkers th, table.talkers td,
table.anomalies th, table.anomalies td,
table.detail-table th, table.detail-table td {
  text-align: left; padding: 6px 10px; border-bottom: 1px solid var(--panel-border);
}
table.anomalies tr.sev-critical td { color: var(--down); }
table.anomalies tr.sev-warning td { color: var(--warn); }
.empty { color: var(--text-dim); font-style: italic; padding: 8px 0; }

#src-ip-search { width: 100%; padding: 8px 12px; background: #0d1117; color: var(--text);
                  border: 1px solid var(--panel-border); border-radius: 6px;
                  font-family: monospace; font-size: 1em; }
#search-results { display: flex; flex-wrap: wrap; gap: 6px; margin-top: 8px; }
.search-result { background: rgba(88, 166, 255, 0.08); color: var(--accent);
                 border: 1px solid var(--panel-border); border-radius: 4px;
                 padding: 4px 10px; cursor: pointer; font-family: monospace; }
.search-result:hover { background: rgba(88, 166, 255, 0.20); }

.detail-empty { color: var(--text-dim); font-style: italic; }
.detail-header { display: flex; justify-content: space-between; margin-bottom: 8px; }
.detail-header strong { font-family: monospace; color: var(--accent); font-size: 1.1em; }
.detail-ecn { color: var(--text-dim); font-size: 0.9em; }
.detail-sparkline { margin: 8px 0; }
```

- [ ] **Step 4: Create `ui/Dockerfile`**

```dockerfile
FROM python:3.12-slim

WORKDIR /app

RUN pip install --no-cache-dir \
      "fastapi>=0.110" \
      "uvicorn[standard]>=0.27" \
      "jinja2>=3.1" \
      "httpx>=0.27"

COPY ui /app/ui
ENV PYTHONPATH=/app

EXPOSE 8080
CMD ["uvicorn", "ui.app:app", "--host", "0.0.0.0", "--port", "8080"]
```

- [ ] **Step 5: Build and bring up**

```bash
docker build -t nt-ui-check -f ui/Dockerfile .
docker rmi nt-ui-check
docker compose up -d --build ui
sleep 5
curl -s http://localhost:8080/ | head -5
# Expected: HTML containing "Network Telemetry"
```

- [ ] **Step 6: Commit**

```bash
git add ui/static/ ui/Dockerfile
git commit -m "feat(ui): app.js + app.css + vendored htmx/uplot + Dockerfile"
```

---

## Phase 9 — Tests tier 2 & 3, CI

### Task 23: Tier 2 scenario tests + conftest

**Files:**
- Create: `tests/conftest.py`
- Create: `tests/test_scenarios/__init__.py`
- Create: `tests/test_scenarios/test_congestion_hotspot.py`
- Create: `tests/test_scenarios/test_east_west_burst.py`

Tier-2 testing the full 5-node compose via testcontainers is the trickiest test surface in the portfolio so far. **Expect this to need iteration during real CI bring-up.** The plan is to ship a working harness skeleton + per-scenario assertions; when the harness needs fixes, iterate inline.

- [ ] **Step 1: Create `tests/conftest.py`**

```python
"""Shared pytest fixtures.

Tier-2 scenario tests boot the multi-node compose via docker-compose
directly (not via the `testcontainers` library — multi-container
orchestration is easier with compose than wiring 5 individual
DockerContainer instances and their dependencies together).
"""

from __future__ import annotations

import os
import shutil
import subprocess
import time
from collections.abc import Iterator
from pathlib import Path

import httpx
import pytest


REPO_ROOT = Path(__file__).resolve().parents[1]
INFLUX_URL = "http://127.0.0.1:8181"


def _docker_available() -> bool:
    return shutil.which("docker") is not None


@pytest.fixture(scope="session")
def repo_root() -> Path:
    return REPO_ROOT


@pytest.fixture(scope="module")
def cluster_up() -> Iterator[None]:
    """Bring the 5-node cluster up via docker compose. Skip if Docker unavailable
    or INFLUXDB3_ENTERPRISE_EMAIL not set (CI sets a maintainer email)."""
    if not _docker_available():
        pytest.skip("Docker not available")
    if not os.environ.get("INFLUXDB3_ENTERPRISE_EMAIL"):
        pytest.skip("INFLUXDB3_ENTERPRISE_EMAIL not set (CI must set it)")

    subprocess.run(["docker", "compose", "up", "-d"],
                   cwd=REPO_ROOT, check=True, capture_output=True)
    try:
        deadline = time.time() + 240
        while time.time() < deadline:
            try:
                if httpx.get(f"{INFLUX_URL}/health", timeout=2).status_code in (200, 401):
                    break
            except Exception:
                pass
            time.sleep(2)
        else:
            pytest.fail("influxdb3-query never became reachable")
        # Give init container time to finish (it depends on all 5 server services
        # being healthy, then runs trigger registration).
        time.sleep(15)
        yield
    finally:
        subprocess.run(["docker", "compose", "down"],
                       cwd=REPO_ROOT, check=False, capture_output=True)


def _query(sql: str) -> list[dict]:
    token_path = REPO_ROOT / ".test-token"
    token = token_path.read_text().strip() if token_path.exists() else ""
    headers = {"Authorization": f"Bearer {token}"} if token else {}
    r = httpx.post(
        f"{INFLUX_URL}/api/v3/query_sql",
        json={"db": "nt", "q": sql, "format": "json"},
        headers=headers, timeout=10.0,
    )
    r.raise_for_status()
    return r.json() or []


@pytest.fixture
def influx_query():
    return _query
```

- [ ] **Step 2: Create test files**

```bash
touch /Users/pauldix/codez/reference_architectures/influxdb3-ref-network-telemetry/tests/test_scenarios/__init__.py
```

`tests/test_scenarios/test_congestion_hotspot.py`:

```python
"""Tier-2: congestion_hotspot triggers schedule_anomaly_detector to write a row."""

from __future__ import annotations

import subprocess
import time

import pytest

pytestmark = pytest.mark.scenario


def test_high_util_anomaly_fires(cluster_up, influx_query, repo_root):
    # Run the scenario via compose
    subprocess.run(
        ["docker", "compose", "--profile", "scenarios", "run", "--rm",
         "-e", "SCENARIO=congestion_hotspot", "scenarios"],
        cwd=repo_root, check=True, capture_output=True,
    )
    time.sleep(10)  # allow anomaly_detector's next 5s tick to fire
    rows = influx_query(
        "SELECT * FROM anomalies "
        "WHERE switch='leaf-07' AND interface='et-0/0/12' "
        "ORDER BY time DESC LIMIT 5"
    )
    assert any(r.get("kind") == "high_util" for r in rows), \
        f"expected at least one high_util anomaly; got: {rows}"
```

`tests/test_scenarios/test_east_west_burst.py`:

```python
"""Tier-2: east_west_burst populates the DVC and the detail plugin returns the IP."""

from __future__ import annotations

import json
import subprocess
import time

import httpx
import pytest

pytestmark = pytest.mark.scenario


def test_typeahead_finds_burst_source_via_dvc(cluster_up, influx_query, repo_root):
    subprocess.run(
        ["docker", "compose", "--profile", "scenarios", "run", "--rm",
         "-e", "SCENARIO=east_west_burst", "scenarios"],
        cwd=repo_root, check=True, capture_output=True,
    )
    time.sleep(2)
    # Typeahead via the DVC TVF
    rows = influx_query(
        "SELECT src_ip FROM distinct_cache('flow_records', 'src_ip_distinct') "
        "WHERE src_ip LIKE '10.4%' LIMIT 50"
    )
    src_ips = [r["src_ip"] for r in rows]
    assert "10.4.7.91" in src_ips


def test_src_ip_detail_returns_payload_for_burst_source(cluster_up, repo_root):
    # Read token from the validated volume
    res = subprocess.run(
        ["docker", "compose", "exec", "-T", "influxdb3-query",
         "cat", "/var/lib/influxdb3/.nt-token-plain"],
        cwd=repo_root, check=True, capture_output=True, text=True,
    )
    token = res.stdout.strip()
    r = httpx.post(
        "http://127.0.0.1:8181/api/v3/engine/src_ip_detail",
        json={"src_ip": "10.4.7.91"},
        headers={"Authorization": f"Bearer {token}"},
        timeout=10.0,
    )
    r.raise_for_status()
    body = r.json()
    if "body" in body and "src_ip" in body.get("body", {}):
        body = body["body"]
    assert body["src_ip"] == "10.4.7.91"
    assert "sparkline" in body
    assert "top_destinations" in body
```

- [ ] **Step 3: Run if Docker available**

```bash
pytest tests/test_scenarios -v -m scenario
```

If they fail, iterate on conftest until both pass. Common failures and fixes:
- Cluster doesn't reach healthy in 240s → increase the deadline, or check the influxdb3-init logs for trigger-registration errors.
- Trigger not registered → init.sh's `ensure_triggers` may fail silently if a CLI flag is wrong; check `docker compose logs influxdb3-init`.
- `nt-process` plugin write-back fails → check it can resolve `nt-ingest-1` (compose DNS) and that the token file is mounted.

- [ ] **Step 4: Commit**

```bash
git add tests/conftest.py tests/test_scenarios/
git commit -m "test(scenarios): tier-2 multi-node testcontainers tests"
```

### Task 24: Tier 3 smoke test + CI workflows

**Files:**
- Create: `tests/test_smoke.py`
- Create: `.github/workflows/{unit,scenarios,smoke,lint}.yml`
- Create: `FOR_MAINTAINERS.md`

- [ ] **Step 1: Create `tests/test_smoke.py`**

```python
"""Tier-3 smoke test: bring up the full 5-node cluster, verify plugins fire.

Requires: Docker, INFLUXDB3_ENTERPRISE_EMAIL set, license-validated
influxdb-data volume restored from CI artifact.
"""

from __future__ import annotations

import os
import shutil
import subprocess
import time
from pathlib import Path

import httpx
import pytest

pytestmark = pytest.mark.smoke

REPO_ROOT = Path(__file__).resolve().parents[1]
INFLUX_URL = "http://127.0.0.1:8181"
UI_URL = "http://127.0.0.1:8080"


@pytest.fixture(scope="module")
def stack_up():
    if shutil.which("docker") is None:
        pytest.skip("docker not available")
    if not os.environ.get("INFLUXDB3_ENTERPRISE_EMAIL"):
        pytest.skip("INFLUXDB3_ENTERPRISE_EMAIL not set")
    subprocess.run(["docker", "compose", "up", "-d"], cwd=REPO_ROOT, check=True, capture_output=True)
    try:
        deadline = time.time() + 300
        while time.time() < deadline:
            try:
                if httpx.get(f"{INFLUX_URL}/health", timeout=2).status_code in (200, 401):
                    break
            except Exception:
                pass
            time.sleep(2)
        else:
            pytest.fail("influxdb3-query /health never reached (license validation needed?)")
        time.sleep(30)  # let simulator + plugins do work
        yield
    finally:
        subprocess.run(["docker", "compose", "down"], cwd=REPO_ROOT, check=False, capture_output=True)


def _query(sql: str) -> list[dict]:
    r = httpx.post(
        f"{INFLUX_URL}/api/v3/query_sql",
        json={"db": "nt", "q": sql, "format": "json"},
        timeout=10.0,
    )
    r.raise_for_status()
    return r.json() or []


def test_simulator_writing_interface_counters(stack_up):
    rows = _query("SELECT COUNT(*) AS n FROM interface_counters WHERE time > now() - INTERVAL '1 minute'")
    assert int(rows[0]["n"]) > 0


def test_lvc_returns_bgp_sessions(stack_up):
    rows = _query("SELECT COUNT(*) AS n FROM last_cache('bgp_sessions', 'bgp_session_last')")
    assert int(rows[0]["n"]) >= 100  # 128 sessions, allow slack for boot


def test_dvc_has_src_ips(stack_up):
    rows = _query("SELECT COUNT(*) AS n FROM distinct_cache('flow_records', 'src_ip_distinct')")
    assert int(rows[0]["n"]) > 50  # simulator generates many distinct IPs


def test_fabric_health_being_written_by_schedule_plugin(stack_up):
    rows = _query("SELECT COUNT(*) AS n FROM fabric_health WHERE time > now() - INTERVAL '30 seconds'")
    # 5s cadence × 30s × 4 layers = up to 24 rows; allow slack
    assert int(rows[0]["n"]) >= 6


def test_top_talkers_endpoint_reachable(stack_up):
    r = httpx.post(f"{INFLUX_URL}/api/v3/engine/top_talkers",
                   json={"window_minutes": 5, "limit": 10}, timeout=10.0)
    assert r.status_code in (200, 401)


def test_ui_root_loads(stack_up):
    r = httpx.get(UI_URL, timeout=10.0)
    assert r.status_code == 200
    assert "Network Telemetry" in r.text or "fabric" in r.text.lower()
```

- [ ] **Step 2: Create CI workflows** (4 files)

`.github/workflows/unit.yml`:

```yaml
name: unit
on:
  push:
    branches: [main]
  pull_request:
jobs:
  unit:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      - uses: actions/setup-python@v5
        with: { python-version: "3.12" }
      - run: pip install -e ".[dev]"
      - run: pytest tests -q -m "not scenario and not smoke"
```

`.github/workflows/scenarios.yml`:

```yaml
name: scenarios
on:
  pull_request:
jobs:
  scenarios:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      - name: Restore license-validated influxdb-data volume
        uses: actions/download-artifact@v4
        with:
          name: influxdb-data-validated-nt
          path: ./.ci-volume
        continue-on-error: true
      - uses: actions/setup-python@v5
        with: { python-version: "3.12" }
      - run: pip install -e ".[dev]"
      - run: |
          if [ ! -d .ci-volume ]; then
            echo "no validated volume artifact; skipping"; exit 0
          fi
          docker volume create influxdb3-ref-network-telemetry_influxdb-data
          docker run --rm \
            -v influxdb3-ref-network-telemetry_influxdb-data:/data \
            -v "$PWD/.ci-volume":/src busybox sh -c "cp -r /src/. /data/"
          export INFLUXDB3_ENTERPRISE_EMAIL=ci+nt@example.com
          pytest tests/test_scenarios -q -m scenario
```

`.github/workflows/smoke.yml`:

```yaml
name: smoke
on:
  push:
    branches: [main]
  schedule:
    - cron: "0 7 * * *"
jobs:
  smoke:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      - name: Restore license-validated influxdb-data volume
        uses: actions/download-artifact@v4
        with:
          name: influxdb-data-validated-nt
          path: ./.ci-volume
        continue-on-error: true
      - uses: actions/setup-python@v5
        with: { python-version: "3.12" }
      - run: pip install -e ".[dev]"
      - run: |
          if [ ! -d .ci-volume ]; then
            echo "no validated volume artifact; skipping"; exit 0
          fi
          docker volume create influxdb3-ref-network-telemetry_influxdb-data
          docker run --rm \
            -v influxdb3-ref-network-telemetry_influxdb-data:/data \
            -v "$PWD/.ci-volume":/src busybox sh -c "cp -r /src/. /data/"
          export INFLUXDB3_ENTERPRISE_EMAIL=ci+nt@example.com
          pytest tests/test_smoke.py -q -m smoke
```

`.github/workflows/lint.yml`:

```yaml
name: lint
on:
  push:
  pull_request:
jobs:
  lint:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      - uses: actions/setup-python@v5
        with: { python-version: "3.12" }
      - run: pip install ruff
      - run: ruff check .
      - run: ruff format --check .
```

- [ ] **Step 3: Create `FOR_MAINTAINERS.md`**

```markdown
# For maintainers

## Refreshing the license-validated volume artifact

Tier 2 (scenarios) and Tier 3 (smoke) both require a license-validated
`influxdb-data` volume so CI doesn't have to click a validation email
on every run. **Multi-node twist:** the validated volume must contain
the catalog state from a cluster where all five InfluxDB nodes have
booted at least once (so the catalog is fully populated and consistent).

1. Locally:
   ```
   make clean && INFLUXDB3_ENTERPRISE_EMAIL=ci+nt@example.com make up
   ```
2. Click the validation link in the email.
3. Wait until `make ps` shows all five InfluxDB nodes (`nt-ingest-1`,
   `nt-ingest-2`, `nt-query`, `nt-compact`, `nt-process`) as `healthy`.
4. Wait additional 60 s so the schedule plugins have had time to write
   `fabric_health` rows (proves the plugin writeback path works).
5. `make down` (preserves the volume).
6. Export:
   ```
   docker run --rm -v influxdb3-ref-network-telemetry_influxdb-data:/data -v "$PWD":/out busybox \
     sh -c "tar czf /out/influxdb-data.tar.gz -C /data ."
   ```
7. Upload `influxdb-data.tar.gz` as a GitHub Actions artifact named
   `influxdb-data-validated-nt`.

Refresh cadence: monthly, or when license terms change.

## Common gotchas

- 6-field cron OR `every:5s` schedule format — CONVENTIONS in the meta
  repo lists when to use which.
- LVC reads via `last_cache(table, cache_name)` TVF.
- DVC reads via `distinct_cache(table, cache_name)` TVF.
- Plugin write-back via httpx through ingest nodes (NOT
  `LineBuilder` / `influxdb3_local.write()`).
- Tables created via `/api/v3/configure/table` (or CLI), not sentinel
  rows. No `__init` filter needed in queries.
```

- [ ] **Step 4: Commit**

```bash
git add tests/test_smoke.py .github/workflows/ FOR_MAINTAINERS.md
git commit -m "test+ci: tier-3 smoke, four workflows, maintainer doc"
```

---

## Phase 10 — Documentation

### Task 25: `CLI_EXAMPLES.md`, `SCENARIOS.md`, `ARCHITECTURE.md`

**Files:**
- Create: `CLI_EXAMPLES.md`, `SCENARIOS.md`, `ARCHITECTURE.md`

- [ ] **Step 1: Create `CLI_EXAMPLES.md`**

````markdown
# CLI Examples

Curated `influxdb3` commands and `curl` snippets for the `nt` cluster. Run via:

```
make cli-example name=<example>
```

(or copy the SQL into `make query sql='...'`)

## list-databases

```bash
influxdb3 show database --token "$TOKEN"
```

## list-tables

```bash
influxdb3 query --database nt --token "$TOKEN" "SHOW TABLES"
```

## show-triggers

```bash
influxdb3 show trigger --database nt --token "$TOKEN"
```

## fabric-state

Latest fabric_health rollup row:

```bash
influxdb3 query --database nt --token "$TOKEN" "SELECT * FROM fabric_health WHERE layer='plant' ORDER BY time DESC LIMIT 1"
```

## bgp-up-count

LVC-served BGP up-count:

```bash
influxdb3 query --database nt --token "$TOKEN" "SELECT COUNT(*) AS bgp_up FROM last_cache('bgp_sessions','bgp_session_last') WHERE state='established'"
```

## distinct-src-ips-prefix

DVC-served typeahead — prefix search on src_ip:

```bash
influxdb3 query --database nt --token "$TOKEN" "SELECT src_ip FROM distinct_cache('flow_records','src_ip_distinct') WHERE src_ip LIKE '10.4%' LIMIT 20"
```

## top-talkers-via-plugin

Direct call to the request plugin running on the query node:

```bash
curl -s -H "Authorization: Bearer $TOKEN" -X POST http://localhost:8181/api/v3/engine/top_talkers \
     -H "Content-Type: application/json" \
     -d '{"window_minutes":5,"limit":10}' | python3 -m json.tool | head -30
```

## src-ip-detail-via-plugin

```bash
curl -s -H "Authorization: Bearer $TOKEN" -X POST http://localhost:8181/api/v3/engine/src_ip_detail \
     -H "Content-Type: application/json" \
     -d '{"src_ip":"10.4.7.91"}' | python3 -m json.tool | head -40
```

## active-anomalies

```bash
influxdb3 query --database nt --token "$TOKEN" "SELECT time, kind, severity, switch, interface, reason, value FROM anomalies WHERE time > now() - INTERVAL '5 minutes' AND severity != 'info' ORDER BY time DESC LIMIT 20"
```

## retention-on-fabric-health

Confirm the 24h retention is set on `fabric_health` (not on other tables):

```bash
influxdb3 show table fabric_health --database nt --token "$TOKEN"
```
````

- [ ] **Step 2: Create `SCENARIOS.md`**

```markdown
# Scenarios

Two scenarios. Each is a Python script under `simulator/scenarios/`.
Run with:

```
make scenario name=<scenario>
```

## congestion_hotspot

**One-line:** `leaf-07 / et-0/0/12` climbs to 94% utilization; anomaly detector fires; banner flips DEGRADED.

**Steps:**
1. Wait 10 s baseline at ~30% on the target interface.
2. Ramp utilization 30% → 94% over 10 s.
3. Hold at 94% for 30 s. The next `schedule_anomaly_detector` run within 5 s of crossing the 30-s sustain window writes a `kind=high_util` row.
4. Recovery: drop back to 30%. The detector writes a `severity=info, reason="cleared"` row on its next tick.

**Plugin exercised:** `schedule_anomaly_detector` (windowed-detection pattern).

**What to watch in the dashboard:**
- Banner flips RUNNING → DEGRADED.
- Active anomalies panel populates with the high_util row.
- Throughput chart shows leaf_uplink layer climbing.

**Verify with SQL:**

```sql
SELECT time, kind, severity, switch, interface, reason, value
FROM anomalies
WHERE switch='leaf-07' AND interface='et-0/0/12'
ORDER BY time DESC LIMIT 5;
```

## east_west_burst

**One-line:** 10× traffic burst from `10.4.7.91`; typeahead finds it sub-ms; detail panel shows the spike.

**Steps:**
1. Wait 10 s baseline.
2. Inject ~200 extra flow records per second from `src_ip=10.4.7.91` for 30 s.
3. **No fabric-level anomaly fires** — single-source bursts don't necessarily breach thresholds. Banner stays HEALTHY.
4. The DVC has now picked up `10.4.7.91`; the user discovers the burst proactively via the source-IP search box.

**Plugin exercised:** `request_src_ip_detail` (composite-payload pattern, called by the user via the detail panel).

**What to watch in the dashboard:**
- Type "10.4" in the source-IP search box. The DVC SQL latency badge shows sub-ms; `10.4.7.91` appears in results.
- Click it. The detail panel populates: sparkline shows the spike, top destinations are concentrated on a few `dst_ip`, leaf distribution narrows.

**Verify with SQL:**

```sql
SELECT src_ip FROM distinct_cache('flow_records','src_ip_distinct') WHERE src_ip LIKE '10.4%' LIMIT 20;
```

(should include `10.4.7.91`)
```

- [ ] **Step 3: Create `ARCHITECTURE.md`** (long but mostly structural)

```markdown
# Architecture

Deep-dive companion to `README.md`. Read the README first for the
quickstart and headline story; read this for the multi-node compose
shape, plugin write-back path, schema rationale, and scaling notes.

## Table of contents

1. Domain model
2. Schema and per-table retention
3. Multi-node compose
4. Plugin write-back path (cross-node)
5. Three patterns for feeding a UI panel
6. Processing Engine triggers
7. Enterprise features used
8. Token bootstrap (cluster-wide)
9. Plugin conventions and gotchas
10. Security notes
11. Scaling to production
12. Extending the cluster

## 1. Domain model

A data-center Clos fabric: 8 spines, 16 leaves, 48 servers per leaf
rack (768 servers total, modeled only as flow `src_ip`/`dst_ip`).
Total ~1024 fabric interfaces, ~128 leaf↔spine BGP sessions, ~5,000
sampled flow records per second, 64 latency probe pairs.

Vocabulary: fabric, spine, leaf, ECMP, BGP, peer, prefix, flap, flow
record, sampled, top-N talkers, microburst, ECN-mark, PFC pause,
oversubscription, hotspot.

## 2. Schema and per-table retention

Six tables (see `influxdb/schema.md` for the detailed reference).

Tables are created **explicitly** at init time via the configure API
(`POST /api/v3/configure/table` or the `influxdb3 create table` CLI),
not via implicit creation from the first write. This means:

- Caches and triggers can reference tables immediately at init time
  without a sentinel-row workaround.
- Schemas (tag set, field types, retention) are declared up front
  rather than inferred.
- LVC and DVC reads don't have to filter out a `__init` sentinel row.

`fabric_health` is the only table with a retention period (24 hours),
demonstrating per-table retention. Other tables have no retention in
the demo. In production, typical retention values would be:

- `interface_counters`, `bgp_sessions`, `latency_probes`: 7-30 days
- `flow_records`: 30-90 days (regulatory)
- `fabric_health`, `anomalies`: 365 days (operational history)

## 3. Multi-node compose

Five InfluxDB 3 Enterprise nodes plus a one-shot token-bootstrap and
init container, plus simulator/UI/scenarios. All five InfluxDB nodes
mount the same `influxdb-data` named volume at `/var/lib/influxdb3`.
**Sharing the disk is what makes the cluster a cluster** — every node
sees the same object store and catalog, so writes from one ingest node
are immediately visible from the query and process nodes, and the
catalog (databases, tables, caches, triggers) stays consistent across
all nodes without explicit coordination.

| Node | Mode | Purpose |
|---|---|---|
| `nt-ingest-1`, `nt-ingest-2` | `ingest` | Accept writes; simulator round-robins per batch |
| `nt-query` | `query` | Serves UI partials, browser direct fetches, CLI; hosts request plugins |
| `nt-compact` | `compact` | Background compaction only |
| `nt-process` | `process,query` | Hosts schedule plugins; queries locally, writes back via httpx through an ingest node |

The process node uses the `process,query` mode combo so plugin code can
call `influxdb3_local.query()` against the local engine without
HTTP-hopping to another node for reads. (Setting `--plugin-dir`
implicitly adds `process` mode; explicitly setting `--mode query`
keeps the query engine available.)

## 4. Plugin write-back path

This is **a new convention** introduced by this repo (now codified in
the meta repo's `CONVENTIONS.md`).

A schedule plugin running on a process-only node has no obvious local
ingest target — the engine doesn't accept writes locally on a
non-ingest node. The plugin module-level code therefore loads the
admin token once at import time (from the shared volume) and uses
`httpx` to POST line protocol back through an ingest node's
`/api/v3/write_lp` endpoint. The shared `plugins/_writeback.py`
module factors this out: round-robin over the configured ingest URLs,
one fallback hop on connection error.

Configuration via env vars on the process node (set in
`docker-compose.yml`):

```
NT_INGEST_URLS=http://nt-ingest-1:8181,http://nt-ingest-2:8181
NT_DB=nt
NT_TOKEN_FILE=/var/lib/influxdb3/.nt-operator-token
```

`LineBuilder` is **not used** by the schedule plugins in this repo —
the cross-node write-back replaces it.

## 5. Three patterns for feeding a UI panel

This repo demonstrates all three ways to get data into the dashboard,
side-by-side, each with its own latency badge:

| Pattern | Where the call goes | Used for |
|---|---|---|
| **SQL via FastAPI** (Python proxy) | browser → `nt-ui:8080/partials/...` → `nt-query:8181/api/v3/query_sql` | Banner, KPIs, throughput chart, anomalies |
| **SQL from browser** (DVC TVF) | browser → `nt-query:8181/api/v3/query_sql` directly | Source-IP typeahead. Sub-ms badge teaches DVC speed. |
| **Request plugin from browser** (Processing Engine) | browser → `nt-query:8181/api/v3/engine/<name>` directly | Top-N talkers, source-IP detail. Composite payloads. |

When to pick which: SQL through FastAPI when the response is HTML
fragments (HTMX swaps); SQL direct from browser when the cache speed
is the headline (typeahead); request plugin when the response is a
composite shape that joins multiple queries' worth of data.

## 6. Processing Engine triggers

| Name | Type | Spec | Where it runs | Effect |
|---|---|---|---|---|
| `fabric_health` | Schedule | `every:5s` | process | Writes one row per layer to `fabric_health` |
| `anomaly_detector` | Schedule | `every:5s` | process | Detects and writes anomalies to `anomalies` |
| `top_talkers` | Request | `request:top_talkers` | query | Top src_ip aggregates |
| `src_ip_detail` | Request | `request:src_ip_detail` | query | Composite drill-down for one IP |

The repo uses `every:5s` exclusively for schedule triggers — short,
regular intervals don't need cron's time-of-day alignment, and `every:`
is more readable.

## 7. Enterprise features used

| Feature | Where |
|---|---|
| Multi-node ingest | 2 ingest nodes; simulator round-robins |
| Multi-node split (ingest/query/compact/process) | 5-node compose |
| Last Value Cache | `bgp_session_last`; powers banner BGP up-count |
| Distinct Value Cache | `src_ip_distinct`; powers typeahead with sub-ms badge |
| **Per-table retention** | `fabric_health` 24h. Exclusive to this repo in the portfolio. |
| **Schedule trigger via `every:` syntax** | Exclusive to this repo in the portfolio. |
| **Schedule plugin with cross-node write-back** | Both schedule plugins via `_writeback.py`. New convention. |
| Request trigger | top_talkers + src_ip_detail on query node |
| Custom UI | Three patterns side-by-side |

## 8. Token bootstrap (cluster-wide)

A single `token-bootstrap` compose service generates one offline admin
token at first boot, written to the shared volume. All five InfluxDB
nodes start with `--admin-token-file` pointing at the same path; the
simulator, UI, init, and process node read the same token from the
same volume. License validation also happens once per cluster.

## 9. Plugin conventions and gotchas

See [CONVENTIONS.md](https://github.com/influxdata/influxdb3-reference-architectures/blob/main/CONVENTIONS.md) in the meta repo. Highlights specific to this repo:

- `LineBuilder` is INJECTED — not used by this repo's schedule plugins (they use httpx).
- 6-field cron OR `every:` interval — this repo uses `every:5s`.
- LVC reads via `last_cache(table, cache_name)` TVF.
- DVC reads via `distinct_cache(table, cache_name)` TVF.
- Multiple unaliased `COUNT(*)` scalar subqueries don't compose under DataFusion.
- `date_bin()` returns ns-integer strings on the wire.
- Browser-facing endpoints need `INFLUX_PUBLIC_URL`.

## 10. Security notes

Demo simplifications, called out for production users:

- One admin token, shared by all services. Production should issue scoped tokens per service (read-only for UI, write-only for simulator, scoped for plugin write-back).
- The browser sees the admin token (passed in template context for the direct-fetch panels). Production should proxy through the UI backend or use a token-exchange flow.
- No TLS in compose. Production needs TLS between nodes and to clients.

## 11. Scaling to production

- **More ingest**: add ingest-3, ingest-4, etc. Simulator's round-robin scales without code change. Production would put them behind a load balancer.
- **Multi-query**: add a second query node for read scaling. Both serve the same SQL endpoints; UI hits whichever responds first or load-balanced.
- **Object store**: swap `file` for S3/GCS/Azure. No code changes; one env var per node.
- **Retention**: extend per-table retention to all tables per the production guidance in §2.
- **K8s**: the compose service shape maps 1:1 to a Helm chart per node-role. Not shipped here per portfolio policy.

## 12. Extending the cluster

To add a new schedule plugin:
1. Create `plugins/schedule_<name>.py` following the existing pattern; import `from _writeback import write_lines`.
2. Add a trigger registration in `init.sh`'s `ensure_triggers()`.
3. Add a unit test under `tests/test_plugins/`.
4. `make down && make up` — init.sh registers the new trigger on next boot.

To add a new request plugin:
1. Create `plugins/request_<name>.py`.
2. Add a trigger registration with `--trigger-spec request:<name>`.
3. Add a unit test.
4. The plugin is reachable at `/api/v3/engine/<name>` after restart.
```

- [ ] **Step 4: Commit**

```bash
git add CLI_EXAMPLES.md SCENARIOS.md ARCHITECTURE.md
git commit -m "docs: CLI_EXAMPLES + SCENARIOS + ARCHITECTURE (multi-node deep dive)"
```

### Task 26: Architecture diagram

**Files:**
- Create: `diagrams/architecture.mmd`, `diagrams/architecture.png`

- [ ] **Step 1: Mermaid source**

```bash
mkdir -p /Users/pauldix/codez/reference_architectures/influxdb3-ref-network-telemetry/diagrams
```

`diagrams/architecture.mmd`:

```
flowchart LR
  subgraph Simulator["Simulator"]
    SIM[Fabric tick<br/>~10k pts/s]
  end

  subgraph Cluster["InfluxDB 3 Enterprise cluster (shared influxdb-data volume)"]
    direction TB
    subgraph Ingest["Ingest tier"]
      ING1[nt-ingest-1]
      ING2[nt-ingest-2]
    end
    QRY[nt-query<br/>request plugins]
    CMP[nt-compact<br/>background]
    PRC[nt-process,query<br/>schedule plugins]
  end

  subgraph UI["UI"]
    UIB[nt-ui<br/>FastAPI]
    BRW[Browser]
  end

  SIM -- "round-robin writes" --> ING1
  SIM -- "round-robin writes" --> ING2

  PRC -- "httpx writeback" --> ING1
  PRC -- "httpx writeback (fallback)" --> ING2

  ING1 -.-> QRY
  ING2 -.-> QRY

  UIB -- "SQL via /partials/*" --> QRY
  BRW -- "direct SQL (DVC typeahead)" --> QRY
  BRW -- "POST /api/v3/engine/* (request plugins)" --> QRY
  BRW -- "HTMX partials" --> UIB
```

- [ ] **Step 2: Render to PNG**

```bash
npx -y @mermaid-js/mermaid-cli@latest -i diagrams/architecture.mmd -o diagrams/architecture.png \
    --backgroundColor '#0d1117' --theme dark
```

If mmdc fails (no Node available), render manually via mermaid.live and save the export.

- [ ] **Step 3: Commit**

```bash
git add diagrams/
git commit -m "docs: 5-node architecture diagram (mermaid + PNG)"
```

### Task 27: `scripts/demo.sh` + polished `README.md`

**Files:**
- Create: `scripts/demo.sh`
- Modify: `README.md`

The demo script narrates the multi-node story: 5-node startup, three teaching patterns, two scenarios. Mirror iiot's structure.

- [ ] **Step 1: Copy iiot's demo.sh as the starting template**

```bash
cp /Users/pauldix/codez/reference_architectures/influxdb3-ref-iiot/scripts/demo.sh \
   /Users/pauldix/codez/reference_architectures/influxdb3-ref-network-telemetry/scripts/demo.sh
chmod +x /Users/pauldix/codez/reference_architectures/influxdb3-ref-network-telemetry/scripts/demo.sh
```

- [ ] **Step 2: Apply mechanical replacements**

```bash
cd /Users/pauldix/codez/reference_architectures/influxdb3-ref-network-telemetry
# Container prefix and DB name
sed -i '' 's/iiot-/nt-/g; s/database iiot/database nt/g; s/db.:.iiot/db":"nt/g; s|/.iiot-token-plain|/.nt-token-plain|g' scripts/demo.sh
# Volume name
sed -i '' 's/influxdb3-ref-iiot_influxdb-data/influxdb3-ref-network-telemetry_influxdb-data/g' scripts/demo.sh
bash -n scripts/demo.sh
```

(Linux users: drop the `''` after `-i`.)

- [ ] **Step 3: Replace IIoT-specific narrative content**

The mechanical replacements handle names, but the demo's storyline is IIoT-flavored. Open `scripts/demo.sh` and apply these specific text replacements (all are unique strings — search-and-replace will hit each exactly once):

**Banner title:**
- Old: `banner "IIoT Reference Architecture — One-Shot Demo"`
- New: `banner "Network Telemetry Reference Architecture — One-Shot Demo (5-node cluster)"`

**"What this demo shows" paragraph** (after `echo "${BOLD}What this demo shows${RESET}"`):
- Replace the entire bess/iiot factory paragraph with this network-telemetry version:
  ```
  echo "  ${FG_TEXT}A complete reference architecture for using InfluxDB 3 Enterprise"
  echo "  in a clustered deployment to monitor a data-center Clos fabric: 8 spines,"
  echo "  16 leaves, ~1000 fabric interfaces, ~128 BGP sessions, ~5000 sampled flow"
  echo "  records/sec — total ~10,000 telemetry points/sec — across a 5-node InfluxDB"
  echo "  3 Enterprise cluster (2 ingest + 1 query + 1 compact + 1 process,query)."
  echo "  Two schedule plugins on the process node detect anomalies and roll up"
  echo "  fabric health every 5 seconds. Two request plugins on the query node"
  echo "  serve top-N talkers and source-IP drill-downs to the browser directly.${RESET}"
  ```

**"Headline Enterprise features"** — replace the four bullet points with:
1. Multi-node split (5 nodes, each role on its own service)
2. Schedule plugins with cross-node write-back (`every:5s`, httpx round-robin to ingest)
3. Per-table retention on `fabric_health` (24h)
4. Three teaching patterns (SQL via FastAPI, SQL from browser via DVC, request plugin from browser) — each with its own latency badge

**"Who's doing what"** actor list — keep the same color-coded format but update labels:
- `[ingest]` × 2 — `nt-ingest-1`, `nt-ingest-2` containers (writers)
- `[query]` — `nt-query` container (request plugins + UI partials)
- `[process]` — `nt-process` container (schedule plugins, write-back)
- `[ui]` — browser at localhost:8080
- `[sim]` — `nt-simulator` container
- `[script]` — this terminal

**"What's about to happen" numbered list** — replace 6 steps with these 7:
1. Bring up the 10-service stack (5 InfluxDB nodes + token-bootstrap + init + simulator + ui + scenarios)
2. Wait for **all five** InfluxDB nodes healthy
3. Open the dashboard
4. Inject the `congestion_hotspot` scenario; `schedule_anomaly_detector` writes a `high_util` anomaly within 5s of crossing the 30s sustain window
5. Query the `anomalies` table directly to confirm
6. Call the `top_talkers` request plugin via curl with timing
7. Read the LVC: `SELECT COUNT(*) FROM last_cache('bgp_sessions','bgp_session_last')`

**Wait-for-ready spinner block** — replace iiot's single `wait_for_influxdb3` spinner with a loop that waits for **all 5 nodes' healthchecks**:

```bash
for n in nt-ingest-1 nt-ingest-2 nt-query nt-compact nt-process; do
    spin_until "${n} healthy" \
        "docker compose ps --format json $n 2>/dev/null | grep -q '\"Health\":\"healthy\"'" 180 \
        || { fail "${n} never became healthy"; docker logs --tail 30 ${n} 2>&1; exit 1; }
done
spin_until "influxdb3-init complete" \
    "docker logs --tail 50 nt-influxdb3-init 2>&1 | grep -q 'initialization complete'" 60 \
    || { fail "init did not complete"; docker logs nt-influxdb3-init 2>&1; exit 1; }
```

**Scenario step** — replace `unplanned_downtime_cascade` with:

```bash
step "Firing scenario: congestion_hotspot"
note "Pushes leaf-07 / et-0/0/12 utilization to 94%, holds 30s, then recovers."
note "schedule_anomaly_detector (every:5s) detects sustained high util and writes an anomaly row."
note "Watch the UI — banner flips to DEGRADED 🟡; anomalies panel populates; throughput chart shows the leaf_uplink layer climbing."
info "Running the scenario (~70s — 10s baseline + 10s ramp + 30s hold + 5s recovery + buffer)."
cmd "make scenario name=congestion_hotspot"
docker compose --profile scenarios run --rm \
    -e SCENARIO=congestion_hotspot scenarios 2>&1 \
    | grep --line-buffered -E "scenario step|DONE" \
    | sed -u "s|^|${FG_GREY}│${RESET}  ${DIM}|; s|\$|${RESET}|" \
    || true
ok "Scenario complete"
sleep 8  # allow next anomaly_detector tick to fire
close_step
```

**Show-anomalies step** — replace the bess `alerts` query with:

```bash
step "Anomalies written by schedule_anomaly_detector"
note "These rows were written by the schedule plugin, NOT by the simulator."
note "Latency below is the wall-clock /api/v3/query_sql round-trip from this host."
cmd "SELECT time, kind, severity, switch, interface, reason, value FROM anomalies WHERE switch='leaf-07' ORDER BY time DESC LIMIT 5"
timed_query_table "SELECT time, kind, severity, switch, interface, reason, value FROM anomalies WHERE switch='leaf-07' ORDER BY time DESC LIMIT 5" \
    | sed "s|^|${FG_GREY}│${RESET}  |"
close_step
```

**Request-trigger step** — replace `pack_health` with `top_talkers`:

```bash
step "Calling the Request trigger (top_talkers — Python plugin on query node)"
note "POST /api/v3/engine/top_talkers — same call the browser's top-talkers panel makes."
cmd "curl -X POST http://localhost:8181/api/v3/engine/top_talkers -d '{\"window_minutes\":5,\"limit\":10}'"
timed_post_json "http://localhost:8181/api/v3/engine/top_talkers" \
    '{"window_minutes":5,"limit":10}' \
    | head -40 \
    | sed "s|^|${FG_GREY}│${RESET}  |"
close_step
```

You'll need to add a `timed_post_json` helper alongside the existing `timed_get_json`:

```bash
timed_post_json() {
    local url="$1" body="$2"
    local tok body_file
    tok=$(token)
    body_file=$(mktemp)
    local time_total
    time_total=$(curl -s -w '%{time_total}' \
        -o "$body_file" \
        -H "Authorization: Bearer ${tok}" \
        -H "Content-Type: application/json" \
        -X POST "$url" -d "$body")
    local ms
    ms=$(awk -v t="$time_total" 'BEGIN{printf "%.3f", t*1000}')
    python3 -m json.tool < "$body_file" 2>/dev/null || cat "$body_file"
    rm -f "$body_file"
    printf "${FG_TEXT}responded in ${BOLD}${FG_GREEN}%s ms${RESET}${FG_TEXT} ← same endpoint the browser's top-talkers panel hits${RESET}\n" "$ms"
}
```

**LVC step** — replace the bess `cell_last` with the network-telemetry `bgp_session_last`:

```bash
step "Last Value Cache — 128 BGP sessions, sub-millisecond"
note "Read via the table-valued function last_cache('bgp_sessions','bgp_session_last')."
cmd "SELECT COUNT(*) AS sessions_in_cache FROM last_cache('bgp_sessions','bgp_session_last')"
timed_query_table "SELECT COUNT(*) AS sessions_in_cache FROM last_cache('bgp_sessions','bgp_session_last')" \
    | sed "s|^|${FG_GREY}│${RESET}  |"
close_step
```

**Add a NEW step before the LVC step: typeahead via DVC SQL** (this is the dashboard's third teaching pattern and worth showing in the demo):

```bash
step "Source-IP typeahead — direct SQL against the DVC"
note "SELECT src_ip FROM distinct_cache('flow_records','src_ip_distinct') WHERE src_ip LIKE '10.4%' LIMIT 20"
note "Same query the browser's search box runs — DVC makes it sub-ms regardless of flow_records size."
timed_query_table "SELECT src_ip FROM distinct_cache('flow_records','src_ip_distinct') WHERE src_ip LIKE '10.4%' LIMIT 20" \
    | sed "s|^|${FG_GREY}│${RESET}  |"
close_step
```

**Next-things-to-try summary block** — replace iiot's tips with:

```bash
echo "    ${FG_MAGENTA}make scenario name=east_west_burst${RESET}      # 10x burst from 10.4.7.91 — try the typeahead"
echo "    ${FG_MAGENTA}make cli${RESET}                                 # shell into the QUERY node with TOKEN + iql"
echo "    ${FG_MAGENTA}make cli-example name=top-talkers-via-plugin${RESET}  # call top_talkers via curl"
echo "    ${FG_MAGENTA}make cli-example name=distinct-src-ips-prefix${RESET} # DVC typeahead from the CLI"
echo "    ${FG_MAGENTA}./.venv/bin/pytest tests/test_plugins -v${RESET}      # all plugin unit tests"
```

- [ ] **Step 4: Verify syntax**

```bash
bash -n scripts/demo.sh
```

Expected: no output, exit 0.

- [ ] **Step 3: Replace `README.md` with the polished version**

```markdown
# influxdb3-ref-network-telemetry

Reference architecture: **InfluxDB 3 Enterprise multi-node cluster monitoring a data-center fabric.**

5-node cluster (2 ingest + 1 query + 1 compact + 1 process,query), 8×16 Clos topology with ~1024 interfaces, 128 BGP sessions, ~5k flow records/sec — total ~10k pts/sec — runnable in three minutes via `docker compose`. Two schedule plugins on the process node detect anomalies and roll up fabric health every 5 seconds (writing back through an ingest node via httpx). Two request plugins on the query node serve top-N talkers and source-IP drill-downs to the browser directly. The dashboard demonstrates **all three patterns** for feeding a panel — SQL via FastAPI, SQL from browser via DVC, and request-plugin from browser — each with its own latency badge.

![architecture](diagrams/architecture.png)

## Quickstart

```bash
git clone https://github.com/influxdata/influxdb3-ref-network-telemetry.git
cd influxdb3-ref-network-telemetry
make up   # prompts for INFLUXDB3_ENTERPRISE_EMAIL on first run
# Click the validation link in the email
open http://localhost:8080
make scenario name=congestion_hotspot
make scenario name=east_west_burst
```

Or run the full scripted demo:

```bash
make demo
```

## What's in this repo

| Path | Purpose |
|------|---------|
| `simulator/` | Python simulator generating fabric telemetry; round-robins writes across two ingest nodes |
| `plugins/` | Four Processing Engine plugins (Python) + `_writeback.py` shared httpx writer for cross-node write-back |
| `ui/` | FastAPI + HTMX + uPlot dashboard with three teaching patterns |
| `influxdb/init.sh` | Bootstraps DB + 6 tables (via configure API) + LVC + 2 DVCs + 4 triggers |
| `docker-compose.yml` | 10-service stack: token-bootstrap + 5 InfluxDB nodes + init + simulator + ui + scenarios |
| `Makefile` | `up / down / clean / scenario / cli / test / demo` |
| `tests/` | Three tiers: unit (no Docker) / scenario (testcontainers) / smoke (full 5-node stack) |
| `ARCHITECTURE.md` | Multi-node compose, plugin write-back path, schema rationale, scaling notes |
| `SCENARIOS.md` | Per-scenario walkthroughs |
| `CLI_EXAMPLES.md` | Curated `influxdb3` CLI commands and `curl` snippets |

## Headline Enterprise features

### ⬢ Multi-node split — ingest / query / compact / process

Five-node cluster with each role on its own service in compose. The process node runs schedule plugins; the query node hosts request plugins; ingest is split across two nodes. The shared `influxdb-data` named volume across all five gives the cluster catalog and object-store consistency without explicit coordination.

### ⬢ Two schedule plugins on the process node — `every:5s`

Both `schedule_fabric_health` and `schedule_anomaly_detector` run every 5 seconds via the `every:` schedule format (this repo's preferred syntax for short intervals). Each plugin reads via `influxdb3_local.query()` (local), then writes back via httpx to an ingest node — see `plugins/_writeback.py` for the round-robin pattern.

### ⬢ Two request plugins on the query node — direct browser fetch

`request_top_talkers` and `request_src_ip_detail` are called by the browser directly, with **"served by Processing Engine: N ms" latency badges** on each panel.

### ⬢ Source-IP typeahead — DVC SQL from the browser

The search box runs `SELECT src_ip FROM distinct_cache('flow_records', 'src_ip_distinct') WHERE src_ip LIKE '...' LIMIT 20` directly from JavaScript against `/api/v3/query_sql`, with a sub-millisecond latency badge. **No Python wrapper between the browser and the cache.**

### ⬢ Per-table retention on `fabric_health` — 24 hours

Set at table-create time via the configure API. The `schedule_fabric_health` plugin writes ~17k rows/day at the 5s cadence; retention drops anything older. **Exclusive demo of per-table retention in the portfolio.**

> **⚡ Three teaching patterns side-by-side.** Watch the latency badges. Direct SQL via FastAPI runs at backend speed (10-50 ms typical, network-hop bound). Direct SQL from the browser via the DVC runs at cache speed (1-5 ms). Request plugins run at plugin speed (10-100 ms depending on payload composition). Pick the right pattern for the job — see `ARCHITECTURE.md` § "Three patterns for feeding a UI panel".

## Running the tests

```bash
make test            # tier 1 + tier 2 (skip smoke)
make test-unit       # tier 1 only (no Docker)
make test-scenarios  # tier 2 (testcontainers; multi-node)
make test-smoke      # tier 3 (real 5-node stack; ~5 min)
```

## Scaling to production

The 5-node compose is the smallest viable shape for the multi-node split. For larger deployments, see `ARCHITECTURE.md` § "Scaling to production" and the other portfolio repos.

## License

Apache 2.0 — see [LICENSE](LICENSE).
```

- [ ] **Step 4: Commit**

```bash
git add scripts/demo.sh README.md
git commit -m "docs: scripted demo + polished README"
```

---

## Phase 11 — Ship to GitHub + meta-repo CONVENTIONS amendments

### Task 28: Final lint + full integration run

The integration run actually executes the full demo end-to-end (cluster up → UI loads → both scenarios fire → all three teaching patterns return data → tear down). This is what makes the handoff worth your review time. **All `docker compose` / `make` calls in this task need `dangerouslyDisableSandbox: true`.**

- [ ] **Step 1: Run linters and commit fixes**

```bash
cd /Users/pauldix/codez/reference_architectures/influxdb3-ref-network-telemetry
source .venv/bin/activate
ruff check --fix . && ruff format .
ruff check . && ruff format --check .
# If anything moved:
if ! git diff --quiet; then
    git add -A
    git commit -m "chore: ruff lint/format pass"
fi
```

- [ ] **Step 2: Run unit tests — must all pass**

```bash
pytest tests -q -m "not scenario and not smoke" 2>&1 | tail -10
# Expected: every test passes; no failures, no errors.
```

If any unit test fails, fix it before proceeding. Don't ship a repo with red tests.

- [ ] **Step 3: Wipe state and run a clean integration boot**

```bash
docker ps -a --format '{{.Names}}' | grep '^nt-' | xargs -r docker rm -f 2>/dev/null
docker compose down -v 2>/dev/null || true
cat > .env <<'EOF'
INFLUXDB3_ENTERPRISE_EMAIL=paul+refiiot@influxdata.com
INFLUXDB3_ENTERPRISE_LICENSE_TYPE=trial
EOF

docker compose up -d 2>&1 | tail -5

# Wait for all 5 nodes healthy
deadline=$(($(date +%s) + 300))
ok=0
while [ $(date +%s) -lt $deadline ]; do
    healthy=$(docker compose ps --format json | grep -o '"Health":"healthy"' | wc -l)
    if [ "$healthy" -ge 5 ]; then ok=1; break; fi
    sleep 5
done
[ $ok -eq 1 ] || { docker compose logs --tail 20; exit 1; }
docker compose ps --format 'table {{.Name}}\t{{.Status}}'

# Wait for init to complete
sleep 15
docker compose logs influxdb3-init 2>&1 | tail -5
docker compose logs influxdb3-init 2>&1 | grep -q "initialization complete" || \
    { echo "init failed"; docker compose logs influxdb3-init; exit 1; }
```

- [ ] **Step 4: Let simulator + schedule plugins generate data, then assert basic health**

```bash
sleep 60  # 1 minute of simulator + 12 schedule plugin ticks

TOKEN=$(docker compose exec -T influxdb3-query cat /var/lib/influxdb3/.nt-token-plain | tr -d '\r\n')
_q() {
  curl -s -X POST -H "Authorization: Bearer $TOKEN" \
       -H "Content-Type: application/json" \
       http://localhost:8181/api/v3/query_sql \
       -d "{\"db\":\"nt\",\"q\":\"$1\",\"format\":\"json\"}"
}

# Each of these prints "✓ <thing> OK" or fails the integration run
python3 - <<PY
import json, subprocess
def q(sql):
    r = subprocess.run(["curl","-s","-X","POST","-H",f"Authorization: Bearer $TOKEN".replace("\$TOKEN","${TOKEN}"),"-H","Content-Type: application/json","http://localhost:8181/api/v3/query_sql","-d",json.dumps({"db":"nt","q":sql,"format":"json"})], capture_output=True, text=True)
    return json.loads(r.stdout) if r.stdout.strip() else []
PY

# Simpler: shell asserts
n=$(_q "SELECT COUNT(*) AS n FROM interface_counters WHERE time > now() - INTERVAL '1 minute'" | grep -oE '"n":[0-9]+' | head -1 | cut -d: -f2)
[ "$n" -gt 30000 ] && echo "✓ interface_counters: $n rows in last 1m" || { echo "✗ interface_counters: only $n rows"; exit 1; }

n=$(_q "SELECT COUNT(*) AS n FROM flow_records WHERE time > now() - INTERVAL '1 minute'" | grep -oE '"n":[0-9]+' | head -1 | cut -d: -f2)
[ "$n" -gt 100000 ] && echo "✓ flow_records: $n rows in last 1m" || { echo "✗ flow_records: only $n rows"; exit 1; }

n=$(_q "SELECT COUNT(*) AS n FROM fabric_health WHERE time > now() - INTERVAL '1 minute'" | grep -oE '"n":[0-9]+' | head -1 | cut -d: -f2)
[ "$n" -ge 30 ] && echo "✓ fabric_health: $n rows from schedule plugin" || { echo "✗ schedule plugin not firing"; exit 1; }

n=$(_q "SELECT COUNT(*) AS n FROM last_cache('bgp_sessions','bgp_session_last')" | grep -oE '"n":[0-9]+' | head -1 | cut -d: -f2)
[ "$n" -ge 100 ] && echo "✓ LVC: $n BGP sessions" || { echo "✗ LVC not populated"; exit 1; }
```

- [ ] **Step 5: UI loads and shows data**

```bash
curl -s -o /tmp/ui.html -w 'HTTP=%{http_code}\n' http://localhost:8080/
[ "$(grep -c 'Network Telemetry\|fabric' /tmp/ui.html)" -gt 0 ] || { echo "✗ UI root missing expected text"; exit 1; }
echo "✓ UI root loads"

# FastAPI partials respond
for partial in fabric_state kpi_row throughput anomalies; do
    code=$(curl -s -o /dev/null -w '%{http_code}' http://localhost:8080/partials/$partial)
    [ "$code" = "200" ] && echo "✓ /partials/$partial → 200" || { echo "✗ /partials/$partial → $code"; exit 1; }
done
```

- [ ] **Step 6: Run congestion_hotspot scenario — assert anomaly visible**

```bash
docker compose --profile scenarios run --rm \
  -e SCENARIO=congestion_hotspot scenarios 2>&1 | tail -5
sleep 10  # next anomaly_detector tick

n=$(_q "SELECT COUNT(*) AS n FROM anomalies WHERE switch='leaf-07' AND kind='high_util' AND time > now() - INTERVAL '5 minutes'" | grep -oE '"n":[0-9]+' | head -1 | cut -d: -f2)
[ "$n" -ge 1 ] && echo "✓ congestion_hotspot fired high_util anomaly ($n rows)" || { echo "✗ no anomaly"; exit 1; }
```

- [ ] **Step 7: Run east_west_burst — assert typeahead + detail plugin work**

```bash
docker compose --profile scenarios run --rm \
  -e SCENARIO=east_west_burst scenarios 2>&1 | tail -5
sleep 5

# DVC typeahead
_q "SELECT src_ip FROM distinct_cache('flow_records', 'src_ip_distinct') WHERE src_ip LIKE '10.4%' LIMIT 20" \
  | grep -q '"src_ip":"10.4.7.91"' && echo "✓ typeahead found 10.4.7.91" || { echo "✗ DVC missed burst source"; exit 1; }

# src_ip_detail plugin
curl -s -H "Authorization: Bearer ${TOKEN}" \
     -X POST http://localhost:8181/api/v3/engine/src_ip_detail \
     -H "Content-Type: application/json" \
     -d '{"src_ip": "10.4.7.91"}' > /tmp/detail.json
python3 -c '
import json
d = json.load(open("/tmp/detail.json"))
b = d.get("body", d)
assert b.get("src_ip") == "10.4.7.91", f"bad src_ip: {b}"
assert len(b.get("sparkline", [])) > 0, "empty sparkline"
assert len(b.get("top_destinations", [])) > 0, "empty top_destinations"
print(f"✓ src_ip_detail: sparkline {len(b[\"sparkline\"])} points, {len(b[\"top_destinations\"])} dests")
'
```

- [ ] **Step 8: top_talkers plugin returns non-empty results**

```bash
curl -s -H "Authorization: Bearer ${TOKEN}" \
     -X POST http://localhost:8181/api/v3/engine/top_talkers \
     -H "Content-Type: application/json" \
     -d '{"window_minutes": 5, "limit": 10}' > /tmp/talkers.json
python3 -c '
import json
d = json.load(open("/tmp/talkers.json"))
b = d.get("body", d)
talkers = b.get("talkers", [])
assert len(talkers) > 0, f"empty talkers: {b}"
print(f"✓ top_talkers: {len(talkers)} rows, top src_ip = {talkers[0][\"src_ip\"]}")
'
```

- [ ] **Step 9: Tear down (preserve volume) — repo is verified working**

```bash
docker compose down
git status --short
# Expected: empty (clean tree)
```

If every assertion above passed, the repo is fully verified and ready for the user's review.

If anything failed mid-run, capture the failure mode in `notes/integration-failure.md`, fix the root cause, commit, and re-run this entire task from Step 3.

### Task 29: Create GitHub repo, push, update meta repo

- [ ] **Step 1: Verify gh auth and that the repo doesn't exist yet**

```bash
gh auth status 2>&1 | head -5
gh repo view influxdata/influxdb3-ref-network-telemetry 2>&1 | head -2
# Expected: "Could not resolve to a Repository" — meaning it doesn't exist yet
```

- [ ] **Step 2: Create the public repo**

```bash
gh repo create influxdata/influxdb3-ref-network-telemetry --public \
  --description "Reference architecture: InfluxDB 3 Enterprise multi-node cluster (5 nodes, 2 ingest + query + compact + process) monitoring a data-center fabric. Two schedule plugins (every:5s, cross-node write-back) + two request plugins, three UI patterns side-by-side, per-table retention. ~10k pts/s, runnable in 3 minutes via docker compose." \
  --source /Users/pauldix/codez/reference_architectures/influxdb3-ref-network-telemetry
```

- [ ] **Step 3: Push main**

```bash
git -C /Users/pauldix/codez/reference_architectures/influxdb3-ref-network-telemetry push -u origin main
```

If the push fails immediately after `gh repo create` with "Repository not found", wait 5s and retry — same propagation race bess and iiot hit.

- [ ] **Step 4: Update the meta repo's portfolio README + CONVENTIONS.md amendments**

Edit `/Users/pauldix/codez/reference_architectures/influxdb3-reference-architectures/README.md`:

Change the network-telemetry row from:

```
| 3 | `influxdb3-ref-network-telemetry` | Network Telemetry | 🚧 Coming soon |
```

to:

```
| 3 | [`influxdb3-ref-network-telemetry`](https://github.com/influxdata/influxdb3-ref-network-telemetry) | Network Telemetry | ✅ Available |
```

Append a new comparison column for `network-telemetry` to the "What's different across the available repos" table (or update the existing two-column comparison to three). Highlights to add:

- Cardinality story: ~10k pts/s, multi-tier (interface counters + BGP + flow records)
- Cluster: 5 InfluxDB nodes (vs 1 in bess and iiot)
- Plugins: 2 schedule + 2 request, no WAL (vs WAL-heavy in iiot)
- Schedule cadence: `every:5s` (vs cron-based in iiot)
- New convention demos: cross-node plugin write-back, per-table retention, three UI patterns

- [ ] **Step 5: Amend `CONVENTIONS.md` in the meta repo**

Add new sections to `/Users/pauldix/codez/reference_architectures/influxdb3-reference-architectures/CONVENTIONS.md`:

**New section: Multi-node compose pattern** (placed after "Compose / init.sh"):

```markdown
## Multi-node compose pattern

For repos that ship a clustered InfluxDB 3 Enterprise stack (more than one
server node), the conventions below hold. The `influxdb3-ref-network-telemetry`
repo is the trailblazer; subsequent clustered repos (Fleet, Data Center)
should follow this shape unless their domain genuinely requires a different
topology.

### Shared `influxdb-data` volume

Every InfluxDB node mounts the same named volume at `/var/lib/influxdb3`.
This is what gives the cluster catalog and object-store consistency: every
node sees the same database list, table schemas, caches, and triggers, and
writes from any ingest node are immediately readable from query and process
nodes. Use the `file` object store backend in compose; production swaps to
S3/GCS/Azure with no code change.

### Mode flags per node

- `--mode ingest` — ingest-only nodes; accept writes
- `--mode query` — query-only nodes; serve reads + host request plugins
- `--mode compact` — compaction-only; no HTTP traffic
- `--mode process,query` — process node combo: query engine available locally
  for plugins to use `influxdb3_local.query()`, plus plugin runtime
- Setting `--plugin-dir` automatically adds `process` mode

### Plugin write-back via httpx (cross-node)

A schedule plugin running on a process-only node has no local ingest target.
The plugin module-level code:

1. Reads the admin token from a JSON file in the shared volume at import time.
2. Round-robins POSTs of line protocol to a list of ingest URLs (compose-DNS
   names like `http://nt-ingest-1:8181`), with one fallback hop on
   connection error.

A shared `plugins/_writeback.py` module factors out this pattern. **Do NOT
use `LineBuilder` / `influxdb3_local.write()`** in schedule plugins on a
process-only node — those are for nodes that accept writes locally.

The required env vars on the process node are:

- `<DB>_INGEST_URLS` (comma-separated)
- `<DB>_DB`
- `<DB>_TOKEN_FILE`

(replace `<DB>` with the per-repo prefix, e.g. `NT_INGEST_URLS`)

### Trigger registration in the cluster

Triggers are catalog objects (shared via the volume). Registering them once
via the init container (talking to any node) makes them visible everywhere.
Execution is routed by the engine based on each node's mode + plugin-dir
configuration:

- Schedule triggers run on nodes with `process` mode + `--plugin-dir`
- Request triggers run on nodes with `query` mode + `--plugin-dir`

The cluster uses the same plugin directory mounted on both the query and
process nodes; the engine handles the routing.
```

**New section: `every:` vs `cron:` schedule format** (in "Processing Engine plugins"):

```markdown
### Schedule format: `every:` vs `cron:`

The Processing Engine accepts two formats for schedule trigger specs:

- `cron:<sec> <min> <hour> <dom> <mon> <dow>` — 6-field cron. Use when the
  cadence is aligned to time-of-day (e.g., `cron:0 0 6,14,22 * * *` for
  shift boundaries). iiot uses cron.
- `every:<duration>` — interval string (`every:5s`, `every:1m`, `every:1h`).
  Use for short, regular cadences with no time-of-day alignment requirement.
  Network-telemetry uses `every:5s`.

Pick whichever matches the cadence. For new repos: prefer `every:` when the
trigger is a heartbeat / live rollup; reserve `cron:` for time-of-day work.
```

**New section: Tables created via the configure API (replaces the sentinel-row pattern)**:

```markdown
### Table creation via the configure API (preferred)

Newer repos in the portfolio (network-telemetry+) create tables explicitly
via `POST /api/v3/configure/table` (or the `influxdb3 create table` CLI)
at init time, with the full tag set, field types, and any per-table
retention. This is **strictly better** than the sentinel-row pattern bess
and iiot use:

- Tables become real catalog objects with declared schemas before caches
  and triggers reference them.
- No `__init` row leaks into LVC/DVC reads. No `WHERE site <> '__init'`
  filter needed.
- Per-table retention can be set at create time.

For the older repos (bess, iiot), the sentinel-row trick still works but
the configure-API approach should be preferred for new code paths. Both
repos can be migrated to the configure-API pattern as a follow-up.
```

- [ ] **Step 6: Commit and push the meta-repo updates**

```bash
cd /Users/pauldix/codez/reference_architectures/influxdb3-reference-architectures
git add README.md CONVENTIONS.md
git commit -m "docs: mark network-telemetry Available + CONVENTIONS amendments (multi-node, every:, configure API)"
git push origin main
```

- [ ] **Step 7: Verify both repos live**

```bash
gh repo view influxdata/influxdb3-ref-network-telemetry --json url,visibility,description
gh repo view influxdata/influxdb3-reference-architectures --json url
```

- [ ] **Step 8: Post-ship sanity check**

Open the meta-repo README on GitHub; confirm the network-telemetry row links to the new repo. Open the new repo; confirm the architecture diagram renders, README displays correctly, LICENSE detected.

🎉 Network telemetry reference architecture shipped end-to-end. Multi-node trailblazer template established for the rest of the portfolio.

---

## Done-condition (integration-tested handoff)

By the time the implementer hands the repo back, **every item below has been observed to pass** as part of executing the plan — they are not just self-review checkpoints, they are the actual gate before push to GitHub.

Phase 4 (Task 9) confirms:
- [ ] All 5 InfluxDB nodes reach `healthy` with `paul+refiiot@influxdata.com` license auto-validation.
- [ ] `init.sh` creates DB + 6 tables (via configure API) + LVC + 2 DVCs cleanly.
- [ ] 24h retention is reported on `fabric_health`.
- [ ] Both ingest nodes accept writes (round-robin verified in their logs).
- [ ] LVC populated with 128 BGP sessions; DVC has > 100 distinct src_ips.
- [ ] License persists across `down` / `up`.

Phase 7 (Task 19) confirms:
- [ ] All 4 triggers registered + enabled (`fabric_health`, `anomaly_detector`, `top_talkers`, `src_ip_detail`).
- [ ] Schedule plugins firing — `fabric_health` table grows at the every:5s cadence.
- [ ] `congestion_hotspot` writes a `high_util` anomaly via the schedule plugin's cross-node httpx write-back.
- [ ] `east_west_burst` populates the DVC; `request_src_ip_detail` returns the burst source's composite payload.
- [ ] `request_top_talkers` returns a non-empty list.

Phase 11 (Task 28) confirms (final clean run, fresh state, full demo):
- [ ] Lint clean; all unit tests pass.
- [ ] Cluster boots from clean state (license auto-picked up).
- [ ] FastAPI UI partials all return 200; root page contains expected text.
- [ ] `congestion_hotspot` end-to-end produces visible anomaly via the API.
- [ ] `east_west_burst` end-to-end produces visible typeahead match + detail-panel data.
- [ ] `top_talkers` plugin returns data.
- [ ] All three teaching patterns (FastAPI / direct SQL / request plugin) confirmed by curl.
- [ ] Cluster shuts down cleanly.

Phase 11 (Task 29) confirms:
- [ ] GitHub repo `influxdata/influxdb3-ref-network-telemetry` is public and contains main with all commits.
- [ ] Meta-repo README reflects ✅ Available; CONVENTIONS amendments are present and pushed.

**If any item fails during execution, the implementer halts, fixes the root cause, commits, and re-runs the failed phase before continuing.** The repo is not handed back until every item is green.
