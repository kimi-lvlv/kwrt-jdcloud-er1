# ER1 SSR Plus Handoff

## 状态时间

最近一次路由器现场处理：`2026-08-16 13:40`（Asia/Shanghai）。

> **重要方向校正（2026-08-16）**：之前把「国内服务变慢」归因于 SSR/nft 规则（快照污染、UDP whitelist 顺序、白名单 DNS 等）的方向是**错的**，那些排查点反复把人带进坑。本次实际根因与 SSR 配置完全无关，见下方「实际根因」。

最后一次旧记录（按旧方向）：`2026-08-15 22:32`（Asia/Shanghai）。

目标设备：京东云太乙 ER1，管理地址 `10.0.0.1`。本次工作只涉及路由器，没有修改本机代理、浏览器或本机 DNS。

## 当前状态（2026-08-16 13:40 后）

- SSR Plus 主节点：`cfg014a8f`（`Junqi-EO-HK-Direct`）；分流模式：`gfw`
- Xray/V2Ray 进程：1 个；dns2tcp、dnsmasq 均运行中
- **已停止 docker 容器 `cloudflare-mesh`（Cloudflare WARP / warp-svc）**：它会内存泄漏，占用 ~1GB RSS，是本次「国内慢」真正的根因。
- 内存：停止后 `MemAvailable` 从 ~627MB 恢复到 ~1566MB，`MemFree` 从 67MB 恢复到 ~1GB。
- 国内 URL 实测：jd/taobao/baidu/bilibili 均 200，耗时 0.28–0.38s。
- 现有 SSR 分流本身是正常的（国内 URL 代理计数增量为 0；外部 URL 走代理），本次未改动任何 SSR/nft 文件。

## 实际根因（2026-08-16，用户要求跳出旧排查方向）

用户反复报「SSR+ 运行久了之后，国内服务打开慢、图片加载慢」，且「SSR 设置没改过」。

之前的 AI 把它当成 SSR 分流问题排查（nft 快照、UDP whitelist 顺序、白名单 DNS / 双表 nftset、gfwlist DNS 走 dns2tcp 等），每次都陷入坑里。**实际根因与 SSR 完全无关**：

- 路由器上跑着一个 docker 容器 **`cloudflare-mesh`（镜像 `cloudflare/mesh:latest-arm64`，进程 `warp-svc`）**。
- `warp-svc` 从 2026-08-13 21:40(Asia/Shanghai) 起连续运行约 2.6 天，**内存泄漏到 `VmRSS ≈ 1010MB`**（约占整台 2GB 内存的一半），当时系统 `MemFree` 仅 67MB。
- 内存被无关的 WARP 吃光后，系统整体变慢，表现为「运行多日后国内服务和图片加载变慢」；重启 WARP（或路由器）即恢复，所以很难复现抓取——每次抓到前就被重启掉了。
- **处置**：经用户确认后 `docker stop cloudflare-mesh`（容器进入 `Exited (137)`，未删除，可随时 `docker start` 恢复）。内存释放后国内访问恢复。

诊断时的辅助证据（供复现参考）：
- `cat /proc/<warp-pid>/status` 的 `VmRSS` 高达约 1000MB。
- `docker stats --no-stream` 显示 `cloudflare-mesh` 占用 ~983MiB。
- conntrack 中 `172.17.0.2 → 162.159.197.2:443` 有大量 UDP/QUIC（WARP 隧道流量）。
- 排查时曾误观察到的 `drop443` 计数增长（UDP 443 drop），主因正是该 WARP 容器的 QUIC 流量，非 SSR 问题。

## 二次追加：gfw 分流 bug（2026-08-16 14:05，用户切 20k gfw 版后）

> **注意**：上面「与 SSR 完全无关」的结论**不完全正确**——WARP 内存泄漏是「运行久了变慢」的主因之一，但**还查出一个独立的 SSR 分流 bug**，会导致 gfw 规则不生效。

