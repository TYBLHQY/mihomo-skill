# Mihomo Skill

> Mihomo（虚空终端 / Clash Meta）Claude Code 技能 — 代理配置完整指南

[![License: MIT](https://img.shields.io/badge/License-MIT-yellow.svg)](LICENSE)
[![GitHub Repo](https://img.shields.io/badge/GitHub-mihomo--skill-blue)](https://github.com/TYBLHQY/mihomo-skill)
[![Claude Code](https://img.shields.io/badge/Claude%20Code-Skill-purple)](https://claude.ai/code)

**Mihomo**（[Clash Meta](https://github.com/MetaCubeX/mihomo) 的二次开发分支）是一个基于规则的高性能跨平台代理客户端核心。本仓库提供了 Claude Code 的 Mihomo 技能，包含完整的配置指南和使用参考。

## ✨ 特点

- **完整覆盖**：从基础配置到高级特性，覆盖 Mihomo 所有核心功能
- **场景模板**：内置 4 种常见使用场景的生产级配置
- **DNS 架构**：详解三层级 DNS 架构方案
- **故障排查**：10 类常见问题及解决方法
- **Claude Code 集成**：可在 Claude Code 中直接使用 `/mihomo` 快速调取

## 📖 文档

| 文件 | 说明 |
|------|------|
| [`SKILL.md`](./SKILL.md) | Mihomo 配置完整指南（主文档） |
| [`evals/evals.json`](./evals/evals.json) | 技能评估与测试数据 |

### SKILL.md 涵盖内容

- **核心概念** — Proxies / Proxy Groups / Rules / DNS / TUN
- **全局配置** — 端口、网络、日志、Geo 数据库、Hosts
- **代理节点** — Shadowsocks / VMess / VLESS / Trojan / Hysteria / TUIC / WireGuard 等 20+ 协议
- **代理集合 (Proxy Providers)** — 订阅管理、过滤器、锚点复用
- **代理组 (Proxy Groups)** — select / url-test / fallback / load-balance / relay / smart
- **规则系统** — 20+ 规则类型、规则集合 (Rule Providers)、MRS 二进制格式
- **DNS 配置** — 三层级架构（nameserver / proxy-server-nameserver / direct-nameserver）
- **TUN 模式** — 虚拟网卡全局代理、system/gvisor/mixed 栈对比
- **域名嗅探 (Sniffer)** — 协议识别、强制嗅探
- **入站监听** — 多端口分流、加密入站（服务器模式）
- **Smart 内核** — LightGBM 机器学习预测节点延迟
- **常见模板** — 基础客户端 / HTTP 代理 / 局域网共享 / 精细化分流
- **故障排查** — 10 类常见问题及解决方案

## 🚀 在 Claude Code 中使用

在 Claude Code 会话中直接输入：

```
/mihomo <你的问题>
```

例如：
- `/mihomo 帮我写一个 TUN 模式的配置`
- `/mihomo 三层级 DNS 怎么配置`
- `/mihomo 规则不生效是什么原因`

## 🔗 参考链接

| 资源 | 地址 |
|------|------|
| 官方文档 | [wiki.metacubex.one](https://wiki.metacubex.one) |
| Mihomo 源码 | [github.com/MetaCubeX/mihomo](https://github.com/MetaCubeX/mihomo) |
| Smart 内核 (魔改版) | [github.com/vernesong/mihomo](https://github.com/vernesong/mihomo) |
| 规则集数据 | [github.com/MetaCubeX/meta-rules-dat](https://github.com/MetaCubeX/meta-rules-dat) |
| Web 面板 (metacubexd) | [github.com/MetaCubeX/metacubexd](https://github.com/MetaCubeX/metacubexd) |
| Qure 图标集 | [github.com/Koolson/Qure](https://github.com/Koolson/Qure) |
| 秋风广告拦截 | [awavenue.top](https://awavenue.top) |

## 📄 许可

[MIT](./LICENSE)
