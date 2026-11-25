# Anti-Patterns

Patterns to avoid and modern replacements.

## 1) `unwrap`/`expect` in production paths

- Problem: turns recoverable failures into crashes.
- Prefer: `Result` propagation with `?` and boundary mapping.

## 2) Clone-first ownership strategy

- Problem: hidden allocations and unclear data flow.
- Prefer: borrowing and ownership redesign.

## 3) Blocking work in async contexts

- Problem: runtime stalls and latency spikes.
- Prefer: `spawn_blocking` or dedicated worker systems.

## 4) Unbounded channels for bursty pipelines

- Problem: unbounded memory growth under load.
- Prefer: bounded channels and explicit backpressure.

## 5) Wide undocumented `unsafe` regions

- Problem: unverifiable assumptions and maintenance risk.
- Prefer: narrow unsafe blocks with documented invariants and safe wrappers.

## 6) API types tied to concrete containers

- Problem: reduced flexibility (`&String`, `&Vec<T>`).
- Prefer: `&str`, `&[T]`, iterators, and domain newtypes.

## 7) Silent error swallowing (`let _ = ...`) on critical operations

- Problem: latent failures and poor observability.
- Prefer: explicit handling, logging at boundaries, and propagated errors.
