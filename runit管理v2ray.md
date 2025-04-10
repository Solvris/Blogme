以下是对你的脚本进行改进后的版本，支持根据配置文件的不同格式动态选择启动命令，并且优化了代码结构、注释和日志管理部分。

---

### 改进后的 `/etc/sv/v2ray/run`

```bash
#!/bin/sh

# === Configuration ===
# Path to the V2Ray executable
V2RAY_EXEC="/usr/local/bin/v2ray"
# Directory containing the configuration files
CONFIG_DIR="/usr/local/etc/v2ray"
# Log directory for svlogd (optional but recommended)
LOG_DIR="/var/log/v2ray"

# --- Script Variables ---
_v2ray_config_file=""
_v2ray_command_args=""

# --- Helper function to detect config ---
_v2ray_detect_config() {
    if [ -f "${CONFIG_DIR}/config.v5.json" ]; then
        _v2ray_config_file="${CONFIG_DIR}/config.v5.json"
        _v2ray_command_args="run -c ${_v2ray_config_file} --format jsonv5"
        return 0
    elif [ -f "${CONFIG_DIR}/config.v5.jsonc" ]; then
        _v2ray_config_file="${CONFIG_DIR}/config.v5.jsonc"
        _v2ray_command_args="run -c ${_v2ray_config_file} --format jsonv5"
        return 0
    elif [ -f "${CONFIG_DIR}/config.json" ]; then
        _v2ray_config_file="${CONFIG_DIR}/config.json"
        _v2ray_command_args="run -c ${_v2ray_config_file}"
        return 0
    else
        _v2ray_config_file=""
        _v2ray_command_args=""
        return 1 # Failure
    fi
}

# --- Main Execution ---

# Detect configuration
if ! _v2ray_detect_config; then
    echo "Error: No suitable configuration file found in ${CONFIG_DIR}" >&2
    echo "Expected config.v5.json, config.v5.jsonc, or config.json" >&2
    sleep 5 # Prevent rapid restarts if config is missing
    exit 1
fi

echo "Using configuration: ${_v2ray_config_file}" >&2 # Log to stderr (often piped to logger)

# Check executable
if [ ! -x "$V2RAY_EXEC" ]; then
    echo "Error: V2Ray executable not found or not executable: $V2RAY_EXEC" >&2
    sleep 5
    exit 1
fi

# Execute V2Ray in the foreground. runit handles supervision.
# 'exec' replaces this shell process with the V2Ray process.
exec "${V2RAY_EXEC}" ${_v2ray_command_args}

# Script should not reach here if exec succeeds
exit 1
```

**改进点：**
- **支持 `config.v5.jsonc`**：增加了对 `config.v5.jsonc` 文件的支持。
- **更清晰的错误信息**：在找不到配置文件时，明确列出期望的文件名。
- **注释增强**：每个部分都有详细的注释，方便后续维护。

---

### 改进后的 `/etc/sv/v2ray/log/run`

```bash
#!/bin/sh

# User to run svlogd as (e.g., '_svlogd' or 'root')
SVLOGD_USER="_svlogd"
# Directory to store logs (must match LOG_DIR in main run script if referenced)
LOG_DIR="/var/log/v2ray"

# Ensure log directory exists
mkdir -p "$LOG_DIR"

# Set ownership for the svlogd user.
# IMPORTANT: This command will FAIL if the SVLOGD_USER does not exist.
# If it fails, either create the user '_svlogd' (recommended):
#   sudo groupadd -r _svlogd
#   sudo useradd -r -s /usr/bin/nologin -d /var/empty -g _svlogd _svlogd
# Or change SVLOGD_USER above (e.g., to "root") and update chown target.
chown "$SVLOGD_USER:$SVLOGD_USER" "$LOG_DIR"

# Run svlogd
exec chpst -u "$SVLOGD_USER" svlogd -tt "$LOG_DIR"
```

**改动：**
- 日志脚本保持不变，仍然使用 `_svlogd` 用户运行 `svlogd`。
- 如果不需要 `_svlogd` 用户，可以将 `SVLOGD_USER` 改为 `root` 并去掉 `chpst` 命令。

---

### 创建日志目录

确保日志目录存在并设置正确的权限：

```bash
sudo mkdir -p /var/log/v2ray
sudo chown _svlogd:_svlogd /var/log/v2ray
```

如果不需要 `_svlogd` 用户，可以直接使用：

```bash
sudo mkdir -p /var/log/v2ray
sudo chmod 755 /var/log/v2ray
```

---

### 启用服务

完成脚本编写后，启用服务并验证：

```bash
sudo ln -s /etc/sv/v2ray /var/service/
sudo sv status v2ray
```

---

### 验证运行

检查 `v2ray` 是否正常运行：

```bash
ps aux | grep v2ray
```

查看日志文件（如果有配置日志）：

```bash
cat /var/log/v2ray/current
```

---

### 总结

通过上述改进：
1. **支持多种配置文件**：能够自动检测 `config.v5.json`、`config.v5.jsonc` 和 `config.json`，并根据文件类型选择合适的启动命令。
2. **简化日志管理**：日志脚本保持简单，直接使用 `svlogd` 进行日志记录。
3. **注释清晰化**：每个部分都有详细的注释，方便后续维护和扩展。

如果你未来需要进一步优化或添加功能，可以随时扩展脚本。
