# Hi3798CV200 AArch32 → AArch64 启动架构与固件边界

本文说明 Hi3798CV200 量产板从 AArch32 厂商 Fastboot 启动 AArch64 Linux 时涉及的执行状态、镜像容器、内存布局、安全边界和 UART 下载机制。

目标是能够判断故障发生在哪一层，并避免把相同的 `Fastboot 3.3.0` 横幅、分区名称或文件扩展名误认为兼容性证明。

本文的核心结论是：当前已验证的 AArch64 启动方案并不是让原有 Fastboot 直接执行 AArch64 Linux，也不是在运行时依赖一份完整 TF-A 固件完成交接。

Fastboot 仍在 AArch32 状态执行；它先启动一个 AArch32 bundle，bundle 再请求 AArch64 warm reset。

warm reset 后进入 bundle 自带的 AArch64 EL handoff entry，由这个 entry 完成 EL3 → non-secure EL2、GICv2 交接、CPU1–3 释放和 Linux 入口寄存器设置。

## 术语和边界

### 本文所说的 Fastboot

本文中的 **Fastboot** 是海思 SDK 中基于 U-Boot 派生的板上启动固件。

它会打印 `Fastboot 3.3.0`，能够解析环境、分区、Android boot image、legacy uImage 以及厂商认证包装。

它必须与以下机制分开理解：

- **Android USB Fastboot**：主机通过 `fastboot` 工具和设备 USB gadget 通信的标准协议；
- **BootROM 下载模式**：芯片复位后由 BootROM 决定是否接受 USB、UART 或其他下载入口；
- **厂商 UART bootstrap**：已经运行的海思 Fastboot 根据 SYSCTRL 标志进入的隐藏串口命令循环；
- **HiTool**：海思主机端烧录工具及其 XML / 分区 描述；它不是 CPU 执行状态切换机制；
- **Compressed-boot**：某些产品中位于 Fastboot 之前的解压层，不是 Linux kernel 的 gzip 属性。

这些入口可能共存，也可能只编入其中一部分。发现其中一种不能证明其他入口存在。

### 必须分别确认的三件事

- **CPU 能力**：Cortex-A53 支持 AArch64；
- **镜像声明**：Android header 或 legacy uImage 可以把 payload 标记为 ARM64；
- **当前执行状态**：跳入 payload 时，处理器究竟处于 AArch32 还是 AArch64，以及当前异常级是什么。

前两项都不能代替第三项。

AArch32 Fastboot 直接跳到 AArch64 指令时，合法的 ARM64 header 也无法替处理器切换执行状态。

rootfs 是 armhf 还是 arm64，同样不能改变内核入口之前的 CPU 状态。

### 源码能力不等于量产二进制能力

公开 CV200 boot 源码和 HiSTB Linux SDK 中可以找到 `IH_ARCH_ARM64`、`go64`、warm reset 和 EL3 平台代码，但具体量产 Fastboot 是否编入这些宏和命令，只能由同一份二进制的完整横幅、命令表、反汇编或实机日志证明。

相同的 `Fastboot 3.3.0` 版本字符串不能证明以下内容相同：

- 构建日期、构建用户和客户分支；
- Novel、ADVCA、CCDT 或 TrustedCore 策略；
- 环境偏移、`bootcmd` 和分区布局；
- Compressed-boot、Android boot 和 legacy uImage 的嵌套顺序；
- DDR、eMMC 和 pinmux 初始化参数。

## 已验证的 AArch64 主启动链

### 完整链路

