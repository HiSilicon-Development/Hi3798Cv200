# DVB-IP100x 产品与硬件总览

本文档统一记录歌华 **DVB-IP1001、DVB-IP1002、DVB-IP1004 与 DVB-IP1006** 四款机型的硬件参数、设备树与驱动映射、内存保留区、板级总线差异、HiTool 分区布局及软件交付规范。

- [公共平台规格](#公共平台规格)
- [四款机型硬件差异矩阵](#四款机型硬件差异矩阵)
- [设备树、驱动与用户态节点](#设备树驱动与用户态节点)
- [内存映射与媒体保留区](#内存映射与媒体保留区)
- [公共子系统架构实现](#公共子系统架构实现)
- [分型号板级链路与节点树](#分型号板级链路与节点树)
  - [DVB-IP1001](#dvb-ip1001)
  - [DVB-IP1002](#dvb-ip1002)
  - [DVB-IP1004](#dvb-ip1004)
  - [DVB-IP1006](#dvb-ip1006)
- [HiTool 物理布局](#hitool-物理布局)
- [固件、文件系统与软件交付](#固件文件系统与软件交付)

AArch32 Fastboot 到 AArch64 Linux 的启动链与容器结构见 [Hi3798CV200 AArch32 → AArch64 启动架构](Hi3798Cv200-AArch64-Boot-Doc.md)；项目入口与刷机更新指南见 [README](README.md)。

---

## 公共平台规格

系统版本：Linux 6.12.118、Debian 12／13 AArch64。

| 模块 | 公共规格与实现 |
| --- | --- |
| SoC | HiSilicon Hi3798CV200（高安 CA 版） |
| CPU | 4 × Arm Cortex-A53 r0p4（ARMv8-A / AArch64），支持 400 / 600 / 800 / 1200 / 1600 MHz DVFS 与软件 AVS/HPM |
| GPU | Arm Mali-T720 MP（`Panfrost` 开源驱动，200–675 MHz 六档 OPP，按 SVB 工艺分档调压） |
| 内存 | 2 GiB DDR（`0x00000000..0x7fffffff`） |
| 显示与 HDMI | 片内 VDP（`histb-vdp`，含图形层 `G0` 与硬件视频层 `V0/V1`、ZME 缩放、BT.601/709/2020 CSC）+ HDMI 2.0 TX/PHY（`histb-hdmi`，最高 4K60、SCDC、HPD 轮询、`/dev/cec0`、HDCP 1.4/2.2） |
| 硬件编解码 | HiVXE 媒体子系统：`VDEC`（VDH V4R3）、`VPSS`（去块/缩放/四场硬件 DEI）、`VENC`（H.264 VEDU）、`JPGD`（JPEG 硬解）、`JPGE`（JPEG 硬编） |
| 板载射频与解复用 | DVB-C Annex A：1001／1002／1006 和 1004 四路版使用 MxL214 系列；1004 双路版使用 Hi3130v200 + R858。TS 接入 `histb-dmx`，提供 32 槽 CSA2 |
| 音频 | AIAO 声卡链路：Tianlai 模拟立体声、HDMI I2S、S/PDIF |
| 智能卡 | 片内 ISO/IEC 7816 接口 `SCI0`（`hi3798cv200-sci`，`/dev/ttySCI0`） |
| 红外与串口 | 红外接收器 `hix5hd2-ir`（`/dev/lirc0`）；`UART0`（`ttyAMA0` 115200 8N1 控制台）+ `UART2`（`ttyAMA2` 辅助串口） |
| USB 主控 | 1 × OHCI（`usb@9880000`）、1 × EHCI（`usb@9890000`）、2 × xHCI USB 3.0（`usb@98a0000`、`usb@98b0000`） |
| 内核与系统 | Linux `6.12.118` 更新包（`hi3798cv200_defconfig` + `dvbip_100{1,2,4,6}_defconfig`），Debian 12 / 13 ARM64 |

---

## 四款机型硬件差异矩阵

按下表选择 DTB。1004 根据板载 DVB-C 芯片选择变体。

| 型号 | eMMC 与 TF | 板载 DVB-C 与 TS 总线 | 千兆以太网（GMAC1 / MDIO 3） | 声卡、SCI0 与板载无线 | DTB |
| --- | --- | --- | --- | --- | --- |
| **DVB-IP1001** | 8 GB（8GTF4R），8-bit DDR52 @ 52 MHz（`MUX_M2`）；`sd0` 禁用 | MxL214，`i2c1@8b11000` / `0x50`，GPIO7_4 低有效复位；`serial-nosync`，共用 `TSI2_CLK`，四路反相，16-block DMA 队列 | RTL8211E，`rgmii-id` | `DVBIP-1001`，模拟 TX0、HDMI TX3；SCI0 低有效供电和检卡；外接 USB 无线 | `hi3798cv200-dvbip-1001-arm64.dtb` |
| **DVB-IP1002** | 8 GB（DG4008），8-bit HS400 @ 150 MHz；启用 `sd0@9820000` | MxL214C，`i2c1@8b11000` / `0x50`，GPIO7_4 低有效复位；`serial-nosync`，共用 `TSI2_CLK`，四路反相，32-block 队列和硬件 PID 过滤 | RTL8211F，`rgmii-id`，`CONFIG_HISTB_RTL8211F_LED` | `DVBIP-1002`，模拟 TX0、HDMI TX3；SCI0 低有效供电、高有效检卡；RTL8192EU（`0bda:818b`） | `hi3798cv200-dvbip-1002-arm64.dtb` |
| **DVB-IP1004：MxL214** | 8-bit HS400 @ 150 MHz；启用 `sd0@9820000` | 四路 MxL214，`i2c2@8b12000` / `0x50`，GPIO8_3 低有效复位；`serial-nosync-novalid`，独立时钟，TSI0/2 正相、TSI1/3 反相 | RTL8211E，`rgmii-id` | `DVBIP-1004`，GPIO9_0 低有效模拟静音；SCI0 低有效供电和检卡；MT7662T | `hi3798cv200-dvbip-1004-arm64.dtb` |
| **DVB-IP1004：Hi3130v200 + R858** | 8-bit HS400 @ 150 MHz；启用 `sd0@9820000` | 双 Hi3130 使用 I2C2 和 I2C1，双路 R858 使用 I2C3；`serial-nosync`，时钟非反相，交叉接入 TSI0/1 | RTL8211E，`rgmii-id` | `DVBIP-1004`，GPIO9_0 低有效模拟静音；SCI0 开漏 VCC；RTL8192EU（`0bda:818b`） | `hi3798cv200-dvbip-1004-r858-arm64.dtb` |
| **DVB-IP1006** | 8 GB（H8G4a），8-bit HS400 @ 150 MHz；`sd0` 禁用 | MxL214C，`i2c1@8b11000` / `0x50`，内部上拉、`force-polling`，GPIO9_7 低有效复位；`serial-nosync`，共用 `TSI2_CLK`，四路反相，32-block 队列和硬件 PID 过滤 | RTL8211E，`rgmii`，`realtek,rgmii-use-strap-delays` | `DVBIP-1006`，AIAO `hisilicon,tx-channel = <3>`；SCI0 高有效供电和检卡；外接 USB 无线 | `hi3798cv200-dvbip-1006-arm64.dtb` |

四款机型的 USB 2.0／3.0 主控和电源／状态灯位于公共 `hi3798cv200-dvbip.dtsi`。

---

## 设备树、驱动与用户态节点

| 子系统 | 设备树路径 / 节点 | 内核驱动 / 模块 | 用户态接口 |
| --- | --- | --- | --- |
| CPU 调频与调压 | `/cpus/cpu@0..3`、`cpu-avs@8a23078`、`regulator@8a23018` | `cpufreq-dt`、`hi3798cv200-avs`、`hi3798cv200-pwm-regulator` | `/sys/devices/system/cpu/cpufreq/policy0` |
| CPU 空闲 | `/cpus/cpu@0..3` | Arm WFI CPUIdle | `/sys/devices/system/cpu/cpuN/cpuidle/state0` |
| 温度传感器与看门狗 | `temperature-sensor@8a23028`、`watchdog@8a2c000` | `hisi_thermal`、`sp805-wdt`（`restart_load=0x100`） | `/sys/class/thermal/thermal_zone0`、`/dev/watchdog0` |
| 硬件随机数 | `rng@8005200` | `hisi-rng`（`hisilicon,hi3798cv200-rng`） | `/dev/hwrng` |
| 板载 eMMC | `mmc@9830000` | `dw_mmc-hi3798cv200` | `/dev/mmcblk0`、`mmcblk0p*`、`mmcblk0boot0/1` |
| TF / SD 卡槽 | `mmc@9820000`（1002、1004 启用） | `dw_mmc-hi3798cv200` | 插卡后用 `lsblk` 查看块设备 |
| 千兆以太网 | `ethernet@9841000`、`ethernet-phy@3` | `hix5hd2_gmac` + `realtek` PHY（`CONFIG_REALTEK_PHY=y`） | `eth0` |
| 板载 DVB-C I2C | 1001／1002／1006 使用 `i2c@8b11000`；1004 MxL214 使用 `i2c@8b12000`；1004 R858 使用 I2C1、I2C2 和 I2C3 | `i2c-hix5hd2`（1006 启用 `hisilicon,force-polling`） | `/dev/i2c-N` |
| 板载 DVB-C 与解扰 | 四路版接入 `demux@9c00000/input@0..3`；R858 双路版接入 `input@0..1` | `mxl214`、`hi3130` + `r858`；TS 由 `histb-dmx` 处理 | `/dev/dvb/adapterN/` 下的 `frontend0`、`demux0`、`dvr0` 和对应 `ca*` |
| 显示与 HDMI | `display-controller@8cc0000`、`hdmi@8ce0000` | `histb-vdp.ko`、`histb-hdmi.ko` | `/dev/dri/cardN`、`cardN-HDMI-A-1`、`/dev/cec0` |
| 3D GPU | `/gpu@f9200000`（顶层节点，脱离 `/soc` 1 GiB 窗口） | `panfrost.ko`（`hisilicon,hi3798cv200-mali`） | `/dev/dri/renderD128`（及对应 DRM card） |
| 音频（模拟 + HDMI） | `audio-controller@8cd0000`、`perictrl@8a20000` | `snd-soc-hi3798cv200-aiao.ko`、`snd-soc-hi3798cv200-tianlai.ko` | `/dev/snd/*`（声卡名 `DVBIP-1001/1002/1004/1006`） |
| 智能卡 / 串口 / 红外 | `serial@8b18000`、`serial@8b00000`、`serial@8b02000`、`ir@8001000` | `hi3798cv200-sci.ko`、`amba-pl011`、`ir-hix5hd2.ko`（含 `rc-gehua`） | `/dev/ttySCI0`、`/dev/ttyAMA0`、`/dev/ttyAMA2`、`/dev/lirc0`、`/dev/input/eventN` |
| 视频硬解与后处理 | `video-codec@8c30000`、`video-post-processing@8cb0000` | `histb-vdec.ko`、`histb-vpss.ko`、`histb-media-smmu.ko` | `/dev/videoN`、`/dev/mediaN` |
| 视频硬编与 JPEG 编解码 | `video-codec@8c80000`、`jpeg-decoder@8c40000`、`jpeg-encoder@8c90000` | `histb-venc.ko`、`histb-jpgd.ko`、`histb-jpge.ko` | `/dev/videoN` |
| USB 2.0 / 3.0 主控 | `usb@9880000`、`usb@9890000`、`usb@98a0000`、`usb@98b0000` | `ohci-platform`、`ehci-platform`、`xhci-histb` | `/dev/bus/usb/*`、外置存储/串口/网卡等 USB 设备 |

> 设备编号约定：`adapterN`、`videoN`、`mediaN`、`cardN`、`i2c-N` 由内核并行探测顺序动态分配，自动化脚本应通过 sysfs 设备路径、驱动名称、`media-ctl` 拓扑或 `/dev/v4l/by-path` 识别硬件节点，避免硬编码序号。

---

## 内存映射与媒体保留区

Hi3798CV200 外设总线（`/soc@f0000000`）配置 `dma-ranges = <0x0 0x0 0x0 0x40000000>`，将常规外设 DMA 限制在低 1 GiB（`ZONE_DMA`）。GPU（`/gpu@f9200000`）位于设备树根节点并具备 40-bit MMU，可直接访问全部 2 GiB 物理内存；HiVXE 媒体引擎则结合低 1 GiB 专用保留池与共享 `histb-media-smmu` 页表运行。

`hi3798cv200-dvbip.dtsi` 统一定义的保留内存如下：

| 节点名 | 物理地址范围 | 容量与属性 | 用途 |
| --- | --- | --- | --- |
| `smp-firmware@11000000` | `0x11000000..0x1100ffff` | 64 KiB，`no-map` | EL3/EL2 驻留代码与数据 |
| `decoder-capture@12000000` | `0x12000000..0x1bffffff` | 160 MiB，`reusable` CMA | 启动后回收 `0x12000000` 跳板区，供 VDEC/VPSS 线性输出帧缓冲共享分配 |
| `vdec-pool@20000000` | `0x20000000..0x2effffff` | 240 MiB，`no-map` | VDH 硬件解码参考面（10-bit/8-bit tile DPB）、PMV 与消息描述符专用池 |
| `linux,cma` | `0x30000000..0x3fffffff` 窗口内 | 192 MiB，`reusable` 默认 CMA | DRM/KMS（`histb-vdp` 扫描缓冲）及通用连续物理内存分配 |
| `venc-pool@3c000000` | `0x3c000000..0x3fffffff` | 64 MiB，`no-map` | H.264 VENC 重建面与编码硬件工作缓冲专用池 |

---

## 公共子系统架构实现

### 1. CPU DVFS、软件 AVS/HPM 与温控策略

四款机型共用 `hi3798cv200-dvbip.dtsi` 的五档 CPU OPP、PWM 调压（`650..1150 mV`）与软件 HPM 闭环调压策略，默认启用 `performance` 调速器（`CONFIG_CPU_FREQ_DEFAULT_GOV_PERFORMANCE=y`）并保留完整多档调速与热保护能力：

| CPU 频率 | 初始目标电压 / AVS 下限 / 上限 | HPM 目标值 | 调压方式 |
| --- | --- | --- | --- |
| 400 MHz | `1.00 V`（范围 `0.84–1.00 V`） | `0xc9` | 固定 OPP 电压（`enable_avs = 0`） |
| 600 MHz | `1.00 V`（范围 `0.84–1.00 V`） | `0xc9` | 固定 OPP 电压（`enable_avs = 0`） |
| 800 MHz | `1.00 V` / `0.84 V` / `1.00 V` | `0xe5` | 软件 HPM 闭环 AVS（`10 mV` 步进） |
| 1200 MHz | `1.05 V` / `0.90 V` / `1.05 V` | `0x110` | 软件 HPM 闭环 AVS（`10 mV` 步进） |
| 1600 MHz | `1.15 V` / `1.02 V` / `1.15 V` | `0x16a`（362） | 软件 HPM 闭环 AVS（`10 mV` 步进） |

- `hi3798cv200-avs` 关闭硬件自动 AVS，在 `800 / 1200 / 1600 MHz` 且结温低于 `95°C` 时运行有界软件 HPM 反馈；切换频率前自动恢复对应 OPP 安全上限电压。
- 温控阈值（`soc-thermal`）：`80°C` 触发 CPU 被动降频（1002/1004/1006 另设 `95°C` 二级被动点），`105°C` 触发 GPU 降频，`120°C` 触发系统保护关机/复位。

### 2. GPU（Mali-T720 / Panfrost）与显示（VDP / HDMI）

- **CV200 专用 Panfrost 初始化**：`panfrost_hi3798cv200.c` 匹配 `hisilicon,hi3798cv200-mali`，启动时读取 `SYSCTRL_SVB`（`0xf80000c4` 高字节）自动选择芯片工艺分档（`FF / TT / SS`，未知值回退至保守 `SS` 表），初始化 PMC HPM 监测寄存器，并启用 `GPU_PM_KEEP_L2_ON | GPU_PM_RESET_LOW_FREQ`（高于 400 MHz 触发软复位时先保持高电压、临时切至 200 MHz 完成安全复位再恢复原频率）及作业提交前 fence 发布同步。

| GPU 频率 | FF 分档电压 | TT 分档电压 | SS 分档电压 |
| --- | --- | --- | --- |
| 200 / 300 / 400 / 500 MHz | 850 mV | 880 mV | 900 mV |
| 600 MHz | 880 mV | 920 mV | 930 mV |
| 675 MHz | 920 mV | 980 mV | 1000 mV |

- **显示控制器与 HDMI 2.0**：`histb-vdp` 暴露图形平面 `G0`（支持 `XRGB8888 / ARGB8888` 逐像素 Alpha OSD）与硬件视频叠加平面 `V0 / V1`（支持 `NV12 / NV21 / P010`、硬件 ZME 缩放及色彩空间矩阵），配合 Debian 12/13 官方 `Mesa`（`panfrost_dri`）可直接驱动 DRM/KMS、GBM/EGL 硬件合成与视频直通叠层。`histb-hdmi` 支持最高 4K60（594 MHz TMDS / SCDC 加扰）、专用 DDC EDID 读取、HPD 轮询热插拔、HDMI-CEC（`/dev/cec0`）与 HDCP 1.4/2.2。

### 3. DVB-C 解复用、音频、智能卡与红外

- **MxL214 系列 DVB-C**：加载 `/lib/firmware/mxl214/mxl214.fw`，提供 4 路 DVB-C Annex A（44–1002 MHz、1.0–7.125 MSps、16/32/64/128/256-QAM）及 DVBv5 功率、CNR、BER、误码块统计。`histb-dmx` 支持 `PID_COPY` 和 CSA2，1002、1004 MxL214、1006 使用 96 通道硬件 PID 过滤和 32-block DMA 队列。
- **音频（AIAO + Tianlai ADAC）**：`hi3798cv200-aiao` 基于 `933.888 MHz` MCLK 驱动模拟立体声 DAI（`hi3798-aiao-analog`）与 HDMI I2S DAI（`hi3798-aiao-hdmi`），支持 `S16_LE / S32_LE`。`hi3798cv200-tianlai` 借用父级 `perictrl` 共享 syscon regmap 完成无爆音上下电斜坡控制（组件卸载时仅解绑而不释放共享 regmap）。
- **S/PDIF 光纤音频**：AIAO DAI 2 使用独立的 SPDIF-TX DMA 通道，公共设备树已连接 `spdif-dit` 与 TX1/pad1（模块 `SND_SOC_SPDIF`）。支持双声道 `S16_LE` / `S32_LE`（24 位有效）、32–192 kHz 采样率；可通过 `IEC958 Playback Default` 的 `NONAUDIO` 位启用 AC-3/DTS 源码直通（下次 PCM `prepare` 时生效）。模拟、HDMI 与 S/PDIF 三路输出互斥使用，每次打开其中一路。
- **智能卡（`ttySCI0`）**：`hi3798cv200-sci` 实现完整 ISO/IEC 7816 状态机（正向/反向编码约定、T=0 / T=1 ATR 解析、`TIOCSISO7816` ioctl），并在 `/sys/bus/platform/devices/f8b18000.serial/sci/` 下提供 22 项诊断与恢复属性。
- **红外接收（`hix5hd2-ir`）**：AArch64 EL3 引导阶段统一向 `SEC_ATTR_CTRL_0_REG`（`0xf8a80010`）写入 `0xfff9ffff` 开放非安全态红外寄存器访问；内核驱动按 7-bit 位宽（`GENMASK(6, 0)`）读取 64 深度的 `IR_DATAH` FIFO 计数，支持 LIRC 原始脉冲波形捕获及内置 `rc-gehua`（NECX）等按键映射。

### 4. 休眠管理与遥控器电源键策略

Linux 6.12.118 默认配置了遥控器电源键的短按与长按行为：

| 机型 | 短按松开 | 长按 5 秒 | 休眠与红外唤醒说明 |
| --- | --- | --- | --- |
| **1001 / 1002 / 1004 MxL214 / 1006** | 请求进入深睡（`deep`） | 有序关机 | 1004 MxL214 与 1006 支持深睡及红外唤醒 |
| **1004 Hi3130v200 + R858** | 无动作 | 有序关机 | DTB 通过 `hisilicon,suspend-disabled` 禁用系统休眠；关机后按红外电源键执行冷启动 |

- 如需自定义电源键策略，可通过 `/sys/bus/platform/devices/standby-controller/power_key_policy` 调整，格式为 `owner short_action long_action hold_ms`（交由 `systemd-logind` 管理时将 `owner` 设为 `userspace`；R858 默认配合 `dvbip-power-key-policy.service` 使用 `userspace ignore poweroff 5000`）。

---

## 分型号板级链路与节点树

### DVB-IP1001

#### 以太网与存储
- **eMMC**：8 GB（`8GTF4R`），8-bit DDR52 模式，最高 52 MHz（`MUX_M2` 引脚组）；无 TF 卡槽。
- **以太网**：`GMAC1`（`ethernet@9841000`）经 MDIO 地址 `3` 连接 Realtek `RTL8211E`（PHY ID `0x001cc915`），配置标准 `phy-mode = "rgmii-id"`（由 PHY 提供 TX/RX 双向时钟延时）。

#### 板载 DVB-C 与 TS 总线
```text
CATV 同轴 -> RF 匹配/滤波 -> MxL214
                              |-- i2c1@8b11000 / 0x50 / 400 kHz
                              |-- GPIO7_4 低有效硬件复位
                              |-- channel 0..3 -> TSI0..TSI3
                              `-- serial-4wire -> serial-nosync（共用 TSI2_CLK，四路反相采样）
```

#### 节点树
```text
/
|-- gpu@f9200000 --------------------------> Mali-T720 /dev/dri/renderD128
|-- leds ----------------------------------> GPIO5_1 (power), GPIO5_2 (display)
`-- soc@f0000000
    |-- mmc@9830000 -----------------------> /dev/mmcblk0（8 GB DDR52 eMMC，p5 rootfs）
    |-- ethernet@9841000/ethernet-phy@3 ---> eth0（RTL8211E，rgmii-id）
    |-- i2c@8b11000/demodulator@50 --------> MxL214 -> TSI0..3
    |-- demux@9c00000/input@0..3 ----------> /dev/dvb/adapter0..3（frontend0/demux0/dvr0/ca0..3）
    |-- display-controller@8cc0000 --------> /dev/dri/cardN（G0 + V0/V1）
    |-- hdmi@8ce0000 ----------------------> cardN-HDMI-A-1, /dev/cec0
    |-- audio-controller@8cd0000 ----------> /dev/snd/*（声卡 DVBIP-1001）
    |-- serial@8b18000 --------------------> /dev/ttySCI0（VCC 低有效，CD 低有效）
    |-- serial@8b00000 / serial@8b02000 ---> /dev/ttyAMA0, /dev/ttyAMA2
    |-- ir@8001000 ------------------------> /dev/lirc0
    |-- video-codec@8c30000 / vpss@8cb0000 -> VDEC + VPSS /dev/videoN, /dev/mediaN
    |-- video-codec@8c80000 ---------------> VENC /dev/videoN
    |-- jpeg-decoder@8c40000 / jpge@8c90000 -> JPGD + JPGE /dev/videoN
    `-- usb@9880000/9890000/98a0000/98b0000 -> USB 2.0 / 3.0 Root Hubs
```

---

### DVB-IP1002

#### 以太网与存储
- **eMMC 与 TF**：8 GB（`DG4008`），8-bit 1.8 V HS400 模式（150 MHz，继承引导采样/strobe 相位）；启用 `sd0@9820000`（4-bit 3.3 V TF 卡槽，最高 50 MHz）。
- **以太网**：`GMAC1` 连接 `RTL8211F`（PHY ID `0x001cc916`，MDIO `3`），使用 `rgmii-id` 并启用 `CONFIG_HISTB_RTL8211F_LED=y` 驱动网口指示灯。GMAC 启动时先打开 MAC/接口时钟并恢复 PHY，再连接链路（避免早期内核可能出现的 `phy_poll_reset failed: -110` 问题）。

#### 板载 DVB-C 与 TS 总线
```text
CATV 同轴 -> RF 匹配/滤波 -> MxL214C
                              |-- i2c1@8b11000 / 0x50 / 400 kHz
                              |-- GPIO7_4 低有效硬件复位
                              |-- channel 0..3 -> TSI0..TSI3
                              `-- serial-4wire -> serial-nosync（共用 TSI2_CLK，四路反相采样，32-block + 硬件 PID 过滤）
```

#### 节点树
```text
/
|-- gpu@f9200000 --------------------------> Mali-T720 /dev/dri/renderD128
`-- soc@f0000000
    |-- mmc@9830000 -----------------------> /dev/mmcblk0（8 GB HS400 eMMC，p5 rootfs）
    |-- mmc@9820000 -----------------------> /dev/mmcblk1（TF 卡槽）
    |-- ethernet@9841000/ethernet-phy@3 ---> eth0（RTL8211F，rgmii-id）
    |-- i2c@8b11000/demodulator@50 --------> MxL214C -> TSI0..3
    |-- demux@9c00000/input@0..3 ----------> /dev/dvb/adapter0..3（frontend0/demux0/dvr0/ca0..3）
    |-- display-controller@8cc0000 --------> /dev/dri/cardN（G0 + V0/V1）
    |-- hdmi@8ce0000 ----------------------> cardN-HDMI-A-1, /dev/cec0
    |-- audio-controller@8cd0000 ----------> /dev/snd/*（声卡 DVBIP-1002）
    |-- serial@8b18000 --------------------> /dev/ttySCI0（VCC 低有效，CD 高有效）
    |-- serial@8b00000 / serial@8b02000 ---> /dev/ttyAMA0, /dev/ttyAMA2
    |-- ir@8001000 ------------------------> /dev/lirc0
    |-- video-codec@8c30000 / vpss@8cb0000 -> VDEC + VPSS /dev/videoN, /dev/mediaN
    |-- video-codec@8c80000 ---------------> VENC /dev/videoN
    |-- jpeg-decoder@8c40000 / jpge@8c90000 -> JPGD + JPGE /dev/videoN
    |-- usb@9890000 -----------------------> 板载 05e3:0608 USB Hub + 外置 USB 口
    `-- usb@98b0000 -----------------------> 板载 RTL8192EU Wi-Fi (0bda:818b) + USB 3.0
```

---

### DVB-IP1004

DVB-IP1004 存在 **四路 MxL214** 与 **双路 Hi3130v200 + R858** 两个硬件版本，共用 `hi3798cv200-dvbip-1004.dtsi` 中的存储、网络和音频基础配置，射频与无线配置见下文各自小节。

#### 1. 公共以太网、存储与音频
- **eMMC 与 TF**：8 GB（`8GME4R`，2+8 机型）或 4 GB 级（`004G60`，2+4 机型），8-bit 1.8 V HS400 模式（150 MHz，继承引导调谐）；启用 `sd0@9820000` TF 卡槽。`004G60` 已经实测验证支持从原厂 HS200 平滑重调谐进入 HS400。
- **以太网**：`GMAC1` 连接 `RTL8211E`（PHY ID `0x001cc915`，MDIO `3`），使用 `rgmii-id`。
- **音频**：声卡 `DVBIP-1004`，模拟音频带 `GPIO9_0` 低有效静音控制。

#### 2. 硬件变体 A：四路 MxL214（`hi3798cv200-dvbip-1004-arm64.dtb`）
- **适用机型**：常见于 2+8 GB 批次。
- **板载 DVB-C 拓扑**：
```text
CATV 同轴 -> RF 匹配/滤波 -> MxL214
                              |-- i2c2@8b12000 / 0x50 / 400 kHz
                              |-- GPIO8_3 低有效硬件复位
                              |-- channel 0..3 -> TSI0..TSI3
                              `-- serial-3wire -> serial-nosync-novalid（四路独立时钟，TSI0/2 正相、TSI1/3 反相，32-block + 硬件 PID 过滤）
```
- **智能卡（SCI0）**：标准低有效供电与低有效检卡（`sci_pins` + `GPIO9_1`）。
- **无线网卡**：板载 `05e3:0608` USB Hub + MT7662T Wi-Fi/BT（`0e8d:76a0`）。
- **eMMC 暖重启**：MxL214 DTB 启用 `hisilicon,hold-reset-on-reboot`。eMMC 须启用硬件复位。MMC 驱动在卡关机且控制器空闲后拉低复位释放信号，保持复位至下次 Fastboot 初始化。2+8／8GME4R 的暖重启后，四核、eMMC 和四路 DVB 前端恢复。
- **升级旧复位方案**：安装并校验同次构建的内核、MxL214 容器和模块。重启前取消旧复位插件的自动加载；新内核启动后清理旧模块文件。

#### 3. 硬件变体 B：双路 Hi3130v200 + R858（`hi3798cv200-dvbip-1004-r858-arm64.dtb`）
- **双路接收**：支持不同频点同时解调和录制，已在 2+4／`004G60` 机型运行。
- **板载 DVB-C 拓扑与独立 I2C3 调谐总线**：
```text
CATV 同轴 -> R858 双路调谐器封装（独立 I2C3，0xf8b13000）
              |-- Receiver 0: 7-bit 0x0a（8-bit 0x14），5.0 MHz IF（频率控制字 0x1550）
              |-- Receiver 1: 7-bit 0x1a（8-bit 0x34），5.5 MHz IF（频率控制字 0x162b）
              `-- 共享 24 MHz 晶振，统一执行镜像与滤波器校准
                     |
                     +--> Hi3130v200 端口 0 (i2c2@8b12000 / 0x50 / GPIO7_4 复位) -> TSI1 (交叉输入)
                     +--> Hi3130v200 端口 1 (i2c1@8b11000 / 0x50 / GPIO8_3 复位) -> TSI0 (交叉输入)
                                    `-- serial-nosync，时钟非反相（严禁使用 novalid）
```
- **智能卡（SCI0）**：使用 `hisilicon,vcc-enable-open-drain`、CMOS 时钟与复位。ISO 7816 T=0 可读取 16 字节 ATR。
- **无线网卡**：板载 `05e3:0608` USB Hub + RTL8192EU（`0bda:818b`），提供 `rtlwifi/rtl8192eu_nic.bin` 及 `regulatory.db` 即可正常联网。

#### 4. 节点树（以 Hi3130+R858 双前端变体为例）
```text
/
|-- gpu@f9200000 --------------------------> Mali-T720 /dev/dri/renderD128
`-- soc@f0000000
    |-- mmc@9830000 -----------------------> /dev/mmcblk0（HS400 eMMC，p6 rootfs）
    |-- mmc@9820000 -----------------------> /dev/mmcblk1（TF 卡槽）
    |-- ethernet@9841000/ethernet-phy@3 ---> eth0（RTL8211E，rgmii-id）
    |-- i2c@8b12000/demodulator@50 --------> Hi3130 端口 0 -> TSI1
    |-- i2c@8b11000/demodulator@50 --------> Hi3130 端口 1 -> TSI0
    |-- i2c@8b13000 -----------------------> R858 双路调谐器总线 (0x0a: 5MHz IF, 0x1a: 5.5MHz IF)
    |-- demux@9c00000/input@0..1 ----------> /dev/dvb/adapter0..1（frontend0/demux0/dvr0/ca0..1）
    |-- display-controller@8cc0000 --------> /dev/dri/cardN（G0 + V0/V1）
    |-- hdmi@8ce0000 ----------------------> cardN-HDMI-A-1, /dev/cec0
    |-- audio-controller@8cd0000 ----------> /dev/snd/*（声卡 DVBIP-1004，Tianlai 带 GPIO9_0 静音）
    |-- serial@8b18000 --------------------> /dev/ttySCI0（开漏 VCC 使能，CMOS 时钟与复位）
    |-- serial@8b00000 / serial@8b02000 ---> /dev/ttyAMA0, /dev/ttyAMA2
    |-- ir@8001000 ------------------------> /dev/lirc0
    |-- video-codec@8c30000 / vpss@8cb0000 -> VDEC + VPSS /dev/videoN, /dev/mediaN
    |-- video-codec@8c80000 ---------------> VENC /dev/videoN
    |-- jpeg-decoder@8c40000 / jpge@8c90000 -> JPGD + JPGE /dev/videoN
    `-- usb@9890000 / usb@98a0000 ---------> 板载 USB Hub + 无线网卡 + 外置 USB 端口
```
---

### DVB-IP1006

#### 以太网与存储
- **eMMC**：8 GB（`H8G4a`），8-bit 1.8 V HS400 模式（150 MHz，继承引导调谐）；无 TF 卡槽。
- **以太网**：`GMAC1` 连接 `RTL8211E`（PHY ID `0x001cc915`，MDIO `3`），使用 `phy-mode = "rgmii"` 配合 PHY 节点 `realtek,rgmii-use-strap-delays` 保留板级硬件 strap 时钟延时。

#### 板载 DVB-C 与 TS 总线
```text
CATV 同轴 -> RF 匹配/滤波 -> MxL214C
                              |-- i2c1@8b11000 / 0x50（SoC 内部上拉 + force-polling）
                              |-- GPIO9_7 低有效硬件复位
                              |-- channel 0..3 -> TSI0..TSI3
                              `-- serial-4wire -> serial-nosync（共用 TSI2_CLK，四路反相采样，32-block + 硬件 PID 过滤）
```

#### 节点树
```text
/
|-- gpu@f9200000 --------------------------> Mali-T720 /dev/dri/renderD128
`-- soc@f0000000
    |-- mmc@9830000 -----------------------> /dev/mmcblk0（8 GB HS400 eMMC，p5 rootfs）
    |-- ethernet@9841000/ethernet-phy@3 ---> eth0（RTL8211E，rgmii + strap delays）
    |-- i2c@8b11000/demodulator@50 --------> MxL214C（force-polling）-> TSI0..3
    |-- demux@9c00000/input@0..3 ----------> /dev/dvb/adapter0..3（frontend0/demux0/dvr0/ca0..3）
    |-- display-controller@8cc0000 --------> /dev/dri/cardN（G0 + V0/V1）
    |-- hdmi@8ce0000 ----------------------> cardN-HDMI-A-1, /dev/cec0
    |-- audio-controller@8cd0000 ----------> /dev/snd/*（声卡 DVBIP-1006，AIAO tx-channel = <3>）
    |-- serial@8b18000 --------------------> /dev/ttySCI0（VCC 高有效，CD 高有效）
    |-- serial@8b00000 / serial@8b02000 ---> /dev/ttyAMA0, /dev/ttyAMA2
    |-- ir@8001000 ------------------------> /dev/lirc0
    |-- video-codec@8c30000 / vpss@8cb0000 -> VDEC + VPSS /dev/videoN, /dev/mediaN
    |-- video-codec@8c80000 ---------------> VENC /dev/videoN
    |-- jpeg-decoder@8c40000 / jpge@8c90000 -> JPGD + JPGE /dev/videoN
    `-- usb@9880000/9890000/98a0000/98b0000 -> USB 2.0 / 3.0 Root Hubs
```

---

## HiTool 物理布局

| 型号 | eMMC 物理分区布局（起始偏移 + 容量，单位 MiB） | Linux `blkdevparts` 视图与硬件身份源 | rootfs Sparse 编码 |
| --- | --- | --- | --- |
| **1001 / 1006** | `fastboot 0+4`、`bootargs 4+4`、`deviceinfo 8+152`、`signed kernel 160+40`、`misc 200+20`、`trustedcore 220+10`、`rootfs 230+-` | `p1` fastboot、`p2` bootargs、`p3` deviceinfo（212M，MAC/SN 位于 `+12 MiB`）、`p4` **trustedcore（项目内核）**、`p5` **rootfs** | `android-standard`（RAW + FILL） |
| **1002** | `fastboot 0+4`、`bootargs 4+4`、`hwconfig 8+4`、`kernel 12+16`、`rootfs 28+-` | `p1` fastboot、`p2` bootargs、`p3` hwconfig（MAC 位于 `+160`、SN 位于 `+168`）、`p4` **kernel（项目内核）**、`p5` **rootfs** | `raw-only-full-overwrite`（纯 RAW） |
| **1004** | `fastboot 0+1`、`bootargs 1+1`、`recovery 2+11`、`misc 13+1`、`hwconfig 14+4`、`rootfs 18+-` | `p1` boot、`p2` bootargs、`p3` **recovery（项目 11M 内核）**、`p4` misc、`p5` hwconfig（MAC `+160`、SN `+168`）、`p6` **rootfs** | `raw-only-full-overwrite`（纯 RAW） |

- **刷机文件**：按上表选用本机型的 `full.xml`、`bootargs`、内核容器与 `rootfs.img`。1004 按板载射频芯片选包。
- **保留身份分区**：`full.xml` 默认不勾选 `deviceinfo` / `hwconfig`（`Sel="0"`），系统启动时由 `dvbip-hardware-identity.service` 自动读取本机原厂 MAC 与序列号并应用到 `eth0`。
- **HiTool 配置**：关闭“启动自动更新 bootargs 中分区表信息”，按 Android sparse 格式刷入 `rootfs.img`。首次启动由 `systemd-growfs-root.service` 扩展 rootfs。

---

## 固件、文件系统与软件交付

### 1. 系统环境与内核特性

- **基础系统**：Debian 12（`bookworm`）或 Debian 13.7（`trixie`）AArch64，带 `openssh-server`、`sudo`、`curl`、`dialog`、`cifs-utils`、`keyutils`。初始登录信息见对应[镜像下载页](https://github.com/HiSilicon-Development/firmware/releases)，登录后执行 `passwd` 更新密码。
- **文件系统与容器支持**：内核内置或模块化支持本地文件系统（`ext4`、`vfat`、`msdos`、`exfat`、`ntfs3`、`xfs`、`btrfs`、`f2fs`、`squashfs`、`iso9660`、`udf`、`fuse`、`overlayfs`）及网络客户端文件系统（`NFSv3 / v4.0 / v4.1 / v4.2`、`CIFS / SMB2.1 / SMB3.1.1`，禁用不安全的旧版 SMB1/NFSv2），同时完整开启 Docker 容器所需的 Namespace、Cgroups 与 Netfilter/nftables 特性及常见 USB 串口/有线网卡/大容量存储驱动。
- **内核更新**：更新包写入对应内核分区（1001／1002／1006：`/dev/mmcblk0p4`；1004：`/dev/mmcblk0p3`），安装 `/lib/modules/<release>/`，回读校验分区 SHA-256。沿用设备上现有的运行时固件。

### 2. 标准固件清单（`/lib/firmware/`）

按板型及外设从下表选取固件，放在 `/lib/firmware/`（Debian merged-usr 下为 `/usr/lib/firmware/`）。

| 类别 | 固件路径（相对于 `/lib/firmware/`） | 对应驱动与说明 |
| --- | --- | --- |
| **板载 DVB-C** | `mxl214/mxl214.fw` | MxL214／MxL214C 驱动加载。Hi3130v200 + R858 的解调微码与调谐表内嵌在驱动中。 |
| **海思 HiVXE 硬解** | `hisilicon/histb-h264-cabac.bin`<br>`hisilicon/histb-hevc-cabac.bin`<br>`hisilicon/histb-avsp.bin` | `histb-vdec.ko` H.264/HEVC CABAC 表与 CV200 AVS/AVS+ SCD 固件（可由内核 `make hisilicon-firmware` 生成） |
| **无线网络与监管域** | `rtlwifi/rtl8192eu_nic.bin`<br>`regulatory.db`<br>`regulatory.db.p7s` | 板载/外置 Realtek Wi-Fi 固件与 Linux 802.11 无线频段签名数据库 |

### 3. 扩展媒体组件

按 [编译教程](https://github.com/HiSilicon-Development/manifest) 生成 HiVXE FFmpeg，按 [extra](https://github.com/HiSilicon-Development/extra) 说明安装。转码参数见 [HiVXE 文档](https://github.com/HiSilicon-Development/HiVXE-2.0)。
