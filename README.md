# IoT ML Engineering — Predictive Maintenance on Palantir Foundry

Practical documentation for building production ML pipelines on IoT sensor data. Covers the full ML lifecycle — from edge device ingestion through feature engineering, unsupervised anomaly detection, safe deployment, and continual learning — all running on Palantir Foundry.

Written from experience building predictive maintenance systems for 100K+ refrigeration devices (14 sensors each, ~14M readings/min at scale).

## Design References

The system design draws on two foundational texts:

- **Designing Machine Learning Systems** (Chip Huyen, O'Reilly 2022) — ML lifecycle, distribution shift taxonomy, feature engineering principles, safe deployment patterns, continual learning
- **Designing Data-Intensive Applications** (Martin Kleppmann, O'Reilly 2017) — distributed systems reliability, data contracts, schema evolution, exactly-once semantics

## Contents

### 01 — Data Ingestion
- [IoT Ingestion Architecture](./01-ingestion/architecture.md) — Azure IoT Hub → Event Hubs → Foundry streaming dataset, retention tiers (hot/cold), dataset snapshots
- [Streaming vs Batch Tradeoffs](./01-ingestion/streaming-vs-batch.md) — when to use each, hybrid patterns, dual consumer model (inference + training)
- [Schema Management & Evolution](./01-ingestion/schema-management.md) — handling sensor schema changes, leakage prevention fields, schema change tracking

### 02 — Feature Engineering (Foundry Transforms / PySpark)
- [Time-Domain Features](./02-feature-engineering/time-domain.md) — rolling statistics, lag features, rate of change, scaling and leakage prevention
- [Frequency-Domain Features](./02-feature-engineering/frequency-domain.md) — FFT, spectral entropy, dominant frequencies, spectral drift monitoring
- [Cross-Sensor Features](./02-feature-engineering/cross-sensor.md) — correlations, ratios, fleet-level aggregations, covariate shift detection (PSI)
- [Windowing Strategies](./02-feature-engineering/windowing.md) — tumbling, sliding, session windows, temporal train/test split guidance
- [Feature Store Patterns](./02-feature-engineering/feature-store.md) — offline/online serving, backfill, point-in-time correctness, distribution monitoring

### 03 — Production
- [Data Quality & Monitoring](./03-production/data-quality.md) — sensor dropout, outlier detection, drift, dataset expectations, training-serving skew detection
- [Pipeline Testing](./03-production/testing.md) — unit testing Transforms, integration tests, infrastructure tests, fallback system tests
- [Model Serving](./03-production/model-serving.md) — batch scoring, streaming anomalies, multi-model scoring, shadow deployments, SLAs
- [Monitoring](./03-production/monitoring.md) — multi-method drift detection (KS/PSI/KL/MMD), drift taxonomy, retraining triggers, slice-based monitoring

### 04 — Palantir Foundry
- [Platform Reference](./04-palantir/foundry-platform-reference.md) — Foundry architecture overview, model frameworks, monitoring capabilities
- [Streaming Ingestion](./04-palantir/streaming-ingestion.md) — Event Hubs connector, materialization, backpressure, ML pipeline considerations
- [Transform Patterns](./04-palantir/transform-patterns.md) — @transform_df, incremental, windowed aggregations, ML-specific patterns (point-in-time joins, skew prevention)
- [Ontology Design](./04-palantir/ontology-design.md) — Device, Alert, AnomalyScore object types, online-offline consistency, feature versioning
- [Model Integration](./04-palantir/model-integration.md) — training, publishing, batch scoring, shadow deployment, automated retraining triggers

### 05 — Architecture
- [System Overview](./05-architecture/system-overview.md) — end-to-end data flow with Mermaid diagrams, training/retraining subgraph, feedback loops
- [Data Contracts](./05-architecture/data-contracts.md) — schemas, SLAs, freshness guarantees, training data assembly contract, label data contract
- [ADR-001: Foundry-Native Processing](./05-architecture/adr-001-foundry-native.md) — ML lifecycle capabilities evaluation
- [ADR-002: Batch + Streaming Hybrid](./05-architecture/adr-002-batch-plus-streaming.md) — threshold synchronization, model deployment strategy
- [ADR-003: Anomaly Detection First](./05-architecture/adr-003-anomaly-detection-first.md) — drift detection, retraining triggers, Phase 2 transition criteria

### 06 — Modeling
- [Model Selection](./06-modeling/model-selection.md) — Isolation Forest, Autoencoder, statistical baselines, SHAP interpretability, Phase 2 class imbalance handling
- [Training Pipeline](./06-modeling/training-pipeline.md) — temporal splits, device holdout, stateless vs stateful retraining, sampling strategy, data augmentation
- [Evaluation Framework](./06-modeling/evaluation-framework.md) — proxy metrics, baseline comparison, statistical drift detection, shadow → canary → full rollout
- [Experiment Tracking](./06-modeling/experiment-tracking.md) — Foundry-native experiment management, training data statistics, loss curves
- [Model Cards](./06-modeling/model-cards.md) — template and examples for Isolation Forest and Autoencoder, staleness indicators, fairness section

## Cross-Cutting Concerns

These topics are addressed across multiple documents rather than in a single place:

| Concern | Key Documents |
|---------|--------------|
| **Distribution shift** | [Monitoring](./03-production/monitoring.md), [Evaluation Framework](./06-modeling/evaluation-framework.md), [Feature Store](./02-feature-engineering/feature-store.md) |
| **Data leakage prevention** | [Time-Domain](./02-feature-engineering/time-domain.md), [Windowing](./02-feature-engineering/windowing.md), [Cross-Sensor](./02-feature-engineering/cross-sensor.md), [Schema Management](./01-ingestion/schema-management.md) |
| **Safe deployment** | [Model Serving](./03-production/model-serving.md), [Evaluation Framework](./06-modeling/evaluation-framework.md), [Model Integration](./04-palantir/model-integration.md) |
| **Continual learning** | [Training Pipeline](./06-modeling/training-pipeline.md), [Monitoring](./03-production/monitoring.md), [ADR-003](./05-architecture/adr-003-anomaly-detection-first.md) |
| **Training-serving consistency** | [Feature Store](./02-feature-engineering/feature-store.md), [Transform Patterns](./04-palantir/transform-patterns.md), [Data Quality](./03-production/data-quality.md) |

## Tech Stack

- **Ingestion**: Azure IoT Hub → Event Hubs → Foundry streaming dataset
- **Processing**: Foundry Transforms (PySpark on Spark)
- **Orchestration**: Palantir Foundry
- **Feature Store**: Offline on Delta Lake (Foundry datasets), online via Ontology pre-hydrated properties
- **Model Serving**: Batch scoring via Transforms + streaming threshold rules
- **Monitoring**: Multi-method drift detection (KS/PSI/KL/MMD), Foundry dataset expectations, pipeline SLAs
- **Deployment**: Shadow → Canary (5%) → A/B (50/50) → Full rollout with auto-rollback

## About

By [Mathias Villerabel](https://mreleden.github.io) — AI Engineer at Equans, previously Swiss Re and Synspective.
