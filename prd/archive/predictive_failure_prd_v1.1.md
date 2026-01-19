# Product Requirements Document
## Predictive Failure Detection System
### Version 1.1 | Draft

---

## 1. Executive Summary

This document defines a unified system for predicting equipment failures by analyzing multiple data sources (sensors, oil samples, events) against historical failure patterns. The system replaces fragmented, rule-based approaches with a data-driven pattern matching architecture that learns from historical outcomes.

**Core value proposition:** Instead of maintaining thousands of SME-defined rules that become contradictory and unvalidatable, let the data tell you what actually predicts failures.

---

## 2. Problem Statement

### 2.1 Current State
- Multiple disconnected data sources (sensors, oil samples, maintenance events, fault codes)
- SME-defined rules numbering in the thousands
- Rules are contradictory, unversioned, and impossible to validate
- High false positive rates eroding trust
- No systematic way to learn from historical failures
- Tribal knowledge not captured in systems

### 2.2 Desired State
- Unified system that ingests all data types
- Pattern matching against historical failure-adjacent data
- Explainable flags with links to historical evidence
- System that improves with each failure
- SMEs validate patterns rather than author rules

---

## 3. System Overview

### 3.1 Core Architecture

```
[Any Data Source]
       â†“
[Chunk + Embed]
  - Numeric: stats + delta, fleet baseline normalized
  - Text: text embedding
  - Events: rolling window, counts + combos
       â†“
[Score]
  - Temporal: distance-weighted from failure-adjacent neighbors
  - Relevance: component mapping matrix
  - Trust: source reliability (static v1)
       â†“
[Store in single index with labels]
       â†“
[Inference]
  - New chunk â†’ KNN â†’ distance-weighted neighbor labels â†’ score
       â†“
[Flag if threshold met]
       â†“
[Flag â†’ Event]
  - Individual flags feed back into event layer
  - Event clustering catches: "lots of low-value flags = high value signal"
       â†“
[Agent Enrichment]
  - Context, explanation, recommendation
  - Filters superficial matches
       â†“
[Human Review]
  - Final decision
  - Feedback captured
```

### 3.2 Core Concepts
- Everything becomes a chunk with an embedding, scores, and metadata
- Single index with all chunks; labels (temporal scores) do the heavy lifting
- Distance-weighted KNN (1/d) to infer danger level from neighbor labels
- Flags become events, enabling cascade detection
- Agent handles nuance that pure math cannot

---

## 4. Data Types and Processing

### 4.1 Sensor Data

#### 4.1.1 Chunking Strategy
- Fixed time windows (default: 5 minutes)
- Each chunk contains summary statistics plus delta from previous window

#### 4.1.2 Chunk Representation

| Feature | Description |
|---------|-------------|
| mean | Average value over window |
| std | Standard deviation over window |
| min | Minimum value over window |
| max | Maximum value over window |
| delta_mean | Change in mean from previous window |
| delta_std | Change in std from previous window |
| delta_min | Change in min from previous window |
| delta_max | Change in max from previous window |
| trend_slope | Slope over last N chunks (optional) |

If no previous window exists, delta values default to zero.

#### 4.1.3 Normalization
- Fleet baseline per signal per asset model
- Trimmed mean/std (exclude top/bottom 5%) to handle outlier sensors
- Baseline refreshed daily via batch job
- No per-asset scalers; fleet-wide baseline only
- Stored as small lookup table (~hundreds of rows)

#### 4.1.4 Distance Metric
- Euclidean distance (default)
- Cosine similarity as potential secondary metric (noted for future validation)

---

### 4.2 Oil Samples

#### 4.2.1 Chunking Strategy
- Single sample = one chunk
- Point-in-time snapshot

#### 4.2.2 Chunk Representation
- Normalized analyte values (iron ppm, silicon ppm, viscosity, etc.)
- Optional: delta from previous sample for same asset

#### 4.2.3 Distance Metric
- Euclidean distance on normalized analytes

---

### 4.3 Events (Maintenance Records, Fault Codes, etc.)

#### 4.3.1 Chunking Strategy
- Rolling window per asset (e.g., 14 days)
- Captures event patterns, not just individual events

#### 4.3.2 Chunk Representation

| Feature | Description |
|---------|-------------|
| event_counts_by_type | Count of each event type in window |
| days_since_first | Days since first occurrence of each type |
| event_trend | Increasing, decreasing, stable |
| total_events | Total event count in window |
| description_concat | Concatenated text descriptions |
| co_occurrences | Event type combinations observed |

