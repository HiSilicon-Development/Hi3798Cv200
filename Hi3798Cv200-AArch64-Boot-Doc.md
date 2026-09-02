# Hi3798CV200 AArch32 → AArch64 启动架构

本文说明 DVB-IP1001、DVB-IP1002、DVB-IP1004（2+8）和 DVB-IP1006 当前使用的Linux 6.12 启动链：原厂 AArch32 Fastboot 加载项目启动容器，经 ARM32 跳板、AArch64 entry 和四核交接进入 Linux。
板级参数、完整分区表及系统节点见[DVB-IP100x 产品总览](DVBIP-100x-INTRODUCTION.md)。

## 当前使用的 Fastboot 与内核槽位

本项目沿用各型号自己的原厂 Fastboot 和配套 bootargs，不要求把 Fastboot改成 AArch64，也不使用完整 TF-A 固件作为当前启动链的运行时依赖。
这里的 Fastboot 是海思板上启动固件，不是 Android USB `fastboot` 协议。

| 型号 | 项目 Linux 启动容器 | eMMC 物理起点 / 容量 | 启动要求 |
| --- | --- | --- | --- |
| 1001 | `trustedcore.img` | 220 MiB / 10 MiB | 保留原厂 Fastboot、bootargs 及 160 MiB 起始的签名 `kernel.img`；项目 Linux 放在 trustedcore |
| 1002 | `kernel.img` | 12 MiB / 16 MiB | 使用本型号完整 kernel 分区容器，不写裸 ARM64 Image |
| 1004（2+8） | `recovery.img` | 2 MiB / 11 MiB | 使用本型号 recovery 容器，保持完整容量 |
| 1006 | `trustedcore.img` | 220 MiB / 10 MiB | 保留本型号原厂 Fastboot、bootargs 和签名 kernel；固定读取 trustedcore 启动项目 Linux |

`kernel`、`recovery`、`trustedcore` 是容器/槽位名称，不代表运行的用户空间类型。
1004 从 recovery 启动的仍是 Debian/Linux；1001/1006 的项目 trustedcore 容器也不能与原厂签名 kernel 互换。
1001 与 1006 布局相同，但不是同一套启动镜像。

Fastboot 的版本横幅相同，不代表 bootcmd、认证范围、DDR 初始化或容器格式相同。
具体写入起点、读取长度和文件校验值以对应型号的 `full.xml` 与 `SHA256SUM` 为准，不按固件年份选包。

## AArch64 主启动链

```text
上电或硬复位
  -> Hi3798CV200 BootROM
  -> 原厂 Fastboot（AArch32；原包带解压前置层时保持不变）
  -> 按 bootargs/bootcmd 读取本型号启动容器并执行认证/格式检查
  -> 解压并执行 AArch32 bundle，地址 0x12000000
  -> 复制 Linux Image、DTB 和 AArch64 entry，清理缓存
  -> 设置 RVBAR 和 AArch64 reset 状态，请求 warm reset
  -> AArch64 entry 在 EL3 完成 GICv2、timer 和 CPU1–3 交接
  -> ERET 到 non-secure EL2，x0 指向 DTB
  -> Linux AArch64，唤醒次级核、挂载 rootfs、启动 /sbin/init
```

Fastboot 解压 bundle 不等于 CPU 已进入 AArch64。真正的执行状态切换发生在bundle 请求的 warm reset；rootfs 是 armhf 还是 arm64，不会改变内核入口前的 CPU 状态。

### AArch32 bundle

bundle 是包含 ARM32 跳板、AArch64 entry、DTB 和 Linux Image 的启动对象，不是另一个 Linux 内核。ARM32 部分完成：

1. 屏蔽 IRQ/FIQ，保存进入时的 CPSR、SCTLR、ACTLR。
2. 将内嵌 Image 复制到 `0x02000000`，DTB 复制到 `0x10000000`，AArch64 entry 复制到 `0x11000000`。
3. 清理复制目标的数据缓存到一致性点，失效指令缓存与分支预测状态，执行 `DSB`/`ISB`。
4. 配置平台系统计数器，设置 CPU0 的 RVBAR 和 AArch64 reset 模式。
5. 通过 Reset Management Register 请求 warm reset，并在 `WFI` 中等待。

只复制代码不清理缓存，warm reset 后可能读到旧内容；把 ARM64 指令紧接在ARM32 指令后面，也不会自动切换处理器状态。

### AArch64 entry 与 Linux 入口

当前冷启动主路径从 EL3 进入 entry，随后：

