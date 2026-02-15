# ADR-001: Foundry-Native Processing

## Status

Accepted

## Context

We need a processing platform that can handle ~14 million sensor readings per minute from 100K+ refrigeration devices, run feature engineering transforms, train and score anomaly detection models, and serve results to operational dashboards — all with lineage, governance, and monitoring.

Two realistic options exist:

1. **Foundry-native**: all transforms, feature computation, model training, and scoring run inside Palantir Foundry using Foundry Transforms, Foundry ML, and the Ontology.
2. **Standalone Spark + external services**: run our own Spark clusters (e.g., on Azure Databricks or HDInsight), manage our own orchestration (Airflow), our own feature store (Feast or Tecton), and push results into Foundry only for visualization.

The organization already has Foundry deployed and uses it for other data products. The IoT sensor data already flows into Foundry via the Azure IoT Hub → Event Hubs → Foundry connector path.

## Decision

All data processing, feature engineering, model training, and model scoring will run as Foundry-native workloads. No standalone Spark clusters, no external orchestrators, no external feature stores.

Concretely, this means:
- **Transforms**: all pipeline code uses `@transform_df` or `@transform` decorators, runs on Foundry-managed Spark
- **Orchestration**: Foundry schedules and pipeline monitoring — no Airflow, no cron
- **Feature store**: Delta Lake datasets (offline) + Ontology properties (online) — no Feast, no Tecton
- **Model training**: Foundry Code Workspaces or Code Repositories with `foundry-ml` — no SageMaker, no standalone MLflow
- **Model serving**: Foundry model objectives for batch scoring, Ontology Actions for real-time responses — no custom serving infrastructure
- **Monitoring**: Foundry dataset expectations, data health checks, pipeline SLAs — no standalone Prometheus/Grafana for data pipeline metrics

## Consequences

### Benefits

**Unified lineage and governance.** Every dataset, transform, model, and dashboard lives in one platform with automatic lineage tracking. When a sensor column changes, we can trace the impact from raw ingestion through features to model scores to alerts. With standalone Spark, we'd need to build and maintain this lineage ourselves.

**No infrastructure management.** Foundry handles Spark cluster provisioning, autoscaling, job scheduling, and failure recovery. With standalone Spark on Azure, we'd own cluster sizing, spot instance management, driver/executor configuration, and version upgrades. For a team focused on ML, this is undifferentiated heavy lifting.

**Consistent incremental processing.** Foundry's incremental transform framework handles change tracking, watermarks, and exactly-once semantics out of the box. Building equivalent incremental pipelines on standalone Spark requires manual checkpoint management and careful state handling.

**Ontology integration.** The online feature store (Ontology properties) and action layer (Ontology Actions) are native to Foundry. Using standalone Spark would require a separate sync mechanism to push features and model scores into the Ontology, adding latency and failure modes.

**Team velocity.** Engineers write PySpark transforms and deploy through Foundry CI/CD. No Docker, no Kubernetes, no Terraform for compute infrastructure. Faster iteration on feature engineering and model experiments.

### Costs and Risks

**Platform lock-in.** All pipeline code uses Foundry-specific decorators (`@transform_df`, `@transform`, `@incremental`). Migrating to another platform would require rewriting every transform. This is a significant commitment, but the organization has already made this commitment for other data products.

**Foundry-imposed constraints.** Some PySpark patterns that work on standalone Spark don't work in Foundry (e.g., arbitrary file system access, custom Spark configurations, some UDF patterns). We accept these constraints in exchange for the benefits above. See [Foundry Platform Reference](../04-palantir/foundry-platform-reference.md) for platform capabilities.

**Scaling ceiling.** At ~14M readings/min, we're pushing volume that requires careful Foundry Spark profile configuration. If we hit Foundry's scaling limits, we have limited options — we can't just throw more Spark executors at the problem the way we could on standalone clusters. Mitigation: start with incremental transforms everywhere, monitor job durations, and engage Palantir support proactively if jobs approach SLA boundaries.

