# DVB-IP100x 产品与硬件总览

本文档统一记录歌华 **DVB-IP1001、DVB-IP1002、DVB-IP1004（2+8）与 DVB-IP1006** 四款机型的硬件参数、设备树与驱动映射、内存保留区、板级总线差异、HiTool 分区布局及软件交付规范。

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

AArch32 Fastboot 到 AArch64 Linux 的启动链与容器结构见 [Hi3798CV200 AArch32 → AArch64 启动架构](Hi3798Cv200-AArch64-Boot-Doc.md)；项目入口与外置 USB DTMB 外设支持见 [README](README.md)。

---

## 公共平台规格

四款设备的板载硬件与核心外设在当前 Linux 6.12.y（`6.12.114` 统一内核树）及 Debian 12 / Debian 13（`trixie`）AArch64 系统下均已完整支持并可正常使用。

| 模块 | 公共规格与实现 |
| --- | --- |
| SoC | HiSilicon Hi3798CV200（高安 CA 版） |
| CPU | 4 × Arm Cortex-A53 r0p4（ARMv8-A / AArch64），支持 400 / 600 / 800 / 1200 / 1600 MHz DVFS 与软件 AVS/HPM |
| GPU | Arm Mali-T720 MP（`Panfrost` 开源驱动，200–675 MHz 六档 OPP，按 SVB 工艺分档调压） |
| 内存 | 2 GiB DDR（`0x00000000..0x7fffffff`） |
| 显示与 HDMI | 片内 VDP（`histb-vdp`，含图形层 `G0` 与硬件视频层 `V0/V1`、ZME 缩放、BT.601/709/2020 CSC）+ HDMI 2.0 TX/PHY（`histb-hdmi`，最高 4K60、SCDC、HPD 轮询、`/dev/cec0`、HDCP 1.4/2.2） |
| 硬件编解码 | HiVXE 媒体子系统：`VDEC`（VDH V4R3）、`VPSS`（去块/缩放/四场硬件 DEI）、`VENC`（H.264 VEDU）、`JPGD`（JPEG 硬解）、`JPGE`（JPEG 硬编） |
| 板载射频与解复用 | 板载单芯片四路有线数字电视（DVB-C Annex A，`MaxLinear MxL214 / MxL214C`）→ `TSI0..TSI3` → `histb-dmx`（含 32 槽硬件 CSA2 解扰器 `ca0..3`）。**注：四款机型均无板载地面波（DTMB）硬件，DTMB 仅通过外置 USB 接收器扩展支持** |
| 音频 | 片内 AIAO（`snd-soc-hi3798cv200-aiao`）+ 内置立体声 Tianlai ADAC（`snd-soc-hi3798cv200-tianlai`）+ HDMI I2S 数字音频 |
| 智能卡 | 片内 ISO/IEC 7816 接口 `SCI0`（`hi3798cv200-sci`，`/dev/ttySCI0`） |
| 红外与串口 | 红外接收器 `hix5hd2-ir`（`/dev/lirc0`）；`UART0`（`ttyAMA0` 115200 8N1 控制台）+ `UART2`（`ttyAMA2` 辅助串口） |
| USB 主控 | 1 × OHCI（`usb@9880000`）、1 × EHCI（`usb@9890000`）、2 × xHCI USB 3.0（`usb@98a0000`、`usb@98b0000`） |
| 内核与系统基线 | Linux `6.12.114`（`hi3798cv200_defconfig` + `dvbip_100{1,2,4,6}_defconfig`），Debian 12 / 13 ARM64 |

---

## 四款机型硬件差异矩阵

虽然四款设备共用同一颗 Hi3798CV200 SoC，但各自的 eMMC 时序、TF 卡槽、以太网 PHY 延时策略、MxL214 I2C/复位/TS 接线、音频通道及智能卡极性均存在板级差异，内核通过各板独立 DTB 精确描述：

