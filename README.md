# Mihomo 配置说明

本文说明仓库中的 `config_mihomo.yaml`：使用规则分流、TUN、Fake-IP 和 DoH，并保存策略组选择与 Fake-IP 映射。仓库当前没有 `config.yaml`。

规则源主要来自：

- [MetaCubeX/meta-rules-dat](https://github.com/MetaCubeX/meta-rules-dat)
- [blackmatrix7/ios_rule_script](https://github.com/blackmatrix7/ios_rule_script)
- [qichiyuhub/rule](https://github.com/qichiyuhub/rule)
- 本仓库 `main` 分支的自定义列表

## 快速开始

1. 在本地配置副本的 `proxy-providers` 中填写自己的订阅 URL。

2. 启动 Mihomo

```bash
mihomo -f config_mihomo.yaml
```

3. 配置系统代理

HTTP 和 SOCKS5 均可使用 Mixed 端口 `127.0.0.1:7890`。当前文件没有单独配置 HTTP、SOCKS、Redir 或 TProxy 端口；使用时同时遵循本地配置的认证设置。

## 端口与基础参数

| 项目 | 值 |
|---|---|
| Mixed 端口（HTTP / SOCKS5） | `7890` |
| DNS 监听 | `0.0.0.0:1053` |
| `allow-lan` | `true` |
| `bind-address` | `*` |
| `ipv6` | `true` |
| `log-level` | `info` |
| `tcp-concurrent` | `true` |
| `unified-delay` | `true` |

## TUN 配置

当前配置默认启用 TUN：

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

以下默认选择指没有已保存选择时的首选项。`profile.store-selected: true` 会保存手动选择，因此加载配置后的实际选择可能不同。

| 组名 | 默认选择 | 可调整的行为 |
|---|---|---|
| `🚀 Proxies` | `🇭🇰 HongKong` | 可改选其他地区组或引入的出站节点 |
| `🎯 Direct` | `Direct`（`type: direct` 出站） | 也可选择 `🚀 Proxies`，并非固定直连 |
| `🐟 Others` | `🚀 Proxies` | 兜底流量也可切换到 `🎯 Direct` |

### 区域测速组（`url-test`）

- `🇭🇰 HongKong`
- `🇸🇬 Singapore`
- `🇯🇵 Japan`
- `🇨🇳 Taiwan`
- `🇺🇲 UnitedStates`

地区组通过名称正则筛选引入的节点，测速间隔为 `300` 秒，容差为 `20` 毫秒。组内使用 `include-all: true`，并按各自 `filter` 过滤；主代理组是手动选择组，不会自动在五个地区之间择优。

### 业务分组

| 业务组 | 默认选择 |
|---|---|
| `Binance` / `Crypto` / `CEX` / `Telegram` / `YouTube` / `AI` / `Steam` / `Spotify` / `🎞️ PikPak` | `🚀 Proxies` |
| `Apple` / `Bilibili` | `🎯 Direct` |

这些业务组均为 `select`；各组可选项以配置为准。设置了 `include-all: true` 的组还会引入本地出站和订阅节点，包括本地 `Direct` 出站，所以组名本身不保证最终一定走代理。

## 规则优先级（按配置顺序）

规则自上而下匹配，前面的命中会优先决定出口。以下顺序对应当前 `rules`：

1. `AND`：同时满足 `cn!_domain`、目标端口 `443`、协议 `UDP` → `REJECT`。
2. `myRules` → `🚀 Proxies`。
3. `private_domain`，随后 `private_ip`（`no-resolve`）→ `🎯 Direct`。
4. `binance_domain`，随后 `bybit_domain` → `Binance`。
5. `crypto` → `Crypto`；随后 `kraken_domain`、`ibkr_domain`、`cex` → `CEX`。
6. `apple_cn_domain` → `🎯 Direct`；随后 `apple_domain` → `Apple`。
7. `steam_cn_domain` → `🎯 Direct`；随后 `steam_domain` → `Steam`。
8. `youtube_domain` → `YouTube`；随后 `spotify_domain` → `Spotify`。
9. `telegram_domain` → `Telegram`；随后 `ai_domain`、`openai_domain` → `AI`。
10. `bilibili_domain`、`bilibili_intl_domain` → `Bilibili`；随后 `pikpak_domain` → `🎞️ PikPak`。
11. `microsoft_cn_domain` → `🎯 Direct`；随后 `microsoft_domain` → `🚀 Proxies`。
12. `game`，随后 `google_ip` → `🚀 Proxies`。
13. `telegram_ip` → `Telegram`。
14. `gfw_domain`，随后 `cn!_domain`（来源为 `geolocation-!cn`）→ `🚀 Proxies`。
15. 配置中两条指定目标 IP 的 `/32` 规则（`no-resolve`）→ `🎯 Direct`。
16. `cn_domain`，随后 `cn_ip` → `🎯 Direct`。
17. `MATCH` → `🐟 Others`。

首条 `AND` 要求三个条件同时满足，主要针对命中集合的 QUIC 流量，并非整站屏蔽；同一域名的 TCP 请求继续向下匹配。支持回退的应用可转用 TCP。

`MyRules.list` 中的 `DOMAIN-SUFFIX,push.apple.com` 是有意保留的优先代理规则，用于满足接收国外服务推送的使用场景。未被首条拦截时，它先于 Apple 业务规则命中主代理组；调整 `Apple` 组不会改变这项分流。

国内子集先于对应业务大集合，属于有意的直连组例外。两条 `/32` 例外则位于境外域名规则之后，不能覆盖更早的域名命中。`no-resolve` 表示该 IP 规则不主动触发解析，已有目标 IP 时仍可匹配；`google_ip`、`telegram_ip`、`cn_ip` 未设置此选项，可能触发解析后判断。

## 规则集来源

### MRS（MetaCubeX）

- `apple_cn_domain` / `apple_domain`
- `steam_cn_domain` / `steam_domain`
- `microsoft_cn_domain` / `microsoft_domain`
- `youtube_domain`
- `spotify_domain`
- `bilibili_domain` / `bilibili_intl_domain`
- `pikpak_domain`
- `binance_domain` / `bybit_domain` / `kraken_domain` / `ibkr_domain`
- `telegram_domain` / `ai_domain` / `openai_domain`
- `private_domain` / `gfw_domain` / `cn!_domain` / `cn_domain`
- `private_ip` / `cn_ip` / `google_ip` / `telegram_ip`

这些集合使用 MetaCubeX `meta-rules-dat` 的 `meta` 分支；域名集合为 `behavior: domain`，IP 集合为 `behavior: ipcidr`，均使用 `format: mrs`。其中 `ai_domain` 对应 `category-ai-!cn.mrs`，`cn!_domain` 对应 `geolocation-!cn.mrs`。

### TEXT（第三方与自定义）

| Provider | 来源 | 格式与用途 |
|---|---|---|
| `myRules` | 本仓库 `main` 分支的 `MyRules.list` | `classical` / `text`，主代理优先规则 |
| `cex` | 本仓库 `main` 分支的 `CEX.list` | `classical` / `text`，送往 CEX 组 |
| `crypto` | 本仓库 `main` 分支的 `Crypto.list` | `classical` / `text`，送往 Crypto 组 |
| `game` | blackmatrix7 `ios_rule_script` 的 `master` 分支，`rule/Clash/Game/Game.list` | `classical` / `text`，送往主代理组 |
| `fakeipfilter_cn` / `fakeipfilter_!cn` | qichiyuhub `rule` 的 `main` 分支，`rules/fakeipfilter-cn.list` / `rules/fakeipfilter-!cn.list` | `domain` / `text`，用于 Fake-IP 过滤和 DNS policy |

上述 provider 全部为 `type: http`，更新间隔为 `86400` 秒。自定义列表通过 GitHub Raw 读取远端版本，不直接读取本地同名文件；仅修改工作区列表不会立即改变客户端加载的内容。

本地 `Binance.list` 未被 `config_mihomo.yaml` 引用，当前 Binance 分流使用公共 `binance_domain` 集合。AI 分流使用 MetaCubeX 集合，未引用 ACL4SSR AI 列表。远端集合会更新，本文不固定其内容条数或域名覆盖快照。

## DNS 与 Fake-IP

启用 `enhanced-mode: fake-ip`，IPv4 Fake-IP 范围为 `198.18.0.1/16`，过滤模式为 `blacklist`。过滤集合包括 `fakeipfilter_cn`、`fakeipfilter_!cn`、`private_domain`、`cn_domain`、`apple_cn_domain`、`steam_cn_domain`、`microsoft_cn_domain`；命中过滤时不下发 Fake-IP。是否使用 Fake-IP 与流量是否直连是两件事，出口仍由 `rules` 和所选策略决定。

| 配置项 | 当前解析策略 |
|---|---|
| `default-nameserver` | `https://223.5.5.5/dns-query`，用于解析 DNS 服务器自身的域名 |
| `proxy-server-nameserver` | `https://dns.alidns.com/dns-query` 与 `https://doh.pub/dns-query`，用于代理节点域名 |
| `direct-nameserver` | 同上两个 DoH；启用 `direct-nameserver-follow-policy: true`，直连出口的解析也遵循 policy |
| policy：`cn_domain,private_domain,fakeipfilter_cn,steam_cn_domain,microsoft_cn_domain,apple_cn_domain` | 阿里与腾讯 DoH，均附加 `disable-qtype-65=true` |
| policy：`fakeipfilter_!cn` | `https://1.1.1.1/dns-query` 与 `https://8.8.8.8/dns-query`，均指定 `🚀 Proxies` 并附加 `disable-qtype-65=true` |
| `nameserver` | `https://8.8.8.8/dns-query`，指定 `🚀 Proxies`，附加 `ecs=223.5.5.0/24` |
| `fallback` | 同一 Google DoH、同一主代理组，不附加上述 ECS；`fallback-filter` 启用 GeoIP，国家代码为 `CN` |

`nameserver-policy` 优先于普通 `nameserver` / `fallback` 查询；`disable-qtype-65` 屏蔽 HTTPS 类型 DNS 回应。`respect-rules: true` 让 DNS 连接遵循路由规则，带 `#🚀 Proxies` 的上游则显式指定该组。主、备用默认 DNS 共用地址和主代理组，并非独立故障备用通道。

`private_domain` 也被 policy 分配到公网 DoH：若名称只存在于局域网 DNS、且未命中 hosts，这份配置不会自动转向局域网解析器，需按实际环境另外配置。

同时启用 `use-hosts`、`use-system-hosts`、`cache-algorithm: arc`，`prefer-h3: false`。Fake-IP 持久化使用顶层 `profile.store-fake-ip: true`，没有配置单独的 `fakeip.db` 路径。

## 自定义规则维护

当前配置引用的自定义列表为：

- `MyRules.list`
- `CEX.list`
- `Crypto.list`

列表采用 `classical` / `text` 格式，每行写匹配条件，不附加策略组名称；出口由配置中的 `RULE-SET,集合名,策略组` 指定。例如：

```text
DOMAIN,api.example.com
DOMAIN-SUFFIX,example.com
DOMAIN-KEYWORD,keyword
IP-CIDR,192.0.2.1/32,no-resolve
```

`DOMAIN-KEYWORD` 是子串匹配，不是服务归属判断；例如当前 `okx` 关键词也会命中 `bookxnote.com`。自定义规则位置靠前，可能覆盖后续业务或国内规则。本说明不代表这些关键词已被收窄；维护时应核对所需主域名及备用域名，再调整覆盖范围。

## 使用与验证

- 当前订阅与规则 provider 的更新间隔均为 `86400` 秒；更新成功、缓存刷新和客户端实际加载情况需要分别确认。
- 本文描述仓库配置。使用 OpenClash 等客户端时，源配置、生成的运行时配置与客户端覆盖设置可能不同，不能仅凭 README 判断路由器已生效。
- 语义参考：[路由规则](https://wiki.metacubex.one/config/rules/)、[代理组](https://wiki.metacubex.one/config/proxy-groups/)、[DNS](https://wiki.metacubex.one/config/dns/)、[规则集合格式](https://wiki.metacubex.one/config/rule-providers/content/)。