用户把 gfw 规则切到 ~23k 条版本后，发现国内访问仍偏慢、外网异常。排查发现：

**Bug：gfwlist 集合写入失败，gfw 分流不生效**
- `/usr/share/shadowsocksr/gfw2ipset.sh`（nftables 分支）生成 gfw_base/gfw_list/blacklist 的 nftset 时用的是 `nftset=/域名/inet#ss_spec#gfwlist`，**缺少 dnsmasq 2.93 要求的地址族前缀（应为 `4#`）**，导致 dnsmasq 写不进 gfwlist 集合。
- 佐证：`nft list set inet ss_spec gfwlist` 为空（0 元素）；而白名单那行用了 `4#inet#...4#ip#...` 所以 whitelist 正常写入。同一脚本里「一个能写、一个不能写」。
- 用 nslookup 解析 google/youtube/github 后 gfwlist 集合仍为 0，证实 dnsmasq 未写入。
- 后果：`ip daddr @gfwlist jump ss_spec_wan_fw` 永不命中，20k gfw 规则实际未生效（外网多被直连/被墙）；但不直接导致国内直连变慢。

**修复**（已改 `gfw2ipset.sh.bak-20260816-135930` 备份下）：
- 第 37 行 gfw 规则 nftset：`inet#ss_spec#\2` → `4#inet#ss_spec#\2,4#ip#ss_spec_mangle#\2`（双表，TCP+UDP 都写，集合在 `ip ss_spec_mangle` 表存在）。
- 第 89 行 blacklist nftset 加 `4#` 前缀（仅写 `inet`，因 `ip ss_spec_mangle` 表没有 blacklist 集合，避免 nftset 指向不存在的集合）。
- 改后 `sh -n` 语法通过，`/etc/init.d/shadowsocksr restart` 生效；验证 gfw_base/gfw_list 均为 `4#inet#ss_spec#gfwlist,4#ip#ss_spec_mangle#gfwlist`，解析 gfw 域名后 gfwlist 集合从 0 增长，外网 google/github/youtube 走代理成功，国内 jd/taobao 直连正常。

**「国内偏慢」的另外一层（已随 dns2tcp 恢复而缓解）**：
- 用户切 20k 版重启 SSR 时 `dns2tcp` 未起来，5335 端口无监听；而 gfw 规则把所有 gfw 域名指到 `127.0.0.1#5335` → 每次解析等 ~5 秒超时。页面一旦引用任一 gfw 域名资源（统计/广告/CDN/外链）就整体卡顿。
- 用户切回 dns2tcp 后（pdnsd_enable 恢复），该层缓解。当前 dnsmasq 冷缓存时白名单域名首次解析仍偶发偏慢（223.5.5.5 冷响应），缓存命中后正常（京东 0.3s、淘宝 0.1s、优酷 2.5s 属站方本身）。

**待观察边界**：以上均基于路由器本机（output 路径）测试；LAN 客户端（prerouting 路径）及 IPv6 未完整覆盖。优酷 total≈2.5s 是它页面本身重，非路由误判（解析/直连正常）。

## 三次追加：分流失效的用户主因 = iPhone iCloud 私有中继（2026-08-16 14:20）

用户反馈「用 ip.net.coffee 检测看到全部流量走代理 / 只有一台 iPhone(10.0.0.187) 慢」。排查结论：

- **不是 SSR 分流问题**：conntrack 里 10.0.0.187 及其他 LAN 设备 0 条连接回包来自 127.0.0.1（即无一条被 SSR redirect）。ip.net.coffee 解析到 `172.67.213.99 / 104.21.53.138`（Cloudflare 境外），属境外服务，gfw 分流下走代理本属正常。
- **真正原因**：该 iPhone 开着 **iCloud 私有中继（Private Relay）**。它把 iPhone 的流量（含国内）全部经 Apple 境外代理中转，造成「全部走代理」且国内访问慢；只影响开中继的那台设备。关闭后恢复。
- 附带查明：gfw `dns2tcp` 进程若未随 SSR 启动（5335 无监听），gfw 域名会被 `server=/域/127.0.0.1#5335` 拖到 ~5s 超时，是国内页面卡顿的一个次要诱因（用户切回 dns2tcp 已缓解）。

