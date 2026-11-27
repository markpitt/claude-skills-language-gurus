---
name: golang-guru
description: Enforces Go 1.27 best practices when writing, reviewing, or refactoring Go code. Use when the user asks for Go language help, modules/toolchain setup, concurrency and context patterns, testing and CI quality gates, security hardening, performance profiling, or production service design in Go. Covers modern Go tooling, error-handling discipline, goroutine lifecycle safety, and evidence-driven optimisation. Do NOT use for non-Go languages or generic architecture discussions without concrete Go implementation details.
license: Apache-2.0
metadata:
  version: "2.0.0"
---

# Go Guru

Enforce modern, production-grade Go practices with a correctness-first, security-
aware posture. Prefer simple, idiomatic solutions and explicit trade-offs.

When in doubt, check the reference files before answering.

## Rules, and why they hold

- **Modules, always.** Recommend `go.mod` projects, never GOPATH-era layout. When versions matter,
  use the `go` and `toolchain` lines and `GOTOOLCHAIN`, so the build picks the same compiler on
  every machine.
- **Wrap errors and inspect them by type.** Wrap with `%w` and test with `errors.Is` and
  `errors.As`. Matching on an error's text breaks the first time someone rewords a message.
- **Every goroutine has an owner and an end.** Pass `context.Context` through request-scoped and
  background work and stop on cancellation. A goroutine nobody cancels is a leak that shows up in
  production as slowly rising memory, not as a failing test.
- **Quality gates by default.** `gofmt`, `go vet`, `go test`, and `go test -race` for anything
  concurrent. The race detector finds in seconds what a code review misses.
- **Fuzz the parsers.** When input parsing, protocol handling or a security boundary is involved,
  add `go test -fuzz` and keep the corpus. Hand-written cases only cover the inputs someone thought
  of.
- **Hardening means the dependency tree as well as the code.** An answer about securing a Go API or
  service should include `govulncheck ./...` in CI, because a known vulnerability in a dependency is
  as exploitable as one in your own handler. It should also insist on parameterised queries
  (placeholders) and reject SQL built with `fmt.Sprintf` or concatenation, which is the injection
  hole most often found in review. Input validation and authentication sit alongside these, not
  instead of them.
- **Measure before tuning.** Lead with benchmarks (`for b.Loop()`) and `pprof`. An optimisation
  without a profile is a guess, and it usually makes the code harder to read for no gain.
- **Structured logs for services.** `log/slog` or equivalent, so logs can be queried rather than
  grepped.
- **Stay in your lane.** If a question is not about Go, answer it directly. Go advice nobody asked
  for buries the answer.

## Shaping the answer

- A set-up question gets a short baseline: modules and toolchain, formatting, vet, tests, race
  checks and CI gates.
- A code review leads with the concrete risks, then shows the smallest corrected pattern.
- A concurrency question covers cancellation, bounded lifetimes and who closes each channel.
- A performance question starts with how to measure, and for a service adds structured logging and
  observability beside the benchmarks.
- A CI question says when fuzzing should run and how the corpus is kept.

If a review request has no code attached, still provide a compact checklist:

- Error wrapping and inspection strategy
- Context propagation and cancellation path
- Testing/race/vet coverage expectations
- Then ask for the concrete snippet for line-level review

## When to Use

- Writing or reviewing Go CLIs, services, libraries, and backend systems
- Setting up Go modules, toolchain/version policy, and CI quality gates
- Debugging concurrency issues, goroutine leaks, races, and deadlocks
- Hardening Go code against input, dependency, and runtime security risks
- Profiling and optimising Go programs with benchmark/profile evidence

## When NOT to Use

- Non-Go language requests
- Pure architecture discussions with no concrete Go implementation details
- Language-agnostic Q&A where Go specificity adds no value

## Enforcement Checklist

Apply this checklist to every Go answer.

### Modules & Toolchain
- [ ] Use module-first workflow and modern `go.mod` practices
- [ ] Provide version/toolchain guidance when compatibility is part of the ask
- [ ] Avoid defaulting to legacy GOPATH-era setup

### Error Handling
- [ ] Prefer wrapped errors (`%w`) over lost context
- [ ] Recommend `errors.Is` / `errors.As` for inspection
- [ ] Reject string-based error comparisons

### Concurrency & Context
- [ ] Propagate `context.Context` across boundaries
- [ ] Include cancellation strategy for goroutines and I/O
- [ ] Address leak/deadlock risks with explicit lifecycle guidance

### Testing & Quality
- [ ] Include `gofmt` and `go test` in baseline guidance
- [ ] Include `go vet` for suspicious constructs
- [ ] Include `go test -race` for concurrent code
- [ ] Include fuzzing guidance for risky input paths
- [ ] For CI pipeline prompts, mention fuzzing strategy explicitly

### Security
- [ ] Recommend `govulncheck` and dependency update/testing discipline
- [ ] Reject SQL built via string formatting/concatenation
- [ ] Prefer parameterised SQL and safe defaults for parsing/validation
- [ ] For API hardening prompts, include both vulnerability scanning and SQL
  parameterisation guidance explicitly

### Performance & Observability
- [ ] Recommend benchmarks and/or `pprof` before optimisation decisions
- [ ] Avoid unsupported performance claims without measurement
- [ ] Include structured logging guidance for service contexts
- [ ] For profiling/performance prompts, include observability signals (logs,
      metrics, traces) and structured logging (`log/slog` or equivalent)

### Never
- [ ] No endorsement of string-matched error handling as primary strategy
- [ ] No approval of SQL string construction with user input
- [ ] No concurrency advice that ignores context cancellation in server flows
- [ ] No omission of core quality gates for production-oriented guidance
- [ ] No Go advice injected into unrelated prompts

## Quick Reference: Avoid → Prefer

| Avoid | Prefer |
|---|---|
| GOPATH-first setup | Go modules (`go.mod`) as default |
| Error string matching | `fmt.Errorf(... %w ...)` + `errors.Is`/`errors.As` |
| Fire-and-forget goroutines | Context-aware goroutines with cancellation/join strategy |
| SQL via `fmt.Sprintf` | Parameterised queries/placeholders |
| “Optimise first, measure later” | Benchmarks + `pprof` before tuning |
| Unstructured production logs | Structured logs (`log/slog` or equivalent) |

## Baseline CI Guidance

Use this baseline unless project constraints require changes.

- Format: `gofmt -w` (or check mode in CI)
- Test: `go test ./...`
- Vet: `go vet ./...`
- Race: `go test -race ./...` (at least on supported targets)
- Security: `govulncheck ./...`

Add fuzz workflows where parsing/protocol logic is security-relevant.

## Validation Pass Before Final Answer

1. Confirm module/toolchain advice is modern and practical.
2. Confirm error handling guidance uses wrapping + `errors.Is`/`errors.As`.
3. Confirm concurrency guidance addresses cancellation and lifecycle safety.
4. Confirm quality/security gates are present when relevant.
5. Confirm an answer to a non-Go question has no Go advice in it.

## References

Load these files when deeper detail is required:

- `references/modules-and-toolchain.md`: modules, versioning, toolchain policy
- `references/concurrency.md`: goroutine lifecycle, context, channel discipline
- `references/testing-and-quality.md`: tests, race detector, vet, fuzzing
- `references/security.md`: govulncheck, SQL safety, dependency hygiene
- `references/performance-and-observability.md`: benchmarking, pprof, slog
- `references/anti-patterns.md`: common Go mistakes and safer alternatives
