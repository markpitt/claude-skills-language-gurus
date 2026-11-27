# Safety Reference

Modern Bash safety: set options, shopt flags, traps, variable scoping, and
argument parsing. These patterns eliminate the most common failure modes.

## Table of Contents

1. [The Safety Preamble](#the-safety-preamble)
2. [set Options Explained](#set-options-explained)
3. [shopt Options](#shopt-options)
4. [Trap Patterns](#trap-patterns)
5. [Variable Scoping & Declarations](#variable-scoping--declarations)
6. [The local x=$(cmd) Trap](#the-local-xcmd-trap)
7. [IFS Handling](#ifs-handling)
8. [Arithmetic Safety](#arithmetic-safety)
9. [Argument Parsing: getopts](#argument-parsing-getopts)
10. [Script Directory Detection](#script-directory-detection)

---

## The Safety Preamble

Every Bash script starts with this: no exceptions:

```bash
#!/usr/bin/env bash
set -Eeuo pipefail
shopt -s inherit_errexit
```

For scripts that process file lists or globs, also add:

```bash
shopt -s nullglob    # globs that match nothing expand to nothing (not the literal pattern)
shopt -s globstar    # ** matches recursively across directories
```

---

## set Options Explained

### `set -e` / `set -o errexit`

Exits the script immediately when a command returns a non-zero exit code.

**Edge cases** that do NOT trigger exit (the command is treated as a conditional):
- Commands in `if` tests: `if cmd; then`
- Commands in `while`/`until` conditions: `while cmd; do`
- Commands negated with `!`: `! cmd`
- Commands connected with `&&`/`||` except the last: `a && b`, where `a` failing does not exit
- The last command in a pipeline (pipefail covers this: see below)

```bash
# Safe pattern when you want to check the result yourself
if ! cmd; then
  err "cmd failed"
fi

# Intentionally allow failure
cmd || true

# Capture exit code without triggering errexit
cmd || rc=$?
```

### `set -u` / `set -o nounset`

Exits on reference to an unset variable.

```bash
# Always initialise variables before use
declare -i count=0
declare -a items=()           # empty array — avoids "unbound variable" on ${items[@]}
declare -A map=()             # empty associative array

# Default values instead of error
name="${USERNAME:-anonymous}"

# Mandatory variables (exit with message if unset or empty)
api_key="${API_KEY:?API_KEY must be set}"

# Test whether a variable is set without triggering nounset
[[ -v MY_VAR ]] && echo "set"   # -v: var set at all (even if empty)
[[ -n "${MY_VAR:-}" ]]          # safe: default to empty if unset
```

**Safe empty array expansion** with `set -u`:

```bash
declare -a args=()
# "${args[@]}" on an empty array is SAFE in bash 4.4+ — zero words, no error
# On older bash (< 4.4), use: "${args[@]+"${args[@]}"}"
```

### `set -o pipefail`

Pipeline exits with the status of the **last (rightmost) command that failed**,
or 0 if none did; without `pipefail` it is the last command's status alone:

```bash
# Without pipefail: the status is the last command's (cat, so 0)
# grep non-match returns 1 which DOES cause problems — use || true explicitly
false | grep 'x' | cat    # exit code 0 without pipefail — silently wrong!

# With pipefail: any failure in the pipeline is caught
set -o pipefail
false | cat               # now exits non-zero

# Allow known-failing pipes
{ cmd || true; } | process_output
```

**No match is a failure to grep.** `grep` exits 1 when nothing matches, and
`grep -c` prints 0 and still exits 1. Under `set -e`, `n=$(grep -c PATTERN file)`
aborts the script when nothing matches, so a check that counts what survives a
cleanup aborts exactly when the cleanup worked. Under `set -o pipefail`,
`n=$(grep -o PATTERN file | wc -l)` aborts for the same reason; without
`pipefail` it does not. Absorb the miss at the grep:
`n=$( { grep -o PATTERN file || true; } | wc -l )`. A dry run where every
pattern matches never runs this branch, so test the zero-match case.

**An early-exit reader breaks the producer.** Under `set -o pipefail`,
`producer | grep -q PATTERN` returns 141 (128 plus SIGPIPE) when the pattern is
found before the producer has finished writing, so a hit reads as a failure. It
shows only when the producer writes more than the pipe buffer holds: 5 lines
returned 0, 2,000,000 lines returned 141, and a miss returned 1. Sending a large
variable through `printf '%s' "$text" | grep -q` fails the same way. Remedies
that returned 0 on a match: read all the input (`grep PATTERN >/dev/null`, or
`grep -c`), or avoid the pipe (`grep -q PATTERN <<<"$text"`, or a `case`
pattern for a fixed string). Braces around `grep -q` change nothing (141).

### `set -E` / `set -o errtrace`

The `ERR` trap is inherited by functions, command substitutions, and subshells.
`-E` is included via `set -E` in the compound flag `-Eeuo`.

```bash
set -E
trap 'on_error "$LINENO"' ERR

on_error() {
  err "Failed at line $1: ${BASH_COMMAND}"
}

fail_deep() {
  false  # ERR trap fires here, then propagates up
}
fail_deep  # ERR trap fires for fail_deep itself too
```

---

## shopt Options

```bash
# Subshells (command substitution, () groups) inherit errexit
shopt -s inherit_errexit   # Bash 4.4+

# Globs that match nothing become empty (0 words) instead of literal pattern
shopt -s nullglob

# Globs that match nothing are a fatal error (alternative to nullglob)
shopt -s failglob

# ** matches recursively: for f in **/*.sh
shopt -s globstar

# case-insensitive globbing
shopt -s nocaseglob

# Extended patterns: +(pattern), *(pattern), ?(pattern), @(pattern), !(pattern)
shopt -s extglob
```

### nullglob vs failglob

Use **nullglob** when iterating over files that might not exist yet:

```bash
shopt -s nullglob
for f in logs/*.log; do
  process "$f"            # loop body simply doesn't run if no .log files exist
done
```

Use **failglob** in safety-critical scripts where a missing match is an error.

---

## Trap Patterns

### Standard cleanup + error trap

```bash
tmpfile=""

cleanup() {
  local _exit=$?
  # Remove temp files; || true so cleanup doesn't itself fail
  [[ -f "${tmpfile:-}" ]] && rm -f "$tmpfile" || true
  exit "$_exit"           # preserve original exit code
}

on_error() {
  local line_no="$1"
  err "Unexpected failure at line ${line_no}: ${BASH_COMMAND}"
  # Stack trace (bash 4+)
  local i
  for (( i=1; i<${#BASH_LINENO[@]}; i++ )); do
    printf '  #%d  %s:%s  in  %s\n' \
      "$i" "${BASH_SOURCE[$i]:-?}" "${BASH_LINENO[$i]:-?}" "${FUNCNAME[$i]:-main}" >&2
  done
}

trap cleanup  EXIT
trap cleanup  INT TERM     # also call cleanup on Ctrl-C or kill
trap 'on_error "$LINENO"' ERR
```

### Temporary directory

```bash
tmpdir=""
cleanup() {
  local _exit=$?
  [[ -d "${tmpdir:-}" ]] && rm -rf "$tmpdir"
  exit "$_exit"
}
trap cleanup EXIT INT TERM

tmpdir=$(mktemp -d)        # unique dir guaranteed by the OS
```

### Reset ERR trap inside functions that may legitimately fail

```bash
check_optional_tool() {
  # This function may return 1 — don't want ERR trap firing
  trap - ERR
  if ! which optional-tool &>/dev/null; then
    trap 'on_error "$LINENO"' ERR
    return 1
  fi
  trap 'on_error "$LINENO"' ERR
  return 0
}
```

---

## Variable Scoping & Declarations

```bash
# Constants — always readonly at the top of a script
readonly MAX_RETRIES=3
readonly SCRIPT_NAME="${BASH_SOURCE[0]##*/}"

# Integer variables — arithmetic without $ prefix
declare -i retry_count=0
(( retry_count++ ))         # no $ needed inside (( ))

# Arrays
declare -a files=()
files+=("one.txt" "two.txt")

# Associative arrays (bash 4+)
declare -A config=()
config[host]="localhost"
config[port]="5432"

# Nameref (bash 4.3+) — pass variable names to functions
update_array() {
  declare -n _arr="$1"      # -n: nameref
  _arr+=("new_item")
}
declare -a my_list=()
update_array my_list

# Global from inside function
set_global() {
  declare -g GLOBAL_RESULT="computed"
}
```

### Function-local variables

Always declare all variables inside a function with `local`:

```bash
process_file() {
  local file="$1"
  local -i line_count=0
  local -a parts=()
  local output

  output=$(wc -l < "$file")
  line_count="$output"
  IFS='/' read -ra parts <<< "$file"
}
```

---

## The `local x=$(cmd)` Trap

**Critical bug**: `local` always exits with 0, masking the exit code of `$(cmd)`:

```bash
# BAD — set -e will NOT catch the failure of false here
bad_function() {
  local result=$(false)   # local returns 0 even though false returned 1!
  echo "This still runs"  # this executes when it shouldn't
}

# GOOD — separate declaration from assignment
good_function() {
  local result
  result=$(false)         # now the exit code of false propagates correctly
  echo "This won't run if false fails (with set -e)"
}
```

The same applies to `declare`, `export`, and `readonly`:

```bash
# BAD
readonly config=$(load_config)    # shellcheck SC2155

# GOOD
config=$(load_config)
readonly config
```

---

## IFS Handling

The Internal Field Separator controls word splitting. Modify it locally: never
change the global IFS permanently without restoring it.

```bash
# Read CSV line into array
IFS=',' read -ra fields <<< "$csv_line"

# Process filename list with newlines as separator (safe for spaces in names)
while IFS= read -r -d $'\0' file; do
  process "$file"
done < <(find . -name "*.sh" -print0)

# Temporarily change IFS in a subshell (safest — auto-restored)
(
  IFS=':'
  read -ra path_parts <<< "$PATH"
  printf '%s\n' "${path_parts[@]}"
)

# Or save-restore pattern
old_IFS="$IFS"
IFS=':'
read -ra path_parts <<< "$PATH"
IFS="$old_IFS"
```

Reading a file line-by-line (correct pattern: handles lines without trailing newline):

```bash
while IFS= read -r line || [[ -n "$line" ]]; do
  process "$line"
done < input.txt
```

Reading all lines into an array:

```bash
mapfile -t lines < input.txt     # bash 4+
# or
readarray -t lines < input.txt   # alias for mapfile
```

---

## Arithmetic Safety

```bash
# (( expr )) returns 1 when expr evaluates to 0 — triggers set -e!
declare -i count=0
(( count++ ))          # BAD when set -e is active and count starts at 0

# Safe patterns when count might be 0:
(( count++ )) || true              # explicit allow
(( ++count ))                      # pre-increment: result is always >= 1 if count was 0
count=$(( count + 1 ))             # arithmetic expansion: exit code always 0
(( count += 1 ))                   # also safe — same as above inside (( ))

# Correct way to check arithmetic results with set -e
if (( count > 0 )); then ...       # safe: (( )) in if context doesn't trigger exit
[[ $count -gt 0 ]]                 # also safe

# Integer validation
if [[ "$input" =~ ^[0-9]+$ ]]; then
  declare -i validated_num="$input"
fi
```

---

## Argument Parsing: getopts

Use `getopts` for POSIX-portable option parsing (no long options):

```bash
usage() {
  printf 'Usage: %s [-v] [-h] [-f file] <args>\n' "$(basename "$0")" >&2
  printf '  -v        Verbose output\n' >&2
  printf '  -f file   Input file\n' >&2
  printf '  -h        Show this help\n' >&2
}

declare -i verbose=0
file=""

while getopts ':vhf:' opt; do
  case "$opt" in
    v) verbose=1 ;;
    h) usage; exit 0 ;;
    f) file="$OPTARG" ;;
    :) err "Option -${OPTARG} requires an argument"; usage; exit 1 ;;
    \?) err "Unknown option: -${OPTARG}"; usage; exit 1 ;;
  esac
done
shift $(( OPTIND - 1 ))
# Remaining positional args are in "$@"
```

For **long option** support, use manual parsing (portable, no GNU `getopt`):

```bash
while [[ $# -gt 0 ]]; do
  case "$1" in
    -v|--verbose)         verbose=1; shift ;;
    -f|--file)            file="$2"; shift 2 ;;
    --file=*)             file="${1#--file=}"; shift ;;
    --)                   shift; break ;;
    -*)                   err "Unknown option: $1"; exit 1 ;;
    *)                    break ;;           # first non-option arg
  esac
done
```

---

## Script Directory Detection

To source sibling files reliably regardless of where the script is invoked from:

```bash
# Correct: resolves symlinks and relative paths
SCRIPT_DIR="$(cd "$(dirname "${BASH_SOURCE[0]}")" && pwd)"
readonly SCRIPT_DIR

# Source a library relative to the script
# shellcheck source=lib/helpers.bash
source "${SCRIPT_DIR}/../lib/helpers.bash"
```

Do **not** use `$0` alone: it reflects how the script was invoked, not where it lives.
Do **not** use `dirname "$0"` without `cd && pwd`: it gives a relative path.
