---
name: shell-guru
description: Enforces current shell scripting best practices when writing, reviewing, or refactoring Bash/POSIX sh scripts. Use when the user asks for shell scripting help, Bash/sh code review, idiomatic shell patterns, script portability, debugging, or security guidance. Covers safety options (set -Eeuo pipefail), quoting rules, arrays, error handling, testing (bats-core), tooling (shellcheck, shfmt), security, and anti-pattern elimination. Do NOT use for general system administration tasks unrelated to shell script authorship.
license: Apache-2.0
metadata:
  version: "1.1.0"
---

# Shell Guru

Enforce modern, safe, idiomatic Bash 5+ across every task. Never suggest
unquoted variables, unsafe patterns, or legacy syntax. Default to Bash unless
POSIX portability is explicitly required. Flag bashisms when writing portable
POSIX sh.

When in doubt, check the reference files before answering.

## When to Use

- Writing new shell scripts of any kind
- Reviewing or refactoring existing Bash/sh scripts
- Setting up shell script projects with linting and CI
- Debugging scripts, unexpected word splitting, or pipefail surprises
- Advising on error handling, traps, portability, or security
- User asks: "is this idiomatic bash?", "how do I do X in shell?", "review my
  script", "why does my script fail silently?"

## When NOT to Use

- Pure interactive shell config (`.zshrc`, aliases, prompts): this skill covers scripts only
- Other languages where shell is incidental (e.g. Python `subprocess` calls)
- System administration without script authorship

## Enforcement Checklist

Run through this for **every** shell script you write or review.

### Script Header
- [ ] Shebang is `#!/usr/bin/env bash` (or `#!/bin/sh` for POSIX-only targets)
- [ ] `set -Eeuo pipefail` on the first non-comment line after the shebang
- [ ] `shopt -s inherit_errexit` added for Bash 4.4+ (subshells inherit `errexit`)
- [ ] `readonly` used for all script-level constants
- [ ] Script is executable: `chmod +x script`

### Variables & Quoting
- [ ] All variable expansions double-quoted: `"$var"`, `"${var}"`, `"$(cmd)"`
- [ ] Arrays expanded with `"${arr[@]}"`. Never bare `${arr[*]}` (word splits)
- [ ] Default values with `${var:-default}`; mandatory vars with `${var:?error}`
- [ ] `declare -r` for readonly, `-i` integer, `-a` array, `-A` associative map
- [ ] `local` declared for every variable inside a function
- [ ] `local x; x=$(cmd)`. **Never** `local x=$(cmd)` (masks the exit code!)
- [ ] `read -r` always (prevents backslash interpretation)
- [ ] `$()` for command substitution; backticks are obsolete

### Conditionals & Loops
- [ ] `[[ ]]` for all conditionals in Bash (not `[ ]` or `test`)
- [ ] `(( ))` for integer arithmetic tests: `(( count > 0 ))`
- [ ] `==` / `!=` inside `[[ ]]`; `=` inside POSIX `[ ]`
- [ ] Loops over files: `for f in dir/*` or `while IFS= read -r line`
- [ ] No iterating `$(ls ...)`. Use globs or `find`

### Error Handling & Traps
- [ ] `trap 'cleanup' EXIT INT TERM` for temp-file and state cleanup
- [ ] `trap 'on_error "$LINENO"' ERR` for actionable failure messages
- [ ] Exit code preserved in cleanup: `local _exit=$?; ...; exit "$_exit"`
- [ ] `cd` result always checked: `cd /some/dir || exit 1`
- [ ] Non-critical teardown steps use `|| true`

### File & Process Safety
- [ ] Temp files created with `tmpfile=$(mktemp)`. Never a hardcoded `/tmp/name`
- [ ] `mapfile -t arr < file` or `while IFS= read -r line` to read files line-by-line
- [ ] `find` or globs used instead of parsing `ls` output
- [ ] `--` added before user-supplied arguments passed to commands
- [ ] `printf '%s\n' "$var"` preferred over `echo "$var"` for non-literal output

