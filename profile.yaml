#全局模块
#端口配置
mixed-port: 7890
allow-lan: true
bind-address: '*'
mode: rule
ipv6: true
log-level: debug
external-controller: '127.0.0.1:9090'
#允许连接的 IP 地址段
lan-allowed-ips:
  - 0.0.0.0/0
  - ::/0
#数据库
geodata-mode: true
geo-auto-update: true # 是否自动更新 geodata
geo-update-interval: 24 # 更新间隔，单位：小时
geodata-loader: standard
#自定义 geodata url
geox-url:
  geoip: "https://github.com/MetaCubeX/meta-rules-dat/releases/download/latest/geoip.dat"
  geosite: "https://github.com/MetaCubeX/meta-rules-dat/releases/download/latest/geosite.dat"
  mmdb: "https://github.com/MetaCubeX/meta-rules-dat/releases/download/latest/country.mmdb"

#控制面板
#external-controller: 0.0.0.0:9093 
#scert:
#external-ui: /path/to/ui/folder/
#external-ui-name: xd
#external-ui-url: "https://github.com/MetaCubeX/metacubexd/archive/refs/heads/gh-pages.zip"

#DNS模块
dns:
    enable: true
    ipv6: true
    enhanced-mode: fake-ip
    fake-ip-range: 198.18.0.1/16
    use-hosts: true
    #遵循流量规则
    respect-rules: true
    #三者配置中的dns服务器如果出现域名会采用default-nameserver配置项解析
    default-nameserver: ['https://120.53.53.53/dns-query', 'https://223.5.5.5/dns-query']
    #代理节点域名解析服务器，仅用于解析代理节点的域名，如果不填则遵循nameserver-policy、nameserver和fallback的配置
    proxy-server-nameserver: [223.5.5.5, 119.29.29.29]
    nameserver-policy:
        "rule-set:cn-domain":
        - https://120.53.53.53/dns-query
        - https://223.5.5.5/dns-query
        "rule-set:gfw-domain":
          - https://1.1.1.1/dns-query
          - https://8.8.8.8/dns-query
        "rule-set:not-cn-domain":
        - https://1.1.1.1/dns-query
        - https://8.8.8.8/dns-query
        "geosite:cn":
          - https://120.53.53.53/dns-query
          - https://223.5.5.5/dns-query

    nameserver: ['https://120.53.53.53/dns-query', 'https://223.5.5.5/dns-query']
    #用于direct出口域名解析的 DNS 服务器，如果不填则遵循nameserver-policy、nameserver和fallback的配置
    direct-nameserver: ['https://120.53.53.53/dns-query', 'https://223.5.5.5/dns-query']
    fallback: ['https://1.1.1.1/dns-query', 'https://8.8.8.8/dns-query','https://doh.dns.sb/dns-query']
    fallback-filter: 
      geoip: true
      geoip-code: CN
      geosite:
        - gfw
        - geolocation-!cn


#tun模块
tun:
  enable: true
  stack: mixed
  auto-route: true
  auto-redirect: true
  auto-detect-interface: true
  dns-hijack:
    - any:53
    - tcp://any:53
  device: utun0
  mtu: 9000
  strict-route: true
  gso: true
  gso-max-size: 65536
  udp-timeout: 300
  iproute2-table-index: 2022
  iproute2-rule-index: 9000
  endpoint-independent-nat: false
#流量嗅探模块
sniffer:
  enable: false
  force-dns-mapping: true
  parse-pure-ip: true
  override-destination: false
  sniff:
    HTTP:
      ports: [80, 8080-8880]
      override-destination: true
    TLS:
      ports: [443, 8443]
    QUIC:
      ports: [443, 8443]



