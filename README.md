# IoT ML Engineering — Predictive Maintenance on Palantir Foundry

Practical documentation for building production ML pipelines on IoT sensor data. Covers the full path from edge devices through feature engineering to unsupervised anomaly detection models running in Foundry.

Written from experience building predictive maintenance systems for 100K+ refrigeration devices.

## Contents

### 01 — Data Ingestion
- [IoT Ingestion Architecture](./01-ingestion/architecture.md) — Azure IoT Hub → Event Hubs → Foundry streaming dataset
- [Streaming vs Batch Tradeoffs](./01-ingestion/streaming-vs-batch.md) — when to use each, hybrid patterns
- [Schema Management & Evolution](./01-ingestion/schema-management.md) — handling sensor schema changes without breaking pipelines

### 02 — Feature Engineering (Foundry Transforms / PySpark)
- [Time-Domain Features](./02-feature-engineering/time-domain.md) — rolling statistics, lag features, rate of change
- [Frequency-Domain Features](./02-feature-engineering/frequency-domain.md) — FFT, spectral entropy, dominant frequencies
- [Cross-Sensor Features](./02-feature-engineering/cross-sensor.md) — correlations, ratios, fleet-level aggregations
- [Windowing Strategies](./02-feature-engineering/windowing.md) — tumbling, sliding, session windows on sensor streams
- [Feature Store Patterns](./02-feature-engineering/feature-store.md) — offline/online serving, backfill, point-in-time correctness

### 03 — Production
- [Data Quality & Monitoring](./03-production/data-quality.md) — sensor dropout, outlier detection, drift, dataset expectations
- [Pipeline Testing](./03-production/testing.md) — unit testing Transforms, integration tests with synthetic data
- [Model Serving](./03-production/model-serving.md) — batch scoring, streaming anomalies, multi-model parallel scoring
- [Monitoring](./03-production/monitoring.md) — infrastructure, data drift, model degradation, retraining triggers

### 04 — Palantir Foundry
- [Platform Reference](./04-palantir/foundry-platform-reference.md) — Foundry architecture overview
- [Streaming Ingestion](./04-palantir/streaming-ingestion.md) — Event Hubs connector, materialization, backpressure
- [Transform Patterns](./04-palantir/transform-patterns.md) — @transform_df, incremental, windowed aggregations
- [Ontology Design](./04-palantir/ontology-design.md) — Device, Alert, AnomalyScore object types
- [Model Integration](./04-palantir/model-integration.md) — training, publishing, batch scoring in Foundry

### 05 — Architecture
- [System Overview](./05-architecture/system-overview.md) — end-to-end data flow with Mermaid diagrams
- [Data Contracts](./05-architecture/data-contracts.md) — schemas, SLAs, and freshness guarantees between pipeline stages
- [ADR-001: Foundry-Native Processing](./05-architecture/adr-001-foundry-native.md)
- [ADR-002: Batch + Streaming Hybrid](./05-architecture/adr-002-batch-plus-streaming.md)
- [ADR-003: Anomaly Detection First](./05-architecture/adr-003-anomaly-detection-first.md)

### 06 — Modeling
- [Model Selection](./06-modeling/model-selection.md) — multi-approach comparison (statistical, Isolation Forest, DBSCAN, Autoencoder)
- [Training Pipeline](./06-modeling/training-pipeline.md) — temporal splits, device holdout, training without labels
- [Evaluation Framework](./06-modeling/evaluation-framework.md) — proxy metrics, baseline comparison, lead-time analysis
- [Experiment Tracking](./06-modeling/experiment-tracking.md) — Foundry-native experiment management
- [Model Cards](./06-modeling/model-cards.md) — template and examples for Isolation Forest and Autoencoder

## Tech Stack

- **Ingestion**: Azure IoT Hub → Event Hubs → Foundry streaming dataset
- **Processing**: Foundry Transforms (PySpark on Spark)
- **Orchestration**: Palantir Foundry
- **Feature Store**: Offline on Delta Lake (Foundry datasets), online via Ontology pre-hydrated properties
- **Model Serving**: Batch scoring via Transforms + streaming threshold rules
- **Monitoring**: Foundry dataset expectations, data health, pipeline SLAs

## About

By [Mathias Villerabel](https://mreleden.github.io) — AI Engineer at Equans, previously Swiss Re and Synspective.
