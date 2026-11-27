# Testing TypeScript Code

## Layered strategy

1. Fast unit tests (Vitest)
2. Integration tests for service boundaries
3. Browser/e2e tests (Playwright) where user flows matter
4. Type-level safety gate (`tsc --noEmit`) in CI

## Unit/integration baseline

- Use Vitest for TS-first fast feedback.
- Keep tests deterministic and isolated.
- Cover success and error paths.

## Browser/e2e baseline

- Use Playwright for cross-browser flow validation.
- Favour locator-based assertions and built-in auto-wait behavior.
- Keep tests independent; avoid serial mode unless unavoidable.
- Use worker limits strategically in CI.

## CI quality gates

- `npm run typecheck`
- `npm run lint`
- `npm run test`

Optional:

- `npm run test:e2e` for Playwright suites

## Sources

- Vitest guide: https://vitest.dev/guide/
- Playwright writing tests: https://playwright.dev/docs/writing-tests
- Playwright parallelism: https://playwright.dev/docs/test-parallel
