# Testing and Quality Gates

## Baseline quality workflow

Use these as default CI checks:

1. `go test ./...`
2. `go vet ./...`
3. `go test -race ./...` (where supported)

## Fuzz testing

- Use fuzz tests (`func FuzzXxx(*testing.F)`) for input-heavy and parser logic.
- Provide seed corpus via `F.Add(...)` and/or `testdata/fuzz/<Name>`.
- Run targeted fuzzing with `go test -fuzz <Name>`.
- Treat discovered crashing inputs as permanent regression tests.

## Modern testing APIs

- Prefer subtests/table-driven tests for coverage and maintainability.
- For benchmarks, modern style prefers `for b.Loop() { ... }` (Go 1.24+).
- New in Go 1.26: `T.ArtifactDir`, `B.ArtifactDir`, `F.ArtifactDir` to capture
  test artifacts in CI-friendly paths.

## Vet and static analysis

- `go vet` catches suspicious constructs (including `printf`, `loopclosure`,
  `slog`, `waitgroup`, `testinggoroutine`, and more).
- Staticcheck complements vet with broader correctness and performance checks.

## Sources

- `testing` package docs: https://pkg.go.dev/testing
- Fuzzing tutorial: https://go.dev/doc/tutorial/fuzz
- `go vet` docs: https://pkg.go.dev/cmd/vet
- Staticcheck checks: https://staticcheck.dev/docs/checks
