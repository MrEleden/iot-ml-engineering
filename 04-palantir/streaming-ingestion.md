# Streaming Ingestion — Azure Event Hubs to Foundry

How 14M sensor readings per minute flow from Azure IoT Hub through Event Hubs into Foundry streaming datasets, and how those streaming datasets materialize into batch datasets for downstream Transforms.

## Why Streaming Datasets

At 100K devices × 14 sensors × 1 reading/min, batch polling would introduce unacceptable latency for urgent anomalies (compressor failure, refrigerant leak) and would need to handle checkpointing, deduplication, and schema enforcement manually. Foundry streaming datasets handle all of this natively — they connect directly to Event Hubs, manage consumer offsets, deserialize messages, and make data available to both streaming and batch consumers.

See [Foundry Platform Reference](./foundry-platform-reference.md) for general background on Foundry's processing modes (batch, incremental, streaming).

## Source Configuration

### Event Hubs Connection

Foundry connects to Azure Event Hubs via its built-in Event Hubs connector. Configuration requires:

- **Connection string**: the Event Hubs namespace connection string (stored as a Foundry secret, never in code)
- **Event Hub name**: the specific hub receiving IoT Hub-routed messages
- **Consumer group**: a dedicated consumer group for Foundry — never share `$Default` with other consumers, or you'll lose messages
- **Starting offset**: `earliest` for initial backfill, `latest` for production steady-state

### Authentication

Use Foundry's Source integration to store the Event Hubs connection string. The connection string is encrypted at rest and scoped to the project. Rotate the shared access key on a schedule and update the Foundry Source — downstream streaming datasets pick up the new key without pipeline changes.

## Message Format and Deserialization

### Expected Message Structure

Each IoT Hub message arrives as a JSON payload. The IoT Hub message enrichment adds device metadata to the system properties, but the body contains the sensor reading. The primary format is a wide/batched message carrying all 14 sensor values in a single JSON object (matching the [ingestion architecture](../01-ingestion/architecture.md)):

```json
{
  "device_id": "REF-US-W-00042371",
  "timestamp": "2026-02-15T10:32:17.000Z",
  "readings": {
    "temperature_evaporator": -18.4,
    "temperature_condenser": 35.2,
    "temperature_ambient": 22.1,
    "temperature_discharge": 72.5,
    "temperature_suction": -12.3,
    "pressure_high_side": 1620.0,
    "pressure_low_side": 310.0,
    "current_compressor": 12.3,
    "vibration_compressor": 2.1,
    "humidity": 45.6,
    "door_open_close": false,
    "defrost_cycle": false,
    "superheat": 6.1,
    "subcooling": 8.4
  }
}
```

Not all devices send all 14 fields. Firmware version differences mean some devices omit certain sensors. Missing fields arrive as absent JSON keys, which map to null columns in the streaming dataset schema.

> **Note**: Some legacy devices may send per-sensor narrow messages (`{"device_id": "...", "timestamp": "...", "sensor_type": "temperature_evaporator", "value": -18.4, "unit": "celsius", "quality": "good"}`). The streaming dataset deserializer handles both formats, but the wide format is canonical and preferred.

### Deserialization in Foundry

The streaming dataset is configured with a JSON deserializer. Foundry applies the schema you define on the streaming dataset to each incoming message. Messages that fail deserialization are routed to a dead-letter sidecar dataset — monitor this dataset; a spike means a device firmware update changed the payload format.

### Schema Considerations

- Define all numeric fields as `double` even if the source sends integers — mixed-type fields across devices cause deserialization failures
- `timestamp` must be `string` at ingestion (ISO 8601), then cast to `timestamp` in the first downstream Transform — Foundry's streaming deserializer handles timestamp parsing inconsistently across timezones
- Include a `_foundry_ingest_timestamp` column (auto-populated) for tracking ingestion delay vs device-reported timestamp

## Streaming-to-Batch Materialization

### How It Works

Foundry streaming datasets persist data in two representations simultaneously:

