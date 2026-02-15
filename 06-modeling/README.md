# Modeling

Owned by: **ML Scientist Agent** (`.claude/agents/ml-scientist.md`)

## Why This Section Exists

We need to detect refrigeration equipment anomalies across 100K+ devices — without labeled failure data. This forces every modeling decision through a single constraint: **we cannot compute precision, recall, or any supervised metric directly.** Every document in this section addresses that constraint honestly, explains what we can do instead, and describes when supervised approaches become viable.

The modeling strategy follows [ADR-003: Unsupervised Anomaly Detection First](../05-architecture/adr-003-anomaly-detection-first.md). Models consume features from the [Feature Store](../02-feature-engineering/feature-store.md) (Contract 3) and produce scores conforming to [Contract 5](../05-architecture/data-contracts.md) — a normalized `anomaly_score` in [0, 1], an `anomaly_flag`, and `top_contributors` explaining which sensors drove the score.

## Modeling Strategy Summary

**Phase 1 (current)**: Compare four unsupervised approaches — statistical baselines, Isolation Forest, DBSCAN clustering, and Autoencoders — across three granularity levels (fleet-wide, cohort, per-device). Start with Isolation Forest fleet-wide as the primary model. No single approach is best for all failure modes; running multiple approaches and comparing outputs reduces blind spots.

**Phase 2 (future)**: As field engineers confirm or reject anomaly alerts over 3–6 months, accumulate pseudo-labels. These enable supervised failure prediction models with calibrated probabilities and lower false positive rates.

## Section Contents

| Document | What It Covers |
|----------|----------------|
| [Model Selection](./model-selection.md) | Why these four approaches, how they compare (interpretability, compute, drift robustness), three granularity levels with tradeoffs, phased rollout plan |
| [Training Pipeline](./training-pipeline.md) | How to train without labels — temporal splits (no random splits), device holdouts, contaminated training data, retraining cadence, Foundry integration |
| [Evaluation Framework](./evaluation-framework.md) | The core challenge of evaluating without ground truth — proxy strategies (expert review, maintenance correlation, stability metrics), threshold selection, A/B testing |
| [Experiment Tracking](./experiment-tracking.md) | Foundry-native experiment management — naming conventions, what to log, comparing experiments, promotion criteria, reproducibility |
| [Model Cards](./model-cards.md) | Model card template and two filled-in examples (Isolation Forest fleet-wide, Autoencoder cohort), maintenance schedule |

## Reading Order

Start with [Model Selection](./model-selection.md) to understand what we're building and why. Then [Training Pipeline](./training-pipeline.md) for how models are trained without labels. [Evaluation Framework](./evaluation-framework.md) addresses the hardest question — how do we know if a model is good? [Experiment Tracking](./experiment-tracking.md) covers the operational side of running experiments in Foundry. [Model Cards](./model-cards.md) provides the documentation template for each production model.

## Key Constraints

- **No labeled data**: All evaluation is proxy-based until field feedback accumulates (see [Evaluation Framework](./evaluation-framework.md))
- **Foundry-native**: All training runs as Foundry Transforms, models publish via model adapters (see [Model Integration](../04-palantir/model-integration.md))
- **100K+ devices**: Per-device models are expensive — start with fleet-wide and cohort before scaling to per-device for high-value assets
- **14 sensors → ~45 features**: Features are time-windowed aggregations defined in [Contract 3](../05-architecture/data-contracts.md)
- **Batch + streaming scoring**: Hourly batch for routine monitoring, near-real-time for acute anomalies (see [ADR-002](../05-architecture/adr-002-batch-plus-streaming.md))
- **Monitoring ownership**: ML Scientist defines model performance metrics (maintenance lift, score distribution shape, drift thresholds) and degradation criteria (when metrics indicate retraining is needed). ML Platform implements the monitoring infrastructure (scheduled drift detection Transforms, alerting pipelines, dashboards). Drift detection — including feature distribution shift (KS test, PSI) and concept drift (declining maintenance lift) — is a modeling responsibility defined here and operationalized in [Monitoring](../03-production/monitoring.md)

## Upstream Dependencies

- [Feature Engineering](../02-feature-engineering/) — defines the ~45 features consumed by all models
- [Data Contracts](../05-architecture/data-contracts.md) — schema definitions for model input (Contract 4) and output (Contract 5)
- [Model Integration](../04-palantir/model-integration.md) — Foundry training, publishing, and scoring patterns
- [ADR-003](../05-architecture/adr-003-anomaly-detection-first.md) — rationale for unsupervised-first approach
