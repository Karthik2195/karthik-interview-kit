# Bash - Senior Level Interview Questions & Answers

**Contributed by:** Karthik Reddy

## Table of Contents
1. Advanced Script Optimization
2. Error Handling & Debugging
3. Performance & Scalability
4. Security Best Practices
5. Complex Real-World Scenarios

---

## 1. Advanced Script Optimization

### Q (High-Level): I wrote a Bash script to process 1 million log lines. It took 10 hours. How do I make it faster?

**A (Real-Time Performance):**
Your log processing script is way too slow:
```bash
# Your current script (takes 10 HOURS!)
for line in $(cat huge.log); do
  if [[ $line == *"ERROR"* ]]; then
    echo "$line" >> errors.log
  fi
done
```

**Why it's slow:**
```
- for loop processes 1,000,000 lines
- Each line spawns subprocess = 1 million processes!
- I/O overhead = massive
= 10 hours ❌
```

**Fast solution (uses single process):**
```bash
# ✅ Fast (takes 10 SECONDS!)
grep "ERROR" huge.log > errors.log

# Why:
# - grep is compiled (not interpreted Bash)
# - Single process, not 1 million
# - Optimized for this exact task
```

**Real-time result:**
- Before: 10 hours (unusable)
- After: 10 seconds (instant)
= 3,600x faster! 🚀

---

### Q (High-Level): My production script failed silently. I had no idea it broke. How do I prevent this?

**A (Silent Failure Nightmare):**
```bash
#!/bin/bash

# ❌ BAD: If database backup fails, script continues silently
tar -czf /backup/db.tar.gz /data/database
cp /backup/db.tar.gz /mnt/backup-disk/

# ❌ What if /mnt/backup-disk was unmounted?
# - cp fails silently
# - Backup never happens
# - Nobody knows
# - 3 weeks later, disk dies, data lost, RIP ❌
```

**Production-ready fix:**
```bash
#!/bin/bash
set -euo pipefail  # Exit on ANY error

tar -czf /backup/db.tar.gz /data/database || {
  echo "ERROR: Database backup failed!" >&2
  mail -s "BACKUP FAILED" ops@company.com
  exit 1
}

cp /backup/db.tar.gz /mnt/backup-disk/ || {
  echo "ERROR: Backup copy failed!" >&2
  mail -s "BACKUP FAILED" ops@company.com
  exit 1
}

echo "Backup successful!"
```

**What changed:**
- `set -e`: Exit if ANY command fails
- `|| {`: Catch error and notify
- Email alert: Ops team knows immediately

**Real-time result:**
- Silent script breaks at 2 AM, nobody knows, data lost
- Production-ready script breaks at 2 AM, ops notified, investigates immediately ✅

---

### Q: How would you optimize a Bash script that processes millions of log lines?

**A:** 
```bash
# ❌ INEFFICIENT - Launches grep, awk, sed as separate processes
cat huge.log | grep "ERROR" | awk '{print $1}' | sed 's/\[//g'

# ✅ EFFICIENT - Using single awk process with better buffering
awk '/ERROR/ {gsub(/\[/, ""); print $1}' huge.log

# ✅ MOST EFFICIENT - Use compiled tools and parallel processing
# 1. Use mawk (faster than awk)
mawk '/ERROR/ {gsub(/\[/, ""); print $1}' huge.log

# 2. Parallel processing for multiple files
find . -name "*.log" -print0 | \
  xargs -0 -P 4 -I {} mawk '/ERROR/ {print $1}' {} > results.txt

# 3. Use built-in Bash features to avoid subshells
while IFS= read -r line; do
    [[ $line =~ ERROR ]] && echo "$line"
done < huge.log
```

**Key Points:**
- Avoid unnecessary pipes and subprocesses
- Use built-in Bash features when possible
- Consider parallel processing for large datasets
- Use faster tools (mawk vs awk, rg vs grep)

---

### Q: Explain the difference between `$(...)` and backticks, and when would you use each?

**A:**
```bash
# Backticks (deprecated but still used)
result=`ls -la`

# Command substitution with $(...) - PREFERRED
result=$(ls -la)

# KEY DIFFERENCES:
# 1. Nesting
# ❌ Backticks - difficult to nest
nested=`echo \`ls\``  # Requires escaping

