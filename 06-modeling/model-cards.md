# Model Cards

## Why Model Cards

Every model that scores production devices needs documentation that answers: what does this model do, what data was it trained on, how well does it work, and where does it fail? Without model cards, knowledge about model behavior lives only in the heads of whoever trained it. When the model drifts, produces unexpected results, or needs retraining, the card is the starting point for investigation.

Model cards are especially important for unsupervised models because evaluation is proxy-based ([Evaluation Framework](./evaluation-framework.md)). The card must be explicit about what we know, what we don't know, and what assumptions we're making.

---

## Template

Every production model must have a model card following this structure. The template is derived from the ML Scientist role definition (`.claude/agents/ml-scientist.md`) with extensions for unsupervised anomaly detection.

```markdown
# Model: [model_name]

## Overview
- **Task**: Unsupervised anomaly detection
- **Target**: Identify refrigeration devices exhibiting abnormal behavior relative to [fleet / cohort / device baseline]
- **Model type**: [Isolation Forest / Autoencoder / Statistical / DBSCAN]
- **Granularity**: [fleet / cohort / device]
- **Cohort**: [cohort name and definition, if cohort granularity. "N/A" for fleet/device.]
- **Model age**: [number of days since last training completion] days
- **Staleness warning**: [if model age > 90 days, display: "⚠️ MODEL STALE — last trained [date]. Recommended retraining cadence: [weekly/monthly]. Investigate why retraining has not occurred." Otherwise: "Model is within expected retraining cadence."]

## Features
- **Feature store version**: [Contract 3 dataset path and transaction ID]
- **Feature count**: [number]
- **Feature list**: [ordered list of feature names, or reference to experiment_log entry]
- **Feature selection rationale**: [why this subset? all features, or a filtered set?]

## Training Data
- **Training window**: [start date] to [end date]
- **Training rows**: [count]
- **Training devices**: [count of unique devices]
- **Exclusion criteria applied**: [what data was excluded and why]
- **Contamination handling**: [iterative cleaning applied? how many rows removed?]

## Validation Data
- **Validation window**: [start date] to [end date]
- **Validation rows**: [count]
- **Validation devices**: [count, confirming these are held-out devices]
- **Temporal gap**: [days between training end and validation start]

## Hyperparameters
- [hyperparameter_1]: [value] — [brief rationale]
- [hyperparameter_2]: [value] — [brief rationale]
- **Random seed**: [value]

## Metrics
Since this is an unsupervised model, traditional classification metrics (precision, recall, F1)
are not available. The following proxy metrics are reported:

- **Anomaly rate (validation)**: [%] — fraction of validation devices exceeding threshold
- **Score distribution skewness**: [value] — higher = more separation between normal and anomalous
- **Score P95 (validation)**: [value] — 95th percentile anomaly score
- **Score P99 (validation)**: [value] — 99th percentile anomaly score
- **Maintenance lift**: [value]× — flagged devices are [value]× more likely to receive maintenance within 7 days
- **Expert-reviewed precision (top 50)**: [%] or "Not yet evaluated"
- **Median lead time**: [hours] before maintenance event, or "Not yet measured"
- **Flag flip rate**: [%] — fraction of devices changing flag between consecutive scoring windows
- **Rank correlation (consecutive windows)**: [value]

## Baseline Comparison
- **Baseline model**: [threshold_baseline / previous model version]
- **Baseline maintenance lift**: [value]×
- **Baseline anomaly rate**: [%]
- **This model vs. baseline**: [summary of improvement or parity]

## Output Schema
Conforms to Contract 5 (model_scores):
- `anomaly_score`: [0.0, 1.0], normalized via [method: percentile / min-max / CDF]
- `anomaly_flag`: true if anomaly_score > [threshold]
- `threshold_used`: [value]
- `top_contributors`: top 5 features by [method: permutation importance / reconstruction error / z-score magnitude]
- `contributor_scores`: magnitude values for top contributors

## Known Limitations
- [Limitation 1 — be specific]
- [Limitation 2]
- [Limitation 3]

## Fairness and Equity
Anomaly detection models can systematically underserve certain device populations. Document potential equity concerns:

- **Device type coverage**: which device types are underrepresented in the training data? Models trained predominantly on one device family (e.g., walk-in coolers) may have lower sensitivity for underrepresented types (e.g., display cases, blast chillers). List device types with <5% representation in the training set and note whether their anomaly rates differ from the fleet average.
- **Geographic equity**: climate zone affects normal operating ranges. A model trained primarily on data from temperate zones may produce higher false positive rates for devices in tropical or arctic environments. Report anomaly rates by climate zone — if any zone's false positive rate is >2× the fleet average, the model is inequitable for that region.
- **Customer tier impact**: if device monitoring quality varies by customer tier (e.g., enterprise customers have more complete sensor data, higher `sensor_completeness`), the model may be more accurate for well-instrumented devices and less sensitive for devices with sparser data. Report model performance metrics (maintenance lift, expert precision) by customer tier or data completeness quartile.
- **Mitigation actions**: for each equity gap identified above, document planned or implemented mitigations (e.g., cohort-specific models for underrepresented device types, data augmentation for low-data regions, adjusted thresholds for low-completeness devices).

## Failure Modes
- [Scenario where the model produces false positives]
- [Scenario where the model produces false negatives]

## Operational Notes
- **Scoring mode**: [batch hourly / batch daily / both]
- **Scoring Transform path**: [Foundry path]
- **Retraining cadence**: [weekly / monthly]
- **Retraining trigger conditions**: [when to retrain outside of schedule]
- **Fallback behavior**: [what happens if this model fails to score — which model takes over?]

## Experiment Reference
- **Experiment ID**: [from experiment_log]
- **Code commit**: [SHA]
- **Data transaction**: [Foundry transaction ID]

## Version History
| Version | Date | Change | Experiment ID |
|---------|------|--------|---------------|
| v1 | [date] | Initial release | [id] |
| v2 | [date] | [what changed] | [id] |

## Review
- **Last reviewed**: [date]
- **Reviewed by**: [name/role]
- **Next review due**: [date]
```

