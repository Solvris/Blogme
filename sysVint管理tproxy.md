以下是优化后的完整 SysVinit 脚本，用于管理 TProxy 的路由规则和 `nftables` 配置。该脚本在原有基础上增强了健壮性、错误处理和日志规范性，同时支持通过环境变量指定 `nftables` 配置文件路径。

---

### **完整 SysVinit 管理脚本**

```sh
#!/bin/sh

### BEGIN INIT INFO
# Provides:          tproxy
# Required-Start:    $network
# Required-Stop:     $network
# Default-Start:     2 3 4 5
# Default-Stop:      0 1 6
# Short-Description: Start and stop TProxy routing rules
# Description:       Configures ip rules and nftables for transparent proxying
### END INIT INFO

# Load LSB function library
. /lib/lsb/init-functions

# Configuration variables
CONFIG_FILE=${TROXY_NFT_CONFIG:-"/etc/nftables.conf.d/tproxy.nft"} # Default path to nftables config file

# Check prerequisites
command -v ip >/dev/null 2>&1 || { log_failure_msg "'ip' command not found"; exit 1; }
command -v nft >/dev/null 2>&1 || { log_failure_msg "'nft' command not found"; exit 1; }

case "$1" in
  start)
    log_daemon_msg "Starting TProxy rules..."

    # Add IP rule
    ip rule add fwmark 1 table 100 2>/dev/null || { log_failure_msg "Failed to add ip rule"; exit 1; }

    # Add IP route
    ip route add local 0.0.0.0/0 dev lo table 100 2>/dev/null || { log_failure_msg "Failed to add ip route"; exit 1; }

    # Load nftables configuration
    if [ -f "$CONFIG_FILE" ]; then
        nft -f "$CONFIG_FILE" || { log_failure_msg "Failed to load nftables rules from $CONFIG_FILE"; exit 1; }
    else
        log_failure_msg "nftables config file $CONFIG_FILE not found"
        exit 1
    fi

    log_success_msg "TProxy rules started successfully."
    ;;

  stop)
    log_daemon_msg "Stopping TProxy rules..."

    # Remove IP rule
    ip rule del fwmark 1 table 100 2>/dev/null || log_warning_msg "Failed to delete ip rule (may already be removed)"

    # Remove IP route
    ip route del local 0.0.0.0/0 dev lo table 100 2>/dev/null || log_warning_msg "Failed to delete ip route (may already be removed)"

    # Delete nftables tables
    nft delete table ip tproxy4 2>/dev/null || log_warning_msg "Failed to delete nftables tproxy4 table (may already be removed)"
    nft delete table ip tproxy6 2>/dev/null || log_warning_msg "Failed to delete nftables tproxy6 table (may already be removed)"

    log_success_msg "TProxy rules stopped successfully."
    ;;

  restart)
    log_daemon_msg "Restarting TProxy rules..."
    $0 stop || { log_failure_msg "Failed to stop TProxy rules"; exit 1; }
    sleep 1
    $0 start || { log_failure_msg "Failed to start TProxy rules"; exit 1; }
    log_success_msg "TProxy rules restarted successfully."
    ;;

  status)
    log_daemon_msg "Checking TProxy rules status..."

    # Check IP rule
    if ip rule show | grep -q "fwmark 1 table 100"; then
        log_success_msg "IP rule is active."
    else
        log_failure_msg "IP rule is missing or inactive."
    fi

    # Check IP route
    if ip route show table 100 | grep -q "local 0.0.0.0/0 dev lo"; then
        log_success_msg "IP route is active."
    else
        log_failure_msg "IP route is missing or inactive."
    fi

    # Check nftables tables
    if nft list tables | grep -q "tproxy4"; then
        log_success_msg "nftables tproxy4 table is active."
    else
        log_failure_msg "nftables tproxy4 table is missing or inactive."
    fi

    if nft list tables | grep -q "tproxy6"; then
        log_success_msg "nftables tproxy6 table is active."
    else
        log_failure_msg "nftables tproxy6 table is missing or inactive."
    fi
    ;;

  *)
    echo "Usage: $0 {start|stop|restart|status}"
    exit 1
    ;;
esac

exit 0
```

---

### **主要改进点**

1. **依赖检查**：
   - 增加了对 `ip` 和 `nft` 工具的检查，确保它们已安装。

2. **返回值逻辑**：
   - 统一了返回值逻辑，确保每个分支都有明确的退出码（`exit 0` 表示成功，`exit 1` 表示失败）。

3. **错误处理**：
   - 在忽略错误输出的同时，增加了日志记录或提示。
   - 如果某个命令失败，会记录详细的错误信息并退出。

4. **重启逻辑**：
   - 在 `restart` 功能中增加了对 `stop` 和 `start` 的错误检查，避免因中途失败导致问题。

5. **状态检查**：
   - 增强了 `status` 功能，不仅显示当前规则，还检查规则是否实际生效。

6. **日志规范性**：
   - 使用了 `lsb` 函数库中的日志函数（如 `log_daemon_msg`、`log_success_msg` 和 `log_failure_msg`），提高了日志规范性和可读性。

7. **灵活性**：
   - 支持通过环境变量 `TROXY_NFT_CONFIG` 指定 `nftables` 配置文件路径，默认路径为 `/etc/nftables.conf.d/tproxy.nft`。

---

### **验证方法**

1. **安装脚本**：
   将脚本保存为 `/etc/init.d/tproxy`，并赋予可执行权限：
   ```bash
   sudo cp script.sh /etc/init.d/tproxy
   sudo chmod +x /etc/init.d/tproxy
   ```

2. **测试功能**：
   - 启动 TProxy：
     ```bash
     sudo /etc/init.d/tproxy start
     ```
   - 停止 TProxy：
     ```bash
     sudo /etc/init.d/tproxy stop
     ```
   - 查看状态：
     ```bash
     sudo /etc/init.d/tproxy status
     ```
   - 重启 TProxy：
     ```bash
     sudo /etc/init.d/tproxy restart
     ```

3. **设置开机自启**：
   在基于 SysVinit 的系统上，启用开机自启：
   ```bash
   sudo update-rc.d tproxy defaults
   ```

---

### **总结**
该脚本结构清晰、逻辑健壮，适合直接管理 TProxy 的路由规则和 `nftables` 配置。它在原有基础上增强了错误处理、日志规范性和灵活性，便于维护和扩展。建议在实际部署前进行充分测试，并根据需要调整配置文件路径或其他参数。

如果有其他问题或需要进一步的帮助，请随时告诉我！