#### 4.3.3 Embedding Approach
- Text embedding for description content (sentence transformer or similar)
- Structured prefix recommended: "component | event_type | description"

#### 4.3.4 Distance Metric
- Cosine similarity (standard for text embeddings)
- Convert to distance: `distance = 1 - cosine_similarity`

#### 4.3.5 Uniqueness Detection (Text Only)

Because cosine similarity is bounded (0-1), low similarity scores inherently indicate novel/unique patterns:
- If `max(neighbor_similarities) < 0.7`: flag as "novel, no historical match"
- This provides uniqueness detection for free with text embeddings

---

## 5. Metadata Schema

All chunks carry consistent metadata for filtering and audit:

| Field | Description | Required |
|-------|-------------|----------|
| asset_id | Unique asset identifier | Yes |
| site_id | Site/location identifier | Yes |
| timestamp | Chunk timestamp or window end | Yes |
| data_source | Origin system (sensor, oil lab, maintenance system) | Yes |
| component_code | Component classification | Yes |
| asset_model | Asset model (e.g., 789C) | Yes |
| machine_type | Machine type (e.g., Haul Truck) | Yes |
| event_id | Event identifier if applicable | No |
| sample_rate | For sensors: original sample rate | Conditional |
| window_size | For sensors: chunk window size | Conditional |
| has_prior_window | For sensors: whether delta is meaningful | Conditional |
| embedding_version | Version tag for embedding approach | Yes |

---

## 6. Scoring Model

### 6.1 Temporal Score (The Label)

Each chunk is labeled with its temporal proximity to a failure event. This score decays with time distance from failure.

- Chunk immediately before failure: high score (e.g., 0.95)
- Chunk 30 days before failure: low score (e.g., 0.1)
- Chunk with no associated failure: score = 0

The decay function (linear, exponential, etc.) is configurable per failure type.

### 6.2 Relevance Score

Filters out spurious correlations by mapping data sources to relevant failure types.

- SME-defined component mapping matrix (maintained in Excel, imported)
- Maps: signal/source â†’ component â†’ failure types
- Upstream/downstream relationships captured
- If chunk component does not map to failure component: low/zero relevance

Example: Suspension pressure data should not score high for turbo failures.

### 6.3 Trust Score

Weights data by source reliability.

- Static per source type for v1
- Examples: Live sensor (0.8), Operator report (0.6), Oil lab (0.9)
- Maintained as simple lookup table

### 6.4 Combined Scoring at Inference

When a new chunk arrives:

1. Embed the chunk
2. Filter index by metadata (asset model, component, etc.)
3. KNN to find top K neighbors (default K=10)
4. Calculate distance-weighted score from neighbor labels

Distance weighting formula (1/d):

```
score = Î£(weight Ã— neighbor_label) / Î£(weight)
where weight = 1 / distance
```

Closer neighbors dominate the score. Even if 8/10 neighbors are normal (low labels), two very close failure-adjacent neighbors will drive a high score.

### 6.5 Flagging Threshold

- If combined score > threshold: generate flag
- Threshold is tunable, calibrate after initial data run
- Start conservative (lower threshold, more flags), tune based on feedback
- Consider: Top N per day/shift as additional constraint to prevent overwhelming humans

---

## 7. Storage Architecture

### 7.1 Vector Database
- Single index containing all chunks
- Simple vector DB (pgvector, Qdrant, or similar)
- Metadata stored alongside embeddings for filtering

### 7.2 What Gets Stored
- All chunks that are failure-adjacent (have temporal score > 0)
- Sampled normal chunks (TBD: sampling rate)
- Raw values preserved alongside normalized/embedded values
- Embedding version tagged for future reprocessing

### 7.3 Fleet Baseline Table

Small lookup table for normalization:

| Field | Description |
|-------|-------------|
| asset_model | Asset model identifier |
| signal | Signal name |
| trimmed_mean | Fleet mean (excluding outliers) |
| trimmed_std | Fleet std (excluding outliers) |
| updated_at | Last refresh timestamp |

Approximately 30 signals Ã— 10 asset models = ~300 rows. Refreshed daily.

---

## 8. Inference Flow

Step-by-step process when new data arrives:

1. Raw data arrives from source
2. Signal definition applied (SQL-based transformations, external to system)
3. Chunk created based on data type strategy
4. Fleet baseline lookup for normalization (sensor data)
5. Embedding generated
6. KNN query against index (metadata filtered)
7. Distance-weighted score calculated from neighbor labels
8. If score > threshold: flag generated
9. Flag stored and added to human queue
10. Agent enrichment triggered (async)

---

## 9. Flags as Events

Critical design decision: flags from any data source become events themselves.

### 9.1 Why This Matters
- Individual sensor flag might be low value
- 10 sensor flags on same asset in 6 hours = high value signal
- Event clustering layer handles density, frequency, co-occurrence

### 9.2 Implementation
- When flag generated: create event record
- Event record contains: asset, timestamp, source type, flag details
- Event layer processes flags alongside maintenance events, fault codes, etc.
- Event clustering can now detect: "lots of low-value flags = high value signal"

### 9.3 Loop Prevention
- Flags marked as `source_type = "sensor_flag"` or `"oil_flag"` or similar
- Derivative events (flags) do not generate new flags
- Only raw events can trigger the flagging process

---

## 10. Agent Architecture

### 10.1 Role

The agent handles nuance that pure mathematical scoring cannot:

- Reviews every flag (human always makes final decision in v1)
- Pulls context: asset history, related sensors, recent events
- Checks for superficial matches (e.g., "matched oil pump failure but oil sensors healthy")
- Outputs structured reasoning and recommendation

### 10.2 Structured Output Schema

```json
{
  "flag_id": "...",
  "recommendation": "escalate | monitor | dismiss",
  "confidence": 0.0-1.0,
  "reasoning": {
    "similar_chunks": [...],
    "context_signals": [...],
    "key_factors": [...]
  },
  "summary": "Human-readable explanation",
  "agent_version": "1.0"
}
```

### 10.3 Example Agent Reasoning

Scenario: Chunk matches historical oil pump failure pattern, but oil sensors are currently healthy.

```json
{
  "recommendation": "dismiss",
  "confidence": 0.75,
  "reasoning": {
    "matched_failures": ["oil_pump_failure_042"],
    "key_indicators_in_historical": ["oil_temp_spike", "oil_pressure_drop"],
    "current_status": "oil_temp normal, oil_pressure normal",
    "conclusion": "Pattern match driven by non-causal sensors"
  },
  "summary": "Matched oil pump failure pattern but oil sensors are healthy. Likely false positive."
}
```

### 10.4 Phased Autonomy

Designed for eventual autonomous operation:

- **Phase 1 (v1):** Agent enriches all flags, human decides
- **Phase 2:** Agent earns trust on specific categories, selective autonomy
- **Phase 3:** Full autonomy with human override capability

Structured output enables measuring agent accuracy against human decisions.

---

## 11. Human Review Interface

### 11.1 Overview

Human review is delivered via two interfaces (see Section 16: Application Architecture):
- **TUI:** Power users, developers, SSH-accessible triage
- **Web Dashboard:** Multi-user access, richer visualizations

Both interfaces consume the same API and provide identical functionality.

### 11.2 Queue Structure
- Ticket-based queue (integrate with existing systems or standalone)
- Prioritized by: score Ã— asset criticality
- Each ticket contains: flag details, matched historical chunks, agent analysis

### 11.3 Available Actions
- **Dismiss:** False positive, no action needed
- **Monitor:** Worth watching, no immediate action
- **Escalate:** Create work order, immediate attention needed

### 11.4 Information Displayed
- Current chunk data and context
- Matched historical chunks with links
- What happened after historical matches (failures, downtimes)
- Agent recommendation and reasoning
- Asset history summary

---

