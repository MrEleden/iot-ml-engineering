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
| T-013 | DONE | ML Platform | Write `03-production/model-serving.md` and `03-production/monitoring.md` — multi-model scoring, 3-layer monitoring | T-010 || T-014 | DONE | All Agents | Huyen review — review all docs against *Designing Machine Learning Systems* (Chip Huyen, 2022), apply HIGH/MEDIUM fixes | T-001–T-013 |

## T-014 Details: Huyen Review — Changes Applied

All documents reviewed by each specialist agent against Huyen's ML Systems framework. HIGH and MEDIUM severity gaps addressed. Summary of cross-cutting themes added:

### Architecture (Architect)
- System overview: added training/retraining subgraph, feedback loop, monitoring component, model lifecycle table, adaptability section
- Data contracts: added Contract 3.5 (training data assembly), Contract 7 (label data placeholder), feature baselines, point-in-time correctness clause, imputation strategy
- ADR-001: added ML lifecycle capabilities evaluation for Foundry
- ADR-002: added threshold synchronization, model deployment strategy, batch output monitoring
- ADR-003: added drift detection & retraining triggers, evaluation without labels, Phase 2 transition criteria, feedback loop latency

### Ingestion (Data Engineer)
- architecture.md: added retention strategy (hot/cold tiers), dataset snapshots (Delta Lake time travel), columnar format note, distribution shift monitoring at ingestion
- streaming-vs-batch.md: added training data as second consumer pattern, training data access row in table
- schema-management.md: added `timestamp_available` for leakage prevention, training data consumer backward compatibility guidance
- README.md: added dual consumer patterns, training retention metrics

### Feature Engineering (Feature Engineer)
- time-domain.md: added feature scaling, lag feature leakage warning, distribution monitoring, feature importance validation
- frequency-domain.md: added validation protocol for spectral features, spectral drift monitoring
- cross-sensor.md: added z-score leakage warning, covariate shift monitoring, equipment model generalization
- windowing.md: added temporal train/test split guidance, empty window distribution note
- feature-store.md: added feature distribution monitoring, training-serving consistency, feature statistics as model artifact
- README.md: added feature quality criteria, cross-cutting concerns table

### Modeling (ML Scientist)
- model-selection.md: added SHAP interpretability, complexity vs data size, Phase 2 class imbalance handling
- training-pipeline.md: added stateless vs stateful retraining, sampling strategy, rolling window forgetting risk, data augmentation
- evaluation-framework.md: added statistical drift detection (KS/PSI/KL), distribution shift taxonomy, shadow + canary deployment, feedback loop risks
- experiment-tracking.md: added training_data_stats, loss curves, latency metrics to schemas
- model-cards.md: added staleness indicator, fairness and equity section
- README.md: added monitoring ownership note

### Production (ML Platform)
- monitoring.md: added multi-method drift detection, drift type taxonomy, latency/throughput metrics, slice-based monitoring, window size specification
- data-quality.md: added training-serving skew detection, statistical baselines for soft checks
- model-serving.md: added shadow deployments, prediction service SLA, model optimization section
- testing.md: added infrastructure tests, fallback system tests, data distribution assumption tests
- README.md: added design principles, system SLA summary, continual learning loop

### Palantir (Palantir Expert)
- foundry-platform-reference.md: added model framework, monitoring/observability, ML lifecycle mapping
- streaming-ingestion.md: added ML pipeline considerations (versioning, point-in-time, schema evolution)
- transform-patterns.md: added ML-specific patterns (point-in-time joins, train/serve skew, backfill, expectations)
- ontology-design.md: added online-offline consistency, feature versioning, drift detection, model version comparison
- model-integration.md: added shadow deployment, production monitoring, automated retraining triggers, rollback
- README.md: updated reading order and descriptions to include monitoring/retraining lifecycle