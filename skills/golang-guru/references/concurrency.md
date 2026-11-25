# Concurrency and Goroutine Lifecycle

## Context-first concurrency

- Pass `context.Context` through request and service boundaries.
- Cancel work on timeout, client disconnect, shutdown, or upstream failure.
- In fan-out patterns, propagate cancellation to all child goroutines.

## Leak and deadlock prevention

- Define who owns channel close responsibility; avoid ambiguous ownership.
- Ensure goroutines can exit: avoid blocking sends/receives with no cancellation
  path.
- Use `select` with `ctx.Done()` where blocking operations are possible.
- Prefer structured coordination (`errgroup` patterns or equivalent) over
  untracked fire-and-forget goroutines.

## Signals and shutdown

- Use buffered channels with `os/signal.Notify` to avoid dropped signals.
- Implement graceful shutdown with bounded deadlines.

## Validation tools

- `go test -race` for runtime race detection (coverage-dependent).
- `go vet` checks include goroutine/testing misuse patterns.
- Go 1.26 introduces experimental goroutine leak profiling (`goroutineleak`)
  behind `GOEXPERIMENT=goroutineleakprofile`.

## Sources

- Go security best practices: https://go.dev/doc/security/best-practices
- Race detector article: https://go.dev/doc/articles/race_detector
- `go vet` docs: https://pkg.go.dev/cmd/vet
- Go 1.26 release notes (goroutine leak profile): https://go.dev/doc/go1.26
