# Ontology Design — Refrigeration Fleet

How we model the refrigeration fleet domain in Foundry's Ontology layer, covering object types, properties, links, online feature serving, and action types for the maintenance workflow.

## Why Ontology

The Ontology exists to make data actionable by non-engineers. A maintenance technician doesn't think in DataFrames — they think in devices, alerts, and work orders. The Ontology maps our processed datasets into these domain concepts so that Workshop dashboards, AIP agents, and OSDK-based field apps can query "show me all devices with anomaly scores above 0.8 in the Chicago region" without writing SQL.

Equally important: the Ontology is how we serve features online. Pre-hydrated properties on the `Device` object type give the model-scoring pipeline and operational apps access to the latest feature values without querying the offline feature store directly.

See [Foundry Platform Reference](./foundry-platform-reference.md) for general Ontology concepts (object types, link types, actions).

## Object Types

### Device

Represents a single refrigeration unit in the fleet.

| Property | Type | Source | Description |
|---|---|---|---|
| `device_id` | string (PK) | Device registry | Unique identifier, e.g., `REF-US-W-00042371` |
| `region` | string | Device registry | Geographic region for routing maintenance crews |
| `site_id` | string | Device registry | Physical site (store, warehouse, distribution center) |
| `model` | string | Device registry | Equipment model number — affects baseline sensor ranges |
| `install_date` | date | Device registry | Commissioning date — age affects failure probability |
| `firmware_version` | string | Device registry | Current firmware — relevant for message format compatibility |
| `last_reading_timestamp` | timestamp | Sensor readings (pre-hydrated) | When the device last reported — null or stale = offline |
| `latest_anomaly_score` | double | Anomaly scores (pre-hydrated) | Most recent composite anomaly score (0.0–1.0) |
| `anomaly_trend` | string | Anomaly scores (pre-hydrated) | `stable`, `increasing`, `decreasing` — direction over last 24h |
| `health_status` | string | Derived | `healthy`, `warning`, `critical`, `offline` — computed from anomaly score + connectivity |
| `avg_temperature_evaporator_24h` | double | Features (pre-hydrated) | 24-hour average evaporator temperature — key operational metric |
| `avg_compressor_current_24h` | double | Features (pre-hydrated) | 24-hour average compressor current — early degradation signal |
| `reading_count_24h` | integer | Features (pre-hydrated) | Number of readings in last 24h — low count signals connectivity issues |

**Backing dataset**: joined view of device registry + latest feature table + latest anomaly scores. Updated by a scheduled Transform that runs hourly (see [Transform Patterns](./transform-patterns.md)).

**Why pre-hydrate features onto Device?** Online feature serving. When a Workshop dashboard loads 500 devices on a map, it reads `Device` properties — it does not run a PySpark job. Pre-hydrating `latest_anomaly_score` and `avg_temperature_evaporator_24h` as properties makes them queryable at Ontology speed (milliseconds) rather than feature store speed (seconds to minutes).

### Site

A single device tells you whether one refrigerator is failing. A site tells you whether a store is at risk. Maintenance crews are dispatched to physical locations, not individual serial numbers — a technician drives to a warehouse, not to a compressor. The Site object type models this physical hierarchy so that Workshop dashboards can answer fleet management questions ("show me all devices at Site X"), aggregate anomaly signals at the location level ("this site has 3 devices flagged"), and route maintenance crews to the right place.

