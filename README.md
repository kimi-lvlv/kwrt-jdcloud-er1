# 京东云太乙 ER1 稳定版 KWRT 固件

> **致力于打造最好用的太乙 ER1固件**

这是面向 **京东云太乙 RE-CS-07（ER1）** 的定制 KWRT 固件。基于 IPQ6018、
2 GB 内存和 8 GB eMMC 的实际硬件条件，聚焦稳定网络、常用服务和可恢复升级；
不把不必要的服务与高风险升级工具塞入默认系统。

- 当前固件标识：**ER1 Stable Firmware 1.0.2**
- 维护者：**Cao-Junqi**

## 上游与构建基础

- 固件基础：[KWRT 25.12.5](https://github.com/KwrtLab/kwrt)，目标为
  `qualcommax/ipq60xx`。
- 软件包：使用仓库锁定的 KWRT 公共软件源；每次构建先校验索引 SHA-256。
- 设备适配：仅面向 `JDCloud RE-CS-07` / ER1 的已验证兼容 U-Boot 和大分区布局。

## 适用设备

- 设备型号：京东云太乙 `JDC-RE-CS-07` / `RE-CS-07` / `ER1`
- SoC：Qualcomm IPQ6018，ARM64
- 已验证硬件：2 GB RAM、8 GB eMMC、USB 3.0
- 固件基础：KWRT 25.12.5，`qualcommax/ipq60xx`
- 默认管理地址：`10.0.0.1/24`

> [!IMPORTANT]
> 本项目当前面向已经完成兼容 U-Boot 和大分区布局的 ER1。已验证布局包含
> 12 MiB `0:HLOS` 和 1 GiB `rootfs`。项目不会更新 U-Boot，也不会修改 GPT、
> BOOTCONFIG、APPSBL、ART 或 CDT。原厂分区或分区情况不明的设备不要直接刷入。

## 固件亮点

| 方向 | 默认内容与优化 |
| --- | --- |
| 稳定网络 | 关闭软件/硬件流量分载、NSS ECM 与 NSS PPPoE 加速；保留网口必需的 NSS dataplane 和 SSDK，避免代理与加速路径相互干扰。 |
| 代理能力 | 预装 SSR Plus、Xray-core、OpenClash；SSR Plus 提供常用单节点协议支持，OpenClash 作为完整的 Clash 管理入口。 |
| 升级恢复 | 内置 `er1-upgrade`：升级前备份系统配置、SSR Plus/OpenClash/FRP/Adblock/主题数据和已装插件清单；重启后自动恢复仍在软件源中的插件。 |
| 容器服务 | 预装 Docker、Dockerd、Dockerman 与 TUN；容器数据固定在独立 eMMC 分区 `/mnt/mmcblk0p24/docker/`，系统升级不会删除该分区。 |
| 日常服务 | 预装 `adblock-fast`（默认 1Hosts Lite）、FRPC 与 LuCI 管理界面，默认不开放 FRP 穿透端口。 |
| 存储与界面 | USB 3.0/UAS、ext4、exFAT、NTFS3、VFAT；默认 Aurora，保留 Argon；简体中文和 `Asia/Shanghai` 默认配置。 |
| 可复现构建 | 锁定 ImageBuilder、软件源索引与外部包的 SHA-256；Actions 会校验 manifest、生成 SHA256SUMS 与构建信息。 |

### 真机验证

已在真实 ER1 上完成启动、网络、代理分流、Docker、升级与插件恢复路径验证。稳定运行天数必须以设备当前在线状态为准；本次 README 更新时构建机无法连通该设备，因此不写未经实时核实的天数。

## 主题入口

- Aurora 设置：`系统 -> Aurora Theme Design -> Design Studio`
- Argon 设置：`系统 -> Argon Config`
- 切换主题：`系统 -> 系统 -> 语言和界面`

## 下载与刷机

固件通过 GitHub Actions 手动构建。构建成功后会同时上传到 Actions Artifact 和对应
版本的 GitHub Release。

产物包含两种固件，不能混用：

- `*-squashfs-sysupgrade.bin`：在已经运行的兼容系统中升级，建议使用
  `sysupgrade -n`，不要保留旧固件配置。
- `*-squashfs-factory.bin`：通过现有 U-Boot 网页恢复界面刷入。

当前已验证的 U-Boot 版本不支持在网页中解析 `sysupgrade.bin`，因此 U-Boot 中只能
选择 `factory.bin`。刷机过程中不要断电，也不要上传任何 U-Boot、APPSBL、GPT、ART、
BOOTCONFIG 或 CDT 文件。

## 默认行为

- 主机名：`ER1`
- LAN：`10.0.0.1/24`
- DHCP 地址池：`10.0.0.100` 至 `10.0.0.249`
- DHCP 租期：12 小时
- WAN 不使用 PPPoE 下发的 DNS，固定使用阿里云 `223.5.5.5` 和腾讯 DNSPod
  `119.29.29.29`；SSR Plus 的代理 DNS 使用 `8.8.8.8`
- SSR Plus、FRPC 需要用户添加自己的节点或服务端信息
- Docker 默认启用，数据保存在 eMMC 的 `/mnt/mmcblk0p24/docker/`

## 构建方式

仓库只提供一个手动触发的 `Build ER1 firmware` 工作流，不在本地编译固件。工作流会：

1. 校验固定版本的 KWRT ImageBuilder、软件源索引和外部主题包。
2. 构建 ER1 的 factory 与 sysupgrade 镜像。
3. 检查必要软件包存在，并确认已排除不需要或存在稳定性风险的软件包。
4. 生成 `SHA256SUMS`、软件包 manifest 和 `BUILD-INFO.txt`。
5. 上传 Actions Artifact，并同步发布到对应版本的 GitHub Release。

设备基线、U-Boot 和分区探测记录见
[`docs/device-baseline.md`](docs/device-baseline.md)。

## 维护文档

- [`AGENTS.md`](AGENTS.md)：后续 Agent 的路由器操作边界、检查顺序和交付格式。
- [`HANDOFF.md`](HANDOFF.md)：本次 SSR Plus 修复的现场状态、证据、备份和回滚信息。
- [`docs/ssr-plus-routing-troubleshooting.md`](docs/ssr-plus-routing-troubleshooting.md)：SSR Plus 分流、DNS、TCP/UDP 集合和 nft 规则排查手册。

## 终端升级与插件恢复

固件内置 `er1-upgrade`，用于升级前的配置与插件数据保护。通过终端升级时使用：

```sh
er1-upgrade upgrade /tmp/xxx-squashfs-sysupgrade.bin
```

也可以直接提供固件下载 URL。该命令会先将系统配置，以及 SSR Plus、OpenClash
配置和自定义规则、FRP、Adblock、主题等插件数据与插件清单备份到
`/mnt/mmcblk0p24/er1-upgrade/`，再调用 `sysupgrade`。Docker 数据位于独立 eMMC
分区，不会随系统升级删除。

新系统启动后会自动从当前软件源补装缺失的 LuCI 插件和代理内核。自动恢复失败时，
可以运行：

```sh
er1-upgrade status
er1-upgrade restore
```

恢复过程不会把旧内核模块或整套旧系统二进制覆盖到新固件。只有仍存在于已配置软件源
中的插件可以自动重装；手工下载且没有软件源的第三方 IPK 需要重新提供安装包。

## 项目边界

本项目只负责系统固件及默认配置，不包含代理节点、FRP 密钥、Cloudflare Tunnel 凭据
或其他私人配置。固件不会自动写入或升级引导程序，也不会自动调整磁盘分区。
