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
| 2 | `influxdb3-ref-iiot` | IIoT / Factory Floor Monitoring | 🚧 Coming soon |
| 3 | `influxdb3-ref-network-telemetry` | Network Telemetry | 🚧 Coming soon |
| 4 | `influxdb3-ref-renewables` | Renewable Energy (Solar + Wind) | 🚧 Coming soon |
| 5 | `influxdb3-ref-ev-charging` | EV Charging Network | 🚧 Coming soon |
| 6 | `influxdb3-ref-fleet-telematics` | Connected Vehicle / Fleet Telematics | 🚧 Coming soon |
| 7 | `influxdb3-ref-datacenter` | Data Center / Infrastructure Monitoring | 🚧 Coming soon |
| 8 | `influxdb3-ref-oilgas` | Oil & Gas Upstream / SCADA | 🚧 Coming soon |

## Shared conventions

Every reference architecture in the portfolio follows the same conventions so that a reader who learns one can navigate the others quickly:

- **Python-first.** Simulator, Processing Engine plugins, UI backend, tests, and CLI glue are all Python.
- **Client library:** [`influxdb3-python`](https://github.com/InfluxCommunity/influxdb3-python).
- **UI stack:** FastAPI + HTMX + Jinja2 + uPlot (vendored, no frontend toolchain).
- **One-command demo:** `docker compose up` brings up the full stack.
- **Processing Engine plugins** (WAL, Schedule, Request triggers) live in `plugins/` and are the centerpiece of every demo.
- **License:** Apache 2.0.

## What's in this repo

This is the **meta-repo** for the portfolio. It holds the portfolio-level design, shared conventions, and links out to the per-vertical repos above.

```
docs/superpowers/
├── specs/   # Portfolio-level design docs
└── plans/   # Per-repo implementation plans
```

## License

[Apache 2.0](LICENSE).
