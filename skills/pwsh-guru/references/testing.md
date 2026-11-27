# Testing

## Pester v5 Baseline

- Use `Describe`, `Context`, `It` structure.
- Keep tests close to module/script source with clear naming.
- Keep assertions behavior-focused, not implementation-detail heavy.

## Isolation

- Use `TestDrive:` for file-system side effects.
- Mock external calls (`Mock`) to keep tests deterministic.
- Avoid real network/service dependencies in unit tests.

## Reliability

- Use `BeforeAll`/`BeforeEach` for setup symmetry and clarity.
- Make sure tests pass in clean sessions (no hidden global-state assumptions).
- Keep failures actionable with precise assertion messages.

## CI

- Run `Invoke-Pester` in CI after linting.
- Publish test results/coverage as pipeline artifacts where useful.
- Keep module import/version setup explicit in CI jobs.
