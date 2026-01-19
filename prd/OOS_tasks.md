# Out of Scope Tasks

## Overview

This document defines work that is **outside the boundary** of the Predictive Failure Detection (PFD) system. The PFD system consumes clean, structured data—it does not own the pipelines that produce it.

**Responsibility is variable:** These tasks may be owned by a future DE team, a separate repo, or manual processes during validation. The PFD system is agnostic to how data arrives.

---

## ETL Pipelines

### What's Needed

| Pipeline | Source | Destination | Frequency |
|----------|--------|-------------|-----------|
| Failures | CMMS (direct DB or CSV export) | `failures` table | Daily or on-demand |
| Events | CMMS (direct DB or CSV export) | `events` table | Daily or on-demand |
| Oil Samples | External lab CSV drops | `oil_samples` table | As received |
| Sensor Data | InfluxDB | `sensor_chunks` table | Hourly (future) |

### Why Out of Scope

- **Different concern:** ETL handles messy source data, schema drift, deduplication, cleaning. PFD handles pattern matching and scoring.
- **Different change cadence:** ETL changes when source systems change. PFD changes when detection logic evolves.
- **Different ownership:** ETL may eventually belong to a DE team. PFD is data science/reliability tooling.
- **Scope creep risk:** Mixing ETL into PFD bloats the codebase and muddies responsibilities.

---

## Expected Table Schemas

PFD expects these tables to exist with these minimum fields. If missing or malformed, the system should fail loudly.

### `failures`

The ground truth for labeling. Everything else gets scored by proximity to these.

| Field | Type | Description |
|-------|------|-------------|
| failure_id | string | Unique identifier |
| asset_id | string | Asset that failed |
| failure_type | string | Category (e.g., oil_pump_failure, turbo_failure) |
| component_code | string | Component classification |
| occurred_at | timestamp | When failure occurred |
| description | string | Free text description (optional) |
| downtime_hours | float | Duration of downtime (optional) |
| root_cause | string | Root cause if determined (optional) |

### `events`

Maintenance records, fault codes, operator reports, etc.

| Field | Type | Description |
|-------|------|-------------|
| event_id | string | Unique identifier |
| asset_id | string | Asset involved |
| event_type | string | Category (fault_code, maintenance, operator_report, etc.) |
| component_code | string | Component classification (if known) |
| timestamp | timestamp | When event occurred |
| description | string | Free text description |
| source_system | string | Origin system (optional) |

### `oil_samples`

Lab results from oil analysis.

| Field | Type | Description |
|-------|------|-------------|
| sample_id | string | Unique identifier |
| asset_id | string | Asset sampled |
| component_code | string | Component sampled (engine, transmission, etc.) |
| sample_date | timestamp | When sample was taken |
| iron_ppm | float | Iron concentration |
| silicon_ppm | float | Silicon concentration |
| aluminum_ppm | float | Aluminum concentration |
| copper_ppm | float | Copper concentration |
| lead_ppm | float | Lead concentration |
| viscosity_40c | float | Viscosity at 40°C |
| viscosity_100c | float | Viscosity at 100°C |
| water_ppm | float | Water content |
| fuel_dilution_pct | float | Fuel dilution percentage |
| soot_pct | float | Soot percentage |
| oxidation | float | Oxidation level |
| nitration | float | Nitration level |
| tan | float | Total acid number |
| tbn | float | Total base number |

*Note: Actual analytes may vary by lab. Schema should match what's available.*

### `sensor_chunks` (Future)

Pre-aggregated sensor windows. May be populated by a separate sensor ETL pipeline or computed within PFD from raw InfluxDB queries.

| Field | Type | Description |
|-------|------|-------------|
| chunk_id | string | Unique identifier |
| asset_id | string | Asset |
| sensor_name | string | Sensor identifier |
| window_start | timestamp | Window start time |
| window_end | timestamp | Window end time |
| mean | float | Mean value over window |
| std | float | Standard deviation |
| min | float | Minimum value |
| max | float | Maximum value |
| delta_mean | float | Change from prior window |
| delta_std | float | Change from prior window |
| delta_min | float | Change from prior window |
| delta_max | float | Change from prior window |

---

## Validation Phase Approach

During validation, skip formal ETL:

1. Export CSVs manually from CMMS
2. Drop into `/data/inbox/`
3. Load directly into DuckDB via notebook or simple script

This is fast, requires no integration work, and sufficient to prove the core premise.

---

## Future State

When/if a DE team exists or production demands it:

- Formal ETL in separate repo
- Scheduled jobs (Airflow, cron, Azure Data Factory, etc.)
- Proper monitoring, alerting, data quality checks
- PFD remains a consumer, not an owner
