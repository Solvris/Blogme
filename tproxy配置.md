以下是根据你的需求整理的完整文档，包括 `tproxy` 模式防火墙配置和相关脚本。文档采用 Markdown 格式编写，便于阅读和维护。

---

# **TProxy 防火墙配置与启动/关停脚本**

## **1. 概述**
TProxy 是一种透明代理模式，适用于处理经过代理工具（如 `mihomo`、`sing-box` 或 `v2ray`）的流量。以下脚本和配置文件实现了 TProxy 的启动、关停以及防火墙规则设置，并确保经过代理处理的流量被打上标记 `255`。

---

## **2. 启动脚本 (`tproxy-up.sh`)**

以下脚本用于启动 TProxy 模式并加载防火墙规则。

```sh
#!/bin/sh
# tproxy-up.sh: 启动 TProxy 模式

# 添加路由规则和表
ip rule add fwmark 1 table 100
ip route add local 0.0.0.0/0 dev lo table 100

# 加载 nftables 规则
nft -f /etc/tproxy-nf.nft

echo "TProxy 已启动并加载防火墙规则。"
```

---

## **3. 关停脚本 (`tproxy-down.sh`)**

以下脚本用于关闭 TProxy 模式并清理防火墙规则。

```sh
#!/bin/sh
# tproxy-down.sh: 关闭 TProxy 模式

# 删除路由规则和表
ip rule del fwmark 1 table 100
ip route del local 0.0.0.0/0 dev lo table 100

# 清理 nftables 表
nft delete table ip tproxy4
nft delete table ip tproxy6

echo "TProxy 已关闭并清理防火墙规则。"
```

---

## **4. 防火墙配置 (`tproxy-nf.nft`)**

以下是一个完整的 `nftables` 配置文件，用于定义 TProxy 的规则。

```nft
#!/usr/sbin/nft -f
# tproxy-nf.nft: 定义 TProxy 的 nftables 规则

table ip tproxy4 {
    # 定义不需要代理的 IP 地址集合
    set NOT_TPROXY_IP {
        type ipv4_addr
        flags interval
        elements = {
            10.8.8.0/24,          # LAN 网段
            192.168.100.0/24,     # 另一个 LAN 网段
            224.0.0.0/3,          # 多播地址
            127.0.0.0/8           # 本地回环地址
        }
    }

    # 入站流量链 (PREROUTING)
    chain TPROXY_IN {
        ip daddr 255.255.255.255 return       # 广播地址直接返回
        ip daddr @NOT_TPROXY_IP th dport != 53 return  # 排除指定网段和非 DNS 流量
        ip protocol { tcp, udp } meta mark set 0x00000001 tproxy to 127.0.0.1:7893
    }

    chain PREROUTING {
        type filter hook prerouting priority mangle; policy accept;
        jump TPROXY_IN
    }

    # 本机流量链 (OUTPUT)
    chain TPROXY_SELF {
        ip daddr 255.255.255.255 return       # 广播地址直接返回
        ip daddr @NOT_TPROXY_IP return        # 排除指定网段
        meta mark 0x000000ff return           # 已标记为 255 的流量直接返回
        meta mark set 0x1                     # 标记流量为 1
    }

    chain OUTPUT {
        type route hook output priority mangle; policy accept;
        ip protocol { tcp, udp } jump TPROXY_SELF
    }
}

table ip tproxy6 {
    # IPv6 规则可以根据需要添加
}
```

---

## **5. 使用 `iptables` 的替代脚本**

如果你更倾向于使用 `iptables` 而不是 `nftables`，可以使用以下脚本：

### **启动脚本 (`tproxy-iptables-up.sh`)**

```sh
#!/bin/sh
# tproxy-iptables-up.sh: 使用 iptables 启动 TProxy 模式

# 创建自定义链
iptables -t mangle -N PROXY
iptables -t mangle -A PROXY -d 255.255.255.255/32 -j RETURN
iptables -t mangle -A PROXY -d 10.8.8.0/24 -j RETURN
iptables -t mangle -A PROXY -d 192.168.12.0/24 -j RETURN
iptables -t mangle -A PROXY -p tcp -j TPROXY --on-port 2727 --on-ip 127.0.0.1 --tproxy-mark 1
iptables -t mangle -A PROXY -p udp -j TPROXY --on-port 2727 --on-ip 127.0.0.1 --tproxy-mark 1
iptables -t mangle -A PREROUTING -j PROXY

# 代理本机流量
iptables -t mangle -N PROXY_SELF
iptables -t mangle -A PROXY_SELF -d 255.255.255.255/32 -j RETURN
iptables -t mangle -A PROXY_SELF -d 10.8.8.0/24 -j RETURN
iptables -t mangle -A PROXY_SELF -d 192.168.12.0/24 -j RETURN
iptables -t mangle -A PROXY_SELF -m mark --mark 0xff -j RETURN
iptables -t mangle -A PROXY_SELF -j MARK --set-mark 1
iptables -t mangle -A OUTPUT -j PROXY_SELF

echo "TProxy 已通过 iptables 启动并加载防火墙规则。"
```

### **关停脚本 (`tproxy-iptables-down.sh`)**

```sh
#!/bin/sh
# tproxy-iptables-down.sh: 使用 iptables 关闭 TProxy 模式

# 清理自定义链
iptables -t mangle -D PREROUTING -j PROXY
iptables -t mangle -F PROXY
iptables -t mangle -X PROXY

iptables -t mangle -D OUTPUT -j PROXY_SELF
iptables -t mangle -F PROXY_SELF
iptables -t mangle -X PROXY_SELF

echo "TProxy 已通过 iptables 关闭并清理防火墙规则。"
```

---

## **6. 注意事项**

1. **标记流量**：
   - 经过代理工具（如 `mihomo`、`sing-box` 或 `v2ray`）处理的流量应被打上标记 `255`，以避免循环代理。
   - 在 `nftables` 和 `iptables` 中，分别通过 `meta mark` 和 `MARK` 实现。

2. **IP 地址排除**：
   - 确保将本地网络（如 `10.8.8.0/24` 和 `192.168.100.0/24`）以及广播地址（如 `255.255.255.255`）排除在代理范围之外。

3. **端口配置**：
   - `tproxy` 的监听端口（如 `7893` 或 `2727`）需与代理工具的配置一致。

4. **兼容性**：
   - 如果系统内核版本较低，可能需要安装或升级 `nftables` 或 `iptables`。

---

## **7. 总结**

以上脚本和配置文件提供了一个完整的解决方案，用于实现基于 TProxy 的透明代理模式。你可以根据实际需求选择使用 `nftables` 或 `iptables`，并调整 IP 地址和端口配置以适配你的网络环境。如果有其他问题或需要进一步的帮助，请随时告诉我！
