---
name: pwsh-guru
description: Enforces PowerShell 7.6 best practices when writing, reviewing, or refactoring PowerShell scripts and modules. Use when the user asks for PowerShell scripting help, advanced function design, module packaging, script linting/testing, security hardening, or automation reliability. Covers CmdletBinding and ShouldProcess patterns, approved verbs, PSScriptAnalyzer, Pester 5, PSResourceGet, and stream-aware output design. Do NOT use for non-PowerShell languages or generic infrastructure discussions without concrete PowerShell implementation details.
license: Apache-2.0
metadata:
  version: "1.1.0"
---

# PowerShell Guru

Enforce modern, safe, idiomatic PowerShell 7.6 across every task. Prefer
advanced functions, approved verbs, analyzer-backed quality gates, and
automation-safe behaviour. Never recommend legacy or risky practices when a safer
first-class PowerShell approach exists.

Target PowerShell 7.6, the current Long Term Support release (18 March 2026, built on .NET 10,
supported until November 2028). 7.4, the previous LTS, went out of support on 10 November 2026, so
scripts that still run on it should move. Windows PowerShell 5.1 is a separate product: say so when
an answer depends on a 7-only feature.

## When to Use

- Writing new PowerShell scripts or modules
- Reviewing/refactoring existing PowerShell automation
- Designing advanced functions and command UX
- Building CI checks for PowerShell quality and compatibility
- Advising on security posture, remoting, and signing practices
- User asks: "is this good PowerShell?", "review my pwsh script", "best practice for function/module"

## When NOT to Use

- Non-PowerShell languages unless PowerShell is the implementation target
- Pure cloud/infrastructure architecture with no PowerShell code path
- Generic sysadmin advice not involving script/module authoring

## Critical

- Treat execution policy as a safety feature, not a security boundary.
- On non-Windows, state clearly that execution policy is effectively not enforced
  like Windows zone-based policy; avoid relying on it for protection.
- For state-changing commands, require `SupportsShouldProcess` and use
  `$PSCmdlet.ShouldProcess(...)`.
- For command names, require approved verbs and Verb-Noun format.
- For quality gates, require PSScriptAnalyzer in CI and Pester tests for behaviour.
- For secret handling, prohibit plaintext credentials and recommend
  SecretManagement/secure providers.
- For package management guidance, prefer PSResourceGet cmdlets and only mention
  PowerShellGet v2 compatibility when migration context is explicit.

## Enforcement Checklist

Run through this checklist for every PowerShell code answer.

### Command Design
- [ ] Function uses `[CmdletBinding()]` (advanced function behaviour)
- [ ] Name uses approved Verb-Noun pattern (verify with `Get-Verb`)
- [ ] State-changing commands implement `SupportsShouldProcess`
- [ ] Destructive actions honour `-WhatIf` / `-Confirm`
- [ ] Parameters use explicit types and validation attributes

### Parameters & Pipeline
- [ ] Parameter sets are clear and unambiguous
- [ ] `ValueFromPipeline` / `ValueFromPipelineByPropertyName` used intentionally
- [ ] Switch parameters are non-mandatory unless distinguishing parameter sets
- [ ] Do not rely on implicit positional binding for readability-sensitive commands
- [ ] Use splatting for long command invocations

### Output & Error Semantics
- [ ] Business output goes to success stream (objects, not formatted text)
- [ ] `Write-Verbose`, `Write-Debug`, `Write-Warning`, `Write-Error` used correctly
- [ ] Avoid `Write-Host` in reusable automation unless explicitly host-UX only
- [ ] Non-terminating vs terminating error behaviour is explicit
- [ ] Stream behaviour supports automation, piping, and redirection

### Tooling & Packaging
- [ ] Linting uses `Invoke-ScriptAnalyzer` with repo settings file
- [ ] CI includes analyzer and test tasks
- [ ] Module/package installs use `Install-PSResource` workflow
- [ ] Module manifests are present and complete for published modules
- [ ] Cross-version compatibility checks are considered (`PSUseCompatible*` rules)

### Testing
- [ ] Tests use Pester v5 style (`Describe`/`Context`/`It`)
- [ ] File-system side effects isolated via `TestDrive:` where applicable
- [ ] Mocks used for external dependencies and expensive calls
- [ ] Assertions validate behaviour and error paths
- [ ] Tests are CI-runnable and deterministic

### Security
- [ ] No plaintext secrets in script source
- [ ] Prefer SecretManagement/cert-based/managed identity approaches
- [ ] Avoid `Invoke-Expression` with untrusted input
- [ ] Script signing guidance is Windows-aware and practical
- [ ] Remoting guidance requires secure transport and least privilege

## Anti-Patterns (instant fail)

- Unapproved verbs like `Delete-*` instead of approved verb alternatives
- State-changing function without `SupportsShouldProcess`
- Treating execution policy as the main security control
- Returning formatted strings/tables where objects are expected
- Plaintext credentials/API keys embedded in scripts
- Skipping analyzer/test quality gates for reusable automation

## Quick Reference: Old → Modern

| Old / Risky | Modern / Preferred |
|---|---|
| `Install-Module`-only guidance | `Install-PSResource` / PSResourceGet-first guidance |
| Simple function for reusable command | Advanced function with `[CmdletBinding()]` |
| `Delete-*` / non-approved verbs | Approved Verb-Noun from `Get-Verb` |
| Manual confirmations only | `SupportsShouldProcess` + `-WhatIf` / `-Confirm` |
| `Write-Host` for script output | Object output + stream-specific messaging |
| No lint gate | `Invoke-ScriptAnalyzer` with settings in CI |
| Ad-hoc script testing | Pester v5 suites with clear structure |
| Plaintext secrets in code | SecretManagement/secure credential source |

## References

- `references/tooling.md`: PSScriptAnalyzer, PSResourceGet, CI setup
- `references/advanced-functions.md`: CmdletBinding, ShouldProcess, parameters
- `references/security.md`: execution policy reality, signing, secrets, remoting
- `references/testing.md`: Pester v5 patterns and TestDrive usage
- `references/anti-patterns.md`: common pitfalls and concrete fixes
- `references/web-verified-baseline.md`: source-grounded claims and recency map
