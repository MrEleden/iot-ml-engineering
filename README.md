# IoT ML Engineering — From Ingestion to Feature Store

Practical documentation for building production ML pipelines on IoT sensor data. Covers the full path from edge devices to feature-engineered datasets ready for model training.

Written from experience building predictive maintenance systems for industrial equipment.

## Contents

### Data Ingestion
- [IoT Ingestion Architecture](./01-ingestion/architecture.md) — Azure IoT Hub → Kafka → data lake design
- [Streaming vs Batch Tradeoffs](./01-ingestion/streaming-vs-batch.md) — when to use each, hybrid patterns
- [Schema Management & Evolution](./01-ingestion/schema-management.md) — handling sensor schema changes without breaking pipelines

### Feature Engineering in PySpark
- [Time-Domain Features](./02-feature-engineering/time-domain.md) — rolling statistics, lag features, rate of change
- [Frequency-Domain Features](./02-feature-engineering/frequency-domain.md) — FFT, spectral entropy, dominant frequencies
- [Cross-Sensor Features](./02-feature-engineering/cross-sensor.md) — correlations, ratios, fleet-level aggregations
- [Windowing Strategies](./02-feature-engineering/windowing.md) — tumbling, sliding, session windows on sensor streams
- [Feature Store Patterns](./02-feature-engineering/feature-store.md) — offline/online serving, backfill, point-in-time correctness

### Production Considerations
- [Data Quality & Monitoring](./03-production/data-quality.md) — sensor dropout, outlier detection, drift
- [Pipeline Testing](./03-production/testing.md) — unit testing PySpark transforms, integration tests with synthetic data

## Tech Stack

- **Ingestion**: Azure IoT Hub, Kafka / Event Hubs
- **Processing**: PySpark, Polars
- **Orchestration**: Palantir Foundry (or Airflow/Dagster equivalents)
- **Storage**: Delta Lake / Parquet

## About

By [Mathias Villerabel](https://mreleden.github.io) — AI Engineer at Equans, previously Swiss Re and Synspective.
