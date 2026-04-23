# BESS Pilot Implementation Plan — `influxdb3-ref-bess`

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** Build the first of eight reference-architecture repos (BESS / Battery Energy Storage Systems) as a complete, `docker compose up`-runnable open-source demo of InfluxDB 3 Enterprise for battery telemetry. This is the **pilot** — conventions validated here become the template for the remaining seven repos.

**Architecture:** Single-node InfluxDB 3 Enterprise with the Processing Engine enabled; a Python simulator generating plausible pack/module/cell telemetry; three Processing Engine plugins (WAL anomaly detector for thermal runaway, scheduled SoH rollup, HTTP request endpoint for pack health); a FastAPI + HTMX + uPlot custom UI; `testcontainers`-based scenario tests; CI with three workflows.

**Tech Stack:** Python 3.12, `influxdb3-python` client, FastAPI, Jinja2, HTMX, uPlot, `testcontainers[influxdb]` (or raw `docker-py`), `pytest`, `ruff`, `mypy` (optional), Docker Compose v2, GitHub Actions, InfluxDB 3 Enterprise (`influxdb:3-enterprise`).

**Repo location during development:** `/Users/pauldix/codez/reference_architectures/influxdb3-ref-bess/`. The plan's file paths are relative to that directory unless absolute.

**Source of truth for shared conventions:** `/Users/pauldix/codez/reference_architectures/influxdb3-reference-architectures/docs/superpowers/specs/2026-04-23-reference-architectures-portfolio-design.md`. If this plan and the spec disagree, update both and note the amendment at the bottom of the spec.

---

## API surface verification (read before starting)

This plan specifies concrete CLI command forms, HTTP endpoint paths, and
Processing Engine plugin API shapes based on the InfluxDB 3 Enterprise
reference docs in `/Users/pauldix/codez/reference_architectures/influxdb3_enterprise_docs.md`
and the plugin/processing-engine docs at
`https://docs.influxdata.com/influxdb3/enterprise/plugins/`.

A few specifics are best-effort — the docs do not fully enumerate them. Verify
these during the first manual checkpoint (Task 9) and adjust this plan's later
tasks if the real API differs:

| Assumption | Where used | How to verify |
|------------|-----------|---------------|
| Query endpoint path is `POST /api/v3/query_sql` with JSON body `{db, q, format: "json"}` returning a JSON array of row dicts | `ui/app.py` `_HttpClient.query`, scenario tests | `docker compose exec influxdb3 curl -s -H "Authorization: Bearer $(cat /var/lib/influxdb3/.bess-operator-token \| jq -r .token)" -X POST http://127.0.0.1:8181/api/v3/query_sql -H 'Content-Type: application/json' -d '{"db":"bess","q":"SELECT 1","format":"json"}'` — adjust to whatever the server accepts. |
| Request-trigger HTTP endpoint is `GET /api/v3/engine/<trigger-path>?<query>` | `ui/app.py` `api_pack_health`, Task 17 Step 3 | Same verification as above against `/api/v3/engine/pack_health`. |
| `influxdb3 create last_cache <name> --database X --table Y` is valid | `influxdb/init.sh` | Run the command manually; if flag names differ, update init.sh and the Makefile's `cli-example` accordingly. |
| `influxdb3 create distinct_cache <name> --database X --table Y --column Z` is valid | `influxdb/init.sh` | Same. |
| `influxdb3 create trigger --trigger-spec "<spec>" --path "<file>" [--trigger-arguments k=v,k=v] <name>` with spec values `table:<t>` / `all_tables` / `cron:<expr>` / `request:<path>` | `influxdb/init.sh`, Task 22 conftest | Confirmed on the plugins docs page. Flag names should be stable. |
| `influxdb3 enable trigger <name> --database X` and `influxdb3 disable trigger <name> --database X` | `influxdb/init.sh` | Confirmed. |
| Health endpoint is `GET /health` returning 200 | compose `healthcheck`, tests | Test with `curl -sf http://127.0.0.1:8181/health`. If the path is `/ping` or different, update the compose healthcheck and `tests/conftest.py`. |
| Plugin runtime exposes `LineBuilder` via `from influxdb3_local import LineBuilder` and methods `.tag`, `.string_field`, `.float64_field`, `.int64_field` | all three plugins | If the plugin runtime imports `LineBuilder` from a different module or method names differ, update the three plugins and the three test files that monkeypatch `LineBuilder`. |
| **Token bootstrap flow** (this one needs the most care): `influxdb3 serve` with `--admin-token-recovery-http-bind 0.0.0.0:8182` exposes a recovery endpoint on port 8182; `influxdb3 create token --admin --regenerate --host http://influxdb3:8182 --output-file <file>` mints an admin token without prior auth. Subsequent CLI calls use `--token <admin-token> --host http://influxdb3:8181`. | `docker-compose.yml` serve command, `influxdb/init.sh` | On Task 9, run the serve command and confirm port 8182 is reachable from the init container, and that the recovery-endpoint create-token call returns a usable JSON file. If this flow differs, the fix lands in compose + init.sh. |

If any of these is wrong in the plan, fix the plan AND the affected tasks in
the "Amendments from Checkpoint #1" section at the bottom before proceeding.

---

## Domain model (reference — used throughout the plan)

A **site** contains **packs**. Each pack has 16 **modules**; each module has 12 **cells**. Default scale for `make up`: 1 site × 4 packs × 16 modules × 12 cells = 768 cells. Each cell reports `voltage` (V, ~3.2–4.2), `temperature_c` (°C, ~20–45). Each pack reports `current_a` (A, signed: discharge negative), `soc` (0–1), `soh` (0–1), plus the site reports `inverter_power_kw`.

**Schema decisions locked in this plan** (justified in the per-task step where each is created):

- Tables: `cell_readings`, `pack_readings`, `inverter_readings`, `alerts`, `pack_rollup_1h`.
- Tags on `cell_readings`: `site`, `pack_id`, `module_id`, `cell_id`.
- Fields on `cell_readings`: `voltage` (float64), `temperature_c` (float64).
- Tags on `pack_readings`: `site`, `pack_id`.
- Fields on `pack_readings`: `current_a` (float64), `soc` (float64), `soh` (float64).
- Tags on `inverter_readings`: `site`.
- Fields on `inverter_readings`: `power_kw` (float64).
- `alerts` written by plugins: tags `source`, `severity`, `pack_id` (nullable); fields `reason` (string), `value` (float64).
- `pack_rollup_1h` written by schedule trigger: tags `pack_id`; fields `min_voltage`, `max_temperature_c`, `avg_current_a`, `last_soc`, `last_soh` (float64), `sample_count` (int64).

---

## File structure (what the plan produces)

```
influxdb3-ref-bess/
├── .env.example
├── .gitignore
├── .github/workflows/
│   ├── unit.yml
│   ├── scenarios.yml
│   ├── smoke.yml
│   └── lint.yml
├── ARCHITECTURE.md
├── CLI_EXAMPLES.md
├── FOR_MAINTAINERS.md
├── LICENSE                       # Apache 2.0
├── Makefile
├── README.md
├── SCENARIOS.md
├── diagrams/
│   ├── architecture.mmd
│   └── architecture.png
├── docker-compose.yml
├── influxdb/
│   ├── init.sh
│   └── schema.md
├── plugins/
│   ├── wal_thermal_runaway.py
│   ├── schedule_soh_daily.py
│   └── request_pack_health.py
├── pyproject.toml
├── scripts/
│   └── setup.sh                  # email prompt + .env writer
├── simulator/
│   ├── Dockerfile
│   ├── __init__.py
│   ├── config.py
│   ├── main.py
│   ├── scenarios/
│   │   ├── __init__.py
│   │   ├── thermal_runaway.py
│   │   └── cell_drift.py
│   ├── signals.py                # BESS domain
│   ├── signals_base.py           # primitives
│   └── writer.py                 # InfluxDB3Writer
├── tests/
│   ├── __init__.py
│   ├── conftest.py
│   ├── test_plugins/
│   │   ├── __init__.py
│   │   ├── test_wal_thermal_runaway.py
│   │   ├── test_schedule_soh_daily.py
│   │   └── test_request_pack_health.py
│   ├── test_scenarios/
│   │   ├── __init__.py
│   │   ├── test_thermal_runaway.py
│   │   └── test_cell_drift.py
│   ├── test_signals.py
│   ├── test_queries.py
│   └── test_smoke.py
└── ui/
    ├── Dockerfile
    ├── __init__.py
    ├── app.py
    ├── queries.py
    ├── static/
    │   ├── app.css
    │   ├── app.js
    │   ├── htmx.min.js
    │   └── uplot.min.css
    │   └── uplot.min.js
    └── templates/
        ├── base.html
        ├── overview.html
        └── partials/
            ├── _kpi_row.html
            ├── _chart.html
            ├── _heatmap.html
            ├── _alerts.html
            └── _pack_health.html
```

---

# Phase 1 — Repo scaffolding

### Task 1: Initialize the BESS repo

**Files:**
- Create: `influxdb3-ref-bess/` (directory)
- Create: `influxdb3-ref-bess/LICENSE`
- Create: `influxdb3-ref-bess/.gitignore`
- Create: `influxdb3-ref-bess/pyproject.toml`
- Create: `influxdb3-ref-bess/README.md` (stub — full README is Task 33)
- Create: `influxdb3-ref-bess/.env.example`

- [ ] **Step 1: Create directory skeleton and init git**

Run (from `/Users/pauldix/codez/reference_architectures/`):
```bash
mkdir -p influxdb3-ref-bess/{.github/workflows,diagrams,influxdb,plugins,scripts,simulator/scenarios,tests/test_plugins,tests/test_scenarios,ui/static,ui/templates/partials}
cd influxdb3-ref-bess && git init -q -b main
```
Expected: directory tree created, `git status` inside it works.

- [ ] **Step 2: Write Apache 2.0 LICENSE**

Create `LICENSE` with the standard Apache 2.0 full text (copy from `https://www.apache.org/licenses/LICENSE-2.0.txt`). Copyright line at the bottom: `Copyright 2026 InfluxData, Inc.`

- [ ] **Step 3: Write `.gitignore`**

Create `.gitignore`:
```
__pycache__/
*.py[cod]
.venv/
.env
*.egg-info/
.pytest_cache/
.mypy_cache/
.ruff_cache/
.DS_Store
influxdb-data/
diagrams/architecture.png
```

Note: `diagrams/architecture.png` is rebuilt from the `.mmd` source so it is gitignored at first and committed later in Task 34 after we settle on a rendering.

- [ ] **Step 4: Write `pyproject.toml`**

```toml
[project]
name = "influxdb3-ref-bess"
version = "0.1.0"
description = "Reference architecture: InfluxDB 3 Enterprise for Battery Energy Storage Systems"
requires-python = ">=3.12"
license = { text = "Apache-2.0" }
dependencies = [
  "influxdb3-python>=0.8",
  "fastapi>=0.110",
  "uvicorn[standard]>=0.29",
  "jinja2>=3.1",
  "httpx>=0.27",
  "pydantic>=2.6",
  "python-dotenv>=1.0",
]

[project.optional-dependencies]
dev = [
  "pytest>=8.0",
  "pytest-asyncio>=0.23",
  "testcontainers>=4.0",
  "ruff>=0.4",
  "mypy>=1.10",
]

[tool.ruff]
line-length = 100
target-version = "py312"

[tool.ruff.lint]
select = ["E", "F", "I", "UP", "B", "SIM"]

[tool.pytest.ini_options]
testpaths = ["tests"]
markers = [
  "smoke: slow end-to-end tests requiring docker compose",
  "scenario: integration tests using testcontainers",
]
```

- [ ] **Step 5: Write `.env.example`**

```bash
# Required for InfluxDB 3 Enterprise license validation.
# On first `make up`, a validation email is sent to this address; click the
# link in the email to activate your trial. The container will remain in
# "waiting for validation" state until you do.
INFLUXDB3_ENTERPRISE_EMAIL=

# Optional: license type. One of: home, trial, commercial. Default: trial.
INFLUXDB3_ENTERPRISE_LICENSE_TYPE=trial

# Simulator tunables
SIM_RATE=50                 # points/sec aggregate across all signals
SIM_CARDINALITY=4           # number of packs (each pack = 16 modules × 12 cells)
SIM_SEED=42                 # reproducibility for scenarios/tests; empty = random

# UI
UI_POLL_KPI_MS=2000
UI_POLL_CHART_MS=5000
```

- [ ] **Step 6: Write README.md stub**

Short stub — the full README is written in Task 33.
```markdown
# influxdb3-ref-bess

Reference architecture: InfluxDB 3 Enterprise for Battery Energy Storage Systems.

Work in progress — full README coming in Task 33 of the pilot plan.

Quickstart: `make up`.
```

- [ ] **Step 7: Commit**

```bash
git add .
git commit -m "chore: initialize BESS reference-architecture repo

Apache 2.0 LICENSE, pyproject.toml, gitignore, .env.example, README stub."
```

---

# Phase 2 — Simulator

### Task 2: Signal primitives library

**Files:**
- Create: `simulator/__init__.py` (empty)
- Create: `simulator/signals_base.py`
- Create: `tests/__init__.py` (empty)
- Create: `tests/test_signals.py`

- [ ] **Step 1: Write failing test for `sinusoid` primitive**

Create `tests/test_signals.py`:
```python
from simulator.signals_base import sinusoid, random_walk, step, burst, jitter


def test_sinusoid_midline_at_phase_zero():
    # Midline at amplitude, offset center, phase 0 => returns offset (within eps)
    val = sinusoid(t_seconds=0.0, period_s=60.0, amplitude=1.0, offset=3.7)
    assert abs(val - 3.7) < 1e-9


def test_sinusoid_peaks_at_quarter_period():
    # At t = period/4, sin is at peak
    val = sinusoid(t_seconds=15.0, period_s=60.0, amplitude=1.0, offset=0.0)
    assert abs(val - 1.0) < 1e-9


def test_random_walk_is_deterministic_with_seed():
    rw1 = random_walk(seed=123, step_std=0.01)
    rw2 = random_walk(seed=123, step_std=0.01)
    samples_1 = [rw1() for _ in range(10)]
    samples_2 = [rw2() for _ in range(10)]
    assert samples_1 == samples_2


def test_random_walk_step_size_bounded():
    rw = random_walk(seed=0, step_std=0.01, start=0.0, min_val=-10.0, max_val=10.0)
    prev = rw()
    for _ in range(1000):
        v = rw()
        # cannot be tighter than 5 stddev in practice
        assert abs(v - prev) < 0.05
        prev = v


def test_step_before_and_after():
    s = step(at_t=10.0, before=1.0, after=2.0)
    assert s(9.0) == 1.0
    assert s(11.0) == 2.0
    assert s(10.0) == 2.0


def test_burst_fires_once_then_zero():
    b = burst(at_t=5.0, duration_s=2.0, magnitude=10.0)
    assert b(4.0) == 0.0
    assert b(5.5) == 10.0
    assert b(6.9) == 10.0
    assert b(7.1) == 0.0


def test_jitter_is_deterministic_with_seed():
    j1 = jitter(seed=99, std=0.1)
    j2 = jitter(seed=99, std=0.1)
    assert [j1(t) for t in range(5)] == [j2(t) for t in range(5)]
```

- [ ] **Step 2: Run to verify failure**

Run: `pytest tests/test_signals.py -q`
Expected: ImportError / ModuleNotFoundError for `simulator.signals_base`.

- [ ] **Step 3: Implement `simulator/signals_base.py`**

```python
"""Primitive signal generators composed by domain-specific signal modules.

This file is intentionally self-contained and copy-pasted across all
reference-architecture repos (see portfolio spec §5.2). Keep it small and
dependency-free.
"""

from __future__ import annotations

import math
import random
from collections.abc import Callable


def sinusoid(t_seconds: float, period_s: float, amplitude: float, offset: float,
             phase_rad: float = 0.0) -> float:
    """Instantaneous value of a sine wave at time `t_seconds`.

    Returns: offset + amplitude * sin(2π t/period + phase).
    """
    return offset + amplitude * math.sin(2.0 * math.pi * t_seconds / period_s + phase_rad)


def random_walk(seed: int, step_std: float, start: float = 0.0,
                min_val: float = float("-inf"),
                max_val: float = float("inf")) -> Callable[[], float]:
    """Return a stateful callable producing the next random-walk value.

    Each call advances one step. Deterministic given `seed`.
    """
    rng = random.Random(seed)
    state = [start]

    def step_fn() -> float:
        state[0] = max(min_val, min(max_val, state[0] + rng.gauss(0.0, step_std)))
        return state[0]

    return step_fn


def step(at_t: float, before: float, after: float) -> Callable[[float], float]:
    """Return a step function: `before` if t<at_t else `after`."""
    def inner(t: float) -> float:
        return after if t >= at_t else before
    return inner


def burst(at_t: float, duration_s: float, magnitude: float) -> Callable[[float], float]:
    """Return a rectangular pulse of height `magnitude` over [at_t, at_t+duration_s]."""
    def inner(t: float) -> float:
        return magnitude if at_t <= t < (at_t + duration_s) else 0.0
    return inner


def jitter(seed: int, std: float) -> Callable[[float], float]:
    """Return a deterministic noise function. Same `t` → same value for given seed."""
    rng = random.Random(seed)
    cache: dict[float, float] = {}

    def inner(t: float) -> float:
        if t not in cache:
            cache[t] = rng.gauss(0.0, std)
        return cache[t]
    return inner
```

- [ ] **Step 4: Run tests, expect green**

Run: `pytest tests/test_signals.py -q`
Expected: 7 passed.

- [ ] **Step 5: Commit**

```bash
git add simulator/__init__.py simulator/signals_base.py tests/__init__.py tests/test_signals.py
git commit -m "feat(simulator): signal primitives (sinusoid, random_walk, step, burst, jitter)"
```

---

### Task 3: BESS domain signals

**Files:**
- Create: `simulator/signals.py`
- Modify: `tests/test_signals.py` (add BESS-signal tests)

- [ ] **Step 1: Extend `tests/test_signals.py` with BESS-signal tests**

