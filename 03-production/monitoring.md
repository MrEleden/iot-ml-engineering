# Production Monitoring for ML Systems

## Why ML Monitoring Is Three Problems, Not One

Traditional application monitoring asks: "Is the service up?" ML monitoring asks three different questions:

1. **Infrastructure**: Are the pipelines running on time?
2. **Data**: Is the input data still shaped like the data the model was trained on?
3. **Model**: Are the model's predictions still meaningful?

Each layer can fail independently. A Transform can run on time (infrastructure healthy) with shifted sensor distributions (data drifted) and still produce reasonable-looking scores (model appears fine) — until operators notice that anomaly alerts stopped firing for devices that are clearly failing. Monitoring must watch all three layers to catch the full range of failure modes.

## Layer 1: Infrastructure Monitoring

Infrastructure monitoring is the foundation. If pipelines don't run, nothing downstream matters.

### What to Monitor

| Metric | Source | Alert Threshold | Why It Matters |
|---|---|---|---|
| Transform build status | Foundry pipeline health | Any failed build | A failed Transform blocks downstream data flow |
| Transform runtime | Build history timeseries | > 2× median runtime for that Transform | Slow builds risk SLA violations or signal data volume spikes |
| Scheduling delays | Gap between expected and actual build start | > 15 minutes behind schedule | Cluster contention or scheduling bugs |
| Dataset freshness | Latest transaction timestamp vs wall clock | Exceeds [contract SLA](../05-architecture/data-contracts.md) | Stale data propagates stale scores |
| Dataset row count per build | Transaction metadata | Zero rows in a build that should always produce rows | Transform ran but produced nothing — logic bug or empty input |
| Compute profile usage | Foundry cluster metrics | Consistent use of x-large when medium would suffice | Cost waste; or conversely, OOMs on small profiles |
| Scoring latency (p50, p95, p99) | Build duration of scoring Transforms | p95 > 30 min for hourly scoring | Scoring that takes too long risks missing the next cycle; sustained high latency signals growing data or model complexity |
| Scoring throughput | Devices scored per minute during batch scoring run | < 2,000 devices/min | Low throughput indicates compute profile is undersized or model inference is inefficient |

### Pipeline SLA Configuration

Configure SLAs on the critical datasets in the pipeline, aligned with [data contract freshness targets](../05-architecture/data-contracts.md):

