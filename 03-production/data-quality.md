# Data Quality & Monitoring

Sensor data is messy. Devices drop offline, send garbage values, drift out of calibration, or get replaced with different firmware. Your pipeline must detect and handle all of this — silently propagating bad data into features is worse than having no data at all.

## Sensor Dropout Detection

Track when devices stop reporting:

```python
from pyspark.sql import functions as F

# Last seen timestamp per device
df_heartbeat = (
    df
    .groupBy("device_id")
    .agg(F.max("timestamp").alias("last_seen"))
    .withColumn("hours_since_last_reading",
        (F.current_timestamp().cast("long") - F.col("last_seen").cast("long")) / 3600
    )
)

# Flag devices that haven't reported in 2x their expected interval
df_stale = df_heartbeat.where(F.col("hours_since_last_reading") > 2.0)
```

**Why this matters for ML**: a device that stops reporting often precedes a failure. But if your feature pipeline silently forward-fills the last known value, the model never sees the dropout — and misses the signal.

## Outlier Detection

### Statistical bounds

```python
# Per-device, per-sensor bounds from historical data
df_bounds = (
    df_historical
    .groupBy("device_id", "sensor_type")
    .agg(
        F.avg("value").alias("mean"),
        F.stddev("value").alias("std"),
        F.percentile_approx("value", 0.01).alias("p1"),
        F.percentile_approx("value", 0.99).alias("p99"),
    )
)

# Flag outliers
df_flagged = (
    df
    .join(df_bounds, on=["device_id", "sensor_type"])
    .withColumn("is_outlier",
        (F.col("value") < F.col("p1")) | (F.col("value") > F.col("p99"))
    )
    .withColumn("z_score",
        (F.col("value") - F.col("mean")) / F.col("std")
    )
)
```

### Physics-based bounds

More reliable than statistical bounds for known equipment:

```python
PHYSICS_BOUNDS = {
    "temperature": {"min": -50.0, "max": 60.0},    # Celsius
    "pressure": {"min": 0.0, "max": 30.0},          # bar
    "current": {"min": 0.0, "max": 100.0},          # amps
    "vibration": {"min": 0.0, "max": 50.0},         # mm/s RMS
}

# Flag physically impossible values
for sensor, bounds in PHYSICS_BOUNDS.items():
    df = df.withColumn(
        f"{sensor}_valid",
        F.when(
            (F.col("sensor_type") == sensor) &
            (F.col("value").between(bounds["min"], bounds["max"])),
            True,
        ).otherwise(False),
    )
```

A temperature reading of 500°C from a fridge sensor is not an outlier — it's a sensor fault. Don't let it contaminate your features.

## Data Drift Detection

Feature distributions shift over time. Detect this before your model degrades.

```python
# Compare current week's feature distribution to baseline
df_current = features.where(F.col("date") >= "2026-02-08")
df_baseline = features.where(F.col("date").between("2026-01-01", "2026-01-31"))

# Simple drift check: mean and std shift
for feature_col in MONITORED_FEATURES:
    current_stats = df_current.agg(
        F.avg(feature_col).alias("current_mean"),
        F.stddev(feature_col).alias("current_std"),
    ).first()

    baseline_stats = df_baseline.agg(
        F.avg(feature_col).alias("baseline_mean"),
        F.stddev(feature_col).alias("baseline_std"),
    ).first()

    # Z-score of the mean shift
    mean_shift_z = abs(
        (current_stats.current_mean - baseline_stats.baseline_mean)
        / baseline_stats.baseline_std
    )

    if mean_shift_z > 3.0:
        alert(f"Feature drift detected: {feature_col}, z-score={mean_shift_z:.1f}")
```

**Common drift causes**:
- Seasonal changes (summer vs winter affects temperature baselines)
- Fleet composition changes (new devices with different characteristics)
- Firmware updates changing sensor calibration
- Sensor degradation over time

## Monitoring Dashboard Metrics

Track these daily:

| Metric | Query | Alert threshold |
|--------|-------|-----------------|
| Active devices | `COUNT(DISTINCT device_id) WHERE last_seen > now() - 2h` | < 95% of expected |
| Reading volume | `COUNT(*) per hour` | < 80% of baseline |
| Null feature rate | `COUNT(NULL) / COUNT(*) per feature` | > 10% |
| Outlier rate | `COUNT(is_outlier) / COUNT(*)` | > 5% |
| Feature mean shift | Z-score of weekly mean vs baseline | > 3.0 |
| Pipeline latency | `MAX(processing_time - event_time)` | > 15 minutes |

## Quarantine, Don't Drop

Bad readings should be quarantined, not dropped:

```python
valid = df.where(F.col("is_valid") == True)
invalid = df.where(F.col("is_valid") == False)

valid.write.format("delta").mode("append").save(".../clean/")
invalid.write.format("delta").mode("append").save(".../quarantine/")
```

The quarantine table is valuable: it reveals sensor faults, firmware bugs, and data pipeline issues. Review it weekly.
