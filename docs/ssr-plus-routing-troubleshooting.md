# SSR Plus 分流规则问题排查

## 适用现象

这份手册针对以下路由器侧现象：

- SSR Plus 开启后，京东、淘宝、拼多多、B 站、爱奇艺等国内站点变慢。
- 网站出现境外定位、地区提示或 CDN 选路异常。
- 关闭 SSR Plus 后恢复，重新开启又出现。
- 外部网站能访问，但国内视频或购物站的首屏、图片、播放器卡顿。

排查范围是 ER1 路由器，不包含本机代理、浏览器代理和本机 DNS 设置。

## 数据路径

当前 SSR Plus nft 路径大致分成四层：

1. dnsmasq 根据域名把查询送到 WAN DNS、代理 DNS 或白名单 DNS。
2. dnsmasq 的 `nftset` 把解析得到的 IPv4 写入 `china`、`gfwlist`、`whitelist` 等集合。
3. TCP 流量经过 `inet ss_spec`，主要链为 `ss_spec_wan_ac` -> `ss_spec_wan_fw`。
4. UDP/QUIC 流量经过 `ip ss_spec_mangle`，主要链为 `ss_spec_tproxy`；不能代理的 UDP/80、443 可能被丢弃。

因此，“DNS 能解析”并不代表“TCP/UDP 最终走了正确出口”。必须同时看集合成员和 nft 计数器。

## 第一轮只读检查

```sh
uci -q get shadowsocksr.@global[0].global_server
uci -q get shadowsocksr.@global[0].run_mode
uci -q get shadowsocksr.@global[0].pdnsd_enable
uci -q get shadowsocksr.@global[0].tunnel_forward

ps w | grep -E '[v]2ray|[x]ray|[m]ihomo|[d]nsmasq|[d]ns2tcp|[c]hinadns'

nft list tables
nft list chain inet ss_spec ss_spec_wan_ac
nft list chain inet ss_spec ss_spec_wan_fw
nft list chain ip ss_spec_mangle ss_spec_tproxy
nft list set inet ss_spec china
nft list set inet ss_spec gfwlist
nft list set inet ss_spec whitelist
nft list set ip ss_spec_mangle whitelist
```

同时检查是否有旧快照：

```sh
find /usr/share/nftables.d /etc/ssrplus /tmp -type f \
  \( -name '99-shadowsocksr.nft' -o -name '*shadowsocksr*save*' \
     -o -name 'dnsmasq-ssrplus.conf' \) -print
```

## 关键根因一：动态 nft 快照污染

风险文件通常是：

- `/usr/share/nftables.d/ruleset-post/99-shadowsocksr.nft`
- `/etc/ssrplus/ssrplus-persist/99-shadowsocksr.save`

这些文件可能包含当时解析得到的 IP，而不是稳定的域名策略。CDN IP 会变化，也可能被多个国内外服务共享。重启或 firewall include 恢复旧集合后，国内站点会被误判。

判断方法：

```sh
ls -l /usr/share/nftables.d/ruleset-post/99-shadowsocksr.nft \
      /etc/ssrplus/ssrplus-persist/99-shadowsocksr.save 2>/dev/null
logread | grep -iE 'persist|restore|NFTables rules persisted|restored from persistence'
```

修复要求：

- SSR 服务每次按当前配置重建动态集合。
- `ssr-rules` 不保存、恢复或通过 firewall include 加载旧快照。
- 重启后再次检查快照，不能只在刚删除时检查。

## 关键根因二：UDP 白名单没有真正生效

典型错误顺序如下：

```text
@china return
udp dport 443 drop
```

如果某个国内 CDN IP 不在 `china` 集合，QUIC 会被丢弃。浏览器通常回退到 TCP，用户看到的就是视频、图片或购物页面加载慢。

正确顺序应为：

```text
@whitelist return
@china return
udp dport 80 drop
udp dport 443 drop
```

检查：

```sh
nft -a list chain ip ss_spec_mangle ss_spec_tproxy | \
  grep -E '@whitelist|@china|udp dport (80|443)'
```

`@whitelist return` 必须出现在两个 drop 规则之前。只创建 `whitelist` 集合、但没有引用它，等同于没有修复。

## 关键根因三：白名单只写入 TCP 表

如果 dnsmasq 生成配置只有：

```text
nftset=/example.cn/inet#ss_spec#whitelist
```

而没有写入 `ip ss_spec_mangle`，TCP 可能直连，UDP/QUIC 仍会被丢弃。当前 dnsmasq 2.93 适合使用一条带地址族前缀的合并配置：

```text
nftset=/example.cn/4#inet#ss_spec#whitelist,4#ip#ss_spec_mangle#whitelist
```

验证方式：

```sh
nslookup www.jd.com 127.0.0.1
nft list set inet ss_spec whitelist
nft list set ip ss_spec_mangle whitelist
```

同一解析到的 IPv4 应同时出现在两张表中。若只在第一张表出现，检查 `gfw2ipset.sh` 的 `nftset` 生成格式。

## 关键根因四：白名单 DNS 回环

下面这种生成方式有风险：

```text
server=/example.cn/127.0.0.1
```

