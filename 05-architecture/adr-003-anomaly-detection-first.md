# ADR-003: Unsupervised Anomaly Detection First

## Status

Accepted

## Context

We want to predict refrigeration equipment failures 12–24 hours before they happen across 100K+ devices. The fundamental challenge: **we have no labeled failure data.**

The fleet generates ~14M sensor readings per minute, but none of those readings are tagged with "this device failed 12 hours later" or "this reading is normal." Maintenance records exist but are inconsistent — different sites use different ticketing systems, failure descriptions are free-text, and the mapping between a maintenance event and the sensor readings that preceded it is ambiguous (was the failure gradual or sudden? when exactly did the anomaly start?).

Two approaches are possible:

1. **Supervised failure prediction**: train a classifier that predicts "will this device fail in the next 12–24 hours?" Requires labeled examples of pre-failure and normal operation.
2. **Unsupervised anomaly detection**: learn what "normal" looks like and flag deviations. Requires no labels — only sufficient data to characterize normal behavior.

### ML Approach Requirements

- Must work at three granularity levels:
  - **Fleet-wide**: what's abnormal compared to the entire fleet?
  - **Cohort**: what's abnormal for this type/model/location of device?
  - **Per-device**: what's abnormal compared to this specific device's history?
- Must compare multiple approaches: statistical methods, tree-based methods, deep learning
- Must provide interpretable outputs (which sensors are driving the anomaly?)

## Decision

Start with unsupervised anomaly detection. Do not attempt supervised failure prediction until we have accumulated enough anomaly detections — confirmed by field engineers — to build a labeled dataset.

### Concrete Model Strategy

**Phase 1 — Unsupervised anomaly detection (current)**

Three model families, evaluated against each other:

| Model Family | Method | Granularity | Strengths | Weaknesses |
|-------------|--------|-------------|-----------|------------|
| **Statistical** | Z-score, Mahalanobis distance | Fleet, Cohort | Fast, interpretable, low compute. Good baseline. | Assumes distributional shape. Misses complex multi-sensor interactions. |
| **Clustering** | DBSCAN | Cohort | Discovers dense regions of normal behavior; points outside clusters are anomalies. No assumption on distribution shape. | Sensitive to distance metric and epsilon parameter. Requires cohort-level grouping for meaningful cluster structure. |
| **Tree-based** | Isolation Forest | Fleet, Cohort, Device | Handles high-dimensional features well. No distributional assumptions. Fast training. | Less interpretable than statistical. Sensitivity to contamination ratio. |
| **Deep learning** | Autoencoder (reconstruction error) | Cohort (Phase 1) | Learns complex nonlinear patterns. Can detect subtle multi-sensor degradation. | Requires more data per device. Harder to interpret. Higher compute cost. Per-device granularity deferred to Phase 2 due to fleet-scale compute constraints. |

Each model operates on the `device_features` schema defined in [Data Contracts](./data-contracts.md). All three produce the same `model_scores` output schema — a normalized anomaly score between 0 and 1.

**Granularity levels** determine what "normal" means:

- **Fleet-wide**: one model trained on features aggregated across all devices. Catches devices that are outliers relative to the fleet. Good for detecting gross failures but blind to device-specific normal ranges.
- **Cohort**: one model per device cohort (grouped by model, location climate, or age). Catches devices that are outliers within their peer group. More sensitive than fleet-wide because the "normal" baseline is more specific.
- **Per-device**: one model per device, trained on that device's own history. Catches drift from the device's individual baseline. Most sensitive but requires sufficient history (≥30 days) and is expensive to maintain at 100K devices.

Not all model families run at all granularity levels. Statistical methods are cheap enough for all three. Autoencoders are too expensive for per-device at fleet scale in Phase 1.

**Phase 2 — Pseudo-labeled supervised models (future)**

Once anomaly detections are confirmed or rejected by field engineers over 3–6 months, we accumulate a labeled dataset:
- Confirmed anomaly + subsequent failure → positive label (pre-failure)
- Confirmed anomaly + no failure → false positive (normal, mislabeled by anomaly model)
- No anomaly + no failure → negative label (normal)

This labeled dataset enables supervised models that directly predict "failure within 12–24 hours" rather than "this looks unusual." Supervised models will be more precise (fewer false positives) because they learn the specific patterns that precede failures, not just anything unusual.

**The anomaly detection system becomes the labeling engine for the supervised system.**

## Consequences

### Benefits

**Works immediately without labels.** We can deploy anomaly detection models as soon as we have 30 days of sensor data. No need to wait months or years for labeled failure events to accumulate. Time-to-value is weeks, not quarters.

**Catches unknown failure modes.** Supervised models can only predict failure types present in the training data. Anomaly detection catches any deviation from normal — including failure modes we haven't seen yet. For a new fleet deployment, this is critical: we don't know all the ways this equipment can fail.

