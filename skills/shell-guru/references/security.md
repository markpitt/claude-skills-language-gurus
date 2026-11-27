# Security Reference

Shell script security: injection prevention, input validation, temp file
safety, secrets management, and privilege hygiene. Shell scripts are
particularly vulnerable to injection: treat all external input as hostile.

## Table of Contents

1. [Command Injection](#command-injection)
2. [Argument Injection](#argument-injection)
3. [Path Traversal & Filename Safety](#path-traversal--filename-safety)
4. [Temp File Safety](#temp-file-safety)
5. [Secrets Management](#secrets-management)
6. [Privilege & Environment Hygiene](#privilege--environment-hygiene)
7. [Input Validation](#input-validation)
8. [Output Safety](#output-safety)

---

## Command Injection

The #1 shell security vulnerability. **Never** construct commands from
untrusted data.

```bash
# BAD — command injection via eval
user_input="$(get_user_input)"
eval "process $user_input"           # attacker can inject: ; rm -rf /

# BAD — injection via bash -c with string interpolation
bash -c "echo $user_input"           # attacker: '$(rm -rf /)'

# BAD — injection via unquoted variable in command position
cmd=$("$user_choice")                # word splitting executes arbitrary code

# GOOD — pass as argument, not as shell code
process "$user_input"                # user_input is data, never re-evaluated

# GOOD — use arrays to build commands safely
declare -a cmd_args=("grep" "-r" "--" "$search_term" "$directory")
"${cmd_args[@]}"                     # each element is a distinct argument

# GOOD — validate before allowing in command context
if [[ "$action" == "start" || "$action" == "stop" || "$action" == "restart" ]]; then
  systemctl "$action" myservice
else
  err "Invalid action: $action"
  exit 1
fi
```

### `eval`: avoid almost always

`eval` re-parses its argument as shell code. Only use it when there is
**no other way** and input is fully controlled and validated.

```bash
# BAD — eval with any external data
eval "$(user_provided_string)"

# ACCEPTABLE — eval of known, static content (e.g. output of trusted tools)
eval "$(ssh-agent -s)"              # trusted: ssh-agent output is well-defined
eval "$(mise activate bash)"        # trusted: mise is a known tool

# Where eval seems necessary, use arrays instead
# BAD
eval "tar $tar_options -czf archive.tgz $files"

# GOOD
declare -a tar_cmd=("tar" "-czf" "archive.tgz")
tar_cmd+=($tar_options)             # still risky if tar_options is external
"${tar_cmd[@]}" "${files[@]}"       # pass arrays — no re-evaluation
```

---

## Argument Injection

External data passed as arguments to commands can manipulate option parsing.

```bash
# BAD — filename starting with - is interpreted as an option
rm $filename                    # if filename="-rf /", disaster

# GOOD — use -- to terminate option processing
rm -- "$filename"
grep -- "$pattern" "$file"
find . -- -name "$name"         # find doesn't support --, use -name instead

# GOOD — validate format before use
if [[ ! "$filename" =~ ^[a-zA-Z0-9._-]+$ ]]; then
  err "Invalid filename: $filename"
  exit 1
fi
rm -- "$filename"

# Passing arrays with -- to subcommands
run_cmd() {
  local cmd="$1"; shift
  "$cmd" -- "$@"
}
```

### Filename-safe operations with `find`

```bash
# Files with spaces or special chars in names — use -print0 + read -d ''
while IFS= read -r -d '' file; do
  process_file "$file"
done < <(find /path -name "*.txt" -print0)

# Or with xargs
find /path -name "*.txt" -print0 | xargs -0 -I{} process_file {}
```

---

## Path Traversal & Filename Safety

```bash
# BAD — path traversal: ../../etc/passwd
target_dir="/var/data"
user_path="$1"
cat "${target_dir}/${user_path}"     # can escape target_dir

# GOOD — realpath canonicalises and allows prefix validation
target_dir="/var/data"
user_path="$1"
resolved=$(realpath -m "${target_dir}/${user_path}")
if [[ "$resolved" != "${target_dir}/"* ]]; then
  err "Path traversal attempt detected"
  exit 1
fi
cat "$resolved"

# GOOD — reject dangerous path components early
if [[ "$user_path" == *".."* || "$user_path" == "/"* ]]; then
  err "Invalid path: $user_path"
  exit 1
fi

# GOOD — restrict allowed characters in filenames
if [[ ! "$filename" =~ ^[a-zA-Z0-9._-]+$ ]]; then
  err "Filename contains invalid characters"
  exit 1
fi
```

---

## Temp File Safety

Hardcoded temp filenames are vulnerable to symlink attacks and race conditions.

```bash
# BAD — predictable name, attackable by race condition + symlink
tmpfile="/tmp/myscript-$$.tmp"       # $$ is the PID — guessable

# BAD — world-readable for any duration
tmpfile="/tmp/output.txt"

# GOOD — mktemp creates uniquely named, mode 600 files
tmpfile=$(mktemp)                    # /tmp/tmp.XXXXXXXXXX
tmpdir=$(mktemp -d)                  # temporary directory

# GOOD — temp in script-owned directory
tmpfile=$(mktemp -t "myscript.XXXXXXXXXX")

# GOOD — always clean up via EXIT trap
cleanup() {
  local _exit=$?
  [[ -f "${tmpfile:-}" ]] && rm -f  "$tmpfile"
  [[ -d "${tmpdir:-}" ]]  && rm -rf "$tmpdir"
  exit "$_exit"
}
trap cleanup EXIT INT TERM

# GOOD — restrict permissions immediately
tmpfile=$(mktemp)
chmod 600 "$tmpfile"                 # only owner can read/write
```

---

## Secrets Management

```bash
# BAD — hardcoded in script (committed to version control)
API_KEY="sk-abc123"
DB_PASSWORD="hunter2"

# BAD — visible in process listing: ps aux shows arguments
curl -u "user:$PASSWORD" https://api.example.com

# BAD — exported to child processes unnecessarily
export API_KEY="$API_KEY"

# GOOD — read from environment variable
API_KEY="${API_KEY:?API_KEY environment variable must be set}"

# GOOD — read from restricted credential file (not env, for daemons)
if [[ -f /etc/myapp/credentials ]]; then
  # shellcheck source=/dev/null
  source /etc/myapp/credentials      # file should be chmod 600, owner root/app
fi

# GOOD — pass secret via stdin or file descriptor instead of argument
# Using stdin:
mysql --user="$DB_USER" --password < <(printf '%s\n' "$DB_PASSWORD") <<< "SELECT 1"

# Using a netstring fd (avoids env and args):
curl --header "Authorization: Bearer $(cat /run/secrets/api_token)" https://api.example.com

# GOOD — mask secrets from xtrace output
set_secret() {
  local secret="$1"
  { set +x; } 2>/dev/null           # turn off xtrace without logging the turn-off
  API_KEY="$secret"
  set -x
}

# GOOD — use password managers/vaults (e.g. pass, gopass, vault)
API_KEY=$(pass show myapp/api-key)

# Ensure temp files with secrets are cleaned up
secret_file=$(mktemp)
chmod 600 "$secret_file"
trap 'rm -f "$secret_file"' EXIT
printf '%s' "$SECRET" > "$secret_file"
```

---

## Privilege & Environment Hygiene

```bash
# Run with minimal privilege — never run a script as root unless required
# Check if accidentally running as root:
if [[ "$(id -u)" -eq 0 ]]; then
  err "Do not run this script as root"
  exit 1
fi

# Require root explicitly when needed:
require_root() {
  if [[ "$(id -u)" -ne 0 ]]; then
    err "This script must be run as root"
    exit 1
  fi
}

# Drop privileges after privileged operations
# Run specific commands with sudo instead of running entire script as root:
sudo systemctl restart myservice
sudo cp config.new /etc/myapp/config

# Sanitise and restrict PATH in security-critical scripts
# (prevents PATH hijacking)
export PATH="/usr/local/sbin:/usr/local/bin:/usr/sbin:/usr/bin:/sbin:/bin"

# Clear inherited environment for privilege-separated subcommands
env -i HOME="$HOME" USER="$USER" PATH="/usr/bin:/bin" cmd_to_run

# umask: restrict default permissions for files created by the script
umask 077    # only owner can read/write new files (no group/other perms)
```

---

## Input Validation

Validate external input at the boundary before using it anywhere:

```bash
# Validate integer
validate_integer() {
  local val="$1" name="${2:-value}"
  if [[ ! "$val" =~ ^-?[0-9]+$ ]]; then
    err "${name} must be an integer, got: ${val}"
    return 1
  fi
}

# Validate non-empty string with max length
validate_string() {
  local val="$1" name="${2:-value}" max_len="${3:-255}"
  if [[ -z "$val" ]]; then
    err "${name} must not be empty"
    return 1
  fi
  if (( ${#val} > max_len )); then
    err "${name} exceeds maximum length of ${max_len}"
    return 1
  fi
}

# Validate against allowlist
validate_env() {
  local val="$1"
  case "$val" in
    production|staging|development) ;;
    *) err "Invalid environment: $val (must be production, staging, or development)"
       return 1 ;;
  esac
}

# Validate file exists and is readable
validate_readable_file() {
  local file="$1"
  if [[ ! -f "$file" ]]; then
    err "File not found: $file"
    return 1
  fi
  if [[ ! -r "$file" ]]; then
    err "File not readable: $file"
    return 1
  fi
}
```

---

## Output Safety

```bash
# printf is preferable to echo — consistent across implementations
printf '%s\n' "$var"           # never interprets escape sequences in $var
printf '%b\n' "$text"          # explicit: interpret \n, \t etc.

# echo -e is non-portable (not all implementations support it)
# echo without -e won't interpret escapes (safe for most use)
echo "$plain_message"          # OK for static strings

# Log injection: don't blindly log external input
# BAD — attacker can inject fake log lines with newlines
info "Processing user input: $user_input"

# GOOD — sanitise or quote the input in log messages
info "Processing user input: $(printf '%q' "$user_input")"
# or: show length + hash instead of raw value
info "Processing ${#user_input} bytes of user input"

# HTML/URL output — shell doesn't have built-in encoding; use a tool
# For JSON output, use jq instead of string interpolation:
jq -n --arg name "$name" --arg value "$value" '{"name": $name, "value": $value}'
```
