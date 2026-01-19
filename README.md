# Predictive Failure Detection (PFD) System

## Executive Summary

A unified system for predicting equipment failures by analyzing multiple data sources (sensors, oil samples, events) against historical failure patterns. Replaces fragmented, rule-based approaches with data-driven pattern matching that learns from historical outcomes.

**Core value proposition:** Instead of maintaining thousands of SME-defined rules that become contradictory and unvalidatable, let the data tell you what actually predicts failures.

## The Problem

- Multiple disconnected data sources (sensors, oil samples, maintenance events, fault codes)
- SME-defined rules numbering in the thousands
- Rules are contradictory, unversioned, and impossible to validate
- High false positive rates eroding trust
- No systematic way to learn from historical failures
- Tribal knowledge not captured in systems

## The Solution

Everything becomes a **chunk** (sensor window, oil sample, event rollup) → embedded → compared via KNN/BM25 to historical patterns → scored → flagged if threshold met → agent investigates → human decides.

## Why "BigBoy"

This system is infrastructure for a larger vision: a fleet of AI agents that review every asset daily, investigating events, oil samples, sensor anomalies, and asset history to surface issues before they become failures.

Full agent coverage at 800 assets is currently cost-prohibitive (~thousands of API calls/day for proper investigation). BigBoy serves as:

1. **Data Foundation** - Clean, unified tables agents can query
2. **Knowledge Base** - Historical patterns stored as searchable embeddings
3. **Cost Optimization** - Pre-filter that surfaces 10-50 candidates/day instead of 800
4. **Feedback System** - Captures decisions for future learning

When agent costs drop or value is proven, BigBoy becomes a tool the agent fleet uses rather than the primary system.

## Core Concept

```
[Raw Data] → [Chunk + Embed] → [Score against history] → [Flag suspects] → [Agent investigates] → [Human decides] → [Feedback captured]
```

Flags also become events themselves, enabling cascade detection: "many low-value flags = high value signal."

## Current Phase

**Validation** - Proving the core premise before building infrastructure.

The entire system depends on one assumption: failure-adjacent chunks look different from normal operation. If they don't cluster distinctly, no amount of infrastructure saves us.

Validation order:
1. Oil samples (easiest - discrete, known predictive value)
2. Events (already labeled, patterns human-verifiable)  
3. Sensors (hardest - chunking strategy critical, noisy)

## Success Criteria

**Phase 1 (Validation):** At least one data type shows clear clustering in notebooks.

**Phase 2 (Pilot):** System catches at least one failure before breakdown.

**Phase 3 (Production):** System becomes trusted tool for reliability team.

**Ultimate metric:** "Did we catch something before it failed that we would have missed otherwise?"
