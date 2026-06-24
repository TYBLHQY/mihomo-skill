# Mihomo Skill

Mihomo（虚空终端 / Clash Meta）Claude Code 技能 — 代理配置完整指南。

## 内容

| 文件 | 说明 |
|------|------|
| [`SKILL.md`](./SKILL.md) | Mihomo 配置完整指南 |
| [`evals/`](./evals) | 技能评估数据 |

## 关于

该仓库是 Claude Code 的 [Mihomo 技能](https://claude.ai/code)，提供 Mihomo (Clash Meta) 代理客户端的配置指南，涵盖：

- 核心概念（Proxies / Proxy Groups / Rules / DNS / TUN）
- 配置文件结构（YAML 语法详解）
- 全局配置、代理节点、代理组配置
- 代理集合 (Proxy Providers) 与规则集合 (Rule Providers)
- 三层级 DNS 架构
- TUN 模式、域名嗅探、入站监听
- 常见配置模板与故障排查

## 参考

- [官方文档](https://wiki.metacubex.one)
- [Mihomo 源码](https://github.com/MetaCubeX/mihomo)
- [规则集数据](https://github.com/MetaCubeX/meta-rules-dat)