1. **Streaming view**: a live, append-only log that Foundry streaming Transforms (Flink-based) can consume with sub-second latency
2. **Batch view**: periodic snapshots materialized as Foundry dataset transactions, which batch Transforms (`@transform_df`) can consume

The batch view materializes on a configurable schedule (e.g., every 5 minutes). Each materialization creates a new transaction containing only the records that arrived since the last materialization. This is critical: downstream incremental Transforms can process only the new transaction rather than re-reading the entire dataset.

### Materialization Interval Tradeoffs

| Interval | Batch freshness | Transaction count/day | Downstream incremental overhead |
|---|---|---|---|
| 1 min | ~1 min | 1,440 | High — many small transactions, metadata pressure |
| 5 min | ~5 min | 288 | Good balance for hourly feature computation |
| 15 min | ~15 min | 96 | Appropriate if features only update hourly |

For our pipeline: **5-minute materialization** for the primary sensor readings dataset. Urgent anomaly detection runs on the streaming view directly; batch feature engineering runs on the batch view.

### Failure Mode: Materialization Backlog

If the streaming dataset falls behind (e.g., during an Event Hubs partition rebalance or a Foundry platform upgrade), the next materialization may contain a very large batch. Downstream incremental Transforms will still process it correctly, but they may be slow or OOM if they assume small batches. Design downstream Transforms to handle variable batch sizes — don't hardcode partition counts or memory assumptions.

## Partitioning Strategy

### Why Partitioning Matters

14M readings/min at 5-minute materialization windows means ~70M records per batch transaction. Without partitioning, any downstream Transform that filters by time or device must scan the entire transaction.

### Recommended Partitioning

Partition the materialized batch dataset by **date** and **hour**:

```
/date=2026-02-15/hour=10/part-00000.parquet
/date=2026-02-15/hour=10/part-00001.parquet
/date=2026-02-15/hour=11/part-00000.parquet
```

This is configured on the streaming dataset's batch materialization settings, not in downstream code. Set `date` as `yyyy-MM-dd` derived from `timestamp` and `hour` as integer (0–23).

**Why not partition by `device_id`?** 100K devices would create 100K partitions per hour — far too many small files. Parquet's columnar format already makes `device_id` filtering efficient via predicate pushdown.

### Event Hubs Partition Count

The Event Hubs partition count controls ingestion parallelism, not Foundry storage partitioning. For 14M readings/min:

- **Minimum**: 32 partitions (each partition handles ~440K msgs/min, within Event Hubs throughput unit limits)
- **Recommended**: 64 partitions (headroom for traffic spikes, device reconnection storms after network outages)
- **Maximum practical**: 128 (diminishing returns, more consumer group coordination overhead)

Ensure the IoT Hub routes messages with `device_id` as the partition key — this guarantees per-device message ordering within a partition, which matters for time-series correctness.

## Backpressure and Scaling

### Foundry's Scaling Model

Foundry manages the streaming infrastructure (Flink clusters, consumer scaling) internally. You do not configure the number of streaming consumers or Flink task slots directly. However, you influence scaling through:

- **Event Hubs partitions**: more partitions = more parallel consumers Foundry can deploy
- **Streaming dataset throughput tier**: configurable in Foundry, determines the compute allocated to this streaming dataset
- **Materialization interval**: longer intervals amortize per-transaction overhead but increase memory pressure per batch

### Backpressure Signals

Monitor these metrics to detect backpressure:

- **Consumer lag**: the delta between the Event Hubs latest offset and Foundry's committed offset. Sustained lag > 5 minutes means Foundry is falling behind.
- **Materialization delay**: the gap between scheduled materialization time and actual completion time. Growing delays mean the batch view is lagging.
- **Dead-letter rate**: a spike in deserialization failures may indicate corrupted messages from a firmware update, not a scaling issue — check before scaling up.

### Common Scaling Failure Modes

**Reconnection storms**: When a network outage resolves, 100K devices reconnect simultaneously and send buffered readings. This can produce 10–50x normal throughput for several minutes. The Event Hubs partition count must handle this burst; Foundry's streaming consumer will lag briefly but catch up if the burst is temporary.