| 型号 | eMMC 与 TF 卡槽 | 板载四路 DVB-C 与 TS 总线 | 千兆以太网（GMAC1 / MDIO 3） | 板载 USB / 无线与外围细节 | 状态 |
| --- | --- | --- | --- | --- | --- |
| **DVB-IP1001** | 8 GB（8GTF4R），8-bit **DDR52 @ 52 MHz**（`MUX_M2`）；无 TF 卡槽 | `MxL214`，`i2c1@8b11000`（`0x50`），`GPIO7_4` 低有效复位；`serial-4wire` → `serial-nosync`；共用 `TSI2_CLK`，四路反相采样；16-block TS DMA 队列 | RTL8211E（`0x001cc915`），`rgmii-id`（PHY 提供 TX/RX 双向延时） | 外置 USB 3.0 + USB 2.0 接口；板载 `GPIO5_1` 电源灯、`GPIO5_2` 状态灯；SCI0 低有效供电/低有效检卡；声卡 `DVBIP-1001`（模拟 TX0 + HDMI TX3） | 已全部支持 |
| **DVB-IP1002** | 8 GB（DG4008），8-bit **HS400 @ 150 MHz**（继承引导调谐）；带 TF 卡槽（`sd0@9820000`） | `MxL214C`，`i2c1@8b11000`（`0x50`），`GPIO7_4` 低有效复位；`serial-4wire` → `serial-nosync`；共用 `TSI2_CLK`，四路反相采样；32-block 队列 + 硬件 PID 过滤 | RTL8211F（`0x001cc916`），`rgmii-id`，启用 `CONFIG_HISTB_RTL8211F_LED` 链路/活动灯 | 板载 `05e3:0608` USB Hub + RTL8192EU Wi-Fi（`0bda:818b`）；SCI0 低有效供电、**高有效检卡**；声卡 `DVBIP-1002`（模拟 TX0 + HDMI TX3） | 已全部支持 |
| **DVB-IP1004（2+8）** | 8 GB（8GME4R），8-bit **HS400 @ 150 MHz**（继承引导调谐）；带 TF 卡槽（`sd0@9820000`）；仅限 2+8 版本 | `MxL214`，`i2c2@8b12000`（`0x50`），`GPIO8_3` 低有效复位；`serial-3wire` → `serial-nosync-novalid`；**四路独立 TS 时钟**，TSI0/2 正相、TSI1/3 反相；32-block 队列 + 硬件 PID 过滤 | RTL8211E（`0x001cc915`），`rgmii-id` | 板载 `05e3:0608` USB Hub + MT7662T Wi-Fi/BT（`0e8d:76a0`）；模拟音频带 `GPIO9_0` 低有效静音脚；SCI0 低有效供电/低有效检卡；声卡 `DVBIP-1004` | 已全部支持 |
| **DVB-IP1006** | 8 GB（H8G4a），8-bit **HS400 @ 150 MHz**（继承引导调谐）；无 TF 卡槽 | `MxL214C`，`i2c1@8b11000`（`0x50`，内部上拉 + `force-polling`），`GPIO9_7` 低有效复位；`serial-4wire` → `serial-nosync`；共用 `TSI2_CLK`，四路反相采样；32-block 队列 + 硬件 PID 过滤 | RTL8211E（`0x001cc915`），`rgmii` + `realtek,rgmii-use-strap-delays`（保持硬件 strap 延时） | 外置 USB 接口；SCI0 **高有效供电**（移除 `vcc-active-low`）、**高有效检卡**；AIAO 指定 `hisilicon,tx-channel = <3>`，声卡 `DVBIP-1006` | 已全部支持 |

> 说明：DVB-IP1004 仅交付 2 GiB DDR + 8 GB eMMC（2+8）版本，2+4 版本的硬件拓扑与分区不在本项目覆盖范围内。

---

## 设备树、驱动与用户态节点