proxies:
    - { name: '剩余流量：60.5 GB', type: ss, server: hk01.iopen.cloud, port: 10716, cipher: chacha20-ietf-poly1305, password: c415223c-7637-4a7d-b8e2-289d6eedf042, udp: true }
    - { name: '距离下次重置剩余：24 天', type: ss, server: hk01.iopen.cloud, port: 10716, cipher: chacha20-ietf-poly1305, password: c415223c-7637-4a7d-b8e2-289d6eedf042, udp: true }
    - { name: 套餐到期：2025-02-28, type: ss, server: hk01.iopen.cloud, port: 10716, cipher: chacha20-ietf-poly1305, password: c415223c-7637-4a7d-b8e2-289d6eedf042, udp: true }
    - { name: 香港-01-隧道, type: ss, server: hk01.iopen.cloud, port: 10716, cipher: chacha20-ietf-poly1305, password: c415223c-7637-4a7d-b8e2-289d6eedf042, udp: true }
    - { name: 香港x2-02-隧道, type: ss, server: hk02.iopen.cloud, port: 10717, cipher: chacha20-ietf-poly1305, password: c415223c-7637-4a7d-b8e2-289d6eedf042, udp: true }
    - { name: 香港-04-隧道, type: ss, server: hytron.iopen.cloud, port: 29874, cipher: aes-128-gcm, password: c415223c-7637-4a7d-b8e2-289d6eedf042, udp: true }
    - { name: 香港-05-隧道, type: ss, server: hytron.iopen.cloud, port: 29875, cipher: aes-128-gcm, password: c415223c-7637-4a7d-b8e2-289d6eedf042, udp: true }
    - { name: 香港-06-隧道, type: ss, server: hytron.iopen.cloud, port: 29876, cipher: aes-128-gcm, password: c415223c-7637-4a7d-b8e2-289d6eedf042, udp: true }
    - { name: 韩国春川-02, type: ss, server: seo-group1.iopen.cloud, port: 19709, cipher: chacha20-ietf-poly1305, password: c415223c-7637-4a7d-b8e2-289d6eedf042, udp: true }
    - { name: 韩国春川-03, type: ss, server: seo-group1.iopen.cloud, port: 19710, cipher: chacha20-ietf-poly1305, password: c415223c-7637-4a7d-b8e2-289d6eedf042, udp: true }
    - { name: 韩国春川-04, type: ss, server: seo-group2.iopen.cloud, port: 41256, cipher: chacha20-ietf-poly1305, password: c415223c-7637-4a7d-b8e2-289d6eedf042, udp: true }
    - { name: 韩国春川-05, type: ss, server: seo-group2.iopen.cloud, port: 48420, cipher: chacha20-ietf-poly1305, password: c415223c-7637-4a7d-b8e2-289d6eedf042, udp: true }
    - { name: 新加坡-05-隧道, type: ss, server: xjpgroup1.iopen.cloud, port: 12548, cipher: chacha20-ietf-poly1305, password: c415223c-7637-4a7d-b8e2-289d6eedf042, udp: true }
    - { name: 新加坡-06-隧道, type: ss, server: xjpgroup3.iopen.cloud, port: 28243, cipher: aes-128-gcm, password: c415223c-7637-4a7d-b8e2-289d6eedf042, udp: true }
    - { name: 新加坡-07-中转, type: ss, server: xjpgroup3.iopen.cloud, port: 28242, cipher: aes-128-gcm, password: c415223c-7637-4a7d-b8e2-289d6eedf042, udp: true }
    - { name: 新加坡-08-中转, type: ss, server: xjpgroup3.iopen.cloud, port: 28241, cipher: aes-128-gcm, password: c415223c-7637-4a7d-b8e2-289d6eedf042, udp: true }
    - { name: 大版X2-01-隧道, type: ss, server: db-01.iopen.cloud, port: 12157, cipher: chacha20-ietf-poly1305, password: c415223c-7637-4a7d-b8e2-289d6eedf042, udp: true }
    - { name: 东京x2-02-隧道, type: ss, server: db-01.iopen.cloud, port: 28641, cipher: chacha20-ietf-poly1305, password: c415223c-7637-4a7d-b8e2-289d6eedf042, udp: true }
    - { name: 首尔X2-01-中转, type: vmess, server: se-01.iopen.cloud, port: 62368, uuid: c415223c-7637-4a7d-b8e2-289d6eedf042, alterId: 0, cipher: auto, udp: true, tls: true, network: ws, ws-opts: { path: /api, headers: { Host: aliyun4.iopen.cloud } }, ws-path: /api, ws-headers: { Host: aliyun4.iopen.cloud } }
    - { name: 洛杉矶-01-中转2, type: ss, server: group1.iopen.cloud, port: 27066, cipher: chacha20-ietf-poly1305, password: c415223c-7637-4a7d-b8e2-289d6eedf042, udp: true }
    - { name: 洛杉矶-02-中转, type: ss, server: los.iopen.cloud, port: 46410, cipher: chacha20-ietf-poly1305, password: c415223c-7637-4a7d-b8e2-289d6eedf042, udp: true }
    - { name: 洛杉矶-03-中转2, type: ss, server: hk.iopen.cloud, port: 21676, cipher: chacha20-ietf-poly1305, password: c415223c-7637-4a7d-b8e2-289d6eedf042, udp: true }
    - { name: 旧金山-04-中转, type: ss, server: hk.iopen.cloud, port: 21677, cipher: chacha20-ietf-poly1305, password: c415223c-7637-4a7d-b8e2-289d6eedf042, udp: true }
    - { name: 洛杉矶-04-中转, type: ss, server: los-04.iopen.cloud, port: 28870, cipher: chacha20-ietf-poly1305, password: c415223c-7637-4a7d-b8e2-289d6eedf042, udp: true }
    - { name: 洛杉矶-05-中转, type: ss, server: los-04.iopen.cloud, port: 28875, cipher: chacha20-ietf-poly1305, password: c415223c-7637-4a7d-b8e2-289d6eedf042, udp: true }
    - { name: 荷兰X0.5-03-中转, type: vmess, server: hk.iopen.cloud, port: 21675, uuid: c415223c-7637-4a7d-b8e2-289d6eedf042, alterId: 0, cipher: auto, udp: true, tls: true, network: ws, ws-opts: { path: /api, headers: { Host: api10.iopen.cloud } }, ws-path: /api, ws-headers: { Host: api10.iopen.cloud } }
    - { name: 荷兰X0.5-01-中转, type: vmess, server: hk.iopen.cloud, port: 21674, uuid: c415223c-7637-4a7d-b8e2-289d6eedf042, alterId: 0, cipher: auto, udp: true, tls: true, network: ws, ws-opts: { path: /api, headers: { Host: api6.iopen.cloud } }, ws-path: /api, ws-headers: { Host: api6.iopen.cloud } }
    - { name: 法兰克福-隧道, type: vmess, server: group1.iopen.cloud, port: 29843, uuid: c415223c-7637-4a7d-b8e2-289d6eedf042, alterId: 0, cipher: auto, udp: true, tls: true, network: ws, ws-opts: { path: /api, headers: { Host: node1.iopen.cloud } }, ws-path: /api, ws-headers: { Host: node1.iopen.cloud } }
    - { name: 荷兰X0.5-02-中转, type: vmess, server: group1.iopen.cloud, port: 29844, uuid: c415223c-7637-4a7d-b8e2-289d6eedf042, alterId: 0, cipher: auto, udp: true, tls: true, network: ws, ws-opts: { path: /api, headers: { Host: api9.iopen.cloud } }, ws-path: /api, ws-headers: { Host: api9.iopen.cloud } }
    - { name: "直连", type: direct, udp: true}
