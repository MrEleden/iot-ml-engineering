# IoT Ingestion Architecture

## Why This Architecture Exists

100K+ refrigeration devices, each with 14 sensors reading once per minute, produce ~14M readings/min. The data must land in Foundry reliably, in order (per device), with latency low enough to support both hourly batch anomaly detection and sub-minute acute failure detection.

The path is: **Azure IoT Hub → Event Hubs → Foundry streaming dataset → Landing zone (batch view)**.

Every component exists because it solves a specific problem. Remove any one and the pipeline either breaks or requires custom code that Foundry already handles.

## Component Responsibilities

### Azure IoT Hub

**Role**: device gateway and management plane.

- **Device authentication**: each of the 100K+ devices has a unique credential in the IoT Hub device registry. Devices that aren't registered are rejected at the gate — not downstream.
- **Device-to-cloud messaging**: receives sensor readings over MQTT/AMQP. Handles reconnection, message queuing when devices are transiently offline, and per-device throttling.
- **Message routing**: forwards all telemetry messages to the connected Event Hub. IoT Hub's built-in Event Hub endpoint is the bridge to Foundry.
- **Message enrichment**: stamps each message with server-side metadata (`timestamp_ingested`, routing info) before forwarding.

IoT Hub does **not** transform, filter, or aggregate data. It is a pass-through with authentication and device lifecycle management.

### Event Hubs

**Role**: buffering and partitioned delivery.

- **Buffering**: decouples the device send rate from Foundry's consumption rate. During reconnection storms (100K devices reconnecting after a network outage and flushing buffered readings), Event Hubs absorbs spikes that would overwhelm a direct consumer.
- **Partitioning**: messages are partitioned by `device_id` (set as the partition key in IoT Hub routing). This guarantees per-device ordering — critical for time-series correctness. Cross-device ordering is not guaranteed and not needed.
- **Retention**: Event Hubs retains messages for a configurable window (7 days recommended). If Foundry's consumer falls behind, it can replay from the retention window without data loss.
- **Consumer groups**: Foundry uses a dedicated consumer group. Never the `$Default` group — sharing consumer groups causes offset conflicts and message loss.

### Foundry Streaming Dataset

**Role**: continuous ingestion, deserialization, and dual-view materialization.

- **Consumer management**: Foundry manages Event Hubs consumer offsets (checkpointing). No manual offset tracking.
- **Deserialization**: parses JSON payloads against the schema defined on the streaming dataset. Messages that fail deserialization route to a dead-letter sidecar dataset.
- **Streaming view**: real-time append-only log consumed by streaming transforms (Flink-based) for acute anomaly detection.
- **Batch view**: periodic materialization (every 5 minutes) into Foundry dataset transactions. Downstream batch transforms (`@transform_df`) consume these transactions incrementally.

See [Streaming Ingestion](../04-palantir/streaming-ingestion.md) for Foundry-specific configuration details.

## Why This Path — And Not Something Else

### Alternative: Direct API Polling

Pull data from devices via REST APIs on a schedule. Rejected because:

- 100K devices × 1 poll/min = 100K API calls/min from the Foundry side. Foundry is not a polling engine.
- No built-in device authentication or lifecycle management.
- No buffering — if the poller is down, data is lost.
- Devices behind NAT/firewalls can't be reached by pull-based architectures.

### Alternative: MQTT Broker → Foundry (Direct)

Point an MQTT broker directly at a Foundry ingestion API. Rejected because:

- Foundry doesn't expose a native MQTT endpoint. A custom bridge would be needed.
- Loses the buffering and partitioning guarantees that Event Hubs provides.
- Device management (auth, registry, lifecycle) would require a separate system anyway — IoT Hub already solves this.

### Alternative: Kafka Instead of Event Hubs

Self-managed Kafka or Confluent Cloud as the message bus. Rejected because:

- Event Hubs is the _built-in_ endpoint of IoT Hub. Using Kafka would require a separate bridge from IoT Hub to Kafka — adding a component, adding latency, adding failure modes.
- Foundry has a native Event Hubs connector. Kafka connectivity is possible but requires more configuration and doesn't benefit from first-party Foundry integration.
- For the Azure-native IoT stack, Event Hubs is the natural fit. Kafka would make sense if we were multi-cloud or had existing Kafka infrastructure — we have neither (per project scope, the infrastructure is Azure-only with no existing Kafka).

### Alternative: Azure Stream Analytics as an Intermediate

