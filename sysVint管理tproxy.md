以下是一个符合 **SysVinit** 风格的脚本，用于管理 `tproxy` 模式的服务。该脚本会检查配置文件是否存在、清理残留路由规则，并加载 `nftables` 配置；在关闭时会删除相关的路由规则和 `nftables` 表。

---

### SysVinit 脚本 (`/etc/init.d/tproxy`)

```bash
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

case "$1" in
  start)
    echo "Starting TProxy rules..."
    ip rule add fwmark 1 table 100 2>/dev/null
    ip route add local 0.0.0.0/0 dev lo table 100 2>/dev/null
    nft -f /etc/nftables.conf.d/tproxy.nft
    ;;

  stop)
    echo "Stopping TProxy rules..."
    ip rule del fwmark 1 table 100 2>/dev/null
    ip route del local 0.0.0.0/0 dev lo table 100 2>/dev/null
    nft delete table ip tproxy4 2>/dev/null
    nft delete table ip tproxy6 2>/dev/null
    ;;

  restart)
    $0 stop
    sleep 1
    $0 start
    ;;

  status)
    echo "ip rule:"
    ip rule show | grep table
    echo "ip route (table 100):"
    ip route show table 100
    echo "nftables:"
    nft list tables | grep tproxy
    ;;

  *)
    echo "Usage: /etc/init.d/tproxy {start|stop|restart|status}"
    exit 1
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

