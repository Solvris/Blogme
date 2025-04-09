以下是一个符合 **SysVinit** 风格的脚本，用于管理 `tproxy` 模式的服务。该脚本会检查配置文件是否存在、清理残留路由规则，并加载 `nftables` 配置；在关闭时会删除相关的路由规则和 `nftables` 表。

---

### SysVinit 脚本 (`/etc/init.d/tproxy`)

```bash
#!/bin/sh
### BEGIN INIT INFO
# Provides:          tproxy
# Required-Start:    $local_fs $network
# Required-Stop:     $local_fs $network
# Default-Start:     2 3 4 5
# Default-Stop:      0 1 6
# Short-Description: Start/stop tproxy service
# Description:       Manages tproxy mode with nftables and routing rules.
### END INIT INFO

PATH=/sbin:/usr/sbin:/bin:/usr/bin:/usr/local/sbin:/usr/local/bin
DESC="TPROXY Service"                    # 服务描述
NAME=tproxy                              # 基础服务名
SCRIPTNAME=/etc/init.d/$NAME             # 本脚本的路径

# 配置文件路径
NFTABLES_CONF="/etc/nftables.conf.d/tproxy.nft"
LOG_FILE="/var/log/${NAME}.log"          # 日志文件路径

# 日志记录函数
log_message() {
    local action="$1"
    local type="$2"
    local timestamp
    timestamp=$(date "+%Y-%m-%d %H:%M:%S")
    shift 2
    local message="$*"
    echo "${NAME} ${action} ${timestamp} [${type}] ${message}" >> "$LOG_FILE"
}

# 启动服务
do_start() {
    log_message "start" "INFO" "Starting TPROXY service."

    # 检查 nftables 配置文件是否存在
    if [ ! -f "$NFTABLES_CONF" ]; then
        log_message "start" "ERROR" "Configuration file not found: $NFTABLES_CONF"
        return 1
    fi

    # 检查并清理残留路由规则
    local rule_exists=0
    ip rule show | grep -q "lookup 100" && rule_exists=1
    if [ $rule_exists -eq 1 ]; then
        log_message "start" "WARN" "Residual routing rules detected. Cleaning up..."
        ip rule del fwmark 1 table 100 2>/dev/null || true
        ip route del local 0.0.0.0/0 dev lo table 100 2>/dev/null || true
    fi

    # 添加路由规则
    log_message "start" "INFO" "Adding routing rules."
    ip rule add fwmark 1 table 100 2>/dev/null || true
    ip route add local 0.0.0.0/0 dev lo table 100 2>/dev/null || true

    # 加载 nftables 配置
    log_message "start" "INFO" "Loading nftables configuration from $NFTABLES_CONF."
    if ! nft -f "$NFTABLES_CONF"; then
        log_message "start" "ERROR" "Failed to load nftables configuration. Check syntax in $NFTABLES_CONF."
        return 1
    fi

    log_message "start" "INFO" "TPROXY service started successfully."
    return 0
}

# 停止服务
do_stop() {
    log_message "stop" "INFO" "Stopping TPROXY service."

    # 删除路由规则
    log_message "stop" "INFO" "Cleaning up routing rules."
    ip rule del fwmark 1 table 100 2>/dev/null || true
    ip route del local 0.0.0.0/0 dev lo table 100 2>/dev/null || true

    # 删除 nftables 表
    log_message "stop" "INFO" "Deleting nftables tables."
    nft delete table ip tproxy4 2>/dev/null || true
    nft delete table ip6 tproxy6 2>/dev/null || true

    log_message "stop" "INFO" "TPROXY service stopped successfully."
    return 0
}

# 主逻辑
case "$1" in
  start)
    do_start
    RETVAL=$?
    [ $RETVAL -eq 0 ] && log_daemon_msg "Started" "$NAME" || log_failure_msg "Failed to start $NAME"
    ;;
  stop)
    do_stop
    RETVAL=$?
    [ $RETVAL -eq 0 ] && log_daemon_msg "Stopped" "$NAME" || log_failure_msg "Failed to stop $NAME"
    ;;
  restart|force-reload)
    do_stop
    sleep 1
    do_start
    RETVAL=$?
    [ $RETVAL -eq 0 ] && log_daemon_msg "Restarted" "$NAME" || log_failure_msg "Failed to restart $NAME"
    ;;
  *)
    echo "Usage: $SCRIPTNAME {start|stop|restart}" >&2
    exit 3
    ;;
esac

exit 0
```

---

### 脚本功能说明

#### **启动逻辑**
1. **检查配置文件**：
   - 确保 `/etc/nftables.conf.d/tproxy.nft` 存在。如果不存在，记录错误日志并退出。

2. **清理残留路由规则**：
   - 使用 `ip rule show` 检查是否存在与 `table 100` 相关的路由规则。
   - 如果存在残留规则，执行以下命令清理：
     ```bash
     ip rule del fwmark 1 table 100
     ip route del local 0.0.0.0/0 dev lo table 100
     ```

3. **添加路由规则**：
   - 执行以下命令添加路由规则：
     ```bash
     ip rule add fwmark 1 table 100
     ip route add local 0.0.0.0/0 dev lo table 100
     ```
   - 这些规则也可以扩展为支持 IPv6（见下文注释）。

4. **加载 nftables 配置**：
   - 执行 `nft -f /etc/nftables.conf.d/tproxy.nft` 加载防火墙规则。
   - 如果加载失败（例如语法错误），记录错误日志并退出。

#### **停止逻辑**
1. **删除路由规则**：
   - 再次清理与 `table 100` 相关的路由规则：
     ```bash
     ip rule del fwmark 1 table 100
     ip route del local 0.0.0.0/0 dev lo table 100
     ```

2. **删除 nftables 表**：
   - 删除 `tproxy4` 和 `tproxy6` 表：
     ```bash
     nft delete table ip tproxy4
     nft delete table ip6 tproxy6
     ```

#### **日志记录**
- 所有操作都会记录到 `/var/log/tproxy.log` 文件中，格式为：
  ```
  tproxy start 时间 [类型] 消息
  tproxy stop 时间 [类型] 消息
  ```

---

### 添加对 IPv6 的支持（注释）

为了便于扩展 IPv6 支持，可以在脚本中添加以下注释：

```bash
# IPv6 示例（取消注释以启用）
# ip rule add fwmark 1 table 100
# ip -6 route add local ::/0 dev lo table 100
# nft delete table ip6 tproxy6
```

这些规则可以根据实际需求进行修改和启用。

---

### 使用方法

1. **保存脚本**：
   将脚本保存为 `/etc/init.d/tproxy`，并赋予可执行权限：
   ```bash
   sudo chmod +x /etc/init.d/tproxy
   ```

2. **创建配置文件**：
   确保 `/etc/nftables.conf.d/tproxy.nft` 文件存在，并包含正确的 `nftables` 规则。

3. **设置开机自启**：
   如果使用 SysVinit，可以启用服务：
   ```bash
   sudo update-rc.d tproxy defaults
   ```

4. **管理服务**：
   - 启动服务：
     ```bash
     sudo /etc/init.d/tproxy start
     ```
   - 停止服务：
     ```bash
     sudo /etc/init.d/tproxy stop
     ```
   - 重启服务：
     ```bash
     sudo /etc/init.d/tproxy restart
     ```

5. **查看日志**：
   查看日志文件 `/var/log/tproxy.log`：
   ```bash
   cat /var/log/tproxy.log
   ```

---

