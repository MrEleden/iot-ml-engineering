# Experiment Tracking

## Why Track Experiments

Model development for unsupervised anomaly detection is inherently iterative. There is no single accuracy number to optimize — instead, we compare proxy metrics ([Evaluation Framework](./evaluation-framework.md)) across different model types, hyperparameters, feature sets, and training windows. Without systematic tracking, experiment comparison becomes guesswork: "which Isolation Forest worked better — the one with contamination=0.03 or 0.05? What features did it use? What was the training window?"

Every experiment must be reproducible (same data + same code + same config = same model) and comparable (metrics computed on the same validation set, stored in the same format).

---

## Foundry-Native Approach

### No MLflow

Foundry does not natively support MLflow, W&B, or similar experiment tracking platforms. These tools assume direct filesystem and network access that Foundry Code Repositories don't provide. Do not attempt to install or run them — they will fail at import or connection time.

Instead, experiment tracking uses **Foundry datasets** — the primitive that Foundry handles well. Every experiment produces a row in an experiment log dataset, and every experiment's detailed metrics are written to a metrics dataset. This is less ergonomic than MLflow's UI but is fully queryable, versionable, and integrated with Foundry's access controls and data lineage.

### Experiment Tracking Architecture

```
Training Transform
    ├── Model artifact → Foundry Model Store
    ├── Experiment log row → experiment_log dataset
    └── Detailed metrics → experiment_metrics dataset
```

The training Transform writes:
1. The model artifact via `ModelOutput` (standard Foundry model publishing).
2. A summary row to the `experiment_log` dataset (one row per experiment run).
3. Detailed per-fold and per-metric rows to the `experiment_metrics` dataset.

Both datasets are append-only. No experiment records are deleted or overwritten — this provides a full audit trail.

---

## Experiment Naming and Versioning

### Naming Convention

```
{model_type}_{granularity}_{date}_{sequence}
```

Examples:
- `isolation_forest_fleet_20250115_001`
- `autoencoder_cohort_hvac_20250201_003`
- `zscore_fleet_20250115_001`
- `dbscan_fleet_20250120_001`

**Components**:
- `model_type`: `isolation_forest`, `autoencoder`, `autoencoder_lstm`, `zscore`, `mahalanobis`, `dbscan`
- `granularity`: `fleet`, `cohort_{name}`, `device_{id}` (for per-device models, use a the device ID or "batch" for a batch of per-device models)
- `date`: `YYYYMMDD` of when the experiment was run
- `sequence`: three-digit counter for multiple experiments on the same day (`001`, `002`, ...)

### Version Tracking

Each experiment references:
- **Code version**: the Git commit SHA from the Foundry Code Repository. This pins the exact training code.
- **Data version**: the Foundry dataset transaction ID of the input feature dataset. This pins the exact data used for training.
- **Config version**: hyperparameters and configuration stored in the experiment log. If a config file is used, its content is snapshotted.

Together, code version + data version + config version = full reproducibility. Rerunning the same Transform on the same data transaction with the same config produces the same model (given fixed random seeds).

---

## What to Track Per Experiment

### Experiment Log Schema: `experiment_log`

| Column | Type | Description |
|--------|------|-------------|
| `experiment_id` | `string` | Unique identifier following the naming convention above. |
| `model_type` | `string` | One of: `isolation_forest`, `autoencoder`, `autoencoder_lstm`, `zscore`, `mahalanobis`, `dbscan`, `threshold_baseline`. |
| `granularity` | `string` | One of: `fleet`, `cohort`, `device`. |
| `cohort_id` | `string` | Null for fleet/device granularity. Cohort identifier for cohort models. |
| `training_window_start` | `timestamp` | Start of the training data window. |
| `training_window_end` | `timestamp` | End of the training data window. |
| `validation_window_start` | `timestamp` | Start of the validation data window. |
| `validation_window_end` | `timestamp` | End of the validation data window. |
| `training_rows` | `long` | Number of rows in the training set. |
| `validation_rows` | `long` | Number of rows in the validation set. |
| `training_devices` | `integer` | Number of unique devices in training set. |
| `validation_devices` | `integer` | Number of unique devices in validation set (held out). |
| `feature_count` | `integer` | Number of features in the feature vector. |
| `feature_names` | `array<string>` | Ordered list of feature names used. |
| `hyperparameters` | `string` | JSON-serialized hyperparameter dictionary. |
| `random_seed` | `integer` | Random seed used for reproducibility. |
| `code_commit_sha` | `string` | Git commit SHA from the Code Repository. |
| `data_transaction_id` | `string` | Foundry transaction ID of the input dataset. |
| `python_env_version` | `string` | Python runtime version (e.g., `3.9.16`). |
| `key_dependencies` | `string` | JSON-serialized dict of critical package versions (`{"scikit-learn": "1.3.0", "torch": "2.0.1"}`). |
| `model_artifact_rid` | `string` | Foundry resource ID of the published model artifact. Null if experiment was not published. |
| `status` | `string` | One of: `completed`, `failed`, `promoted`. |
| `promoted_at` | `timestamp` | Null until the experiment is promoted to production. |
| `notes` | `string` | Free-text notes from the ML Scientist. Why was this experiment run? What was the hypothesis? |
| `created_at` | `timestamp` | When the experiment log row was written. |