## GFW/国内分流最终验收（2026-08-16 14:25，gfw 模式）

- `run_mode=gfw`、`global_server=cfg014a8f`。
- 集合规模：china 3811 个 IP、gfwlist 52 个、whitelist 124、blacklist 7。
- 已确认 `gfw2ipset.sh` 含修复后的 `4#inet#ss_spec#\2,4#ip#ss_spec_mangle#\2`（gfwlist 双表）与 `4#inet#ss_spec#blacklist`。
- wan_ac 顺序：blacklist jump → whitelist return → fplan jump → bplan return → music return → **china return → gfwlist jump** → gmlan。
- 实测（路由器本机，代理计数器增量）：
  - 境内直连：jd/taobao/bilibili 增量 **0**，均 200（jd 0.5s、taobao 0.34s、bilibili 0.22s）。
  - 境外走代理：google +3 / youtube +3 / twitter +5 / github +3，均 200。
  - 境外域名的解析 IP 均落入 gfwlist 集合（google 142.251.156.119 等、github 140.82.113.3、twitter 162.159.140.229）。
- 结论：**gfw 国内直连 + 境外代理分流功能正常**，修复的 nftset 前缀 bug 已生效并验证。

## （旧）SSR 分流状态——不再视为根因，仅供参考

SSR 分流本身本次验证是正常的：国内直连（代理计数增量 0）、外部走代理，无改动。
- SSR Plus 主节点：`cfg014a8f`（`Junqi-EO-HK-Direct`）
- 分流模式：`gfw`
- Xray/V2Ray 进程：1 个
- SSR Plus 自动规则快照进程：0 个
- 动态 nft 快照：不存在
- dnsmasq：运行中，配置语法检查通过
- 国内直连白名单：同时写入 `inet ss_spec` 和 `ip ss_spec_mangle`，验证时有 61 个 IPv4 元素

## 已解决的问题

用户现象是 SSR Plus 开启时，国内购物和视频网站加载慢，部分站点表现出境外定位或异常加载。

确认的共享根因有两处：

1. SSR Plus 的 `ssr-rules` 会把动态 `china/gfwlist/whitelist` 集合保存到 nft 快照，后续重启或防火墙 include 可能恢复过期的 IP 分类。
2. UDP/TProxy 规则创建了 `whitelist` 集合，却没有在 `udp/443 drop` 之前检查它。国内 CDN IP 未落入 China IP 集合时，QUIC 被丢弃，客户端回退到 TCP 后变慢。

另外，原来的白名单 dnsmasq 规则使用 `server=/domain/127.0.0.1`，容易把白名单解析重新送回本地 dnsmasq；现在改为使用当前 WAN 的第一个 IPv4 DNS，并用一条合并的 `nftset` 同时写入两张表。

## 已修改的路由器文件

- `/usr/bin/ssr-rules`
  - 禁止保存和恢复动态 nft 快照。
  - 防火墙 include 只保留空操作，不再恢复旧快照。
  - UDP TProxy 链在 China IP 和 UDP/443 丢弃规则之前增加 `@whitelist return`。
- `/usr/share/shadowsocksr/gfw2ipset.sh`
  - 白名单使用当前 WAN IPv4 DNS。
  - 生成 `4#inet#ss_spec#whitelist,4#ip#ss_spec_mangle#whitelist` 合并 nftset。
- `/etc/init.d/shadowsocksr`
  - nft 后端不再传递 `-A` 自动持久化参数。
  - 保留启动/停止时清理旧 SSR 规则和缓存的逻辑。
- `/etc/ssrplus/white.list`
  - 增加京东、淘宝/天猫、拼多多、B 站、爱奇艺、优酷、芒果、腾讯视频、抖音、快手等常用国内域名及 CDN 后缀。

