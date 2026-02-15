# Cross-Sensor Features

Features derived from relationships between multiple sensors — ratios, spreads, and efficiency metrics that capture system-level behavior no single sensor can reveal.

## Why Cross-Sensor Features

A compressor drawing 15 amps is not inherently anomalous — it depends on the thermal load. A discharge temperature of 90°C is alarming in mild ambient conditions but expected in a heat wave. Cross-sensor features normalize sensor readings against each other and against operating context, turning absolute measurements into efficiency and health indicators.

These features are the highest-value inputs for unsupervised anomaly detection because they encode the physics of the refrigeration cycle. An Isolation Forest model trained only on individual sensor statistics will flag extreme values but miss subtle inefficiencies. Add `pressure_ratio`, `current_per_temp_spread`, and `temp_spread_evap_cond`, and the model can detect a compressor working harder for less cooling — a failure mode that appears nowhere in any single sensor.

## The Refrigeration Cycle — A Primer

Understanding why these features work requires understanding what the sensors are measuring in context.

A vapor-compression refrigeration system has four stages:

```
                    ┌──────────────────┐
                    │   CONDENSER      │ ← Heat rejected to environment
     High pressure  │  (hot side)      │   temp_condenser, pressure_high_side
     hot gas ──────►│                  │──────► High pressure
                    └──────────────────┘        subcooled liquid
                            │
                            │  Expansion valve
                            │  (pressure drop)
                            ▼
                    ┌──────────────────┐
     Low pressure   │   EVAPORATOR     │ ← Heat absorbed from cooled space
     cold liquid ──►│  (cold side)     │   temp_evaporator, pressure_low_side
                    │                  │──────► Low pressure
                    └──────────────────┘        superheated vapor
                            │
                            │
                            ▼
                    ┌──────────────────┐
                    │   COMPRESSOR     │ ← Does the mechanical work
                    │                  │   current_compressor, vibration_compressor
                    │  suction ──► discharge │
                    │  (low P)    (high P)   │   temp_suction, temp_discharge
                    └──────────────────┘
```

**Key relationships the sensors capture:**

1. **Pressure ratio** (`pressure_high_side / pressure_low_side`): determined by the temperature difference between the cooled space and the environment. A rising pressure ratio with stable temperatures means the compressor is working harder to maintain the same cooling — early sign of degradation.

2. **Superheat** (suction temperature minus evaporator saturation temperature): controls how much the refrigerant is heated above its boiling point before entering the compressor. Too low = liquid refrigerant enters the compressor (liquid slugging, can destroy the compressor). Too high = poor efficiency, compressor runs hot.

3. **Subcooling** (condenser saturation temperature minus liquid line temperature): indicates refrigerant charge level. Low subcooling = refrigerant undercharge (possible leak). High subcooling = overcharge.

4. **Temperature spreads**: the difference between condenser and evaporator temperatures reflects the total thermal lift the system must provide. The difference between discharge and suction temperatures reflects how much the compressor is heating the gas — a proxy for compressor work and efficiency.

## Features in the Contract 3 Schema

The following cross-sensor features are defined in the [`device_features` schema](../05-architecture/data-contracts.md) (Contract 3).

### `pressure_ratio`

**Definition**: `pressure_high_mean / pressure_low_mean`

**Unit**: dimensionless ratio (typically 3–8 for commercial refrigeration)

**Why it matters**: pressure ratio is the single best indicator of compressor efficiency. In a healthy system, pressure ratio stays within a narrow band determined by the setpoint temperature and ambient conditions. Changes indicate:

| Pressure Ratio Change | Possible Cause | Urgency |
|----------------------|----------------|---------|
| Increasing gradually over weeks | Compressor wear, valve leakage | Monitor — schedule maintenance |
| Sudden increase | Condenser airflow blockage, fan failure | Investigate within hours |
| Decreasing | Refrigerant leak (low-side pressure dropping), expansion valve stuck open | Investigate immediately |
| Unstable (high std) | Expansion valve hunting, control system oscillation | Diagnose root cause |

