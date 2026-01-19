# Operations

## Scope Boundary

The PFD system **consumes** clean, structured data—it does not own the pipelines that produce it.

ETL is explicitly out of scope because:
- Different concern (schema drift, deduplication, cleaning vs. pattern matching)
- Different change cadence (source systems vs. detection logic)
- Different ownership (DE team vs. data science/reliability)
- Scope creep risk

---

## Expected Input Tables

PFD expects these tables to exist. If missing or malformed, fail loudly.

### failures

Ground truth for labeling. Everything else scored by proximity to these.

| Field | Type | Required |
|-------|------|----------|
| failure_id | string | Yes |
| asset_id | string | Yes |
| failure_type | string | Yes |
| component_code | string | Yes |
| occurred_at | timestamp | Yes |
| description | string | No |
| downtime_hours | float | No |
| root_cause | string | No |

### events

Maintenance records, fault codes, operator reports.

| Field | Type | Required |
|-------|------|----------|
| event_id | string | Yes |
| asset_id | string | Yes |
| event_type | string | Yes |
| component_code | string | No |
| timestamp | timestamp | Yes |
| description | string | Yes |
| source_system | string | No |

### oil_samples

Lab results from oil analysis.

| Field | Type |
|-------|------|
| sample_id | string |
| asset_id | string |
| component_code | string |
| sample_date | timestamp |
| iron_ppm, silicon_ppm, aluminum_ppm, copper_ppm, lead_ppm | float |
| viscosity_40c, viscosity_100c | float |
| water_ppm, fuel_dilution_pct, soot_pct | float |
| oxidation, nitration, tan, tbn | float |

Actual analytes vary by lab.

### sensor_chunks (Future)

Pre-aggregated sensor windows. May come from separate ETL or computed within PFD from InfluxDB.

| Field | Type |
|-------|------|
| chunk_id | string |
| asset_id | string |
| sensor_name | string |
| window_start, window_end | timestamp |
| mean, std, min, max | float |
| delta_mean, delta_std, delta_min, delta_max | float |

---

## Signal Definition

Derived signals (transformations, ratios, rolling averages) are defined **externally** as SQL.

Examples:
```sql
-- Transmission ratio
SELECT timestamp, asset_id,
  (transmission_speed / engine_rpm) as trans_ratio
FROM sensor_data WHERE engine_rpm > 0

-- EGT delta
SELECT timestamp, asset_id,
  (egt_temp_1 - egt_temp_2) as egt_delta
FROM sensor_data
```

**Why external:**
- Signal definition is data engineering, not pattern matching
- Different tooling (SQL, dbt vs. Python)
- Different ownership

PFD consumes derived signals—doesn't own their creation.

---

## Feedback Capture

Every decision is stored for future learning:

```json
{
  "flag_id": "...",
  "chunk": {...},
  "matches": [
    {
      "chunk_id": "historical_chunk_123",
      "distance": 0.002,
      "weight": 500,
      "temporal_score": 0.92,
      "failure_event_id": "failure_456",
      "failure_type": "oil_pump_failure",
      "days_before_failure": 3
    }
  ],
  "weighted_score": 0.87,
  "agent_analysis": {...},
  "human_decision": {
    "action": "dismissed | escalated | monitored",
    "agreed_with_agent": true,
    "rationale": "..."
  },
  "timestamp": "..."
}
```

**V1:** Capture only, no automated action on feedback.

**Future:** Training data for improved scoring, threshold tuning, agent autonomy.

---

## Metrics from Feedback

Track over time:
- **Precision:** % of flags that were escalated or monitored
- **Agent agreement:** % of decisions matching agent recommendation
- **Time to decision:** From flag to human action
- **Asset patterns:** Which assets generate most false positives

---

## Validation Phase Approach

During validation, skip formal ETL:

1. Export CSVs manually from CMMS
2. Drop into `/data/inbox/`
3. Load directly into notebook or DuckDB

Fast, no integration work, sufficient to prove the core premise.
