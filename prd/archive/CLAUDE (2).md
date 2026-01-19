# CLAUDE.md - Predictive Failure Detection System

## Project Overview

Predictive failure detection system that replaces rule-based approaches with data-driven pattern matching against historical failures. See `predictive_failure_prd_v1.1.md` for full specification.

## Core Concept

Everything becomes a **chunk** (sensor window, oil sample, event rollup) â†’ embedded â†’ compared via KNN to historical failure-adjacent chunks â†’ scored â†’ flagged if threshold met â†’ agent enriches â†’ human decides.

## Current Phase: Validation

**Proving core premise before building infrastructure.**

The entire system depends on one assumption: failure-adjacent chunks cluster distinctly from normal operation. If they don't, no amount of infrastructure saves us.

### Validation Order

| Order | Data Type | Why This Order |
|-------|-----------|----------------|
| 1 | Oil samples | Easiestâ€”discrete samples, known predictive value, clear physical intuition |
| 2 | Events | Already labeled, text embeddings off-the-shelf, patterns human-verifiable |
| 3 | Sensors | Hardestâ€”chunking strategy matters, noisy, less obvious signal |

### Validation Approach

- Jupyter notebooks, local only
- Historical data for assets with known failures
- Visual inspection (UMAP/t-SNE) + basic clustering metrics
- No infra until signal confirmed

### Success Criteria

| Result | Meaning | Action |
|--------|---------|--------|
| Clear separation | Strong signal | Proceed to build |
| Some separation with overlap | Might work | Proceed with tuning expectations |
| Total mush | Core premise fails | Rethink approach |

### Validation Outputs

Each validation notebook should answer:
1. Do pre-failure chunks look different from baseline?
2. How far back does the signal appear? (days, weeks?)
3. Is the signal consistent across different assets/failure types?

## Build Order (Post-Validation)

1. **Core + API** - Backend logic, testable via curl
2. **TUI** - Immediate usability, validates API contracts
3. **Web** - Broader access once workflows validated

## Architecture

```
/
├── pfd/                  # Main package
│   ├── __init__.py
│   ├── core/             # Pure logic, no IO (scoring, KNN, chunking)
│   ├── api/              # FastAPI - single API serves both UIs
│   ├── cli/              # Scripts, can bypass API for dev/debug
│   ├── tui/              # Textual - power user interface
│   └── web/              # HTMX + Jinja - multi-user dashboard
├── tests/                # Mirrors pfd/ structure
├── notebooks/
├── data/
├── pyproject.toml
└── Dockerfile
```

**Imports:** `from pfd.core.scoring import ...`

**Local dev:** `pip install -e .`

**Docker:** `pip install .` (non-editable, installs to site-packages)

**Key principle:** All business logic in `pfd/core` and `pfd/api`. UIs are thin clients.

## Key Formulas

**Distance-weighted scoring:**
```
score = Î£(weight Ã— neighbor_label) / Î£(weight)
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
- **Duplicate chunks OK** - same data can be stored multiple times with different failure labels; simpler than multi-label, math handles overlap naturally

## Tech Stack

- Python 3.11+
- FastAPI (API)
- pgvector on Azure Database for PostgreSQL (vector DB)
- Textual (TUI)
- HTMX + Jinja (Web)
- Pydantic (shared contracts)

## Infrastructure (Production)

| Component | Service |
|-----------|---------|
| Database | Azure Database for PostgreSQL + pgvector |
| API | Azure Container Apps |
| Container Registry | Azure Container Registry |
| Network | VNet (DB private, API public endpoint) |
| Dev Access | VPN Gateway or jump box VM |

No staging environmentâ€”dev flows direct to prod.

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
- Tests in `tests/`, mirror `pfd/` structure
- No business logic in UI layers

## Data Locations

**Validation phase:** `notebooks/`

**Sample data:** `data/samples/` (sensor_readings.csv, oil_samples.csv, events.csv, failures.csv)

## When Implementing

1. **Validation phase:** Start with notebooks, prove signal exists
2. Read relevant PRD section
3. Start with data models (Pydantic)
4. Implement core logic with tests
5. Add API endpoints
6. UI last