| 子系统 | 设备树路径 / 节点 | 内核驱动 / 模块 | 用户态接口 |
| --- | --- | --- | --- |
| CPU 调频与调压 | `/cpus/cpu@0..3`、`cpu-avs@8a23078`、`regulator@8a23018` | `cpufreq-dt`、`hi3798cv200-avs`、`hi3798cv200-pwm-regulator` | `/sys/devices/system/cpu/cpufreq/policy0` |
| CPU 空闲 | `/cpus/cpu@0..3` | Arm WFI CPUIdle | `/sys/devices/system/cpu/cpuN/cpuidle/state0` |
| 温度传感器与看门狗 | `temperature-sensor@8a23028`、`watchdog@8a2c000` | `hisi_thermal`、`sp805-wdt`（`restart_load=0x100`） | `/sys/class/thermal/thermal_zone0`、`/dev/watchdog0` |
| 硬件随机数 | `rng@8005200` | `hisi-rng`（`hisilicon,hi3798cv200-rng`） | `/dev/hwrng` |
| 板载 eMMC | `mmc@9830000` | `dw_mmc-hi3798cv200` | `/dev/mmcblk0`、`mmcblk0p*`、`mmcblk0boot0/1` |
| TF / SD 卡槽 | `mmc@9820000`（1002 / 1004 启用；1001 / 1006 关闭） | `dw_mmc-hi3798cv200` | 插卡后通常为 `/dev/mmcblk1` |
| 千兆以太网 | `ethernet@9841000`、`ethernet-phy@3` | `hix5hd2_gmac` + `realtek` PHY（`CONFIG_REALTEK_PHY=y`） | `eth0` |
| 板载 DVB-C I2C | 1001/1002/1006：`i2c@8b11000`；1004：`i2c@8b12000` | `i2c-hix5hd2`（1006 启用 `hisilicon,force-polling`） | `/dev/i2c-N`（MxL214 挂载于 `0x50`） |
| 板载四路 DVB-C 与解扰 | `demodulator@50` → `demux@9c00000/input@0..3` | `mxl214.ko` + `histb-dmx.ko` | `/dev/dvb/adapter0..3/{frontend0,demux0,dvr0,ca0..3}` |
| 显示与 HDMI | `display-controller@8cc0000`、`hdmi@8ce0000` | `histb-vdp.ko`、`histb-hdmi.ko` | `/dev/dri/cardN`、`cardN-HDMI-A-1`、`/dev/cec0` |
| 3D GPU | `/gpu@f9200000`（顶层节点，脱离 `/soc` 1 GiB 窗口） | `panfrost.ko`（`hisilicon,hi3798cv200-mali`） | `/dev/dri/renderD128`（及对应 DRM card） |
| 音频（模拟 + HDMI） | `audio-controller@8cd0000`、`perictrl@8a20000` | `snd-soc-hi3798cv200-aiao.ko`、`snd-soc-hi3798cv200-tianlai.ko` | `/dev/snd/*`（声卡名 `DVBIP-1001/1002/1004/1006`） |
| 智能卡 / 串口 / 红外 | `serial@8b18000`、`serial@8b00000`、`serial@8b02000`、`ir@8001000` | `hi3798cv200-sci.ko`、`amba-pl011`、`ir-hix5hd2.ko`（含 `rc-gehua`） | `/dev/ttySCI0`、`/dev/ttyAMA0`、`/dev/ttyAMA2`、`/dev/lirc0`、`/dev/input/eventN` |
| 视频硬解与后处理 | `video-codec@8c30000`、`video-post-processing@8cb0000` | `histb-vdec.ko`、`histb-vpss.ko`、`histb-media-smmu.ko` | `/dev/videoN`、`/dev/mediaN` |
| 视频硬编与 JPEG 编解码 | `video-codec@8c80000`、`jpeg-decoder@8c40000`、`jpeg-encoder@8c90000` | `histb-venc.ko`、`histb-jpgd.ko`、`histb-jpge.ko` | `/dev/videoN` |
| USB 2.0 / 3.0 主控 | `usb@9880000`、`usb@9890000`、`usb@98a0000`、`usb@98b0000` | `ohci-platform`、`ehci-platform`、`xhci-histb` | `/dev/bus/usb/*`、外置存储/串口/网卡/USB DTMB 设备 |

> 设备编号约定：`adapterN`、`videoN`、`mediaN`、`cardN`、`i2c-N` 由内核并行探测顺序动态分配，自动化脚本应通过 sysfs 设备路径、驱动名称、`media-ctl` 拓扑或 `/dev/v4l/by-path` 识别硬件节点，避免硬编码序号。

---

## 内存映射与媒体保留区

