# 3. OpenTelemetry instrumentation

Date: 2025-08-28

## Status

Proposed

## Context

Integrating OpenTelemetry instrumentation universally allows us to collect
traces and metrics in a standardized way that works with our preferred
observability platform, and support the operations and support teams in
troubleshooting production issues.

## Decision

OpenTelemetry tracing MUST be used for all new projects, including propogating
trace and parent-span context from both inbound HTTP requests and outbound HTTP
requests.

Typically most of our metrics come from infrastructure integrations; however,
if an application or service needs to send custom metrics, it MUST use
OpenTelemetry to send these.

Structured logs emitted by the application over `stdout` or `stderr` MUST
include trace and span IDs for log correlation. Logs SHOULD include both
OpenTelemetry-formatted trace & span IDs (lowercase hex-encoded 128-bit and
64-bit unsigned integers, respectively, as `trace_id` and `span_id`) _and_
DataDog-formatted trace and span IDs (both 64-bit unsigned ints, as
`dd.trace_id` and `dd.span_id`).

Container deployments SHOULD use an `opentelemetry-collector` sidecar to route
traces and metrics. In production & staging, it MUST send traces and metrics to
DataDog. In development & local environments, traces SHOULD be sent to a Jaeger
container, and metrics SHOULD use the standard-output exporter.

In AWS Elastic Container Service (ECS) deployments (production & staging), a
`dd-agent` sidecar (using its OTEL GRPC receiver for both traces and metrics)
SHOULD be used instead of `opentelemetry-collector` due to its ease of
configuration with environmental variables—ECS lacks an equivalent to
ConfigMaps for file-based configuration.

## Consequences

Adding observability to applications which lack it will increase APM vendor
costs. LFX services providing interactive services for logged-in users are
typically sampled at 100%, whereas higher-throughput services should use a
lower sampling rate to help control costs.

Not routing logs via OpenTelemetry means the team will not fully be leveraging
OpenTelemetry's capabilities at this time, and refactoring may be needed if
this decision is later superseded.
