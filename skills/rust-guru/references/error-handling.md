# Error Handling

Prefer explicit, composable failure handling.

## Core Rules

- Use `Result<T, E>` for fallible operations.
- Use `?` to propagate errors while preserving readability.
- Avoid `unwrap`/`expect` in production paths (acceptable in tests, one-off scripts, or impossible states with documented proof).

## Error Type Strategy

- Libraries: use domain-specific enums (commonly with `thiserror`).
- Applications/binaries: use top-level aggregation/context (commonly with `anyhow`).
- At boundaries (HTTP/CLI/FFI): map internal errors to user-facing representations intentionally.

## Context and Observability

- Add meaningful context before crossing module boundaries.
- Log errors once at a boundary; avoid duplicate logging at each layer.
- Preserve source errors for debugging and incident analysis.

## Panic Policy

- Treat panic as bug-level or invariant-violation behavior, not user-input control flow.
- Do not panic on malformed/untrusted external input.