Hi3798CV200 外设总线（`/soc@f0000000`）配置 `dma-ranges = <0x0 0x0 0x0 0x40000000>`，将常规外设 DMA 限制在低 1 GiB（`ZONE_DMA`）。GPU（`/gpu@f9200000`）位于设备树根节点并具备 40-bit MMU，可直接访问全部 2 GiB 物理内存；HiVXE 媒体引擎则结合低 1 GiB 专用保留池与共享 `histb-media-smmu` 页表运行。

`hi3798cv200-dvbip.dtsi` 统一定义的保留内存如下：

| 节点名 | 物理地址范围 | 容量与属性 | 用途 |
| --- | --- | --- | --- |
| `smp-firmware@11000000` | `0x11000000..0x1100ffff` | 64 KiB，`no-map` | AArch64 EL3/EL2 引导驻留代码与 `0x1100f000` 次级核 `spin-table` 邮箱 |
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

### 3. 四路 DVB-C 解复用、音频、智能卡与红外

- **四路 DVB-C 与硬件解扰（`mxl214` + `histb-dmx`）**：单颗 `MxL214 / MxL214C` 加载 `/lib/firmware/mxl214/mxl214.fw`，提供 4 个独立的 DVB-C Annex A 前端（44–1002 MHz，1.0–7.125 MSps，16/32/64/128/256-QAM 自动锁频，支持完整 DVBv5 射频功率、CNR、BER 与误码块统计）。`histb-dmx` 开启 `PID_COPY`（允许多适配器并发过滤相同 PID），在 1002/1004/1006 上启用 96 通道硬件 PID 过滤与 32-block 队列调优，并为每个适配器暴露 32 槽硬件 CSA2 解扰设备（`ca0..ca3`，兼容 OSCam `CA_SET_DESCR` / `CA_SET_PID`）。
- **音频（AIAO + Tianlai ADAC）**：`hi3798cv200-aiao` 基于 `933.888 MHz` MCLK 驱动模拟立体声 DAI（`hi3798-aiao-analog`）与 HDMI I2S DAI（`hi3798-aiao-hdmi`），支持 `S16_LE / S32_LE`。`hi3798cv200-tianlai` 借用父级 `perictrl` 共享 syscon regmap 完成无爆音上下电斜坡控制（组件卸载时仅解绑而不释放共享 regmap）。
- **智能卡（`ttySCI0`）**：`hi3798cv200-sci` 实现完整 ISO/IEC 7816 状态机（正向/反向编码约定、T=0 / T=1 ATR 解析、`TIOCSISO7816` ioctl），并在 `/sys/bus/platform/devices/f8b18000.serial/sci/` 下提供 22 项诊断与恢复属性。
- **红外接收（`hix5hd2-ir`）**：AArch64 EL3 引导阶段统一向 `SEC_ATTR_CTRL_0_REG`（`0xf8a80010`）写入 `0xfff9ffff` 开放非安全态红外寄存器访问；内核驱动按 7-bit 位宽（`GENMASK(6, 0)`）读取 64 深度的 `IR_DATAH` FIFO 计数，支持 LIRC 原始脉冲波形捕获及内置 `rc-gehua`（NECX）等按键映射。

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
- **以太网**：`GMAC1` 连接 `RTL8211F`（PHY ID `0x001cc916`，MDIO `3`），使用 `rgmii-id` 并启用 `CONFIG_HISTB_RTL8211F_LED=y` 驱动网口指示灯。

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

#### 以太网与存储
- **eMMC 与 TF**：8 GB（`8GME4R`，2+8 机型），8-bit 1.8 V HS400 模式（150 MHz，继承引导调谐）；启用 `sd0@9820000` TF 卡槽。
- **以太网**：`GMAC1` 连接 `RTL8211E`（PHY ID `0x001cc915`，MDIO `3`），使用 `rgmii-id`。

#### 板载 DVB-C 与 TS 总线
```text
CATV 同轴 -> RF 匹配/滤波 -> MxL214
                              |-- i2c2@8b12000 / 0x50 / 400 kHz
                              |-- GPIO8_3 低有效硬件复位
                              |-- channel 0..3 -> TSI0..TSI3
                              `-- serial-3wire -> serial-nosync-novalid（四路独立时钟，TSI0/2 正相、TSI1/3 反相，32-block + 硬件 PID 过滤）