```text
上电或硬复位
  │
  ▼
Hi3798CV200 BootROM
  │  读取启动介质和安全状态
  │  决定正常启动或是否接受受支持的下载入口
  ▼
可选的 Compressed-boot 解压层
  │  只负责恢复下一阶段 Fastboot
  │  并不表示处理器已经进入 AArch64
  ▼
海思 Fastboot / U-Boot 派生固件（AArch32）
  │  读取环境和 bootargs
  │  选择启动分区并执行 Novel/ADVCA/CCDT 等检查
  │  解析厂商外层、Android boot、legacy uImage 和 gzip
  ▼
AArch32 bundle，加载并运行于 0x12000000
  │  复制 AArch64 Linux Image 到 0x02000000
  │  复制 DTB 到 0x10000000
  │  复制 AArch64 EL handoff entry 到 0x11000000
  │  清理缓存、设置计数器、RVBAR 和 AArch64 reset 状态
  │  请求 warm reset
  ▼
自定义 AArch64 EL handoff entry，首先在 EL3 执行
  │  建立每核一致性和计时器状态
  │  初始化并交接 GICv2
  │  释放 CPU1–3，使其进入 spin-table 等待
  │  配置 EL3 返回到 non-secure EL2
  ▼
AArch64 EL2
  │  x0 = 0x10000000（DTB）
  │  x1 = x2 = x3 = 0
  │  branch 0x02000000（Linux Image）
  ▼
Linux 6.12 AArch64
  │  CPU0 进入内核
  │  通过 0x1100f000 spin-table mailbox 唤醒 CPU1–3
  ▼
挂载 rootfs 并启动 /sbin/init
```

这条链中有两次容易混淆的“解压”：

- `Compressed-boot v1.0.0` 解压的是 Fastboot 阶段或其前置包装；
- legacy uImage 显示的 `gzip compressed` 解压的是它承载的 kernel payload，在当前方案中得到的是 AArch32 bundle。

看到前一种 `Uncompress...Ok` 不能说明 Linux 已经加载；看到后一种 `Uncompressing Kernel Image...OK` 也不能说明 CPU 已经进入 AArch64。

### BootROM 和 Compressed-boot 的职责

BootROM 是芯片复位后的硬件信任起点。

它选择启动介质、读取芯片安全配置，并决定当前启动对象是否可以继续执行。

BootROM 是否接受某个 DDR programmer、USB 下载器或 UART 下载入口，取决于芯片状态和具体固件，不应从另一台 CV200 设备类推。

部分产品会先打印：

```text
Compressed-boot v1.0.0
Uncompress......................Ok

System startup
...
Fastboot 3.3.0 ...
```

这表明 Compressed-boot 位于可见 Fastboot 横幅之前。

它的输入格式、链接地址和解压目标必须与同一份 Fastboot 构建匹配。

不能因为两个解压后的 Fastboot 都打印 `3.3.0`，就互换压缩的 Fastboot 分区。

### AArch32 Fastboot 的职责

Fastboot 仍以 ARMv7/AArch32 指令运行。它负责：

- 初始化足以读取启动介质的 DDR、eMMC、时钟和引脚；
- 读取环境、bootargs、分区起点和读取长度；
- 根据 `bootcmd` 选择名为 `kernel`、`recovery` 或其他名称的启动对象；
- 执行产品固件要求的 Novel、ADVCA、CCDT 或 TrustedCore 检查；
- 解析外层容器、Android boot header、legacy uImage header 和压缩格式；
- 把最终 AArch32 bundle 放到约定地址并跳入其入口。

分区名 `recovery` 只说明 Fastboot 从哪个槽位读取数据。

该槽位完全可以承载 Debian/Linux 启动容器，并不意味着之后运行的是 Android recovery 用户空间。

### AArch32 bundle 的职责

AArch32 bundle 是状态切换前最后一个 ARM32 程序。

它不是 Linux kernel，也不是 ARM32 用户空间。

当前实现按以下顺序工作：

1. 保存进入时的 CPSR、SCTLR 和 ACTLR，以便调试继承状态；
2. 屏蔽 IRQ 和 FIQ，避免复制和切换期间被旧中断路径打断；
3. 将内嵌的 AArch64 `Image`、DTB 和 AArch64 entry 复制到固定物理地址；
4. 按 CPU cache line 大小把复制目标清理到 Point of Coherency；
5. 失效指令缓存和分支预测状态，并执行 `DSB`/`ISB`；
6. 把系统计数器配置为与平台相符的 24 MHz 状态；
7. 将 CPU0 的 reset vector 指向 `0x11000000`；
8. 选择 AArch64 reset 状态并通过 Reset Management Register 请求 warm reset；
9. 进入 `WFI` 等待 reset 生效，而不是继续顺序执行 AArch64 字节。

这一步必须同时解决“内容已经写入 RAM” 和 “下一次取指能看到新内容” 两个问题。

只复制 payload、不清理数据缓存，可能使 warm reset 后的 CPU 从旧内存内容取指；只清理数据缓存、不失效指令侧状态，也可能留下难以复现的早期异常。

