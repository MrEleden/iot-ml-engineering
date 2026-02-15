# Training Pipeline

## Why Training Is Unusual Here

In a supervised setting, training is straightforward: you have labeled examples, you split them, you fit a model, you measure accuracy. In unsupervised anomaly detection, training means teaching a model what "normal" looks like — using data that almost certainly contains anomalies we can't identify. Every decision in this pipeline addresses that contamination problem.

This document covers how training data is constructed, how splits are designed to prevent leakage, how contaminated training data is handled, and how models are trained within Foundry's Transform framework.

---

## Training Data Construction

### Source

All training data comes from the `device_features` dataset ([Contract 3](../05-architecture/data-contracts.md)), which contains time-windowed aggregations of the 14 raw sensor parameters — ~45 feature columns per device per window. See [Feature Engineering](../02-feature-engineering/) for how these features are computed.

### Selecting "Presumably Normal" Data

We have no labeled anomalies, so training data is "normal" by assumption — specifically, we assume that the majority of readings in a given time period are normal. This assumption is reasonable for refrigeration systems: catastrophic failure rates are low (typically <5% of fleet at any given time), and most devices operate within normal parameters most of the time.

**Inclusion criteria**:

- Recent N months of feature data (initially 3 months; extend to 6–12 months for seasonal coverage).
- Only windows with `sensor_completeness >= 0.7` (from Contract 3). Incomplete data creates noisy features that the model may learn as "normal."

> **Note on scoring threshold gap**: The scoring pipeline uses a lower threshold (0.5) to maximize device coverage. Scores on windows with 0.5–0.7 completeness should be treated with lower confidence — the model has limited training data at these quality levels. The `sensor_completeness` field is carried through to Contract 5 output so downstream consumers can apply their own confidence weighting.
- Only devices with `reading_count >= 10` per window (15-minute window should have ~15 readings). Sparse data windows are excluded.

**Exclusion criteria**:

- **Known outage periods**: if the operations team has dates of planned maintenance, firmware updates, or known incidents, exclude those windows. This is the most reliable decontamination step — but it requires operational records, which may be incomplete.
- **Extreme quality flags**: windows where `quality_flag_count > 5` (from Contract 3) are excluded. High quality flag counts suggest sensor issues, not device issues.
- **Recently repaired devices**: exclude 48 hours of data after a known maintenance event. Post-repair behavior may differ from normal operation and shouldn't define "normal."

**What we cannot exclude**:

- Unknown anomalies in the training set. A device slowly degrading over 3 months will have its degradation pattern included in "normal." This is the contamination problem — there is no way to fully solve it without labels.

### Volume Estimation

- 100K devices × 3 months × 96 windows/day (15-min) ≈ 860M training rows for fleet-wide models
- Subsampling is necessary. For Isolation Forest: random subsample of 100K–500K rows (the algorithm internally subsamples anyway). For Autoencoder: stratified subsample of ~1M rows covering all device types proportionally.
- Cohort models: subset by cohort first, then apply the same volume limits per cohort.
- Per-device models: all available data for that device (30–90 days × 96 windows/day ≈ 3K–9K rows per device).

---

## Temporal Train/Validation Split

### No Random Splits

Random splitting is not valid for time series data. If training data is from January–March and a random 20% is held out for validation, the validation set contains readings from the *middle* of the training period. This creates temporal leakage: the model has seen data from before *and* after each validation point, making validation performance artificially good.

**All splits are temporal**: the validation set is always *after* the training set in time.

### Split Design

```
Training Window (older)              Validation Window (newer)
|========================|    gap    |===============|
Month 1       Month 2      (7 days)   Month 3
```

- **Training set**: oldest N-1 months of the selected data period.
- **Gap**: 7-day buffer between training and validation. This prevents information leakage from sliding-window features (e.g., a 24-hour rolling mean computed on data that spans the split boundary).
- **Validation set**: most recent 1 month of data.

For a 3-month data period: 2 months training, 7-day gap, ~3 weeks validation.

### Why 7-Day Gap

