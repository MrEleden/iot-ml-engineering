# Model Selection

## Why Multiple Approaches

There is no single best anomaly detection model for refrigeration IoT data. Each approach makes different assumptions about what "normal" looks like, and each has blind spots:

- A statistical method catches obvious single-sensor outliers but misses subtle multi-sensor degradation.
- An Isolation Forest handles high-dimensional interactions but can't model temporal sequences.
- An Autoencoder learns complex nonlinear patterns but is harder to interpret and more expensive to train.
- A clustering method finds natural device groupings but struggles with continuous anomaly scoring.

Running multiple approaches and comparing their outputs is not hedging — it's the only honest strategy when we lack labeled data to definitively prove one approach is better. When two models agree that a device is anomalous, confidence goes up. When they disagree, we learn something about the failure mode.

This document compares four approaches, evaluates three granularity levels, and recommends a phased rollout.

---

## Approach 1: Statistical Baseline (Z-Score / Mahalanobis Distance)

### How It Works

For each feature, compute the fleet-wide (or cohort-wide, or per-device) mean and standard deviation from historical "normal" data. A new observation's anomaly score is its distance from the mean in standard deviation units.

- **Univariate z-score**: compute `|x - μ| / σ` per feature, then aggregate (max, mean, or weighted sum across features). Simple but ignores correlations between sensors.
- **Mahalanobis distance**: compute distance from the mean vector using the inverse covariance matrix. Accounts for correlations — a compressor drawing high current is normal *when* ambient temperature is high, but abnormal when ambient is low.

The anomaly score is normalized to [0, 1] using a cumulative distribution function or percentile mapping to conform to [Contract 5](../05-architecture/data-contracts.md).

### Strengths

- **Fastest to train and score.** Computing mean/std/covariance on 100K devices is seconds in Spark. Scoring is a vector subtraction.
- **Fully interpretable.** The z-score per feature directly tells operators which sensor is out of range and by how much. Maps cleanly to `top_contributors` in Contract 5.
- **Easy to debug.** When the model produces a false positive, you can inspect the z-scores per feature and explain exactly why.
- **Serves as the irreducible baseline.** If a more complex model can't beat the statistical baseline, the complexity isn't justified.

### Weaknesses