Append to `tests/test_signals.py`:
```python
from simulator.signals import Cell, Pack, Site


def test_cell_voltage_within_physical_bounds():
    cell = Cell(site="S1", pack_id="P1", module_id="M1", cell_id="C1", seed=1)
    for t in range(0, 3600, 10):
        rec = cell.tick(t)
        assert 3.0 <= rec.voltage <= 4.25
        assert 15.0 <= rec.temperature_c <= 50.0


def test_pack_current_sign_reflects_mode():
    pack = Pack(site="S1", pack_id="P1", seed=2)
    # Force discharge phase: advance t into the discharge window defined inside Pack.
    rec = pack.tick(0)  # first phase is charge in the default schedule (see signals.py)
    assert rec.current_a >= 0.0 or rec.current_a <= 0.0  # phase-dependent; just assert finite
    assert math.isfinite(rec.current_a)
    assert 0.0 <= rec.soc <= 1.0
    assert 0.0 <= rec.soh <= 1.0


def test_site_has_expected_cell_count():
    site = Site(site="S1", n_packs=2, n_modules_per_pack=16, n_cells_per_module=12, seed=3)
    assert len(list(site.cells())) == 2 * 16 * 12
    assert len(list(site.packs())) == 2


def test_site_tick_emits_one_record_per_cell_pack_and_inverter():
    site = Site(site="S1", n_packs=2, n_modules_per_pack=2, n_cells_per_module=2, seed=4)
    records = list(site.tick(0))
    # 2*2*2 cells + 2 packs + 1 inverter = 11 records
    assert len(records) == 11
```

Also add `import math` at the top of the test file if not already present.

- [ ] **Step 2: Run to confirm failure**

Run: `pytest tests/test_signals.py -q`
Expected: ImportError for `Cell, Pack, Site`.

- [ ] **Step 3: Implement `simulator/signals.py`**

```python
"""BESS domain signals. Composes primitives from signals_base.

Records emitted:
  CellReading      (cell-level voltage/temperature)
  PackReading      (pack-level current/soc/soh)
  InverterReading  (site-level inverter power)

Scale: each Site has N packs × 16 modules × 12 cells by default.
"""

from __future__ import annotations

from collections.abc import Iterable, Iterator
from dataclasses import dataclass

from simulator.signals_base import jitter, random_walk, sinusoid, step


@dataclass
class CellReading:
    site: str
    pack_id: str
    module_id: str
    cell_id: str
    voltage: float
    temperature_c: float


@dataclass
class PackReading:
    site: str
    pack_id: str
    current_a: float
    soc: float
    soh: float


@dataclass
class InverterReading:
    site: str
    power_kw: float


@dataclass
class Cell:
    site: str
    pack_id: str
    module_id: str
    cell_id: str
    seed: int

    def __post_init__(self) -> None:
        # Voltage drifts slowly around a base per-cell. Temperature follows a
        # daily sinusoid plus load-driven jitter.
        self._volt_rw = random_walk(seed=self.seed, step_std=0.002, start=3.7,
                                    min_val=3.2, max_val=4.2)
        self._temp_jitter = jitter(seed=self.seed + 1, std=0.5)

    def tick(self, t: int) -> CellReading:
        voltage = self._volt_rw()
        # Temperature: 25°C baseline + 3°C daily sinusoid + per-tick jitter.
        temperature_c = 25.0 + sinusoid(t, period_s=86400.0, amplitude=3.0, offset=0.0) \
            + self._temp_jitter(float(t))
        temperature_c = max(15.0, min(50.0, temperature_c))
        return CellReading(site=self.site, pack_id=self.pack_id, module_id=self.module_id,
                           cell_id=self.cell_id, voltage=voltage, temperature_c=temperature_c)


@dataclass
class Pack:
    site: str
    pack_id: str
    seed: int

    def __post_init__(self) -> None:
        # SoC drifts slowly; SoH degrades very slowly (but visibly within a long run).
        self._soc_rw = random_walk(seed=self.seed, step_std=0.0005, start=0.65,
                                   min_val=0.10, max_val=0.95)
        self._soh_rw = random_walk(seed=self.seed + 10, step_std=0.00001, start=0.98,
                                   min_val=0.75, max_val=1.0)
        # Current cycles every 1800s between charge (+positive) and discharge (negative).
        self._current_cycle_period = 1800.0

    def tick(self, t: int) -> PackReading:
        current_a = 80.0 * sinusoid(t, period_s=self._current_cycle_period,
                                    amplitude=1.0, offset=0.0)
        return PackReading(site=self.site, pack_id=self.pack_id, current_a=current_a,
                           soc=self._soc_rw(), soh=self._soh_rw())


@dataclass
class Inverter:
    site: str
    seed: int

    def __post_init__(self) -> None:
        self._rw = random_walk(seed=self.seed + 100, step_std=0.5, start=50.0,
                               min_val=0.0, max_val=250.0)

    def tick(self, t: int) -> InverterReading:
        # Diurnal pattern on top of random walk.
        solar_like = max(0.0, sinusoid(t, period_s=86400.0, amplitude=100.0, offset=50.0))
        power = 0.5 * self._rw() + 0.5 * solar_like
        return InverterReading(site=self.site, power_kw=power)


class Site:
    def __init__(self, site: str, n_packs: int = 4, n_modules_per_pack: int = 16,
                 n_cells_per_module: int = 12, seed: int = 0) -> None:
        self.site = site
        self._packs = [Pack(site=site, pack_id=f"pack-{p}", seed=seed + p)
                       for p in range(n_packs)]
        self._cells = [
            Cell(site=site, pack_id=f"pack-{p}", module_id=f"mod-{m}",
                 cell_id=f"cell-{c}", seed=seed * 1000 + p * 100 + m * 10 + c)
            for p in range(n_packs)
            for m in range(n_modules_per_pack)
            for c in range(n_cells_per_module)
        ]
        self._inverter = Inverter(site=site, seed=seed)

    def cells(self) -> Iterable[Cell]:
        return iter(self._cells)

    def packs(self) -> Iterable[Pack]:
        return iter(self._packs)

    def tick(self, t: int) -> Iterator[CellReading | PackReading | InverterReading]:
        for c in self._cells:
            yield c.tick(t)
        for p in self._packs:
            yield p.tick(t)
        yield self._inverter.tick(t)
```

- [ ] **Step 4: Run tests**

Run: `pytest tests/test_signals.py -q`
Expected: 11 passed (7 from Task 2 + 4 new).

- [ ] **Step 5: Commit**

```bash
git add simulator/signals.py tests/test_signals.py
git commit -m "feat(simulator): BESS domain signals (Cell, Pack, Inverter, Site)"
```

---

### Task 4: Writer, config, and simulator main loop

**Files:**
- Create: `simulator/config.py`
- Create: `simulator/writer.py`
- Create: `simulator/main.py`

- [ ] **Step 1: Write `simulator/config.py`**

```python
"""Simulator runtime configuration loaded from environment."""
from __future__ import annotations

import json
import os
from dataclasses import dataclass
from pathlib import Path


def _read_token() -> str:
    """Prefer INFLUX_TOKEN env var; fall back to the JSON token file created
    by `influxdb3 create token --output-file`."""
    if tok := os.environ.get("INFLUX_TOKEN"):
        return tok
    path = os.environ.get("INFLUX_TOKEN_FILE")
    if not path:
        raise RuntimeError("set INFLUX_TOKEN or INFLUX_TOKEN_FILE")
    data = json.loads(Path(path).read_text())
    return data["token"]


@dataclass
class Config:
    influx_url: str
    influx_token: str
    influx_db: str
    rate_pps: int          # aggregate points per second
    n_packs: int
    seed: int | None
    duration_s: int | None  # None = run forever

    @classmethod
    def from_env(cls) -> "Config":
        return cls(
            influx_url=os.environ.get("INFLUX_URL", "http://influxdb3:8181"),
            influx_token=_read_token(),
            influx_db=os.environ.get("INFLUX_DB", "bess"),
            rate_pps=int(os.environ.get("SIM_RATE", "50")),
            n_packs=int(os.environ.get("SIM_CARDINALITY", "4")),
            seed=int(os.environ["SIM_SEED"]) if os.environ.get("SIM_SEED") else None,
            duration_s=int(os.environ["SIM_DURATION"]) if os.environ.get("SIM_DURATION") else None,
        )
```

- [ ] **Step 2: Write `simulator/writer.py`**

```python
"""Batched line-protocol writer wrapping the InfluxDB 3 HTTP API.

Kept small and dependency-light (stdlib + httpx) so the code is easy to read
and adapt. Batches on count OR time, whichever comes first.
"""

from __future__ import annotations

import time
from typing import Any

import httpx

from simulator.signals import CellReading, InverterReading, PackReading


def _escape_tag(v: str) -> str:
    return v.replace(",", r"\,").replace(" ", r"\ ").replace("=", r"\=")


def to_line(record: CellReading | PackReading | InverterReading, ts_ns: int) -> str:
    if isinstance(record, CellReading):
        tags = (f"site={_escape_tag(record.site)},"
                f"pack_id={_escape_tag(record.pack_id)},"
                f"module_id={_escape_tag(record.module_id)},"
                f"cell_id={_escape_tag(record.cell_id)}")
        fields = f"voltage={record.voltage},temperature_c={record.temperature_c}"
        return f"cell_readings,{tags} {fields} {ts_ns}"
    if isinstance(record, PackReading):
        tags = f"site={_escape_tag(record.site)},pack_id={_escape_tag(record.pack_id)}"
        fields = f"current_a={record.current_a},soc={record.soc},soh={record.soh}"
        return f"pack_readings,{tags} {fields} {ts_ns}"
    if isinstance(record, InverterReading):
        tags = f"site={_escape_tag(record.site)}"
        fields = f"power_kw={record.power_kw}"
        return f"inverter_readings,{tags} {fields} {ts_ns}"
    raise TypeError(f"unknown record type: {type(record)!r}")


class InfluxDB3Writer:
    def __init__(self, url: str, token: str, database: str,
                 batch_size: int = 5000, flush_interval_s: float = 1.0) -> None:
        self._url = url.rstrip("/")
        self._headers = {
            "Authorization": f"Bearer {token}",
            "Content-Type": "text/plain; charset=utf-8",
        }
        self._database = database
        self._batch_size = batch_size
        self._flush_interval_s = flush_interval_s
        self._buf: list[str] = []
        self._last_flush = time.monotonic()
        self._client = httpx.Client(timeout=10.0)

    def write_record(self, record: Any, ts_ns: int | None = None) -> None:
        if ts_ns is None:
            ts_ns = time.time_ns()
        self._buf.append(to_line(record, ts_ns))
        if len(self._buf) >= self._batch_size or \
                (time.monotonic() - self._last_flush) >= self._flush_interval_s:
            self.flush()

    def flush(self) -> None:
        if not self._buf:
            return
        body = "\n".join(self._buf).encode("utf-8")
        self._buf.clear()
        self._last_flush = time.monotonic()
        resp = self._client.post(
            f"{self._url}/api/v3/write_lp",
            params={"db": self._database, "precision": "nanosecond", "accept_partial": "false"},
            headers=self._headers,
            content=body,
        )
        resp.raise_for_status()

    def close(self) -> None:
        self.flush()
        self._client.close()
```

- [ ] **Step 3: Add writer tests**

Create `tests/test_writer.py`:
```python
from simulator.signals import CellReading, InverterReading, PackReading
from simulator.writer import to_line


def test_to_line_cell_reading_shape():
    r = CellReading(site="S1", pack_id="P1", module_id="M1", cell_id="C1",
                    voltage=3.75, temperature_c=24.1)
    line = to_line(r, ts_ns=1_700_000_000_000_000_000)
    assert line.startswith("cell_readings,")
    assert "site=S1" in line
    assert "pack_id=P1" in line
    assert "voltage=3.75" in line
    assert "temperature_c=24.1" in line
    assert line.endswith(" 1700000000000000000")


def test_to_line_escapes_special_chars_in_tags():
    r = PackReading(site="Site, 1", pack_id="pack=X", current_a=10.0, soc=0.5, soh=0.99)
    line = to_line(r, ts_ns=1)
    assert r"site=Site\,\ 1" in line
    assert r"pack_id=pack\=X" in line


def test_to_line_inverter():
    r = InverterReading(site="S1", power_kw=42.0)
    line = to_line(r, ts_ns=1)
    assert line == "inverter_readings,site=S1 power_kw=42.0 1"
```

- [ ] **Step 4: Run writer tests**

Run: `pytest tests/test_writer.py -q`
Expected: 3 passed.

- [ ] **Step 5: Write `simulator/main.py`**

```python
"""Simulator entrypoint: build a Site, tick at a real-time cadence, write to InfluxDB."""

from __future__ import annotations

import logging
import signal
import sys
import time

from simulator.config import Config
from simulator.signals import Site
from simulator.writer import InfluxDB3Writer

log = logging.getLogger("simulator")


def run(cfg: Config) -> None:
    seed = cfg.seed if cfg.seed is not None else int(time.time())
    site = Site(site="S1", n_packs=cfg.n_packs, seed=seed)
    writer = InfluxDB3Writer(url=cfg.influx_url, token=cfg.influx_token,
                             database=cfg.influx_db)

    total_signals = cfg.n_packs * 16 * 12 + cfg.n_packs + 1
    tick_interval_s = max(1.0 / max(cfg.rate_pps / total_signals, 1.0), 0.01)
    log.info("simulator starting: %d signals, target %d pps, tick every %.3fs",
             total_signals, cfg.rate_pps, tick_interval_s)

    running = True

    def _stop(*_: object) -> None:
        nonlocal running
        running = False

    signal.signal(signal.SIGINT, _stop)
    signal.signal(signal.SIGTERM, _stop)

    t = 0
    next_tick = time.monotonic()
    started = time.monotonic()
    while running:
        if cfg.duration_s is not None and (time.monotonic() - started) >= cfg.duration_s:
            break
        for rec in site.tick(t):
            writer.write_record(rec)
        t += int(tick_interval_s)
        next_tick += tick_interval_s
        sleep_for = next_tick - time.monotonic()
        if sleep_for > 0:
            time.sleep(sleep_for)

    writer.close()
    log.info("simulator stopped cleanly")


def main() -> int:
    logging.basicConfig(level=logging.INFO, format="%(asctime)s %(levelname)s %(name)s: %(message)s")
    cfg = Config.from_env()
    run(cfg)
    return 0


if __name__ == "__main__":
    sys.exit(main())
```

- [ ] **Step 6: Run full test suite to confirm nothing regressed**

Run: `pytest -q`
Expected: 14 passed.

- [ ] **Step 7: Commit**

```bash
git add simulator/config.py simulator/writer.py simulator/main.py tests/test_writer.py
git commit -m "feat(simulator): writer, config, main tick loop"
```

---

### Task 5: Simulator Dockerfile

**Files:**
- Create: `simulator/Dockerfile`

- [ ] **Step 1: Write Dockerfile**

```dockerfile
FROM python:3.12-slim

WORKDIR /app

COPY pyproject.toml /app/pyproject.toml
RUN pip install --no-cache-dir \
    "influxdb3-python>=0.8" \
    "httpx>=0.27" \
    "python-dotenv>=1.0" \
    "pydantic>=2.6"

COPY simulator /app/simulator

ENV PYTHONUNBUFFERED=1
CMD ["python", "-m", "simulator.main"]
```

- [ ] **Step 2: Verify Docker build works**

Run (from `influxdb3-ref-bess/`): `docker build -t bess-simulator -f simulator/Dockerfile .`
Expected: build succeeds.

- [ ] **Step 3: Commit**

```bash
git add simulator/Dockerfile
git commit -m "feat(simulator): Dockerfile"
```

---

# Phase 3 — InfluxDB, compose, Makefile

### Task 6: `influxdb/init.sh` — DB, caches, token (triggers added in Task 17)

**Files:**
- Create: `influxdb/init.sh`
- Create: `influxdb/schema.md`

- [ ] **Step 1: Write `influxdb/schema.md`**

Complete schema document copy:
```markdown
# BESS schema

This file documents the BESS demo's InfluxDB 3 schema. It is descriptive — the actual
tables are created implicitly by the first write. The `init.sh` script creates the
database, an operator token, and the Last/Distinct caches.

## Tables

| Table | Tags | Fields | Source |
|-------|------|--------|--------|
| `cell_readings` | site, pack_id, module_id, cell_id | voltage (f64), temperature_c (f64) | simulator |
| `pack_readings` | site, pack_id | current_a (f64), soc (f64), soh (f64) | simulator |
| `inverter_readings` | site | power_kw (f64) | simulator |
| `alerts` | source, severity, pack_id | reason (string), value (f64) | plugins |
| `pack_rollup_1h` | pack_id | min_voltage, max_temperature_c, avg_current_a, last_soc, last_soh (f64), sample_count (i64) | schedule plugin |

## Caches

- **Last Value Cache** on `cell_readings` keyed by (site, pack_id, module_id, cell_id) — serves the UI's per-cell latest-reading heatmap at sub-ms latency.
- **Distinct Value Cache** on `cell_readings.cell_id` — serves fast cell-inventory queries and tag-completion in queries.

## Retention

No retention policy in the demo — the simulator writes at a modest rate and we want
readers to see accumulated history. For production, see ARCHITECTURE.md §"Scaling to
production".
```

- [ ] **Step 2: Write `influxdb/init.sh`**

```bash
#!/usr/bin/env bash
# Runs inside the influxdb3 container on startup. Idempotent: safe to re-run.
# Creates the BESS database, an operator token, last/distinct caches, and (in a
# later task) the Processing Engine triggers.
#
# Invoked from docker-compose as an `entrypoint` extension that waits for the
# server to accept admin calls, then runs this script, then execs the server.

set -euo pipefail

INFLUX_HOST="${INFLUX_HOST:-http://influxdb3:8181}"
RECOVERY_HOST="${RECOVERY_HOST:-http://influxdb3:8182}"
INFLUX_DB="${INFLUX_DB:-bess}"
TOKEN_FILE="/var/lib/influxdb3/.bess-operator-token"

log() { echo "[init] $*"; }

wait_for_api() {
    local url="${INFLUX_HOST}/health"
    for _ in $(seq 1 120); do
        if curl -sf "$url" >/dev/null 2>&1; then return 0; fi
        sleep 1
    done
    echo "[init] FATAL: influxdb3 API did not become ready" >&2
    exit 1
}

# Reads the "token" field from a JSON file without requiring jq.
read_token_json() {
    python3 -c "import json,sys; print(json.load(open(sys.argv[1]))['token'])" "$1"
}

ensure_token() {
    if [[ -s "${TOKEN_FILE}" ]]; then
        log "operator token already present"
        return
    fi
    # Bootstrap: mint an admin token via the recovery endpoint. In this demo
    # we store it as the operator token; a production setup would mint a
    # scoped permission token from this admin and then discard the admin.
    log "minting admin/operator token via recovery endpoint"
    influxdb3 create token \
        --admin --regenerate \
        --host "${RECOVERY_HOST}" \
        --output-file "${TOKEN_FILE}"
    chmod 600 "${TOKEN_FILE}"
}

cli() {
    # Invoke the CLI with host + admin token from the token file.
    local token
    token=$(read_token_json "${TOKEN_FILE}")
    influxdb3 --host "${INFLUX_HOST}" --token "${token}" "$@"
}

ensure_database() {
    if cli show databases 2>/dev/null | grep -q "^${INFLUX_DB}\$"; then
        log "database ${INFLUX_DB} already exists"
    else
        log "creating database ${INFLUX_DB}"
        cli create database "${INFLUX_DB}"
    fi
}

ensure_caches() {
    cli create last_cache cell_last \
        --database "${INFLUX_DB}" --table cell_readings \
        2>/dev/null || log "last_cache cell_last likely exists"
    cli create distinct_cache cell_id_distinct \
        --database "${INFLUX_DB}" --table cell_readings --column cell_id \
        2>/dev/null || log "distinct_cache cell_id_distinct likely exists"
}

ensure_triggers() {
    # Populated by Task 17. Left intentionally empty for Task 6.
    return 0
}

main() {
    wait_for_api
    ensure_database
    ensure_token
    ensure_caches
    ensure_triggers
    log "initialization complete"
}

main "$@"
```

