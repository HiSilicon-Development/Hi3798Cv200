# DVB-IP100x 产品总览

本文是 DVB-IP1001、DVB-IP1002、DVB-IP1004 和 DVB-IP1006 的统一参数、节点拓扑、HiTool 布局和适配状态入口。

- [DVB-IP1001](#dvb-ip1001)
- [DVB-IP1002](#dvb-ip1002)
- [DVB-IP1004](#dvb-ip1004)
- [DVB-IP1006](#dvb-ip1006)

## 公共平台

| 项目 | 公共参数 |
| --- | --- |
| SoC | HiSilicon Hi3798CV200（CA） |
| CPU | 4 × Arm Cortex-A53 r0p4，ARMv8-A/AArch64 |
| GPU | Arm Mali-T720，3 个 shader core |
| 内存 | 2 GiB DDR |
| 视频 | VDEC、VPSS、VENC、JPGE；HDMI 由片内 VDP + HDMI TX/PHY 提供 |
| 音频 | Hi3798CV200 AIAO/I2S + Tianlai codec |
| 智能卡 | SoC SCI0，DET/DATA/RST/CLK/PWREN |
| 串口 | UART0 为启动日志/控制台，UART2 为辅助串口，默认 115200 8N1 |
| USB 主控 | OHCI、EHCI、两组 xHCI |
| Linux 基线 | Linux 6.12.111-DVBIP-REBASE |

## 型号差异

| 型号 | 存储与 TF | DVB-C 前端与 TS | 以太网 | USB / 无线 | 当前状态 |
| --- | --- | --- | --- | --- | --- |
| 1001 | 8GTF4R，8 GB；无 TF | MxL214C，I2C1 `0x50`，GPIO7_4 低有效；serial-4wire/serial-nosync；TSI3 共用 TSI2_CLK | RTL8211E，PHY ID `0x001cc915`，RGMII，MDIO 3 | 1 × USB 3.x + 2 × USB 2.0 | DTS、内核和 rootfs 已适配；板级锁频、显示、音频等仍按专用文档验收 |
| 1002 | DG4008，8 GB；有 TF | MxL214C，I2C1 `0x50`，GPIO7_4 低有效；serial-4wire/serial-nosync；TSI3 共用 TSI2_CLK | RTL8211F，PHY ID `0x001cc916`，RGMII-ID，MDIO 3 | `05e3:0608` USB hub；RTL8192EU `0bda:818b` | DTS、内核、rootfs、HiVXE 和 DTMB 交付已适配 |
| 1004 | 8GME4R，8 GB；有 TF；当前只支持 2+8 | MxL214，I2C2 控制器 `i2c@8b12000`，`0x50`，GPIO8_3 低有效；serial-3wire/serial-nosync-novalid；四路独立时钟 | RTL8211E，PHY ID `0x001cc915`，RGMII-ID，MDIO 3 | `05e3:0608` USB hub；当前记录为 MT7662T `0e8d:76a0` | 2+8 DTS、内核和 rootfs 已适配；2+4 不属于当前交付 |
| 1006 | H8G4a；8 GB；无 TF | MxL214C，I2C1 `i2c@8b11000`、`0x50`、force-polling、GPIO9_7 低有效、serial-4wire，四路 endpoint 全部接入 demux | GMAC1/RGMII，DTS 已配置 reset delay；PHY 细节按实机记录 | OHCI/EHCI/xHCI、HD2312/LME2510C 模块和 rootfs 已交付 | 1006 已完成 DTS、内核、trustedcore、rootfs 和 DTMB 适配；DVB-C 锁频、PHY 型号和外接设备属于后续实机结果，不再写成“未适配” |

## 设备树、platform device 和用户态节点

| 功能 | 设备树节点 | Linux platform / 驱动 | 用户态节点 |
| --- | --- | --- | --- |
| CPUFreq | `/cpus/cpu@0..3` | `cpufreq-dt` | `/sys/devices/system/cpu/cpufreq/policy0` |
| CPUIdle | `/cpus/cpu@0..3` | Arm WFI CPUIdle | `/sys/devices/system/cpu/cpuN/cpuidle/state0` |
| 温度 / watchdog | `temperature-sensor@8a23028`、SP805 | `f8a23028.temperature-sensor`、SP805 | `/sys/class/thermal/thermal_zone0`、`/dev/watchdog0` |
| eMMC | `mmc@9830000` | `f9830000.mmc` | `/dev/mmcblk0`、`mmcblk0p*`、boot0/1 |
| TF/SD | `mmc@9820000`（1002/1004）；1001/1006 disabled | `f9820000.mmc` | 插卡后通常为 `/dev/mmcblk1` |
| Ethernet | `ethernet@9841000`、`ethernet-phy@3` | `f9841000.ethernet` / `gmac1` | `eth0` |
| DVB-C I2C | 1001/1002 `i2c@8b11000`；1004 `i2c@8b12000`；1006 `i2c@8b11000` | `f8b11000.i2c` 或 `f8b12000.i2c`；1006 force-polling | `/dev/i2c-N`，MxL214 地址 `0x50` |
| DVB-C / demux | `demodulator@50` → `demux@9c00000/input@0..3` | MxL214 + `histb-dmx` | `/dev/dvb/adapterN/frontend0`、`demux0`、`dvr0` |
| VDP / HDMI | `display-controller@8cc0000`、`hdmi@8ce0000` | `histb-vdp`、`histb-hdmi` | `/dev/dri/cardN`、`cardN-HDMI-A-1` |
| GPU | `gpu@9200000` | Panfrost | `/dev/dri/renderD128` |
| 音频 | `audio-controller@8cd0000` | AIAO / Tianlai ALSA | `/dev/snd/*` |
| SCI / UART / IR | `serial@8b18000`、`serial@8b00000`、`serial@8b02000`、`ir@8001000` | SCI0、PL011、IR（1006 按 DTS disabled） | `/dev/ttySCI0`、`/dev/ttyAMA0`、`/dev/ttyAMA2`、`/dev/lirc0` |
| VDEC / VPSS | `video-codec@8c30000`、`video-post-processing@8cb0000` | `histb-vdec`、`histb-vpss` | `/dev/videoN`、`/dev/mediaN` |
| VENC / JPGE | `video-codec@8c80000`、`jpeg-encoder@8c90000` | `histb-venc`、`histb-jpge` | `/dev/videoN` |
| USB | `usb@9880000/9890000/98a0000/98b0000` | OHCI/EHCI/xHCI | USB sysfs、`/dev/bus/usb` |

节点编号由 Linux 探测顺序决定。脚本应按 sysfs 设备名、设备树路径或 `by-path` 链接识别硬件，不把 `adapterN`、`videoN`、`cardN`、`i2c-N` 或 USB 设备号写成永久编号。

## 板级链路、DVB-C 总线和节点树

下面保留四份型号资料中最容易重复、但实际不能混用的部分：以太网、DVB-C/TS 总线、设备树拓扑和用户态节点。

### DVB-IP1001

#### 以太网

```text
GMAC1 / ethernet@9841000
  |-- MDC/MDIO -> RTL8211E，PHY address 3，ID 0x001cc915
  |-- RGMII
  `-- 千兆磁性器件 -> RJ45 -> eth0
```

#### DVB-C 前端和 TS 总线

```text
CATV 同轴 -> RF 匹配/滤波 -> MxL214C
                              |-- i2c@8b11000 / I2C1 / 0x50 / 400 kHz
                              |-- GPIO7_4 低有效复位
                              |-- channel 0..3 -> TSI0..TSI3
                              `-- serial-4wire -> serial-nosync
```

TSI0、TSI1、TSI2 使用各自时钟；TSI3 使用 TSI2_CLK；四路均反相采样。

#### 节点树

```text
/soc@f0000000
|-- mmc@9830000 ------------------------> /dev/mmcblk0（板载 eMMC）
|-- ethernet@9841000/ethernet-phy@3 ---> eth0
|-- i2c@8b11000
|   `-- demodulator@50 ---------------> MxL214C -> TSI0..3
|-- demux@9c00000/input@0..3 ----------> /dev/dvb/adapter0..3
|-- display-controller@8cc0000 ---------> /dev/dri/cardN
|-- hdmi@8ce0000 -----------------------> cardN-HDMI-A-1
|-- audio-controller@8cd0000 -----------> /dev/snd/*
|-- serial@8b18000 ---------------------> /dev/ttySCI0
|-- serial@8b00000 ---------------------> /dev/ttyAMA0
|-- serial@8b02000 ---------------------> /dev/ttyAMA2
|-- video-codec@8c30000 ---------------> VDEC /dev/videoN
|-- video-post-processing@8cb0000 -----> VPSS / VDEC media chain
|-- video-codec@8c80000 ---------------> VENC /dev/videoN
|-- jpeg-encoder@8c90000 --------------> JPGE /dev/videoN
`-- usb@9880000/9890000/98a0000/98b0000 -> USB root hubs
```

1001 的 `sd0`/TF 路径关闭；REBASE 也没有把 IR 写成可用产品功能。

### DVB-IP1002

#### 以太网

```text
GMAC1 / ethernet@9841000
  |-- MDC/MDIO -> RTL8211F，PHY address 3，ID 0x001cc916
  |-- RGMII-ID
  `-- 千兆磁性器件 -> RJ45 -> eth0
```

#### DVB-C 前端和 TS 总线

```text
CATV 同轴 -> RF 匹配/滤波 -> MxL214C
                              |-- i2c@8b11000 / I2C1 / 0x50 / 400 kHz
                              |-- GPIO7_4 低有效复位
                              |-- channel 0..3 -> TSI0..TSI3
                              `-- serial-4wire -> serial-nosync
```

TSI0、TSI1、TSI2 使用各自时钟；TSI3 使用 TSI2_CLK；四路均反相采样。

#### 节点树

```text
/soc@f0000000
|-- mmc@9830000 ------------------------> /dev/mmcblk0（8 GiB eMMC）
|-- mmc@9820000 ------------------------> /dev/mmcblk1（TF，插卡后）
|-- ethernet@9841000/ethernet-phy@3 ---> eth0
|-- i2c@8b11000
|   `-- demodulator@50 ---------------> MxL214C -> TSI0..3
|-- demux@9c00000/input@0..3 ----------> /dev/dvb/adapter0..3
|-- display-controller@8cc0000 ---------> /dev/dri/cardN
|-- hdmi@8ce0000 -----------------------> cardN-HDMI-A-1
|-- gpu@9200000 ------------------------> /dev/dri/renderD128
|-- audio-controller@8cd0000 -----------> /dev/snd/*
|-- serial@8b18000 ---------------------> /dev/ttySCI0
|-- serial@8b00000 ---------------------> /dev/ttyAMA0
|-- serial@8b02000 ---------------------> /dev/ttyAMA2
|-- video-codec@8c30000 ---------------> VDEC /dev/videoN
|-- video-post-processing@8cb0000 -----> VPSS / VDEC media chain
|-- video-codec@8c80000 ---------------> VENC /dev/videoN
|-- jpeg-encoder@8c90000 --------------> JPGE /dev/videoN
|-- usb@9890000 -----------------------> 05e3:0608 USB hub
|-- usb@98b0000 -----------------------> RTL8192EU 0bda:818b
`-- ir@8001000 ------------------------> /dev/lirc0 / input eventN
```

### DVB-IP1004

#### 以太网

```text
GMAC1 / ethernet@9841000
  |-- MDC/MDIO -> RTL8211E，PHY address 3，ID 0x001cc915
  |-- RGMII-ID
  `-- 千兆磁性器件 -> RJ45 -> eth0
```

#### DVB-C 前端和 TS 总线

```text
CATV 同轴 -> RF 匹配/滤波 -> MxL214
                              |-- i2c@8b12000 / I2C2 / 0x50 / 400 kHz
                              |-- GPIO8_3 低有效复位
                              |-- channel 0..3 -> TSI0..TSI3
                              `-- serial-3wire -> serial-nosync-novalid
```

四路使用独立 TS 时钟；channel 0/2 非反相，channel 1/3 反相。当前 Linux
把该控制器注册为 `/dev/i2c-1`，`/dev/i2c-0` 是 HDMI SCDC 内部总线。

#### 节点树

```text
/soc@f0000000
|-- mmc@9830000 ------------------------> /dev/mmcblk0（8 GiB eMMC）
|-- mmc@9820000 ------------------------> /dev/mmcblk1（TF，插卡后）
|-- ethernet@9841000/ethernet-phy@3 ---> eth0
|-- i2c@8b12000
|   `-- demodulator@50 ---------------> MxL214 -> TSI0..3
|-- demux@9c00000/input@0..3 ----------> /dev/dvb/adapter0..3
|-- display-controller@8cc0000 ---------> /dev/dri/card0
|-- hdmi@8ce0000 -----------------------> card0-HDMI-A-1 /dev/i2c-0
|-- gpu@9200000 ------------------------> /dev/dri/card1, renderD128
|-- audio-controller@8cd0000 -----------> /dev/snd/*
|-- serial@8b18000 ---------------------> /dev/ttySCI0
|-- serial@8b00000 ---------------------> /dev/ttyAMA0
|-- serial@8b02000 ---------------------> /dev/ttyAMA2
|-- video-codec@8c30000 ---------------> VDEC /dev/videoN
|-- video-post-processing@8cb0000 -----> VPSS / VDEC media chain
|-- video-codec@8c80000 ---------------> VENC /dev/videoN
|-- jpeg-encoder@8c90000 --------------> JPGE /dev/videoN
|-- usb@9890000 -----------------------> 05e3:0608 USB hub
|-- usb@98b0000 -----------------------> MT7662T 0e8d:76a0
`-- ir@8001000 ------------------------> /dev/lirc0 / input eventN
```

当前交付只针对 2+8；2+4 的器件和板级连接不从 2+8 配置推导。

### DVB-IP1006

#### 以太网和 eMMC

```text
mmc@9830000 ---------------------------> H8G4a 候选，8-bit / 1.8 V / HS400 / 150 MHz
ethernet@9841000 / gmac1 --------------> RGMII -> eth0
                                         reset delay 已写入 DTS
```

#### DVB-C 前端和 TS 总线

```text
CATV 同轴 -> MxL214C
              |-- i2c@8b11000 / I2C1 / 0x50 / force-polling
              |-- GPIO9_7 低有效复位
              |-- serial-4wire
              `-- port@0..3 -> demux@9c00000/input@0..3 -> TSI0..TSI3
```

#### 节点树

```text
/soc@f0000000
|-- mmc@9830000 ------------------------> /dev/mmcblk0，p5 rootfs
|-- ethernet@9841000 -------------------> eth0
|-- i2c@8b11000（force-polling）
|   `-- demodulator@50 ----------------> MxL214C -> port@0..3
|-- demux@9c00000/input@0..3 ----------> /dev/dvb/adapter0..3
|-- display-controller@8cc0000 ---------> /dev/dri/cardN
|-- hdmi@8ce0000 -----------------------> cardN-HDMI-A-1
|-- audio-controller@8cd0000 -----------> /dev/snd/*，声卡名 DVBIP-1006
|-- serial@8b18000 ---------------------> /dev/ttySCI0
|-- serial@8b00000 ---------------------> /dev/ttyAMA0
|-- serial@8b02000 ---------------------> /dev/ttyAMA2
|-- video-codec@8c30000 ---------------> VDEC /dev/videoN
|-- video-post-processing@8cb0000 -----> VPSS / VDEC media chain
|-- video-codec@8c80000 ---------------> VENC /dev/videoN
|-- jpeg-encoder@8c90000 --------------> JPGE /dev/videoN
|-- usb@9880000/9890000/98a0000/98b0000 -> USB host controllers
`-- ir@8001000 ------------------------> 按当前 DTS disabled
```

## HiTool 物理布局

| 型号 | 物理布局（MiB） | Linux 重点 |
| --- | --- | --- |
| 1001 / 1006 | `fastboot 0+4`、`bootargs 4+4`、`deviceinfo 8+152`、`signed kernel 160+40`、`misc 200+20`、`trustedcore 220+10`、`rootfs 230+` | Linux 视图为 p1 fastboot、p2 bootargs、p3 deviceinfo、p4 trustedcore、p5 rootfs |
| 1002 | `fastboot 0+4`、`bootargs 4+4`、`hwconfig 8+4`、`kernel 12+16`、`rootfs 28+` | Linux 视图为 p1 fastboot、p2 bootargs、p3 identity、p4 kernel、p5 rootfs |
| 1004 | `fastboot 0+1`、`bootargs 1+1`、`recovery 2+11`、`misc 13+1`、`hwconfig 14+4`、`rootfs 18+` | Linux 视图为 p1 fastboot、p2 bootargs、p3 recovery、p4 misc、p5 identity、p6 rootfs；其中 recovery 是 11M 内核 |

不要跨型号混用 `full.xml`、bootargs、kernel/recovery/trustedcore、rootfs 或 identity 文件。rootfs 是 Android sparse ext4，必须通过 HiTool 的 sparse-aware 路径写入。

## 软件交付

- 主 rootfs 使用匹配 release 的 Linux 模块；不从旧型号或旧 release 复制 `.ko`。
- USB DTMB 只保留 HD2312 和 LME2510C/LGS8GL5 两类模块。
- LME2510C 固件文件为：
  - `dvb-usb-lme2510c-dtmb-win-rom.bin`
  - `dvb-usb-lme2510c-dtmb-win-fw.bin`
  - `dvb-usb-lme2510c-dtmb-win-fw-pid0xd0.bin`
- 三份 LME 固件直接放在 rootfs 的 `/lib/firmware/dvb-cn-usb/`，不单独打 USB modules ZIP。
- 主 rootfs 不内置 OSCam、FFmpeg；需要请前往本项目的 [extra](https://github.com/HiSilicon-Development/extra) 仓库查看。
- 当前公共 rootfs 的初始登录是 `root / 123456`；首次登录后应立即修改密码。

## 验收边界

刷机后应分别确认：

- 冷启动、UART0、eMMC/TF、SSH、网络和温度；
- 四路 DVB-C 调谐、锁定、TS 连续性；
- HD2312、LME2510C 固件加载、DTMB 锁定和持续 TS；
- HDMI/EDID/热插拔、1080i/4K、模拟/HDMI 音频；
- SCI、IR、USB、无线和板级 PHY；
- HiVXE 解码、编码、转码、温度、CMA 和长时间稳定性。
