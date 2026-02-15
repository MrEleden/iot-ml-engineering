# Feature Store

How `device_features` are stored for offline model training and served online for operational applications — covering the offline Delta Lake store, online Ontology pre-hydration, point-in-time correctness, backfill, schema versioning, and freshness monitoring.

## Two Stores, One Source of Truth

The feature store has two surfaces, both derived from the same computed features:

| | Offline Store | Online Store |
|---|---|---|
| **Backing** | `device_features` dataset on Delta Lake (Foundry) | `Device` Ontology object properties |
| **Data** | Full history: all devices × all windows × all time | Latest window only: most recent feature values per device |
| **Consumers** | Model training, batch scoring, historical analysis | Workshop dashboards, AIP agents, OSDK field apps |
| **Access pattern** | Scan: read months of data for thousands of devices | Point lookup: read current features for one or a few devices |
| **Latency** | Seconds to minutes (Spark query) | Milliseconds (Ontology API) |
| **Freshness** | Within 30 minutes of `clean_sensor_readings` update ([Contract 3 SLA](../05-architecture/data-contracts.md)) | Within 1 hour (bounded by pre-hydration transform schedule) |

The offline store is the system of record. The online store is a read-optimized projection of it.

## Offline Store: `device_features` on Delta Lake

### Dataset Structure

The `device_features` dataset is a Foundry dataset backed by Delta Lake. Each row is one device, one window, one window size:

```
device_features/
├── window_date=2026-02-14/
│   ├── window_duration_minutes=15/
│   │   └── part-00000.parquet  (all 15-min windows for all devices on this date)
│   ├── window_duration_minutes=60/
│   │   └── part-00000.parquet
│   └── window_duration_minutes=1440/
│       └── part-00000.parquet
├── window_date=2026-02-15/
│   └── ...
```

### Partitioning Strategy

Partitioned by:

1. **`window_date`** (derived from `window_start`): the date portion of the window start timestamp. Enables efficient time-range pruning for training queries ("give me 90 days of features") and for retention management.

2. **`window_duration_minutes`**: separates the three window sizes. Training queries typically target a single window size (e.g., "1-hour windows for the last 30 days"). Without this partition, every query would scan all three window sizes and discard two-thirds of the data.

This two-level partitioning produces approximately 3 partitions per day (one per window size). Partition granularity is coarse by design — over-partitioning (e.g., by `device_id`) would create 300K+ partitions/day (100K devices × 3 window sizes), which degrades Spark query planning performance.

### Row Volume

| Window Size | Rows/Day (100K devices) | Rows/Year |
|-------------|------------------------|-----------|
| 15-min | 9.6M | ~3.5B |
| 1-hour | 2.4M | ~876M |
| 1-day | 100K | ~36.5M |
| **Total** | **~12M** | **~4.4B** |

At ~50 columns per row, each row is roughly 400 bytes in Parquet. Daily data volume: ~4.8 GB. Annual: ~1.7 TB. This is well within Foundry's capabilities for a single dataset, but merits periodic compaction (see below).

### Compaction and Retention

Append-mode [incremental transforms](../04-palantir/transform-patterns.md) create many small transactions. After 30 days, the `device_features` dataset accumulates hundreds of transactions, each containing a few parquet files. This increases metadata overhead — listing files and planning queries gets slower.

**Compaction**: schedule periodic dataset compaction (weekly) to merge small files into larger ones. This is a Foundry-native operation; it rewrites the data without changing the logical contents.

**Retention**: define a retention policy based on downstream needs:
- Model training needs 6–12 months of history
- Historical analysis may need longer
- Data older than the retention window can be archived to a separate cold dataset

Retention is not a feature-engineering decision — coordinate with the [data platform](../05-architecture/system-overview.md). But the partitioning strategy above makes retention easy: drop old `window_date` partitions.

## Online Store: Ontology Pre-Hydration

### How It Works

The online feature store is implemented as pre-hydrated properties on the `Device` Ontology object type, as described in [Ontology Design](../04-palantir/ontology-design.md). The flow:

```
device_features     →     pre-hydration     →     Device object
(Delta Lake)               transform               (Ontology)
                          (hourly)
```

The pre-hydration transform:

1. Reads the latest 1-day window from `device_features` for each device
2. Joins with the device registry to get device metadata
3. Writes the result to the `Device` backing dataset

Workshop dashboards, AIP agents, and OSDK mobile apps query `Device` properties — they never touch the offline feature store directly.

### What Gets Pre-Hydrated

Only a subset of features are promoted to Ontology properties — those needed by operational applications. From the [Device object type](../04-palantir/ontology-design.md):

- `avg_temperature_evaporator_24h` ← from `temp_evaporator_mean` (1-day window)
- `avg_compressor_current_24h` ← from `current_compressor_mean` (1-day window)
- `reading_count_24h` ← from `reading_count` (1-day window)
- `latest_anomaly_score` ← from the scoring pipeline (not a feature, but shares the same pre-hydration transform)

The full 50+ feature columns remain in the offline store. Pre-hydrating all of them onto the Device object would bloat the Ontology sync pipeline and slow dashboard rendering — most features are only needed for model training (offline), not operational dashboards (online).

### Staleness Budget

| Component | Latency |
|-----------|---------|
| Sensor → clean_sensor_readings | ≤ 15 minutes ([Contract 2 SLA](../05-architecture/data-contracts.md)) |
| clean_sensor_readings → device_features | ≤ 30 minutes ([Contract 3 SLA](../05-architecture/data-contracts.md)) |
| device_features → Device Ontology | ≤ 1 hour (pre-hydration schedule) |
| **End-to-end** | **≤ ~1 hour 45 minutes** |

For the refrigeration use case, hourly freshness is acceptable — maintenance decisions don't require sub-minute feature data. If sub-minute freshness is ever needed for specific properties (e.g., for real-time anomaly scoring), consider Foundry's streaming Ontology sync as described in the [Ontology Design doc](../04-palantir/ontology-design.md).

### Training-Serving Consistency

A subtle source of model degradation is divergence between the feature values a model saw during training (offline store) and the values presented at inference time (online store or model-input transform). Even small differences compound across 50+ features.

**Risk areas**:

- **Pre-hydration transform divergence**: the transform that writes Ontology properties may apply rounding, casting, or formatting that differs from how the offline store presents the same feature. For example, casting a `double` to `float` for Ontology storage introduces precision loss that the model did not encounter during training.
- **Rounding and casting differences**: PySpark and Pandas may produce slightly different floating-point results for the same aggregation. If the training pipeline uses PySpark but the scoring pipeline uses a Pandas-based model-input transform, numerical differences can accumulate.
- **Feature subset mismatch**: the model expects features `[A, B, C]` but the serving path provides `[A, B, D]` due to a schema evolution where C was renamed to D. The model silently receives incorrect input.

**Prevention via weekly reconciliation check**: run a weekly validation transform that:

1. Selects 1,000 random device-window pairs from the latest day
2. Reads feature values from the offline store (`device_features` Delta Lake)
3. Reads the same features from the online serving path (Ontology properties or model-input transform output)
4. Compares values pair-wise with a tolerance of 1e-6 for floating-point features
5. Alerts the data engineering team if more than 0.1% of comparisons exceed tolerance

This reconciliation catches transform bugs, schema drift, and numerical inconsistencies before they affect model performance.

### Time Series as Complementary Access Pattern

The Ontology also serves raw sensor history via Time Series properties on the Device object (see [Ontology Design — Time Series Properties](../04-palantir/ontology-design.md)). Time Series properties serve raw 1-minute readings for per-device drill-down and trend visualization. Pre-hydrated scalar properties (described above) serve current-state summaries for fleet overview. The feature store's windowed aggregations sit between these two granularities — more processed than raw Time Series, less summarized than scalar properties.

## Point-in-Time Correctness

### Why It Matters

When training an anomaly detection model, each training example consists of features from a specific window and the corresponding label (or pseudo-label from a later time period). If the features include any information from after the window ended, the model learns to "cheat" — it uses future data that won't be available at inference time. This is called **data leakage**.

