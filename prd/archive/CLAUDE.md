# CLAUDE.md - Predictive Failure Detection System

## Project Overview

Predictive failure detection system that replaces rule-based approaches with data-driven pattern matching against historical failures. See `predictive_failure_prd_v1.1.md` for full specification.

## Core Concept

Everything becomes a **chunk** (sensor window, oil sample, event rollup) → embedded → compared via KNN to historical failure-adjacent chunks → scored → flagged if threshold met → agent enriches → human decides.

## Architecture

```
/src
  /core     # Pure logic, no IO (scoring, KNN, chunking)
  /api      # FastAPI - single API serves both UIs
  /cli      # Scripts, can bypass API for dev/debug
  /tui      # Textual - power user interface
  /web      # HTMX + Jinja - multi-user dashboard
```

**Key principle:** All business logic in `/core` and `/api`. UIs are thin clients.

## Build Order

1. **Core + API** - Backend logic, testable via curl
2. **TUI** - Immediate usability, validates API contracts
3. **Web** - Broader access once workflows validated

## Key Formulas

**Distance-weighted scoring:**
```
score = Σ(weight × neighbor_label) / Σ(weight)
where weight = 1 / distance
```

**Temporal score:** Decays with time from failure (configurable decay function).

## Data Types

| Type | Chunking | Embedding | Distance |
|------|----------|-----------|----------|
| Sensor | 5-min windows, stats + delta | Numeric vector (normalized) | Euclidean |
| Oil | Single sample | Numeric vector (normalized) | Euclidean |
| Events | 14-day rolling window | Text embedding | Cosine |

## Important Design Decisions

- **Flags become events** - enables cascade detection ("many low-value flags = high value")
- **Loop prevention** - derivative flags don't trigger new flags
- **Fleet baseline normalization** - no per-asset scalers, fleet-wide only
- **Agent always enriches, human always decides (v1)**

## Tech Stack

- Python 3.11+
- FastAPI (API)
- pgvector or Qdrant (vector DB)
- Textual (TUI)
- HTMX + Jinja (Web)
- Pydantic (shared contracts)

## API Endpoints

| Group | Purpose |
|-------|---------|
| `/flags` | List, filter, get details |
| `/flags/{id}/decision` | Submit review decision |
| `/queue` | Prioritized review queue |
| `/assets/{id}` | Asset history |
| `/chunks/{id}` | Chunk details, neighbors |
| `/admin/thresholds` | Threshold config |
| `/admin/mappings` | Component mapping CRUD |

## Conventions

- Type hints everywhere
- Pydantic models for all data contracts
- Tests in `/tests`, mirror `/src` structure
- No business logic in UI layers

## Sample Data

Place sample data in `/data/samples/`:
- `sensor_readings.csv` - raw sensor data
- `oil_samples.csv` - oil analysis results  
- `events.csv` - maintenance/fault records
- `failures.csv` - labeled failure events

## When Implementing

1. Read relevant PRD section first
2. Start with data models (Pydantic)
3. Implement core logic with tests
4. Add API endpoints
5. UI last
