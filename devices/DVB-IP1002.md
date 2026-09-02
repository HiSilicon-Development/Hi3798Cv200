# DVB-IP1002 Hardware

本文记录 DVB-IP1002 **2 GiB DDR + 8 GiB eMMC** 版本的器件、板级连接、电气属性以及 Linux 6.12 下的节点对应关系。

## 硬件信息

| 硬件 | 型号或规格 | 板级连接 |
| --- | --- | --- |
| SoC | HiSilicon Hi3798CV200 (CA) | 主处理器，集成 CPU、GPU、VDP、HDMI TX/PHY、DVB demux、GMAC、USB、音频、SCI、IR 和 GPIO |
| CPU | 4 x Arm Cortex-A53 r0p4 | ARMv8-A，支持 AArch64 和 32-bit EL0 |
| GPU | Arm Mali-T720，3 个 shader core | SoC 片内 GPU，无外置 GPU |
| DDR | 2 GiB | 板载内存 |
| eMMC | `DG4008`，8 GB 标称容量 | 8-bit eMMC 总线，1.8 V I/O，MMC 5.1，HS400 150 MHz |
| TF | SD0，4-bit | 最高 50 MHz，3.3 V 信号，不切换到 1.8 V |
| DVB-C 前端 | MaxLinear MxL214C | 单芯片四通道 DVB-C，I2C1 控制，四组串行 TS 接 TSI0..TSI3 |
| 以太网 PHY | Realtek RTL8211F，PHY ID `0x001cc916` | GMAC1 通过 RGMII-ID 连接，MDIO 地址 3 |
| HDMI | Hi3798CV200 片内 HDMI TX/PHY | VDP 直接送入片内 HDMI，外接 HDMI Type-A 插座，无外置 HDMI bridge |
| 音频 | Hi3798CV200 AIAO + Tianlai codec | I2S/片内音频链路，提供模拟音频并参与 HDMI 音频输出 |
| USB hub | `05e3:0608` | USB 2.0 高速上行，4 个下游端口 |
| 无线模块 | Realtek RTL8192EU，USB `0bda:818b` | USB 2.0 高速连接，提供 Wi-Fi |
| 智能卡 | Hi3798CV200 SCI0 | DET、DATA、RST、CLK、PWREN 五组信号接智能卡接口 |
| 红外 | Hi3798CV200 IR 控制器 | 接板载红外接收线路 |
| 调试串口 | PL011 UART0、UART2 | TTL 串口，其中 UART0 为启动日志和系统控制台 |
| 外部接口 | HDMI Type-A、RJ45、USB、CATV 同轴、TF、智能卡、IR、模拟音频 | 各接口连接对应的 SoC 控制器或板载收发器件 |

## 以太网

```text
Hi3798CV200 GMAC1
  |-- RGMII TX/RX/CLK ----------> Realtek RTL8211F
  |-- MDC/MDIO ------------------> MDIO 地址 3
  `-- PHY reset -----------------> 低有效复位
                                  -> 千兆磁性器件 -> RJ45
```

| 层级 | 硬件信息 |
| --- | --- |
| MAC | Hi3798CV200 GMAC1，物理寄存器基址 `0xf9841000` |
| MAC-PHY 总线 | RGMII-ID，RTL8211F 内部提供 TX/RX 时钟延迟 |
| PHY | Realtek RTL8211F，PHY ID `0x001cc916` |
| PHY 管理 | MDC/MDIO，地址 3 |
| 线缆侧 | RTL8211F -> 千兆磁性器件 -> RJ45 |
| 指示灯 | RTL8211F LED0、LED1 输出链路状态，LED2 输出收发活动；10/100/1000 Mbit/s 均可指示链路 |

## DVB-C 前端和 TS 总线

### 射频、控制和复位

DVB-IP1002 使用一颗 MaxLinear MxL214C 提供四个可独立调谐的 DVB-C 前端。

四个 Linux frontend 来自同一颗 MxL214C 的四个通道。

```text
CATV 同轴座 -> RF 匹配/滤波 -> MaxLinear MxL214C
                                   |-- I2C1，7-bit 地址 0x50，400 kHz
                                   |-- GPIO7_4，低有效复位
                                   |-- channel 0 -> TSI0
                                   |-- channel 1 -> TSI1
                                   |-- channel 2 -> TSI2
                                   `-- channel 3 -> TSI3
```