| Property | Type | Source | Description |
|---|---|---|---|
| `site_id` | string (PK) | Site registry | Unique site identifier, e.g., `SITE-US-CHI-0042` |
| `site_name` | string | Site registry | Human-readable name, e.g., "Chicago Distribution Center 3" |
| `address` | string | Site registry | Full street address |
| `region` | string | Site registry | Geographic region — matches Device `region` for filtering |
| `site_type` | string | Site registry | `store`, `warehouse`, `distribution_center` — determines baseline expectations (a warehouse has more devices than a store) |
| `device_count` | integer | Pre-hydrated | Total devices registered at this site |
| `devices_in_alert` | integer | Pre-hydrated | Number of devices with at least one open alert — the single most important number for a site manager |
| `avg_anomaly_score` | double | Pre-hydrated | Mean `latest_anomaly_score` across all devices at the site — useful for ranking sites by urgency |
| `health_status` | string | Derived | `healthy`, `degraded`, `critical` — derived from `devices_in_alert` relative to `device_count` and max anomaly score at site |
| `site_manager` | string | Site registry | Name or ID of the responsible site manager |
| `maintenance_contact` | string | Site registry | Primary maintenance contact (phone or email) for dispatch |

**Backing dataset**: joined view of site registry + per-site device aggregates. The aggregation Transform groups Device-level metrics by `site_id` and computes `device_count`, `devices_in_alert`, and `avg_anomaly_score`. Updated hourly alongside the Device pre-hydration Transform (see [Transform Patterns](./transform-patterns.md)).

**Why pre-hydrate aggregates onto Site?** Same reason as Device pre-hydration: a Workshop dashboard that lists 500 sites needs `devices_in_alert` as a property, not a live aggregation query over 100K devices. The hourly refresh is sufficient — site-level health doesn't change minute-to-minute.

### SensorReading

Individual sensor readings from the fleet. High-volume — this object type exists primarily for drill-down and debugging, not for dashboards that list thousands of readings.

| Property | Type | Source | Description |
|---|---|---|---|
| `reading_id` | string (PK) | Derived | Composite key: `{device_id}_{timestamp}_{sensor_type}` |
| `device_id` | string | Ingestion | Device that produced this reading |
| `timestamp` | timestamp | Ingestion | When the reading was taken (UTC) |
| `sensor_type` | string | Ingestion | One of the 14 sensor parameters (see scope) |
| `value` | double | Ingestion | Sensor measurement value |
| `unit` | string | Ingestion | Measurement unit (celsius, kPa, amps, etc.) |
| `quality` | string | Ingestion | `good`, `suspect`, `bad` — set by device or quality checks |

**Backing dataset**: the materialized streaming dataset after quality tagging. Partitioned by `date` and `hour` (see [Streaming Ingestion](./streaming-ingestion.md)).

**Note on format**: The upstream `clean_sensor_readings` dataset uses a wide format (one row per device per timestamp, with 14 sensor columns — see [Data Contracts](../05-architecture/data-contracts.md) Contract 2). A pivot Transform converts this wide-format dataset into the narrow-format SensorReading backing dataset, producing one row per device–timestamp–sensor combination. The narrow format is the correct choice for the Ontology: each sensor reading becomes a separate object, which enables per-sensor Time Series properties on the Device object type and supports fine-grained filtering by `sensor_type`.

**Caution**: do not enable full Ontology indexing on this object type. At 14M readings/min, full indexing would overwhelm the Ontology sync pipeline. Instead, configure time-bounded indexing (last 7 days) or use the backing dataset directly for historical queries.

### Alert

Raised when a device's anomaly score crosses a threshold or a specific sensor reading violates a known safety limit.

