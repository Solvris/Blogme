以下是完整的 SysVinit 脚本，用于管理 `xray` 服务。该脚本基于之前的讨论进行了优化，移除了对 `/etc/default/$NAME` 的依赖，同时增强了健壮性、安全性和可维护性。

---

### **完整 SysVinit 管理脚本**

```sh
#!/bin/sh

### BEGIN INIT INFO
# Provides:          xray
# Required-Start:    $local_fs $network
# Required-Stop:     $local_fs $network
# Default-Start:     2 3 4 5
# Default-Stop:      0 1 6
# Short-Description: Start/stop xray proxy daemon
# Description:       Manages the xray proxy daemon.
### END INIT INFO

# Paths and Variables
PATH=/sbin:/usr/sbin:/bin:/usr/bin:/usr/local/sbin:/usr/local/bin
DESC="xray proxy daemon"                # Service description
NAME=xray                               # Service name
DAEMON=/usr/local/bin/xray             # Path to the xray binary
CONFIG_FILE=/usr/local/etc/xray/config.json # Path to the config file
SCRIPTNAME=/etc/init.d/$NAME           # Path to this script

RUN_BASE_DIR=/var/run/xray              # Base directory for PID file
PIDFILE="$RUN_BASE_DIR/$NAME.pid"       # Path to the PID file

# Xray start arguments
DAEMON_ARGS="run -c \"$CONFIG_FILE\""

# Load LSB function library
. /lib/lsb/init-functions

# Check prerequisites
test -x "$DAEMON" || { log_failure_msg "$DAEMON not found or not executable"; exit 1; }
test -r "$CONFIG_FILE" || { log_failure_msg "Config file $CONFIG_FILE not found or not readable"; exit 1; }

# Function to start the service
do_start() {
    # Check if runtime directory exists, create if not
    if [ ! -d "$RUN_BASE_DIR" ]; then
        mkdir -p "$RUN_BASE_DIR" || { log_failure_msg "Failed to create $RUN_BASE_DIR"; return 1; }
    fi

    # Check if already running
    if [ -f "$PIDFILE" ] && kill -0 $(cat "$PIDFILE") 2>/dev/null; then
        log_progress_msg "$DESC is already running."
        return 1 # Already running, return non-zero
    fi

    # Start the daemon using start-stop-daemon
    log_progress_msg "Starting $DESC process..."
    start-stop-daemon --start --quiet --pidfile "$PIDFILE" --make-pidfile \
        --background --exec "$DAEMON" -- $DAEMON_ARGS > /dev/null 2>&1
    RETVAL=$?

    # Give the daemon a moment to potentially fail immediately
    sleep 1
    if ! start-stop-daemon --status --pidfile "$PIDFILE" > /dev/null; then
        log_failure_msg "Daemon failed to start. Check config or run manually for errors."
        rm -f "$PIDFILE" # Clean up PID file if daemon died
        return 1 # Indicate failure
    fi

    if [ $RETVAL -eq 0 ]; then
        log_success_msg "$DESC started successfully."
        return 0
    else
        log_failure_msg "Failed to start $DESC (exit code $RETVAL)."
        rm -f "$PIDFILE"
        return 1
    fi
}

# Function to stop the service
do_stop() {
    log_progress_msg "Stopping $DESC process..."
    # Use TERM signal first, wait, then KILL if necessary
    start-stop-daemon --stop --quiet --retry=TERM/15/KILL/5 --pidfile "$PIDFILE"
    STOP_RETVAL=$?

    # Handle stop results
    case "$STOP_RETVAL" in
        0) # Stop command successful (process stopped or wasn't running)
            rm -f "$PIDFILE"
            log_success_msg "$DESC stopped successfully."
            return 0
            ;;
        1) # Process not found based on PID file
            log_progress_msg "(Process not found or already stopped)"
            rm -f "$PIDFILE" # Clean up potentially stale PID file
            return 0
            ;;
        *) # Error stopping
            log_failure_msg "Failed to stop process cleanly (exit code $STOP_RETVAL)."
            rm -f "$PIDFILE" # Attempt to remove PID file even on failure
            return 1
            ;;
    esac
}

# Function to check status
do_status() {
    if [ -f "$PIDFILE" ]; then
        PID=$(cat "$PIDFILE")
        if ps -p "$PID" > /dev/null 2>&1 && grep -q "$DAEMON" "/proc/$PID/cmdline"; then
            log_success_msg "$DESC is running with PID $PID."
            return 0
        else
            log_failure_msg "$DESC is dead but PID file exists (PID: $PID)."
            return 1
        fi
    else
        log_failure_msg "$DESC is not running."
        return 3
    fi
}

# --- Main logic ---
case "$1" in
    start)
        log_daemon_msg "Starting $DESC ($NAME)..."
        do_start
        case "$?" in
            0) log_end_msg 0 ;; # Started successfully
            1) log_end_msg 1 ;; # Failed to start or already running
        esac
        ;;
    stop)
        log_daemon_msg "Stopping $DESC ($NAME)..."
        do_stop
        case "$?" in
            0) log_end_msg 0 ;; # Stopped successfully
            1) log_end_msg 1 ;; # Failed to stop
        esac
        ;;
    status)
        do_status
        ;;
    restart|force-reload)
        log_daemon_msg "Restarting $DESC ($NAME)..."
        do_stop
        STOP_STATUS=$?
        if [ $STOP_STATUS -eq 0 ] || [ $STOP_STATUS -eq 1 ]; then
            sleep 1
            do_start
            START_STATUS=$?
            if [ $START_STATUS -eq 0 ]; then
                log_end_msg 0 # Restart successful
            else
                log_end_msg 1 # Start failed during restart
            fi
        else
            log_end_msg 1 # Stop failed, cannot restart
        fi
        ;;
    *)
        echo "Usage: $SCRIPTNAME {start|stop|status|restart|force-reload}" >&2
        exit 3
        ;;
esac

exit 0
```

