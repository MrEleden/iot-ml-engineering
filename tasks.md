# Task Backlog

## Status Legend
- `TODO` — not started
- `IN PROGRESS` — actively being worked on
- `BLOCKED` — waiting on another task
- `DONE` — completed and reviewed

## Tasks

| ID | Status | Agent | Description | Blocked By |
|----|--------|-------|-------------|------------|
| T-001 | DONE | Architect | Define data contracts between pipeline stages (input/output schemas, SLAs, freshness guarantees) | — |
| T-002 | DONE | Data Engineer | Write `01-ingestion/architecture.md` — Foundry-native ingestion patterns | T-001 |
| T-003 | DONE | Data Engineer | Write `01-ingestion/streaming-vs-batch.md` — hybrid pattern with Foundry Transforms | T-001 |
| T-004 | DONE | Data Engineer | Write `01-ingestion/schema-management.md` — dataset expectations, schema evolution | T-001 |
| T-005 | DONE | Palantir Expert | Write `04-palantir/` foundational docs (streaming ingestion, Transform patterns, Ontology design, model integration) | T-001 |
| T-006 | DONE | Architect | Write `05-architecture/` system overview, data contracts, and 3 ADRs | — |
| T-007 | DONE | Data Engineer | Landing zone schema defined in `05-architecture/data-contracts.md` Contract 1 | T-001, T-005 |
| T-008 | DONE | Feature Engineer | Write `02-feature-engineering/` — time-domain, frequency-domain, cross-sensor, windowing, feature store | T-005, T-007 |
| T-009 | DONE | Feature Engineer | Write `02-feature-engineering/feature-store.md` — Ontology online serving, Delta Lake offline | T-005 |
| T-010 | DONE | ML Scientist | Write `06-modeling/` — model selection, training pipeline, evaluation framework, experiments, model cards | T-008 |
| T-011 | DONE | ML Platform | Write `03-production/data-quality.md` — Foundry dataset expectations for all contract boundaries | T-005 |
| T-012 | DONE | ML Platform | Write `03-production/testing.md` — Transform testing, synthetic data generation | T-005 |
| T-013 | DONE | ML Platform | Write `03-production/model-serving.md` and `03-production/monitoring.md` — multi-model scoring, 3-layer monitoring | T-010 |
