---
name: typescript-guru
description: Enforces TypeScript 7 best practices when writing, reviewing, or refactoring TypeScript code. Use when the user asks for TypeScript setup, tsconfig guidance, type-system design, Node or bundler module configuration, testing, linting, runtime validation, or migration from JavaScript. Covers the TypeScript 7 native compiler and its migration from 6, strict compiler options, ESLint flat config with typescript-eslint, module-resolution correctness, Node native type-stripping limits, and security-focused handling of untrusted data. Do NOT use for non-TypeScript languages or generic architecture discussions without concrete TypeScript implementation details.
license: Apache-2.0
metadata:
  version: "2.0.0"
---

# TypeScript Guru

Enforce modern, production-grade TypeScript. Prefer strictness, explicit runtime
boundaries, and platform-correct module configuration over convenience shortcuts.

## Versions

TypeScript 7.0 (8 July 2026) is the current release, and the first built on the Go-based native
compiler: full builds run roughly eight to twelve times faster, and `tsc` is still the command.
Options that 6.0 deprecated are now errors, among them `target: "es5"` and `baseUrl`, so fix the
6.0 deprecation warnings before upgrading.

7.0 has no stable programmatic API. That arrives in 7.1, in beta at the time of writing, and until
then tools built on the compiler API, typescript-eslint among them, cannot run on 7. Install
`@typescript/typescript6` beside it for those tools, and type-check with 7. Check the
typescript-eslint release notes before saying it supports 7.

## Critical

- Treat `tsconfig` as a correctness gate, not editor preferences.
- Default to `strict: true` and keep strictness high (`noUncheckedIndexedAccess`,
  `exactOptionalPropertyTypes`, `noImplicitOverride`, `useUnknownInCatchVariables`).
- Never trust external input types; validate untrusted data at runtime before use.
- Keep module settings aligned with runtime (`nodenext` vs `bundler`) to avoid
  "works-in-dev, fails-in-prod" import bugs.
- Use `import type` and `verbatimModuleSyntax` to keep type/value imports correct.
- Use ESLint + typescript-eslint for code quality; use a dedicated formatter
  (Prettier/dprint) for formatting.
- Always include CI quality gates: typecheck, lint, test.
- Default to Vitest for TypeScript unit/integration test guidance unless the
  user explicitly asks for Jest compatibility.
- Node can run `.ts` files directly, but only by stripping the types. It does not read
  `tsconfig`, type-check, or transform syntax such as enums and parameter properties, so a project
  that relies on any of those still needs `tsc`, `tsx` or a bundler. Saying "Node runs TypeScript
  now" without that caveat sends people into runtime errors.
- If a question is not about TypeScript, answer it directly and briefly. TypeScript advice nobody
  asked for buries the answer.

## Response Guardrails

- For new projects, provide a concise baseline covering: `tsconfig`, linting,
  formatter, tests, and CI checks.
- For Node runtime questions, distinguish:
  - Node native type-stripping mode (lightweight, limited transforms)
  - Full TypeScript workflows (`tsc`, `tsx`, bundlers) with `tsconfig` support
- For testing stack questions, recommend Vitest first for modern TypeScript.
  Mention Jest only for migration/interoperability contexts.
- For module advice, ask or infer target runtime first:
  - App + bundler: `moduleResolution: "bundler"`
  - Node runtime output: `module: "nodenext"`
- For `import type` questions, cover `verbatimModuleSyntax: true` as well as `import type`:
  without it, the compiler may keep or drop an import differently from what the code says.
- For API data handling, reject direct casts from unknown (`as Foo`) and provide
  runtime validation/narrowing patterns.
- For migration/refactor tasks, prefer incremental strictness with measurable gates.

## When to Use

- Writing new TypeScript apps/libraries/CLIs/services
- Reviewing or refactoring existing TypeScript code
- Designing TypeScript type models and generic APIs
- Configuring `tsconfig.json`, ESLint, formatter, and CI
- Debugging module-resolution/import/runtime mismatches
- Improving test quality and runtime safety for TS code

## When NOT to Use

- Non-TypeScript languages where TS is incidental
- Pure architecture discussions with no concrete TS implementation
- Ops-only tasks unrelated to TypeScript code authoring

## Enforcement Checklist

Apply this checklist to every TypeScript answer.

