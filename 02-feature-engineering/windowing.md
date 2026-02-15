# Windowing Strategies

How raw per-reading sensor data becomes window-level feature rows — covering window sizes, alignment, tumbling vs sliding tradeoffs, late data, completeness, and multi-resolution detection.

## Why Window, Not Per-Reading

The input dataset (`clean_sensor_readings`, [Contract 2](../05-architecture/data-contracts.md)) has one row per reading per device — roughly 1 reading/minute × 100K devices = ~144M rows/day. Computing and storing per-reading features would produce a feature dataset of the same scale, which creates three problems:

1. **Anomaly detection noise**: a single reading can be an outlier due to sensor noise, not equipment failure. Aggregating over a window smooths noise and reveals actual trends.

2. **Storage cost**: 144M feature rows/day with ~50 columns each is expensive to store and slow to query for model training over months of history.

3. **Model input cardinality**: unsupervised anomaly detection models (Isolation Forest, Autoencoder) work on fixed-length feature vectors. A window-level row is a natural fixed-length representation — one row = one device's operating state over a period.

Windowed aggregation trades temporal granularity for signal quality and manageable data volumes. The right window sizes preserve the information needed to detect anomalies while reducing data volume by 15–1440x depending on window duration.

## Window Sizes

The [`device_features` schema](../05-architecture/data-contracts.md) (Contract 3) defines three standard window durations:

| `window_duration_minutes` | Duration | Purpose | Rows/device/day |
|---------------------------|----------|---------|-----------------|
| 15 | 15 minutes | Fast anomaly detection — catches acute events within minutes | 96 |
| 60 | 1 hour | Trend confirmation — validates that a 15-min anomaly persists | 24 |
| 1440 | 1 day | Long-term degradation tracking — slow drift over days/weeks | 1 |

### 15-Minute Window: Fast Detection

~15 readings per window. Detects:

- Sudden temperature spikes (compressor failure, door left open)
- Pressure drops (refrigerant leak, valve failure)
- Current spikes (mechanical binding, electrical fault)
- Compressor stop (vibration and current drop to zero)

This is the primary window for operational alerting. A 15-minute anomaly score drives the near-real-time alerting path described in [Contract 6](../05-architecture/data-contracts.md). The tradeoff: 15 minutes of data produces noisier statistics than longer windows. `std` on 15 readings is less reliable than on 60 readings — but for acute events, speed matters more than precision.

### 1-Hour Window: Trend Confirmation

~60 readings per window. Detects:

- Sustained temperature trends (warming that persists beyond a single defrost cycle)
- Compressor short-cycling patterns (multiple on/off cycles in an hour — see [Frequency-Domain](./frequency-domain.md))
- Efficiency degradation (rising `current_per_temp_spread` over an hour)

This window size provides enough data for reliable slope estimation (see [Time-Domain — Slope](./time-domain.md)) and filters out transient events. A compressor that spikes in one 15-min window but returns to normal in the next is likely experiencing a normal startup transient. A compressor that shows elevated current across a 1-hour window has a real problem.

The 1-hour window also serves as the primary comparison unit for fleet-level z-scores (see [Cross-Sensor](./cross-sensor.md)) — hourly snapshots are frequent enough to catch daily patterns but stable enough for meaningful fleet comparison.

### 1-Day Window: Long-Term Degradation

~1440 readings per window. Detects:

- Slow efficiency loss over weeks (compressor wear, gradual refrigerant leak)
- Seasonal effects (ambient temperature changes causing baseline shifts)
- Long-term sensor drift (calibration degradation)

Daily features are the training backbone for models that learn "normal" baselines. A 30-day history of 1-day windows gives the model a compact but complete picture of each device's operational profile. Monthly and seasonal patterns emerge clearly at this resolution.

Daily windows are also used for the pre-hydrated properties on the `Device` Ontology object (see [Ontology Design](../04-palantir/ontology-design.md)) — properties like `avg_temperature_evaporator_24h` and `avg_compressor_current_24h` come from the 1-day window.

## Clock-Aligned Tumbling Windows

