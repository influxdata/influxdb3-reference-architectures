# InfluxDB 3 Enterprise Reference Architectures

A portfolio of open-source, runnable reference architectures for **InfluxDB 3 Enterprise**, each targeted at a specific vertical. Every repo is independent: clone it, run `docker compose up`, and see live data in a dashboard within two minutes.

The repos serve two audiences:

1. **Developers** evaluating InfluxDB 3 Enterprise for a specific vertical.
2. **AI coding agents** using these repos as grounded examples when asked to build in that vertical.

Both benefit from the same property: reading two files (`README.md` and `ARCHITECTURE.md`) is enough to understand what the repo does and how to adapt it.

## The portfolio

| # | Repo | Vertical | Status |
|---|------|----------|--------|
| 1 | [`influxdb3-ref-bess`](https://github.com/influxdata/influxdb3-ref-bess) | Battery Energy Storage Systems | ✅ Available |
| 2 | [`influxdb3-ref-iiot`](https://github.com/influxdata/influxdb3-ref-iiot) | IIoT / Factory Floor Monitoring | ✅ Available |
| 3 | [`influxdb3-ref-network-telemetry`](https://github.com/influxdata/influxdb3-ref-network-telemetry) | Network Telemetry | ✅ Available |
| 4 | [`influxdb3-ref-auto-manufacturing`](https://github.com/influxdata/influxdb3-ref-auto-manufacturing) | Motor Vehicle Manufacturing | ✅ Available |
| 5 | [`influxdb3-ref-scientific-infrastructure`](https://github.com/influxdata/influxdb3-ref-scientific-infrastructure) | Scientific Research Infrastructure | ✅ Available |
| 6 | `influxdb3-ref-renewables` | Renewable Energy (Solar + Wind) | 🚧 Coming soon |
| 7 | `influxdb3-ref-ev-charging` | EV Charging Network | 🚧 Coming soon |
| 8 | `influxdb3-ref-fleet-telematics` | Connected Vehicle / Fleet Telematics | 🚧 Coming soon |
| 9 | `influxdb3-ref-datacenter` | Data Center / Infrastructure Monitoring | 🚧 Coming soon |
| 10 | `influxdb3-ref-oilgas` | Oil & Gas Upstream / SCADA | 🚧 Coming soon |

## What's different across the available repos

The five shipped repos share a template but make different choices in service of their domain (four are Python-first with a FastAPI/HTMX/uPlot UI and three-tier tests; the fifth is Grafana-only, no Python). The variety is intentional — readers shopping for a starting point should pick the repo whose patterns match their problem.

| Dimension | `influxdb3-ref-bess` | `influxdb3-ref-iiot` | `influxdb3-ref-network-telemetry` | `influxdb3-ref-auto-manufacturing` | `influxdb3-ref-scientific-infrastructure` |
|---|---|---|---|---|---|
| Domain | Battery Energy Storage Systems | Discrete-assembly factory floor | Data-center Clos fabric monitoring | Paint-shop telemetry cleanup (historian augmentation) | Scientific research infrastructure (DAQ / compute / storage node monitoring) |
| Compose shape | Single node | Single node | **5-node cluster** (2 ingest + query + compact + process,query) | Single node + **plugin-installer one-shot** (no simulator service at all) | Single node + **three remote Telegraf agents** (one fed by collectd) + **Grafana** — no custom UI, no Python |
| Cardinality story | 768 cells (high entity count) | 24 machines + ~700K parts/day (high event-tag count drives DVC) | ~1,024 fabric interfaces + 128 BGP sessions + ~5k flow records/sec (DVC drives a typeahead with thousands of distinct src_ips) | 1 station (deliberately tiny — the `station` tag is the unit of growth; scenarios add stations live with zero config) | 3 hosts (deliberately tiny — the `host` tag is the unit of growth; a node is one Telegraf config) |
| Write rate | ~2,000 pts/s | ~300 pts/s | **~10,000 pts/s** | ~40 pts/s (10 Hz feed × pipeline stages) | ~15 pts/s (3 hosts × 5 measurements × 1 Hz), **nanosecond timestamps** end to end |
| Plugins | 3: 1 WAL · 1 Schedule · 1 Request | 4: **2 WAL** · 1 Schedule · 1 Request | 4: **0 WAL** · 2 Schedule · 2 Request | 6-stage pipeline: **5 registry plugins + 1 local stub** (2 WAL · 4 Schedule) | 1 registry plugin (`downsampler`) · **5 Schedule** · 0 WAL · 0 Request |
| Plugin provisioning | Local `plugins/` bind mount | Same | Same | **Installed from the plugin registry via `POST /api/v3/plugins/files`** — pinned + sha256-verified, no bind mount | Installed from the plugin registry via `POST /api/v3/plugins/files` (installer copied from auto-manufacturing) |
| WAL patterns shown | Transition-detect (thermal threshold) | Transition-detect (downtime) **and** windowed/derivative (scrap rate) | None — multi-node WAL ownership is awkward; this repo leans on schedule plugins instead | **Chained WAL** (a plugin's write fires the next table's trigger) + streaming stateful IIR filter | None |
| Schedule cadence + format | Daily, `cron:0 5 0 * * *` | Shift-based, `cron:0 0 6,14,22 * * *` | **Live, `every:5s`** | Live, `every:1s` (generator/resample/downsample) + `every:10s` (chronos forecast) | **Wall-clock aligned `cron:*/5 * * * * *`** with `offset=5s,window=5s`, so each 5 s bucket is read complete and written once |
| Schedule plugin write path | `LineBuilder` + `influxdb3_local.write()` (local) | Same | **httpx → ingest node's `/api/v3/write_lp`** (cross-node, plugin runs on dedicated process node, must round-trip back through ingest) | `LineBuilder` / `write_sync` (local) | Plugin's own write (local, single node) |
| Request-trigger UI integration | Diagnostic panel calls `pack_health` endpoint | Andon panel direct-fetches `andon_board`; same response drives the chart history | **Three patterns side-by-side, each with its own latency badge:** SQL via FastAPI · SQL from browser via DVC TVF · request plugin from browser | None — SQL-via-FastAPI only (request trigger deliberately retired; the forecast is always on the chart) | None — **Grafana only**, provisioned from files (InfluxDB SQL datasource over Flight SQL, token via `$__file{}`) |
| Per-table retention | None | None | **24h on `fabric_health`** — exclusive demo of per-table retention in the portfolio | 24h on raw + all intermediate stages (the clean rollup is what you keep) | **5 y on the database**, inherited by all ten tables; writes older than the cutoff are rejected (400) |
| Domain-specific view | Pack/cell heatmap | Andon board grid + per-line OEE breakdown | Aggregate-led: fabric-state banner, layered throughput chart, top-talkers, source-IP typeahead+detail, active-anomalies (drill-on-anomaly only) | Raw-vs-processed overlay chart (red/green, group-delay-compensated) + false-alarm comparison + live pipeline panel | Grafana fleet overview (UP/DOWN + uptime per node, alert list) + three identical node dashboards generated from one template |
| Aggregate KPI | Pack SoH/SoC daily rollup | A × P × Q live + per-shift `shift_summary` rollup | Live `fabric_health` rollup written by the schedule plugin every 5s | Compression ratio from the downsampler's own `record_count` metadata + in-database chronos forecast | 5 s rollups with `record_count = 5` on every bucket; threshold alerts (cpu / temperature / node down) that fire and go nowhere — the storage node flaps and runs a sine-wave temperature so something is always alerting |

If you're evaluating which patterns to copy:

- **Pick the bess patterns** when your domain is dominated by high entity cardinality and slow-moving rollups (rollup-once-a-day; LVC/DVC for entity inventory).
- **Pick the iiot patterns** when your domain has high event cardinality, real-time alerting on multiple signal types, or a request-trigger that should genuinely replace a backend service (the andon board pattern).
- **Pick the network-telemetry patterns** when you need a multi-node InfluxDB 3 Enterprise cluster: this repo is the trailblazer for the multi-node compose template, the cross-node plugin write-back convention, and the three-pattern UI (SQL via backend / SQL from browser / request plugin from browser).
- **Pick the auto-manufacturing patterns** when your problem is a data-processing pipeline rather than a monitoring surface: registry-based plugin install over the API, chained WAL triggers (stage N's write fires stage N+1), signal-processing plugins (resample / IIR filter / median downsample / zero-shot forecast), and a demo whose data source is itself a Processing Engine plugin — no simulator service.
- **Pick the scientific-infrastructure patterns** when your world is Telegraf agents on many hosts and Grafana on top: per-agent filtering and reshaping (`namepass` / `fieldinclude` / `taginclude` / `tagpass` / `rename`) so collectd and mock sources converge on one schema, nanosecond timestamps (`precision = "1ns"`), a 5-year retention database, wall-clock-aligned Processing Engine downsampling into dashboard tables, and Grafana provisioned entirely from files (datasource, dashboards, alerts).

## Shared conventions

Every reference architecture in the portfolio follows the same conventions so that a reader who learns one can navigate the others quickly:

- **Python-first.** Simulator, Processing Engine plugins, UI backend, tests, and CLI glue are all Python. (`influxdb3-ref-scientific-infrastructure` is the deliberate exception: Telegraf + collectd + Grafana, no Python beyond the copied plugin installer, no custom UI, manual validation gates instead of tests.)
- **Client library:** [`influxdb3-python`](https://github.com/InfluxCommunity/influxdb3-python).
- **UI stack:** FastAPI + HTMX + Jinja2 + uPlot (vendored, no frontend toolchain) — or Grafana provisioned from files where the domain's operators already live in Grafana.
- **One-command demo:** `docker compose up` brings up the full stack.
- **Processing Engine plugins** (WAL, Schedule, Request triggers) live in `plugins/` and are the centerpiece of every demo.
- **License:** Apache 2.0.

For the technical conventions and gotchas every repo follows, see [`CONVENTIONS.md`](CONVENTIONS.md).

## What's in this repo

This is the **meta-repo** for the portfolio. It holds the portfolio-level design, shared conventions, and links out to the per-vertical repos above.

```
docs/superpowers/
├── specs/   # Portfolio-level design docs
└── plans/   # Per-repo implementation plans
```

## License

[Apache 2.0](LICENSE).
