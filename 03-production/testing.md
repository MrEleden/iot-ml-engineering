# Testing Strategy for ML Pipelines in Foundry

## Why Testing ML Pipelines Is Different

Traditional software tests assert that a function returns the correct output for a given input. ML pipeline tests must also assert:

- **Schema stability**: the output matches the [data contract](../05-architecture/data-contracts.md) even when the input data changes.
- **Statistical properties**: the model produces scores in the right distribution, not just the right type.
- **Temporal correctness**: windowed features don't leak future data into past windows.
- **Edge case resilience**: the pipeline doesn't crash when a device reports all nulls, a single sensor, or extreme values.

If a test doesn't catch a bug _before_ it reaches 100K devices, you discover it at 3 AM when the anomaly rate spikes to 40% and on-call gets paged.

## Unit Testing Foundry Transforms

### Test Harness Pattern for @transform_df

Foundry Transforms are decorated functions that receive PySpark DataFrames and return PySpark DataFrames. To unit test them, create a local Spark session, construct input DataFrames, call the underlying function, and assert on the output.

```python
import pytest
from pyspark.sql import SparkSession
from pyspark.sql import types as T
from myproject.transforms.cleansing import clean_sensor_readings

@pytest.fixture(scope="session")
def spark():
    return SparkSession.builder.master("local[2]").appName("test").getOrCreate()

def test_range_violation_flags_out_of_range_temperature(spark):
    """Temperature outside [-60, 30]°C should be nullified and flagged."""
    raw_schema = T.StructType([
        T.StructField("event_id", T.StringType()),
        T.StructField("device_id", T.StringType()),
        T.StructField("timestamp_device", T.TimestampType()),
        T.StructField("timestamp_ingested", T.TimestampType()),
        T.StructField("temperature_evaporator", T.DoubleType()),
        # ... other sensor columns
    ])

    raw_data = spark.createDataFrame([
        ("evt-001", "dev-001", ts("2026-01-15T10:00:00"), ts("2026-01-15T10:00:01"), 999.0),
    ], schema=raw_schema)

    result = clean_sensor_readings(raw_data)

    row = result.collect()[0]
    assert row["temperature_evaporator"] is None, "Out-of-range value should be nullified"
    assert "RANGE_VIOLATION" in row["quality_flags"], "Should flag RANGE_VIOLATION"
```

### Key Testing Rules

1. **Use `local[2]`** — not `local[*]`. Two cores expose parallelism bugs (wrong partition ordering, race conditions) without making tests slow. Tests should complete in under 60 seconds total.

2. **Test the function, not the decorator** — extract your transform logic into a plain function that receives DataFrames and returns DataFrames. Test that function. The `@transform_df` decorator is Foundry infrastructure you don't need to test.

   ```python
   # In your transform module
   def _clean_sensor_readings_logic(raw_df):
       """Pure business logic — no Foundry decorator."""
       return raw_df.filter(...).withColumn(...)

   @transform_df(
       Output("..."),
       raw=Input("..."),
   )
   def clean_sensor_readings(raw):
       return _clean_sensor_readings_logic(raw)

   # In your test module
   def test_cleansing(spark):
       result = _clean_sensor_readings_logic(input_df)
       # assertions...
   ```

3. **Never mock SparkSession** — use a real local session. Mocking Spark hides serialization bugs, type coercion surprises, and null handling differences between Python and JVM.

4. **Test null propagation explicitly** — for every sensor column, test: what happens when it's null in the input? PySpark's null handling is different from Python's `None`, and `F.when` conditions with nulls can silently produce wrong results.

### Testing @transform (Non-DF) Patterns

