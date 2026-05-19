# PRD: Production Telemetry and Observability Readiness

## Summary

LFX Self Serve has meaningful observability already implemented across the main UI and supporting services, but production readiness is uneven. `lfx-v2-ui` has the most complete implementation with OpenTelemetry tracing, structured Pino logging, Datadog RUM, backend service-layer logging, custom NATS and Snowflake spans, and Kubernetes health probes. The Go API services behind `LFX_V2_SERVICE` are part of the production backend service layer and must be included in ops readiness decisions. `lfx-v2-query-service`, `lfx-v2-committee-service`, and `lfx-v2-meeting-service` have OpenTelemetry scaffolding and probes, while `lfx-v2-survey-service` has health/probe/logging support but lacks active tracing setup in the application entrypoint. Jira production verification for `LFXV2-1728` also reports that `lfx-v2-voting-service` and `lfx-v2-survey-service` have zero Datadog APM traces, and that cross-service propagation from RUM/UI into Go APIs is currently broken. `lfx-changelog` has Datadog tracing, structured logging, and an application health endpoint, but lacks chart-level probes and standardized liveness/readiness behavior.

This PRD defines what exists, what is missing, and what is required for ops to confidently monitor, debug, alert, and support these services in production.

## Goals

- Provide production-grade telemetry for LFX services used by ops, support, and engineering.
- Ensure every production service exposes reliable health signals for deployment and incident response.
- Correlate frontend sessions, backend traces, and structured logs across user requests.
- Standardize observability behavior across `lfx-v2-ui`, the Go API service layer, and `lfx-changelog`.
- Make telemetry configurable through release values and secrets without code changes.

## Non-Goals

- Replacing Datadog as the production observability backend.
- Building a custom metrics platform.
- Assessing unrelated Linux Foundation applications outside LFX Self Serve production paths.
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

### Fact-Finding Snapshot

| Service / repo | Role reviewed | Health endpoints and probes | Tracing found in code | Trace-log correlation | Metrics found in code | Dashboards / alerts / runbooks | Production verification needed |
| --- | --- | --- | --- | --- | --- | --- | --- |
| `lfx-v2-ui` | SSR/BFF and frontend | `/livez`, `/readyz`, Helm probes found | OpenTelemetry server tracing, Undici/HTTP/Express instrumentation, custom NATS and Snowflake spans, Datadog RUM | Pino logs include `trace_id` and `span_id` when active span exists | No explicit app metrics found | Not found in repo | Confirm OTLP endpoint, service tags, sampler, RUM allowed origins, dashboards, alerts |
| `lfx-v2-query-service` | Go Query API | `/livez`, `/readyz`, Helm probes found | OpenTelemetry SDK, `otelhttp` inbound handler, outbound HTTP/OpenSearch transport instrumentation | `slog` JSON with `slog-otel` trace/span fields | OTEL metrics exporter supported but disabled by default; no custom business metrics found | Not found in repo | Confirm OTLP export enabled, UI-to-Go trace propagation, OpenSearch/FGA visibility, dashboards, alerts |
| `lfx-v2-committee-service` | Go Committee API | `/livez`, `/readyz`, Helm probes found | OpenTelemetry SDK and `otelhttp` inbound handler | `slog` JSON with `slog-otel` trace/span fields | OTEL metrics exporter supported but disabled by default; no custom business metrics found | Not found in repo | Confirm OTLP export enabled, NATS KV/request/publish visibility, dashboards, alerts |
| `lfx-v2-meeting-service` | Go Meeting/ITX API | `/livez`, `/readyz`, Helm probes found | OpenTelemetry SDK and `otelhttp` inbound handler | `slog` JSON with `slog-otel` trace/span fields | OTEL metrics exporter supported but disabled by default; no custom business metrics found | Not found in repo | Confirm OTLP export enabled, ITX outbound tracing, NATS event processing visibility, dashboards, alerts |
| `lfx-v2-survey-service` | Go Survey API | `/health`, `/livez`, `/readyz`, Helm probes found | OpenTelemetry dependencies and Helm env placeholders found, but no SDK initialization or `otelhttp` server wrapper found in `cmd/survey-api/main.go` | `slog` JSON with `slog-otel` handler present, but correlation depends on traced request context that is currently missing | OTEL dependencies present but no active metrics exporter setup or custom business metrics found | Not found in repo | Add active tracing setup, verify Datadog APM service visibility, add outbound HTTP/NATS spans, dashboards, alerts |
| `lfx-v2-voting-service` | Go Voting API | Pending local repo inspection | Jira production verification reports zero Datadog APM traces | Pending local repo inspection | Pending local repo inspection | Pending local repo inspection | Locate/audit repo, add active tracing if missing, verify Datadog APM service visibility |
| `lfx-changelog` | Changelog API | `/health` found; Helm probes not found | Datadog `dd-trace` found, not OpenTelemetry | Depends on Datadog log injection unless explicit fields are added | Datadog runtime metrics enabled | Not found in repo | Confirm trace-log correlation, split liveness/readiness behavior, probes, dashboards, alerts |