### AArch64 EL handoff entry 的职责

warm reset 后，CPU0 从 `0x11000000` 进入自定义 AArch64 entry。

entry 会读取 `CurrentEL`，对 EL3、EL2 和 EL1 入口分别处理；已验证的冷启动主路径从 EL3 进入。

在 EL3 路径中，entry 负责：

- 屏蔽 DAIF 并失效指令缓存；
- 为 Cortex-A53 设置 SMPEN，使各核加入 inner-shareable 一致性域；
- 关闭从 AArch32/Fastboot 继承的物理、虚拟和 hypervisor timer 控制状态；
- 初始化 GICv2 distributor 与每核 CPU interface，并把 Linux 所需中断状态交给 non-secure 世界；
- 使用海思平台 CPU_ON 顺序设置 CPU1–3 的 AArch64 模式、RVBAR、低功耗控制和 reset 位；
- 把 CPU1–3 引导到同一 AArch64 entry，随后让它们在 spin-table mailbox 等待；
- 设置 `SCR_EL3`、`SPSR_EL3`、`ELR_EL3`、`CPTR_EL3` 和 24 MHz `CNTFRQ_EL0`；
- 通过 `ERET` 进入 non-secure EL2。

在 EL2 路径中，entry 清理继承的 EL2 timer、虚拟计时偏移、HCR 和 MMU/cache 控制状态，最后建立 Linux AArch64 boot protocol 所需寄存器：

```text
x0 = 0x0000000010000000   DTB 物理地址
x1 = 0
x2 = 0
x3 = 0
PC = 0x0000000002000000   Linux Image 入口
```

因此，`x0=DTB`、EL3 → EL2、GICv2 和多核释放是由 bundle 内的自定义 AArch64 entry 完成的。

实现参考了海思厂商 ATF 平台端的 CPU/GIC 顺序，但这不等于启动时有一份完整 TF-A 二进制在替 bundle 执行这些步骤。

### CPU1–3 的 spin-table 路径

当前设备树删除 PSCI，并为 CPU1–3 使用 `spin-table`：

```text
enable-method = "spin-table"
cpu-release-addr = <0x0 0x1100f000>
```

AArch64 entry 在 EL3 释放次级核后，各次级核完成自己的 SMPEN、timer、GIC CPU interface 和 EL3 → EL2 设置，然后等待 `0x1100f000` 从零变成 Linux 写入的 `secondary_holding_pen` 地址。

Linux 写入入口并执行唤醒事件后，次级核清零 `x0..x3` 并跳入内核次级入口。

这也是为什么 `0x11000000..0x1100ffff` 必须在设备树中声明为 `no-map` 保留内存：其中既有 warm-reset entry，也有 Linux 与次级核共享的 release mailbox，普通内存分配器不能覆盖它。

## 当前实现的内存布局

| 物理地址或范围 | 内容 | 约束 |
| --- | --- | --- |
| `0x02000000` | 解包后的 AArch64 Linux `Image` | 必须在 DTB 目标之前结束；当前构建上限为 224 MiB |
| `0x10000000` | DTB | Linux 入口的 `x0`；必须在 `0x11000000` 前结束，当前上限为 16 MiB |
| `0x11000000` | AArch64 EL handoff entry 和次级核驻留代码 | 设备树保留 64 KiB；普通 Linux 内存不得使用 |
| `0x1100d000..0x1100efff` | 可选的只读早期显示快照窗口 | 仅诊断构建使用；生产构建不依赖此内容 |
| `0x1100f000` | CPU1–3 spin-table release mailbox | 启动前清零，Linux 写入次级入口地址 |
| `0x12000000` | Fastboot 解压并执行的完整 AArch32 bundle | 包含 A32 代码、A64 entry、DTB 和 Linux Image |

布局关系如下：

```text
0x02000000  AArch64 Linux Image
     ...
0x10000000  DTB
     ...
0x11000000  AArch64 entry / SMP firmware reserved area
0x1100d000  optional diagnostic snapshot
0x1100f000  spin-table mailbox
0x11010000  reserved area end
     ...
0x12000000  AArch32 bundle load/entry
```

这些地址是当前 bundle 与设备树共同定义的 ABI，不是 Hi3798CV200 对所有固件的固定规范。

