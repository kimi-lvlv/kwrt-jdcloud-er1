# ER1 Agent 操作说明

本文件是后续 Agent 处理 ER1 固件、SSR Plus 和路由分流问题时的本地约束。详细现场状态见 [`HANDOFF.md`](HANDOFF.md)；SSR Plus 的诊断流程见 [`docs/ssr-plus-routing-troubleshooting.md`](docs/ssr-plus-routing-troubleshooting.md)。

## 任务边界

- 目标是路由器上的网络路径，不是本机代理。
- 直接 SSH 到 `root@10.0.0.1`，不要求用户打开浏览器、不批量开页面、不用浏览器页面代替路由器证据。
- 不读取、记录或回显密码、节点 UUID、订阅 URL、Bearer、私钥等凭据。
- 不修改本机代理、浏览器代理、hosts、系统 DNS 或用户设备配置，除非用户明确改了范围。
- 不重启路由器、不刷固件、不改分区、不删除用户数据。

## 诊断原则

1. 先看 live state，再看静态脚本：`uci`、进程、监听端口、dnsmasq 生成文件、nft 表/集合/计数器。
2. 把 DNS、TCP 透明代理、UDP/TProxy、IPv4/IPv6 分开判断；不要把“解析成功”当成“路由正确”。
3. 先找共享根因。SSR Plus 当前最重要的两个检查是：
   - `/usr/share/nftables.d/ruleset-post/99-shadowsocksr.nft` 是否重新出现。
   - `ip ss_spec_mangle ss_spec_tproxy` 是否在 UDP/443 drop 前有 `ip daddr @whitelist return`。
4. 每次修改前做带时间戳的文件备份；修改范围要能对应到用户现象。
5. 验证必须包含真实请求和计数器：国内 URL 不应增加代理链计数，外部 URL 应能增加代理链计数。

## 推荐检查顺序

```sh
uci -q get shadowsocksr.@global[0].global_server
uci -q get shadowsocksr.@global[0].run_mode
ps w | grep -E '[v]2ray|[x]ray|[m]ihomo|[d]nsmasq'
find /usr/share/nftables.d /etc/ssrplus /tmp -type f \
  \( -iname '*shadow*' -o -iname '*ssr*' \) -print 2>/dev/null
nft list chain inet ss_spec ss_spec_wan_ac
nft list chain ip ss_spec_mangle ss_spec_tproxy
nft list set inet ss_spec china
nft list set ip ss_spec_mangle whitelist
```

命令输出如包含敏感字段，只保留字段名、节点 section ID、别名和状态，不把服务端地址、UUID 或订阅内容写入文档。

## 修改约束

- SSR Plus 运行时使用 `gfw`、`router` 等模式必须以当前 UCI 值为准，不凭记忆覆盖用户选择。
- 动态 DNS/IP 分类不能做成持久 nft 快照；重启应根据当前配置和当前 DNS 结果重建。
- 白名单域名如果要绕过 UDP/443 丢弃，必须同时进入 `inet ss_spec` 和 `ip ss_spec_mangle`，并且 return 规则必须排在 China 判断和 drop 之前。
- 白名单 DNS 不要指向本地 dnsmasq 自身；使用当前 WAN 的明确上游 IPv4 DNS，并通过 dnsmasq `nftset` 绑定集合。
- 不为了“看起来完整”增加浏览器自动化、全局抓包、批量 tab 或无关依赖。
- `/etc/init.d/firewall reload` 在当前固件上已知可能触发 `nft` 管道段错误。SSR 规则验证优先使用 SSR 服务重启、`fw4 check` 和 nft 只读检查。

## 修改后的最低验证

- `sh -n` 检查所有修改的 shell 文件。
- `dnsmasq --test -C /var/etc/dnsmasq.conf.<id>` 返回 `syntax check OK`。
- 检查旧快照不存在，且短暂等待后没有被自动生成。
- 查询至少一个购物域名和一个视频网站域名，确认 IP 同时写入两张白名单表。
- 以正确方式读取代理计数：

```sh
nft list chain inet ss_spec ss_spec_wan_fw | \
  awk '/redirect to :1234/ { for (i = 1; i <= NF; i++) if ($i == "packets") print $(i + 1) }'
```

- 国内 URL 的计数增量应为 0；外部 URL 的计数应能增加。若只在路由器本机测试，必须在交付中注明 LAN 客户端和 IPv6 尚未完全覆盖。

## Handoff 格式

交付给下一 Agent 时，至少写明：

- 现场时间和时区。
- 当前节点 section、分流模式和代理进程数量。
- 修改过的绝对路径。
- 每个备份的绝对路径。
- 关键验证结果和未验证边界。
- 已知残留，例如当前固件 `fw4 reload` 的 nft 段错误。

不要只写“已修复”。要写清楚什么证据证明了哪一段路径。
