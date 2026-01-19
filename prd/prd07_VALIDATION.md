# Validation

## Current Phase

**Proving core premise before building infrastructure.**

The entire system depends on one assumption: failure-adjacent chunks cluster distinctly from normal operation. If they don't, no amount of infrastructure saves us.

---

## Validation Order

| Order | Data Type | Why This Order |
|-------|-----------|----------------|
| 1 | Oil samples | Easiest—discrete samples, known predictive value, clear physical intuition |
| 2 | Events | Already labeled, patterns human-verifiable |
| 3 | Sensors | Hardest—chunking strategy matters, noisy, less obvious signal |

---

## Approach

- Jupyter notebooks, local only
- Historical data for assets with known failures
- Visual inspection (UMAP/t-SNE) + basic clustering metrics
- No infra until signal confirmed

---

## Validation Questions

Each notebook should answer:

1. Do pre-failure chunks look different from baseline?
2. How far back does the signal appear? (days, weeks?)
3. Is the signal consistent across different assets/failure types?

---

## Success Criteria

| Result | Meaning | Action |
|--------|---------|--------|
| Clear separation | Strong signal | Proceed to build |
| Some separation with overlap | Might work | Proceed with tuned expectations |
| Total mush | Core premise fails | Rethink approach |

**Minimum bar:** Silhouette score > 0.3 for at least one data type.

---

## Decision Gates

### Gate 1: Validation Complete

Before building infrastructure:

- [ ] Oil samples show clear clustering?
- [ ] Events show BM25 similarity signal?
- [ ] Sensors show pattern separation?

If any fail: reassess approach for that data type. May proceed with subset.

### Gate 2: Infrastructure Build

Before expanding UI:

- [ ] API functional and tested?
- [ ] TUI validates workflows?
- [ ] Feedback from initial users positive?

### Gate 3: Production Deployment

Before declaring success:

- [ ] System catches at least one failure before breakdown?
- [ ] Precision acceptable (>30% flags actionable)?
- [ ] Agent enrichment adding value?

---

## Known Constraints

### Oil Sample Survivorship Bias

Bad oil sample → intervention → no failure recorded.

Your best predictor has survivorship bias baked in. The failures you have are likely:
- Sudden/catastrophic (no oil sample warning)
- Ignored warnings
- Components not covered by oil analysis

**Options:**
1. Reframe label: "preceded intervention OR failure"
2. Use intervention as ground truth
3. Accept limitation—oil validates math, events/sensors catch what oil misses

### Oil Sample Lag

Lab results take 1-2 weeks. Events and sensors become critical during the lag period.

---

## Open Technical Questions

| Item | Status | Notes |
|------|--------|-------|
| Distance threshold calibration | TBD | Tune after initial data run |
| K value for KNN | Default K=10 | May adjust per data type |
| Distance weighting function | Default 1/d | May need 1/d² if separation weak |
| BM25 parameters | Default k1=1.5, b=0.75 | Tune for event matching |
| Event token structure | TBD | Final delimiter and ordering |
| Normal chunk sampling rate | TBD | What % to store for baseline |

---

## Success Metrics

### Primary

| Metric | Target (v1) | Target (v2) |
|--------|-------------|-------------|
| Precision | >30% | >50% |
| Recall | >20% | >40% |
| Time to action | <4 hours median | <2 hours |
| Agent agreement | >60% | >75% |

### Secondary

- False positive rate (monitor alert fatigue)
- Lead time (days between flag and failure)
- Coverage (% of asset-components with sufficient data)
- Queue backlog (ensure humans can keep up)

### Ultimate Metric

**"Did we catch something before it failed that we would have missed otherwise?"**

Even once is a win.