### Experiment Metrics Schema: `experiment_metrics`

| Column | Type | Description |
|--------|------|-------------|
| `experiment_id` | `string` | Foreign key to `experiment_log`. |
| `metric_name` | `string` | Metric identifier (see below). |
| `metric_value` | `double` | Metric value. |
| `fold` | `integer` | Temporal fold number (1, 2, 3...). Null for aggregate metrics. |
| `split` | `string` | One of: `train`, `validation`, `test`. |
| `computed_at` | `timestamp` | When the metric was computed. |

### Required Metrics

Every experiment must log these metrics (model-type-specific metrics are in addition):

| Metric Name | Description | Applies To |
|-------------|-------------|------------|
| `anomaly_rate_train` | Fraction of training data exceeding the threshold | All |
| `anomaly_rate_validation` | Fraction of validation data exceeding the threshold | All |
| `score_mean_train` | Mean anomaly score on training set | All |
| `score_mean_validation` | Mean anomaly score on validation set | All |
| `score_std_train` | Std of anomaly scores on training set | All |
| `score_std_validation` | Std of anomaly scores on validation set | All |
| `score_skewness_validation` | Skewness of validation score distribution | All |
| `score_p95_validation` | 95th percentile of validation anomaly scores | All |
| `score_p99_validation` | 99th percentile of validation anomaly scores | All |
| `rank_correlation_train_val` | Spearman correlation of device rankings between train and validation scores | All |
| `training_time_seconds` | Wall-clock training time | All |
| `scoring_time_seconds` | Wall-clock time to score the validation set | All |
| `reconstruction_error_mean_train` | Mean MSE on training set | Autoencoder |
| `reconstruction_error_mean_val` | Mean MSE on validation set | Autoencoder |
| `reconstruction_error_std_val` | Std of MSE on validation set | Autoencoder |
| `silhouette_score` | Cluster quality metric | DBSCAN |
| `noise_fraction` | Fraction of devices not assigned to any cluster | DBSCAN |
| `average_path_length_normal` | Mean path length for devices below threshold | Isolation Forest |
| `average_path_length_anomalous` | Mean path length for devices above threshold | Isolation Forest |

---

## Comparing Experiments

### Side-by-Side Evaluation

To compare two experiments (e.g., `isolation_forest_fleet_20250115_001` vs `isolation_forest_fleet_20250122_001`):

1. **Query `experiment_log`**: confirm both experiments used the same validation window and test split. If validation windows differ, the comparison is confounded by temporal differences in the data, not model quality.

2. **Query `experiment_metrics`**: pull metrics for both experiment IDs. Compare:
   - Score distribution metrics (mean, std, skewness, p95, p99) — different distributions mean the models are scoring differently.
   - Anomaly rate — if one model flags 3% and the other flags 8%, they define "anomaly" very differently.
   - Rank correlation between the two models' device rankings — high correlation means they mostly agree on which devices are most anomalous.

3. **Divergence analysis**: find devices scored high by one model and low by the other. Inspect these devices' feature values manually or with expert review. This reveals what each model is sensitive to.

4. **Maintenance lift comparison**: if maintenance data is available, compute maintenance lift for both models on the same time period. The model with higher lift has better predictive value.

### Experiment Comparison Transform

Implement a Foundry Transform that reads `experiment_log` and `experiment_metrics` and produces a comparison summary dataset:

```
Input:  experiment_log, experiment_metrics, experiment_ids to compare
Output: experiment_comparison (side-by-side metric table, divergence summary)
```

This Transform is run manually when evaluating candidate models for promotion. Its output feeds the model promotion decision.

### Workshop Dashboard

Build a Foundry Workshop dashboard for experiment comparison:

- **Experiment table**: filterable list of experiments from `experiment_log` (model type, granularity, date, status).
- **Metric charts**: overlay score distributions from selected experiments.
- **Anomaly rate trends**: plot anomaly rate over time across experiments — is the rate stable or drifting?
- **Divergence explorer**: for two selected experiments, show devices with the largest score difference.

This dashboard replaces the model comparison UI that Foundry's built-in model tooling provides for supervised models (which is unusable for unsupervised models — see [Model Integration, Limitation 1](../04-palantir/model-integration.md)).

---

## Promoting Experiments to Production

### Promotion Criteria

An experiment is promoted to production (scoring the full fleet on a schedule) when it meets all of the following:

1. **Status is `completed`** — the experiment ran without errors.
2. **Validation anomaly rate is within expected range** — between 1% and 10% (configurable). An anomaly rate of 0% means the model isn't detecting anything. An anomaly rate of 40% means the model is too sensitive.
3. **Score distribution is right-skewed** — skewness > 1.0 on the validation set. A symmetric distribution means the model isn't discriminating between normal and anomalous devices.
4. **No regression on proxy metrics** — if expert labels or maintenance lift data is available, the new experiment's metrics are not significantly worse than the current production model's.
5. **Stability check passes** — scoring the same data twice with the model produces identical results (determinism, per Contract 5). This verifies the random seed is set correctly.
6. **Alert volume impact is acceptable** — estimated alert volume (anomaly rate × fleet size × threshold mapping) is within ±30% of current levels, or the operations team has approved the change.

### Promotion Process

1. ML Scientist runs the experiment comparison Transform and reviews the Workshop dashboard.
2. If criteria are met, ML Scientist updates the `experiment_log` row: `status = "promoted"`, `promoted_at = now()`.
3. The scoring Transform's `ModelInput` is updated to reference the new model artifact's `model_artifact_rid`.
4. The next scheduled scoring run uses the new model.
5. Monitor the first 3 scoring runs closely — compare alert volume and score distribution against expectations.

### Rollback

If a promoted model produces unexpected results (alert volume spike, maintenance lift drop, operator complaints):

1. Update the scoring Transform's `ModelInput` back to the previous model artifact.
2. Update `experiment_log`: `status = "rolled_back"`, add a note explaining why.
3. Investigate the issue — rerun the comparison analysis between the new and old models on the problematic scoring period.

---

## Reproducibility

### What Must Be Reproducible

Given an `experiment_id`, it should be possible to reproduce the exact model artifact. This requires:

1. **Code**: the `code_commit_sha` in `experiment_log` points to the exact Code Repository commit. Check out that commit and you have the training code.
2. **Data**: the `data_transaction_id` in `experiment_log` points to the exact Foundry dataset transaction. Reading the dataset at that transaction gives you the exact training data.
3. **Config**: the `hyperparameters` field in `experiment_log` contains the full hyperparameter set.
4. **Random seed**: the `random_seed` field controls all randomness in the training pipeline.
5. **Environment**: `python_env_version` and `key_dependencies` in `experiment_log` specify the runtime.

### What Is Not Reproducible

- **Floating-point determinism across hardware**: different CPU architectures may produce slightly different results for the same computation. Foundry doesn't guarantee the same hardware for every Transform execution. Accept small numerical differences (< 1e-6) in reproduced models.
- **Order of partitions in Spark**: if the training Transform reads a Spark DataFrame and converts to Pandas, row order depends on Spark partitioning. Use explicit sorting (by `device_id`, `window_start`) before conversion.

### Reproducibility Checklist

- [ ] `random_seed` is set in the training code and logged to `experiment_log`
- [ ] Input dataset is referenced by path and transaction ID (not "latest")
- [ ] Python dependencies are pinned to exact versions in `meta.yml`
- [ ] Training data sort order is explicit (not reliant on Spark partition order)
- [ ] Hyperparameters are logged as a complete dictionary (no defaults left implicit — log the default value even if you didn't change it)
- [ ] The experiment can be re-run from the logged `code_commit_sha` + `data_transaction_id` + `hyperparameters` and produces a model with score differences < 1e-4

---

## Cross-References

- [Evaluation Framework](./evaluation-framework.md) — the proxy metrics tracked in experiment_metrics
- [Training Pipeline](./training-pipeline.md) — how training data is constructed and splits are designed
- [Model Integration](../04-palantir/model-integration.md) — Foundry model publishing and scoring patterns
- [Model Selection](./model-selection.md) — the model types being compared in experiments
- [Model Cards](./model-cards.md) — experiment_id reference in production model documentation
