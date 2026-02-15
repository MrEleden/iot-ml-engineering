# Windowing Strategies

How you define time windows determines what your features can capture. Wrong windowing leads to data leakage, missed patterns, or exploding compute costs.

## Window Types

### Tumbling Windows (non-overlapping)

Fixed-size, non-overlapping. Every reading belongs to exactly one window.

```python
# 1-hour tumbling windows
df = (
    df
    .withColumn("window", F.window("timestamp", "1 hour"))
    .groupBy("device_id", "window")
    .agg(
        F.avg("value").alias("temp_mean"),
        F.stddev("value").alias("temp_std"),
        F.min("value").alias("temp_min"),
        F.max("value").alias("temp_max"),
        F.count("*").alias("reading_count"),
    )
)
```

**Use when**: computing aggregated features for model training at a fixed cadence (hourly features, daily features). Simple, efficient, no overlap.

**Drawback**: boundary effects. An event that spans two windows gets split.

### Sliding Windows (overlapping)

Fixed-size, moves by a step smaller than the window size.

```python
# 1-hour window, sliding every 15 minutes
df = (
    df
    .withColumn("window", F.window("timestamp", "1 hour", "15 minutes"))
    .groupBy("device_id", "window")
    .agg(F.avg("value").alias("temp_mean"))
)
```

**Use when**: you need finer-grained feature updates without missing boundary events. Common for real-time feature computation.

**Drawback**: 4x more output rows than tumbling (1h window / 15min slide = 4 overlapping windows per reading). Watch memory and storage.

### Range-Based Windows (PySpark Window functions)

Not Spark's `window()` function — these are SQL-style window functions with `rangeBetween`:

```python
from pyspark.sql import Window

# Look back exactly 24 hours from each reading
w = (
    Window
    .partitionBy("device_id")
    .orderBy(F.col("timestamp").cast("long"))
    .rangeBetween(-86400, 0)
)

df = df.withColumn("temp_mean_24h", F.avg("value").over(w))
```

**Use when**: computing per-reading features (every row gets its own trailing window). Most flexible, but most expensive — computes a window per row.

## Choosing the Right Window

| Question | Tumbling | Sliding | Range-based |
|----------|----------|---------|-------------|
| Feature cadence = model cadence? | Yes — hourly features for hourly predictions | | |
| Need smooth feature updates? | | Yes — updated every 15 min | |
| Need per-reading features? | | | Yes — every row has trailing stats |
| Compute cost | Low | Medium | High |
| Backfill cost | Low | Medium | High |
| Storage | 1x | Nx (overlap factor) | Nx (per-reading) |

## Window Size Selection

No universal answer. Depends on the physics of your system:

| Equipment | Relevant timescales | Suggested windows |
|-----------|-------------------|-------------------|
| Refrigeration compressor | Cycle: 10-30 min, drift: hours-days | 1h, 6h, 24h, 7d |
| HVAC system | Day/night: 12h, seasonal: months | 6h, 24h, 7d, 30d |
| Vibration monitoring | Fault onset: minutes, degradation: weeks | 5min, 1h, 24h |
| Electrical systems | Transients: seconds, load patterns: hours | 1min, 15min, 1h, 24h |

**Practical approach**: start with 1h, 24h, 7d. Add shorter/longer windows only when the model needs them.

## Alignment and Boundaries

### Align to clock time, not data time

```python
# GOOD: aligned to calendar hours (00:00, 01:00, 02:00, ...)
F.window("timestamp", "1 hour")

# BAD: custom start time that drifts
F.window("timestamp", "1 hour", startTime="13 minutes")
```

Calendar-aligned windows are easier to debug, join with other data sources, and explain to stakeholders.

### Handle timezone correctly

```python
# Convert to UTC before windowing
df = df.withColumn("timestamp_utc", F.to_utc_timestamp("timestamp", "Europe/Paris"))

# Window on UTC
df = df.withColumn("window", F.window("timestamp_utc", "1 hour"))
```

Devices in different timezones should all window on UTC. Local-time windowing creates subtle bugs around DST transitions (a 23-hour or 25-hour "day").

## Data Leakage in Windowing

The most common ML bug in time-series feature engineering:

```python
# WRONG: window includes future data
w_centered = (
    Window
    .partitionBy("device_id")
    .orderBy(F.col("timestamp").cast("long"))
    .rangeBetween(-43200, 43200)  # ±12 hours — includes future!
)

# RIGHT: window only looks backward
w_trailing = (
    Window
    .partitionBy("device_id")
    .orderBy(F.col("timestamp").cast("long"))
    .rangeBetween(-86400, 0)  # trailing 24 hours only
)
```

**Rule**: features must only use data from `timestamp` and before. Never include the future. This seems obvious but centered windows, improper joins, and global statistics (computed on full dataset) are common leakage sources.

## Performance: Reducing Window Compute Cost

Range-based windows are O(N * W) where N = rows and W = window size. For 10M rows with a 7-day window, this is expensive.

**Optimization 1**: pre-aggregate to hourly, then window on hourly data:

```python
# Instead of windowing over 604,800 raw readings (7 days at 1/sec)
# Window over 168 hourly aggregates
df_hourly = (
    df
    .withColumn("hour", F.date_trunc("hour", "timestamp"))
    .groupBy("device_id", "hour")
    .agg(F.avg("value").alias("temp_mean"), F.stddev("value").alias("temp_std"))
)

w_7d = Window.partitionBy("device_id").orderBy("hour").rowsBetween(-168, 0)
df_hourly = df_hourly.withColumn("temp_mean_7d", F.avg("temp_mean").over(w_7d))
```

This is ~3600x less data to window over, with minimal loss of information for long-horizon features.

**Optimization 2**: materialize intermediate window results:

```python
# Compute 24h features, save, then compute 7d features on top of 24h aggregates
# Don't recompute from raw every time
```
