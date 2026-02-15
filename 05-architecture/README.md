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