# ✅ $(...) - easy to nest
nested=$(echo $(ls))

# 2. Performance
# Backticks: spawn subshell
# $(...): more efficient

# 3. Variable expansion
# ✅ Safer with $()
var="test with spaces"
result=$( echo "$var" )  # Works correctly

# When to use backticks:
# - Legacy scripts (backward compatibility)
# - Minimal complexity
# - When you know there's no nesting

# When to use $(...):
# - New scripts (always)
# - Complex nesting required
# - Better readability
```

---

### Q: How would you handle large arrays in Bash efficiently?

**A:**
```bash
# Problem: Bash arrays in memory are inefficient for millions of items
# Solution: Use temporary files or databases

# Method 1: Temporary file approach (efficient for large datasets)
temp_array=$(mktemp)
trap "rm -f $temp_array" EXIT

# Store data
echo "item1" >> "$temp_array"
echo "item2" >> "$temp_array"

# Process efficiently
while IFS= read -r item; do
    process_item "$item"
done < "$temp_array"

# Method 2: Use associative arrays (hashing)
declare -A cache
cache["key1"]="value1"
cache["key2"]="value2"

# Check membership efficiently
if [[ -v cache["key1"] ]]; then
    echo "Found"
fi

# Method 3: Use database for massive data
# SQLite3 is lightweight and powerful
sqlite3 data.db "INSERT INTO items VALUES ('item1');"
sqlite3 data.db "SELECT * FROM items WHERE id > 1000 LIMIT 100;"

# Method 4: Process streaming data (don't store)
while read -r line; do
    process_line "$line"
done < huge_file.txt
```

---

## 2. Error Handling & Debugging

### Q: Design a production-grade error handling mechanism for Bash scripts

**A:**
```bash
#!/bin/bash
set -euo pipefail
IFS=$'\n\t'

# Comprehensive error handling framework
readonly SCRIPT_NAME="$(basename "$0")"
readonly SCRIPT_DIR="$(cd "$(dirname "${BASH_SOURCE[0]}")" && pwd)"
readonly LOG_FILE="${SCRIPT_DIR}/logs/${SCRIPT_NAME%.*}.log"
readonly ERROR_LOG="${SCRIPT_DIR}/logs/${SCRIPT_NAME%.*}_error.log"

# Error codes
readonly E_GENERAL=1
readonly E_MISUSE=2
readonly E_NOINPUT=66
readonly E_UNAVAILABLE=69
readonly E_SOFTWARE=70

# Logging functions
log() {
    local level="$1"
    shift
    local message="$*"
    local timestamp
    timestamp=$(date '+%Y-%m-%d %H:%M:%S')
    
    echo "[$timestamp] [$level] $message" | tee -a "$LOG_FILE"
    
    if [[ "$level" == "ERROR" ]]; then
        echo "[$timestamp] [$level] $message" >> "$ERROR_LOG"
    fi
}

# Error handler
error_handler() {
    local line_no=$1
    local exit_code=$2
    local command="$3"
    
    log ERROR "Command failed at line $line_no with exit code $exit_code"
    log ERROR "Failed command: $command"
    log ERROR "Stack trace:"
    
    local frame=0
    while caller "$frame"; do
        ((frame++))
    done >> "$ERROR_LOG"
    
    cleanup
    exit "$exit_code"
}

# Trap errors
trap 'error_handler ${LINENO} $? "$BASH_COMMAND"' ERR
trap cleanup EXIT INT TERM

# Cleanup function
cleanup() {
    log INFO "Cleaning up..."
    # Remove temporary files, close connections, etc.
}

# Safe command execution with retry
retry_command() {
    local max_attempts=3
    local timeout=5
    local attempt=1
    
    while [[ $attempt -le $max_attempts ]]; do
        if timeout "$timeout" "$@"; then
            return 0
        else
            local exit_code=$?
            log WARN "Attempt $attempt failed with code $exit_code"
            ((attempt++))
            sleep $((2 ** (attempt - 1)))  # Exponential backoff
        fi
    done
    
    log ERROR "Command failed after $max_attempts attempts"
    return 1
}

# Usage
retry_command curl -f https://api.example.com || {
    log ERROR "Failed to fetch from API"
    exit "$E_UNAVAILABLE"
}
```

---

### Q: How would you debug a complex Bash script in production?

**A:**
```bash
# Debug techniques for production scripts

