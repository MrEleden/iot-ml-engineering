# Foundry Platform Reference

Source: [Palantir Foundry Platform Summary](https://www.palantir.com/docs/foundry/getting-started/foundry-platform-summary-llm)

## Architecture
Foundry has two primary layers:
- **Data layer** — raw datasets and processing
- **Ontology layer** — semantic mapping to real-world concepts

## Data Layer

### Datasets
Wrappers around file collections. Support structured (tabular), unstructured (images, videos, PDFs), and semi-structured (JSON, XML) data. Versioned through transactions.

### Transforms
Code blocks that process input datasets into outputs. Written in Python, SQL, or Java through Code Repositories or Code Workspaces. Python transforms support PySpark DataFrames or lightweight engines (Pandas, Polars, DuckDB).

### Pipeline Builder
Point-and-click pipeline construction without coding. Supports batch and streaming workflows.

### Connectors
Pre-built integrations for external data sources (databases, APIs, cloud storage, enterprise systems).

## Ontology Layer

- **Object Types**: Schema definitions for real-world entities with properties, primary keys, and backing datasources
- **Link Types**: Relationship definitions between object types with cardinality specifications
- **Properties**: Attributes with types (string, integer, date, array, geoshape, etc.)
- **Action Types**: User-triggered modifications with parameters, rules, and side effects
- **Interfaces**: Abstract types enabling polymorphic workflows across multiple object types

## Processing Modes

- **Batch**: fully recomputes datasets on each run
- **Incremental**: handles only new/changed data for efficiency
- **Streaming**: near-real-time processing via Flink

## Functions
Server-side business logic in TypeScript and Python. Supports Ontology queries, aggregations, external API calls, and Ontology edits.

## Applications
- **Workshop**: low-code app builder with 60+ widgets
- **Slate**: custom HTML/CSS/JavaScript apps
- **OSDK**: type-safe SDKs (TypeScript, Python, Java, OpenAPI) for external apps

## AI Platform (AIP)
- **AIP Agent Studio**: interactive assistants with retrieval context
- **AIP Logic**: no-code LLM-powered functions
- **AIP Assist**: in-platform navigation

## Model Framework

Foundry provides a structured model lifecycle separate from the general Transform pipeline.

### Model Objectives
A Model Objective defines the problem a model solves (e.g., "anomaly detection on refrigeration devices"). It groups model versions, evaluation results, and promotion decisions under a single logical unit. Each objective specifies the expected input/output schema and the metrics used for comparison.

### Model Adapters
A Model Adapter is a Python class that wraps any ML model (scikit-learn, PyTorch, custom) and exposes a standard `predict()` interface. The adapter defines the input/output schema so Foundry can validate data contracts at scoring time. All model inference — batch or interactive — flows through the adapter.

### Model Publishing
Model artifacts are published from Git-backed Code Repositories through a staging → release promotion flow:
1. **Draft**: training code runs on a feature branch, producing a draft artifact
2. **Staging**: merging to the staging branch publishes the artifact to the staging model store, where evaluation runs
3. **Release**: promotion to the release branch makes the artifact available to production scoring Transforms

### Model Versioning
Each published model version is tied to three coordinates:
- **Training data transaction**: the specific dataset transaction (snapshot) used for training, ensuring reproducibility
- **Code commit**: the Git SHA in the Code Repository that produced the artifact
- **Dependency versions**: pinned Python package versions from `meta.yml`

This triple guarantees that any model version can be reproduced and audited.

### Model Evaluation
Model Objectives support custom evaluation metrics. For supervised models, Foundry computes standard metrics (accuracy, AUC, F1) against a labeled evaluation dataset. For unsupervised models, custom evaluation Transforms compute proxy metrics (score distribution stability, contamination rate consistency) and write them to a metrics dataset attached to the objective.

## Monitoring and Observability

Foundry provides several mechanisms for monitoring data pipelines and ensuring production reliability.

### Dataset Expectations
Declarative checks attached to Transform outputs. Expectations validate row counts, null rates, value ranges, and schema conformance after each Transform run. A failed expectation marks the output transaction as unhealthy and triggers alerts without blocking downstream consumers (configurable to block if desired).

### Pipeline Health Checks
SLA monitoring on critical datasets: define a maximum acceptable age for a dataset's latest transaction (e.g., "the `daily_device_health_scores` dataset must have a transaction newer than 6 hours"). If the SLA is violated — because an upstream Transform failed, a schedule was missed, or a build timed out — Foundry triggers an alert to the configured notification channel.

### Data Lineage
Foundry maintains a full provenance graph across all datasets, Transforms, and model artifacts. Every dataset transaction records which input transactions it consumed and which Transform code produced it. This lineage graph supports impact analysis ("which downstream datasets are affected if this source schema changes?"), root cause investigation ("why is this feature dataset stale?"), and audit compliance.

## ML Lifecycle Mapping

Foundry's primitives map directly to ML lifecycle stages:

| ML Lifecycle Stage | Foundry Primitive | Example |
|---|---|---|
| Data engineering | Datasets, Transforms | Raw sensor readings ingested via streaming datasets, cleaned by incremental Transforms |
| Feature engineering | Incremental Transforms, dataset expectations | Rolling aggregations computed incrementally, validated by expectations on feature datasets |
| Model development | Code Repositories, Model Objectives | Training code in Git-backed repos, model artifacts versioned under objectives |
| Deployment | Model Publishing, batch scoring Transforms | Model promoted staging → release, scoring Transform loads published model |
| Monitoring | Dataset Expectations, Pipeline Health Checks | Feature drift detected by expectation failures, SLA alerts on stale scoring outputs |
| Feedback & retraining | Ontology Actions, retraining Transforms | False positive labels from operators feed back into training data, scheduled retraining triggers |

## Security & Governance
Project-based access control, encryption at rest/transit, audit logging, row/column-level security, data lineage tracking, data retention policies.