```

#### 节点树
```text
/
|-- gpu@f9200000 --------------------------> Mali-T720 /dev/dri/renderD128
`-- soc@f0000000
    |-- mmc@9830000 -----------------------> /dev/mmcblk0（8 GB HS400 eMMC，p6 rootfs）
    |-- mmc@9820000 -----------------------> /dev/mmcblk1（TF 卡槽）
    |-- ethernet@9841000/ethernet-phy@3 ---> eth0（RTL8211E，rgmii-id）
    |-- i2c@8b12000/demodulator@50 --------> MxL214 -> TSI0..3（三线独立时钟）
    |-- demux@9c00000/input@0..3 ----------> /dev/dvb/adapter0..3（frontend0/demux0/dvr0/ca0..3）
    |-- display-controller@8cc0000 --------> /dev/dri/cardN（G0 + V0/V1）
    |-- hdmi@8ce0000 ----------------------> cardN-HDMI-A-1, /dev/cec0
    |-- audio-controller@8cd0000 ----------> /dev/snd/*（声卡 DVBIP-1004，Tianlai 带 GPIO9_0 低有效静音控制）
    |-- serial@8b18000 --------------------> /dev/ttySCI0（VCC 低有效，CD 低有效）
    |-- serial@8b00000 / serial@8b02000 ---> /dev/ttyAMA0, /dev/ttyAMA2
    |-- ir@8001000 ------------------------> /dev/lirc0
    |-- video-codec@8c30000 / vpss@8cb0000 -> VDEC + VPSS /dev/videoN, /dev/mediaN
    |-- video-codec@8c80000 ---------------> VENC /dev/videoN
    |-- jpeg-decoder@8c40000 / jpge@8c90000 -> JPGD + JPGE /dev/videoN
    |-- usb@9890000 -----------------------> 板载 05e3:0608 USB Hub + 外置 USB 口
    `-- usb@98b0000 -----------------------> 板载 MT7662T Wi-Fi/BT (0e8d:76a0) + USB 3.0
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
| **1004（2+8）** | `fastboot 0+1`、`bootargs 1+1`、`recovery 2+11`、`misc 13+1`、`hwconfig 14+4`、`rootfs 18+-` | `p1` boot、`p2` bootargs、`p3` **recovery（项目 11M 内核）**、`p4` misc、`p5` hwconfig（MAC `+160`、SN `+168`）、`p6` **rootfs** | `raw-only-full-overwrite`（纯 RAW） |

- **严禁跨型号混刷**：`full.xml`、`bootargs`、内核启动容器（`trustedcore.img` / `kernel.img` / `recovery.img`）及 `rootfs.img` 必须按对应型号严格匹配；1001 与 1006 虽分区偏移相同，但 DTB、eMMC 时序、复位引脚与声卡/SCI 配置不同，不可互换。
- **保留身份分区**：`full.xml` 默认不勾选 `deviceinfo` / `hwconfig`（`Sel="0"`），系统启动时由 `dvbip-hardware-identity.service` 自动读取本机原厂 MAC 与序列号并应用到 `eth0`。
- **HiTool 配置注意**：刷机前须在 HiTool 首选项中关闭“启动自动更新 bootargs 中分区表信息”（旧版无此项可忽略），并保持 `rootfs.img` 以 Android sparse 格式刷入（首次启动时由 `systemd-growfs-root.service` 自动无损扩展至剩余全部 eMMC 空间）。

---

## 固件、文件系统与软件交付

### 1. 系统环境与内核特性

