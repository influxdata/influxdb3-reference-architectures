# InfluxDB 3 Enterprise Reference Architectures — Portfolio Design

**Date:** 2026-04-23
**Status:** Approved portfolio-level design. Per-repo designs and plans follow separately.

## 1. Goals and audience

Ship eight open-source, independent GitHub repos under the `influxdata` org, each a complete, runnable reference architecture for InfluxDB 3 Enterprise targeted at a specific vertical. Plus a ninth meta-repo that indexes and binds the portfolio together.

The repos serve two audiences:

1. **Developers** evaluating InfluxDB 3 Enterprise for a specific vertical.
2. **AI coding agents** using these repos as grounded examples when asked to build in that vertical.

Both audiences benefit from the same property: **reading two files (`README.md` and `ARCHITECTURE.md`) should be enough to understand what the repo does and how to adapt it.**

## 2. The portfolio

Eight vertical reference architectures plus one meta-repo. All under the `influxdata` GitHub org. License: **Apache 2.0**.

| # | Repo name | Vertical |
|---|-----------|----------|
| 1 | `influxdb3-ref-iiot` | IIoT / Factory Floor Monitoring |
| 2 | `influxdb3-ref-bess` | Battery Energy Storage Systems |
| 3 | `influxdb3-ref-network-telemetry` | Network Telemetry |
| 4 | `influxdb3-ref-renewables` | Renewable Energy (Solar + Wind) |
| 5 | `influxdb3-ref-ev-charging` | EV Charging Network |
| 6 | `influxdb3-ref-fleet-telematics` | Connected Vehicle / Fleet Telematics |
| 7 | `influxdb3-ref-datacenter` | Data Center / Infrastructure Monitoring |
| 8 | `influxdb3-ref-oilgas` | Oil & Gas Upstream / SCADA |
| — | `influxdb3-reference-architectures` | Meta-repo: index, shared template, conventions |

## 3. Success criteria (per repo)

Every vertical repo must satisfy all five:

1. `git clone && docker compose up` shows live data in the dashboard within two minutes of completing one-time license validation.
2. `README.md` contains an architecture diagram, a "what's in this repo" table, and a five-minute quickstart.
3. At least two scenario scripts demonstrably trigger Processing Engine plugins and produce observable outcomes (alerts, rollups, API responses).
4. CI is green: plugin unit tests on every push, scenario tests on every PR, smoke test on `main` push and nightly.
5. Every file a reader is likely to adapt opens with a short comment explaining its role.

## 4. Shared conventions across all repos

### 4.1 Language and runtime

- **Python-first, everywhere.** Simulator, Processing Engine plugins, UI backend, tests, and CLI glue are all Python.
- Client library: `influxdb3-python`.
- UI stack: **FastAPI + HTMX + Jinja2 + uPlot** (vendored, ~55 KB of JS total, no frontend toolchain).

### 4.2 Repo directory layout

Every vertical repo has this skeleton:

```
influxdb3-ref-<usecase>/
├── README.md                   # quickstart, architecture diagram, adaptation guide
├── ARCHITECTURE.md             # deep dive: schema, design choices, scaling notes
├── SCENARIOS.md                # curated scenarios and how to run them
├── CLI_EXAMPLES.md             # curated influxdb3 CLI commands for this vertical
├── LICENSE                     # Apache 2.0
├── .env.example                # INFLUXDB3_ENTERPRISE_EMAIL= and other tunables
├── docker-compose.yml
├── Makefile                    # make up | down | scenario name=X | cli | test
├── pyproject.toml              # single project (simulator + ui + tests + plugin tests)
├── diagrams/
│   ├── architecture.mmd        # mermaid source
│   └── architecture.png        # rendered
├── influxdb/
│   ├── init.sh                 # creates DBs, tokens, caches, triggers on first boot
│   └── schema.md               # tables, tags, fields, retention
├── plugins/
│   ├── wal_<name>.py           # Processing Engine WAL triggers
│   ├── schedule_<name>.py      # Processing Engine schedule triggers
│   └── request_<name>.py       # Processing Engine request triggers
├── simulator/
│   ├── main.py
│   ├── signals.py              # domain-specific signal generators
│   ├── signals_base.py         # shared primitives (copied, not shared package)
│   └── scenarios/<name>.py
├── ui/
│   ├── app.py                  # FastAPI routes
│   ├── queries.py              # all SQL named and documented
│   ├── templates/              # Jinja2
│   └── static/                 # uPlot, HTMX, app.css
└── tests/
    ├── test_smoke.py
    ├── test_plugins/
    └── test_scenarios/
```

### 4.3 Makefile surface (identical across repos)

