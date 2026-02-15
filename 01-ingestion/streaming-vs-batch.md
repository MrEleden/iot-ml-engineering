# Streaming vs Batch Tradeoffs

## When to Use Each

| | Streaming | Batch |
|---|---|---|
| **Latency** | Seconds to minutes | Minutes to hours |
| **Use case** | Real-time alerts, live dashboards | Feature engineering, model training, reporting |
| **Complexity** | Higher (state management, exactly-once) | Lower (idempotent rewrites) |
| **Cost** | Always-on compute | Scheduled, scales to zero |
| **Debugging** | Harder (stateful, time-dependent) | Easier (reproducible, replayable) |

## The Hybrid Pattern (Lambda-ish)

In practice, most IoT ML systems need both. The pattern:

```
Kafka ──┬──▶ Stream Processor ──▶ Real-time alerts (seconds)
        │
        └──▶ Landing Zone (Parquet) ──▶ Batch PySpark ──▶ Feature Store ──▶ Model Training
```

**Stream path**: lightweight rules — threshold alerts ("temperature > -10°C for fridge"), device health monitoring, live dashboards. Minimal state, minimal feature computation.

**Batch path**: heavy feature engineering — rolling windows, cross-sensor correlations, frequency-domain features. Runs on schedule (hourly or daily), reads from the landing zone.

## Why Not Pure Streaming?

It's tempting to do everything in streaming (Spark Structured Streaming, Flink, etc.), but for ML feature engineering:

1. **Feature computation is expensive** — computing a 7-day rolling standard deviation across 10,000 devices in a stream requires maintaining large state. In batch, it's a simple window function over Parquet files.

2. **Backfill is painful** — when you add a new feature or fix a bug, you need to recompute history. Streaming has no concept of "reprocess last 6 months." Batch does this natively.

3. **Point-in-time correctness is critical** — ML features must be computed as of a specific timestamp (no future data leakage). This is straightforward in batch, error-prone in streaming.

4. **Cost** — streaming jobs run 24/7. Batch jobs run for 30 minutes and shut down.

## When Streaming Is Worth It

- **Real-time alerting** — "compressor pressure exceeding threshold" must fire in seconds, not hours
- **Live operational dashboards** — operators need current state of equipment
- **Feature freshness for online models** — if your serving model needs features computed in the last 5 minutes

## Practical Rule

> Do alerting in streaming, feature engineering in batch, and feature serving from a pre-computed store.

This gives you sub-second alerts, cost-effective feature computation, and low-latency model serving — without the complexity of maintaining a full streaming feature pipeline.

## PySpark Example: Batch Reads from Streaming Landing Zone

```python
# Read from the landing zone written by the streaming job
df = (
    spark.read
    .format("delta")
    .load("s3://datalake/raw/sensor_telemetry/")
    .where(col("timestamp") >= "2026-02-14")
    .where(col("timestamp") < "2026-02-15")
)

# Compute features in batch — simple, debuggable, replayable
features = (
    df
    .groupBy("device_id")
    .agg(
        F.avg("value").alias("temp_mean_24h"),
        F.stddev("value").alias("temp_std_24h"),
        F.count("*").alias("reading_count_24h"),
    )
)
```

No state management, no watermarks, no late-arrival handling — the batch job just reads what's there and computes.