Example of leakage: training a model on Monday's features but including a feature that was recomputed on Tuesday with late-arriving data from Monday. The model might learn that "recomputed features look different" rather than learning the actual anomaly patterns.

### How We Prevent Leakage

1. **Features only look backward**: every feature is computed from readings within the window boundaries (`window_start` to `window_end`). No feature uses data from after `window_end`. The slope computation in [Time-Domain](./time-domain.md) uses `minutes_offset` relative to `window_start`, not absolute time, so it's bounded to the window.

2. **Append-only offline store**: once a window's features are written, they are not updated (for the append-mode strategy). A model training query that reads `window_start = 2026-02-14T10:00:00` always gets the same feature values, regardless of when the query runs.

3. **Model training reads by window boundaries**: the model-input transform ([Contract 4](../05-architecture/data-contracts.md)) selects features by `window_start` and `window_end`, not by "latest available." This ensures the training set uses features as they would have been available at inference time.

4. **Recomputed windows are versioned**: if Strategy 2 (snapshot mode, see [Windowing — Late Data](./windowing.md)) is used to recompute recent windows with late data, the recomputed rows overwrite the originals. Model training should use a snapshot timestamp to read features as of a specific point in time — Delta Lake's time travel (via Foundry's transaction history) supports this.

### Point-in-Time Join Pattern

When creating training datasets that combine features with labels from a later time period:

```python
# Training set: features from window W, labels from W+1 or later
# CORRECT — uses window boundaries for temporal alignment
training_data = (
    device_features.alias("features")
    .join(
        labels.alias("labels"),
        (F.col("features.device_id") == F.col("labels.device_id"))
        & (F.col("features.window_end") <= F.col("labels.event_time"))
        & (F.col("features.window_end") >= F.col("labels.event_time") - F.expr("INTERVAL 1 HOUR"))
    )
)

# WRONG — uses processing timestamps, which leak information
# training_data = features.join(labels, ...).filter(features.processed_at < labels.created_at)
```

Join on `window_end <= event_time`, not on processing timestamps. Processing timestamps vary with pipeline delays; window boundaries are deterministic.

## Backfill

### When Backfill Is Needed

A new feature column is added to the `device_features` schema — for example, `vibration_compressor_p95` didn't exist in v1 but is added in v2. All historical windows need this new feature computed so that model training has a complete feature matrix.

### How to Backfill

1. **Add the column to the feature transform**: update the `@transform_df` to compute the new feature alongside existing ones.

2. **Trigger a full recompute**: in Foundry, [force a non-incremental build](../04-palantir/transform-patterns.md) of the feature transform. This reprocesses the full `clean_sensor_readings` history and writes a complete `device_features` dataset with the new column.

3. **Cost estimation**: at 4.4B rows/year, a full recompute is expensive. Plan for:
   - Compute cost: hours of Spark cluster time depending on the profile
   - Storage: the new dataset version is a full rewrite, temporarily doubling storage until old transactions are compacted
   - Pipeline disruption: downstream transforms (model training, scoring) should be paused during backfill to avoid reading partially-backfilled data

4. **Incremental backfill (alternative)**: instead of a full recompute, run the new feature computation on historical data in date-range chunks (e.g., one month at a time). This spreads compute and reduces peak resource usage, but requires careful orchestration to avoid gaps.

### Avoiding Full Backfill

For features that don't affect critical models, consider:
- Allowing the new column to be null for historical windows (models handle nulls via imputation or exclusion)
- Computing the new feature only from the backfill date forward
- Running backfill only for the time range the model training pipeline needs (e.g., last 90 days)

Full backfill should be reserved for features that are expected to significantly improve model performance and that models need across the full training history.

## Feature Versioning

### Schema Evolution

The `device_features` schema will evolve as new features are added, existing features are refined, or obsolete features are removed. The [schema evolution strategy](../05-architecture/data-contracts.md) governs how changes propagate:

