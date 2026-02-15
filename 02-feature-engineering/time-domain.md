# Time-Domain Features

The most common and interpretable features for IoT sensor data. These capture statistical properties of sensor readings within time windows.

## Rolling Statistics

The workhorse of sensor feature engineering. Computed over a sliding window per device.

```python
from pyspark.sql import Window
import pyspark.sql.functions as F

# Define a 24-hour rolling window per device, ordered by timestamp
w_24h = (
    Window
    .partitionBy("device_id")
    .orderBy(F.col("timestamp").cast("long"))
    .rangeBetween(-86400, 0)  # 86400 seconds = 24 hours
)

df_features = (
    df
    .withColumn("temp_mean_24h", F.avg("value").over(w_24h))
    .withColumn("temp_std_24h", F.stddev("value").over(w_24h))
    .withColumn("temp_min_24h", F.min("value").over(w_24h))
    .withColumn("temp_max_24h", F.max("value").over(w_24h))
    .withColumn("temp_range_24h", F.col("temp_max_24h") - F.col("temp_min_24h"))
    .withColumn("reading_count_24h", F.count("*").over(w_24h))
)
```

**Why `rangeBetween` instead of `rowsBetween`**: sensor readings aren't evenly spaced. A device might report every 10 seconds normally but every 60 seconds under load. `rangeBetween` on the timestamp ensures a true 24-hour window regardless of message frequency.

### Multiple Window Sizes

Different window sizes capture different dynamics:

```python
WINDOWS = {
    "1h": 3600,
    "6h": 21600,
    "24h": 86400,
    "7d": 604800,
}

for suffix, seconds in WINDOWS.items():
    w = (
        Window
        .partitionBy("device_id")
        .orderBy(F.col("timestamp").cast("long"))
        .rangeBetween(-seconds, 0)
    )
    df = (
        df
        .withColumn(f"temp_mean_{suffix}", F.avg("value").over(w))
        .withColumn(f"temp_std_{suffix}", F.stddev("value").over(w))
    )
```

Short windows (1h) capture recent anomalies. Long windows (7d) capture baseline behavior. The ratio between them is itself a useful feature (see below).

## Lag Features

Compare current reading to past readings:

```python
w_ordered = Window.partitionBy("device_id").orderBy("timestamp")

df = (
    df
    .withColumn("temp_lag_1h", F.lag("value", 60).over(w_ordered))  # 60 readings back at 1/min
    .withColumn("temp_lag_24h", F.lag("value", 1440).over(w_ordered))
    .withColumn("temp_diff_1h", F.col("value") - F.col("temp_lag_1h"))
    .withColumn("temp_diff_24h", F.col("value") - F.col("temp_lag_24h"))
)
```

**Caveat**: `lag(n)` counts rows, not time. If readings are irregular, use a self-join on timestamp range instead:

```python
df_lagged = (
    df.alias("current")
    .join(
        df.alias("past"),
        on=[
            F.col("current.device_id") == F.col("past.device_id"),
            F.col("past.timestamp").between(
                F.col("current.timestamp") - F.expr("INTERVAL 1 HOUR 5 MINUTES"),
                F.col("current.timestamp") - F.expr("INTERVAL 55 MINUTES"),
            ),
        ],
        how="left",
    )
    .select(
        F.col("current.*"),
        F.col("past.value").alias("temp_approx_1h_ago"),
    )
)
```

## Rate of Change

How fast is the sensor value changing? Critical for predictive maintenance — a slowly drifting temperature is normal, a rapid spike is not.

```python
w_ordered = Window.partitionBy("device_id").orderBy("timestamp")

df = (
    df
    .withColumn("prev_value", F.lag("value", 1).over(w_ordered))
    .withColumn("prev_ts", F.lag("timestamp", 1).over(w_ordered))
    .withColumn(
        "temp_rate_of_change",
        (F.col("value") - F.col("prev_value"))
        / (F.col("timestamp").cast("long") - F.col("prev_ts").cast("long")),
    )
    .drop("prev_value", "prev_ts")
)
```

This gives you degrees per second. For a fridge, `temp_rate_of_change > 0.01 °C/s` sustained over 5 minutes might indicate a compressor failure.

## Ratio Features

Ratios between different time windows detect regime changes:

```python
df = (
    df
    .withColumn(
        "temp_mean_ratio_1h_24h",
        F.col("temp_mean_1h") / F.col("temp_mean_24h"),
    )
    .withColumn(
        "temp_std_ratio_1h_7d",
        F.col("temp_std_1h") / F.col("temp_std_7d"),
    )
)
```

- `mean_ratio_1h_24h >> 1` → recent readings are much higher than the daily average (warming event)
- `std_ratio_1h_7d >> 1` → recent readings are much more volatile than usual (instability)

## Handling Nulls

Sensors drop out. Readings are missing. Never silently propagate nulls into features.

```python
# Count missing readings as a feature itself
df = df.withColumn(
    "missing_pct_24h",
    1.0 - (F.col("reading_count_24h") / F.lit(EXPECTED_READINGS_24H)),
)

# Filter out features computed on too few readings
df = df.withColumn(
    "temp_std_24h",
    F.when(F.col("reading_count_24h") >= MIN_READINGS, F.col("temp_std_24h"))
     .otherwise(F.lit(None)),
)
```

`missing_pct_24h` is often one of the most predictive features — a device that stops reporting frequently is likely having issues.
