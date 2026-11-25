# Web-Verified Baseline (2026-02)

This file records key claims enforced by `pwsh-guru` and the primary source used
for verification.

## Advanced Functions and CmdletBinding

- Advanced functions should use `[CmdletBinding()]` and cmdlet-like binding.
  - Source: `about_Functions_Advanced`
  - URL: <https://learn.microsoft.com/en-us/powershell/module/microsoft.powershell.core/about/about_functions_advanced?view=powershell-7.5>

- `SupportsShouldProcess` adds `-WhatIf`/`-Confirm` semantics and should be used
  for state-changing commands.
  - Source: `about_Functions_CmdletBindingAttribute`
  - URL: <https://learn.microsoft.com/en-us/powershell/module/microsoft.powershell.core/about/about_functions_cmdletbindingattribute?view=powershell-7.5>

## Approved Verbs and Naming

- Use approved verbs; avoid synonyms (for example, prefer `Remove` over
  `Delete`).
  - Source: Approved verbs guidance
  - URL: <https://learn.microsoft.com/en-us/powershell/scripting/developer/cmdlet/approved-verbs-for-windows-powershell-commands?view=powershell-7.5>

- `Get-Verb` helps inspect approved verbs and command-name consistency.
  - Source: `Get-Verb`
  - URL: <https://learn.microsoft.com/en-us/powershell/module/microsoft.powershell.core/get-verb?view=powershell-7.5>

## Execution Policy and Security

- Execution policy is a safety feature, not a security boundary.
- On non-Windows, policy changes are unsupported and behavior effectively maps to
  bypass-like semantics.
  - Source: `about_Execution_Policies`
  - URL: <https://learn.microsoft.com/en-us/powershell/module/microsoft.powershell.core/about/about_execution_policies?view=powershell-7.5>

## Analyzer and Rule Enforcement

- Use `Invoke-ScriptAnalyzer` in CI with settings files; compatibility rules are
  available (`PSUseCompatible*`).
  - Source: Using ScriptAnalyzer
  - URL: <https://learn.microsoft.com/en-us/powershell/utility-modules/psscriptanalyzer/using-scriptanalyzer?view=ps-modules>

- Relevant rules include approved verbs, should-process, plain text password,
  and invoke-expression checks.
  - Source: Rules index
  - URL: <https://learn.microsoft.com/en-us/powershell/utility-modules/psscriptanalyzer/rules/readme?view=ps-modules>

## Package Management (Modern)

- PSResourceGet is the modern package-management surface replacing legacy-first
  guidance.
  - Source: Package management overview
  - URL: <https://learn.microsoft.com/en-us/powershell/gallery/powershellget/overview?view=powershellget-3.x>

- Core cmdlets: `Find-PSResource`, `Install-PSResource`, `Publish-PSResource`.
  - Sources:
    - <https://learn.microsoft.com/en-us/powershell/module/microsoft.powershell.psresourceget/find-psresource?view=powershellget-3.x>
    - <https://learn.microsoft.com/en-us/powershell/module/microsoft.powershell.psresourceget/install-psresource?view=powershellget-3.x>
    - <https://learn.microsoft.com/en-us/powershell/module/microsoft.powershell.psresourceget/publish-psresource?view=powershellget-3.x>

## Testing

- Pester v5 remains the primary testing framework with `Describe`/`Context`/`It`
  patterns and CI integration.
  - Source: Pester quick start
  - URL: <https://pester.dev/docs/quick-start>

## Secrets

- SecretManagement provides secure vault-backed secret handling (`Get-Secret`,
  `Set-Secret`, `Register-SecretVault`, etc.).
  - Source: SecretManagement module docs
  - URL: <https://learn.microsoft.com/en-us/powershell/module/microsoft.powershell.secretmanagement/?view=ps-modules>
