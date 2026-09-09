# Implementation plan — `influxdb3-ref-scientific-infrastructure`

**Date:** 2026-09-09
**Category:** Scientific Research Infrastructure
**Name:** Precision Scientific Infrastructure Monitoring
**Repo:** `influxdb3-ref-scientific-infrastructure` (initialised, `main`, no commits yet)
**Stack:** Telegraf 1.40 · InfluxDB 3 Enterprise · Grafana 13.2 · collectd 5.12 (TIG + collectd)
**Headline features:** nanosecond timestamps · 5-year retention · Processing Engine downsampling · multiple Telegraf agents
**Templates:** `influxdb3-ref-auto-manufacturing` (token bootstrap, registry plugin installer, explicit tables, idempotent `init.sh`) and `influxdb3-ref-network-telemetry` (plain-token healthcheck)
**Status:** approved 2026-09-09 with two amendments (temperature is `inputs.mock` on all three agents; registry installer confirmed). One commit per step after its gate passes.

## 0. How this plan works

Ten steps, strictly sequential. Each step has a **Build** list and a numbered **Gate** (`G<step>.<n>`). A gate line is a command plus the exact expected result. Step *n*+1 does not start until every line of gate *n* passes. A failed gate means fix, then re-run the whole gate. Earlier gates stay valid and can be re-run at any later step. One commit per step, after its gate passes.

Gate commands assume the repo root and work from bash or fish:

```
make query sql="<SQL>"        # influxdb3 query --database sci, inside the influxdb3 container
make cli                      # shell in the container; TOKEN exported; iql "<SQL>" helper
docker compose logs --since 60s <service>
curl -s -u admin:admin http://localhost:3000/api/...      # Grafana HTTP API
```

Cadence facts every count-based assertion relies on: raw data is 1 row per second per node per measurement; rollups are 1 row per 5 seconds per node. Unless a gate says otherwise, run count assertions only after the relevant containers have been up for at least 90 seconds.

## 1. Scope

**In:** four network nodes on one compose network — a sink node (InfluxDB 3 Enterprise + Grafana) and three remote nodes (`daq` = Telegraf + collectd, `compute` and `storage` = Telegraf + `inputs.mock` random). Every node ships CPU, load, temperature, memory and uptime once per second with nanosecond timestamps. One database with 5-year retention, explicit tables, Processing Engine downsampling from raw tables to 5-second dashboard tables, Grafana provisioned from files with one fleet overview, three identical node dashboards and threshold alerts that fire but are never delivered.

**Out:** custom UI, tests, CI workflows, Python packaging, backfill, scenarios, LVC/DVC, TLS, multi-node InfluxDB, real hardware temperature sensors.

**Deviations from `CONVENTIONS.md` (deliberate, per the brief):**

- No custom UI: no `ui/`, no FastAPI/HTMX/uPlot, no `pyproject.toml`, no Python at all except the copied stdlib-only plugin installer.
- No test tiers, no `tests/`, no GitHub workflows. The manual gates below replace them.
- Makefile keeps `help up down clean logs ps demo demo-fresh cli query cli-example`, adds `dashboards` (regenerate node dashboards) and `open` (Grafana in the browser), drops `test* lint format scenario*`.
- The data source is Telegraf (mock + collectd), not a Python simulator and not a plugin.
- Everything else is kept: token bootstrap, plain-token healthcheck, `INFLUXDB3_UNSET_VARS`, explicit table creation, registry plugin install through the files API (installer copied verbatim from auto-manufacturing), delete-and-recreate triggers, the README / ARCHITECTURE / CLI_EXAMPLES / FOR_MAINTAINERS / diagram / demo-script structure.

## 2. Design contract

Everything below is what the gates assert against. Change it here first if a step forces a change.

### 2.1 Topology

| Node | Containers (`container_name`) | Image | Role |
|---|---|---|---|
| sink | `sci-influxdb3` | `influxdb:3-enterprise` | single node, `--mode all`, `--plugin-dir /var/lib/influxdb3/plugins` (inside the data volume, no bind mount), port 8181 |
| sink | `sci-grafana` | `grafana/grafana:13.2.1` | provisioned datasource + dashboards + alert rules, port 3000 |
| sink (one-shots) | `sci-token-bootstrap`, `sci-plugin-installer`, `sci-influxdb3-init` | `influxdb:3-enterprise`, `python:3.12-slim`, `influxdb:3-enterprise` | admin token, registry plugin upload, database/tables/triggers |
| daq | `sci-telegraf-daq` | `telegraf:1.40` | `inputs.socket_listener` (collectd binary protocol, UDP 25826) + `inputs.system` + `inputs.mock` (temperature, same block as the other nodes) |
| daq | `sci-collectd` | `debian:bookworm-slim` + `collectd-core` (own `collectd/Dockerfile`) | cpu, load, memory, uptime, interface, df → network plugin → `telegraf-daq:25826` |
| compute | `sci-telegraf-compute` | `telegraf:1.40` | `inputs.mock` ×4 + `inputs.system` + `inputs.processes` |
| storage | `sci-telegraf-storage` | `telegraf:1.40` | same shape as compute, different hostname and value ranges |

Names: database `sci`; token files `/var/lib/influxdb3/.sci-operator-token` (JSON, mode 600) and `/var/lib/influxdb3/.sci-token-plain` (mode 644 so non-root readers — the healthcheck, Telegraf, Grafana — can read it). Telegraf and Grafana mount the `influxdb-data` volume read-only at `/tokens`. That is a demo shortcut and ARCHITECTURE.md says so: real remote nodes get a scoped write token through config management.

Boot order: `token-bootstrap` → `influxdb3` (healthy) → `plugin-installer` → `influxdb3-init` → `telegraf-daq` (+ `collectd`), `telegraf-compute`, `telegraf-storage`, `grafana`. Telegraf waits for init so explicit table creation never races implicit creation-on-first-write.