---

## Example Model Card 1: Isolation Forest Fleet-Wide

```markdown
# Model: isolation_forest_fleet_v1

## Overview
- **Task**: Unsupervised anomaly detection
- **Target**: Identify refrigeration devices exhibiting abnormal behavior relative to the full fleet baseline
- **Model type**: Isolation Forest (scikit-learn `IsolationForest`)
- **Granularity**: Fleet-wide (one model for all 100K+ devices)
- **Cohort**: N/A

## Features
- **Feature store version**: `/Company/pipelines/refrigeration/features/device_features` txn `ri.foundry.main.transaction.00000042`
- **Feature count**: 45
- **Feature list**: all numeric features from Contract 3 — `temp_evaporator_mean`, `temp_evaporator_std`, `temp_evaporator_min`, `temp_evaporator_max`, `temp_evaporator_slope`, `temp_condenser_mean`, `temp_condenser_std`, `temp_condenser_min`, `temp_condenser_max`, `temp_condenser_slope`, `temp_ambient_mean`, `temp_ambient_std`, `temp_discharge_mean`, `temp_discharge_std`, `temp_discharge_max`, `temp_suction_mean`, `temp_suction_std`, `pressure_high_mean`, `pressure_high_std`, `pressure_high_max`, `pressure_low_mean`, `pressure_low_std`, `pressure_low_min`, `pressure_ratio`, `current_compressor_mean`, `current_compressor_std`, `current_compressor_max`, `vibration_compressor_mean`, `vibration_compressor_std`, `vibration_compressor_max`, `vibration_compressor_p95`, `humidity_mean`, `humidity_std`, `door_open_fraction`, `defrost_active_fraction`, `superheat_mean`, `superheat_std`, `subcooling_mean`, `subcooling_std`, `temp_spread_evap_cond`, `temp_spread_discharge_suction`, `current_per_temp_spread`, `sensor_completeness`, `quality_flag_count`, `reading_count`
- **Feature selection rationale**: all numeric features included — Isolation Forest handles high-dimensional data and automatically downweights irrelevant features through random feature selection at each split

## Training Data
- **Training window**: 2025-10-01 to 2025-12-31
- **Training rows**: 432,000 (subsampled from ~650M available rows — stratified by device model)
- **Training devices**: 85,000 (80% of active fleet)
- **Exclusion criteria applied**: `sensor_completeness < 0.7` excluded (4.2% of rows); `reading_count < 10` excluded (1.1% of rows); known maintenance windows excluded for 2,300 devices with work order records
- **Contamination handling**: one round of iterative self-scoring cleaning — top 2% of initial model scores removed from training set (8,640 rows). Retraining on cleaned data.

## Validation Data
- **Validation window**: 2026-01-08 to 2026-01-14
- **Validation rows**: 14,000 (from held-out devices during validation window)
- **Validation devices**: 10,500 (10% holdout — devices not in training set)
- **Temporal gap**: 7 days (2025-12-31 to 2026-01-08)

## Hyperparameters
- `n_estimators`: 200 — standard; increasing to 500 showed <0.5% score change
- `contamination`: 0.05 — initial conservative estimate; aligns with operations team expectation of ~5% fleet anomaly rate
- `max_samples`: 256 — default; balances speed and accuracy
- `max_features`: 1.0 — use all features at each split (random feature at each node is default behavior)
- `random_state`: 42

## Metrics
Since this is an unsupervised model, traditional classification metrics are not available.

- **Anomaly rate (validation)**: 4.8% — 504 of 10,500 held-out devices flagged
- **Score distribution skewness**: 2.7 — clear right skew, most devices score low
- **Score P95 (validation)**: 0.62
- **Score P99 (validation)**: 0.84
- **Maintenance lift**: 2.8× — flagged devices are 2.8× more likely to have a maintenance work order within 7 days (based on available work order data for 40% of fleet)
- **Expert-reviewed precision (top 50)**: 68% — 34 of 50 top-scoring devices confirmed as having genuine anomalies by field engineers (initial review, small sample)
- **Median lead time**: 22 hours — for devices with subsequent maintenance, the first anomaly flag appeared a median of 22 hours before the work order
- **Flag flip rate**: 3.2% — of flagged devices, 3.2% changed flag between consecutive hourly scoring runs
- **Rank correlation (consecutive windows)**: 0.96

## Baseline Comparison
- **Baseline model**: `threshold_baseline` (rule-based: alert if any single sensor exceeds fixed threshold)
- **Baseline maintenance lift**: 1.9×
- **Baseline anomaly rate**: 7.2%
- **This model vs. baseline**: Isolation Forest achieves higher maintenance lift (2.8× vs 1.9×) with a lower anomaly rate (4.8% vs 7.2%). The threshold baseline flags more devices with less predictive value. Expert review confirmed the Isolation Forest finds multi-sensor anomalies (e.g., moderate current + moderate vibration + low subcooling) that no single-sensor threshold catches.

## Output Schema
Conforms to Contract 5 (`model_scores`):
- `anomaly_score`: [0.0, 1.0], normalized via min-max on the `decision_function` output (inverted so higher = more anomalous)
- `anomaly_flag`: true if `anomaly_score > 0.60`
- `threshold_used`: 0.60 (corresponds to ~5% contamination rate on training distribution)
- `top_contributors`: top 5 features by permutation importance (computed as mean decrease in anomaly score when feature values are shuffled)
- `contributor_scores`: absolute decrease in anomaly score per feature

## Known Limitations
- **No per-device sensitivity**: a device with naturally higher vibration (e.g., older compressor model) may be chronically flagged. The fleet model has one threshold for all devices — it cannot distinguish "high for this device" from "high for all devices."
- **Seasonal blind spot**: trained on Oct–Dec (autumn/winter). Summer ambient temperature patterns are not represented. The model will likely flag increased condenser temperatures in summer as anomalous. Retraining in spring with 6+ months of data will mitigate this.
- **Contaminated training data**: despite iterative cleaning, some degrading devices were likely in the training set. The model's sensitivity to slow degradation is reduced because slow degradation patterns are partially learned as "normal."
- **No temporal awareness**: scores each window independently. A device trending upward across 12 windows is no more alarming than one high-scoring window followed by 11 normal ones. Temporal logic is handled downstream in the alert pipeline (sustained anomalies trigger ANOMALY_SUSTAINED alerts per Contract 6).

## Failure Modes
- **False positives during seasonal transitions**: spring/autumn temperature shifts cause ambient, condenser, and evaporator temperatures to change regime. The model trained on winter data flags these transitions. Expected FP rate increases ~2× during March–April and September–October.
- **False negatives for slow leaks**: gradual refrigerant loss reduces subcooling and pressure over weeks. If the leak rate is slow enough, the daily change is within noise, and the model doesn't flag it until the device is significantly degraded. Per-device models with trend features would catch this earlier.
- **False positives for devices with unusual but healthy operating patterns**: a display case in a high-traffic store with 40% door-open fraction looks anomalous to the fleet model (fleet mean is ~10%), but is operating normally for its environment.

## Operational Notes
- **Scoring mode**: batch hourly (routine fleet monitoring) + batch daily (full model comparison with other model types)
- **Scoring Transform path**: `/Company/pipelines/refrigeration/scoring/isolation_forest_fleet_scorer`
- **Retraining cadence**: weekly (Sunday 02:00 UTC)
- **Retraining trigger conditions**: false positive rate from expert feedback >30%; fleet anomaly rate deviates >2× from expected 5%; >5% new devices deployed in past 14 days
- **Fallback behavior**: if scoring Transform fails, use `threshold_baseline` model scores. Alert the ML team. Do not skip scoring — threshold baseline ensures continuous coverage.

## Experiment Reference
- **Experiment ID**: `isolation_forest_fleet_20260115_001`
- **Code commit**: `a3f7b2c` (branch: `release/v1.0`)
- **Data transaction**: `ri.foundry.main.transaction.00000042`

## Version History
| Version | Date | Change | Experiment ID |
|---------|------|--------|---------------|
| v1 | 2026-01-15 | Initial fleet-wide Isolation Forest release | `isolation_forest_fleet_20260115_001` |

## Review
- **Last reviewed**: 2026-02-01
- **Reviewed by**: ML Scientist, Operations Lead
- **Next review due**: 2026-03-01 (monthly)
```

