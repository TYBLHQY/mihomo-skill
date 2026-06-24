---
name: mihomo
description: >-
  Mihomo (虚空终端 / Clash Meta) proxy configuration and usage. Use this skill whenever the user asks about
  configuring Mihomo/Clash Meta/clash.meta, setting up proxy rules, writing config.yaml for Mihomo,
  troubleshooting proxy connections, understanding proxy groups/rules/DNS/TUN modes, or migrating from
  Clash to Mihomo. Also trigger when the user mentions proxy subscription URLs, rule-sets, fake-ip vs
  redir-host, or any of the proxy protocols Mihomo supports (Shadowsocks, VMess, VLESS, Trojan, Hysteria,
  TUIC, WireGuard, etc.). If the user is talking about proxy/VPN configuration on any platform (Windows,
  macOS, Linux, Android, OpenWrt) and the context suggests they're using or should use a Clash-compatible
  client, activate this skill.
compatibility:
  - yaml
  - json
---
# Mihomo (虚空终端 / Clash Meta) 使用指南

Mihomo 是 Clash Meta 的二次开发分支，是一个基于规则的跨平台代理客户端核心。它将来自订阅或本地配置的代理节点通过灵活的规则引擎进行路由分发，支持 TUN 虚拟网卡实现全局透明代理。

官方文档站：<https://wiki.metacubex.one>
源码仓库：<https://github.com/MetaCubeX/mihomo>

> **命名限制:** 任何与 MetaCubeX 无关的下游项目不得在名称中包含 "mihomo"。

---

## 核心概念

| 概念 | 说明 |
|------|------|
| **Proxies** | 代理节点，即一个可连接的上游代理服务器 |
| **Proxy Groups** | 代理组，将多个节点组合并定义选择策略（手动/自动/回退/负载均衡/链式） |
| **Rules** | 路由规则，定义什么样的流量走哪个代理组 |
| **Proxy Providers** | 代理集合，从外部订阅 URL 自动拉取节点列表 |
| **Rule Providers** | 规则集合，从外部 URL 拉取规则片段 |
| **TUN** | 虚拟网卡模式，捕获所有系统流量 |
| **DNS** | 内置 DNS 模块，支持 fake-ip 和 redir-host 两种模式 |

---

## 快速安装

### 下载二进制文件