## 12. Feedback Capture

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
    },
    {
      "chunk_id": "historical_chunk_789",
      "distance": 0.005,
      "weight": 200,
      "temporal_score": 0.85,
      "failure_event_id": "failure_012",
      "failure_type": "oil_pump_failure",
      "days_before_failure": 5
    }
  ],
  "weighted_score": 0.87,
  "agent_analysis": {...},
  "human_decision": {
    "action": "dismissed | escalated | monitored",
    "agreed_with_agent": true | false,
    "rationale": "..."
  },
  "timestamp": "..."
}
```

### 12.1 Match Details

Each match in the `matches` array captures full traceability:

| Field | Description |
|-------|-------------|
| chunk_id | ID of the matched historical chunk |
| distance | Distance from new chunk to this neighbor |
| weight | Calculated weight (1/distance) |
| temporal_score | The neighbor's label (proximity to failure) |
| failure_event_id | The failure this chunk preceded |
| failure_type | Type/category of failure |
| days_before_failure | How many days before failure this chunk occurred |

This allows human reviewers to see exactly why the flag was generated and trace back to specific historical failures.

**V1:** Capture only, no automated action on feedback.

**Future:** Training data for improved scoring, threshold tuning, agent autonomy.

---

## 13. Component Mapping

### 13.1 Structure

SME-maintained Excel file, imported into system:

| Field | Description |
|-------|-------------|
| signal_source | Data source or signal name |
| component_code | Component classification |
| upstream_components | Components that feed into this one |
| downstream_components | Components affected by this one |
| relevant_failure_types | Failure types this signal relates to |

### 13.2 Usage
- At scoring time: lookup relevance between chunk component and failure component
- High relevance: full weight on temporal score
- Low relevance: reduced weight or filter out
- Unmapped signals: default to low relevance, queue for SME mapping

---

## 14. Signal Definition

Transformations and derived signals are defined externally as SQL queries.

### 14.1 Examples

```sql
-- Transmission ratio
SELECT timestamp, asset_id,
  (transmission_speed / engine_rpm) as trans_ratio
FROM sensor_data
WHERE engine_rpm > 0
```

```sql
-- EGT delta (balance check)
SELECT timestamp, asset_id,
  (egt_temp_1 - egt_temp_2) as egt_delta
FROM sensor_data
```

### 14.2 Management
- SQL queries stored in version control
- Executed by existing data infrastructure (dbt, Airflow, etc.)
- Output lands in tables that chunking system reads from
- Signal registry tracks: name, query reference, output schema, owner

---

## 15. Technical Architecture

### 15.1 Infrastructure
- Docker-based deployment
- Local or cloud (not edge for v1)
- Simple vector DB (pgvector, Qdrant, or similar)

### 15.2 Processing Cadence
- Batch processing for v1
- Sensor chunks: processed hourly (or configurable)
- Oil samples: processed daily
- Events: processed as they arrive or daily batch

### 15.3 Scale Considerations

Estimated volume:
- 800 assets Ã— 30 sensors = 24,000 signals
- 5 min chunks â†’ ~288,000 chunks/hour across fleet
- Fleet baseline table: ~300 rows
- Failure-adjacent chunk storage: manageable subset of total

---

## 16. Application Architecture

### 16.1 Design Principle

Single API serving multiple frontends. All business logic lives in the API layer; UI clients are thin presentation layers only.

### 16.2 Component Structure

```
/src
  /core           # Pure logic (scoring, KNN, chunking) - no IO
  /api            # FastAPI - serves both UIs
  /cli            # Direct scripts, can bypass API for dev/debug
  /tui            # Textual (Python) - power user interface
  /web            # HTMX + Jinja - multi-user dashboard
```

### 16.3 API Layer

RESTful API (FastAPI) providing:

| Endpoint Group | Purpose |
|----------------|---------|
| `/flags` | List, filter, get flag details |
| `/flags/{id}/decision` | Submit review decision |
| `/queue` | Get prioritized review queue |
| `/assets/{id}` | Asset history, related flags |
| `/chunks/{id}` | Chunk details, matched neighbors |
| `/admin/thresholds` | View/update flagging thresholds |
| `/admin/mappings` | Component mapping CRUD |
| `/health` | System health, processing status |

Both TUI and Web hit the same endpoints. CLI can use API or import core directly for scripting/debugging.

### 16.4 TUI (Terminal User Interface)

**Technology:** Textual (Python)

**Target users:** Power users, developers, on-call engineers

**Key screens:**
- Queue view with keyboard navigation
- Flag detail with full context
- Quick decision input (d/m/e for dismiss/monitor/escalate)
- Asset deep-dive

**Advantages:**
- Fast iteration during development
- SSH-accessible for remote triage
- No browser/deployment overhead
- Stays in Python ecosystem

### 16.5 Web Dashboard

**Technology:** HTMX + Jinja (server-rendered, minimal JS)

**Target users:** Reliability engineers, supervisors, SMEs

**Key views:**
- Review queue (filterable, sortable)
- Flag detail page with visualizations
- Asset history timeline
- Admin: threshold tuning, mapping editor
- Metrics dashboard (precision, volume, agent agreement)

**Advantages:**
- Multi-user without terminal access
- Richer visualizations (charts, timelines)
- Mobile/tablet accessible
- Shareable URLs for specific flags

### 16.6 Shared Contracts

Both UIs consume identical API responses. Shared Pydantic models define:

```python
class FlagSummary(BaseModel):
    flag_id: str
    asset_id: str
    score: float
    timestamp: datetime
    agent_recommendation: str
    status: str  # pending | reviewed