---

## Example Model Card 2: Autoencoder Cohort (Commercial Walk-In Coolers)

```markdown
# Model: autoencoder_cohort_walkin_v1

## Overview
- **Task**: Unsupervised anomaly detection
- **Target**: Identify commercial walk-in cooler devices exhibiting abnormal behavior relative to their cohort baseline
- **Model type**: Dense Autoencoder (PyTorch, 3-layer encoder/decoder)
- **Granularity**: Cohort
- **Cohort**: `walkin_commercial` — commercial walk-in coolers (device model families WC-100, WC-200, WC-300) in temperate climate zones (ASHRAE zones 4–5). 8,200 devices.

## Features
- **Feature store version**: `/Company/pipelines/refrigeration/features/device_features` txn `ri.foundry.main.transaction.00000058`
- **Feature count**: 45
- **Feature list**: same as Isolation Forest fleet model — all numeric features from Contract 3
- **Feature selection rationale**: Autoencoder performs implicit feature selection through the bottleneck layer. All features provided; the model learns which are informative for this cohort. Input features are normalized to [0, 1] range (min-max scaling fitted on training data).

## Training Data
- **Training window**: 2025-07-01 to 2025-12-31 (6 months — longer window for seasonal coverage)
- **Training rows**: 590,000 (from 8,200 devices, 1-hour windows, subsampled from available 15-min windows)
- **Training devices**: 6,560 (80% of cohort)
- **Exclusion criteria applied**: `sensor_completeness < 0.7` excluded; windows during known maintenance excluded; first 48 hours after maintenance events excluded
- **Contamination handling**: no iterative cleaning applied — Autoencoder capacity control (bottleneck dimension) handles contamination by limiting the model's ability to memorize rare anomalous patterns

## Validation Data
- **Validation window**: 2026-01-08 to 2026-01-31
- **Validation rows**: 72,000
- **Validation devices**: 820 (10% holdout from cohort)
- **Temporal gap**: 7 days

## Hyperparameters
- **Architecture**: encoder [45 → 32 → 16 → 10], decoder [10 → 16 → 32 → 45] (symmetric)
- **Latent dimension**: 10 — selected via elbow method on validation reconstruction error across [5, 10, 15, 20]. Dim=10 is the point where validation error stops decreasing meaningfully.
- **Activation**: ReLU (hidden layers), Sigmoid (output layer, since inputs are [0,1] normalized)
- **Learning rate**: 1e-3 (Adam optimizer)
- **Batch size**: 256
- **Epochs**: 47 (early stopping with patience=5 on validation loss; max epochs=100)
- **Dropout**: 0.1 (applied to encoder layers)
- **Random seed**: 42 (PyTorch manual seed + numpy seed)

## Metrics
- **Anomaly rate (validation)**: 3.9% — 32 of 820 held-out devices flagged
- **Score distribution skewness**: 3.4 — strong right skew, cleaner separation than Isolation Forest on this cohort
- **Score P95 (validation)**: 0.55
- **Score P99 (validation)**: 0.79
- **Maintenance lift**: 3.2× — flagged devices are 3.2× more likely to have maintenance within 7 days
- **Expert-reviewed precision (top 30)**: 73% — 22 of 30 top-scoring devices confirmed anomalous (initial cohort-specific review)
- **Median lead time**: 26 hours — longer lead time than fleet Isolation Forest because the Autoencoder detects subtler multi-sensor degradation patterns earlier
- **Flag flip rate**: 2.1% — more stable than Isolation Forest (3.2%) because reconstruction error is less sensitive to small feature fluctuations
- **Rank correlation (consecutive windows)**: 0.97
- **Reconstruction error (mean, validation normal devices)**: 0.018
- **Reconstruction error (mean, validation flagged devices)**: 0.142

## Baseline Comparison
- **Baseline model**: `isolation_forest_fleet_v1` (fleet-wide Isolation Forest)
- **Baseline maintenance lift on this cohort**: 2.4× (lower than fleet-wide average because fleet model doesn't capture cohort-specific normals)
- **Baseline anomaly rate on this cohort**: 6.1% (higher than Autoencoder — fleet model overfits warm-climate normals)
- **This model vs. baseline**: Autoencoder cohort model achieves higher maintenance lift (3.2× vs 2.4×) and lower anomaly rate (3.9% vs 6.1%) on this cohort. The key improvement: the Autoencoder detects compressor efficiency degradation (rising `current_per_temp_spread` + falling `subcooling_mean`) that the fleet model misses because these values are within fleet-wide normal range but abnormal for walk-in coolers.

## Output Schema
Conforms to Contract 5 (`model_scores`):
- `anomaly_score`: [0.0, 1.0], normalized via percentile mapping against the training set's reconstruction error distribution (50th percentile → 0.0, 99.9th percentile → 1.0, capped)
- `anomaly_flag`: true if `anomaly_score > 0.58`
- `threshold_used`: 0.58 (corresponds to ~4% anomaly rate on training distribution)
- `top_contributors`: top 5 features by per-feature reconstruction error (features with highest absolute difference between input and reconstruction)
- `contributor_scores`: per-feature reconstruction error values

## Known Limitations
- **Cohort boundary effects**: devices at the edge of the `walkin_commercial` cohort (e.g., a WC-100 in a borderline climate zone 3/4) may not fit the cohort model well. These devices should be evaluated against both the cohort model and the fleet model.
- **PyTorch dependency**: requires PyTorch in `meta.yml`. Longer build times and potential dependency resolution issues (see [Model Integration, Limitation 4](../04-palantir/model-integration.md)).
- **Longer training time**: ~12 minutes per training run (CPU, Foundry large compute profile) vs. ~5 seconds for Isolation Forest. Limits experimentation velocity.
- **Subtle reconstruction artifacts**: the sigmoid output layer clips reconstructed values to [0, 1]. Features with true values near 0 or 1 (e.g., `door_open_fraction` often at 0.0) may have artificially low reconstruction error, reducing sensitivity to anomalies in those features.
- **No LSTM variant yet**: this dense Autoencoder processes one window at a time. Temporal patterns are captured only through slope and rolling features in the feature layer, not through sequential modeling. LSTM-AE is planned for Phase 2.

## Failure Modes
- **False positives during defrost cycles**: defrost events cause temporary temperature and pressure spikes. If the defrost fraction is unusually high in a window (e.g., a manual defrost), the reconstruction error increases even though the device is healthy. Mitigation: `defrost_active_fraction` is included as a feature so the model should learn to expect some defrost variation — but extreme defrost events are rare in training data.
- **False negatives for new failure modes**: the Autoencoder only flags deviations from *trained* normal patterns. A completely novel failure mode (e.g., a new refrigerant interaction not present in training data) may produce reconstruction patterns the model hasn't seen, but if the affected features happen to have low reconstruction error weighting, the anomaly score may not exceed the threshold.
- **False positives after firmware updates**: firmware changes can shift sensor calibration slightly (e.g., a new vibration measurement algorithm). The model detects the shift as anomalous because reconstruction error increases on the affected features. Requires retraining after fleet-wide firmware updates.

## Operational Notes
- **Scoring mode**: batch daily only (not hourly — compute cost is higher than Isolation Forest). Hourly coverage for this cohort comes from the fleet Isolation Forest model.
- **Scoring Transform path**: `/Company/pipelines/refrigeration/scoring/autoencoder_cohort_walkin_scorer`
- **Retraining cadence**: monthly (1st of month, 01:00 UTC)
- **Retraining trigger conditions**: reconstruction error distribution shift (mean validation error changes by >50% from training baseline); false positive rate from expert feedback >25%; >500 new devices added to cohort
- **Fallback behavior**: if scoring Transform fails, the fleet Isolation Forest model provides coverage. Autoencoder cohort scores are supplementary — they add sensitivity but are not the only model scoring these devices.

## Experiment Reference
- **Experiment ID**: `autoencoder_cohort_walkin_20260201_003`
- **Code commit**: `e9c1d4f` (branch: `release/v1.0-ae`)
- **Data transaction**: `ri.foundry.main.transaction.00000058`

## Version History
| Version | Date | Change | Experiment ID |
|---------|------|--------|---------------|
| v1 | 2026-02-01 | Initial cohort Autoencoder for walk-in coolers | `autoencoder_cohort_walkin_20260201_003` |

## Review
- **Last reviewed**: 2026-02-10
- **Reviewed by**: ML Scientist, Refrigeration Domain Expert
- **Next review due**: 2026-03-10 (monthly)
```

