# Testing

Build confidence with layered, deterministic testing.

## Baseline Strategy

- Unit tests for small logic and invariants.
- Integration tests in `tests/` for crate-level behavior.
- Property-based tests (`proptest`) for pure/state-model logic.

## What to Test

- Happy path + boundary cases.
- Failure modes and error mappings.
- Parsing and serialisation of untrusted input.
- Concurrency/cancellation behaviour for async code.

## Tooling

- Run `cargo test --all-targets --all-features` in CI.
- Use `cargo test --doc` for executable docs where relevant.
- Consider `cargo nextest` for faster large-suite execution.

## Advanced Safety Testing

- Use fuzzing (`cargo fuzz`) for parser/protocol/security-critical code.
- Use Miri for undefined-behavior checks in unsafe-sensitive code (nightly toolchain required).
- Use sanitizer builds where environment supports them.

## Test Quality

- Keep tests deterministic and isolated.
- Prefer explicit fixtures/builders over global mutable state.
- Avoid brittle snapshot overuse for frequently changing data structures.
