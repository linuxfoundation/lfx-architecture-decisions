# 1. Python projects use uv

Date: 2025-08-27

## Status

Accepted

## Context

Over the last few years, Python toolchains have become increasingly fractured.
Adopting a standard Python delivery framework accelerates and improves the
maintainability of product-team-managed Python projects, especially as the team
launches more AI-enabled applications, which frequently rely on Python
components.

## Decision

New Python projects MUST use `uv` with `pyproject.toml`, `uv.lock`, and
`.python-version` files. The version specified in `.python-version` MUST meet
the requirements of the `requires-python` constraint in `pyproject.toml`.

Existing Python projects SHOULD migrate to `uv` when feasible.

GitHub Actions workflows that run Python SHOULD use the `actions/checkout`
action, then the `actions/setup-python` action with `python-version-file:
.python-version`, followed by the `astral-sh/setup-uv` action with
`enable-cache: true`, then `uv sync --locked` to install dependencies and `uv
run` to execute commands.

Docker builds SHOULD use a multi-stage, dependency-caching approach as
demonstrated in the upstream
[multistage.Dockerfile](https://github.com/astral-sh/uv-docker-example/blob/main/multistage.Dockerfile)
example.

## Consequences

Developers may need to learn how to use `uv` and adapt existing workflows and
build toolchains to align with this decision.

By not establishing a recommendation for source or binary (wheel) distributions
at this time, the team may experience fragmentation among `uv build`, `poetry`,
`setuptools`, etc.
