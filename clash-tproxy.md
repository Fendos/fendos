# clash Linux Tproxy

# 创建clash.service
clash.service
```
[Unit]
Description=Clash Daemon
After=network.target

[Service]
User=root
ExecStart=/usr/local/bin/clash -d /usr/local/etc/clash
Restart=on-failure

[Install]
WantedBy=multi-user.target

```

config.yaml
```
# Transparent proxy server port for Linux (TProxy TCP and TProxy UDP)
tproxy-port: 7893

# 允许局域网的连接（可用来共享代理）
allow-lan: true

# 规则模式：Rule（规则） / Global（全局代理）/ Direct（全局直连）
mode: Rule

# 设置日志输出级别 (默认级别：info，级别越高日志输出量越大，越倾向于调试)
# 四个级别：silent / info / warning / error / debug
log-level: info

# When set to false, resolver won't translate hostnames to IPv6 addresses
ipv6: true

# Clash 的 RESTful API
external-controller: 0.0.0.0:9090

# you can put the static web resource (such as clash-dashboard) to a directory, and clash would serve in `${API}/ui`
# input is a relative path to the configuration directory or an absolute path
#external-ui: dashboard

# Secret for RESTful API (Optional)
#secret: ""

# experimental feature
experimental:
  ignore-resolve-fail: true # ignore dns reslove fail, default value is true

dns:
  enable: true
  listen: 0.0.0.0:53
  ipv6: false
  enhanced-mode: fake-ip
  fake-ip-range: 198.18.0.1/16
  fake-ip-filter: # fake ip white domain list
    - '*.lan'
    - localhost.ptlogin2.qq.com
  default-nameserver:
    - 101.6.6.6
    - 166.111.8.29
  nameserver:
    - 101.6.6.6
    - 166.111.8.29
  fallback:
    - https://101.6.6.6/dns-query
    - https://dns.alidns.com/dns-query

tun:
  enable: false
  macOS-auto-route: true
  macOS-auto-detect-interface: true

# 代理节点
proxies:
- { name: 'HKG-01', type: trojan, server: hkg01.example.com, port: 443, password: password01, sni: hkg01.example.com, udp: true}
- { name: "HKG-02", type: trojan, server: hkg02.example.com, port: 443, password: password02, sni: hkg02.example.com, udp: true}

# 代理组策略
proxy-groups:
- { name: "Proxy", type: select, proxies: ["HKG"] }
- { name: "HKG", type: fallback, proxies: ["HKG-01", "HKG-02"], url: "http://www.msftconnecttest.com/connecttest.txt", interval: 300 }

rule-providers:
  # name: # Provider 名称
  #   type: http # http 或 file
  #   behavior: classical # 或 ipcidr、domain
  #   path: # 文件路径
  #   url: # 只有当类型为 HTTP 时才可用，您不需要在本地空间中创建新文件。
  #   interval: # 自动更新间隔，仅在类型为 HTTP 时可用
    Scholar:
      type: http
      behavior: classical
      path: ./RuleSet/Extra/Scholar.yaml
      url: https://raw.fastgit.org/DivineEngine/Profiles/master/Clash/RuleSet/Extra/Scholar.yaml
      interval: 86400

    PayPal:
      type: http
      behavior: classical
      path: ./RuleSet/Extra/Paypal.yaml
      url: https://raw.fastgit.org/DivineEngine/Profiles/master/Clash/RuleSet/Extra/PayPal.yaml
      interval: 86400

    Streaming:
      type: http
      behavior: classical
      path: ./RuleSet/StreamingMedia/Streaming.yaml
      url: https://raw.fastgit.org/DivineEngine/Profiles/master/Clash/RuleSet/StreamingMedia/Streaming.yaml
      interval: 86400

    Global:
      type: http
      behavior: classical
      path: ./RuleSet/Global.yaml
      url: https://raw.fastgit.org/DivineEngine/Profiles/master/Clash/RuleSet/Global.yaml
      interval: 86400

    China:
      type: http
      behavior: classical
      path: ./RuleSet/China.yaml
      url: https://raw.fastgit.org/DivineEngine/Profiles/master/Clash/RuleSet/China.yaml
      interval: 86400

    ChinaIP:
      type: http
      behavior: ipcidr
      path: ./RuleSet/Extra/ChinaIP.yaml
      url: https://raw.fastgit.org/DivineEngine/Profiles/master/Clash/RuleSet/Extra/ChinaIP.yaml
      interval: 86400

# 规则
rules:
  # 不同于 Surge，后面要用到的策略组要放到前面
  # Global Area Network
  # (Paypal)
  - RULE-SET,PayPal,DIRECT
  # (edu.tw Domain)
  - DOMAIN-SUFFIX,edu.tw,DIRECT
  # (Outlook)
  - DOMAIN-SUFFIX,hotmail.com,Proxy
  - DOMAIN-SUFFIX,outlook.com,Proxy
  - DOMAIN,outlook.office365.com,Proxy
  - DOMAIN,smtp.office365.com,Proxy
  # (Streaming Media)
  - RULE-SET,Streaming,Proxy
  # (DNS Cache Pollution) / (IP Blackhole) / (Region-Restricted Access Denied) / (Network Jitter)
  - RULE-SET,Global,Proxy
  # (Academic)
  - RULE-SET,Scholar,DIRECT

  # China Area Network
  - RULE-SET,China,DIRECT

  # Local Area Network
  - IP-CIDR,192.168.0.0/16,DIRECT
  - IP-CIDR,10.0.0.0/8,DIRECT
  - IP-CIDR,172.16.0.0/12,DIRECT
  - IP-CIDR,127.0.0.0/8,DIRECT
  - IP-CIDR,100.64.0.0/10,DIRECT
  - IP-CIDR,224.0.0.0/4,DIRECT

  # China IP
  - RULE-SET,ChinaIP,DIRECT
  # Tencent
  #- IP-CIDR,119.28.28.28/32,DIRECT
  #- IP-CIDR,182.254.116.0/24,DIRECT
  # GeoIP China
  #- GEOIP,CN,DIRECT

  - MATCH,Proxy

```

