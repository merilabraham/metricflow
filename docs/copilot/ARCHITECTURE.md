# MetricFlow Architecture Reference for Copilot

Use this document as the first stop to understand where functionality lives before reviewing or changing code.

## Repository Layers

MetricFlow is intentionally split into three main Python packages with strict boundaries:

| Package | Primary Role | Typical Contents |
|---|---|---|
| `metricflow_semantic_interfaces/` | OSI semantic schema contracts | Shared dataclass-like objects, parsing, references, validations |
| `metricflow_semantics/` | Semantic reasoning and query intent | Spec resolution, query-level semantics, naming, filters, graph logic |
| `metricflow/` | Execution and SQL compilation | Dataflow planning, plan conversion, SQL rendering, execution wiring |

Boundary guidance:
- Keep `metricflow_semantic_interfaces/` execution-agnostic and engine-agnostic.
- Keep `metricflow_semantics/` focused on semantic logic, not warehouse execution.
- Keep `metricflow/` focused on planning/rendering/execution concerns.

## Feature Lookup Index

When asked to find or update a feature, start with this index and then follow usages/tests.

### Public API and protocols
- Query/public interfaces: `metricflow_semantics/api/`, `metricflow/protocols/`
- Semantic interfaces/protocols: `metricflow_semantic_interfaces/protocols/`

## Semantic Layer Interfaces

Use this section when you need to find the interface contracts that define semantic-layer inputs and outputs.

### Query parameter interfaces (`metricflow_semantics/protocols/query_parameter.py`)

These protocols define the typed query parameters used by semantic query parsing and resolver input creation:

- `MetricQueryParameter`
- `DimensionOrEntityQueryParameter`
- `TimeDimensionQueryParameter`
- `OrderByQueryParameter`
- `SavedQueryParameter`

Type aliases in the same module:

- `GroupByQueryParameter`
- `InputOrderByQueryParameter`

### Semantic manifest contracts (`metricflow_semantic_interfaces/protocols/`)

The semantic layer depends on these protocol contracts for manifest structure and semantic definitions:

- Core manifest/model: `SemanticManifest`, `SemanticModel`, `SemanticModelDefaults`
- Metrics/measures: `Metric`, `MetricInput`, `MetricTypeParams`, `Measure`, `MeasureAggregationParameters`
- Dimensions/entities: `Dimension`, `DimensionTypeParams`, `Entity`
- Saved queries and filters: `SavedQuery`, `WhereFilter`, `WhereFilterIntersection`
- Time spine contracts: `TimeSpine`, `TimeSpinePrimaryColumn`

Reference import hub:

- `metricflow_semantic_interfaces/protocols/__init__.py`

### Semantic API entry points (`metricflow_semantics/api/v0_1/`)

Current v0.1 API-facing logic includes:

- `SavedQueryDependencyResolver`
- `SavedQueryDependencySet`

Module:

- `metricflow_semantics/api/v0_1/saved_query_dependency_resolver.py`

## Connectivity Surfaces

MetricFlow in this repository is currently exposed through CLI and Python interfaces.

### Supported today

- CLI surface (`mf` / dbt-metricflow CLI):
  - `dbt-metricflow/dbt_metricflow/cli/main.py`
  - Query command entry point: `query()`

- Python library/programmatic surface:
  - `metricflow/engine/metricflow_engine.py`
  - Request/result and engine types: `MetricFlowQueryRequest`, `MetricFlowQueryResult`, `MetricFlowEngine`

- SQL execution abstraction:
  - `metricflow/protocols/sql_client.py` (`SqlClient` protocol)
  - `dbt-metricflow/dbt_metricflow/cli/dbt_connectors/adapter_backed_client.py` (`AdapterBackedSqlClient`)

### Not implemented in this repo (as of 2026-05-27)

- REST API server endpoints
- GraphQL schema/resolvers
- MCP server for semantic-layer querying

If needed, REST/GraphQL/MCP can be added as an adapter layer on top of the existing Python engine request/response surface, while keeping package boundaries intact.

For implementation guidance, see `docs/copilot/ADAPTER_EXTENSION_GUIDE.md`.

### Metric and semantic model behavior
- Semantic graph/model/specs: `metricflow_semantics/semantic_graph/`, `metricflow_semantics/model/`, `metricflow_semantics/specs/`
- Instances/entities/dimensions: `metricflow_semantics/instances.py`, `metricflow_semantics/naming/`
- Validation rules: `metricflow_semantic_interfaces/validations/`, `metricflow/validation/`

### Dataflow planning and SQL generation
- Dataflow nodes and plan IR: `metricflow/dataflow/`
- Plan conversion: `metricflow/plan_conversion/`
- SQL expression/rendering layers: `metricflow/sql/`, `metricflow_semantics/sql/`
- Engine/execution glue: `metricflow/engine/`, `metricflow/execution/`

### Query behavior and metric evaluation
- Metric evaluation runtime: `metricflow/metric_evaluation/`
- Query request structures: `metricflow/sql_request/`, `metricflow/dataset/`, `metricflow/data_table/`

### CLI surface
- dbt MetricFlow CLI and flags: `dbt-metricflow/dbt_metricflow/`

## Test Location Map

Use tests in the same package boundary as the changed code:

- Changes in `metricflow/` -> `tests_metricflow/`
- Changes in `metricflow_semantics/` -> `tests_metricflow_semantics/`
- Changes in `metricflow_semantic_interfaces/` -> `tests_metricflow_semantic_interfaces/`
- Changes in CLI (`dbt-metricflow/`) -> `dbt-metricflow/tests_dbt_metricflow/`

If SQL output changes, inspect/update snapshots under `tests_metricflow/snapshots/` as appropriate.

## Copilot Workflow for Feature Discovery

For any request that asks where a feature is implemented, where to add logic, or how architecture is organized:

1. Read this file first.
2. Identify the likely package layer using the index above.
3. Inspect matching tests in the mapped test directory.
4. Follow symbol usages to confirm the final implementation path.
5. Call out cross-package boundary risks when proposing or reviewing changes.
