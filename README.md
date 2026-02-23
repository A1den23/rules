# Mihomo 配置说明

本仓库提供一套基于 Mihomo 的规则分流配置，包含两份可直接使用的配置文件：

- `config.yaml`：精简稳定版
- `config_mihomo.yaml`：增强 DNS 版（更多缓存与策略项）

规则源主要来自：

- [MetaCubeX/meta-rules-dat](https://github.com/MetaCubeX/meta-rules-dat)
- [blackmatrix7/ios_rule_script](https://github.com/blackmatrix7/ios_rule_script)
- [ACL4SSR/ACL4SSR](https://github.com/ACL4SSR/ACL4SSR)

## 快速开始

1. 编辑订阅

```yaml
proxy-providers:
  FlowerCloud:
    url: "你的订阅链接"
```

2. 启动 Mihomo

```bash
mihomo -f config.yaml
# 或
mihomo -f config_mihomo.yaml
```

3. 配置系统代理

- HTTP: `127.0.0.1:7890`
- SOCKS5: `127.0.0.1:7891`
- Mixed: `127.0.0.1:7893`

## 端口与基础参数

| 项目 | 值 |
|---|---|
| HTTP 端口 | `7890` |
| SOCKS 端口 | `7891` |
| Redir 端口 | `7892` |
| Mixed 端口 | `7893` |
| TProxy 端口 | `7894` |
| DNS 监听 | `0.0.0.0:1053` |
| `allow-lan` | `true` |
| `bind-address` | `*` |
| `ipv6` | `true` |
| `log-level` | `warning` |
| `tcp-concurrent` | `true` |
| `unified-delay` | `true` |

## TUN 配置

两份配置均默认启用 TUN：

```yaml
tun:
  enable: true
  stack: mixed
  mtu: 1400
  dns-hijack: ["any:53", "tcp://any:53"]
  auto-route: true
  auto-redirect: true
  auto-detect-interface: true
```

## 代理组

### 主分组

- `🚀 Proxies`：主代理组
- `🎯 Direct`：直连优先
- `🐟 Others`：兜底分流

### 区域测速组（`url-test`）

- `🇭🇰 HongKong`
- `🇸🇬 Singapore`
- `🇯🇵 Japan`
- `🇨🇳 Taiwan`
- `🇺🇲 UnitedStates`

### 业务分组

- `Binance`
- `CEX`
- `Telegram`
- `YouTube`
- `AI`
- `Apple`
- `Steam`
- `Spotify`
- `Bilibili`
- `🎞️ PikPak`

## 规则优先级（按配置顺序）

1. Steam CN / 私网直连
2. Binance / Hyperliquid → `Binance`
3. Bybit / Kraken / IBKR / CEX → `CEX`
4. Apple / Steam / YouTube / Spotify
5. Telegram / AI / OpenAI
6. Bilibili / PikPak
7. Game / MyRules / GFW / Google IP
8. Telegram IP → `Telegram`
9. `geolocation-!cn` → `🚀 Proxies`
10. `cn_domain` / `cn_ip` → `🎯 Direct`
11. `MATCH` → `🐟 Others`

## 规则集来源

### MRS（MetaCubeX）

- `apple_cn_domain` / `apple_domain`
- `steam_cn_domain` / `steam_domain`
- `binance_domain`
- `youtube_domain`
- `spotify_domain`
- `bilibili_domain` / `bilibili_intl_domain`
- `pikpak_domain`
- `bybit_domain` / `kraken_domain` / `ibkr_domain`
- `telegram_domain` / `openai_domain`
- `private_domain` / `gfw_domain` / `geolocation-!cn` / `cn_domain`
- `cn_ip` / `google_ip` / `telegram_ip`

### TEXT（第三方与自定义）

- `myRules`: `MyRules.list`
- `cex`: `CEX.list`
- `Hyperliquid`: `Hyperliquid.list`
- `game`: blackmatrix7 Game 列表
- `ai`: ACL4SSR AI 列表

### 共同点（两份配置）

- `enhanced-mode: fake-ip`
- `fake-ip-range: 198.18.0.1/16`
- 国内 DNS：`223.5.5.5` / `119.29.29.29`
- 启用 `respect-rules: true`
- Fake-IP 过滤包含：`private_domain`、`cn_domain`、微软连通性检测域名等

## 自定义规则维护

你可以直接修改以下文件：

- `MyRules.list`
- `CEX.list`
- `Hyperliquid.list`

示例格式：

```text
DOMAIN,suffix.example.com,🚀 Proxies
DOMAIN-KEYWORD,keyword,🚀 Proxies
IP-CIDR,1.2.3.4/32,🚀 Proxies,no-resolve
```

## 使用建议

- 日常使用优先 `config.yaml`（更简洁）。
- 需要更细 DNS 策略时使用 `config_mihomo.yaml`。
- 若客户端支持配置热更新，建议将订阅更新周期与规则更新周期都保持为 `86400`（当前配置已设置）。
