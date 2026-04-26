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
| 4 | `influxdb3-ref-renewables` | Renewable Energy (Solar + Wind) | 🚧 Coming soon |
| 5 | `influxdb3-ref-ev-charging` | EV Charging Network | 🚧 Coming soon |
| 6 | `influxdb3-ref-fleet-telematics` | Connected Vehicle / Fleet Telematics | 🚧 Coming soon |
| 7 | `influxdb3-ref-datacenter` | Data Center / Infrastructure Monitoring | 🚧 Coming soon |
| 8 | `influxdb3-ref-oilgas` | Oil & Gas Upstream / SCADA | 🚧 Coming soon |

## What's different across the available repos

The three shipped repos share a Python-first template (FastAPI/HTMX/uPlot UI, three-tier tests) but make different choices in service of their domain. The variety is intentional — readers shopping for a starting point should pick the repo whose patterns match their problem.

| Dimension | `influxdb3-ref-bess` | `influxdb3-ref-iiot` | `influxdb3-ref-network-telemetry` |
|---|---|---|---|
| Domain | Battery Energy Storage Systems | Discrete-assembly factory floor | Data-center Clos fabric monitoring |
| Compose shape | Single node | Single node | **5-node cluster** (2 ingest + query + compact + process,query) |
| Cardinality story | 768 cells (high entity count) | 24 machines + ~700K parts/day (high event-tag count drives DVC) | ~1,024 fabric interfaces + 128 BGP sessions + ~5k flow records/sec (DVC drives a typeahead with thousands of distinct src_ips) |
| Write rate | ~2,000 pts/s | ~300 pts/s | **~10,000 pts/s** |
| Plugins | 3: 1 WAL · 1 Schedule · 1 Request | 4: **2 WAL** · 1 Schedule · 1 Request | 4: **0 WAL** · 2 Schedule · 2 Request |
| WAL patterns shown | Transition-detect (thermal threshold) | Transition-detect (downtime) **and** windowed/derivative (scrap rate) | None — multi-node WAL ownership is awkward; this repo leans on schedule plugins instead |
| Schedule cadence + format | Daily, `cron:0 5 0 * * *` | Shift-based, `cron:0 0 6,14,22 * * *` | **Live, `every:5s`** |
| Schedule plugin write path | `LineBuilder` + `influxdb3_local.write()` (local) | Same | **httpx → ingest node's `/api/v3/write_lp`** (cross-node, plugin runs on dedicated process node, must round-trip back through ingest) |
| Request-trigger UI integration | Diagnostic panel calls `pack_health` endpoint | Andon panel direct-fetches `andon_board`; same response drives the chart history | **Three patterns side-by-side, each with its own latency badge:** SQL via FastAPI · SQL from browser via DVC TVF · request plugin from browser |
| Per-table retention | None | None | **24h on `fabric_health`** — exclusive demo of per-table retention in the portfolio |
| Domain-specific view | Pack/cell heatmap | Andon board grid + per-line OEE breakdown | Aggregate-led: fabric-state banner, layered throughput chart, top-talkers, source-IP typeahead+detail, active-anomalies (drill-on-anomaly only) |
| Aggregate KPI | Pack SoH/SoC daily rollup | A × P × Q live + per-shift `shift_summary` rollup | Live `fabric_health` rollup written by the schedule plugin every 5s |

If you're evaluating which patterns to copy:

- **Pick the bess patterns** when your domain is dominated by high entity cardinality and slow-moving rollups (rollup-once-a-day; LVC/DVC for entity inventory).
- **Pick the iiot patterns** when your domain has high event cardinality, real-time alerting on multiple signal types, or a request-trigger that should genuinely replace a backend service (the andon board pattern).
- **Pick the network-telemetry patterns** when you need a multi-node InfluxDB 3 Enterprise cluster: this repo is the trailblazer for the multi-node compose template, the cross-node plugin write-back convention, and the three-pattern UI (SQL via backend / SQL from browser / request plugin from browser).

## Shared conventions

Every reference architecture in the portfolio follows the same conventions so that a reader who learns one can navigate the others quickly:

- **Python-first.** Simulator, Processing Engine plugins, UI backend, tests, and CLI glue are all Python.
- **Client library:** [`influxdb3-python`](https://github.com/InfluxCommunity/influxdb3-python).
- **UI stack:** FastAPI + HTMX + Jinja2 + uPlot (vendored, no frontend toolchain).
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
