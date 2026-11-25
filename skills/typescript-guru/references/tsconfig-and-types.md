# TSConfig and Type-System Guidance (2026)

## Strictness baseline

Start with:

```json
{
  "compilerOptions": {
    "strict": true,
    "noUncheckedIndexedAccess": true,
    "exactOptionalPropertyTypes": true,
    "noImplicitOverride": true,
    "useUnknownInCatchVariables": true,
    "verbatimModuleSyntax": true
  }
}
```

## Why these options matter

- `strict`: enables the full strict-family baseline.
- `noUncheckedIndexedAccess`: adds `undefined` for undeclared indexed access,
  preventing false confidence in dictionary/index signature reads.
- `exactOptionalPropertyTypes`: distinguishes "property absent" from
  "property present but undefined".
- `noImplicitOverride`: catches inheritance drift by requiring `override`.
- `useUnknownInCatchVariables`: prevents assuming caught values are `Error`.
- `verbatimModuleSyntax`: preserves import/export intent and avoids fragile
  emit rewrites.

## Modern type-system habits

- Use `unknown` at boundaries and narrow explicitly.
- Use discriminated unions for state machines and API result variants.
- Use `satisfies` for shape checking without widening literals.
- Keep exported API return types explicit.
- Prefer utility types and mapped types over ad-hoc duplication.

## Notes on generated defaults

Recent `tsc --init` templates favor stricter defaults and minimal config,
including strictness and module-related options. Treat generated templates as
starting points, not final architecture.

## Sources

- TS 5.9 release notes (`tsc --init` updates):
  https://www.typescriptlang.org/docs/handbook/release-notes/typescript-5-9.html
- TS config strict: https://www.typescriptlang.org/tsconfig/strict.html
- TS config noUncheckedIndexedAccess:
  https://www.typescriptlang.org/tsconfig/noUncheckedIndexedAccess.html
- TS config exactOptionalPropertyTypes:
  https://www.typescriptlang.org/tsconfig/exactOptionalPropertyTypes.html
- TS config noImplicitOverride:
  https://www.typescriptlang.org/tsconfig/noImplicitOverride.html
- TS config useUnknownInCatchVariables:
  https://www.typescriptlang.org/tsconfig/useUnknownInCatchVariables.html
