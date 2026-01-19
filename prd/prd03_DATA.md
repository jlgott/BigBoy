# Data Types and Processing

## Overview

Three data types, each with different chunking and similarity strategies:

| Type | Chunking | Representation | Distance Metric |
|------|----------|----------------|-----------------|
| Sensors | 5-min windows | Numeric vector (stats + delta) | Euclidean |
| Oil | Single sample | Numeric vector (analytes) | Euclidean |
| Events | 14-day rolling window | Token document | BM25 |

---

## Sensor Data

### Chunking Strategy

Fixed time windows (default: 5 minutes). Each chunk contains summary statistics plus delta from previous window.

### Chunk Features

| Feature | Description |
|---------|-------------|
| mean | Average value over window |
| std | Standard deviation |
| min | Minimum value |
| max | Maximum value |
| delta_mean | Change from previous window |
| delta_std | Change from previous window |
| delta_min | Change from previous window |
| delta_max | Change from previous window |

If no previous window exists, delta values default to zero.

### Normalization

- Fleet baseline per signal per asset model
- Trimmed mean/std (exclude top/bottom 5%) to handle outliers
- Baseline refreshed daily via batch job
- No per-asset scalers—fleet-wide only
- Stored as small lookup table (~hundreds of rows)

---

## Oil Samples

### Chunking Strategy

Single sample = one chunk. Point-in-time snapshot.

### Features

Normalized analyte values: iron ppm, silicon ppm, aluminum ppm, copper ppm, lead ppm, viscosity (40°C, 100°C), water ppm, fuel dilution %, soot %, oxidation, nitration, TAN, TBN.

Actual analytes vary by lab—schema matches what's available.

Optional: delta from previous sample for same asset.

---

## Events

### Design Rationale

Event data is primarily structured (fault codes, maintenance types) with minimal rich free-text. Descriptions aren't detailed enough for semantic embeddings.

Events use **structured token documents** with BM25 similarity:
- Handles new component codes without re-encoding
- Captures event frequency naturally (term frequency)
- Preserves co-occurrence patterns
- Provides interpretable match explanations

### Chunking Strategy

Rolling window per asset (default: 14 days). Captures patterns, not individual events.

### Token Format

`COMPONENT|SUBCOMPONENT|BREAKDOWN|TYPE`

Example 14-day window:
```
ENGINE|OIL_PUMP|MECHANICAL|FAULT
ENGINE|OIL_PUMP|MECHANICAL|FAULT
ENGINE|OIL_PUMP|MECHANICAL|FAULT
TRANSMISSION|TORQUE_CONV|ELECTRICAL|MAINT
ENGINE|COOLANT|SENSOR|FAULT
ENGINE|OIL_PUMP|MECHANICAL|MAINT
```

Token repetition naturally captures event frequency.

### Why BM25

| Alternative | Problem |
|-------------|---------|
| Euclidean on one-hot | Hyper-sparse, breaks with new codes |
| Text embedding | Loses structural patterns, requires re-encoding |
| Jaccard | Ignores frequency (1 fault = 15 faults) |
| **BM25** | ✓ Handles new tokens, captures frequency, interpretable |

---

## Scoring Model

### Temporal Score (The Label)

Each chunk is labeled by proximity to failure:
- Immediately before failure: high score (~0.95)
- 30 days before failure: low score (~0.1)
- No associated failure: score = 0

Decay function (linear, exponential) configurable per failure type.

### Distance-Weighted Scoring

**For sensors/oil (KNN):**
```
score = Σ(weight × neighbor_label) / Σ(weight)
where weight = 1 / distance
```

**For events (BM25):**
```
score = Σ(similarity × neighbor_label) / Σ(similarity)
where similarity = BM25 score (normalized 0-1)
```

Closer/more-similar neighbors dominate the score.

### Relevance Score (Optional)

Component mapping filters spurious correlations:
- Direct component match: relevance = 1.0
- Upstream component: relevance = 0.7
- Downstream: relevance = 0.5
- No relationship: relevance = 0.1

```
final_score = temporal_score × relevance
```

### Flagging

If combined score > threshold → generate flag.

Start conservative (lower threshold, more flags), tune based on feedback.

---

## Storage

### Sensors/Oil: pgvector

```sql
CREATE TABLE chunks (
    chunk_id text PRIMARY KEY,
    data_source text,  -- 'sensor' or 'oil'
    asset_id text,
    timestamp timestamptz,
    component_code text,
    asset_model text,
    embedding vector(N),
    temporal_score float,
    metadata jsonb
);
```

### Events: tsvector

```sql
CREATE TABLE event_chunks (
    chunk_id text PRIMARY KEY,
    asset_id text,
    window_start timestamptz,
    window_end timestamptz,
    event_tokens tsvector,
    temporal_score float,
    metadata jsonb
);

CREATE INDEX ON event_chunks USING GIN (event_tokens);
```

### Fleet Baseline Table

| Field | Description |
|-------|-------------|
| asset_model | Asset model identifier |
| feature_name | Feature name |
| trimmed_mean | Fleet mean (excluding outliers) |
| trimmed_std | Fleet std (excluding outliers) |
| updated_at | Last refresh |

---

## Metadata Schema

All chunks carry consistent metadata:

| Field | Required | Description |
|-------|----------|-------------|
| asset_id | Yes | Unique asset identifier |
| site_id | Yes | Site/location |
| timestamp | Yes | Chunk timestamp or window end |
| data_source | Yes | Origin (sensor, oil, event) |
| component_code | Yes | Component classification |
| asset_model | Yes | Asset model (e.g., 789C) |
| machine_type | Yes | Machine type (e.g., Haul Truck) |
| feature_version | Yes | Version tag for reprocessing |

---

## Component Mapping

SME-maintained Excel file mapping signals to components:

| Field | Description |
|-------|-------------|
| signal_source | Data source or signal name |
| component_code | Component classification |
| upstream_components | Components that feed into this one |
| downstream_components | Components affected by this one |
| relevant_failure_types | Related failure types |

Used at scoring time to weight relevance. Unmapped signals default to low relevance.
