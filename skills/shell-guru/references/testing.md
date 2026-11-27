# Testing Reference

Modern shell script testing with bats-core. No ad hoc manual testing: every
non-trivial script has automated tests that run in CI.

## Table of Contents

1. [bats-core Basics](#bats-core-basics)
2. [setup & teardown](#setup--teardown)
3. [Assertions with bats-assert](#assertions-with-bats-assert)
4. [Testing Scripts End-to-End](#testing-scripts-end-to-end)
5. [Mocking & Stubbing](#mocking--stubbing)
6. [Testing Library Functions](#testing-library-functions)
7. [File Assertions with bats-file](#file-assertions-with-bats-file)
8. [Parametrised Tests](#parametrised-tests)
9. [Test Organisation](#test-organisation)
10. [CI Integration](#ci-integration)

---

## bats-core Basics

Every `.bats` file is a valid Bash script. Tests are annotated with `@test`:

```bash
#!/usr/bin/env bats
# tests/test_my_script.bats

# Load helper libraries (if installed)
load 'test_helper/bats-support/load'
load 'test_helper/bats-assert/load'

@test "script exits 0 on success" {
  run ./bin/my-script --version
  assert_success
}

@test "script prints version string" {
  run ./bin/my-script --version
  assert_output --partial "1.0"
}

@test "script fails on missing required argument" {
  run ./bin/my-script
  assert_failure
  assert_output --partial "Usage:"
}
```

`run` executes a command and captures:
- `$status`: exit code
- `$output`: combined stdout + stderr
- `$lines`: array of output lines: `${lines[0]}`, `${lines[1]}`, etc.

Without bats-assert, use plain test expressions:

```bash
@test "exit code check" {
  run my_command
  [ "$status" -eq 0 ]
}

@test "output check" {
  run my_command
  [ "$output" = "expected output" ]
}

@test "output contains string" {
  run my_command
  [[ "$output" == *"expected"* ]]
}

@test "specific line" {
  run my_command
  [ "${lines[0]}" = "first line" ]
}
```

---

## setup & teardown

`setup` runs before **each** test. `teardown` runs after each test (even on failure).
`setup_file` / `teardown_file` run once per `.bats` file.

```bash
#!/usr/bin/env bats

setup_file() {
  # Runs once before all tests in this file
  export TEST_TMPDIR
  TEST_TMPDIR="$(mktemp -d)"
}

teardown_file() {
  # Runs once after all tests in this file
  rm -rf "$TEST_TMPDIR"
}

setup() {
  # Runs before each @test
  # Create per-test fixtures
  cp "${BATS_TEST_DIRNAME}/fixtures/config.yaml" "${TEST_TMPDIR}/config.yaml"
}

teardown() {
  # Runs after each @test (even if test failed)
  rm -f "${TEST_TMPDIR}/output.txt"
}

# Useful bats variables:
# $BATS_TEST_DIRNAME    — directory of the .bats file
# $BATS_TEST_NAME       — name of the current test
# $BATS_TMPDIR          — bats-managed temp dir (auto-cleaned per run)
# $BATS_TEST_NUMBER     — test number within the file
```

---

## Assertions with bats-assert

Install: `git submodule add https://github.com/bats-core/bats-assert tests/test_helper/bats-assert`

```bash
load 'test_helper/bats-support/load'   # required by bats-assert
load 'test_helper/bats-assert/load'

# Exit code
assert_success              # status -eq 0
assert_failure              # status -ne 0
assert_equal "$status" 2   # specific exit code

# Output matching
assert_output "exact match"
assert_output --partial "substring"
assert_output --regexp "regex.*pattern"
refute_output "should not appear"
refute_output --partial "not here"

# Specific lines
assert_line "line content"                   # any line matches
assert_line --index 0 "first line"           # specific line index
assert_line --partial "partial"              # partial line match
assert_line --regexp "^prefix"               # regex line match
refute_line "should not be a line"

# Variable equality
assert_equal "$actual" "$expected"
refute_equal "$a" "$b"

# Regular assertions
assert [ -f "$file" ]        # file exists
assert [ -n "$var" ]         # var non-empty
```

---

## Testing Scripts End-to-End

Test scripts as black boxes via `run`:

```bash
#!/usr/bin/env bats
load 'test_helper/bats-support/load'
load 'test_helper/bats-assert/load'

SCRIPT="${BATS_TEST_DIRNAME}/../bin/deploy"

@test "deploy requires --env flag" {
  run "$SCRIPT"
  assert_failure
  assert_output --partial "--env"
}

@test "deploy fails for unknown environment" {
  run "$SCRIPT" --env=invalid
  assert_failure
  assert_output --partial "Invalid environment"
}

@test "deploy succeeds for staging" {
  run "$SCRIPT" --env=staging --dry-run
  assert_success
  assert_output --partial "Deploying to staging"
}

@test "deploy output contains deployment ID" {
  run "$SCRIPT" --env=staging --dry-run
  assert_output --regexp "\[deploy-[0-9]+\]"
}
```

### Capturing stderr separately

By default, `run` captures stdout and stderr combined. For separate capture:

```bash
@test "error goes to stderr" {
  run bash -c '"$SCRIPT" --invalid 2>/tmp/stderr.txt; cat /tmp/stderr.txt'
  # Or use bats 1.5+ run --separate-stderr
  run --separate-stderr "$SCRIPT" --invalid
  assert_failure
  assert_stderr --partial "Error:"        # bats 1.5+
}
```

---

## Mocking & Stubbing

Replace external commands with controlled fakes using a temporary `PATH` prefix:

```bash
setup() {
  # Per-test mock directory prepended to PATH
  MOCK_BIN="$(mktemp -d)"
  export MOCK_BIN
  export PATH="${MOCK_BIN}:${PATH}"
}

teardown() {
  rm -rf "$MOCK_BIN"
}

# Create a mock that records its arguments
create_mock() {
  local cmd="$1"
  cat > "${MOCK_BIN}/${cmd}" << 'EOF'
#!/usr/bin/env bash
printf '%s\n' "$@" >> "${MOCK_BIN}/calls.log"
EOF
  chmod +x "${MOCK_BIN}/${cmd}"
}

# Create a mock with a specific exit code and output
create_mock_with_output() {
  local cmd="$1" output="$2" exit_code="${3:-0}"
  cat > "${MOCK_BIN}/${cmd}" << EOF
#!/usr/bin/env bash
printf '%s\n' "${output}"
exit ${exit_code}
EOF
  chmod +x "${MOCK_BIN}/${cmd}"
}

@test "script calls curl with correct URL" {
  create_mock curl
  run my-script --fetch https://example.com
  assert_success
  assert [ -f "${MOCK_BIN}/calls.log" ]
  grep -q "https://example.com" "${MOCK_BIN}/calls.log"
}

@test "script handles curl failure gracefully" {
  create_mock_with_output curl "" 1    # curl returns exit code 1
  run my-script --fetch https://example.com
  assert_failure
  assert_output --partial "Failed to fetch"
}
```

### Stubbing functions in sourced libraries

```bash
setup() {
  # Override a function after sourcing the library
  source lib/helpers.bash
  send_notification() { true; }       # stub: do nothing
  export -f send_notification
}

@test "process completes without sending real notifications" {
  run bash -c 'source lib/helpers.bash; source lib/process.bash; run_process'
  assert_success
}
```

---

## Testing Library Functions

Source the library directly and call functions:

```bash
#!/usr/bin/env bats
load 'test_helper/bats-support/load'
load 'test_helper/bats-assert/load'

setup() {
  # Source the library under test
  # shellcheck source=lib/string_utils.bash
  source "${BATS_TEST_DIRNAME}/../lib/string_utils.bash"
}

@test "trim_whitespace removes leading spaces" {
  run trim_whitespace "   hello"
  assert_output "hello"
}

@test "trim_whitespace removes trailing spaces" {
  run trim_whitespace "hello   "
  assert_output "hello"
}

@test "to_uppercase converts string" {
  run to_uppercase "hello world"
  assert_output "HELLO WORLD"
}

@test "join_by joins array elements" {
  run join_by "," "a" "b" "c"
  assert_output "a,b,c"
}
```

---

## File Assertions with bats-file

Install: `git submodule add https://github.com/bats-core/bats-file tests/test_helper/bats-file`

```bash
load 'test_helper/bats-support/load'
load 'test_helper/bats-assert/load'
load 'test_helper/bats-file/load'

@test "script creates output file" {
  run my-script --output "${BATS_TMPDIR}/out.txt"
  assert_success
  assert_file_exists "${BATS_TMPDIR}/out.txt"
}

@test "script creates non-empty output" {
  run my-script --output "${BATS_TMPDIR}/out.txt"
  assert_file_not_empty "${BATS_TMPDIR}/out.txt"
}

@test "script creates directory" {
  run my-script --init-project "${BATS_TMPDIR}/new-project"
  assert_dir_exists "${BATS_TMPDIR}/new-project"
}

@test "output file has correct permissions" {
  run my-script --output "${BATS_TMPDIR}/secret.txt"
  assert_file_permission 600 "${BATS_TMPDIR}/secret.txt"
}
```

---

## Parametrised Tests

bats-core doesn't have built-in parametrise, but there are clean patterns:

```bash
# Pattern 1: loop generating tests with eval (bats-core 1.5+)
for input in "" "   " $'\t'; do
  @test "is_blank returns true for: $(printf '%q' "$input")" {
    run is_blank "$input"
    assert_success
  }
done

# Pattern 2: helper function + multiple @test blocks (most portable)
check_extension_detected() {
  local file="$1" expected="$2"
  run get_extension "$file"
  assert_output "$expected"
}

@test "detects .sh extension"   { check_extension_detected "script.sh" "sh"; }
@test "detects .bash extension" { check_extension_detected "script.bash" "bash"; }
@test "detects no extension"    { check_extension_detected "Makefile" ""; }

# Pattern 3: data-driven with arrays
declare -a VALID_ENVS=("production" "staging" "development")
for env in "${VALID_ENVS[@]}"; do
  @test "validate_env accepts: $env" {
    run validate_env "$env"
    assert_success
  }
done
```

---

## Test Organisation

```
tests/
├── test_helper.bash              # shared setup: load bats-support/assert/file
├── unit/
│   ├── test_string_utils.bats    # lib/string_utils.bash
│   └── test_validators.bats      # lib/validators.bash
├── integration/
│   └── test_deploy.bats          # bin/deploy (mocked external commands)
├── e2e/
│   └── test_full_workflow.bats   # real network/filesystem (slow; CI only)
└── fixtures/
    ├── config.yaml
    └── sample_input.txt
```

`test_helper.bash`:

```bash
# tests/test_helper.bash
SCRIPT_DIR="$(cd "$(dirname "${BASH_SOURCE[0]}")" && pwd)"
load "${SCRIPT_DIR}/test_helper/bats-support/load"
load "${SCRIPT_DIR}/test_helper/bats-assert/load"
load "${SCRIPT_DIR}/test_helper/bats-file/load"

# Shared fixtures path
FIXTURES_DIR="${SCRIPT_DIR}/fixtures"
export FIXTURES_DIR
```

---

## CI Integration

```yaml
# .github/workflows/shell-ci.yml
jobs:
  test:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
        with:
          submodules: recursive        # bats helper libraries as submodules

      - uses: jdx/mise-action@v2       # installs shellcheck, shfmt, bats

      - name: Lint
        run: shellcheck -x bin/* lib/*.bash

      - name: Format check
        run: shfmt -i 2 -ci -bn -sr -d bin/* lib/*.bash

      - name: Unit tests
        run: bats tests/unit/

      - name: Integration tests
        run: bats tests/integration/

      # TAP output for test result reporting
      - name: Tests with TAP reporter
        run: bats --tap tests/ | tee test-results.tap

      # JUnit XML for test result upload (requires bats-junit reporter)
      - name: Tests with JUnit output
        run: bats --formatter junit tests/ > test-results.xml

      - name: Upload test results
        uses: actions/upload-artifact@v4
        if: always()
        with:
          name: test-results
          path: test-results.xml
```

### Common Anti-Patterns in Shell Tests

| Anti-pattern | Fix |
|---|---|
| `run cmd; echo "$output"` to debug | Use `assert_output` properly; use `bats --tap` for verbose output during development |
| Testing implementation details (checking temp filenames) | Test observable behaviour (outputs, exit codes, side effects) |
| No `setup`/`teardown`: shared state between tests | Isolate each test with per-test `setup` creating fresh fixtures |
| Hardcoded paths in tests | Use `$BATS_TEST_DIRNAME`, `$BATS_TMPDIR` |
| Tests that require network (in unit tests) | Mock external commands; separate e2e tests |
| Brittle full output match: `assert_output "line1\nline2\nline3"` | Match specific lines: `assert_line --index 0 "line1"` |
