# Architecture

## System Overview

```
[Any Data Source]
       ↓
[Chunk + Vectorize]
  - Sensors: stats + delta, fleet baseline normalized
  - Oil: normalized analytes  
  - Events: structured token documents
       ↓
[Score]
  - KNN (sensors/oil) or BM25 (events)
  - Distance/similarity-weighted neighbor labels
       ↓
[Store with labels]
  - Sensors/Oil: pgvector (Euclidean)
  - Events: PostgreSQL tsvector (BM25)
       ↓
[Inference: new chunk → query → weighted score]
       ↓
[Flag if threshold met]
       ↓
[Flag → Event (cascade detection)]
       ↓
[Agent Enrichment]
       ↓
[Human Review → Feedback Captured]
```

## Package Structure

```
/
├── pfd/                    # Main package
│   ├── __init__.py
│   ├── core/               # Pure logic, no IO
│   │   ├── chunking.py
│   │   ├── scoring.py
│   │   ├── normalization.py
│   │   └── models.py
│   ├── api/                # FastAPI - single API serves both UIs
│   ├── cli/                # Scripts, can bypass API for dev/debug
│   ├── tui/                # Textual - power user interface
│   └── web/                # HTMX + Jinja - multi-user dashboard
├── tests/                  # Mirrors pfd/ structure
├── notebooks/              # Validation work
├── data/                   # Sample data
├── pyproject.toml
└── Dockerfile
```

**Key principle:** All business logic in `pfd/core` and `pfd/api`. UIs are thin clients.

**Imports:** `from pfd.core.scoring import distance_weighted_score`

**Local dev:** `pip install -e .`

**Docker:** `pip install .` (non-editable)

## Tech Stack

| Component | Technology |
|-----------|------------|
| Language | Python 3.11+ |
| Database | PostgreSQL 15+ with pgvector |
| Text Search | PostgreSQL tsvector |
| API | FastAPI |
| TUI | Textual |
| Web | HTMX + Jinja2 |
| Data Models | Pydantic |
| Container | Docker |

## Production Infrastructure (Azure)

| Component | Service |
|-----------|---------|
| Database | Azure Database for PostgreSQL + pgvector |
| API | Azure Container Apps |
| Container Registry | Azure Container Registry |
| Network | VNet (DB private, API public endpoint) |
| Dev Access | VPN Gateway or jump box VM |

No staging environment—dev flows direct to prod.

## Build Order

1. **Core + API** - Backend logic, testable via curl
2. **TUI** - Immediate usability, validates API contracts
3. **Web** - Broader access once workflows validated

## Processing Cadence

- **Sensors:** Batch hourly (configurable)
- **Oil samples:** Daily
- **Events:** As they arrive or daily batch

## Scale Estimates

- 800 assets × 30 sensors = 24,000 signals
- 5-min chunks → ~288,000 chunks/hour across fleet
- Fleet baseline table: ~300 rows
- Failure-adjacent storage: manageable subset of total
