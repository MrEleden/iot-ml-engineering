# Cross-Sensor Features

Individual sensor readings tell you what's happening. Cross-sensor features tell you *why*. A temperature spike alone is ambiguous — but a temperature spike with stable pressure and dropping current strongly suggests a compressor failure.

## Sensor Ratios

The simplest cross-sensor feature: ratios between related measurements.

```python
import pyspark.sql.functions as F

# Pivot sensor types into columns first
df_wide = (
    df
    .groupBy("device_id", F.window("timestamp", "5 minutes").alias("window"))
    .pivot("sensor_type", ["temperature", "pressure", "current", "vibration"])
    .agg(F.avg("value"))
)

# Cross-sensor ratios
df_features = (
    df_wide
    .withColumn("pressure_per_temp", F.col("pressure") / F.col("temperature"))
    .withColumn("current_per_pressure", F.col("current") / F.col("pressure"))
    .withColumn("vibration_per_current", F.col("vibration") / F.col("current"))
)
```

**Why ratios matter**: a refrigeration compressor has a predictable relationship between suction pressure and discharge temperature. When that ratio drifts, the system is degrading — even if both values are individually within spec.

## Rolling Correlations

How tightly are two sensors tracking each other? A change in correlation indicates a regime change.

```python
from pyspark.sql import Window

w_24h = (
    Window
    .partitionBy("device_id")
    .orderBy(F.col("window.start").cast("long"))
    .rowsBetween(-288, 0)  # 288 x 5min windows = 24 hours
)

df_features = (
    df_wide
    .withColumn("corr_temp_pressure_24h", F.corr("temperature", "pressure").over(w_24h))
    .withColumn("corr_current_vibration_24h", F.corr("current", "vibration").over(w_24h))
)
```

**Interpretation**:
- `corr_temp_pressure` dropping from 0.9 to 0.3 → the compressor is no longer efficiently converting pressure to cooling
- `corr_current_vibration` rising toward 1.0 → motor is struggling (current and vibration both increasing together)

## Fleet-Level Aggregations

Compare a device to its peers. Is this unit behaving differently from others in the same environment?

```python
# Fleet average for the same site, same equipment type
w_fleet = Window.partitionBy("site_id", "equipment_type", "window")

df_features = (
    df_wide
    .withColumn("fleet_temp_mean", F.avg("temperature").over(w_fleet))
    .withColumn("fleet_temp_std", F.stddev("temperature").over(w_fleet))
    .withColumn(
        "temp_z_score_vs_fleet",
        (F.col("temperature") - F.col("fleet_temp_mean")) / F.col("fleet_temp_std"),
    )
)
```

A `temp_z_score_vs_fleet` of 3.0 means this unit is running 3 standard deviations warmer than its peers in the same warehouse. That's a strong signal — something is wrong with *this specific unit*, not the environment.

## Delta from Expected

If you have physics-based models or manufacturer specs, compute deviations:

```python
# Expected temperature given ambient conditions and load
df_features = (
    df_features
    .withColumn(
        "temp_delta_from_setpoint",
        F.col("temperature") - F.col("setpoint_temperature"),
    )
    .withColumn(
        "cop_estimated",  # Coefficient of Performance
        F.col("cooling_output") / F.col("current"),
    )
    .withColumn(
        "cop_degradation_pct",
        (F.col("cop_nominal") - F.col("cop_estimated")) / F.col("cop_nominal") * 100,
    )
)
```

## The Pivot Problem

Cross-sensor features require wide-format data (one column per sensor type), but IoT data usually arrives in long format (one row per reading). The pivot step can be expensive.

**Optimization**: pre-aggregate before pivoting:

```python
# BAD: pivot all raw readings (millions of rows per device)
df_wide = df.pivot("sensor_type").agg(F.avg("value"))

# GOOD: aggregate into time buckets first, then pivot (much fewer rows)
df_bucketed = (
    df
    .withColumn("bucket", F.window("timestamp", "5 minutes"))
    .groupBy("device_id", "bucket", "sensor_type")
    .agg(
        F.avg("value").alias("mean_value"),
        F.stddev("value").alias("std_value"),
        F.count("*").alias("count"),
    )
)

df_wide = (
    df_bucketed
    .groupBy("device_id", "bucket")
    .pivot("sensor_type")
    .agg(
        F.first("mean_value").alias("mean"),
        F.first("std_value").alias("std"),
        F.first("count").alias("count"),
    )
)
```

This reduces the pivot from O(billions) to O(millions) of rows — often 100x faster.

## Null Handling for Cross-Sensor Features

Not all sensors report at the same frequency. Temperature might report every 10s, pressure every 60s.

```python
# Don't compute ratios when one sensor is missing
df_features = df_features.withColumn(
    "pressure_per_temp",
    F.when(
        F.col("temperature").isNotNull() & F.col("pressure").isNotNull(),
        F.col("pressure") / F.col("temperature"),
    ).otherwise(F.lit(None)),
)
```

**Never fill missing sensor readings with zeros** — 0 pressure is a physically meaningful (and alarming) value. Use `None` and let the model or imputation step handle it explicitly.
