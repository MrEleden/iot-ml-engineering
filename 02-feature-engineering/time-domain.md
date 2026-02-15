# Time-Domain Features

Rolling statistics computed over time windows — the foundation of the feature set. These features capture what a sensor is doing (mean), how stable it is (std), its extremes (min, max), and where it's heading (slope).

## Why Time-Domain Aggregations

A single sensor reading at one instant tells you almost nothing about equipment health. A temperature of -18°C in the evaporator could be normal operation or the beginning of a freeze-up — the context is in the trajectory. Time-domain aggregations compress a window of readings into descriptive statistics that anomaly detection models can compare across devices and across time.

Each aggregation type serves a distinct purpose:

| Aggregation | What It Captures | Why It Matters for Refrigeration |
|-------------|-----------------|----------------------------------|
| `mean` | Central tendency over the window | Baseline operating point — drift in mean signals gradual degradation |
| `std` | Variability within the window | High std → instability (e.g., short-cycling compressor, flapping defrost) |
| `min` | Lowest value in the window | Floor violations (e.g., evaporator freezing below safe range) |
| `max` | Highest value in the window | Ceiling violations (e.g., discharge temperature spike) |
| `slope` | Linear trend direction and rate | Rising evaporator temp slope → losing cooling capacity; slope detects this before the mean shifts enough to flag |
| `p95` | 95th percentile | Robust alternative to max for noisy sensors — captures sustained high values while ignoring outlier spikes |

## Sensor-to-Aggregation Map

Not every sensor needs every aggregation. The map below shows which aggregations are computed for each sensor, aligned with the [`device_features` schema](../05-architecture/data-contracts.md) (Contract 3).

| Sensor | mean | std | min | max | slope | p95 | Rationale |
|--------|:----:|:---:|:---:|:---:|:-----:|:---:|-----------|
| `temperature_evaporator` | ✓ | ✓ | ✓ | ✓ | ✓ | | Primary cooling indicator — full stat profile needed |
| `temperature_condenser` | ✓ | ✓ | ✓ | ✓ | ✓ | | Heat rejection indicator — slope reveals airflow blockage |
| `temperature_ambient` | ✓ | ✓ | | | | | Context sensor — mean/std sufficient, extremes less informative |
| `temperature_discharge` | ✓ | ✓ | | ✓ | | | Max matters — peak discharge temp signals compressor stress |
| `temperature_suction` | ✓ | ✓ | | | | | Paired with discharge for spread feature (see [Cross-Sensor](./cross-sensor.md)) |
| `pressure_high_side` | ✓ | ✓ | | ✓ | | | Max detects pressure spikes — potential blockages |
| `pressure_low_side` | ✓ | ✓ | ✓ | | | | Min detects pressure drops — potential refrigerant leaks |
| `current_compressor` | ✓ | ✓ | | ✓ | | | Max detects current spikes — mechanical binding |
| `vibration_compressor` | ✓ | ✓ | | ✓ | | ✓ | p95 more robust than max for vibration (noisy sensor) |
| `humidity` | ✓ | ✓ | | | | | Environmental context for condensation risk |
| `superheat` | ✓ | ✓ | | | | | Mean tracks refrigerant flow health |
| `subcooling` | ✓ | ✓ | | | | | Mean tracks charge level |

Sensors like `door_open_close` and `defrost_cycle` are boolean and get fraction-based aggregations instead (`door_open_fraction`, `defrost_active_fraction`). These are covered in the output schema but don't use the statistical aggregations above.

### Feature Importance Validation

With 50+ feature columns spanning 12 sensors × multiple aggregations, not all features contribute equally to anomaly detection. Feature importance must be validated before expanding the feature set, and periodically re-validated as the fleet evolves.

**For Isolation Forest**: use permutation importance — randomly shuffle each feature column in the validation set and measure the change in anomaly score distribution. Features whose shuffling does not change the anomaly ranking are candidates for removal. Permutation importance is model-agnostic and avoids the bias of tree-based feature importance metrics.

**For Autoencoders**: measure per-feature reconstruction error contribution. Features with consistently low reconstruction error relative to their variance are not being used by the model and may add noise. Decompose the total reconstruction loss into per-feature components and rank by contribution.