| Target | Effect |
|--------|--------|
| `make up` | Prompts for `INFLUXDB3_ENTERPRISE_EMAIL` if `.env` missing it, writes `.env`, runs `docker compose up`. |
| `make down` | `docker compose down` (preserves `influxdb-data` volume). |
| `make clean` | `docker compose down -v` (drops volume — requires re-validation on next `up`). |
| `make scenario name=<foo>` | Runs scenario `<foo>` against the running stack. |
| `make scenario list` | Lists available scenarios with descriptions. |
| `make cli` | Drops into an interactive shell inside the `influxdb3` container with the CLI on PATH. |
| `make cli-example name=<foo>` | Runs a named curated CLI example from `CLI_EXAMPLES.md`. |
| `make test` | Runs all three tiers. |
| `make test-unit` / `make test-scenarios` / `make test-smoke` | Individual tiers. |

### 4.4 Enterprise license validation flow

InfluxDB 3 Enterprise requires email-based license validation on first startup.

1. `make up` checks for `INFLUXDB3_ENTERPRISE_EMAIL` in `.env`. If absent, prompts for it and writes `.env`.
2. `docker compose up` starts.
3. Terminal prints a clear banner: *"InfluxDB 3 Enterprise is waiting for license validation. Check `<email>` and click the validation link. The simulator and UI will start automatically once validation completes."*
4. Compose uses `depends_on: condition: service_healthy` so `simulator` and `ui` do not start until the `influxdb3` healthcheck passes, which happens only after validation completes.
5. The `influxdb-data` named volume preserves validation across `make down`/`make up`. `make clean` drops it and requires re-validation.

### 4.5 Compose stack (single-node repos)

Four services per compose file:

- **`influxdb3`** — `influxdb:3-enterprise` image; mounts `./plugins` into `/plugins`; healthcheck on `/health`.
- **`simulator`** — built from `./simulator/Dockerfile`; waits for `influxdb3` health; writes line protocol.
- **`ui`** — built from `./ui/Dockerfile`; waits for `influxdb3` health; serves port 8080.
- **`scenarios`** — one-shot service behind profile `scenarios`; invoked via `make scenario`.

Ports: `8181` (InfluxDB HTTP API), `8080` (UI). Object store: local `file` backend. README includes a "scaling to production" section describing swap-in to S3/GCS/Azure and K8s/Helm.

### 4.6 Compose stack (multi-node repos)

For repos whose feature map calls for clustered deployment (Network Telemetry, Fleet, Data Center), the compose file replaces `influxdb3` with three services: `influxdb3-ingest`, `influxdb3-query`, `influxdb3-compact`, sharing an object-store volume. Same simulator/ui/scenarios layout on top. Plugin directory mounted into the `process`-mode node(s).

### 4.7 Replication (O&G only)

The O&G repo adds a second `influxdb3-edge` service that replicates to `influxdb3-core`, modeling the well-pad → central pattern.

## 5. Simulator pattern

### 5.1 Skeleton (identical across repos)

```python
# simulator/main.py
def main():
    cfg = load_config()              # rate, cardinality, duration, target
    writer = InfluxDB3Writer(cfg)    # batched line-protocol client
    signals = build_signals(cfg)     # repo-specific
    scheduler = Scheduler(cfg.rate)
    for tick in scheduler:
        for signal in signals:
            writer.write(signal.tick(tick))
    writer.flush()
```

### 5.2 Signal primitives (`simulator/signals_base.py`)

Shared primitive library copied (not packaged) into every repo: `sinusoid`, `random_walk`, `step`, `burst`, `jitter`, `correlation`. Repos compose these in `signals.py` to produce plausible domain shapes.

### 5.3 Domain signals (per repo)

| Repo | Signal classes |
|------|----------------|
| IIoT | `MachineState`, `VibrationSensor`, `TemperatureSensor`, `PartCount` |
| BESS | `CellVoltage`, `CellTemperature`, `PackCurrent`, `SoC`, `SoH`, `InverterPower` |
| Network | `InterfaceCounters`, `BGPSession`, `FlowRecord`, `Latency` |
| Renewables | `InverterPower`, `IrradianceMetStation`, `WindSpeed`, `CurtailmentCommand` |
| EV Charging | `ChargeSession`, `StationPower`, `StationStatus` |
| Fleet | `GPSPosition`, `CANSignal`, `DTCEvent` |
| Data Center | `RackPower`, `RackThermal`, `PDUReading`, `ServerSensor` |
| O&G | `WellheadPressure`, `FlowRate`, `PumpState`, `TankLevel`, `GasComposition` |

