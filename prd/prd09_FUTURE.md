# Future Directions

## The Vision

A fleet of AI agents that review every asset daily, investigating events, oil samples, sensor anomalies, and asset history to surface issues before they become failures.

BigBoy is the infrastructure that makes this possible.

---

## Agent Fleet Architecture

### The Dream

800 assets × daily agent review. Each agent:
- Pulls today's events → reasons over them
- Checks recent oil samples → flags concerns
- Cross-references events ↔ oil ↔ component relationships
- Scans sensor anomalies across the asset
- Reviews asset history, open work orders
- Synthesizes → recommendation

That's 5-10 tool calls per asset minimum. 8,000+ calls/day.

### Why Not Now

At current costs, ~$100-200/day for full coverage. Not unreasonable, but:
- Need to prove value first
- Need clean data infrastructure first
- Need agent skills defined first

### BigBoy's Role

Once agent fleet scales:
- BigBoy becomes a **tool** agents use, not the primary system
- "Hey BigBoy, anything similar to this pattern in history?"
- KNN is one query among many the agent makes
- Knowledge base is shared across all agents

---

## SKILLS.md Files

Agent capabilities defined as skill documents:

| Skill | Purpose |
|-------|---------|
| `SKILLS_events.md` | How to interpret event patterns, what combinations are concerning |
| `SKILLS_oil.md` | Oil analysis interpretation, thresholds, component-specific concerns |
| `SKILLS_sensors.md` | Anomaly detection heuristics per sensor type |
| `SKILLS_components.md` | What connects to what, failure propagation paths |
| `SKILLS_history.md` | How to weigh asset history, open WOs, repeat offenders |
| `SKILLS_triage.md` | When to escalate vs. monitor vs. dismiss |

Each skill is domain knowledge currently in SME heads or scattered docs.

---

## Synthetic Memories

### The Concept

Three sources of "what's bad":

1. **Historical** — Patterns that preceded actual failures
2. **Synthetic** — SME says "if you ever see X + Y + Z, that's bad" (hasn't happened yet)
3. **Learned** — Agent flags something, SME confirms, pattern gets stored

All three feed the same retrieval system.

### Open Problems

**Input mechanism:** SMEs won't write JSON. Need something like:
- Plain text description → agent converts to structured chunk
- Guided form with dropdowns for components, event types
- "Show me similar patterns" → SME confirms/denies

**Validation:** How do you know a synthetic memory is even matchable? SME might describe something that never triggers.

**Confidence:** Real failures have ground truth. Synthetic is opinion—weight differently?

### Possible Workflow

1. SME describes a bad scenario in plain text
2. Agent interprets → proposes structured representation
3. SME reviews, adjusts
4. Pattern stored with `source_type = "synthetic"`
5. Lower initial weight, gains weight if it matches real events that become failures

---

## Learning Loop

### Current (v1)

Feedback captured but not acted on.

### Future

1. **Threshold tuning:** Automated adjustment based on precision/recall
2. **Weight learning:** Which features actually matter for which failure types
3. **Agent calibration:** Track agent vs. human agreement, adjust prompts
4. **Pattern refinement:** Historical chunks that lead to dismissals get downweighted

---

## Fleet-Wide Detection

### The Problem

Currently: flag per asset.

Missing: "5 different 789Cs flagged for same thing this week."

### Future Capability

- Cross-asset pattern detection
- "Fleet-wide alert: unusual turbo event frequency across Site A"
- Potential recall/bulletin detection
- Shared failure modes surfaced automatically

---

## Edge Deployment

### Current

All processing centralized (cloud or on-prem server).

### Future Possibility

- Lightweight scoring on vehicles
- Only upload chunks that score above threshold
- Reduce data transmission costs
- Enable real-time alerting

Requires: embedded runtime, model quantization, reliable connectivity.

---

## Parked Features

Not planned for v1, revisit based on validation:

| Feature | Notes |
|---------|-------|
| Uniqueness/anomaly detection | Shelved—adds complexity |
| SME rules as data source | Rules firing become events; requires rule cleanup first |
| Dynamic trust scores | Trust adapts based on sensor health |
| Text embeddings for event descriptions | If rich free-text proves valuable |

---

## Decision Points

### When to Scale Agents

- Agent costs drop below $X/call
- BigBoy demonstrates clear value
- SKILLS.md files documented and validated
- Clean data infrastructure stable

### When to Add Synthetic Memories

- Input mechanism designed and tested
- SME workflow validated
- Weighting strategy decided

### When to Go Fleet-Wide

- Single-asset detection mature
- Cross-asset patterns observed manually
- Infrastructure can handle correlation queries
