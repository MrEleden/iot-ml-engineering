# Architecture

System design decisions and data contracts for the IoT predictive maintenance platform. These docs explain **why** the system is shaped this way — read them before writing any pipeline code.

## Documents

| Document | Purpose |
|----------|---------|
| [System Overview](./system-overview.md) | End-to-end data flow, component responsibilities, latency budgets |
| [Data Contracts](./data-contracts.md) | Schemas, column types, and SLAs at every pipeline boundary |
| [ADR-001: Foundry-Native Processing](./adr-001-foundry-native.md) | Why all processing runs inside Foundry instead of standalone Spark |
| [ADR-002: Batch + Streaming Scoring](./adr-002-batch-plus-streaming.md) | Why we use hybrid scoring instead of pure batch or pure streaming |
| [ADR-003: Anomaly Detection First](./adr-003-anomaly-detection-first.md) | Why we start with unsupervised anomaly detection instead of supervised failure prediction |

## Reading Order

1. **System Overview** — understand the full pipeline before diving into details
2. **Data Contracts** — know exactly what crosses each boundary
3. **ADRs** — understand the reasoning behind key tradeoffs

## Related Sections

- [Ingestion Architecture](../01-ingestion/architecture.md) — details on IoT Hub → Event Hubs → Foundry
- [Feature Engineering](../02-feature-engineering/) — transform logic that consumes the schemas defined here
- [Production Considerations](../03-production/) — monitoring and testing for the architecture described here
- [Foundry Platform Reference](../04-palantir/foundry-platform-reference.md) — Foundry primitives referenced throughout
- [Modeling](../06-modeling/) — ML approaches that consume the feature store outputs defined here

## Full ML Lifecycle

The architecture documents above describe the data pipeline and scoring infrastructure, but a production ML system extends beyond inference. The following documents — maintained alongside the architecture — define the full model lifecycle:

| Document | Lifecycle Stage | Connection to Architecture |
|----------|----------------|---------------------------|
| [Experiment Tracking](../06-modeling/experiment-tracking.md) | Development | Defines how model experiments are logged, compared, and reproduced within Foundry Code Workspaces. Outputs feed the model registry referenced in [System Overview](./system-overview.md). |
| [Training Pipeline](../06-modeling/training-pipeline.md) | Training & Retraining | Implements the Training & Retraining subgraph from [System Overview](./system-overview.md). Consumes training data assembled per [Contract 3.5](./data-contracts.md) with point-in-time join semantics. |
| [Model Selection](../06-modeling/model-selection.md) | Evaluation | Governs which model families and granularities are promoted to production, as decided in [ADR-003](./adr-003-anomaly-detection-first.md). |
| [Evaluation Framework](../06-modeling/evaluation-framework.md) | Validation | Defines metrics and hold-out protocols used before a model enters the shadow scoring phase described in [ADR-002](./adr-002-batch-plus-streaming.md). |
| [Model Cards](../06-modeling/model-cards.md) | Documentation | Standardized documentation for every production model, including the feature list, training data snapshot, and performance characteristics. |
| [Monitoring](../03-production/monitoring.md) | Production | Tracks model health (drift, score distribution, alert volume) and pipeline health (latency, throughput). Triggers retraining when drift thresholds are exceeded — see [ADR-003](./adr-003-anomaly-detection-first.md) drift detection. |
| [Testing](../03-production/testing.md) | Validation & CI | Point-in-time correctness tests, feature consistency checks, and scoring determinism tests that enforce the [Data Contracts](./data-contracts.md). |

These documents are part of the architecture — they define the operational boundaries and quality gates that keep the system reliable as models evolve.