## 备份

聚焦修复前的备份：

- `/usr/bin/ssr-rules.bak-20260815-222849`
- `/usr/share/shadowsocksr/gfw2ipset.sh.bak-20260815-222849`
- `/etc/init.d/shadowsocksr.bak-20260815-222849`
- `/etc/ssrplus/white.list.bak-20260815-222849`

第一阶段的备份：

- `/etc/init.d/shadowsocksr.bak-20260815-215733`
- `/etc/config/shadowsocksr.bak-20260815-215733`

备份中不包含或不应包含 SSH 密码、节点 UUID、订阅地址等凭据。不要把登录凭据写入仓库、日志或 handoff。

## 验证证据

路由器本机在 SSR Plus 保持运行时测试：

| URL | HTTP | 总耗时 | `ss_spec_wan_fw` 代理计数增量 |
| --- | ---: | ---: | ---: |
| `https://www.jd.com` | 200 | 0.346 s | 0 |
| `https://www.taobao.com` | 200 | 0.173 s | 0 |
| `https://mobile.yangkeduo.com` | 200 | 0.653 s | 0 |
| `https://www.bilibili.com` | 200 | 0.323 s | 0 |
| `https://www.iqiyi.com` | 200 | 0.154 s | 0 |
| `https://www.google.com` | 200 | 4.727 s | 3 |

其他验证：

- `sh -n` 检查三个脚本返回 0。
- `dnsmasq --test` 返回 `syntax check OK`。
- UDP 链顺序为 `@whitelist return` -> `@china return` -> UDP/80、443 drop。
- 国内域名解析后的 IP 同时出现在 TCP 和 UDP 白名单集合。
- `/usr/share/nftables.d/ruleset-post/99-shadowsocksr.nft` 在重启后仍不存在。

这些是路由器侧验证，不等同于每个 LAN 客户端、每个 IPv6 连接和每个应用的完整端到端测试。

## 已知残留

`/etc/init.d/firewall reload` 在这台固件上返回 `1`，详细表现是 `/sbin/fw4` 通过管道把规则交给 `nft` 时发生段错误；同一份渲染规则用 `fw4 check` 校验通过，SSR 规则和 WAN 访问保持正常。后续排查 SSR 时不要把这个固件级 nft 问题误判成分流根因，也不要为了验证反复 reload 防火墙。

## 后续 Agent 启动步骤

1. 先读本文件和 [`docs/ssr-plus-routing-troubleshooting.md`](docs/ssr-plus-routing-troubleshooting.md)。
2. SSH 直接进入 `root@10.0.0.1`，不要打开浏览器标签或触碰本机代理。
3. 只读检查 `uci`、进程、dnsmasq 生成文件、nft 集合和计数器。
4. 先确认是否出现新的持久化快照，再判断国内 IP 是否命中 `china` 或 `whitelist`。
5. 修改前复制当前文件并记录时间戳；修改后运行脚本语法检查、dnsmasq 语法检查和路由器侧实际请求。
6. 交付时报告实际状态、证据、备份路径和未验证边界，不把静态检查写成端到端已验证。

## 回滚提示

回滚前先备份当前文件。代码回滚示例：

```sh
cp -f /usr/bin/ssr-rules.bak-20260815-222849 /usr/bin/ssr-rules
cp -f /usr/share/shadowsocksr/gfw2ipset.sh.bak-20260815-222849 /usr/share/shadowsocksr/gfw2ipset.sh
cp -f /etc/init.d/shadowsocksr.bak-20260815-222849 /etc/init.d/shadowsocksr
cp -f /etc/ssrplus/white.list.bak-20260815-222849 /etc/ssrplus/white.list
chmod 755 /usr/bin/ssr-rules /usr/share/shadowsocksr/gfw2ipset.sh /etc/init.d/shadowsocksr
rm -f /usr/share/nftables.d/ruleset-post/99-shadowsocksr.nft
/etc/init.d/shadowsocksr restart
```