**Dimensionality concern**: the current feature set produces approximately 50 columns. For Isolation Forest, performance degrades as dimensionality grows because random splits become less effective. If feature count exceeds 50, evaluate whether marginal features improve detection or merely add noise. Consider PCA as a dimensionality reduction step, though this trades interpretability for compactness.

Feature importance validation is a joint responsibility between the Feature Engineer and the ML Scientist. The Feature Engineer proposes features based on domain knowledge (this document); the ML Scientist validates them empirically using the [evaluation framework](../06-modeling/evaluation-framework.md). Features that fail importance validation should be excluded from [Contract 4](../05-architecture/data-contracts.md) model input, not removed from [Contract 3](../05-architecture/data-contracts.md) storage — they may prove useful for future model architectures.

## Rolling Statistics

### Mean and Standard Deviation

The most fundamental pair. Mean establishes the operating point; standard deviation measures volatility around it.

**Why std matters for refrigeration**: a compressor that cycles on and off rapidly ("short-cycling") produces high temperature std even when the mean looks normal. Short-cycling is a leading indicator of compressor failure — it stresses the motor and reduces equipment lifespan. Without std, the anomaly model only sees the average and misses this behavior.

```python
# Conceptual PySpark — within a Foundry @transform_df
from pyspark.sql import functions as F

stats = (
    clean_readings
    .groupBy("device_id", "window_start", "window_end", "window_duration_minutes")
    .agg(
        F.avg("temperature_evaporator").alias("temp_evaporator_mean"),
        F.stddev("temperature_evaporator").alias("temp_evaporator_std"),
        F.min("temperature_evaporator").alias("temp_evaporator_min"),
        F.max("temperature_evaporator").alias("temp_evaporator_max"),
        # ... repeat for other sensors per the map above
    )
)
```

**Null behavior**: if all readings in the window are null for a given sensor, the aggregation produces null. If some readings are non-null, aggregations compute over the available values. This matches the [Contract 3 null policy](../05-architecture/data-contracts.md): "Feature is null only when all underlying readings in the window are null for that sensor."

**Edge case — single reading in window**: `stddev` of a single value is null in Spark (undefined for n=1). This is acceptable — a single-reading window already signals sensor dropout, captured by `reading_count`. Downstream models should treat null std as a quality issue, not a zero-variability signal.

### Min and Max

Extremes within a window. Applied selectively — see the sensor map above.

**Why min for `pressure_low_side`**: low-side pressure dropping to the floor of its validated range (50 kPa from [Contract 2](../05-architecture/data-contracts.md)) is a strong signal of refrigerant leak. The mean might look acceptable if the drop is brief, but min captures it.

**Why max for `temp_discharge`**: discharge temperature spikes above 120°C indicate compressor overheating. A single spike in a 15-minute window matters even if the mean stays within range. Peak discharge temp (`temp_discharge_max`) is a direct compressor stress indicator.

**Why max for `current_compressor`**: current spikes during compressor startup are normal, but sustained high current or spikes outside startup windows indicate mechanical binding, bearing wear, or electrical faults.

### Percentile (p95)

Applied to `vibration_compressor` as `vibration_compressor_p95`.

**Why p95 instead of (or in addition to) max**: vibration sensors are noisy. A single spike caused by a door slam, a forklift driving past, or electrical interference produces a misleading max. The 95th percentile captures sustained high-vibration episodes while filtering transient spikes. When p95 is elevated but max is similar to normal, the cause is likely external. When both p95 and max are elevated, the source is likely mechanical.

```python
# p95 in PySpark via percentile_approx
F.percentile_approx("vibration_compressor", 0.95).alias("vibration_compressor_p95")
```

`percentile_approx` is used instead of exact percentile for performance on large windows. The default relative error (1/10000) is more than sufficient for anomaly detection.

## Slope (Linear Trend)

Slope measures the direction and rate of change of a sensor over the window, computed as the coefficient of a least-squares linear fit of sensor value against time.

### Why Slope Matters

