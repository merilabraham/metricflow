# Adapter Packaging and Deployment Guide

This guide provides implementation instructions for packaging and deploying a GraphQL adapter service on top of MetricFlow so it can be consumed by many clients and scaled safely.

Use this document when implementing production deployment work in a separate adapter repository.

## Scope

- Package the adapter as a Python artifact and container image.
- Deploy a stateless GraphQL service for horizontal scaling.
- Add security, caching, observability, and CI/CD controls.
- Keep all endpoint logic outside MetricFlow core packages.

## Non-Goals

- Do not modify code under metricflow/, metricflow_semantics/, metricflow_semantic_interfaces/, or dbt-metricflow/.
- Do not implement full SQL/JDBC protocol coverage in the first deployment phase.

## Target Runtime Architecture

Build a stateless GraphQL service with this shape:

1. Client calls /graphql over HTTPS.
2. GraphQL resolver maps input to MetricFlowQueryRequest.
3. Resolver executes MetricFlowEngine query/explain/list operations.
4. Service returns tabular results and metadata.
5. Redis caches metadata and selected query results.
6. Service runs behind a load balancer with N replicas.

## Packaging Instructions

## Python Package

1. Use pyproject.toml and build with python -m build.
2. Publish wheel/sdist to an internal package registry.
3. Pin MetricFlow compatibility in dependency constraints.

Recommended dependency policy:

- metricflow >=X.Y,<X.(Y+1)
- Lock transitive dependencies for reproducible builds.

## Container Image

Use a multi-stage Docker build:

1. Builder stage installs dependencies and builds wheel.
2. Runtime stage is slim, non-root, and only installs runtime artifacts.
3. Entrypoint runs ASGI server for graphql_api/server.py.

Container requirements:

- Run as non-root user.
- Read-only filesystem where feasible.
- Expose one HTTP port.
- Health endpoints: /healthz and /readyz.

## GraphQL Service Requirements

## API Surface

Required operations:

- queryMetrics(input)
- explainMetrics(input)
- listMetrics
- listDimensions(metrics)
- listEntities(metrics)
- listSavedQueries

## Resolver Rules

1. Validate input before request mapping.
2. Map to MetricFlowQueryRequest.create(...).
3. Execute via shared app context (engine + sql client + semantic lookup).
4. Return normalized response shape.
5. Convert internal exceptions to stable API errors.

## Statelessness Rules

- No per-request state in process memory beyond request lifecycle.
- No sticky-session requirement.
- Externalize shared state to Redis/postgres if required.

## Scaling and Reliability

## Horizontal Scaling

Deploy multiple replicas and autoscale by:

- CPU
- p95 latency
- request concurrency

## Backpressure and Limits

Enforce:

- per-tenant rate limits
- max concurrency per pod
- query timeout
- max response rows limit

## Caching Strategy

Use layered caching:

1. L1 in-memory cache for short-lived metadata calls.
2. L2 Redis cache for deterministic query responses.

Cache key components:

- tenant id
- normalized request payload
- semantic manifest/version hash
- adapter version

## Security Requirements

## Authentication and Authorization

1. Authenticate at gateway or middleware (OIDC/API key).
2. Authorize by metric/dimension/saved-query access policy.
3. Deny unauthorized fields before engine execution.

## GraphQL Hardening

- query depth limit
- query complexity limit
- persisted queries (preferred for BI clients)
- disable introspection in production where appropriate

## Secrets Management

- Use managed secrets store.
- Do not bake credentials into images.
- Rotate credentials with zero-downtime rollout.

## Deployment Targets

Supported target platforms:

- Kubernetes
- ECS/Fargate
- Cloud Run

Preferred baseline: Kubernetes.

## Kubernetes Baseline

Deploy these resources:

1. Deployment (2+ replicas)
2. Service
3. Ingress or API gateway integration
4. HorizontalPodAutoscaler
5. PodDisruptionBudget
6. ConfigMap and Secret references

Readiness/liveness behavior:

- /healthz confirms process up.
- /readyz confirms dependencies available (warehouse adapter, manifest load, cache reachability).

## Observability Requirements

## Logs

Emit structured logs with:

- request_id
- tenant_id
- operation_name
- query_hash
- status_code
- latency_ms

## Metrics

Track at minimum:

- request_count
- error_count
- p50/p95/p99 latency
- cache_hit_ratio
- backend_query_duration
- timeout_count

## Tracing

Use OpenTelemetry for distributed tracing across gateway, service, and warehouse calls.

## CI/CD Pipeline

## Pull Request Pipeline

Required checks:

1. Lint and type checks
2. Unit tests
3. Contract tests (GraphQL input/output mapping)
4. Integration tests against adapter context
5. Security scanning (deps and container)

## Release Pipeline

1. Build and tag image from git tag.
2. Push image and package artifacts.
3. Deploy to staging.
4. Run smoke and load tests.
5. Progressive rollout to production (canary or blue/green).

## Rollback

- Keep immutable image tags.
- Roll back by deployment revision/image tag.
- Keep schema changes backward-compatible to simplify rollback.

## Copilot Implementation Backlog

When using Copilot in the adapter repo, implement in this order:

1. Create adapters/app_context.py with engine bootstrap.
2. Create GraphQL schema and resolvers with request mapping.
3. Add server startup with health endpoints.
4. Add auth middleware and query guardrails.
5. Add Redis caching abstraction.
6. Add observability hooks.
7. Add Dockerfile and runtime config.
8. Add Helm chart or platform manifests.
9. Add CI workflows and release pipeline.

## Copilot Prompt Contract

Use this instruction block in implementation prompts:

1. Do not modify MetricFlow core packages.
2. Implement only in adapter service modules.
3. Use MetricFlowQueryRequest.create(...) for request construction.
4. Add tests for every mapping/resolver change.
5. Keep API responses backward-compatible unless explicitly versioned.

## Definition of Done

Deployment is ready when all are true:

1. Service is stateless and horizontally scalable.
2. Security controls and rate limits are active.
3. p95 latency and error budget SLOs are monitored.
4. CI/CD supports canary and rollback.
5. Documentation includes runbook and known limits.
