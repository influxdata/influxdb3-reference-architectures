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
| 8 | [`influxdb3-ref-fleet-telematics`](https://github.com/influxdata/influxdb3-ref-fleet-telematics) | Connected Vehicle / Fleet Telematics | ✅ Available |
| 9 | `influxdb3-ref-datacenter` | Data Center / Infrastructure Monitoring | 🚧 Coming soon |
| 10 | `influxdb3-ref-oilgas` | Oil & Gas Upstream / SCADA | 🚧 Coming soon |

## What's different across the available repos

The six shipped repos share a template but make different choices in service of their domain (five are Python-first with a FastAPI/HTMX/uPlot UI and tiered tests; `influxdb3-ref-scientific-infrastructure` is Grafana-only, no Python). The variety is intentional — readers shopping for a starting point should pick the repo whose patterns match their problem.

| Dimension | `influxdb3-ref-bess` | `influxdb3-ref-iiot` | `influxdb3-ref-network-telemetry` | `influxdb3-ref-auto-manufacturing` | `influxdb3-ref-scientific-infrastructure` | `influxdb3-ref-fleet-telematics` |
|---|---|---|---|---|---|---|
| Domain | Battery Energy Storage Systems | Discrete-assembly factory floor | Data-center Clos fabric monitoring | Paint-shop telemetry cleanup (historian augmentation) | Scientific research infrastructure (DAQ / compute / storage hosts) | Regional trucking fleet (24 trucks between 5 warehouses) |
| Compose shape | Single node | Single node | **5-node cluster** (2 ingest + query + compact + process,query) | Single node + **plugin-installer one-shot**, no simulator | Single node + **3 remote Telegraf agents** (1 fed by collectd) + Grafana | Single node + plugin-installer one-shot + simulator |
| Cardinality story | 768 cells (high entity count) | 24 machines + ~700K parts/day (high event-tag count drives DVC) | ~1,024 fabric interfaces + 128 BGP sessions + ~5k flow records/sec (DVC drives a typeahead with thousands of distinct src_ips) | 1 station (the `station` tag is the unit of growth) | 3 hosts (the `host` tag is the unit of growth) | 24 trucks (the `vehicle_id` tag is the unit of growth; every geo attribute is a field) |
| Write rate | ~2,000 pts/s | ~300 pts/s | **~10,000 pts/s** | ~40 pts/s (10 Hz feed × pipeline stages) | ~15 pts/s, **nanosecond timestamps** | ~30 pts/s, plus ~2× from in-place enrichment |
| Plugins | 3: 1 WAL · 1 Schedule · 1 Request | 4: **2 WAL** · 1 Schedule · 1 Request | 4: **0 WAL** · 2 Schedule · 2 Request | 6-stage pipeline: **5 registry + 1 local** (2 WAL · 4 Schedule) | 1 registry plugin (`downsampler`): 5 Schedule · 0 WAL · 0 Request | 1 registry plugin (`geo_enrichment`: 3 WAL · 1 Request) + 3 local (1 WAL · 1 Schedule · 1 Request) |
| Plugin provisioning | Local `plugins/` bind mount | Same | Same | **Registry via `POST /api/v3/plugins/files`**, pinned + sha256-verified | Registry via `POST /api/v3/plugins/files` (same installer) | Registry + local, both via `POST /api/v3/plugins/files` (same installer) |
| WAL patterns shown | Transition-detect (thermal threshold) | Transition-detect (downtime) **and** windowed/derivative (scrap rate) | None — multi-node WAL ownership is awkward; this repo leans on schedule plugins instead | **Chained WAL** + streaming stateful IIR filter | None | **In-place enrichment** (geo fields merged into the raw row) + chained WAL (geofence enter/exit fires on the enrichment's write) |
| Schedule cadence + format | Daily, `cron:0 5 0 * * *` | Shift-based, `cron:0 0 6,14,22 * * *` | **Live, `every:5s`** | Live, `every:1s` (pipeline) + `every:10s` (forecast) | **Wall-clock aligned `cron:*/5 * * * * *`** (5 s rollups) | Every minute, `cron:0 * * * * *` (per-truck "today so far") |
| Schedule plugin write path | `LineBuilder` + `influxdb3_local.write()` (local) | Same | **httpx → ingest node's `/api/v3/write_lp`** (cross-node, plugin runs on dedicated process node, must round-trip back through ingest) | `LineBuilder` / `write_sync` (local) | `influxdb3_local.write()` (local) | `LineBuilder` + `influxdb3_local.write()` (local) |
| Request-trigger UI integration | Diagnostic panel calls `pack_health` endpoint | Andon panel direct-fetches `andon_board`; same response drives the chart history | **Three patterns side-by-side, each with its own latency badge:** SQL via FastAPI · SQL from browser via DVC TVF · request plugin from browser | None — SQL via FastAPI only | None — **Grafana only**, provisioned from files | **Map, truck table, trail and events panel all run on one request plugin** (`vehicle_state`, latency badge); a registry request trigger enriches imported history on demand |
| Per-table retention | None | None | **24h on `fabric_health`** — exclusive demo of per-table retention in the portfolio | 24h on raw + intermediate stages | **5 y on the database**, inherited by every table | 7d on raw GPS/diagnostics, 30d on events, alerts and summaries |
| Domain-specific view | Pack/cell heatmap | Andon board grid + per-line OEE breakdown | Aggregate-led: fabric-state banner, layered throughput chart, top-talkers, source-IP typeahead+detail, active-anomalies (drill-on-anomaly only) | Raw-vs-processed overlay chart + false-alarm comparison + live pipeline panel | Fleet overview + 3 identical node dashboards | Live Leaflet map (OpenStreetMap tiles): trucks, trails, geofences, nearest-warehouse distances |
| Aggregate KPI | Pack SoH/SoC daily rollup | A × P × Q live + per-shift `shift_summary` rollup | Live `fabric_health` rollup written by the schedule plugin every 5s | Compression ratio from `record_count` + in-database chronos forecast | 5 s rollups (`record_count = 5`) + threshold alerts that go nowhere | Per-truck daily drive summary (distance, moving/dock/fuel time, yard visits) + LVC-backed fleet status and fuel |

If you're evaluating which patterns to copy:

- **Pick the bess patterns** when your domain is dominated by high entity cardinality and slow-moving rollups (rollup-once-a-day; LVC/DVC for entity inventory).
- **Pick the iiot patterns** when your domain has high event cardinality, real-time alerting on multiple signal types, or a request-trigger that should genuinely replace a backend service (the andon board pattern).
- **Pick the network-telemetry patterns** when you need a multi-node InfluxDB 3 Enterprise cluster: this repo is the trailblazer for the multi-node compose template, the cross-node plugin write-back convention, and the three-pattern UI (SQL via backend / SQL from browser / request plugin from browser).
- **Pick the auto-manufacturing patterns** when your problem is a data-processing pipeline rather than a monitoring surface: registry-based plugin install over the API, chained WAL triggers (stage N's write fires stage N+1), signal-processing plugins (resample / IIR filter / median downsample / zero-shot forecast), and a demo whose data source is itself a Processing Engine plugin — no simulator service.
- **Pick the fleet-telematics patterns** when your data is location-based and you need geospatial context without a geo backend: a registry plugin (`geo_enrichment`) adds nearest-site distances, geofences and reverse-geocoded places as fields on the raw row itself, a chained WAL plugin turns zone changes into enter/exit events and alerts, and one request plugin serves the whole live map.
- **Pick the scientific-infrastructure patterns** when your world is Telegraf agents on many hosts with Grafana on top: per-agent filtering and reshaping so collectd and mock sources converge on one schema, nanosecond timestamps, a 5-year retention database, wall-clock-aligned Processing Engine downsampling into dashboard tables, and Grafana provisioned entirely from files.

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