### 2.2 Nanosecond timestamps

`[agent] precision = "1ns"` in every Telegraf config. The default (`0s`) rounds collected timestamps to 1 s whenever the interval is ≥ 1 s (`agent.go` `getPrecision`), which is exactly what a "precision" demo must not do. Regular inputs (`mock`, `system`, `processes`) get 1 ns; collectd is a service input whose own high-resolution timestamps (2⁻³⁰ s) pass through untouched. `outputs.influxdb_v2` serialises nanosecond line protocol to `/api/v2/write` (InfluxDB 3's v2-compatible endpoint, default precision ns).

### 2.3 Schema (database `sci`, retention 5 years, all tables created explicitly)

Raw tables, written by Telegraf, 1 row per second per host:

| Table | Tags | Fields |
|---|---|---|
| `cpu` | `host` | `usage_active` f64 (%) |
| `load` | `host` | `load1`, `load5`, `load15` f64 |
| `mem` | `host` | `used_percent` f64 (%) |
| `temperature` | `host` | `temp_c` f64 |
| `uptime` | `host` | `uptime` i64 (s) |

Dashboard tables, written by the downsampler, 1 row per 5 s per host. The registry plugin names output fields `<field>_<calc>` (CONVENTIONS gotcha), and adds `record_count`, `time_from`, `time_to` on its first write:

| Table | Fields declared by `init.sh` |
|---|---|
| `cpu_5s` | `usage_active_avg` f64 |
| `load_5s` | `load1_avg`, `load5_avg`, `load15_avg` f64 |
| `mem_5s` | `used_percent_avg` f64 |
| `temperature_5s` | `temp_c_avg` f64 |
| `uptime_5s` | `uptime_max` i64 |

Retention is set once, on the database (`create database sci --retention-period 5y`); the ten tables inherit it. No backfill.

### 2.4 Telegraf: filtering and reshaping, per agent

Common to all three agents:

```toml
[agent]
  interval = "1s"          # collect every second
  flush_interval = "1s"    # write every second
  precision = "1ns"        # keep nanosecond timestamps (default would round to 1s)
  round_interval = true
  hostname = "<node>"      # daq | compute | storage

[[outputs.influxdb_v2]]
  urls = ["http://influxdb3:8181"]
  bucket = "sci"
  token = "${INFLUX_TOKEN}"                                  # entrypoint reads /tokens/.sci-token-plain
  namepass = ["cpu", "load", "mem", "temperature", "uptime"] # the write contract: only canonical measurements leave the agent
  taginclude = ["host"]                                      # tag-key allowlist: every other tag is stripped
```

Filter choices (one of each pair, as the brief allows): `namepass` (not `namedrop`), `fieldinclude` (not `fieldexclude`), `taginclude` as the tag-key allowlist. The DAQ agent additionally uses `tagpass` — a tag-*value* allowlist — because collectd multiplexes one measurement across `type_instance` values (`memory/percent-used`, `-free`, `-cached`, …); without selecting `used` first, the six memory rows would collapse onto one series once the tags are stripped.

**compute / storage (mock):**

| Input emits | Filter / reshape | Lands as |
|---|---|---|
| `[[inputs.mock]] cpu{cpu=cpu-total}` random `usage_active`, `usage_user`, `usage_system`, `usage_iowait`, `usage_steal` | `fieldinclude = ["usage_active"]`; output `taginclude` strips `cpu` | `cpu.usage_active` |
| `[[inputs.mock]] load` random `load1`, `load5`, `load15` | — | `load.*` |
| `[[inputs.mock]] mem` random `used_percent`, `available_percent`, `cached_percent` | `fieldinclude = ["used_percent"]` | `mem.used_percent` |
| `[[inputs.mock]] temp{sensor=coretemp}` random `temp` (mimics `inputs.temp`) | `[[processors.rename]]` measurement `temp`→`temperature`, field `temp`→`temp_c`; `taginclude` strips `sensor` | `temperature.temp_c` |
| `[[inputs.system]]` (legacy layout: `load1/5/15`, `n_cpus`, `n_users`, `uptime`, `uptime_format`) | `fieldinclude = ["uptime"]`; `[[processors.rename]]` measurement `system`→`uptime` | `uptime.uptime` |
| `[[inputs.processes]]` (`processes`: running, sleeping, zombies, …) — collected on purpose | dropped by output `namepass` | nothing |

Mock ranges (keep dashboards distinguishable and alerts quiet): compute cpu 40–75, load 1–4, mem 50–70, temp 45–60; storage cpu 5–25, load 0.2–1.5, mem 70–85, temp 30–40.

**daq (collectd + system + mock temperature):** `inputs.socket_listener` with `data_format = "collectd"`, `collectd_parse_multivalue = "join"` (one row per value list, so `load` arrives as three fields), `collectd_typesdb = ["/etc/telegraf/types.db"]`, `name_prefix = "collectd_"`, and

```toml
  [inputs.socket_listener.tagpass]      # OR across keys: keep cpu/active, memory/used, load
    type_instance = ["active", "used"]
    type = ["load"]
```

| collectd sends (measurement after prefix, distinguishing tags) | Fate |
|---|---|
| `collectd_cpu{type=percent,type_instance=active}` field `value` | keep; scoped rename → `cpu.usage_active` |
| `collectd_load{type=load}` fields `shortterm`, `midterm`, `longterm` | keep; scoped rename → `load.load1/load5/load15` |
| `collectd_memory{type=percent,type_instance=used}` field `value` | keep; scoped rename → `mem.used_percent` |
| `collectd_memory{type_instance=free,cached,buffered,slab_*}` | dropped by `tagpass` |
| `collectd_uptime{type=uptime}` (no `type_instance`) | dropped by `tagpass` — uptime is standardised on `inputs.system` on every node |
| `collectd_interface{instance=eth0,type=if_octets|if_packets|if_errors}` (no `type_instance`) | dropped by `tagpass` |
| `collectd_df{instance=root,type=percent_bytes,type_instance=used}` | passes `tagpass`, dropped by output `namepass` (why both filters exist) |
| `system` | `fieldinclude = ["uptime"]`; rename → `uptime.uptime` |
| `[[inputs.mock]] temp{sensor=coretemp}` random `temp` — identical block to the mock nodes, daq range 20–30 | rename → `temperature.temp_c`; `taginclude` strips `sensor` |
| leftover tags `type`, `type_instance`, `instance` | stripped by output `taginclude = ["host"]` |

Renames are one `[[processors.rename]]` block per source measurement with `namepass = ["collectd_cpu"]` etc., because a field rename (`value`→`usage_active`) in an unscoped block would hit every metric that has a `value` field. *(Recorded 2026-09-09: in `join` mode the parser names the measurement after the collectd plugin only — `collectd_cpu`, not `collectd_cpu_percent`; the type is the `type` tag. `split` mode is where `<plugin>_<dsname>` names such as `cpu_value` come from.)*

`telegraf/types.db` is a six-line file of our own (`percent`, `load`, `temperature`, `uptime`, `memory`, `cpu`): Telegraf ships no built-in types.db, and without one multi-value types such as `load` arrive as fields `0`, `1`, `2`.

### 2.5 collectd (DAQ)

`collectd/collectd.conf`: `Hostname "daq"`, `FQDNLookup false`, `Interval 1` (matches Telegraf's 1 s), `logfile` → stderr, plugins `cpu` (`ReportByCpu false`, `ReportByState false`, `ValuesPercentage true` → one `percent-active` value), `load`, `memory` (`ValuesAbsolute false`, `ValuesPercentage true`), `uptime`, `interface`, `df` (the last three are sent so Telegraf can visibly drop them), `network` → `Server "telegraf-daq" "25826"`.

Temperature is not a collectd concern on any node: all three agents use the same `inputs.mock` temperature block (Docker Desktop's Linux VM exposes no `/sys/class/thermal` zones, so neither collectd `thermal` nor Telegraf `inputs.temp` would produce data on the target machine). ARCHITECTURE.md documents the swap to Telegraf `inputs.temp` on a Linux host that has sensors.

### 2.6 Processing Engine downsampling

`installer/` (Dockerfile + `install_plugins.py`, copied verbatim from auto-manufacturing) and `installer/plugins.lock` pinning `downsampler` `1.4.0` (newest in the registry index today; sha256 in the index). Five schedule triggers registered by `init.sh` after the tables exist, `downsample_<table>`:

```
--trigger-spec "cron:*/5 * * * * *"           # 6-field cron, fires on wall-clock :00 :05 :10 …
--path "downsampler-1.4.0/downsampler.py"
--trigger-arguments "source_measurement=cpu,target_measurement=cpu_5s,target_database=sci,interval=5s,window=5s,offset=5s,calculations=avg"
```

`uptime` uses `calculations=max`. Why cron + `offset=5s` + `window=5s` rather than `every:5s`: the plugin queries `[call_time − offset − window, call_time − offset)` and formats both bounds as whole seconds, so a wall-clock-aligned tick at `:10` queries exactly `[:00, :05)` — one complete bucket, written once. `every:5s` is not wall-clock aligned, would straddle two buckets every tick, and would rewrite each bucket twice with partial data (and rewrites are not reliably last-write-wins, per CONVENTIONS). The 5 s offset also gives Telegraf's 1 s flush and the WAL a comfortable margin.

### 2.7 Grafana

All from files under `grafana/provisioning/`; nothing is created by hand in the UI.

- **Datasource** uid `influxdb3`: `type: influxdb`, `url: http://influxdb3:8181`, `jsonData: {version: SQL, dbName: sci, httpMode: POST, insecureGrpc: true}`, `secureJsonData.token: $__file{/tokens/.sci-token-plain}`.
- **Env:** `GF_AUTH_ANONYMOUS_ENABLED=true` (Viewer) so dashboards open with no login; `GF_SECURITY_ADMIN_PASSWORD` from `.env` (default `admin`) for editing and the API; `GF_USERS_ALLOW_SIGN_UP=false`.
- **Dashboards** in `grafana/dashboards/`, all queries against the `_5s` tables only, 5 s refresh:
  - `fleet-overview.json` (uid `sci-fleet`): per node a status stat (UP if the newest `uptime_5s` row is < 30 s old, else DOWN; no data → DOWN), an uptime stat (seconds, shown as duration), and an alert-list panel. Node titles link to the node dashboards. *(Corrected 2026-09-09 from 15 s: a healthy node's newest rollup row is 10–16 s old by construction — bucket start + 5 s offset + up to 5 s to the next tick.)*
  - `node-daq.json`, `node-compute.json`, `node-storage.json` (uids `sci-node-<host>`): generated from `node.json.tmpl` by `scripts/gen-node-dashboards.sh` (sed on `__HOST__`), committed. Three rows: **Status** (UP/DOWN, uptime, seconds since last sample) · **Current values** (gauges: cpu %, load1, mem %, temp °C) · **History** (time series: cpu, load1/5/15, mem, temp over `$__timeFilter`). Note: inside Docker all three agents report the same uptime — `inputs.system` reads the kernel uptime, and every container shares Docker Desktop's VM kernel. On real hosts it is per node.
- **Alert rules** (`alerting/rules.yaml`, one group, 10 s evaluation, multi-dimensional on `host` — the SQL returns `host` + one number per row and Grafana's expression engine turns that into one instance per host): `sci-cpu-high` (`usage_active_avg` > 90 for 30 s), `sci-temp-high` (`temp_c_avg` > 80 for 30 s), `sci-node-down` (age of the newest `uptime_5s` row > 45 s for 30 s, over a 24 h look-back so a stopped node stays in the result set and fires instead of vanishing). **Delivery goes nowhere, twice over:** Grafana 13 ships no contact points and a built-in `empty` root receiver, and a catch-all child route (`alertname =~ .+`) carries an `always` mute timing (00:00–24:00). Alerts show as Firing in the UI and the alert-list panel; nothing is sent, nothing errors. *(Recorded 2026-09-09: a mute timing on the root route is rejected — "root route must not have any mute time intervals"; weekday ranges must start on Sunday.)*

### 2.8 File tree

```
influxdb3-ref-scientific-infrastructure/
├── README.md  ARCHITECTURE.md  CLI_EXAMPLES.md  FOR_MAINTAINERS.md  LICENSE  Makefile
├── .env.example  .gitignore  docker-compose.yml
├── diagrams/architecture.mmd  diagrams/architecture.png
├── influxdb/init.sh  influxdb/schema.md
├── installer/Dockerfile  installer/install_plugins.py  installer/plugins.lock
├── telegraf/daq.conf  telegraf/compute.conf  telegraf/storage.conf  telegraf/types.db
├── collectd/Dockerfile  collectd/collectd.conf
├── grafana/provisioning/datasources/influxdb3.yaml
├── grafana/provisioning/dashboards/dashboards.yaml
├── grafana/provisioning/alerting/rules.yaml
├── grafana/dashboards/fleet-overview.json  node.json.tmpl  node-daq.json  node-compute.json  node-storage.json
└── scripts/setup.sh  scripts/demo.sh  scripts/gen-node-dashboards.sh
```

## 3. Decisions made on your behalf (override at approval)

1. **Temperature is `inputs.mock` on all three agents, DAQ included** (amended at approval; no thermal zones inside Docker Desktop). collectd carries no temperature plugin.
2. **Canonical names** `cpu.usage_active`, `load.load1/5/15`, `mem.used_percent`, `temperature.temp_c`, `uptime.uptime`; dashboard tables `<table>_5s` with `<field>_<calc>` fields (forced by the plugin, aliased back in Grafana SQL).
3. **Where the filters live:** `namepass` + `taginclude` on the output of every agent (the write contract); `fieldinclude` on the inputs that have noisy fields; `tagpass` on the collectd input only. Two tag filters on DAQ, for the reason in §2.4.
4. **Three Telegraf config files**, one per node, rather than one env-templated file. Readable and self-contained; the price is ~60 duplicated lines between `compute.conf` and `storage.conf`.
5. **Downsampler from the registry through the copied installer** (portfolio convention: pinned, sha256-verified, no plugin bind mount). Alternative if you would rather not carry `installer/`: `--path "gh:influxdata/downsampler/downsampler.py"` in `init.sh`, one line, unpinned, needs network at boot.
6. **Five triggers** (one per raw table) on a `*/5` cron with `offset=5s,window=5s`; `avg` for everything, `max` for uptime.
7. **Database-level 5 y retention** (`5y`; fall back to `1825d` if the CLI rejects the unit). The brief says all data is 5 y, so raw and rollups are treated alike; the raw-shorter/rollup-longer variant is mentioned in ARCHITECTURE.md as the production choice.
8. **Three generated node dashboards** from one template (brief: one per node, identical). Alternative: one dashboard with a `$host` variable.
9. **Three alert rules**, multi-dimensional on `host`; delivery muted by an always-on mute timing rather than a dead contact point.
10. **Image pins:** `telegraf:1.40`, `grafana/grafana:13.2.1`, `debian:bookworm-slim` (collectd 5.12.0), `influxdb:3-enterprise` floating (portfolio convention; 3.11.4 today).
11. **No tests, no CI, no Python packaging.** Gates only.
12. **Plan lives here (meta-repo), per convention.** The per-repo design spec (`docs/superpowers/specs/`) is not written; §2 is the spec-equivalent and can be split out into the new repo on request.

## 4. Steps and gates

### Step 1 — Repo scaffold; InfluxDB boots

**Build**
- `LICENSE` (Apache 2.0), `.gitignore` (`.env`, `.DS_Store`), `.env.example` (`INFLUXDB3_ENTERPRISE_EMAIL`, `INFLUXDB3_ENTERPRISE_LICENSE_TYPE`, `GRAFANA_PORT=3000`, `GRAFANA_ADMIN_PASSWORD=admin`, `REGISTRY_INDEX_URL`, `INSTALLER_OFFLINE_DIR`).
- `scripts/setup.sh` (copied from auto-manufacturing; prompts for the licence email).
- `Makefile` with the surface in §1 (`dashboards`/`open` may be stubs until steps 7–8).
- `docker-compose.yml` with only `token-bootstrap` (creates `plugins/`, writes the JSON token at 600 **and** the plain token at 644) and `influxdb3` (`--mode all`, `--plugin-dir /var/lib/influxdb3/plugins`, `--virtual-env-location /var/lib/influxdb3/plugin-venv`, `--admin-token-file`, `INFLUXDB3_UNSET_VARS: LOG_FILTER`, plain-token healthcheck).

**Gate G1**
- G1.1 `docker compose config -q` → exit 0.
- G1.2 `make up` → prompts once for the email, `.env` contains it, stack starts. Click the licence validation link.
- G1.3 `docker inspect --format '{{.State.Health.Status}}' sci-influxdb3` → `healthy` within 10 minutes of clicking the link.
- G1.4 `docker inspect --format '{{.State.ExitCode}}' sci-token-bootstrap` → `0`; `docker compose exec influxdb3 ls -l /var/lib/influxdb3/` shows `.sci-operator-token` mode `-rw-------`, `.sci-token-plain` mode `-rw-r--r--`, and a `plugins` directory.
- G1.5 `make cli`, then `influxdb3 show databases --token $TOKEN` → exit 0 (empty list is fine).

**Commit:** `scaffold: compose, token bootstrap, influxdb3 single node`

### Step 2 — Database, 5-year retention, explicit tables

**Build**
- `influxdb/init.sh` (idempotent; `wait_for_api`, `ensure_token`, `ensure_database` with `--retention-period 5y`, `ensure_tables` creating the ten tables of §2.3 with `--tags host --fields ...`; trigger section left as a stub until step 6).
- `influxdb/schema.md` (tables, fields, types, retention, who writes what).
- Compose service `influxdb3-init` (one-shot, depends on `influxdb3` healthy).

**Gate G2**
- G2.1 `docker inspect --format '{{.State.ExitCode}}' sci-influxdb3-init` → `0`; `docker compose logs influxdb3-init` contains no `FATAL`.
- G2.2 `make cli` → `influxdb3 show databases --token $TOKEN` lists `sci`.
- G2.3 `influxdb3 show retention --token $TOKEN` → one row per `sci` table with `retention_period` = `43830.0000h` (5 × 365.25 d) and `source` = `database`. *(Recorded 2026-09-09: `5y` accepted by `create database`; the CLI prints hours.)*
- G2.4 `make query sql="SELECT table_name FROM information_schema.tables WHERE table_schema = 'iox' ORDER BY table_name"` → exactly 10 rows: `cpu, cpu_5s, load, load_5s, mem, mem_5s, temperature, temperature_5s, uptime, uptime_5s`.
- G2.5 `make query sql="SELECT table_name, column_name, data_type FROM information_schema.columns WHERE table_schema = 'iox' AND table_name IN ('cpu','uptime','uptime_5s') ORDER BY 1, 2"` → `cpu`: `host Dictionary(Int32, Utf8)`, `time Timestamp(ns)`, `usage_active Float64`; `uptime`: `host`, `time`, `uptime Int64`; `uptime_5s`: `host`, `time`, `uptime_max Int64`. Nothing else.
- G2.6 Retention behaviour probe (bash; `TOKEN` from `make cli` or `docker compose exec influxdb3 cat /var/lib/influxdb3/.sci-token-plain`):
  ```bash
  NOW=$(date +%s)
  T4=$(( NOW - 4*365*86400 ))000000000      # 4 years ago, inside retention
  T6=$(( NOW - 6*365*86400 ))000000000      # 6 years ago, outside retention
  curl -s -o /dev/null -w '%{http_code}\n' -X POST "http://localhost:8181/api/v3/write_lp?db=sci&precision=nanosecond" -H "Authorization: Bearer $TOKEN" --data-binary "retention_probe,host=probe v=1i $T4"
  curl -s -o /dev/null -w '%{http_code}\n' -X POST "http://localhost:8181/api/v3/write_lp?db=sci&precision=nanosecond" -H "Authorization: Bearer $TOKEN" --data-binary "retention_probe,host=probe v=2i $T6"
  ```
  → first write `204`; second write `400` with `write timestamp … is older than the retention period cutoff …` *(recorded 2026-09-09: the server hard-rejects out-of-retention writes)*. Then `make query sql="SELECT v FROM retention_probe"` → exactly one row, `v = 1`. Clean up with `influxdb3 delete table retention_probe --database sci --hard-delete now --yes --token $TOKEN` — a plain delete only soft-deletes (the table lingers as `retention_probe-<timestamp>` and cannot be hard-deleted afterwards, 409); G2.4 passes again afterwards.
- G2.7 Second run is a no-op: `docker compose run --rm influxdb3-init` → exit 0, log lines say `already exists` for the database and all ten tables, no `FATAL`.

**Commit:** `influxdb: database sci with 5y retention, explicit raw and 5s tables`

### Step 3 — `compute` node (mock): the reference Telegraf agent

**Build**
- `telegraf/compute.conf` exactly as §2.4 (mock ranges for compute).
- Compose service `telegraf-compute`: `telegraf:1.40`, depends on `influxdb3-init` completed, mounts `./telegraf/compute.conf:/etc/telegraf/telegraf.conf:ro` and `influxdb-data:/tokens:ro`, `command: ["sh", "-c", "export INFLUX_TOKEN=$(cat /tokens/.sci-token-plain) && exec telegraf --config /etc/telegraf/telegraf.conf"]`.

**Gate G3** (after ≥ 90 s)
- G3.1 `docker compose ps telegraf-compute` → `Up`; `docker compose logs --since 60s telegraf-compute | grep -c 'E!'` → `0`.
- G3.2 Rate, for each `t` in `cpu load mem temperature uptime`: `make query sql="SELECT count(*) AS n FROM t WHERE host = 'compute' AND time > now() - INTERVAL '30 seconds'"` → `28 ≤ n ≤ 32`.
- G3.3 Nothing leaked: G2.4 still returns exactly the same 10 tables (no `processes`, `system`, `temp`). `make query sql="SELECT table_name, column_name FROM information_schema.columns WHERE table_schema = 'iox' AND table_name IN ('cpu','load','mem','temperature','uptime') ORDER BY 1, 2"` → exactly 17 rows: `host` and `time` in each of the five tables plus the seven fields of §2.3; in particular no `usage_user`, `n_cpus`, `available_percent`, `cpu`, `sensor`.
- G3.4 Nanosecond timestamps: `make query sql="SELECT count(*) AS whole_second_rows FROM cpu WHERE host = 'compute' AND CAST(time AS BIGINT) % 1000000000 = 0"` → `0` (with ≥ 60 rows present). `make query sql="SELECT time FROM cpu WHERE host = 'compute' ORDER BY time DESC LIMIT 3"` → timestamps print nine fractional digits with non-zero digits below the millisecond, e.g. `…T14:03:21.004128733`. (If `CAST` is refused, use `date_part('nanosecond', time) % 1000000000 = 0`.)
- G3.5 Uptime is real and 1 Hz: `make query sql="SELECT max(uptime) - min(uptime) AS span_s, count(*) AS n FROM uptime WHERE host = 'compute' AND time > now() - INTERVAL '30 seconds'"` → `span_s` between 28 and 30, `n` between 28 and 32.
- G3.6 Values are in the configured mock ranges: `make query sql="SELECT min(usage_active), max(usage_active) FROM cpu WHERE host = 'compute'"` → within 40–75; same check for `temp_c` (45–60) and `used_percent` (50–70).
- G3.7 Types held: G2.5 passes unchanged (no schema-conflict rejections; Telegraf logs contain no `400`/`field type conflict`).

**Commit:** `telegraf: compute node (mock inputs, fieldinclude/namepass/taginclude, rename)`

### Step 4 — `storage` node (mock)

**Build**
- `telegraf/storage.conf` = `compute.conf` with `hostname = "storage"` and the storage ranges; compose service `telegraf-storage` identical in shape.

**Gate G4** (after ≥ 90 s)
- G4.1 `docker compose logs --since 60s telegraf-storage | grep -c 'E!'` → `0`.
- G4.2 `make query sql="SELECT host, count(*) AS n FROM cpu WHERE time > now() - INTERVAL '30 seconds' GROUP BY host ORDER BY host"` → exactly 2 rows, `compute` and `storage`, each `28 ≤ n ≤ 32`. Repeat for `uptime`.
- G4.3 G3.3, G3.4 and G3.5 pass with `host = 'storage'`.
- G4.4 `make query sql="SELECT min(usage_active), max(usage_active) FROM cpu WHERE host = 'storage'"` → within 5–25 (storage range, proving the two agents are independently configured).

**Commit:** `telegraf: storage node`

### Step 5 — `daq` node: collectd → Telegraf

**Build**
- `collectd/Dockerfile` (`debian:bookworm-slim`, `apt-get install collectd-core`, copy `collectd.conf`) and `collectd/collectd.conf` per §2.5.
- `telegraf/daq.conf` per §2.4 and `telegraf/types.db`.
- Compose services `telegraf-daq` (same shape as compute, plus `./telegraf/types.db:/etc/telegraf/types.db:ro`) and `collectd` (build, depends on `telegraf-daq` started).

**Gate G5** (after ≥ 90 s)
- G5.1 `docker compose build collectd` → exit 0; `docker compose ps collectd` → `Up`; `docker compose logs collectd | grep -iE 'error|fail|not found'` → empty.
- G5.2 `docker compose logs --since 60s telegraf-daq | grep -c 'E!'` → `0`.
- G5.3 All three nodes, all five tables: `make query sql="SELECT host, count(*) AS n FROM cpu WHERE time > now() - INTERVAL '30 seconds' GROUP BY host ORDER BY host"` → exactly 3 rows (`compute`, `daq`, `storage`), each `28 ≤ n ≤ 32`. Repeat for `load`, `mem`, `temperature`, `uptime`.
- G5.4 `tagpass` kept only `used`: the `mem` count for `daq` in G5.3 is ~30, not ~180; `make query sql="SELECT used_percent FROM mem WHERE host = 'daq' ORDER BY time DESC LIMIT 1"` → `0 < used_percent < 100`.
- G5.5 Multi-value join + field renames worked: `make query sql="SELECT load1, load5, load15 FROM load WHERE host = 'daq' ORDER BY time DESC LIMIT 1"` → all three non-null.
- G5.6 Nothing leaked: G2.4 still 10 tables (no `collectd_*`, no `processes`); G3.3 still 17 columns (no `type`, `type_instance`, `instance`).
- G5.7 Timestamps: G3.4 with `host = 'daq'` → `0` whole-second rows (collectd's high-resolution time survives).
- G5.8 Temperature mock on daq: `make query sql="SELECT min(temp_c), max(temp_c) FROM temperature WHERE host = 'daq'"` → within 20–30.
- G5.9 Source split is real: `docker compose stop collectd`; wait 15 s; `make query sql="SELECT count(*) FROM cpu WHERE host = 'daq' AND time > now() - INTERVAL '10 seconds'"` → `0` (same for `load`, `mem`) while the same query on `uptime` and `temperature` → `8–12`. `docker compose start collectd` → `cpu` rows for `daq` resume within 10 s.

**Commit:** `daq node: collectd container, socket_listener with tagpass, scoped renames, mock temperature`

### Step 6 — Processing Engine downsampling

**Build**
- `installer/Dockerfile`, `installer/install_plugins.py` (copied verbatim), `installer/plugins.lock` (downsampler 1.4.0 only; point `LOCAL_PLUGINS_DIR` at an empty or absent dir, whichever the script tolerates).
- Compose service `plugin-installer` (one-shot, after `influxdb3` healthy); `influxdb3-init` now depends on it.
- `init.sh`: `ensure_triggers` — delete-and-recreate the five `downsample_<table>` triggers per §2.6, then `enable`.

**Gate G6** (after ≥ 2 minutes)
- G6.1 `docker inspect --format '{{.State.ExitCode}}' sci-plugin-installer` → `0`; its logs show `downsampler-1.4.0` uploaded with the sha256 verified.
- G6.2 `make query sql="SELECT trigger_name, plugin_filename, trigger_specification, disabled FROM system.processing_engine_triggers ORDER BY trigger_name"` → 5 rows `downsample_cpu … downsample_uptime`, path `downsampler-1.4.0/downsampler.py`, spec `{"schedule":{"schedule":"*/5 * * * * *"}}` (the catalog stores the cron as JSON), `disabled = false`.
- G6.3 Rate: `make query sql="SELECT host, count(*) AS n FROM cpu_5s WHERE time > now() - INTERVAL '60 seconds' GROUP BY host ORDER BY host"` → 3 rows, each `n = 10` (accept 9–11). A 60 s window holds 12 bucket starts, but the two newest buckets are always still pending: a bucket starting at T is written at T + 10 s (5 s offset, then up to 5 s to the next tick). Repeat for `load_5s`, `mem_5s`, `temperature_5s`, `uptime_5s`. *(Corrected 2026-09-09: the original expectation of 12 ignored the write lag.)*
- G6.4 Buckets are on the 5 s grid: `make query sql="SELECT count(*) AS off_grid FROM cpu_5s WHERE CAST(time AS BIGINT) % 5000000000 <> 0"` → `0`.
- G6.5 Every bucket is complete and written once: `make query sql="SELECT min(record_count), max(record_count) FROM cpu_5s WHERE time > now() - INTERVAL '60 seconds' AND time < now() - INTERVAL '15 seconds'"` → `5` and `5`. A `4` or a `1` means the window/offset alignment in §2.6 is wrong for this server version; fix before continuing (fallbacks, in order: check what `call_time` the engine passes; adjust `offset`; last resort a 40-line local `schedule_downsample.py` that aligns explicitly).
- G6.6 Values are right (avg over the same 5 s of raw data, per host):
  ```
  make query sql="SELECT c.time, c.host, c.usage_active_avg, r.avg_raw FROM cpu_5s c JOIN (SELECT date_bin(INTERVAL '5 seconds', time) AS b, host, avg(usage_active) AS avg_raw FROM cpu WHERE time > now() - INTERVAL '2 minutes' GROUP BY b, host) r ON r.b = c.time AND r.host = c.host WHERE c.time > now() - INTERVAL '2 minutes' AND c.time < now() - INTERVAL '15 seconds' ORDER BY c.time DESC LIMIT 9"
  ```
  → 9 rows, `|usage_active_avg − avg_raw| < 1e-6` on every row.
- G6.7 Uptime rollup: `make query sql="SELECT host, max(uptime_max) - min(uptime_max) AS span FROM uptime_5s WHERE time > now() - INTERVAL '60 seconds' GROUP BY host"` → each host `span = 45` (10 buckets, 9 intervals of 5 s; accept 40–50). *(Corrected 2026-09-09, same write-lag reason as G6.3.)*
- G6.8 Plugin health: `make query sql="SELECT count(*) FROM system.processing_engine_logs WHERE log_level = 'ERROR' AND event_time > now() - INTERVAL '5 minutes'"` → `0`; `make query sql="SELECT trigger_name, count(*) FROM system.processing_engine_logs WHERE event_time > now() - INTERVAL '5 minutes' GROUP BY trigger_name"` → 5 triggers, all logging. (The log table is persisted lazily and can trail real time by a minute or more, hence the 5-minute window.)
- G6.9 Dashboard-table columns: `make query sql="SELECT column_name FROM information_schema.columns WHERE table_schema = 'iox' AND table_name = 'cpu_5s' ORDER BY 1"` → exactly `host, record_count, time, time_from, time_to, usage_active_avg` (`time_from`/`time_to` are Int64 nanoseconds, not timestamps).
- G6.10 `docker compose run --rm influxdb3-init` → exit 0, triggers deleted and recreated, G6.3 still passes two minutes later.

**Commit:** `processing engine: registry downsampler, five aligned 5s rollup triggers`

### Step 7 — Grafana: datasource and fleet overview

**Build**
- `grafana/provisioning/datasources/influxdb3.yaml`, `grafana/provisioning/dashboards/dashboards.yaml`, `grafana/dashboards/fleet-overview.json` per §2.7.
- Compose service `grafana` (depends on `influxdb3-init` completed; mounts provisioning and dashboards read-only, `influxdb-data:/tokens:ro`; env per §2.7; port `${GRAFANA_PORT:-3000}:3000`). Makefile `open` target.

**Gate G7**
- G7.1 `curl -s http://localhost:3000/api/health` → `"database": "ok"`.
- G7.2 `curl -s -u admin:admin http://localhost:3000/api/datasources/uid/influxdb3/health` → `"status": "OK"` (the `$__file{}` token worked and Flight SQL over plain gRPC is reachable).
- G7.3 `curl -s 'http://localhost:3000/api/search?type=dash-db'` (anonymous) → includes `"uid": "sci-fleet"`, title `Fleet overview`.
- G7.4 A dashboard query returns all three nodes through Grafana:
  ```
  curl -s -u admin:admin -H 'Content-Type: application/json' http://localhost:3000/api/ds/query -d '{"from":"now-5m","to":"now","queries":[{"refId":"A","datasource":{"uid":"influxdb3"},"rawSql":"SELECT host, max(uptime_max) AS uptime FROM uptime_5s WHERE time > now() - INTERVAL '\''1 minute'\'' GROUP BY host ORDER BY host","format":"table"}]}'
  ```
  → `results.A.frames[0]` has 3 rows: `compute`, `daq`, `storage`, all `uptime > 0`. (Adjust the query-model keys to what the datasource expects if the first attempt errors; record the working body.)
- G7.5 Visual, `http://localhost:3000/d/sci-fleet`: three status stats green `UP`; three uptime stats showing a duration > 0 that advances on refresh; the alert-list panel renders (empty); each node title links to `/d/sci-node-<host>` (404 until step 8 — fine). Headless equivalent: `GET /api/dashboards/uid/sci-fleet` shows 7 panels with the three links, and the panels' SQL run through `POST /api/ds/query` gives `age_s < 30` and `uptime > 0` for every host.
- G7.6 Status reacts: `docker compose stop telegraf-storage` → within 45 s the storage stat shows red `DOWN`, the other two stay `UP`; `docker compose start telegraf-storage` → `UP` within 45 s.

**Commit:** `grafana: provisioned influxdb3 SQL datasource and fleet overview`

### Step 8 — Grafana: per-node dashboards

**Build**
- `grafana/dashboards/node.json.tmpl` per §2.7, `scripts/gen-node-dashboards.sh` (`for h in daq compute storage; sed "s/__HOST__/$h/g" …`), the three generated JSON files committed, Makefile `dashboards` target.

**Gate G8**
- G8.1 `make dashboards` then `git status --porcelain grafana/dashboards` → empty (committed files match the template).
- G8.2 `curl -s 'http://localhost:3000/api/search?type=dash-db'` → includes uids `sci-node-daq`, `sci-node-compute`, `sci-node-storage`.
- G8.3 Identical modulo host: `for h in compute storage; do diff <(sed 's/daq/__HOST__/g' grafana/dashboards/node-daq.json) <(sed "s/$h/__HOST__/g" grafana/dashboards/node-$h.json); done` → no output.
- G8.4 Visual, each of the three: three rows titled `Status`, `Current values`, `History`; Status shows `UP`, an uptime duration and "seconds since last sample" ≤ 10; four gauges non-empty and inside that node's mock/collectd ranges; four time series over the last 15 minutes with ~180 points per series (5 s resolution).
- G8.5 Overview node titles navigate to the matching node dashboard.

**Commit:** `grafana: identical per-node dashboards generated from one template`

### Step 9 — Grafana: threshold alerts that go nowhere

**Build**
- `grafana/provisioning/alerting/rules.yaml`: the three rules, the `always` mute timing, root policy with `mute_time_intervals: [always]`, per §2.7.

**Gate G9**
- G9.1 `curl -s -u admin:admin http://localhost:3000/api/v1/provisioning/alert-rules` → exactly 3 rules: `sci-cpu-high`, `sci-temp-high`, `sci-node-down`.
- G9.2 Steady state: `curl -s -u admin:admin http://localhost:3000/api/prometheus/grafana/api/v1/rules` → every rule `"state": "inactive"`, `"health": "ok"`, with one instance per host in state `Normal`; `curl -s -u admin:admin http://localhost:3000/api/prometheus/grafana/api/v1/alerts` → no instance in a state other than `Normal` (the endpoint lists Normal instances too).
- G9.3 Node-down fires: `docker compose stop telegraf-compute` → within 2 minutes the alerts endpoint shows `sci-node-down` with label `host=compute` in state `firing`, and the overview alert-list panel shows it; `docker compose start telegraf-compute` → the instance leaves `firing` within 2 minutes.
- G9.4 Threshold fires: edit `telegraf/storage.conf` temperature mock range to 85–95, `docker compose restart telegraf-storage` → within 2 minutes `sci-temp-high{host=storage}` is `firing`; revert the range and restart → resolves within 2 minutes; `git diff --stat` empty.
- G9.5 Nothing is delivered: `curl -s -u admin:admin http://localhost:3000/api/v1/provisioning/policies` → root `receiver: empty` with one child route `alertname =~ .+` carrying `mute_time_intervals: ["always"]`; `curl -s -u admin:admin http://localhost:3000/api/v1/provisioning/contact-points` → `[]`; `curl -s -u admin:admin http://localhost:3000/api/v1/provisioning/mute-timings` → `always` present; during G9.3–G9.4 `docker compose logs grafana | grep -ci 'failed to send'` → `0`.

**Commit:** `grafana: cpu/temperature/node-down alert rules, delivery muted`

### Step 10 — Docs, diagram, demo script, cold and warm boot, portfolio

**Build**
- `README.md` (diagram first, quickstart, what's-in-this-repo table, the four headline features, filter/reshape summary, scaling pointer, portfolio link), `ARCHITECTURE.md` (topology, the two §2.4 tables, timestamps, retention, downsampling alignment, Grafana provisioning, token shortcut, gotchas, scaling to production incl. scoped tokens and the raw-shorter retention variant), `influxdb/schema.md` (final), `CLI_EXAMPLES.md` (list-databases, show-retention, rate-per-node, nanosecond-timestamps, rollup-vs-raw, plugin-logs, list-triggers), `FOR_MAINTAINERS.md` (licence-validated volume, bumping the plugin pin, regenerating dashboards), `diagrams/architecture.mmd` + `.png`, `scripts/demo.sh` (narrative: prereqs → bring-up → licence → data flowing on three nodes → rollups → open Grafana → summary).

**Gate G10**
- G10.1 `./scripts/demo.sh --no-browser --no-pause` → exit 0; every spinner reports ✓ (influxdb3 healthy, installer, init, raw data for 3 hosts, rollups).
- G10.2 Warm boot: `make down && make up` → within 3 minutes and with no manual action G3.2, G5.3, G6.3, G7.2 and G9.2 pass; `docker compose logs influxdb3-init` has no `FATAL`; installer logs show the overwrite no-op.
- G10.3 Cold boot: `make clean && make up`, click the licence link → `docker compose ps --format '{{.Name}} {{.Status}}'` shows the three one-shots `Exited (0)`, `sci-influxdb3` `(healthy)`, and `sci-grafana`, `sci-collectd`, `sci-telegraf-daq`, `sci-telegraf-compute`, `sci-telegraf-storage` `Up`; G5.3 and G6.3 pass within 3 minutes of the licence click.
- G10.4 `make cli-example name=<x>` exits 0 for every `## <x>` heading in `CLI_EXAMPLES.md`.
- G10.5 Doc checklist ticked against `CONVENTIONS.md` "Repo conventions": diagram is the first thing in the README; `schema.md` matches G2.4/G2.5/G6.9; ARCHITECTURE states the plugin-state and token caveats; `.env.example` documents every variable compose reads.
- G10.6 Meta-repo PR opened: README portfolio row + comparison-table column for this repo; CONVENTIONS.md additions — Telegraf `precision = "1ns"`, collectd needs a `types.db` in the Telegraf container, downsampler cron alignment + whole-second window truncation, Grafana `$__file{}` token provisioning.

**Commit:** `docs, diagram, demo script`

## 5. Definition of done

All ten gates pass in sequence on the target machine (macOS, Docker Desktop, arm64); G10.2 and G10.3 pass back-to-back; the meta-repo PR is open. Anything recorded during the gates (retention representation, retention-write behaviour, the working Grafana query body, the `call_time` finding) is copied into ARCHITECTURE.md or CONVENTIONS.md, whichever is the right home.
