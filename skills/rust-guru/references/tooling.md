# Tooling

Use Rust-native tooling and enforce a reproducible quality pipeline.

## Core Setup

- Install toolchain with `rustup`; prefer stable by default.
- Ensure required components are installed (for example `rustfmt`, `clippy`).
- For project consistency, use `rust-toolchain.toml` when teams need pinned channel/components.
- Set `edition` to the latest stable value supported by project constraints.
- Set `rust-version` in `Cargo.toml` to declare MSRV policy.

## CI Quality Gates

Use this sequence in CI:

1. `cargo fmt --check`
2. `cargo clippy --all-targets --all-features -- -D warnings`
3. `cargo test --all-targets --all-features`
4. Optional: `cargo test --doc`

## Cargo Patterns

- Use workspaces for multi-crate repos and shared lint/profile configuration.
- Build/test with relevant feature matrices for crates with optional features.
- Keep dependencies minimal and reviewed (`cargo tree` and regular dependency maintenance checks).

## Linting Practices

- Treat warnings as failures in CI for high-signal quality.
- Use targeted `#[allow]` only with rationale; avoid broad lint suppression.
- Prefer fixing root causes over suppressing clippy findings.
