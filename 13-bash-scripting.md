# Bash Scripting Cheat Sheet

> Bash scripting for DevOps — variables, conditionals, loops, functions, error handling, and production-ready snippets.

---

## Table of Contents

- [Script Basics](#1-script-basics)
- [Variables & Special Parameters](#2-variables--special-parameters)
- [String & Array Operations](#3-string--array-operations)
- [Conditionals](#4-conditionals)
- [Loops](#5-loops)
- [Functions](#6-functions)
- [Input & Output](#7-input--output)
- [Error Handling & Set Options](#8-error-handling--set-options)
- [Trap & Cleanup](#9-trap--cleanup)
- [Practical Snippets](#10-practical-snippets)
- [Style Checklist](#11-style-checklist)

---

## 1. Script Basics

```bash
#!/usr/bin/env bash
# Always start with a shebang (env > hardcoded path for portability)

set -euo pipefail
# -e  exit on first error
# -u  treat unset variables as errors
# -o pipefail  pipeline fails if ANY command fails (not just the last)

readonly SCRIPT_DIR="$(cd "$(dirname "${BASH_SOURCE[0]}")" && pwd)"
# Reliable way to get the script's own directory
```

**Run modes:** `bash script.sh` / `chmod +x script.sh && ./script.sh` / `source script.sh` (runs in current shell).

## 2. Variables & Special Parameters

```bash
NAME="value"                 # NO spaces around =
readonly VERSION="1.0.0"     # Cannot be reassigned
declare -i COUNT=0           # Integer type
declare -r API_URL="https://api.example.com"

echo "$NAME"                 # Always quote variables!
echo "${NAME}"               # Braces: required for ${NAME}_suffix

# Special parameters
$0        # Script name
$1..$9    # Positional args (use ${10} beyond 9)
$#        # Number of args
$@        # All args as separate words ("$@" — quoted for correctness)
$*        # All args as one word
$?        # Exit code of last command
$$        # PID of script
$!        # PID of last background job
$_        # Last argument of previous command

shift                      # Drop $1, promote the rest ($2 -> $1)
shift 2                    # Drop two

# Defaults & substitutions
"${VAR:-default}"          # Default if unset/empty (no assignment)
"${VAR:=default}"          # Default AND assigns if unset
"${VAR:?error message}"    # Fail with message if unset
"${VAR:+alternative}"      # Use alternative if VAR is set
```

## 3. String & Array Operations

```bash
STR="hello world"
${#STR}                    # Length: 11
${STR:6}                   # Substring from index 6: "world"
${STR:0:5}                 # First 5 chars: "hello"
${STR/world/bash}          # Replace first match: "hello bash"
${STR//l/L}                # Replace all: "heLLo worLd"
${STR#*o}                  # Remove shortest prefix up to 'o': " world"
${STR##*o}                 # Remove longest prefix up to last 'o': "rld"
${STR%o*}                  # Remove shortest suffix from last... : "hell"
${STR^^}                   # Uppercase
${STR,,}                   # Lowercase

# Arrays
FRUITS=("apple" "banana" "cherry")
echo "${FRUITS[0]}"        # apple
echo "${FRUITS[@]}"        # all elements
echo "${#FRUITS[@]}"       # count: 3
echo "${!FRUITS[@]}"       # indices: 0 1 2
FRUITS+=("date")           # append
for f in "${FRUITS[@]}"; do echo "$f"; done

# Command substitution & arithmetic
TODAY="$(date +%F)"        # capture output
FILES=$(ls *.log 2>/dev/null || true)
RESULT=$(( 4 + 2 * 3 ))    # 10
(( COUNT++ ))              # increment (arithmetic context)
(( X > 5 )) && echo "big"

# Process substitution
diff <(sort file1.txt) <(sort file2.txt)
```

## 4. Conditionals

```bash
# File tests
[ -f /etc/passwd ]         # file exists
[ -d /var/log ]            # directory exists
[ -s file.txt ]            # exists and non-empty
[ -x script.sh ]           # executable
[ -w file ]                # writable
[ file1 -nt file2 ]        # newer than

# String tests
[ -z "$STR" ]              # empty
[ -n "$STR" ]              # non-empty
[ "$A" = "$B" ]            # equal (spaces around =)
[ "$A" != "$B" ]
[[ "$STR" == *"pattern"* ]]        # glob match (bash only)
[[ "$STR" =~ ^[0-9]{4}$ ]]         # regex match (bash only)

# Numeric tests
[ "$COUNT" -eq 0 ]   -ne   -lt   -le   -gt   -ge
(( COUNT == 0 && MAX > 5 ))

# if / elif / else
if [[ -f "$CONFIG" ]]; then
    source "$CONFIG"
elif [[ -n "${ENV:-}" ]]; then
    echo "Using env $ENV"
else
    echo "No configuration found" >&2
    exit 1
fi

# case — clean arg parsing
case "${1:-}" in
    start)   start_app ;;
    stop)    stop_app ;;
    restart) stop_app; start_app ;;
    -h|--help|help)
        echo "Usage: $0 {start|stop|restart}"
        exit 0
        ;;
    *)
        echo "Unknown command: ${1:-}" >&2
        exit 64   # EX_USAGE
        ;;
esac

# Ternary-style & short circuits
[[ "$ENV" == "prod" ]] && DRY_RUN=false || DRY_RUN=true
[ -z "$VAR" ] && { echo "VAR required"; exit 1; }
```

## 5. Loops

```bash
# for — lists & ranges
for f in *.log; do
    gzip "$f"
done

for i in {1..10}; do echo "$i"; done
for i in $(seq 1 2 10); do echo "$i"; done   # step 2

for server in web1 web2 db1; do
    ssh "deploy@$server" "systemctl restart app"
done

# C-style
for (( i=0; i<${#ARRAY[@]}; i++ )); do
    echo "$i: ${ARRAY[$i]}"
done

# while — read files line by line (safe for spaces)
while IFS= read -r line; do
    echo "$line"
done < input.txt

# read output of a command
docker ps --format '{{.Names}}' | while read -r name; do
    echo "Container: $name"
done

# until — retry pattern
until curl -sf http://localhost:8080/health; do
    echo "Waiting for app..."; sleep 2
done

# break / continue
for i in {1..100}; do
    [ $(( i % 2 )) -eq 0 ] && continue
    [ $i -gt 9 ] && break
    echo "$i"
done
```

## 6. Functions

```bash
log() {
    local level="$1"; shift
    printf '%s [%s] %s\n' "$(date '+%Y-%m-%d %H:%M:%S')" "$level" "$*" >&2
}
log INFO "Starting deployment"          # call: name args (no parens!)

die() { log ERROR "$*"; exit 1; }

# With return value via output capture
urlencode() { python3 -c 'import urllib.parse,sys; print(urllib.parse.quote(sys.argv[1]))' "$1"; }
ENCODED="$(urlencode "hello world")"

# Return codes (0-255)
is_running() { pgrep -f "$1" >/dev/null; }
if is_running "myapp"; then
    echo "running"
else
    echo "stopped"
fi

# Local variables are function-scoped
process() {
    local file="$1" count=0
    # count changes here don't leak outside
    (( count++ ))
    echo "$count"
}
```

## 7. Input & Output

```bash
# Reading input
read -p "Continue? (y/n): " -n 1 -r; echo
[[ $REPLY =~ ^[Yy]$ ]] || exit 0

read -r -s -p "Password: " PASSWORD; echo        # silent input
read -r -a PARTS <<< "one two three"             # split into array
IFS=',' read -r COL1 COL2 COL3 <<< "a,b,c"       # split string by delimiter

# Output
echo "plain"
printf '%-20s %10s\n' "Name" "Count"            # formatted columns
echo "to stderr" >&2
{ echo "grouped"; echo "output"; } > file.txt

# Heredocs
cat <<EOF > config.yml
app:
  env: $ENV
  port: 8080
EOF

cat <<'EOF' > literal.sh                    # quoted delimiter = NO expansion
echo "\$1 will NOT expand"
EOF

tee /etc/app.conf > /dev/null <<EOF         # sudo + heredoc
setting=value
EOF
```

## 8. Error Handling & Set Options

```bash
set -euo pipefail
set -x                     # Debug: print each command (or use bash -x script.sh)
set +x                     # Turn off

# Capture error context
if ! OUTPUT=$(some_command 2>&1); then
    log ERROR "Command failed: $OUTPUT"
    exit 1
fi

# Explicit exit codes
exit 0                     # success
exit 1                     # generic error
# Standard codes: 64 usage | 65 data error | 126 not executable | 127 command not found | 130 SIGINT | 137 SIGKILL

# Custom trap on ERR
trap 'log ERROR "Failed at line $LINENO: $BASH_COMMAND"' ERR

# Verbose/quiet modes via flags
VERBOSE=false
while [[ $# -gt 0 ]]; do
    case "$1" in
        -v|--verbose) VERBOSE=true; shift ;;
        -n|--dry-run) DRY_RUN=true; shift ;;
        *) die "Unknown option: $1" ;;
    esac
done
$VERBOSE && set -x
$DRY_RUN && log INFO "DRY RUN — no changes will be made"
```

## 9. Trap & Cleanup

```bash
#!/usr/bin/env bash
set -euo pipefail

TMPDIR_WORK="$(mktemp -d)"
trap 'rm -rf "$TMPDIR_WORK"' EXIT          # always cleanup

cleanup() {
    local code=$?
    echo "Exiting with code $code"
    # stop background jobs, remove locks, etc.
}
trap cleanup EXIT                          # EXIT runs on any exit path
trap 'echo "Interrupted"; exit 130' INT TERM

# Acquire a lock (prevent concurrent runs)
exec 200>"/tmp/myapp.lock"
flock -n 200 || { echo "Another instance is running"; exit 1; }
```

## 10. Practical Snippets

```bash
# --- Retry with backoff ---
retry() {
    local attempts=5 delay=2 n=0
    until "$@"; do
        (( n++ ))
        [ $n -ge $attempts ] && return 1
        log WARN "Attempt $n failed, retrying in ${delay}s..."
        sleep "$delay"; delay=$(( delay * 2 ))
    done
}
retry curl -sf https://api.example.com/health

# --- Backup with rotation ---
backup() {
    local src="$1" dest="$2"
    local stamp; stamp="$(date +%Y%m%d_%H%M%S)"
    tar -czf "$dest/backup_${stamp}.tar.gz" -C "$(dirname "$src")" "$(basename "$src")"
    # Keep last 7
    ls -1t "$dest"/backup_*.tar.gz | tail -n +8 | xargs -r rm -f
}

# --- Parallel SSH with xargs ---
cat servers.txt | xargs -P5 -I{} ssh -o ConnectTimeout=5 deploy@{} 'uptime'

# --- Health check then deploy ---
if curl -sf -o /dev/null -w '%{http_code}' http://staging/health | grep -q 200; then
    ./deploy.sh prod
else
    die "Staging health check failed — aborting deploy"
fi

# --- Wait for pod ready (kubectl) ---
kubectl wait --for=condition=ready pod -l app=myapp --timeout=300s -n prod

# --- Compare versions (sort -V = version sort) ---
if [ "$(printf '%s\n' "$REQUIRED" "$CURRENT" | sort -V | head -n1)" != "$REQUIRED" ]; then
    echo "Upgrade required: $CURRENT -> $REQUIRED"
fi
```

## 11. Style Checklist

- [ ] Shebang `#!/usr/bin/env bash` on line 1
- [ ] `set -euo pipefail` near the top
- [ ] Quote all variables: `"$VAR"` (never bare `$VAR`)
- [ ] `local` inside functions
- [ ] UPPER_CASE for constants/env, lower_case for locals
- [ ] Meaningful exit codes; `die()` helper for fatal errors
- [ ] Trap-based cleanup for temp files/locks
- [ ] Usage message for scripts with args (`-h` flag)
- [ ] Log to stderr (`>&2`); reserve stdout for real output
- [ ] Test with `shellcheck script.sh` and `bash -n script.sh` (syntax)

```bash
# shellcheck — catch bugs before running
shellcheck deploy.sh
```

---
