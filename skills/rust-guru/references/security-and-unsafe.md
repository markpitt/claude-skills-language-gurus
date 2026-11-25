# Security and Unsafe

Rust improves memory safety by default, but secure outcomes still require discipline.

## Unsafe Policy

- Use `unsafe` only when safe Rust cannot express required behavior.
- Minimize unsafe block scope.
- Document safety invariants adjacent to each unsafe block.
- Encapsulate unsafe internals behind safe APIs.

## Input and Boundary Security

- Validate length/format/range of untrusted input before processing.
- Avoid panic-oriented parsing paths for user-controlled data.
- Apply timeouts/limits for potentially expensive operations.

## Dependency and Crypto Hygiene

- Prefer widely audited crates and keep dependencies current.
- Never implement custom cryptography primitives.
- Keep secrets out of source and logs.

## Reliability Practices

- Include failure-path tests for malformed/untrusted input.
- Use fuzzing/Miri/sanitizers for safety-critical components where feasible (Miri requires nightly).
