# Security

## Execution Policy Reality

- Execution policy is a safety feature, not a full security boundary.
- Enforcement semantics differ by platform.
- On non-Windows, the reported default is `Unrestricted` and policy changes via
  `Set-ExecutionPolicy` are not supported; behavior effectively maps to
  `Bypass` semantics because Windows Security Zones aren't implemented.

## Script Signing

- On Windows, signing helps satisfy `AllSigned`/`RemoteSigned` scenarios.
- Use trusted code-signing certificates for distributed scripts.
- Use self-signed certificates only for local/testing scenarios.
- Do not frame signing as a sole control; combine with app control, logging, and
  least-privilege execution.

## Secrets & Credentials

- Never embed plaintext secrets in scripts or repos.
- Prefer secret abstractions and secure stores (for example SecretManagement).
- Favor certificate-based, managed identity, or integrated auth when possible.
- For repository publishing workflows, avoid plaintext API keys/PATs in script
  source; retrieve secrets at runtime from a vault.

## Dangerous Patterns to Reject

- `Invoke-Expression` on untrusted input.
- Building command strings from user data and executing them.
- Storing credentials in source or verbose logs.

## Logging and Defense in Depth

- Use module/script block logging in managed environments where required.
- Treat AMSI, constrained language mode, and application control policies as
  defense-in-depth controls.
- For remoting, require secure transport and least-privilege endpoints.
