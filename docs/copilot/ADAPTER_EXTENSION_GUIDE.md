# Adapter Extension Guide

This guide explains how to add new external endpoints (for example GraphQL API and JDBC) on top of MetricFlow without modifying existing MetricFlow core packages.

For packaging and remote deployment guidance for scalable GraphQL services, see `docs/copilot/ADAPTER_PACKAGING_AND_DEPLOYMENT.md`.

## Goal

Add integration surfaces while preserving package boundaries and avoiding changes to:

- metricflow/
- metricflow_semantics/
- metricflow_semantic_interfaces/
- dbt-metricflow/

## Core Rule

Treat MetricFlow as a library and build a thin adapter layer around the existing engine and request objects.

Existing entry points to wrap:

- Engine: metricflow/engine/metricflow_engine.py
- Request model: MetricFlowQueryRequest
- SQL client abstraction: metricflow/protocols/sql_client.py
- Existing example integration path: dbt-metricflow/dbt_metricflow/cli/

## Recommended Layout For New Adapters

Use a separate package or repository so upgrades to MetricFlow are simpler.

Example structure:

```text
metricflow-adapters/
  pyproject.toml
  adapters/
    app_context.py
    request_mapping.py
    response_mapping.py
  graphql_api/
    schema.py
    resolvers.py
    server.py
  jdbc_endpoint/
    server.py
    sql_to_metricflow.py
    metadata_provider.py
  tests/
    test_graphql_contract.py
    test_jdbc_contract.py
```

## Shared Adapter Context

Create one initialization module that builds and caches:

1. SemanticManifestLookup
2. SqlClient implementation
3. MetricFlowEngine

All endpoints (GraphQL, JDBC, future MCP) should call this shared context.

## GraphQL Adapter Pattern

### Schema shape

Define GraphQL types for:

- QueryRequestInput: metrics, groupBy, where, orderBy, startTime, endTime, limit, savedQuery
- QueryResult: sql, columns, rows, rowCount
- ExplainResult: sql, planText
- Metadata queries: listMetrics, listDimensions, listEntities, listSavedQueries

### Resolver mapping

Map GraphQL input to MetricFlowQueryRequest.create(...), then call:

- engine.query(...) for data requests
- engine.explain(...) for explain requests
- engine.list_* methods for metadata

### Resolver constraints

- Validate user input before constructing MetricFlowQueryRequest.
- Keep adapter-specific auth and rate limits in GraphQL middleware.
- Do not add endpoint behavior in MetricFlow core.

## JDBC Endpoint Pattern

JDBC requires a wire protocol endpoint and metadata responses.

### Practical approach

Build a standalone JDBC server layer that:

1. Accepts JDBC client SQL
2. Translates supported SQL patterns into MetricFlowQueryRequest
3. Executes with MetricFlowEngine
4. Returns tabular rows and JDBC metadata

### Recommended SQL surface (initial)

Start with a constrained SQL subset to reduce complexity:

- SELECT with projected fields
- WHERE filters
- ORDER BY
- LIMIT
- Optional saved query invocation pattern

Avoid full ANSI SQL coverage in v1. Add support incrementally.

### Translation module

Implement translation in a dedicated module such as jdbc_endpoint/sql_to_metricflow.py.

Keep rules explicit:

- Metric tokens map to metric_names
- Dimension/entity tokens map to group_by_names
- Time filters map to time_constraint_start and time_constraint_end
- Other predicates map to where_constraints

### JDBC metadata

Expose database/schema/table/column metadata from semantic manifest and engine list methods.

## Feature Extension Without Core Changes

When adding new endpoint features, extend only adapter modules:

- Input validation
- AuthN/AuthZ
- Query caching
- Usage quotas
- Timeout/retry policy
- Response shaping
- Endpoint-specific observability

If a new behavior appears reusable across adapters, add it to the shared adapter package, not MetricFlow core.

## Versioning Strategy

Pin adapter package compatibility against MetricFlow versions.

Example:

- metricflow-adapters 0.1.x supports metricflow >=X.Y,<X.(Y+1)

## Testing Strategy

Minimum coverage:

1. Contract tests: endpoint input to MetricFlow request mapping.
2. Result tests: engine output to endpoint output mapping.
3. Error tests: invalid inputs and engine exceptions.
4. Compatibility tests: run against supported engines (DuckDB plus target warehouses).

For SQL-generation-impacting behavior, validate snapshots in MetricFlow test style before releasing adapter changes.

## Security And Governance

Keep these concerns in adapter layer:

- Authentication
- Authorization per metric/dimension/saved-query
- Row-level and column-level restrictions
- Audit logging per request

Do not embed endpoint auth concerns in MetricFlow core packages.

## Rollout Plan

1. Build GraphQL adapter first (lower protocol complexity).
2. Reuse the same shared adapter context.
3. Add JDBC endpoint with constrained SQL subset.
4. Expand SQL support and metadata coverage by observed client demand.
5. Add MCP endpoint later using the same mapping primitives.

## Definition Of Done For New Adapter

- No code changes required in existing MetricFlow core packages.
- Adapter has explicit request and response mapping layers.
- Adapter has version-compatibility policy.
- Adapter includes contract and integration tests.
- Adapter docs describe supported features and known limitations.
