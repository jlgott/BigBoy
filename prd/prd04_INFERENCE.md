# Inference and Flagging

## Inference Flow

### Sensors and Oil

1. Raw data arrives
2. Chunk created based on data type strategy
3. Feature vector constructed
4. Fleet baseline lookup for normalization
5. KNN query against pgvector index
6. Distance-weighted score calculated from neighbor labels
7. If score > threshold: flag generated
8. Agent enrichment triggered (async)

### Events

1. Event records arrive from CMMS
2. Rolling 14-day window constructed for asset
3. Token document created from events in window
4. BM25 search against historical event corpus
5. Similarity-weighted score calculated from neighbor labels
6. If score > threshold: flag generated
7. Agent enrichment triggered (async)

---

## Flag Generation

When a flag is generated:

```json
{
  "flag_id": "...",
  "chunk_id": "...",
  "asset_id": "...",
  "score": 0.87,
  "threshold": 0.75,
  "data_source": "sensor|oil|event",
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
  "timestamp": "..."
}
```

Full traceability: every flag links back to specific historical failures that drove the score.

---

## Flags as Events

**Critical design decision:** Flags become events themselves.

Why:
- Individual sensor flag might be low value
- 10 sensor flags on same asset in 6 hours = high value signal
- Event layer can detect density, frequency, co-occurrence

Implementation:
- Flag generates an event record with `source_type = "sensor_flag"` (or oil_flag, event_flag)
- Event layer processes flags alongside maintenance events, fault codes
- **Loop prevention:** Derivative flags don't generate new flags

```json
{
  "event_id": "flag_sensor_123_20260115_143022",
  "event_type": "SYSTEM_FLAG",
  "source_type": "sensor_flag",
  "asset_id": "truck_789_042",
  "timestamp": "2026-01-15T14:30:22Z",
  "component_code": "ENGINE_OIL_PUMP",
  "flag_details": {
    "flag_id": "...",
    "score": 0.87,
    "data_source": "sensor",
    "primary_signal": "oil_pressure_mean"
  }
}
```

---

## Agent Role

The agent handles nuance that pure math cannot:

- Reviews every flag (human always decides in v1)
- Pulls context: asset history, related sensors, recent events
- Checks for superficial matches
- Outputs structured reasoning and recommendation

### What the Agent Does

1. **Context gathering:**
   - Recent sensor readings for related components
   - Recent oil sample results
   - Recent maintenance events
   - Previous flags on same asset
   - Historical failure patterns for this asset model

2. **Superficial match detection:**
   - Cross-component confusion (matched turbo failure but turbo sensors healthy)
   - Temporal mismatch (historical showed 2-week degradation, current shows sudden spike)
   - Magnitude differences (historical had 10x signal strength)
   - Missing co-indicators (historical had correlated signals, current doesn't)

3. **Recommendation:** escalate | monitor | dismiss

### Agent Output Schema

```json
{
  "flag_id": "...",
  "recommendation": "escalate | monitor | dismiss",
  "confidence": 0.85,
  "reasoning": {
    "similar_chunks": [...],
    "context_signals": [...],
    "key_factors": [...]
  },
  "summary": "Human-readable explanation"
}
```

### Example

Scenario: Chunk matches historical oil pump failure pattern, but oil sensors currently healthy.

```json
{
  "recommendation": "dismiss",
  "confidence": 0.75,
  "reasoning": {
    "matched_failures": ["oil_pump_failure_042"],
    "key_indicators_in_historical": ["oil_temp_spike", "oil_pressure_drop"],
    "current_status": "oil_temp normal, oil_pressure normal"
  },
  "summary": "Matched oil pump failure pattern but oil sensors are healthy. Likely false positive."
}
```

---

## Phased Autonomy

- **Phase 1 (v1):** Agent enriches all flags, human decides
- **Phase 2:** Agent earns trust on specific categories, selective autonomy
- **Phase 3:** Full autonomy with human override

Structured output enables measuring agent accuracy against human decisions.
