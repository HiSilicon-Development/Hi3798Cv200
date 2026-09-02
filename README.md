# HiSilicon Hi3798Cv200 6.12.y 适配项目 (歌华 1001/1002/1004/1006)

本仓库是 HiSilicon Hi3798Cv200 机顶盒 SoC 上的 Linux 6.12 实现范围，以及设备启动日志中可见的硬件现象

四款设备实际使用的 Fastboot、内核容器、AArch32 → AArch64 转交见 [Hi3798CV200 AArch64 启动架构](Hi3798Cv200-AArch64-Boot-Doc.md)

Hi3798Cv200 面向 DVB/IPTV 机顶盒，包含四核 64 位 Cortex-A53、Mali-T720 GPU、视频编解码、显示、DVB 传输流、千兆网络、USB、eMMC/TF 和智能卡等模块

企鹅群：[1121762879](https://qm.qq.com/q/6nYIaOoW6Q)

## 免责声明与风险提示

本项目（含内核源码、引导文档、硬件驱动及相关固件）仅供个人技术研究、学习与交流使用，不涉及任何商业盈利目的。所有固件（含二进制包、Modules模块）均不包含 TvHeadend、OSCam 等流媒体或条件接收（CA）解密软件。

本项目仅提供硬件底层的驱动适配与系统运行环境，不提供、不集成、不引导任何可能涉及版权争议或违反法律法规的第三方应用。严禁任何人将本项目提供的源码、固件或模块用于任何非法用途或灰色产业（包括但不限于商业盗播、非法解密、网络攻击等）。

任何使用者因滥用本项目内容所产生的一切法律后果及连带责任，均由使用者本人承担，与本项目作者无关。作者不对任何因使用本固件导致的直接或间接损失负责。

刷机涉及底层操作，有极小概率变砖。

本项目为开源实验性项目，任何操作风险由使用者自行承担。

下载、编译或使用本项目即代表您已阅读并同意本免责声明。

## 支持的设备

| 设备 | 状态 | 下载 |
| --- | --- | --- |
| [DVB-IP1001/2/4/6](DVBIP-100x-INTRODUCTION.md) | 已全部支持 | [firmware](https://github.com/HiSilicon-Development/firmware/releases) |

同一颗 Hi3798Cv200 可以搭配不同的 eMMC 模式、DVB 前端、供电参数、分区布局和安全启动容器

HiTool 刷机包必须与设备型号严格对应，不能跨设备混刷

对于某些新版HiTool可能会自动更新bootargs内容，要在HiTool首选项中关闭“启动自动更新bootargs中分区表信息”，没有此选项可忽略，否则会损坏刷入的bootargs镜像导致启动失败

## 公开参考资料

- [Hi3798C V200 Brief Data Sheet](https://www.silicondevice.com/file.upload/images/Gid1562Pdf_Hi3798CV200.pdf) 芯片功能框图和公开规格参考。
- [Linux 主线 Poplar DTS](https://github.com/torvalds/linux/blob/master/arch/arm64/boot/dts/hisilicon/hi3798cv200-poplar.dts) 公开开发板上的基础设备树参考。
- [U-Boot Poplar README](https://github.com/ARM-software/u-boot/blob/master/board/hisilicon/poplar/README) l-loader → TF-A → AArch64 U-Boot 的公开启动链示例。
- [Trusted Firmware-A](https://github.com/ARM-software/arm-trusted-firmware) Armv8-A EL3、安全监控和标准启动固件参考实现。
- [HiSTBLinux R005 SPC050](https://github.com/JasonFreeLab/HiSTBLinuxV100R005C00SPC050) / [SPC060](https://github.com/JasonFreeLab/HiSTBLinuxV100R005C00SPC060) 公开可检索的较晚 SDK 镜像，仅作资料对照。