修改其中任一地址时，至少要同步检查：

- A32 复制目标；
- A64 linker address 和 RVBAR；
- Linux `x0` 与 Image branch 地址；
- 设备树 `reserved-memory`；
- CPU `cpu-release-addr`；
- Fastboot/uImage 的 load、entry 和解压目标；
- 各区间最大长度和重叠检查。

如果只修改 linker script 或 header 中的地址，镜像可能仍通过 CRC，却在 warm reset 后覆盖 DTB、entry 或 mailbox。

## 镜像容器与分区契约

### 容器是逐层解析的

实际启动对象可能由多层组合而成：

```text
Novel / CCDT / 客户签名外层
  └─ Android boot v0（ANDROID!、page size、kernel size 等）
       └─ legacy uImage（magic、CRC、OS、arch、type、compression、load、entry）
            └─ gzip 数据
                 └─ AArch32 bundle
                      ├─ AArch64 EL handoff entry
                      ├─ DTB
                      └─ AArch64 Linux Image
```

并非每个样本都有上图全部层次，顺序也不能跨固件推断。

应从外到内逐层记录 magic、长度、偏移、hash、load 和 entry，不能仅凭文件名判断。

### ARM64 header 不能切换 CPU 状态

legacy uImage 的 `IH_ARCH_ARM64` 只是一项元数据。

它能帮助 bootloader 选择处理分支，但不能让处于 AArch32 的处理器自动开始执行 AArch64 指令。

项目曾把合法的 ARM64 legacy uImage 放入正确的外层容器。

Fastboot 能完成认证和 header 解析，打印 `Starting kernel`，随后在第一批 AArch64 指令处触发 undefined instruction。

这证明失败点位于执行状态交接，而不是文件扩展名、CRC 或 ARM64 header 本身。

当前方案让 uImage 最终执行的入口仍是 AArch32 bundle；真正的 AArch64 Linux 位于 bundle 内部，在 warm reset 完成后才被执行。

### 分区名、读取长度和镜像大小

Fastboot 根据环境和 `bootcmd` 读取明确的起点与长度。以下推论都不成立：

- 分区在 XML 中更大，所以 Fastboot 一定会读取全部空间；
- eMMC 上存在可识别 magic，所以 Fastboot 会自动扫描到它；
- `kernel` 和 `recovery` 名称相同，所以两个产品使用同一容器；
- 原始 bundle 小于分区，所以可以直接替代完整分区镜像；
- 写入命令返回成功，所以设备端数据一定完整。

生产交付必须使用目标槽位要求的完整容器和固定长度分区镜像。

裸 ARM64 `Image`、裸 DTB 或裸 bundle 只适用于已经证明的 RAM-only 测试入口，不能直接写入要求 Novel/Android/uImage 包装的启动槽。

### Android sparse rootfs 是另一层协议

Android sparse ext4 与 kernel 容器无关。常见 chunk 为：

| chunk | 含义 | 写入风险 |
| --- | --- | --- |
| `0xCAC1` / RAW | 后续是实际块数据 | 必须核对块数、长度和摘要 |
| `0xCAC2` / FILL | 使用一个填充值重复生成块 | 解析器必须检查填充范围和溢出 |
| `0xCAC3` / DONT_CARE | 逻辑上跳过一段 | 可能保留设备旧块，不能当作安全擦除 |
| `0xCAC4` / CRC32 | sparse 数据校验信息 | 不能替代完整设备读回 hash |

稀疏文件体积较小，不代表目标分区所有逻辑块都被覆盖。

为了避免旧 ext4 元数据和旧文件残留，生产 profile 应明确哪些范围必须完整写入，并在设备端读回后验证文件系统和摘要。

## 已观察到的 Fastboot 与认证路径

以下年份只是根据横幅和日志命名的构建年代，不是海思发布的兼容等级。

### 2017：R002/SPC031、Novel 与 legacy uImage

已验证样本包含以下横幅和启动日志：

```text
Fastboot 3.3.0 (May 22 2017 - 13:40:21)
CPU:           Hi3798Cv200 (CA)
SDK Version:   HiSTBAndroidV600R002C00SPC031_v2016051915

NovelCheck BOOTARGS ... ok
NovelCheck kernel ... ok
Boot with Noval CA,Header length 128K
Check Hisilicon_ADVCA ...
Not hisilicon ADVCA image ...
## Booting kernel from Legacy Image ...
```