| Property | Type | Source | Description |
|---|---|---|---|
| `alert_id` | string (PK) | Generated | UUID |
| `device_id` | string | Anomaly pipeline | Device that triggered the alert |
| `alert_type` | string | Anomaly pipeline | `ANOMALY_DETECTED`, `ANOMALY_SUSTAINED`, `ANOMALY_ESCALATED`, `ANOMALY_RESOLVED` |
| `trigger_category` | string | Anomaly pipeline | What triggered the alert: `anomaly_score_high`, `sensor_out_of_range`, `device_offline`, `pattern_anomaly` |
| `severity` | string | Anomaly pipeline | `LOW`, `MEDIUM`, `HIGH`, `CRITICAL` |
| `title` | string | Anomaly pipeline | Human-readable summary, e.g., "Compressor vibration anomaly detected on device X" |
| `created_at` | timestamp | Anomaly pipeline | When the alert was created |
| `first_detected_at` | timestamp | Anomaly pipeline | When the anomaly was first detected (may be earlier than `created_at` if sustained) |
| `resolved_at` | timestamp | Action (manual) | When a technician acknowledged or resolved the alert |
| `status` | string | Workflow | `open`, `acknowledged`, `in_progress`, `resolved`, `false_positive` |
| `model_id` | string | Anomaly pipeline | Model that triggered this alert |
| `anomaly_score` | double | Anomaly pipeline | Score at time of alert creation |
| `top_contributors` | array\<string\> | Anomaly pipeline | Feature names most responsible for the anomaly (top 5). Inherited from `model_scores`. |
| `description` | string | Anomaly pipeline | Detailed description: which sensors are abnormal, trend direction, estimated time to action |
| `suppressed` | boolean | Anomaly pipeline | `true` if suppressed by deduplication rules (same device + same failure mode within 4h) |
| `ontology_action_id` | string | Workflow | If an Ontology Action (e.g., work order creation) was triggered, its ID. Null if alert-only. |
| `escalation_level` | string | Workflow | `L1` (site technician), `L2` (regional engineer), `L3` (vendor/specialist). Starts at `L1`. |
| `escalated_at` | timestamp | Workflow | When the alert was last escalated — null if still at `L1` |
| `escalated_to` | string | Workflow | Person or team the alert was escalated to |
| `sla_deadline` | timestamp | Alert rules | Timestamp by which the alert must be acknowledged. Computed at creation: `created_at` + SLA window (e.g., 30 min for critical, 4h for warning). |
| `sla_breached` | boolean | Derived | `true` if `status` is still `open` and current time exceeds `sla_deadline`. Enables SLA compliance dashboards. |
| `correlated_alert_ids` | array\<string\> | Alert correlation | IDs of related alerts — alerts on other devices at the same site within the same time window, or alerts sharing the same dominant `top_contributors`. Helps distinguish site-level issues (refrigerant supply problem) from individual device failures. |

**Backing dataset**: output of the alert generation Transform, which reads anomaly scores and applies threshold logic. Escalation and SLA properties are written by the Escalate Alert and Acknowledge Alert action types.

### MaintenanceEvent

Records of maintenance actions taken on devices — both proactive (triggered by alerts) and scheduled.

| Property | Type | Source | Description |
|---|---|---|---|
| `event_id` | string (PK) | Generated | UUID |
| `device_id` | string | Maintenance system | Device serviced |
| `event_type` | string | Maintenance system | `preventive`, `corrective`, `emergency` |
| `scheduled_date` | date | Maintenance system | When the maintenance is/was scheduled |
| `completed_date` | date | Maintenance system | When the maintenance was actually completed |
| `technician_id` | string | Maintenance system | Assigned technician |
| `work_description` | string | Maintenance system | What was done |
| `parts_replaced` | array\<string\> | Maintenance system | Components replaced |
| `triggered_by_alert_id` | string | Workflow | Alert that prompted this maintenance (null for scheduled maintenance) |
| `pre_maintenance_anomaly_score` | double | Anomaly pipeline | Anomaly score before maintenance — validates whether the model was right |
| `post_maintenance_anomaly_score` | double | Anomaly pipeline | Anomaly score after maintenance — measures repair effectiveness |

**Backing dataset**: synced from external maintenance management system via Foundry connector, enriched with anomaly scores by a Transform.

### AnomalyScore

Per-device, per-run anomaly scores from each model variant. Kept as a separate object type (rather than only pre-hydrating onto Device) to preserve historical scoring data for model evaluation.

