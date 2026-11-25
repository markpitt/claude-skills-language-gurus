# Tooling Reference

Modern shell scripting toolchain as of 2026. Every tool is authoritative in
its role — use all of them together for a professional-grade shell project.

## Table of Contents

1. [shellcheck — Static Analysis](#shellcheck)
2. [shfmt — Formatter](#shfmt)
3. [bats-core — Test Runner](#bats-core)
4. [mise — Runtime & Task Runner](#mise)
5. [pre-commit Integration](#pre-commit)
6. [CI: GitHub Actions](#ci-github-actions)

---

## shellcheck — Static Analysis

Replaces: manual code review for quoting, scoping, and portability bugs.

```bash
# Install via mise (recommended)
mise use -g shellcheck

# Basic usage
shellcheck -x script.sh               # -x: follow sourced files

# Target a specific shell
shellcheck -s bash script.sh
shellcheck -s sh   posix-script.sh

# Target minimum bash version
shellcheck -s bash --shell=bash script.sh

# Multiple files / entire project
shellcheck -x bin/* lib/*.bash tests/**/*.bash

# Output formats
shellcheck -f gcc script.sh           # GCC-style (for editors)
shellcheck -f json script.sh          # JSON (for tooling)
```

### `.shellcheckrc`

Place at project root to avoid repeating flags:

```
shell=bash
enable=all
source-path=lib
source-path=SCRIPTDIR
```

`source-path=SCRIPTDIR` tells shellcheck to look for sourced files relative to
the script being checked — useful when scripts do `source "$(dirname "$0")/lib/helper.bash"`.

### Disabling Rules

Only disable rules that you understand and have consciously accepted:

```bash
# shellcheck disable=SC2034   # VAR is used indirectly (nameref, eval)
# shellcheck disable=SC2086   # word splitting intentional here

# Disable for a single line
cmd_with_intentional_splitting  # shellcheck disable=SC2086

# Disable for entire file (at top, AFTER shebang)
# shellcheck disable=SC1091    # sourced file is generated at runtime
```

### Key Rule Groups

| Rule(s) | What shellcheck catches |
|---|---|
| SC2086 | Unquoted variable: `$var` → `"$var"` |
| SC2046 | Unquoted command substitution: `$(cmd)` → `"$(cmd)"` |
| SC2006 | Backtick substitution: `` `cmd` `` → `$(cmd)` |
| SC2148 | Missing shebang |
| SC2064 | `trap "..."` with variables expanded at definition time |
| SC2155 | `local x=$(cmd)` masks exit code |
| SC2164 | `cd` without error check |
| SC2206 | Array assignment loses quoting |
| SC1090/1091 | Can't follow source (use `source-path`) |
| SC2034 | Variable assigned but never used |

---

## shfmt — Formatter

Replaces: manual style decisions, inconsistent indentation, tab/space debates.

```bash
# Install via mise (recommended)
mise use -g shfmt

# Format in-place
shfmt -i 2 -ci -bn -sr -w script.sh

# Check formatting (CI mode — exits non-zero if formatting differs)
shfmt -i 2 -ci -bn -sr -d script.sh

# Format entire project
shfmt -i 2 -ci -bn -sr -w bin/* lib/*.bash
```

### Flag Reference

| Flag | Meaning |
|---|---|
| `-i 2` | 2-space indent (prefer over tabs for readability) |
| `-ci` | Indent `case` branches (more readable) |
| `-bn` | Binary ops (`&&`, `\|\|`) at start of next line |
| `-sr` | Redirect operators followed by a space: `> file` not `>file` |
| `-w` | Write changes in place |
| `-d` | Diff mode: show changes without applying (CI use) |
| `-l` | List files that would be reformatted |

### Editor Integration

VS Code: install **shell-format** extension; point it to your `shfmt` binary.

`.editorconfig` (placed at project root, honoured by shfmt):

```ini
[*.{sh,bash,bats}]
indent_style = space
indent_size = 2
end_of_line = lf
charset = utf-8
trim_trailing_whitespace = true
insert_final_newline = true
```

---

## bats-core — Test Runner

Replaces: ad hoc manual testing, shell script unit tests written in bash.

See `references/testing.md` for full test patterns. Quick install:

```bash
# Install via mise (recommended)
mise use -g bats

# Run all tests
bats tests/

# Run a specific file
bats tests/test_my_script.bats

# Tap output (for CI parsers)
bats --tap tests/

# Parallel execution (bats 1.7+)
bats --jobs 4 tests/
```

### Helper Libraries (install as git submodules or bats plugins)

```bash
# bats-support: better failure output
git submodule add https://github.com/bats-core/bats-support tests/test_helper/bats-support

# bats-assert: assertion helpers
git submodule add https://github.com/bats-core/bats-assert tests/test_helper/bats-assert

# bats-file: filesystem assertions
git submodule add https://github.com/bats-core/bats-file   tests/test_helper/bats-file
```

Load in your tests:

```bash
# tests/test_helper.bash
load 'test_helper/bats-support/load'
load 'test_helper/bats-assert/load'
load 'test_helper/bats-file/load'
```

---

## mise — Runtime & Task Runner

Replaces: `make`, `Makefile`, manual tool version management.

Full `mise.toml` for a shell script project:

```toml
[tools]
shellcheck = "latest"
shfmt      = "latest"
bats       = "latest"
# Optionally pin versions for reproducibility:
# shellcheck = "0.10.0"

[env]
# Scripts can reference $MISE_PROJECT_ROOT for the project root

[tasks.lint]
description = "Run shellcheck on all scripts"
run         = "shellcheck -x bin/* lib/*.bash tests/**/*.bash"

[tasks.format]
description = "Format all scripts with shfmt"
run         = "shfmt -i 2 -ci -bn -sr -w bin/* lib/*.bash tests/**/*.bash"

[tasks."format:check"]
description = "Check formatting without modifying files (CI)"
run         = "shfmt -i 2 -ci -bn -sr -d bin/* lib/*.bash tests/**/*.bash"

[tasks.test]
description = "Run bats test suite"
run         = "bats --jobs 4 tests/"

[tasks.check]
description = "Full pre-push quality gate"
depends     = ["lint", "format:check", "test"]

[tasks."check:fix"]
description = "Fix all auto-fixable issues"
depends     = ["format"]
```

Run tasks:

```bash
mise run lint
mise run check
mise run format
```

---

## pre-commit Integration

```yaml
# .pre-commit-config.yaml
repos:
  - repo: https://github.com/shellcheck-py/shellcheck-py
    rev: v0.10.0.1
    hooks:
      - id: shellcheck
        args: [--severity=warning, -x]

  - repo: https://github.com/scop/pre-commit-shfmt
    rev: v3.10.0-1
    hooks:
      - id: shfmt
        args: [-i, "2", -ci, -bn, -sr, -w]

  - repo: https://github.com/pre-commit/pre-commit-hooks
    rev: v5.0.0
    hooks:
      - id: check-executables-have-shebangs
      - id: check-shebang-scripts-are-executable
      - id: end-of-file-fixer
      - id: trailing-whitespace
      - id: mixed-line-ending
        args: [--fix=lf]
```

---

## CI: GitHub Actions

```yaml
name: Shell CI

on:
  push:
    branches: [main]
  pull_request:

jobs:
  quality:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
        with:
          submodules: recursive        # for bats helper libraries

      - uses: jdx/mise-action@v2       # installs mise + project tools

      - name: Lint (shellcheck)
        run: mise run lint

      - name: Format check (shfmt)
        run: mise run format:check

      - name: Tests (bats)
        run: mise run test

  # Optional: test on multiple bash versions
  compat:
    runs-on: ubuntu-latest
    strategy:
      matrix:
        bash: ["4.4", "5.0", "5.2"]
    steps:
      - uses: actions/checkout@v4
      - name: Install bash ${{ matrix.bash }}
        run: |
          sudo apt-get install -y bash
          # or: use a container image with the target bash version
      - name: Run tests
        run: bats tests/
```