---

## When to Update Model Cards

Update the model card whenever:

- **Model is retrained**: update the training/validation data section, metrics, and hyperparameters (if changed). Add a version history row.
- **New evaluation data is available**: update expert-reviewed precision, maintenance lift, or lead time as new data accumulates.
- **A known limitation is addressed**: remove or modify the limitation entry. Add a note in version history.
- **A new failure mode is discovered**: add it to the failure modes section immediately — don't wait for the next retrain.
- **Operational changes**: scoring schedule, retraining cadence, or fallback behavior changes.

Model cards should be reviewed monthly (or at each retrain, whichever is more frequent). The review confirms that metrics are current, limitations are accurate, and operational notes reflect the actual scoring pipeline state.

---

## Where Model Cards Live in Foundry

Model cards are stored as Markdown files in the Foundry Code Repository alongside the model training code:

```
/Company/code-repositories/refrigeration-models/
├── isolation_forest/
│   ├── src/
│   │   └── train_isolation_forest.py
│   ├── MODEL_CARD.md          ← model card for this model
│   └── meta.yml
├── autoencoder/
│   ├── src/
│   │   └── train_autoencoder.py
│   ├── MODEL_CARD.md
│   └── meta.yml
└── README.md
```

Model cards are version-controlled with the training code. When the model is retrained, the model card is updated in the same commit. This ensures the card and the code are always in sync.

A central index of all production model cards is maintained in this documentation repository ([06-modeling/](./)) for cross-reference. The Foundry Code Repository versions are the source of truth; this doc provides the template and examples.

---

## Cross-References

- [Evaluation Framework](./evaluation-framework.md) — definitions of all proxy metrics referenced in model cards
- [Experiment Tracking](./experiment-tracking.md) — experiment_log and experiment_metrics schemas referenced in model cards
- [Data Contracts](../05-architecture/data-contracts.md) — Contract 5 (model output schema) referenced in output schema section
- [Model Integration](../04-palantir/model-integration.md) — Foundry model adapter and scoring patterns
- [Model Selection](./model-selection.md) — rationale for each model type
- [Training Pipeline](./training-pipeline.md) — training data construction and split design
