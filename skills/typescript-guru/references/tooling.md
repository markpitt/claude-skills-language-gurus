# TypeScript Tooling (2026)

## Core stack

- Compiler: `typescript` (latest stable)
- Linting: `eslint` + `typescript-eslint`
- Formatting: dedicated formatter (`prettier` or `dprint`)
- Unit testing: `vitest`
- E2E/browser testing: `@playwright/test` when browser behaviour is relevant

## Linting and formatting separation

- Keep linting focused on correctness, code quality, and maintainability.
- Keep formatting in a dedicated formatter.
- `typescript-eslint` explicitly recommends against using ESLint as the formatter.

## ESLint setup direction

- Use flat config (`eslint.config.*`) for modern ESLint.
- Use `typescript-eslint` recommended/strict configs depending on project maturity.
- Add typed linting where cost/performance tradeoffs are acceptable.

## CI baseline

Run all three gates on pull requests:

1. `npm run typecheck` (`tsc --noEmit`)
2. `npm run lint` (`eslint .`)
3. `npm run test` (Vitest and/or Playwright suites)

## Useful package scripts

```json
{
  "scripts": {
    "typecheck": "tsc --noEmit",
    "lint": "eslint .",
    "format": "prettier --write .",
    "format:check": "prettier --check .",
    "test": "vitest run"
  }
}
```

## Sources

- TypeScript-ESLint Getting Started: https://typescript-eslint.io/getting-started/
- TypeScript-ESLint Formatting guidance: https://typescript-eslint.io/users/what-about-formatting/
- ESLint config files: https://eslint.org/docs/latest/use/configure/configuration-files
- ESLint getting started: https://eslint.org/docs/latest/use/getting-started
- Vitest guide: https://vitest.dev/guide/
- Playwright writing tests: https://playwright.dev/docs/writing-tests