### Tooling & Project Setup
- [ ] TypeScript latest stable release line used for project target
- [ ] ESLint flat config with `typescript-eslint` configured
- [ ] Formatter handled separately (Prettier/dprint), not lint-rule formatting
- [ ] `tsconfig.json` committed and treated as contract
- [ ] `npm`/`pnpm` scripts include `typecheck`, `lint`, and `test`

### TSConfig Strictness
- [ ] `strict: true`
- [ ] `noUncheckedIndexedAccess: true`
- [ ] `exactOptionalPropertyTypes: true`
- [ ] `noImplicitOverride: true` (especially for class-heavy code)
- [ ] `useUnknownInCatchVariables: true`
- [ ] `verbatimModuleSyntax: true`
- [ ] `isolatedModules: true` where toolchain expects per-file transforms
- [ ] `skipLibCheck` only as pragmatic dependency-speed tradeoff, not a safety feature

### Type-System Quality
- [ ] Prefer `unknown` over `any` at boundaries
- [ ] Use discriminated unions for variant states
- [ ] Use `satisfies` to preserve inference while enforcing shape
- [ ] Prefer explicit return types on exported APIs
- [ ] Avoid over-assertion (`as`) and non-null assertions (`!`) in business logic
- [ ] Keep generics minimal and intention-revealing

### Modules & Runtime Correctness
- [ ] Module settings match runtime reality (`bundler` vs `nodenext`)
- [ ] Type-only imports use `import type`
- [ ] Relative import extension rules follow chosen runtime/toolchain
- [ ] Avoid path alias assumptions in runtimes that do not transform them
- [ ] For Node native TS execution, document unsupported non-erasable syntax limits

### Testing & CI
- [ ] Unit test runner is modern and TS-friendly (Vitest default recommendation)
- [ ] Typecheck gate runs in CI (`tsc --noEmit`)
- [ ] Lint gate runs in CI (`eslint .`)
- [ ] Tests include happy-path + failure-path coverage
- [ ] E2E/UI tests use Playwright when browser behaviour matters

### Security & Data Handling
- [ ] Validate all untrusted input at runtime (schema validation/type guards)
- [ ] Secrets/config from environment, not hardcoded
- [ ] Avoid dynamic code execution (`eval`, `new Function`) on external input
- [ ] Build DB/query layers with parameterised APIs and validated inputs
- [ ] Treat deserialised JSON as unknown until validated

### Anti-Patterns (Instant Fail)
- [ ] No `any` spread through core domain logic
- [ ] No `as SomeType` on untrusted external data without validation
- [ ] No silent swallowing of promise rejections
- [ ] No module config mismatch likely to break runtime imports
- [ ] No TSLint-era or deprecated tooling recommendations
- [ ] No forcing ESLint to be primary formatter in typed projects

## Quick Reference: Outdated → Modern

| Avoid | Prefer |
|---|---|
| TSLint | ESLint flat config + `typescript-eslint` |
| Lint rules as formatter | Dedicated formatter (Prettier/dprint) + lint for logic |
| `any` at boundaries | `unknown` + narrowing/validation |
| `as`-casting API payloads | Schema validation + inferred types |
| Runtime-agnostic module guesses | Runtime-aligned `module`/`moduleResolution` |
| Ad-hoc CI | `typecheck` + `lint` + `test` gates |

## Project Bootstrap

Use this baseline for a modern TypeScript project:

```text
my-ts-project/
├── src/
├── tests/
├── package.json
├── tsconfig.json
├── eslint.config.mjs
└── vitest.config.ts
```

Baseline scripts:

```json
{
  "scripts": {
    "typecheck": "tsc --noEmit",
    "lint": "eslint .",
    "test": "vitest run"
  }
}
```

## Validation Guidelines

Before final output:

1. Confirm runtime target (Node vs bundler) and align module settings.
2. Check strictness posture (`strict` + key strict options).
3. Check boundary safety for untrusted data (validation before usage).
4. Check CI quality gates (`typecheck`, `lint`, `test`).
5. Prefer the simplest implementation that is type-safe at compile time and runtime.

## References

Load these files when deeper detail is required:

- `references/tooling.md`: ESLint flat config, typescript-eslint, formatter strategy, CI
- `references/tsconfig-and-types.md`: strict compiler options and type-system patterns
- `references/modules-and-runtime.md`: Node vs bundler settings, import semantics, native TS runtime limits
- `references/testing.md`: Vitest/Playwright guidance and TS test quality gates
- `references/security-and-validation.md`: runtime validation, boundary safety, secret handling
- `references/anti-patterns.md`: common TypeScript mistakes and modern remediations
