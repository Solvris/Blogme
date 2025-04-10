好的，如果你不需要引入专用用户和 `setcap` 的使用，我们可以简化脚本，去掉这些与安全性相关的改进部分。以下是简化后的脚本版本：

---

### 简化版 `/etc/sv/sing-box/run`

```bash
#!/bin/sh

# === Configuration ===
# Path to the sing-box executable
SINGBOX_EXEC="/usr/local/bin/sing-box"
# Path to the configuration file
CONFIG_FILE="/usr/local/etc/sing-box/config.json"
# Optional: Working directory (if needed by sing-box, e.g., for relative paths)
# Uncomment and set if required:
# WORKING_DIR="/usr/local/etc/sing-box"
# === End Configuration ===

# Optional: Change to working directory if defined
[ -n "$WORKING_DIR" ] && cd "$WORKING_DIR"

# Ensure config file exists
if [ ! -f "$CONFIG_FILE" ]; then
  echo "Error: Config file not found at $CONFIG_FILE"
  sleep 5 # Prevent rapid restarts by runit if config is missing
  exit 1
fi

# Ensure executable exists and is executable
if [ ! -x "$SINGBOX_EXEC" ]; then
  echo "Error: sing-box executable not found or not executable at $SINGBOX_EXEC"
  sleep 5 # Prevent rapid restarts by runit
  exit 1
fi

# Execute sing-box in the foreground as root.
exec "$SINGBOX_EXEC" run -c "$CONFIG_FILE"
```

**改动：**
- 去掉了 `RUN_AS_USER` 和用户权限管理的部分。
- 直接以 `root` 用户运行 `sing-box`。

---

### 简化版 `/etc/sv/sing-box/log/run`

```bash
#!/bin/sh

# Directory to store log files
LOG_DIR="/var/log/sing-box"

# Ensure log directory exists
mkdir -p "$LOG_DIR"

# Execute svlogd directly without changing user.
# -tt adds timestamps to the log lines.
exec svlogd -tt "$LOG_DIR"
```

**改动：**
- 去掉了 `SVLOGD_USER` 和用户权限管理的部分。
- 直接以 `root` 用户运行 `svlogd`。

---

### 创建日志目录
确保日志目录存在并设置正确的权限：

```bash
sudo mkdir -p /var/log/sing-box
sudo chmod 755 /var/log/sing-box
```

---

### 启用服务
完成后，启用服务并验证：

```bash
sudo ln -s /etc/sv/sing-box /var/service/
sudo sv status sing-box
```

---

### 验证运行
检查 `sing-box` 是否正常运行：

```bash
ps aux | grep sing-box
```

查看日志文件（如果有配置日志）：

```bash
cat /var/log/sing-box/current
```

---

### 总结
通过上述简化：
1. **去掉了用户权限管理**：脚本直接以 `root` 用户运行 `sing-box` 和 `svlogd`。
2. **保留了核心功能**：仍然能够启动、停止和监控 `sing-box` 服务。
3. **减少了复杂性**：适合对安全性要求不高的场景。

如果你未来需要提高安全性，可以随时重新引入用户管理和 `setcap` 的功能。
