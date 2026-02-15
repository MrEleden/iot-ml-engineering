# Transform Patterns for IoT Time-Series Data

How to structure Foundry Transforms for processing 14M sensor readings per minute from 100K refrigeration devices — covering decorator choice, incremental processing, windowed aggregations, scheduling, and common pitfalls.

See [Foundry Platform Reference](./foundry-platform-reference.md) for general Transform concepts. This document focuses on patterns specific to high-volume IoT time-series.

## @transform_df vs @transform — When to Use Each

### @transform_df

Use `@transform_df` when your logic is a pure DataFrame-to-DataFrame transformation — which covers 90% of IoT feature engineering.

```python
from transforms.api import transform_df, Input, Output

@transform_df(
    Output("/Company/pipelines/refrigeration/features/rolling_stats"),
    sensor_readings=Input("/Company/pipelines/refrigeration/raw/sensor_readings"),
)
def compute_rolling_stats(sensor_readings):
    # sensor_readings is already a PySpark DataFrame
    # Return a PySpark DataFrame — Foundry handles writing it
    return sensor_readings.groupBy("device_id").agg(...)
```

**Why it fits IoT**: the decorator manages reading inputs as DataFrames and writing the output as a dataset. You never touch file I/O, transaction management, or Spark session creation. This is enforced — Foundry will reject Transforms that attempt raw `SparkSession` access.

### @transform

Use `@transform` when you need access to the raw `TransformInput` / `TransformOutput` objects. Concrete use cases in our pipeline:

- **Writing partitioned output with explicit partition columns**: `@transform` lets you call `output.write_dataframe(df, partition_cols=["date", "hour"])`
- **Reading file metadata**: when you need to inspect schema versions or file counts before processing
- **Multi-format output**: rare, but if a Transform produces both tabular data and a model artifact

```python
from transforms.api import transform, Input, Output

@transform(
    output=Output("/Company/pipelines/refrigeration/features/partitioned_features"),
    sensor_readings=Input("/Company/pipelines/refrigeration/raw/sensor_readings"),
)
def compute_partitioned_features(sensor_readings, output):
    df = sensor_readings.dataframe()
    result = df.groupBy("device_id", "date", "hour").agg(...)
    output.write_dataframe(result, partition_cols=["date", "hour"])
```

### Decision Rule

Default to `@transform_df`. Switch to `@transform` only when you need explicit control over output writing (partitioning, multi-output). Never use `@transform` just because it "feels more flexible" — the extra boilerplate increases the surface area for bugs.

## Incremental Transforms for Append-Only Ingestion

### Why Incremental

The streaming dataset materializes ~70M records every 5 minutes as new transactions. A full-recompute Transform would re-read the entire history (~20B+ records/day) on every run. Incremental Transforms process only the new transactions, reducing compute by orders of magnitude.

### Incremental Decorator Pattern

```python
from transforms.api import transform_df, Input, Output, incremental

@incremental()
@transform_df(
    Output("/Company/pipelines/refrigeration/features/hourly_device_stats"),
    sensor_readings=Input("/Company/pipelines/refrigeration/raw/sensor_readings"),
)
def compute_hourly_stats(sensor_readings):
    # sensor_readings contains ONLY new rows since the last successful run
    return (
        sensor_readings
        .groupBy("device_id", "date", "hour")
        .agg(
            F.avg("value").alias("mean_value"),
            F.stddev("value").alias("stddev_value"),
            F.count("*").alias("reading_count"),
        )
    )
```

### Append vs Snapshot Semantics

Incremental Transforms can produce output in two modes:

- **Append** (default): each run appends new rows to the output dataset. Previous rows are never modified. Correct for our use case because IoT readings are immutable — a reading from 10:00 AM never changes.
- **Snapshot with modifications**: each run can update or delete existing rows. Required if you need to re-aggregate a window that spans multiple transactions (e.g., a reading arrives late and changes the hourly average). More expensive — use only when late-arriving data is common enough to affect feature quality.

