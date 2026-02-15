# Palantir Foundry — Refrigeration Fleet ML Platform

How we use Foundry as the end-to-end platform for ingesting, processing, and acting on data from 100K+ refrigeration devices.

## Why Foundry

The fleet produces ~14M sensor readings per minute across 14 sensor parameters per device. We need a platform that handles streaming ingestion, batch feature engineering, model scoring, and operational workflows without stitching together a dozen standalone services. Foundry provides all of these as integrated primitives — streaming datasets, Transforms, Ontology, and model objectives — which eliminates the integration tax of a multi-tool stack.

## Documents

| Document | What it covers |
|---|---|
| [Foundry Platform Reference](./foundry-platform-reference.md) | General Foundry concepts and capabilities — start here if you're new to Foundry |
| [Streaming Ingestion](./streaming-ingestion.md) | How 14M readings/min flow from Azure IoT Hub through Event Hubs into Foundry streaming datasets |
| [Transform Patterns](./transform-patterns.md) | `@transform_df` vs `@transform`, incremental transforms, windowed aggregations for IoT time-series |
| [Ontology Design](./ontology-design.md) | Object types, properties, links, and actions for the refrigeration fleet domain model |
| [Model Integration](./model-integration.md) | Training, publishing, and scoring unsupervised anomaly detection models within Foundry |

## Reading Order

1. **[Foundry Platform Reference](./foundry-platform-reference.md)** — understand the building blocks
2. **[Streaming Ingestion](./streaming-ingestion.md)** — how data enters the platform
3. **[Transform Patterns](./transform-patterns.md)** — how data is processed and features are computed
4. **[Ontology Design](./ontology-design.md)** — how processed data becomes actionable domain objects
5. **[Model Integration](./model-integration.md)** — how models score devices and produce alerts

## Relationship to Other Sections

- **[Ingestion Architecture](../01-ingestion/architecture.md)** covers the Azure IoT Hub → Event Hubs path *before* data reaches Foundry. The [Streaming Ingestion](./streaming-ingestion.md) doc picks up where that path enters Foundry.
- **[Feature Engineering](../02-feature-engineering/)** documents the PySpark feature logic. [Transform Patterns](./transform-patterns.md) explains how that logic runs *inside* Foundry Transforms.
- **[Data Quality & Monitoring](../03-production/data-quality.md)** describes quality checks. In Foundry, those checks are implemented as dataset expectations — see [Transform Patterns](./transform-patterns.md).
