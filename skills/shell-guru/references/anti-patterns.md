# Anti-Patterns Catalogue

Comprehensive list of shell scripting anti-patterns, each with a fix.
Most are caught by `shellcheck` — the relevant rule code is noted where applicable.

---

## Quoting Anti-Patterns

### Unquoted variable expansion (shellcheck SC2086)

```bash
# BAD — word splits and glob-expands $filename
cp $filename $destination

# BAD — if filename contains spaces, this breaks
echo Hello $name

# GOOD — always double-quote
cp "$filename" "$destination"
echo "Hello $name"
```

### Unquoted command substitution (shellcheck SC2046)

```bash
# BAD — output of cmd is word-split and glob-expanded
for f in $(find . -name "*.sh"); do ...

# GOOD — use while + read -d '' for filenames with spaces
while IFS= read -r -d '' f; do
  process "$f"
done < <(find . -name "*.sh" -print0)

# GOOD — for simple single-word outputs, quote explicitly
dir="$(dirname "$file")"
```

### Unquoted array expansion

```bash
# BAD — ${arr[*]} joins with IFS and word-splits: one string, not multiple args
cmd ${arr[*]}

# GOOD — "${arr[@]}" expands each element as a distinct word
cmd "${arr[@]}"

# Exception: joining with a delimiter
csv="${arr[*]}"    # IFS must be set to desired delimiter first
IFS=',' csv="${arr[*]}"
```

---

## Command Substitution Anti-Patterns

### Backtick substitution (shellcheck SC2006)

```bash
# BAD — deprecated syntax, poor nesting support
result=`cmd`
nested=`cmd1 \`cmd2\``    # backslash nesting is error-prone

# GOOD — $() is cleaner and nestable
result=$(cmd)
nested=$(cmd1 "$(cmd2)")
```

### Trailing newline stripping

```bash
# Command substitution strips trailing newlines — be aware
content=$(cat file)       # trailing newlines removed!

# GOOD — preserve trailing newlines with a suffix trick
content=$(cat file; printf x)
content="${content%x}"    # strip the suffix, not the newlines
```

---

## Test / Conditional Anti-Patterns

### `[ ]` instead of `[[ ]]` in Bash

```bash
# BAD — [ ] is POSIX test: no pattern matching, no &&/||, word splitting on vars
if [ $x == "hello world" ]; then    # word splits if x contains spaces

# BAD — && and || must be outside [ ]
if [ $a -eq 1 ] && [ $b -eq 2 ]; then   # two separate commands

# GOOD — [[ ]] is bash-native: no word splitting, supports &&/||, =~, patterns
if [[ "$x" == "hello world" ]]; then
if [[ "$a" -eq 1 && "$b" -eq 2 ]]; then

# EXCEPTION — use [ ] or test for POSIX sh portability
```

### Checking `$?` explicitly (shellcheck SC2181)

```bash
# BAD — unnecessary: $? is the exit code of the previous command
cmd
if [ $? -eq 0 ]; then echo "ok"; fi

# GOOD — use the command directly as the condition
if cmd; then echo "ok"; fi

# GOOD — capture result when needed later
if cmd; then
  some_result="$?"     # only if you need the code later
fi
```

### Arithmetic in `[ ]` vs `(( ))`

```bash
# BAD — string comparison operators used for numbers
if [ "$count" > "10" ]; then    # > is redirection here, not comparison!

# BAD — -gt in [ ] works but (( )) is cleaner
if [ "$count" -gt 10 ]; then

