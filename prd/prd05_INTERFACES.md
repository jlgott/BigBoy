# Interfaces

## Design Principle

Single API serving multiple frontends. All business logic in API layer; UI clients are thin presentation layers.

---

## API Endpoints

| Group | Endpoint | Purpose |
|-------|----------|---------|
| Flags | `GET /flags` | List, filter flags |
| | `GET /flags/{id}` | Flag details with matches |
| | `POST /flags/{id}/decision` | Submit review decision |
| Queue | `GET /queue` | Prioritized review queue |
| Assets | `GET /assets/{id}` | Asset history, related flags |
| Chunks | `GET /chunks/{id}` | Chunk details, matched neighbors |
| Admin | `GET /admin/thresholds` | View flagging thresholds |
| | `PUT /admin/thresholds` | Update thresholds |
| | `GET /admin/mappings` | Component mapping |
| | `PUT /admin/mappings` | Update mappings |
| Health | `GET /health` | System health, processing status |

---

## Shared Data Models

Both UIs consume identical API responses:

```python
class FlagSummary(BaseModel):
    flag_id: str
    asset_id: str
    score: float
    timestamp: datetime
    data_source: str
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

---

## TUI (Terminal User Interface)

**Technology:** Textual (Python)

**Target users:** Power users, developers, on-call engineers

**Why TUI:**
- Fast iteration during development
- SSH-accessible for remote triage
- No browser/deployment overhead
- Stays in Python ecosystem

**Key capabilities:**
- Queue view with keyboard navigation
- Flag detail with full context
- Quick decision input (d/m/e for dismiss/monitor/escalate)
- Asset deep-dive

---

## Web Dashboard

**Technology:** HTMX + Jinja (server-rendered, minimal JS)

**Target users:** Reliability engineers, supervisors, SMEs

**Why HTMX:**
- Server-rendered simplicity
- No frontend build process
- Progressive enhancement
- Mobile/tablet accessible

**Key capabilities:**
- Review queue (filterable, sortable)
- Flag detail page with visualizations
- Asset history timeline
- Admin: threshold tuning, mapping editor
- Metrics dashboard

---

## Human Review Workflow

### Queue Structure
- Prioritized by: score × asset criticality
- Each item contains: flag details, matched chunks, agent analysis

### Available Actions
- **Dismiss:** False positive, no action needed
- **Monitor:** Worth watching, no immediate action
- **Escalate:** Create work order, immediate attention needed

### Information Displayed
- Current chunk data and context
- Matched historical chunks with links
- What happened after historical matches
- Agent recommendation and reasoning
- Asset history summary