| 信号 | 板级连接 |
| --- | --- |
| RF 输入 | 单个 CATV 同轴输入经板级匹配/滤波网络进入 MxL214C |
| 控制总线 | Hi3798CV200 I2C1，400 kHz，7-bit 地址 `0x50` |
| 复位 | `GPIO7_4`（legacy 全局编号 GPIO60），Linux GPIO chip 7 offset 4，低有效 |
| 前端 TS 格式 | `serial-4wire`，输出 D0、VALID、SYNC、CLK |
| SoC TS 格式 | `serial-nosync`，接收 D0、VALID、CLK，不单独采样 SYNC |
| TS 时钟 | demux 共享时钟选择 TSI2_CLK；TSI0..TSI2 各有独立 CLK，引出的 TSI3 未接独立 CLK，使用 TSI2_CLK |
| 采样边沿 | TSI0..TSI3 均使用反相采样时钟 |

### 四路 TS 电气映射

| MxL214C 通道 | SoC 输入 | SoC 接收信号 | 采样时钟 |
| --- | --- | --- | --- |
| channel 0 | TSI0 | D0、VALID、CLK | TSI0_CLK，反相 |
| channel 1 | TSI1 | D0、VALID、CLK | TSI1_CLK，反相 |
| channel 2 | TSI2 | D0、VALID、CLK | TSI2_CLK，反相 |
| channel 3 | TSI3 | D0、VALID | 共用 TSI2_CLK，反相 |

| TSI | Pinmux 偏移 | 信号顺序 | 复用功能 |
| --- | --- | --- | --- |
| TSI3 | `0x0cc/0x0d0` | VALID/D0 | MUX_M2；时钟使用 `0x0dc` 的 TSI2_CLK |
| TSI2 | `0x0d4/0x0d8/0x0dc` | VALID/D0/CLK | MUX_M2 |
| TSI1 | `0x0e8/0x0ec/0x0f0` | VALID/CLK/D0 | MUX_M2 |
| TSI0 | `0x0f4/0x0f8/0x0fc` | VALID/D0/CLK | MUX_M2 |

MxL214C 的 I2C1 SDA/SCL 分别使用 pinmux 偏移 `0x0e0/0x0e4`（MUX_M2）；复位使用偏移 `0x0c8`（MUX_M0）。

### Linux DVB 节点拓扑

设备树中的 MxL214C 位于 `/soc@f0000000/i2c@8b11000/demodulator@50`，四个输出端点逐一连接 Hi3798CV200 demux 的四个输入：

```text
i2c@8b11000 (SoC I2C1)
`-- demodulator@50 (MaxLinear MxL214C)
    |-- port@0 -> demux@9c00000/input@0 -> TSI0 -> adapter0
    |-- port@1 -> demux@9c00000/input@1 -> TSI1 -> adapter1
    |-- port@2 -> demux@9c00000/input@2 -> TSI2 -> adapter2
    `-- port@3 -> demux@9c00000/input@3 -> TSI3 -> adapter3
```

I2C 总线号由 Linux 探测顺序决定，因此用户态路径写作 `/dev/i2c-N`，MxL214C 对应 `/sys/bus/i2c/devices/N-0050`。

应通过设备树路径和设备名称识别总线，不应把 `N` 固定为某个数字。

每个前端对应一套独立 DVB 节点：

```text
/dev/dvb/
|-- adapter0/ -> MxL214C channel 0 -> TSI0
|-- adapter1/ -> MxL214C channel 1 -> TSI1
|-- adapter2/ -> MxL214C channel 2 -> TSI2
`-- adapter3/ -> MxL214C channel 3 -> TSI3

每个 adapterN/
|-- frontend0  调谐、锁定状态和信号参数
|-- demux0     硬件 PID/section 过滤
|-- dvr0       MPEG-TS 数据读取
`-- ca0..ca3   条件接收接口别名
```

四个 frontend 的 Linux 注册名称均为 `MaxLinear MxL214 DVB-C`。

## HDMI、显示和音频

### 显示链路

```text
DDR framebuffer
  -> Hi3798CV200 VDP G0/GP0
  -> Hi3798CV200 片内 HDMI TX/PHY
  -> TMDS data/clock -> HDMI Type-A
                        |-- HPD
                        |-- DDC/EDID
                        |-- CEC
                        `-- +5 V
