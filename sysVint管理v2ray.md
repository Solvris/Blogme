以下是根据你的需求重新编写的 SysVinit 脚本。该脚本会检查 `/usr/local/etc/v2ray/` 目录下的配置文件，并根据文件名和格式选择正确的启动命令。如果未找到符合要求的配置文件，则提示错误信息。

---

### **完整 SysVinit 脚本**

```sh
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

    # 检查是否已经在运行
    if [ -f "$PIDFILE" ] && kill -0 $(cat "$PIDFILE") 2>/dev/null; then
        echo "$DESC is already running."
        return 1 # 已在运行，返回非零值
    fi

    # 获取启动参数
    DAEMON_ARGS=$(get_daemon_args)

    # 启动守护进程
    log_progress_msg "Starting $DESC process..."
    start-stop-daemon --start --quiet --pidfile "$PIDFILE" --make-pidfile \
        --background --exec "$DAEMON" -- $DAEMON_ARGS > /dev/null 2>&1
    RETVAL=$?

    if [ $RETVAL -eq 0 ]; then
        log_success_msg "$DESC started successfully."
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

### **主要功能说明**
1. **动态检测配置文件**：
   - 使用 `get_daemon_args` 函数动态检测 `/usr/local/etc/v2ray/` 下的配置文件。
   - 如果存在 `config.json`，启动命令为 `v2ray run -c config.json`。
   - 如果存在 `config.v5.json` 或 `config.v5.jsonc`，启动命令为 `v2ray run -c config.v5.json --format jsonv5`。
   - 如果未找到符合要求的配置文件，输出错误信息并退出。

2. **健壮性检查**：
   - 检查二进制文件和配置目录是否存在且可读。
   - 在状态检查时验证 PID 文件的有效性。

3. **日志输出**：
   - 使用 `log_daemon_msg`、`log_success_msg` 和 `log_failure_msg` 记录操作状态。
   - 避免显示 `(warning)` 提示信息。

4. **启动和停止逻辑**：
   - 使用 `start-stop-daemon` 管理守护进程。
   - 在停止时清理 PID 文件。

---

### **验证方法**
1. **安装脚本**：
   将脚本保存为 `/etc/init.d/v2ray`，并赋予可执行权限：
   ```bash
   sudo cp script.sh /etc/init.d/v2ray
   sudo chmod +x /etc/init.d/v2ray
   ```

2. **测试功能**：
   - 启动服务：
     ```bash
     sudo /etc/init.d/v2ray start
     ```
   - 检查状态：
     ```bash
     sudo /etc/init.d/v2ray status
     ```
   - 停止服务：
     ```bash
     sudo /etc/init.d/v2ray stop
     ```

3. **设置开机自启**：
   在基于 SysVinit 的系统上，启用开机自启：
   ```bash
   sudo update-rc.d v2ray defaults
   ```

---

### **总结**
该脚本实现了动态检测配置文件的功能，并根据文件名和格式选择正确的启动命令。脚本结构清晰，逻辑健壮，适合直接管理 `v2ray` 服务。建议在实际部署前进行充分测试，并根据需要调整配置文件路径或其他参数。如果有其他问题或需要进一步的帮助，请随时告诉我！