# 1. Enable debug mode selectively
set -x          # Print all commands
set -v          # Print all input lines
set +x          # Disable debugging

# Better: Debug only specific sections
debug=1
if [[ ${debug:-0} -eq 1 ]]; then
    set -x
fi

# 2. Use PS4 for detailed debug output
export PS4='+(${BASH_SOURCE}:${LINENO}): ${FUNCNAME[0]:+${FUNCNAME[0]}(): }'

# 3. Function call tracing
trace_calls() {
    local func="$1"
    shift
    echo ">> CALLING: $func with args: $@" >&2
    "$func" "$@"
    echo "<< RETURNED: $func with status: $?" >&2
}

# 4. Create debug wrapper
debug_exec() {
    if [[ ${DEBUG:-0} -eq 1 ]]; then
        echo "[DEBUG] Executing: $*" >&2
    fi
    "$@"
}

# 5. Assert statements for validation
assert_equals() {
    local expected="$1"
    local actual="$2"
    local message="${3:-Assertion failed}"
    
    if [[ "$expected" != "$actual" ]]; then
        log ERROR "$message: expected '$expected' but got '$actual'"
        return 1
    fi
}

# 6. Add checkpoints
checkpoint() {
    local name="$1"
    echo "✓ Checkpoint: $name" >&2
}

# Usage
DEBUG=1 ./my_script.sh
```

---

## 3. Performance & Scalability

### Q: How would you process a 100GB log file efficiently?

**A:**
```bash
# Efficient processing strategies

# 1. Use stream processing (no buffering)
mawk 'NR % 100000 == 0 {print FILENAME": line "NR}' huge.log

# 2. Parallel processing with GNU Parallel
find . -name "*.log" | \
  parallel --pipe --block 10M 'grep ERROR | wc -l'

# 3. Distributed processing with awk
awk '{
    # Process in chunks
    buffer = buffer $0 "\n"
    if (NR % 10000 == 0) {
        # Process buffer
        process_buffer(buffer)
        buffer = ""
    }
}
END {
    if (buffer) process_buffer(buffer)
}' huge.log

# 4. Memory-efficient pattern matching
grep -E "ERROR|CRITICAL" huge.log | \
  while IFS= read -r line; do
      process_line "$line"
  done

# 5. Binary search for timestamps
get_lines_after_timestamp() {
    local file="$1"
    local timestamp="$2"
    local start_line=1
    local end_line
    end_line=$(wc -l < "$file")
    
    # Binary search for timestamp
    while [[ $start_line -lt $end_line ]]; do
        local mid=$(( (start_line + end_line) / 2 ))
        local line_timestamp=$(sed -n "${mid}p" "$file" | cut -d' ' -f1)
        
        if [[ "$line_timestamp" < "$timestamp" ]]; then
            start_line=$((mid + 1))
        else
            end_line=$mid
        fi
    done
    
    tail -n +$start_line "$file"
}
```

---

## 4. Security Best Practices

### Q: What are critical security vulnerabilities in Bash scripts and how to prevent them?

**A:**
```bash
# 1. Command Injection Prevention
# ❌ VULNERABLE
user_input="$(read -p 'Enter search term: ')"
eval "grep '$user_input' file.txt"  # DANGEROUS!

# ✅ SAFE
user_input="$(read -p 'Enter search term: ')"
grep -- "$user_input" file.txt  # Quotes and -- prevent injection

# 2. Unquoted variables
# ❌ VULNERABLE
rm -rf /tmp/$user_input  # Word splitting and globbing

# ✅ SAFE
rm -rf "/tmp/$user_input"  # Quoted

# 3. Unsafe IFS usage
# ❌ VULNERABLE
IFS=":"
read -r user pass < /etc/passwd  # Unset IFS can cause security issues

# ✅ SAFE
IFS=":" read -r user pass < /etc/passwd
unset IFS  # Always restore

# 4. Proper permissions for scripts
chmod 755 script.sh  # Executable but not world-writable
chmod 700 secret.sh  # Only owner can execute

# 5. Validate all inputs
validate_input() {
    local input="$1"
    local pattern="$2"
    
    if [[ ! "$input" =~ $pattern ]]; then
        echo "Invalid input"
        return 1
    fi
}

