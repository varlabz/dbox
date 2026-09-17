# Bash Coding Best Practices

This guide outlines the best practices for writing clean, robust, and safe Bash scripts within the `dbox` ecosystem and general automation/agent tasks.

---

## 1. Safety First: The Three Pillars of Robust Bash

Always start your scripts with configuration flags that prevent silent failures:

```bash
set -euo pipefail
```

* **`set -e`**: Exit immediately if any command exits with a non-zero status.
* **`set -u`**: Exit immediately if an uninitialized variable is referenced.
* **`set -o pipefail`**: Prevent pipelines from masking errors (the exit status of the pipe will be the status of the last command to exit with a non-zero status).

### Exception: Working with Arrays under `set -u`
In Bash, referencing an empty array when `set -u` is active can trigger an "unbound variable" error. To reference arrays safely, use the following syntax:

```bash
# Safe expansion of an array that may be empty or unset
for item in ${MY_ARRAY[@]+"${MY_ARRAY[@]}"}; do
    echo "$item"
done
```

---

## 2. Quoting and Variable Expansion

Unquoted variables are subject to word splitting and globbing, leading to unexpected behavior and security bugs.

* **Rule**: **Always quote your variables** unless you explicitly want word splitting.
* **Rule**: Use double quotes `"` for strings containing variables or subshells. Use single quotes `'` for literal strings.

```bash
# BAD: Will fail if directory contains spaces
mkdir $TARGET_DIR

# GOOD: Safe and robust
mkdir "$TARGET_DIR"
```

---

## 3. Arrays for Arguments and Lists

Never store lists of arguments or options as a space-separated string. Use Bash arrays.

```bash
# BAD: Hard to handle arguments with spaces
DOCKER_ARGS="-v /tmp:/tmp -e VAR=\"value with spaces\""
docker run $DOCKER_ARGS alpine

# GOOD: Safe argument handling
docker_args=(
    -v "/tmp:/tmp"
    -e "VAR=value with spaces"
)
docker run "${docker_args[@]}" alpine
```

---

## 4. Conditional Tests and Comparisons

Use double brackets `[[ ... ]]` instead of single `[ ... ]` or `test`. Double brackets are a Bash builtin and are safer because they don't perform word splitting or glob expansion on variables.

```bash
# BAD
if [ $name = "admin" ]; then ... fi

# GOOD
if [[ "$name" == "admin" ]]; then ... fi
```

### String vs. Integer Comparison
* **Strings**: Use `==` and `!=`.
* **Integers**: Use `-eq`, `-ne`, `-lt`, `-le`, `-gt`, `-ge` or use arithmetic evaluation `(( ... ))`.

```bash
# String check
if [[ "$status" == "running" ]]; then ... fi

# Integer check (arithmetic evaluation)
if (( count > 5 )); then ... fi
```

---

## 5. Scope and Functions

Structure your code using functions to make it modular and easy to test.

* **Rule**: Always declare variables inside functions with the `local` keyword to prevent modifying global state.
* **Rule**: Use lowercase names for local and private variables. Keep uppercase names for environment variables and configuration constants.

```bash
# GOOD
calculate_sum() {
    local first="$1"
    local second="$2"
    local result=$((first + second))
    echo "$result"
}
```

---

## 6. Output and Logging

Use `printf` instead of `echo` for structured or formatted output. `echo` behavior varies across shell versions and operating systems (e.g., handling of `-n` or escape sequences).

```bash
# BAD: Non-portable and unpredictable formatting
echo -n "Installing... "

# GOOD: Predictable behavior
printf "Installing... "
```

Redirect error messages to standard error (`stderr`):

```bash
log_error() {
    printf "Error: %s\n" "$1" >&2
}
```

---

## 7. Cleanups with `trap`

If your script creates temporary files or starts background processes/Docker containers, use `trap` to ensure they are cleaned up even if the script crashes or is terminated.

```bash
cleanup() {
    printf "Cleaning up temporary files...\n"
    rm -f "$TEMP_FILE"
}

# Run cleanup on script exit, interruption, or termination
trap cleanup EXIT INT TERM

TEMP_FILE=$(mktemp)
```

---

## 8. ShellCheck and Linting

Always run [ShellCheck](https://www.shellcheck.net/) on your scripts before committing. ShellCheck is a static analysis tool that detects common bugs, style issues, and security vulnerabilities.

If you must bypass a ShellCheck rule, use a directive:

```bash
# shellcheck disable=SC2086
docker run $UNQUOTED_PARAMS alpine
```
