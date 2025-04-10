好的，如果你不需要脚本自动创建 `_svlogd` 用户的部分，我们可以简化日志脚本，去掉用户检查和自动创建逻辑。以下是简化后的版本：

---

### 简化后的 `/etc/sv/mihomo/run`

```bash
#!/bin/sh

# === Configuration ===
# Path to the mihomo executable
MIHOMO_EXEC="/usr/local/bin/mihomo"
# Directory containing the configuration files
CONFIG_DIR="/usr/local/etc/mihomo"
# Optional: Working directory (if needed by mihomo, e.g., for relative paths)
# Uncomment and set if required:
# WORKING_DIR="/usr/local/etc/mihomo"
# === End Configuration ===

# Optional: Change to working directory if defined
[ -n "$WORKING_DIR" ] && cd "$WORKING_DIR"

# Ensure config directory exists
if [ ! -d "$CONFIG_DIR" ]; then
  echo "Error: Config directory not found at $CONFIG_DIR"
  sleep 5 # Prevent rapid restarts by runit if config is missing
  exit 1
fi

# Ensure executable exists and is executable
if [ ! -x "$MIHOMO_EXEC" ]; then
  echo "Error: mihomo executable not found or not executable at $MIHOMO_EXEC"
  sleep 5 # Prevent rapid restarts by runit
  exit 1
fi

# Execute mihomo in the foreground with the specified config directory.
exec "$MIHOMO_EXEC" -d "$CONFIG_DIR"
```

**改动：**
- 无变化，保持原样。

---

### 简化后的 `/etc/sv/mihomo/log/run`

```bash
#!/bin/sh

# Directory to store log files
LOG_DIR="/var/log/mihomo"

# Ensure log directory exists
mkdir -p "$LOG_DIR"

# Execute svlogd directly without changing user.
# -tt adds timestamps to the log lines.
exec svlogd -tt "$LOG_DIR"
```

**改动：**
- 去掉了 `_svlogd` 用户的检查和自动创建逻辑。
- 直接以 `root` 用户运行 `svlogd`。

---

### 创建日志目录

确保日志目录存在并设置正确的权限：

```bash
sudo mkdir -p /var/log/mihomo
sudo chmod 755 /var/log/mihomo
```

---

### 启用服务

完成脚本编写后，启用服务并验证：

```bash
sudo ln -s /etc/sv/mihomo /var/service/
sudo sv status mihomo
```

---

### 验证运行

检查 `mihomo` 是否正常运行：

```bash
ps aux | grep mihomo
```

查看日志文件（如果有配置日志）：

```bash
cat /var/log/mihomo/current
```

---

### 总结

通过上述简化：
1. **去掉了用户管理部分**：不再检查或创建 `_svlogd` 用户，直接以 `root` 用户运行 `svlogd`。
2. **保留了核心功能**：仍然能够启动、停止和监控 `mihomo` 服务。
3. **减少了复杂性**：适合对安全性要求不高的场景。

如果你未来需要提高安全性，可以随时重新引入用户管理的功能。