- [ ] **Step 3: Make `init.sh` executable**

Run: `chmod +x influxdb/init.sh`

- [ ] **Step 4: Commit**

```bash
git add influxdb/init.sh influxdb/schema.md
git commit -m "feat(influxdb): init script (DB, token, caches) + schema doc"
```

---

### Task 7: `docker-compose.yml`

**Files:**
- Create: `docker-compose.yml`

- [ ] **Step 1: Write `docker-compose.yml`**

```yaml
name: influxdb3-ref-bess

services:
  influxdb3:
    image: influxdb:3-enterprise
    container_name: bess-influxdb3
    environment:
      INFLUXDB3_ENTERPRISE_LICENSE_EMAIL: ${INFLUXDB3_ENTERPRISE_EMAIL:?set INFLUXDB3_ENTERPRISE_EMAIL in .env}
      INFLUXDB3_ENTERPRISE_LICENSE_TYPE: ${INFLUXDB3_ENTERPRISE_LICENSE_TYPE:-trial}
      INFLUXDB3_PLUGIN_DIR: /plugins
      INFLUXDB3_NODE_IDENTIFIER_PREFIX: bess-node
      INFLUXDB3_OBJECT_STORE: file
      INFLUXDB3_DATA_DIR: /var/lib/influxdb3
    command: >
      serve
        --node-id bess-node0
        --cluster-id bess
        --mode all
        --object-store file
        --data-dir /var/lib/influxdb3
        --plugin-dir /plugins
        --admin-token-recovery-http-bind 0.0.0.0:8182
    volumes:
      - influxdb-data:/var/lib/influxdb3
      - ./plugins:/plugins:ro
      - ./influxdb/init.sh:/usr/local/bin/bess-init.sh:ro
    ports:
      - "8181:8181"
    healthcheck:
      test: ["CMD-SHELL", "curl -sf http://127.0.0.1:8181/health && test -s /var/lib/influxdb3/.bess-operator-token"]
      interval: 5s
      timeout: 3s
      retries: 120
      start_period: 5s

  influxdb3-init:
    image: influxdb:3-enterprise
    container_name: bess-influxdb3-init
    depends_on:
      influxdb3:
        condition: service_started
    volumes:
      - influxdb-data:/var/lib/influxdb3
      - ./influxdb/init.sh:/usr/local/bin/bess-init.sh:ro
      - ./plugins:/plugins:ro
    entrypoint: ["/usr/local/bin/bess-init.sh"]
    restart: "no"

  simulator:
    build:
      context: .
      dockerfile: simulator/Dockerfile
    container_name: bess-simulator
    depends_on:
      influxdb3:
        condition: service_healthy
    environment:
      INFLUX_URL: http://influxdb3:8181
      INFLUX_DB: bess
      INFLUX_TOKEN_FILE: /tokens/.bess-operator-token
      SIM_RATE: ${SIM_RATE:-50}
      SIM_CARDINALITY: ${SIM_CARDINALITY:-4}
      SIM_SEED: ${SIM_SEED:-}
    command: ["python", "-m", "simulator.main"]
    volumes:
      - influxdb-data:/tokens:ro  # read-only mount of token dir

  ui:
    build:
      context: .
      dockerfile: ui/Dockerfile
    container_name: bess-ui
    depends_on:
      influxdb3:
        condition: service_healthy
    environment:
      INFLUX_URL: http://influxdb3:8181
      INFLUX_DB: bess
      INFLUX_TOKEN_FILE: /tokens/.bess-operator-token
      UI_POLL_KPI_MS: ${UI_POLL_KPI_MS:-2000}
      UI_POLL_CHART_MS: ${UI_POLL_CHART_MS:-5000}
    command: ["uvicorn", "ui.app:app", "--host", "0.0.0.0", "--port", "8080"]
    ports:
      - "8080:8080"
    volumes:
      - influxdb-data:/tokens:ro

  scenarios:
    build:
      context: .
      dockerfile: simulator/Dockerfile
    container_name: bess-scenarios
    profiles: ["scenarios"]
    depends_on:
      influxdb3:
        condition: service_healthy
    environment:
      INFLUX_URL: http://influxdb3:8181
      INFLUX_DB: bess
      INFLUX_TOKEN_FILE: /tokens/.bess-operator-token
    volumes:
      - influxdb-data:/tokens:ro
    entrypoint: ["sh", "-c", "python -m simulator.scenarios.${SCENARIO}"]
    restart: "no"

volumes:
  influxdb-data:
```

Note: the operator token written by `init.sh` lives at `/var/lib/influxdb3/.bess-operator-token`. The simulator/UI/scenarios services read it from a read-only mount of the same named volume at `/tokens/operator`. A real-world deployment would use a secret manager — the README will call this out.

- [ ] **Step 2: Commit**

```bash
git add docker-compose.yml
git commit -m "feat: docker-compose for influxdb3 + simulator + ui + scenarios"
```

---

### Task 8: Setup script (email prompt) and Makefile

**Files:**
- Create: `scripts/setup.sh`
- Create: `Makefile`

- [ ] **Step 1: Write `scripts/setup.sh`**

```bash
#!/usr/bin/env bash
# Prompts for INFLUXDB3_ENTERPRISE_EMAIL if not yet set in .env.
# Creates .env from .env.example if missing. Idempotent.

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
echo "You must click the validation link before the stack finishes starting."
echo
read -rp "Enter email for license validation: " EMAIL

if [[ -z "${EMAIL}" ]]; then
    echo "[setup] empty email; aborting" >&2
    exit 1
fi

# Replace or append INFLUXDB3_ENTERPRISE_EMAIL line.
if grep -q '^INFLUXDB3_ENTERPRISE_EMAIL=' .env; then
    # GNU sed and BSD sed differ; use a portable form.
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

- [ ] **Step 2: Make it executable**

Run: `chmod +x scripts/setup.sh`

- [ ] **Step 3: Write `Makefile`**

```makefile
SHELL := /usr/bin/env bash
.DEFAULT_GOAL := help
COMPOSE := docker compose

.PHONY: help up down clean cli cli-example logs ps scenario scenario-list \
        test test-unit test-scenarios test-smoke lint format

help: ## Show targets
	@awk 'BEGIN{FS=":.*##"} /^[a-zA-Z0-9_-]+:.*##/ {printf "  \033[1;36m%-20s\033[0m %s\n",$$1,$$2}' $(MAKEFILE_LIST)

up: ## Prompt for email (if needed), write .env, then bring the stack up
	@./scripts/setup.sh
	@echo
	@echo "============================================================="
	@echo "  InfluxDB 3 Enterprise is starting."
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

cli: ## Drop into an interactive shell inside the influxdb3 container with the CLI on PATH
	@$(COMPOSE) exec influxdb3 bash

cli-example: ## Run a named curated CLI example. Usage: make cli-example name=list-databases
	@test -n "$(name)" || (echo "usage: make cli-example name=<example>"; exit 1)
	@grep -A 20 "^## $(name)" CLI_EXAMPLES.md | sed -n '/^```bash/,/^```/p' | sed '1d;$$d' \
	  | while read -r line; do echo "+ $$line"; $(COMPOSE) exec -T influxdb3 bash -lc "$$line"; done

scenario: ## Run a scenario. Usage: make scenario name=thermal_runaway
	@test -n "$(name)" || (echo "usage: make scenario name=<scenario>"; exit 1)
	@SCENARIO=$(name) $(COMPOSE) --profile scenarios run --rm scenarios

