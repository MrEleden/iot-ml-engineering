# Feature Store Patterns

A feature store decouples feature computation from model training and serving. Without one, you end up recomputing the same features in every notebook, with slightly different implementations that cause training/serving skew.

## Core Concept

```
Raw Sensor Data → Feature Pipeline (PySpark) → Feature Store → Model Training (offline)
                                                      │
                                                      └──────→ Model Serving (online)
```

The feature store is a single source of truth for feature values, indexed by entity (device_id) and timestamp.

## Offline vs Online Serving

| | Offline (batch) | Online (real-time) |
|---|---|---|
| **Consumer** | Model training, batch scoring | Real-time inference API |
| **Latency** | Minutes to hours | < 100ms |
| **Storage** | Delta Lake / Parquet | Redis, DynamoDB, or Palantir Object Store |
| **Query** | "Give me all features for all devices for the last 6 months" | "Give me current features for device-0042 right now" |

## Offline Feature Store in PySpark

```python
# Feature pipeline output — one row per (device_id, timestamp)
features = (
    spark.read.format("delta").load(".../sensor_telemetry/")
    .transform(compute_time_domain_features)
    .transform(compute_frequency_features)
    .transform(compute_cross_sensor_features)
)

# Write to offline store, partitioned by date for efficient range queries
(
    features
    .write
    .format("delta")
    .partitionBy("date")
    .mode("overwrite")
    .option("replaceWhere", f"date = '{target_date}'")
    .save("s3://feature-store/predictive_maintenance/v1/")
)
```

**Partition by date, not device_id**: training queries scan time ranges (last 6 months), not device ranges. Date partitioning enables partition pruning on the most common access pattern.

## Online Feature Store

For real-time model serving, pre-compute and cache the latest features per device:

```python
# Compute latest features per device
latest_features = (
    features
    .withColumn("rank", F.row_number().over(
        Window.partitionBy("device_id").orderBy(F.desc("timestamp"))
    ))
    .where(F.col("rank") == 1)
    .drop("rank")
)

# Push to online store (Redis example)
for row in latest_features.collect():
    redis_client.hset(
        f"features:{row.device_id}",
        mapping={col: str(row[col]) for col in feature_columns},
    )
    redis_client.expire(f"features:{row.device_id}", ttl=7200)  # 2h TTL
```

In Palantir Foundry, this is handled via **Ontology objects with pre-hydrated properties** — the feature values are materialized onto the object representing each device, ready for model serving endpoints.

## Point-in-Time Correctness

The most important property of a feature store for ML. When training a model, you must use features *as they were at prediction time*, not as they are now.

**The problem**: you're training a model to predict failures. A device failed on Feb 10. When computing training features for Feb 9, you must use the feature values that existed on Feb 9 — not recompute them from data that includes Feb 10.

```python
# WRONG: compute features on all historical data, then join with labels
# This uses future data to compute features

# RIGHT: for each label timestamp, join features computed strictly before that time
training_data = (
    labels.alias("l")
    .join(
        features.alias("f"),
        on=[
            F.col("l.device_id") == F.col("f.device_id"),
            F.col("f.timestamp") <= F.col("l.label_timestamp"),
            F.col("f.timestamp") > F.col("l.label_timestamp") - F.expr("INTERVAL 1 HOUR"),
        ],
        how="inner",
    )
)
```

## Backfill

When you add a new feature or fix a computation bug, you need to recompute historical features:

```python
# Backfill last 6 months for a new feature
dates = pd.date_range("2025-08-01", "2026-02-15", freq="D")

for date in dates:
    df_day = spark.read.format("delta").load(f".../sensor_telemetry/date={date}/")
    features_day = compute_features(df_day)  # includes new feature
    (
        features_day
        .write
        .format("delta")
        .mode("overwrite")
        .option("replaceWhere", f"date = '{date}'")
        .save("s3://feature-store/predictive_maintenance/v2/")
    )
```

**Key rules**:
- Version your feature sets (`v1`, `v2`). Never mutate `v1` in place.
- Backfill writes to `v2`. Old models keep reading `v1`.
- Only switch models to `v2` after validating that the backfill is correct.

## Feature Metadata

Track what each feature means and how it's computed:

```python
FEATURE_REGISTRY = {
    "temp_mean_24h": {
        "description": "Mean temperature over trailing 24-hour window",
        "unit": "celsius",
        "window": "24h trailing",
        "source_sensors": ["temperature"],
        "null_behavior": "NULL if < 100 readings in window",
        "added_version": "v1",
    },
    "spectral_entropy_1h": {
        "description": "Entropy of FFT power spectrum over 1-hour window",
        "unit": "bits",
        "window": "1h tumbling",
        "source_sensors": ["vibration"],
        "null_behavior": "NULL if < 60 readings in window",
        "added_version": "v2",
    },
}
```

Without this, features become black boxes within weeks. New team members can't trust features they don't understand, and they'll recompute from scratch — which is the exact problem the feature store was supposed to solve.
