以下是基于你提供的知识库内容编写的 OpenRC 管理脚本，用于管理 `tproxy` 模式的服务。该脚本会检查配置文件是否存在、清理残留路由规则，并加载 `nftables` 配置；在关闭时会删除相关的路由规则和 `nftables` 表。

---

### **1. 配置文件 (`/etc/conf.d/tproxy`)**

```sh
# /etc/conf.d/tproxy
# Service Name: Used for logging, marker file, log filename, run directory.
SERVICE_NAME="tproxy"

# Base directory for log files (must be writable by root)
LOG_DIR="/var/log"
# Log file will be ${LOG_DIR}/${SERVICE_NAME}.log

# Base directory for runtime state files (must be writable by root)
RUN_DIR="/run"
# Runtime directory will be ${RUN_DIR}/${SERVICE_NAME}
# Marker file will be ${RUN_DIR}/${SERVICE_NAME}/${SERVICE_NAME}.running

# --- Custom Commands ---
# START_COMMANDS: Array of shell commands to execute on service start.
START_COMMANDS=(
    "ip rule add fwmark 1 table 100"
    "ip route add local 0.0.0.0/0 dev lo table 100"
    "nft -f /etc/nftables.conf.d/tproxy.nft"
)

# STOP_COMMANDS: Array of shell commands to execute on service stop.
STOP_COMMANDS=(
    "ip rule del fwmark 1 table 100"
    "ip route del local 0.0.0.0/0 dev lo table 100"
    "nft delete table ip tproxy4"
    "nft delete table ip6 tproxy6"
)

# Optional: Override the user/group if needed, but typically root for system commands
command_user="root"
command_group="root"
```

---

### **2. Init 脚本 (`/etc/init.d/tproxy`)**

