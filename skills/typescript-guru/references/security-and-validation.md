# Security and Runtime Validation

## Core rule

TypeScript types do not validate runtime data. Anything from network, storage,
or user input is untrusted until validated.

## Boundary safety pattern

1. Receive unknown input as `unknown`.
2. Validate with schema/type guards.
3. Convert to trusted domain types only after successful validation.

## Practical recommendations

- Avoid direct assertions on external input (`as DomainType`).
- Use schema validation libraries (for example Zod) to parse and infer trusted
  output types.
- Keep secrets and deploy-specific config in environment variables.
- Reject dynamic code execution patterns on untrusted input.
- Use parameterized query APIs in data access layers.

## Example

```ts
import { z } from "zod";

const Payload = z.object({
  id: z.string().uuid(),
  count: z.number().int().nonnegative(),
});

function parsePayload(input: unknown) {
  return Payload.parse(input);
}
```

## Sources

- Zod docs: https://zod.dev/
- Zod repository: https://github.com/colinhacks/zod
- OWASP Top 10 (2025): https://owasp.org/Top10/2025/
- Twelve-Factor config: https://12factor.net/config
- Node `process.env`: https://nodejs.org/api/process.html#processenv
