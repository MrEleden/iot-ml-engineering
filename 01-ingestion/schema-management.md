# Schema Management

## Why Schema Changes Are Dangerous Here

The ingestion pipeline has one upstream (100K+ devices with heterogeneous firmware) and many downstreams (cleansing, feature engineering, model scoring, alerting). A schema change at the device level propagates through every pipeline stage. If the change is uncontrolled — a firmware update adding a field, a vendor renaming a sensor — it can break cleansing transforms, produce null features, score models on incomplete data, and suppress legitimate alerts.

The risk is amplified by scale: 100K devices don't upgrade firmware simultaneously. During any rollout, the pipeline receives a mix of old-format and new-format messages. The schema must handle this mix without manual intervention.

## Schema Evolution Strategy

This section aligns with the [Schema Evolution Strategy](../05-architecture/data-contracts.md) in the data contracts document. The rules below are specific to the ingestion layer (Contracts 1 and 2).

### Core Principle: Additive Changes Only

Schema changes to the landing zone and clean dataset are always additive. Columns are added, never removed or renamed in-place. This guarantees backward compatibility: any downstream transform written against the current schema continues to work after a schema change.

### Adding a New Sensor

A new sensor is the most common schema change. Example: the fleet adds a `power_consumption` sensor to next-generation units.

**Step 1 — Landing zone (`raw_sensor_readings`)**: add a nullable `power_consumption` column of type `double`. Existing rows (from devices without the sensor) get null. New rows from upgraded devices populate the field. No existing code breaks because the column is nullable and didn't previously exist.

**Step 2 — Streaming dataset schema update**: update the Foundry streaming dataset's JSON deserialization schema to include the new field. Messages without the field deserialize successfully (the field is null). Messages with the field populate it. No dead-letter impact — the JSON deserializer handles absent keys as null for nullable columns.

**Step 3 — Cleansing transform (`clean_sensor_readings`)**: add range validation for the new sensor. Define the valid range (e.g., 0–5000 W for `power_consumption`). Out-of-range values become null with a `RANGE_VIOLATION` quality flag. The transform must be updated and rebuilt — this triggers a full recompute of the clean dataset because the output schema changed. Schedule this during a low-traffic window. See [Transform Patterns — Full Recompute Triggers](../04-palantir/transform-patterns.md) for the implications.

**Step 4 — Feature engineering**: add feature columns to `device_features` (e.g., `power_consumption_mean`, `power_consumption_std`). These are nullable — computed when data is available, null when the device doesn't report the sensor. Models either incorporate the new features (requires retraining) or ignore them (feature selection per model).

**Step 5 — Update the contracts doc**: add the column to [Contract 1 and Contract 2](../05-architecture/data-contracts.md). This is not optional — the contracts doc is the single source of truth for schema.

### Removing a Sensor

Sensor removal is a deprecation process, not a delete.

**Step 1 — Stop populating**: the device firmware update stops sending the sensor value. The column in the landing zone receives null for all readings from updated devices. The column remains in the schema.

**Step 2 — Feature graceful degradation**: feature computation handles all-null windows correctly (the feature value is null). Models trained without the feature continue to work. Models that used the feature must be retrained or fall back to imputation.

**Step 3 — Deprecation period (90 days)**: the column exists but is progressively null as more devices upgrade. During this period, monitor the fraction of non-null values. When it reaches 0% (or an acceptable threshold), the column is a candidate for removal.

**Step 4 — Column removal**: after the 90-day deprecation period, remove the column from the landing zone and clean dataset schemas. This is a breaking change — all downstream transforms that reference the column must be updated first. Treat this as a coordinated migration.

**Step 5 — Update the contracts doc.**

### Changing a Column Type

Column types are never changed in-place. Instead:

1. Add a new column with the target type (e.g., `pressure_high_side_v2` as `float` if the original was `double`).
2. Populate both old and new columns during the migration period.
3. Migrate downstream transforms to the new column one at a time.
4. After all consumers are migrated, deprecate and eventually remove the old column.

This avoids the scenario where a type change silently breaks a downstream transform that casts differently, or where Foundry's incremental checkpointing is invalidated by a schema change it can't reconcile.

