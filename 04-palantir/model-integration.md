# Model Integration — Unsupervised Anomaly Detection in Foundry

How to train, publish, version, and serve unsupervised anomaly detection models within Foundry's model framework — including the workarounds required because Foundry's model tooling was designed primarily for supervised learning.

## Why This Is Non-Trivial

Foundry's model framework (model objectives, model evaluation, model staging/release) assumes supervised models: you have labeled ground truth, you compute accuracy/precision/recall, and you compare model versions on holdout sets. Unsupervised anomaly detection has none of that — there's no labeled "anomalous" column in the training data. This means we need to adapt Foundry's model primitives to work with unsupervised evaluation strategies (reconstruction error distributions, silhouette scores, expert-labeled false positives).

See [Foundry Platform Reference](./foundry-platform-reference.md) for general model and Transform concepts.

## Model Training in Foundry

### Code Repositories

All model training code lives in Foundry Code Repositories. This gives us:

- **Version control**: Git-backed, with branch-based development (feature branch → staging → release)
- **Reproducibility**: the repository pins Python package versions and references input datasets by path, so any build can be reproduced
- **CI checks**: Foundry runs checks on merge — linting, type checking, and (if configured) test execution

### Training Transform Pattern

A model training Transform reads the feature dataset and produces a serialized model artifact:

```python
from transforms.api import transform, Input, Output
from palantir_models.transforms import ModelOutput

@transform(
    model_output=ModelOutput("/Company/models/refrigeration/isolation_forest"),
    features=Input("/Company/pipelines/refrigeration/features/device_features"),
)
def train_isolation_forest(features, model_output):
    import pandas as pd
    from sklearn.ensemble import IsolationForest
    
    df = features.dataframe().toPandas()
    
    feature_cols = [
        "temp_evaporator_mean",
        "temp_evaporator_std",
        "current_compressor_mean",
        "current_compressor_std",
        "pressure_high_mean",
        "vibration_compressor_mean",
        "superheat_mean",
        "subcooling_mean",
        "reading_count",
        "current_per_temp_spread",
    ]
    
    X = df[feature_cols].fillna(0)
    
    model = IsolationForest(
        n_estimators=200,
        contamination=0.05,  # expected anomaly rate — tuned from expert feedback
        random_state=42,
    )
    model.fit(X)
    
    model_output.publish(
        model_adapter=RefrigerationAnomalyAdapter(model, feature_cols),
    )
```

### Model Adapter

Foundry requires a model adapter class that defines how to call the model at inference time. This is the bridge between Foundry's model framework and your `sklearn`/`PyTorch`/custom model:

```python
from palantir_models.models import ModelAdapter
import palantir_models as pm

class RefrigerationAnomalyAdapter(ModelAdapter):
    
    def __init__(self, model, feature_cols):
        self.model = model
        self.feature_cols = feature_cols
    
    @classmethod
    def api(cls):
        # Define the input/output schema for inference
        inputs = {"features_df": pm.Pandas(columns=[
            pm.Column("device_id", pm.String),
            pm.Column("temp_evaporator_mean", pm.Double),
            pm.Column("temp_evaporator_std", pm.Double),
            # ... remaining feature columns per Contract 3
        ])}
        outputs = {"scores_df": pm.Pandas(columns=[
            pm.Column("device_id", pm.String),
            pm.Column("anomaly_score", pm.Double),
            pm.Column("anomaly_flag", pm.Boolean),
            pm.Column("threshold_used", pm.Double),
            pm.Column("top_contributors", pm.Array(pm.String)),
            pm.Column("contributor_scores", pm.Array(pm.Double)),
            pm.Column("sensor_completeness", pm.Double),
            pm.Column("raw_score", pm.Double),
        ])}
        return inputs, outputs
    
    def predict(self, features_df):
        import numpy as np

        THRESHOLD = 0.7  # configured per model; matches Contract 6 LOW severity floor

        X = features_df[self.feature_cols].fillna(0)
        
        raw_scores = self.model.decision_function(X)
        # Normalize to 0-1 range (higher = more anomalous)
        anomaly_scores = 1 - (raw_scores - raw_scores.min()) / (raw_scores.max() - raw_scores.min())

        # Compute per-feature contributions via deviation from mean
        contributions = np.abs(X.values - X.mean().values) / (X.std().values + 1e-8)
        top_k = 5
        top_indices = np.argsort(-contributions, axis=1)[:, :top_k]
        top_contributors = [[self.feature_cols[i] for i in row] for row in top_indices]
        contributor_scores_list = [contributions[r, top_indices[r]].tolist() for r in range(len(contributions))]

        # Compute sensor_completeness from non-null fraction of input features
        sensor_completeness = features_df[self.feature_cols].notna().mean(axis=1).values

        result = features_df[["device_id"]].copy()
        result["anomaly_score"] = anomaly_scores
        result["anomaly_flag"] = anomaly_scores >= THRESHOLD
        result["threshold_used"] = THRESHOLD
        result["top_contributors"] = top_contributors
        result["contributor_scores"] = contributor_scores_list
        result["sensor_completeness"] = sensor_completeness
        result["raw_score"] = raw_scores
        
        return {"scores_df": result}
```