- **Adding a column**: add as nullable. Existing rows have null for the new column until backfilled. No downstream breakage — models that don't use the new feature ignore it.
- **Removing a column**: deprecate first (stop computing but keep the column as null), then remove after all consumers have updated. Never remove a column that active models depend on.
- **Changing a computation**: change the transform logic. New windows reflect the updated computation. Historical windows retain the old values unless backfilled. Record the change date in documentation.

### Version Tracking

Each feature should have metadata that records when it was added and how its computation has changed:

| Feature | Added | Description | Change Log |
|---------|-------|-------------|------------|
| `temp_evaporator_mean` | v1 | Mean evaporator temperature in window | — |
| `vibration_compressor_p95` | v1 | 95th percentile vibration | — |
| `pressure_ratio` | v1 | `pressure_high_mean / pressure_low_mean` | — |

This metadata lives in documentation (this file and [data contracts](../05-architecture/data-contracts.md)), not in the dataset itself. Foundry's dataset lineage provides the authoritative history of when transforms changed.

### Feature Statistics as Model Artifact

When a model is trained, the feature statistics from the training set must be saved as part of the model artifact. This serves two purposes: drift detection baseline and debugging out-of-distribution inputs at inference time.

**What to save per feature**:

| Statistic | Purpose |
|-----------|---------|
| Mean | Drift detection baseline, z-score scaling parameter |
| Std | Drift detection baseline, z-score scaling parameter |
| Min / Max | Detect out-of-range inputs at inference |
| p5 / p25 / p50 / p75 / p95 | Distribution shape reference for PSI computation |
| Null fraction | Baseline for data quality monitoring |

**What to save globally**:

- **Training time range**: `min(window_start)` to `max(window_end)` of the training set. Enables auditing which data the model learned from.
- **Device count**: number of distinct `device_id` values in the training set. Documents fleet coverage.
- **Feature list**: ordered list of feature names matching the model's input schema ([Contract 4](../05-architecture/data-contracts.md)).

Store this as a JSON file alongside the serialized model in Foundry's model artifact storage. The monitoring system reads this reference snapshot to compute PSI and other drift metrics. When a new model is trained, a new reference snapshot is generated automatically.

### Model-Feature Compatibility

Each trained model records the feature list it was trained on (see [Contract 4](../05-architecture/data-contracts.md): `feature_names` in model input). When the feature schema changes:

- Models trained before the change continue to work (they select their known features from the wider schema)
- New models can use the new features after retraining
- Feature removal requires retraining any model that depends on the removed feature

This decoupling — models declare their feature dependencies, the feature store provides a superset — prevents tight coupling between feature engineering and model lifecycle.

## Freshness Monitoring

### What to Monitor

Feature staleness directly impacts anomaly detection quality. A model scoring features from 6 hours ago might miss a compressor failure that happened 30 minutes ago. Monitor:

| Metric | Target | Alert If |
|--------|--------|----------|
| Latest `window_end` in `device_features` | Within 30 min of now | > 45 min behind current time |
| Device coverage per window | ≥ 99% of active devices | < 95% of devices have features for the latest window |
| Pre-hydration lag (Ontology) | Within 1 hour | > 90 min since last Device property update |
| Backfill completeness | 100% for backfill-in-progress columns | Any date gaps in backfilled feature |

### How to Monitor in Foundry

**Dataset expectations**: Foundry expectations can validate freshness by checking that the maximum `window_end` in the latest transaction is within an acceptable range. This runs as part of the transform and fails the build if violated. See [ADR-001](../05-architecture/adr-001-foundry-native.md) for Foundry-native monitoring.

```python
# Conceptual expectation: latest window_end should be recent
@transform_df(
    Output("/Company/pipelines/refrigeration/features/device_features"),
    clean=Input("/Company/pipelines/refrigeration/clean/clean_sensor_readings"),
)
@expect("latest_window_is_fresh", "max(window_end) >= current_timestamp() - interval 45 minutes")
def compute_features(clean):
    # ... feature computation
```

**Pipeline SLAs**: configure Foundry pipeline SLAs on the `device_features` dataset. If the dataset doesn't receive a new transaction within the expected interval (e.g., 30 minutes for 15-min windows), Foundry triggers an alert to the data engineering team.

