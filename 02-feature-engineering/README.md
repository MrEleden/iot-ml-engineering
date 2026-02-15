# Feature Engineering

Transforms clean sensor readings into ML-ready features for refrigeration anomaly detection. This section covers what features to compute, why they matter for catching equipment failures, and how to implement them as Foundry Transforms.

## Pipeline Position

```
clean_sensor_readings (Contract 2)  →  Feature Transforms  →  device_features (Contract 3)
```

- **Input**: [`clean_sensor_readings`](../05-architecture/data-contracts.md) — deduplicated, range-validated, UTC-normalized sensor data from 100K+ refrigeration units.
- **Output**: [`device_features`](../05-architecture/data-contracts.md) — time-windowed aggregations, one row per device per window. ~50 feature columns consumed by anomaly detection models.

All transforms run as `@transform_df` with incremental processing where possible. See [Transform Patterns](../04-palantir/transform-patterns.md) for Foundry-specific idioms.

## Why These Features

Refrigeration anomaly detection is unsupervised — there are no labeled "failure" examples to train on. The models (Isolation Forest, Autoencoder, statistical methods) learn what "normal" looks like and flag deviations. This means feature quality directly determines detection quality. A noisy or irrelevant feature adds dimensions without adding signal, making anomaly detection harder.

Every feature in this section exists because it captures a specific physical behavior of a refrigeration system:

- **Temperature trends** reveal whether the unit is losing cooling capacity.
- **Pressure ratios** expose compressor health and refrigerant charge levels.
- **Current and vibration patterns** indicate mechanical degradation before catastrophic failure.
- **Cross-sensor relationships** catch problems that no single sensor shows in isolation — a compressor can draw normal current while producing abnormal temperatures, but the ratio reveals inefficiency.

## Section Contents

| Document | What It Covers |
|----------|----------------|
| [Time-Domain Features](./time-domain.md) | Rolling statistics (mean, std, min, max, slope, p95) per sensor per window. Rate of change and lag features. |
| [Frequency-Domain Features](./frequency-domain.md) | FFT-based features for vibration and current. Honest assessment of what's computable at 1 reading/min. |
| [Cross-Sensor Features](./cross-sensor.md) | Derived features from sensor combinations — pressure ratio, temperature spreads, efficiency metrics, fleet comparisons. Includes refrigeration cycle primer. |
| [Windowing Strategies](./windowing.md) | Time-window design: 15-min, 1-hour, 1-day. Tumbling vs sliding, late arrivals, window completeness, multi-resolution detection. |
| [Feature Store](./feature-store.md) | Offline store on Delta Lake, online serving via Ontology pre-hydration. Point-in-time correctness, backfill, versioning, freshness monitoring. |

## Reading Order

Start with [Windowing Strategies](./windowing.md) to understand how raw readings become window-level rows. Then read [Time-Domain](./time-domain.md) and [Cross-Sensor](./cross-sensor.md) for the core features. [Frequency-Domain](./frequency-domain.md) covers a specialized (and limited) subset. Finish with [Feature Store](./feature-store.md) for how features are stored and served.

## Output Schema Reference

All feature names in these docs align with the `device_features` schema defined in [Contract 3](../05-architecture/data-contracts.md). The naming convention is:

```
{sensor_short}_{aggregation}
```

Where `sensor_short` is one of: `temp_evaporator`, `temp_condenser`, `temp_ambient`, `temp_discharge`, `temp_suction`, `pressure_high`, `pressure_low`, `current_compressor`, `vibration_compressor`, `humidity`, `door_open`, `defrost`, `superheat`, `subcooling`. And `aggregation` is one of: `mean`, `std`, `min`, `max`, `slope`, `p95`, `fraction`.

Cross-sensor features use descriptive names: `pressure_ratio`, `temp_spread_evap_cond`, `temp_spread_discharge_suction`, `current_per_temp_spread`.

## Related Documents

- [Data Contracts](../05-architecture/data-contracts.md) — schema definitions for input (Contract 2) and output (Contract 3)
- [Transform Patterns](../04-palantir/transform-patterns.md) — `@transform_df`, incremental processing, windowed aggregation patterns
- [Ontology Design](../04-palantir/ontology-design.md) — online feature serving via pre-hydrated properties
- [System Overview](../05-architecture/system-overview.md) — where feature engineering sits in the overall architecture
