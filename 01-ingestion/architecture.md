# IoT Ingestion Architecture

## Overview

A typical IoT ML pipeline ingests data from thousands of sensors, normalizes it into a common schema, and lands it in a data lake for both real-time and batch consumption. The architecture below handles industrial equipment (e.g., refrigeration units, HVAC, turbines) where sensors emit readings every 1–60 seconds.

```
┌──────────────┐     ┌──────────────┐     ┌──────────────────┐     ┌───────────┐
│  IoT Devices │────▶│ Azure IoT Hub│────▶│ Kafka / Event Hub│────▶│ Data Lake │
│  (sensors)   │     │  (gateway)   │     │  (streaming bus) │     │ (landing) │
└──────────────┘     └──────────────┘     └──────────────────┘     └───────────┘
                                                   │
                                                   ▼
                                          ┌──────────────────┐
                                          │ Stream Processor  │
                                          │ (real-time alerts)│
                                          └──────────────────┘
```

## Component Roles

### Azure IoT Hub (Edge Gateway)

IoT Hub acts as the first point of contact for device telemetry. It handles:

- **Device authentication** — each device has a unique identity and symmetric key or X.509 certificate
- **Protocol translation** — devices can speak MQTT, AMQP, or HTTPS; IoT Hub normalizes to a single internal format
- **Message routing** — route messages to different endpoints (Kafka, storage, functions) based on message properties
- **Device twin** — stores desired/reported state per device (firmware version, config, last seen timestamp)

**Key design decision**: IoT Hub should be a thin pass-through, not a processing layer. Don't put business logic here — just authenticate, route, and forward.

### Kafka / Event Hubs (Streaming Bus)

Kafka (or Azure Event Hubs with Kafka API) decouples producers from consumers:

- **Partitioning by device ID** — ensures all messages from a single device land in the same partition, preserving ordering per device
- **Retention** — set to 7+ days to allow replay during pipeline failures or backfill
- **Consumer groups** — separate groups for real-time alerting vs. batch landing vs. monitoring

```
Topic: sensor-telemetry
├── Partition 0: device-001, device-004, device-007 ...
├── Partition 1: device-002, device-005, device-008 ...
└── Partition 2: device-003, device-006, device-009 ...
```

**Partition count**: rule of thumb is 1 partition per 1 MB/s of throughput. For 10,000 devices at 1 msg/sec of ~500 bytes each, that's ~5 MB/s → 8–16 partitions with headroom.

### Landing Zone (Data Lake)

Raw messages land in a structured path in the data lake:

```
raw/
└── sensor_telemetry/
    └── year=2026/
        └── month=02/
            └── day=15/
                └── hour=14/
                    ├── part-00000.parquet
                    └── part-00001.parquet
```

**Why Parquet**: columnar format, efficient compression (snappy), predicate pushdown for time-range queries. Delta Lake adds ACID transactions, schema enforcement, and time travel on top.

## Message Schema

A minimal sensor message:

```json
{
  "device_id": "fridge-unit-0042",
  "timestamp": "2026-02-15T14:30:00.123Z",
  "sensor_type": "temperature",
  "value": -18.4,
  "unit": "celsius",
  "metadata": {
    "site_id": "warehouse-paris-01",
    "firmware": "v2.3.1"
  }
}
```

**Design rules**:
- `device_id` + `timestamp` is the natural key
- Always include `unit` — mixed units across device generations is a common production bug
- `metadata` is semi-structured — new fields can appear as firmware updates roll out (see [Schema Management](./schema-management.md))
- Timestamps must be UTC with millisecond precision

## Throughput Sizing

| Fleet Size | Msg Frequency | Messages/sec | Data/day (raw) | Data/day (Parquet) |
|------------|---------------|-------------|----------------|-------------------|
| 1,000 | 1/min | ~17 | ~1 GB | ~200 MB |
| 10,000 | 1/min | ~167 | ~10 GB | ~2 GB |
| 10,000 | 1/sec | ~10,000 | ~500 GB | ~100 GB |
| 100,000 | 1/sec | ~100,000 | ~5 TB | ~1 TB |

These numbers drive decisions on partition count, Spark cluster sizing, and storage tiering (hot/warm/cold).

## Failure Modes to Design For

1. **Device dropout** — sensors go offline. The pipeline must distinguish "no reading" from "reading of 0". Track last-seen timestamps per device.
2. **Late arrivals** — network delays can push messages minutes or hours late. Use event-time (device timestamp), not processing-time.
3. **Duplicate messages** — IoT Hub guarantees at-least-once delivery. Deduplicate on `(device_id, timestamp)` in the landing layer.
4. **Burst traffic** — fleet-wide firmware updates can cause all devices to reconnect simultaneously. Kafka absorbs the burst; size retention accordingly.
5. **Schema drift** — new sensor types or fields appear as firmware updates. See [Schema Management](./schema-management.md).