---

### **主要改进点**
1. **移除 `/etc/default/$NAME`**：
   - 删除了对默认环境变量文件的依赖，简化了逻辑。

2. **返回值逻辑统一**：
   - 统一返回值逻辑，使 `0` 表示成功，非零表示失败。

3. **直接调用守护进程**：
   - 使用 `--exec` 参数直接调用 `DAEMON`，避免通过 `/bin/bash` 启动子进程。

4. **增强 PID 文件处理**：
   - 在检查 PID 文件时，增加对 PID 内容的校验，确保其对应一个有效的进程。

5. **日志输出优化**：
   - 统一日志输出格式，避免重复或混淆。

6. **注释完善**：
   - 为关键逻辑添加详细注释，便于后续维护。

---

### **验证方法**
1. **安装脚本**：
   将脚本保存为 `/etc/init.d/xray`，并赋予可执行权限：
   ```bash
   sudo cp script.sh /etc/init.d/xray
   sudo chmod +x /etc/init.d/xray
   ```

2. **测试功能**：
   - 启动服务：
     ```bash
     sudo /etc/init.d/xray start
     ```
   - 检查状态：
     ```bash
     sudo /etc/init.d/xray status
     ```
   - 停止服务：
     ```bash
     sudo /etc/init.d/xray stop
     ```

3. **设置开机自启**：
   在基于 SysVinit 的系统上，启用开机自启：
   ```bash
   sudo update-rc.d xray defaults
   ```

---

### **总结**
该脚本结构清晰、逻辑健壮，适合直接管理 `xray` 服务。它移除了不必要的复杂性，同时增强了安全性和可维护性。建议在实际部署前进行充分测试，并根据需要调整配置文件路径或其他参数。如果有其他问题或需要进一步的帮助，请随时告诉我！
