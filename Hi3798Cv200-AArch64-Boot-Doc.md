# Hi3798CV200 AArch32 → AArch64 启动架构

适用版本：DVB-IP1001／1002／1004／1006 的 Linux 6.12.118。原厂 AArch32 Fastboot 加载板型容器，Linux 在 AArch64 non-secure EL2 启动，次级核使用 PSCI，电源管理使用 CVPM-2。

板级参数、完整分区表及系统节点见 [DVB-IP100x 产品与硬件总览](DVBIP-100x-INTRODUCTION.md)。

---

## 原厂 Fastboot 与各机型内核槽位

保留本机的原厂 Fastboot 和 `bootargs`。从下表选用对应的内核容器。

| 型号 | 项目 Linux 启动容器 | eMMC 物理起点 / 容量 | Linux 分区设备 | 容器与引导约束 |
| --- | --- | --- | --- | --- |
| **DVB-IP1001** | `trustedcore.img` | 220 MiB / 10 MiB | `/dev/mmcblk0p4` | 保留原厂 Fastboot、bootargs 及 160 MiB 处的 40 MiB 签名 `kernel.img`；项目 Linux 固定从 `trustedcore` 引导 |
| **DVB-IP1002** | `kernel.p4.img` | 12 MiB / 16 MiB | `/dev/mmcblk0p4` | 写入带 `0x20000` 前缀与 Android Boot Header 的完整容器 |
| **DVB-IP1004** | `recovery.img` | 2 MiB / 11 MiB | `/dev/mmcblk0p3` | 配合 `misc` 分区引导标志，从 `recovery` 槽位读取 11 MiB legacy `uImage` 容器直接进入 Debian/Linux |
| **DVB-IP1006** | `trustedcore.img` | 220 MiB / 10 MiB | `/dev/mmcblk0p4` | 保留本型号原厂 Fastboot、bootargs 及 40 MiB 签名 `kernel.img`；项目 Linux 固定从 `trustedcore` 引导 |

1004 按板载射频芯片选 `recovery.img`。用本机型的 `full.xml` 与 `SHA256SUMS` 核对容器、写入偏移和长度。

---

## AArch32 → AArch64 主启动链

```text
上电或硬复位
  -> Hi3798CV200 BootROM
  -> 原厂 AArch32 Fastboot（含原厂解压前置层与安全校验）
  -> 按 bootargs/bootcmd 读取本型号启动容器并完成头部/完整性检查
  -> gzip 解压 legacy uImage（IH_ARCH_ARM）至 0x12000000 并跳转执行 AArch32 bundle
  -> AArch32 跳板复制 Image (0x02000000)、DTB (0x10000000)、EL3 entry (0x11000000) 并刷写缓存
  -> 初始化系统计数器，写入 CPU0 RVBAR 与 64 位复位模式，通过 CP15 RMR 触发 warm reset
  -> CPU0 在 AArch64 EL3 初始化计数器、GICv2 和驻留监控器
  -> ERET 切换至 non-secure EL2h，清理 EL2 状态后以 x0 = 0x10000000 跳入 0x02000000
  -> Linux 通过 PSCI 启动次级核、挂载 rootfs、执行 /sbin/init
```

### 1. AArch32 跳板（Bundle 前置层）

AArch32 bundle 是将 **ARM32 跳板、AArch64 EL3/EL2 entry、本机 DTB 与 AArch64 Linux `Image`** 链接在一起的单体引导载荷：

1. **屏蔽中断并输出串口标记**：执行 `cpsid if`，保存入口 `CPSR / SCTLR / ACTLR`，向 `UART0`（`0xf8b00000`）打印 `[A32] unpacking bundled ARM64 payload`。
2. **搬运载荷到目标物理地址**：将内嵌的 Linux `Image` 复制到 `0x02000000`、本机 DTB 复制到 `0x10000000`、AArch64 entry 复制到 `0x11000000`。
3. **维护缓存一致性**：按 `CTR.DminLine` 将三段载荷的数据缓存清理至一致性点（PoC），失效指令缓存和分支预测器，执行 `DSB SY` / `ISB`。
4. **配置系统计数器与 64 位复位向量**：打印 `[A32] requesting AArch64 warm reset`；向海思系统计数器写入 `0xf8bb0020 = 1`（`SYS_COUNTER_ENABLE`）与 `0xf8bb0000 = 0x103`（`SYS_COUNTER_CTRL`）；将 `PERI_CPU_RVBARADDR`（`0xf8a80034`）设为 `0x11000000`，置位 `PERI_CPU_AARCH_MODE`（`0xf8a80030`）bit 0。
5. **请求 AArch64 热复位**：置位 CP15 Reset Management Register（`RMR`，`AA64 | RR = 0x3`）请求 warm reset，并在 `WFI` 循环中等待复位生效。

### 2. AArch64 EL3 → Non-Secure EL2 交接与内核入口