**Null behavior**: null if either `pressure_high_mean` or `pressure_low_mean` is null. Also null if `pressure_low_mean` is zero (divide-by-zero protection), though this should not occur with valid range-checked data from [Contract 2](../05-architecture/data-contracts.md) (range: 50–800 kPa).

```python
F.when(
    F.col("pressure_low_mean") > 0,
    F.col("pressure_high_mean") / F.col("pressure_low_mean")
).otherwise(F.lit(None)).alias("pressure_ratio")
```

### `temp_spread_evap_cond`

**Definition**: `temp_condenser_mean - temp_evaporator_mean`

**Unit**: °C (typically 40–70°C for commercial refrigeration)

**Why it matters**: this spread represents the total temperature lift the system provides. A widening spread with stable ambient temperature means the system is working harder — possibly due to dirty coils, low airflow, or refrigerant issues. A narrowing spread could mean the evaporator is warming up (losing cooling) or the condenser is losing heat rejection capacity.

**Relationship to pressure ratio**: these features are correlated — temperature spread and pressure ratio both reflect the thermodynamic operating point. Including both provides redundancy (if one sensor drifts, the other catches it) and captures non-linear relationships that a simple ratio misses.

```python
(F.col("temp_condenser_mean") - F.col("temp_evaporator_mean")).alias("temp_spread_evap_cond")
```

### `temp_spread_discharge_suction`

**Definition**: `temp_discharge_mean - temp_suction_mean`

**Unit**: °C (typically 50–100°C)

**Why it matters**: the discharge-suction spread reflects compressor work — how much the compressor heats the refrigerant gas during compression. This spread is sensitive to:

- **Compressor efficiency**: a worn compressor with leaking valves produces a lower discharge temperature for the same suction conditions (less effective compression)
- **Superheat changes**: elevated suction temperature (high superheat) increases the spread
- **Liquid slugging**: if liquid refrigerant enters the compressor (low superheat), discharge temperature drops abnormally because the compressor is compressing liquid instead of gas — dangerous condition

This feature complements `temp_spread_evap_cond` by focusing on the compressor specifically rather than the overall system.

```python
(F.col("temp_discharge_mean") - F.col("temp_suction_mean")).alias("temp_spread_discharge_suction")
```

### `current_per_temp_spread`

**Definition**: `current_compressor_mean / temp_spread_evap_cond`

**Unit**: A/°C (amps per degree of temperature lift)

**Why it matters**: this is a compressor efficiency proxy — how much electrical energy the compressor consumes per degree of cooling it delivers. In a healthy system, this ratio is relatively stable. Rising `current_per_temp_spread` means the compressor is consuming more energy for the same cooling work. Causes include:

- Refrigerant undercharge (compressor works harder to maintain pressure)
- Mechanical wear (higher friction → higher current for same output)
- Dirty condenser (higher discharge pressure → higher current)
- Electrical degradation (winding resistance increasing)

This feature is among the most valuable for early degradation detection because it normalizes current draw against the actual cooling load. A hot summer day increases both current and temperature spread proportionally, so the ratio stays stable. But mechanical degradation increases current without increasing cooling, so the ratio rises.

**Null behavior**: null if `current_compressor_mean` is null or if `temp_spread_evap_cond` is null or zero.

```python
F.when(
    (F.col("temp_spread_evap_cond").isNotNull()) & (F.col("temp_spread_evap_cond") != 0),
    F.col("current_compressor_mean") / F.col("temp_spread_evap_cond")
).otherwise(F.lit(None)).alias("current_per_temp_spread")
```

### `superheat_mean` and `superheat_std`

**Definition**: mean and standard deviation of `superheat` readings in the window.

The `superheat` value comes directly from the clean sensor readings — it's measured by the device (or computed on-device from suction temperature and pressure). The feature engineering layer aggregates it per window.