### Multi-Model Approach

We train multiple unsupervised models for comparison:

| Model | Strengths | Foundry-Specific Notes |
|---|---|---|
| Isolation Forest | Fast, handles high-dimensional data, interpretable feature importances | Scikit-learn ships with Foundry's default Python environment — no extra dependencies |
| Autoencoder | Captures nonlinear relationships, reconstruction error is intuitive | Requires PyTorch — add to `meta.yml` dependencies. Training is slower; consider using a large compute profile |
| DBSCAN / clustering | Detects group-level anomalies (fleet segments behaving differently) | Not a scorer — use it to segment the fleet, then train per-segment models |
| Statistical baselines | Per-sensor z-scores, simple thresholds | Implement as a Transform, not a model — no model adapter needed. Useful as a comparison baseline |

Each model gets its own Code Repository and model artifact. The scoring pipeline runs all models and writes results to the `AnomalyScore` object type (see [Ontology Design](./ontology-design.md)), allowing analysts to compare model outputs side by side.

### Model Cards

Each model variant should have a corresponding model card — a structured document stored alongside the model artifact in the Code Repository. The model card provides essential context for anyone reviewing, deploying, or auditing the model:

| Field | Content |
|---|---|
| **Name / Version** | e.g., `isolation_forest_v3`, version `2026.02.01` |
| **Intended Use** | Unsupervised anomaly detection on refrigeration device sensor data for predictive maintenance prioritization |
| **Training Data** | Dataset path, transaction tag, date range, device count, schema version |
| **Evaluation Metrics** | Score distribution stability (KL divergence < 0.05), contamination rate (4.8% ± 0.5%), precision-at-top-100 (72% on expert-labeled holdout) |
| **Limitations** | Trained on temperate-climate fleet; performance on tropical-climate devices is unvalidated. Requires minimum 100 readings in the scoring window for reliable scores. |
| **Ethical Considerations** | Scores may vary by equipment age; older devices naturally drift toward higher scores. Alerts should not be used as sole basis for equipment decommissioning decisions. |

Update the model card on every model version publish. Store model cards as Markdown files in the Code Repository's `docs/` directory and link them from the model objective's description field in Foundry.

## Model Publishing and Versioning

### Publishing Flow

