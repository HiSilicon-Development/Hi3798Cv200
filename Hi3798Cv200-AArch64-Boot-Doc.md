# Hi3798CV200 AArch32 → AArch64 启动架构

本文说明 **DVB-IP1001、DVB-IP1002、DVB-IP1004（2+8）与 DVB-IP1006** 当前使用的 Linux 6.12.y（`6.12.114`）64 位启动链：沿用各机型原厂 AArch32 Fastboot 加载项目启动容器，经 ARM32 跳板、CP15 RMR 热复位（warm reset）、AArch64 EL3 安全态初始化与四核 `spin-table` 交接进入 non-secure EL2 Linux。

板级参数、完整分区表及系统节点见 [DVB-IP100x 产品与硬件总览](DVBIP-100x-INTRODUCTION.md)。

---

## 原厂 Fastboot 与各机型内核槽位

本项目直接复用各型号机器自带的原厂 AArch32 海思 Fastboot 与配套 `bootargs`，既不替换原厂 Fastboot，也不引入外部 TF-A 二进制作为运行时前置依赖（注：此处 Fastboot 指海思机顶盒板载引导固件，而非 Android USB `fastboot` 协议）。

| 型号 | 项目 Linux 启动容器 | eMMC 物理起点 / 容量 | Linux 分区设备 | 容器与引导约束 |
| --- | --- | --- | --- | --- |
| **DVB-IP1001** | `trustedcore.img` | 220 MiB / 10 MiB | `/dev/mmcblk0p4` | 保留原厂 Fastboot、bootargs 及 160 MiB 处的 40 MiB 签名 `kernel.img`；项目 Linux 固定从 `trustedcore` 引导 |
| **DVB-IP1002** | `kernel.img` | 12 MiB / 16 MiB | `/dev/mmcblk0p4` | 使用带 `0x20000` 前缀与 Android boot 头部的 16 MiB 完整容器，不可直接写入裸 ARM64 `Image` |
| **DVB-IP1004（2+8）** | `recovery.img` | 2 MiB / 11 MiB | `/dev/mmcblk0p3` | 配合 `misc` 分区引导标志，从 `recovery` 槽位读取 11 MiB legacy `uImage` 容器直接进入 Debian/Linux |
| **DVB-IP1006** | `trustedcore.img` | 220 MiB / 10 MiB | `/dev/mmcblk0p4` | 保留本型号原厂 Fastboot、bootargs 及 40 MiB 签名 `kernel.img`；项目 Linux 固定从 `trustedcore` 引导 |

- `kernel`、`recovery`、`trustedcore` 仅代表原厂 Fastboot 读取的存储槽位名称，与启动后的操作系统类型无关，四款机型最终均进入标准 Debian/Linux AArch64 用户态。
- 1001 与 1006 虽采用相同的槽位偏移与容器封装，但内嵌的设备树（DTB）不同，启动镜像不可互刷。
- 不同机型的 Fastboot 即使串口版本号相近，其认证链、DDR 初始化参数与容器偏移也互不兼容；刷写偏移、长度与摘要必须严格以本型号 `full.xml` 与 `SHA256SUMS` 为准。

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
  -> CPU0 在 AArch64 EL3 执行安全属性解锁、SMPEN、timer、GICv2 Group 1 与 CPU1–3 释放
  -> ERET 切换至 non-secure EL2h，清理 EL2 状态后以 x0 = 0x10000000 跳入 0x02000000
  -> Linux AArch64 启动，通过 spin-table 唤醒 CPU1–3、挂载 rootfs、执行 /sbin/init