`v2016051915` 是 SDK 标签的一部分，与 Fastboot 在 2017 年编译并不冲突。该日志能证明：

- 启动对象先经过 Novel 检查；
- 一个已审计容器使用 128 KiB Novel 外层；
- 随后进入 Android boot/legacy uImage 解析；
- `Not hisilicon ADVCA image` 在此处表示选择了非 ADVCA image 解析分支。

它不能证明整个安全启动被关闭，也不能证明所有 2017 构建都使用相同的 128 KiB 外层、槽位大小或签名范围。

### 2018-03：DCAS/TVOS-safe 与前置 Compressed-boot

另一类实机先运行 Compressed-boot，再进入以下 Fastboot：

```text
Compressed-boot v1.0.0
Uncompress......................Ok

Fastboot 3.3.0 (dcas@DCAS-TVOS-SAFE) (Mar 07 2018 - 17:55:19)
CPU:           Hi3798Cv200 (CA)
SDK Version:   HiSTBAndroidV600R002C00SPC031_v2016051915
```

这类配置还观察到：

- 启动对象常使用 `recovery` 名称；
- 环境区域、Android boot page size 和槽位容量与其他产品不同；
- Fastboot 分区的压缩内容必须匹配同一构建的链接地址和解压地址；
- legacy uImage 可以承载 AArch32 bundle，再由 bundle 启动 AArch64 Linux。

因此，`DCAS/TVOS-safe` 和 `Compressed-boot` 描述的是一条具体产品启动链，不能缩写成“2018 Fastboot 通用格式”。

### 2018-08：隐藏命令态和 RAM-only 执行

一份不同的 2018 构建明确显示：

```text
Fastboot 3.3.0 (root@tvos-dcas) (Aug 06 2018 - 15:55:56)
go      - start application at address 'addr'
bootm   - boot application image from memory
tftp    - download or upload image via network using TFTP protocol
```

在同一运行时，8 字节 AArch32 探针 `mov r0, #42; bx lr` 经 `go` 返回 `0x2a`，证明该命令把 RAM 地址作为 AArch32 函数入口调用。

对这一特定构建的反汇编还表明，手工 `bootm` 会在输入地址上固定增加 `0x110`：前 `0x100` 字节是签名区，`0x100..0x107` 是标签，`0x108..0x10f` 是小端镜像范围字段，Android header 从 `+0x110` 开始。

该偏移只属于这一二进制，不能套给 2018-03 或 2017 构建。

RAM-only 表示断电后不保留，不表示操作没有风险。

RAM 程序仍可能改写存储、环境和外设寄存器。

公开探针必须明确标注 `RAM-ONLY / NOT-FLASHABLE`，并默认排除 `saveenv`、`erase`、升级及任意 `mmc write`。

### 2020：TVOS/CCDT 和受限 rescue

2020 样本由 Compressed-boot 进入 `jenkins@ubuntu` 构建的 AArch32 Fastboot。

正常冷启动依次涉及 bootargs、TrustedCore 和 kernel 认证，自编 kernel 在进入 Linux 前出现：

```text
CCDT_Novel_CaCheckByName: check kernel fail! magic str failed!
CCDT_Novel_Android_Authenticate: Fail to verify kernel! (ret: -1)
```

这能证明认证失败发生在 Linux 入口之前。

扩大分区、重做普通 Android header、更换 rootfs 或复制一段 magic 都不会改变签名结论。

项目仅在已经运行的隐藏命令态验证过一次性 RAM rescue。

它没有修改正常冷启动的信任链，也不能证明该能力会跨断电保留。

## Novel、ADVCA、CCDT 和环境 CRC

### 安全层的作用不能互相替代

| 层 | 可以确认的作用 | 不能从日志推出的内容 |
| --- | --- | --- |
| Novel/Noval | 产品定制的 bootargs/kernel 包装检查和 CA 模式选择 | 算法、密钥和全部签名覆盖范围 |
| Hisilicon ADVCA | 镜像头、长度、块表、hash、RSA 签名及可选加密 | 量产私钥、全部 OTP 状态 |
| CCDT/TrustedCore | 客户安全链和可信执行环境相关认证 | 仅凭一条错误日志无法还原完整策略 |
| OTP/安全配置 | 信任根、锁位、客户身份和 self-boot 等永久策略 | 另一台设备或 SDK 默认值不能代表本机 fuse |