proxy-groups:
    - { name: REMOTE, type: select, proxies: [自动选择, 故障转移, '剩余流量：60.5 GB', '距离下次重置剩余：24 天', 套餐到期：2025-02-28, 香港-01-隧道, 香港x2-02-隧道, 香港-04-隧道, 香港-05-隧道, 香港-06-隧道, 韩国春川-02, 韩国春川-03, 韩国春川-04, 韩国春川-05, 新加坡-05-隧道, 新加坡-06-隧道, 新加坡-07-中转, 新加坡-08-中转, 大版X2-01-隧道, 东京x2-02-隧道, 首尔X2-01-中转, 洛杉矶-01-中转2, 洛杉矶-02-中转, 洛杉矶-03-中转2, 旧金山-04-中转, 洛杉矶-04-中转, 洛杉矶-05-中转, 荷兰X0.5-03-中转, 荷兰X0.5-01-中转, 法兰克福-隧道, 荷兰X0.5-02-中转] }
    - { name: 自动选择, type: url-test, proxies: ['剩余流量：60.5 GB', '距离下次重置剩余：24 天', 套餐到期：2025-02-28, 香港-01-隧道, 香港x2-02-隧道, 香港-04-隧道, 香港-05-隧道, 香港-06-隧道, 韩国春川-02, 韩国春川-03, 韩国春川-04, 韩国春川-05, 新加坡-05-隧道, 新加坡-06-隧道, 新加坡-07-中转, 新加坡-08-中转, 大版X2-01-隧道, 东京x2-02-隧道, 首尔X2-01-中转, 洛杉矶-01-中转2, 洛杉矶-02-中转, 洛杉矶-03-中转2, 旧金山-04-中转, 洛杉矶-04-中转, 洛杉矶-05-中转, 荷兰X0.5-03-中转, 荷兰X0.5-01-中转, 法兰克福-隧道, 荷兰X0.5-02-中转], url: 'http://www.gstatic.com/generate_204', interval: 86400 }
    - { name: 故障转移, type: fallback, proxies: ['剩余流量：60.5 GB', '距离下次重置剩余：24 天', 套餐到期：2025-02-28, 香港-01-隧道, 香港x2-02-隧道, 香港-04-隧道, 香港-05-隧道, 香港-06-隧道, 韩国春川-02, 韩国春川-03, 韩国春川-04, 韩国春川-05, 新加坡-05-隧道, 新加坡-06-隧道, 新加坡-07-中转, 新加坡-08-中转, 大版X2-01-隧道, 东京x2-02-隧道, 首尔X2-01-中转, 洛杉矶-01-中转2, 洛杉矶-02-中转, 洛杉矶-03-中转2, 旧金山-04-中转, 洛杉矶-04-中转, 洛杉矶-05-中转, 荷兰X0.5-03-中转, 荷兰X0.5-01-中转, 法兰克福-隧道, 荷兰X0.5-02-中转], url: 'http://www.gstatic.com/generate_204', interval: 7200 }