**Schema poison pills**: A single malformed message can cause a streaming consumer to retry indefinitely if the deserializer is configured to fail-fast. Configure the streaming dataset to route bad messages to the dead-letter dataset rather than blocking the consumer.

**Timezone drift**: Devices reporting timestamps in local time (instead of UTC) produce out-of-order data that confuses windowed aggregations downstream. Enforce UTC at the IoT Hub level; detect drift by comparing `_foundry_ingest_timestamp` to `timestamp` in a dataset expectation.

## Relationship to Downstream Processing

Once sensor readings land in the batch view of the streaming dataset, they flow into:

- **[Transform Patterns](./transform-patterns.md)** — incremental Transforms pick up each new materialization transaction and compute features
- **[Ontology Design](./ontology-design.md)** — the `SensorReading` object type is backed by the materialized dataset
- **[Model Integration](./model-integration.md)** — batch scoring Transforms consume feature datasets derived from this ingestion pipeline

## ML Pipeline Considerations

Streaming datasets introduce specific challenges for ML workflows. The following subsections address how to handle training data versioning, temporal correctness, and schema evolution when building ML pipelines on top of streaming ingestion.

### Training Data Versioning

Never train a model directly on a live streaming dataset. The streaming dataset is continuously appending new transactions, so two training runs at different times would see different data — making reproducibility impossible.

Instead, define training sets by **transaction range**:

1. **Snapshot the batch view**: identify the start and end transaction IDs that correspond to the desired training window (e.g., 2026-01-01 to 2026-01-31)
2. **Tag the snapshot**: use Foundry dataset tagging to label the pair of transactions as `training_set_v3_jan2026` — this creates a human-readable, immutable reference
3. **Reference the tag in training code**: the training Transform reads from the tagged transaction range, not from "latest" — guaranteeing that re-running the Transform produces identical training data
4. **Store the tag in model metadata**: when the trained model is published, record the training data tag alongside the model version for full provenance

This approach ensures every model version can answer the question: "exactly which data was this model trained on?"

### Point-in-Time Correctness

The materialized batch view orders data by **transaction** (the order in which 5-minute batches were committed to Foundry), not by **device timestamp** (the time the sensor reading was actually recorded). These can differ significantly:

- A device that was offline for 2 hours sends buffered readings when it reconnects — the transaction timestamp is "now" but the device timestamps span the past 2 hours
- Network latency and Event Hubs partition rebalancing introduce variable delays between device timestamp and Foundry transaction timestamp

For ML pipelines, always use the **device timestamp** (`timestamp` field) as the temporal key for label joins and feature computation — never `_foundry_ingest_timestamp`. Specifically:

- When joining labels (e.g., maintenance events) to feature windows, ensure the feature window's time boundary is defined by device timestamp
- When computing time-windowed aggregations for training features, order by device timestamp to avoid data leakage from future readings that arrived in the same transaction
- When splitting data into train/validation/test sets, split by device timestamp ranges, not by transaction boundaries

### Schema Evolution for ML Pipelines

The streaming dataset's schema evolves as new firmware versions add sensor fields or change payload formats. This has direct consequences for ML pipelines:

- **New sensor fields produce null for old messages**: when a new sensor (e.g., `oil_pressure`) is added to the schema, historical messages lack this field. The streaming dataset backfills these as null columns. Training pipelines must handle these nulls explicitly — either drop the column for models trained on historical data, or impute with a documented default.
- **Training pipelines must handle schema transitions**: a model trained on data from January (10 sensor fields) cannot score February data (12 sensor fields) without adapter changes. Pin the feature column list in the model adapter and validate that all expected columns are present at scoring time.
- **Consider a `schema_version` column**: add a `schema_version` field to the streaming dataset schema, populated by the device firmware version or a mapping table. This allows training pipelines to filter training data to a consistent schema version, and allows scoring pipelines to apply version-specific preprocessing.

Document each schema change with the date it took effect and which devices are affected. A schema changelog dataset (mapping `schema_version` → column list → effective date) is invaluable for debugging feature drift caused by schema evolution rather than genuine behavioral change.