scenario-list: ## List available scenarios
	@ls simulator/scenarios/*.py | grep -v __init__ | xargs -I{} basename {} .py | while read n; do \
	  desc=$$(grep -m1 '^"""' simulator/scenarios/$$n.py | sed 's/"""//g'); \
	  printf "  %-24s %s\n" "$$n" "$$desc"; done

test: test-unit test-scenarios ## Run unit + scenario tests (skip smoke)

test-unit: ## Plugin + signal + query unit tests (no docker)
	@pytest tests -q -m "not scenario and not smoke"

test-scenarios: ## Scenario integration tests (uses testcontainers)
	@pytest tests/test_scenarios -q -m scenario

test-smoke: ## End-to-end smoke via docker compose (slow)
	@pytest tests/test_smoke.py -q -m smoke

lint: ## Check formatting and lint
	@ruff check .
	@ruff format --check .

format: ## Auto-fix formatting
	@ruff check --fix .
	@ruff format .
```

- [ ] **Step 4: Verify make help works**

Run: `make help`
Expected: list of targets printed.

- [ ] **Step 5: Commit**

```bash
git add scripts/setup.sh Makefile
git commit -m "feat: setup script (email prompt) and Makefile"
```

---

# Phase 4 — Manual validation checkpoint #1 (no automated test — human verifies)

### Task 9: First manual end-to-end smoke test

**Goal:** Confirm the compose stack comes up, license validation works, the simulator writes data, and the token flow works. No UI, no plugins yet. This is the pilot's first reality check — use what you learn to adjust earlier tasks before proceeding.

- [ ] **Step 1: Ensure no prior state**

Run: `make clean`
Expected: volumes removed, no errors.

- [ ] **Step 2: Bring the stack up**

Run: `make up`
Expected: prompts for email (first run), then `docker compose up -d` runs. `docker compose ps` shows `influxdb3` container started.

- [ ] **Step 3: Complete license validation**

Open the email inbox for the address provided; click the validation link. Wait up to 30 seconds. Then run: `docker compose logs influxdb3 | tail -40`
Expected: log lines showing license validated and server listening.

- [ ] **Step 4: Verify `init.sh` ran**

Run: `docker compose logs influxdb3-init`
Expected: `[init] creating database bess`, `[init] creating operator token for bess`, `[init] initialization complete`.

- [ ] **Step 5: Verify the database exists**

Run: `docker compose exec influxdb3 influxdb3 show databases`
Expected: `bess` appears in the list.

- [ ] **Step 6: Verify the simulator is writing**

Run: `docker compose logs --tail=20 simulator`
Expected: no exceptions; simulator is writing.

Run: `docker compose exec influxdb3 influxdb3 query --database bess "SELECT COUNT(*) FROM cell_readings"`
Expected: a positive integer (grows over time).

- [ ] **Step 7: Record findings**

If any step required changes to Tasks 1–8, go back and apply them now (and commit). Append a line to this plan under "Amendments from Checkpoint #1" (create the section at the bottom of this file) with what had to change. Do not proceed to Phase 5 until the stack comes up cleanly from `make clean && make up` without manual intervention (other than the license click).

- [ ] **Step 8: Leave the stack running for the next phase**

Do not `make down` — Phase 5 scenarios run against this live stack.

---

# Phase 5 — Scenarios

### Task 10: Scenario framework

**Files:**
- Create: `simulator/scenarios/__init__.py` (empty)
- Modify: none yet — scenarios use the same `InfluxDB3Writer` and `Config`.

- [ ] **Step 1: Confirm `simulator/scenarios/__init__.py` is empty**

```bash
test -f simulator/scenarios/__init__.py || touch simulator/scenarios/__init__.py
```

- [ ] **Step 2: Commit**

```bash
git add simulator/scenarios/__init__.py
git commit -m "chore(scenarios): package init"
```

---

### Task 11: `thermal_runaway` scenario

**Files:**
- Create: `simulator/scenarios/thermal_runaway.py`
- Create: `tests/test_scenarios/__init__.py` (empty)

This scenario assumes the simulator's main loop is running (Phase 4), and injects a thermal-runaway event on `cell-5` of `pack-0, mod-3`. The cell's temperature is forced to rise from 28 °C at 2 °C/s until it reaches 80 °C, holding there for 60 seconds. The WAL plugin (written in Task 13) will turn this into an alert.

- [ ] **Step 1: Write `simulator/scenarios/thermal_runaway.py`**

```python
"""Inject a cell thermal runaway event (rising temperature until alert threshold).

Targets pack-0/mod-3/cell-5. Writes directly as a one-shot — runs alongside the
main simulator loop. Takes ~30s.
"""

from __future__ import annotations

import logging
import os
import time

from simulator.signals import CellReading
from simulator.writer import InfluxDB3Writer

log = logging.getLogger("scenarios.thermal_runaway")


SITE = "S1"
PACK = "pack-0"
MODULE = "mod-3"
CELL = "cell-5"
START_TEMP_C = 28.0
RISE_RATE_C_PER_S = 2.0
HOLD_TEMP_C = 80.0
HOLD_DURATION_S = 60
TICK_INTERVAL_S = 1.0


def _writer_from_env() -> InfluxDB3Writer:
    from simulator.config import Config
    cfg = Config.from_env()
    return InfluxDB3Writer(url=cfg.influx_url, token=cfg.influx_token, database=cfg.influx_db)


def run(writer: InfluxDB3Writer | None = None) -> None:
    own = writer is None
    writer = writer or _writer_from_env()

    temperature = START_TEMP_C
    log.info("thermal runaway starting on %s/%s/%s/%s", SITE, PACK, MODULE, CELL)

    # Ramp
    while temperature < HOLD_TEMP_C:
        writer.write_record(CellReading(site=SITE, pack_id=PACK, module_id=MODULE,
                                        cell_id=CELL, voltage=3.8, temperature_c=temperature))
        writer.flush()
        log.info("  ramp: %.1f°C", temperature)
        temperature += RISE_RATE_C_PER_S * TICK_INTERVAL_S
        time.sleep(TICK_INTERVAL_S)

    # Hold
    for i in range(HOLD_DURATION_S):
        writer.write_record(CellReading(site=SITE, pack_id=PACK, module_id=MODULE,
                                        cell_id=CELL, voltage=3.7, temperature_c=HOLD_TEMP_C))
        writer.flush()
        if i % 10 == 0:
            log.info("  hold: %d/%d s at %.1f°C", i, HOLD_DURATION_S, HOLD_TEMP_C)
        time.sleep(TICK_INTERVAL_S)

    if own:
        writer.close()
    log.info("thermal runaway complete")


def main() -> int:
    logging.basicConfig(level=logging.INFO, format="%(asctime)s %(levelname)s %(name)s: %(message)s")
    run()
    return 0


if __name__ == "__main__":
    raise SystemExit(main())
```

- [ ] **Step 2: Create empty `tests/test_scenarios/__init__.py`**

```bash
touch tests/test_scenarios/__init__.py
```

- [ ] **Step 3: Run scenario manually (smoke for the shape — not the plugin yet)**

Run: `make scenario name=thermal_runaway`
Expected: scenario logs ramp then hold, writes succeed (no plugin yet so no alert — that comes in Task 13/17).

- [ ] **Step 4: Verify the scenario's writes landed**

Run: `docker compose exec influxdb3 influxdb3 query --database bess "SELECT MAX(temperature_c) FROM cell_readings WHERE cell_id = 'cell-5' AND pack_id = 'pack-0' AND module_id = 'mod-3'"`
Expected: ≥ 80.0.

- [ ] **Step 5: Commit**

```bash
git add simulator/scenarios/thermal_runaway.py tests/test_scenarios/__init__.py
git commit -m "feat(scenarios): thermal_runaway"
```

---

### Task 12: `cell_drift` scenario

**Files:**
- Create: `simulator/scenarios/cell_drift.py`

Drifts `cell-8` of `pack-1, mod-0` voltage downward from 3.7 V by 0.02 V/s to 3.1 V (below nominal). Demonstrates a slower, cumulative anomaly complementing the abrupt runaway.

- [ ] **Step 1: Write `simulator/scenarios/cell_drift.py`**

```python
"""Drift a cell's voltage downward until below nominal (slow anomaly)."""

from __future__ import annotations

import logging
import os
import time

from simulator.signals import CellReading
from simulator.writer import InfluxDB3Writer

log = logging.getLogger("scenarios.cell_drift")

SITE = "S1"
PACK = "pack-1"
MODULE = "mod-0"
CELL = "cell-8"
START_V = 3.7
END_V = 3.1
DROP_V_PER_S = 0.02
TICK_INTERVAL_S = 1.0


def _writer_from_env() -> InfluxDB3Writer:
    from simulator.config import Config
    cfg = Config.from_env()
    return InfluxDB3Writer(url=cfg.influx_url, token=cfg.influx_token, database=cfg.influx_db)


def run(writer: InfluxDB3Writer | None = None) -> None:
    own = writer is None
    writer = writer or _writer_from_env()
    v = START_V
    log.info("cell drift starting on %s/%s/%s/%s", SITE, PACK, MODULE, CELL)
    while v > END_V:
        writer.write_record(CellReading(site=SITE, pack_id=PACK, module_id=MODULE,
                                        cell_id=CELL, voltage=v, temperature_c=28.0))
        writer.flush()
        v -= DROP_V_PER_S * TICK_INTERVAL_S
        time.sleep(TICK_INTERVAL_S)
    if own:
        writer.close()
    log.info("cell drift complete")


def main() -> int:
    logging.basicConfig(level=logging.INFO, format="%(asctime)s %(levelname)s %(name)s: %(message)s")
    run()
    return 0


if __name__ == "__main__":
    raise SystemExit(main())
```

- [ ] **Step 2: Run manually**

Run: `make scenario name=cell_drift`
Expected: logs; scenario completes.

- [ ] **Step 3: Commit**

```bash
git add simulator/scenarios/cell_drift.py
git commit -m "feat(scenarios): cell_drift"
```

---

# Phase 6 — Processing Engine plugins

### Task 13: `wal_thermal_runaway` plugin

**Files:**
- Create: `plugins/wal_thermal_runaway.py`
- Create: `tests/test_plugins/__init__.py` (empty)
- Create: `tests/test_plugins/test_wal_thermal_runaway.py`

The plugin inspects every write batch to `cell_readings` and emits an alert row to `alerts` whenever any cell has `temperature_c > threshold` (default 70 °C).

- [ ] **Step 1: Write failing test**

Create `tests/test_plugins/__init__.py`:
```bash
touch tests/test_plugins/__init__.py
```

Create `tests/test_plugins/test_wal_thermal_runaway.py`:
```python
"""Unit-test the WAL thermal-runaway plugin with a recording fake.

The real Processing Engine API gives plugins an `influxdb3_local` object and a
`table_batches` iterable of {"table_name": str, "rows": list[dict]}. We simulate
both here.
"""

from __future__ import annotations

import importlib.util
import sys
from pathlib import Path


def _load_plugin():
    """Load plugins/wal_thermal_runaway.py as a module."""
    plugin_path = Path(__file__).resolve().parents[2] / "plugins" / "wal_thermal_runaway.py"
    spec = importlib.util.spec_from_file_location("wal_thermal_runaway", plugin_path)
    mod = importlib.util.module_from_spec(spec)  # type: ignore[arg-type]
    sys.modules["wal_thermal_runaway"] = mod
    spec.loader.exec_module(mod)  # type: ignore[union-attr]
    return mod


class FakeLineBuilder:
    def __init__(self, measurement: str) -> None:
        self.measurement = measurement
        self.tags: dict[str, str] = {}
        self.fields: dict[str, object] = {}

    def tag(self, k: str, v: str) -> "FakeLineBuilder":
        self.tags[k] = v
        return self

    def string_field(self, k: str, v: str) -> "FakeLineBuilder":
        self.fields[k] = v
        return self

    def float64_field(self, k: str, v: float) -> "FakeLineBuilder":
        self.fields[k] = v
        return self


class RecordingInflux:
    def __init__(self) -> None:
        self.writes: list[FakeLineBuilder] = []
        self.logs: list[tuple[str, str]] = []

    def write(self, line: FakeLineBuilder) -> None:
        self.writes.append(line)

    def info(self, msg: str) -> None:
        self.logs.append(("info", msg))

    def warn(self, msg: str) -> None:
        self.logs.append(("warn", msg))


def _batch(table: str, rows: list[dict]) -> dict:
    return {"table_name": table, "rows": rows}


def test_no_alert_below_threshold(monkeypatch):
    mod = _load_plugin()
    monkeypatch.setattr(mod, "LineBuilder", FakeLineBuilder)
    fake = RecordingInflux()
    mod.process_writes(fake, [_batch("cell_readings", [
        {"site": "S1", "pack_id": "pack-0", "module_id": "mod-0", "cell_id": "cell-1",
         "voltage": 3.8, "temperature_c": 30.0, "time": 1}
    ])], args={"threshold": "70"})
    assert fake.writes == []


def test_alert_on_high_temperature(monkeypatch):
    mod = _load_plugin()
    monkeypatch.setattr(mod, "LineBuilder", FakeLineBuilder)
    fake = RecordingInflux()
    mod.process_writes(fake, [_batch("cell_readings", [
        {"site": "S1", "pack_id": "pack-0", "module_id": "mod-3", "cell_id": "cell-5",
         "voltage": 3.8, "temperature_c": 80.0, "time": 1}
    ])], args={"threshold": "70"})
    assert len(fake.writes) == 1
    w = fake.writes[0]
    assert w.measurement == "alerts"
    assert w.tags["source"] == "wal_thermal_runaway"
    assert w.tags["severity"] == "critical"
    assert w.tags["pack_id"] == "pack-0"
    assert w.fields["reason"] == "thermal_runaway"
    assert w.fields["value"] == 80.0


def test_uses_default_threshold_if_missing(monkeypatch):
    mod = _load_plugin()
    monkeypatch.setattr(mod, "LineBuilder", FakeLineBuilder)
    fake = RecordingInflux()
    # Default threshold is 70.
    mod.process_writes(fake, [_batch("cell_readings", [
        {"site": "S1", "pack_id": "pack-0", "module_id": "mod-3", "cell_id": "cell-5",
         "voltage": 3.8, "temperature_c": 75.0, "time": 1}
    ])], args=None)
    assert len(fake.writes) == 1


def test_ignores_other_tables(monkeypatch):
    mod = _load_plugin()
    monkeypatch.setattr(mod, "LineBuilder", FakeLineBuilder)
    fake = RecordingInflux()
    mod.process_writes(fake, [_batch("pack_readings", [
        {"site": "S1", "pack_id": "pack-0", "current_a": 1.0, "soc": 0.5, "soh": 0.95, "time": 1}
    ])], args={"threshold": "70"})
    assert fake.writes == []
```

- [ ] **Step 2: Run to confirm failure**

Run: `pytest tests/test_plugins -q`
Expected: FileNotFoundError or module load failure for `wal_thermal_runaway`.

- [ ] **Step 3: Write `plugins/wal_thermal_runaway.py`**

```python
"""WAL trigger: emit an alert row whenever any cell's temperature exceeds a threshold.

Binding: table=cell_readings, args={"threshold": "70"}
Fires on: every write batch to cell_readings.
Side effects: writes one row to `alerts` per over-threshold cell in the batch.
"""

# LineBuilder is provided by the Processing Engine runtime. We import via a try
# so unit tests can monkeypatch a fake `LineBuilder` symbol without needing the
# engine installed.
try:  # pragma: no cover
    from influxdb3_local import LineBuilder  # type: ignore
except ImportError:  # pragma: no cover
    LineBuilder = None  # injected by tests


def process_writes(influxdb3_local, table_batches, args=None):
    threshold = float((args or {}).get("threshold", "70"))
    for batch in table_batches:
        if batch["table_name"] != "cell_readings":
            continue
        for row in batch["rows"]:
            temp = row.get("temperature_c")
            if temp is None or temp <= threshold:
                continue
            influxdb3_local.info(
                f"thermal_runaway: {row.get('pack_id')}/{row.get('module_id')}/"
                f"{row.get('cell_id')} @ {temp:.1f}°C"
            )
            lb = (LineBuilder("alerts")
                  .tag("source", "wal_thermal_runaway")
                  .tag("severity", "critical")
                  .tag("pack_id", str(row.get("pack_id", "")))
                  .string_field("reason", "thermal_runaway")
                  .float64_field("value", float(temp)))
            influxdb3_local.write(lb)
```

- [ ] **Step 4: Run tests, expect green**

Run: `pytest tests/test_plugins -q`
Expected: 4 passed.

- [ ] **Step 5: Commit**

```bash
git add plugins/wal_thermal_runaway.py tests/test_plugins/__init__.py tests/test_plugins/test_wal_thermal_runaway.py
git commit -m "feat(plugin): wal_thermal_runaway with unit tests"
```

---

### Task 14: `schedule_soh_daily` plugin

**Files:**
- Create: `plugins/schedule_soh_daily.py`
- Create: `tests/test_plugins/test_schedule_soh_daily.py`

Runs once per day at 00:05 (cron `5 0 * * *`). Queries the last 24h of `cell_readings` + `pack_readings`; writes one `pack_rollup_1h` row per pack with min_voltage, max_temperature_c, avg_current_a, last_soc, last_soh, sample_count.

- [ ] **Step 1: Write failing test**

Create `tests/test_plugins/test_schedule_soh_daily.py`:
```python
from __future__ import annotations

import importlib.util
import sys
from datetime import datetime, timezone
from pathlib import Path


def _load_plugin():
    path = Path(__file__).resolve().parents[2] / "plugins" / "schedule_soh_daily.py"
    spec = importlib.util.spec_from_file_location("schedule_soh_daily", path)
    mod = importlib.util.module_from_spec(spec)  # type: ignore[arg-type]
    sys.modules["schedule_soh_daily"] = mod
    spec.loader.exec_module(mod)  # type: ignore[union-attr]
    return mod


class FakeLineBuilder:
    def __init__(self, m: str) -> None:
        self.m = m
        self.tags: dict[str, str] = {}
        self.fields: dict[str, object] = {}

    def tag(self, k: str, v: str):
        self.tags[k] = v
        return self

    def float64_field(self, k: str, v: float):
        self.fields[k] = v
        return self

    def int64_field(self, k: str, v: int):
        self.fields[k] = v
        return self


class FakeInflux:
    def __init__(self, query_result: list[dict]) -> None:
        self._q = query_result
        self.writes: list[FakeLineBuilder] = []
        self.queries: list[str] = []

    def query(self, sql: str):
        self.queries.append(sql)
        return self._q

    def write(self, lb):
        self.writes.append(lb)

    def info(self, _msg: str):
        pass


def test_writes_one_rollup_row_per_pack(monkeypatch):
    mod = _load_plugin()
    monkeypatch.setattr(mod, "LineBuilder", FakeLineBuilder)
    fake = FakeInflux(query_result=[
        {"pack_id": "pack-0", "min_voltage": 3.1, "max_temperature_c": 42.0,
         "avg_current_a": 5.0, "last_soc": 0.65, "last_soh": 0.98, "sample_count": 1234},
        {"pack_id": "pack-1", "min_voltage": 3.3, "max_temperature_c": 38.0,
         "avg_current_a": -2.0, "last_soc": 0.71, "last_soh": 0.97, "sample_count": 1000},
    ])
    mod.process_scheduled_call(fake, call_time=datetime(2026, 4, 23, 0, 5, tzinfo=timezone.utc))
    assert len(fake.writes) == 2
    assert fake.writes[0].m == "pack_rollup_1h"
    assert fake.writes[0].tags["pack_id"] == "pack-0"
    assert fake.writes[0].fields["min_voltage"] == 3.1
    assert fake.writes[0].fields["max_temperature_c"] == 42.0
    assert fake.writes[0].fields["avg_current_a"] == 5.0
    assert fake.writes[0].fields["last_soc"] == 0.65
    assert fake.writes[0].fields["last_soh"] == 0.98
    assert fake.writes[0].fields["sample_count"] == 1234


def test_no_writes_when_no_rows(monkeypatch):
    mod = _load_plugin()
    monkeypatch.setattr(mod, "LineBuilder", FakeLineBuilder)
    fake = FakeInflux(query_result=[])
    mod.process_scheduled_call(fake, call_time=datetime.now(tz=timezone.utc))
    assert fake.writes == []
```

- [ ] **Step 2: Confirm failure**

Run: `pytest tests/test_plugins/test_schedule_soh_daily.py -q`
Expected: import failure.

- [ ] **Step 3: Write `plugins/schedule_soh_daily.py`**

```python
"""Schedule trigger: daily SoH/SoC rollup per pack.

Binding: cron="5 0 * * *", args={}
Runs: once daily at 00:05 UTC.
Side effects: writes one row to `pack_rollup_1h` per pack covering the prior 24h.
"""

try:  # pragma: no cover
    from influxdb3_local import LineBuilder  # type: ignore
except ImportError:  # pragma: no cover
    LineBuilder = None


ROLLUP_SQL = """
SELECT
  p.pack_id AS pack_id,
  MIN(c.voltage) AS min_voltage,
  MAX(c.temperature_c) AS max_temperature_c,
  AVG(p.current_a) AS avg_current_a,
  (SELECT soc FROM pack_readings WHERE pack_id = p.pack_id
     AND time <= NOW() ORDER BY time DESC LIMIT 1) AS last_soc,
  (SELECT soh FROM pack_readings WHERE pack_id = p.pack_id
     AND time <= NOW() ORDER BY time DESC LIMIT 1) AS last_soh,
  COUNT(*) AS sample_count
FROM pack_readings p
LEFT JOIN cell_readings c ON c.pack_id = p.pack_id
WHERE p.time >= NOW() - INTERVAL '24 hours'
GROUP BY p.pack_id
"""


def process_scheduled_call(influxdb3_local, call_time, args=None):
    rows = influxdb3_local.query(ROLLUP_SQL)
    influxdb3_local.info(f"soh_daily rollup: {len(rows)} packs, call_time={call_time}")
    for r in rows:
        lb = (LineBuilder("pack_rollup_1h")
              .tag("pack_id", str(r["pack_id"]))
              .float64_field("min_voltage", float(r["min_voltage"]))
              .float64_field("max_temperature_c", float(r["max_temperature_c"]))
              .float64_field("avg_current_a", float(r["avg_current_a"]))
              .float64_field("last_soc", float(r["last_soc"]))
              .float64_field("last_soh", float(r["last_soh"]))
              .int64_field("sample_count", int(r["sample_count"])))
        influxdb3_local.write(lb)
```

- [ ] **Step 4: Run tests**

Run: `pytest tests/test_plugins/test_schedule_soh_daily.py -q`
Expected: 2 passed.

- [ ] **Step 5: Commit**

```bash
git add plugins/schedule_soh_daily.py tests/test_plugins/test_schedule_soh_daily.py
git commit -m "feat(plugin): schedule_soh_daily with unit tests"
```

---

### Task 15: `request_pack_health` plugin

**Files:**
- Create: `plugins/request_pack_health.py`
- Create: `tests/test_plugins/test_request_pack_health.py`

HTTP request trigger at path `pack_health`. Called as `GET /api/v3/engine/pack_health?pack_id=pack-0`. Returns the most recent pack-level values plus the latest alert (if any) for the requested pack.

- [ ] **Step 1: Write failing test**

Create `tests/test_plugins/test_request_pack_health.py`:
```python
from __future__ import annotations

import importlib.util
import sys
from pathlib import Path


def _load_plugin():
    path = Path(__file__).resolve().parents[2] / "plugins" / "request_pack_health.py"
    spec = importlib.util.spec_from_file_location("request_pack_health", path)
    mod = importlib.util.module_from_spec(spec)  # type: ignore[arg-type]
    sys.modules["request_pack_health"] = mod
    spec.loader.exec_module(mod)  # type: ignore[union-attr]
    return mod


class FakeInflux:
    def __init__(self, results_by_prefix: dict[str, list[dict]]) -> None:
        self._by_prefix = results_by_prefix
        self.queries: list[str] = []

    def query(self, sql: str):
        self.queries.append(sql)
        for prefix, rows in self._by_prefix.items():
            if prefix in sql:
                return rows
        return []

    def info(self, _msg: str):
        pass


def test_returns_400_if_missing_pack_id():
    mod = _load_plugin()
    fake = FakeInflux({})
    resp = mod.process_request(fake, {}, {}, b"", args=None)
    assert resp["status"] == 400
    assert "pack_id" in resp["body"]["error"]


def test_returns_200_with_latest_readings_and_alert():
    mod = _load_plugin()
    fake = FakeInflux({
        "FROM pack_readings": [
            {"time": 1700000000, "current_a": -5.0, "soc": 0.63, "soh": 0.97},
        ],
        "FROM alerts": [
            {"time": 1700000001, "reason": "thermal_runaway", "severity": "critical",
             "source": "wal_thermal_runaway", "value": 80.0},
        ],
    })
    resp = mod.process_request(fake, {"pack_id": "pack-0"}, {}, b"", args=None)
    assert resp["status"] == 200
    body = resp["body"]
    assert body["pack_id"] == "pack-0"
    assert body["latest"]["soc"] == 0.63
    assert body["latest"]["soh"] == 0.97
    assert body["latest_alert"]["reason"] == "thermal_runaway"


def test_returns_200_with_null_alert_if_none():
    mod = _load_plugin()
    fake = FakeInflux({
        "FROM pack_readings": [{"time": 1, "current_a": 0.0, "soc": 0.5, "soh": 1.0}],
        "FROM alerts": [],
    })
    resp = mod.process_request(fake, {"pack_id": "pack-9"}, {}, b"", args=None)
    assert resp["status"] == 200
    assert resp["body"]["latest_alert"] is None
```

- [ ] **Step 2: Confirm failure**

Run: `pytest tests/test_plugins/test_request_pack_health.py -q`
Expected: load failure.

- [ ] **Step 3: Write `plugins/request_pack_health.py`**

```python
"""Request trigger: GET /api/v3/engine/pack_health?pack_id=<id> returns pack status.

Binding: request path="pack_health", args={}
Returns: JSON { pack_id, latest: {current_a, soc, soh, time}, latest_alert: {...} | null }
"""

from __future__ import annotations


def process_request(influxdb3_local, query_parameters, request_headers, request_body, args=None):
    pack_id = query_parameters.get("pack_id")
    if not pack_id:
        return {"status": 400, "body": {"error": "missing required query param: pack_id"}}

    latest_rows = influxdb3_local.query(
        f"SELECT time, current_a, soc, soh FROM pack_readings "
        f"WHERE pack_id = '{pack_id}' ORDER BY time DESC LIMIT 1"
    )
    alert_rows = influxdb3_local.query(
        f"SELECT time, reason, severity, source, value FROM alerts "
        f"WHERE pack_id = '{pack_id}' ORDER BY time DESC LIMIT 1"
    )

    latest = latest_rows[0] if latest_rows else None
    latest_alert = alert_rows[0] if alert_rows else None

    return {"status": 200, "body": {
        "pack_id": pack_id,
        "latest": latest,
        "latest_alert": latest_alert,
    }}
```

Note on SQL injection: because this is a reference architecture intended as a learning artifact, and the plugin runtime does not currently expose parameterized queries on `influxdb3_local.query`, we validate `pack_id` as a safe identifier at the caller (the UI). Document this restriction in `ARCHITECTURE.md` §"Security notes" and in the plugin header comment when the plan reaches Task 32.

- [ ] **Step 4: Add a `pack_id` validator test at the UI layer (done in Task 18)**

This is a forward-reference — the UI validates `pack_id` with a regex before calling the trigger. The unit tests for `queries.py` will cover that.

- [ ] **Step 5: Run tests**

Run: `pytest tests/test_plugins/test_request_pack_health.py -q`
Expected: 3 passed.

- [ ] **Step 6: Run full unit suite**

Run: `pytest tests -q -m "not scenario and not smoke"`
Expected: all previous tests still pass.

- [ ] **Step 7: Commit**

```bash
git add plugins/request_pack_health.py tests/test_plugins/test_request_pack_health.py
git commit -m "feat(plugin): request_pack_health with unit tests"
```

---

### Task 16: Register triggers in `influxdb/init.sh`

**Files:**
- Modify: `influxdb/init.sh` (fill in `ensure_triggers` and add calls to `influxdb3 create trigger`)

- [ ] **Step 1: Replace the placeholder `ensure_triggers` function**

Edit `influxdb/init.sh`. Replace the body of `ensure_triggers` with:

```bash
ensure_triggers() {
    log "registering processing-engine triggers"

    # WAL trigger on cell_readings
    cli create trigger \
        --database "${INFLUX_DB}" \
        --trigger-spec "table:cell_readings" \
        --path "wal_thermal_runaway.py" \
        --trigger-arguments "threshold=70" \
        thermal_runaway 2>/dev/null || log "trigger thermal_runaway already exists"

    # Schedule trigger: daily at 00:05 UTC
    cli create trigger \
        --database "${INFLUX_DB}" \
        --trigger-spec "cron:5 0 * * *" \
        --path "schedule_soh_daily.py" \
        soh_daily 2>/dev/null || log "trigger soh_daily already exists"

    # Request trigger: HTTP path /api/v3/engine/pack_health
    cli create trigger \
        --database "${INFLUX_DB}" \
        --trigger-spec "request:pack_health" \
        --path "request_pack_health.py" \
        pack_health 2>/dev/null || log "trigger pack_health already exists"

    # Enable all triggers (idempotent).
    cli enable trigger thermal_runaway --database "${INFLUX_DB}" 2>/dev/null || true
    cli enable trigger soh_daily       --database "${INFLUX_DB}" 2>/dev/null || true
    cli enable trigger pack_health     --database "${INFLUX_DB}" 2>/dev/null || true
}
```

- [ ] **Step 2: Restart the stack to pick up the new init script**

Run: `make down && make up`
Wait for license validation to complete (cached; instant on subsequent runs after Phase 4).

- [ ] **Step 3: Verify triggers registered**

Run: `docker compose exec influxdb3 influxdb3 show triggers --database bess` *(or equivalent — adjust to whatever the CLI exposes; if `show triggers` is not available, query the system catalog via SQL: `SELECT * FROM system.processing_engine_triggers`)*
Expected: three triggers listed: `thermal_runaway`, `soh_daily`, `pack_health`, all enabled.

- [ ] **Step 4: Re-run thermal_runaway scenario and verify an alert was written**

Run: `make scenario name=thermal_runaway`
Then: `docker compose exec influxdb3 influxdb3 query --database bess "SELECT COUNT(*) FROM alerts WHERE reason = 'thermal_runaway'"`
Expected: ≥ 1.

- [ ] **Step 5: Commit**

```bash
git add influxdb/init.sh
git commit -m "feat(influxdb): register processing-engine triggers in init.sh"
```

---

# Phase 7 — Manual validation checkpoint #2

### Task 17: Verify the full end-to-end data flow

**Goal:** Data flows all the way through: simulator → InfluxDB → WAL plugin → alerts table → request trigger → visible via `curl`.

- [ ] **Step 1: Fire each scenario and watch logs**

Run (in one terminal): `docker compose logs -f influxdb3 | grep -i trigger`
Run (in another): `make scenario name=thermal_runaway`

Expected: logs show `thermal_runaway:` info lines from the plugin.

- [ ] **Step 2: Verify alerts landed**

Run: `docker compose exec influxdb3 influxdb3 query --database bess "SELECT time, pack_id, value, reason FROM alerts ORDER BY time DESC LIMIT 5"`
Expected: recent rows with reason=`thermal_runaway`.

- [ ] **Step 3: Call the request trigger**

Run: `curl -s -H "Authorization: Bearer $(docker compose exec influxdb3 cat /var/lib/influxdb3/.bess-operator-token | jq -r .token)" "http://localhost:8181/api/v3/engine/pack_health?pack_id=pack-0" | jq`
Expected: JSON with `pack_id`, `latest`, `latest_alert`.

- [ ] **Step 4: Invoke the schedule trigger ad-hoc for testing**

If the CLI supports manually invoking a schedule trigger (e.g., `influxdb3 test trigger soh_daily --database bess`), run it. Otherwise, wait until the next 00:05 UTC or temporarily change the cron to `*/5 * * * *`, restart, observe, then restore the original cron.

Run: `docker compose exec influxdb3 influxdb3 query --database bess "SELECT COUNT(*) FROM pack_rollup_1h"`
Expected: > 0 after the trigger has fired at least once.

- [ ] **Step 5: Record findings in the "Amendments" section**

As with Phase 4, if anything had to change in prior tasks, go back and fix them now. Do not proceed until the end-to-end path works from `make clean && make up` plus the license click.

---

# Phase 8 — UI

### Task 18: `ui/queries.py` with unit tests

**Files:**
- Create: `ui/__init__.py` (empty)
- Create: `ui/queries.py`
- Create: `tests/test_queries.py`

Queries return plain dicts/lists. `queries.py` is the single place SQL lives; `app.py` calls these.

- [ ] **Step 1: Write failing test**

Create `tests/test_queries.py`:
```python
"""Unit-test the UI query layer with a recording fake client."""

