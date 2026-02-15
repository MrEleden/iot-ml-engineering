# Evaluation Framework

## The Fundamental Challenge

We cannot compute precision, recall, F1, or any classification metric because we have no ground truth labels. No one has reliably tagged which sensor readings preceded a failure and which were normal. This is not a gap we can close quickly — maintaining 100K+ refrigeration devices across many sites with inconsistent ticketing systems means labeled data accumulates slowly and unreliably.

This is not a failure of data collection. It's the nature of the problem: refrigeration failures are rare, gradual, and ambiguous. A technician replaces a compressor — was that because the compressor was failing, or a preventive replacement? The sensor readings 12 hours before don't carry a label.

Everything in this document is a workaround. None of these evaluation strategies are as reliable as precision/recall on a labeled test set. But together, they provide enough signal to make model comparison and promotion decisions with reasonable confidence. Be honest about this limitation in every model report and stakeholder communication — see [Model Cards](./model-cards.md) for how to document it.

---

## Proxy Evaluation Strategies

### Strategy 1: Expert Review

**What**: sample a set of anomaly alerts and have field technicians or domain experts label them as "real anomaly" (the device had a problem) or "false positive" (the device was fine).

**How**:
1. From the most recent scoring run, select the top 50–100 highest-scoring devices.
2. Include a stratified sample of 20–50 medium-scoring devices (anomaly score 0.5–0.7) to test the boundary.
3. Present each device's sensor trends, recent feature values, anomaly score, and `top_contributors` to a domain expert.
4. The expert classifies each case: true positive, false positive, or uncertain.
5. Compute precision-at-top-K: of the top K alerts, how many were true positives?

**Strengths**:
- Most direct proxy for supervised evaluation. If experts say the model's top alerts are real, the model is working.
- Generates labeled data that feeds future supervised models ([ADR-003, Phase 2](../05-architecture/adr-003-anomaly-detection-first.md)).
- Catches failure modes that other proxy metrics miss — a model could have stable score distributions but flag the wrong devices.

**Weaknesses**:
- Expensive. Expert time is limited and labeling is slow (each device requires reviewing sensor time series).
- Biased. Experts only review flagged devices — we never learn about anomalies the model missed (false negatives). Precision is estimable; recall is not.
- Subjective. Two experts may disagree on whether a device with mildly elevated vibration is "anomalous." Clear labeling guidelines help but don't eliminate subjectivity.
- Small sample size. 50–100 labeled examples per review cycle provide noisy precision estimates.

**Cadence**: monthly review cycles. Archive labeled examples in a Foundry dataset (`expert_labels`) with schema: `device_id`, `window_start`, `model_id`, `anomaly_score`, `expert_label` (true_positive / false_positive / uncertain), `expert_id`, `labeled_at`, `notes`.

### Strategy 2: Maintenance Correlation

**What**: correlate anomaly scores with subsequent maintenance events. If devices flagged as anomalous have more maintenance work orders within a 7-day lead-time window, the model is catching real problems.

**How**:
1. Join `model_scores` with the maintenance work order dataset on `device_id`, where work order `created_at` is within 1–7 days after `window_end`.
2. Compare the maintenance rate for flagged devices (anomaly_flag = true) vs. unflagged devices.
3. Compute the **maintenance lift**: `P(maintenance | flagged) / P(maintenance | unflagged)`. A lift >1 means flagged devices are more likely to need maintenance — the model has predictive value.
4. Compute by severity: the lift should be higher for higher anomaly scores (higher scores → more likely to need maintenance).

**Strengths**:
- Uses data that already exists (work orders, even if messy).
- Aggregate metric — doesn't require per-device labeling. A lift of 3× is meaningful even if individual work orders don't map cleanly to sensor anomalies.
- Directly measures the business-relevant question: "does flagging this device lead to useful action?"

**Weaknesses**:
- Work orders are noisy. Preventive maintenance, customer complaints, and real failures are mixed together. A device with a work order wasn't necessarily failing — it might have been on a regular maintenance schedule.
- Lag ambiguity. A 7-day window is arbitrary. Some failures develop over weeks; others are acute. The window choice affects the correlation strength.
- Selection bias. If operators *use* the model's alerts to dispatch maintenance, the correlation becomes circular — flagged devices get maintained *because* they were flagged, not because they were actually failing. This inflates the lift metric.
- Requires work order data joined to device IDs. This data may be in a separate system with inconsistent device identifiers.

