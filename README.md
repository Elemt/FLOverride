# FLOverride

FlClash / Mihomo 覆写脚本：保留机场订阅的节点，自动按地区分组，按服务分流，并做了防 DNS 泄露处理。

文件：`flclash-override.js`

---

## 一、怎么用（FlClash 安卓 / 桌面）

> 不同版本菜单名可能略有差异。

1. 打开本仓库的 `flclash-override.js`，点右上角 **Copy raw file** 复制全部内容。
2. FlClash → **配置** → 选中你的订阅 → 右上角 **⋮** → **覆写**（Override）。
3. 覆写方式选 **脚本**，新建脚本，把内容粘贴进去，保存并启用。
4. 回到 **工具 / 设置**：
   - 关闭 **覆写 DNS**（否则界面里的 DNS 设置会替换脚本里的 DNS，防泄露失效）。
   - 打开 **虚拟网卡 / TUN**。
5. 安卓系统：**设置 → 网络 → 私人 DNS → 关闭**（否则 DoT 会绕过劫持）。
6. 重启 FlClash，更新一次订阅。

> 仓库是私有的，raw 链接带临时 token，不适合做远程订阅地址，所以用复制粘贴的方式。

### 检查是否生效

- 打开 <https://browserleaks.com/dns> 或 <https://ipleak.net>
- 正常结果：DNS 服务器只显示**代理节点所在地**，不出现国内运营商 / 阿里 / 腾讯 DNS。

---

## 二、分组说明

| 分组 | 作用 | 默认 |
|---|---|---|
| 一键代理 | 兜底，所有未命中规则的流量 | 第一个地区组 |
| MEXC | 域名含 `mexc` 的流量 | 台湾（没有台湾节点时退回一键代理） |
| 广告拦截 | 广告域名 | REJECT，误杀时切 DIRECT |
| AI | OpenAI / Claude / Gemini 等 | 第一个地区组 |
| 加密货币 | 交易所、行情网站 | 第一个地区组 |
| YouTube / Google / GitHub | 对应服务 | 第一个地区组 |
| Disney / Netflix / Spotify | 流媒体 | 第一个地区组 |
| Steam下载 | Steam 下载 CDN | 国内直连（省流量） |
| Steam / Telegram / TikTok | 对应服务 | 第一个地区组 |
| Apple / Microsoft | 对应服务 | 国内直连 |
| 网络检测 | ipinfo、ipleak、browserleaks 等 | 第一个地区组 |
| 国内直连 | DIRECT | — |

**地区组**（url-test，自动选延迟最低的节点；没有该地区节点时不生成）：
香港、台湾、日本、新加坡、韩国、菲律宾、美国、加拿大、英国、其他地区。

机场自带的「自动选择 / 故障转移 / 全部节点」类分组会被保留。

---

## 三、规则顺序

```
局域网直连
DOMAIN-KEYWORD,mexc        → MEXC
广告                        → 广告拦截
网络检测                    → 网络检测
AI                          → AI
加密货币                    → 加密货币
YouTube（必须在 Google 前）  → YouTube
Google / GitHub / Disney / Netflix / Spotify
Steam下载（必须在 Steam 前） → Steam下载
Steam / Telegram / TikTok / Apple / Microsoft
GEOSITE/GEOIP private, cn   → 国内直连
MATCH                       → 一键代理
```

规则集来自 [n0de-sudo/Perfect-Rules](https://github.com/n0de-sudo/Perfect-Rules)，每 24 小时自动更新。

---

## 四、DNS 防泄露设计

- `enhanced-mode: fake-ip`，`fake-ip-range: 198.18.0.1/16`（不和 TUN 的 172.19.0.1/30 冲突）
- `respect-rules: true`：DNS 查询跟着分流规则走，国外域名的 DNS 经代理发出
- 默认 `nameserver` 用 Cloudflare / Google DoH；只有 `geosite:cn,private` 用阿里 / 腾讯 DoH
- `proxy-server-nameserver` 用国内 DoH 解析节点域名，避免死循环
- 删除了 `fallback`，避免国内外 DNS 并发查询
- TUN `dns-hijack: any:53` + `tcp://any:53`
- `ipv6: false`，避免 IPv4 / IPv6 出口不一致

---

## 五、常见修改

**让某个网站走指定地区**（例：binance 走日本）——在 `config["rules"]` 里、MEXC 那行旁边加：

```js
"DOMAIN-KEYWORD,binance,日本",
```

目标必须是已存在的分组名；如果你的机场可能没有该地区节点，参照脚本里 MEXC 的写法建一个带退路的分组。

**新增地区分组**（例：越南）——改 5 处：

1. `groupIcons` 加 `"越南": "Other.png",`
2. `managedGroups` 加 `"越南": true,`
3. `regionPatterns` 加 `"越南": [/越南/i, /\bVN\b/i, /Vietnam/i],`
4. `regionNodes` 加 `"越南": [],`
5. `regionOrder` 里放到想要的位置

**广告误杀**：在 FlClash 里把「广告拦截」组切到 DIRECT 即可，不用改脚本。

---

## 六、排错

- 配置加载失败：多半是规则里引用了不存在的分组名，检查拼写。
- 检测仍有国内 DNS：确认「覆写 DNS」和系统「私人 DNS」都已关闭，并重启 FlClash。
- 规则集下载失败：jsDelivr 偶尔不稳定，稍后点「更新」或重启。
