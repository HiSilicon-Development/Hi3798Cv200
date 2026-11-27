# HiSilicon Hi3798CV200 Linux 6.12.y 适配项目

本项目为 **海思 Hi3798CV200** 机顶盒 SoC 的 **AArch64 Linux 6.12.y** 开源移植工程，当前面向 **歌华 DVB-IP1001 / 1002 / 1004 / 1006** 四款机型，完整适配 Debian 12 / 13 ARM64 系统。

- **最新内核更新包**：[Linux 6.12.118 Release](https://github.com/HiSilicon-Development/firmware/releases/tag/kernel)
- **交流企鹅群**：[1121762879](https://qm.qq.com/q/6nYIaOoW6Q)

---

## 芯片与平台概览

Hi3798CV200 是海思面向 4K 超高清机顶盒的 64 位 SoC。本项目在保留原厂高安 Fastboot 引导的前提下，通过自研 AArch32 → AArch64 启动链运行标准 64 位 Linux 内核，脱离原厂闭源 MSP 中间件，全面接入 Linux 开源子系统：

| 模块 | 硬件规格与 Linux 6.12.y 实现 |
| --- | --- |
| **CPU** | 四核 Arm Cortex-A53（最高 1.6 GHz），支持 DVFS、软件 AVS/HPM 闭环调压与温控降频 |
| **GPU** | Arm Mali-T720，采用 Linux 主线开源 `Panfrost` 驱动与 Mesa 3D 渲染栈 |
| **显示与 HDMI** | 片内 VDP（`histb-vdp` 图形层 + 硬件视频叠加层）与 HDMI 2.0（最高 4K60、CEC、HDCP） |
| **硬件媒体加速** | HiVXE 2.0（标准 V4L2 Request / M2M）：视频硬解、H.264 硬编、JPEG/PNG 硬解、JPEG 硬编及四场硬件去隔行（VPSS/DEI） |
| **网络与存储** | 千兆以太网（GMAC1 + RTL8211E/F）、8-bit eMMC、TF 卡槽（部分机型）、USB 2.0 / 3.0 |
| **广播与外设** | 板载多路 DVB-C（MxL214 系列 / Hi3130v200+R858）与硬件解复用、模拟/HDMI/S/PDIF 音频、ISO 7816 智能卡（`SCI0`）、红外接收 |

---

## 文档导航与相关仓库

| 文档 / 仓库 | 内容说明 |
| --- | --- |
| [硬件总览（DVBIP-100x-INTRODUCTION.md）](DVBIP-100x-INTRODUCTION.md) | 公共子系统架构、四款机型硬件差异矩阵、设备树节点树、HiTool 分区布局与固件清单 |
| [启动架构（Hi3798Cv200-AArch64-Boot-Doc.md）](Hi3798Cv200-AArch64-Boot-Doc.md) | 原厂 Fastboot 槽位、AArch32 → AArch64 引导链、PSCI/CVPM 与 TTL 串口排障指南 |
| [HiVXE 2.0 媒体文档](https://github.com/HiSilicon-Development/HiVXE-2.0) | Hi3798CV200 / Hi3798MV310 / GK6323V100C 硬件编解码矩阵、FFmpeg 9.1 与 Tvheadend 转码配置 |
| [编译教程（manifest）](https://github.com/HiSilicon-Development/manifest) | 源码同步清单与 [build 编译脚本](https://github.com/HiSilicon-Development/build)，用于自行构建内核与 FFmpeg |
| [扩展工具（extra）](https://github.com/HiSilicon-Development/extra) | HiVXE 转码脚本、FFmpeg 及常用配套组件的安装说明 |
| [国科微 GK6323V100C 仓库](https://github.com/HiSilicon-Development/GK6323V100C) | 同系列开源内核适配的国科微 GK6323V100C SoC 与设备说明 |

---

## 支持的机型与版本选择

选购或刷机前请核对机身型号；**DVB-IP1004 存在两种不同的板载射频硬件**，请按拆机看到的射频芯片选择对应更新包和刷机包：

| 机型 | 板载 DVB-C 前端 | 6.12.118 更新包前缀 | 休眠与遥控器电源键行为 |
| --- | --- | --- | --- |
| [DVB-IP1001](DVBIP-100x-INTRODUCTION.md#dvb-ip1001) | 四路 MxL214 | `update-1001-` | 短按请求深睡，长按 5 秒关机 |
| [DVB-IP1002](DVBIP-100x-INTRODUCTION.md#dvb-ip1002) | 四路 MxL214C | `update-1002-` | 短按请求深睡，长按 5 秒关机 |
| [DVB-IP1004（MxL214 版）](DVBIP-100x-INTRODUCTION.md#dvb-ip1004) | 四路 MxL214 | `update-1004-mxl214-` | 支持深睡与红外唤醒，长按 5 秒关机 |
| [DVB-IP1004（R858 版）](DVBIP-100x-INTRODUCTION.md#dvb-ip1004) | 双路 Hi3130v200 + R858 | `update-1004-r858-` | 禁用系统休眠；短按无动作，长按 5 秒关机，红外开机执行冷启动 |
| [DVB-IP1006](DVBIP-100x-INTRODUCTION.md#dvb-ip1006) | 四路 MxL214C | `update-1006-` | 支持深睡与红外唤醒，长按 5 秒关机 |

> 详细的各机型存储、网口 PHY、无线网卡与设备树节点差异，请查看 [四款机型硬件差异矩阵](DVBIP-100x-INTRODUCTION.md#四款机型硬件差异矩阵)。

---

## 安装与更新指南

### 1. 已安装 DVB-IP Linux：在线升级内核

1. 从 [Release 页面](https://github.com/HiSilicon-Development/firmware/releases/tag/kernel) 下载与本机型号（及 1004 射频版本）严格匹配的更新包，并核对 `SHA256SUMS`。
2. 解压后先执行预检：`sudo sh update.sh --check`。
3. 预检通过后执行安装：`sudo sh update.sh`，完成后重启生效。

### 2. 首次安装或救砖：HiTool 全量刷机

1. 选择对应机型与射频版本的 `full.xml`、`bootargs`、内核容器与 `rootfs.img`。
2. **关键设置**：在 HiTool 首选项中**关闭“启动自动更新 bootargs 中分区表信息”**，避免刷机工具破坏分区表导致无法挂载根文件系统。
3. 保持 `deviceinfo` / `hwconfig` 分区不勾选，以保留本机原厂 MAC 地址与序列号。

---

## 免责声明与风险提示

本项目（含内核源码、引导文档、硬件驱动及相关固件）仅供个人技术研究、学习与交流使用，不涉及任何商业盈利目的。所有固件与模块均不包含 Tvheadend、OSCam 等流媒体或条件接收（CA）解密软件。

严禁将本项目源码、固件或模块用于任何非法用途或灰色产业。刷机涉及底层存储操作，存在一定变砖风险，下载、编译或使用本项目即代表您已阅读并同意自行承担全部操作与法律责任。

---

## 公开参考资料

- [Hi3798C V200 Brief Data Sheet](https://www.silicondevice.com/file.upload/images/Gid1562Pdf_Hi3798CV200.pdf)：芯片功能框图与公开硬件规格参考。
- [Linux 主线 Poplar DTS](https://github.com/torvalds/linux/blob/master/arch/arm64/boot/dts/hisilicon/hi3798cv200-poplar.dts)：公开开发板基础设备树参考。
- [U-Boot Poplar README](https://github.com/ARM-software/u-boot/blob/master/board/hisilicon/poplar/README)：`l-loader → TF-A → AArch64 U-Boot` 公开启动链示例。
- [Trusted Firmware-A](https://github.com/ARM-software/arm-trusted-firmware)：Armv8-A EL3 安全监控与标准启动固件参考实现。
- [HiSTBLinux R005 SPC050](https://github.com/JasonFreeLab/HiSTBLinuxV100R005C00SPC050) / [SPC060](https://github.com/JasonFreeLab/HiSTBLinuxV100R005C00SPC060)：公开 BSP 寄存器与驱动参考源码。
