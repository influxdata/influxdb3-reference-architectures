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
| 3 | `influxdb3-ref-network-telemetry` | Network Telemetry | 🚧 Coming soon |
| 4 | `influxdb3-ref-renewables` | Renewable Energy (Solar + Wind) | 🚧 Coming soon |
| 5 | `influxdb3-ref-ev-charging` | EV Charging Network | 🚧 Coming soon |
| 6 | `influxdb3-ref-fleet-telematics` | Connected Vehicle / Fleet Telematics | 🚧 Coming soon |
| 7 | `influxdb3-ref-datacenter` | Data Center / Infrastructure Monitoring | 🚧 Coming soon |
| 8 | `influxdb3-ref-oilgas` | Oil & Gas Upstream / SCADA | 🚧 Coming soon |

## What's different across the available repos

The two shipped repos share a template (Python-first, FastAPI/HTMX/uPlot UI, single-node compose, three-tier tests) but make different choices in service of their domain. The variety is intentional — readers shopping for a starting point should pick the repo whose patterns match their problem.

| Dimension | `influxdb3-ref-bess` | `influxdb3-ref-iiot` |
|---|---|---|
| Domain | Battery Energy Storage Systems | Discrete-assembly factory floor |
| Cardinality story | 768 cells (high entity count) | 24 machines + ~700K parts/day (high event-tag count drives DVC) |
| Write rate | ~2,000 pts/s | ~300 pts/s |
| Plugins | 3: 1 WAL · 1 Schedule · 1 Request | 4: **2 WAL** · 1 Schedule · 1 Request |
| WAL plugin patterns shown | Transition-detect (thermal threshold) | Transition-detect (downtime) **and** windowed/derivative (rolling scrap rate) — two different patterns side-by-side |
| Schedule cadence | Daily (`0 5 0 * * *`) | Shift-based, 3×/day (`0 0 6,14,22 * * *`) |
| Request-trigger UI integration | Diagnostic panel calls `pack_health` endpoint | **Andon panel direct-fetches from the browser** to `andon_board` with a "served by Processing Engine: N ms" badge; the same response also drives the per-line OEE chart history |
| Domain-specific view | Pack/cell heatmap | Andon board grid + per-line OEE breakdown |
| OEE / aggregate KPI | Pack SoH/SoC daily rollup | A × P × Q live + per-shift `shift_summary` rollup |

If you're evaluating which patterns to copy:

- **Pick the bess patterns** when your domain is dominated by high entity cardinality and slow-moving rollups (rollup-once-a-day; LVC/DVC for entity inventory).
- **Pick the iiot patterns** when your domain has high event cardinality, real-time alerting on multiple signal types, or a request-trigger that should genuinely replace a backend service (the andon board pattern).

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
