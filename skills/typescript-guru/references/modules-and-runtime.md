# Modules and Runtime Correctness

## Pick module settings by execution model

### App with bundler

Use bundler-oriented settings:

```json
{
  "compilerOptions": {
    "module": "esnext",
    "moduleResolution": "bundler",
    "verbatimModuleSyntax": true,
    "noEmit": true
  }
}
```

### Compile and run output in Node.js

Use Node-oriented settings:

```json
{
  "compilerOptions": {
    "module": "nodenext",
    "verbatimModuleSyntax": true
  }
}
```

## Import correctness

- Use `import type` for type-only symbols.
- Keep relative import specifiers runtime-correct for target environment.
- Avoid assuming path aliases are transformed unless toolchain guarantees it.

## Node native TypeScript support

Node provides lightweight TypeScript type stripping.

- Good for simple scripts and erasable TypeScript syntax.
- Not a replacement for full TypeScript transforms and tsconfig-driven builds.
- Features with runtime semantics (for example enums/parameter properties in
  strict strip mode scenarios) can require other toolchains or flags.

For full compatibility and ecosystem workflows, use `tsc`, `tsx`, or bundlers.

## Practical guidance

- If shipping libraries for broad consumers, prefer stricter compatibility and
  runtime-accurate module checking.
- Be explicit about `module`/`moduleResolution` in docs and templates.
- Validate imports on CI with `tsc --noEmit` and runtime smoke tests.

## Sources

- TypeScript module option guidance:
  https://www.typescriptlang.org/docs/handbook/modules/guides/choosing-compiler-options.html
- TSConfig moduleResolution:
  https://www.typescriptlang.org/tsconfig/moduleResolution.html
- Node TypeScript docs:
  https://nodejs.org/api/typescript.html
- TS 5.8 release notes (`erasableSyntaxOnly`):
  https://www.typescriptlang.org/docs/handbook/release-notes/typescript-5-8.html
