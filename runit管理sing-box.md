好的，根据你的需求，我们将启动命令改为 `sing-box run -D $CONFIG_DIR`，其中 `$CONFIG_DIR` 是配置文件所在的目录（例如 `/usr/local/etc/sing-box`）。以下是更新后的脚本。

---

### 更新后的 `/etc/sv/sing-box/run`

```bash
#!/bin/sh

# === Configuration ===
# Path to the sing-box executable
SINGBOX_EXEC="/usr/local/bin/sing-box"
# Directory containing the configuration files
CONFIG_DIR="/usr/local/etc/sing-box"
# Optional: Working directory (if needed by sing-box, e.g., for relative paths)
# Uncomment and set if required:
# WORKING_DIR="/usr/local/etc/sing-box"
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
if [ ! -x "$SINGBOX_EXEC" ]; then
  echo "Error: sing-box executable not found or not executable at $SINGBOX_EXEC"
  sleep 5 # Prevent rapid restarts by runit
  exit 1
fi

# Execute sing-box in the foreground with the specified config directory.
exec "$SINGBOX_EXEC" run -D "$CONFIG_DIR"
```

**改动：**
- 将 `CONFIG_FILE` 替换为 `CONFIG_DIR`。
- 启动命令改为 `sing-box run -D "$CONFIG_DIR"`。
- 检查配置路径时，改为检查目录是否存在（`-d`）。

---

### 日志脚本 `/etc/sv/sing-box/log/run`
日志脚本不需要更改，仍然可以使用之前的版本：

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
通过上述修改：
1. **启动命令调整**：将启动命令改为 `sing-box run -D $CONFIG_DIR`，符合你的需求。
2. **配置目录检查**：脚本现在检查配置目录是否存在，而不是单独的配置文件。
3. **保持简单性**：脚本逻辑清晰，适合直接以 `root` 用户运行的场景。

如果你未来需要进一步优化或添加功能，可以随时扩展脚本。