```

| 模块 | 硬件信息 |
| --- | --- |
| VDP | 物理寄存器基址 `0xf8cc0000`，负责图层取数、合成和输出时序 |
| HDMI TX/PHY | 物理寄存器基址 `0xf8ce0000`，SoC 片内模块 |
| 外部接口 | HDMI Type-A；TMDS、HPD、DDC、CEC 和 +5 V |
| GPU | Mali-T720，物理寄存器基址 `0xf9200000` |
| 视频加速 | VPSS、VDEC、VENC 和 JPEG encoder 均为 SoC 片内模块 |

### 音频链路

```text
Hi3798CV200 AIAO -> Tianlai codec -> 模拟 L/R
                 `---------------> HDMI 音频链路
```

AIAO 寄存器基址为 `0xf8cd0000`，Tianlai 为 Hi3798CV200 片内 codec。

## USB 和无线

Hi3798CV200 提供一组 OHCI、一组 EHCI 和两组 xHCI 主控制器：

| 控制器 | 物理寄存器基址 | 连接 |
| --- | --- | --- |
| OHCI | `0xf9880000` | USB 1.1 companion host |
| EHCI | `0xf9890000` | USB 2.0 host；板载 `05e3:0608` hub 接在此路径 |
| xHCI0 | `0xf98a0000` | USB 2.0/USB 3.x root hub |
| xHCI1 | `0xf98b0000` | USB 2.0/USB 3.x root hub；RTL8192EU 使用其 USB 2.0 路径 |

```text
EHCI f9890000 -> USB 2.0 hub 05e3:0608 -> 4 x downstream port
xHCI f98b0000 -> Realtek RTL8192EU 0bda:818b -> Wi-Fi
```

外部接口为 2 x USB 2.0 和 1 x USB 3.x。USB 的 `usbN`、`N-M` 编号由控制器探测顺序生成，不属于固定板级编号。

## 智能卡、红外、串口和 GPIO

| 功能 | 控制器和连接 | Linux 接口 |
| --- | --- | --- |
| 智能卡 | SCI0，基址 `0xf8b18000`；DET/DATA/RST/CLK/PWREN，DET 高有效，VCC/PWREN 低有效 | `/dev/ttySCI0` |
| 红外 | IR，基址 `0xf8001000` | `/dev/lirc0`、`/dev/input/eventN`、`/sys/class/rc/rc0` |
| 调试串口 | UART0，基址 `0xf8b00000` | `/dev/ttyAMA0`，115200 8N1，启动和系统控制台 |
| 辅助串口 | UART2，基址 `0xf8b02000` | `/dev/ttyAMA2` |
| GPIO | 13 组 PL061 GPIO 控制器 | `/dev/gpiochip0`..`/dev/gpiochip12` |
| 硬件随机数 | Hi3798CV200 RNG，基址 `0xf8005200` | `/dev/hwrng` |
| 看门狗 | SP805，基址 `0xf8a2c000` | `/dev/watchdog0` |

`eventN` 等带数字后缀的 input 节点可能随启动顺序变化，应通过 `/sys/class/input/eventN/device/name` 查找对应的红外设备。

## Linux 6.12 节点总览

### 设备树、platform device 和用户态接口