**Why it matters**:

- **Low superheat (< 3K)**: risk of liquid slugging. Liquid refrigerant entering the compressor can cause immediate mechanical damage. This is the most urgent refrigeration failure mode.
- **High superheat (> 15K)**: poor efficiency — the refrigerant is overly heated before compression, wasting compressor energy. Also indicates possible refrigerant undercharge.
- **Unstable superheat (high std)**: expansion valve hunting — the valve is oscillating between too open and too closed, causing refrigerant flow instability.

### `subcooling_mean` and `subcooling_std`

**Definition**: mean and standard deviation of `subcooling` readings in the window.

**Why it matters**:

- **Low subcooling (< 3K)**: probable refrigerant undercharge. The system doesn't have enough liquid refrigerant to fully subcool at the condenser. Often the first detectable sign of a slow refrigerant leak.
- **High subcooling (> 15K)**: refrigerant overcharge or condenser subcooling section restriction.
- **Declining subcooling trend over weeks/months**: the most reliable indicator of a slow refrigerant leak. Cross-reference with `pressure_low_min` trending downward.

## Fleet-Level Features (Design Consideration)

The [Contract 3 schema](../05-architecture/data-contracts.md) defines per-device features. Fleet-level comparisons—how a device compares to its peers—are not in the current schema but are discussed here as a design consideration for downstream model-input transforms.

### Device vs Fleet Median

For any feature (e.g., `pressure_ratio`), compute the fleet-wide median for the same window and same equipment model. The difference between a device's value and the fleet median is a normalized anomaly signal. A device with a pressure ratio of 6.0 is not inherently anomalous — but if the fleet median for that model is 4.5, the device is an outlier.

### Z-Score Relative to Cohort

Group devices by equipment `model` (from the device registry in the [Ontology](../04-palantir/ontology-design.md)), compute mean and std of each feature across the cohort, then express each device's features as z-scores. A z-score > 3 means the device is more than 3 standard deviations from its cohort — a strong anomaly signal even when absolute values are within valid ranges.

### Why Not Store Fleet Features in Contract 3

Fleet-level features require reading data across all devices, which makes them expensive to compute and impossible to compute incrementally per device. They also change every time any device in the cohort changes, which would require re-computing stored features for all devices whenever one device reports new data.

Better approach: compute fleet statistics as a separate aggregation (one row per model per window), then join at model-input time. This keeps `device_features` per-device and independently computable.

```python
# Fleet statistics: separate dataset, not in device_features
fleet_stats = (
    device_features
    .join(device_registry, "device_id")  # Get equipment model
    .groupBy("model", "window_start", "window_duration_minutes")
    .agg(
        F.percentile_approx("pressure_ratio", 0.5).alias("fleet_pressure_ratio_median"),
        F.avg("pressure_ratio").alias("fleet_pressure_ratio_mean"),
        F.stddev("pressure_ratio").alias("fleet_pressure_ratio_std"),
        # ... per feature
    )
)

# Z-score: computed at model-input time, not stored in device_features
model_input = (
    device_features
    .join(fleet_stats, ["model", "window_start", "window_duration_minutes"])
    .withColumn(
        "pressure_ratio_zscore",
        (F.col("pressure_ratio") - F.col("fleet_pressure_ratio_mean"))
        / F.col("fleet_pressure_ratio_std")
    )
)
```

## Computation Order and Dependencies

Cross-sensor features depend on time-domain aggregations. The computation order within a window:

1. **Time-domain aggregations** (mean, std, min, max per sensor) — see [Time-Domain](./time-domain.md)
2. **Cross-sensor features** (ratios, spreads) — computed from the aggregated means
3. **Data quality features** (sensor_completeness, quality_flag_count) — computed alongside step 1

In a Foundry Transform, steps 1 and 2 can be combined in a single `@transform_df` if the cross-sensor features are computed as derived columns after the groupBy. Alternatively, step 2 can be a separate downstream Transform that reads the output of step 1 — this adds a Transform dependency but keeps each Transform simple and independently testable.