不要直接恢复旧 nft 快照；如果需要恢复 UCI 配置，先确认备份中的 `global_server` 和 `run_mode` 是否符合用户当前意图。

## 四次追加：补全 AI/Claude/Gemini 代理域名（2026-08-16 14:40）

用户反馈现象：手机端代理软件（Clash）能访问 Gemini/ChatGPT/Claude，但经路由器 SSR Plus 后部分 AI 站点访问不了（或解析不进 gfwlist 集合）。

排查结论：
- 修复 gfw2ipset.sh 的 nftset 前缀 bug 后，openai.com / chatgpt.com / oai* / anthropic.com / claude.ai / perplexity.ai / grok.com / x.ai 等**核心 AI 域名已在 gfw_list 命中并能走代理**（解析 IP 落入 gfwlist 集合）。
- **Gemini 系列在 gfw_list 文本里没有显式收录**（gemini.google.com、ai.google.dev、makersuite.google.com、bard.google.com、generativelanguage.googleapis.com、proactivebackend-pa.googleapis.com 等），但其解析 IP 大多能落入 gfwlist 集合（靠 gfw 里的 Google IP 段），所以之前实测大部分也能通。
- 决定仍补一份「AI 第一方域名」到 `black.list`，作为**显式保底**（防止某些 AI 子域解析 IP 不落入 gfwlist 集合时漏走代理），并让 DNS 走 dns2tcp 隧道解析。

**采购 /etc/ssrplus/black.list（已生效）**：在原有 `air.gidd.tv` 基础上追加：
- OpenAI 第一方：`chat.openai.com.cdn.cloudflare.net` `openai-api.arkoselabs.com` `client-api.arkoselabs.com` `openaicom-api-...azurefd.net` `openaicomproduction...blob.core.windows.net` `production-openaicom-storage.azureedge.net` `openaiapi-site.azureedge.net` `openaicom.imgix.net` `observeit.net` `ai.com`
- Claude：`cdn.usefathom.com`
- Gemini/Google AI：`ai.google.dev` `makersuite.google.com` `alkalimakersuite-pa.clients6.google.com` `generativeai.google` `proactivebackend-pa.googleapis.com` `gemini.google.com` `bard.google.com` `deepmind.google`

**明确不加入 black.list（避免误伤国内其他站点）**：`static.cloudflareinsights.com` `algolia.net` `api.statsig.com` `events.statsigapi.net` `featuregates.org` `launchdarkly.com` `segment.io` —— 这些是第三方 SaaS（Cloudflare 统计、Algolia、Statsig、LaunchDarkly、Segment），国内很多网站也引用，整域无条件代理会造成国内访问劣化。SSR 的 black.list 是域名级（`server=/域/` 匹配子域）无条件代理，不同于 Clash 的精确分流。

**black.list 生效机制**：`gfw2ipset.sh` 第 89 行把 black.list 每行生成 `server=/域/127.0.0.1#5335` + `nftset=/域/4#inet#ss_spec#blacklist`，写入 `blacklist_forward.conf`（dnsmasq 读取）；注释行和空行会被 sed 过滤，安全。改完需重启 SSR。

**验证（路由器本机）**：
- `blacklist_forward.conf` 含 20 个新 AI 域名规则。
- `blacklist` 集合 IP 数 6 → 29。
- gemini.google.com / ai.google.dev / makersuite.google.com / proactivebackend-pa.googleapis.com 解析 IP 均落入 `blacklist`。
- 访问 gemini.google.com=200、aistudio.google.com=200、makersuite.google.com=200，gemini 代理计数增量 +1。
- claude.ai=403、chatgpt.com=403：能建立连接、已走代理，403 是节点地区/IP 限制（与分流无关），需换可用地区节点。

**固件仓库同步**：已将更新后的 black.list 放入 `files/etc/ssrplus/black.list`，下次编译固件内置。