The largest window duration in [Contract 3](../05-architecture/data-contracts.md) is 1440 minutes (1 day). Features like rolling 24-hour means and slopes use data from the preceding 24 hours. A 7-day gap ensures no feature in the validation set was computed using any raw reading from the training period. One day would be sufficient for current features, but 7 days provides safety margin for future features with longer lookback windows.

### Multiple Temporal Folds (Expanding Window)

For more robust validation, use expanding-window cross-validation:

```
Fold 1:  |=== Train ===|  gap  |= Val =|
Fold 2:  |====== Train ======|  gap  |= Val =|
Fold 3:  |========= Train =========|  gap  |= Val =|
```

Each successive fold uses a longer training period and validates on the next time segment. This tests whether more training data improves the model and whether the model degrades on more recent data (a sign of concept drift).

Use 3–5 folds. More folds become expensive (each fold requires a full model training run) and the later folds have very long training periods with short validation periods.

---

## Device Holdout

### Why Device Holdout Matters

Even with temporal splits, if the same device appears in both training and validation, the model has seen that device's "normal" patterns during training. When it scores that device in validation, it performs well — not because it generalized, but because it memorized. To test generalization, hold out devices entirely.

### Device Holdout Design

- **80% of devices**: in the training set (features from these devices during the training window).
- **10% of devices**: in the validation set (features from these devices during the validation window). These devices are *not seen during training*.
- **10% of devices**: reserved for future test evaluation (not used during any hyperparameter tuning).

Device assignment is stratified by cohort attributes (device model, climate zone) to ensure each split is representative of the fleet.

### When Device Holdout Is Not Feasible

For per-device models, device holdout is meaningless — each model is trained on one device's data. In this case, use temporal holdout only: train on the first 70% of the device's history, validate on the last 30% (with a gap).

---

## Handling Contaminated Training Data

### The Contamination Problem

Our training data contains anomalies that we can't identify. A device that's slowly losing refrigerant charge has readings that drift from normal over weeks — this drift is in the training set, teaching the model that "slowly losing refrigerant" is normal.

There is no perfect solution. But there are mitigation strategies, from simple to complex:

### Strategy 1: Robust Statistics

For statistical models (z-score, Mahalanobis), use robust estimators instead of standard mean/variance:

- **Median** instead of mean (resistant to outliers).
- **Median absolute deviation (MAD)** instead of standard deviation.
- **Minimum covariance determinant (MCD)** for Mahalanobis distance — estimates the covariance matrix using the most "central" subset of data, ignoring outliers.

These don't remove anomalies from training data — they reduce their influence on the model parameters.

### Strategy 2: Training-Set Self-Scoring (Iterative Cleaning)

1. Train an initial model on the full training set (contaminated).
2. Score the training set with the model.
3. Remove the top N% highest-scoring points (likely anomalies or near-anomalies).
4. Retrain on the cleaned set.
5. Repeat 1–2 times (more iterations risk removing legitimate tail-of-distribution normal points).

This is a form of iterative outlier rejection. It works well for gross contamination (devices that were clearly failing during the training period) but is risky for borderline cases — aggressive cleaning narrows the model's definition of "normal" and increases false positives.

**Safeguard**: never remove more than 10% of training data through iterative cleaning. If more than 10% of the training set is "anomalous," the data period is wrong — choose a cleaner time range.

### Strategy 3: Isolation Forest's Built-In Robustness

Isolation Forest is naturally resistant to contamination because it subsamples the data during tree construction. Anomalous points in a random subsample of 256 points are rare enough that they don't substantially affect tree structure. This is a key advantage of Isolation Forest over statistical methods for this use case.

### Strategy 4: Autoencoder Capacity Control

For Autoencoders, limit model capacity (undercomplete architecture) so the model *cannot* memorize rare anomalies. A bottleneck layer with dimension << input dimension forces the model to learn only the dominant (normal) patterns. Rare anomalous patterns are sacrificed because the latent space is too small to represent them. The tradeoff: too-small bottleneck → underfitting on normal patterns → high reconstruction error on normal data → low sensitivity.

---

## Retraining Cadence

### When to Retrain

Models must be retrained periodically because "normal" changes over time:

- **Seasonal changes**: ambient temperature shifts affect condenser and evaporator temperatures. A model trained on winter data will flag normal summer behavior as anomalous. Retraining on recent data recalibrates.
- **Equipment aging**: compressor vibration baselines increase slowly over years. A model trained 6 months ago expects lower vibration than a model trained today.
- **Fleet composition changes**: new device deployments, device retirements, or firmware updates change the fleet distribution.
- **Concept drift**: the relationship between sensors may change (e.g., a new refrigerant type with different pressure-temperature characteristics).

### Recommended Cadence

| Model Type | Retraining Frequency | Rationale |
|-----------|---------------------|-----------|
| Statistical baseline | Weekly | Cheap to retrain. Frequent updates capture recent shifts. |
| Isolation Forest | Weekly to biweekly | Fast training (~seconds). Weekly captures seasonal drift. Biweekly is acceptable if drift monitoring shows stability. |
| Autoencoder | Monthly | Expensive to train (minutes-hours). Reconstruction error distributions are relatively stable. |
| Per-device models | Monthly per device | Staggered — retrain 1/30th of devices per day to spread compute load. |

### Trigger-Based Retraining

In addition to scheduled retraining, retrain immediately when:

- **False positive rate exceeds threshold**: if field feedback (from [Ontology action feedback](../04-palantir/ontology-design.md)) shows >30% of HIGH/CRITICAL alerts are false positives, the model's threshold or learned behavior is miscalibrated.
- **Fleet composition changes significantly**: >5% of fleet consists of new devices deployed in the past 14 days.
- **Anomaly rate deviates from expected**: if the model flags >2× or <0.5× the expected contamination rate, something shifted.
- **Feature distribution shift**: if any input feature's mean or variance changes by >3σ from the training period's value, the model is scoring out-of-distribution.

### Training Window for Retraining

Use a **rolling window**, not an expanding window. Train on the most recent N months, not all historical data. This prevents ancient data (which may reflect decommissioned device types or outdated operating conditions) from diluting current patterns.

Recommended: 3-month rolling window for Isolation Forest, 6-month for Autoencoder (needs more data and seasonal coverage).

---

## Foundry Integration

### Training as a Transform

All model training runs as Foundry Transforms, not ad-hoc notebooks. This ensures reproducibility and enables scheduled retraining. See [Model Integration](../04-palantir/model-integration.md) for the Transform pattern.

```
Input:  device_features (Contract 3 dataset)
    ↓
Transform: train_isolation_forest
    ↓
Output: ModelOutput → published model artifact in Foundry model store
```

The training Transform:
1. Reads the feature dataset (scoped to the training window).
2. Applies exclusion criteria (completeness, quality flags, known outages).
3. Performs the temporal split internally (or reads pre-split datasets).
4. Trains the model.
5. Computes validation metrics.
6. Publishes the model artifact via `ModelOutput` and the model adapter.

### Model Adapter

The model adapter bridges Foundry's model framework and the underlying scikit-learn/PyTorch model. It defines input/output schemas and the `predict()` method. See [Model Integration](../04-palantir/model-integration.md) for the adapter pattern.

Key decisions for unsupervised models:
- The adapter normalizes raw model output (e.g., Isolation Forest `decision_function` scores) to the [0, 1] range required by Contract 5.
- The adapter computes `top_contributors` and `contributor_scores` from feature importance / reconstruction error.
- The adapter applies the `threshold_used` to set `anomaly_flag`.

### Hyperparameter Selection Without Labels

Without labels to compute accuracy or F1, hyperparameter tuning uses internal metrics — metrics derived from the model's own behavior on training and validation data:

**Isolation Forest**:
- `contamination`: start at 0.05, adjust based on expert feedback on alert volume. Not tuned automatically — it's a business decision ("how many alerts can the operations team handle daily?").
- `n_estimators`: use 200 (standard). More trees marginally improve stability but don't change scores meaningfully above 100.
- `max_samples`: use 256 (default). Larger subsample sizes make training slower without improving anomaly detection.
- Tuning signal: score distribution shape on validation set. The distribution should be right-skewed (most devices score low, a tail scores high). A uniform distribution suggests the model isn't learning structure.