Mean is a lagging indicator — by the time mean temperature rises noticeably, the problem has been developing for a while. Slope is a leading indicator: it detects *the rate at which things are changing* before the absolute values leave the normal range.

Example: an evaporator at -20°C with a slope of +0.1°C/min looks fine right now but will be at -11°C within 90 minutes. Without slope, the anomaly model doesn't flag this until the mean actually hits an abnormal range.

### Which Sensors Get Slope

Only `temperature_evaporator` and `temperature_condenser` have slope features in the [Contract 3 schema](../05-architecture/data-contracts.md):

- `temp_evaporator_slope` (°C/min): rising slope → warming trend → losing cooling capacity or defrost cycle
- `temp_condenser_slope` (°C/min): rising slope → condenser heat rejection degrading (dirty coils, fan failure, high ambient load)

Other sensors don't have slope columns in Contract 3. Pressure and current can have trends, but the cross-sensor derived features (like `pressure_ratio` and `current_per_temp_spread`) capture those relationships more effectively.

### Computing Slope in PySpark

Slope requires a linear regression of value against time within each window. This isn't a built-in Spark aggregation, so it's implemented via a Pandas UDF or manual computation using sum-of-products formulas.

```python
# Approach: manual least-squares slope via aggregation
# slope = (n * sum(x*y) - sum(x) * sum(y)) / (n * sum(x^2) - sum(x)^2)
# where x = minutes since window start, y = sensor value

from pyspark.sql import functions as F

# Add minutes-since-window-start as a numeric column
with_minutes = readings.withColumn(
    "minutes_offset",
    (F.col("timestamp_utc").cast("long") - F.col("window_start").cast("long")) / 60.0
)

slope_components = with_minutes.groupBy(
    "device_id", "window_start", "window_end", "window_duration_minutes"
).agg(
    F.count("temperature_evaporator").alias("n"),
    F.sum("minutes_offset").alias("sum_x"),
    F.sum("temperature_evaporator").alias("sum_y"),
    F.sum(F.col("minutes_offset") * F.col("temperature_evaporator")).alias("sum_xy"),
    F.sum(F.col("minutes_offset") ** 2).alias("sum_x2"),
)

slope = slope_components.withColumn(
    "temp_evaporator_slope",
    F.when(
        (F.col("n") * F.col("sum_x2") - F.col("sum_x") ** 2) != 0,
        (F.col("n") * F.col("sum_xy") - F.col("sum_x") * F.col("sum_y"))
        / (F.col("n") * F.col("sum_x2") - F.col("sum_x") ** 2)
    ).otherwise(F.lit(None))  # Undefined slope if all readings at same timestamp
)
```

**Unit**: °C per minute. A slope of +0.1 means the sensor is rising at 0.1°C every minute over the window.

