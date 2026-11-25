# Anti-Patterns

## Naming and UX

- Unapproved verbs (`Delete-Thing`) instead of approved command verbs.
- Inconsistent or non-idiomatic command naming that breaks discoverability.

## Function Safety

- Mutating commands without `SupportsShouldProcess`.
- Ignoring `-WhatIf`/`-Confirm` semantics in state-changing operations.

## Output and Streams

- `Write-Host` as primary output for reusable automation.
- Emitting formatted strings where callers expect objects.

## Security

- Plaintext credentials/tokens in source.
- Assuming execution policy alone provides strong security guarantees.
- `Invoke-Expression` with dynamic untrusted input.

## Quality and Maintainability

- No analyzer/test gate for reusable scripts/modules.
- Suppressing analyzer rules globally without rationale.
- Complex scripts with no help metadata or examples.

## Preferred Fixes

- Rename to approved Verb-Noun patterns.
- Add `[CmdletBinding(SupportsShouldProcess)]` and gate mutating actions with
  `$PSCmdlet.ShouldProcess(...)`.
- Emit objects and use proper output streams.
- Move secrets to secure providers, not script literals.
- Enforce `Invoke-ScriptAnalyzer` and `Invoke-Pester` in CI.