### 5.4 Configuration

Env vars: `SIM_RATE` (points/sec), `SIM_CARDINALITY` (scale factor), `SIM_DURATION` (optional), `SIM_SEED` (reproducibility for scenario tests). Defaults per repo are realistic for the domain.

### 5.5 Scenarios

`simulator/scenarios/<name>.py`. Each scenario:

1. Declares a one-line description for `make scenario list`.
2. Implements `run(writer, duration)` injecting a specific event pattern.
3. Prints step-by-step so readers can follow along.

Scenarios exist to make Processing Engine triggers demonstrable (e.g., BESS `thermal_runaway` pushes cell[42] temperature up at 2°C/s until the `wal_thermal_runaway` plugin fires).

## 6. Processing Engine plugin pattern

### 6.1 File layout

One file per trigger. Filename encodes trigger type:

- `plugins/wal_<name>.py` — WAL triggers (fire on writes).
- `plugins/schedule_<name>.py` — schedule triggers (fire on cron).
- `plugins/request_<name>.py` — request triggers (fire on HTTP).

### 6.2 Required header comment (every plugin)

Purpose, binding (table/cron/path + args), side effects. Three lines minimum. The header is what makes a plugin legible without reading the body.

### 6.3 Auto-installation

`influxdb/init.sh` scans `plugins/` on first boot and registers each file as a trigger via `influxdb3 create trigger`, convention-mapping filename prefix to trigger type and reading binding details from a header block.

### 6.4 Per-repo plugin inventory

| Repo | Plugins |
|------|---------|
| IIoT | `wal_oee_drop_detector`, `schedule_oee_rollup`, `request_machine_status` |
| BESS | `wal_thermal_runaway`, `schedule_soh_daily`, `request_pack_health` |
| Network | `wal_interface_errors`, `schedule_topN_flows`, `request_device_status` |
| Renewables | `wal_inverter_fault`, `schedule_curtailment_rollup`, `request_site_forecast` |
| EV Charging | `wal_session_close`, `schedule_station_utilization`, `request_last_session` |
| Fleet | `wal_geofence_breach`, `schedule_daily_drive_summary`, `request_vehicle_state` |
| Data Center | `wal_rack_power_anomaly`, `schedule_hourly_pue`, `request_rack_status` |
| O&G | `wal_pressure_spike`, `schedule_wellpad_daily`, `request_well_status` |

Each plugin has a corresponding unit test in `tests/test_plugins/` and a line in the README's "Processing Engine triggers" table.

## 7. UI pattern

### 7.1 Layout (shared across repos)

Single-page dashboard:

- Top KPI row (4–6 tiles).
- Main chart grid (2–4 time-series charts, uPlot).
- Recent-events table (alerts written by Processing Engine WAL triggers).
- One domain-specific view (e.g., BESS pack/cell heatmap; Fleet map; Network topology; Data Center rack elevation).

### 7.2 Data flow

Jinja2 renders the shell. HTMX polls partial endpoints on intervals (KPIs every 2s, charts every 5s, configurable). Each partial route invokes a named function in `ui/queries.py`, runs SQL against InfluxDB 3, renders a Jinja2 fragment. Charts return JSON arrays; a small `app.js` feeds uPlot.

### 7.3 `queries.py` discipline

All SQL lives here. Every query function has a docstring with (a) what it returns, (b) which UI route uses it, (c) why this query is the right one for this vertical. This file is the primary teaching artifact for domain modeling.

### 7.4 Request-trigger integration

Where a repo exposes a Processing Engine request trigger, the UI has a panel that calls it via `fetch`, demonstrating both direct querying and Processing-Engine-mediated querying side by side. The README calls out the distinction.

### 7.5 Styling

One `app.css`, dark-friendly, no CSS framework. Competent, not fancy.

## 8. Per-repo enterprise feature map

Every repo demonstrates the **common core**: ingest, Last Value Cache, Processing Engine (≥1 WAL trigger and ≥1 Schedule trigger), and the custom UI.

On top of that, each repo showcases the enterprise features that fit its vertical naturally:

| Repo | Extra enterprise features |
|------|---------------------------|
| IIoT | Distinct Cache (machine/part IDs), Request trigger (alert webhook API) |
| BESS | Distinct Cache (cell IDs), Schedule trigger (rolling SoH computation) |
| Network | Multi-node ingest + Enterprise Compactor (scale story) |
| Renewables | Schedule trigger (curtailment calc), Request trigger (forecast API) |
| EV Charging | Request trigger (session lookup API), Distinct Cache (station IDs) |
| Fleet | Multi-node ingest, Distinct Cache (VINs) |
| Data Center | Enterprise Compactor under load, Multi-node query (read scaling) |
| O&G | Replication (edge-to-core), Schedule trigger (well-pad aggregates) |