# Usage
if validate_input "$user_input" '^[a-zA-Z0-9_-]+$'; then
    process "$user_input"
fi

# 6. Avoid using sudo in scripts
# ❌ VULNERABLE
sudo commands_here  # User must enter password

# ✅ SAFE - Use proper privilege escalation
[[ $EUID -eq 0 ]] || exec sudo "$0" "$@"

# 7. Secure temporary files
# ❌ VULNERABLE
tmpfile="/tmp/mytemp.txt"  # Predictable, world-readable

# ✅ SAFE
tmpfile=$(mktemp)  # Random, only user readable (mode 0600)
trap "rm -f '$tmpfile'" EXIT

# 8. Handle sensitive data
# ❌ VULNERABLE
password="secret123"
echo "$password" | command

# ✅ SAFE
# Use stdin redirection to avoid command-line visibility
command <<< "$password"
# Or use /dev/null for sensitive output
command 2>/dev/null

# 9. Disable history for sensitive operations
set +o history
sensitive_command
set -o history

# 10. Review scripts regularly
shellcheck script.sh  # Use static analysis tools
```

---

## 5. Complex Real-World Scenarios

### Q: Design a production deployment script that handles rollback

**A:**
```bash
#!/bin/bash
set -euo pipefail

readonly DEPLOY_DIR="/opt/myapp"
readonly BACKUP_DIR="/opt/backups"
readonly DEPLOY_USER="deploy"

# State management
declare -A deployment_state
deployment_state["start_time"]="$(date +%s)"
deployment_state["status"]="in_progress"

save_state() {
    local state_file="${BACKUP_DIR}/.deployment_state"
    for key in "${!deployment_state[@]}"; do
        echo "${key}=${deployment_state[$key]}" >> "$state_file"
    done
}

rollback() {
    local backup_version="$1"
    log WARN "Initiating rollback to version $backup_version"
    
    if [[ ! -d "${BACKUP_DIR}/${backup_version}" ]]; then
        log ERROR "Backup not found: ${backup_version}"
        return 1
    fi
    
    # Stop current service
    systemctl stop myapp || true
    
    # Restore from backup
    cp -r "${BACKUP_DIR}/${backup_version}"/* "$DEPLOY_DIR/"
    
    # Restart service
    systemctl start myapp
    
    # Verify health
    if ! check_health; then
        log ERROR "Health check failed after rollback"
        return 1
    fi
    
    log INFO "Rollback successful"
}

deploy() {
    local version="$1"
    local backup_name="backup_$(date +%Y%m%d_%H%M%S)"
    
    # Pre-deployment checks
    check_prerequisites || return 1
    
    # Create backup
    mkdir -p "${BACKUP_DIR}/${backup_name}"
    cp -r "$DEPLOY_DIR" "${BACKUP_DIR}/${backup_name}/"
    
    # Deploy
    log INFO "Deploying version $version"
    download_and_extract "$version" "$DEPLOY_DIR" || {
        log ERROR "Deployment failed, rolling back"
        rollback "$backup_name"
        return 1
    }
    
    # Run migrations
    run_migrations || {
        log ERROR "Migrations failed, rolling back"
        rollback "$backup_name"
        return 1
    }
    
    # Health checks
    if ! check_health; then
        log ERROR "Health check failed, rolling back"
        rollback "$backup_name"
        return 1
    fi
    
    log INFO "Deployment successful"
    deployment_state["status"]="success"
}

check_health() {
    local retries=30
    local delay=2
    
    for ((i=1; i<=retries; i++)); do
        if curl -sf http://localhost:8080/health > /dev/null; then
            return 0
        fi
        sleep $delay
    done
    
    return 1
}

# Main execution
main() {
    trap 'rollback backup_$(date +%Y%m%d_%H%M%S)' ERR
    deploy "${1:-latest}"
}

main "$@"
```

---

## Summary

**Key Takeaways for Senior Bash Development:**
1. ✅ Use `$(...)` over backticks
2. ✅ Implement comprehensive error handling
3. ✅ Optimize for performance with large datasets
4. ✅ Always validate and quote variables
5. ✅ Use proper logging and debugging
6. ✅ Implement rollback mechanisms for deployments
7. ✅ Follow security best practices
8. ✅ Use static analysis tools (shellcheck)