```

原厂 Fastboot 解压并跳转到 `uImage` 时，CPU 仍处于 32 位 AArch32 状态；架构规定 AArch32 到 AArch64 的切换只能通过异常级提升或复位完成，因此真正的 64 位切换发生在 AArch32 跳板请求的硬件 warm reset 时刻。

### 1. AArch32 跳板（Bundle 前置层）

AArch32 bundle 是将 **ARM32 跳板、AArch64 EL3/EL2 entry、本机 DTB 与 AArch64 Linux `Image`** 链接在一起的单体引导载荷：

1. **屏蔽中断并输出串口标记**：执行 `cpsid if`，保存入口 `CPSR / SCTLR / ACTLR`，向 `UART0`（`0xf8b00000`）打印 `[A32] unpacking bundled ARM64 payload`。
2. **搬运载荷到目标物理地址**：将内嵌的 Linux `Image` 复制到 `0x02000000`、本机 DTB 复制到 `0x10000000`、AArch64 entry 复制到 `0x11000000`。
3. **维护缓存一致性**：按 `CTR.DminLine` 缓存行大小将三段目标内存的数据缓存清理至一致性点（PoC），失效指令缓存与分支预测器并执行 `DSB SY` / `ISB`（若省略缓存清理，warm reset 后 CPU 可能从内存读到未落盘的旧数据）。
4. **配置系统计数器与 64 位复位向量**：打印 `[A32] requesting AArch64 warm reset`；向海思系统计数器写入 `0xf8bb0020 = 1`（`SYS_COUNTER_ENABLE`）与 `0xf8bb0000 = 0x103`（`SYS_COUNTER_CTRL`）；将 `PERI_CPU_RVBARADDR`（`0xf8a80034`）设为 `0x11000000`，置位 `PERI_CPU_AARCH_MODE`（`0xf8a80030`）bit 0。
5. **请求 AArch64 热复位**：置位 CP15 Reset Management Register（`RMR`，`AA64 | RR = 0x3`）请求 warm reset，并在 `WFI` 循环中等待复位生效。

### 2. AArch64 EL3 → Non-Secure EL2 交接与内核入口

CPU0 经 warm reset 在 **AArch64 EL3** 进入 `0x11000000` 处的自定义 entry，依次完成以下初始化（串口依次输出 `[A64] entered at EL3; dropping to EL2`、`[A64] GICv2 handed to non-secure Linux`、`[A64] CPU1-3 released into spin-table`、`[A64] entering Linux at EL2`）：

- **外设安全属性解锁**：向 `SEC_ATTR_CTRL_0_REG`（`0xf8a80010`）写入 `0xfff9ffff` 并执行 `DSB SY` / `ISB`，解除红外接收器（`0xf8001000`）的安全态访问封锁，确保所有 CV200 机型在 non-secure Linux 下均可正常使用 `hix5hd2-ir`（不触碰会引起热复位锁死的 DDRC/TZASC 锁位）。
- **多核缓存一致性与定时器基线**：置位 Cortex-A53 `CPUACTLR_EL1.SMPEN`（bit 6）加入硬件缓存一致性域；清零继承的物理/虚拟定时器控制位，设置 `CNTFRQ_EL0 = 24000000`（24 MHz 架构定时器）。
- **GICv2 非安全移交**：将 GICD（`0xf1001000`）全部 SPI 及每核 GICC（`0xf1002000`）SGI/PPI 的 `IGROUPR` 设为 Group 1（non-secure），清除残留挂起中断，开启 SGI0 并配置 `GICC_CTLR = 0x1e7`、`GICC_PMR = 0x80`。
- **释放次级核 CPU1–3**：先将 `0x1100f000` 邮箱清零，设置 `PERI_CPU_AARCH_MODE = 0xf`（四核全部 AArch64）与 `PERI_CPU_RVBARADDR = 0x11000000`，再通过 `CPU_LP_CTRL`（`0xf8a22048`）与 `CPU_RESET_CTRL`（`0xf8a22050`）逐核解除上电/时钟/复位门控。
- **异常级切换至 EL2**：配置 `SCR_EL3 = 0x5b1`（`NS | HCE | SMD | RW`）、`CPTR_EL3 = 0`、`SPSR_EL3 = 0x3c9`（EL2h，屏蔽 DAIF），执行 `ERET` 降级到 non-secure EL2；随后开放 EL1 物理定时器访问（`CNTHCTL_EL2 |= 3`）、清零 `CNTVOFF_EL2` 与 `HCR_EL2`、关闭 EL2 MMU/缓存位，正式跳入 Linux。

跳转到 Linux 内核时的 CPU0 状态严格遵循 [Booting AArch64 Linux](https://docs.kernel.org/arch/arm64/booting.html) 规范：

```text
CPU0 状态 = AArch64 non-secure EL2h（MMU/D-cache 关闭，DAIF 屏蔽）
x0        = 0x10000000   本型号 DTB 物理地址
x1..x3    = 0
PC        = 0x02000000   AArch64 Linux Image 入口
```

### 3. 次级核（CPU1–3）的 `spin-table` 唤醒

`hi3798cv200-dvbip.dtsi` 显式移除上游参考板的 `/psci` 节点，采用无运行时安全固件依赖的 `spin-table` 多核唤醒协议：

```dts
enable-method = "spin-table";
cpu-release-addr = <0x0 0x1100f000>;
```

CPU1–3 被 CPU0 释放后同样从 EL3 进入 `0x11000000`，独立完成本核 `SMPEN`、GICC Group 1、`CNTFRQ_EL0` 与 EL3→EL2 降级，随后在 non-secure EL2 循环等待（`WFE`）邮箱 `0x1100f000` 变为非零。Linux 启动 SMP 时将 `secondary_holding_pen` 物理入口写入 `0x1100f000` 并发送 `SEV`，三个次级核清零 `x0..x3` 后同步上线。

因此，`0x11000000..0x1100ffff`（64 KiB）必须在 DTB `reserved-memory` 中声明为 `no-map`（`smp-firmware@11000000`），严禁被内核页分配器覆盖。

---

## 启动期与运行期内存布局

| 物理地址 / 范围 | 用途 | 尺寸上限与协同约束 |
| --- | --- | --- |
| `0x02000000..0x0fffffff` | AArch64 Linux `Image` 运行入口 | 最大 224 MiB，必须在 DTB 起点 `0x10000000` 前结束 |
| `0x10000000..0x10ffffff` | 板级 DTB（通过 `x0` 传入内核） | 最大 16 MiB，必须在 `0x11000000` 前结束 |
| `0x11000000..0x1100ffff` | `smp-firmware@11000000` 保留区 | 64 KiB `no-map`；其中 `0x11000000..0x1100cfff` 为 EL3/EL2 驻留代码（最大 52 KiB），`0x1100d000..0x1100efff` 预留可选早期显示快照区，`0x1100f000` 为 CPU1–3 `spin-table` 释放邮箱 |
| `0x12000000`（启动期） | AArch32 bundle 解压与入口地址 | legacy `uImage` 的 `load / entry = 0x12000000`，解压后体积上限 16 MiB；内核启动完成后该区间由 `decoder-capture@12000000`（160 MiB `reusable` CMA）安全复用 |

任何引导地址调整均须同步修改 ARM32 搬运目标、A64 链接地址与 `RVBAR`、DTB `reserved-memory` / `cpu-release-addr` 以及 `uImage` 封装参数，并校验各段无重叠。

---

## 启动容器封装与刷写规范

### 1. 各型号启动容器内部结构

所有机型的内层载荷均为 **ARM32 架构（`IH_ARCH_ARM`）gzip 压缩的 legacy `uImage`**（`load = 0x12000000, entry = 0x12000000`），外层按各机型原厂 Fastboot 的解析规则封装并补齐到固定分区容量：

| 型号 | 容器文件与总容量 | 容器字节布局 |
| --- | --- | --- |
| **1001 / 1006** | `trustedcore.img`（10 MiB） | `0x0` 处为 Android Boot Header（`ANDROID!`，`page_size = 0x4000`，含 SHA-1 ID）；`0x4000` 处为 legacy `uImage`；尾部零填充至 10 MiB |
| **1002** | `kernel.img`（16 MiB） | 保留 `0x0..0x1ffff`（128 KiB）前置头；`0x20000` 处为 Android Boot Header（`page_size = 0x4000`）；`0x24000` 处为 legacy `uImage`；尾部填充至 16 MiB |
| **1004（2+8）** | `recovery.img`（11 MiB） | `0x0` 处直接放置 legacy `uImage`；尾部零填充至 11 MiB |

```text
型号专属容器（trustedcore.img / kernel.img / recovery.img）
  `-- legacy uImage（IH_ARCH_ARM，gzip，load/entry = 0x12000000）
        `-- AArch32 bundle
              |-- ARM32 搬运与 warm-reset 跳板
              |-- AArch64 EL3/EL2 entry + spin-table 驻留代码
              |-- 本型号专属 DTB
              `-- AArch64 Linux Image
```