**Cadence**: computed automatically as a scheduled Transform after each scoring run. Tracked over time — if maintenance lift is declining, the model is losing predictive power.

### Strategy 3: Stability Metrics

**What**: measure whether the model produces consistent results across consecutive scoring runs. A good anomaly detector should flag the same devices consistently (they're genuinely anomalous) and not flip-flop (flagging a device one hour, clearing it the next, flagging it again).

**How**:
1. **Score consistency**: for each device, compute the standard deviation of anomaly scores across the last N scoring windows (e.g., 24 windows for 1 day of hourly scoring). A device with scores [0.8, 0.2, 0.9, 0.1, 0.8] has high variance — either the device's behavior is genuinely oscillating, or the model is unstable.
2. **Flag stability**: compute the fraction of devices whose anomaly_flag changes between consecutive windows. If >10% of devices flip between flagged and unflagged every hour, the model or threshold is too sensitive to noise.
3. **Rank stability**: compute rank correlation (Spearman) between the device rankings by anomaly score in consecutive windows. High rank correlation means the model consistently identifies the same devices as most/least anomalous.

**Strengths**:
- Entirely automated — no expert involvement, no external data.
- Catches a specific failure mode: noisy or unstable models that erode operator trust. If every alert clears itself in the next window, operators learn to ignore alerts.
- Complementary to other strategies — a model can have good maintenance lift but poor stability, meaning it catches the right devices but the timing is inconsistent.

**Weaknesses**:
- A model that scores every device at 0.0 has perfect stability but is useless. Stability is a necessary but not sufficient property.
- Some instability is legitimate — a device that's borderline anomalous *should* oscillate near the threshold. Only high instability in the score (not just the flag) indicates a problem.
- Doesn't measure accuracy at all. A consistently wrong model looks great on stability metrics.

**Cadence**: computed on every scoring run. Alert if flag flip rate exceeds 10% or rank correlation drops below 0.9.

### Strategy 4: Internal Model Metrics

**What**: use metrics derived from the model's own behavior — not from external data — to assess quality.

**Specific metrics by model type**:

**Isolation Forest**:
- **Score distribution shape**: the distribution of anomaly scores should be right-skewed — most devices score low (normal), a tail scores high (anomalous). If the distribution is uniform or bimodal, the model isn't learning meaningful structure.
- **Average path length distribution**: isolatable anomalies should have significantly shorter path lengths than the bulk of normal data.
- **Feature importance stability**: the top contributing features should be consistent across scoring runs (unless the fleet's dominant failure mode changes). Volatile feature importance suggests the model is fitting noise.

**Autoencoder**:
- **Reconstruction error distribution**: plot the MSE distribution over the validation set. Should have a clear body (low error, normal devices) and a tail (high error, anomalous devices). If the tail is absent, the model is too powerful (reconstructing even anomalies well). If there's no body, the model is underfitting.
- **Training loss convergence**: training loss should decrease steadily and plateau. Validation loss should follow a similar curve — if validation loss diverges from training loss, the model is overfitting to training data contamination.
- **Per-feature reconstruction error**: some features should be consistently well-reconstructed (stable sensors like ambient temperature) and others should have higher error (volatile sensors like vibration). If all features have similar reconstruction error, the model isn't learning sensor-specific patterns.

**DBSCAN** (for cohort discovery):
- **Silhouette score**: measures cohort separation. Higher is better. Track across retraining runs — declining silhouette score means cohorts are becoming less distinct.
- **Noise fraction**: fraction of devices classified as noise (no cluster). Should be low (<5%) — if many devices don't fit any cluster, the `eps` parameter needs adjustment.

**Cadence**: computed during training (as part of the training Transform) and written to a metrics dataset. See [Experiment Tracking](./experiment-tracking.md).

---

## Establishing Baselines

### The "No Model" Baseline

Before evaluating ML models, define what "no ML" looks like. This is the system's current state — typically rule-based threshold alerts on raw sensor values:

- Alert if `temperature_evaporator > -10°C` (warmer than normal for a freezer).
- Alert if `vibration_compressor > 10 mm/s` (above acceptable vibration).
- Alert if `current_compressor > 30 A` (above rated current).
- Alert if `pressure_ratio < 2.0` or `pressure_ratio > 5.0` (outside normal compression ratio).

These threshold rules are easy to implement, fully interpretable, and have zero training cost. They form the lower bound. Any ML model that doesn't outperform threshold rules on maintenance correlation or expert review is not earning its complexity.

Implement the threshold-based baseline as a Foundry Transform that produces `model_scores` in the same Contract 5 format (with `model_id = "threshold_baseline"`). This allows direct comparison in the same evaluation pipeline.

### Rule-Based Alert Rate

Measure the rule-based system's alert rate, maintenance lift, and expert-reviewed precision. These numbers are the bar that ML models must clear. Document them in each model card (see [Model Cards](./model-cards.md), "Baseline" section).

---

## Threshold Selection

### The Problem

Contract 5 defines `anomaly_flag` as a boolean derived from `anomaly_score > threshold`. But what threshold? With no labels, there's no ROC curve to optimize. The threshold is a business decision wrapped in a statistical problem.

### Approach 1: Contamination Rate Assumption

Assume a target anomaly rate based on domain knowledge. If the operations team believes ~3–5% of devices typically have some anomaly, set the threshold to flag the top 5% of scores.

For Isolation Forest, this is the `contamination` parameter. For Autoencoder and statistical models, compute the Nth percentile of the score distribution and use it as the threshold.

**Risk**: the assumed contamination rate may be wrong. If the true anomaly rate is 1%, a 5% threshold generates 4× more false positives than true positives. If the true rate is 10%, the threshold misses half of anomalies.

**Mitigation**: start conservative (low contamination, e.g., 3%). This produces fewer but higher-confidence alerts. Increase the rate only as expert review confirms the model's top alerts are predominantly true positives.

### Approach 2: Percentile-Based Thresholds

Set the threshold as a fixed percentile of the training set's score distribution (e.g., 95th percentile = flag the top 5%). This is technically equivalent to contamination rate assumption but frames the threshold relative to the model's learned score distribution rather than a domain assumption.

**Advantage**: percentile thresholds are stable across model versions — even if the raw score scale changes, the 95th percentile always flags 5%.

**Disadvantage**: if the model version change genuinely shifts the anomaly ranking (some devices that were scored low are now scored high), a percentile threshold masks this shift.

### Approach 3: Adaptive Thresholds from Expert Feedback

After accumulating expert labels (Strategy 1), optimize the threshold:

1. For each threshold value, compute estimated precision using expert labels.
2. Select the threshold that achieves a target precision (e.g., 80% precision — 4 out of 5 HIGH alerts are real).
3. Update the threshold in the scoring Transform configuration.

This requires sufficient expert labels (≥100 labeled examples spanning the score range) and must be re-computed after each model retrain.

### Threshold by Severity

Different severity levels ([Contract 6](../05-architecture/data-contracts.md)) use different thresholds:

| Severity | Threshold Range | Alert Action |
|----------|----------------|--------------|
| LOW | 0.6 – 0.7 | Dashboard only |
| MEDIUM | 0.7 – 0.85 | Dashboard + notification |
| HIGH | 0.85 – 0.95 | Notification + work order |
| CRITICAL | > 0.95 | Immediate escalation |

These thresholds are stored in the scoring Transform configuration (not hardcoded) and logged in `threshold_used` per Contract 5 for auditability.

---

## Lead-Time Analysis

### What We're Measuring

When the model flags a device as anomalous, how far in advance of actual maintenance or failure does the flag appear? This is the key operational metric — the goal is 12–24 hours of lead time (see [ADR-003](../05-architecture/adr-003-anomaly-detection-first.md)).

### How to Compute

1. For each device that received corrective maintenance (from work order data), find the most recent maintenance event.
2. Look backward in `model_scores`: when did the device *first* exceed the anomaly threshold before this maintenance event?
3. Lead time = `maintenance_event_time - first_anomaly_flag_time`.
4. Compute statistics: median lead time, 25th/75th percentile, fraction of maintenance events with >12 hours lead time.

### Interpretation Caveats

- **Not all maintenance events are preceded by anomalies.** Some maintenance is preventive (no anomaly to detect). Some failures are acute (no gradual buildup for the model to catch). The fraction of maintenance events preceded by anomalies is a recall proxy — but it's noisy.
- **Some anomaly flags have no subsequent maintenance.** These are either false positives or early warnings that didn't result in maintenance yet. Without follow-up information, we can't distinguish.
- **Lead time depends on the failure mode.** Compressor bearing wear (vibration increases over weeks → long lead time) vs. refrigerant leak (sudden pressure drop → short lead time). Aggregate lead time masks this variation — compute lead time by `top_contributors[0]` (dominant failure sensor) for a more actionable breakdown.

---

## A/B Testing Framework

### Comparing Model Variants

When a new model version is trained (different hyperparameters, new features, or a different model type), evaluate it against the current production model before promoting:

**Offline A/B test**: score the same historical data with both models. Compare:
- Score rank correlation (how much do the models agree on which devices are most/least anomalous?)
- Alert overlap (what fraction of flagged devices are flagged by both models?)
- Divergence cases (devices flagged by one model but not the other — review with experts)
- Maintenance lift (computed on the same time period for both models)

**Online A/B test** (if operational maturity allows): route 50% of scoring runs to the new model, 50% to the existing model. Compare alert rates, expert feedback, and maintenance correlation over 2–4 weeks. This requires the alert pipeline to track which model produced each alert.

**Implementation**: the scoring Transform produces `model_scores` with `model_id` and `model_version` columns (Contract 5). Both models write to the same dataset. Downstream analysis Transforms compare metrics between model IDs.

### Graduation Criteria

A new model version replaces the production model when:

1. **Maintenance lift is equal or better** (not significantly worse — p-value > 0.05 on a two-proportion z-test).
2. **Expert-reviewed precision is equal or better** (if expert labels are available for both models).
3. **Alert volume is within operational tolerance** (±30% of current alert volume — a 5× increase in alerts is unacceptable even if precision improves).
4. **Stability metrics are equal or better** (flag flip rate, rank correlation).
5. **No regression on internal metrics** (score distribution shape, reconstruction error distribution).

If any criterion is ambiguous, require manual approval from the ML Scientist and operations lead before promotion.

---

## Model Agreement

### When Multiple Models Disagree

Running multiple model types (statistical, Isolation Forest, Autoencoder) means a device can be flagged by some models and not others. How to interpret disagreement:

**All models agree (flagged or not)**: high confidence. If all flag the device, it's almost certainly anomalous. If none flag it, it's almost certainly normal.

**One model flags, others don't**: low confidence. The flagging model may be detecting a pattern the others miss — or it may be producing a false positive. The alert severity should be LOW regardless of the anomaly score. Log these cases for expert review — they are the most informative for understanding model differences.

**Majority flags**: moderate confidence. Use severity rules as defined in [Contract 6](../05-architecture/data-contracts.md). At least two models must agree for MEDIUM or higher severity.

### Consensus Scoring

An optional ensemble approach: compute a weighted average of anomaly scores across models. Weights are initially equal and adjusted based on each model's maintenance lift over time (models with higher lift get more weight).

`ensemble_score = Σ(weight_i × score_i) / Σ(weight_i)`

This is written as a separate `model_id = "ensemble"` entry in `model_scores` and treated as another model for evaluation purposes. The ensemble should outperform any individual model on maintenance lift — if it doesn't, the weights are miscalibrated or one model is strictly dominant.

---

## What Good Looks Like

In the absence of traditional accuracy metrics, here are the targets for each proxy metric:

| Metric | Target | Concerning Level |
|--------|--------|-----------------|
| Expert-reviewed precision (top 50 alerts) | ≥ 70% true positive | < 50% |
| Maintenance lift (flagged vs. unflagged) | ≥ 2.0× | < 1.5× |
| Flag flip rate (consecutive windows) | ≤ 5% | > 15% |
| Rank correlation (consecutive windows) | ≥ 0.95 | < 0.85 |
| Median lead time (before maintenance) | ≥ 18 hours | < 6 hours |
| Score distribution skewness | > 2.0 (right-skewed) | < 1.0 |

These targets are initial estimates and will be refined as evaluation data accumulates. Document the achieved values in [Model Cards](./model-cards.md).

---

## Cross-References

- [Data Contracts](../05-architecture/data-contracts.md) — Contract 5 (model output), Contract 6 (alert rules and severity thresholds)
- [ADR-003](../05-architecture/adr-003-anomaly-detection-first.md) — rationale for unsupervised approach and Phase 2 supervised models
- [Model Selection](./model-selection.md) — which models are being evaluated
- [Training Pipeline](./training-pipeline.md) — temporal splits and device holdouts used for validation
- [Experiment Tracking](./experiment-tracking.md) — logging evaluation metrics per experiment
- [Model Cards](./model-cards.md) — documenting evaluation results per production model
