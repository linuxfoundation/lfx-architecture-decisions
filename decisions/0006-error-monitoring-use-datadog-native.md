# 6. Error monitoring uses Datadog-native capabilities

Date: 2026-03-17

## Status

Proposed

## Context

A custom service (`lfx-error-monitor`) has been proposed that polls CloudWatch logs
every 30 minutes, sends them to Claude via LiteLLM for AI-powered analysis, creates
GitHub issues, and posts Slack notifications. The service requires a Python application,
PostgreSQL database, Helm chart, ArgoCD manifests across three environments, IRSA roles
with CloudWatch permissions, a CI/CD pipeline, and Docker image builds.

CloudWatch logs are already forwarded to Datadog via the Datadog Forwarder. Datadog
natively provides Error Tracking (automatic error grouping and deduplication), Watchdog
(AI-powered anomaly detection with zero configuration), and Workflow Automation (with
GitHub and Slack actions) — all operating on the same log data in near real-time.

## Decision

Services and applications MUST use Datadog Error Tracking, Watchdog, and Workflows for
automated error detection, triage, and notification rather than deploying custom
monitoring services that duplicate existing data pipelines.

Error detection and anomaly identification SHOULD use Datadog Watchdog, which baselines
normal log patterns and surfaces deviations natively without external AI/LLM
dependencies.

Automated actions (GitHub issue creation, Slack notifications) triggered by error
conditions SHOULD be implemented as Datadog Workflows using template variables from
the trigger payload.

Custom services that independently poll CloudWatch (or other log sources already
forwarded to Datadog) MUST NOT be deployed, as they duplicate existing data pipelines
and introduce unnecessary infrastructure.

```
CloudWatch → Datadog Forwarder → Datadog Log Management (all existing)
                                          │
                                ┌─────────┴─────────┐
                                ▼                   ▼
                          Watchdog              Error Tracking
                          (anomaly detect)      (auto-group/dedup)
                                └─────────┬─────────┘
                                          ▼
                                   Datadog Workflow
                                          │
                                   ┌──────┴──────┐
                                   ▼             ▼
                              GitHub Issue   Slack Alert
                              (template vars)
```

## Consequences

Teams will need to implement error monitoring workflows within Datadog rather than
building standalone services. This reduces infrastructure overhead and on-call burden
but requires familiarity with Datadog Workflows.

Watchdog's anomaly detection replaces the need for LLM-based log analysis for error
identification, eliminating external API dependencies and associated costs. However,
GitHub issue descriptions will use structured template variables rather than
LLM-generated prose. An LLM summarization step can be added later via a Workflow HTTP
action if template-based issues prove insufficient.

Existing hand-crafted per-error-string monitors (currently 25+) can be gradually retired
as Error Tracking and Watchdog prove effective, reducing monitor sprawl and maintenance
burden.

This decision deepens the organization's dependency on Datadog for operational tooling.
If Datadog changes Workflow pricing or capabilities, alternatives would need to be
evaluated.
