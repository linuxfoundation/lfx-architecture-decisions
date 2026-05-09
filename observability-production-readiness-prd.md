# PRD: Production Telemetry and Observability Readiness

## Summary

LFX Self Serve has meaningful observability already implemented across the main UI and supporting services, but production readiness is uneven. `lfx-v2-ui` has the most complete implementation with OpenTelemetry tracing, structured Pino logging, Datadog RUM, backend service-layer logging, custom NATS and Snowflake spans, and Kubernetes health probes. `lfx-changelog` has Datadog tracing, structured logging, and an application health endpoint, but lacks chart-level probes and standardized liveness/readiness behavior.

This PRD defines what exists, what is missing, and what is required for ops to confidently monitor, debug, alert, and support these services in production.

## Goals

- Provide production-grade telemetry for LFX services used by ops, support, and engineering.
- Ensure every production service exposes reliable health signals for deployment and incident response.
- Correlate frontend sessions, backend traces, and structured logs across user requests.
- Standardize observability behavior across `lfx-v2-ui` and `lfx-changelog`.
- Make telemetry configurable through release values and secrets without code changes.

## Non-Goals

- Replacing Datadog as the production observability backend.
- Building a custom metrics platform.
- Assessing applications outside `lfx-v2-ui` and `lfx-changelog`.
- Adding product analytics events unrelated to operational health.

## Current State

### Architecture Standards

The architecture decision records define the intended baseline:

- Structured JSON logs are required, with one valid JSON log line per event.
- OpenTelemetry tracing is required for new projects.
- Logs should include trace and span IDs for correlation.
- Production and staging traces/metrics should route to Datadog through an OpenTelemetry Collector or Datadog agent.

Relevant files:

- `lfx-architecture-decisions/decisions/0002-structured-json-logging.md`
- `lfx-architecture-decisions/decisions/0003-opentelemetry-instrumentation.md`

### `lfx-v2-ui`

Implemented:

- Server-side OpenTelemetry bootstrap exists in `lfx-v2-ui/apps/lfx-one/otel.mjs`.
- Tracing is started before the SSR server through PM2 `node_args: '--import ./otel.mjs'` in `lfx-v2-ui/apps/lfx-one/ecosystem.config.js`.
- HTTP, Express, and Undici instrumentation are enabled.
- Health endpoints are excluded from tracing noise.
- Trace sampling is configurable with `OTEL_TRACES_SAMPLER` and `OTEL_TRACES_SAMPLER_ARG`.
- Structured Pino logging is implemented in `lfx-v2-ui/apps/lfx-one/src/server/server-logger.ts`.
- Logs include OpenTelemetry `trace_id` and `span_id` when an active span exists.
- Sensitive request/response fields are filtered or redacted.
- The backend service layer uses a shared `LoggerService` in `lfx-v2-ui/apps/lfx-one/src/server/services/logger.service.ts`; 55 service/controller files call `logger.*` for operation, warning, error, and duration-oriented logs.
- Custom OpenTelemetry client spans exist for NATS requests in `lfx-v2-ui/apps/lfx-one/src/server/services/nats.service.ts`.
- Custom OpenTelemetry client spans exist for Snowflake query execution in `lfx-v2-ui/apps/lfx-one/src/server/services/snowflake.service.ts`, including DB semantic attributes and returned row counts.
- The backend service layer routes many LFX API and Query Service calls through `MicroserviceProxyService` and `ApiClientService`, which rely on `fetch`/Undici auto-instrumentation for outbound HTTP spans.
- Browser Datadog RUM is implemented in `lfx-v2-ui/apps/lfx-one/src/app/shared/providers/datadog-rum.provider.ts`.
- Runtime config injects Datadog RUM client/application IDs and allowed tracing URLs.
- `/livez` and `/readyz` endpoints exist in `lfx-v2-ui/apps/lfx-one/src/server/server.ts`.
- Helm chart supports startup, liveness, and readiness probes in `lfx-v2-ui/charts/lfx-v2-ui/values.yaml`.

Known gaps:

- `OTEL_EXPORTER_OTLP_ENDPOINT` is not set in default Helm values, so backend tracing is disabled unless production release values inject it.
- No chart-level default exists for `OTEL_SERVICE_NAME`, `OTEL_TRACES_SAMPLER`, `OTEL_TRACES_SAMPLER_ARG`, or deployment environment labels.
- Application metrics are not explicitly exported from the app. The current approach relies mostly on infrastructure metrics and traces.
- The shared microservice proxy/API client does not add explicit domain-level span names or attributes for LFX API and Query Service calls. Auto-instrumentation should capture HTTP spans, but ops may still see low-semantic route/host spans rather than business operations like `query.resources`, `committee.create`, or `meeting.update`.
- NATS and Snowflake have custom spans, but there is no comparable explicit span convention for Auth0/CDP, Copilot/AI proxy, Credly, TI, Rewards, or other direct `fetch`-based service clients.
- Service-layer logs provide durations, warnings, and errors, but no custom metrics counters/histograms are emitted for downstream dependency failures, retries, or saturation.
- Readiness intentionally does not check lazy dependencies such as NATS or Snowflake. This is valid for SSR availability, but ops still needs separate dependency-health visibility.
- There is no confirmed dashboard or alert definition in the repo.

### `lfx-changelog`

Implemented:

- Datadog tracing initializes in production in `lfx-changelog/apps/lfx-changelog/src/server/setup/tracer.ts`.
- Runtime metrics are enabled through `dd-trace`.
- Pino HTTP logging is configured in `lfx-changelog/apps/lfx-changelog/src/server/setup/logger.ts`.
- Health endpoint exists at `/health` in `lfx-changelog/apps/lfx-changelog/src/server/setup/routes.ts`.
- `/health` checks OpenSearch status when `OPENSEARCH_URL` is configured.
- Datadog RUM dependencies and runtime configuration fields exist.

Known gaps:

- Helm deployment does not wire startup, liveness, or readiness probes.
- Tracing uses Datadog-specific instrumentation rather than the OpenTelemetry standard required for new projects.
- Logs do not explicitly add OpenTelemetry-style `trace_id` and `span_id`; correlation depends on Datadog log injection behavior.
- `/health` combines process health with OpenSearch status but there is no separate `/livez` and `/readyz`.
- No visible dashboard, monitor, SLO, or alert configuration exists in the repo.

## User Personas

### Ops Engineer

Needs to quickly answer:

- Is the service alive?
- Is the service ready to receive production traffic?
- Which dependency is causing degraded behavior?
- Are error rates, latency, or restarts above threshold?
- Can a failing user request be traced across frontend, SSR, backend, and downstream APIs?

### Support Engineer

Needs to quickly answer:

- Did a user session encounter browser errors or failed API calls?
- What backend trace corresponds to a browser action?
- Was the failure auth, dependency, validation, or service-side?

### Developer

Needs to quickly answer:

- Which route, controller, or downstream call is slow or failing?
- What logs belong to the same trace?
- Did a deploy introduce a regression in latency, errors, or dependency failures?

## Requirements

### R1: Standard Service Health Endpoints

Every production service must expose:

- `/livez`: returns 200 when the process is alive.
- `/readyz`: returns 200 only when the service can safely receive traffic.
- `/health`: optional aggregate diagnostic endpoint for humans and smoke tests.

Acceptance criteria:

- `lfx-v2-ui` keeps `/livez` and `/readyz`.
- `lfx-changelog` adds `/livez` and `/readyz`, while preserving `/health`.
- Health endpoints are unauthenticated.
- Health endpoints are excluded from noisy request logs and traces.

### R2: Kubernetes Probe Wiring

Every Helm/Kubernetes deployment must configure:

- `startupProbe` for slow-start services.
- `livenessProbe` against `/livez`.
- `readinessProbe` against `/readyz`.

Acceptance criteria:

- `lfx-v2-ui` existing probes are verified in production release values.
- `lfx-changelog` Helm chart adds configurable startup, liveness, and readiness probes.
- Probe paths, periods, timeouts, and failure thresholds are configurable per environment.

### R3: Distributed Tracing

Every production service must emit traces for:

- Inbound HTTP requests.
- Outbound HTTP requests.
- Key service dependencies, including NATS, Snowflake, OpenSearch, Postgres, and external APIs where applicable.
- High-value custom operations where auto-instrumentation is insufficient.

Acceptance criteria:

- `lfx-v2-ui` production values set `OTEL_EXPORTER_OTLP_ENDPOINT`.
- `lfx-v2-ui` production values set service name, environment, version, and sampler.
- `lfx-v2-ui` verifies existing custom NATS and Snowflake spans in Datadog.
- `lfx-v2-ui` adds explicit service-layer spans or span attributes for shared `MicroserviceProxyService` / `ApiClientService` calls so Query Service and LFX API dependencies are distinguishable by operation, service, route template, status, and error code.
- `lfx-v2-ui` reviews direct `fetch` clients, including Auth0/CDP, Copilot/AI proxy, Credly, TI, and Rewards, and adds explicit spans where auto-instrumentation lacks useful business context.
- `lfx-changelog` either migrates to OpenTelemetry or documents a Datadog-only exception.
- Traces reach Datadog in staging and production.
- Trace sampling is configurable without code changes.

### R4: Trace and Log Correlation

Every backend log line emitted during a traced request must be correlatable to the trace.

Acceptance criteria:

- Logs include `trace_id` and `span_id` where OpenTelemetry is used.
- Datadog-formatted IDs are included where Datadog correlation requires them.
- `lfx-v2-ui` validates that log fields appear under production traffic.
- `lfx-changelog` validates Datadog log injection or explicitly adds trace fields.
- Sensitive headers, cookies, tokens, and secrets remain redacted.

### R5: Frontend Real User Monitoring

Frontend apps must capture:

- Page views.
- Browser errors.
- Long tasks.
- Resource failures.
- API call tracing to allowed backend origins.
- User/session context after authentication.

Acceptance criteria:

- `lfx-v2-ui` Datadog RUM is configured through runtime env, not hardcoded secrets.
- RUM allowed tracing URLs match production frontend/backend origins.
- Session replay privacy level is reviewed and approved for production.

### R6: Operational Metrics

Ops must be able to monitor:

- Request rate.
- Error rate.
- Latency percentiles.
- Saturation: CPU, memory, restarts, pod availability.
- Dependency health: NATS, Snowflake, OpenSearch, Postgres, external APIs.
- Frontend error rate and failed resource/API calls.

Acceptance criteria:

- Infrastructure metrics are available for every deployment.
- Datadog APM service metrics are available for traced services.
- Dependency-specific metrics are represented by traces, health checks, or custom metrics.
- Any custom app metric uses OpenTelemetry metrics unless an exception is documented.

### R7: Dashboards

Ops must have dashboards for each production service.

Minimum dashboard widgets:

- Service availability.
- Request throughput.
- Error rate.
- p50/p95/p99 latency.
- Top failing routes.
- Top slow routes.
- Pod restarts.
- CPU and memory.
- Dependency failures.
- Frontend RUM errors and failed API calls.

Acceptance criteria:

- Dashboards exist for `lfx-v2-ui` and `lfx-changelog`.
- Dashboard links are documented in the deployment runbook.
- Dashboards show environment and version tags.

### R8: Alerts and SLOs

Ops must have actionable alerts.

Minimum alerts:

- Service unavailable or readiness failing.
- 5xx error rate above threshold.
- p95 latency above threshold.
- Pod restart loop.
- Memory nearing limit or OOMKilled.
- Dependency unavailable.
- Frontend error spike.
- Trace ingestion disabled or missing after deploy.

Acceptance criteria:

- Each alert has a severity, owner, route, and runbook link.
- Alerts are tested in staging before production rollout.
- SLO targets are agreed with product and ops.

### R9: Runbooks

Each service must have a production observability runbook.

Runbook must include:

- Health endpoint URLs.
- Dashboard links.
- Alert definitions.
- How to find logs by trace ID.
- How to find traces from a RUM session.
- Known dependency failure modes.
- Rollback or mitigation steps.

Acceptance criteria:

- Runbooks exist for every production service.
- Runbooks are linked from alerts.
- Runbooks are reviewed by ops.

## Proposed Implementation Plan

### Phase 1: Production Config Verification

Scope:

- Confirm production/staging release values for `lfx-v2-ui`.
- Add missing OTEL env vars.
- Confirm Datadog RUM env vars.
- Confirm traces reach Datadog.

Deliverables:

- Release values include `OTEL_EXPORTER_OTLP_ENDPOINT`.
- Release values include service name, environment, version, and sampler.
- Datadog dashboard shows `lfx-v2-ui` traces.
- Logs can be filtered by trace ID.

### Phase 2: Changelog Probe and Health Standardization

Scope:

- Add `/livez` and `/readyz` to `lfx-changelog`.
- Preserve `/health` as aggregate diagnostics.
- Add Helm startup/liveness/readiness probes.

Deliverables:

- Chart values mirror the `lfx-v2-ui` probe pattern.
- `/readyz` has clear dependency semantics.
- Health endpoints are excluded from noisy logging.

### Phase 3: Trace-Log Correlation Fixes

Scope:

- Validate log correlation for `lfx-v2-ui`.
- Validate backend service-layer spans for existing NATS and Snowflake integrations.
- Add or standardize explicit service-layer spans for `MicroserviceProxyService` / `ApiClientService` so Query Service and LFX API calls carry useful operation names and attributes beyond raw HTTP auto-instrumentation.
- Review direct backend `fetch` clients and add explicit spans for high-value dependencies where needed.
- Enable or replace log injection for `lfx-changelog`.

Deliverables:

- A production request can be followed from RUM to trace to logs.
- A slow or failing dashboard/API request can be broken down by SSR route, service-layer operation, downstream dependency, and dependency status/error.
- Sensitive data remains redacted.
- Correlation fields are documented.

### Phase 4: Dashboards, Alerts, and Runbooks

Scope:

- Create Datadog dashboards.
- Create monitors.
- Add runbooks.
- Validate staging alert behavior.

Deliverables:

- Ops-approved dashboard set.
- Alert routing and escalation configured.
- Runbooks linked from alert messages.

## Open Questions

- What is the production Datadog or OTLP endpoint for Kubernetes workloads?
- Is the preferred production path OTEL Collector sidecar, cluster collector, or Datadog agent?
- Should `lfx-changelog` migrate from `dd-trace` to OpenTelemetry now, or is a Datadog-only exception acceptable?
- What SLO targets should apply to each service?
- Who owns dashboard and monitor creation: app teams, platform, or ops?
- What session replay privacy level is approved for production?

## Risks

- Tracing may appear implemented but remain disabled if release values omit `OTEL_EXPORTER_OTLP_ENDPOINT`.
- Health endpoints that do not check dependencies can show green while feature-specific paths fail.
- Overly aggressive readiness dependency checks can remove otherwise healthy SSR pods from service.
- Session replay and user context require privacy review.
- 100% sampling may increase Datadog cost for high-traffic routes.

## Success Metrics

- 100% of production services have liveness and readiness probes.
- 100% of production backend services emit traces to Datadog.
- 100% of traced backend logs are correlatable by trace ID.
- Ops can identify the cause of a synthetic dependency failure within 10 minutes.
- Frontend RUM sessions can be connected to backend traces for supported origins.
- All critical alerts link to runbooks.

## Production Readiness Checklist

- [ ] `lfx-v2-ui` production values set `OTEL_EXPORTER_OTLP_ENDPOINT`.
- [ ] `lfx-v2-ui` production values set service name, environment, version, and sampler.
- [ ] `lfx-v2-ui` Datadog RUM runtime config is populated in production.
- [ ] `lfx-v2-ui` traces and logs correlate in Datadog.
- [ ] `lfx-v2-ui` NATS and Snowflake service-layer spans are visible in Datadog.
- [ ] `lfx-v2-ui` Query Service and LFX API calls through `MicroserviceProxyService` / `ApiClientService` have useful service-layer operation names or attributes.
- [ ] `lfx-v2-ui` direct backend `fetch` clients are reviewed for explicit spans where auto-instrumentation is not enough.
- [ ] `lfx-changelog` adds `/livez` and `/readyz`.
- [ ] `lfx-changelog` Helm chart adds startup/liveness/readiness probes.
- [ ] `lfx-changelog` trace-log correlation is verified.
- [ ] Dashboards exist for all production services.
- [ ] Alerts exist for availability, error rate, latency, restarts, and dependency failure.
- [ ] Runbooks are linked from alerts.
- [ ] Session replay privacy is reviewed and approved.
- [ ] Sampling and retention settings are approved for cost control.

## Recommendation

Treat `lfx-v2-ui` as the reference implementation, but do not mark the overall platform production-ready for ops until release configuration and operational artifacts are complete. The highest-priority fixes are:

1. Enable and verify `lfx-v2-ui` OTLP export in production.
2. Add Kubernetes probes for `lfx-changelog`.
3. Verify trace-log correlation across all services.
4. Create dashboards, alerts, and runbooks.