| Property | Type | Source | Description |
|---|---|---|---|
| `score_id` | string (PK) | Generated | Composite: `{device_id}_{model_version}_{scored_at}` |
| `device_id` | string | Scoring pipeline | Device scored |
| `window_start` | timestamp | Scoring pipeline | Start of the feature aggregation window (UTC) |
| `window_end` | timestamp | Scoring pipeline | End of the feature aggregation window (UTC) |
| `model_id` | string | Scoring pipeline | Model identifier (e.g., `isolation_forest_v3`, `autoencoder_v2`) |
| `model_version` | string | Scoring pipeline | Model version tag |
| `granularity` | string | Scoring pipeline | `fleet`, `cohort`, or `device` — scoring granularity level |
| `scored_at` | timestamp | Scoring pipeline | When the score was computed |
| `anomaly_score` | double | Scoring pipeline | 0.0 (normal) to 1.0 (highly anomalous) |
| `anomaly_flag` | boolean | Scoring pipeline | `true` if `anomaly_score` exceeds the model's threshold |
| `threshold_used` | double | Scoring pipeline | Threshold applied for this scoring run — stored for auditability |
| `top_contributors` | array\<string\> | Scoring pipeline | Feature names most responsible for the anomaly score (top 5) |
| `contributor_scores` | array\<double\> | Scoring pipeline | Contribution magnitude for each feature in `top_contributors` (same order) |
| `sensor_completeness` | double | Scoring pipeline | Fraction of non-null sensor readings in the scored window (0.0–1.0). Low values → lower confidence. |
| `component_scores` | map\<string, double\> | Scoring pipeline | Per-sensor-group sub-scores (e.g., `thermal: 0.7`, `mechanical: 0.3`) |
| `feature_snapshot` | string | Scoring pipeline | JSON blob of input features at scoring time — for debugging and explainability |

**Backing dataset**: output of the batch scoring Transform (see [Model Integration](./model-integration.md)).

## Link Types

| Link | From | To | Cardinality | Description |
|---|---|---|---|---|
| `device_readings` | Device | SensorReading | one-to-many | All readings for a device |
| `device_alerts` | Device | Alert | one-to-many | All alerts for a device |
| `device_maintenance` | Device | MaintenanceEvent | one-to-many | Maintenance history for a device |
| `device_anomaly_scores` | Device | AnomalyScore | one-to-many | All anomaly scores for a device |
| `alert_maintenance` | Alert | MaintenanceEvent | one-to-one | Maintenance event triggered by an alert |
| `alert_anomaly_score` | Alert | AnomalyScore | many-to-one | Anomaly score that triggered the alert |
| `site_devices` | Site | Device | one-to-many | All devices registered at a site. Joined on `site_id`. |
| `site_alerts` | Site | Alert | one-to-many | All open and historical alerts for devices at a site. Resolved via `site_id` on Alert's backing dataset (joined through Device). |

### Link Implementation

Links in Foundry are defined by foreign key relationships between backing datasets. For example, `device_readings` uses `device_id` on both `Device` and `SensorReading`. The link is resolved at query time by the Ontology — no denormalization required.

**Pitfall**: the `device_readings` link with one-to-many cardinality on a high-volume object type (SensorReading) means that traversing this link on a single device can return millions of rows. Workshop widgets that traverse this link must include a time filter — unbounded traversal will time out.

**Note on `site_alerts`**: this link is resolved indirectly. Alerts reference a `device_id`, and Devices reference a `site_id`. The `site_alerts` link is backed by a Transform that denormalizes `site_id` onto the Alert backing dataset — this avoids a two-hop join at query time which the Ontology does not support natively.

## Online Feature Serving via Pre-Hydrated Properties

### The Pattern

The offline feature store is a Foundry dataset (Delta Lake). The online feature store is the Ontology itself. Pre-hydrated properties on the `Device` object type serve as the online feature cache.

Flow:
1. **Feature computation** (hourly): Transforms compute rolling stats, anomaly scores, health metrics → write to feature datasets
2. **Pre-hydration** (hourly): a Transform joins feature datasets with the device registry and writes the result to the `Device` backing dataset
3. **Online serving** (on-demand): Workshop, AIP agents, and OSDK apps query `Device.latest_anomaly_score` as a property — no feature store API needed

