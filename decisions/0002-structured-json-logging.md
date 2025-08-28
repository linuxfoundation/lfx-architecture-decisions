# 2. Structured JSON logging

Date: 2025-08-28

## Status

Proposed

## Context

Unstructured logging, especially multiline logs or stack traces, have
limited use at scale in downstream log analytics systems, and as such they
mostly contribute noise, rather than value, and increase log ingestion
costs—sometimes dramatically.

Additionally, not using consistent log levels for different types of error or
warning conditions hampers the ability of the operations team to monitor
applications for issues that need immediate attention.

## Decision

Applications MUST support structured-JSON logging with no leading or trailing
characters: the entirety of the log line MUST be a valid JSON.

Logs MAY be on `stdout` or `stderr`—keeping the default for the library is
recommended.

Applications MUST NOT emit logs with literal newlines—though newlines MAY be
present (and escaped!) inside structured JSON logs.

Applications MUST ONLY log **errors** for non-recoverable network or service
disruptions, malformed data coming from a documented endpoint, or from the
service's own database.

Application SHOULD log **warnings** for invalid user request payloads,
temporary or recoverable network or service disruptions, or unsupported or
unparsable data from a service provider or database—where the data is not
governed by this service and the parsing is not directly correlated to a
documented endpoint contract.

Applications MUST emit **info** level logs for a successful requests that
changes (creates, updates, deletes) _non-ephemeral_ data. Applications MUST NOT
emit **info** log for "reads" (requests that do not result in changes to
_non-ephemeral_ data). Applications SHOULD limit each such request to a single
log line.

Debug logging SHOULD NOT be used for comprehensive function entry/exit
logging—consider adding custom tracing spans for critical functions. Debug
logging SHOULD be used judiciously to assist developers in understanding
application state in key workflows or when debugging.

## Consequences

Structured JSON logging will make it easier to analyze logs at scale, but
requires more diligence from developers (and code reviewers) to ensure that
logs are meaningful and consistent.
