好的！为了提高脚本的灵活性和可维护性，我们可以引入一个变量 `USER` 来存储运行 V2Ray 的用户名（如 `_v2ray`）。这样可以在脚本中统一管理用户信息，并避免硬编码。

以下是更新后的 SysVinit 脚本，使用了一个 `USER` 变量来表示运行 V2Ray 的用户：

---

### **更新后的 SysVinit 脚本**

```bash
#!/bin/sh

### BEGIN INIT INFO
# Provides:          v2ray
# Required-Start:    $local_fs $network
# Required-Stop:     $local_fs $network
# Default-Start:     2 3 4 5
# Default-Stop:      0 1 6
# Short-Description: Start/stop v2ray daemon
# Description:       Manages the v2ray daemon. Relies on internal capabilities for setup/cleanup.
# WARNING:           Running network services as root poses a security risk.
### END INIT INFO

PATH=/sbin:/usr/sbin:/bin:/usr/bin:/usr/local/sbin:/usr/local/bin
DESC="v2ray daemon"                      # 服务描述
NAME=v2ray                               # 基础服务名
DAEMON=/usr/local/bin/v2ray              # v2ray 程序路径
SCRIPTNAME=/etc/init.d/$NAME             # 本脚本的路径
USER=_v2ray                              # 运行 V2Ray 的用户

CONFIG_DIR="/usr/local/etc/v2ray"        # 配置文件目录
RUN_BASE_DIR=/var/run/v2ray              # PID 文件基础目录
PIDFILE="$RUN_BASE_DIR/$NAME.pid"        # PID 文件路径

# Load LSB function library
. /lib/lsb/init-functions

# Function to determine the startup command based on configuration files
get_daemon_args() {
    if [ -r "$CONFIG_DIR/config.json" ]; then
        echo "run -c $CONFIG_DIR/config.json"
    elif [ -r "$CONFIG_DIR/config.v5.json" ]; then
        echo "run -c $CONFIG_DIR/config.v5.json --format jsonv5"
    elif [ -r "$CONFIG_DIR/config.v5.jsonc" ]; then
        echo "run -c $CONFIG_DIR/config.v5.jsonc --format jsonv5"
    else
        log_failure_msg "No valid configuration file found in $CONFIG_DIR."
        exit 1
    fi
}

# Check prerequisites
test -x "$DAEMON" || { log_failure_msg "$DAEMON not found or not executable"; exit 1; }
test -d "$CONFIG_DIR" || { log_failure_msg "Config directory $CONFIG_DIR not found or not readable"; exit 1; }

# Function to start the service
do_start() {
    # 检查运行目录是否存在
    if [ ! -d "$RUN_BASE_DIR" ]; then
        mkdir -p "$RUN_BASE_DIR" || { log_failure_msg "Failed to create $RUN_BASE_DIR"; return 1; }
    fi

    # 确保运行目录对指定用户可写
    chown "$USER" "$RUN_BASE_DIR" || { log_failure_msg "Failed to set ownership of $RUN_BASE_DIR to $USER"; return 1; }

    # 检查是否已经在运行
    if [ -f "$PIDFILE" ] && kill -0 $(cat "$PIDFILE") 2>/dev/null; then
        echo "$DESC is already running."
        return 1 # 已在运行，返回非零值
    fi

    # 获取启动参数
    DAEMON_ARGS=$(get_daemon_args)

    # 启动守护进程
    log_progress_msg "Starting $DESC process as user '$USER'..."
    start-stop-daemon --start --quiet --pidfile "$PIDFILE" --make-pidfile \
        --background --exec "$DAEMON" --chuid "$USER" -- $DAEMON_ARGS > /dev/null 2>&1
    RETVAL=$?

    if [ $RETVAL -eq 0 ]; then
        log_success_msg "$DESC started successfully as user '$USER'."
        return 0
    else
        log_failure_msg "Failed to start $DESC (exit code $RETVAL)."
        return 1
    fi
}

# Function to stop the service
do_stop() {
    # 检查是否在运行
    if [ ! -f "$PIDFILE" ] || ! kill -0 $(cat "$PIDFILE") 2>/dev/null; then
        echo "$DESC is not running."
        return 1 # 未运行，返回非零值
    fi

    # 停止守护进程
    log_progress_msg "Stopping $DESC process..."
    start-stop-daemon --stop --quiet --retry=TERM/15/KILL/5 --pidfile "$PIDFILE"
    STOP_RETVAL=$?

    # 清理 PID 文件
    rm -f "$PIDFILE"

    if [ $STOP_RETVAL -eq 0 ]; then
        log_success_msg "$DESC stopped successfully."
        return 0
    else
        log_failure_msg "Failed to stop $DESC (exit code $STOP_RETVAL)."
        return 1
    fi
}

# Function to check status
do_status() {
    if [ -f "$PIDFILE" ]; then
        PID=$(cat "$PIDFILE")
        if ps -p "$PID" > /dev/null 2>&1 && grep -q "$DAEMON" "/proc/$PID/cmdline"; then
            echo "$DESC is running with PID $PID."
            return 0
        else
            echo "$DESC is dead but PID file exists (PID: $PID)."
            return 1
        fi
    else
        echo "$DESC is not running."
        return 3
    fi
}

### Main logic ###
case "$1" in
    start)
        log_daemon_msg "Starting $DESC ($NAME)..."
        do_start
        case "$?" in
            0) log_end_msg 0 ;; # 启动成功
            1) log_end_msg 1 ;; # 启动失败或已在运行
        esac
        ;;
    stop)
        log_daemon_msg "Stopping $DESC ($NAME)..."
        do_stop
        case "$?" in
            0) log_end_msg 0 ;; # 停止成功
            1) log_end_msg 1 ;; # 停止失败或未运行
        esac
        ;;
    status)
        do_status
        ;;
    restart|force-reload)
        log_daemon_msg "Restarting $DESC ($NAME)..."
        do_stop
        sleep 1
        do_start
        case "$?" in
            0) log_end_msg 0 ;; # 重启成功
            1) log_end_msg 1 ;; # 重启失败
        esac
        ;;
    *)
        echo "Usage: $SCRIPTNAME {start|stop|status|restart}" >&2
        exit 3
        ;;
esac

exit 0
```

