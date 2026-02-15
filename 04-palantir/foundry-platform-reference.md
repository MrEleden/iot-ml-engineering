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

## Security & Governance
Project-based access control, encryption at rest/transit, audit logging, row/column-level security, data lineage tracking, data retention policies.
