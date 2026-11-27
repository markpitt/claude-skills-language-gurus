# Go Modules and Toolchain (2026)

## Module-first baseline

- Use Go modules as the default project model.
- Keep a committed `go.mod` (and usually `go.sum`) for reproducible builds.
- Prefer explicit dependency operations (`go get`, `go mod tidy`) over ad-hoc
  edits.

## `go` line and toolchain semantics

- The `go` line in `go.mod` expresses the minimum required Go language/toolchain
  semantics for the module.
- Modern Go supports explicit toolchain selection via `toolchain` directives and
  environment policy via `GOTOOLCHAIN`.
- For teams on mixed local environments, set clear policy:
  - minimum supported `go` line,
  - optional preferred toolchain,
  - CI pinned to a known stable release.

## Practical workflow

```bash
go mod tidy
go test ./...
go vet ./...
```

For version/toolchain updates, validate in CI before broad rollout.

## Notes from current release cycle

- Go 1.27 (released 2026-08-19) is the current release, with generic methods as
  its headline language change. Go 1.26 (released 2026-02-10) is the other
  supported release. The six-month cadence and compatibility guarantees continue.
- `go mod init` behaviour now encourages compatibility with currently supported
  versions by defaulting a lower initial `go` version; teams can still adjust
  via `go get go@version`.

## Sources

- Toolchains doc: https://go.dev/doc/toolchain
- Modules reference: https://go.dev/ref/mod
- Go release history: https://go.dev/doc/devel/release
- Go 1.27 notes: https://go.dev/doc/go1.27
- Go 1.26 notes: https://go.dev/doc/go1.26