**Autoencoder**:
- `latent_dim`: start at 10 (for ~50 input features). Sweep [5, 10, 15, 20] and select based on validation reconstruction error — the latent dimension where validation error stops decreasing meaningfully (elbow method).
- `learning_rate`: sweep [1e-4, 1e-3, 1e-2]. Select by validation loss convergence speed and stability.
- `epochs`: use early stopping on validation reconstruction loss (patience = 5 epochs).
- Tuning signal: validation reconstruction error distribution. Should cleanly separate most devices (low error) from a tail (high error). If the distribution is approximately uniform, the model is underfitting or the latent dim is too small.

**DBSCAN** (for cohort discovery):
- `eps`: sweep a range and compute silhouette score for each. Select the eps that maximizes silhouette score (tightest clusters).
- `min_samples`: set to 20–50 for fleet-level clustering (small clusters are unstable and not operationally useful as cohorts).

### Scheduled Training Pipelines

Training Transforms are scheduled via Foundry's build scheduling:

- **Weekly Isolation Forest retraining**: scheduled build every Sunday at 02:00 UTC. Reads the latest 3-month window of `device_features`, trains, validates, publishes.
- **Monthly Autoencoder retraining**: scheduled build on the 1st of each month at 01:00 UTC. Reads the latest 6-month window.
- **Scoring Transforms reference published models**: the scoring Transform's `ModelInput` automatically uses the latest published model version. No manual version bumping needed.

### Reproducibility

Every training run must be reproducible:

- **Random seeds**: set `random_state=42` for all randomized algorithms (Isolation Forest, train/test split, Autoencoder weight initialization). Document the seed in [experiment tracking](./experiment-tracking.md).
- **Data snapshot**: the training Transform references specific input dataset transactions via Foundry. Re-running the Transform on the same transaction produces the same model.
- **Dependency pinning**: Python package versions are pinned in `meta.yml` in the Code Repository. Use exact versions (`scikit-learn==1.3.0`), not ranges.
- **Environment**: Foundry Code Repositories pin the Python and Spark runtime versions. Document the environment version in experiment tracking.

---

## Failure Modes

### Training Data Is Too Clean

If aggressive exclusion criteria remove too many windows, the model sees an artificially narrow view of "normal." This increases false positives on legitimate operational variation (e.g., high door-open fraction during restocking events).

**Mitigation**: monitor the fraction of data excluded. If >20% of windows are excluded, loosen criteria or investigate whether the exclusion rules are too aggressive.

### Training Data Is Too Contaminated

If a significant fraction of devices were degrading during the training period, "normal" includes degradation patterns. The model won't detect these patterns as anomalous.

**Mitigation**: iterative cleaning (Strategy 2 above). Also, compare models trained on different time periods — if they disagree on which devices are anomalous, the training data quality differs across periods.

### Temporal Split Leakage via Features

If a feature uses a lookback window that crosses the temporal split (e.g., a 7-day rolling mean computed on data from both training and validation periods), the split is invalid.

**Mitigation**: the 7-day gap between training and validation sets prevents this for all current features (max lookback is 24 hours). If new features with longer lookback windows are added, increase the gap accordingly.

### Model Publishes with Poor Validation Metrics

A model that passes Foundry CI checks but has degenerate validation metrics (e.g., all devices score the same, or no devices exceed the threshold) will produce useless scores in production.

**Mitigation**: add dataset expectations to the validation metrics dataset. Fail the build if anomaly rate on validation set is <0.5% or >20% (unexpected extremes). See [Experiment Tracking](./experiment-tracking.md) for validation criteria.

---

## Cross-References

- [Data Contracts](../05-architecture/data-contracts.md) — Contract 3 (feature store schema), Contract 4 (model input), Contract 5 (model output)
- [Feature Engineering](../02-feature-engineering/) — how the ~45 features are computed
- [Model Integration](../04-palantir/model-integration.md) — Foundry Transform patterns for training and publishing
- [Model Selection](./model-selection.md) — which models are trained and why
- [Evaluation Framework](./evaluation-framework.md) — how validation metrics are computed without labels
- [Experiment Tracking](./experiment-tracking.md) — logging training runs and comparing experiments
