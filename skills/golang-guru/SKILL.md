---
name: golang-guru
description: Enforces Go 1.26 best practices when writing, reviewing, or refactoring Go code. Use when the user asks for Go language help, modules/toolchain setup, concurrency and context patterns, testing and CI quality gates, security hardening, performance profiling, or production service design in Go. Covers modern Go tooling, error-handling discipline, goroutine lifecycle safety, and evidence-driven optimisation. Do NOT use for non-Go languages or generic architecture discussions without concrete Go implementation details.
license: Apache-2.0
metadata:
  version: "1.0.0"
---

# Go Guru

Enforce modern, production-grade Go practices with a correctness-first, security-
aware posture. Prefer simple, idiomatic solutions and explicit trade-offs.

When in doubt, check the reference files before answering.

## Critical

- Default to module-based workflows; do not recommend GOPATH-era project setup
  as the standard approach.
- Use modern toolchain guidance via `go.mod` version requirements and toolchain
  management semantics (`go`, `toolchain`, `GOTOOLCHAIN`) when versioning is
  relevant.
- Enforce disciplined error handling: wrap with `%w`, inspect with `errors.Is`
  and `errors.As`, avoid brittle string matching.
- Treat context propagation and cancellation as mandatory for request-scoped and
  background concurrency; prevent goroutine leaks explicitly.
- Recommend quality gates by default: `gofmt`, `go test`, `go vet`, and
  `go test -race` for concurrency-sensitive code.
- Include fuzzing guidance when input parsing, protocol handling, security
  boundaries, or CI test strategy is discussed.
- Include security workflow guidance with `govulncheck`, dependency hygiene, and
  parameterised SQL queries (never string-built SQL).
- For security or API-hardening prompts, explicitly include both:
  1) `govulncheck` in CI and
  2) parameterised SQL/placeholders (with explicit rejection of `fmt.Sprintf`
     SQL construction).
- Promote observability and performance discipline: measure first with benchmarks
  and `pprof`; avoid speculative micro-optimisations.
- Prefer structured logging for services (`log/slog` or equivalent) over
  ad-hoc unstructured logs.
- For unrelated/non-Go prompts, answer directly and concisely without injecting
  Go tooling or language advice.

## Response Guardrails

- For setup prompts, provide a concise baseline: modules/toolchain, formatting,
  lint/vet, tests, race checks, CI gates.
- For API/service code review prompts, identify concrete risks first, then show
  minimal corrected patterns.
- For concurrency prompts, mention cancellation, bounded lifetimes, and owner-
  ship of channel closing where relevant.
- For data access prompts, reject SQL string concatenation and recommend
  placeholders/parameter binding.
- For API hardening prompts, always include concrete dependency/vulnerability
  scanning guidance (`govulncheck`) in addition to input/authn/authz advice.

### Mandatory Security Template

For prompts about hardening/securing Go APIs/services, include this minimum
checklist explicitly (wording can vary; items cannot be omitted):

- `govulncheck ./...` in CI (or equivalent govulncheck workflow)
- SQL safety: use parameterised queries/placeholders; never `fmt.Sprintf` SQL

### Conditional Guidance Template

When relevant to the user request, also include:

- `go test -race ./...` for runtime race detection
- `go vet ./...` for suspicious construct checks
- For performance prompts, lead with measurement methodology before code-level
  tuning.
- For CI test pipeline prompts, include whether/when to run fuzzing
  (`go test -fuzz`) and corpus management guidance.
- For profiling/performance prompts in services, include observability/logging
  guidance (for example structured logging with `log/slog`) in addition to
  benchmarks/pprof.
- For unrelated prompts, do not force Go guidance.

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

### Anti-Patterns (Instant Fail)
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
5. Confirm unrelated prompts remain concise and non-injected.

## References

Load these files when deeper detail is required:

- `references/modules-and-toolchain.md`: modules, versioning, toolchain policy
- `references/concurrency.md`: goroutine lifecycle, context, channel discipline
- `references/testing-and-quality.md`: tests, race detector, vet, fuzzing
- `references/security.md`: govulncheck, SQL safety, dependency hygiene
- `references/performance-and-observability.md`: benchmarking, pprof, slog
- `references/anti-patterns.md`: common Go mistakes and safer alternatives