For Transforms that use `@transform` with explicit `TransformInput`/`TransformOutput` (see [Transform Patterns](../04-palantir/transform-patterns.md#transform)), create a lightweight mock for the output:

```python
class MockTransformOutput:
    def __init__(self):
        self.written_df = None
        self.partition_cols = None

    def write_dataframe(self, df, partition_cols=None):
        self.written_df = df
        self.partition_cols = partition_cols

def test_partitioned_feature_output(spark):
    input_df = spark.createDataFrame([...])
    output = MockTransformOutput()
    compute_partitioned_features_logic(input_df, output)
    assert output.partition_cols == ["date", "hour"]
    assert output.written_df.count() > 0
```

## Integration Testing: End-to-End Pipeline

Unit tests verify individual Transforms. Integration tests verify the chain: raw → clean → features → model input → model scores → alerts. Run the full pipeline on synthetic data and assert that the final alerts make sense given the input.

### Integration Test Structure

```python
def test_full_pipeline_anomalous_device(spark, synthetic_data_generator):
    """A device with extreme compressor vibration should produce a HIGH alert."""
    raw = synthetic_data_generator.generate(
        device_count=10,
        anomalous_devices={"dev-003": "compressor_vibration_spike"},
        duration_hours=2,
    )

    clean = _clean_sensor_readings_logic(raw)
    features = _compute_features_logic(clean, window_minutes=15)
    model_input = _prepare_model_input(features, model_id="isolation_forest_v2")
    scores = _score_with_model(model_input, model=load_test_model())
    alerts = _generate_alerts(scores)

    dev_003_alerts = alerts.filter(F.col("device_id") == "dev-003")
    assert dev_003_alerts.count() > 0, "Anomalous device should generate at least one alert"

    severity = dev_003_alerts.collect()[0]["severity"]
    assert severity in ("HIGH", "CRITICAL"), f"Expected HIGH or CRITICAL, got {severity}"
```

### What Integration Tests Catch That Unit Tests Don't

- **Schema mismatches between stages**: Transform A produces a column named `temp_evap_mean`, but Transform B expects `temp_evaporator_mean`. Unit tests of A and B individually pass; the chain fails.
- **Null handling across stages**: a null in raw data cascades through cleansing → features → model input. Does the entire chain handle it gracefully, or does it crash in the model adapter that doesn't expect nulls?
- **Window boundary alignment**: feature aggregation produces windows aligned to :00/:15/:30/:45. Does the scoring Transform correctly join features to windows?

## Synthetic Data Generation

Real production data is sensitive (device IDs, locations, usage patterns) and copyright-encumbered. Synthetic data lets us test without production data access and with controlled edge cases that rarely occur naturally.

### Refrigeration Sensor Data Generator

The generator produces physically realistic sensor data that respects inter-sensor correlations. Random noise on each sensor independently would produce data where evaporator temperature has no relationship to compressor current — a pattern no real refrigeration system exhibits.

```python
import numpy as np
import pandas as pd
from datetime import datetime, timedelta

class RefrigerationSyntheticGenerator:
    """Generate realistic synthetic sensor data for refrigeration devices."""

    # Baseline operating ranges (normal steady-state operation)
    BASELINES = {
        "temperature_evaporator": (-25.0, 3.0),     # mean, std
        "temperature_condenser": (40.0, 5.0),
        "temperature_ambient": (22.0, 4.0),
        "temperature_discharge": (70.0, 8.0),
        "temperature_suction": (-10.0, 3.0),
        "pressure_high_side": (1500.0, 150.0),
        "pressure_low_side": (200.0, 30.0),
        "current_compressor": (12.0, 2.0),
        "vibration_compressor": (3.0, 0.8),
        "humidity": (45.0, 10.0),
        "superheat": (8.0, 2.0),
        "subcooling": (5.0, 1.5),
    }

    # Cross-sensor correlations to enforce
    # When evaporator temp rises, compressor current rises (working harder)
    # When pressure_high rises, condenser temp rises
    # When superheat drops, suction temp drops
    CORRELATIONS = [
        ("temperature_evaporator", "current_compressor", 0.6),
        ("pressure_high_side", "temperature_condenser", 0.7),
        ("superheat", "temperature_suction", 0.5),
        ("current_compressor", "vibration_compressor", 0.4),
    ]

    def generate(
        self,
        device_count: int = 10,
        duration_hours: int = 24,
        readings_per_minute: int = 1,
        anomalous_devices: dict = None,
        null_rate: float = 0.02,
        seed: int = 42,
    ) -> pd.DataFrame:
        """
        Generate synthetic sensor readings.

        Args:
            device_count: number of devices
            duration_hours: how many hours of data to generate
            readings_per_minute: reading frequency per device
            anomalous_devices: dict of device_id → anomaly type
                Anomaly types: "compressor_vibration_spike",
                "refrigerant_leak", "temperature_drift",
                "sensor_failure", "intermittent_dropout"
            null_rate: fraction of sensor values randomly set to null
            seed: random seed for reproducibility
        """
        np.random.seed(seed)
        anomalous_devices = anomalous_devices or {}
        records = []

        for dev_idx in range(device_count):
            device_id = f"REF-TEST-{dev_idx:05d}"
            # Each device has slightly different baselines (manufacturing variation)
            device_baselines = {
                k: (v[0] + np.random.normal(0, v[1] * 0.1), v[1])
                for k, v in self.BASELINES.items()
            }

            timestamps = pd.date_range(
                start=datetime(2026, 1, 15, 0, 0, 0),
                periods=duration_hours * 60 * readings_per_minute,
                freq=f"{60 // readings_per_minute}s",
            )

            for ts in timestamps:
                reading = self._generate_reading(
                    device_id, ts, device_baselines, null_rate
                )

                # Apply anomaly pattern if this device is anomalous
                anomaly_type = anomalous_devices.get(device_id)
                if anomaly_type:
                    reading = self._apply_anomaly(
                        reading, anomaly_type, ts, timestamps[0]
                    )

                records.append(reading)

        return pd.DataFrame(records)

    def _generate_reading(self, device_id, ts, baselines, null_rate):
        reading = {
            "event_id": f"{device_id}_{ts.isoformat()}",
            "device_id": device_id,
            "timestamp_device": ts,
            "timestamp_ingested": ts + timedelta(seconds=np.random.randint(1, 10)),
        }

        # Generate correlated sensor values
        for sensor, (mean, std) in baselines.items():
            if np.random.random() < null_rate:
                reading[sensor] = None
            else:
                reading[sensor] = np.random.normal(mean, std)

        # Enforce correlations
        for s1, s2, strength in self.CORRELATIONS:
            if reading.get(s1) is not None and reading.get(s2) is not None:
                deviation = (reading[s1] - baselines[s1][0]) / baselines[s1][1]
                reading[s2] += deviation * strength * baselines[s2][1]

        # Boolean sensors
        reading["door_open_close"] = np.random.random() < 0.05  # 5% open
        reading["defrost_cycle"] = np.random.random() < 0.08   # 8% defrost

        return reading

    def _apply_anomaly(self, reading, anomaly_type, current_ts, start_ts):
        hours_elapsed = (current_ts - start_ts).total_seconds() / 3600

        if anomaly_type == "compressor_vibration_spike":
            # Sudden spike after 12 hours
            if hours_elapsed > 12 and reading.get("vibration_compressor"):
                reading["vibration_compressor"] *= 4.0
                reading["current_compressor"] = (
                    reading.get("current_compressor", 12) * 1.5
                )

        elif anomaly_type == "refrigerant_leak":
            # Gradual pressure drop and temperature rise
            if reading.get("pressure_low_side"):
                leak_factor = max(0.3, 1 - hours_elapsed * 0.03)
                reading["pressure_low_side"] *= leak_factor
            if reading.get("temperature_evaporator"):
                reading["temperature_evaporator"] += hours_elapsed * 0.5

        elif anomaly_type == "temperature_drift":
            # Slow temperature increase in evaporator
            if reading.get("temperature_evaporator"):
                reading["temperature_evaporator"] += hours_elapsed * 0.2

        elif anomaly_type == "sensor_failure":
            # Sensor stuck at fixed value after 6 hours
            if hours_elapsed > 6:
                reading["temperature_evaporator"] = -25.0  # stuck

        elif anomaly_type == "intermittent_dropout":
            # 30% of readings are fully null after 8 hours
            if hours_elapsed > 8 and np.random.random() < 0.3:
                for sensor in self.BASELINES:
                    reading[sensor] = None

        return reading
```

### Why Correlations Matter

If you generate each sensor independently, the test data doesn't exercise cross-sensor features like `current_per_temp_spread` or `pressure_ratio` (see [Cross-Sensor Features](../02-feature-engineering/cross-sensor.md)). A model tested on uncorrelated synthetic data may pass all tests but fail on real data where sensors are physically linked.

The correlations in the generator don't need to perfectly match real physics — they need to be non-zero so that cross-sensor features are non-trivial.

### Anomaly Scenarios

Each anomaly type maps to a known refrigeration failure mode:

| Anomaly Type | Failure Mode | Sensors Affected | Expected Detection |
|---|---|---|---|
| `compressor_vibration_spike` | Bearing wear or foreign object | `vibration_compressor`, `current_compressor` | High anomaly score from vibration/current features |
| `refrigerant_leak` | Refrigerant charge loss | `pressure_low_side`, `temperature_evaporator`, `superheat` | Gradual score increase as pressure drops and temp rises |
| `temperature_drift` | Evaporator icing or blocked airflow | `temperature_evaporator`, `defrost_cycle` | Moderate score increase; slope features trigger |
| `sensor_failure` | Sensor hardware stuck | Single sensor with zero variance | Model may not flag (value is "normal"); detected via variance = 0 |
| `intermittent_dropout` | Power or connectivity issues | All sensors | Low `sensor_completeness`, possible dropout detection before model scoring |

## Schema Tests

Schema tests validate that Transform outputs conform to [data contracts](../05-architecture/data-contracts.md). These are separate from dataset expectations (which run in production) — schema tests run in CI before code is merged.

```python
from myproject.schemas import CLEAN_SENSOR_SCHEMA, MODEL_SCORES_SCHEMA

def test_clean_output_matches_contract(spark):
    """Output schema must exactly match Contract 2."""
    raw = generate_minimal_raw_data(spark)
    result = _clean_sensor_readings_logic(raw)

    actual_fields = {f.name: f.dataType for f in result.schema.fields}
    expected_fields = {f.name: f.dataType for f in CLEAN_SENSOR_SCHEMA.fields}

    assert actual_fields == expected_fields, (
        f"Schema mismatch.\n"
        f"Missing: {set(expected_fields) - set(actual_fields)}\n"
        f"Extra: {set(actual_fields) - set(expected_fields)}"
    )

def test_model_scores_schema(spark):
    """Scoring output must match Contract 5."""
    features = generate_minimal_features(spark)
    result = _score_with_model(features, model=load_test_model())

    for col_name, expected_type in MODEL_SCORES_SCHEMA.items():
        assert col_name in result.columns, f"Missing column: {col_name}"
        actual_type = result.schema[col_name].dataType
        assert actual_type == expected_type, (
            f"Column {col_name}: expected {expected_type}, got {actual_type}"
        )
```

### Contract 3: Device Features Schema

```python
def test_device_features_schema(spark):
    """Feature output must match Contract 3."""
    clean = generate_minimal_clean_data(spark)
    result = _compute_features_logic(clean, window_minutes=15)

    # Required feature columns must be present
    required_feature_cols = [
        "temp_evaporator_mean", "temp_evaporator_std", "temp_evaporator_slope",
        "vibration_compressor_max", "current_compressor_mean",
        "pressure_ratio", "current_per_temp_spread",
    ]
    for col in required_feature_cols:
        assert col in result.columns, f"Missing feature column: {col}"

    rows = result.collect()
    for row in rows:
        assert row["window_duration_minutes"] in (15, 60, 1440), (
            f"Invalid window_duration_minutes: {row['window_duration_minutes']}"
        )
        assert 0.0 <= row["sensor_completeness"] <= 1.0, (
            f"sensor_completeness out of [0,1]: {row['sensor_completeness']}"
        )
```

### Contract 4: Model Input Schema

```python
def test_model_input_schema(spark):
    """Model input must match Contract 4 — no nulls in feature vector."""
    features = generate_minimal_features(spark)
    result = _prepare_model_input(features, model_id="isolation_forest_v2")

    # feature_vector must contain no nulls
    null_vectors = result.filter(
        F.exists("feature_vector", lambda x: x.isNull())
    ).count()
    assert null_vectors == 0, "feature_vector must not contain null values"

    # feature_names length must match feature_vector length
    mismatched = result.filter(
        F.size("feature_names") != F.size("feature_vector")
    ).count()
    assert mismatched == 0, "feature_names length must match feature_vector length"
```

### Contract 6: Device Alerts Schema

```python
def test_device_alerts_schema(spark):
    """Alert output must match Contract 6."""
    scores = generate_minimal_scores(spark)
    result = _generate_alerts(scores)

    rows = result.collect()
    valid_severities = {"LOW", "MEDIUM", "HIGH", "CRITICAL"}
    valid_alert_types = {
        "ANOMALY_DETECTED", "ANOMALY_SUSTAINED",
        "ANOMALY_ESCALATED", "ANOMALY_RESOLVED",
    }
    for row in rows:
        assert row["severity"] in valid_severities, (
            f"Invalid severity: {row['severity']}"
        )
        assert row["alert_type"] in valid_alert_types, (
            f"Invalid alert_type: {row['alert_type']}"
        )
        assert row["sla_deadline"] > row["alert_created_at"], (
            "sla_deadline must be after alert_created_at"
        )
```

### Schema Definition Source of Truth

Define contract schemas as Python `StructType` objects in a shared module (`schemas.py`), derived from the column tables in [data-contracts.md](../05-architecture/data-contracts.md). This is a manual synchronization — if a contract changes, the schema definition and the doc must both be updated. A CI check that the two are in sync is ideal but non-trivial; in practice, treat the markdown as authoritative and the Python schema as a tested copy.

## Data Distribution Assumption Tests

Schema tests validate structure. Distribution tests validate that the data the model is scoring still resembles the data it was trained on. These tests compare current feature statistics against the training-time statistics stored alongside the model (see [Training-Serving Skew Detection](./data-quality.md#training-serving-skew-detection)).

```python
def test_feature_distributions_within_training_bounds(spark):
    """Current feature means should be within 2 stddevs of training means.
    If multiple features drift simultaneously, the model is operating
    outside its training distribution."""
    current_features = load_latest_features(spark)  # latest scoring window
    training_stats = load_training_statistics(spark, model_id="isolation_forest_v2")

    training_stats_pd = training_stats.toPandas().set_index("feature_name")
    current_stats_pd = (
        current_features
        .select([F.mean(c).alias(c) for c in training_stats_pd.index])
        .toPandas()
        .iloc[0]
    )

    drifted_features = []
    for feature_name in training_stats_pd.index:
        training_mean = training_stats_pd.loc[feature_name, "training_mean"]
        training_stddev = training_stats_pd.loc[feature_name, "training_stddev"]
        current_mean = current_stats_pd[feature_name]

        if training_stddev > 0:
            z_score = abs(current_mean - training_mean) / training_stddev
            if z_score > 2.0:
                drifted_features.append(
                    f"{feature_name}: z={z_score:.2f} "
                    f"(current={current_mean:.4f}, training={training_mean:.4f})"
                )

    assert len(drifted_features) <= 3, (
        f"{len(drifted_features)} features drifted beyond 2 stddevs of training mean:\n"
        + "\n".join(drifted_features)
    )


def test_feature_ranges_not_expanding(spark):
    """No more than 10% of values should fall outside the training range."""
    current_features = load_latest_features(spark)
    training_stats = load_training_statistics(spark, model_id="isolation_forest_v2")

    training_stats_pd = training_stats.toPandas().set_index("feature_name")
    total_rows = current_features.count()

    for feature_name in training_stats_pd.index:
        t_min = training_stats_pd.loc[feature_name, "training_min"]
        t_max = training_stats_pd.loc[feature_name, "training_max"]

        out_of_range = current_features.filter(
            (F.col(feature_name) < t_min) | (F.col(feature_name) > t_max)
        ).count()

        fraction_outside = out_of_range / total_rows if total_rows > 0 else 0
        assert fraction_outside <= 0.10, (
            f"{feature_name}: {fraction_outside:.1%} of values outside "
            f"training range [{t_min}, {t_max}]"
        )
```

These tests run in CI against the regression test dataset and can also be scheduled as a post-scoring validation step in production.

## Model-Specific Tests

### Determinism

```python
def test_isolation_forest_deterministic(spark):
    """Same input must produce same scores — no randomness in inference."""
    features = generate_features(spark, device_count=50)
    model = load_test_model("isolation_forest_v2")

    scores_1 = _score_with_model(features, model).toPandas()
    scores_2 = _score_with_model(features, model).toPandas()

    pd.testing.assert_frame_equal(
        scores_1.sort_values("device_id").reset_index(drop=True),
        scores_2.sort_values("device_id").reset_index(drop=True),
    )
```

### Edge Cases

```python
def test_all_sensors_null(spark):
    """A device with all null sensors should be excluded, not crash."""
    features = generate_features(spark, device_count=5, null_rate=1.0)
    model = load_test_model("isolation_forest_v2")

    # Should not raise an exception
    scores = _score_with_model(features, model)
    # Device should either be excluded or scored with a low-confidence flag
    assert scores.count() >= 0  # doesn't crash

def test_single_sensor_available(spark):
    """Only one sensor reporting — model should handle gracefully."""
    features = generate_features(
        spark, device_count=5,
        available_sensors=["temperature_evaporator"]
    )
    model = load_test_model("isolation_forest_v2")
    scores = _score_with_model(features, model)
    assert scores.filter(F.col("anomaly_score").isNotNull()).count() >= 0

def test_extreme_values_within_range(spark):
    """Values at the boundary of valid ranges should not crash the model."""
    features = generate_features(spark, device_count=5, extreme_values=True)
    model = load_test_model("isolation_forest_v2")
    scores = _score_with_model(features, model)

    # Scores should still be in [0, 1]
    out_of_range = scores.filter(
        (F.col("anomaly_score") < 0) | (F.col("anomaly_score") > 1)
    ).count()
    assert out_of_range == 0, "Scores must be in [0, 1]"
```

### Fallback System Tests

The [rules-based fallback](./model-serving.md#rules-based-fallback) is the safety net when ML models are unavailable. If the fallback itself is buggy, the system has no last line of defense. These tests validate that the fallback catches obvious anomalies, does not flag normal devices, and activates correctly when scores go stale.

```python
def test_rules_fallback_catches_obvious_anomaly(spark):
    """Fallback rules should flag a device with clearly anomalous readings."""
    anomalous_features = spark.createDataFrame([{
        "device_id": "REF-TEST-00001",
        "window_start": datetime(2026, 1, 15, 10, 0),
        "window_end": datetime(2026, 1, 15, 10, 15),
        "temp_evaporator_mean": 5.0,          # above freezing — should trigger
        "vibration_compressor_max": 25.0,      # extreme vibration — should trigger
        "pressure_low_mean": 80.0,             # dangerously low — should trigger
        "current_compressor_max": 38.0,        # near overload — should trigger
        "sensor_completeness": 0.9,
    }])

    result = rules_based_fallback(anomalous_features)
    row = result.collect()[0]
    assert row["anomaly_flag"] is True, (
        "Fallback should flag device with extreme evaporator temp, "
        "vibration, low pressure, and high current"
    )


def test_rules_fallback_does_not_flag_normal(spark):
    """Fallback rules should not flag a device with normal readings."""
    normal_features = spark.createDataFrame([{
        "device_id": "REF-TEST-00002",
        "window_start": datetime(2026, 1, 15, 10, 0),
        "window_end": datetime(2026, 1, 15, 10, 15),
        "temp_evaporator_mean": -22.0,         # normal
        "vibration_compressor_max": 4.0,        # normal
        "pressure_low_mean": 210.0,             # normal
        "current_compressor_max": 14.0,         # normal
        "sensor_completeness": 0.95,
    }])

    result = rules_based_fallback(normal_features)
    row = result.collect()[0]
    assert row["anomaly_flag"] is False, (
        "Fallback should not flag device with all normal sensor readings"
    )


def test_staleness_triggers_fallback(spark):
    """When scores exceed the staleness threshold, the system should
    switch to rules-based fallback."""
    from datetime import datetime, timedelta

    stale_scores = spark.createDataFrame([{
        "device_id": "REF-TEST-00003",
        "anomaly_score": 0.3,
        "anomaly_flag": False,
        "scored_at": datetime.utcnow() - timedelta(hours=3),  # 3 hours old
    }])

    staleness_hours = (
        (datetime.utcnow() - stale_scores.collect()[0]["scored_at"])
        .total_seconds() / 3600
    )
    assert staleness_hours > 2.0, "Scores should be detected as stale (>2h)"

    # When stale, system should use fallback instead of stale ML scores
    features = spark.createDataFrame([{
        "device_id": "REF-TEST-00003",
        "window_start": datetime(2026, 1, 15, 10, 0),
        "window_end": datetime(2026, 1, 15, 10, 15),
        "temp_evaporator_mean": 5.0,           # anomalous
        "vibration_compressor_max": 25.0,
        "pressure_low_mean": 80.0,
        "current_compressor_max": 38.0,
        "sensor_completeness": 0.9,
    }])

    fallback_result = rules_based_fallback(features)
    row = fallback_result.collect()[0]
    assert row["anomaly_flag"] is True, (
        "Fallback should catch obvious anomaly when ML scores are stale"
    )
```

## Regression Testing

When retraining a model, the new version should at least match the old version's quality. Without labeled data, we use proxy metrics (see [Model Integration — Evaluation](../04-palantir/model-integration.md#model-evaluation)).

### Regression Test Pattern

```python
def test_retrained_model_not_worse(spark):
    """New model version should produce similar score distributions."""
    features = load_regression_test_features(spark)  # fixed dataset, versioned
    old_model = load_model("isolation_forest_v2")
    new_model = load_model("isolation_forest_v3")

    old_scores = _score_with_model(features, old_model).toPandas()
    new_scores = _score_with_model(features, new_model).toPandas()

    # Check 1: contamination rate within ±2%
    old_rate = (old_scores["anomaly_flag"] == True).mean()
    new_rate = (new_scores["anomaly_flag"] == True).mean()
    assert abs(old_rate - new_rate) < 0.02, (
        f"Contamination rate shifted: {old_rate:.3f} → {new_rate:.3f}"
    )

    # Check 2: score distribution KL divergence is small
    kl_div = compute_kl_divergence(
        old_scores["anomaly_score"], new_scores["anomaly_score"]
    )
    assert kl_div < 0.1, f"Score distribution diverged: KL={kl_div:.4f}"

    # Check 3: top anomalous devices overlap ≥ 80%
    old_top = set(old_scores.nlargest(100, "anomaly_score")["device_id"])
    new_top = set(new_scores.nlargest(100, "anomaly_score")["device_id"])
    overlap = len(old_top & new_top) / len(old_top)
    assert overlap >= 0.8, f"Top-100 overlap: {overlap:.1%}"
```

### Regression Test Dataset

Maintain a fixed, versioned feature dataset (`regression_test_features_v1`) in a dedicated test project. This dataset should:

- Cover ~1,000 devices (1% of fleet — enough for distribution-level checks)
- Include known edge cases: devices with low `sensor_completeness`, devices at range boundaries
- Be updated only when the feature schema changes (Contract 3 update), not on every retrain
- Be stored as a Foundry dataset in a separate test project with read-only access from CI builds

## Test Data Management in Foundry

### Test Projects

Create a dedicated Foundry project for test data and test pipelines:

```
/Company/test-resources/
├── test-data/
│   ├── synthetic_raw_readings/        # Generated by RefrigerationSyntheticGenerator
│   ├── regression_test_features_v1/   # Fixed feature dataset for regression tests
│   ├── golden_alerts/                 # Expected alerts for known input scenarios
│   └── edge_case_readings/            # Curated edge cases (all nulls, extremes, etc.)
├── test-pipelines/
│   ├── integration_test_pipeline/     # End-to-end transform chain for integration tests
│   └── schema_validation_pipeline/    # Validates output schemas against contracts
└── test-models/
    ├── isolation_forest_test_v1/      # Trained on synthetic data, for test use only
    └── autoencoder_test_v1/           # Same
```

### Isolation from Production

- Test projects use separate Foundry projects with no cross-references to production datasets
- Test models are trained on synthetic data — never on production features
- CI builds run against test project datasets only
- Production datasets are never read in test pipelines (avoids permission issues and data sensitivity concerns)

## CI/CD Patterns for Foundry Code Repositories

### Branch Build Checks

Foundry Code Repositories support branch-based development. Configure checks that run on every merge request:

| Check | What It Does | Failure Blocks Merge |
|---|---|---|
| Lint (flake8/ruff) | Code style and common errors | Yes |
| Type check (mypy) | Type annotation violations | Yes — catches schema mismatches early |
| Unit tests (pytest) | Run all `test_*.py` files with `local[2]` Spark | Yes |
| Schema validation | Output schema matches data contract | Yes |
| Transform build (dry run) | Foundry compiles the Transform chain — catches import errors, missing inputs | Yes |

### Branch Build to Staging to Release

```
feature/add-vibration-feature
    └── CI checks pass
        └── Merge to staging
            └── Staging build (runs on small sample of real data in staging project)
                └── Manual review of staging output
                    └── Merge to release
                        └── Production build
```

- **Feature branch**: CI runs unit tests and dry builds. No real data access.
- **Staging**: Foundry builds the full Transform chain against a staging project with a small sample of production-like data. Results are manually inspected.
- **Release**: production deployment. The Transform reads real production inputs.

### CI Timing Constraint

All CI checks should complete in under 5 minutes. Slow CI encourages developers to skip running it. The biggest risk is Spark startup time — use the local Spark session approach (one session per test suite, not per test) and keep test datasets small (100–1,000 rows, not 100K).

## Infrastructure Tests

A model can pass every unit test, schema test, and regression test and still fail in production because of infrastructure problems. Infrastructure tests validate that the serving environment is correctly configured before a model runs.

### Why Infrastructure Tests Matter

The most common production failures are not model bugs — they are environment issues: the model artifact can't be loaded, an input dataset doesn't exist or isn't accessible, the compute profile is too small, or a Python dependency is missing. These failures happen at deploy time, not development time, and are invisible to unit tests that run in a local Spark session.

### Infrastructure Test Suite

```python
def test_model_loads_successfully():
    """Model artifact can be deserialized without errors."""
    from palantir_models.transforms import ModelInput

    model = ModelInput("/Company/models/refrigeration/isolation_forest")
    assert model is not None, "Model artifact failed to load"
    # Verify model has expected interface
    assert hasattr(model, "transform"), "Model must expose transform() method"


def test_input_dataset_accessible(spark):
    """Scoring Transform can read from the input feature dataset."""
    features = spark.read.format("foundry").load(
        "/Company/pipelines/refrigeration/features/device_features"
    )
    assert features.count() > 0, "Feature dataset is empty or inaccessible"
    # Verify expected columns exist
    required_cols = ["device_id", "window_end", "sensor_completeness"]
    for col in required_cols:
        assert col in features.columns, f"Missing required column: {col}"


def test_output_dataset_writable(spark):
    """Scoring Transform can write to the output scores dataset."""
    test_row = spark.createDataFrame([{
        "device_id": "INFRA-TEST-00001",
        "anomaly_score": 0.5,
        "anomaly_flag": False,
        "model_id": "infrastructure_test",
        "scored_at": datetime.utcnow(),
    }])
    # Attempt write to a test partition (will be cleaned up)
    try:
        test_row.write.format("foundry").mode("append").save(
            "/Company/test-resources/test-data/infra_test_output"
        )
    except Exception as e:
        pytest.fail(f"Cannot write to output dataset: {e}")


def test_compute_profile_adequate(spark):
    """Compute profile has enough memory for expected data volume."""
    features = spark.read.format("foundry").load(
        "/Company/pipelines/refrigeration/features/device_features"
    )
    latest_window = features.agg(F.max("window_end")).collect()[0][0]
    current_features = features.filter(F.col("window_end") == latest_window)
    device_count = current_features.count()

    # Pandas conversion is the memory bottleneck — estimate required memory
    estimated_memory_mb = device_count * 50 * 8 / (1024 * 1024)  # 50 features, 8 bytes each
    # Driver should have at least 2x the estimated memory for overhead
    assert estimated_memory_mb < 8000, (
        f"Estimated Pandas memory ({estimated_memory_mb:.0f} MB) may exceed "
        f"driver capacity. Consider Spark UDF-based inference or larger profile."
    )


def test_dependency_availability():
    """Required Python packages are importable."""
    required_packages = [
        "sklearn",          # Isolation Forest
        "numpy",            # Numerical operations
        "pandas",           # Model adapter interface
        "pyspark",          # Spark operations
        "joblib",           # Model serialization
    ]
    missing = []
    for package in required_packages:
        try:
            __import__(package)
        except ImportError:
            missing.append(package)

    assert len(missing) == 0, f"Missing required packages: {missing}"
```

### When to Run Infrastructure Tests

| Trigger | Rationale |
|---|---|
| Before every production deployment | Catch environment issues before they affect scoring |
| After compute profile changes | Verify new profile has adequate resources |
| After Foundry platform upgrades | Platform changes can affect dataset access or model loading |
| After dependency version updates | New package versions may break model serialization |

## Cross-References

- [Data Contracts](../05-architecture/data-contracts.md) — the schemas validated by schema tests
- [Transform Patterns](../04-palantir/transform-patterns.md) — the Transform structure being tested
- [Model Integration](../04-palantir/model-integration.md) — model training and adapter patterns tested here
- [Data Quality](./data-quality.md) — dataset expectations that complement these tests in production
- [Feature Engineering](../02-feature-engineering/) — the feature computations being tested
