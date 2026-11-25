---
name: rust-guru
description: Enforces current Rust best practices when writing, reviewing, or refactoring Rust code. Use when the user asks for Rust help, crate/project setup, ownership and borrowing guidance, async/concurrency patterns (Tokio), error handling, performance tuning, testing, or unsafe-code review. Covers rustup/cargo tooling, current stable-edition idioms, Result-first error handling, Clippy/rustfmt quality gates, property testing, and security hardening. Do NOT use for non-Rust languages or generic architecture discussions without concrete Rust implementation details.
license: Apache-2.0
metadata:
  version: "1.0.0"
---

# Rust Guru

Enforce modern, idiomatic Rust for production code. Prefer stable Rust, the
latest stable edition available in the project/toolchain,
and standard library or widely adopted ecosystem crates before niche solutions.

## Critical

- Treat warnings as quality signals; enforce `clippy` with denied warnings in CI.
- Prefer explicit ownership/borrowing design over clone-heavy workarounds.
- Use `Result`-based error handling in production paths; avoid `unwrap`/`expect` except in tests/prototypes.
- Keep async runtimes non-blocking; move CPU/blocking work to `spawn_blocking`.
- Keep `unsafe` minimal, document invariants, and expose safe wrappers.
- For setup guidance, always include a Rust quality gate block:
  - `cargo fmt --check`
  - `cargo clippy --all-targets --all-features -- -D warnings`
  - `cargo test --all-targets --all-features`

## Response Guardrails

- For new projects, recommend stable toolchains via `rustup` and pin project intent with `rust-toolchain.toml` when needed.
- For language guidance, default to the latest stable edition in the repo/toolchain; if unspecified, say “latest stable edition supported by your toolchain”.
- For CI/linting questions, include `cargo clippy --all-targets --all-features -- -D warnings` explicitly.
- For `unwrap`/`expect` production questions, always include a concise `Result` + `?` propagation example.
- For async guidance, always mention cancellation/timeout strategy and blocking isolation.
- For security/perf-sensitive guidance, prefer measurable, simple changes over speculative micro-optimisations.

## When to Use

- Writing new Rust applications, libraries, CLIs, services, or embedded code
- Reviewing/refactoring Rust code for correctness, safety, or performance
- Solving ownership, borrowing, lifetimes, trait/object model questions
- Advising on async concurrency with Tokio or async ecosystem crates
- Designing error handling, crate APIs, and module boundaries
- Setting up Rust project tooling, CI, tests, and quality gates

## When NOT to Use

- Non-Rust language tasks where Rust is incidental
- High-level architecture discussions without concrete Rust code
- Ops-only tasks unrelated to Rust code authoring

## Enforcement Checklist

Apply this checklist to every Rust response.

### Tooling & Project Setup
- [ ] Use `rustup` + Cargo-native workflows
- [ ] Use latest stable edition available for the project/toolchain
- [ ] Define and communicate MSRV policy (`rust-version` in `Cargo.toml`)
- [ ] Enforce `rustfmt` and `clippy` in CI (`-D warnings` gate)
- [ ] Use `cargo check`, `cargo test`, and feature-aware builds (`--all-features` where appropriate)
- [ ] Prefer workspace-level consistency (`[workspace]`, shared lints/profiles)

### API Design, Ownership & Types
- [ ] Use borrowing (`&T`, `&str`, slices) before cloning owned data
- [ ] Use `String`/`Vec<T>` ownership only where mutation/ownership transfer is required
- [ ] Prefer enums for closed sets and exhaustive matching
- [ ] Avoid public APIs requiring unnecessary generic complexity
- [ ] Prefer newtypes for domain constraints over raw primitives
- [ ] Keep lifetimes explicit only when inference is insufficient for clarity

### Error Handling
- [ ] Return `Result<T, E>` in fallible APIs; use `?` for propagation
- [ ] Use domain errors (`thiserror`) in libraries
- [ ] Use app-level context aggregation (`anyhow`) in binaries/entrypoints
- [ ] Avoid `unwrap`/`expect` in production paths
- [ ] Convert and map errors intentionally at boundaries (HTTP/CLI/FFI)