**Vendor dependency for debugging.** When Foundry Spark jobs fail with opaque errors, we depend on Palantir support for root-cause analysis. With standalone Spark, we'd have full access to Spark UI, driver logs, and executor metrics. Foundry exposes some of this, but not all.

**Cost opacity.** Foundry compute costs are bundled into the platform license. It's harder to attribute specific dollar costs to specific pipeline stages compared to standalone Spark on a cloud provider where cost per cluster-hour is transparent.

### ML Lifecycle Capabilities

Choosing Foundry-native means relying on Foundry's ML lifecycle tooling. An honest evaluation of Foundry's support for each lifecycle stage:

**Experiment tracking.** Foundry Code Workspaces support notebook-style experimentation with parameter logging and metric tracking. Results can be persisted to datasets, providing basic experiment tracking. However, this is less mature than dedicated tools like MLflow or Weights & Biases — there is no built-in experiment comparison UI, no hyperparameter search orchestration, and metric visualization requires manual dashboard construction. Mitigation: we define a standardized experiment output schema (see [Experiment Tracking](../06-modeling/experiment-tracking.md)) and build comparison logic as a Foundry transform.

**Model registry.** Foundry ML provides model resource objects that version model artifacts alongside their training metadata. Models are stored as Foundry resources with lineage to their training data and code, which is stronger than standalone MLflow (where lineage must be manually maintained). The registry supports promoting models through stages (development → staging → production) via Foundry's branching and tagging primitives. This is adequate for our needs.

**Model versioning.** Every model artifact in Foundry is version-controlled through the platform's resource versioning. Model versions are tied to specific code commits and training data snapshots, enabling full reproducibility. Semantic versioning is applied manually as a naming convention rather than enforced by the platform.

**A/B testing and shadow deployment.** Foundry does not natively support A/B testing or shadow scoring for models. Implementing shadow deployment (running a new model version alongside the production model and comparing outputs without acting on the new model's results) requires building a custom scoring transform that reads from two model versions and writes both score columns to the output dataset. This is tractable but adds pipeline complexity. See [ADR-002](./adr-002-batch-plus-streaming.md) for the shadow scoring design.

**Automated retraining triggers.** Foundry supports scheduled pipeline runs but does not have built-in concept of drift-triggered retraining. We implement this by running a monitoring transform that computes drift metrics (PSI, score distribution divergence) on each batch scoring run. When drift exceeds configured thresholds, the monitoring transform writes a trigger record to a `retraining_triggers` dataset, which a downstream scheduling mechanism picks up. This is more manual than platforms like SageMaker Model Monitor but is implementable within Foundry's transform framework.

## Alternatives Considered

### Azure Databricks + Delta Lake + Airflow

Would give us full Spark flexibility, native Delta Lake support, and mature orchestration. Rejected because:
- Data already lives in Foundry — we'd be moving data out and back in
- Doubles the infrastructure surface area (Foundry for viz + Databricks for compute)
- Loses Foundry lineage, governance, and Ontology integration
- Requires a separate feature store (Feast/Tecton) and model registry (MLflow)

### Hybrid: Databricks for heavy compute, Foundry for orchestration and serving

Would let us use Databricks for expensive feature computations while keeping Foundry for orchestration and the Ontology layer. Rejected because:
- Adds a cross-platform data sync boundary with its own latency and failure modes
- Foundry's Spark is sufficient for our volume if we use incremental transforms correctly
- Operational complexity of debugging across two platforms outweighs the compute flexibility

## Cross-References

- [System Overview](./system-overview.md) — shows all components running inside Foundry
- [Data Contracts](./data-contracts.md) — contracts enforced via Foundry dataset expectations
- [Foundry Platform Reference](../04-palantir/foundry-platform-reference.md) — platform capabilities and constraints
- [ADR-002: Batch + Streaming](./adr-002-batch-plus-streaming.md) — scoring architecture also Foundry-native