> 注意：将 `uImage` 头部改成 `IH_ARCH_ARM64` 并不能让 AArch32 Fastboot 切换到 64 位，反而会导致 Fastboot 拒绝引导或直接把 ARM64 指令当作 ARM32 执行而触发未定义指令异常。

### 2. 原厂认证链与看门狗复位说明

- **保留原厂前置认证**：原厂 Fastboot 执行的 `bootargs`、Novel/ADVCA 签名或头部格式检查在进入本项目 bundle 之前运行。1001 与 1006 必须保留 160 MiB 处的原厂签名 `kernel.img` 作为认证前置门，仅将项目内核写入 220 MiB 处的 `trustedcore`；若串口出现 `NovelCheck`、`Authenticate` 或 `check kernel fail`，说明在 Fastboot 阶段即未通过容器/签名校验，与 Linux rootfs 无关。
- **看门狗与系统复位**：Linux 内核为片内 `sp805-wdt`（`0xf8a2c000`）配置 `restart_load = 0x100`（Hi3798 平台写入 `WDTLOAD = 0` 不会触发硬件复位）。对于 1004（`CA_ID = 0x6edbe953`），因脱离原厂系统后未加载私有 C51 PMOC 待机固件，软件 `reboot` 可能停留在看门狗溢出阶段而无法拉起整板硬复位，此时通过物理断电重上电即可正常冷启动。