### Renaming a Column

Column renames follow the same pattern as type changes: add a new column with the target name, populate both during migration, migrate consumers, deprecate the old name. Never rename in-place.

## Firmware Version Differences

### The Problem

Not all devices report all 14 sensors. The reasons:

- **Older firmware**: pre-2025 units lack vibration sensors. Their messages have 12 fields instead of 14.
- **Regional variants**: some markets don't require humidity sensors. Those devices send 13 fields.
- **Sensor hardware failures**: a device with a broken temperature probe sends null for that sensor indefinitely, which looks identical to "firmware doesn't support this sensor."

The pipeline cannot distinguish between "sensor not supported by this firmware version" and "sensor broken" from the message alone. Both result in null values.

### How It's Handled

**At ingestion (Contract 1)**: all 14 sensor columns are nullable. A message with only 12 fields still deserializes correctly — the two missing fields are null. No special handling is needed at the landing zone level.

**At cleansing (Contract 2)**: the `sensor_null_count` column tracks how many of the 14 sensor fields are null per reading (0–14). This enables downstream filtering — e.g., "only use readings with at least 10 non-null sensors for training data." The cleansing transform also adds a `SENSOR_GAP` quality flag when sensor columns that the device _previously reported_ are suddenly null — i.e., per-sensor unexpected nulls that may indicate a hardware fault rather than a firmware limitation. `SENSOR_GAP` strictly means "one or more sensor columns that this device previously populated are now null in this reading." It does **not** refer to temporal gaps between consecutive readings (e.g., a device going silent for several minutes). Temporal reading gaps should be detected separately by the cleansing Transform via timestamp differences between consecutive readings per device, and flagged as `READING_GAP`.

**Detecting the difference**: to distinguish firmware limitations from hardware failures, the pipeline needs a device capability registry — a mapping of `device_id` → `firmware_version` → `expected_sensors`. This registry lives in the Foundry Ontology as a property of the `Device` object type. The cleansing transform joins against it to determine whether a null represents "expected absence" or "unexpected absence." An unexpected absence gets the `SENSOR_GAP` flag; an expected absence does not.

### Firmware Rollout Monitoring

When a firmware update rolls out across the fleet, monitor:

- **Dead-letter rate**: a spike indicates the new firmware changed the JSON payload format in a way the streaming dataset schema doesn't expect. This is the first signal of a schema problem.
- **`sensor_null_count` distribution shift**: if the average null count drops (new firmware reports more sensors) or rises (new firmware drops a sensor), the distribution will shift. A dataset expectation on null count distribution can detect this.
- **New fields appearing in dead-letter messages**: if the firmware adds a field the streaming dataset schema doesn't include, the field is silently dropped (JSON deserialization ignores unknown fields). The data isn't corrupted, but the new field's data is lost until the schema is updated.

## Schema Enforcement in Foundry

### Dataset Expectations

Foundry dataset expectations are the enforcement mechanism for data contracts. They run as part of the transform build and fail the build if violated. For the ingestion layer:

**Landing zone expectations (Contract 1)**:
- `event_id` is non-null and unique within each transaction
- `device_id` is non-null
- `timestamp_device` is non-null and parseable as a timestamp
- `timestamp_ingested` is non-null and ≥ `timestamp_device` (if device clock is correct)
- All sensor columns, if non-null, are numeric (not strings, not NaN)

**Clean dataset expectations (Contract 2)**:
- `event_id` is unique across the entire dataset (not just per transaction — deduplication guarantee). When duplicates are removed, the retained copy carries a `DUPLICATE_REMOVED` quality flag meaning "this `event_id` appeared more than once; this is the kept copy and the duplicates were discarded." The flag is on the _retained_ row, not on a removed row.
- `device_id` exists in the device registry (post-quarantine)
- `timestamp_utc` is non-null and in UTC
- All sensor columns, if non-null, are within their defined range
- `quality_flags` is non-null (empty array for clean rows)
- `sensor_null_count` matches the actual count of null sensor columns

### What Happens When an Expectation Fails

A failed expectation stops the transform build. The output dataset is not updated. This is the correct behavior — it's better to stop the pipeline and alert operators than to propagate bad data downstream.