# GOOD — (( )) for arithmetic comparisons
if (( count > 10 )); then
if (( ${#arr[@]} == 0 )); then  # array length check
```

### `=~` right-hand side quoting

```bash
# BAD — quoting the RHS of =~ prevents pattern matching (treated as literal)
if [[ "$str" =~ "^[0-9]+" ]]; then    # shellcheck SC2076

# GOOD — unquoted regex on RHS
re='^[0-9]+'
if [[ "$str" =~ $re ]]; then    # variable: no quotes
if [[ "$str" =~ ^[0-9]+ ]]; then  # literal: no quotes
```

---

## Loop Anti-Patterns

### Parsing `ls` output (shellcheck SC2045)

```bash
# BAD — breaks on filenames with spaces, newlines, or globbing characters
for f in $(ls *.txt); do
  echo "$f"
done

# GOOD — use glob directly (nullglob handles zero matches)
shopt -s nullglob
for f in *.txt; do
  echo "$f"
done

# GOOD — find with -print0 for recursive or complex queries
while IFS= read -r -d '' f; do
  echo "$f"
done < <(find . -name "*.txt" -print0)
```

### `for` loop over command output with spaces

```bash
# BAD — word-splits on whitespace
IFS=$'\n'                        # still fragile with embedded newlines
for line in $(cat file.txt); do

# GOOD — while + read
while IFS= read -r line; do
  process "$line"
done < file.txt

# GOOD — mapfile (bash 4+), then loop over array
mapfile -t lines < file.txt
for line in "${lines[@]}"; do
  process "$line"
done
```

### Useless Use of Cat (UUOC)

```bash
# BAD — unnecessary subprocess
cat file.txt | grep "pattern"
cat file.txt | wc -l

# GOOD — redirect directly
grep "pattern" file.txt
wc -l < file.txt

# GOOD — when piping multiple files, cat is justified:
cat file1.txt file2.txt | sort
```

---

## Variable & Scope Anti-Patterns

### `local x=$(cmd)` masks exit code (shellcheck SC2155)

```bash
# BAD — local always exits 0, hiding failures from set -e
local result=$(parse_config "$file")    # SC2155

# GOOD — separate declaration from assignment
local result
result=$(parse_config "$file")

# Same applies to declare, export, readonly:
# BAD
readonly config=$(load_config)

# GOOD
config=$(load_config)
readonly config
```

### Uninitialized variables with `set -u`

```bash
# BAD — set -u exits when accessing unset variable
count=${count+1}      # nounset error if count is unset

# GOOD — initialise before use
declare -i count=0
(( count++ ))

# GOOD — safe default
count=$(( ${count:-0} + 1 ))
```

### Global state pollution from functions

```bash
# BAD — function modifies global without indication
process() {
  result="computed"    # silently changes global $result
}

# GOOD — use local; return via output or explicit global declaration
process() {
  local result="computed"
  printf '%s' "$result"    # caller captures: result=$(process)
}

# GOOD — where a global return value is necessary, be explicit
process() {
  declare -g PROCESS_RESULT="computed"    # explicit global
}
```

---

## Error Handling Anti-Patterns

### Ignoring `cd` failure (shellcheck SC2164)

```bash
# BAD — if cd fails, remaining commands run in the wrong directory
cd /some/dir
rm -rf *         # disaster if cd failed

# GOOD — always check cd
cd /some/dir || { err "Cannot cd to /some/dir"; exit 1; }
cd /some/dir || exit 1

# GOOD — use subshell to isolate directory change
(
  cd /some/dir
  rm -rf *
)  # parent directory is unchanged
```

### Missing `set -o pipefail`

```bash
# BAD — pipeline failure is hidden
set -e
false | cat     # exits 0! (cat succeeded, false is ignored)

# GOOD — catch any pipeline failure
set -Eeuo pipefail
false | cat     # now exits non-zero

# GOOD — allow a specific command in a pipeline to fail
{ false || true; } | cat    # explicitly allow false to fail
```

### Using `||` incorrectly with `set -e`

```bash
# BAD — set -e does NOT fire for commands in && / || lists (except the last)
set -e
parse_config || echo "config failed, continuing"   # parse_config failure ignored
do_something_else     # this runs even if parse_config critically failed

# GOOD — be explicit about which failures are acceptable
if ! parse_config; then
  err "Config parse failed"
  exit 1
fi
```

### Swallowing errors silently

```bash
# BAD — errors from cmd are silently discarded
cmd 2>/dev/null

# ACCEPTABLE — when failure of cmd is genuinely expected and non-critical
cmd 2>/dev/null || true

# GOOD — capture and log errors, don't just discard
if ! output=$(cmd 2>&1); then
  err "cmd failed: $output"
  exit 1
fi
```

---

## Process Substitution & Subshell Anti-Patterns

### Variable assignment in subshell not visible to parent

```bash
# BAD — a pipeline runs in a subshell; result is lost
count=0
cat file.txt | while read -r line; do
  (( count++ ))        # modifies subshell's count, not parent's
done
echo "$count"          # always prints 0

# GOOD — use process substitution (no subshell for the while loop)
count=0
while IFS= read -r line; do
  (( count++ ))
done < <(cat file.txt)     # < <() runs while in current shell
echo "$count"              # correct

# GOOD — use lastpipe shopt option (bash 4.2+)
shopt -s lastpipe
cat file.txt | while IFS= read -r line; do
  (( count++ ))
done                    # lastpipe: last cmd in pipeline runs in current shell
```

### `trap` strings with variables expanded too early (shellcheck SC2064)

```bash
# BAD — $tmpfile is expanded when trap is defined, not when it fires
tmpfile=$(mktemp)
trap "rm -f $tmpfile" EXIT     # SC2064: $tmpfile locked at trap definition

# GOOD — single quotes defer expansion to when the trap fires
tmpfile=$(mktemp)
trap 'rm -f "$tmpfile"' EXIT   # $tmpfile evaluated at exit time

# GOOD — use a cleanup function
cleanup() { rm -f "$tmpfile"; }
trap cleanup EXIT
```

---

## `printf` vs `echo` Anti-Patterns

### Using `echo` for non-literal output

```bash
# BAD — echo interprets -n, -e inconsistently across shells
echo -n "no newline"    # not portable: some echo implementations ignore -n

# BAD — echo interprets escape sequences with -e (non-portable)
echo -e "line1\nline2"

# GOOD — printf is consistent and portable
printf '%s' "no newline"
printf '%s\n' "$var"                    # one line
printf '%s\n' "line1" "line2"           # multiple lines
printf '%b\n' "escape\tsequences\nhere" # explicit escape interpretation

# GOOD — but echo is fine for simple literal strings
echo "Hello, world"
```

---

## String Operation Anti-Patterns

### `grep` / `sed` for simple string operations

```bash
# BAD — spawning a subprocess for a string check
if echo "$str" | grep -q "prefix"; then

# GOOD — use bash built-ins (faster, no subprocess)
if [[ "$str" == prefix* ]]; then
if [[ "$str" =~ ^prefix ]]; then

# BAD — sed for simple substitution
result=$(echo "$str" | sed 's/foo/bar/')

# GOOD — parameter expansion
result="${str/foo/bar}"        # first occurrence
result="${str//foo/bar}"       # all occurrences
result="${str#prefix}"         # strip shortest prefix match
result="${str##prefix*}"       # strip longest prefix match
result="${str%suffix}"         # strip shortest suffix match
result="${str%%suffix*}"       # strip longest suffix match
result="${str,,}"              # lowercase (bash 4+)
result="${str^^}"              # uppercase (bash 4+)
result="${#str}"               # string length
```

### Hardcoded script name

```bash
# BAD — breaks when script is renamed or symlinked
SCRIPT_NAME="my-script"

# GOOD — derive from $0 or BASH_SOURCE
SCRIPT_NAME="$(basename "${BASH_SOURCE[0]}")"
```

---

## Portability Anti-Patterns

### Bashisms in `#!/bin/sh` scripts

```bash
# BAD — these are bash-only features in a #!/bin/sh script
#!/bin/sh
[[ "$x" == "y" ]]    # [[ ]] is bash-only
arr=(a b c)          # arrays are bash-only
source lib.sh        # source is bash; use . in POSIX sh

# GOOD — in sh, stick to POSIX constructs
#!/bin/sh
[ "$x" = "y" ]       # POSIX [ ]
# no arrays in POSIX sh — use positional params or IFS-delimited strings
. ./lib.sh           # POSIX source
```

### `/bin/bash` shebang (hardcoded path)

```bash
# BAD — /bin/bash may not exist (e.g. NixOS, some Docker images)
#!/bin/bash

# GOOD — env lookup finds bash on PATH
#!/usr/bin/env bash

# Exception: #!/bin/bash is acceptable in Docker, where PATH is controlled
```

### Using `which` (not portable, may return wrong result)

```bash
# BAD — which may not be available; returns non-zero awkwardly
which python3

# GOOD — use command -v (POSIX)
if command -v python3 &>/dev/null; then
  python3 script.py
fi

# Or type -P for absolute path (bash)
python_bin=$(type -P python3) || { err "python3 not found"; exit 1; }
```
