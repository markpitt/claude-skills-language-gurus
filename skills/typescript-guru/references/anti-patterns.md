# TypeScript Anti-Patterns

## 1) `as`-casting untrusted data

- Smell: `const payload = input as Payload`
- Risk: bypasses runtime safety and allows invalid shapes into business logic
- Fix: validate first (schema/type guard), then use inferred trusted type

## 2) `any` in core domain code

- Smell: broad `any` on critical paths
- Risk: loses compiler guarantees and propagates unchecked states
- Fix: replace with `unknown` + narrowing, or explicit domain types

## 3) Non-null assertion abuse (`!`)

- Smell: frequent `value!.prop`
- Risk: runtime crashes hidden from compiler checks
- Fix: narrow with guards and improve API contracts

## 4) Module config mismatch

- Smell: using `moduleResolution: bundler` for Node-runtime outputs without
  understanding resulting runtime import requirements
- Risk: import behavior differs between dev tooling and production runtime
- Fix: align `module` + `moduleResolution` with actual execution environment

## 5) Using ESLint as formatter

- Smell: formatting rules and plugin-based formatting as primary formatter
- Risk: slower/lower consistency in typed linting workflows
- Fix: use dedicated formatter and keep linting focused on logic/style quality

## 6) Skipping CI typecheck

- Smell: relying only on editor diagnostics
- Risk: regressions slip into CI/CD due to environment drift
- Fix: enforce `tsc --noEmit` in CI

## 7) Ignoring promise rejection paths

- Smell: async operations without explicit error handling strategy
- Risk: hidden failures and unstable runtime behavior
- Fix: handle/rethrow errors intentionally and test failure paths