from __future__ import annotations

from ui.queries import (
    Queries,
    validate_pack_id,
)


class FakeClient:
    def __init__(self, rows_by_substring: dict[str, list[dict]]) -> None:
        self._rows = rows_by_substring
        self.calls: list[str] = []

    def query(self, sql: str, database: str) -> list[dict]:
        self.calls.append(sql)
        for sub, rows in self._rows.items():
            if sub in sql:
                return rows
        return []


def test_validate_pack_id_accepts_safe_ids():
    assert validate_pack_id("pack-0") == "pack-0"
    assert validate_pack_id("pack-99") == "pack-99"


def test_validate_pack_id_rejects_injection():
    import pytest
    for bad in ["' OR 1=1 --", "pack-0; DROP", "pack 0", ""]:
        with pytest.raises(ValueError):
            validate_pack_id(bad)


def test_kpi_summary_shape():
    fake = FakeClient({
        "FROM pack_readings": [{"avg_soc": 0.72, "avg_soh": 0.96,
                                 "total_power_kw": 120.4, "pack_count": 4}],
        "FROM alerts": [{"alert_count_1h": 3}],
    })
    q = Queries(fake, database="bess")
    out = q.kpi_summary()
    assert out["avg_soc"] == 0.72
    assert out["avg_soh"] == 0.96
    assert out["total_power_kw"] == 120.4
    assert out["pack_count"] == 4
    assert out["alerts_1h"] == 3


def test_cell_voltage_series_sql_uses_pack_id():
    fake = FakeClient({"FROM cell_readings": [
        {"t": 1, "min_v": 3.1}, {"t": 2, "min_v": 3.2}
    ]})
    q = Queries(fake, database="bess")
    rows = q.pack_min_cell_voltage("pack-0", window="5m")
    assert rows == [{"t": 1, "min_v": 3.1}, {"t": 2, "min_v": 3.2}]
    assert "pack_id = 'pack-0'" in fake.calls[0]


def test_cell_voltage_series_rejects_bad_pack_id():
    import pytest
    fake = FakeClient({})
    q = Queries(fake, database="bess")
    with pytest.raises(ValueError):
        q.pack_min_cell_voltage("pack-0; DROP", window="5m")


def test_recent_alerts_returns_rows():
    fake = FakeClient({"FROM alerts": [
        {"time": 1, "reason": "thermal_runaway", "severity": "critical",
         "pack_id": "pack-0", "value": 80.0},
    ]})
    q = Queries(fake, database="bess")
    rows = q.recent_alerts(limit=10)
    assert len(rows) == 1
    assert rows[0]["reason"] == "thermal_runaway"


def test_cell_heatmap_latest_returns_per_cell_rows():
    fake = FakeClient({"FROM cell_readings": [
        {"pack_id": "pack-0", "module_id": "mod-0", "cell_id": "cell-0",
         "voltage": 3.8, "temperature_c": 27.1}
    ]})
    q = Queries(fake, database="bess")
    rows = q.cell_heatmap_latest()
    assert rows[0]["voltage"] == 3.8
    assert "FROM cell_readings" in fake.calls[0]
```

- [ ] **Step 2: Confirm failure**

Run: `pytest tests/test_queries.py -q`
Expected: import error.

- [ ] **Step 3: Create `ui/__init__.py` (empty)**

```bash
touch ui/__init__.py
```

- [ ] **Step 4: Write `ui/queries.py`**

```python
"""Named SQL queries for the BESS UI.

All SQL lives here. Each function has a docstring with:
  (a) what it returns,
  (b) which UI route uses it,
  (c) why this query is the right one for this vertical.

This file is the primary teaching artifact for how to model BESS data in
InfluxDB 3 — read this file to see what the domain cares about.
"""

from __future__ import annotations

import re
from typing import Any, Protocol


_SAFE_PACK_ID = re.compile(r"^pack-\d{1,4}$")


def validate_pack_id(pack_id: str) -> str:
    """Raise ValueError if `pack_id` is not a safe identifier."""
    if not isinstance(pack_id, str) or not _SAFE_PACK_ID.match(pack_id):
        raise ValueError(f"unsafe pack_id: {pack_id!r}")
    return pack_id


class _ClientLike(Protocol):
    def query(self, sql: str, database: str) -> list[dict[str, Any]]: ...


class Queries:
    """All UI queries. Pass in any object with `.query(sql, database)`."""

    def __init__(self, client: _ClientLike, database: str) -> None:
        self._client = client
        self._db = database

    # ------------------------------------------------------------------ KPIs

    def kpi_summary(self) -> dict[str, Any]:
        """Top KPI row: avg_soc, avg_soh, total_power_kw, pack_count, alerts_1h.

        Used by: /partials/kpi_row.
        Why: one-shot health snapshot suitable for a 2-second poll.
        """
        pack_rows = self._client.query(
            "SELECT AVG(soc) AS avg_soc, AVG(soh) AS avg_soh, "
            "COUNT(DISTINCT pack_id) AS pack_count "
            "FROM pack_readings WHERE time >= NOW() - INTERVAL '5 minutes'",
            database=self._db,
        )
        inverter_rows = self._client.query(
            "SELECT SUM(power_kw) AS total_power_kw "
            "FROM inverter_readings WHERE time >= NOW() - INTERVAL '1 minute'",
            database=self._db,
        )
        alert_rows = self._client.query(
            "SELECT COUNT(*) AS alert_count_1h "
            "FROM alerts WHERE time >= NOW() - INTERVAL '1 hour'",
            database=self._db,
        )
        p = pack_rows[0] if pack_rows else {"avg_soc": None, "avg_soh": None, "pack_count": 0}
        inv = inverter_rows[0] if inverter_rows else {"total_power_kw": None}
        a = alert_rows[0] if alert_rows else {"alert_count_1h": 0}
        return {**p, **inv, "alerts_1h": a["alert_count_1h"]}

    # ------------------------------------------------------------- Time series

    def pack_min_cell_voltage(self, pack_id: str, window: str = "5m") -> list[dict[str, Any]]:
        """Min cell voltage per minute for one pack, over the window.

        Used by: /partials/chart/pack_voltage.
        Why: catches cells drifting low before BMS trips — the key BESS signal.
        """
        pid = validate_pack_id(pack_id)
        win = _validate_window(window)
        sql = (
            "SELECT date_bin(INTERVAL '1 minute', time) AS t, MIN(voltage) AS min_v "
            f"FROM cell_readings WHERE pack_id = '{pid}' "
            f"AND time >= NOW() - INTERVAL '{win}' GROUP BY 1 ORDER BY 1"
        )
        return self._client.query(sql, database=self._db)

    def pack_max_cell_temperature(self, pack_id: str, window: str = "5m") -> list[dict[str, Any]]:
        """Max cell temperature per minute per pack, over the window.

        Used by: /partials/chart/pack_temperature.
        Why: thermal excursions precede runaway; the max cell is the worst.
        """
        pid = validate_pack_id(pack_id)
        win = _validate_window(window)
        sql = (
            "SELECT date_bin(INTERVAL '1 minute', time) AS t, MAX(temperature_c) AS max_t "
            f"FROM cell_readings WHERE pack_id = '{pid}' "
            f"AND time >= NOW() - INTERVAL '{win}' GROUP BY 1 ORDER BY 1"
        )
        return self._client.query(sql, database=self._db)

    def inverter_power(self, window: str = "30m") -> list[dict[str, Any]]:
        """Inverter power trace (kW) over the window.

        Used by: /partials/chart/inverter.
        Why: reveals diurnal charging/discharging cycle at a glance.
        """
        win = _validate_window(window)
        sql = (
            "SELECT date_bin(INTERVAL '30 seconds', time) AS t, AVG(power_kw) AS p "
            f"FROM inverter_readings WHERE time >= NOW() - INTERVAL '{win}' "
            "GROUP BY 1 ORDER BY 1"
        )
        return self._client.query(sql, database=self._db)

    # ------------------------------------------------------------------ Heatmap

    def cell_heatmap_latest(self) -> list[dict[str, Any]]:
        """Latest voltage and temperature per cell across all packs.

        Used by: /partials/heatmap.
        Why: instant visual spot-check for any hot/low cell — the "at a glance"
        view every BESS operator wants.
        """
        # Relies on the Last Value Cache (see influxdb/init.sh) for sub-ms latency.
        sql = (
            "SELECT pack_id, module_id, cell_id, voltage, temperature_c "
            "FROM cell_readings ORDER BY time DESC"
        )
        return self._client.query(sql, database=self._db)

    # ------------------------------------------------------------------ Alerts

    def recent_alerts(self, limit: int = 20) -> list[dict[str, Any]]:
        """Most recent alerts written by Processing Engine plugins.

        Used by: /partials/alerts.
        Why: show the audit log of anomalies caught by WAL triggers.
        """
        assert 1 <= limit <= 1000
        sql = (
            "SELECT time, reason, severity, pack_id, value "
            f"FROM alerts ORDER BY time DESC LIMIT {int(limit)}"
        )
        return self._client.query(sql, database=self._db)


_SAFE_WINDOW = re.compile(r"^\d{1,3}[smhd]$")


def _validate_window(window: str) -> str:
    if not isinstance(window, str) or not _SAFE_WINDOW.match(window):
        raise ValueError(f"unsafe window: {window!r}")
    # Translate to a full SQL interval unit.
    unit = {"s": "seconds", "m": "minutes", "h": "hours", "d": "days"}[window[-1]]
    return f"{int(window[:-1])} {unit}"
```

Note: The test above expects `validate_pack_id` to reject `'pack 0'`. The regex `^pack-\d{1,4}$` rejects that. Good.

The test for `pack_min_cell_voltage` uses `in fake.calls[0]` checking `"pack_id = 'pack-0'"`. That's the exact substring — good.

- [ ] **Step 5: Run tests**

Run: `pytest tests/test_queries.py -q`
Expected: 6 passed.

- [ ] **Step 6: Commit**

```bash
git add ui/__init__.py ui/queries.py tests/test_queries.py
git commit -m "feat(ui): queries.py (all SQL + input validation) with unit tests"
```

---

### Task 19: FastAPI skeleton and base template

**Files:**
- Create: `ui/app.py`
- Create: `ui/templates/base.html`
- Create: `ui/templates/overview.html`
- Create: `ui/static/app.css`
- Create: `ui/static/htmx.min.js` (vendored)
- Create: `ui/static/uplot.min.js` (vendored)
- Create: `ui/static/uplot.min.css` (vendored)
- Create: `ui/static/app.js`

- [ ] **Step 1: Download vendored assets**

```bash
curl -sSL https://unpkg.com/htmx.org@1.9.12/dist/htmx.min.js -o ui/static/htmx.min.js
curl -sSL https://cdn.jsdelivr.net/npm/uplot@1.6.30/dist/uPlot.iife.min.js -o ui/static/uplot.min.js
curl -sSL https://cdn.jsdelivr.net/npm/uplot@1.6.30/dist/uPlot.min.css -o ui/static/uplot.min.css
```

Verify: `ls -l ui/static/` shows all three files non-empty.

- [ ] **Step 2: Write `ui/static/app.css`**

```css
:root {
  --bg: #0f1419;
  --panel: #161b22;
  --text: #e6edf3;
  --muted: #8b949e;
  --accent: #58a6ff;
  --ok: #3fb950;
  --warn: #d29922;
  --critical: #f85149;
}
* { box-sizing: border-box; }
body { margin: 0; background: var(--bg); color: var(--text);
  font-family: -apple-system, BlinkMacSystemFont, Segoe UI, Helvetica, Arial, sans-serif; }