## 9. Testing pattern

Three tiers per repo, runnable individually or all together.

### 9.1 Tier 1 — Plugin unit tests (`tests/test_plugins/`)

One file per plugin. Plugins tested as pure functions with fake payloads and a recording `influxdb3_local` fake. No Docker required. Fast (<1s per file).

### 9.2 Tier 2 — Scenario tests (`tests/test_scenarios/`)

One test per curated scenario. Uses `testcontainers-python` to run the real `influxdb:3-enterprise` image with real plugins mounted. Test writes synthetic data directly and asserts observable outcomes. Moderate (30–60s per scenario). Runs on PR. Skipped when Docker unavailable.

### 9.3 Tier 3 — Smoke test (`tests/test_smoke.py`)

Runs `make up`, waits for health, lets the simulator run ~30s, queries 3–5 key metrics, hits UI partials, runs `make down`. Slow (2–3 min). Runs on `main` push + nightly.

### 9.4 CI

GitHub Actions, one workflow per repo, shared template:

- `unit.yml` — every push.
- `scenarios.yml` — every PR.
- `smoke.yml` — push to `main` + nightly.
- `lint.yml` — `ruff check`, `ruff format --check`, `markdownlint`.

### 9.5 License-validated volume for CI

A maintainer-validated `influxdb-data` volume is stored as a GitHub Actions artifact (refreshed periodically). `smoke.yml` restores it so CI runs do not require clicking a validation email. A `FOR_MAINTAINERS.md` file in each repo documents how to refresh it.

## 10. Meta-repo structure

`influxdb3-reference-architectures/` contains:

- `README.md` — index of the eight repos with a comparison table and a "which repo should I start with?" decision tree.
- `CONVENTIONS.md` — the shared conventions documented in this spec, turned into prescriptive guidance for contributors (extracted after the pilot ships, not before — see §11).
- `template/` — the canonical scaffolding new repos copy from (directory skeleton, Makefile, compose templates, CI workflows, `.env.example`, `signals_base.py`, UI base template).
- `docs/superpowers/specs/` — portfolio-level design docs and per-repo design docs.
- `CONTRIBUTING.md`, `CODE_OF_CONDUCT.md`, `LICENSE` (Apache 2.0).

## 11. Implementation sequencing

### Phase 1 — BESS pilot (manual end-to-end validation)

Build `influxdb3-ref-bess` first, single-node, exercising the full common core (LVC, Distinct Cache, WAL/Schedule/Request triggers, UI). Manual validation includes running every scenario by hand, watching the UI, verifying alerts and rollups, running `make cli-example` for each curated CLI command. This phase battle-tests every shared convention before we multiply the work.

### Phase 2 — Template extraction

Extract what actually worked from BESS into the meta-repo's `template/` directory. Write `CONVENTIONS.md` from real-world evidence, not from this spec's predictions. Fix any portfolio-level choices this spec made that didn't pan out in practice, with an amendment log at the bottom of this document.

### Phase 3 — Multi-node trailblazer (Network Telemetry)

Build `influxdb3-ref-network-telemetry` next to lock in the clustered compose shape (ingest/query/compact services, shared object-store volume, plugin distribution) before three other multi-node repos depend on the same pattern.

### Phase 4 — Parallel build

The remaining six repos proceed in parallel, each with its own design doc and implementation plan, each starting from the template:

- Single-node: IIoT, Renewables, EV Charging.
- Clustered (ingest/query/compact): Fleet, Data Center.
- Replication (edge + core): O&G.

Each parallel track produces both its vertical repo and a meta-repo PR adding itself to the index.

### Phase 5 — Meta-repo polish

Index README comparison table, decision tree, contribution guide, end-to-end link audit.

## 12. Open questions and future work

- **GitHub Actions license-volume refresh policy**: how often, by whom, how tested.
- **Docs site?** Whether to publish a unified docs site (e.g., MkDocs) on top of the eight repos, or rely on GitHub's rendered READMEs. Decide after pilot.
- **K8s/Terraform companion artifacts**: deliberately deferred out of v1 for all eight repos. Each repo's ARCHITECTURE.md discusses the shape of a production deployment without shipping manifests. If a specific vertical proves popular, add manifests as a follow-up.
- **Replication topology for O&G**: the detailed edge/core pattern is resolved in the O&G per-repo design, not here.
- **Per-repo naming consistency for plugin binding arguments**: finalized during BESS pilot, then codified in `CONVENTIONS.md`.