```sh
#!/sbin/openrc-run
# /etc/init.d/tproxy
# OpenRC script to manage tproxy mode with nftables and routing rules.

description="Manages tproxy mode with nftables and routing rules."

# --- Default Configuration (can be overridden in /etc/conf.d/${RC_SVCNAME}) ---
SERVICE_NAME="${RC_SVCNAME}" # Default SERVICE_NAME to the script filename
LOG_DIR="/var/log"
RUN_DIR="/run"
START_COMMANDS=() # Default to empty arrays
STOP_COMMANDS=()
command_user="root" # Commands usually need root
command_group="root"

# --- Calculated Paths (based on config values) ---
# These are calculated *after* conf.d is sourced by OpenRC automatically
get_calculated_paths() {
    _log_file="${LOG_DIR}/${SERVICE_NAME}.log"
    _run_subdir="${RUN_DIR}/${SERVICE_NAME}"
    _marker_file="${_run_subdir}/${SERVICE_NAME}.running"
}

# --- Log Function ---
# $1: action (start/stop)
# $2: type (INFO/ERROR/WARN/CMD/OUTPUT)
# $3...: message
log_action() {
    local action="$1"
    local type="$2"
    local timestamp
    timestamp=$(date '+%Y-%m-%d %H:%M:%S')
    shift 2
    local message="$*"
    # Ensure log directory exists
    mkdir -p "${LOG_DIR}" || ewarn "Could not create log directory ${LOG_DIR}"
    echo "${SERVICE_NAME} ${action} ${timestamp} [${type}] ${message}" >> "${_log_file}" \
        || ewarn "Failed to write to log file ${_log_file}"
}

# --- Dependencies ---
depend() {
    need localmount net # Need filesystems and basic network
    # Add other dependencies if your commands require them
    # use dns logger firewall
}

# --- Service Functions ---
start() {
    get_calculated_paths # Calculate paths based on final config
    ebegin "Starting ${SERVICE_NAME} service"

    # Check if marker file exists. If it does, assume already started.
    if [ -f "${_marker_file}" ]; then
        log_action "start" "WARN" "Service already marked as running: ${_marker_file}"
        eend 0 "Already started?"
        return 0
    fi

    # Check if nftables configuration file exists
    if [ ! -f "/etc/nftables.conf.d/tproxy.nft" ]; then
        log_action "start" "ERROR" "Configuration file not found: /etc/nftables.conf.d/tproxy.nft"
        eend 1 "Missing nftables configuration file."
        return 1
    fi

    # Clean up residual routing rules
    local rule_exists=0
    ip rule show | grep -q "lookup 100" && rule_exists=1
    if [ $rule_exists -eq 1 ]; then
        log_action "start" "WARN" "Residual routing rules detected. Cleaning up..."
        ip rule del fwmark 1 table 100 2>/dev/null || true
        ip route del local 0.0.0.0/0 dev lo table 100 2>/dev/null || true
    fi

    # Execute start commands
    log_action "start" "INFO" "Executing start commands..."
    local start_failed=0
    local cmd_index=0
    while [ ${cmd_index} -lt ${#START_COMMANDS[@]} ]; do
        local cmd="${START_COMMANDS[${cmd_index}]}"
        log_action "start" "CMD" "Executing: ${cmd}"
        output=$(eval "${cmd}" 2>&1)
        status=$?
        if [ ${status} -ne 0 ]; then
            log_action "start" "ERROR" "Command failed (exit code ${status}): ${cmd}"
            log_action "start" "OUTPUT" "${output}"
            start_failed=1
            break
        else
            log_action "start" "OUTPUT" "${output}"
        fi
        cmd_index=$((cmd_index + 1))
    done

    # Create marker file only if all commands succeeded
    if [ ${start_failed} -eq 0 ]; then
        log_action "start" "INFO" "Start commands completed successfully."
        touch "${_marker_file}"
        if [ $? -ne 0 ]; then
            log_action "start" "ERROR" "Failed to create marker file: ${_marker_file}"
            start_failed=1
        else
            log_action "start" "INFO" "Created running marker file: ${_marker_file}"
        fi
    else
        log_action "start" "ERROR" "One or more start commands failed. Service not fully started."
    fi

    eend ${start_failed} "Start sequence $( [ ${start_failed} -eq 0 ] && echo "finished." || echo "failed.")"
    return ${start_failed}
}

stop() {
    get_calculated_paths # Calculate paths based on final config
    ebegin "Stopping ${SERVICE_NAME} service"

    # Check if marker file exists. If not, assume already stopped.
    if [ ! -f "${_marker_file}" ]; then
        log_action "stop" "WARN" "Service not marked as running (marker file missing): ${_marker_file}"
        eend 0 "Already stopped?"
        return 0
    fi

    # Execute stop commands
    log_action "stop" "INFO" "Executing stop commands..."
    local stop_failed=0
    local cmd_index=0
    while [ ${cmd_index} -lt ${#STOP_COMMANDS[@]} ]; do
        local cmd="${STOP_COMMANDS[${cmd_index}]}"
        log_action "stop" "CMD" "Executing: ${cmd}"
        output=$(eval "${cmd}" 2>&1)
        status=$?
        if [ ${status} -ne 0 ]; then
            log_action "stop" "ERROR" "Command failed (exit code ${status}): ${cmd}"
            log_action "stop" "OUTPUT" "${output}"
            stop_failed=1
            break
        else
            log_action "stop" "OUTPUT" "${output}"
        fi
        cmd_index=$((cmd_index + 1))
    done

    # Remove marker file
    rm -f "${_marker_file}"
    if [ $? -eq 0 ]; then
        log_action "stop" "INFO" "Removed running marker file: ${_marker_file}"
    else
        log_action "stop" "WARN" "Failed to remove marker file (or it was already gone): ${_marker_file}"
    fi

    eend ${stop_failed} "Stop sequence $( [ ${stop_failed} -eq 0 ] && echo "finished." || echo "failed.")"
    return ${stop_failed}
}
```

---

### **3. 使用方法**

1. **保存文件**：
   - 将配置文件保存为 `/etc/conf.d/tproxy`。
   - 将脚本保存为 `/etc/init.d/tproxy`。

2. **赋予权限**：
   ```bash
   sudo chmod +x /etc/init.d/tproxy
   ```

3. **添加到运行级别**：
   ```bash
   sudo rc-update add tproxy default
   ```

4. **管理服务**：
   - 启动服务：
     ```bash
     sudo rc-service tproxy start
     ```
   - 停止服务：
     ```bash
     sudo rc-service tproxy stop
     ```
   - 查看状态：
     ```bash
     sudo rc-service tproxy status
     ```
   - 查看日志：
     ```bash
     tail -f /var/log/tproxy.log
     ```

---

### **4. 扩展性**

- 如果需要支持 IPv6，可以在 `START_COMMANDS` 和 `STOP_COMMANDS` 中添加相关命令。例如：
  ```sh
  START_COMMANDS+=(
      "ip -6 rule add fwmark 1 table 100"
      "ip -6 route add local ::/0 dev lo table 100"
  )
  STOP_COMMANDS+=(
      "ip -6 rule del fwmark 1 table 100"
      "ip -6 route del local ::/0 dev lo table 100"
  )
  ```

- 这些规则可以根据实际需求进行修改和启用。

---

通过以上脚本，你可以轻松管理 `tproxy` 模式的服务，并确保启动和停止过程的安全性和可靠性。如果有其他需求或问题，请随时补充说明！
