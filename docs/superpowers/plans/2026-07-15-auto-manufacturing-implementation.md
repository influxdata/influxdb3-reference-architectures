# Implementation plan — `influxdb3-ref-auto-manufacturing`

**Date:** 2026-07-15
**Design:** `influxdb3-ref-auto-manufacturing/docs/superpowers/specs/2026-07-10-auto-manufacturing-design.md` (per-repo, per convention A1)
**Template:** `influxdb3-ref-iiot` (canonical single-node), with the deviations listed in the design's §10.

## Sequencing

Tasks are ordered so the two risky assumptions are exercised as early as a running
stack exists, and so every later task builds on a bootable stack.

### Task 1 — Repo bootstrap
- LICENSE (Apache 2.0), .gitignore, design doc committed. GitHub repo under
  `influxdata` org (blocked on org permissions at time of writing — local git
  until resolved).

### Task 2 — Core infra (bootable stack, no UI)
- `docker-compose.yml`: token-bootstrap → influxdb3 (`--mode all`,
  `--plugin-dir /var/lib/influxdb3/plugins` inside the data volume, **no plugin
  bind mount**) → plugin-installer (one-shot, python:3.12-slim) → influxdb3-init
  (one-shot) → ui → scenarios (profile).
- `installer/install_plugins.py` + `plugins.lock`: registry index fetch, pinned
  resolve, sha256 verify, entry-point classify, files-API upload
  (`{name}-{version}/<relpath>`), deps install (chronos deferred), local stub
  upload under `local/`, overwrite-on-boot idempotency, `INSTALLER_OFFLINE_DIR`.
- `influxdb/init.sh`: db `auto`, six tables w/ retention via configure API, LVC,
  triggers (signal_generator via configure API — JSON tags; stubs + downsampler +
  chronos via CLI), delete-and-recreate trigger convention, FORECAST_ENABLED gate.
- Stubs `plugins/wal_{resample,fill,filter}.py`: pass-through copy with header
  contract comments.
- `Makefile`, `.env.example`, `pyproject.toml`, `influxdb/schema.md`.

### Task 3 — Risk verification (gate for everything downstream)
With the Task-2 stack running:
1. **files-API path separators**: does `plugin_name: "signal_generator-0.2.0/signal_generator.py"`
   land and resolve as a trigger `--path`? Fallback: flat names.
2. **Chained WAL triggers**: does a stub's `influxdb3_local.write()` fire the next
   `table:` trigger? Fallback: stubs become `every:1s` schedule triggers with
   last-time cache (design §3.3).
3. Downsampler `interval=1s,window=5s,calculations=median` produces `sensor_clean`
   rows with `record_count`.
Record outcomes in the design doc's amendment section.

### Task 4 — UI
- `ui/queries.py` (all SQL named + docstring'd), `ui/app.py` (overview `/`,
  station `/station/{station}`, JSON overlay endpoint, HTMX partials).
- Templates: base/overview/station + partials (station cards, KPI strip, pipeline
  panel, alarm panel).
- `static/app.js`: overlay chart component (raw red / clean green shifted
  `CLEAN_SHIFT_MS` / forecast dashed + 80% band past now), episode strip, latency
  badges, `bucketToSeconds()` parser reuse. `app.css` black theme. Vendored
  uPlot/HTMX copied from iiot.

### Task 5 — Scenarios
- `scenarios/_api.py` (configure-API helpers), `spike_storm.py` (temp booth-02
  trigger), `heat_event.py` (booth-01 ramp trigger). Narrative stdout.

### Task 6 — Tests + CI
- Unit: stubs (fake `influxdb3_local`, monkeypatched `LineBuilder`), installer
  (fixture artifacts, tampered-hash rejection, payload shapes), queries.
- Scenario tier: testcontainers, `FORECAST_ENABLED=false`,
  `INSTALLER_OFFLINE_DIR` fixtures — hermetic.
- Smoke: real `make up`. Workflows: unit/scenarios/smoke/lint (iiot templates).

### Task 7 — Docs + demo
- README (diagram first, quickstart, what's-in-this-repo table, headline
  features, scaling pointer), ARCHITECTURE, SCENARIOS, CLI_EXAMPLES,
  FOR_MAINTAINERS, `diagrams/architecture.mmd` (+png), `scripts/demo.sh`
  narrative script, `scripts/setup.sh`.

### Task 8 — Portfolio integration
- Meta-repo PR: README row #9 + comparison-table column + CONVENTIONS.md
  additions (registry-install pattern, chained-WAL findings).

## Definition of done
Design §12 success criteria, plus: risk-verification outcomes recorded; CI green;
meta-repo PR open.