## iptables.sh
```
#!/bin/bash

# ROUTE RULES
#ip rule add fwmark 1 table 100
#ip route add local 0.0.0.0/0 dev lo table 100

# CREATE TABLE
iptables -t mangle -N clash

# RETURN LOCAL AND LANS
iptables -t mangle -A clash -d 0.0.0.0/8 -j RETURN
iptables -t mangle -A clash -d 10.0.0.0/8 -j RETURN
iptables -t mangle -A clash -d 127.0.0.0/8 -j RETURN
iptables -t mangle -A clash -d 169.254.0.0/16 -j RETURN
iptables -t mangle -A clash -d 172.16.0.0/12 -j RETURN
iptables -t mangle -A clash -d 192.168.50.0/16 -j RETURN
iptables -t mangle -A clash -d 192.168.9.0/16 -j RETURN

iptables -t mangle -A clash -d 224.0.0.0/4 -j RETURN
iptables -t mangle -A clash -d 240.0.0.0/4 -j RETURN

# RETURN TSINGHUA
#iptables -t mangle -A clash -d 166.111.0.0/16 -j RETURN
#iptables -t mangle -A clash -d 59.66.0.0/16 -j RETURN
#iptables -t mangle -A clash -d 118.229.0.0/19 -j RETURN
#iptables -t mangle -A clash -d 183.172.0.0/16 -j RETURN
#iptables -t mangle -A clash -d 183.173.0.0/16 -j RETURN
#iptables -t mangle -A clash -d 101.5.0.0/16 -j RETURN
#iptables -t mangle -A clash -d 101.6.0.0/16 -j RETURN

# FORWARD ALL
iptables -t mangle -A clash -p udp -j TPROXY --on-port 7893 --tproxy-mark 1
iptables -t mangle -A clash -p tcp -j TPROXY --on-port 7893 --tproxy-mark 1

# HIJACK ICMP (untested)
# iptables -t mangle -A clash -p icmp -j DNAT --to-destination 127.0.0.1

# REDIRECT
iptables -t mangle -A PREROUTING -j clash

exit

```

## tproxy.service
```
[Unit]
Description=Clash TProxy Rule
After=network.target
Wants=network.target

[Service]
User=root
Type=oneshot
RemainAfterExit=yes
# there must be spaces before and after semicolons
ExecStart=/sbin/ip rule add fwmark 1 table 100 ; /sbin/ip route add local 0.0.0.0/0 dev lo table 100 ; /usr/bin/bash /usr/local/etc/clash/iptables.sh
ExecStop=/sbin/ip rule del fwmark 1 table 100 ; /sbin/ip route del local 0.0.0.0/0 dev lo table 100 ; /sbin/iptables -t mangle -F

[Install]
WantedBy=multi-user.target
```
