# Pipeline Testing

PySpark pipelines are notoriously hard to test. They're slow to start, operate on distributed data, and have subtle behaviors around nulls, types, and ordering. But untested pipelines are a liability — a silent bug in feature computation corrupts every model downstream.

## Unit Testing PySpark Transforms

### Setup: local SparkSession for tests

```python
# conftest.py
import pytest
from pyspark.sql import SparkSession

@pytest.fixture(scope="session")
def spark():
    return (
        SparkSession.builder
        .master("local[2]")
        .appName("tests")
        .config("spark.sql.shuffle.partitions", "2")  # speed up tests
        .config("spark.ui.enabled", "false")
        .getOrCreate()
    )
```

### Test a feature computation

```python
# test_time_domain.py
from datetime import datetime
from features.time_domain import compute_rolling_stats

def test_rolling_mean_24h(spark):
    # Arrange: 3 readings over 24 hours for one device
    data = [
        ("device-001", datetime(2026, 2, 15, 0, 0), -18.0),
        ("device-001", datetime(2026, 2, 15, 12, 0), -20.0),
        ("device-001", datetime(2026, 2, 15, 23, 59), -19.0),
    ]
    df = spark.createDataFrame(data, ["device_id", "timestamp", "value"])

    # Act
    result = compute_rolling_stats(df)

    # Assert: the last row should have mean of all 3 readings
    last_row = result.orderBy("timestamp").collect()[-1]
    assert abs(last_row.temp_mean_24h - (-19.0)) < 0.01


def test_rolling_mean_excludes_old_data(spark):
    """Readings older than 24h should not be included in the window."""
    data = [
        ("device-001", datetime(2026, 2, 14, 0, 0), -100.0),  # >24h ago
        ("device-001", datetime(2026, 2, 15, 12, 0), -20.0),
        ("device-001", datetime(2026, 2, 15, 23, 59), -19.0),
    ]
    df = spark.createDataFrame(data, ["device_id", "timestamp", "value"])

    result = compute_rolling_stats(df)

    last_row = result.orderBy("timestamp").collect()[-1]
    # Should only average the two recent readings, not the -100
    assert abs(last_row.temp_mean_24h - (-19.5)) < 0.01


def test_handles_single_reading(spark):
    """A device with only one reading should still produce valid features."""
    data = [("device-001", datetime(2026, 2, 15, 12, 0), -20.0)]
    df = spark.createDataFrame(data, ["device_id", "timestamp", "value"])

    result = compute_rolling_stats(df)
    row = result.collect()[0]

    assert row.temp_mean_24h == -20.0
    assert row.temp_std_24h is None  # stddev of 1 value is undefined
```

## Testing Edge Cases

### Null handling

```python
def test_null_values_dont_crash(spark):
    data = [
        ("device-001", datetime(2026, 2, 15, 12, 0), None),
        ("device-001", datetime(2026, 2, 15, 13, 0), -20.0),
    ]
    df = spark.createDataFrame(data, ["device_id", "timestamp", "value"])

    result = compute_rolling_stats(df)

    # Should not throw, should handle nulls gracefully
    assert result.count() == 2
```

### Device isolation

```python
def test_windows_are_per_device(spark):
    """Features for device A should not include readings from device B."""
    data = [
        ("device-A", datetime(2026, 2, 15, 12, 0), -20.0),
        ("device-B", datetime(2026, 2, 15, 12, 0), 100.0),  # very different value
    ]
    df = spark.createDataFrame(data, ["device_id", "timestamp", "value"])

    result = compute_rolling_stats(df)

    device_a = result.where("device_id = 'device-A'").collect()[0]
    assert device_a.temp_mean_24h == -20.0  # not contaminated by device B
```

### No future data leakage

```python
def test_no_future_leakage(spark):
    """Features at time T should only use data from T and before."""
    data = [
        ("device-001", datetime(2026, 2, 15, 10, 0), -18.0),
        ("device-001", datetime(2026, 2, 15, 12, 0), -20.0),
        ("device-001", datetime(2026, 2, 15, 14, 0), -22.0),
    ]
    df = spark.createDataFrame(data, ["device_id", "timestamp", "value"])

    result = compute_rolling_stats(df)

    # At 12:00, the feature should not know about the 14:00 reading
    row_noon = result.where("timestamp = '2026-02-15 12:00:00'").collect()[0]
    assert abs(row_noon.temp_mean_24h - (-19.0)) < 0.01  # avg of -18 and -20
```

## Integration Tests with Synthetic Data

Generate realistic synthetic data to test full pipeline runs:

```python
import numpy as np
from datetime import datetime, timedelta

def generate_synthetic_device(
    device_id: str,
    start: datetime,
    duration_hours: int = 24,
    interval_seconds: int = 60,
    base_temp: float = -18.0,
    noise_std: float = 0.5,
    dropout_rate: float = 0.02,
) -> list:
    """Generate synthetic sensor readings for one device."""
    readings = []
    current = start
    end = start + timedelta(hours=duration_hours)

    while current < end:
        if np.random.random() > dropout_rate:  # simulate dropout
            value = base_temp + np.random.normal(0, noise_std)
            readings.append((device_id, current, "temperature", value, "celsius"))
        current += timedelta(seconds=interval_seconds)

    return readings


def generate_synthetic_fleet(n_devices: int = 100, **kwargs) -> list:
    """Generate data for a fleet with varying characteristics."""
    all_readings = []
    for i in range(n_devices):
        device_id = f"device-{i:04d}"
        # Vary parameters to simulate fleet heterogeneity
        base_temp = -18.0 + np.random.normal(0, 2.0)
        noise_std = 0.5 + abs(np.random.normal(0, 0.2))
        readings = generate_synthetic_device(
            device_id, base_temp=base_temp, noise_std=noise_std, **kwargs
        )
        all_readings.extend(readings)
    return all_readings
```

## What to Test at Each Layer

| Layer | What to test | Example |
|-------|-------------|---------|
| Ingestion | Schema validation, quarantine routing | Invalid messages go to quarantine |
| Feature engineering | Correctness, null handling, no leakage | Rolling mean matches manual calculation |
| Cross-sensor | Pivot correctness, null propagation | Missing sensor → null ratio (not 0) |
| Feature store | Point-in-time correctness, backfill | Features at T don't include data from T+1 |
| Full pipeline | End-to-end on synthetic data | 100-device fleet processes without error |

## Running Tests

```bash
# Run with pytest, limit Spark log noise
PYSPARK_PYTHON=python3 pytest tests/ -v --log-cli-level=WARN

# Run specific test file
pytest tests/test_time_domain.py -v

# Run with coverage
pytest tests/ --cov=features --cov-report=html
```

Keep tests fast by using `local[2]` mode and small datasets (10–100 rows). Save large-scale performance tests for CI nightly runs.
