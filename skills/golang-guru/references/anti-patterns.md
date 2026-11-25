# Common Go Anti-Patterns

## Error handling

Avoid:
- Comparing errors by message text
- Dropping context when returning errors

Prefer:
- `fmt.Errorf("...: %w", err)`
- `errors.Is` / `errors.As`

## Concurrency

Avoid:
- Fire-and-forget goroutines in request paths
- Goroutines blocked forever on channels with no cancellation path
- `sync.WaitGroup.Add` inside goroutines

Prefer:
- Context-aware goroutines with explicit shutdown/cancel logic
- Structured fan-out/fan-in patterns with lifecycle ownership

## SQL and input handling

Avoid:
- SQL built with string formatting/concatenation

Prefer:
- Parameterized queries/placeholders

## Testing and CI

Avoid:
- Relying only on `go test` for concurrent systems
- No vet/static checks in CI

Prefer:
- `go test`, `go vet`, `go test -race`, and fuzz workflows where relevant

## Performance

Avoid:
- Micro-optimizing without benchmark/profile evidence

Prefer:
- Benchmark + `pprof` driven optimization plans

## Observability

Avoid:
- Unstructured logs for complex services

Prefer:
- Structured logs with stable fields (`log/slog` style)