路由器 dnsmasq 本身监听 `127.0.0.1:53` 时，这条规则可能把白名单查询重新送回自身，造成等待、重复查询或不可预测的上游选择。应使用明确的 WAN IPv4 DNS，例如当前 `/tmp/resolv.conf.d/resolv.conf.auto` 中的第一个 nameserver：

```text
server=/example.cn/223.5.5.5
```

不要在文档或日志中写入用户私有 DNS 凭据；这里只涉及公开 DNS 地址。

## 关键根因五：China IP 集合不完整

查询某个实际目的 IP：

```sh
nft get element inet ss_spec china '{ 27.36.125.193 }'
nft get element ip ss_spec_mangle china '{ 27.36.125.193 }'
```

返回成功表示命中对应集合。若不命中，先确认：

- `/etc/ssrplus/china_ssr.txt` 是否存在、是否为有效 CIDR。
- `nft list set ... china` 是否真的有元素，而不是空集合。
- 日志是否有 `chinaipset: invalid SKIP_INET=...`。
- 当前 IP 是否属于国际 CDN，而不是简单把所有“看起来像国内域名”的 IP 强行归入 China。

稳定的域名策略应使用白名单补充，而不是无限扩大 China IP 文件。

## 关键根因六：IPv4 与 IPv6 不一致

当前 SSR Plus 规则主体是 IPv4 `ip daddr` 规则；DNS 可能同时返回 AAAA。检查：

```sh
nslookup www.taobao.com 127.0.0.1
nft list ruleset | grep -E 'ip6|ip daddr|ip6 daddr'
uci show dhcp | grep -E 'filter_aaaa|ra|dhcpv6'
```

如果 IPv6 连接没有对应的分流规则，必须把它作为独立路径验证。不要看到 IPv4 命中就宣称 IPv6 也已修好。

## 路由计数器验证

先记录 TCP 代理链计数：

```sh
proxy_packets() {
  nft list chain inet ss_spec ss_spec_wan_fw 2>/dev/null | \
    awk '/redirect to :1234/ {
      for (i = 1; i <= NF; i++) if ($i == "packets") print $(i + 1)
    }'
}
```

然后对一个国内 URL 和一个外部 URL 各请求一次：

```sh
before=$(proxy_packets)
curl -4 -L -sS --max-time 15 -o /dev/null \
  -w 'code=%{http_code} ip=%{remote_ip} total=%{time_total}s\n' \
  https://www.jd.com
after=$(proxy_packets)
echo "domestic_proxy_delta=$((after-before))"

before=$(proxy_packets)
curl -4 -L -sS --max-time 15 -o /dev/null \
  -w 'code=%{http_code} ip=%{remote_ip} total=%{time_total}s\n' \
  https://www.google.com
after=$(proxy_packets)
echo "external_proxy_delta=$((after-before))"
```

预期：国内站 `domestic_proxy_delta=0`，外部站在 gfw 模式下可以增加代理计数。这里的请求来自路由器本机，不能替代所有 LAN 设备和 IPv6 的端到端验证。

## 当前 ER1 的已验证结果

2026-08-15 22:32，SSR Plus 保持运行、`run_mode=gfw` 时：

| 目标 | HTTP | 总耗时 | 代理计数增量 |
| --- | ---: | ---: | ---: |
| 京东 | 200 | 0.346 s | 0 |
| 淘宝 | 200 | 0.173 s | 0 |
| 拼多多 | 200 | 0.653 s | 0 |
| B 站 | 200 | 0.323 s | 0 |
| 爱奇艺 | 200 | 0.154 s | 0 |
| Google | 200 | 4.727 s | 3 |

同次验证还确认：dnsmasq 语法通过、UDP 白名单有 61 个 IPv4 元素、旧 nft 快照重启后不存在。

## 防火墙 reload 注意事项

本固件上的 `/etc/init.d/firewall reload` 曾返回 `1`，详细错误为 `/sbin/fw4` 管道调用 `nft` 时段错误；`fw4 check` 对渲染文件返回 0。这个问题独立于 SSR 分流修复。

排查 SSR 时优先使用：

```sh
/etc/init.d/shadowsocksr restart
fw4 check
nft list ruleset
```

不要为了制造“reload 成功”而改动固件 nft 二进制或关闭校验。

## 回滚

回滚前先保留当前版本。聚焦修复前的代码备份为 `20260815-222849`：

```sh
cp -f /usr/bin/ssr-rules.bak-20260815-222849 /usr/bin/ssr-rules
cp -f /usr/share/shadowsocksr/gfw2ipset.sh.bak-20260815-222849 /usr/share/shadowsocksr/gfw2ipset.sh
cp -f /etc/init.d/shadowsocksr.bak-20260815-222849 /etc/init.d/shadowsocksr
cp -f /etc/ssrplus/white.list.bak-20260815-222849 /etc/ssrplus/white.list
chmod 755 /usr/bin/ssr-rules /usr/share/shadowsocksr/gfw2ipset.sh /etc/init.d/shadowsocksr
rm -f /usr/share/nftables.d/ruleset-post/99-shadowsocksr.nft
/etc/init.d/shadowsocksr restart
```

回滚后仍要重新检查 `global_server`、`run_mode`、nft 集合和实际 URL；恢复文件不等于恢复行为。