**Generates training data for future supervised models.** Every anomaly detection that a field engineer confirms or rejects is a labeled example. Over 3–6 months, we build a labeled dataset organically through operations rather than through a separate, expensive labeling project.

**Multi-model comparison reduces blind spots.** Different model families have different strengths (see table above). Running all three and comparing their outputs means a failure mode missed by one model may be caught by another. When multiple models agree, confidence is higher — this drives the [severity classification](./data-contracts.md) in the alert pipeline.

**Three granularity levels catch different problems.** A device failing in a way that's common for other devices won't be caught by fleet-wide models (it looks normal at fleet scale). Per-device models catch it because it's a change from that device's baseline. Conversely, a fleet-wide model catches systematic issues (e.g., a bad firmware update affecting all devices) that per-device models might not flag if the change is gradual.

### Costs and Risks

**Higher false positive rate than supervised models.** Anomaly detection flags anything unusual — including operationally irrelevant anomalies (seasonal temperature changes, planned maintenance events, sensor calibration shifts). Field engineers will receive alerts that aren't real failures.

Mitigation: conservative thresholds initially (high precision, lower recall). Multi-model agreement required for `HIGH`/`CRITICAL` severity. Anomaly score thresholds tuned iteratively based on field feedback. Known non-failure patterns (defrost cycles, door openings) are excluded via feature engineering before scoring.

**No direct failure probability estimate.** Anomaly detection says "this is unusual" — not "this device will fail in 14 hours." The anomaly score correlates with failure risk but isn't calibrated to it. Operators must interpret scores as "investigate this" rather than "this will fail."

Mitigation: clear communication in dashboards and alerts. Anomaly scores are presented with contributing sensors and trend direction, not as failure predictions. Phase 2 supervised models will provide calibrated failure probabilities.

**Computational cost of running three model families.** Training and scoring statistical models, Isolation Forests, and Autoencoders across fleet/cohort/device granularities is more compute than running a single supervised model.

Mitigation: not all combinations run at all granularities (see table). Statistical models are cheap. Isolation Forest is moderate. Autoencoders are limited to cohort-level in Phase 1. Total compute is manageable within Foundry's batch scoring budget — see [System Overview](./system-overview.md) for volume estimates.

**Per-device models don't work for new devices.** A device needs ≥30 days of history before a per-device model can establish a baseline. During onboarding, new devices are scored only by fleet-wide and cohort models.

Mitigation: cohort models provide reasonable anomaly detection for new devices if we know the device's model/type. Fleet-wide models are the fallback. Per-device models activate automatically after the history threshold is met.

**Model drift.** "Normal" behavior changes over time — seasonal effects, equipment aging, operational changes. Models trained on historical data may flag normal seasonal shifts as anomalies.

Mitigation: periodic retraining (monthly for statistical/tree-based, quarterly for autoencoders). Seasonal features (day of year, ambient temperature baseline) included in the feature set to make models season-aware. See [Data Quality & Monitoring](../03-production/data-quality.md) for drift detection.

## Alternatives Considered

### Start with Supervised Failure Prediction

Would provide more precise predictions with calibrated failure probabilities. Rejected because:
- We have no labeled failure data and building a labeled dataset from maintenance records would take 3–6 months of data engineering with uncertain quality
- Even with labels, supervised models can only predict failure modes present in the training data — they'd miss novel failure types
- The anomaly detection approach generates labeled data as a byproduct, giving us a path to supervised models without a separate labeling effort

### Rule-Based Thresholds Only

Simple threshold rules (e.g., "alert if compressor vibration > 15 mm/s") are easy to implement and interpret. Rejected as the primary approach because:
- Cannot detect multi-sensor interaction patterns (e.g., current rising while temperature spread decreases — an efficiency degradation that no single sensor exceeds its threshold)
- Cannot adapt to device-specific baselines (a vibration level that's normal for one compressor model is alarming for another)
- Threshold tuning is manual and doesn't scale to 100K devices

However, threshold rules are used in the [streaming scoring path](./adr-002-batch-plus-streaming.md) for acute anomaly detection where speed matters more than sophistication.

### Transfer Learning from Similar Fleets

Train supervised models on labeled data from a similar fleet and fine-tune on our fleet. Rejected because:
- We don't have access to a similar fleet's labeled data
- Equipment model heterogeneity makes transfer learning risky (the failure patterns of one refrigeration system model don't necessarily apply to another)
- If we obtain such data in the future, it's an enhancement to the current approach, not a prerequisite

## Cross-References

- [System Overview](./system-overview.md) — where model scoring fits in the end-to-end pipeline
- [Data Contracts](./data-contracts.md) — `model_input`, `model_scores`, and `device_alerts` schemas that all model types produce
- [ADR-002: Batch + Streaming](./adr-002-batch-plus-streaming.md) — how these models are scored (batch for full ML, streaming for thresholds)
- [Modeling](../06-modeling/) — detailed model implementation and evaluation
- [Feature Engineering](../02-feature-engineering/) — features consumed by these models
