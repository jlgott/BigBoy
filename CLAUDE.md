# CLAUDE.md

Quick reference for this project.

## What This Is

Predictive Failure Detection (PFD) system—codename "BigBoy."

Pre-filter for an eventual AI agent fleet. Surfaces suspect assets so agents investigate 10-50/day instead of 800.

## Current Phase

**Validation.** Proving failure-adjacent chunks cluster distinctly before building infrastructure.

Order: Oil samples → Events → Sensors

## Key Files

| File | Purpose |
|------|---------|
| prd01_README.md | Vision and summary |
| prd02_ARCHITECTURE.md | System design, tech stack |
| prd03_DATA.md | Data types, chunking, scoring, storage |
| prd04_INFERENCE.md | How scoring and flagging works |
| prd05_INTERFACES.md | API and UI specs |
| prd06_OPERATIONS.md | ETL expectations, feedback capture |
| prd07_VALIDATION.md | Current phase, success criteria |
| prd08_GLOSSARY.md | Terms |
| prd09_FUTURE.md | Agent fleet, synthetic memories, roadmap |
| OOS_tasks.md | Describes critical ETL processes that are Out-Of-Sample for this agent |
| BigBoy_Overview.md | C-suite level pitch doc with no technical jargon |

## Core Formula

```
score = Σ(weight × neighbor_label) / Σ(weight)
where weight = 1 / distance
```

## Build Order

1. Validation notebooks (current)
2. Core + API
3. TUI
4. Web

## Tech Stack

Python 3.11+, FastAPI, PostgreSQL + pgvector, Textual (TUI), HTMX + Jinja (web).

## Package Structure

```
pfd/
├── core/     # Pure logic, no IO
├── api/      # FastAPI
├── cli/      # Scripts
├── tui/      # Textual
└── web/      # HTMX + Jinja
```

All business logic in `core` and `api`. UIs are thin clients.