CPU0 热复位后，从 `0x11000000` 的 AArch64 EL3 入口继续执行：

- **开放红外寄存器访问**：向 `SEC_ATTR_CTRL_0_REG`（`0xf8a80010`）写入 `0xfff9ffff`，执行 `DSB SY` / `ISB`，由 non-secure Linux 驱动 `hix5hd2-ir`。
- **多核缓存一致性与定时器基线**：置位 Cortex-A53 `CPUACTLR_EL1.SMPEN`（bit 6）加入硬件缓存一致性域；清零继承的物理/虚拟定时器控制位，设置 `CNTFRQ_EL0 = 24000000`（24 MHz 架构定时器）。
- **GICv2 非安全移交**：将 GICD（`0xf1001000`）全部 SPI 及每核 GICC（`0xf1002000`）SGI/PPI 的 `IGROUPR` 设为 Group 1（non-secure），清除残留挂起中断，开启 SGI0 并配置 `GICC_CTLR = 0x1e7`、`GICC_PMR = 0x80`。
- **次级核**：Linux 发起 PSCI `CPU_ON`，EL3 监控器负责次级核上电、复位与进入。
- **交接到 EL2**：EL3 执行 `ERET`，保留 SMC 接口。EL2 关闭 MMU 和缓存后进入 Linux。

