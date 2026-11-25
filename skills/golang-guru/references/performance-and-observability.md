# Performance and Observability

## Evidence-driven optimization

- Optimize only after measurement.
- Start with representative benchmarks and production-like profiles.
- Use `pprof` for CPU/memory/contention analysis before code changes.

## Benchmarking practices

- Keep setup outside timed loops.
- For modern benchmarks, prefer `for b.Loop() { ... }` style.
- Compare changes with statistical tooling (for example benchstat) when possible.

## Profiling and PGO

- Profile real workloads, not synthetic edge-only traces.
- Use Profile-Guided Optimization where appropriate (`default.pgo` or `-pgo`)
  and keep profiles representative of production behavior.

## Structured logging and signals

- Prefer structured logging (`log/slog`) in services.
- Capture stable keys for correlation (request IDs, subsystem, operation,
  duration), avoid free-form message-only logs.

## Sources

- PGO guide: https://go.dev/doc/pgo
- `testing` benchmarks: https://pkg.go.dev/testing
- `log/slog` blog post: https://go.dev/blog/slog
- Go 1.26 release notes (`slog` updates): https://go.dev/doc/go1.26
