# Tooling

## Baseline

- Target modern PowerShell (`pwsh`) and current stable feature set (7.5+).
- Use `PSScriptAnalyzer` as the required lint/static-analysis gate.
- Use `PSResourceGet` (`Install-PSResource`, `Find-PSResource`,
  `Publish-PSResource`) for module/script package workflows.
- Treat `Microsoft.PowerShell.PSResourceGet` as the modern package-management
  surface replacing legacy-first PowerShellGet/PackageManagement guidance.

## PSScriptAnalyzer

- Run `Invoke-ScriptAnalyzer -Path <project> -Recurse` in CI.
- Prefer settings-file driven rules (`PSScriptAnalyzerSettings.psd1`) to keep
  policy explicit and reviewable.
- Prefer explicit, scoped suppressions with justification instead of global
  disable patterns.
- Enforce naming and safety rules first:
  - `PSUseApprovedVerbs`
  - `PSUseShouldProcessForStateChangingFunctions`
  - `PSAvoidUsingInvokeExpression`
  - `PSAvoidUsingPlainTextForPassword`
  - Compatibility checks (`PSUseCompatibleCmdlets`, `PSUseCompatibleSyntax`,
    etc.) where version portability is required.

## PSResourceGet

- Prefer `Install-PSResource` over legacy-only PowerShellGet v2 workflows.
- Use repository registration and trust models intentionally:
  - `Register-PSResourceRepository`
  - `Set-PSResourceRepository`
  - `Install-PSResource -Repository <name>`
- Default install scope for local development should be `CurrentUser` unless
  automation/system context requires `AllUsers`.
- Use `-WhatIf`/`-Confirm` where available during automation development and
  high-impact package operations.
- Never hardcode tokens or PAT values in scripts.

## CI Pattern

- Typical quality gate order:
  1. Script/module import sanity
  2. `Invoke-ScriptAnalyzer`
  3. `Invoke-Pester`
- Fail build on analyzer warnings for critical rules in reusable modules.
- Keep PowerShell and module versions explicit in CI runners to reduce drift.