CPU0 进入 Linux 时的寄存器状态（[Booting AArch64 Linux](https://docs.kernel.org/arch/arm64/booting.html)）：

```text
CPU0 状态 = AArch64 non-secure EL2h（MMU/D-cache 关闭，DAIF 屏蔽）
x0        = 0x10000000   本型号 DTB 物理地址
x1..x3    = 0
PC        = 0x02000000   AArch64 Linux Image 入口
```

### 3. 次级核与驻留电源管理

`hi3798cv200-dvbip.dtsi` 指定 `enable-method = "psci"`，Linux 通过 SMC 调用 EL3 监控器。将 `smp-firmware@11000000` 设置为 64 KiB 的 `no-map` 保留区。

1004 R858 DTB 设置 `hisilicon,suspend-disabled`。PSCI、CVPM 与 C51 保持启用。

---

## 启动期与运行期内存布局

| 物理地址 / 范围 | 用途 | 尺寸上限与协同约束 |
| --- | --- | --- |
| `0x02000000..0x0fffffff` | AArch64 Linux `Image` 运行入口 | 最大 224 MiB，必须在 DTB 起点 `0x10000000` 前结束 |
| `0x10000000..0x10ffffff` | 板级 DTB（通过 `x0` 传入内核） | 最大 16 MiB，必须在 `0x11000000` 前结束 |
| `0x11000000..0x1100ffff` | `smp-firmware@11000000` 保留区 | 64 KiB `no-map`，存放 EL3/EL2 驻留代码及数据 |
| `0x12000000`（启动期） | AArch32 bundle 解压与入口地址 | legacy `uImage` 的 `load / entry = 0x12000000`，解压后体积上限 16 MiB；内核启动完成后该区间由 `decoder-capture@12000000`（160 MiB `reusable` CMA）安全复用 |

调整引导地址时，同步更新 ARM32 跳板、AArch64 入口、DTB 保留区和容器参数。

---

## 启动容器封装与刷写规范

### 1. 各型号启动容器内部结构

所有机型的内层载荷均为 **ARM32 架构（`IH_ARCH_ARM`）gzip 压缩的 legacy `uImage`**（`load = 0x12000000, entry = 0x12000000`），外层按各机型原厂 Fastboot 的解析规则封装并补齐到固定分区容量：

| 型号 | 容器文件与总容量 | 容器字节布局 |
| --- | --- | --- |
| **1001 / 1006** | `trustedcore.img`（10 MiB） | `0x0` 处为 Android Boot Header（`ANDROID!`，`page_size = 0x4000`，含 SHA-1 ID）；`0x4000` 处为 legacy `uImage`；尾部零填充至 10 MiB |
| **1002** | `kernel.p4.img`（16 MiB） | 开头保留 128 KiB；`0x20000` 为 Android Boot Header（`page_size = 0x4000`），`0x24000` 为 legacy `uImage`；尾部填充至 16 MiB |
| **1004（MxL214 / Hi3130+R858）** | 各自的 `recovery.img`（11 MiB） | 从 `0x0` 存放 legacy `uImage`，零填充到 11 MiB；按 RF 芯片选对应容器 |

```text
型号专属容器（trustedcore.img / kernel.p4.img / recovery.img）
  `-- legacy uImage（IH_ARCH_ARM，gzip，load/entry = 0x12000000）
        `-- AArch32 bundle
              |-- ARM32 搬运与 warm-reset 跳板
              |-- AArch64 EL3/EL2 entry + PSCI/CVPM 监控器及配套 C51
              |-- 本型号专属 DTB
              `-- AArch64 Linux Image
```

### 2. 原厂认证与红外开机

- **1001、1006**：保留 160 MiB 处的原厂签名 `kernel.img`。将项目的 `trustedcore.img` 写入 220 MiB 处的对应槽位。
- **1004 R858**：使用同次构建的 CVPM-2 启动容器和 R858 DTB。关机后按红外电源键执行冷启动。

### 3. Rootfs 与内核更新

- **Rootfs**：使用 Android sparse ext4 镜像。1001／1002／1006 挂载 `/dev/mmcblk0p5`，1004 挂载 `/dev/mmcblk0p6`。1002 和 1004 使用 RAW-only 编码。
- **内核更新**：已装 DVB-IP Linux 的设备下载匹配机型和射频的更新包，核对 SHA-256，解压后运行 `sudo sh update.sh --check`。预检通过后执行 `sudo sh update.sh`，然后重启。

---

## TTL 串口日志与启动排障

`UART0`（`ttyAMA0`）固定参数为 **115200 8N1、无流控**。排查启动问题时应从断电冷启动开始完整抓取日志：

| 串口最后停留现象 | 排查重点 |
| --- | --- |
| 无任何 Fastboot 输出 | 检查供电、TTL RX/TX 交叉接线、3.3 V 电平及 eMMC 硬件状态 |
| Fastboot 报 `NovelCheck` / `Authenticate` / `check kernel fail` | 检查是否误动了 1001/1006 原厂签名 `kernel.img`、跨型号混刷了 `bootargs` 或容器偏移不匹配 |
| `uImage` CRC 或格式报错 | 检查容器下载完整性、HiTool 读取长度及对应型号封装头（如 1002 的 `0x20000` 前缀） |
| `Starting kernel ...` 后立即卡死或报 Undefined Instruction | 核对容器包含 `IH_ARCH_ARM` 的 `uImage` 和 AArch32 跳板 |
| 已输出 `[A32] requesting AArch64 warm reset`，未见 `[A64]` | 检查 `0x11000000` 复制/缓存清理、`RVBAR`（`0xf8a80034`）与 `AARCH_MODE`（`0xf8a80030`） |
| 已输出 `[A64] entered at EL3`，未进入 Linux EL2 | 核对驻留监控器、板型 DTB 和 Linux `Image` 来自同次构建 |
| 已输出 `[A64] entering Linux at EL2`，无内核 `earlycon` 输出 | 检查 `0x10000000` DTB 型号匹配、`0x02000000` `Image` 完整性及 `earlycon=pl011,mmio32,0xf8b00000` |
| 进入 Linux 后仅在线 1 个 CPU 核心 | 6.12.118 检查 DTB 的 `enable-method = "psci"`、SMC 和 EL3 驻留监控器；已装旧 `spin-table` 容器时检查 `cpu-release-addr` |
| 内核正常启动但等待 `root=` 超时 | 检查 HiTool 是否开启了“自动更新 bootargs 分区表”导致 `blkdevparts` 被篡改，以及 `rootfs.img` 是否正确刷入 |

---

## 构建与封装自检清单

| 阶段 | 必检项 |
| --- | --- |
| **编译与链接** | 外层 bundle 为 32 位 ARM ELF（入口 `0x12000000`）；内嵌 `entry` 与 Linux `Image` 为 AArch64；DTB `compatible` 与目标板型一致；各段体积未越界且保留 `0x11000000` 64 KiB `no-map` |
| **容器封装** | `uImage` 架构为 `ARM`、压缩为 `gzip`、`load/entry = 0x12000000`；1001/1006 `0x0` 处与 1002 `0x20000` 处具备合法 `ANDROID!` 头（`page_size = 0x4000`）；输出文件精确对齐 10 / 16 / 11 MiB |
| **落盘校验** | 刷写或在线更新后，分区回读字节流与容器文件 SHA-256 完全一致；`uname -r` 与 `/lib/modules/<release>/` 严格同名匹配 |

---

## 公开参考资料

- [Booting AArch64 Linux](https://docs.kernel.org/arch/arm64/booting.html)：ARM64 Linux 内核入口寄存器、异常级、缓存一致性与 DTB 规范。
- [U-Boot `bootm`](https://docs.u-boot.org/en/latest/usage/cmd/bootm.html) 与 [Android Boot Image](https://docs.u-boot.org/en/stable/android/boot-image.html)：legacy `uImage` 及 Android Boot 容器头部定义。
- [HiSTBLinux R005 SPC050](https://github.com/JasonFreeLab/HiSTBLinuxV100R005C00SPC050) / [SPC060](https://github.com/JasonFreeLab/HiSTBLinuxV100R005C00SPC060)：Hi3798CV200 平台寄存器与外围驱动公开参考源码。
- [AOSP libsparse `sparse_format.h`](https://android.googlesource.com/platform/system/core/+/master/libsparse/sparse_format.h)：Android sparse 稀疏镜像格式规范。
