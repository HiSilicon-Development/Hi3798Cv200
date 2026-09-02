# HiSilicon Hi3798Cv200 Linux 手册

本仓库是 HiSilicon Hi3798Cv200 机顶盒 SoC 上的 Linux 6.12 实现范围，以及设备启动日志中可见的硬件现象。

启动容器、Fastboot 类型和 AArch32 → AArch64 状态转交的技术说明见 [Hi3798CV200 AArch64 启动架构](Hi3798Cv200-AArch64-Boot-Doc.md) 文档。

Hi3798Cv200 面向 DVB/IPTV 机顶盒，包含四核 64 位 Cortex-A53、Mali-T720 GPU、视频编解码、显示、DVB 传输流、千兆网络、USB、eMMC/TF 和智能卡等模块。

## 支持的设备

| 设备 | 状态 | 下载 |
| --- | --- | --- |
| [DVB-IP1001](devices/DVB-IP1001.md) | 已支持 |  |
| [DVB-IP1002](devices/DVB-IP1002.md) | 已支持 |  |
| [DVB-IP1004](devices/DVB-IP1004.md) | 已支持 |  |

同一颗 Hi3798Cv200 可以搭配不同的 eMMC 模式、DVB 前端、供电参数、分区布局和安全启动容器。

HiTool 刷机包必须与设备型号严格对应，不能跨设备混刷。

## 公开参考资料

- [Hi3798C V200 Brief Data Sheet](https://www.silicondevice.com/file.upload/images/Gid1562Pdf_Hi3798CV200.pdf) 芯片功能框图和公开规格参考。
- [Linux 主线 Poplar DTS](https://github.com/torvalds/linux/blob/master/arch/arm64/boot/dts/hisilicon/hi3798cv200-poplar.dts) 公开开发板上的基础设备树参考。
- [U-Boot Poplar README](https://github.com/ARM-software/u-boot/blob/master/board/hisilicon/poplar/README) l-loader → TF-A → AArch64 U-Boot 的公开启动链示例。
- [Trusted Firmware-A](https://github.com/ARM-software/arm-trusted-firmware) Armv8-A EL3、安全监控和标准启动固件参考实现。
- [HiSTBLinux R005 SPC050](https://github.com/JasonFreeLab/HiSTBLinuxV100R005C00SPC050) / [SPC060](https://github.com/JasonFreeLab/HiSTBLinuxV100R005C00SPC060) 公开可检索的较晚 SDK 镜像，仅作资料对照。