- 设置 Cortex-A53 SMPEN，使各核加入一致性域；
- 清理继承的 timer 状态，建立 24 MHz architected timer 频率；
- 配置 GICv2 distributor 和每核 CPU interface，交给 non-secure Linux；
- 按平台 CPU_ON 顺序配置 CPU1–3 的模式、RVBAR、电源/reset 状态；
- 设置 `SCR_EL3`、`SPSR_EL3`、`ELR_EL3`，通过 `ERET` 转到 non-secure EL2；
- 清理 EL2 timer、虚拟计时偏移、HCR 和 MMU/cache 状态，再跳入 Linux。

Linux 入口寄存器和地址为：

```text
CPU0 = AArch64 non-secure EL2
x0   = 0x10000000   DTB 物理地址
x1   = 0
x2   = 0
x3   = 0
PC   = 0x02000000   Linux Image 入口
```

入口时 MMU 和数据缓存关闭，中断屏蔽；Image/DTB 已写回内存。
完整要求见 [Booting AArch64 Linux](https://docs.kernel.org/arch/arm64/booting.html)。
这些工作由 bundle 内的自定义 entry 完成，不是 Fastboot 的 ARM64 header或一份额外 TF-A 二进制替它完成。

### CPU 1–3 的 spin-table

当前设备树使用 `spin-table`，不依赖 PSCI：

```dts
enable-method = "spin-table";
cpu-release-addr = <0x0 0x1100f000>;
```

entry 先释放 CPU1–3，让各核完成一致性、timer、GIC 和异常级设置，再等待`0x1100f000` 中的 Linux 次级入口地址。
Linux 写入地址并发送唤醒事件后，次级核清零 `x0..x3` 并进入内核。

`0x11000000..0x1100ffff` 必须在 DTB 中作为 `no-map` 保留内存；其中包含entry、次级核驻留代码和 release mailbox，不能让普通内存分配器覆盖。

## 内存布局

| 物理地址 / 范围 | 内容 | 约束 |
| --- | --- | --- |
| `0x02000000` | AArch64 Linux Image | 在 DTB 目标之前结束；构建尺寸上限 224 MiB |
| `0x10000000` | DTB | Linux `x0`；在 entry 之前结束，构建尺寸上限 16 MiB |
| `0x11000000..0x1100ffff` | AArch64 entry 与 SMP 保留区 | DTB `reserved-memory`，64 KiB、`no-map` |
| `0x1100f000` | CPU1–3 release mailbox | Linux 写入次级入口地址 |
| `0x12000000` | AArch32 bundle load / entry | 包含跳板、entry、DTB、Image；不得覆盖运行代码或目标区 |

这些地址是当前 bundle 与 DTB 的共同约定，不是所有 CV200 固件的通用规范。
修改时必须同时更新 A32 复制目标、A64 链接地址/RVBAR、Linux 入口、`reserved-memory`、`cpu-release-addr` 和 uImage load/entry，并检查区间重叠。

## 启动容器与认证

### 当前封装格式

| 型号 | 容器内部布局 |
| --- | --- |
| 1001 / 1006 | Android boot header 位于 `0x0`，page size 为 `0x4000`；legacy uImage 位于 `0x4000`；文件补齐为 10 MiB |
| 1002 | 保留 `0x20000` 前缀，Android boot header 位于 `0x20000`；page size 为 `0x4000`，legacy uImage 位于 `0x24000`；文件补齐为 16 MiB |
| 1004（2+8） | legacy uImage 从 `0x0` 开始；文件补齐为 11 MiB |

三种封装最终都使用 ARM 类型的 legacy uImage，load/entry 为 `0x12000000`，
gzip 解压后得到 AArch32 bundle：

```text
本型号启动容器
  -> legacy uImage（ARM，load/entry 0x12000000）
  -> gzip
  -> AArch32 bundle
       |-- AArch64 entry
       |-- 本型号 DTB
       `-- Linux Image
```

`IH_ARCH_ARM64` 只是镜像元数据，不能将正在执行 AArch32 的 CPU 切为AArch64。
当前 uImage 入口必须仍是 ARM32 跳板，不能直接换成裸 ARM64 Image。

### 保留原厂认证输入

原厂 Fastboot 负责本机要求的 bootargs、Novel/ADVCA/CCDT 等检查；项目启动容器不等于关闭这些检查，也不提供通用的安全启动替换方案。

- 1001/1006 保留原厂签名 kernel 认证门，项目 Linux 位于物理 220 MiB 的trustedcore；不能用新 Image 替换签名 `kernel.img`。
- 1002/1004 按各自已使用的 kernel/recovery 包装和读取范围封装 bundle，不能跨型号复制前缀、header 或 bootargs。
- environment CRC、Android image ID 和 uImage CRC 是完整性/格式检查，不等于有效的客户签名。重算 CRC 不会改变原厂认证要求。
- 出现 `NovelCheck`、`Authenticate` 或 `check kernel fail` 时，Linux尚未执行，应先检查原厂输入、容器和读取范围，而不是调整 rootfs。

### 分区与 rootfs

启动容器必须符合 Fastboot 实际读取的起点和长度；分区较大不表示固件会自动读取全部空间。
物理分区表见[产品总览](DVBIP-100x-INTRODUCTION.md#hitool-物理布局)。

rootfs 是独立的 Android sparse ext4，不属于 bundle。
1001/1002/1006 的当前 Linux root 为 `/dev/mmcblk0p5`，1004 为 `/dev/mmcblk0p6`；不能把历史分区编号直接用作当前写入目标。

HiTool 应使用 sparse-aware 写入。特别是 1002 的 2017 writer，`DONT_CARE`块只前移写入位置，可能保留旧数据；当前 rootfs 使用 RAW chunk 完整覆盖逻辑块。
sparse 文件摘要用于校验交付文件，设备端则应按展开后的实际写入内容核对，不能将压缩/稀疏文件 hash 直接当作分区 hash。

## TTL 日志与故障定位

UART0 使用 **115200 8N1、无流控**，应从完全断电后上电开始记录日志。
TTL 在这里用于观察原厂 Fastboot、A32/A64 交接和 Linux，不作为额外下载或 reboot-mode 控制接口。

| 最后一条可靠日志 / 现象 | 优先检查 |
| --- | --- |
| 没有 Fastboot 日志 | 电源、串口接线、电平、启动介质 |
| 原厂解压或认证失败 | 对应型号的 Fastboot/bootargs、认证输入和容器完整性 |
| uImage CRC / format 错误 | header、gzip 数据、读取范围、load/entry |
| `Starting kernel` 后 undefined instruction | 是否直接执行 ARM64 字节、A32 跳板和 reset 模式 |
| 已打印 A32 warm reset，未进入 A64 | 复制/cache clean、RVBAR、reset 状态和 entry 地址 |
| 已进入 A64 EL3，未进入 Linux EL2 | EL3 返回状态、GIC、timer、次级核释放 |
| 已进入 Linux EL2，内核没有继续输出 | DTB、Image 地址、入口寄存器和缓存 |
| Linux 只有一个 CPU | spin-table、mailbox、保留内存和次级核 reset |
| 找不到 rootfs | bootargs、当前分区号、eMMC 模式、ext4 和驱动 |
| `/sbin/init` 已运行但外设异常 | 对应驱动、DTS、时钟/reset/pinmux 和固件遗留状态 |

当前跳板会输出 A32 解包/warm-reset、A64 异常级交接等标记。一次完整启动还应能确认 Linux 型号、四核上线、rootfs 挂载和 `/sbin/init` 运行。
不要只截取 `Starting kernel` 就判断 AArch64 交接成功。

warm reset 不会自动重置所有外设。Linux 驱动仍需处理 Fastboot 留下的时钟、eMMC HS400 tuning、电压、VDP/HDMI、USB PHY、GMAC/TSI 和 DMA 状态；已经进入用户态的 HDMI 黑屏不应一律归因于 AArch32→AArch64。

## 构建、封装与写入检查

| 阶段 | 检查内容 |
| --- | --- |
| 构建 | A32 bundle 为 ARM ELF；entry/Image 为 AArch64；DTB 型号正确；地址、尺寸、保留内存和 mailbox 一致 |
| 封装 | gzip 解包与输入 bundle 一致；uImage CRC/load/entry；Android header/page size/image ID（适用时）；固定容量和尾部填充 |

## 公开参考资料

- [Booting AArch64 Linux](https://docs.kernel.org/arch/arm64/booting.html)：Linux 入口寄存器、异常级、缓存、DTB 和多核要求。
- [U-Boot `bootm`](https://docs.u-boot.org/en/latest/usage/cmd/bootm.html) 与 [Android Boot Image](https://docs.u-boot.org/en/stable/android/boot-image.html)：uImage、Android boot 容器格式参考。
- [HiSTBLinux R005 SPC050](https://github.com/JasonFreeLab/HiSTBLinuxV100R005C00SPC050) / [SPC060](https://github.com/JasonFreeLab/HiSTBLinuxV100R005C00SPC060)：CV200 平台启动和外围实现参考，不作为本项目量产 Fastboot 的替换镜像。
- [AOSP sparse image header](https://android.googlesource.com/platform/system/core/+/master/libsparse/sparse_format.h)：Android sparse 格式定义。