### Production-Verified Requirement Status

This table incorporates the `LFXV2-1728` Jira comment titled "Observability PRD: Requirements Status (as of 2026-05-11)", which was reported as verified against Datadog production data, latest main branches, and `lfx-v2-argocd`.

| Requirement | Status | Production/Jira finding | Related follow-up |
| --- | --- | --- | --- |
| R1: Standard Service Health Endpoints | Mostly done | `lfx-v2-ui` and reviewed Go services expose liveness/readiness endpoints; `lfx-changelog` is missing `/livez` and `/readyz` | `LFXV2-1741` |
| R2: Kubernetes Probe Wiring | Mostly done | Reviewed UI and Go charts have probes; `lfx-changelog` has no Helm probes | `LFXV2-1741` |
| R3: Distributed Tracing | Partial / critical gaps | RUM reaches `lfx-v2-ui`, but Go API spans start new traces instead of continuing UI traces; `lfx-v2-survey-service` and `lfx-v2-voting-service` have zero Datadog APM traces; outbound HTTP and NATS spans are incomplete; intermittent OTLP export failures were reported | `LFXV2-1734`, `LFXV2-1735`, `LFXV2-1736`, `LFXV2-1737`, `LFXV2-1739`, `LFXV2-1742`, `LFXV2-1743`, `LFXV2-1744` |
| R4: Trace and Log Correlation | Mostly done | Verified for `lfx-v2-ui`, `lfx-v2-query-service`, and `lfx-v2-committee-service`; `lfx-changelog` correlation is broken or incomplete | `LFXV2-1738` |
| R5: Frontend Real User Monitoring | Done | Datadog RUM is configured through runtime env and verified in Datadog | None |
| R6: Operational Metrics | Not done | No custom application metrics are exported from any reviewed service; Go services have metrics exporter disabled in default values | `LFXV2-1745` |
| R7: Dashboards | Not done | No dashboard definitions were found in the reviewed repos | `LFXV2-1747` |
| R8: Alerts and SLOs | Not done | No monitor, alert, or SLO definitions were found in the reviewed repos | `LFXV2-1748` |
| R9: Runbooks | Not done | No production observability runbooks were found in the reviewed repos | `LFXV2-1749` |

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

### Go API Services Behind `LFX_V2_SERVICE`

Status:

- The first Go API service repos have now been checked out locally and reviewed for observability scaffolding.
- Local UI and workflow context show that `lfx-v2-ui` proxies production backend requests to `LFX_V2_SERVICE`.
- Checked repos include:
  - `linuxfoundation/lfx-v2-query-service` for `/query/resources`, `/query/resources/count`, and related query endpoints.
  - `linuxfoundation/lfx-v2-committee-service` for `/committees` and committee-related endpoints.
  - `linuxfoundation/lfx-v2-meeting-service` for `/itx/meetings`, `/itx/past_meetings`, and related meeting endpoints.
  - `linuxfoundation/lfx-v2-survey-service` for survey-related API and event processing paths.
- Jira production verification for `LFXV2-1728` identifies `lfx-v2-voting-service` as part of the required coverage and reports zero traces for it, but the repo was not present in the local workspace at the time of this update.
- Additional service ownership still needs confirmation for `/groupsio`, `/projects`, and other proxied API paths.

Implemented:

- All three reviewed Go services bootstrap OpenTelemetry SDK configuration from environment in startup code.
- All three services wrap their HTTP server handlers with `otelhttp.NewHandler` and exclude `/livez` and `/readyz` from trace noise.
- All three services expose `/livez` and `/readyz` in the Goa design/generated server layer.
- All three Helm charts configure startup, liveness, and readiness probes against `/readyz` and `/livez`.
- All three services use structured `slog` JSON logging.
- All three services wrap the log handler with `slog-otel`, which adds `trace_id` and `span_id` when logs are emitted with a traced context.
- `lfx-v2-query-service` wraps outbound generic HTTP calls and OpenSearch HTTP transport with `otelhttp.NewTransport`.
- `lfx-v2-query-service` readiness checks the resource search service, which covers OpenSearch readiness.
- `lfx-v2-committee-service` readiness checks NATS-backed storage readiness.
- `lfx-v2-meeting-service` readiness currently reports OK because the ITX proxy path is stateless at readiness time.