### Why Clock-Aligned

All windows are aligned to the clock:

- **15-min**: start at :00, :15, :30, :45 of each hour
- **1-hour**: start at the top of each hour (:00)
- **1-day**: start at 00:00 UTC

Why alignment matters:

1. **Comparability**: a device's 10:00–10:15 window can be directly compared to its 10:00–10:15 window yesterday, or to another device's same window today. Unaligned windows (starting at arbitrary offsets) make temporal comparisons ambiguous.

2. **Deterministic joins**: when joining features across window sizes (e.g., attaching the 1-day feature to each 1-hour window within that day), aligned windows guarantee clean containment — every 1-hour window falls entirely within exactly one 1-day window. Unaligned windows create partial overlaps.

3. **Idempotency**: the same input data always produces the same windows. Re-running the transform doesn't shift window boundaries.

### Why Tumbling, Not Sliding

A **tumbling window** produces non-overlapping, adjacent windows. A **sliding window** produces overlapping windows (e.g., a 1-hour window sliding every 15 minutes produces 4x as many rows, with each reading appearing in up to 4 windows).

We use tumbling windows for the `device_features` dataset. The tradeoffs:

| Aspect | Tumbling | Sliding |
|--------|---------|---------|
| Storage | 1x | N× (where N = window_size / slide_interval) |
| Computation | 1x | N× |
| Boundary artifacts | A sudden change at :14 might split across two windows | Smoothed — every offset is a window center |
| Temporal resolution | Equal to window size | Equal to slide interval |
| Complexity | Simple GROUP BY on window boundaries | Requires window functions or repeated aggregation |

For 100K devices × 3 window sizes, sliding windows would multiply storage and compute by 4× (if we slid 15-min windows every 5 minutes) or more. Since the downstream anomaly models tolerate tumbling-window boundary effects (anomalies spanning two windows are detected in both, just with lower magnitude in each), the storage and compute savings justify tumbling windows.

If a specific use case later requires sub-window temporal resolution, consider sliding windows for a single window size on a subset of devices, not fleet-wide.

### Assigning Readings to Windows

Each reading in `clean_sensor_readings` is assigned to a window based on its `timestamp_utc`:

```python
from pyspark.sql import functions as F

# 15-minute tumbling windows, clock-aligned
windowed = clean_readings.withColumn(
    "window_start",
    F.date_trunc("minute", F.col("timestamp_utc"))  # Truncate seconds
    - F.expr("INTERVAL (minute(timestamp_utc) % 15) MINUTES")  # Align to :00/:15/:30/:45
    - F.expr("INTERVAL second(timestamp_utc) SECONDS")  # Remove seconds
)
```

Alternatively, use Spark's `window()` function for cleaner syntax:

```python
# Using Spark's built-in window function
windowed = clean_readings.select(
    "*",
    F.window("timestamp_utc", "15 minutes").alias("time_window")
).select(
    "*",
    F.col("time_window.start").alias("window_start"),
    F.col("time_window.end").alias("window_end"),
)
```

Spark's `window()` function produces clock-aligned tumbling windows by default when the slide duration equals the window duration (or is omitted). The `window_start` and `window_end` columns from this function match the Contract 3 schema.

## Late-Arriving Data

### The Problem

A reading with `timestamp_utc = 10:12` is expected to arrive and be processed before the 10:00–10:15 window closes. But IoT data is messy:

- Device connectivity drops for 5 minutes, then reconnects and sends buffered readings
- The cleansing transform ([Contract 2](../05-architecture/data-contracts.md)) uses `timestamp_ingested` when device clock drift is detected, potentially reassigning timestamps
- Network congestion delays event delivery by several minutes

If the feature transform has already computed the 10:00–10:15 window and a late reading arrives at 10:20, the window's statistics are now stale — computed on incomplete data.

### Handling Strategies

**Strategy 1: Accept Staleness for Recent Windows**