- **Assumes Gaussian distributions.** Sensor data often isn't Gaussian — bimodal distributions (e.g., compressor current during on/off cycling), heavy tails (vibration spikes), bounded ranges. Z-scores on non-Gaussian data produce miscalibrated anomaly scores.
- **Misses nonlinear interactions.** A z-score on `current_compressor_mean` and a z-score on `temp_spread_evap_cond` are computed independently. The model can't learn that a specific combination of "slightly elevated current + slightly reduced temperature spread" is more anomalous than either alone.
- **Sensitive to contaminated training data.** If the training set contains anomalous readings (and it will — we're training on data we assume is normal), the mean and variance are skewed, making the model less sensitive.
- **Covariance matrix instability.** Mahalanobis distance requires inverting the covariance matrix. With ~45 features and potential multicollinearity, the matrix may be ill-conditioned. Regularization (shrinkage estimator) is needed.

### When to Use

As the first model deployed on any new fleet or cohort. The statistical baseline should always run alongside more complex models so there's a reference point for "how much does the complex model add?"

---

## Approach 2: Isolation Forest

### How It Works

Isolation Forest builds an ensemble of random trees that partition the feature space. At each node, a feature is randomly selected and a split point is randomly chosen between the feature's min and max values. Anomalies are points that are isolated (reach a leaf node) in fewer splits — they're in sparse regions of the feature space.

The anomaly score is derived from the average path length across all trees. Shorter average path = more anomalous. The raw `decision_function` output is normalized to [0, 1] for [Contract 5](../05-architecture/data-contracts.md): `anomaly_score = 1 - (raw - raw_min) / (raw_max - raw_min)`.

Feature contributions are computed via permutation importance or mean decrease in path length per feature, populating `top_contributors` in Contract 5.

### Strengths

- **No distributional assumptions.** Works on non-Gaussian, multimodal, and heavy-tailed data. This is critical for refrigeration sensors where distributions vary by operating mode (defrost vs. normal, high-load vs. low-load).
- **Handles high-dimensional feature spaces.** With ~45 features from [Contract 3](../05-architecture/data-contracts.md), Isolation Forest performs well without feature selection or dimensionality reduction.
- **Fast training and scoring.** Tree construction scales as O(n·t·log(ψ)) where t = number of trees, ψ = subsample size. With default settings (200 trees, 256 subsample), training on 100K devices takes seconds.
- **Scikit-learn ships with Foundry's default Python environment.** No extra dependencies in `meta.yml` — reduces deployment friction (see [Model Integration](../04-palantir/model-integration.md)).
- **Contamination parameter provides direct threshold control.** Setting `contamination=0.05` means roughly 5% of the fleet will be flagged. This is tunable without retraining.

### Interpretability Enhancement: TreeSHAP

Isolation Forest's default feature importance is global (which features matter across all predictions). For per-prediction explanations — especially critical for MEDIUM, HIGH, and CRITICAL alerts — use **TreeSHAP** (SHAP values for tree-based models). TreeSHAP computes each feature's contribution to a specific device's anomaly score, including direction (pushing the score higher or lower). Log SHAP values alongside `top_contributors` in Contract 5 output: store as `shap_values` (JSON-serialized dict of feature → SHAP value) for post-hoc analysis and expert review. TreeSHAP is computationally efficient for tree ensembles (polynomial in tree depth, not exponential) and is available via the `shap` library (add to `meta.yml`).

### Weaknesses

- **Less interpretable than statistical methods.** Feature importance tells you which features mattered across the ensemble, but not the direction of the anomaly (is the value too high or too low?). Operators see "vibration_compressor_mean is a top contributor" but not "it's 3x higher than normal." TreeSHAP (see above) mitigates this by providing directional per-prediction explanations.
- **Sensitive to contamination ratio.** If the true anomaly rate is 2% but `contamination=0.05`, the model flags 5% of devices — inflating false positives. If the true anomaly rate is 10% (during a fleet-wide issue), the model undercounts. The contamination parameter is a guess.
- **No temporal modeling.** Each row (device × window) is scored independently. The model doesn't know that a device's anomaly score has been rising over the past 6 windows — it only sees the current window's features. Temporal trends must be captured in the feature engineering layer (e.g., `slope` features in [Time-Domain Features](../02-feature-engineering/time-domain.md)).
- **Random subsampling introduces variance.** Different random seeds produce slightly different models. Use `random_state=42` and [experiment tracking](./experiment-tracking.md) to control this. Score determinism for Contract 5 requires fixed random seeds.

### When to Use

The recommended primary model for Phase 1. Good balance of performance, speed, interpretability, and operational simplicity. Start fleet-wide, then train cohort-level models as cohort definitions stabilize.

---

## Approach 3: DBSCAN / Clustering-Based

### How It Works

DBSCAN (Density-Based Spatial Clustering of Applications with Noise) groups devices into clusters based on feature similarity. Points in dense regions form clusters; points in sparse regions are classified as noise (outliers). Outlier points — devices that don't belong to any cluster — are candidate anomalies.

The anomaly score is derived from the distance to the nearest cluster core point, normalized to [0, 1]. Devices deep inside a cluster score near 0; devices far from any cluster score near 1.

An alternative: train a separate model (e.g., Isolation Forest) per cluster, so "normal" is defined relative to each device's natural peer group rather than the full fleet. This combines the clustering insight with a proper anomaly scorer.

### Strengths

- **Discovers natural device groupings.** Devices cluster by similar operating conditions — same ambient temperature range, same usage patterns, same age. These clusters may not map to any known device attribute (model, location). Useful for defining cohorts data-driven rather than heuristically.
- **No need to pre-specify number of clusters.** Unlike k-means, DBSCAN finds the number of clusters from the data. This matters because we don't know how many distinct operating regimes exist in the fleet.
- **Outlier detection is a first-class output.** Noise points are devices that don't fit any pattern — exactly what anomaly detection should find.
- **Robust to cluster shape.** Works with non-convex clusters, unlike Gaussian mixture models.

### Weaknesses

- **Not a natural scorer.** DBSCAN outputs a binary label (cluster member or noise), not a continuous score. Converting to a 0–1 score requires post-processing (distance to nearest core point), which is less principled than Isolation Forest's path-length-based scoring.
- **Hyperparameter sensitivity.** DBSCAN requires `eps` (neighborhood radius) and `min_samples` (minimum cluster size). These are hard to set without labels. Too-small `eps` → every device is noise. Too-large `eps` → one giant cluster, no anomalies detected. Grid search with internal metrics (silhouette score) helps but isn't definitive.
- **Does not scale well to high dimensions.** With ~45 features, distance metrics become less meaningful (curse of dimensionality). Dimensionality reduction (PCA, UMAP) before clustering is often necessary, adding a preprocessing step and information loss.
- **Expensive at fleet scale.** DBSCAN's complexity is O(n²) in the worst case. With 100K devices and 50 features, this is computationally heavy. HDBSCAN (hierarchical variant) is more robust but even slower.
- **Static clusters.** Cluster assignments don't update incrementally — the entire fleet must be re-clustered. New devices don't have a cluster until the next full run.

### When to Use

Primarily as a **fleet segmentation tool** rather than a standalone anomaly scorer. Use DBSCAN to discover natural device cohorts from the data, then train per-cohort Isolation Forests or Autoencoders. This is more principled than hand-defining cohorts by device model or location.

Also useful as a secondary signal: if a device is noise in DBSCAN *and* has a high Isolation Forest score, the evidence for anomaly is stronger.

---

## Approach 4: Autoencoder (LSTM-AE or Dense AE)

### How It Works

An Autoencoder is a neural network trained to reconstruct its input. The architecture has an encoder (compresses the input feature vector to a lower-dimensional latent representation) and a decoder (reconstructs the input from the latent representation). Training minimizes reconstruction error on "normal" data.

At inference, anomalous inputs — patterns the model hasn't seen during training — produce high reconstruction error. The anomaly score is the reconstruction error (mean squared error across features), normalized to [0, 1] via a fitted distribution (percentile of the training error distribution).

Two variants:

- **Dense Autoencoder**: processes a single feature vector (one device × one window). Simpler, faster, suitable when temporal patterns are captured in the features themselves (slope, moving averages).
- **LSTM Autoencoder**: processes a sequence of feature vectors (one device × multiple consecutive windows). Captures temporal dynamics directly. More powerful but requires sequence-formatted input and more training data per device.

Feature contributions are computed as per-feature reconstruction error: features with high individual reconstruction error are listed in `top_contributors` (Contract 5). This is approximate — in a dense network, all features interact — but provides a useful signal.

### Strengths

- **Captures complex nonlinear interactions.** A refrigerant leak might manifest as a simultaneous gradual decrease in subcooling, slight increase in compressor current, and specific shift in pressure ratio. An Autoencoder can learn this multi-sensor pattern as a single "normal" manifold. Statistical methods and even Isolation Forest may miss it if each sensor is only slightly outside its individual range.
- **Reconstruction error is intuitive.** "The model couldn't reconstruct this device's readings accurately" is an easy concept for operators. High reconstruction error literally means "this doesn't look like what I've seen before."
- **LSTM variant models temporal patterns.** A gradual warming trend over 6 hours might be normal (daily ambient temperature cycle) or abnormal (failing evaporator). The LSTM sees the sequence and can distinguish these — dense models and Isolation Forest cannot.
- **Learns a compressed representation.** The latent space can be used for device similarity analysis and cohort discovery, similar to DBSCAN but continuous and potentially richer.

### Weaknesses

- **Requires more training data.** A meaningful Autoencoder needs thousands of training examples per normal pattern. For per-device models, a device needs months of history. For fleet-wide or cohort models, data is abundant (100K devices × daily features = plenty).
- **Harder to interpret.** Per-feature reconstruction error provides some interpretability, but the latent space is opaque. Operators can see *which* sensors are poorly reconstructed but not *why* in the same direct way as a z-score.
- **Higher compute cost.** PyTorch is required (add to `meta.yml` dependencies — see [Model Integration](../04-palantir/model-integration.md)). Training is GPU-beneficial but Foundry Transforms don't reliably expose GPU resources, so CPU training is slower.
- **Architecture requires tuning.** Number of layers, latent dimension, learning rate, batch size, dropout, activation function — all are choices that affect performance. Without labels, tuning is guided by reconstruction error distribution shape, which is an imperfect signal.
- **Prone to memorization.** With enough capacity, an Autoencoder can reconstruct anomalies accurately (low reconstruction error even for anomalous inputs), producing false negatives. Undercomplete architectures (bottleneck much smaller than input) mitigate this but require careful sizing.
- **LSTM variant has additional complexity.** Requires sequence-formatted input (not a single feature vector), which changes the data pipeline. Foundry model adapters expect tabular input — the adapter must handle sequence construction internally.

### When to Use

As the second model deployed after Isolation Forest, at cohort granularity. The dense Autoencoder is practical for Phase 1; the LSTM-AE is a Phase 2 exploration. Use the Autoencoder to catch failure modes that Isolation Forest misses — particularly slow multi-sensor degradation patterns.

---

## Comparison Matrix

| Criterion | Statistical Baseline | Isolation Forest | DBSCAN / Clustering | Autoencoder |
|-----------|---------------------|-----------------|---------------------|-------------|
| **Interpretability** | Excellent — z-score per feature with direction | Good — feature importance, no direction | Moderate — distance to cluster, hard to explain "why" | Limited — per-feature reconstruction error, no clear "why" |
| **Compute cost (training)** | Very low — mean/std/covariance | Low — seconds for 200 trees on 100K rows | Moderate-High — O(n²) without index | High — minutes to hours, GPU beneficial |
| **Compute cost (scoring)** | Very low — vector subtraction | Very low — tree traversal | Low — distance to nearest core | Moderate — forward pass through network |
| **Handles ~45 features** | Yes (Mahalanobis) / Partially (z-score ignores correlations) | Yes — designed for high-dimensional data | Poorly — needs dimensionality reduction | Yes — learns compressed representation |
| **Sensitivity to hyperparameters** | Low — mean/std have no hyperparams; Mahalanobis needs regularization | Low-Moderate — contamination ratio is the main knob | High — eps and min_samples are critical and hard to set | High — architecture, learning rate, latent dim all matter |
| **Drift robustness** | Low — mean/std drift with any distribution shift | Moderate — tree structure less sensitive to small shifts | Low — cluster boundaries are fixed | Moderate — latent representation is somewhat robust to small shifts |
| **Handles contaminated training data** | Poorly — outliers skew mean/std | Well — by design (subsampling isolates outliers) | Moderately — noise points are excluded from clusters | Poorly — model learns to reconstruct anomalies if they're in training data |
| **Temporal modeling** | No (unless features include slope/lag) | No | No | Yes (LSTM variant only) |
| **Foundry deployment complexity** | Minimal — implement as Transform, no model adapter needed | Low — scikit-learn included in Foundry default env | Moderate — scikit-learn included, but post-processing needed | High — PyTorch dependency, larger model artifacts |
| **False positive rate (expected)** | Higher — flags anything outside ±2-3σ | Moderate — controlled by contamination parameter | Variable — depends on eps/min_samples tuning | Lower — learns complex "normal" manifold | 

---

## Granularity Levels

The same model type can be trained at three different granularity levels, each defining "normal" differently. See [ADR-003](../05-architecture/adr-003-anomaly-detection-first.md) for architectural rationale.

### Fleet-Wide: One Model for All Devices

**What "normal" means**: the average behavior across all 100K+ devices.

**Training**: aggregate all device feature vectors into one training set. One model learned from the full fleet.

**Strengths**:
- Simple to operate — one model artifact, one scoring Transform, one set of hyperparameters.
- Maximum training data — 100K devices × N windows gives millions of training examples.
- Catches outliers relative to the fleet — a device behaving completely unlike all others.
- Works for new devices immediately (no device-specific history needed).

**Weaknesses**:
- "Normal" is a blended average across all device types, locations, and operating conditions. A commercial walk-in cooler in Miami has different normal temperatures than a display case in Minneapolis. The fleet model treats both as the same population.
- Misses device-specific anomalies where the device's behavior changed relative to *its own* baseline but is still within the fleet's range. A compressor with vibration slowly increasing from 2.0 to 4.0 mm/s is still within fleet normal (0–8 mm/s) but is anomalous for *that specific device*.
- Higher false positive rate for devices at the tails of the normal fleet distribution — a device that naturally runs warmer will be flagged more often.

**When to deploy**: Always. The fleet model is the baseline that runs on every device. It catches gross anomalies and fleet-wide issues (e.g., a firmware update causing abnormal readings across many devices simultaneously).

### Cohort: One Model per Device Group

**What "normal" means**: the average behavior of similar devices.

**Cohort definition options**:
- **Device model/manufacturer**: same hardware → similar operating characteristics
- **Climate zone/location**: ambient temperature range affects normal condenser and evaporator temperatures
- **Installation age**: new devices have different baselines than 5-year-old devices (compressor wear)
- **Data-driven clustering**: use DBSCAN or k-means on feature vectors to discover cohorts from the data — may reveal groupings that don't map to any known attribute

**Training**: group devices into cohorts, then train one model per cohort. Each cohort model sees only data from devices in that cohort.

**Strengths**:
- "Normal" is more specific — a cohort of walk-in coolers in warm climates has a different baseline than small display cases in cold climates. This reduces false positives from legitimate operating differences.
- More sensitive to within-cohort anomalies — the noise floor is lower because the normal range is tighter.
- Balanced compute cost — tens to hundreds of cohorts, not 100K individual models.
- Interpretable groupings — operators can understand "this device is abnormal compared to similar devices in similar locations."

**Weaknesses**:
- Cohort definitions require domain knowledge (or data-driven clustering, which has its own hyperparameters).
- Small cohorts have small training sets — if a cohort has only 50 devices, there may not be enough data for a robust model.
- Cohort boundaries are somewhat arbitrary — a device at the edge of two cohorts could be assigned to either.
- New cohort definitions require retraining all cohort models.
- New devices need cohort assignment rules — if a new device type doesn't match existing cohorts, it falls back to fleet-wide.

**When to deploy**: After the fleet model is stable and initial cohort definitions are validated. Cohort models should demonstrate lower false positive rates than the fleet model to justify the added complexity. Start with 5–10 broad cohorts (by device model × climate zone), expand to finer-grained cohorts based on data-driven insights.

### Per-Device: Individual Baseline per Device

**What "normal" means**: this specific device's own historical behavior.

**Training**: for each device, train a model on that device's historical feature vectors (≥30 days of data). The model learns only that device's patterns.

**Strengths**:
- Highest sensitivity — detects any change from the device's own normal, even if that change is within fleet or cohort norms.
- Catches slow drift (gradual compressor degradation over weeks) that fleet and cohort models miss because the change is small relative to the population variance.
- Personalized thresholds — a device that naturally runs at 2.5 mm/s vibration flags at 3.5 mm/s, while a fleet model might not flag until 8 mm/s.
- No cohort definition needed — each device is its own cohort.

**Weaknesses**:
- **Scale**: 100K devices × one model each = 100K model artifacts. Training, storing, versioning, and scoring becomes a major infrastructure challenge. Foundry's model framework is not designed for 100K model artifacts.
- **Cold start**: new devices have no history. Per-device models can't score until ≥30 days of data accumulate. During this period, only fleet and cohort models cover the device.
- **Overfitting**: with ~30 days of data from one device, the model may memorize rather than generalize. Seasonal patterns not in the 30-day window are flagged as anomalies when they first appear (e.g., summer temperature increase if trained on spring data).
- **Model complexity vs. data size**: per-device training yields 3K–9K rows (30–90 days × 96 windows/day). This is sufficient for Isolation Forest (which subsamples to 256 rows per tree regardless) but marginal for a per-device Autoencoder. A dense AE with a 45→10 bottleneck has ~2K parameters — fitting 2K parameters on 3K rows risks overfitting, especially with contaminated data. For per-device AE, require ≥90 days of history (≥8.6K rows) and use aggressive regularization (dropout=0.2, weight decay=1e-4). Consider using the cohort AE as a pretrained backbone and fine-tuning only the final encoder/decoder layers on device-specific data (transfer learning) to reduce the effective parameter count.
- **Operational overhead**: 100K model retraining jobs, 100K scoring jobs, 100K sets of metrics to monitor. Debugging a single device's model behavior requires finding and inspecting one of 100K artifacts.
- **Stale models**: devices that change operating mode (moved to a different location, connected to different equipment) need model retraining. Detecting that a change is "new normal" vs. "anomaly" is the exact problem we're trying to solve.

**When to deploy**: Phase 2, and only for high-value or high-risk devices. Choose a targeted subset (e.g., devices guarding critical inventory, devices with >$10K replacement cost) rather than the full fleet. For the remaining devices, per-device insights come from cohort models augmented with device-specific features (e.g., deviation from that device's own 7-day rolling mean — computed in the feature layer, not the model layer).

### Granularity Comparison

| Criterion | Fleet-Wide | Cohort | Per-Device |
|-----------|-----------|--------|------------|
| **Number of models** | 1 | 10–100 | 100K+ |
| **Training data per model** | 100K+ device-windows | 100–10K device-windows | 1 device × 30+ days |
| **Sensitivity** | Low — detects gross outliers | Medium — detects cohort-relative anomalies | High — detects device-specific drift |
| **False positive rate** | Higher — "normal" is too broad | Lower — "normal" is specific to peer group | Lowest — "normal" is device-specific |
| **New device coverage** | Immediate | Immediate (if cohort is assigned) | After 30+ days |
| **Compute cost** | Low | Moderate | Very high |
| **Operational complexity** | Low | Moderate | Very high |
| **Seasonal robustness** | Good — fleet-wide seasonal patterns are captured | Good — cohort seasonal patterns are captured | Poor — unless trained on ≥1 year of data |
| **Infrastructure fit (Foundry)** | Excellent — one model artifact | Good — tens of model artifacts | Poor — not designed for 100K model artifacts |

---

## Recommended Phased Approach

### Phase 1a: Isolation Forest Fleet-Wide (Weeks 1–4)

Deploy a single Isolation Forest trained on the full fleet's `device_features` (Contract 3). This is the fastest path to a running anomaly detection system.

- **Why Isolation Forest**: best balance of performance, speed, interpretability, and Foundry compatibility (scikit-learn is pre-installed). No distribution assumptions.
- **Why fleet-wide**: one model, one scoring Transform, minimal operational overhead. Catches gross anomalies immediately.
- **Alongside**: run the statistical baseline (z-score) as a comparison model. If Isolation Forest doesn't meaningfully outperform z-scores, something is wrong.
- **Output**: `model_scores` dataset conforming to Contract 5. Anomaly score [0, 1], binary flag, top 5 contributing features.
- **Threshold setting**: start with `contamination=0.05` (5% flagged). Adjust based on alert volume and initial field feedback.

### Phase 1b: Add Cohort Models (Weeks 5–12)

Define initial cohorts (device model × climate zone) and train per-cohort Isolation Forests.

- **Why now**: fleet model is running and producing initial field feedback. False positive patterns reveal which device groups need their own baselines.
- **Cohort discovery**: run DBSCAN on fleet feature vectors to validate or refine the device model × climate zone groupings.
- **Comparison**: evaluate cohort models vs. fleet model using the proxy metrics from [Evaluation Framework](./evaluation-framework.md). Cohort models should have lower false positive rates for the same recall (estimated via maintenance correlation).
- **Output**: multiple `model_scores` rows per device per window — one from the fleet model, one from the cohort model. The [alert pipeline](../05-architecture/data-contracts.md) uses the maximum score or a consensus rule.

### Phase 1c: Add Autoencoder Cohort Models (Weeks 12–20)

Train dense Autoencoders per cohort to catch failure modes that Isolation Forest misses.

- **Why Autoencoder**: captures nonlinear multi-sensor interactions. Expected to detect slow degradation patterns (compressor efficiency decline) that tree-based methods miss.
- **Why cohort only**: Autoencoders need more data per model and are more expensive to train. Fleet-wide has enough data but lacks specificity. Per-device doesn't have enough data for most devices.
- **PyTorch dependency**: add to `meta.yml` and test dependency resolution before merging — see [Model Integration, Limitation 4](../04-palantir/model-integration.md).
- **Comparison**: run Autoencoder alongside Isolation Forest. Log cases where they disagree — these are the most informative for understanding model blind spots.

### Phase 2: Per-Device for High-Value Assets (Months 6+)

For devices guarding critical inventory (>$10K loss on failure) or with known failure history, train per-device Isolation Forests.

- **Scope**: top 1–5% of fleet by business value, not all 100K devices.
- **Prerequisite**: device must have ≥90 days of history (not 30 — longer history for better seasonal coverage).
- **Infrastructure**: batch scoring partitioned by device, model artifacts organized by device group in Foundry.

### Phase 2+: Supervised Models from Pseudo-Labels (Months 6–12)

Accumulate confirmed/rejected anomaly alerts from field engineers. When sufficient labeled examples exist (target: ≥500 confirmed true positives, ≥500 confirmed false positives), train supervised failure prediction models. These models directly predict "failure within 12–24 hours" with calibrated probabilities.

The unsupervised models continue running as complementary detectors for novel failure modes.

### Phase 2: Handling Class Imbalance

As pseudo-labels accumulate (confirmed anomalies from field feedback), class imbalance becomes a concrete problem. True failures are rare — expect a 20:1 to 50:1 ratio of normal-to-anomalous labeled examples. Imbalanced training data causes supervised models to optimize for the majority class (predicting "normal" for everything achieves 95%+ accuracy but is useless).

**Mitigation strategies, in recommended order**:

1. **Loss weighting (start here)**. Apply class weights inversely proportional to class frequency: `weight_anomalous = N_normal / N_anomalous`. For a 20:1 ratio, anomalous examples receive 20× the loss contribution. This is the simplest approach and requires no data manipulation — just a parameter change in the loss function. Supported natively in scikit-learn (`class_weight='balanced'`) and PyTorch (`CrossEntropyLoss(weight=...)`).

2. **Undersampling the majority class**. Randomly sample the normal class down to a manageable ratio (e.g., 5:1 or 3:1). This reduces training set size and speeds up training. Risk: discarding normal examples may lose rare-but-legitimate normal operating modes (e.g., defrost cycles, high-load events). Mitigate by stratifying the undersample across operating modes.

3. **SMOTE (use with caution for IoT data)**. Synthetic Minority Oversampling generates synthetic anomalous examples by interpolating between existing anomalous feature vectors. For tabular IoT features this can work, but be cautious: interpolating between two different failure modes (e.g., a compressor bearing failure and a refrigerant leak) creates synthetic examples that represent no real failure. Validate that SMOTE-generated examples are physically plausible by reviewing a sample with domain experts.

4. **Data augmentation for time series**. For LSTM-AE or sequence-based supervised models, enrich the anomalous class through:
   - **Jittering**: add small Gaussian noise (σ = 0.01–0.05 × feature std) to existing anomalous windows.
   - **Scaling**: multiply feature values by a random factor in [0.9, 1.1] to simulate measurement variation.
   - **Window slicing**: take sub-windows of anomalous sequences (if using sequence models) to create partially-overlapping augmented examples.
   - **Magnitude warping**: apply smooth, random warping curves to the time axis of sensor readings.
   These augmentations are more physically grounded than SMOTE because they simulate realistic sensor noise rather than interpolating between unrelated failures.

5. **Evaluation on imbalanced test sets**. Regardless of training strategy, always evaluate on the natural class distribution (do not balance the test set). Use **Precision-Recall (PR) curves**, not ROC curves. ROC curves are misleadingly optimistic under heavy class imbalance because the false positive rate denominator (N_normal) is large, making even many false positives look like a low rate. PR curves expose the actual tradeoff between precision and recall at each threshold. Report **Average Precision (AP)** as the summary metric.

---

## Cross-References

- [ADR-003: Unsupervised Anomaly Detection First](../05-architecture/adr-003-anomaly-detection-first.md) — strategic rationale
- [Data Contracts](../05-architecture/data-contracts.md) — Contract 3 (feature store), Contract 4 (model input), Contract 5 (model output)
- [Feature Engineering](../02-feature-engineering/) — the ~45 features consumed by all models
- [Model Integration](../04-palantir/model-integration.md) — Foundry training, publishing, and scoring patterns
- [Evaluation Framework](./evaluation-framework.md) — how to evaluate these models without labels
- [Training Pipeline](./training-pipeline.md) — how training data is constructed and splits are designed