For the refrigeration pipeline, use **append mode** for raw-to-feature Transforms and **snapshot mode** only for the final feature table that must reflect corrected late-arriving data.

### What Makes a Transform Eligible for Incremental

- All input datasets must support incremental reads (streaming datasets do by default; batch datasets do if they use append transactions)
- The Transform logic must be able to produce correct output from a partial view of the input (i.e., it doesn't need to see the full history on every run)
- Aggregations that span multiple transactions (e.g., 24-hour rolling average) need special handling — see [Windowed Aggregations](#windowed-aggregations) below

### Incremental State and Checkpointing

Foundry tracks the "high-water mark" — the last transaction ID it successfully processed for each input. If a Transform fails mid-run, Foundry rolls back the output transaction and replays from the last checkpoint on the next run. You don't manage checkpoints yourself, but you should be aware:

- A failed run that processed 50% of a transaction does NOT produce partial output — it's all-or-nothing per transaction
- If you change the Transform logic (e.g., add a new aggregation column), Foundry may require a full recompute to initialize the new column across all historical data. Plan for this — it's expensive on 20B+ records

## Windowed Aggregations

### The Challenge

IoT feature engineering relies heavily on windowed aggregations: rolling averages, rate-of-change over the last N minutes, standard deviation over the last hour. In an incremental Transform, you only see the new records — but a 1-hour rolling average at 10:05 needs data from 09:05 to 10:05, spanning multiple 5-minute materialization transactions.

### Pattern 1: Pre-Aggregate Then Merge (Recommended)

Split the computation into two Transforms:

**Transform 1 (incremental)**: Compute per-transaction micro-aggregates.

```python
@incremental()
@transform_df(
    Output("/Company/pipelines/refrigeration/features/micro_aggs"),
    sensor_readings=Input("/Company/pipelines/refrigeration/raw/sensor_readings"),
)
def micro_aggregates(sensor_readings):
    return (
        sensor_readings
        .groupBy("device_id", "sensor_type",
                 F.window("timestamp", "5 minutes").alias("window"))
        .agg(
            F.avg("value").alias("avg_value"),
            F.sum("value").alias("sum_value"),
            F.count("*").alias("count"),
            F.min("value").alias("min_value"),
            F.max("value").alias("max_value"),
        )
    )
```

**Transform 2 (batch, scheduled hourly)**: Read the micro-aggregates and compute full windows.

```python
@transform_df(
    Output("/Company/pipelines/refrigeration/features/rolling_1h_stats"),
    micro_aggs=Input("/Company/pipelines/refrigeration/features/micro_aggs"),
)
def rolling_hourly_stats(micro_aggs):
    # Read last 2 hours of micro-aggregates (buffer for late data)
    recent = micro_aggs.filter(F.col("window.start") >= F.current_timestamp() - F.expr("INTERVAL 2 HOURS"))
    
    window_spec = Window.partitionBy("device_id", "sensor_type").orderBy("window.start").rangeBetween(-12, 0)
    # -12 means 12 five-minute windows back = 1 hour
    
    return (
        recent
        .withColumn("rolling_avg", F.avg("avg_value").over(window_spec))
        .withColumn("rolling_stddev", F.stddev("avg_value").over(window_spec))
    )
```

**Why two Transforms?** Transform 1 is incremental and cheap — it processes only new data. Transform 2 is a full recompute but operates on micro-aggregates (288 rows/day/device instead of 1,440 raw readings/day/device), so it's affordable.

### Pattern 2: Stateful Incremental (Advanced)

Use `@transform` with explicit state management — read the previous output and merge with new input. This avoids full recomputes but requires careful reasoning about state consistency. Only use this if the pre-aggregate pattern creates too many intermediate datasets.

### Anti-Pattern: Full-History Window

Never write a single incremental Transform that tries to read the full history to compute a rolling window. Foundry will either OOM or silently fall back to full recompute, defeating the purpose of incremental processing.

## Transform Scheduling and Dependencies

### Build Schedules

Transforms don't run continuously — they run on schedules or when triggered by upstream dataset changes. For the refrigeration pipeline:

| Transform | Trigger | Rationale |
|---|---|---|
| Raw sensor ingestion (streaming) | Continuous | Streaming dataset — always on |
| Micro-aggregates | On new transaction in raw dataset | Incremental, runs every ~5 min as new data materializes |
| Hourly rolling features | Scheduled every hour | Batch, recomputes windows over recent micro-aggs |
| Daily device health scores | Scheduled daily at 02:00 UTC | Batch, aggregates full day of features |
| Batch model scoring | Scheduled every hour (or daily) | Depends on model freshness requirements |

### Dependency Chains

Foundry resolves Transform dependencies via the dataset lineage graph. If Transform B reads the output of Transform A, B will not run until A's latest transaction is committed. You don't need to configure this — it's automatic.

**Pitfall**: circular dependencies. If Transform A reads from dataset X and writes to dataset Y, and Transform B reads from dataset Y and writes to dataset X, Foundry will detect the cycle and refuse to schedule either. If you need feedback loops (e.g., model scores feeding back into features), break the cycle with a separate dataset.

### Scheduling and SLAs

Set pipeline SLAs on critical outputs. For example: "the `daily_device_health_scores` dataset must have a transaction newer than 6 hours old." If the SLA is violated (e.g., an upstream Transform failed), Foundry triggers an alert. This is configured in pipeline monitoring, not in Transform code.

## Common Pitfalls

### Full Recompute Triggers

An incremental Transform will fall back to full recompute if:

- **You change the Transform's output schema** (add/remove/rename columns) — Foundry must recompute the full output to apply the new schema consistently
- **You change the Transform logic in a way Foundry considers non-incremental** — e.g., switching from append to snapshot mode
- **An input dataset is compacted** — Foundry dataset compaction rewrites the transaction history, invalidating incremental checkpoints
- **You manually trigger a "force build" with full recompute** — sometimes necessary, but plan for the compute cost

For 20B+ records, a full recompute can take hours. Schedule schema changes during low-traffic windows and notify downstream consumers.

### OOM on Large Windows

A `Window.partitionBy("device_id").orderBy("timestamp").rowsBetween(-100000, 0)` across 100K devices will shuffle the entire dataset into 100K partitions. If any single device has a disproportionate number of readings (e.g., a misconfigured device sending 10x normal rate), one executor OOMs while others idle.

**Fix**: cap the window size in the filter step *before* the window function, not inside the window spec. Pre-filter to the last N hours of data, then apply the window.

### Transform Output Size Creep

Append-mode incremental Transforms grow indefinitely. After 30 days, the `micro_aggs` dataset contains 30 × 288 × 100K × 14 ≈ 12B rows. Downstream Transforms that read this dataset (even with filters) still pay the metadata overhead of listing all transactions.

**Fix**: schedule periodic dataset compaction to merge old transactions. Alternatively, create a "hot" dataset (last 7 days, incremental) and archive older data to a separate "cold" dataset.

### Spark Configuration Assumptions

Foundry manages Spark cluster sizing. You cannot set `spark.executor.memory` or `spark.sql.shuffle.partitions` in Transform code — Foundry overrides these. If a Transform OOMs, the fix is restructuring the logic (smaller windows, pre-filtering, repartitioning the input), not tuning Spark configs.

You *can* set Spark profile hints in the Transform configuration (small, medium, large, x-large compute profiles), which influence the cluster Foundry provisions. Default to medium; scale up only after profiling.

## Related Documents

- [Streaming Ingestion](./streaming-ingestion.md) — how data arrives in the streaming dataset that Transforms consume
- [Ontology Design](./ontology-design.md) — the Ontology objects backed by Transform output datasets
- [Model Integration](./model-integration.md) — how model scoring is implemented as a Transform
- [Foundry Platform Reference](./foundry-platform-reference.md) — general Transform concepts