**Null behavior**: slope is null if fewer than 2 non-null readings exist in the window (can't fit a line to one point), or if all readings have the same timestamp (denominator is zero).

## Boolean Fraction Features

For `door_open_close` and `defrost_cycle`, time-domain aggregation means computing the fraction of the window where the condition was true.

### `door_open_fraction`

Fraction of readings in the window where `door_open_close = true` (0.0 to 1.0).

**Why it matters**: frequent or prolonged door openings cause warm air ingress, forcing the compressor to work harder. A consistently high `door_open_fraction` in a 1-hour window suggests operational issues (broken door seal, high-traffic periods). Anomaly models can detect devices where door-open patterns deviate from the fleet norm.

### `defrost_active_fraction`

Fraction of readings where `defrost_cycle = true` (0.0 to 1.0).

**Why it matters**: defrost cycles are expected — ice builds on the evaporator and must be melted periodically. But excessive defrost (high fraction) signals a problem: faulty defrost timer, ice buildup that won't clear, or a defrost sensor failure. Too little defrost (consistently zero) also matters — it may mean the defrost system is broken and ice is accumulating unchecked.

```python
# Boolean fraction in PySpark
F.avg(F.col("door_open_close").cast("double")).alias("door_open_fraction"),
F.avg(F.col("defrost_cycle").cast("double")).alias("defrost_active_fraction"),
```

Casting boolean to double (true → 1.0, false → 0.0) and averaging gives the fraction directly. Null boolean values are excluded from the average by Spark's default null handling.

## Data Quality Features

Two features in the [`device_features` schema](../05-architecture/data-contracts.md) track data quality at the window level:

### `reading_count`

Number of raw readings in the window. For a 15-minute window at 1 reading/min, the expected count is ~15. Significantly fewer readings indicate sensor dropout or connectivity issues.

This is not a derived feature — it's a `COUNT(*)` within the window grouping. But it's critical context for all other features: a mean computed from 2 readings is far less reliable than one computed from 15.

### `sensor_completeness`

Fraction of non-null sensor values across all readings in the window (0.0 to 1.0). Computed from the raw `sensor_null_count` column in [`clean_sensor_readings`](../05-architecture/data-contracts.md) (Contract 2).

```python
# sensor_completeness: fraction of non-null sensor readings
# Each reading has 14 sensor columns; sensor_null_count says how many are null
F.avg(1.0 - F.col("sensor_null_count") / 14.0).alias("sensor_completeness")
```

### `quality_flag_count`

Number of readings with non-empty `quality_flags` in the window. High counts signal upstream data quality issues that might make features unreliable.

```python
F.sum(
    F.when(F.size("quality_flags") > 0, 1).otherwise(0)
).alias("quality_flag_count")
```

## Lag Features (Window-to-Window Comparison)

Lag features are not explicit columns in the [Contract 3 schema](../05-architecture/data-contracts.md), but they are worth discussion because downstream models may compute them from the stored features.

A lag feature compares the current window's value to the same feature N windows ago. For example: "current 1-hour `temp_evaporator_mean` minus the `temp_evaporator_mean` from 6 hours ago." This captures medium-term drift that slope (within a single window) misses.

**Why not store lag features directly**: lag features multiply the number of columns (each feature × each lag horizon). Since the offline feature store retains historical windows, downstream model-input transforms can compute lags by self-joining `device_features` on `device_id` with a time offset. This keeps the feature store schema manageable and avoids re-computing all lags when a new lag horizon is needed.

**Trade-off**: computing lags at model-input time introduces a self-join on `device_features`. For 100K devices × 3 window sizes, this is feasible but should use partition pruning on `window_start`.

> **Data Leakage Warning**: lag joins must respect temporal train/test splits. When constructing training examples for a model with training cutoff at time T, lag features for windows near the cutoff must not reference any window with `window_end > T`. In practice, this means the lag join condition should include `lagged.window_end <= :training_cutoff` in addition to the time-offset condition. Failing to enforce this allows the model to learn from future data during training — features that will not be available at inference time. See [Feature Store — Point-in-Time Correctness](./feature-store.md) for the broader leakage prevention framework.

## Implementation Notes

### Transform Structure

Following the [pre-aggregate then merge pattern](../04-palantir/transform-patterns.md), time-domain features are computed in two stages:

1. **Micro-aggregation (incremental)**: processes new readings as they arrive, computing per-5-minute statistics per device per sensor. This transform runs incrementally on new `clean_sensor_readings` transactions.

2. **Window aggregation (scheduled)**: rolls up micro-aggregates into the target window sizes (15-min, 1-hour, 1-day). This transform runs on schedule — see [Windowing Strategies](./windowing.md) for details.

### Performance Considerations

- For slope computation, the manual sum-of-products approach avoids UDF overhead and stays within Spark's native execution engine. Pandas UDFs would work but add serialization cost at 100K-device scale.
- `percentile_approx` is preferred over exact percentile for p95 — the approximation error is negligible for anomaly detection, and performance is significantly better.
- All window aggregations should be computed in a single `groupBy` pass per window size, not separate passes per feature. This avoids redundant shuffles.

## Feature Scaling Considerations

Sensor readings span vastly different physical scales: evaporator temperature operates around -25°C to 0°C, high-side pressure around 500–2500 kPa, compressor current around 5–30 A, and vibration in arbitrary acceleration units. When these raw-scale features enter a distance-based or reconstruction-based model, the features with the largest numeric range dominate.

### Standardization for Anomaly Detection

For Isolation Forest, standardization (z-score normalization) is not strictly required because the algorithm splits on individual features. However, when features vary by orders of magnitude, the random split thresholds become unevenly distributed across features, subtly biasing the model. For Autoencoders, standardization is critical — reconstruction error is computed across all features, and unscaled features cause the model to focus on minimizing error for high-magnitude features while ignoring low-magnitude ones.

Recommendation: apply z-score standardization (subtract mean, divide by std) to all numeric features before model input.

### Critical Rule: Scale After Temporal Split

Scaling parameters (mean and std per feature) must be computed exclusively from the training partition. The procedure:

1. Split `device_features` temporally (train on `window_end ≤ T`, validate/test on `window_end > T`)
2. Compute per-feature mean and std from the training partition only
3. Apply those same parameters to the validation and test partitions

Fitting the scaler on the full dataset (including validation/test) leaks distributional information from the future into the training process. This is a common source of overly optimistic evaluation metrics that do not hold in production.

### Scaling Is Not a Feature Store Concern

Features in `device_features` are stored unscaled, in their original physical units. Scaling is applied at model-input transform time ([Contract 4](../05-architecture/data-contracts.md)), not during feature computation. This ensures that:

- The feature store remains model-agnostic (different models may use different scaling strategies)
- Stored values are interpretable by humans and dashboards without inverse-transforming
- Scaling parameters are versioned with the model artifact, not with the feature pipeline

## Feature Distribution Monitoring

Feature distributions shift over time due to seasonal effects, fleet composition changes, firmware updates, and equipment aging. Monitoring these shifts is essential for knowing when models need retraining.

### Per-Feature Fleet-Wide Tracking

For each numeric feature column in `device_features`, track the following statistics per day across the entire fleet:

| Statistic | Purpose |
|-----------|---------|
| Mean | Detect fleet-wide baseline shift |
| Std | Detect variance changes (fleet homogeneity) |
| p5 / p50 / p95 | Detect distributional shape changes beyond mean and std |
| Null fraction | Detect upstream data quality degradation |

Compute these as a daily monitoring transform that reads the 1-day window partition and outputs one summary row per feature per day.

### Seasonal Baselines

Temperature-dependent features (`temp_evaporator_mean`, `temp_condenser_mean`, `temp_spread_evap_cond`, `current_per_temp_spread`) exhibit strong seasonal patterns. A rising `temp_condenser_mean` fleet-wide in summer is expected, not drift. Monitoring must compare against the same calendar period from the prior year, not just the prior week. Build seasonal baseline profiles from the first full year of data and update annually.

### Feature-Level Drift as Retraining Trigger

Drift detection compares the current feature distribution against the reference distribution from the model's training set. Three complementary metrics:

| Metric | Strengths | Threshold |
|--------|-----------|----------|
| KL Divergence | Sensitive to tail changes | > 0.1 nats moderate, > 0.5 nats significant |
| PSI (Population Stability Index) | Industry-standard, interpretable bins | > 0.1 moderate, > 0.25 significant |
| Wasserstein Distance | Metric-aware (respects value magnitude), no binning required | Feature-specific, calibrate empirically |

When multiple features cross the moderate threshold simultaneously, or any single feature crosses the significant threshold, this should trigger a retraining evaluation. Drift detection does not automatically retrain — it creates a ticket for the ML Scientist to investigate whether the drift is benign (seasonal) or degrading model performance.

See [Monitoring](../03-production/monitoring.md) for the operational monitoring framework that consumes these drift signals.

## Related Documents

- [Data Contracts — Contract 3](../05-architecture/data-contracts.md) — authoritative schema for all output feature columns
- [Cross-Sensor Features](./cross-sensor.md) — derived features that combine multiple time-domain aggregations
- [Frequency-Domain Features](./frequency-domain.md) — complementary spectral features for vibration and current
- [Windowing Strategies](./windowing.md) — how windows are defined, aligned, and computed
- [Transform Patterns](../04-palantir/transform-patterns.md) — Foundry `@transform_df` and incremental processing idioms