从 [Releases](https://github.com/MetaCubeX/mihomo/releases) 选择对应平台的预编译二进制：

| 平台 | 架构 |
|------|------|
| Windows | amd64, 386, arm64, armv7 |
| Linux | amd64, 386, arm64, armv7, riscv64, mips |
| macOS | amd64, arm64 |
| FreeBSD | amd64, 386, arm64 |
| Android | arm64 |

### 启动

```bash
# 使用配置文件启动
./mihomo -d /path/to/config/dir -f config.yaml

# -d 指定 Home 目录（配置文件存放位置）
# -f 指定配置文件名（默认 config.yaml）
```

### GUI 客户端

| 平台 | 客户端 |
|------|--------|
| Windows | [Clash.Mini](https://github.com/MetaCubeX/Clash.Mini) |
| macOS | [ClashX.Meta](https://github.com/MetaCubeX/ClashX.Meta) |
| Android | [ClashMetaForAndroid](https://github.com/MetaCubeX/ClashMetaForAndroid/releases/tag/Prerelease-alpha) |

### Web 面板

将面板文件放在 Home 目录的 `ui/` 文件夹，或在配置中设置：

```yaml
external-controller: 127.0.0.1:9090
external-ui: /path/to/dashboard
```

推荐面板：[metacubexd](https://github.com/MetaCubeX/metacubexd)

---

## 配置文件结构

配置文件是 YAML 格式。**不允许使用 Tab 缩进**，必须使用空格。注释以 `#` 开头。

```yaml
# 完整结构概览
port: 7890                    # HTTP 代理端口
socks-port: 7891              # SOCKS5 代理端口
mixed-port: 7890              # 混合代理端口（推荐用这个替代 port + socks-port）
allow-lan: false              # 是否允许局域网访问
mode: rule                    # rule / global / direct
log-level: info               # silent / error / warning / info / debug
ipv6: true                    # 是否处理 IPv6 流量
external-controller: 127.0.0.1:9090  # RESTful API 地址
secret: ""                    # API 密钥

# 各配置段（按需配置）
proxies:                      # 代理节点列表
proxy-groups:                 # 代理组
rules:                        # 路由规则
proxy-providers:              # 代理集合（订阅）
rule-providers:               # 规则集合
dns:                          # DNS 配置
tun:                          # TUN 模式
sniffer:                      # 域名嗅探
tunnels:                      # 隧道转发
ntp:                          # NTP 时间同步
experimental:                 # 实验性功能
```

**重要语法规则：**
- 所有 `key: value` 的冒号后面必须有空格
- 列表元素以 `- ` 开头
- 支持 YAML 锚点 (`&name` / `*name` / `<<: *name`) 减少重复
- 域名字段中的通配符：`*` 匹配一级、`+` 匹配多级（含 apex）、`.` 匹配多级（不含 apex）
- 端口范围：`114-514/810-1919,65530`

**多行 vs 内联写法：** YAML 支持两种等价的列表写法

```yaml
# 多行（标准）
proxies:
  - DIRECT
  - Proxy
  - REJECT

# 内联（紧凑，适合简单列表）
proxies: [DIRECT, Proxy, REJECT]

# 多行对象
- name: 节点选择
  type: select
  proxies:
    - DIRECT
    - Proxy

# 内联对象（紧凑写法）
- { name: 节点选择, type: select, proxies: [DIRECT, Proxy] }
```

---

## 全局配置 (General)

```yaml
# 端口设置（三选一或组合使用）
port: 7890                    # HTTP 代理
socks-port: 7891              # SOCKS5 代理
mixed-port: 7890              # HTTP + SOCKS5 混合

# 局域网访问
allow-lan: false
bind-address: "*"             # 监听地址
lan-allowed-ips:
  - 127.0.0.1/8
  - 10.0.0.0/8
  - 172.16.0.0/12
  - 192.168.0.0/16
  - fe80::/10                  # 链路本地地址
lan-disallowed-ips: []        # 黑名单优先于白名单（有则只允许白名单）

# 认证（HTTP/SOCKS/mixed 代理的用户名密码）
authentication:
  - "user:pass"
skip-auth-prefixes:
  - 127.0.0.1/8
  - "::1/128"

# 运行模式
mode: rule                     # rule: 规则匹配 | global: 全局代理 | direct: 直连

# 日志级别
log-level: info                # silent / error / warning / info / debug

# 网络
ipv6: true
tcp-concurrent: true           # 同时尝试多个 DNS 解析 IP，用第一个成功的
keep-alive-interval: 15        # TCP Keep Alive 间隔（秒）（移动设备节能可改为 1800）
keep-alive-idle: 15            # TCP Keep Alive 空闲时间
disable-keep-alive: false      # Android 上默认强制开启
find-process-mode: strict      # always / strict / off（路由器建议 off）

# TLS 指纹（全局）
global-client-fingerprint: random  # 随机指纹，也可指定 chrome / safari / firefox / ios / android

# API 控制
external-controller: 127.0.0.1:9090  # 或 Unix socket: mihomo.sock
external-controller-tls: 127.0.0.1:9443  # HTTPS API
secret: ""

# 延迟测试
unified-delay: true            # 消除延迟差异

# TLS（仅用于 API HTTPS）
tls:
  certificate: /path/to/cert.pem
  private-key: /path/to/key.pem

# Geo 数据库
geodata-mode: false            # true = .dat 格式 / false = .mmdb 格式
geodata-loader: memconservative # standard / memconservative
geo-auto-update: true
geo-update-interval: 24        # 小时
geox-url:                      # 自定义下载镜像（国内用户建议用镜像）
  geoip: "https://cdn.gh-proxy.org/https://github.com/MetaCubeX/meta-rules-dat/releases/download/latest/geoip.dat"
  geosite: "https://cdn.gh-proxy.org/https://github.com/MetaCubeX/meta-rules-dat/releases/download/latest/geosite.dat"
  mmdb: "https://cdn.gh-proxy.org/https://github.com/MetaCubeX/meta-rules-dat/releases/download/latest/country.mmdb"
  asn: "https://cdn.gh-proxy.org/https://github.com/MetaCubeX/meta-rules-dat/releases/download/latest/GeoLite2-ASN.mmdb"
  # 备选: jsdelivr CDN（国内多个节点可用）
  # geoip: "https://fastly.jsdelivr.net/gh/MetaCubeX/meta-rules-dat@release/geoip.dat"
  # geosite: "https://fastly.jsdelivr.net/gh/MetaCubeX/meta-rules-dat@release/geosite.dat"
  # mmdb: "https://fastly.jsdelivr.net/gh/MetaCubeX/meta-rules-dat@release/geoip.metadb"
  # geoip: "https://testingcf.jsdelivr.net/gh/MetaCubeX/meta-rules-dat@release/geoip.dat"
  # geosite: "https://gcore.jsdelivr.net/gh/MetaCubeX/meta-rules-dat@release/geosite.dat"
  # 备选: ghfast.top 镜像
  # geoip: "https://ghfast.top/https://github.com/MetaCubeX/meta-rules-dat/releases/download/latest/geoip.dat"
  # geosite: "https://ghfast.top/https://github.com/MetaCubeX/meta-rules-dat/releases/download/latest/geosite.dat"
  # mmdb: "https://ghfast.top/https://github.com/MetaCubeX/meta-rules-dat/releases/download/latest/country.mmdb"
  # 备选: Loyalsoldier 数据库（国内用户可优先选用）
  # geoip: "https://raw.githubusercontent.com/Loyalsoldier/geoip/release/geoip.dat"
  # geosite: "https://cdn.jsdelivr.net/gh/v2fly/domain-list-community@release/dlc.dat"
  # mmdb: "https://raw.githubusercontent.com/Loyalsoldier/geoip/release/Country.mmdb"
  # asn: "https://raw.githubusercontent.com/Loyalsoldier/geoip/release/GeoLite2-ASN.mmdb"

# Hosts 配置（域名 → IP 映射）
# 用于预解析 DoH/DoT 服务器、WiFi Calling 等特殊场景
hosts:
  doh.pub:                 [1.12.12.21, 120.53.53.53, 1.12.12.12]
  dns.alidns.com:          [223.5.5.5, 223.6.6.6, 2400:3200::1, 2400:3200:baba::1]
  dns.google:              [8.8.8.8, 8.8.4.4, 2001:4860:4860::8888, 2001:4860:4860::8844]
  cloudflare-dns.com:      [1.1.1.1, 1.0.0.1, 2606:4700:4700::1111, 2606:4700:4700::1001]
  services.googleapis.cn:  services.googleapis.com
  cn.bing.com:             global.bing.com

# 代理认证（HTTP/SOCKS/mixed 代理访问需要用户名密码）
authentication:
  - "user:pass"
skip-auth-prefixes:
  - 127.0.0.1/8
  - 10.0.0.0/8
  - 172.16.0.0/12
  - 192.168.0.0/16

# 管理面板
external-ui: /path/to/ui       # Web 面板路径
external-ui-url: "https://cdn.gh-proxy.org/https://github.com/MetaCubeX/metacubexd/archive/refs/heads/gh-pages.zip"  # 自动下载面板
external-ui-name: ui           # 面板文件夹名（可选，默认 ui）

profile:
  store-selected: true         # 重启后保留代理组选择
  store-fake-ip: true          # 保留 fake-ip 映射
  tracing: false               # 启用数据跟踪面板

# 条件请求支持（减少 geo 数据库重复下载）
etag-support: true

# 网络接口
interface-name: en0            # 出站网络接口
routing-mark: 6666             # Linux 路由标记

### 通用字段

所有代理节点共有的字段：

```yaml
proxies:
  - name: "my-server"          # 节点名（必须唯一）
    type: ss                   # 协议类型
    server: example.com        # 服务器地址
    port: 443                  # 端口
    ip-version: dual           # dual / ipv4 / ipv6 / ipv4-prefer / ipv6-prefer
    udp: false                 # 是否启用 UDP
    interface-name: eth0       # 绑定网卡
    routing-mark: 1234         # 路由标记
    tfo: false                 # TCP Fast Open
    mptcp: false               # Multipath TCP
    dialer-proxy: another-node # 通过另一个代理连接（链式代理）
    smux:                      # 多路复用
      enabled: true
      protocol: h2mux          # smux / yamux / h2mux
      max-connections: 4
      min-streams: 4
      max-streams: 0
      statistic: false
      only-tcp: false
      padding: true
      brutal-opts:
        enabled: true
        up: 50                 # Mbps
        down: 100              # Mbps
```

### 支持的协议类型

| 类型 | 说明 | 典型额外字段 |
|------|------|-------------|
| `ss` | Shadowsocks | cipher, password, udp-over-tcp |
| `ssr` | ShadowsocksR | obfuscation 相关 |
| `http` | HTTP 代理 | （无特殊） |
| `socks` | SOCKS5 代理 | （无特殊） |
| `vmess` | V2Ray VMess | uuid, alterId, cipher, tls, network(ws/grpc/...), ws-path, ws-headers |
| `vless` | V2Ray VLESS | uuid, tls, flow, network, reality-opts |
| `trojan` | Trojan | password, sni, alpn, skip-cert-verify |
| `hysteria` | Hysteria | auth-str, protocol, up/down, sni, alpn |
| `hysteria2` | Hysteria2 | password, up/down, sni, alpn, obfs |
| `tuic` | TUIC | token/uuid, ip, heartbeat-interval, congestion-controller |
| `wireguard` | WireGuard | private-key, public-key, self-ip, ip, dns, mtu |
| `tailscale` | Tailscale | 通过 Tailscale 网络出口 |
| `ssh` | SSH 隧道 | private-key, host-key |
| `snell` | Surge Snell | psk, ip-version |
| `anytls` | AnyTLS | password, sni |
| `mieru` | Mieru | （混淆代理） |
| `sudoku` | Sudoku | （混淆代理） |
| `openvpn` | OpenVPN | （需要额外配置） |
| `direct` | 直连（可配合 ip-version 控制出口协议栈） | ip-version: ipv4-prefer / ipv6-prefer |
| `reject` | 拒绝 | — |
| `masque` | MASQUE | （基于 QUIC 的代理） |
| `trusttunnel` | TrustTunnel | — |
| `dns` | DNS 出站 | — |

### TLS 配置（用于 vmess / vless / trojan 等）

```yaml
proxies:
  - name: "tls-proxy"
    type: trojan
    server: example.com
    port: 443
    password: "your-password"
    tls: true
    sni: example.com
    alpn:
      - h2
      - http/1.1
    skip-cert-verify: false
    fingerprint: chrome          # 全局 TLS 指纹已弃用，在节点内设置
    reality:                     # Reality 相关（vless）
      public-key: "..."
      short-id: "..."
```

### WARP WireGuard 链式代理

通过 `dialer-proxy` 实现 WARP 前置转发，其他节点经过 WARP 再出站：

```yaml
proxies:
  - name: "WARP"
    type: wireguard
    server: engage.cloudflareclient.com
    port: 2408
    ip: "172.16.0.2/32"
    ipv6: "2606::1/128"
    private-key: "private-key"     # 从 1.1.1.1 生成
    public-key: "public-key"       # 从 1.1.1.1 生成
    udp: true
    reserved: "abba"               # 从 1.1.1.1 生成
    mtu: 1280
    dialer-proxy: "WARP前置"       # 通过 WARP前置 出站
    remote-dns-resolve: true
    dns:
      - https://dns.cloudflare.com/dns-query

  # 需要前置经过 WARP 的节点
  - name: "节点-WARP"
    type: trojan
    server: example.com
    port: 443
    password: "password"
    dialer-proxy: "WARP"           # 通过 WARP 节点建立连接

# 相应的代理组
proxy-groups:
  - name: WARP前置
    type: select
    proxies:
      - 节点选择
      - DIRECT
    exclude-type: "wireguard"      # 排除 WARP 自身（防止循环）
```

### DIRECT 类型节点（IP 版本控制）

通过 `type: direct` 创建自定义直连节点，精细控制出口的 IP 协议栈：

```yaml
proxies:
  - {name: IPV4优先, type: direct, udp: true, ip-version: ipv4-prefer}
  - {name: IPV6优先, type: direct, udp: true, ip-version: ipv6-prefer}
  - {name: 仅IPV4,   type: direct, udp: true, ip-version: ipv4}
  - {name: 仅IPV6,   type: direct, udp: true, ip-version: ipv6}
```

然后在代理组中引用这些节点来控制出口协议。

### 传输层配置（用于 vmess / vless）

```yaml
proxies:
  - name: "ws-proxy"
    type: vmess
    ...
    network: ws                  # ws / grpc / h2 / httpupgrade
    ws-opts:
      path: /path
      headers:
        Host: example.com
    # 或 gRPC:
    # network: grpc
    # grpc-opts:
    #   grpc-service-name: "service.name"
```

---

## 代理集合 (Proxy Providers)

从外部 URL 或文件批量加载代理节点。

```yaml
proxy-providers:
  my-subscription:
    type: http                    # http / file / inline
    url: "https://example.com/sub"
    interval: 3600                # 更新间隔（秒）
    path: ./sub.yaml              # 缓存路径（可选，默认 URL 的 MD5）
    proxy: DIRECT                 # 通过哪个代理下载（可选）
    size-limit: 0                 # 下载大小限制，0 为不限制
    header:                       # 自定义请求头（可选）
      User-Agent: ["mihomo/1.18.3"]
      Authorization: ['token xxx']
    age-secret-key: "..."         # Age 加密密钥（可选）
    health-check:
      enable: true
      url: https://www.gstatic.com/generate_204
      # 或: https://cp.cloudflare.com
      interval: 600
      timeout: 5000               # 毫秒
      lazy: true                  # 不在使用时跳过检测
      expected-status: 204
    override:                     # 覆盖节点属性（可选）
      additional-prefix: "🇯🇵"    # 节点名前缀
      additional-suffix: "-jp"    # 节点名后缀
      proxy-name:                # 正则替换节点名
        pattern: "abc"
        target: "xyz"
      udp: true
      skip-cert-verify: false
      ip-version: ipv4-prefer
    filter: "(?i)港|hk|hongkong"  # 只保留匹配的节点
    exclude-filter: "美|日"       # 排除匹配的节点
    exclude-type: "Shadowsocks|Http"  # 排除指定类型

  my-inline-nodes:               # 内联节点
    type: inline
    payload:
      - name: "ss1"
        type: ss
        server: server
        port: 443
        cipher: chacha20-ietf-poly1305
        password: "password"
```

### ProxySet 锚点复用（推荐）

用 YAML 锚点统一管理多个订阅的公共配置：

```yaml
# 定义一个基础订阅模板
ProxySet: &ProxySet
  type: http
  proxy: DIRECT                   # 直连下载订阅（不走代理）
  interval: 86400
  health-check:
    enable: true
    url: "https://cp.cloudflare.com/generate_204"
    interval: 600
  override:
    udp: true
    additional-prefix: ""         # 可选：添加统一前缀
    skip-cert-verify: true

# 使用模板
proxy-providers:
  我的订阅:
    url: "https://example.com/sub"
    override:
      additional-prefix: "🇯🇵"   # 覆盖或追加前缀
    <<: *ProxySet                  # 继承基础配置

  另一个订阅:
    url: "https://example2.com/sub"
    filter: "(?i)港|hk"          # 只保留香港节点
    <<: *ProxySet
```

---

## 代理组 (Proxy Groups)

代理组定义节点的选择策略。

### 通用字段

```yaml
proxy-groups:
  # 通用字段
  - name: "Proxy"                # 组名
    type: select                 # 策略类型
    proxies:                     # 引用的节点或组
      - DIRECT
      - my-server
    use:                         # 引用的代理集合
      - my-subscription
    url: 'https://www.gstatic.com/generate_204'  # 测速 URL
    interval: 300                # 测速间隔（秒）
    timeout: 5000                # 测速超时（毫秒）
    lazy: true                   # 未使用时跳过检测
    max-failed-times: 5          # 最大失败次数后强制检测
    expected-status: 204         # 期望 HTTP 状态码
    # 支持多值和范围: 200/302  400-503  200/302/400-503
    include-all: false           # 包含所有节点和集合（按名排序）
    include-all-proxies: false   # 仅包含所有 proxies 中的节点
    include-all-providers: false # 仅包含所有 proxy-providers 中的节点
    filter: "(?i)港|hk"          # 正则包含筛选
    exclude-filter: "美|日"      # 正则排除筛选
    exclude-type: "Shadowsocks"  # 按类型排除
    disable-udp: true
    hidden: false                # 在 API 中隐藏（需前端支持）
    icon: "https://..."          # 图标 URL（需前端支持）
    # 常用图标集：
    # Koolson/Qure: https://github.com/Koolson/Qure/raw/master/IconSet/Color/
    # DustinWin:    https://github.com/DustinWin/ruleset_geodata/releases/download/icons/
```

### 锚点模板（推荐）

用 YAML 锚点统一管理策略组基础配置（timeout、lazy、include-all 等）：

```yaml
# 基础配置模板
Select: &Select
  type: select
  url: "https://cp.cloudflare.com/generate_204"
  disable-udp: false
  hidden: false
  include-all: true              # 自动包含所有节点和集合

UrlTest: &UrlTest
  type: url-test
  url: "https://cp.cloudflare.com/generate_204"
  interval: 180
  lazy: true
  tolerance: 50
  timeout: 2000
  disable-udp: false
  max-failed-times: 3
  hidden: true
  include-all: true

FallBack: &FallBack
  type: fallback
  url: "https://cp.cloudflare.com/generate_204"
  interval: 180
  lazy: true
  timeout: 2000
  disable-udp: false
  max-failed-times: 3
  hidden: true
  include-all: true

LoadBalance: &LoadBalance
  type: load-balance
  url: "https://cp.cloudflare.com/generate_204"
  interval: 180
  lazy: true
  disable-udp: false
  strategy: consistent-hashing    # round-robin / consistent-hashing / sticky-sessions
  # sticky-sessions: 会话保持，同一目标 IP 始终由同一节点处理
  timeout: 2000
  max-failed-times: 3
  hidden: true
  include-all: true

# 使用示例
proxy-groups:
  - name: 🇭🇰 香港节点
    <<: *UrlTest
    filter: "^(?=.*((?i)🇭🇰|香港|港|HK|Hong))(?!.*((?i)回国|校园|游戏|教育|家宽)).*$"

  - name: 🚀 手动切换
    <<: *Select
    filter: "^(?=.*(.))(?!(.*故障|.*到期|.*剩余)).*$"
```

### 策略类型

#### select — 手动选择
用户在面板中手动切换节点，最常用的策略。

#### url-test — 自动选择最快节点
```yaml
  - name: "Auto"
    type: url-test
    proxies:
      - ss1
      - ss2
    url: "https://www.gstatic.com/generate_204"
    interval: 300
    tolerance: 50                # 延迟容差（ms），0 则始终选最快
```

> **tolerance 说明：** `tolerance: 50` 表示延迟在 50ms 内的节点视为同速度，避免频繁切换。设为 `0` 则始终选择延迟最低的节点。

#### fallback — 自动回退
按列表顺序优先选用可用节点，不可用时自动切换到下一个。

#### load-balance — 负载均衡
```yaml
  - name: "Load-Balance"
    type: load-balance
    proxies:
      - ss1
      - ss2
    url: "https://www.gstatic.com/generate_204"
    interval: 300
    strategy: round-robin        # round-robin / consistent-hashing / sticky-sessions
```

> **strategy 说明：** `round-robin` 轮询分配，`consistent-hashing` 同一域名固定节点，`sticky-sessions` 同一目标 IP 由同一节点处理（适合流媒体）。

#### relay — 链式代理
```yaml
  - name: "Relay-Chain"
    type: relay
    proxies:
      - proxy-a                  # 入口 → proxy-a → proxy-b → 出口
      - proxy-b
```

代理组的 `empty-fallback` 字段可以在组为空时退回到指定策略：
```yaml
    empty-fallback: COMPATIBLE   # 当组为空时的回退策略
```

### 节点筛选正则表达式（filter / exclude-filter）

使用 `filter` 和 `exclude-filter` 字段可以从订阅中按正则筛选节点名。以下是一套常用的地区筛选正则：

```yaml
# 正向筛选 - 只包含匹配的地区
FilterHK: &FilterHK "^(?=.*((?i)🇭🇰|香港|港|HK|Hong))(?!.*((?i)回国|校园|游戏|🎮|教育|久虚|家宽)).*$"
FilterTW: &FilterTW "^(?=.*((?i)🇹🇼|台湾|台|新北|彰化|TW|Taiwan))(?!.*((?i)回国|校园|游戏|🎮|教育|久虚|家宽)).*$"
FilterJP: &FilterJP "^(?=.*((?i)🇯🇵|日本|川日|东京|大阪|泉日|埼玉|沪日|深日|JP|Japan))(?!.*((?i)回国|校园|游戏|🎮|教育|久虚|家宽)).*$"
FilterKR: &FilterKR "^(?=.*((?i)🇰🇷|韩国|韩|韓|首尔|KR|Korea|KOR))(?!.*((?i)回国|校园|游戏|🎮|教育|久虚|家宽)).*$"
FilterSG: &FilterSG "^(?=.*((?i)🇸🇬|新加坡|坡|狮城|SG|Singapore))(?!.*((?i)回国|校园|游戏|🎮|教育|久虚|家宽)).*$"
FilterUS: &FilterUS "^(?=.*((?i)🇺🇸|美国|美|波特兰|达拉斯|俄勒冈|凤凰城|费利蒙|硅谷|拉斯维加斯|洛杉矶|圣何塞|圣克拉拉|西雅图|芝加哥|United States|UnitedStates|America))(?!.*((?i)回国|校园|游戏|🎮|教育|久虚|家宽)).*$"
FilterOthers: &FilterOthers "^(?!.*((?i)🇭🇰|香港|港|HK|Hong|🇹🇼|台湾|台|新北|彰化|TW|Taiwan|🇯🇵|日本|川日|东京|大阪|泉日|埼玉|沪日|深日|JP|Japan|🇸🇬|新加坡|坡|狮城|SG|Singapore|🇺🇸|美国|美|波特兰|达拉斯|俄勒冈|凤凰城|费利蒙|硅谷|拉斯维加斯|洛杉矶|圣何塞|圣克拉拉|西雅图|芝加哥|United States|UnitedStates|America|🇰🇷|韩国|韩|韓|首尔|KR|Korea|KOR|回国|校园|游戏|🎮|教育|久虚|网站|地址|剩余|过期|时间|有效|网址|禁止|邮箱|发布|客服|订阅|节点|家宽)).*$"
FilterNetflix: &FilterNetflix "^(?=.*((?i)NF|奈飞|解锁|Netflix|NETFLIX|Media))(?!.*((?i)家宽)).*$"
FilterHomeWidth: &FilterHomeWidth "^(?=.*((?i)家宽)).*$"
FilterAll: &FilterAll "^(?=.*(.))(?!.*((?i)群|邀请|返利|循环|官网|客服|网站|网址|获取|订阅|流量|到期|机场|下次|版本|官址|备用|过期|已用|联系|邮箱|工单|贩卖|通知|倒卖|防止|国内|地址|频道|无法|说明|使用|提示|特别|访问|支持|教程|关注|更新|作者|加入|家宽|(\\b(USE|USED|TOTAL|EXPIRE|EMAIL|Panel|Channel|Author)\\b|(\\d{4}-\\d{2}-\\d{2}|\\d+G)))).*$"
```

```yaml
# 使用示例
proxy-groups:
  - name: 🇭🇰 香港节点
    <<: *UrlTest
    filter: *FilterHK

  - name: 🎥 奈飞节点
    <<: *Select
    filter: *FilterNetflix

  - name: 🏁 其他节点-手动
    <<: *Select
    filter: *FilterOthers
```

---

## 路由规则 (Rules)

规则从上到下逐条匹配，**优先匹配上面的规则**。如果 UDP 请求匹配到一个不支持 UDP 的代理，会继续往下匹配。

### 规则类型

```yaml
rules:
  # --- 域名匹配 ---
  - DOMAIN,ad.com,REJECT                                  # 精确域名
  - DOMAIN-SUFFIX,google.com,Proxy                        # 域名后缀
  - DOMAIN-KEYWORD,google,Proxy                           # 域名关键词
  - DOMAIN-WILDCARD,*.google.com,Proxy                    # 域名通配符
  - DOMAIN-REGEX,^abc.*\\.com$,Proxy                      # 域名正则

  # --- Geo 数据库 ---
  - GEOSITE,youtube,Proxy                                 # Geosite 分类
  - GEOIP,CN,DIRECT                                       # IP 地理位置

  # --- IP 匹配 ---
  - IP-CIDR,127.0.0.0/8,DIRECT,no-resolve                # IP 段
  - IP-CIDR6,2620:0:2d0:200::7/32,Proxy                  # IPv6
  - IP-SUFFIX,8.8.8.8/24,Proxy                            # IP 后缀
  - IP-ASN,13335,DIRECT                                   # ASN 编号
  - SRC-IP-CIDR,192.168.1.201/32,DIRECT                   # 源 IP

  # --- 端口匹配 ---
  - DST-PORT,80,DIRECT                                    # 目标端口
  - SRC-PORT,7777,DIRECT                                  # 源端口
  - IN-PORT,7890,Proxy                                    # 入站端口

  # --- 入站上下文 ---
  - IN-TYPE,SOCKS/HTTP,Proxy                              # 入站类型
  - IN-USER,mihomo,Proxy                                  # 入站用户名
  - IN-NAME,ss,Proxy                                      # 入站名

  # --- 进程匹配 ---
  - PROCESS-NAME,curl,Proxy                               # 进程名
  - PROCESS-NAME-WILDCARD,*telegram*,Proxy                # 进程名通配符
  - PROCESS-NAME-REGEX,(?i)Telegram,Proxy                 # 进程名正则
  - PROCESS-PATH,/usr/bin/wget,Proxy                      # 完整路径
  - UID,1001,DIRECT                                       # Linux 用户 ID

  # --- 系统/传输 ---
  - NETWORK,udp,DIRECT                                    # TCP / UDP
  - DSCP,4,DIRECT                                         # DSCP 标记

  # --- 规则集合 ---
  - RULE-SET,provider-name,Proxy                          # 外部规则集

  # --- 逻辑复合 ---
  - AND,((DOMAIN,baidu.com),(NETWORK,UDP)),DIRECT         # 与（所有条件满足）
  - OR,((NETWORK,UDP),(DOMAIN,baidu.com)),REJECT          # 或（任一条件满足）
  - NOT,((DOMAIN,baidu.com)),Proxy                        # 非（条件不满足）

  # --- 常用 QUIC 阻断（阻止 UDP 443 的 QUIC 流量走代理）---
  # - AND,((DST-PORT,443),(NETWORK,UDP)),REJECT           # 阻止 QUIC
  # - AND,((DST-PORT,443),(NETWORK,UDP)),DIRECT           # 或让 QUIC 直连

  # --- 子规则（见下方说明） ---
  - SUB-RULE,(NETWORK,tcp),my-sub-rule

  # --- 兜底（必须放在最后） ---
  - MATCH,Proxy
```

### no-resolve 说明

IP 类规则追加 `,no-resolve` 可以跳过 DNS 解析。但如果前面的规则已经触发了解析，后续即使有 `no-resolve` 也会匹配。

---

## 规则集合 (Rule Providers)

从外部 URL 动态加载规则片段。

### 基础语法

```yaml
rule-providers:
  google:
    type: http                      # http / file / inline
    behavior: classical             # domain / ipcidr / classical
    format: yaml                    # yaml / text / mrs（MRS 为二进制格式）
    url: "https://raw.githubusercontent.com/..."
    interval: 600                   # 更新间隔（秒）
    path: ./rule-providers/google.yaml
    proxy: DIRECT                   # 通过哪个代理下载（绕过订阅链路）
    size-limit: 0
    header:
      User-Agent: ["mihomo/1.18.3"]
    payload:                        # type: inline 时的内联规则
      - 'DOMAIN-SUFFIX,google.com'
```

### 广告拦截规则集示例

```yaml
# 秋风广告拦截规则（推荐替代 Anti-AD，误杀率更低）
rule-providers:
  AWAvenue-Ads:
    type: http
    behavior: domain
    format: yaml
    url: "https://ghfast.top/https://raw.githubusercontent.com/TG-Twilight/AWAvenue-Ads-Rule/refs/heads/main/Filters/AWAvenue-Ads-Rule-Clash-Classical.yaml"
    interval: 600
    proxy: DIRECT
```

### MRS 格式（二进制，推荐）

MRS 是编译后的规则集，加载更快、内存更少：

```yaml
  fakeip-filter:
    type: http
    behavior: domain
    format: mrs
    url: "https://cdn.gh-proxy.org/https://github.com/DustinWin/ruleset_geodata/releases/download/mihomo-ruleset/fakeip-filter.mrs"
    interval: 86400
    proxy: DIRECT
```

### YAML 锚点复用

多个 rule-providers 共用相同参数时，用 YAML 锚点减少重复：

```yaml
# 定义基础模板
RuleSet_classical: &RuleSet_classical
  type: http
  behavior: classical
  interval: 86400
  proxy: DIRECT

RuleSet_domain: &RuleSet_domain
  type: http
  behavior: domain
  interval: 86400
  proxy: DIRECT

RuleSet_ipcidr: &RuleSet_ipcidr
  type: http
  behavior: ipcidr
  interval: 86400
  proxy: DIRECT

# 使用锚点
rule-providers:
  fakeip-filter:
    <<: *RuleSet_domain
    format: mrs
    url: "https://cdn.gh-proxy.org/..."
  cn:
    <<: *RuleSet_domain
    format: mrs
    url: "https://cdn.gh-proxy.org/..."
```

**behavior 模式：**
- `domain` — 域名规则（只能包含 DOMAIN/DOMAIN-SUFFIX/DOMAIN-KEYWORD）
- `ipcidr` — IP 段规则（只能包含 IP-CIDR/IP-CIDR6）
- `classical` — 传统混合格式（可包含多种规则类型）

### 内联锚点风格（紧凑写法）

用 `{key: value}` 内联语法定义锚点，配合 `<<:` 引用，适合大量规则集场景：

```yaml
# 定义行为模板（内联）
BehaviorDN: &BehaviorDN {type: http, behavior: domain,  format: mrs,  interval: 86400}
BehaviorIP: &BehaviorIP {type: http, behavior: ipcidr,  format: mrs,  interval: 86400}
BehaviorDY: &BehaviorDY {type: http, behavior: domain,  format: yaml, interval: 86400}
ClassicalYaml: &ClassicalYaml {type: http, behavior: classical, interval: 3600, format: yaml, proxy: DIRECT}

# 大量规则集时极其简洁（如 HenryChiao / 666OS 风格）
rule-providers:
  AI:          {<<: *BehaviorDN, url: https://github.com/666OS/rules/raw/release/mihomo/domain/AI.mrs}
  Netflix:     {<<: *BehaviorDN, url: https://github.com/666OS/rules/raw/release/mihomo/domain/Netflix.mrs}
  YouTube:     {<<: *BehaviorDN, url: https://github.com/666OS/rules/raw/release/mihomo/domain/YouTube.mrs}
  Telegram:    {<<: *BehaviorDN, url: https://github.com/666OS/rules/raw/release/mihomo/domain/Telegram.mrs}
  ChinaIP:     {<<: *BehaviorIP, url: https://github.com/666OS/rules/raw/release/mihomo/ip/China.mrs}
```

> **常用 MRS 规则源：**
> - [666OS/rules](https://github.com/666OS/rules) — 综合规则集（域名 + IP）
> - [DustinWin/ruleset_geodata](https://github.com/DustinWin/ruleset_geodata) — 细分类规则集
> - [Repcz/Tool](https://github.com/Repcz/Tool) — 轻量规则集
> - [echs-top/proxy](https://github.com/echs-top/proxy) — Smart 模式规则集

---

## 子规则 (Sub-Rules)

对特定流量应用独立的路由规则集。

```yaml
sub-rules:
  my-sub-rule:
    - DOMAIN,example.com,DIRECT
    - MATCH,Proxy

# 在 rules 中引用:
rules:
  - SUB-RULE,(NETWORK,tcp),my-sub-rule
  - SUB-RULE,(DOMAIN,example.com),another-sub-rule
```

---

## DNS 配置

DNS 是 Mihomo 配置中最关键的环节之一。推荐使用 **三层级 DNS 架构**，区分对待代理流量、代理节点解析和直连流量。

```yaml
dns:
  enable: true
  cache-algorithm: lru            # lru / arc（arc 更高效但内存略多）
  prefer-h3: false                # DOH 优先 HTTP/3
  listen: 0.0.0.0:1053           # DNS 监听地址
  ipv6: false                     # false 时 AAAA 返回空
  enhanced-mode: fake-ip          # fake-ip / redir-host
  fake-ip-range: 198.18.0.1/16
  fake-ip-range6: ""              # IPv6 fake-ip 范围
  fake-ip-filter:
    - '*.lan'
    - '+.local'
    - "rule-set:fakeip-filter"    # 引用规则集合（推荐）
    - "geosite:private"           # 引用 Geosite
    - "geosite:tracker"           # BT tracker 返回真实 IP
  fake-ip-filter-mode: blacklist  # blacklist / whitelist / rule
  fake-ip-ttl: 3600               # TTL（秒）
  use-hosts: false
  use-system-hosts: true
  respect-rules: true             # DNS 连接遵循路由规则（关键！）

  # 基础 DNS（必须用 IP 地址，用于解析 DNS 服务器自身的域名）
  default-nameserver:
    - "system"                    # 使用系统 DNS（推荐）
    - tls://223.5.5.5
    - tls://223.6.6.6

  # 第一层：代理出口 DNS — 走 MATCH 兜底的国外流量使用国外 DNS
  # 因为 respect-rules: true，这些 DNS 会通过代理节点连接，避免 DNS 污染
  nameserver:
    - https://1.1.1.1/dns-query
    - https://8.8.8.8/dns-query

  # 第二层：代理节点解析 DNS — 解析订阅节点域名时使用国内 DNS
  # 避免"先有鸡还是先有蛋"：不先解析节点域名就无法连接代理
  proxy-server-nameserver:
    - https://dns.alidns.com/dns-query
    - https://doh.pub/dns-query

  # 第三层：直连 DNS — 规则匹配到 DIRECT 时使用国内 DNS
  direct-nameserver:
    - https://dns.alidns.com/dns-query
    - https://doh.pub/dns-query

  # 节点域名策略（针对特定节点覆盖）
  proxy-server-nameserver-policy:
    "www.yournode.com": "114.114.114.114"

  # 域名策略（覆盖 nameserver/direct-nameserver）
  nameserver-policy:
    '+.arpa': 10.0.0.1
    "rule-set:cn":
      - https://doh.pub/dns-query
    "geosite:geolocation-cn":
      - https://dns.alidns.com/dns-query

  # 高级：让 direct-nameserver 也遵循 nameserver-policy
  direct-nameserver-follow-policy: true

  # 高级：用 rcode://success 阻挡广告查询（返回空成功响应）
  # nameserver-policy:
  #   "rule-set:Advertising,Ads": rcode://success

  # 高级：按 rule-set 路由 DNS 到不同上游
  # nameserver-policy:
  #   "rule-set:Direct,Private,China":     # 直连域名 → 国内 DNS
  #     - https://dns.alidns.com/dns-query
  #     - https://doh.pub/dns-query
  #   "rule-set:Netflix,YouTube,Twitter":  # 代理域名 → 国外 DNS
  #     - https://dns.google/dns-query
  #     - https://cloudflare-dns.com/dns-query

  # fallback 备用 DNS（仅在 nameserver 无法满足时使用）
  fallback:
    - tls://8.8.4.4
    - https://doh.dns.sb/dns-query
  fallback-filter:
    geoip: true
    geoip-code: CN               # 非 CN IP 视为被污染
    geosite:
      - gfw
    ipcidr:
      - 240.0.0.0/4
    domain:
      - '+.google.com'
      - '+.youtube.com'
```

> **三层级架构说明：**
> - `nameserver` — 国外 DNS，用于解析走代理的域名。通过 `respect-rules: true` 经由代理节点连接，确保不被污染
> - `proxy-server-nameserver` — 国内 DNS，专门解析代理订阅中的节点域名。避免代理节点域名被国外 DNS 解析或污染
> - `direct-nameserver` — 国内 DNS，用于规则匹配到 DIRECT 的域名。直连场景用国内 DNS 延迟更低、更可靠

### fake-ip-filter-mode: rule

使用路由规则语法来控制哪些域名返回 fake-ip，哪些返回真实 IP：

```yaml
dns:
  enhanced-mode: fake-ip
  fake-ip-filter-mode: rule
  fake-ip-filter:
    - RULE-SET,reject-domain,fake-ip
    - GEOSITE,gfw,fake-ip
    - DOMAIN,www.baidu.com,real-ip
    - DOMAIN-SUFFIX,qq.com,real-ip
    - MATCH,fake-ip
```

### DNS 服务器参数

在 DNS URL 后通过 `#` 添加参数：

```
https://8.8.8.8/dns-query#proxy&ecs=1.1.1.1/24&ecs-override=true
```

| 参数 | 说明 |
|------|------|
| `proxy` | 通过指定代理路由 DNS |
| `interface-name` | 绑定网卡 |
| `#RULES` | 遵循路由规则 |
| `h3` | 强制 HTTP/3 |
| `skip-cert-verify` | 跳过证书验证 |
| `ecs` | EDNS Client Subnet |
| `ecs-override` | 强制覆盖 ECS |
| `disable-ipv4` / `disable-ipv6` | 丢弃对应记录 |
| `disable-qtype-<N>` | 丢弃特定 DNS 记录类型 |

### DNS 部署建议（国内用户）

| 使用场景 | nameserver | proxy-server-nameserver | direct-nameserver |
|----------|-----------|------------------------|-------------------|
| 有代理订阅 | 国外 DNS (1.1.1.1) | 国内 DNS (阿里云) | 国内 DNS (阿里云) |
| 仅代理自己 | 国内 DNS (阿里云) | — | — |
| OpenWrt 路由器 | 国外 DNS | 国内 DNS | 国内 DNS |

> `default-nameserver` 中的 `"system"` 表示使用系统 /etc/resolv.conf 中的 DNS（通常是 127.0.0.53）。也可以直接写国内 DNS 的 IP。

### 备选 DNS 方案：nameserver-policy + redir-host

对于追求兼容性的场景（如路由器），可以使用 `redir-host` 模式 + `nameserver-policy` 实现 geosite 分流：

```yaml
dns:
  enable: true
  listen: :1053
  ipv6: true
  enhanced-mode: redir-host       # redir-host 兼容性更好
  fake-ip-range: 198.18.0.1/16
  fake-ip-filter:
    - '*'
    - '+.lan'
    - '+.local'
  default-nameserver:
    - 223.5.5.5
    - 119.29.29.29
    - 114.114.114.114
  nameserver:
    - 'tls://8.8.4.4'
    - 'tls://1.0.0.1'
  proxy-server-nameserver:
    - https://doh.pub/dns-query
  nameserver-policy:
    "geosite:cn,private":          # 国内域名 → 国内 DNS
      - https://doh.pub/dns-query
      - https://dns.alidns.com/dns-query
    "geosite:!cn,!private":        # 国外域名 → 国外 DNS
      - "tls://dns.google"
      - "tls://cloudflare-dns.com"
```

### DNS 路由代理组（#dns 标签）

可以为 DNS 查询指定专门的代理组：

```yaml
# 1. 先定义一个 dns 代理组
proxy-groups:
  - name: dns
    type: select
    proxies:
      - 节点选择
      - DIRECT

# 2. 在 DNS URL 后添加 #dns 标签
dns:
  nameserver:
    - 'tls://8.8.4.4#dns'         # DNS 查询走 dns 代理组
    - 'tls://1.0.0.1#dns'
```

在 DNS URL 后追加 `#<代理组名>` 可路由 DNS 到指定代理组，适合精细控制 DNS 出口。

---

## TUN 模式

TUN 模式创建虚拟网卡，捕获所有系统流量。

```yaml
tun:
  enable: true
  stack: gvisor                  # system / gvisor / mixed
  device: utun0                  # macOS 必须以 utun 开头
  mtu: 9000
  gso: true                      # Linux 才支持
  gso-max-size: 65536
  udp-timeout: 300               # UDP NAT 超时（秒）

  # 自动路由
  auto-route: true               # 自动配置系统路由
  auto-redirect: true            # Linux 专用，配置 iptables/nftables
  auto-detect-interface: true    # 自动检测出口网卡
  strict-route: true             # 严格路由（防泄漏、DNS 劫持）

  # DNS 劫持
  dns-hijack:
    - any:53
    - tcp://any:53

  # IPv6 地址
  inet6-address: fdfe:dcba:9876::1/126

  # IPRoute2（Linux）
  iproute2-table-index: 2022
  iproute2-rule-index: 9000

  # NAT
  endpoint-independent-nat: false

  # 地址集规则（Linux nftables）
  route-address-set:
    - ruleset-1
  route-exclude-address-set:
    - ruleset-2

  # 静态路由
  route-address:
    - 0.0.0.0/1
    - "::/1"
  route-exclude-address:
    - 192.168.0.0/16

  # 网卡过滤
  include-interface:
    - eth0
  exclude-interface:
    - eth1

  # UID 过滤（Linux）
  include-uid:
    - 0
  include-uid-range:
    - 1000:9999
  exclude-uid:
    - 1000
  exclude-uid-range:
    - 1000:9999

  # Android 用户/应用过滤
  include-android-user:
    - 0           # 机主
    - 10          # 次要用户
  include-package:
    - com.android.chrome
  exclude-package:
    - com.android.captiveportallogin
```

### TUN 栈对比

| 栈 | 优点 | 适用场景 |
|----|------|---------|
| `system` | 稳定、完整、开销低 | 一般使用，但需要放行防火墙 |
| `gvisor` | 安全性、隔离性高 | 有防火墙/安全要求的环境 |
| `mixed` | TCP 用 system, UDP 用 gvisor | 兼顾性能和隔离 |

> 如果有防火墙，`system` 和 `mixed` 栈需要将 mihomo 加入例外才能正常工作。

---

## 域名嗅探 (Sniffer)

对流量进行协议嗅探以获取真实域名（即使使用 IP 连接）。

```yaml
sniffer:
  enable: false                   # 建议启用（true）以获取更准确的路由
  force-dns-mapping: true         # redir-host 模式下强制嗅探
  parse-pure-ip: true             # 对无域名的流量强制嗅探
  override-destination: true      # 用嗅探结果替代原目标

  # 按协议配置
  sniff:
    TLS:
      ports: [443, 8443]          # 仅嗅探 TLS 端口即可
    HTTP:
      ports: [80, 8080-8880]
      override-destination: true
    QUIC:
      ports: [443, 8443]

  # 跳过/强制列表
  force-domain:
    - '+.v2ex.com'               # 强制嗅探的域名
  skip-domain:
    - 'Mijia Cloud'              # 跳过嗅探的域名
  skip-src-address:
    - 192.168.0.3/32
  skip-dst-address:
    - 192.168.0.3/32
```

---

## 入站监听 (Inbound)

### LAN 监听（无加密）

| 监听器 | 配置 | 默认 UDP |
|--------|------|---------|
| SOCKS | `socks-port: 10808` | ✔ |
| HTTP | `port: 10809` | ✘ |
| Mixed | `mixed-port: 10810` | ✔ |
| Redir | `redir-port: 10811` | ✘ |
| TProxy | `tproxy-port: 10812` | ✔ |
| Tunnel | （见 tunnels 段） | ✔ |

### 完整入站配置（替代顶层快捷端口）

```yaml
listeners:
  - name: socks-in              # 可选
    type: socks
    port: 10808
    listen: 0.0.0.0
    udp: true
    rule: my-sub-rule           # 使用子规则（可选）
    proxy: DIRECT               # 直接将流量发往指定代理（可选）
```

### 多入站端口（按地区分流）

为不同地区设置不同代理端口，方便应用程序定向选择出口：

```yaml
listeners:
  - name: hk
    type: mixed
    port: 12991
    proxy: 香港           # 流量直接走"香港"代理组

  - name: tw
    type: mixed
    port: 12992
    proxy: 台湾

  - name: sg
    type: mixed
    port: 12993
    proxy: 新加坡

  - name: jp
    type: mixed
    port: 12994
    proxy: 日本
```

在应用层设置代理 `127.0.0.1:12991` 即固定走香港出口。

### 加密入站（服务器模式）

```yaml
listeners:
  - name: ss-server
    type: shadowsocks
    port: 10813
    listen: 0.0.0.0
    password: "your-password"
    cipher: 2022-blake3-aes-256-gcm

  - name: vmess-server
    type: vmess
    port: 10814
    uuid: "your-uuid"
    alter-id: 0

  - name: tuic-server
    type: tuic
    port: 10815
    token: "your-token"
    certificate: /path/to/cert.pem
    private-key: /path/to/key.pem
```

### 顶层快捷入站

```yaml
# 可直接在顶层配置 Shadowsocks / VMess / TUIC 入站
ss-config: "ss://2022-blake3-aes-256-gcm:password@:23456"
vmess-config: "vmess://1:uuid@:12345"
tuic-server:
  enable: true
  token: "your-token"
  certificate: /path/to/cert.pem
  private-key: /path/to/key.pem
```

---

## 隧道转发 (Tunnels)

```yaml
tunnels:
  - network: [tcp, udp]          # tcp / udp
    address: 1.2.3.4:53          # 目标地址
    listen: 127.0.0.1:1053       # 本地监听
    proxy: Proxy                 # 经过的代理
```

---

## NTP 配置

```yaml
ntp:
  enable: false
  server: time.apple.com         # NTP 服务器
  port: 123
  interval: 30                   # 同步间隔（分钟）
  proxy: DIRECT                  # 通过哪个代理连接 NTP
  write-to-system: false         # 写入系统时间（需 root/管理员权限）
```

---

## 实验性功能

```yaml
experimental:
  quic-go-disable-gso: false     # 禁用 GSO
  quic-go-disable-ecn: false     # 禁用 ECN（显式拥塞通知）
  dialer-ip4p-convert: false     # IP4P 地址转换
```

---

## Smart 内核（魔改版 Mihomo）

[vernesong/mihomo](https://github.com/vernesong/mihomo) 的魔改版支持 Smart 策略组，通过 LightGBM 机器学习模型预测节点延迟，实现更智能的自动选择。

### Smart 策略组

将普通的 `url-test` 替换为 `smart` 即可启用：

```yaml
proxy-groups:
  - name: ♻️ 自动选择
    type: smart                    # 替代 url-test
    use:
      - 订阅一
    # Smart 高级设置
    policy-priority: "Premium:0.9;SG:1.3"  # 节点优先级（<1 更低，>1 更高）
    uselightgbm: false             # 使用 LightGBM 模型预测
    collectdata: false             # 收集数据供模型训练
    sample-rate: 1                 # 数据采集率（0-1）
    prefer-asn: true               # 优先查询 ASN 选择节点
```

### LightGBM 模型

```yaml
# 在全局配置中添加
lgbm-auto-update: true            # 自动更新模型
lgbm-update-interval: 72          # 更新间隔（小时）
lgbm-url: "https://ghfast.top/https://github.com/vernesong/mihomo/releases/download/LightGBM-Model/Model.bin"

profile:
  store-selected: true
  store-fake-ip: true
  smart-collector-size: 100       # 数据收集文件大小（MB）
```

### Smart 与 url-test 对比

| 特性 | url-test | smart |
|------|----------|-------|
| 选择依据 | 实时延迟测试 | ML 预测 + 历史数据 |
| 测速流量 | 定期产生 | 极少（预测为主） |
| 响应变化 | 较快 | 略慢但更稳定 |
| 切换频率 | 频繁 | 保持稳定连接 |

> **注意：** Smart 内核需要使用 [vernesong/mihomo](https://github.com/vernesong/mihomo/releases) 的魔改版。官方 MetaCubeX 版本不支持 smart 类型。

---

## 常见配置模板

## 常见配置模板

### 场景一：基础代理客户端（TUN 全局代理）

```yaml
mixed-port: 7890
allow-lan: false
mode: rule
log-level: info
ipv6: true
external-controller: 127.0.0.1:9090
secret: ""

proxy-providers:
  my-sub:
    type: http
    url: "https://your-subscription-url"
    interval: 86400
    health-check:
      enable: true
      url: https://www.gstatic.com/generate_204
      interval: 600

proxy-groups:
  - name: Proxy
    type: select
    use:
      - my-sub
    proxies:
      - DIRECT
      - Auto
  - name: Auto
    type: url-test
    use:
      - my-sub
    url: https://www.gstatic.com/generate_204
    interval: 300
    tolerance: 50
  - name: Domestic
    type: select
    proxies:
      - DIRECT
      - Proxy

rules:
  - RULE-SET,reject,REJECT
  - RULE-SET,private,DIRECT
  - RULE-SET,cn,DIRECT
  - RULE-SET,proxy,Proxy
  - MATCH,Proxy

rule-providers:
  reject:
    type: http
    behavior: classical
    format: mrs
    url: "https://raw.githubusercontent.com/MetaCubeX/meta-rules-dat/meta/geo/reject.mrs"
    interval: 86400
    proxy: DIRECT
  private:
    type: http
    behavior: classical
    format: mrs
    url: "https://raw.githubusercontent.com/MetaCubeX/meta-rules-dat/meta/geo/private.mrs"
    interval: 86400
    proxy: DIRECT
  cn:
    type: http
    behavior: classical
    format: mrs
    url: "https://raw.githubusercontent.com/MetaCubeX/meta-rules-dat/meta/geo/geolocation-cn.mrs"
    interval: 86400
    proxy: DIRECT
  proxy:
    type: http
    behavior: classical
    format: mrs
    url: "https://raw.githubusercontent.com/MetaCubeX/meta-rules-dat/meta/geo/geolocation-!cn.mrs"
    interval: 86400
    proxy: DIRECT

dns:
  enable: true
  listen: 0.0.0.0:1053
  ipv6: true
  enhanced-mode: fake-ip
  fake-ip-range: 198.18.0.1/16
  fake-ip-filter:
    - '*.lan'
    - '+.local'
    - '+.localhost'
    - 'rule-set:private'
    - 'rule-set:cn'
  default-nameserver:
    - 223.5.5.5
    - tls://1.1.1.1
  nameserver:
    - https://doh.pub/dns-query
    - https://dns.alidns.com/dns-query
  fallback:
    - tls://8.8.4.4
    - https://doh.dns.sb/dns-query
  fallback-filter:
    geoip: true
    geoip-code: CN

tun:
  enable: true
  stack: mixed
  auto-route: true
  auto-detect-interface: true
  dns-hijack:
    - any:53
```

### 场景二：仅 HTTP/SOCKS 代理（无 TUN）

```yaml
mixed-port: 7890
mode: rule
log-level: info
external-controller: 127.0.0.1:9090

# 其他段同上，省略 TUN 配置和 DNS 配置（或保留简单 DNS）
dns:
  enable: true
  enhanced-mode: redir-host
  nameserver:
    - 223.5.5.5
    - 114.114.114.114
```

### 场景三：局域网共享代理

```yaml
mixed-port: 7890
allow-lan: true
bind-address: "*"
lan-allowed-ips:
  - 0.0.0.0/0
  - "::/0"
mode: rule
# ... 其他配置与场景一类似，不需要 TUN
```

### 场景四：精细化分流（多地区 + 服务分类）

完整的生产级配置，使用 YAML 锚点 + 三层级 DNS + geosite 规则：

```yaml
# 全局设置
mixed-port: 7890
allow-lan: false
mode: rule
log-level: info
ipv6: false
external-controller: 0.0.0.0:9090
secret: ""
external-ui: ui
external-ui-name: ui
external-ui-url: "https://cdn.gh-proxy.org/https://github.com/MetaCubeX/metacubexd/archive/refs/heads/gh-pages.zip"
external-controller-tls: ""
global-client-fingerprint: random
keep-alive-interval: 1800
find-process-mode: strict
tcp-concurrent: true
unified-delay: true
etag-support: true

# Hosts 配置
hosts:
  doh.pub:            [1.12.12.21, 120.53.53.53, 1.12.12.12]
  dns.alidns.com:     [223.5.5.5, 223.6.6.6]
  dns.google:         [8.8.8.8, 8.8.4.4]
  cloudflare-dns.com: [1.1.1.1, 1.0.0.1]

# 代理认证
authentication:
  - user:pass
skip-auth-prefixes:
  - 192.168.0.0/16
  - 127.0.0.1/8

geodata-mode: true
geo-auto-update: true
geo-update-interval: 24
geox-url:
  geoip: "https://cdn.gh-proxy.org/https://github.com/MetaCubeX/meta-rules-dat/releases/download/latest/geoip.dat"
  geosite: "https://cdn.gh-proxy.org/https://github.com/MetaCubeX/meta-rules-dat/releases/download/latest/geosite.dat"
  mmdb: "https://cdn.gh-proxy.org/https://github.com/MetaCubeX/meta-rules-dat/releases/download/latest/country.mmdb"
  asn: "https://cdn.gh-proxy.org/https://github.com/MetaCubeX/meta-rules-dat/releases/download/latest/GeoLite2-ASN.mmdb"

# DNS 三层级
dns:
  enable: true
  ipv6: false
  enhanced-mode: fake-ip
  fake-ip-range: 198.18.0.1/16
  fake-ip-filter:
    - rule-set:fakeip-filter
    - geosite:private
    - geosite:tracker
  default-nameserver:
    - "system"
    - tls://223.5.5.5
    - tls://223.6.6.6
  nameserver:
    - https://1.1.1.1/dns-query
    - https://8.8.8.8/dns-query
  proxy-server-nameserver:
    - https://dns.alidns.com/dns-query
    - https://doh.pub/dns-query
  direct-nameserver:
    - https://dns.alidns.com/dns-query
    - https://doh.pub/dns-query
  respect-rules: true

profile:
  store-selected: true

# 规则集合
rule-providers:
  fakeip-filter:
    type: http
    behavior: domain
    format: mrs
    url: "https://cdn.gh-proxy.org/https://github.com/DustinWin/ruleset_geodata/releases/download/mihomo-ruleset/fakeip-filter.mrs"
    interval: 86400
    proxy: DIRECT
  # 广告拦截
  AWAvenue-Ads:
    type: http
    behavior: domain
    format: yaml
    url: "https://ghfast.top/https://raw.githubusercontent.com/TG-Twilight/AWAvenue-Ads-Rule/refs/heads/main/Filters/AWAvenue-Ads-Rule-Clash-Classical.yaml"
    interval: 600
    proxy: DIRECT

# 代理集合（订阅）
proxy-providers:
  订阅名称:
    type: http
    url: "https://your-subscription-url"
    interval: 86400
    proxy: DIRECT
    health-check:
      enable: true
      url: "https://cp.cloudflare.com/generate_204"
      interval: 600
    override:
      udp: true
      skip-cert-verify: true

# 节点分组锚点
UrlTest: &UrlTest
  type: url-test
  url: "https://cp.cloudflare.com/generate_204"
  interval: 180
  lazy: true
  tolerance: 50
  timeout: 2000
  include-all: true
  hidden: true

Select: &Select
  type: select
  url: "https://cp.cloudflare.com/generate_204"
  include-all: true

# 代理组
proxy-groups:
  - name: 🚀 节点选择
    type: select
    proxies:
      - ♻️ 自动选择
      - 🇭🇰 香港节点
      - 🇯🇵 日本节点
      - 🇸🇬 狮城节点
      - 🇺🇲 美国节点
      - 🚀 手动切换
      - 🐟 漏网之鱼
      - DIRECT

  - name: 🚀 手动切换
    <<: *Select
    filter: "^(?=.*(.))(?!.*((?i)故障|到期|剩余|已用|过期)).*$"

  - name: ♻️ 自动选择
    <<: *UrlTest
    filter: "^(?=.*((?i)🇭🇰|香港|港|HK|Hong|🇯🇵|日本|JP|🇸🇬|新加坡|SG|🇺🇸|美国|US))(?!.*((?i)回国|校园|游戏|家宽)).*$"

  - name: 🤖 AI
    type: select
    proxies:
      - 🚀 节点选择
      - ♻️ 自动选择
      - 🇺🇲 美国节点
      - 🇯🇵 日本节点

  - name: 📹 油管视频
    type: select
    proxies:
      - 🚀 节点选择
      - ♻️ 自动选择

  - name: 🎮 游戏平台
    type: select
    proxies:
      - DIRECT
      - 🚀 节点选择

  - name: 🛑 广告拦截
    type: select
    proxies:
      - REJECT
      - DIRECT

  - name: 🎯 全球直连
    type: select
    proxies:
      - DIRECT

  - name: 🇭🇰 香港节点
    <<: *UrlTest
    filter: "^(?=.*((?i)🇭🇰|香港|港|HK|Hong))(?!.*((?i)回国|校园|游戏|家宽)).*$"

  - name: 🇯🇵 日本节点
    <<: *UrlTest
    filter: "^(?=.*((?i)🇯🇵|日本|东京|大阪|JP|Japan))(?!.*((?i)回国|校园|游戏|家宽)).*$"

  - name: 🇸🇬 狮城节点
    <<: *UrlTest
    filter: "^(?=.*((?i)🇸🇬|新加坡|坡|狮城|SG|Singapore))(?!.*((?i)回国|校园|游戏|家宽)).*$"

  - name: 🇺🇲 美国节点
    <<: *UrlTest
    filter: "^(?=.*((?i)🇺🇸|美国|美|洛杉矶|硅谷|西雅图|United States))(?!.*((?i)回国|校园|游戏|家宽)).*$"
    tolerance: 300

  - name: 🐟 漏网之鱼
    type: select
    proxies:
      - 🚀 节点选择
      - ♻️ 自动选择
      - DIRECT
      - 🇭🇰 香港节点
      - 🇯🇵 日本节点
      - 🇸🇬 狮城节点
      - 🇺🇲 美国节点
      - 🚀 手动切换

# 规则
rules:
  - GEOSITE,private,DIRECT
  - RULE-SET,AWAvenue-Ads,🛑 广告拦截
  - GEOSITE,category-ads-all,🛑 广告拦截
  - GEOSITE,googlefcm,🎯 全球直连
  - GEOSITE,google@cn,🎯 全球直连
  - GEOSITE,steam@cn,🎯 全球直连
  - GEOSITE,bing,🎯 全球直连
  - GEOSITE,onedrive,🎯 全球直连
  - GEOSITE,apple,🎯 全球直连
  - GEOSITE,category-ai-!cn,🤖 AI
  - GEOSITE,category-ai-cn,🎯 全球直连
  - GEOSITE,category-games,🎮 游戏平台
  - GEOSITE,youtube,📹 油管视频
  - GEOSITE,gfw,🚀 节点选择
  - GEOSITE,microsoft,🎯 全球直连
  - GEOSITE,cn,🎯 全球直连
  - GEOIP,private,DIRECT,no-resolve
  - GEOIP,telegram,🚀 节点选择,no-resolve
  - GEOIP,CN,🎯 全球直连
  - MATCH,🐟 漏网之鱼
```

---

## Age 加密

Mihomo 支持用 [age](https://age-encryption.org/) 加密配置文件和订阅内容。

```bash
# 生成密钥
mihomo age keygen                   # x25519 密钥
mihomo age keygen-pq                # mlkem768-x25519 密钥

# 导出公钥
mihomo age convert <secret_key>

# 加密/解密
mihomo age encrypt <public_key> <source> <target>
mihomo age decrypt <secret_key> <source> <target>
```

配置中使用：
```yaml
proxy-providers:
  encrypted-sub:
    type: http
    url: "https://..."
    age-secret-key: "AGE-SECRET-KEY-..."
```

---

## 命令行工具

```bash
# 基本启动
mihomo -d /etc/mihomo -f config.yaml

# 测试配置
mihomo -t -d /etc/mihomo -f config.yaml

# 更新 Geo 数据库
mihomo -d /etc/mihomo -update-geo

# 规则集转换
mihomo convert-ruleset domain/ipcidr yaml/text input.yaml output.mrs

# Age 工具（见上方）
mihomo age keygen
mihomo age decrypt <key> <src> <dst>
```

---

## 故障排查

### 1. 无法启动 / 配置错误
- 运行 `mihomo -t` 测试配置语法
- 检查 YAML 缩进（不能用 Tab）
- 检查 `key: value` 冒号后的空格
- 查看日志：设置 `log-level: debug`

### 2. TUN 模式不起作用
- 确认 `tun.enable: true`
- 有防火墙时 `system`/`mixed` 栈需要放行 mihomo
- `auto-route` 需要 root/管理员权限
- Windows 上 `strict-route` 可能与 VirtualBox 冲突
- macOS 上 `device` 必须以 `utun` 开头

### 3. DNS 解析异常
- `default-nameserver` 必须用 IP 地址（不能是域名）
- fake-ip 模式下 `fake-ip-filter` 根据场景调整
- `proxy-server-nameserver` 用于避免"先有鸡还是先有蛋"问题
- `respect-rules: true` 可以让 DNS 也走规则

### 4. 节点连接失败
- 检查节点配置的 server/port/cipher/password 是否正确
- 确认 `skip-cert-verify` 只在必要时开启
- `dialer-proxy` 链路过长可能导致超时
- `smux` 的 `brutal-opts` 的 up/down 带宽设置不要超过实际带宽

### 5. 代理组无节点
- 检查 `proxy-providers` 的 url 是否可达
- 确认 `use` 引用名称与 provider 名称一致
- `filter` / `exclude-filter` 正则可以过滤掉所有节点
- `exclude-type` 可能过滤掉期望的类型

### 6. 规则不生效
- 规则**从上到下**匹配，顺序非常重要
- UDP 请求遇到不支持 UDP 的代理会继续往下匹配
- `no-resolve` 只跳过 DNS 解析，不跳过已触发的解析结果
- `AND`/`OR`/`NOT` 需要仔细使用括号

### 7. 内存占用高
- `geodata-loader: memconservative` 减少内存使用
- 减少 rule-providers 数量或增加更新间隔
- 调整 `fake-ip-ttl` 减少映射表大小

### 8. Smart 内核无法启动
- 确认使用的是 [vernesong/mihomo](https://github.com/vernesong/mihomo) 魔改版，官方版不支持 `type: smart`
- 检查 `lgbm-url` 是否可访问
- Smart 相关配置不要和普通 `url-test` 混用在同一代理组

### 9. 移动端耗电快
- 将 `keep-alive-interval` 设为 1800 秒（30 分钟）
- TUN 模式改为 `stack: gvisor`（更省电）
- 减少 health-check 频率，`interval: 600` → 900

### 10. TLS 指纹被识别
- 设置 `global-client-fingerprint: random` 启用随机指纹
- 或在每个节点单独设置 `fingerprint: chrome` / `safari` / `firefox`

---

## 参考链接

- [官方文档 (wiki.metacubex.one)](https://wiki.metacubex.one)
- [MIhomo 源码 (GitHub)](https://github.com/MetaCubeX/mihomo)
- [Smart 内核魔改版 (vernesong)](https://github.com/vernesong/mihomo)
- [规则集数据 (meta-rules-dat)](https://github.com/MetaCubeX/meta-rules-dat)
- [666OS 规则集](https://github.com/666OS/rules)
- [DustinWin 规则集](https://github.com/DustinWin/ruleset_geodata)
- [Repcz 规则集](https://github.com/Repcz/Tool)
- [echs-top 规则集 (Smart)](https://github.com/echs-top/proxy)
- [Loyalsoldier GeoIP 数据库](https://github.com/Loyalsoldier/geoip)
- [Web 面板: metacubexd](https://github.com/MetaCubeX/metacubexd)
- [Web 面板: zashboard](https://github.com/Zephyruso/zashboard)
- [Qure 图标集](https://github.com/Koolson/Qure)
- [秋风广告拦截规则](https://awavenue.top)
- [本文档参考 Gist](https://gist.github.com/liuran001/5ca84f7def53c70b554d3f765ff86a33)
- [文档源码 (Meta-Docs)](https://github.com/MetaCubeX/Meta-Docs)
- [本文档参考: HenryChiao/mihomo_yamls](https://github.com/HenryChiao/mihomo_yamls)