Failed builds trigger alerts via Foundry's pipeline monitoring. The alert includes which expectation failed and a sample of violating rows.

**Risk**: if an expectation is too strict (e.g., `event_id` uniqueness across transactions when the source has rare legitimate re-sends), it can block the pipeline for valid data. Tune expectations based on observed data patterns, not theoretical purity.

### Expectation vs Dead-Letter: Different Failure Points

| Failure type | Where caught | What happens |
|---|---|---|
| JSON deserialization failure | Streaming dataset dead-letter | Message routed to sidecar; pipeline continues |
| Schema constraint violation | Dataset expectation on transform | Build fails; output not updated; alert fired |
| Business rule violation (range, registry) | Cleansing transform logic | Row quarantined or nulled + quality flag; pipeline continues |

Dead-letter catches malformed messages before they enter the schema. Expectations enforce schema invariants on data that _is_ schema-conformant. Business rules handle data that passes schema checks but violates domain logic.

## Versioning the Device Message Format

### Current Approach

Device messages do not carry an explicit schema version field. The message format is implicitly versioned by the firmware version, which is available in the IoT Hub device registry (and synced to the Foundry Ontology device registry).

This works when firmware versions map cleanly to payload formats. It breaks when multiple teams modify the payload format within the same firmware version, or when devices self-update their payload format without a firmware version bump.

### Recommended Improvement

Add a `schema_version` field to the device message payload:

```json
{
  "schema_version": "1.2",
  "device_id": "REF-US-W-00042371",
  "timestamp": "2026-02-15T10:32:17.000Z",
  "readings": { ... }
}
```

The `schema_version` field enables:

- **Per-version deserialization**: the streaming dataset (or a downstream transform) can parse different payload versions differently, as long as all versions map to the same landing zone schema.
- **Rollout tracking**: query the landing zone for `schema_version` distribution over time to monitor firmware rollout progress.
- **Breaking change detection**: if `schema_version` changes unexpectedly, the cleansing transform can flag or quarantine those rows for investigation.

This is a device-side change that requires coordination with the firmware team. Until implemented, firmware version (from the device registry) serves as the implicit schema version.

### Data Availability Timestamp for Leakage Prevention

The landing zone carries two timestamps: `timestamp_device` (when the device claims the reading occurred) and `timestamp_ingested` (when IoT Hub received the message). Neither precisely captures when the data became queryable by downstream consumers. A third timestamp addresses this gap.

**`timestamp_available`**: the time at which the reading became available in the batch view — i.e., the commit timestamp of the Delta Lake transaction that materialized the streaming dataset batch containing this reading.

**Distinction from existing timestamps**:

| Timestamp | Set by | Represents | Typical lag from event |
|-----------|--------|------------|------------------------|
| `timestamp_device` | Device firmware | When the sensor reading was taken | 0 (if clock is accurate) |
| `timestamp_ingested` | IoT Hub | When the message was received by the cloud gateway | < 1s – 5s |
| `timestamp_available` | Batch materialization | When the reading became queryable in the landing zone batch view | 1–10 min (materialization interval) |

**Why it matters (Huyen, Ch. 5 — Data Leakage)**: when constructing training datasets, the relevant question is not "when did the event occur?" but "when was the data available to the model in production?" A model trained on readings filtered by `timestamp_device` may include data that, in production, would not yet have arrived at scoring time — especially for devices with buffered uploads, reconnection replays, or significant clock drift. Filtering by `timestamp_available` guarantees that every reading in the training set was actually queryable at the stated time, eliminating this form of temporal leakage.

**Implementation**: the `timestamp_available` column is populated by the streaming dataset's batch materialization process. Each 5-minute materialization commit writes the commit timestamp to all rows in that transaction. In Foundry, this can be implemented as a post-materialization lightweight transform that stamps the Delta commit timestamp onto each row, or by reading the Delta transaction log metadata (`_commit_timestamp`) downstream. The column is added to Contract 1 as a nullable field — null for historical data predating the column's introduction, populated for all new data.

### Version Compatibility Matrix

Maintain a compatibility matrix mapping firmware versions to expected sensor sets:

| Firmware Version | Expected Sensors | Notes |
|---|---|---|
| ≤ 2.x | 12 of 14 | No vibration, no subcooling |
| 3.0–3.4 | 13 of 14 | Added vibration, no subcooling |
| ≥ 3.5 | 14 of 14 | Full sensor suite |

This matrix lives in the device registry (Foundry Ontology) and is used by the cleansing transform to distinguish expected absences from sensor failures.

## Backward Compatibility Guarantees

### For Downstream Consumers

1. **Columns are never removed without a 90-day deprecation period.** A consumer reading `temperature_evaporator` today will still find that column 90 days from now, even if the sensor is being phased out.

2. **Column types never change in-place.** If a type change is needed, a new column is added alongside the old one. Consumers migrate on their own schedule.

3. **Column semantics (units, meaning) never change silently.** If `pressure_high_side` changes from kPa to PSI, a new column `pressure_high_side_psi` is created. The old column continues in kPa.

4. **New columns are always nullable.** Adding a column never makes existing transforms fail — the column is null for historical data and populated for new data. Transforms that don't reference the new column are unaffected.

5. **Quality flags are additive.** New flag values (e.g., a new `FIRMWARE_MISMATCH` flag) may appear in the `quality_flags` array. Consumers that check for specific flags are unaffected; consumers that iterate over all flags should handle unknown values gracefully.

### For Training Data Consumers

Training pipelines have additional schema concerns beyond what operational consumers need:

1. **Minimum viable history per feature.** Each feature column has a date from which it is reliably populated across the fleet. Training on a feature that was added 30 days ago but treating it as available for the full 12-month training window produces a dataset that is mostly null for that feature. The `schema_change_log` dataset (below) provides the metadata needed to construct valid training windows per feature.

2. **`schema_change_log` dataset.** A Foundry dataset that tracks schema evolution events relevant to training data construction:

   | Column | Type | Description |
   |--------|------|-------------|
   | `column_name` | string | The sensor or derived column (e.g., `vibration_compressor`) |
   | `date_added` | date | The date the column was added to the landing zone schema |
   | `date_deprecated` | date (nullable) | The date the column entered deprecation (null if active) |
   | `fleet_coverage_pct` | double | Percentage of active devices reporting non-null values for this column, updated weekly |
   | `min_viable_training_date` | date | The earliest date at which `fleet_coverage_pct` exceeded 80% — the recommended start date for training data that uses this feature |
   | `notes` | string (nullable) | Context: firmware version that introduced the sensor, known calibration changes, etc. |

   This dataset is maintained manually (updated when schema changes occur) and consumed programmatically by training pipelines to validate that requested training windows have adequate coverage.

3. **No backfilling raw data.** When a new sensor column is added, historical rows in the landing zone and training archive remain null for that column. The ingestion layer does not backfill raw data — devices that did not report a sensor at the time of the reading cannot retroactively produce that data. Training pipelines must handle the null-to-populated transition gracefully, either by restricting the training window to post-`min_viable_training_date` or by using imputation strategies documented in the feature engineering layer.

### For the Streaming Dataset

1. **Schema additions don't require consumer restart.** Adding a nullable column to the streaming dataset schema and restarting the streaming dataset does not lose data — Event Hubs retains messages, and the consumer resumes from its last checkpoint.

2. **Schema removals are dangerous.** Removing a column from the streaming dataset schema causes messages with that field to lose data silently (the field is ignored during deserialization). Never remove a column from the streaming dataset without first confirming that no device still sends it.

## Cross-References

- [Data Contracts — Schema Evolution Strategy](../05-architecture/data-contracts.md) — the authoritative schema evolution rules
- [Data Contracts — Contract 1 and Contract 2](../05-architecture/data-contracts.md) — the schemas this document manages changes to
- [Streaming Ingestion](../04-palantir/streaming-ingestion.md) — deserialization behavior and dead-letter routing
- [Transform Patterns — Full Recompute Triggers](../04-palantir/transform-patterns.md) — impact of schema changes on incremental transforms
- [Architecture](./architecture.md) — where schema enforcement sits in the data flow