**Device coverage monitoring**: a separate monitoring transform that counts distinct `device_id` values in the latest window and compares against the active device count from the device registry. Coverage drop signals either a transform failure or a mass connectivity event.

### Alerting on Stale Features

Stale feature alerts should go to the data engineering team, not the maintenance operators who receive equipment alerts. A stale feature is a pipeline problem, not an equipment problem. However, downstream scoring and alerting should indicate reduced confidence when features are stale — the `sensor_completeness` and `reading_count` fields already serve this purpose at the individual-device level.

## Feature Distribution Monitoring

Freshness monitoring (above) catches pipeline failures — features stop updating. Distribution monitoring catches a subtler problem: features continue updating but their statistical properties shift, causing trained models to operate on data that no longer resembles the training set.

### What to Track

For each numeric feature column in `device_features`, compute the following statistics daily across the active fleet:

| Statistic | What It Catches |
|-----------|-----------------|
| Mean | Fleet-wide baseline shift (e.g., seasonal temperature change) |
| Std | Variance change (fleet becoming more or less homogeneous) |
| p5 / p50 / p95 | Shape changes beyond mean and std (skewness, heavy tails) |
| Null fraction | Upstream data quality degradation or sensor dropout |
| PSI (Population Stability Index) | Overall distributional shift, binned comparison against reference |

### Reference Distribution

The reference distribution is a snapshot of per-feature statistics from the model's training set, stored with the model artifact (see [Feature Statistics as Model Artifact](#feature-statistics-as-model-artifact) above). When a new model is deployed, its reference snapshot becomes the baseline for drift detection. Multiple models may have different reference snapshots if trained on different time periods.

### Drift Detection Thresholds

PSI thresholds, calibrated for this domain:

| PSI Range | Interpretation | Action |
|-----------|---------------|--------|
| < 0.1 | No significant drift | Continue monitoring |
| 0.1 – 0.25 | Moderate drift | Investigate — is it seasonal, fleet composition, or degradation? |
| > 0.25 | Significant drift | Trigger retraining evaluation with ML Scientist |

These thresholds apply per feature. When 5+ features simultaneously cross the moderate threshold, treat it as a significant fleet-wide event even if no individual feature crosses 0.25.

### Null Fraction Escalation

A rising null fraction for a feature that was historically non-null (e.g., `pressure_ratio` going from 1% null to 15% null) signals upstream data quality degradation — likely a sensor failure pattern or an ingestion pipeline issue. This is not a model retraining trigger; it is a data quality issue that should be escalated to the Data Engineer for root-cause investigation. The monitoring transform should file an automated alert when any feature's null fraction exceeds its training-set baseline by more than 5 percentage points.

### Implementation

Implement as a dedicated monitoring transform in Foundry:

1. Reads the latest 1-day window partition from `device_features`
2. Computes per-feature statistics (mean, std, percentiles, null fraction)
3. Loads the reference snapshot from the active model artifact
4. Computes PSI per feature against the reference
5. Writes results to a `feature_drift_monitoring` dataset
6. Foundry expectations on the monitoring dataset raise alerts when thresholds are crossed

This transform runs daily and is lightweight (aggregating fleet-level statistics, not processing individual readings). See [Monitoring](../03-production/monitoring.md) for how feature drift integrates with the broader operational monitoring framework.

## Related Documents

- [Data Contracts — Contract 3](../05-architecture/data-contracts.md) — authoritative `device_features` schema
- [Data Contracts — Contract 4](../05-architecture/data-contracts.md) — model input schema and feature selection
- [Windowing Strategies](./windowing.md) — how windows are defined and computed before reaching the feature store
- [Time-Domain Features](./time-domain.md) — features stored in the offline store
- [Cross-Sensor Features](./cross-sensor.md) — derived features stored in the offline store
- [Ontology Design](../04-palantir/ontology-design.md) — Device object and pre-hydrated properties for online serving
- [Transform Patterns](../04-palantir/transform-patterns.md) — incremental transforms, backfill, and scheduling
- [System Overview](../05-architecture/system-overview.md) — where the feature store sits in the overall architecture