class FlagDetail(BaseModel):
    flag: FlagSummary
    chunk: ChunkData
    matches: list[MatchDetail]
    agent_analysis: AgentOutput
    asset_context: AssetContext

class ReviewDecision(BaseModel):
    action: Literal["dismiss", "monitor", "escalate"]
    rationale: str | None
    agreed_with_agent: bool
```

### 16.7 Build Order

1. **Core + API** - Functional backend, testable via curl/Postman
2. **TUI** - Immediate usability for development and power users
3. **Web** - Broader team access once workflows validated

TUI development validates API contracts, accelerating web build.

---

## 17. Open Items and Future Work

### 17.1 Validation Required

| Item | Status | Notes |
|------|--------|-------|
| Sensor data pattern separation | Needs validation | Do failure-adjacent chunks cluster distinctly? |
| Distance threshold calibration | TBD | Tune after initial data run |
| K value for KNN | Default K=10 | May need adjustment per data type |
| Distance weighting function | Default 1/d | May need 1/dÂ² if separation weak |

### 17.2 Parked for Future Versions

| Feature | Notes |
|---------|-------|
| Uniqueness/anomaly detection | Shelved due to complexity; cosine similarity on text provides partial solution |
| Synthetic memories | SME-defined "bad patterns" without historical data; needs process design |
| SME rules as data source | Rules firing become events; requires rule cleanup/versioning first |
| Dynamic trust scores | Trust that adapts based on sensor health |
| Learned scoring weights | ML model to learn optimal Temporal Ã— Relevance Ã— Trust combination |
| Feedback loop automation | Acting on human feedback to tune system |
| Multi-asset fleet correlation | "5 different 789Cs flagged for same thing" detection |
| Edge deployment | Processing on vehicles |
| Dual distance metric | Cosine + Euclidean on numeric for shape + magnitude |
| Semantic embedding approach | Convert numeric to semantic descriptions (ref: XMPro MAGS) |

### 17.3 Known Limitations (v1)
- Will miss novel failure patterns (no historical precedent)
- Uniqueness detection limited to text embeddings (cosine bounded)
- Requires sufficient historical failure data to be useful
- Component mapping quality directly affects relevance scoring

---

## 18. Success Metrics

To be defined in detail, but directionally:

- **Precision:** Of flags reviewed, what percentage were actionable?
- **Recall:** Of actual failures, what percentage did we flag beforehand?
- **Time to action:** From flag to human decision
- **Agent agreement rate:** How often do humans agree with agent recommendation?
- **Avoided failures:** Catches before breakdowns (hard to measure, ultimate goal)

**Primary success criterion:** "Did we catch something before it failed that we would have missed otherwise?" Even once is a win.

---

## Appendix A: Glossary

| Term | Definition |
|------|------------|
| Chunk | Unit of data for embedding and comparison |
| Temporal Score | Label indicating proximity to failure (the training signal) |
| Relevance Score | Weight based on component mapping |
| Trust Score | Weight based on source reliability |
| Flag | Generated when new chunk scores above threshold |
| Failure-adjacent | Chunks that preceded a failure event |
| Fleet baseline | Statistical baseline across all assets of a model |
| KNN | K-nearest neighbors algorithm for similarity search |

---

## Appendix B: Data Flow Diagram

```
[Raw Data Sources]
       |
       v
[Signal Definition Layer] -- SQL transformations, cleaning, derived signals
       |
       v
[Chunking Layer] -- Time windows (sensor), point-in-time (oil), rolling window (events)
       |
       v
[Embedding Layer] -- Numeric: stats + delta normalized | Text: text embedding
       |
       v
[Scoring Layer] -- Temporal label from failure proximity
       |
       v
[Storage Layer] -- Vector DB with metadata
       |
       v
[Inference Layer] -- KNN, distance-weighted scoring
       |
       v
[Flagging Layer] -- Threshold check, flag generation
       |
       v
[Event Layer] -- Flags become events, clustering
       |
       v
[Agent Layer] -- Context, explanation, recommendation
       |
       v
[Human Review] -- Final decision, feedback capture
```