### Staleness Guarantee

Features are at most 1 hour stale (bounded by the pre-hydration Transform schedule). For the refrigeration use case, this is acceptable — maintenance decisions don't need sub-minute feature freshness. If sub-minute freshness is ever needed for specific properties, consider Foundry's streaming Ontology sync (available for object types backed by streaming datasets).

### What to Pre-Hydrate

Only properties that are frequently queried by apps and dashboards. Do not pre-hydrate the full feature vector (50+ features) — that bloats the `Device` object and increases Ontology sync time. Pre-hydrate:

- Top-level health indicators (`latest_anomaly_score`, `health_status`, `anomaly_trend`)
- Key operational metrics (`avg_temperature_evaporator_24h`, `avg_compressor_current_24h`)
- Connectivity signals (`last_reading_timestamp`, `reading_count_24h`)

The full feature vector remains in the offline feature dataset for model training and batch scoring.

### How Time Series Complements Pre-Hydrated Scalars

Pre-hydrated properties answer "what is the current state?" — they're scalars optimized for listing many objects at once (fleet overview, site ranking, alert triage). Time Series properties answer "what happened over time?" — they're historical sequences optimized for drilling into a single object (device deep-dive, trend analysis). Both serve features online; they serve different access patterns. See the Time Series Properties section below for the full design.

## Time Series Properties

### Why Time Series on Ontology Objects

Pre-hydrated scalar properties (`latest_anomaly_score: 0.82`) serve the fleet overview — a dashboard that shows 500 devices needs one number per device. But when an operator clicks into a specific device, they need history: "show me evaporator temperature for the last 7 days" or "is the anomaly score trending up?". Without Time Series properties, answering this requires querying the backing dataset directly, which means writing Transforms or Functions to extract and format the data for Workshop widgets.

Foundry's Ontology supports **Time Series properties** on object types. A Time Series property is not a single value — it's a queryable sequence of `(timestamp, value)` pairs bound directly to an object. Workshop time series widgets can bind to these properties without any intermediate query layer.

### Time Series on Device: Sensor Parameters

Each of the 14 sensor parameters can be exposed as a Time Series property on the Device object type:

| Time Series Property | Unit | Description |
|---|---|---|
| `ts_temperature_evaporator` | °C | Evaporator coil temperature over time |
| `ts_temperature_condenser` | °C | Condenser coil temperature over time |
| `ts_temperature_ambient` | °C | Ambient environment temperature over time |
| `ts_temperature_discharge` | °C | Compressor discharge line temperature over time |
| `ts_temperature_suction` | °C | Compressor suction line temperature over time |
| `ts_pressure_high_side` | kPa | High-side refrigerant pressure over time |
| `ts_pressure_low_side` | kPa | Low-side refrigerant pressure over time |
| `ts_current_compressor` | A | Compressor motor current draw over time |
| `ts_vibration_compressor` | mm/s | Compressor vibration level over time |
| `ts_humidity` | %RH | Relative humidity over time |
| `ts_door_open_close` | boolean (0/1) | Door state over time |
| `ts_defrost_cycle` | boolean (0/1) | Defrost cycle state over time |
| `ts_superheat` | K | Superheat over time |
| `ts_subcooling` | K | Subcooling over time |

### Time Series on Device: Anomaly Score

The `anomaly_score` is also exposed as a Time Series property on Device:

| Time Series Property | Unit | Description |
|---|---|---|
| `ts_anomaly_score` | 0.0–1.0 | Composite anomaly score over time |

This enables Workshop trend charts like "anomaly score trending up over the last 48h" without querying the `AnomalyScore` backing dataset. Operators can overlay `ts_anomaly_score` with sensor time series to visually correlate score spikes with sensor behavior — a critical workflow for understanding why a device was flagged.

### How Time Series Properties Are Backed

Each Time Series property is backed by a **time series–synced dataset** with three columns:

| Column | Type | Description |
|---|---|---|
| `device_id` | string | Primary key reference to the Device object |
| `timestamp` | timestamp | When the value was recorded |
| `value` | double | The measurement value |

The Ontology sync pipeline reads this backing dataset and indexes the `(timestamp, value)` pairs per `device_id` into the Time Series store. When a Workshop widget requests `Device[REF-US-W-00042371].ts_temperature_evaporator` for the last 7 days, the Ontology serves it directly from the Time Series index — no Spark job runs.

The backing dataset for sensor time series is derived from `clean_sensor_readings` via a Transform that pivots the wide-format reading (14 columns per row) into narrow-format per-sensor datasets. The backing dataset for `ts_anomaly_score` is the `model_scores` dataset filtered to the primary model.

### When to Use Each: Scalar vs Time Series

| Access Pattern | Use | Why |
|---|---|---|
| Fleet overview: list 500 devices with current status | Pre-hydrated scalar (`latest_anomaly_score`) | One value per device, loaded in bulk — Ontology returns 500 scalars in milliseconds |
| Site ranking: sort 200 sites by urgency | Pre-hydrated scalar (`devices_in_alert`, `avg_anomaly_score`) | Aggregate values, no time dimension needed |
| Device deep-dive: show temperature trend for one device | Time Series (`ts_temperature_evaporator`) | Historical sequence, one device at a time — Time Series index serves this efficiently |
| Alert investigation: correlate anomaly score with sensor behavior | Time Series (`ts_anomaly_score` + sensor time series) | Overlay multiple time series on a single chart |
| Model evaluation: compare predicted vs actual over time | Time Series (`ts_anomaly_score`) + backing dataset for ground truth | Trend analysis over days/weeks |

### Storage and Query Performance

At fleet scale, the Time Series data volume is significant:

- **14 sensor time series × 100K devices × 1 reading/min = ~2B data points/day**
- Plus `ts_anomaly_score` (scored hourly, not per-minute): 100K × 24 = 2.4M points/day

Foundry handles this through several mechanisms:

**Time-bounded retention**: configure the Time Series sync to retain only the last N days of data in the Ontology Time Series index. Raw data older than the retention window remains in the backing dataset (queryable via Transforms/Functions) but is evicted from the Time Series index. Recommended retention:
- Sensor time series: **30 days** in the Ontology index. Most operational queries look at the last 7–14 days. Longer historical analysis uses the backing dataset directly.
- Anomaly score time series: **90 days** in the Ontology index. Model evaluation workflows look at longer windows.

**Downsampling for long-range queries**: querying 30 days of 1-minute data for one device returns ~43,200 points — too many for a chart widget. Foundry Time Series supports automatic downsampling: when a query spans a long range, the response returns pre-aggregated data (e.g., 5-minute or 1-hour averages) instead of raw points. Workshop widgets handle this transparently.

**Time Series resolution tiers**:

| Tier | Resolution | Retention | Use Case |
|---|---|---|---|
| Raw | 1-minute | 7 days | Recent troubleshooting: "what happened in the last 4 hours?" |
| Medium | 5-minute average | 30 days | Operational trends: "show me the last 2 weeks" |
| Coarse | 1-hour average | 90 days | Long-range analysis: "compare this month to last month" |

Pre-aggregation is handled by a scheduled Transform that reads raw sensor data and writes 5-minute and 1-hour rollups to separate backing datasets. The Time Series property can be configured to read from the appropriate tier based on the query's time range.

**Practical limits**: at 2B points/day with 30-day retention, the total indexed volume is ~60B points. Foundry's Time Series infrastructure (built on a columnar time series store, not the general Ontology index) is designed for this scale. However, avoid enabling Time Series properties on the `SensorReading` object type — that would attempt to index the same data twice (once as object properties, once as time series) with no benefit.

### Time Series Sync Lag

Time Series properties are not real-time. The sync pipeline from backing dataset to Time Series index introduces lag:

- **Batch-backed time series** (sensor data via hourly Transform): up to 1 hour lag. Acceptable for most operational views.
- **Streaming-backed time series** (if backed by a streaming dataset): 1–5 minutes lag. Useful for near-real-time device monitoring dashboards, but requires the backing dataset to be a streaming dataset (see [Streaming Ingestion](./streaming-ingestion.md)).

For the current architecture, sensor time series are batch-backed (hourly refresh). If near-real-time sensor trends become a requirement, the backing dataset can be switched to a streaming dataset without changing the Ontology schema — only the sync pipeline configuration changes.

## Action Types

### Acknowledge Alert

**Trigger**: Maintenance operator reviews an alert in a Workshop dashboard.

| Parameter | Type | Description |
|---|---|---|
| `alert_id` | string | Alert to acknowledge |
| `operator_notes` | string | Operator's assessment |

**Effect**: Updates `Alert.status` to `acknowledged`. Logs the operator action in the audit trail.

### Schedule Maintenance

**Trigger**: Operator decides a device needs maintenance based on alert or anomaly trend.

| Parameter | Type | Description |
|---|---|---|
| `device_id` | string | Device to service |
| `event_type` | string | `preventive` or `corrective` |
| `scheduled_date` | date | When to perform maintenance |
| `triggered_by_alert_id` | string (optional) | Alert that prompted this, if any |
| `priority` | string | `low`, `medium`, `high`, `emergency` |

**Effect**: Creates a new `MaintenanceEvent` object. If `triggered_by_alert_id` is provided, updates the corresponding `Alert.status` to `in_progress`. Optionally triggers a notification to the assigned technician's mobile app via OSDK webhook.

### Mark False Positive

**Trigger**: Technician inspects a device and determines the anomaly alert was incorrect.

| Parameter | Type | Description |
|---|---|---|
| `alert_id` | string | Alert to mark |
| `reason` | string | Why it's a false positive (free text) |

**Effect**: Updates `Alert.status` to `false_positive`. Writes the reason to a feedback dataset that model retraining pipelines can use — false positive labels are the most valuable signal for improving unsupervised models (see [Model Integration](./model-integration.md)).

### Resolve Alert

**Trigger**: Maintenance is complete and the device is operating normally.

| Parameter | Type | Description |
|---|---|---|
| `alert_id` | string | Alert to resolve |
| `maintenance_event_id` | string | Associated maintenance event |
| `resolution_notes` | string | What was done |

**Effect**: Updates `Alert.status` to `resolved` and `Alert.resolved_at` to current timestamp. Records `post_maintenance_anomaly_score` on the `MaintenanceEvent` from the latest anomaly score.

### Escalate Alert

**Trigger**: An alert has not been resolved within its SLA window, or a technician determines the issue requires a higher skill level.

| Parameter | Type | Description |
|---|---|---|
| `alert_id` | string | Alert to escalate |
| `target_level` | string | Target escalation level: `L2` or `L3` |
| `escalated_to` | string | Person or team receiving the escalation |
| `reason` | string | Why the alert is being escalated (free text) |

**Effect**: Updates `Alert.escalation_level` to `target_level`, sets `Alert.escalated_at` to current timestamp, and writes `Alert.escalated_to`. If escalating to `L3`, triggers an automatic notification to the vendor/specialist team via OSDK webhook. The escalation is logged in the audit trail for SLA compliance reporting.

**Automation**: an AIP Logic function can auto-escalate alerts where `sla_breached` is `true` and `escalation_level` is still `L1` — this ensures no alert silently breaches SLA without visibility.

## Related Documents

- [Streaming Ingestion](./streaming-ingestion.md) — the backing dataset for `SensorReading` objects
- [Transform Patterns](./transform-patterns.md) — how feature datasets and pre-hydration Transforms are structured
- [Model Integration](./model-integration.md) — how anomaly scores are computed and written to the `AnomalyScore` backing dataset
- [Foundry Platform Reference](./foundry-platform-reference.md) — general Ontology concepts