| 硬件 | 设备树节点 | Linux platform device | 用户态接口 |
| --- | --- | --- | --- |
| CPUFreq | `/cpus/cpu@0`..`cpu@3` | `cpufreq-dt` | `/sys/devices/system/cpu/cpufreq/policy0` |
| CPUIdle | `/cpus/cpu@0`..`cpu@3` | Arm WFI | `/sys/devices/system/cpu/cpuN/cpuidle/state0` |
| 温度传感器 | `/soc@f0000000/temperature-sensor@8a23028` | `f8a23028.temperature-sensor` | `/sys/class/thermal/thermal_zone0` |
| eMMC | `/soc@f0000000/mmc@9830000` | `f9830000.mmc` | `/dev/mmcblk0`、`mmcblk0p1`..`p8`、`mmcblk0boot0/1`、`mmcblk0rpmb` |
| TF/SD | `/soc@f0000000/mmc@9820000` | `f9820000.mmc` | 插卡后为 `/dev/mmcblk1` |
| Ethernet | `/soc@f0000000/ethernet@9841000` | `f9841000.ethernet` | `eth0` |
| I2C1 | `/soc@f0000000/i2c@8b11000` | `f8b11000.i2c` | `/dev/i2c-N` |
| MxL214C | `/soc@f0000000/i2c@8b11000/demodulator@50` | `N-0050` | `/dev/dvb/adapter0`..`adapter3` |
| DVB demux | `/soc@f0000000/demux@9c00000` | `f9c00000.demux` | 每个 adapter 下的 `demux0`、`dvr0`、`ca0..ca3` |
| VDP | `/soc@f0000000/display-controller@8cc0000` | `f8cc0000.display-controller` | `/dev/dri/cardN`；fbdev 启用时有 `/dev/fb0` |
| HDMI | `/soc@f0000000/hdmi@8ce0000` | `f8ce0000.hdmi` | `/sys/class/drm/cardN-HDMI-A-1` |
| GPU | `/soc@f0000000/gpu@9200000` | `f9200000.gpu` | `/dev/dri/cardN`、`/dev/dri/renderD128` |
| AIAO | `/soc@f0000000/audio-controller@8cd0000` | `f8cd0000.audio-controller` | `/dev/snd/controlC0`、`/dev/snd/pcmC0D0p` |
| SCI0 | `/soc@f0000000/serial@8b18000` | `f8b18000.serial` | `/dev/ttySCI0` |
| UART0 | `/soc@f0000000/serial@8b00000` | `f8b00000.serial` | `/dev/ttyAMA0` |
| UART2 | `/soc@f0000000/serial@8b02000` | `f8b02000.serial` | `/dev/ttyAMA2` |
| IR | `/soc@f0000000/ir@8001000` | `f8001000.ir` | `/dev/lirc0`、`/dev/input/eventN` |
| VDEC | `/soc@f0000000/video-codec@8c30000` | `f8c30000.video-codec` | `/dev/videoN` |
| VPSS | `/soc@f0000000/video-post-processing@8cb0000` | `f8cb0000.video-post-processing` | 由 VDEC 媒体链路使用 |
| VENC | `/soc@f0000000/video-codec@8c80000` | `f8c80000.video-codec` | `/dev/videoN` |
| JPEG encoder | `/soc@f0000000/jpeg-encoder@8c90000` | `f8c90000.jpeg-encoder` | `/dev/videoN` |
| OHCI | `/soc@f0000000/usb@9880000` | `f9880000.usb` | USB root hub/sysfs |
| EHCI | `/soc@f0000000/usb@9890000` | `f9890000.usb` | USB root hub/sysfs |
| xHCI0 | `/soc@f0000000/usb@98a0000` | `f98a0000.usb` | USB 2.0/3.x root hub/sysfs |
| xHCI1 | `/soc@f0000000/usb@98b0000` | `f98b0000.usb` | USB 2.0/3.x root hub/sysfs |
| USB hub | EHCI 下游设备 | `05e3:0608` | `/sys/bus/usb/devices/N-M` |
| RTL8192EU | xHCI1 USB 2.0 下游设备 | `0bda:818b` | USB sysfs、无线网络接口 |

### 节点树

```text
/soc@f0000000
|-- mmc@9830000 ------------------------> /dev/mmcblk0
|-- mmc@9820000 ------------------------> /dev/mmcblk1（插卡后）
|-- ethernet@9841000/ethernet-phy@3 ---> eth0
|-- i2c@8b11000 ------------------------> /dev/i2c-N
|   `-- demodulator@50 -----------------> /dev/dvb/adapter0..3
|-- demux@9c00000/input@0..3 ----------> demux0/dvr0/ca0..ca3
|-- display-controller@8cc0000 ---------> /dev/dri/cardN
|-- hdmi@8ce0000 <--- VDP graph link ---> cardN-HDMI-A-1
|-- gpu@9200000 ------------------------> /dev/dri/cardN, renderD128
|-- audio-controller@8cd0000 -----------> /dev/snd/*
|-- serial@8b00000 ---------------------> /dev/ttyAMA0
|-- serial@8b02000 ---------------------> /dev/ttyAMA2
|-- serial@8b18000 ---------------------> /dev/ttySCI0
|-- ir@8001000 -------------------------> rc0, lirc0, input eventN
|-- video-codec@8c30000 ---------------> VDEC /dev/videoN
|-- video-codec@8c80000 ---------------> VENC /dev/videoN
|-- jpeg-encoder@8c90000 --------------> JPGE /dev/videoN
`-- usb@9880000/9890000/98a0000/98b0000 -> USB root hubs
```

`cardN`、`videoN`、`eventN`、`i2c-N` 和 USB 设备号是 Linux 动态编号。程序应优先使用 sysfs 的设备名称、`by-path` 链接或设备树路径识别硬件。