Compute features on whatever data is available when the transform runs. Late-arriving readings are included in the next window they fall into, or simply missed. The `reading_count` column in Contract 3 makes the staleness visible — if a 15-min window has only 8 readings instead of the expected 15, downstream consumers know the statistics are less reliable.

This is the simplest approach and is acceptable when:
- The latency SLA ([Contract 2](../05-architecture/data-contracts.md): 15-minute freshness) means most late data arrives within one window delay
- Anomaly detection tolerates occasional incomplete windows
- The feature store is append-mode — old windows are never updated

**Strategy 2: Recompute Recent Windows (Snapshot Mode)**

Use the [snapshot incremental mode](../04-palantir/transform-patterns.md) for the feature transform. Each run recomputes windows for the last N hours (e.g., the last 2 hours) and overwrites those window rows. Late-arriving data is incorporated on the next recompute cycle.

This is more expensive — each run reads and rewrites recent windows — but produces more accurate features. Use this for the 1-hour and 1-day windows where accuracy matters more than latency.

**Strategy 3: Grace Period**

Delay feature computation by a grace period (e.g., 5 minutes for 15-min windows). The 10:00–10:15 window is not computed until 10:20, giving late data 5 minutes to arrive. This increases end-to-end latency but improves completeness.

**Recommendation**: use Strategy 1 (accept staleness) for 15-minute windows where speed matters, and Strategy 2 (recompute recent windows) for 1-hour and 1-day windows where accuracy matters. The `reading_count` feature signals completeness to downstream models regardless of strategy.

## Window Completeness

### Expected vs Actual Reading Count

At 1 reading/minute, the expected `reading_count` per window is:

| Window | Expected Readings |
|--------|------------------|
| 15-min | ~15 |
| 1-hour | ~60 |
| 1-day  | ~1440 |

Actual reading counts vary due to:
- Sensor dropout (device goes offline)
- Connectivity gaps
- Firmware reboots
- Deliberate rate reduction (some firmware versions send readings every 2 minutes for non-critical sensors)

### Completeness Thresholds

A window with very few readings produces unreliable features. Rather than dropping incomplete windows entirely (which would create gaps in the time series), we compute features on available data and flag completeness:

- **`reading_count`**: raw count, carried in the output for transparency
- **`sensor_completeness`**: fraction of non-null sensor values across readings (explained in [Time-Domain — Data Quality](./time-domain.md))

Downstream consumers apply their own completeness thresholds. The [model input contract](../05-architecture/data-contracts.md) (Contract 4) notes that models may refuse to score if `sensor_completeness < 0.5`. This threshold is a model configuration, not a feature engineering decision.

### Empty Windows

If a device sends zero readings during a window period, no row is produced for that device-window combination. This is intentional — a missing row in `device_features` is the signal that the device was offline during that window. Downstream consumers should interpret a gap as "no data available," not "everything was normal."

An alternative design (write a row with all nulls for missing windows) would create 100K × 96 = 9.6M rows/day from the 15-min window alone, most of which would be null rows for offline devices. The absent-row design keeps the dataset clean and storage efficient.

## Multi-Resolution Detection

The three window sizes complement each other for anomaly detection at different timescales:

```
┌─────────────────────────────────────────────────────────────────┐
│                     1-DAY WINDOW                                │
│   Tracks: long-term degradation, seasonal baselines             │
│   Confirms: "this device has been drifting for days"            │
│                                                                 │
│  ┌──────────────┐ ┌──────────────┐ ┌──────────────┐           │
│  │  1-HOUR       │ │  1-HOUR       │ │  1-HOUR       │  ...     │
│  │  Tracks:      │ │  Tracks:      │ │  Tracks:      │          │
│  │  persistent   │ │  trend        │ │  efficiency   │          │
│  │  anomalies    │ │  confirmation │ │  changes      │          │
│  │               │ │               │ │               │          │
│  │ ┌──┐┌──┐┌──┐ │ │ ┌──┐┌──┐┌──┐ │ │ ┌──┐┌──┐┌──┐ │          │
│  │ │15││15││15│ │ │ │15││15││15│ │ │ │15││15││15│ │     ...  │
│  │ │m ││m ││m │ │ │ │m ││m ││m │ │ │ │m ││m ││m │ │          │
│  │ └──┘└──┘└──┘ │ │ └──┘└──┘└──┘ │ │ └──┘└──┘└──┘ │          │
│  └──────────────┘ └──────────────┘ └──────────────┘           │
└─────────────────────────────────────────────────────────────────┘
```