1. **Train** on a feature branch in Code Repositories → produces a draft model artifact
2. **Evaluate** in a notebook or evaluation Transform (see [Evaluation](#model-evaluation) below) → assess quality
3. **Merge to staging** → Foundry CI runs, model artifact is published to the staging model store
4. **Promote to release** → the model artifact becomes the active model for production scoring Transforms

### Versioning Strategy

Each model publish creates a new version. Foundry tracks:

- **Model version ID**: auto-generated, tied to the Code Repository commit
- **Input dataset transaction**: which version of the training data was used
- **Dependency versions**: Python package versions, adapter code hash

For rollback: the scoring Transform references a specific model version. To roll back, update the scoring Transform to reference the previous version and rebuild.

### Retraining Cadence

- **Isolation Forest**: retrain weekly — fast enough to run as a scheduled Transform
- **Autoencoder**: retrain monthly — training is expensive, and reconstruction error distributions are relatively stable
- **Retraining triggers**: retrain immediately if false positive rate (from [Ontology action feedback](./ontology-design.md)) exceeds a threshold, or if fleet composition changes significantly (large batch of new devices deployed)

## Batch Scoring via Transforms

### Model-as-Transform Pattern

The primary scoring mechanism: a Foundry Transform that loads a published model and applies it to the latest feature dataset.

```python
from transforms.api import transform, Input, Output
from palantir_models.transforms import ModelInput

@transform(
    output=Output("/Company/pipelines/refrigeration/scores/isolation_forest_scores"),
    features=Input("/Company/pipelines/refrigeration/features/device_features"),
    model=ModelInput("/Company/models/refrigeration/isolation_forest"),
)
def score_isolation_forest(features, model, output):
    features_df = features.dataframe().toPandas()
    
    result = model.transform({"features_df": features_df})
    scores_df = result["scores_df"]
    
    # Add metadata
    scores_df["model_id"] = "isolation_forest"
    scores_df["model_version"] = model.version
    scores_df["scored_at"] = datetime.utcnow()
    
    output.write_dataframe(
        spark.createDataFrame(scores_df),
        partition_cols=["scored_at_date"],
    )
```

### Scoring Schedule

| Scoring type | Schedule | Use case |
|---|---|---|
| Hourly batch | Every hour | Routine fleet monitoring — populates `Device.latest_anomaly_score` via pre-hydration |
| Daily batch | Daily at 03:00 UTC | Full feature set scoring with all model variants — feeds model comparison dashboards |
| On-demand | Manual trigger or API | Re-score specific devices after maintenance to validate the intervention |

### Scaling Concerns

Scoring 100K devices with an Isolation Forest is fast — seconds in Pandas. But the Autoencoder scoring Transform runs inference on a PyTorch model, which is slower. For the Autoencoder:

- Use a large compute profile on the scoring Transform
- Score in batches of 10K devices to avoid OOM on the Pandas side
- Consider keeping the model in Pandas-only mode (no GPU) — Foundry Transforms don't reliably expose GPU resources

## Model Evaluation

### The Supervised Evaluation Problem

Foundry model objectives expect a labeled evaluation dataset: ground truth + predictions → accuracy metrics. We don't have ground truth labels for anomaly detection. Workarounds:

### Evaluation Strategy 1: Proxy Metrics

Compute metrics that don't require labels:

- **Score distribution stability**: compare the histogram of anomaly scores between the current and previous model version. A large shift suggests the model is learning something different — investigate before promoting.
- **Reconstruction error distribution** (Autoencoder): the mean and variance of reconstruction error should be stable across retrains.
- **Contamination rate consistency**: the fraction of devices scored above the alert threshold should be roughly consistent with the configured contamination rate.

Implement these as a Transform that reads the model's scoring output and produces a metrics dataset. Attach dataset expectations to flag anomalous metric values.

### Evaluation Strategy 2: Expert-Labeled Holdout

Over time, accumulate labels from the `Mark False Positive` and `Resolve Alert` actions (see [Ontology Design](./ontology-design.md)):

- `false_positive` alerts where the model flagged an anomaly but the technician found nothing wrong
- `resolved` alerts where the model correctly identified a real issue

Build a labeled evaluation dataset from these actions. Initially this dataset is small and biased (technicians only inspect high-confidence alerts), but it grows over time and becomes increasingly valuable. Use it to compute precision-at-top-K (of the top 100 alerts per day, how many were real issues?).

### Evaluation Strategy 3: Maintenance Correlation

Correlate anomaly scores with subsequent maintenance events. If a device had a high anomaly score and received corrective maintenance within 7 days, that's a proxy for a true positive. No individual label is reliable, but aggregate statistics (what fraction of high-score devices needed maintenance?) are a useful model quality signal.

### Wiring Evaluation into Foundry Model Objectives

Create a model objective with custom metric definitions:

- **Primary metric**: precision-at-top-100 (from expert labels, when available)
- **Secondary metric**: score distribution KL divergence vs previous version
- **Guard metric**: contamination rate within ±2% of target — if the new model flags 20% of the fleet instead of 5%, block promotion

If expert labels are insufficient (< 200 labeled examples), fall back to distribution-based metrics only and require manual approval for model promotion.

## Shadow Deployment and Model Comparison

Before promoting a new model version, run it alongside the production model to validate its behavior on live data. This section covers the shadow scoring pattern and progressive rollout strategy.

### Shadow Scoring Pattern

Deploy two models simultaneously:

- **Champion**: the current production model. Its scores drive alerts, populate `Device.latest_anomaly_score`, and feed operational dashboards.
- **Challenger**: the candidate model. Its scores are recorded for comparison but do not trigger alerts or update operational properties.

Both models run in the same scoring Transform (or parallel Transforms on the same schedule), reading the same feature dataset. Results are written to the `AnomalyScore` backing dataset with distinct `model_id` and `deployment_role` values.

**Comparison Transform**: a downstream Transform reads both champion and challenger scores and computes comparison metrics:

```python
@transform_df(
    Output("/Company/pipelines/refrigeration/scores/model_comparison"),
    champion=Input("/Company/pipelines/refrigeration/scores/isolation_forest_scores"),
    challenger=Input("/Company/pipelines/refrigeration/scores/isolation_forest_v4_scores"),
)
def compare_models(champion, challenger):
    joined = champion.join(challenger, on=["device_id", "scored_at_date"], how="inner")
    return (
        joined.select(
            "device_id",
            "scored_at_date",
            F.col("champion.anomaly_score").alias("champion_score"),
            F.col("challenger.anomaly_score").alias("challenger_score"),
            F.abs(F.col("champion.anomaly_score") - F.col("challenger.anomaly_score")).alias("score_diff"),
            (F.col("champion.anomaly_flag") & F.col("challenger.anomaly_flag")).alias("both_alert"),
            (F.col("champion.anomaly_flag") & ~F.col("challenger.anomaly_flag")).alias("champion_only"),
            (~F.col("champion.anomaly_flag") & F.col("challenger.anomaly_flag")).alias("challenger_only"),
        )
    )
```

Key metrics from the comparison:

- **KL divergence** between champion and challenger score distributions: large divergence (>0.1) signals fundamentally different model behavior — investigate before promoting
- **Alert overlap**: fraction of devices flagged by both models. Target > 80% overlap for a safe promotion.
- **Disagreement analysis**: devices flagged only by the challenger are the most interesting — review them with domain experts to determine whether the challenger is finding real anomalies the champion misses

**Promotion gate**: promote the challenger only after 7+ days of parallel scoring, documented expert review of disagreements, and no regressions on known false positive cases.

### Progressive Rollout

For large fleet changes or high-risk model updates, deploy progressively rather than fleet-wide:

1. **Deploy by region**: update the model version for a single region (e.g., US-West) while other regions continue using the champion
2. **Monitor for 48 hours**: track alert rate, false positive rate, and score distribution for the rollout region. Compare against the region's historical baseline.
3. **Expand or roll back**: if metrics are stable, expand to the next region. If degradation is detected, roll back the region to the champion.

Implement progressive rollout via a **configuration dataset** — a small Foundry dataset mapping `region` to `model_version`:

| region | model_version | effective_date |
|---|---|---|
| US-West | isolation_forest_v4 | 2026-02-15 |
| US-East | isolation_forest_v3 | 2026-01-01 |
| EU-Central | isolation_forest_v3 | 2026-01-01 |

The scoring Transform reads this configuration dataset and loads the appropriate model version per region. No code change is needed to roll out or roll back — only a data update to the configuration dataset.

## Limitations and Workarounds

### Limitation 1: No Native Unsupervised Evaluation

**Problem**: Foundry's model comparison UI assumes accuracy/AUC/F1. These are undefined for unsupervised models.

**Workaround**: Write a custom evaluation Transform that computes proxy metrics and writes them to a metrics dataset. Build a Workshop dashboard to visualize model comparison — it won't use the built-in model comparison UI, but it achieves the same goal.

### Limitation 2: No Real-Time Model Serving API

**Problem**: Foundry does not expose models as REST endpoints for sub-second inference. Batch scoring via Transforms has minimum latency of minutes (the Transform must be scheduled, run, and commit).

**Workaround for urgent anomalies**: Use Foundry streaming Transforms (Flink-based) with the model logic embedded directly in the streaming Transform code — not via the model adapter framework. This trades model versioning elegance for latency. The streaming Transform applies a simplified version of the model (e.g., a threshold-based rule derived from the Isolation Forest's learned thresholds) and writes alerts to the `Alert` backing dataset in near-real-time.

For the full model-adapter-based scoring, accept hourly batch latency. Most refrigeration anomalies develop over hours or days — hourly scoring is sufficient for all but refrigerant leak detection.

### Limitation 3: Model Adapter Runs in Pandas, Not Spark

**Problem**: the `ModelAdapter.predict()` method receives and returns Pandas DataFrames, not Spark DataFrames. For 100K devices, this means the entire scoring dataset must fit in a single executor's memory.

**Workaround**: 100K rows × 50 features × 8 bytes ≈ 40 MB — well within memory limits. This is only a concern if the scoring input grows significantly (e.g., per-reading scoring instead of per-device scoring) or if the model itself is very large (e.g., a deep autoencoder with hundreds of millions of parameters).

If memory becomes an issue, partition the scoring Transform by region — score each region separately and union the results.

### Limitation 4: Dependency Management

**Problem**: Foundry Code Repositories use a `meta.yml` file to declare Python dependencies. Complex dependency trees (PyTorch + specific CUDA version + custom libraries) can cause resolution failures.

**Workaround**: Pin exact versions in `meta.yml`. Use Foundry's Conda-based environment (not pip-only) for packages with native dependencies. Test dependency resolution in a branch *before* merging to staging — a broken `meta.yml` in staging blocks all model training.

### Limitation 5: No Native Feature Importance for Unsupervised Models

**Problem**: Foundry's model inspection tools expect feature importance from supervised models. Unsupervised models don't provide this directly.

**Workaround**: Compute feature contributions manually:
- **Isolation Forest**: use `sklearn`'s `feature_importances_` attribute (available since v1.0), or compute per-feature anomaly score contributions via permutation importance
- **Autoencoder**: compute per-feature reconstruction error and rank by contribution to total error

Write feature contributions to the `AnomalyScore.component_scores` property (see [Ontology Design](./ontology-design.md)) so operators can see *why* a device was flagged, not just the overall score.

## Production Model Monitoring

Deployed models degrade over time as fleet composition changes, sensor behavior drifts, and environmental conditions shift. Continuous monitoring detects degradation early and triggers corrective action before operational impact.

### Continuous Performance Tracking

A scheduled Transform (daily) computes production monitoring metrics and writes them to a `model_monitoring` dataset:

| Metric | Computation | Threshold | Action |
|---|---|---|---|
| Score distribution mean | Mean anomaly score across all scored devices | Shift > 2σ from 30-day baseline | Investigate: model or data drift? |
| Alert rate | Fraction of devices with `anomaly_flag = true` | > 10% or < 1% (vs expected ~5%) | Review threshold calibration |
| False positive rate | Fraction of alerts marked `false_positive` in last 7 days | > 15% sustained for 7 days | Trigger retraining |
| Feature drift (per-feature) | KL divergence of each input feature vs training distribution | > 0.1 for any feature | Investigate data pipeline or real change |
| Prediction latency | Time from feature dataset update to scoring output commit | > 2x historical average | Investigate Transform performance |

Each metric row includes `model_id`, `model_version`, `computed_at`, and the metric value. A Workshop dashboard visualizes these metrics as time series with threshold lines for quick triage.

### Automated Retraining Triggers

Three conditions trigger automated retraining:

1. **Performance trigger**: false positive rate exceeds 15% for 7 consecutive days. The `model_monitoring` dataset tracks this via a rolling 7-day window. When the condition is met, a Foundry pipeline trigger initiates the training Transform with the latest feature data.

2. **Data drift trigger**: KL divergence exceeds 0.1 for any input feature, sustained for 3 consecutive daily measurements. This indicates the input data distribution has shifted enough that the model's learned boundaries are no longer appropriate. The retraining pipeline re-fits the model on recent data (last 90 days) to capture the new distribution.

3. **Fleet change trigger**: more than 5% of the fleet consists of new devices (installed within the last 30 days) that the current model has never seen during training. New device models or deployments in new climate zones can introduce sensor behavior patterns not represented in the training data. Detected by comparing the device registry's `install_date` distribution against the training data's device set.

All retraining runs produce a challenger model, not an immediate replacement. The challenger must pass the shadow scoring and promotion gate process described above before becoming the new champion.

### Rollback Procedure

If a promoted model degrades in production:

1. **Detection**: the monitoring Transform flags metric threshold breaches (e.g., alert rate spikes to 12%, false positive rate rises)
2. **Rollback action**: update the model version in the configuration dataset (or the model reference in the scoring Transform) to the previous champion version
3. **Effect**: the next scheduled scoring run uses the rolled-back model. No code change, no deployment, no Transform rebuild required.
4. **Verification**: confirm that the rolled-back model's metrics return to baseline within 24 hours
5. **Root cause**: investigate why the promoted model degraded — was it data drift, a training data issue, or a genuine distribution shift? Document findings in the model card.

Keep the last 3 model versions in the model store at all times to ensure rollback targets are always available.

## Related Documents

- [Streaming Ingestion](./streaming-ingestion.md) — the data pipeline that feeds the feature store
- [Transform Patterns](./transform-patterns.md) — how scoring and training Transforms are structured
- [Ontology Design](./ontology-design.md) — `AnomalyScore` and `Alert` object types that receive model output, plus action-based feedback loops
- [Foundry Platform Reference](./foundry-platform-reference.md) — general model and Transform concepts