### Security
- [ ] No `eval` on external or user-supplied data
- [ ] No `bash -c "$user_input"` (command injection)
- [ ] All file paths from external sources validated before use
- [ ] No hardcoded credentials. Use env vars or permission-restricted files
- [ ] Scripts run with least privilege; `sudo` scope minimised

### Tooling
- [ ] Script passes `shellcheck -x -s bash script` with zero warnings
- [ ] Formatted with `shfmt -i 2 -ci -bn -sr`
- [ ] Any disabled shellcheck rules have an inline comment explaining why
- [ ] Non-trivial logic has `bats-core` tests
- [ ] `mise.toml` tasks defined for `lint`, `format`, and `test`

## Quick-Reference: Old → New

| Old (avoid) | New (use) |
|---|---|
| `` `cmd` `` | `$(cmd)` |
| `[ "$x" = "y" ]` | `[[ "$x" == "y" ]]` |
| `for f in $(ls dir/)` | `for f in dir/*` |
| `if [ $? -eq 0 ]` | `if cmd; then` |
| `cat file \| cmd` | `cmd < file` (UUOC) |
| `local x=$(cmd)` | `local x; x=$(cmd)` |
| `/tmp/hardcoded-tmp` | `tmpfile=$(mktemp)` |
| `read line` | `read -r line` |
| `echo "$var"` | `printf '%s\n' "$var"` |
| `set -e` alone | `set -Eeuo pipefail` + `shopt -s inherit_errexit` |
| `[ -z $var ]` | `[[ -z "$var" ]]` |
| `(( count++ ))` near `set -e` | `(( count++ )) \|\| true`. See safety.md |
| `export VAR=val` in functions | `declare -g VAR=val` (or just `VAR=val` if already global) |

## Script Bootstrap

For a new Bash script, always start with this header:

```bash
#!/usr/bin/env bash
set -Eeuo pipefail
shopt -s inherit_errexit

SCRIPT_DIR="$(cd "$(dirname "${BASH_SOURCE[0]}")" && pwd)"
readonly SCRIPT_DIR

# --- Logging ---
err()  { printf '[ERROR] %s\n' "$*" >&2; }
info() { printf '[INFO]  %s\n' "$*"; }
warn() { printf '[WARN]  %s\n' "$*" >&2; }

# --- Cleanup ---
cleanup() {
  local _exit=$?
  [[ -f "${tmpfile:-}" ]] && rm -f "$tmpfile"
  exit "$_exit"
}
on_error() { err "Unexpected failure at line $1 — command: ${BASH_COMMAND}"; }
trap cleanup EXIT INT TERM
trap 'on_error "$LINENO"' ERR
```

For a **script project**, use this layout:

```text
my-script-project/
├── bin/
│   └── my-script          # main entry point (no .sh extension)
├── lib/
│   └── helpers.bash       # sourced library functions
├── tests/
│   ├── test_helper.bash
│   └── test_my_script.bats
├── .shellcheckrc
└── mise.toml
```

Minimal `.shellcheckrc`:

```ini
shell=bash
enable=all
source-path=lib
```

Minimal `mise.toml`:

```toml
[tools]
shellcheck = "latest"
shfmt      = "latest"
bats       = "latest"

[tasks.lint]
run = "shellcheck -x bin/* lib/*.bash"

[tasks.format]
run = "shfmt -i 2 -ci -bn -sr -w bin/* lib/*.bash"

[tasks."format:check"]
run = "shfmt -i 2 -ci -bn -sr -d bin/* lib/*.bash"

[tasks.test]
run = "bats tests/"

[tasks.check]
depends = ["lint", "format:check", "test"]
description = "Full pre-push check"
```

## References

Load these files when you need depth beyond this overview:

- `references/tooling.md`: shellcheck, shfmt, bats-core, mise, CI/CD setup
- `references/safety.md`: set/shopt options, traps, variable scoping, IFS, getopts
- `references/security.md`: injection prevention, eval risks, temp file safety, secrets
- `references/testing.md`: bats-core test patterns, mocking, fixtures, CI integration
- `references/anti-patterns.md`: full anti-pattern catalogue with fixes