Recommendation: combine steps 1 and 2 in a single Transform to avoid the overhead of an additional dataset write. The cross-sensor computations are trivial column arithmetic, not aggregations, so they don't add meaningful complexity.

```python
# Combined approach: time-domain aggs + cross-sensor features in one pass
features = (
    windowed_readings
    .groupBy("device_id", "window_start", "window_end", "window_duration_minutes")
    .agg(
        # Time-domain aggregations
        F.avg("temperature_evaporator").alias("temp_evaporator_mean"),
        F.avg("temperature_condenser").alias("temp_condenser_mean"),
        F.avg("pressure_high_side").alias("pressure_high_mean"),
        F.avg("pressure_low_side").alias("pressure_low_mean"),
        F.avg("current_compressor").alias("current_compressor_mean"),
        # ... all other aggs
    )
    # Cross-sensor features as derived columns
    .withColumn("pressure_ratio",
        F.when(F.col("pressure_low_mean") > 0,
               F.col("pressure_high_mean") / F.col("pressure_low_mean"))
    )
    .withColumn("temp_spread_evap_cond",
        F.col("temp_condenser_mean") - F.col("temp_evaporator_mean")
    )
    .withColumn("temp_spread_discharge_suction",
        F.col("temp_discharge_mean") - F.col("temp_suction_mean")
    )
    .withColumn("current_per_temp_spread",
        F.when(
            (F.col("temp_spread_evap_cond").isNotNull()) &
            (F.col("temp_spread_evap_cond") != 0),
            F.col("current_compressor_mean") / F.col("temp_spread_evap_cond")
        )
    )
)
```

## Failure Modes

### Division by Zero

`pressure_ratio` divides by `pressure_low_mean`; `current_per_temp_spread` divides by `temp_spread_evap_cond`. Both denominators can be zero or null:

- `pressure_low_mean = 0`: should not happen after range validation (Contract 2 enforces 50–800 kPa), but guard against it.
- `temp_spread_evap_cond = 0`: possible if condenser and evaporator temperatures are equal (system off, or sensor error). Result should be null, not infinity.

Always wrap divisions in `F.when(denominator > 0, ...)`.

### Correlated Sensor Failures

If both pressure sensors fail simultaneously, `pressure_ratio` is null — which correctly signals data unavailability. But if only one pressure sensor fails, the ratio becomes meaningless (one real value divided by a fallback or null). The `sensor_completeness` feature provides context: if completeness is low, downstream models should weight ratio features less. This is a model-level concern, not a feature-engineering fix.

### Physically Impossible Ratios

A `pressure_ratio` below 1.0 means low-side pressure exceeds high-side pressure — physically impossible during normal operation. This indicates a sensor error, calibration drift, or the compressor is off. Rather than clipping or nulling these values, pass them through — they are legitimate anomaly signals. An anomaly model should learn that `pressure_ratio < 1.0` is abnormal.

### Ambient Temperature Confounding

`temp_spread_evap_cond` varies with ambient temperature — a 35°C day naturally produces a wider spread than a 15°C day. This is not a flaw; it's expected physics. The anomaly model should capture regional and seasonal norms. Fleet-relative z-scores (comparing a device against its cohort in the same region) further normalize for ambient conditions.

## Related Documents

- [Time-Domain Features](./time-domain.md) — individual sensor aggregations that these cross-sensor features build on
- [Frequency-Domain Features](./frequency-domain.md) — spectral features for compressor cycling detection
- [Windowing Strategies](./windowing.md) — window definitions and alignment
- [Feature Store](./feature-store.md) — how cross-sensor features are stored alongside time-domain features
- [Data Contracts — Contract 3](../05-architecture/data-contracts.md) — authoritative schema for all output feature columns
- [Ontology Design](../04-palantir/ontology-design.md) — device registry for fleet grouping by equipment model
