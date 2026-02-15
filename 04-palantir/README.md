# Palantir Foundry — Refrigeration Fleet ML Platform

How we use Foundry as the end-to-end platform for ingesting, processing, and acting on data from 100K+ refrigeration devices.

## Why Foundry

The fleet produces ~14M sensor readings per minute across 14 sensor parameters per device. We need a platform that handles streaming ingestion, batch feature engineering, model scoring, and operational workflows without stitching together a dozen standalone services. Foundry provides all of these as integrated primitives — streaming datasets, Transforms, Ontology, and model objectives — which eliminates the integration tax of a multi-tool stack.

## Documents

| Document | What it covers |
|---|---|
| [Foundry Platform Reference](./foundry-platform-reference.md) | General Foundry concepts and capabilities, model framework, monitoring primitives, and ML lifecycle mapping — start here if you're new to Foundry |
| [Streaming Ingestion](./streaming-ingestion.md) | How 14M readings/min flow from Azure IoT Hub through Event Hubs into Foundry streaming datasets, plus ML-specific considerations for training data versioning, point-in-time correctness, and schema evolution |
| [Transform Patterns](./transform-patterns.md) | `@transform_df` vs `@transform`, incremental transforms, windowed aggregations for IoT time-series, plus ML-specific patterns for point-in-time joins, training-serving skew prevention, and feature validation |
| [Ontology Design](./ontology-design.md) | Object types, properties, links, actions, online-offline feature consistency, feature versioning, and drift detection for the refrigeration fleet domain model |
| [Model Integration](./model-integration.md) | Training, publishing, scoring, shadow deployment, production monitoring, automated retraining triggers, and rollback procedures for unsupervised anomaly detection models within Foundry |

## Reading Order

1. **[Foundry Platform Reference](./foundry-platform-reference.md)** — understand the building blocks (including model framework and monitoring primitives)
2. **[Streaming Ingestion](./streaming-ingestion.md)** — how data enters the platform (including ML-specific data versioning and point-in-time correctness)
3. **[Transform Patterns](./transform-patterns.md)** — how data is processed and features are computed (including training-serving skew prevention)
4. **[Ontology Design](./ontology-design.md)** — how processed data becomes actionable domain objects (including online-offline consistency and drift detection)
5. **[Model Integration](./model-integration.md)** — how models score devices and produce alerts (including shadow deployment and promotion gates)
6. **Continue the lifecycle**: models in production require continuous monitoring → operator feedback (false positive labels) → automated retraining triggers → shadow deployment of challenger models → promotion or rollback. These post-deployment stages are covered in Model Integration and Ontology Design.

## Relationship to Other Sections

- **[Ingestion Architecture](../01-ingestion/architecture.md)** covers the Azure IoT Hub → Event Hubs path *before* data reaches Foundry. The [Streaming Ingestion](./streaming-ingestion.md) doc picks up where that path enters Foundry.
- **[Feature Engineering](../02-feature-engineering/)** documents the PySpark feature logic. [Transform Patterns](./transform-patterns.md) explains how that logic runs *inside* Foundry Transforms.
- **[Data Quality & Monitoring](../03-production/data-quality.md)** describes quality checks. In Foundry, those checks are implemented as dataset expectations — see [Transform Patterns](./transform-patterns.md).