Known gaps:

- The default Helm values set OTEL export to disabled: `tracesExporter: "none"`, `metricsExporter: "none"`, and empty OTLP endpoint values. Production release values must explicitly enable OTLP export.
- Jira production verification reports that cross-service trace propagation is broken: RUM traces flow into `lfx-v2-ui`, but Go API spans have `parentid: 0` and start new traces instead of continuing the frontend/UI trace.
- Jira production verification reports that `lfx-v2-survey-service` and `lfx-v2-voting-service` have zero Datadog APM traces.
- No Datadog dashboard, monitor, SLO, or runbook definitions were found in these repos.
- No custom application metrics counters or histograms were found for route-level business operations, NATS publish/request failures, OpenSearch failures, FGA/Authzed checks, ITX proxy failures, queue processing, or saturation.
- `lfx-v2-query-service` has HTTP and OpenSearch transport instrumentation, but no explicit custom spans were found for business operations such as resource query, organization query, FGA tuple reads, or access checks.
- `lfx-v2-committee-service` has inbound HTTP tracing, but no explicit custom spans or instrumented NATS transport were found for NATS KV, request/reply, publisher, or stream consumer operations.
- `lfx-v2-meeting-service` has inbound HTTP tracing, but no explicit custom spans were found for ITX proxy operations, NATS event processing, NATS ID mapping, or event publishing.
- `lfx-v2-meeting-service` builds many outbound ITX HTTP requests, but no `otelhttp.NewTransport` was found for that proxy client, so outbound ITX client spans may be missing.
- `lfx-v2-survey-service` has OpenTelemetry dependencies and commented Helm env placeholders, but no active SDK setup or `otelhttp.NewHandler` wrapping was found in `cmd/survey-api/main.go`.
- `lfx-v2-survey-service` has NATS event processing, NATS publishing, ID mapping, and outbound proxy HTTP paths without confirmed spans.
- `lfx-v2-voting-service` still needs local code inspection to confirm whether the Jira-reported zero-trace state is caused by missing SDK initialization, missing deployment config, or another runtime issue.
- Datadog-compatible `dd.trace_id` and `dd.span_id` fields are not explicitly added; compatibility depends on Datadog/OpenTelemetry ingestion behavior unless production log processing maps the existing `trace_id` and `span_id` fields.

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
- Go API services expose `/livez` and `/readyz`, or documented equivalents with the same liveness/readiness semantics.
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
- Go API service Helm or deployment manifests configure startup, liveness, and readiness probes.
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
- Go API services emit inbound HTTP spans and dependency spans for OpenSearch, FGA/Authzed, NATS, databases, and service-to-service HTTP calls where applicable.
- Trace context propagates from frontend/RUM to `lfx-v2-ui`, through `LFX_V2_SERVICE`, and into the Go API services.
- `lfx-v2-survey-service` and `lfx-v2-voting-service` appear as Datadog APM services under production traffic.
- `lfx-changelog` either migrates to OpenTelemetry or documents a Datadog-only exception.
- Traces reach Datadog in staging and production.
- Trace sampling is configurable without code changes.

### R4: Trace and Log Correlation

Every backend log line emitted during a traced request must be correlatable to the trace.

Acceptance criteria:

- Logs include `trace_id` and `span_id` where OpenTelemetry is used.
- Datadog-formatted IDs are included where Datadog correlation requires them.
- `lfx-v2-ui` validates that log fields appear under production traffic.
- Go API services validate trace-log correlation fields under production-like traffic.
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
- Go API services expose or derive metrics for route latency, route errors, OpenSearch latency/errors, FGA/Authzed latency/errors, NATS publish/request failures, database errors, and service-to-service call failures where applicable.
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

- Dashboards exist for `lfx-v2-ui`, each production Go API service, and `lfx-changelog`.
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

### Phase 3: Go API Service Audit

Scope:

- Identify the exact Go repositories behind production `LFX_V2_SERVICE`.
- Map proxied UI paths to service owners, including `/query/*`, `/committees/*`, `/itx/*`, `/surveys`, `/votes`, `/groupsio`, and `/projects`.
- Audit each Go service for tracing, structured logs, trace-log correlation, health endpoints, Kubernetes probes, metrics, dashboards, alerts, and runbooks.
- Verify trace propagation from `lfx-v2-ui` to Go service handlers and downstream dependencies.

Deliverables:

- Code-backed inventory of Go API observability implementation by repo.
- Gap list by service with required owners and priority.
- Updated PRD findings replacing the current unaudited status.
- Staging trace showing frontend/RUM to UI/BFF to Go API to downstream dependency where applicable.

### Phase 4: Trace-Log Correlation Fixes

Scope:

- Validate log correlation for `lfx-v2-ui`.
- Validate backend service-layer spans for existing NATS and Snowflake integrations.
- Add or standardize explicit service-layer spans for `MicroserviceProxyService` / `ApiClientService` so Query Service and LFX API calls carry useful operation names and attributes beyond raw HTTP auto-instrumentation.
- Review direct backend `fetch` clients and add explicit spans for high-value dependencies where needed.
- Validate trace-log correlation for Go API services.
- Enable or replace log injection for `lfx-changelog`.

Deliverables:

- A production request can be followed from RUM to trace to logs.
- A slow or failing dashboard/API request can be broken down by SSR route, service-layer operation, downstream dependency, and dependency status/error.
- Sensitive data remains redacted.
- Correlation fields are documented.

### Phase 5: Dashboards, Alerts, and Runbooks

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
- What exact Go repositories make up production `LFX_V2_SERVICE`?
- Which repos own `/groupsio`, `/projects`, and other proxied API paths not yet mapped?
- Are Go API services behind a shared gateway, and does that gateway preserve `traceparent` and Datadog propagation headers?
- What SLO targets should apply to each service?
- Who owns dashboard and monitor creation: app teams, platform, or ops?
- What session replay privacy level is approved for production?

## Risks

- Tracing may appear implemented but remain disabled if release values omit `OTEL_EXPORTER_OTLP_ENDPOINT`.
- Health endpoints that do not check dependencies can show green while feature-specific paths fail.
- Overly aggressive readiness dependency checks can remove otherwise healthy SSR pods from service.
- UI/BFF traces may stop at `LFX_V2_SERVICE` if Go API services do not propagate context or emit spans.
- Jira production verification already shows trace continuity stops at the Go API boundary for some flows, which prevents ops from following a user request end to end.
- Until `lfx-v2-voting-service` is locally inspected and `lfx-v2-survey-service` tracing is activated, the production backend service layer remains partially invisible in APM.
- Session replay and user context require privacy review.
- 100% sampling may increase Datadog cost for high-traffic routes.

## Success Metrics

- 100% of production services have liveness and readiness probes.
- 100% of production backend services emit traces to Datadog.
- 100% of traced backend logs are correlatable by trace ID.
- 100% of Go API services behind `LFX_V2_SERVICE` have audited observability status and owners.
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
- [ ] Exact Go API repositories behind `LFX_V2_SERVICE` are identified.
- [ ] Go API services expose liveness and readiness endpoints.
- [ ] Go API service deployments wire startup/liveness/readiness probes.
- [ ] Go API services emit inbound HTTP traces and downstream dependency spans.
- [ ] Trace context propagates from `lfx-v2-ui` into Go API services.
- [ ] `lfx-v2-survey-service` initializes OpenTelemetry SDK and wraps the HTTP handler with `otelhttp`.
- [ ] `lfx-v2-voting-service` repo is checked out and audited for health, probes, tracing, logs, metrics, dashboards, alerts, and runbooks.
- [ ] `lfx-v2-survey-service` and `lfx-v2-voting-service` appear in Datadog APM under production traffic.
- [ ] Go API service logs correlate with traces.
- [ ] Go API service metrics cover route latency/errors and key downstream dependency failures.
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
2. Fix cross-service trace propagation from RUM/UI through `lfx-v2-ui` into Go API services.
3. Activate and verify tracing for `lfx-v2-survey-service` and `lfx-v2-voting-service`.
4. Complete local inspection of `lfx-v2-voting-service` and any remaining production Go services behind `LFX_V2_SERVICE`.
5. Add Kubernetes probes for `lfx-changelog`.
6. Verify trace-log correlation across all services.
7. Create dashboards, alerts, and runbooks.