### Async & Concurrency (Tokio)
- [ ] Avoid blocking runtime threads in async contexts
- [ ] Use `tokio::spawn` / `JoinSet` with explicit shutdown strategy
- [ ] Use timeout/cancellation patterns (`tokio::time::timeout`, `select!`, cancellation tokens)
- [ ] Propagate cancellation and join task handles cleanly
- [ ] Prefer backpressure-aware channels and bounded queues

### Performance
- [ ] Measure before optimising (`cargo bench`, flamegraph/profiler)
- [ ] Reduce allocation churn (`with_capacity`, reuse buffers)
- [ ] Minimise clone/copy on hot paths
- [ ] Prefer iterators/slices over intermediate collections where clear
- [ ] Keep hot-path error handling cheap and predictable

### Security & Unsafe
- [ ] Treat `unsafe` as last resort with documented safety invariants
- [ ] Keep `unsafe` blocks small and encapsulated behind safe APIs
- [ ] Validate untrusted input size/format before parsing
- [ ] Avoid panics for user-controlled input paths
- [ ] Use vetted crypto crates; never roll custom cryptography

### Testing & Quality
- [ ] Use unit tests + integration tests (`tests/`) in Cargo workflow
- [ ] Add property-based tests (`proptest` preferred) for pure logic
- [ ] Include boundary/failure tests, not only happy paths
- [ ] Run docs/examples where applicable (`cargo test --doc`)
- [ ] Use fuzzing and Miri guidance for safety-critical code paths (Miri requires nightly)

### Anti-Patterns (Instant Fail)
- [ ] `unwrap`/`expect` in production request or parsing paths
- [ ] Clone-heavy code to silence borrow-checker issues
- [ ] Blocking I/O/CPU work on async executor worker threads
- [ ] Large undocumented `unsafe` regions without invariants
- [ ] Public APIs exposing `&String` / `&Vec<T>` instead of `&str` / `&[T]`
- [ ] Silent error swallowing (`let _ = ...`) on critical operations

## Quick Reference: Legacy → Modern

| Avoid | Prefer |
|---|---|
| Ad-hoc compiler install | `rustup` toolchain management |
| Unchecked formatting/lints | `cargo fmt --check` + strict `clippy` |
| `unwrap` in app paths | `Result` + `?` + boundary mapping |
| Clone to appease borrow checker | Ownership redesign + borrowing |
| Blocking work inside async task | `spawn_blocking` / dedicated worker pools |
| Wide `unsafe` areas | Minimal audited `unsafe` + safe wrappers |
| Tests only for happy paths | Unit + integration + property tests |

## Project Bootstrap

Use this baseline for a modern Rust service/library:

```text
my-rust-project/
├── Cargo.toml
├── rust-toolchain.toml
├── src/
│   ├── lib.rs or main.rs
│   └── ...
├── tests/
│   └── integration_*.rs
└── .github/workflows/ci.yml
```

Bootstrap commands:

```bash
cargo new my-rust-project
cd my-rust-project
rustup toolchain install stable
cargo fmt
cargo clippy --all-targets --all-features -- -D warnings
cargo test --all-targets --all-features
```

## Validation Guidelines

Before final output:

1. Confirm edition/toolchain guidance is stable and current.
2. Check ownership and error handling choices for clarity and safety.
3. Check async code for cancellation, timeout, and blocking isolation.
4. Confirm tests and quality gates are explicit and runnable.
5. Prefer the simplest design that is safe, idiomatic, and measurable.

## References

Load these files for depth as needed:

- `references/tooling.md`: rustup, cargo, edition/MSRV, CI quality gates
- `references/ownership-and-api.md`: ownership, borrowing, API surface design
- `references/error-handling.md`: Result strategy, `thiserror`/`anyhow`, boundary mapping
- `references/async.md`: Tokio patterns, cancellation, blocking isolation
- `references/testing.md`: unit/integration/property/fuzz testing practices
- `references/security-and-unsafe.md`: unsafe invariants, secure coding practices
- `references/performance.md`: profiling-driven optimisation patterns
- `references/anti-patterns.md`: common Rust pitfalls and modern remediations
