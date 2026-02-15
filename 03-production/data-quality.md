# Data Quality Monitoring and Enforcement

## Why Data Quality Is the First Production Concern

A model can be perfect and still produce garbage if its input is wrong. With 100K devices producing ~14M readings per minute, there are countless ways data can go bad: sensors fail silently, device clocks drift, firmware updates change message formats, network outages create gaps. If we don't catch data problems before they reach the model, the model's anomaly scores become meaningless — you can't distinguish "the compressor is failing" from "the temperature sensor is stuck at -40°C."

Every data quality check maps to a specific [data contract](../05-architecture/data-contracts.md) boundary. If a contract is violated, the pipeline should stop propagating bad data downstream rather than silently producing wrong results.

## Dataset Expectations as the Primary Enforcement Mechanism

Foundry dataset expectations are assertions that run as part of a Transform build. If an expectation fails, the Transform's output transaction is rolled back — no bad data is written. This is the enforcement mechanism for all contract boundaries.

### Why Expectations, Not External Validators

- **Atomic with the Transform**: expectations run inside the Transform execution. There's no window between "data was written" and "data was validated."
- **Foundry-native**: no external tools to deploy, monitor, or keep in sync. Expectations are versioned alongside the Transform code in Code Repositories.
- **Visible**: failed expectations appear in the pipeline health dashboard, triggering alerts via the same mechanism as pipeline SLAs (see [Transform Patterns](../04-palantir/transform-patterns.md#scheduling-and-slas)).

### Expectation Anatomy

```python
from transforms.api import transform_df, Input, Output
from transforms.expectations import expectation, PrimaryKey, SchemaExpectation

@expectation(PrimaryKey(["event_id"]))
@expectation(SchemaExpectation(
    strict=True,  # fail if unexpected columns appear
))
@transform_df(
    Output("/Company/pipelines/refrigeration/clean/sensor_readings"),
    raw=Input("/Company/pipelines/refrigeration/raw/sensor_readings"),
)
def clean_sensor_readings(raw):
    # Transform logic here
    ...
```

`strict=True` is critical. Without it, an upstream schema change (new column added by firmware update) silently passes through. With it, the Transform fails and forces an engineer to explicitly handle the new column.

## Expectations by Contract Boundary

Each pipeline boundary in the [data contracts](../05-architecture/data-contracts.md) has a specific set of expectations. These are not optional — they gate data flow between stages.

### Contract 1 → 2: Raw Ingestion → Clean Dataset

This boundary catches problems introduced by devices, networks, and IoT Hub.

| Expectation | What It Checks | Failure Mode It Prevents |
|---|---|---|
| Primary key on `event_id` | No duplicate events after dedup | Double-counted readings inflating aggregations |
| `device_id` in device registry | Unknown devices routed to quarantine | Spoofed or decommissioned devices polluting fleet data |
| `timestamp_utc` not null | UTC normalization succeeded | Null timestamps breaking time-windowed features |
| `timestamp_utc` within ±24h of `scored_at` wall clock | Clock drift correction is plausible | Future-dated or ancient readings distorting windows |
| All sensor values within [range constraints](../05-architecture/data-contracts.md#contract-2-landing-zone--clean-dataset) or null | Out-of-range values nullified and flagged | Physically impossible sensor values treated as real |
| `quality_flags` is non-null array | Every row has a quality assessment | Downstream consumers can't distinguish clean from dirty |
| `sensor_null_count` matches actual null count | Metadata consistency | Completeness filtering uses wrong values |

```python
from transforms.expectations import expectation, Check

@expectation(Check(
    lambda df: df.filter(
        (F.col("temperature_evaporator").isNotNull()) &
        ((F.col("temperature_evaporator") < -60) | (F.col("temperature_evaporator") > 30))
    ).count() == 0,
    "temperature_evaporator must be in [-60, 30]°C or null"
))
```

### Contract 2 → 3: Clean Dataset → Feature Store

This boundary ensures feature computation is correct and complete.

| Expectation | What It Checks | Failure Mode It Prevents |
|---|---|---|
| Primary key on (`device_id`, `window_start`, `window_duration_minutes`) | One feature row per device per window | Duplicate features causing double-scoring |
| `window_start` < `window_end` | Window boundaries are ordered | Inverted windows producing nonsensical aggregations |
| `window_duration_minutes` in (15, 60, 1440) | Only valid window sizes | Unexpected window sizes confusing downstream consumers |
| `reading_count` > 0 | No empty windows | Aggregation on zero readings producing NaN/null features |
| `sensor_completeness` between 0.0 and 1.0 | Fraction is valid | Completeness filtering produces wrong inclusion/exclusion |
| Feature value ranges (soft checks) | Feature values are physically plausible | Silent computation bugs producing extreme values |

**Soft vs hard expectations**: range checks on raw sensor values (Contract 2) are hard — violations are physically impossible. Range checks on derived features (Contract 3) are soft — a `pressure_ratio` of 15.0 is unusual but not impossible. Soft expectations log warnings instead of failing the build.

> **Deriving soft thresholds from training data**: don't guess soft thresholds for derived features. Compute the 0.1st and 99.9th percentiles of each feature from the training dataset and use those as the soft expectation bounds. This ensures thresholds reflect the actual data distribution the model was trained on, not engineering intuition about what looks "reasonable." Store these percentile bounds alongside the model artifact and update them on every retrain. If a feature value falls outside the training-derived bounds it warrants a warning, but only values far outside (>10× the interquartile range) should trigger a hard failure.

### Contract 3 → 4: Feature Store → Model Input

| Expectation | What It Checks | Failure Mode It Prevents |
|---|---|---|
| No nulls in `feature_vector` | Null imputation or exclusion happened | Model crashes or produces NaN scores |
| `feature_names` matches model's training feature list | Feature alignment | Model scores with wrong feature assignments (silent corruption) |
| `sensor_completeness` ≥ configured threshold (e.g., 0.5) | Rows with too many missing sensors are excluded | Low-confidence scores treated as high-confidence |

### Contract 4 → 5: Model Input → Model Scores

| Expectation | What It Checks | Failure Mode It Prevents |
|---|---|---|
| `anomaly_score` in [0.0, 1.0] | Score normalization is correct | Raw model output leaking into downstream logic |
| `anomaly_flag` consistent with `threshold_used` | Flag matches score vs threshold comparison | Flag and score contradicting each other |
| All `model_id` values are known model identifiers | No unexpected model outputs | Test or debug models leaking into production |
| `scored_at` within last 2 hours | Scores are fresh | Stale scores treated as current |

### Contract 5 → 6: Model Scores → Device Alerts

| Expectation | What It Checks | Failure Mode It Prevents |
|---|---|---|
| `severity` matches [severity rules](../05-architecture/data-contracts.md#severity-rules) | Alert severity derived correctly | LOW alert when score is 0.95 (should be CRITICAL) |
| `suppressed` flag consistent with dedup logic | 4-hour dedup window applied correctly | Alert storms from a single sustained anomaly |
| `alert_type` in valid enum values | Only known alert types | Typos or test values in production alerts |
| `model_id` references a production model | Alerts trace to real models | Orphan alerts with no provenance |

## Sensor Dropout Detection

A device that stops reporting entirely is easy to detect — no readings at all. The harder problem is partial dropout: a device reports 10 of 14 sensors, or reports at half the expected frequency.

### Types of Dropout

| Type | Signal | Risk |
|---|---|---|
| Full device dropout | No readings for > 5 minutes (expected: ~1/min) | Device is offline — alert ops, don't score |
| Sensor subset dropout | `sensor_null_count` increases; specific sensors consistently null | Sensor hardware failure — features computed on partial data may be misleading |
| Frequency degradation | `reading_count` in 15-min window drops below 10 (expected: ~15) | Network instability or firmware issue — aggregations based on fewer samples have higher variance |
| Intermittent dropout | Readings appear and disappear unpredictably | Loose connectors, power cycling — creates gaps in time series |

### Detection Logic

Dropout detection runs as a scheduled Transform that compares device activity against expected baselines:

```python
# Pseudocode — runs hourly
active_devices = device_registry.filter("status = 'active'")
recent_readings = clean_readings.filter("timestamp_utc > now() - interval 1 hour")

device_stats = recent_readings.groupBy("device_id").agg(
    F.count("*").alias("reading_count"),
    F.avg("sensor_null_count").alias("avg_null_sensors"),
    F.countDistinct(
        F.when(F.col("temperature_evaporator").isNotNull(), 1)
    ).alias("has_evaporator"),
    # ... repeat for all 14 sensors
)

# Flag devices with dropout
dropout_flags = active_devices.join(device_stats, "device_id", "left").withColumn(
    "dropout_type",
    F.when(F.col("reading_count").isNull(), "FULL_DROPOUT")
     .when(F.col("reading_count") < 30, "FREQUENCY_DEGRADATION")  # expected ~60
     .when(F.col("avg_null_sensors") > 4, "SENSOR_SUBSET_DROPOUT")
     .otherwise("HEALTHY")
)
```

Dropout flags feed into the [Device object's](../04-palantir/ontology-design.md#device) `health_status` property. A device in `FULL_DROPOUT` state should not be scored — its model scores would be stale or missing features.

## The quality_flags System

Every row in `clean_sensor_readings` carries a `quality_flags` array. This system distinguishes "the data is clean" from "the data has known issues but is usable" — a critical distinction for downstream consumers.

### Flag Definitions

| Flag | Meaning | How It's Set | Downstream Impact |
|---|---|---|---|
| `RANGE_VIOLATION` | At least one sensor value was outside its [physical range](../05-architecture/data-contracts.md#contract-2-landing-zone--clean-dataset) and nullified | Cleansing transform range check | Features computed on remaining non-null sensors; model may see lower `sensor_completeness` |
| `CLOCK_DRIFT` | Device timestamp was >24h off from server timestamp; `timestamp_utc` uses server time instead | Clock drift detection in cleansing transform | Time-windowed features may be slightly misaligned; generally safe but logged for investigation |
| `DUPLICATE_REMOVED` | This reading was originally a duplicate (same `event_id`); the kept copy carries this flag | Dedup logic in cleansing transform | No downstream impact on the kept row, but useful for auditing dedup rates |
| `SENSOR_GAP` | Sensor columns that the device previously reported are suddenly null — possible hardware fault or firmware regression | Per-device comparison of current sensor null pattern vs historical reporting pattern | Features computed on remaining sensors may have lower `sensor_completeness`; investigate if sustained across multiple readings |

> **Distinction from temporal gaps:** `SENSOR_GAP` flags missing *sensor columns* within a reading that arrives on time. Temporal reading gaps (no readings at all for >5 minutes) are a separate concern — detected via timestamp analysis in the [dropout detection logic](#detection-logic) and surfaced through `reading_count` in feature windows. Consider flagging temporal gaps as `READING_GAP` if a distinct quality flag is needed.
| `DEVICE_UNKNOWN` | `device_id` not found in device registry | Device registry lookup in cleansing transform | Row routed to quarantine dataset, not to clean dataset. Flag exists for quarantine analysis. |

### Using quality_flags Downstream

Feature engineering should not blindly aggregate flagged rows. The recommended pattern:

```python
# Count quality issues per window for monitoring
quality_flag_count = readings_in_window.filter(
    F.size("quality_flags") > 0
).count()

# Include flagged rows in feature computation (they've already been cleaned)
# but track the flag count as a feature itself
features = features.withColumn("quality_flag_count", F.lit(quality_flag_count))
```

The `quality_flag_count` feature column in [Contract 3](../05-architecture/data-contracts.md#contract-3-clean-dataset--feature-store-offline) lets models learn whether quality issues correlate with anomaly patterns. A device with many `RANGE_VIOLATION` flags may genuinely be failing (real out-of-range values from a dying sensor) — the model should see this signal, not have it silently removed.

## Data Freshness Monitoring

Stale data is invisible unless you look for it. A pipeline that stopped running 6 hours ago looks exactly like a healthy pipeline with no new data — until someone checks.

### Freshness SLAs by Contract

These freshness targets come from the [data contracts](../05-architecture/data-contracts.md):

| Dataset | Freshness SLA | Check Method |
|---|---|---|
| `raw_sensor_readings` | Within **2 minutes** of IoT Hub ingestion | Compare latest `timestamp_ingested` to wall clock |
| `clean_sensor_readings` | Within **15 minutes** of landing zone append | Compare latest transaction timestamp to latest raw transaction |
| `device_features` | Within **30 minutes** of clean dataset update | Compare latest `window_end` to expected schedule |
| `model_scores` (batch) | Within **1 hour** of feature store update | Compare latest `scored_at` to latest feature timestamp |
| `model_scores` (streaming) | Within **2 minutes** of event ingestion | Compare streaming output lag to input |
| `device_alerts` (batch) | Within **5 minutes** of scoring completion | Compare latest `alert_created_at` to latest `scored_at` |
| `device_alerts` (streaming) | Within **1 minute** of scoring | Alert latency from streaming path |

### Implementing Freshness Checks

Foundry pipeline SLAs are the primary mechanism. Configure SLAs on critical output datasets:

```
Dataset: /Company/pipelines/refrigeration/scores/batch_model_scores
SLA: Latest transaction must be < 2 hours old
Alert: Email + Slack notification to #data-platform-alerts
Escalation: If > 4 hours stale, page on-call engineer
```

SLAs are configured in Foundry's pipeline monitoring UI, not in Transform code. They check the transaction timestamp of the dataset, which Foundry updates automatically when a Transform writes successfully.

**Gotcha**: a Transform that runs successfully but produces zero rows still updates the transaction timestamp. The dataset looks "fresh" but contains no new data. To catch this, add a secondary check: the row count of the latest transaction should be > 0 for datasets that always produce data (like hourly batch scores for 100K active devices).

## Training-Serving Skew Detection

Pure data quality checks (range validation, null checks, schema enforcement) catch data that is *broken*. Training-serving skew detection catches data that is *valid but different from what the model was trained on*. A temperature sensor reading of 15°C passes every range check but may indicate a completely different operating regime than the model saw during training.

### What to Check

| Check | Computation | Alert Threshold | What It Catches |
|---|---|---|---|
| Feature mean shift | \|current_mean − training_mean\| / training_stddev | > 2.0 (i.e., current mean is >2× stddev from training mean) | Gradual operational changes that are individually valid but collectively shift the feature space away from training conditions |
| Feature range expansion | Fraction of current values outside [training_min, training_max] | > 10% of values fall beyond training range | New operating regimes the model never saw — extrapolation risk |
| Feature correlation stability | \|current_corr(f_i, f_j) − training_corr(f_i, f_j)\| for key feature pairs | Change > 0.3 for any monitored pair | Broken cross-sensor relationships (e.g., compressor current no longer correlates with evaporator temp — suggests sensor or mechanical change) |
| Null rate divergence | \|current_null_rate − training_null_rate\| per feature | > 5 percentage points | Sensor dropout patterns changed since training — imputation logic may no longer be appropriate |

### Storing Training-Time Statistics

Training-time statistics must be stored as a versioned dataset alongside the model artifact so that serving-time comparison is always against the correct reference:

```python
@transform_df(
    Output("/Company/pipelines/refrigeration/models/training_statistics"),
    training_features=Input("/Company/pipelines/refrigeration/features/training_snapshot"),
)
def compute_training_statistics(training_features):
    """Compute and store feature statistics from the training dataset.
    Run once per training cycle. Output is versioned with the model."""
    stats = []
    feature_cols = [c for c in training_features.columns
                    if c not in ("device_id", "window_start", "window_end",
                                 "window_duration_minutes", "sensor_completeness")]

    for col_name in feature_cols:
        col_stats = training_features.agg(
            F.mean(col_name).alias("mean"),
            F.stddev(col_name).alias("stddev"),
            F.min(col_name).alias("min_val"),
            F.max(col_name).alias("max_val"),
            F.expr(f"percentile({col_name}, 0.001)").alias("p0_1"),
            F.expr(f"percentile({col_name}, 0.999)").alias("p99_9"),
            (F.count(F.when(F.col(col_name).isNull(), 1)) / F.count("*")).alias("null_rate"),
        ).collect()[0]

        stats.append({
            "feature_name": col_name,
            "training_mean": col_stats["mean"],
            "training_stddev": col_stats["stddev"],
            "training_min": col_stats["min_val"],
            "training_max": col_stats["max_val"],
            "training_p0_1": col_stats["p0_1"],
            "training_p99_9": col_stats["p99_9"],
            "training_null_rate": col_stats["null_rate"],
        })

    return spark.createDataFrame(stats)
```

Version this dataset with the model: when model `isolation_forest_v3` is trained, the corresponding `training_statistics` snapshot is tagged `v3`. The serving-time skew detection Transform reads the statistics version matching the currently deployed model.

### Why This Matters

Training-serving skew is the most common source of silent model degradation. The model doesn't crash, scores stay in [0, 1], and no data quality check fires — but the model is being applied to data it wasn't trained on. Common causes in IoT:

- Seasonal shifts (winter → summer) change ambient temperature distributions
- Fleet expansion adds devices with different baseline operating characteristics
- Firmware updates change sensor calibration or reporting frequency
- Maintenance campaigns fix issues the model was trained to detect, shifting the "normal" baseline

## Data Health Dashboards

Build dashboards in Foundry Workshop that surface quality metrics to different audiences.

### Operations Dashboard

For the on-call engineer — shows pipeline health and data flow.

| Widget | Data Source | Alert Condition |
|---|---|---|
| Pipeline freshness heatmap | Freshness checks across all datasets | Any dataset > 2× its SLA |
| Throughput trend | Row counts per hour for each pipeline stage | Drop > 50% vs previous day same hour |
| Quality flag breakdown | Aggregated `quality_flags` counts per hour | `RANGE_VIOLATION` rate > 5% of readings |
| Device dropout count | Dropout detection Transform output | > 100 devices in `FULL_DROPOUT` simultaneously |
| Quarantine volume | Row counts in `quarantine_sensor_readings` | > 1% of raw readings quarantined |

### Data Science Dashboard

For the ML team — shows data distribution and model input health.

| Widget | Data Source | Purpose |
|---|---|---|
| Feature distribution histograms | `device_features` dataset | Spot distribution shifts before they become model drift |
| Sensor completeness trend | `sensor_completeness` aggregated by day | Detect gradual sensor fleet degradation |
| Quality flag trends by flag type | `quality_flag_count` over time | Identify emerging data issues |
| Null rate per sensor per device model | Null counts segmented by equipment model | Equipment model–specific sensor failures |
| Reading count distribution | `reading_count` histogram per window | Detect fleet-wide frequency degradation |

## Alerting on Quality Degradation

Quality alerts are separate from anomaly alerts. A data quality alert means "the pipeline or data source has a problem." An anomaly alert means "the refrigeration equipment has a problem." Conflating them causes confusion — the on-call engineer doesn't know whether to debug the pipeline or dispatch a technician.

### Alert Routing

| Alert Type | Destination | Example |
|---|---|---|
| Data quality alert | `#data-platform-alerts` Slack channel + on-call data engineer | "15% of readings have RANGE_VIOLATION in the last hour — possible sensor calibration issue" |
| Pipeline staleness alert | Same + auto-create JIRA ticket | "`device_features` dataset is 45 minutes stale (SLA: 30 minutes)" |
| Anomaly alert (from model) | `#refrigeration-ops` + site maintenance team | "Device REF-US-W-00042371 compressor vibration anomaly — score 0.87" |

### Severity Mapping for Data Quality Alerts

| Condition | Severity | Action |
|---|---|---|
| Single dataset 1–2× past SLA | LOW | Log to dashboard, no notification |
| Single dataset > 2× past SLA | MEDIUM | Slack notification to data platform channel |
| Multiple datasets past SLA or > 500 devices in dropout | HIGH | Page on-call data engineer |
| Feature store stale for > 2 hours (model scores become unreliable) | CRITICAL | Page on-call + pause batch scoring + notify ops team |

## Historical Quality Trends and Seasonal Patterns

Data quality is not uniform. Some problems are seasonal, some correlate with external events, and some creep up gradually.

### Known Seasonal Patterns

| Pattern | Season | Cause | Mitigation |
|---|---|---|---|
| Humidity sensor `RANGE_VIOLATION` spikes | Summer (Jun–Aug in northern hemisphere) | Condensation saturates humidity sensors beyond calibrated range | Widen humidity range validation to 0–105 %RH in summer, or accept higher flag rates seasonally |
| Temperature sensor drift | Extreme cold (Dec–Feb) | Ambient sensors in outdoor units read below expected range | Region-specific range validation; colder regions get wider ambient temp bounds |
| Increased `SENSOR_GAP` during holidays | Late Dec, early Jan | Reduced maintenance staff → delayed repairs on connectivity issues | Raise dropout alert thresholds during known low-staffing periods |
| Device dropout after firmware updates | Following OTA rollouts | New firmware changes reporting format or interval | Pre-validate firmware message format in staging; alert on dropout rate spikes within 48h of rollout |

### Tracking Trends

Maintain a `data_quality_daily_summary` dataset that aggregates per-day:

- Total readings count
- Readings per quality flag type
- Devices in each dropout state
- Sensor null rates per sensor type
- Quarantine rate

This dataset feeds trend dashboards and enables year-over-year comparisons. When a quality metric looks anomalous, checking the same week last year often explains it.

## Cross-References

- [Data Contracts](../05-architecture/data-contracts.md) — the schemas and SLAs enforced by these expectations
- [Transform Patterns](../04-palantir/transform-patterns.md) — how expectations integrate with `@transform_df` decorators
- [Ontology Design](../04-palantir/ontology-design.md) — Device health properties that consume quality signals
- [Monitoring](./monitoring.md) — operational monitoring that consumes data quality metrics
- [Testing](./testing.md) — testing that quality expectations actually catch bad data
