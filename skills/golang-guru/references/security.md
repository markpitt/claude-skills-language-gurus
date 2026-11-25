# Security Practices for Go

## Vulnerability scanning and updates

- Run `govulncheck` regularly and in CI.
- Keep Go toolchain and dependencies updated, but validate updates with tests.
- Subscribe to `golang-announce` for security release pre-announcements.

## Input and data handling

- Use parameterized SQL queries/placeholders.
- Do not build SQL with `fmt.Sprintf` or string concatenation.
- Validate/parse untrusted input with explicit error handling paths.

## Runtime safety checks

- Run race detector (`go test -race`) for concurrent code paths.
- Use fuzzing for parser/protocol boundary hardening.
- Use `go vet` as a baseline static analyzer for suspicious constructs.

## Source-level secure defaults

- Return contextual wrapped errors instead of suppressing detail.
- Fail closed on parse/validation errors at boundaries.
- Avoid leaking sensitive values into logs; use structured fields and redaction.

## Sources

- Go security best practices: https://go.dev/doc/security/best-practices
- Govulncheck tutorial: https://go.dev/doc/tutorial/govulncheck
- SQL injection avoidance: https://go.dev/doc/database/sql-injection
- Race detector: https://go.dev/doc/articles/race_detector