| Dataset | SLA | Escalation |
|---|---|---|
| `raw_sensor_readings` | Transaction < **2 min** old | 5 min: alert Slack ∙ 15 min: page on-call |
| `clean_sensor_readings` | Transaction < **15 min** old | 30 min: alert Slack ∙ 1 hour: page on-call |
| `device_features` | Transaction < **30 min** old | 1 hour: alert Slack ∙ 2 hours: page on-call |
| `model_scores` (batch) | Transaction < **1 hour** old | 2 hours: alert Slack ∙ 4 hours: page on-call + activate [rules-based fallback](./model-serving.md#rules-based-fallback) |
| `device_alerts` (batch) | Transaction < **5 min** after scoring | 15 min: alert Slack ∙ 30 min: page on-call |
| `device_alerts` (streaming) | End-to-end latency < **1 min** | 2 min: alert Slack ∙ 5 min: page on-call |

### Foundry-Native Implementation

Foundry provides:

- **Data Health**: automatic monitoring of dataset freshness, schema changes, and row counts. Configure alerts in the dataset's Health tab.
- **Pipeline Monitoring**: visual view of Transform dependency chains with build status indicators. Set SLAs per dataset.
- **Build history**: historical build times, success/failure rates, and compute usage per Transform.

No external monitoring tools needed for infrastructure — Foundry's built-in capabilities cover this layer entirely. This aligns with the [Foundry-native constraint](../05-architecture/adr-001-foundry-native.md).

### Time Series Sync Health

When Foundry Time Series indexes are backed by batch datasets, there is a sync lag between the backing dataset's latest transaction and the Time Series index's latest queryable timestamp. If sync stalls, Ontology-driven dashboards and AIP agents see stale data even though the batch pipeline is healthy.

| Metric | Computation | Alert Threshold |
|---|---|---|
| Sync lag | `backing_dataset_latest_transaction_ts - time_series_index_latest_ts` | > 2× expected interval (e.g., > 2 hours for hourly-batch-backed Time Series) |
| Sync failure rate | Failed sync attempts / total sync attempts over 24h rolling window | > 5% failure rate |
| Index staleness | Wall clock − Time Series index latest timestamp | > 3× expected interval |

**Implementation**: schedule a lightweight monitoring Transform that queries the backing dataset's latest transaction timestamp and compares it to the Time Series API's latest data point. Alert if the gap exceeds 2× the expected update interval.

```python
@transform_df(
    Output("/Company/pipelines/refrigeration/monitoring/timeseries_sync_health"),
    backing_meta=Input("/Company/pipelines/refrigeration/scores/batch_model_scores"),
)
def check_timeseries_sync(backing_meta):
    backing_latest = backing_meta.agg(F.max("scored_at")).collect()[0][0]
    ts_index_latest = query_timeseries_latest("device_anomaly_scores")  # platform API
    sync_lag_minutes = (backing_latest - ts_index_latest).total_seconds() / 60

    return spark.createDataFrame([{
        "check_time": datetime.utcnow(),
        "backing_latest": backing_latest,
        "ts_index_latest": ts_index_latest,
        "sync_lag_minutes": sync_lag_minutes,
        "healthy": sync_lag_minutes <= 120,  # 2h threshold for hourly-batch
    }])
```

## Layer 2: Data Monitoring

Data monitoring detects when the input data changes in ways the model doesn't expect. A model trained on winter data may misbehave when summer data starts flowing — the distributions are different even though every pipeline is green.

### Input Distribution Drift

Compare current data distributions to a reference baseline (typically the training data distribution or a recent stable period).

#### What to Track

| Metric | Computation | Alert Threshold |
|---|---|---|
| Sensor value distributions | Per-sensor histogram of values in `clean_sensor_readings`, compared to 30-day rolling baseline | KS test p-value < 0.01 sustained for 3+ consecutive windows (hourly windows for fleet model, daily windows for cohort models) |
| Feature distributions | Per-feature histogram of values in `device_features` | Same KS test threshold and window policy |
| Null rate per sensor | Fraction of null values per sensor per hour | Change > 5 percentage points vs 7-day average |
| quality_flag composition | Fraction of readings with each [quality flag](./data-quality.md#the-quality_flags-system) | Any flag category > 2× its 30-day average rate |
| Sensor completeness distribution | Histogram of `sensor_completeness` values | Median drops below 0.8 |

#### Drift Detection Transform

```python
@transform_df(
    Output("/Company/pipelines/refrigeration/monitoring/feature_drift"),
    current_features=Input("/Company/pipelines/refrigeration/features/device_features"),
    baseline_stats=Input("/Company/pipelines/refrigeration/monitoring/baseline_statistics"),
)
def detect_feature_drift(current_features, baseline_stats):
    # Compute current distribution stats
    current_stats = compute_distribution_stats(current_features)

    # Compare to baseline
    drift_metrics = current_stats.join(baseline_stats, "feature_name")
    drift_metrics = drift_metrics.withColumn(
        "ks_stat",
        compute_ks_statistic(F.col("current_dist"), F.col("baseline_dist"))
    ).withColumn(
        "drift_detected",
        F.col("ks_stat") > 0.1  # threshold for meaningful drift
    )

    return drift_metrics
```

### Drift Type Taxonomy

Understanding *what* has drifted is as important as detecting *that* something drifted. Three types of distribution shift affect ML systems differently:

| Drift Type | Definition | IoT/Refrigeration Example | Detection Difficulty |
|---|---|---|---|
| **Covariate shift** | Input feature distribution P(X) changes, but the relationship P(Y\|X) stays the same | Summer deployment shifts ambient temperature and humidity distributions, but the same sensor patterns still indicate compressor failure | Moderate — detectable via feature distribution monitoring (KS test, PSI) |
| **Concept drift** | The relationship P(Y\|X) changes, even if input distributions stay the same | After a fleet-wide firmware update, the same vibration pattern that previously indicated bearing wear is now normal operating behavior | Hard — requires outcome labels. Primary signal is alert-to-maintenance hit rate declining over time |
| **Label shift** | The prevalence of anomalies P(Y) changes, but the conditional P(X\|Y) stays the same | A batch of defective compressors enters the fleet, increasing the true anomaly rate from 5% to 12% | Moderate — detectable via anomaly rate monitoring, but distinguishing from model degradation requires maintenance feedback |

**Why concept drift is the hardest to catch**: without ground-truth labels (which require maintenance feedback with multi-week lag), concept drift is invisible to input-only monitoring. The model's scores look normal, the input data looks normal, but the scores no longer mean what they used to. The best proxy signal is the alert-to-maintenance hit rate — if operators are increasingly marking alerts as false positives or if maintenance events are happening on devices the model scored as healthy, concept drift is likely.

### Multi-Method Drift Detection

The KS test is the primary univariate drift detection method, but it has limitations. Different statistical tests catch different types of distribution change. Use a suite of methods for comprehensive drift monitoring:

| Method | Type | What It Detects | When to Use | Threshold |
|---|---|---|---|---|
| **Kolmogorov-Smirnov (KS) test** | Univariate, non-parametric | Any change in CDF shape (location, scale, modality) | Primary check on every continuous feature per scoring window | p-value < 0.01 sustained for 3+ consecutive windows |
| **Population Stability Index (PSI)** | Univariate, binned | Magnitude of distribution shift relative to a reference | Periodic retrain eligibility checks (weekly/monthly) | PSI > 0.2 indicates significant shift requiring retraining |
| **KL divergence** | Univariate, distribution-level | Asymmetric information-theoretic distance between distributions | Comparing score distributions across model versions (canary evaluation) | KL > 0.1 blocks model promotion |
| **Chi-squared test** | Categorical | Changes in category proportions | Categorical features (`quality_flag` composition, device model distribution) | p-value < 0.01 for 3+ consecutive windows |
| **Maximum Mean Discrepancy (MMD)** | Multivariate, kernel-based | Joint distribution shifts that univariate tests miss (e.g., correlation structure changes) | Supplementary check on the full feature vector when univariate tests are inconclusive | MMD statistic exceeds permutation-test threshold at α = 0.05 |

**Recommended combination**: KS test as the primary per-feature check (fast, interpretable, already implemented). PSI as a periodic summary metric for retrain decisions. MMD as a supplementary multivariate check when multiple features show borderline KS results but none individually crosses the threshold — correlated shifts across features can be significant even when individual features look stable.

```python
def compute_psi(reference_dist, current_dist, bins=10):
    """Population Stability Index between two distributions."""
    ref_counts, bin_edges = np.histogram(reference_dist, bins=bins)
    cur_counts, _ = np.histogram(current_dist, bins=bin_edges)

    # Avoid division by zero with small epsilon
    ref_pct = (ref_counts + 1e-6) / ref_counts.sum()
    cur_pct = (cur_counts + 1e-6) / cur_counts.sum()

    psi = np.sum((cur_pct - ref_pct) * np.log(cur_pct / ref_pct))
    return psi
```

### Distinguishing Drift from Anomaly

Not all distribution shifts are problems. Some are seasonal (summer vs winter), some reflect fleet composition changes (new device model deployed), and some are genuine data quality issues.

| Shift Type | Example | Response |
|---|---|---|
| Seasonal | Ambient temperature distributions shift from winter to summer | Update baseline statistics to use season-aware reference windows |
| Fleet composition | 10K new devices with different sensor characteristics deployed | Recompute baselines including new devices; may need per-cohort baselines |
| Sensor degradation | Humidity sensors across the fleet gradually drift upward over months | Investigate hardware batch. May need sensor recalibration campaign. |
| Data pipeline bug | Pressure values all shifted by 100 kPa (unit conversion error) | Fix the pipeline bug. This is a data quality issue, not model drift. |

### Sensor Availability Trends

Track which sensors report data over time. A fleet-wide decline in a specific sensor's availability may indicate:

- A common hardware failure across a device model/batch
- A firmware update that disabled a sensor channel
- A connectivity change that truncates messages before all sensors are transmitted

```
Sensor availability dashboard:
- X axis: time (days/weeks)
- Y axis: fraction of active devices reporting each sensor
- One line per sensor type (14 lines)
- Alert: any sensor drops below 90% availability fleet-wide
```

## Layer 3: Model Monitoring

Model monitoring detects when the model itself stops being useful — even if the infrastructure is running and the data looks normal.

### Prediction Distribution Drift

The distribution of anomaly scores should be roughly stable over time. Sharp changes indicate the model's behavior has shifted.

| Metric | Computation | Alert Threshold | Explanation |
|---|---|---|---|
| Mean anomaly score (fleet) | Average `anomaly_score` across all devices per scoring run | Change > 0.05 from 7-day rolling mean | Fleet-wide shift suggests model or data problem |
| Anomaly rate | Fraction of devices with `anomaly_flag = true` | ±3 percentage points from configured contamination rate (typically ~5%) | Rate too high: model or data problem. Rate too low: model may have gone stale. |
| Score standard deviation | Stddev of `anomaly_score` per scoring run | Drop below 0.05 | Model is scoring everything similarly — may be collapsed |
| Score percentile stability | 95th percentile of scores per run | Change > 0.1 from 30-day rolling p95 | The tail of the distribution is shifting |

> **Slice-based monitoring**: aggregate metrics can hide slice-level degradation. A fleet-wide mean anomaly score may look stable while a specific device cohort or region experiences significant drift. Slice model monitoring metrics by:
> - **Device cohort** (equipment model / manufacturer batch) — different hardware ages and degrades differently
> - **Region** (geographic / climate zone) — seasonal effects vary by location
> - **Equipment model** (product line / generation) — sensor characteristics differ across models
>
> If a slice's anomaly rate diverges from its own baseline by >3 percentage points while the fleet aggregate stays flat, investigate the slice independently. This is especially important after fleet expansions where new device cohorts may behave differently from the existing fleet.

### Model Agreement Rates

When running multiple models in parallel (see [Model Serving — Multi-Model](./model-serving.md#multi-model-parallel-scoring)), track how often they agree.

| Metric | Meaning | Alert Threshold |
|---|---|---|
| Pairwise agreement rate | Fraction of devices where two models give the same `anomaly_flag` | Drops below 85% |
| Unanimous agreement rate | Fraction of devices where all models agree | Drops below 75% |
| Disagreement on high scores | Cases where one model scores > 0.8 and another < 0.3 | More than 50 devices per run |

Declining agreement rates suggest one or more models are drifting. The resolution is usually to investigate which model changed behavior and whether the change is justified (e.g., one model adapted to seasonal patterns while another didn't).

### Threshold Effectiveness

The [severity thresholds](../05-architecture/data-contracts.md#severity-rules) (0.6/0.7/0.85/0.95) are not permanent — they need monitoring and periodic adjustment.

| Signal | Possible Issue | Response |
|---|---|---|
| HIGH alerts sustained for many devices with no maintenance action taken | Threshold too aggressive — generating alert fatigue | Raise HIGH threshold from 0.85 to 0.90 |
| Maintenance events happening on devices with LOW anomaly scores | Threshold too conservative — missing real failures | Lower or investigate why model missed these devices |
| CRITICAL alerts rarely/never triggered | Either the fleet is healthy, or the threshold (0.95) is unreachable | Verify that scores actually reach > 0.95 in historical data |

Track the hit rate: of alerts at each severity, what fraction led to actual maintenance? This requires joining `device_alerts` with `MaintenanceEvent` data from the [Ontology](../04-palantir/ontology-design.md#maintenanceevent).

## Foundry-Native Monitoring Implementation

All monitoring for this system runs inside Foundry. No external Grafana, Datadog, or custom monitoring stacks.

### Monitoring Transforms

| Transform | Schedule | Input | Output |
|---|---|---|---|
| `compute_data_drift` | Hourly | `device_features`, `baseline_statistics` | `feature_drift_metrics` |
| `compute_model_drift` | After each scoring run | `model_scores` | `model_drift_metrics` |
| `compute_model_agreement` | After daily scoring | `model_scores` (all models) | `model_agreement_metrics` |
| `compute_alert_hit_rate` | Weekly | `device_alerts`, `maintenance_events` | `alert_effectiveness_metrics` |
| `update_baseline_statistics` | Monthly (or on retrain) | `device_features` (30-day window) | `baseline_statistics` |

### Dataset Expectations on Monitoring Outputs

The monitoring Transforms themselves have expectations — monitoring that watches the monitors:

```python
@expectation(Check(
    lambda df: df.filter(F.col("drift_detected") == True).count()
        <= df.count() * 0.3,
    "No more than 30% of features should show drift simultaneously "
    "(>30% suggests a systemic issue, not individual feature drift)"
))
@transform_df(
    Output("/Company/pipelines/refrigeration/monitoring/feature_drift"),
    ...
)
def detect_feature_drift(...):
    ...
```

If > 30% of features drift simultaneously, it's probably not genuine feature drift — it's a pipeline bug, a baseline that needs updating, or a mass device deployment. The expectation fires and prevents false "drift detected" signals.

## Dashboards

Different audiences need different views. A single dashboard that tries to serve everyone serves no one.

### Operations Dashboard (On-Call Engineer)

**Purpose**: answer "is everything running and does anything need immediate attention?"

| Widget | Source | Interaction |
|---|---|---|
| Pipeline health status (red/yellow/green per Transform) | Pipeline monitoring | Click to see failed build logs |
| Dataset freshness heatmap | SLA compliance metrics | Click to see which upstream dependency is stale |
| Active CRITICAL and HIGH alerts count | `device_alerts` | Click to drill down to specific devices |
| Throughput plot: readings/hour for last 48h | `raw_sensor_readings` row counts | Spot ingestion drops visually |
| Device dropout count | Dropout detection output | Link to dropout-specific investigation view |

### Data Science Dashboard (ML Team)

**Purpose**: answer "are the models still working correctly?"

| Widget | Source | Interaction |
|---|---|---|
| Anomaly score distribution (histogram, per model) | `model_scores` | Compare today vs 7-day rolling baseline |
| Model agreement heatmap (model × model) | `model_agreement_metrics` | Identify which model pair is disagreeing |
| Feature drift summary table | `feature_drift_metrics` | Sort by KS statistic to find most-drifted features |
| Contamination rate trend (per model, over time) | `model_scores` aggregated | Spot gradual drift in anomaly rates |
| False positive rate from operator feedback | `device_alerts` where `status = false_positive` | Track whether models are improving |
| Top-10 most flagged devices | `model_scores` | Investigate whether these are real failures or model artifacts |

### Management Dashboard (Business Stakeholders)

**Purpose**: answer "is the system delivering value?"

| Widget | Source | Interaction |
|---|---|---|
| Devices monitored (total active count) | Device registry | Simple KPI |
| Alerts generated this week by severity | `device_alerts` aggregated | Week-over-week trend |
| Mean time to maintenance after HIGH alert | `device_alerts` joined with `maintenance_events` | Measures operational response speed |
| Estimated equipment failures prevented | Anomaly alerts confirmed true by maintenance close | Value quantification (approximate) |
| System uptime (pipeline SLA compliance rate) | SLA monitoring metrics | Reliability KPI |

## Incident Response

When monitoring fires, someone has to act. These runbooks cover the most common failure modes.

### Runbook 1: Pipeline Stall

**Symptom**: dataset freshness SLA violated. One or more Transforms haven't run.

| Step | Action |
|---|---|
| 1 | Check the failed Transform's build log in Foundry. Common causes: OOM, input dataset locked, dependency cycle. |
| 2 | If OOM: check if input data volume spiked (new devices? burst of catch-up data after an outage?). Scale compute profile up one step and retry. |
| 3 | If dependency: check upstream Transforms. A stall cascades — find the root cause Transform. |
| 4 | If systemic (Foundry platform issue): check Foundry status page. Open support ticket with build IDs. |
| 5 | If stall > 2 hours on scoring pipeline: activate [rules-based fallback](./model-serving.md#rules-based-fallback). |

### Runbook 2: Anomaly Rate Spike

**Symptom**: anomaly rate jumps from ~5% to >15% of the fleet.

| Step | Action |
|---|---|
| 1 | Check if it's a data problem first — did a sensor distribution shift? Look at feature drift dashboard. |
| 2 | Check if it's a model problem — was a new model version deployed? When was the last retrain? |
| 3 | If data shift: is it seasonal or a bug? Check same period last year in `data_quality_daily_summary`. |
| 4 | If model problem: roll back to previous model version (see [Model Serving — Rollback](./model-serving.md#rollback)). |
| 5 | If neither: check individual devices — is a specific site or device model over-represented in the flagged devices? May be a real fleet-wide issue. |
| 6 | If confirmed real: notify operations team. This is the system working correctly, not a monitoring false alarm. |

### Runbook 3: Model Divergence

**Symptom**: model agreement rate drops below 75%.

| Step | Action |
|---|---|
| 1 | Identify which model is the outlier. Compare pairwise agreement rates. |
| 2 | Check if the outlier model was recently retrained. Post-retrain divergence is expected and should converge within 2–3 scoring cycles. |
| 3 | If not recently retrained: check if the outlier model's input data is different (different feature selection, schema mismatch). |
| 4 | If the outlier is genuinely detecting different patterns: manually review 10 of its highest-scored devices. Are the detections plausible? |
| 5 | If the outlier is wrong: exclude it from the agreement-based severity logic until fixed. Retrain on updated data. |
| 6 | If the outlier is right (and other models are wrong): the other models may need retraining. This is a retraining trigger. |

### Runbook 4: Streaming Alert Storm

**Symptom**: streaming path generates hundreds of CRITICAL alerts in minutes.

| Step | Action |
|---|---|
| 1 | Check if it's a fleet-wide event (power grid issue, site-wide failure) or a threshold miscalibration. |
| 2 | If fleet-wide: real emergency. Escalate to operations. The system is working. |
| 3 | If threshold issue: a firmware update may have changed sensor scaling. Check if affected devices share a firmware version. |
| 4 | Temporarily raise streaming thresholds to stop the alert storm while investigating. |
| 5 | After resolution: review and update device-specific thresholds in the Ontology. |

## Retraining Triggers

Models degrade over time. The question is when to retrain — too early wastes compute, too late degrades detection quality.

### Automated Triggers

These conditions are checked by a monitoring Transform after each daily scoring run. If any trigger fires, it creates a ticket for the ML team and optionally initiates an automated retraining pipeline.

| Trigger | Condition | Check Frequency | Automation Level |
|---|---|---|---|
| Feature drift | > 5 features show significant drift (KS stat > 0.1) for 7+ consecutive days | Daily | Auto-retrain (if drift is seasonal, may auto-update baseline instead) |
| Anomaly rate drift | Contamination rate differs from target by > 3 percentage points for 5+ consecutive runs | After each scoring run | Auto-alert ML team; manual decision on retrain |
| Model agreement decline | Agreement rate < 80% for 3+ consecutive daily scoring runs | Daily | Auto-alert ML team |
| False positive rate spike | Operator-labeled false positives exceed 20% of total alerts in a 7-day window | Weekly | Auto-alert ML team; likely needs threshold tuning, not retrain |
| Fleet composition change | > 5% of fleet replaced with new devices in the last 30 days | Monthly | Auto-retrain to include new device behavior |
| Calendar-based | Model hasn't been retrained in > 14 days (Isolation Forest) or > 45 days (Autoencoder) | Daily check | Auto-trigger retraining pipeline |

> **Note:** These calendar-based thresholds are safety-net alerts for when the scheduled cadence (weekly for Isolation Forest, monthly for Autoencoder) fails to execute — not the retraining cadence itself. They fire only if a scheduled retrain was missed.

### Manual Triggers

Some situations require human judgment:

- **After a major firmware rollout**: sensor semantics may change. Retrain all models on post-rollout data.
- **After a significant fleet expansion**: new devices may have different operating patterns. Retrain to include them in the training distribution.
- **After feedback from operations**: "the model never catches X type of failure." This is a feature engineering or training data problem, not just a retraining problem.
- **After a contract change**: if the [data contracts](../05-architecture/data-contracts.md) change (new sensor, new feature), retraining is required to incorporate the new data.

### Retraining Safeguards

Retraining must not silently degrade model quality:

1. Every retrained model goes through [canary deployment](./model-serving.md#canary-deployments) before promotion.
2. [Regression tests](./testing.md#regression-testing) run against the fixed `regression_test_features` dataset.
3. If the retrained model fails canary or regression checks, the retraining is rejected and the ML team is notified with the specific failure reason.
4. The retraining cadence (see [Model Integration](../04-palantir/model-integration.md#retraining-cadence)) is calibrated per model type — Isolation Forest weekly, Autoencoder monthly.

## Cross-References

- [Data Contracts](../05-architecture/data-contracts.md) — SLAs that define monitoring thresholds
- [Model Serving](./model-serving.md) — the scoring pipelines being monitored, fallback behavior
- [Data Quality](./data-quality.md) — data-layer quality checks that feed into data monitoring
- [Model Integration](../04-palantir/model-integration.md) — model evaluation strategies and retraining cadence
- [Ontology Design](../04-palantir/ontology-design.md) — Alert and MaintenanceEvent objects used for hit-rate analysis
- [Transform Patterns](../04-palantir/transform-patterns.md) — scheduling and SLA configuration
- [ADR-002: Batch + Streaming](../05-architecture/adr-002-batch-plus-streaming.md) — why infrastructure monitoring covers two scoring paths