rule-providers:
  cn-domain:
    behavior: domain 
    format: mrs
    interval: 86400
    type: http # http 的 path 可空置，默认储存路径为 homedir 的 rules 文件夹，文件名为 url 的 md5
    url: "https://raw.githubusercontent.com/MetaCubeX/meta-rules-dat/meta/geo/geosite/cn.mrs"
    proxy: DIRECT
  gfw-domain:
    behavior: domain 
    format: mrs
    interval: 86400
    type: http # http 的 path 可空置，默认储存路径为 homedir 的 rules 文件夹，文件名为 url 的 md5
    url: "https://raw.githubusercontent.com/MetaCubeX/meta-rules-dat/meta/geo/geosite/gfw.mrs"
    proxy: DIRECT
  not-cn-domain:
    behavior: domain 
    format: mrs
    interval: 86400
    type: http # http 的 path 可空置，默认储存路径为 homedir 的 rules 文件夹，文件名为 url 的 md5
    url: "https://raw.githubusercontent.com/MetaCubeX/meta-rules-dat/meta/geo/geosite/geolocation-!cn.mrs"
    proxy: DIRECT

  cn-ip:
    behavior: ipcidr 
    format: mrs
    interval: 86400
    type: http # http 的 path 可空置，默认储存路径为 homedir 的 rules 文件夹，文件名为 url 的 md5
    url: "https://raw.githubusercontent.com/MetaCubeX/meta-rules-dat/meta/geo/geoip/cn.mrs"
    proxy: DIRECT
  google-ip:
    behavior: ipcidr 
    format: mrs
    interval: 86400
    type: http # http 的 path 可空置，默认储存路径为 homedir 的 rules 文件夹，文件名为 url 的 md5
    url: "https://raw.githubusercontent.com/MetaCubeX/meta-rules-dat/meta/geo/geoip/google.mrs"
    proxy: DIRECT
  mydirect:
    behavior: classical
    format: text
    interval: 86400
    type: http # http 的 path 可空置，默认储存路径为 homedir 的 rules 文件夹，文件名为 url 的 md5
    url: "https://raw.githubusercontent.com/venus-25/remote_config/refs/heads/main/direct.list"
    proxy: DIRECT
  myremote:
    behavior: classical
    format: text
    interval: 86400
    type: http # http 的 path 可空置，默认储存路径为 homedir 的 rules 文件夹，文件名为 url 的 md5
    url: "https://raw.githubusercontent.com/venus-25/remote_config/refs/heads/main/remote.list"
    proxy: DIRECT
  