`NovelCheck ... ok` 表示当前对象通过检查，不表示检查被禁用。

把原厂 header 复制到新 kernel 前面通常无效，因为签名会覆盖长度、hash 或后续数据。

### 环境 CRC 只证明完整性

U-Boot environment CRC 只保护环境数据没有发生意外损坏。

即使重新计算 CRC 并成功 `saveenv`，Novel、ADVCA 或 CCDT 仍可能拒绝修改后的 bootargs 或启动对象。

因此必须区分：

```text
environment CRC 正确
    ≠ bootargs 获得有效签名
    ≠ kernel 容器获得有效签名
    ≠ OTP 信任根已经改变
```

## UART 启动与下载机制

### 隐藏 UART bootstrap

对已运行 CV200 下载程序的源码和反汇编交叉检查确认了一个厂商 UART 命令环。

它不是普通串口控制台，也不是 Android USB Fastboot。

已确认协议特征如下：

- 串口参数为 115200 baud、8 data bits、no parity、1 stop bit、no flow control；
- DTR 和 RTS 应保持关闭，避免 USB-UART 转接器的握手线误接到板级复位或电源控制；
- Fastboot 检查 SYSCTRL `SC_GEN12` 是否为 `0x444f574e`，即 ASCII `DOWN`；
- `SC_GEN2=1` 选择 UART 作为 bootstrap transport；
- 条件成立后打印 `start download process.`；
- 命令帧使用 `0xAB` 五字节长度/CRC 头和 `0xCD` command marker；
- CRC 算法为 CRC16-CCITT；
- 成功返回 `0xAA` ACK，失败返回 `0x55` NAK；
- 命令完成以 `[EOT](OK)` 或 `[EOT](ERROR)` 结束；
- payload 最终进入该 Fastboot 二进制自己的 `run_command()`，所以可用命令仍由具体构建决定。

SYSCTRL 在两个执行视图中的地址为：

| 寄存器 | AArch32 bootloader 映射 | AArch64 Linux 物理别名 | 已确认用途 |
| --- | --- | --- | --- |
| `SC_GEN2` | `0xf8000088` | `0x08000088` | 值 `1` 选择 UART |
| `SC_GEN12` | `0xf80000b0` | `0x080000b0` | 值 `0x444f574e` 请求下载入口 |

这两个地址是同一 SYSCTRL 控制器在不同执行视图中的别名，不是两套独立寄存器。

地址和 magic 是已审计实现的证据，仍不能证明另一份签名 Fastboot 保留了相同命令环。

### 从 Linux 请求下一次 UART 下载

普通 Linux `reboot` 只执行平台 restart handler，不会自动设置下一次启动为 UART 下载。

合理的正式接口应使用 `syscon-reboot-mode` 或等价的 reboot notifier：

```text
reboot(LINUX_REBOOT_CMD_RESTART2, "hisi-download")
  -> 写 SC_GEN2 = 1
  -> 读回并执行必要屏障
  -> 写一次性 SC_GEN12 = 0x444f574e
  -> 再次读回并执行必要屏障
  -> 调用正常 SP805 restart handler
  -> Fastboot 清除启动标志并打印 start download process.
```

当前已确认 SYSCTRL syscon 和 SP805 restart handler，但 reboot-mode 寄存器映射和 reset 后保留行为尚未形成生产契约。

因此这仍是 `CANDIDATE / READ-ONLY-DESIGN`，不能在公开说明中写成已经可用的启动选项。

正式实现不应让用户态通过 `/dev/mem` 任意写物理寄存器。

如果 watchdog reset 后没有出现下载标志，应停止实验并回到正常冷启动，不能循环写 magic 或盲目触发复位。

### BootROM programmer 与 UART bootstrap 的边界

厂商 UART bootstrap 说明“已经运行的 Fastboot 愿意接收命令”。它不能证明：

- BootROM 会接受任意未签名 programmer；
- DDR-only programmer 可以绕过芯片安全状态；
- Android USB Fastboot 能访问相同命令；
- UART 命令能够写所有分区；
- 一台板验证过的命令可以直接用于另一份 Fastboot。