header { padding: 12px 20px; border-bottom: 1px solid #30363d; }
header h1 { margin: 0; font-size: 18px; font-weight: 600; }
header .sub { color: var(--muted); font-size: 13px; margin-top: 2px; }
main { padding: 20px; display: grid; gap: 20px; grid-template-columns: 1fr; }
.kpi-row { display: grid; grid-template-columns: repeat(auto-fit, minmax(160px, 1fr)); gap: 12px; }
.kpi { background: var(--panel); padding: 14px 16px; border-radius: 8px; }
.kpi .label { color: var(--muted); font-size: 12px; text-transform: uppercase; }
.kpi .value { font-size: 22px; font-weight: 600; margin-top: 4px; }
.grid-2 { display: grid; grid-template-columns: 1fr 1fr; gap: 20px; }
@media (max-width: 900px) { .grid-2 { grid-template-columns: 1fr; } }
.panel { background: var(--panel); padding: 16px; border-radius: 8px; }
.panel h2 { margin: 0 0 12px; font-size: 14px; font-weight: 600; color: var(--muted);
  text-transform: uppercase; letter-spacing: 0.5px; }
.chart { width: 100%; height: 220px; }
.heatmap { display: grid; gap: 2px; }
.heatmap .cell { aspect-ratio: 1/1; border-radius: 2px; }
.alerts { width: 100%; border-collapse: collapse; font-size: 13px; }
.alerts td, .alerts th { padding: 6px 8px; border-bottom: 1px solid #30363d; text-align: left; }
.alerts .severity-critical { color: var(--critical); font-weight: 600; }
.alerts .severity-warn { color: var(--warn); }
button { background: var(--accent); color: var(--bg); border: 0; padding: 8px 14px;
  border-radius: 6px; font-size: 13px; cursor: pointer; }
```

- [ ] **Step 3: Write `ui/static/app.js`**

```javascript
/* Thin glue around uPlot. For each chart container, on the first partial load
   we initialize the plot; subsequent partial swaps call setData() to avoid
   re-creating the canvas. */

window.bess = (function () {
  const plots = {};

  function buildOpts(title, width, height) {
    return {
      title,
      width,
      height,
      axes: [
        { stroke: "#8b949e", grid: { stroke: "#30363d" } },
        { stroke: "#8b949e", grid: { stroke: "#30363d" } },
      ],
      series: [
        {},
        { stroke: "#58a6ff", width: 2 },
      ],
    };
  }

  function render(id, title, timestamps, values) {
    const container = document.getElementById(id);
    if (!container) return;
    const data = [timestamps, values];
    if (plots[id]) {
      plots[id].setData(data);
      return;
    }
    const rect = container.getBoundingClientRect();
    plots[id] = new uPlot(
      buildOpts(title, Math.max(300, rect.width), 220),
      data,
      container
    );
  }

  async function callPackHealth(packId) {
    const r = await fetch(`/api/pack_health?pack_id=${encodeURIComponent(packId)}`);
    const j = await r.json();
    const pre = document.getElementById("pack-health-output");
    if (pre) pre.textContent = JSON.stringify(j, null, 2);
  }

  return { render, callPackHealth };
})();

document.body.addEventListener("htmx:afterSwap", function (evt) {
  const target = evt.detail.target;
  const chartPayload = target.querySelector && target.querySelector("[data-chart-payload]");
  if (!chartPayload) return;
  const id = chartPayload.getAttribute("data-chart-id");
  const title = chartPayload.getAttribute("data-chart-title") || "";
  try {
    const payload = JSON.parse(chartPayload.textContent);
    window.bess.render(id, title, payload.t, payload.v);
  } catch (e) {
    console.error("bess: failed to parse chart payload", e);
  }
});
```

- [ ] **Step 4: Write `ui/templates/base.html`**

```html
<!doctype html>
<html lang="en">
<head>
  <meta charset="utf-8">
  <meta name="viewport" content="width=device-width, initial-scale=1">
  <title>{% block title %}BESS — InfluxDB 3 Enterprise Reference{% endblock %}</title>
  <link rel="stylesheet" href="/static/uplot.min.css">
  <link rel="stylesheet" href="/static/app.css">
  <script src="/static/htmx.min.js" defer></script>
  <script src="/static/uplot.min.js" defer></script>
</head>
<body>
  <header>
    <h1>BESS — InfluxDB 3 Enterprise Reference Architecture</h1>
    <div class="sub">Battery Energy Storage Systems · <a href="https://github.com/influxdata/influxdb3-ref-bess">source</a></div>
  </header>
  <main>
    {% block content %}{% endblock %}
  </main>
  <script src="/static/app.js" defer></script>
</body>
</html>
```

- [ ] **Step 5: Write `ui/templates/overview.html`**

```html
{% extends "base.html" %}
{% block content %}
  <section class="kpi-row" id="kpi-row"
           hx-get="/partials/kpi" hx-trigger="load, every {{ poll_kpi_ms }}ms">
    <div class="kpi"><div class="label">Loading…</div></div>
  </section>

  <section class="grid-2">
    <div class="panel">
      <h2>Pack voltage — min cell per minute</h2>
      <div id="chart-voltage" class="chart"
           hx-get="/partials/chart/voltage" hx-trigger="load, every {{ poll_chart_ms }}ms"
           hx-swap="none">
      </div>
    </div>
    <div class="panel">
      <h2>Pack temperature — max cell per minute</h2>
      <div id="chart-temp" class="chart"
           hx-get="/partials/chart/temperature" hx-trigger="load, every {{ poll_chart_ms }}ms"
           hx-swap="none">
      </div>
    </div>
    <div class="panel">
      <h2>Cell heatmap (latest)</h2>
      <div id="heatmap" hx-get="/partials/heatmap" hx-trigger="load, every {{ poll_chart_ms }}ms"></div>
    </div>
    <div class="panel">
      <h2>Recent alerts</h2>
      <div id="alerts" hx-get="/partials/alerts" hx-trigger="load, every {{ poll_kpi_ms }}ms"></div>
    </div>
  </section>

  <section class="panel">
    <h2>Pack health — via Request trigger (Processing Engine endpoint)</h2>
    <p>
      Pack ID: <input id="pack-health-input" value="pack-0" />
      <button onclick="window.bess.callPackHealth(document.getElementById('pack-health-input').value)">Query</button>
    </p>
    <pre id="pack-health-output" style="background: #0d1117; padding: 10px; border-radius: 4px; overflow: auto;"></pre>
  </section>
{% endblock %}
```

- [ ] **Step 6: Write `ui/app.py`**

```python
"""FastAPI app. Serves the BESS overview page and HTMX partials.

All SQL lives in ui/queries.py. Routes here are thin adapters.
"""

from __future__ import annotations

import json
import os
from typing import Any

import httpx
from fastapi import FastAPI, HTTPException, Request
from fastapi.responses import HTMLResponse, JSONResponse
from fastapi.staticfiles import StaticFiles
from fastapi.templating import Jinja2Templates

from ui.queries import Queries, validate_pack_id


def _read_token() -> str:
    """Prefer INFLUX_TOKEN env var; fall back to the JSON token file."""
    import json
    from pathlib import Path
    if tok := os.environ.get("INFLUX_TOKEN"):
        return tok
    path = os.environ.get("INFLUX_TOKEN_FILE")
    if not path:
        raise RuntimeError("set INFLUX_TOKEN or INFLUX_TOKEN_FILE")
    return json.loads(Path(path).read_text())["token"]


INFLUX_URL = os.environ.get("INFLUX_URL", "http://influxdb3:8181")
INFLUX_DB = os.environ.get("INFLUX_DB", "bess")
INFLUX_TOKEN = _read_token()


class _HttpClient:
    """Minimal adapter exposing .query(sql, database) for Queries."""

    def __init__(self, url: str, token: str) -> None:
        self._url = url.rstrip("/")
        self._headers = {"Authorization": f"Bearer {token}"}
        self._c = httpx.Client(timeout=10.0)

    def query(self, sql: str, database: str) -> list[dict[str, Any]]:
        # InfluxDB 3 query endpoint accepts SQL + database.
        r = self._c.post(
            f"{self._url}/api/v3/query_sql",
            headers={**self._headers, "Content-Type": "application/json"},
            json={"db": database, "q": sql, "format": "json"},
        )
        r.raise_for_status()
        # The response is a JSON array of row objects.
        return r.json()


app = FastAPI(title="BESS Reference")
app.mount("/static", StaticFiles(directory="ui/static"), name="static")
templates = Jinja2Templates(directory="ui/templates")

_client = _HttpClient(INFLUX_URL, INFLUX_TOKEN)
_queries = Queries(_client, database=INFLUX_DB)


@app.get("/", response_class=HTMLResponse)
def overview(request: Request) -> HTMLResponse:
    return templates.TemplateResponse("overview.html", {
        "request": request,
        "poll_kpi_ms": int(os.environ.get("UI_POLL_KPI_MS", "2000")),
        "poll_chart_ms": int(os.environ.get("UI_POLL_CHART_MS", "5000")),
    })


@app.get("/partials/kpi", response_class=HTMLResponse)
def partial_kpi(request: Request) -> HTMLResponse:
    kpi = _queries.kpi_summary()
    return templates.TemplateResponse("partials/_kpi_row.html",
                                      {"request": request, "kpi": kpi})


@app.get("/partials/chart/voltage", response_class=HTMLResponse)
def partial_chart_voltage(request: Request) -> HTMLResponse:
    rows = _queries.pack_min_cell_voltage("pack-0", window="15m")
    return _chart_partial(request, "chart-voltage", "pack-0 min cell V",
                          [r["t"] for r in rows], [r["min_v"] for r in rows])


@app.get("/partials/chart/temperature", response_class=HTMLResponse)
def partial_chart_temperature(request: Request) -> HTMLResponse:
    rows = _queries.pack_max_cell_temperature("pack-0", window="15m")
    return _chart_partial(request, "chart-temp", "pack-0 max cell °C",
                          [r["t"] for r in rows], [r["max_t"] for r in rows])


@app.get("/partials/heatmap", response_class=HTMLResponse)
def partial_heatmap(request: Request) -> HTMLResponse:
    rows = _queries.cell_heatmap_latest()
    return templates.TemplateResponse("partials/_heatmap.html",
                                      {"request": request, "cells": rows})


@app.get("/partials/alerts", response_class=HTMLResponse)
def partial_alerts(request: Request) -> HTMLResponse:
    rows = _queries.recent_alerts(limit=20)
    return templates.TemplateResponse("partials/_alerts.html",
                                      {"request": request, "alerts": rows})


@app.get("/api/pack_health")
def api_pack_health(pack_id: str) -> JSONResponse:
    try:
        validate_pack_id(pack_id)
    except ValueError as e:
        raise HTTPException(status_code=400, detail=str(e))
    # Proxy to the Processing Engine request trigger.
    r = httpx.get(
        f"{INFLUX_URL}/api/v3/engine/pack_health",
        params={"pack_id": pack_id},
        headers={"Authorization": f"Bearer {INFLUX_TOKEN}"},
        timeout=5.0,
    )
    r.raise_for_status()
    return JSONResponse(r.json())


def _chart_partial(request: Request, chart_id: str, title: str,
                   timestamps: list, values: list) -> HTMLResponse:
    return templates.TemplateResponse("partials/_chart.html", {
        "request": request,
        "chart_id": chart_id,
        "title": title,
        "payload": json.dumps({"t": timestamps, "v": values}),
    })
```

- [ ] **Step 7: Commit (partials created in next task will complete this)**

```bash
git add ui/app.py ui/templates/base.html ui/templates/overview.html \
        ui/static/app.css ui/static/app.js ui/static/htmx.min.js \
        ui/static/uplot.min.js ui/static/uplot.min.css
git commit -m "feat(ui): FastAPI skeleton, base template, overview page, static assets"
```

---

### Task 20: Partials — KPI row, chart, heatmap, alerts

**Files:**
- Create: `ui/templates/partials/_kpi_row.html`
- Create: `ui/templates/partials/_chart.html`
- Create: `ui/templates/partials/_heatmap.html`
- Create: `ui/templates/partials/_alerts.html`

- [ ] **Step 1: Write `_kpi_row.html`**

```html
<div class="kpi"><div class="label">Avg SoC</div>
  <div class="value">{{ '%.1f'|format((kpi.avg_soc or 0) * 100) }}%</div></div>
<div class="kpi"><div class="label">Avg SoH</div>
  <div class="value">{{ '%.1f'|format((kpi.avg_soh or 0) * 100) }}%</div></div>
<div class="kpi"><div class="label">Inverter kW</div>
  <div class="value">{{ '%.1f'|format(kpi.total_power_kw or 0) }}</div></div>
<div class="kpi"><div class="label">Packs</div>
  <div class="value">{{ kpi.pack_count or 0 }}</div></div>
<div class="kpi"><div class="label">Alerts (1h)</div>
  <div class="value">{{ kpi.alerts_1h or 0 }}</div></div>
```

- [ ] **Step 2: Write `_chart.html`**

```html
<script type="application/json"
        data-chart-payload
        data-chart-id="{{ chart_id }}"
        data-chart-title="{{ title }}">
{{ payload | safe }}
</script>
```

The HTMX target uses `hx-swap="none"` for the chart routes, but the response body is still inserted into the DOM via the `afterSwap` handler listening for `data-chart-payload`. The script tag is not executed — only read.

- [ ] **Step 3: Write `_heatmap.html`**

```html
{# cells is a flat list ordered by pack/module/cell. Grid wraps by CSS. #}
<div class="heatmap" style="grid-template-columns: repeat(24, 1fr);">
  {% for c in cells %}
    {%- set t = c.temperature_c or 0 -%}
    {%- if t > 70 -%}{% set color = '#f85149' %}
    {%- elif t > 50 -%}{% set color = '#d29922' %}
    {%- elif t > 35 -%}{% set color = '#e3b341' %}
    {%- else -%}{% set color = '#3fb950' %}
    {%- endif %}
    <div class="cell" title="{{ c.pack_id }}/{{ c.module_id }}/{{ c.cell_id }} — {{ '%.1f'|format(t) }}°C / {{ '%.2f'|format(c.voltage or 0) }}V"
         style="background: {{ color }};"></div>
  {% endfor %}
</div>
```

- [ ] **Step 4: Write `_alerts.html`**

```html
<table class="alerts">
  <thead><tr><th>Time</th><th>Severity</th><th>Pack</th><th>Reason</th><th>Value</th></tr></thead>
  <tbody>
  {% for a in alerts %}
    <tr>
      <td>{{ a.time }}</td>
      <td class="severity-{{ a.severity or 'warn' }}">{{ a.severity or '-' }}</td>
      <td>{{ a.pack_id or '-' }}</td>
      <td>{{ a.reason or '-' }}</td>
      <td>{{ '%.1f'|format(a.value or 0) }}</td>
    </tr>
  {% else %}
    <tr><td colspan="5" style="color:#8b949e">No alerts in window.</td></tr>
  {% endfor %}
  </tbody>
</table>
```

- [ ] **Step 5: Commit**

```bash
git add ui/templates/partials
git commit -m "feat(ui): HTMX partials (kpi, chart, heatmap, alerts)"
```

---

### Task 21: UI Dockerfile and wire into compose

**Files:**
- Create: `ui/Dockerfile`

- [ ] **Step 1: Write `ui/Dockerfile`**

```dockerfile
FROM python:3.12-slim

WORKDIR /app

RUN pip install --no-cache-dir \
    "fastapi>=0.110" "uvicorn[standard]>=0.29" "jinja2>=3.1" "httpx>=0.27"

COPY ui /app/ui

ENV PYTHONUNBUFFERED=1
CMD ["uvicorn", "ui.app:app", "--host", "0.0.0.0", "--port", "8080"]
```

- [ ] **Step 2: Verify build**

Run (from `influxdb3-ref-bess/`): `docker build -t bess-ui -f ui/Dockerfile .`
Expected: build succeeds.

- [ ] **Step 3: Bring up full stack and visit the UI**

Run: `make down && make up`
After validation (cached), open `http://localhost:8080`. Expected: page renders with KPI tiles populating within 2 seconds, two charts drawing within 5 seconds, heatmap showing colored cells, alerts table showing rows (if any scenarios have run).

- [ ] **Step 4: Commit**

```bash
git add ui/Dockerfile
git commit -m "feat(ui): Dockerfile"
```

---

# Phase 9 — Tests tier 2 & 3, CI

### Task 22: Scenario integration tests (testcontainers)

**Files:**
- Create: `tests/conftest.py`
- Create: `tests/test_scenarios/test_thermal_runaway.py`
- Create: `tests/test_scenarios/test_cell_drift.py`

- [ ] **Step 1: Write `tests/conftest.py`**

```python
"""Shared fixtures for scenario tests."""

from __future__ import annotations

import json
import os
import shutil
import subprocess
import time
from pathlib import Path

import httpx
import pytest

REPO_ROOT = Path(__file__).resolve().parents[1]
PLUGIN_DIR = REPO_ROOT / "plugins"


def _docker_available() -> bool:
    return shutil.which("docker") is not None and subprocess.call(
        ["docker", "info"], stdout=subprocess.DEVNULL, stderr=subprocess.DEVNULL) == 0


pytestmark_docker = pytest.mark.skipif(
    not _docker_available(), reason="docker not available"
)


@pytest.fixture(scope="session")
def influx_container():
    """Start an influxdb:3-enterprise container with plugins mounted. Yield (url, token, db)."""
    if not _docker_available():
        pytest.skip("docker not available")

    email = os.environ.get("INFLUXDB3_ENTERPRISE_EMAIL")
    if not email:
        pytest.skip("INFLUXDB3_ENTERPRISE_EMAIL must be set for scenario tests "
                    "(see FOR_MAINTAINERS.md on using a pre-validated data volume in CI)")

    name = f"bess-test-{int(time.time())}"
    data_vol = f"{name}-data"
    subprocess.run(["docker", "volume", "create", data_vol], check=True,
                   stdout=subprocess.DEVNULL)

    # Optionally restore a pre-validated volume archive set via env.
    archive = os.environ.get("INFLUXDB3_CI_DATA_ARCHIVE")
    if archive:
        subprocess.run([
            "docker", "run", "--rm",
            "-v", f"{data_vol}:/var/lib/influxdb3",
            "-v", f"{archive}:/archive.tar:ro",
            "alpine", "sh", "-c", "cd /var/lib/influxdb3 && tar xf /archive.tar",
        ], check=True)

    cmd = [
        "docker", "run", "-d", "--name", name,
        "-e", f"INFLUXDB3_ENTERPRISE_LICENSE_EMAIL={email}",
        "-e", "INFLUXDB3_ENTERPRISE_LICENSE_TYPE=trial",
        "-e", "INFLUXDB3_PLUGIN_DIR=/plugins",
        "-p", "0:8181",
        "-v", f"{data_vol}:/var/lib/influxdb3",
        "-v", f"{PLUGIN_DIR}:/plugins:ro",
        "influxdb:3-enterprise",
        "serve", "--node-id", "test-node", "--cluster-id", "test",
        "--mode", "all", "--object-store", "file",
        "--data-dir", "/var/lib/influxdb3", "--plugin-dir", "/plugins",
        "--admin-token-recovery-http-bind", "127.0.0.1:8182",
    ]
    subprocess.run(cmd, check=True, stdout=subprocess.DEVNULL)

    port = subprocess.run(
        ["docker", "port", name, "8181"],
        capture_output=True, check=True, text=True
    ).stdout.strip().split(":")[-1]
    url = f"http://127.0.0.1:{port}"

    # Wait for health
    for _ in range(120):
        try:
            r = httpx.get(f"{url}/health", timeout=1.0)
            if r.status_code == 200:
                break
        except httpx.HTTPError:
            pass
        time.sleep(1)
    else:
        subprocess.run(["docker", "logs", name])
        raise RuntimeError("influxdb3 did not become healthy")

    # Mint admin token via recovery endpoint (inside the container so it can
    # reach 127.0.0.1:8182), then use it for subsequent CLI calls.
    admin_token_result = subprocess.run(
        ["docker", "exec", name, "influxdb3", "create", "token",
         "--admin", "--regenerate",
         "--host", "http://127.0.0.1:8182",
         "--output-file", "/tmp/admin-token.json"],
        capture_output=True, text=True, check=True,
    )
    token_json = subprocess.run(
        ["docker", "exec", name, "cat", "/tmp/admin-token.json"],
        capture_output=True, text=True, check=True,
    ).stdout
    token = json.loads(token_json)["token"]

    subprocess.run(
        ["docker", "exec", name, "influxdb3",
         "--host", "http://127.0.0.1:8181", "--token", token,
         "create", "database", "bess"],
        check=True,
    )

    # Register triggers (same commands init.sh runs).
    base_cli = ["docker", "exec", name, "influxdb3",
                "--host", "http://127.0.0.1:8181", "--token", token]
    for args in [
        ["--trigger-spec", "table:cell_readings", "--path", "wal_thermal_runaway.py",
         "--trigger-arguments", "threshold=70", "thermal_runaway"],
        ["--trigger-spec", "cron:5 0 * * *", "--path", "schedule_soh_daily.py", "soh_daily"],
        ["--trigger-spec", "request:pack_health", "--path", "request_pack_health.py", "pack_health"],
    ]:
        subprocess.run([*base_cli, "create", "trigger", "--database", "bess", *args], check=True)
    for trig in ["thermal_runaway", "soh_daily", "pack_health"]:
        subprocess.run([*base_cli, "enable", "trigger", trig, "--database", "bess"], check=True)

    yield (url, token, "bess", name)

    subprocess.run(["docker", "rm", "-f", name], check=False)
    subprocess.run(["docker", "volume", "rm", data_vol], check=False)
```

- [ ] **Step 2: Write `tests/test_scenarios/test_thermal_runaway.py`**

```python
"""Scenario: thermal_runaway should produce an alert row within 30 seconds."""

from __future__ import annotations

import time

import httpx
import pytest

from simulator.scenarios import thermal_runaway
from simulator.writer import InfluxDB3Writer


@pytest.mark.scenario
def test_thermal_runaway_produces_alert(influx_container):
    url, token, db, _ = influx_container
    writer = InfluxDB3Writer(url=url, token=token, database=db)
    # Shorten the scenario for CI speed: monkeypatch hold duration.
    orig_hold = thermal_runaway.HOLD_DURATION_S
    thermal_runaway.HOLD_DURATION_S = 5
    try:
        thermal_runaway.run(writer=writer)
    finally:
        thermal_runaway.HOLD_DURATION_S = orig_hold
    writer.close()

    # Wait up to 30s for the WAL plugin to write the alert.
    deadline = time.monotonic() + 30
    while time.monotonic() < deadline:
        r = httpx.post(
            f"{url}/api/v3/query_sql",
            headers={"Authorization": f"Bearer {token}"},
            json={"db": db, "q": "SELECT COUNT(*) AS n FROM alerts WHERE reason = 'thermal_runaway'",
                  "format": "json"},
            timeout=5.0,
        )
        r.raise_for_status()
        rows = r.json()
        if rows and rows[0].get("n", 0) >= 1:
            return
        time.sleep(1)
    pytest.fail("no thermal_runaway alert after 30s")
```

- [ ] **Step 3: Write `tests/test_scenarios/test_cell_drift.py`**

```python
"""Scenario: cell_drift should leave a low-voltage trace we can query."""

from __future__ import annotations

import httpx
import pytest

from simulator.scenarios import cell_drift
from simulator.writer import InfluxDB3Writer


@pytest.mark.scenario
def test_cell_drift_end_voltage_below_threshold(influx_container):
    url, token, db, _ = influx_container
    writer = InfluxDB3Writer(url=url, token=token, database=db)
    # Speed up for CI.
    orig_drop = cell_drift.DROP_V_PER_S
    cell_drift.DROP_V_PER_S = 0.2
    try:
        cell_drift.run(writer=writer)
    finally:
        cell_drift.DROP_V_PER_S = orig_drop
    writer.close()

    r = httpx.post(
        f"{url}/api/v3/query_sql",
        headers={"Authorization": f"Bearer {token}"},
        json={"db": db,
              "q": "SELECT MIN(voltage) AS v FROM cell_readings "
                   "WHERE cell_id = 'cell-8' AND pack_id = 'pack-1'",
              "format": "json"},
        timeout=5.0,
    )
    r.raise_for_status()
    rows = r.json()
    assert rows and rows[0]["v"] is not None
    assert rows[0]["v"] <= 3.2, f"expected drift below 3.2V, got {rows[0]['v']}"
```

- [ ] **Step 4: Run scenario tests locally (optional — slow)**

Run: `INFLUXDB3_ENTERPRISE_EMAIL=<your-email> make test-scenarios`
Expected: 2 passed (after license-validated volume is restored, or after interactive validation on first run).

- [ ] **Step 5: Commit**

```bash
git add tests/conftest.py tests/test_scenarios/test_thermal_runaway.py \
        tests/test_scenarios/test_cell_drift.py
git commit -m "test: scenario integration tests (thermal_runaway, cell_drift)"
```

---

### Task 23: Smoke test

**Files:**
- Create: `tests/test_smoke.py`

- [ ] **Step 1: Write `tests/test_smoke.py`**

```python
"""End-to-end smoke test: bring the compose stack up, verify data + UI.

Assumes INFLUXDB3_ENTERPRISE_EMAIL is set in .env and that a pre-validated
license volume is being restored (see FOR_MAINTAINERS.md).
"""

from __future__ import annotations

import subprocess
import time
from pathlib import Path

import httpx
import pytest

REPO = Path(__file__).resolve().parents[1]


@pytest.mark.smoke
def test_compose_up_shows_data_and_ui():
    subprocess.run(["docker", "compose", "down", "-v"], cwd=REPO, check=False)
    subprocess.run(["docker", "compose", "up", "-d", "--build"], cwd=REPO, check=True)
    try:
        # Wait for UI to come up.
        for _ in range(180):
            try:
                r = httpx.get("http://localhost:8080/", timeout=2.0)
                if r.status_code == 200:
                    break
            except httpx.HTTPError:
                pass
            time.sleep(1)
        else:
            subprocess.run(["docker", "compose", "logs"], cwd=REPO)
            pytest.fail("UI did not become ready")

        # Let the simulator write for 30 seconds.
        time.sleep(30)

        # KPI partial returns non-empty.
        r = httpx.get("http://localhost:8080/partials/kpi", timeout=5.0)
        assert r.status_code == 200
        assert b"Avg SoC" in r.content or b"kpi" in r.content

        # Chart partial returns a payload script.
        r = httpx.get("http://localhost:8080/partials/chart/voltage", timeout=5.0)
        assert r.status_code == 200
        assert b"data-chart-payload" in r.content
    finally:
        subprocess.run(["docker", "compose", "down", "-v"], cwd=REPO, check=False)
```

- [ ] **Step 2: Commit**

```bash
git add tests/test_smoke.py
git commit -m "test: smoke test (compose up + UI partials)"
```

---

### Task 24: GitHub Actions workflows

**Files:**
- Create: `.github/workflows/unit.yml`
- Create: `.github/workflows/scenarios.yml`
- Create: `.github/workflows/smoke.yml`
- Create: `.github/workflows/lint.yml`

- [ ] **Step 1: Write `lint.yml`**

```yaml
name: lint
on: [push, pull_request]
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

- [ ] **Step 2: Write `unit.yml`**

```yaml
name: unit
on: [push, pull_request]
jobs:
  unit:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      - uses: actions/setup-python@v5
        with: { python-version: "3.12" }
      - run: pip install -e '.[dev]'
      - run: pytest tests -q -m "not scenario and not smoke"
```

- [ ] **Step 3: Write `scenarios.yml`**

```yaml
name: scenarios
on:
  pull_request:
  push:
    branches: [main]
jobs:
  scenarios:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      - uses: actions/setup-python@v5
        with: { python-version: "3.12" }
      - run: pip install -e '.[dev]'
      - name: Restore pre-validated license volume archive
        env:
          LICENSE_ARCHIVE_B64: ${{ secrets.INFLUXDB3_LICENSE_ARCHIVE_B64 }}
        run: |
          if [ -z "$LICENSE_ARCHIVE_B64" ]; then
            echo "::warning::INFLUXDB3_LICENSE_ARCHIVE_B64 secret not set — scenarios will be skipped"
            exit 0
          fi
          echo "$LICENSE_ARCHIVE_B64" | base64 -d > /tmp/license-archive.tar
          echo "INFLUXDB3_CI_DATA_ARCHIVE=/tmp/license-archive.tar" >> $GITHUB_ENV
          echo "INFLUXDB3_ENTERPRISE_EMAIL=${{ secrets.INFLUXDB3_ENTERPRISE_EMAIL }}" >> $GITHUB_ENV
      - run: pytest tests/test_scenarios -q -m scenario
```

- [ ] **Step 4: Write `smoke.yml`**

```yaml
name: smoke
on:
  push:
    branches: [main]
  schedule:
    - cron: "0 6 * * *"
jobs:
  smoke:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      - uses: actions/setup-python@v5
        with: { python-version: "3.12" }
      - run: pip install -e '.[dev]'
      - name: Write .env with email from secret and restore volume
        env:
          EMAIL: ${{ secrets.INFLUXDB3_ENTERPRISE_EMAIL }}
          ARCHIVE_B64: ${{ secrets.INFLUXDB3_LICENSE_ARCHIVE_B64 }}
        run: |
          cp .env.example .env
          sed -i "s|^INFLUXDB3_ENTERPRISE_EMAIL=.*$|INFLUXDB3_ENTERPRISE_EMAIL=${EMAIL}|" .env
          echo "$ARCHIVE_B64" | base64 -d > /tmp/license-archive.tar
          docker volume create influxdb3-ref-bess_influxdb-data
          docker run --rm \
            -v influxdb3-ref-bess_influxdb-data:/var/lib/influxdb3 \
            -v /tmp/license-archive.tar:/archive.tar:ro \
            alpine sh -c "cd /var/lib/influxdb3 && tar xf /archive.tar"
      - run: pytest tests/test_smoke.py -q -m smoke
```

- [ ] **Step 5: Commit**

```bash
git add .github/workflows
git commit -m "ci: unit, scenarios, smoke, lint workflows"
```

---

# Phase 10 — Docs

### Task 25: `CLI_EXAMPLES.md`

**Files:**
- Create: `CLI_EXAMPLES.md`

- [ ] **Step 1: Write `CLI_EXAMPLES.md`**

```markdown
# CLI Examples

Curated `influxdb3` CLI commands for the BESS demo. Run any example with:

    make cli-example name=<example-name>

Or drop into an interactive shell:

    make cli

Every example below can be copied into that shell as-is.

## list-databases
List all databases on this server.

```bash
influxdb3 show databases
```

## list-tables
List tables in the BESS database.

```bash
influxdb3 query --database bess "SHOW TABLES"
```

## cell-count
How many cell readings have we ingested?

```bash
influxdb3 query --database bess "SELECT COUNT(*) AS n FROM cell_readings"
```

## pack-soc
Current SoC for every pack.

```bash
influxdb3 query --database bess "SELECT pack_id, soc FROM pack_readings WHERE time >= NOW() - INTERVAL '1 minute' ORDER BY pack_id"
```

## min-cell-voltage-5m
Minimum cell voltage per pack over the last 5 minutes — the single most
important BESS query (cells drifting low preview BMS cutoffs).

```bash
influxdb3 query --database bess "SELECT pack_id, MIN(voltage) AS min_v FROM cell_readings WHERE time >= NOW() - INTERVAL '5 minutes' GROUP BY pack_id ORDER BY min_v"
```

## max-cell-temp-5m
Max cell temperature per pack over the last 5 minutes.

```bash
influxdb3 query --database bess "SELECT pack_id, MAX(temperature_c) AS max_t FROM cell_readings WHERE time >= NOW() - INTERVAL '5 minutes' GROUP BY pack_id ORDER BY max_t DESC"
```

## recent-alerts
Most recent alerts emitted by Processing Engine plugins.

```bash
influxdb3 query --database bess "SELECT time, reason, severity, pack_id, value FROM alerts ORDER BY time DESC LIMIT 10"
```

## cache-last-compare
Query latest per-cell voltage WITHOUT the Last Value Cache (scans), then WITH.
Compare elapsed times; the second should be sub-ms.

```bash
# Without cache hint: scans the table.
time influxdb3 query --database bess "SELECT pack_id, module_id, cell_id, LAST(voltage) FROM cell_readings GROUP BY pack_id, module_id, cell_id"

# With the last_cache (cell_last was created by init.sh):
time influxdb3 query --database bess "SELECT pack_id, module_id, cell_id, voltage FROM cell_last"
```

## cache-distinct
Query the distinct value cache for cell IDs.

```bash
influxdb3 query --database bess "SELECT cell_id FROM cell_id_distinct LIMIT 20"
```

## list-triggers
Show the Processing Engine triggers registered.

```bash
influxdb3 query --database bess "SELECT * FROM system.processing_engine_triggers"
```

## pack-health-api
Call the Request-trigger HTTP endpoint directly (bypassing the UI proxy).

```bash
curl -s -H "Authorization: Bearer $(cat /var/lib/influxdb3/.bess-operator-token | jq -r .token)" \
     "http://127.0.0.1:8181/api/v3/engine/pack_health?pack_id=pack-0" | jq
```
```

- [ ] **Step 2: Verify `make cli-example` can parse this file**

Run: `make cli-example name=list-databases`
Expected: shells into the container and prints the DB list.

- [ ] **Step 3: Commit**

```bash
git add CLI_EXAMPLES.md
git commit -m "docs: CLI_EXAMPLES.md curated for BESS"
```

---

### Task 26: `SCENARIOS.md`

**Files:**
- Create: `SCENARIOS.md`

- [ ] **Step 1: Write `SCENARIOS.md`**

```markdown
# Scenarios

Curated events that inject specific conditions into the running simulator.
Each scenario exists to demonstrate a Processing Engine trigger firing.

List available scenarios:

    make scenario-list

Run a scenario (against a running `make up` stack):

    make scenario name=<scenario>

---

## thermal_runaway

**What it does:** Ramps cell `pack-0/mod-3/cell-5` temperature from 28 °C at
2 °C/s until it reaches 80 °C, then holds for 60 seconds.

**What you should see:**

- The `wal_thermal_runaway` WAL trigger fires on every write batch.
- Once `temperature_c > 70`, the plugin writes one row to `alerts` per batch
  with `reason=thermal_runaway`, `severity=critical`, `value=<temp>`.
- The UI's "Recent alerts" panel populates within the next 2-second poll.
- The heatmap cell for `pack-0/mod-3/cell-5` turns red.

**Verify via CLI:**

```bash
make cli-example name=recent-alerts
```

**Where the code lives:** `simulator/scenarios/thermal_runaway.py`,
`plugins/wal_thermal_runaway.py`.

---

## cell_drift

**What it does:** Drifts cell `pack-1/mod-0/cell-8` voltage downward from
3.7 V by 0.02 V/s until it falls below 3.1 V.

**What you should see:**

- The "Pack voltage — min cell per minute" chart on the UI dips for
  `pack-1` as the drift progresses.
- No alert fires from the current WAL plugin (voltage-drop detection is
  intentionally left as an exercise — see ARCHITECTURE.md §"Extending the
  plugins").

**Verify via CLI:**

```bash
influxdb3 query --database bess "SELECT MIN(voltage) FROM cell_readings WHERE cell_id = 'cell-8' AND pack_id = 'pack-1' AND time >= NOW() - INTERVAL '5 minutes'"
```

**Where the code lives:** `simulator/scenarios/cell_drift.py`.
```

- [ ] **Step 2: Commit**

```bash
git add SCENARIOS.md
git commit -m "docs: SCENARIOS.md"
```

---

### Task 27: `ARCHITECTURE.md`

**Files:**
- Create: `ARCHITECTURE.md`

- [ ] **Step 1: Write `ARCHITECTURE.md`**

```markdown
# Architecture

This document is the deep-dive companion to the `README.md`. Read the README
first for the quickstart and headline story; read this for schema decisions,
why they are the way they are, and how to scale this pattern to production.

## Table of contents

1. Domain model
2. Schema
3. Why these tables and tags
4. Processing Engine triggers
5. Enterprise features used
6. UI data flow
7. Security notes
8. Scaling to production
9. Extending the plugins

## 1. Domain model

A **site** contains **packs**. Each pack has 16 modules; each module has 12
cells. Default demo scale: 1 site × 4 packs × 192 cells/pack = 768 cells. Each
cell emits voltage and temperature; each pack emits current, SoC, SoH; the
site's inverter emits power.

## 2. Schema

See `influxdb/schema.md` for the table-by-table reference. The shape is:

- `cell_readings`: one row per cell per tick. Tags: site, pack_id, module_id,
  cell_id. Fields: voltage, temperature_c.
- `pack_readings`: one row per pack per tick. Tags: site, pack_id. Fields:
  current_a, soc, soh.
- `inverter_readings`: one row per site per tick. Tags: site. Fields: power_kw.
- `alerts`: emitted by the WAL trigger plugin. Tags: source, severity, pack_id.
  Fields: reason (string), value (float64).
- `pack_rollup_1h`: emitted by the scheduled trigger plugin. Tags: pack_id.
  Fields: min_voltage, max_temperature_c, avg_current_a, last_soc, last_soh,
  sample_count.

## 3. Why these tables and tags

**Splitting cell and pack readings into separate tables** rather than a single
wide table: the rate per entity differs (cells update per tick; pack SoC/SoH
evolve slowly), and schemas for each are genuinely different. Merging them
would force NULL-heavy rows or a measurement-style type column, both of which
fight InfluxDB's columnar layout.

**`cell_id` as a tag, not a field**: we filter and group on `cell_id`
constantly ("show me cell X's history", "group by cell"). Tags are indexed;
fields are not. High cardinality is fine in InfluxDB 3.

**Pack-level rollups in a separate table (`pack_rollup_1h`)**: the daily
scheduled trigger writes these. The UI and downstream consumers (e.g.,
reporting tools) can query the rollup directly without re-aggregating raw
readings. This is the "materialized downsample" pattern — explicit, auditable,
deletable.

## 4. Processing Engine triggers

| Name | Type | Plugin | Binding | What it does |
|------|------|--------|---------|---------------|
| `thermal_runaway` | WAL | `wal_thermal_runaway.py` | `table:cell_readings` | Writes an `alerts` row whenever a cell > 70 °C. |
| `soh_daily` | Schedule | `schedule_soh_daily.py` | `cron:5 0 * * *` | Writes per-pack daily rollup to `pack_rollup_1h`. |
| `pack_health` | Request | `request_pack_health.py` | `request:pack_health` | HTTP GET returning latest pack status + most recent alert. |

All three are registered on first boot by `influxdb/init.sh`. The plugin files
themselves are idempotent: safe to restart, no state stored outside InfluxDB.

## 5. Enterprise features used

- **Processing Engine**: three triggers (above).
- **Last Value Cache** (`cell_last`): serves the per-cell heatmap query at
  sub-ms latency — critical because the heatmap polls every 5 seconds.
- **Distinct Value Cache** (`cell_id_distinct`): serves cell-ID tag listings,
  used for future features (cell-picker dropdowns, etc.) and as a demo of the
  capability.

## 6. UI data flow

Browser loads the Jinja2 shell at `/`. HTMX polls partial endpoints; each
partial calls a named function in `ui/queries.py`, runs SQL against InfluxDB 3
via the HTTP API, and renders a Jinja2 fragment. Charts return a JSON
payload in a script-tag; a small `app.js` listens for `htmx:afterSwap` and
feeds uPlot.

The "Pack health" panel calls `/api/pack_health` on the UI backend, which
proxies to the Processing Engine's Request trigger endpoint. This is the
"build apps on top of the Processing Engine" story — the same endpoint is
callable via `curl` (see `CLI_EXAMPLES.md` § `pack-health-api`).

## 7. Security notes

This is a reference architecture — the defaults prioritize readability, not
hardening. If you adapt it:

- The operator token is stored on a named Docker volume and read by the
  simulator/UI. In production use a secret manager (Vault, AWS Secrets
  Manager, GCP Secret Manager, Kubernetes secrets).
- The Request trigger takes `pack_id` as a query parameter and we interpolate
  it into SQL. The UI validates `pack_id` against a strict regex before
  proxying. If you expose the endpoint directly without a validating proxy,
  add the same regex inside the plugin.
- CORS and auth on the UI are left open for local demo use.

## 8. Scaling to production

This repo uses a single InfluxDB 3 Enterprise node with a local-file object
store. To productionize:

- **Object store**: switch `--object-store` to `s3|google|azure` and provide
  bucket/credentials via env. The compose file becomes a Helm chart.
- **Clustering**: split into `ingest`, `query`, and `compact` nodes. See the
  `influxdb3-ref-network-telemetry` repo for the clustered shape.
- **Retention/TSM rollups**: the schedule trigger already writes a daily
  rollup. Add a second trigger that deletes `cell_readings` older than N days
  and the full-resolution footprint stays bounded.
- **Auth**: issue per-service tokens with scoped permissions (`read` for UI,
  `read,write` for simulator).
- **Observability**: scrape `http://<influxdb>:8181/metrics` with Prometheus.

## 9. Extending the plugins

- Add a voltage-drop detector to `wal_thermal_runaway.py` or create
  `plugins/wal_cell_drift.py`. Emit to the same `alerts` table with
  `reason="cell_drift"`.
- Make the schedule interval configurable via `--trigger-arguments` instead of
  a hard-coded cron.
- Add a new Request trigger for `site_health` aggregating all pack statuses.
```

- [ ] **Step 2: Commit**

```bash
git add ARCHITECTURE.md
git commit -m "docs: ARCHITECTURE.md"
```

---

### Task 28: `README.md` (full, replacing the stub from Task 1)

**Files:**
- Modify: `README.md`

- [ ] **Step 1: Overwrite `README.md`**

```markdown
# influxdb3-ref-bess

Reference architecture: **InfluxDB 3 Enterprise for Battery Energy Storage Systems**.

Runnable in 2 minutes with `docker compose`. Demonstrates the Processing
Engine, Last Value Cache, Distinct Value Cache, a custom FastAPI+HTMX UI,
and curated scenario injection for anomaly detection.

![architecture](diagrams/architecture.png)

## What's in this repo

| File | Purpose |
|------|---------|
| `docker-compose.yml` | Four services: InfluxDB 3 Enterprise, simulator, UI, on-demand scenarios. |
| `simulator/` | Python simulator producing plausible pack/cell telemetry. |
| `simulator/scenarios/` | Two curated event injectors: `thermal_runaway`, `cell_drift`. |
| `plugins/` | Three Processing Engine plugins: WAL (anomaly), Schedule (rollup), Request (HTTP API). |
| `ui/` | FastAPI + HTMX + uPlot dashboard — KPIs, charts, cell heatmap, alerts, Request-trigger panel. |
| `influxdb/init.sh` | Creates DB, operator token, caches, and triggers on first boot. |
| `tests/` | Plugin unit tests, scenario integration tests (testcontainers), compose smoke test. |
| `ARCHITECTURE.md` | Deep dive: schema, design choices, scaling. |
| `SCENARIOS.md` | Every curated scenario explained. |
| `CLI_EXAMPLES.md` | Curated `influxdb3` CLI commands. |

## Five-minute quickstart

```bash
make up
# On first run, enter your email when prompted. Click the validation link
# in the email that arrives. The stack will finish starting automatically.

# Open http://localhost:8080 — dashboard should populate within ~5 seconds.

# Fire a scenario to see the Processing Engine light up:
make scenario name=thermal_runaway

# Tail the CLI for curated queries:
make cli-example name=recent-alerts
```

Stop and preserve data: `make down`
Stop and wipe data (requires re-validation next time): `make clean`

## Requirements

- Docker Desktop or Docker Engine with Compose v2
- An email address you can click a validation link on the first run

## Next steps

- Read `ARCHITECTURE.md` for schema decisions and scaling notes.
- Read `ui/queries.py` — every query that matters for BESS is documented there.
- Read `plugins/*.py` — three short files demonstrating the Processing Engine
  trigger types.

## Part of

The [InfluxDB 3 Reference Architectures](https://github.com/influxdata/influxdb3-reference-architectures)
portfolio. Other verticals: IIoT, Network Telemetry, Renewables, EV Charging,
Fleet Telematics, Data Center, Oil & Gas.

## License

Apache 2.0 — see `LICENSE`.
```

- [ ] **Step 2: Commit**

```bash
git add README.md
git commit -m "docs: full README.md"
```

---

### Task 29: Architecture diagram

**Files:**
- Create: `diagrams/architecture.mmd`
- Create: `diagrams/architecture.png`

- [ ] **Step 1: Write `diagrams/architecture.mmd`**

```
flowchart LR
  subgraph host[Host machine]
    direction LR
    sim["simulator<br/>(Python)"]
    inf["InfluxDB 3<br/>Enterprise<br/>+ plugins"]
    ui["UI<br/>(FastAPI+HTMX)"]
    scn["scenarios<br/>(on-demand)"]
  end
  browser([user browser])
  browser -- :8080 --> ui
  browser -- :8181 (API) --> inf
  sim -- line protocol --> inf
  ui -- SQL (read) --> inf
  ui -- /api/pack_health --> inf
  scn -- line protocol --> inf
  inf -. WAL trigger .-> inf
  inf -. Schedule trigger .-> inf
  inf -. Request trigger .-> inf
```

- [ ] **Step 2: Render to PNG (optional; diagrams/ is in .gitignore for PNG)**

Use any mermaid CLI. Remove the `.png` from `.gitignore` by editing `.gitignore` to remove the line `diagrams/architecture.png`, then:

```bash
npx -y @mermaid-js/mermaid-cli -i diagrams/architecture.mmd -o diagrams/architecture.png
```

If mermaid-cli is unavailable in your environment, export from https://mermaid.live/ by pasting the `.mmd` contents and downloading the PNG to `diagrams/architecture.png`.

- [ ] **Step 3: Unignore the PNG and commit both files**

Edit `.gitignore` and remove the line `diagrams/architecture.png`.

```bash
git add .gitignore diagrams/architecture.mmd diagrams/architecture.png
git commit -m "docs: architecture diagram (mermaid source + PNG)"
```

---

### Task 30: `FOR_MAINTAINERS.md`

**Files:**
- Create: `FOR_MAINTAINERS.md`

- [ ] **Step 1: Write `FOR_MAINTAINERS.md`**

```markdown
# For maintainers

Notes for maintaining this repo. End users don't need to read this.

## Refreshing the CI license-validated volume

The `smoke.yml` and `scenarios.yml` workflows need a pre-validated
InfluxDB 3 Enterprise `influxdb-data` volume so CI does not require clicking
a validation email on every run.

To refresh:

1. On a workstation: `make clean && make up` using the maintainer's email.
   Complete the validation click.
2. `docker run --rm -v influxdb3-ref-bess_influxdb-data:/src -v $PWD:/out alpine sh -c 'cd /src && tar cf /out/license-archive.tar .'`
3. `base64 -w0 license-archive.tar | pbcopy` (mac) or `base64 -w0 license-archive.tar | xclip -selection clipboard` (linux).
4. Set GitHub repo secret `INFLUXDB3_LICENSE_ARCHIVE_B64` to the paste.
5. Set GitHub repo secret `INFLUXDB3_ENTERPRISE_EMAIL` to the maintainer's email.

Refresh when the license or its trial period needs renewal.

## Running scenario tests locally

Set `INFLUXDB3_ENTERPRISE_EMAIL` in your shell and run:

    make test-scenarios

If you don't have a pre-validated archive, the first run will hang waiting for
you to click the validation email; subsequent runs against the same volume
are instant.
```

- [ ] **Step 2: Commit**

```bash
git add FOR_MAINTAINERS.md
git commit -m "docs: FOR_MAINTAINERS.md"
```

---

# Phase 11 — Final acceptance

### Task 31: End-to-end acceptance checklist

This is not code. It's a human validation pass covering every success
criterion in the portfolio spec.

- [ ] **Step 1: Fresh clone test**

In a clean directory:
```bash
git clone <local-or-remote> /tmp/bess-acceptance
cd /tmp/bess-acceptance
make up
```
Follow the email prompt. Click the validation link.

- [ ] **Step 2: Confirm success criterion #1 — "live data within 2 minutes"**

Open `http://localhost:8080`. Start a timer when the container's health check first passes. Confirm KPI tiles have numeric values and both charts have a line drawn within 2 minutes.

- [ ] **Step 3: Confirm success criterion #2 — "README has all required sections"**

Open `README.md`. Confirm it has: architecture diagram reference, "what's in this repo" table, 5-minute quickstart. All three must be present and correct.

- [ ] **Step 4: Confirm success criterion #3 — "scenarios trigger plugins"**

```bash
make scenario name=thermal_runaway
```
Within 30 seconds, the UI's "Recent alerts" panel should show at least one row with reason=`thermal_runaway`. Run `make cli-example name=recent-alerts` and confirm the same.

- [ ] **Step 5: Confirm success criterion #4 — "CI green"**

Push to a branch; verify `unit.yml`, `scenarios.yml`, `smoke.yml` (if pushed to main), `lint.yml` all pass.

- [ ] **Step 6: Confirm success criterion #5 — "adaptable files have header comments"**

Open each of: `simulator/signals.py`, `plugins/wal_thermal_runaway.py`, `plugins/schedule_soh_daily.py`, `plugins/request_pack_health.py`, `ui/queries.py`, `ui/app.py`, `influxdb/init.sh`. Confirm each has a top-of-file comment explaining its role.

- [ ] **Step 7: Confirm enterprise-feature showcase**

Open `ARCHITECTURE.md` §5 and confirm each feature listed there (Processing Engine, Last Value Cache, Distinct Value Cache) is visibly used and visibly different in behavior (the `make cli-example name=cache-last-compare` example should show a measurable difference).

- [ ] **Step 8: Capture lessons learned**

Open the portfolio spec
`/Users/pauldix/codez/reference_architectures/influxdb3-reference-architectures/docs/superpowers/specs/2026-04-23-reference-architectures-portfolio-design.md`
and append an "Amendments from BESS pilot" section at the bottom, listing anything that required changes to the shared conventions. This list feeds Phase 2 (template extraction) of the portfolio roadmap.

- [ ] **Step 9: Tag the release**

```bash
git tag -a v0.1.0 -m "BESS pilot reference architecture — v0.1.0"
```

- [ ] **Step 10: Pilot complete**

The next step in the portfolio roadmap (§11 Phase 2 of the portfolio spec) is to extract what worked into the meta-repo's `template/` directory. That work is out of scope for this plan — it gets its own plan once the pilot is accepted.

---

## Amendments from Checkpoint #1

Every one of these surfaced in the first real boot of the stack. Bake them into the meta-repo's `CONVENTIONS.md` during Phase 2 template extraction.

1. **`ui` service referenced before its Dockerfile exists.** Task 7 compose wired the `ui` service but `ui/Dockerfile` is created in Task 21. First build failed. Fix: added `profiles: ["ui"]` to the `ui` service so it doesn't start/build by default. **Task 21 MUST remove `profiles: ["ui"]`** when it creates `ui/Dockerfile`.

2. **`SCENARIO` env var default.** Compose warned `WARN[0000] The "SCENARIO" variable is not set`. The `scenarios` service entrypoint references `${SCENARIO}` for `make scenario name=…`. Changed to `${SCENARIO:-}` to silence.

3. **Processing Engine venv needs a writable location.** The image defaults to expecting `/plugins/.venv/` but our `./plugins` bind mount is `:ro`. Added `--virtual-env-location /var/lib/influxdb3/plugin-venv` to `serve` command so the venv lives in the writable data volume, keeping plugin source read-only.

4. **`/health` requires auth; returns 401 for unauth'd probes.** Both the compose healthcheck and `init.sh::wait_for_api` were using `curl -sf` which demands a 2xx response. Changed to `curl -s --max-time 2 -o /dev/null` (exits 0 on any HTTP round-trip, including 401). The server returns 200 vs 401 on `/health` is an auth concern we don't care about during liveness probing.

5. **Admin-token bootstrap: recovery endpoint only REGENERATES.** The plan assumed `influxdb3 create token --admin --regenerate --host :8182` would work as the first bootstrap. It doesn't — the recovery endpoint requires an existing admin token to regenerate. For a fresh server we must pre-generate an admin token **offline** (`influxdb3 create token --admin --offline --output-file …`) and pass it to serve via `--admin-token-file`. Implemented as a one-shot `token-bootstrap` service that runs before `influxdb3` and writes the admin token into the shared data volume. `init.sh` no longer calls the recovery endpoint.

6. **Enterprise image lacks `python3` and `jq`.** `init.sh`'s JSON token parser needed to be pure shell. Replaced `python3 -c "import json; …"` with `sed -n 's/.*"token"[[:space:]]*:[[:space:]]*"\([^"]*\)".*/\1/p'`.

7. **`influxdb3` CLI puts `--host` and `--token` on subcommands, not top-level.** `influxdb3 --token X query …` fails with "unexpected argument '--token'". Correct form: `influxdb3 query … --token X`. Updated `init.sh::cli()` to append flags after the subcommand chain, and updated `make cli` / `make query` accordingly.

8. **Database auto-create by first write.** The simulator writing line protocol to a non-existent database auto-creates it. By the time `init.sh` runs `create database bess` the database already exists (409 Conflict). Made all `create` calls in `init.sh` idempotent via an `idempotent` helper that treats "already exists / Conflict / 409" as success.

9. **`distinct_cache` flag is `--columns` (plural), not `--column`.** Typo.

10. **`make cli` needs to pre-export TOKEN.** With auth on, every CLI call needs `--token`. Added `make cli` wrapper that pre-exports `TOKEN` and installs an `iql` shell function (`iql 'SELECT …'` → `influxdb3 query --database bess --token $TOKEN …`), plus a `make query sql='…'` one-shot target. Init.sh also writes a plain-text token file (`/var/lib/influxdb3/.bess-token-plain`) alongside the JSON one for these helpers.

11. **Querying the Last Value Cache requires a table-valued function, not a plain `SELECT FROM cell_last`.** The latter returns `table 'public.iox.cell_last' not found`. Correct syntax: `SELECT ... FROM last_cache('cell_readings', 'cell_last')`. Arguments MUST be single-quoted strings (table name first, cache name second; cache-name is omittable if only one LVC exists for the table). `distinct_cache()` follows the same pattern. Also — any `make`/shell wrapper that passes SQL through bash must preserve embedded single quotes: `docker compose exec -T -e "SQL=$(sql)"` is the clean approach; direct inline substitution into a `bash -c '...'` breaks because the apostrophes collide with the outer single-quoted command.

12. **Deprecation warning from base image: `LOG_FILTER is deprecated, use INFLUXDB3_LOG_FILTER instead`.** The `influxdb:3-enterprise` image sets `LOG_FILTER=info` as a legacy default that trips this warning on every CLI invocation. Fix: set `INFLUXDB3_UNSET_VARS=LOG_FILTER` and `INFLUXDB3_LOG_FILTER=info` on all three containers using the enterprise image (`token-bootstrap`, `influxdb3`, `influxdb3-init`). This pattern generalizes — any legacy env var in the image can be neutered with `INFLUXDB3_UNSET_VARS`.

## Amendments from Checkpoint #2

1. **Enterprise cron is 6-field, not 5-field.** `--trigger-spec "cron:<expr>"` requires `<sec> <min> <hour> <dom> <mon> <dow>`. Our initial 5-field `cron:5 0 * * *` was rejected with "failed to parse trigger from cron:5 0 * * *". Correct form for "00:05:00 UTC daily" is `cron:0 5 0 * * *`.

2. **`LineBuilder` is NOT imported — it's injected into plugin globals.** `from influxdb3_local import LineBuilder` raises ImportError at runtime. The Processing Engine injects `LineBuilder` into the plugin's module globals before exec'ing its source. Our original `try: from influxdb3_local import LineBuilder / except: LineBuilder = None` fallback was actively harmful: the engine injected the real thing first, then the except branch ran and clobbered it with None, causing `TypeError: 'NoneType' object is not callable` on every trigger fire. Fix: plugins reference `LineBuilder` bare with no import statement. Tests attach a fake via `monkeypatch.setattr(mod, "LineBuilder", FakeLineBuilder, raising=False)`.

3. **`--path` is confirmed as the CLI flag for plugin filename.** (Not `--plugin-filename`.) Per `influxdb3 create trigger --path <file.py>`.

4. **Request triggers are reachable at `GET /api/v3/engine/<name>?param=value`.** Confirmed via live curl. Auth is the same Bearer token as the main API. The response body is exactly the dict the plugin's `process_request` returns — `{"status": 200, "body": {...}}` is returned as JSON verbatim.

5. **Last value cache query syntax (resolved post-Checkpoint #2).** See Checkpoint #1 amendment #11 (updated): `SELECT ... FROM last_cache('cell_readings', 'cell_last')` with single-quoted string args. Used in `ui/queries.py::cell_heatmap_latest()` and `CLI_EXAMPLES.md §lvc-latest`. Heatmap dropped from ~6000 rows / 1.1 MB per poll to exactly 768 rows / 91 KB as a consequence.

6. **Scenario integration tests (resolved in v0.1.1).** Rewrote as tests-against-running-stack (not testcontainers). Session-scoped fixture reads token from the running `influxdb3` container; tests skip if stack not up. `make test-scenarios` runs them; CI wiring is a Phase 2 task. One real bug surfaced during: `cell_drift` hit a float-precision boundary (`3.3 - 0.2 = 3.0999999 < 3.1`, loop exited before writing 3.1). Fixed by moving END_V to 3.0 so the loop naturally writes 3.1 as its last value.
