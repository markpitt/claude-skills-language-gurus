# Advanced Functions

## Core Pattern

- Use `[CmdletBinding()]` for production-grade functions.
- Use explicit `param(...)` typing and validation attributes.
- Keep command UX consistent with built-in cmdlets.

## ShouldProcess Pattern (state-changing)

- Add `SupportsShouldProcess` in `CmdletBinding`.
- Call `$PSCmdlet.ShouldProcess(target, action)` immediately before the mutating
  operation.
- Pair with sensible `ConfirmImpact` for risk-aware prompts.

## Naming

- Use approved verbs and Verb-Noun format.
- Prefer `Get-Verb` and approved-verb docs for validation.
- Avoid synonyms of approved verbs (for example `Delete` vs `Remove`).

## Parameters

- Prefer explicit parameter names in examples and guidance.
- Use validation attributes (`ValidateSet`, `ValidatePattern`,
  `ValidateNotNullOrEmpty`, `ValidateRange`) where appropriate.
- Keep switch parameters non-mandatory unless they distinguish parameter sets.
- Use pipeline binding only when behaviour is clear and predictable.

## Output Behaviour

- Output rich objects to the success stream.
- Use stream-specific messages for diagnostics:
  - `Write-Verbose`
  - `Write-Debug`
  - `Write-Warning`
  - `Write-Error`
- Avoid turning data output into formatted text in reusable functions.

## Help

- Add comment-based help (`.SYNOPSIS`, `.DESCRIPTION`, `.PARAMETER`, `.EXAMPLE`).
- Keep examples executable and aligned to parameter naming conventions.