### Detection Scenarios

**Acute event (compressor failure)**:
- 15-min: anomaly score spikes immediately (current drops to zero, vibration drops to zero)
- 1-hour: confirms the event persisted (not a restart glitch)
- 1-day: may or may not flag depending on how long the device was down — a 30-minute outage doesn't move the daily mean significantly

**Gradual degradation (bearing wear)**:
- 15-min: occasional vibration spikes, mostly below threshold
- 1-hour: `vibration_compressor_p95` trending upward over successive hourly windows
- 1-day: `vibration_compressor_mean` showing a day-over-day rise — confirms degradation

**Intermittent fault (loose connection)**:
- 15-min: sporadic current spikes in some windows, normal in others
- 1-hour: elevated `current_compressor_std` (high variability)
- 1-day: may mask the intermittent spikes in the daily average — the pattern is visible only at 15-min and 1-hour resolution

The anomaly detection models score each window size independently. The alert logic ([Contract 6](../05-architecture/data-contracts.md)) can use agreement across window sizes as a confidence multiplier: an anomaly flagged at 15-min, confirmed at 1-hour, and visible in the 1-day trend is a higher-confidence alert than a single 15-min spike.

## Implementation in Foundry

### Transform Structure

Following the [pre-aggregate then merge pattern](../04-palantir/transform-patterns.md):

**Transform A — Micro-aggregation (incremental)**

Processes new `clean_sensor_readings` transactions. Computes per-5-minute statistics per device. This runs incrementally — only new readings are processed.

**Transform B — Window assembly (scheduled)**

Reads micro-aggregates and rolls up to 15-min, 1-hour, and 1-day windows. Runs on schedule (every 15 minutes for the 15-min window, every hour for the 1-hour window, daily for the 1-day window).

Each window size can be a separate Transform or a single Transform that produces all three. Separate Transforms allow independent scheduling and failure isolation: a failure in the daily computation doesn't block 15-min feature updates.

### Window Boundary in PySpark

```python
# Three window sizes in a single groupBy (if using one Transform)
from pyspark.sql import functions as F

for window_minutes in [15, 60, 1440]:
    window_label = f"{window_minutes}min"
    window_features = (
        clean_readings
        .select(
            "*",
            F.window("timestamp_utc", f"{window_minutes} minutes").alias("time_window")
        )
        .groupBy(
            "device_id",
            F.col("time_window.start").alias("window_start"),
            F.col("time_window.end").alias("window_end"),
            F.lit(window_minutes).alias("window_duration_minutes"),
        )
        .agg(
            F.count("*").alias("reading_count"),
            # ... all feature aggregations from time-domain.md and cross-sensor.md
        )
    )
    # Union or write separately per window size
```

### Partitioning

The `device_features` dataset should be partitioned by the date portion of `window_start` and by `window_duration_minutes`. This enables efficient pruning when queries target a specific time range or window size — which is the common access pattern for both model training (time-range scans) and operational dashboards (latest window for a device).

See [Feature Store](./feature-store.md) for partitioning details and the offline/online serving split.

## Related Documents

- [Time-Domain Features](./time-domain.md) — aggregations computed within each window
- [Cross-Sensor Features](./cross-sensor.md) — derived features computed from windowed aggregations
- [Frequency-Domain Features](./frequency-domain.md) — spectral features and minimum window sizes for FFT
- [Feature Store](./feature-store.md) — how windowed features are stored and served
- [Data Contracts — Contract 3](../05-architecture/data-contracts.md) — schema including `window_start`, `window_end`, `window_duration_minutes`
- [Transform Patterns](../04-palantir/transform-patterns.md) — pre-aggregate then merge pattern, incremental processing