- **基础用户态**：提供 Debian 12（`bookworm`）与 Debian 13.7（`trixie`）AArch64 无桌面标准环境，预装 `openssh-server`、`sudo`、`curl`、`dialog`、`cifs-utils`、`keyutils`；初始登录凭据为 `root / 123456`（首次接入网络后请及时修改）。
- **文件系统与容器支持**：内核内置或模块化支持本地文件系统（`ext4`、`vfat`、`msdos`、`exfat`、`ntfs3`、`xfs`、`btrfs`、`f2fs`、`squashfs`、`iso9660`、`udf`、`fuse`、`overlayfs`）及网络客户端文件系统（`NFSv3 / v4.0 / v4.1 / v4.2`、`CIFS / SMB2.1 / SMB3.1.1`，禁用不安全的旧版 SMB1/NFSv2），同时完整开启 Docker 容器所需的 Namespace、Cgroups 与 Netfilter/nftables 特性及常见 USB 串口/有线网卡/大容量存储驱动。
- **在线内核更新**：已安装 Debian 12/13 的设备可使用同型号内核升级包（目标分区：1001/1002/1006 为 `/dev/mmcblk0p4`，1004 为 `/dev/mmcblk0p3`），同步更新 `/lib/modules/<uname -r>/` 与 `/lib/firmware/` 并执行完整分区回读校验，切勿跨板或跨版本混用 `.ko` 模块。

### 2. 标准固件清单（`/lib/firmware/`）

完整板级交付在 `/lib/firmware/`（Debian merged-usr 下即 `/usr/lib/firmware/`）中统一部署 **12 份标准固件文件**：

| 类别 | 固件路径（相对于 `/lib/firmware/`） | 对应驱动与说明 |
| --- | --- | --- |
| **板载四路 DVB-C** | `mxl214/mxl214.fw` | `mxl214.ko` 板载四路有线前端主固件（通过 `request_firmware()` 标准加载，不再使用旧 `/kmod/` 路径） |
| **海思 HiVXE 硬解** | `hisilicon/histb-h264-cabac.bin`<br>`hisilicon/histb-hevc-cabac.bin`<br>`hisilicon/histb-avsp.bin` | `histb-vdec.ko` H.264/HEVC CABAC 表与 CV200 AVS/AVS+ SCD 固件（可由内核 `make hisilicon-firmware` 生成） |
| **无线网络与监管域** | `rtlwifi/rtl8192eu_nic.bin`<br>`regulatory.db`<br>`regulatory.db.p7s` | 板载/外置 Realtek Wi-Fi 固件与 Linux 802.11 无线频段签名数据库 |
| **外置 USB DTMB 外设** | `dvb-usb-lme2510c-dtmb-win-rom.bin`<br>`dvb-usb-lme2510c-dtmb-win-fw.bin`<br>`dvb-usb-lme2510c-dtmb-win-fw-pid0xd0.bin`<br>`dvb-usb-it9303-01.fw`<br>`dvb-fe-xc5000-1.6.114.fw` | 分别供外置 USB 电视棒 **LME2510C**（3 份）、**润祺达 AVL6381/IT9303**（1 份）与 **爱国者 DT5236/XC5000**（1 份）加载，直接置于 `/lib/firmware/` 根目录；**HD2312** 与 **SMIT iCast** 无需外部固件文件 |

> **板载硬件与外置 USB DTMB 的边界说明**：DVB-IP1001 / 1002 / 1004 / 1006 机身仅配备板载四路 DVB-C（有线电视）硬件；内核默认编入的五类 DTMB 驱动（`dvb-usb-hd2312`、`dvb-usb-lme2510-dtmb`、`dvb-usb-em28xx-dtmb`、`dvb-usb-avl6381`、`dvb-usb-icast`）与上述 5 份固件均**仅面向外接 USB 数字电视棒外围设备**，详细型号、USB ID 与驱动对照表见 [README 外围 USB DTMB 章节](README.md#外围-usb-dtmb--dvb-c-接收器支持非板载硬件)。

### 3. 扩展媒体组件

- 基础 rootfs 保持精简，不预装第三方流媒体或条件接收应用。
- 如需带 HiVXE V4L2 Request / VPSS DEI / M2M 扩展的 `hivxe-ffmpeg`、`hivxe-ffprobe`、`hivxe-top`、`hivxe-transcode` 及可选组件，请前往 [HiSilicon-Development/extra](https://github.com/HiSilicon-Development/extra) 获取；编解码矩阵与 Tvheadend 转码配置详见 [HiVXE 文档](https://github.com/HiSilicon-Development/HiVXE-2.0)。