Use Azure Stream Analytics between Event Hubs and Foundry to pre-aggregate or filter data before it lands. Rejected because:

- Violates the Foundry-native constraint ([ADR-001](../05-architecture/adr-001-foundry-native.md)). Processing logic outside Foundry cannot be version-controlled, tested, or monitored through Foundry's pipeline tools.
- The landing zone is intentionally raw — no transformation. Pre-aggregating would lose the raw data that's needed for debugging, backfills, and model retraining on historical data.
- Adds an operational dependency outside the Foundry platform.

## Data Flow: Step by Step

### Step 1: Device Sends Reading → IoT Hub

The device publishes a JSON message over MQTT to Azure IoT Hub.

**Message format** (batched — a single message carries all 14 sensor values):

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

Not all devices send all 14 fields. Firmware version differences mean some devices omit certain sensors (e.g., older units without vibration sensors). Missing fields arrive as absent JSON keys, which map to null columns in the streaming dataset schema.

**Latency**: device → IoT Hub is typically **< 1 second** over a stable connection. Devices on cellular or satellite links may see 2–5 seconds.

### Step 2: IoT Hub → Event Hubs (Built-In Endpoint)

IoT Hub forwards the message to its built-in Event Hub endpoint. IoT Hub adds:

- `timestamp_ingested`: server-side receive time (the reliable clock — device clocks drift).
- `event_id`: a unique message identifier.
- System properties: device connection info, routing metadata.

The message is routed to an Event Hubs partition using `device_id` as the partition key. This ensures all readings from a single device land on the same partition and maintain their send order.

**Latency**: IoT Hub → Event Hubs is **< 100ms** (same Azure region). This is an internal Azure transfer, not a network hop.

### Step 3: Event Hubs → Foundry Streaming Dataset

Foundry's Event Hubs connector continuously polls the Event Hubs partitions (one consumer per partition). Each message is deserialized from JSON into the streaming dataset schema. Messages that fail deserialization go to the dead-letter dataset.

The streaming dataset exposes two views:

- **Streaming view**: immediately available to Foundry streaming transforms (Flink). Used for acute anomaly detection via threshold checks.
- **Batch view**: materialized every 5 minutes into a Foundry dataset transaction. Used by incremental batch transforms for feature engineering.

**Latency**: Event Hubs → streaming view available: **< 5 seconds**. Event Hubs → batch view available: **up to 5 minutes** (materialization interval).

### Step 4: Streaming Dataset → Landing Zone

The batch view of the streaming dataset _is_ the landing zone. It stores events exactly as received — no transformation, append-only. The schema matches [Contract 1](../05-architecture/data-contracts.md) (`raw_sensor_readings`).

The batch view materializes in **Delta Lake format** (Parquet-backed, columnar storage). This matters for ML training workloads: training pipelines typically read a subset of columns (e.g., `device_id`, `timestamp_ingested`, and 2–3 sensor columns relevant to a specific model) rather than all 14 sensor fields. Parquet's columnar layout means these column-subset reads skip irrelevant columns entirely — at 100 GB/day raw volume, a training job reading 4 of 18 total columns scans roughly 20–25% of the data. Delta Lake adds ACID transactions and time travel on top of Parquet, enabling reproducible point-in-time reads for training dataset construction (see Continual Learning Support below).

The landing zone is the system of record. Retained for a minimum of 90 days. No deletes or updates.

### End-to-End Latency Summary

| Segment | Typical Latency | Worst Case |
|---------|----------------|------------|
| Device → IoT Hub | < 1s | 5s (poor connectivity) |
| IoT Hub → Event Hubs | < 100ms | 500ms (cross-region) |
| Event Hubs → Streaming view | < 5s | 30s (consumer lag) |
| Event Hubs → Batch view (landing zone) | ≤ 5 min | 10 min (materialization backlog) |
| **Device → Landing zone (batch)** | **< 6 min** | **~16 min** |
| **Device → Streaming view** | **< 10s** | **~36s** |

The 2-minute freshness SLA on the landing zone ([Contract 1](../05-architecture/data-contracts.md)) applies to the delta between `timestamp_ingested` (IoT Hub receive time) and the data appearing in the batch view. The 5-minute materialization interval is the dominant factor.

## Failure Modes

### IoT Hub Throttling

**What happens**: IoT Hub enforces per-device and per-hub message rate limits. If a device sends faster than its tier allows, messages are rejected with HTTP 429.

**How you detect it**: IoT Hub metrics dashboard — `d2c.telemetry.ingress.sendThrottle` counter. The device sees a failure response and should retry with exponential backoff.

