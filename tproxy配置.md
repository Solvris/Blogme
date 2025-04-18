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
#! /usr/sbin/nft
table ip tproxy4 {
        set BROASTCAST_IP {
                type ipv4_addr
                flags interval
                elements = {
#                    255.255.255.255/32,
                    224.0.0.0/3,
                    127.0.0.0/8,
                    }#此处必须要有wan和lan口的ip
        }
        set SELF_IP {
                type ipv4_addr
                flags interval
                elements = {
                    10.8.8.0/24,
                    192.168.100.0/24,
                    }#此处必须要有wan和lan口的ip
        }
        chain TPROXY_IN {
                ip daddr 255.255.255.255 return #低版本nftables不支持把当个地址放在集合里
                ip daddr @BROADCAST_IP return
                ip daddr @SELF_IP  th dport != 53 return #如果并不是dns则返回,dns则匹配下一条走代理
                ip protocol { tcp , udp } meta mark set 0x00000001 tproxy to 127.0.0.1:7893
        }

        chain PREROUTING {
                type filter hook prerouting priority mangle; policy accept;
                 ip protocol { tcp , udp } jump TPROXY_IN
        }

        chain TPROXY_SELF {
                ip daddr 255.255.255.255 return
                ip daddr @BROADCAST_IP return
                ip daddr @SELF_IP  th dport != 53 return
                meta mark 0x000000ff return
                meta mark set 0x00000001
        }

        chain OUTPUT {
                type route hook output priority mangle; policy accept;
                ip protocol { tcp , udp } jump TPROXY_SELF
        }
}
table ip tproxy6 {
  
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

## **7. 使用gid避免流量环回**
好的！以下是按照您的要求，将标题改为 `7.1`、`7.2`、`7.3` 等格式的完整教程：

---

## **透明代理配置教程**

本教程旨在通过 `nftables` 或 `iptables` 实现透明代理，使用 UID 和 GID 区分流量，避免代理流量环回问题，并确保规则简洁高效。

---

### **7.1 创建组和用户**

#### **步骤**
创建一个 GID 为 `998` 的组（`_v2ray`），并创建一个 UID 为 `0`、GID 为 `998` 的系统用户（`_v2ray`）来运行代理服务：

```bash
sudo groupadd -g 998 _v2ray
sudo useradd -r -o -u 0 -g _v2ray -s /usr/sbin/nologin -M _v2ray
```

#### **验证**
使用以下命令验证用户的 UID 和 GID 是否正确：
```bash
id _v2ray
```
输出示例：
```
uid=0(_v2ray) gid=998(_v2ray) groups=998(_v2ray)
```

---

### **7.2 策略路由设置**

透明代理需要通过策略路由将标记为 `1` 的流量路由到本地回环接口。

#### **步骤**
编辑 `/etc/iproute2/rt_tables` 文件，添加自定义路由表 `100`：
```bash
echo "100 custom_table" | sudo tee -a /etc/iproute2/rt_tables
```

然后添加策略路由规则：
```bash
sudo ip rule add fwmark 1 table 100
sudo ip route add local 0.0.0.0/0 dev lo table 100
```

#### **验证**
使用以下命令验证策略路由是否生效：
```bash
ip rule show
ip route show table 100
```

---

### **7.3 配置 nftables 防火墙规则**

以下是优化后的 `nftables` 规则文件，确保逻辑清晰且无冲突。

#### **规则文件**
保存为 `tproxy-nf.nft`：

```bash
#!/usr/sbin/nft -f

# 定义 TProxy 的 nftables 规则
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
        # 广播地址直接返回
        ip daddr 255.255.255.255 return

        # 排除指定网段
        ip daddr @NOT_TPROXY_IP return

        # 放行 DNS 流量
        ip protocol udp udp dport 53 return
        ip protocol tcp tcp dport 53 return

        # 对 TCP 和 UDP 流量进行 TProxy 转发
        ip protocol { tcp, udp } meta mark set 1 tproxy to 127.0.0.1:7893
    }

    chain PREROUTING {
        type filter hook prerouting priority mangle; policy accept;
        jump TPROXY_IN
    }

    # 本机流量链 (OUTPUT)
    chain TPROXY_SELF {
        # 广播地址直接返回
        ip daddr 255.255.255.255 return

        # 排除指定网段
        ip daddr @NOT_TPROXY_IP return

        # 匹配特定 GID 的流量，直接返回（避免代理）
        meta skgid 998 return  # GID 为 998 的流量直接返回

        # 标记其他流量为 1
        meta mark set 1
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

#### **加载规则**
将规则加载到系统中：
```bash
sudo nft -f tproxy-nf.nft
```

#### **验证规则**
使用以下命令查看加载的规则：
```bash
sudo nft list ruleset
```

---

### **7.4 配置 iptables 防火墙规则**

以下是优化后的 `iptables` 规则脚本，确保逻辑清晰且无冲突。

#### **脚本内容**
保存为 `tproxy-iptables-up.sh`：

```bash
#!/bin/sh

# 清理旧规则
iptables -t mangle -F
iptables -t mangle -X

# 创建自定义链
iptables -t mangle -N PROXY
iptables -t mangle -N PROXY_SELF

# PREROUTING 链：处理入站流量
iptables -t mangle -A PROXY -d 255.255.255.255/32 -j RETURN
iptables -t mangle -A PROXY -d 10.8.8.0/24 -j RETURN
iptables -t mangle -A PROXY -d 192.168.12.0/24 -j RETURN
iptables -t mangle -A PROXY -p udp --dport 53 -j RETURN
iptables -t mangle -A PROXY -p tcp --dport 53 -j RETURN
iptables -t mangle -A PROXY -p tcp -j TPROXY --on-port 7893 --on-ip 127.0.0.1 --tproxy-mark 1
iptables -t mangle -A PROXY -p udp -j TPROXY --on-port 7893 --on-ip 127.0.0.1 --tproxy-mark 1
iptables -t mangle -A PREROUTING -j PROXY

# OUTPUT 链：处理本机流量
iptables -t mangle -A PROXY_SELF -d 255.255.255.255/32 -j RETURN
iptables -t mangle -A PROXY_SELF -d 10.8.8.0/24 -j RETURN
iptables -t mangle -A PROXY_SELF -d 192.168.12.0/24 -j RETURN
iptables -t mangle -A PROXY_SELF -m owner --gid-owner 998 -j RETURN
iptables -t mangle -A PROXY_SELF -m mark --mark 1 -j RETURN
iptables -t mangle -A PROXY_SELF -j MARK --set-mark 1
iptables -t mangle -A OUTPUT -j PROXY_SELF

echo "TProxy 已通过 iptables 启动并加载防火墙规则。"
```

#### **加载规则**
赋予脚本执行权限并运行：
```bash
chmod +x tproxy-iptables-up.sh
sudo ./tproxy-iptables-up.sh
```

---

### **7.5 启动代理服务**

确保代理服务（如 V2Ray 或 Xray）以 `_v2ray` 用户运行，并监听指定端口（如 `127.0.0.1:7893`）：

```bash
sudo -u _v2ray v2ray run
```

---

### **7.6 清理规则（可选）**

在测试阶段，清理规则非常重要，以避免重复添加规则导致冲突。可以使用以下脚本清理所有相关规则。

#### **清理 nftables**
```bash
sudo nft flush ruleset
sudo ip rule del fwmark 1 table 100
sudo ip route del local 0.0.0.0/0 dev lo table 100
```

#### **清理 iptables**
```bash
sudo iptables -t mangle -F
sudo iptables -t mangle -X
```

---

### **总结**

通过以上步骤，您可以实现以下目标：
1. **创建特殊用户**：UID 为 `0`、GID 为 `998` 的用户，用于运行代理服务。
2. **配置策略路由**：通过 `fwmark 1` 实现流量分流。
3. **配置防火墙规则**：通过 `nftables` 或 `iptables` 匹配 GID 并避免环回，同时保留 `MARK` 标记为 1 的逻辑。
4. **启动代理服务**：确保代理服务以特殊用户运行，并监听指定端口。

希望这次的教程能够完全满足您的需求！如果您需要进一步打包成 systemd 单元或启动脚本，请告诉我，我可以为您完善这部分内容！ 😊