### 3. Rootfs 与在线内核更新

- `rootfs.img` 为独立的 Android sparse ext4 镜像（不包含在内核启动容器内），1001/1002/1006 挂载 `/dev/mmcblk0p5`，1004 挂载 `/dev/mmcblk0p6`。其中 1002 与 1004 采用纯 `RAW` 块编码（避免旧版 HiTool 忽略 `DONT_CARE` 块而残留旧数据），所有机型均禁用 `DONT_CARE` 并在首次启动时自动扩容占满分区。
- 在已运行本项目 Debian 12/13 的设备上，在线内核更新包会将本型号容器统一命名为包内 `kernel.img`，在校验板型、容量与模块 `vermagic` 后直接写入对应内核分区（1001/1002/1006 为 `mmcblk0p4`，1004 为 `mmcblk0p3`）并完整回读比对 SHA-256。

---

## TTL 串口日志与启动排障

`UART0`（`ttyAMA0`）固定参数为 **115200 8N1、无流控**。排查启动问题时应从断电冷启动开始完整抓取日志：

| 串口最后停留现象 | 排查重点 |
| --- | --- |
| 无任何 Fastboot 输出 | 检查供电、TTL RX/TX 交叉接线、3.3 V 电平及 eMMC 硬件状态 |
| Fastboot 报 `NovelCheck` / `Authenticate` / `check kernel fail` | 检查是否误动了 1001/1006 原厂签名 `kernel.img`、跨型号混刷了 `bootargs` 或容器偏移不匹配 |
| `uImage` CRC 或格式报错 | 检查容器下载完整性、HiTool 读取长度及对应型号封装头（如 1002 的 `0x20000` 前缀） |
| `Starting kernel ...` 后立即卡死或报 Undefined Instruction | 误刷入了裸 ARM64 `Image` 而未经过 AArch32 跳板封装 |
| 已输出 `[A32] requesting AArch64 warm reset`，未见 `[A64]` | 检查 `0x11000000` 复制/缓存清理、`RVBAR`（`0xf8a80034`）与 `AARCH_MODE`（`0xf8a80030`） |
| 已输出 `[A64] entered at EL3`，未进入 Linux EL2 | 检查 EL3 安全寄存器访问、GICv2 初始化或次级核释放流程 |
| 已输出 `[A64] entering Linux at EL2`，无内核 `earlycon` 输出 | 检查 `0x10000000` DTB 型号匹配、`0x02000000` `Image` 完整性及 `earlycon=pl011,mmio32,0xf8b00000` |
| 进入 Linux 后仅在线 1 个 CPU 核心 | 检查 DTB 是否保留 `smp-firmware@11000000`（`no-map`）及 `cpu-release-addr = <0x0 0x1100f000>` |
| 内核正常启动但等待 `root=` 超时 | 检查 HiTool 是否开启了“自动更新 bootargs 分区表”导致 `blkdevparts` 被篡改，以及 `rootfs.img` 是否正确刷入 |

> 外设状态提示：CP15 warm reset 仅复位 CPU 核心以完成 32→64 位切换，不会复位整片 SoC 外设。Linux 各子系统驱动（`dw_mmc` HS400 相位校验、`histb-vdp` / `histb-hdmi` 复位重配、`xhci-histb` PHY、`hix5hd2_gmac` PHY 复位延时等）负责接管并重新初始化 Fastboot 遗留的硬件状态。

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