---

### **主要修改点**

1. **引入 `USER` 变量**
   - 在脚本顶部定义了 `USER` 变量：
     ```bash
     USER=_v2ray
     ```
   - 使用 `$USER` 替代硬编码的 `_v2ray` 用户名。
   - 配置用户
     ```bash
     useradd -u 990 -r -M - U-g 990 -s /sbin/nologin _v2ray
     ```
    - 赋予二进制特权
     ```bash
     setcap cap_net_admin,cap_net_raw+ep /usr/local/bin/v2ray
     ```

2. **设置运行目录权限**
   - 在 `do_start` 函数中，确保运行目录 `/var/run/v2ray` 对 `$USER` 用户具有写权限：
     ```bash
     chown "$USER" "$RUN_BASE_DIR" || { log_failure_msg "Failed to set ownership of $RUN_BASE_DIR to $USER"; return 1; }
     ```

3. **日志输出**
   - 在启动时明确记录以哪个用户身份运行服务：
     ```bash
     log_progress_msg "Starting $DESC process as user '$USER'..."
     ```

4. **兼容性**
   - 如果需要更改运行用户，只需修改 `USER` 变量即可，无需修改其他代码。

---

### **注意事项**

1. **用户权限**
   - 确保 `$USER` 用户已正确创建，并具有运行 V2Ray 所需的权限。
   - 如果需要运行特权操作（如绑定端口 < 1024），可以通过 `setcap` 或其他方式赋予 `$USER` 特定权限。

2. **配置文件权限**
   - 确保 `$USER` 用户对 `/usr/local/etc/v2ray` 配置目录具有读取权限：
     ```bash
     sudo chown -R _v2ray:_v2ray /usr/local/etc/v2ray
     sudo chmod -R 750 /usr/local/etc/v2ray
     ```

3. **测试脚本**
   - 测试脚本功能：
     ```bash
     sudo /etc/init.d/v2ray start
     sudo /etc/init.d/v2ray status
     sudo /etc/init.d/v2ray stop
     sudo /etc/init.d/v2ray restart
     ```

---

### **总结**

通过引入 `USER` 变量，脚本变得更加灵活和易于维护。您可以轻松地通过修改 `USER` 变量来更改运行 V2Ray 的用户，而无需修改其他代码。如果您有任何其他问题或需要进一步调整，请随时告诉我！ 😊