## Linux 入口协议和外围状态

### CPU0 入口必须满足的状态

进入 AArch64 Linux 前，当前实现保证：

- CPU0 处于 AArch64 non-secure EL2；
- `x0` 指向 8-byte 对齐的 DTB，`x1..x3` 为零；
- MMU 关闭；
- 数据缓存处于 Linux 启动协议允许的关闭状态，复制的数据已清理到一致性点；
- 中断处于屏蔽状态；
- 24 MHz architected timer 已建立一致频率；
- GICv2 已从 EL3 配置为 Linux 可以接管的 non-secure 状态；
- Linux Image 位于其链接/入口契约对应的物理地址。

完整要求以 [Booting AArch64 Linux](https://docs.kernel.org/arch/arm64/booting.html) 为准。

### Linux 不会自动修复全部固件遗留状态

AArch64 状态转交成功只证明 CPU 可以进入内核。

Fastboot 仍可能留下以下外围状态：

- PLL、时钟 mux、gate 和 reset 位；
- eMMC HS400 tuning 和 I/O 电压；
- HDMI/VDP framebuffer、像素时钟和 PHY；
- USB PHY、GMAC PHY、DVB/TSI 和 DMA；
- watchdog、timer、中断 pending 位和缓存中的旧 DMA 数据。

每个 Linux 驱动必须明确选择“完整复位后初始化”还是“继承并验证固件状态”。

不能把外围故障一律归因给 AArch32 → AArch64，也不能假设 warm reset 会自动恢复所有控制器。

例如，TTL 已经出现 `Run /sbin/init as init process` 时，HDMI 黑屏通常属于 VDP/HDMI/像素链路或用户态显示问题，而不是 CPU 仍处于 AArch32。

反之，如果 `Starting kernel` 后立即 undefined instruction，则应先检查执行状态切换，不应继续调整 framebuffer 参数。

## 故障定位

### 按最后一条可靠日志定位

```text
没有 BootROM/Fastboot 横幅
  -> 电源、晶振、启动介质、BootROM 下载状态或串口接线

Compressed-boot 解压失败
  -> Fastboot 压缩容器、长度、链接地址或解压目标

Novel/ADVCA/CCDT authenticate fail
  -> Linux 尚未执行；检查签名外层、长度、hash 和目标固件策略

legacy uImage CRC/format fail
  -> 检查 uImage header、压缩数据、load/entry 和读取范围

打印 Starting kernel 后 undefined instruction
  -> AArch32 直接执行了 AArch64 指令，或 reset 状态选择失败

看到 [A32] requesting AArch64 warm reset，未看到 [A64]
  -> 检查 cache clean、RVBAR、AArch64 reset mode、RMR 和 entry 内容

看到 [A64] entered at EL3，未看到 entering Linux at EL2
  -> 检查 EL3 寄存器、GIC、次级核电源顺序和 ERET 状态

看到 [A64] entering Linux at EL2，未看到 Booting Linux
  -> 检查 x0、DTB、Image 地址、覆盖、缓存和内核入口

Linux 启动但只有一个 CPU
  -> 检查 reserved-memory、spin-table、cpu-release-addr 和次级核 reset/GIC

Linux 找不到 rootfs
  -> 检查 bootargs、分区号、eMMC 模式、驱动和文件系统

Linux 已进入用户态但某个外设异常
  -> 检查对应驱动、DTS、时钟/reset/pinmux 和固件遗留状态
```

### 一次成功转交的最小连续证据

```text
[A32] unpacking bundled ARM64 payload
[A32] requesting AArch64 warm reset
[A64] entered at EL3; dropping to EL2
[A64] GICv2 handed to non-secure Linux
[A64] CPU1-3 released into spin-table
[A64] entering Linux at EL2
Booting Linux on physical CPU ...
Linux version ...
Machine model: ...
SMP: Total of 4 processors activated.
CPU: All CPU(s) started at EL2
VFS: Mounted root ...
Run /sbin/init as init process
```

这组日志证明的是“Fastboot + A32 bundle + A64 handoff entry + DTB + Linux Image”的组合成功，不能单独归功于 `IH_ARCH_ARM64`、Fastboot 版本或 rootfs 架构。

## 构建、封装和刷写验证

### 构建阶段

每次构建至少应验证：

- A32 bundle 是 ARM EABI/AArch32 ELF；
- EL handoff entry 和 Linux Image 是 AArch64；
- DTB 能由 `fdtget`/`fdtdump` 正常解析；
- entry、DTB、Image 和最终 bundle 分别记录 SHA-256；
- entry、DTB 和 Image 没有超过内存布局上限；
- `0x11000000..0x1100ffff` 在 DTB 中保留；
- CPU1–3 的 `cpu-release-addr` 与 entry 中 mailbox 地址一致；
- 诊断开关不会悄悄改变生产镜像 ABI。

### 封装阶段

应从内到外验证每一层：

1. AArch64 Image、DTB 和 handoff entry；
2. AArch32 bundle 的边界与入口；
3. gzip 数据能完整解压且 hash 一致；
4. legacy uImage 的 data size、CRC、load 和 entry；
5. Android boot header 的 page size、kernel size 和偏移；
6. Novel/CCDT/客户外层的总长度和认证范围；
7. 最终文件满足目标分区的固定大小和填充规则。

不要让外层 CRC 正确掩盖内层地址错误，也不要让内层 bundle 可执行掩盖外层签名不兼容。

### 写入阶段

生产写入需要：

- 在主机端记录最终镜像大小和 SHA-256；
- 核对目标板、Fastboot 完整横幅、启动分区、偏移和长度；
- 拒绝超出已证明分区边界的镜像；
- 写入完整目标分区要求的容器，而不是裸 payload；
- 写入后读回完整范围，并要求摘要与主机输入一致；
- 从上电开始保存完整 TTL，而不是只截取 Linux 部分；
- 至少验证一次冷启动、一次软重启、四核上线和 rootfs 挂载。

RAM-only 的 `go`、`bootm` 和 `tftp` 不写 eMMC 时仍属于主动操作，因为它们能够执行任意代码并改变当前硬件状态。

应与只读的 `version`、`help`、`printenv` 和内存/寄存器读取分开授权。

## 公开参考资料

- [Hi3798C V200 Brief Data Sheet](https://www.silicondevice.com/file.upload/images/Gid1562Pdf_Hi3798CV200.pdf)：Cortex-A53、外设、启动下载和芯片能力概览。
- [Hi3798CV200 boot 源码镜像](https://github.com/satpt/3798cv200_boot)：`hi3798cv2x`、programmer、ADVCA 条件和厂商 Fastboot 源码结构参考；不代表任一量产二进制的精确源码。
- [HiSTBLinux R005 SPC050](https://github.com/JasonFreeLab/HiSTBLinuxV100R005C00SPC050) 与 [SPC060](https://github.com/JasonFreeLab/HiSTBLinuxV100R005C00SPC060)：用于理解后期 CV200 SDK 构建系统、ATF 平台端和外设实现。
- [U-Boot `bootm`](https://docs.u-boot.org/en/latest/usage/cmd/bootm.html)：legacy image、FIT、kernel、initrd 和 FDT 参数的一般语义。
- [U-Boot Android Boot Image](https://docs.u-boot.org/en/stable/android/boot-image.html)：Android boot v0/v1/v2 的公开格式。
- [AOSP sparse image header](https://android.googlesource.com/platform/system/core/+/master/libsparse/sparse_format.h)：`0xCAC1`、`0xCAC2`、`0xCAC3` 和 `0xCAC4` 的原始定义。
- [Booting AArch64 Linux](https://docs.kernel.org/arch/arm64/booting.html)：DTB、`x0`、异常级、DMA、缓存、MMU 和多核入口的正式要求。
- [Hi3798CV200 Poplar U-Boot README](https://github.com/ARM-software/u-boot/blob/master/board/hisilicon/poplar/README)：公开开发板的 l-loader → TF-A → AArch64 U-Boot 示例；它不是量产 TVOS Fastboot 的替代物。
- [Trusted Firmware-A](https://github.com/ARM-software/arm-trusted-firmware) 与 [Trusted Board Boot](https://github.com/ARM-software/arm-trusted-firmware/blob/master/docs/design/trusted-board-boot.rst)：EL3 固件和通用信任链概念参考。