#规则模块
rules:
  #规则集解析
    - 'RULE-SET,mydirect,DIRECT'
    - 'RULE-SET,myremote,REMOTE'
    - 'RULE-SET,cn-domain,DIRECT'
    - 'RULE-SET,gfw-domain,REMOTE'
    - 'RULE-SET,not-cn-domain,REMOTE'
    - 'RULE-SET,google-ip,REMOTE,no-resolve'
  #保底规则
    - 'DOMAIN-SUFFIX,cn,DIRECT'
    - 'DOMAIN-KEYWORD,-cn,DIRECT'
  #IP解析
    - 'IP-CIDR,91.108.4.0/22,REMOTE,no-resolve'
    - 'IP-CIDR,91.108.8.0/21,REMOTE,no-resolve'
    - 'IP-CIDR,91.108.16.0/22,REMOTE,no-resolve'
    - 'IP-CIDR,91.108.56.0/22,REMOTE,no-resolve'
    - 'IP-CIDR,149.154.160.0/20,REMOTE,no-resolve'
    - 'IP-CIDR6,2001:67c:4e8::/48,REMOTE,no-resolve'
    - 'IP-CIDR6,2001:b28:f23d::/48,REMOTE,no-resolve'
    - 'IP-CIDR6,2001:b28:f23f::/48,REMOTE,no-resolve'
    - 'IP-CIDR,120.232.181.162/32,REMOTE,no-resolve'
    - 'IP-CIDR,120.241.147.226/32,REMOTE,no-resolve'
    - 'IP-CIDR,120.253.253.226/32,REMOTE,no-resolve'
    - 'IP-CIDR,120.253.255.162/32,REMOTE,no-resolve'
    - 'IP-CIDR,120.253.255.34/32,REMOTE,no-resolve'
    - 'IP-CIDR,120.253.255.98/32,REMOTE,no-resolve'
    - 'IP-CIDR,180.163.150.162/32,REMOTE,no-resolve'
    - 'IP-CIDR,180.163.150.34/32,REMOTE,no-resolve'
    - 'IP-CIDR,180.163.151.162/32,REMOTE,no-resolve'
    - 'IP-CIDR,180.163.151.34/32,REMOTE,no-resolve'
    - 'IP-CIDR,203.208.39.0/24,REMOTE,no-resolve'
    - 'IP-CIDR,203.208.40.0/24,REMOTE,no-resolve'
    - 'IP-CIDR,203.208.41.0/24,REMOTE,no-resolve'
    - 'IP-CIDR,203.208.43.0/24,REMOTE,no-resolve'
    - 'IP-CIDR,203.208.50.0/24,REMOTE,no-resolve'
    - 'IP-CIDR,220.181.174.162/32,REMOTE,no-resolve'
    - 'IP-CIDR,220.181.174.226/32,REMOTE,no-resolve'
    - 'IP-CIDR,220.181.174.34/32,REMOTE,no-resolve'
    - 'IP-CIDR,127.0.0.0/8,DIRECT,no-resolve'
    - 'IP-CIDR,172.16.0.0/12,DIRECT,no-resolve'
    - 'IP-CIDR,192.168.0.0/16,DIRECT,no-resolve'
    - 'IP-CIDR,10.0.0.0/8,DIRECT,no-resolve'
    - 'IP-CIDR,17.0.0.0/8,DIRECT,no-resolve'
    - 'IP-CIDR,100.64.0.0/10,DIRECT,no-resolve'
    - 'IP-CIDR,224.0.0.0/4,DIRECT,no-resolve'
    - 'IP-CIDR6,fe80::/10,DIRECT,no-resolve'
    - 'GEOIP,CN,DIRECT'
    - 'RULE-SET,cn-ip,DIRECT'
    - 'MATCH,REMOTE'
