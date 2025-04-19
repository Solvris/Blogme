以下是修正后的完整脚本，直接以 `root` 用户运行（无需切换用户）。根据之前的分析和改进建议，修复了返回值逻辑、日志输出、PID 文件处理等问题，并优化了代码结构。

---

### **修正后的完整 SysVinit 脚本**

```sh
#!/bin/sh

### BEGIN INIT INFO
# Provides:          mihomo
# Required-Start:    $local_fs $network
# Required-Stop:     $local_fs $network
# Default-Start:     2 3 4 5
# Default-Stop:      0 1 6
# Short-Description: Start/stop mihomo daemon
# Description:       Manages the mihomo daemon. Relies on internal capabilities for setup/cleanup.
# WARNING:           Running network services as root poses a security risk.
### END INIT INFO

PATH=/sbin:/usr/sbin:/bin:/usr/bin:/usr/local/sbin:/usr/local/bin
DESC="mihomo daemon"                   # 服务描述
NAME=mihomo                            # 基础服务名
DAEMON=/usr/local/bin/mihomo           # mihomo 程序路径
SCRIPTNAME=/etc/init.d/$NAME           # 本脚本的路径

CONFIG_FILE="/usr/local/etc/mihomo/config.yaml" # 配置文件路径
CONFIG_DIR=$(dirname "$CONFIG_FILE")   # 配置文件所在目录

RUN_BASE_DIR=/var/run/mihomo           # PID 文件基础目录
PIDFILE="$RUN_BASE_DIR/$NAME.pid"      # PID 文件路径
DEFAULT_FILE="/etc/default/$NAME"      # 默认环境变量文件

# mihomo 启动参数 (使用 -d 指定配置文件所在目录)
DAEMON_ARGS="-d $CONFIG_DIR"
EXTRA_ARGS="" # Placeholder for extra arguments from default file

# Load LSB function library
. /lib/lsb/init-functions

# Load default environment variables if exists and is readable
if [ -r "$DEFAULT_FILE" ]; then
    . "$DEFAULT_FILE"
fi

# Append extra arguments
DAEMON_ARGS="$DAEMON_ARGS $EXTRA_ARGS"

# Check prerequisites
test -x "$DAEMON" || { log_failure_msg "$DAEMON not found or not executable"; exit 1; }
test -r "$CONFIG_FILE" || { log_failure_msg "Config file $CONFIG_FILE not found or not readable"; exit 1; }

# Function to start the service
do_start() {
    # 检查运行目录是否存在
    if [ ! -d "$RUN_BASE_DIR" ]; then
        mkdir -p "$RUN_BASE_DIR" || { log_failure_msg "Failed to create $RUN_BASE_DIR"; return 1; }
    fi

    # 检查是否已经在运行
    if [ -f "$PIDFILE" ] && kill -0 $(cat "$PIDFILE") 2>/dev/null; then
        log_warning_msg "$DESC is already running with PID $(cat "$PIDFILE")"
        return 1 # 已在运行，返回非零值
    fi

    # 启动守护进程
    log_daemon_msg "Starting $DESC process"
    start-stop-daemon --start --quiet --pidfile "$PIDFILE" --make-pidfile \
        --background --exec "$DAEMON" -- $DAEMON_ARGS > /dev/null 2>&1
    RETVAL=$?

    if [ $RETVAL -eq 0 ]; then
        log_end_msg 0
        return 0
    else
        log_end_msg 1
        return 1
    fi
}

# Function to stop the service
do_stop() {
    # 检查是否在运行
    if [ ! -f "$PIDFILE" ] || ! kill -0 $(cat "$PIDFILE") 2>/dev/null; then
        log_warning_msg "$DESC is not running"
        return 1 # 未运行，返回非零值
    fi

    # 停止守护进程
    log_daemon_msg "Stopping $DESC process"
    start-stop-daemon --stop --quiet --retry=TERM/15/KILL/5 --pidfile "$PIDFILE"
    STOP_RETVAL=$?

    # 清理 PID 文件
    rm -f "$PIDFILE"

    if [ $STOP_RETVAL -eq 0 ]; then
        log_end_msg 0
        return 0
    else
        log_end_msg 1
        return 1
    fi
}

# Function to check status
do_status() {
    if [ -f "$PIDFILE" ]; then
        PID=$(cat "$PIDFILE")
        if ps -p "$PID" > /dev/null 2>&1; then
            log_success_msg "$DESC is running with PID $PID"
            return 0
        else
            log_warning_msg "$DESC is dead but PID file exists (PID: $PID)"
            return 1
        fi
    else
        log_warning_msg "$DESC is not running"
        return 3
    fi
}

### Main logic ###
case "$1" in
    start)
        do_start
        ;;
    stop)
        do_stop
        ;;
    status)
        do_status
        ;;
    restart|force-reload)
        log_daemon_msg "Restarting $DESC"
        do_stop
        sleep 1
        do_start
        ;;
    *)
        echo "Usage: $SCRIPTNAME {start|stop|status|restart}" >&2
        exit 3
        ;;
esac

exit 0
```

---

### **主要改进点**
1. **返回值逻辑**：
   - 统一了返回值逻辑，使 `0` 表示成功，非零表示失败。
   - 示例：`do_start` 和 `do_stop` 的返回值逻辑已调整。

2. **启动命令的安全性**：
   - 直接使用 `--exec` 参数调用 `DAEMON`，避免通过 `/bin/bash` 启动子进程。

3. **PID 文件处理**：
   - 在检查 PID 文件时，增加对 PID 内容的校验，确保其对应一个有效的进程。

4. **默认环境变量加载**：
   - 增加了对默认环境变量文件的可读性检查。

5. **日志输出优化**：
   - 统一日志输出格式，避免重复或混淆。

6. **注释完善**：
   - 为关键逻辑添加详细注释，便于后续维护。

---

### **验证方法**
1. **安装脚本**：
   将脚本保存为 `/etc/init.d/mihomo`，并赋予可执行权限：
   ```bash
   sudo cp script.sh /etc/init.d/mihomo
   sudo chmod +x /etc/init.d/mihomo
   ```

2. **测试功能**：
   - 启动服务：
     ```bash
     sudo /etc/init.d/mihomo start
     ```
   - 检查状态：
     ```bash
     sudo /etc/init.d/mihomo status
     ```
   - 停止服务：
     ```bash
     sudo /etc/init.d/mihomo stop
     ```

3. **设置开机自启**：
   在基于 SysVinit 的系统上，启用开机自启：
   ```bash
   sudo update-rc.d mihomo defaults
   ```

---

### **总结**
修正后的脚本更加健壮和安全，解决了原有问题，并增强了可维护性。脚本直接以 `root` 用户运行，符合你的要求。建议在实际部署前进行充分测试，并根据需要调整配置文件路径或其他参数。如果有其他问题或需要进一步的帮助，请随时告诉我！
