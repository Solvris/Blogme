根据您的需求，我将脚本修改为使用 `_v2ray` 用户运行 Xray 服务，并引入了一个 `USER` 变量来管理用户信息。以下是更新后的 SysVinit 脚本：

---

### **更新后的 SysVinit 脚本**

```bash
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
DAEMON=/usr/local/bin/xray              # Path to the xray binary
CONFIG_FILE=/usr/local/etc/xray/config.json # Path to the config file
SCRIPTNAME=/etc/init.d/$NAME            # Path to this script
USER=_v2ray                             # User to run Xray as

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

    # Ensure the runtime directory is owned by the specified user
    chown "$USER" "$RUN_BASE_DIR" || { log_failure_msg "Failed to set ownership of $RUN_BASE_DIR to $USER"; return 1; }

    # Check if already running
    if [ -f "$PIDFILE" ] && kill -0 $(cat "$PIDFILE") 2>/dev/null; then
        log_progress_msg "$DESC is already running."
        return 1 # Already running, return non-zero
    fi

    # Start the daemon using start-stop-daemon with --chuid
    log_progress_msg "Starting $DESC process as user '$USER'..."
    start-stop-daemon --start --quiet --pidfile "$PIDFILE" --make-pidfile \
        --background --exec "$DAEMON" --chuid "$USER" -- $DAEMON_ARGS > /dev/null 2>&1
    RETVAL=$?

    # Give the daemon a moment to potentially fail immediately
    sleep 1
    if ! start-stop-daemon --status --pidfile "$PIDFILE" > /dev/null; then
        log_failure_msg "Daemon failed to start. Check config or run manually for errors."
        rm -f "$PIDFILE" # Clean up PID file if daemon died
        return 1 # Indicate failure
    fi

    if [ $RETVAL -eq 0 ]; then
        log_success_msg "$DESC started successfully as user '$USER'."
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

### **主要修改点**

1. **引入 `USER` 变量**
   - 定义了一个变量 `USER` 来存储运行 Xray 的用户名（默认为 `_v2ray`）：
     ```bash
     USER=_v2ray
     ```
   - 使用 `$USER` 替代硬编码的 `_v2ray` 用户名。

2. **设置运行目录权限**
   - 在 `do_start` 函数中，确保运行目录 `/var/run/xray` 对 `$USER` 用户具有写权限：
     ```bash
     chown "$USER" "$RUN_BASE_DIR" || { log_failure_msg "Failed to set ownership of $RUN_BASE_DIR to $USER"; return 1; }
     ```

3. **启动时指定用户**
   - 使用 `--chuid` 参数让 `start-stop-daemon` 以 `$USER` 用户身份运行 Xray：
     ```bash
     start-stop-daemon --start --quiet --pidfile "$PIDFILE" --make-pidfile \
         --background --exec "$DAEMON" --chuid "$USER" -- $DAEMON_ARGS > /dev/null 2>&1
     ```

4. **日志输出**
   - 在启动时明确记录以哪个用户身份运行服务：
     ```bash
     log_progress_msg "Starting $DESC process as user '$USER'..."
     ```

5. **兼容性**
   - 如果需要更改运行用户，只需修改 `USER` 变量即可，无需修改其他代码。

---

### **注意事项**

1. **用户权限**
   - 确保 `_v2ray` 用户已正确创建，并具有运行 Xray 所需的权限。
   - 如果需要运行特权操作（如绑定端口 < 1024），可以通过 `setcap` 或其他方式赋予 `_v2ray` 特定权限。

2. **配置文件权限**
   - 确保 `_v2ray` 用户对 `/usr/local/etc/xray` 配置目录具有读取权限：
     ```bash
     sudo chown -R _v2ray:_v2ray /usr/local/etc/xray
     sudo chmod -R 750 /usr/local/etc/xray
     ```

3. **测试脚本**
   - 测试脚本功能：
     ```bash
     sudo /etc/init.d/xray start
     sudo /etc/init.d/xray status
     sudo /etc/init.d/xray stop
     sudo /etc/init.d/xray restart
     ```

---

### **总结**

通过引入 `USER` 变量，脚本变得更加灵活和易于维护。您可以轻松地通过修改 `USER` 变量来更改运行 Xray 的用户，而无需修改其他代码。如果您有任何其他问题或需要进一步调整，请随时告诉我！ 😊
