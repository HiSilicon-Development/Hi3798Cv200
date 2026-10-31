# HiSilicon Hi3798CV200 Linux 6.12.y 适配项目（歌华 DVB-IP1001 / 1002 / 1004 / 1006）

本仓库记录基于海思 **Hi3798CV200** 机顶盒 SoC 的 **Linux 6.12.y（当前基线 `6.12.114` AArch64）** 硬件适配架构、板级拓扑、64 位引导方案及外围扩展能力。

Hi3798CV200 面向混合有线数字电视（DVB-C）与网络媒体终端，片内集成四核 64 位 Cortex-A53（最高 1.6 GHz + 软件 AVS）、Mali-T720 GPU（`Panfrost`）、HiVXE 视频/图像硬件编解码引擎、VDP 多平面显示控制器与 HDMI 2.0（最高 4K60 / CEC / HDCP）、四路 TSI 传输流解复用与硬件 CSA2 解扰器、千兆以太网、USB 2.0/3.0、eMMC/TF、AIAO/Tianlai 立体声音频、红外接收及 ISO/IEC 7816 智能卡接口。

交流企鹅群：[1121762879](https://qm.qq.com/q/6nYIaOoW6Q)

---

## 文档导航与相关仓库

- **[DVB-IP100x 产品与硬件总览](DVBIP-100x-INTRODUCTION.md)**：1001 / 1002 / 1004（2+8）/ 1006 型号差异矩阵、设备树与节点树、内存保留区、DVFS/GPU/显示/音频实现及 HiTool 分区表。
- **[Hi3798CV200 AArch32 → AArch64 启动架构](Hi3798Cv200-AArch64-Boot-Doc.md)**：原厂 AArch32 Fastboot 容器封装、ARM32 热复位跳板、EL3 → EL2 安全移交、四核 `spin-table` 唤醒与串口排障指南。
- **[HiVXE 硬件编解码与 Tvheadend 配置](https://github.com/HiSilicon-Development/HiVXE-2.0)**：VDEC / VPSS / VENC / JPGD / JPGE 硬件能力矩阵及 `hivxe-ffmpeg` 转码参数说明。
- **[固件与内核更新仓库 (firmware)](https://github.com/HiSilicon-Development/firmware/releases)**：四款机型完整 HiTool 刷机包与在线内核升级包下载。
- **[扩展工具仓库 (extra)](https://github.com/HiSilicon-Development/extra)**：适用于 Debian 12 / 13 ARM64 的 `hivxe-ffmpeg`、`hivxe-top`、`hivxe-transcode` 等扩展安装脚本。

---

## 支持的板载机型

| 设备型号 | 板载前端与核心外设状态 | 固件下载 |
| --- | --- | --- |
| **[DVB-IP1001](DVBIP-100x-INTRODUCTION.md#dvb-ip1001)** | 已全部支持（板载四路 DVB-C、千兆网口、GPU/HDMI、HiVXE、音频、SCI0、IR、USB） | [Releases](https://github.com/HiSilicon-Development/firmware/releases) |
| **[DVB-IP1002](DVBIP-100x-INTRODUCTION.md#dvb-ip1002)** | 已全部支持（同上 + HS400 eMMC、TF 卡槽、板载 RTL8192EU Wi-Fi） | [Releases](https://github.com/HiSilicon-Development/firmware/releases) |
| **[DVB-IP1004（2+8）](DVBIP-100x-INTRODUCTION.md#dvb-ip1004)** | 已全部支持（同上 + 三线独立时钟 DVB-C、HS400 eMMC、TF 卡槽、板载 MT7662T Wi-Fi/BT；仅限 2+8 版） | [Releases](https://github.com/HiSilicon-Development/firmware/releases) |
| **[DVB-IP1006](DVBIP-100x-INTRODUCTION.md#dvb-ip1006)** | 已全部支持（同上 + HS400 eMMC、高有效 SCI0 供电/检卡、RGMII strap 延时） | [Releases](https://github.com/HiSilicon-Development/firmware/releases) |

### 刷机必读注意事项

1. **严禁跨型号混刷**：同一颗 Hi3798CV200 在不同机型上搭配了不同的 eMMC 时序、DVB-C 复位/I2C 引脚、以太网延时、分区表与原厂安全启动容器，刷机包及内核升级包必须与机型严格对应。
2. **关闭 HiTool 自动改写 bootargs**：部分新版 HiTool 默认开启“启动自动更新 bootargs 中分区表信息”，刷机前**必须在 HiTool 首选项中取消勾选该项**（旧版 HiTool 无此选项可忽略），否则会破坏刷入的 `bootargs` 分区并导致无法挂载系统。

---

## 外围 USB DTMB / DVB-C 接收器支持（非板载硬件）

> **重要硬件说明**：歌华 **DVB-IP1001 / 1002 / 1004 / 1006 机身板载射频硬件均为四路有线数字电视（DVB-C，MaxLinear MxL214 / MxL214C），不具备板载地面波（DTMB）硬件**。内核内置的 DTMB 驱动与随附固件**仅作为外接 USB 数字电视棒等外围设备的扩展支持**，插入机顶盒 USB 端口后即可注册额外的 `/dev/dvb/adapterN` 前端节点。

当前 Linux 6.12.y 内核（`hi3798cv200_defconfig`）已模块化集成以下五类常见国产 USB 2.0 地面波（DTMB）/ 有线（DVB-C）接收器驱动，并在 Kconfig 层面对共享 USB VID:PID 的冲突上游驱动设置了互斥保护，完整配套固件均已内置于系统 `/lib/firmware/` 目录：

| 外置 USB 接收器类型 | USB VID:PID | 桥接 / 解调 / 调谐芯片方案 | 内核模块（`.ko`） | `/lib/firmware/` 配套固件 | 支持制式与工作说明 |
| --- | --- | --- | --- | --- | --- |
| **华大 HDIC HD2312 方案**（乐视、爱华 itv720、CVB T302A 等） | `04b4:1004` | Cypress FX2 + HD2312 | `dvb-usb-hd2312.ko` | 无需外部固件（使用棒载 EEPROM 固件） | **DTMB**（向用户态兼容暴露 `SYS_DVBT`，自动识别固件版本并完成 T302A 握手） |
| **CH1 LME2510C 方案** | 冷启动：`3344:1111`<br>运行态：`3344:1120`、`3344:1122`、`3344:22d0`、`3344:22d8`、`3344:e388`、`1b80:e388` | LME2510C + LGS8GL5 + MAX2165 | `dvb-usb-lme2510-dtmb.ko` | `dvb-usb-lme2510c-dtmb-win-rom.bin`<br>`dvb-usb-lme2510c-dtmb-win-fw.bin`<br>`dvb-usb-lme2510c-dtmb-win-fw-pid0xd0.bin` | **DTMB**（同时暴露 `SYS_DTMB` 与 `SYS_DVBT`；与 DVB-S 版 `dvb-usb-lme2510` 互斥） |
| **爱国者 Aigo DT5236** | `eb1a:2885` | eMPIA EM2884 + LGS8GL5 + XC5000 | `dvb-usb-em28xx-dtmb.ko` | `dvb-fe-xc5000-1.6.114.fw` | **DTMB**（暴露 `SYS_DTMB` 与 `SYS_DVBT`；与通用 `em28xx` 模拟采集驱动互斥） |
| **润祺达 AVL6381 方案** | `04b4:9306`<br>`048d:9306` | ITE IT9303 + Availink AVL6381 + MxL603 | `dvb-usb-avl6381.ko`<br>`avl6381.ko`<br>`mxl603.ko` | `dvb-usb-it9303-01.fw`（AVL6381 解调微码已内嵌驱动） | **DTMB + DVB-C Annex A**（与通用 `dvb-usb-af9035` 的 `048d:9306` 绑定互斥） |
| **国微 SMIT iCast** | `29df:0001` | SMIT CI/SAS USB 桥接控制器 | `dvb-usb-icast.ko` | 无需外部固件（棒载主控固件） | 挂载时读取硬件 `GDHW` 能力字：`DTMB:Y` 注册 **DTMB + DVB-C Annex A**，`DTMB:N` 仅注册 **DVB-C Annex A** |

---

## 免责声明与风险提示

本项目（含内核源码、引导文档、硬件驱动及相关固件）仅供个人技术研究、学习与交流使用，不涉及任何商业盈利目的。所有固件（含二进制包、Modules 模块）均不包含 Tvheadend、OSCam 等流媒体或条件接收（CA）解密软件。

本项目仅提供硬件底层的驱动适配与系统运行环境，不提供、不集成、不引导任何可能涉及版权争议或违反法律法规的第三方应用。严禁任何人将本项目提供的源码、固件或模块用于任何非法用途或灰色产业（包括但不限于商业盗播、非法解密、网络攻击等）。

任何使用者因滥用本项目内容所产生的一切法律后果及连带责任，均由使用者本人承担，与本项目作者无关。作者不对任何因使用本固件导致的直接或间接损失负责。

刷机涉及底层操作，有极小概率变砖。本项目为开源实验性项目，任何操作风险由使用者自行承担。下载、编译或使用本项目即代表您已阅读并同意本免责声明。

---

## 公开参考资料

- [Hi3798C V200 Brief Data Sheet](https://www.silicondevice.com/file.upload/images/Gid1562Pdf_Hi3798CV200.pdf)：芯片功能框图与公开硬件规格参考。
- [Linux 主线 Poplar DTS](https://github.com/torvalds/linux/blob/master/arch/arm64/boot/dts/hisilicon/hi3798cv200-poplar.dts)：公开开发板基础设备树参考。
- [U-Boot Poplar README](https://github.com/ARM-software/u-boot/blob/master/board/hisilicon/poplar/README)：`l-loader → TF-A → AArch64 U-Boot` 公开启动链示例。
- [Trusted Firmware-A](https://github.com/ARM-software/arm-trusted-firmware)：Armv8-A EL3 安全监控与标准启动固件参考实现。
- [HiSTBLinux R005 SPC050](https://github.com/JasonFreeLab/HiSTBLinuxV100R005C00SPC050) / [SPC060](https://github.com/JasonFreeLab/HiSTBLinuxV100R005C00SPC060)：公开可检索的 SDK 源码镜像，仅作寄存器与协议资料对照。