**Impact**: individual device data delayed or lost (if source device doesn't retry). No impact on other devices. Fleet-wide throttling would require exceeding the hub-level throughput — unlikely at 1 reading/min/device with a Standard S3 tier, but monitor during reconnection storms.

**Mitigation**: size the IoT Hub tier for peak throughput (reconnection storms produce 10–50x normal traffic for several minutes). Use IoT Hub's auto-scaling if available, or provision for burst capacity.

### Event Hubs Partition Exhaustion

**What happens**: each Event Hubs partition has a throughput limit (1 MB/s ingress, 2 MB/s egress per partition at standard tier). If message volume exceeds partition capacity, messages are rejected.

**How you detect it**: Event Hubs metrics — `ThrottledRequests` counter and `IncomingBytes` per partition. Foundry consumer lag increases.

**Impact**: data delivery delayed. Event Hubs rejects messages at the IoT Hub routing level, causing IoT Hub to buffer and retry internally. If the buffer overflows, messages may be lost.

**Mitigation**: 64 partitions recommended (see Capacity Planning below). Each partition handles ~219K readings/min at our average message size (~300 bytes/message), well within the 1 MB/s limit. The risk is skewed partition assignment — if `device_id` hashing maps too many devices to one partition, that partition saturates while others are idle. Monitor per-partition throughput and rebalance if needed.

### Foundry Connector Failures

**What happens**: the Foundry streaming dataset's Event Hubs consumer fails — network issue, authentication failure (expired connection string), Foundry platform maintenance.

**How you detect it**: consumer lag (Event Hubs latest offset − Foundry committed offset) grows. Foundry pipeline health dashboards show the streaming dataset as unhealthy.

**Impact**: data stops flowing into the streaming and batch views. Event Hubs retains messages for 7 days, so no data is lost — Foundry catches up when the consumer resumes. However, downstream freshness SLAs are violated.

**Mitigation**: monitor consumer lag with alerts (threshold: lag > 5 minutes sustained). Ensure the Event Hubs connection string (stored as a Foundry secret) is rotated before expiry. During planned Foundry maintenance windows, expect and accept temporary lag.

### Message Corruption / Deserialization Failures

**What happens**: a device sends a message that doesn't match the expected JSON schema — wrong field names, non-numeric values in sensor fields, truncated payload, binary garbage.

**How you detect it**: Foundry routes deserialization failures to the dead-letter sidecar dataset. A spike in dead-letter volume is the signal. Monitor this dataset daily; a sudden increase correlates with firmware updates pushing unexpected payload formats.

**Impact**: corrupted messages are not lost — they're in the dead-letter dataset for debugging. Clean messages continue flowing. If corruption is widespread (e.g., bad firmware pushed to thousands of devices), the clean data volume drops noticeably.

**Mitigation**: treat the dead-letter dataset as an alerting source. Set a dataset expectation: dead-letter row count per hour < 0.1% of total ingestion volume. Investigate any spike before it cascades.

### Batch Materialization Backlog

**What happens**: the streaming dataset's batch materialization falls behind — the next materialization contains a much larger batch than normal. This can happen after a consumer lag event or during platform upgrades.

**How you detect it**: materialization delay metric grows — the gap between scheduled and actual materialization time. Downstream incremental transforms process a larger transaction than usual, potentially running slower or OOMing.

**Impact**: landing zone freshness SLA violated. Downstream transforms may fail on unexpectedly large batches.

**Mitigation**: design downstream transforms to handle variable batch sizes — no hardcoded memory assumptions. See [Transform Patterns](../04-palantir/transform-patterns.md) for guidance on handling large incremental batches. Set Foundry pipeline SLA alerts on the landing zone dataset freshness.

### Data Distribution Shift at Ingestion

**What happens**: the statistical distribution of incoming sensor values changes without any pipeline failure. The pipeline continues running, schemas are valid, expectations pass — but the data itself has shifted. Examples: a regional heatwave raises ambient temperature readings fleet-wide; a firmware update changes sensor calibration causing a systematic offset in pressure readings; a subset of devices degrades gradually, shifting the vibration distribution upward.

**How you detect it**: compute rolling statistics (mean, standard deviation, P5/P50/P95) per sensor on each hourly or daily batch of incoming data. Compare these statistics against a 30-day rolling baseline. Flag sensors where the current batch distribution deviates significantly from the baseline (e.g., mean shifts by more than 2σ of the baseline mean, or P95 exceeds the historical P99).

**Why it matters**: distribution shift at ingestion is the earliest possible signal for model degradation. Anomaly detection models are trained on historical distributions. If the input distribution shifts and the model has not been retrained, detection accuracy degrades — false positives increase (the model flags normal-but-shifted readings as anomalous) or false negatives increase (the model's threshold no longer captures the new failure mode). Catching the shift at ingestion gives the team lead time to investigate before model performance visibly degrades.

**Impact**: this is an **advisory signal**, not a pipeline-stopping event. A detected distribution shift does not halt ingestion or block downstream processing. It generates an alert for the data engineering and ML teams to investigate. The shift may be benign (seasonal change, expected firmware update) or indicative of a data quality problem (sensor calibration drift, device hardware degradation).

**Implementation**: implement as a lightweight scheduled batch transform that runs daily (or hourly during known fleet events like firmware rollouts). The transform reads the latest landing zone partition, computes per-sensor statistics, compares against the 30-day baseline stored in an `ingestion_distribution_stats` dataset, and writes the results (sensor name, current stats, baseline stats, deviation magnitude, shift detected flag) back to that dataset. Alternatively, implement as a Foundry dataset expectation on the landing zone that computes summary statistics and warns (but does not fail the build) when thresholds are exceeded. The `ingestion_distribution_stats` dataset is consumed by the [Monitoring](../03-production/monitoring.md) layer for dashboarding and alerting.

## Capacity Planning

### Current Scale

| Dimension | Value | Basis |
|-----------|-------|-------|
| Devices | 100K | Fleet size |
| Sensors per device | 14 | Per project scope: 14 sensor parameters per device |
| Readings per minute | ~14M | 100K × 14 × 1/min |
| Avg message size | ~300 bytes | JSON payload with 14 sensor fields |
| Ingress rate | ~70 MB/min (~1.2 MB/s) | 14M × 300 bytes |
| Daily volume | ~100 GB (raw JSON) | ~1.2 MB/s × 86400s; Parquet compression reduces stored volume to ~20–30 GB/day |
| Batch records per 5-min materialization | ~70M | 14M/min × 5 min |

### Event Hubs Sizing

| Parameter | Recommended | Rationale |
|-----------|-------------|-----------|
| Partition count | 64 | 14M/64 = ~219K msgs/partition/min. At ~300 bytes each, ~65 KB/s/partition — well within the 1 MB/s/partition limit. 64 provides 15× headroom for burst traffic. |
| Throughput units | 4–8 (standard tier) | Each TU provides 1 MB/s ingress. Steady state needs ~1.2 MB/s; 4 TUs give 4 MB/s (3× headroom). Scale to 8 during reconnection storms. |
| Retention | 7 days | Enough time to recover from extended Foundry outages without data loss. |
| Consumer groups | 2 minimum | One for Foundry streaming dataset, one reserved for debugging/replay. Never use `$Default`. |

### IoT Hub Sizing

| Parameter | Recommended | Rationale |
|-----------|-------------|-----------|
| Tier | S3 (Standard) | S3 supports 300M messages/day per unit. At ~20M messages/day (14M readings consolidated into ~1.4M batched messages/min × 1440 min), one S3 unit is sufficient for steady state. Add units for burst headroom. |
| Units | 2–3 | Provides headroom for reconnection storms. |
| D2C partitions | Must match Event Hubs partition count | IoT Hub's built-in Event Hub endpoint inherits the partition count. Set at hub creation (cannot change later). |

### Foundry Streaming Dataset Sizing

| Parameter | Recommended | Rationale |
|-----------|-------------|-----------|
| Materialization interval | 5 minutes | Balances freshness (meets 2-min SLA with ≤5-min batch window) against transaction metadata overhead. See [Streaming Ingestion](../04-palantir/streaming-ingestion.md). |
| Throughput tier | Medium–Large | Foundry-managed; request Large if consumer lag is sustained. |
| Dead-letter monitoring | Alert > 0.1% of volume | Catches deserialization failures from firmware changes. |

### Retention Strategy: Operational vs Training

The landing zone serves two distinct consumers with different retention needs:

**Hot tier (90 days) — Operational**

The landing zone and its Delta Lake batch view retain 90 days of raw sensor readings. This supports:

- Incremental processing by downstream cleansing and feature engineering transforms.
- Debugging and incident investigation ("what did device X report around the time of the failure?").
- Backfill reruns when a cleansing transform is updated and needs to reprocess recent history.
- Freshness SLA monitoring and consumer lag recovery (Event Hubs retains 7 days; the landing zone extends this to 90).

After 90 days, data is removed from the hot tier to bound storage costs and keep Delta Lake transaction logs manageable.

**Cold tier (3+ years) — Training Archive**

A training archive dataset stores historical sensor readings for long-term use. It is:

- **Format**: Parquet files, columnar, partitioned by `date(timestamp_utc)` and `device_id`. Parquet is chosen over Delta Lake for the cold tier because the archive is append-only and never updated — Delta Lake's ACID transaction overhead is unnecessary.
- **Schema**: identical to Contract 1 (`raw_sensor_readings`). No transformation applied.
- **Population**: a scheduled batch transform snapshots the landing zone weekly, appending the most recent 7 days of data to the archive. The transform is idempotent — re-running it for an already-archived date range produces no duplicates (deduplicated on `event_id`).
- **Append-only**: no deletes or updates. Once data enters the archive, it is immutable.

The cold archive supports:

- **Seasonal retraining**: anomaly detection models must be retrained on data spanning multiple seasons (summer vs. winter operating profiles). 90 days is insufficient; 12–24 months is the minimum for a single full seasonal cycle.
- **Evaluation backtesting**: new model versions are evaluated against historical data to measure whether they improve on previous versions. This requires years of labeled ground-truth data.
- **Regulatory audit**: some jurisdictions require retention of equipment monitoring data for 3+ years. The cold archive satisfies this without burdening the operational hot tier.

### Continual Learning Support: Dataset Snapshots

Models are retrained periodically on updated data. Reproducing a training run requires knowing exactly which data the model was trained on. Delta Lake's time travel capabilities provide this without maintaining separate snapshot copies.

**How it works**:

- The landing zone (hot tier) is stored as a Delta Lake table. Every write creates a new Delta version.
- Training pipelines query the landing zone using `VERSION AS OF <version>` or `TIMESTAMP AS OF <timestamp>` to read a consistent, point-in-time snapshot of the data.
- The Delta version number (or commit timestamp) is recorded as metadata alongside each training run in the experiment tracker. To reproduce the run, re-read the same Delta version.

**Retention policy for time travel**:

- Delta Lake's `delta.deletedFileRetentionDuration` is set to `90 days` on the landing zone, matching the hot tier retention. This means any Delta version created within the past 90 days can be read via time travel.
- For snapshots older than 90 days, the training archive (cold tier) is the fallback. The archive does not support Delta time travel (it is plain Parquet), but it is append-only and immutable — any date range that was archived can be queried deterministically by partition.

**Snapshot key: `timestamp_ingested`, not `timestamp_device`**:

- When constructing a training dataset, filter on `timestamp_ingested` (the server-side receive time from IoT Hub) rather than `timestamp_device` (the device-reported time).
- `timestamp_device` reflects when the device claims the reading occurred. Devices with clock drift, buffered readings, or reconnection replays can produce `timestamp_device` values that are hours or days in the past. Filtering on `timestamp_device` can pull in data that was not actually available at the time the model would have run in production — this is temporal leakage.
- `timestamp_ingested` reflects when the data entered the pipeline and became available for processing. It is the correct proxy for "data availability" and prevents leakage in point-in-time training splits.

### Growth Considerations

Scaling from 100K to 500K devices increases throughput 5× to ~70M readings/min (~6 MB/s ingress). This requires:

- Event Hubs: increase partition count to 128–256, throughput units to 16–32.
- IoT Hub: add S3 units (or move to S3 with auto-scale).
- Foundry: increase streaming dataset throughput tier. Materialization interval may need to stay at 5 min or increase to 10 min to manage transaction size.
- Downstream transforms: test with 5× batch sizes before scaling. Pre-aggregate patterns ([Transform Patterns](../04-palantir/transform-patterns.md)) become even more important at 500K devices.

## Cross-References

- [Data Contracts — Contract 1](../05-architecture/data-contracts.md) — landing zone schema and SLAs
- [Streaming Ingestion](../04-palantir/streaming-ingestion.md) — Foundry streaming dataset configuration
- [ADR-001: Foundry-Native](../05-architecture/adr-001-foundry-native.md) — why all processing stays inside Foundry
- [Streaming vs Batch](./streaming-vs-batch.md) — why we stream data in but process it mostly in batch
- [Schema Management](./schema-management.md) — how the device message format evolves without breaking the pipeline
