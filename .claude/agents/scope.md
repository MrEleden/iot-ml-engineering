# Project Scope

## Fleet
- **100K+ devices** (refrigeration systems)
- **14 sensor parameters per device**: temperature (evaporator, condenser, ambient, discharge, suction), pressure (high-side, low-side), current (compressor), vibration (compressor), humidity, door open/close, defrost cycle, superheat, subcooling
- **Reading frequency**: ~1 reading/minute per sensor → ~14M readings/minute fleet-wide

## Infrastructure
- **Ingestion**: Azure IoT Hub → Event Hubs → Palantir Foundry streaming dataset
- **Processing**: Foundry-native only — no raw Kafka consumers, no standalone Spark clusters
- **Batch transforms**: Foundry Transforms (PySpark on Spark)
- **Feature store**: Offline on Delta Lake (Foundry datasets), online via Ontology pre-hydrated properties
- **Model serving**: Foundry model objectives (pattern still being explored — batch scoring vs real-time Ontology actions)

## Constraints
- All code must run as **Foundry Transforms** (`@transform_df`, `@transform`, incremental where possible)
- No generic PySpark that assumes standalone Spark — always Foundry idioms
- Schema changes go through Foundry dataset expectations
- Monitoring uses Foundry-native tools (data health, pipeline SLAs, dataset expectations)

## Non-Goals (for now)
- Edge computing / on-device ML
- Multi-cloud (Azure-only for IoT layer)
- Custom streaming framework (Foundry handles this)
