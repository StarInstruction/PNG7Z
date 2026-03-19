这篇《6 Non Volatile Memory (NVM) Subsystem》不是单纯介绍某个寄存器，而是在讲 **AURIX TC3xx 整个非易失性存储子系统怎么组织、怎么读写、怎么保护、怎么安全更新**。它把 NVM 相关内容按体系结构重新整理成一个大章节，核心覆盖 **DMU、PFI、NVM、UCB** 四大部分。文档开头就明确说，这一章是把原先 PMU 相关内容按 TC3xx 架构重组后的结果，应用可访问的 FSI 寄存器和 PFLASH 读状态/控制寄存器也在这里描述

我先用“先总后细”的方式给你讲。

------

## 1. 这篇文档到底在讲什么

一句话概括：

**它在解释 AURIX TC3xx 里，程序代码、常量、EEPROM 仿真数据、启动配置、安全配置这些内容，分别存在哪里、怎么访问、怎么擦写、出错后怎么处理，以及如何保证安全性和可靠性。**

文档把 NVM 子系统拆成这些核心部件：

- **DMU（Data Memory Unit）**：负责控制 Flash 命令序列
- **FSI（Flash Standard Interface）**：真正执行擦除、编程、校验
- **PFLASH**：存程序代码和常量
- **PFI（Program Flash Interface）**：CPU 和 PFLASH 之间的高速点对点读通路
- **DFLASH**：做 EEPROM 仿真，也保存配置块
- **UCB（User Configuration Blocks）**：保存保护配置、密码、启动头等
- **CFS（Configuration Sector）**：系统配置数据，用户不可直接访问
- **BROM**：上电启动软件和测试/安全相关 ROM 功能

文档第 2 页的框图（Figure 56）其实就是全章总纲：CPU 通过 PFI 快速读 PFLASH，通过 DMU/FSI 访问和控制 DFLASH、UCB、CFS、BROM。也就是说，**PFLASH 更偏“高性能执行”，DFLASH 更偏“可更新数据与配置管理”**。这一点是理解全章的钥匙。

------

## 2. 先建立几个最重要的概念

这篇文档一上来先定义术语，因为后面所有命令和保护机制都靠这些概念。

### 2.1 擦除和编程的逻辑定义

AURIX 这里定义得很特别：

- **擦除后的 Flash 单元逻辑值是 0**
- **编程后的 Flash 单元逻辑值是 1**

所以“erase”是把一片区域强制成 0，“program”是把位写成 1。文档还解释了 retention（保持时间）和 endurance（擦写寿命）的关系：擦写次数越多，保持能力通常越差。

### 2.2 层级结构：Bank / Sector / Page / Wordline

这是整篇文档最基础的结构概念：

- **Bank**：一个 Flash 模块可分成多个 bank
- **Physical Sector**：物理隔离的大块区域
- **Logical Sector**：可被单次擦除命令操作的逻辑扇区
- **Mini Sector**：UCB 里的更小分组
- **Page**：最小可编程单位
- **Wordline**：更大的物理行，是很多页的集合

关键尺寸非常重要：

- **PFLASH page = 32 字节 + 22-bit ECC**
- **DFLASH page = 8 字节 + 22-bit ECC**
- **PFLASH wordline = 1024 字节**
- **DFLASH wordline = 512 字节（single-ended）/ 256 字节（complement sensing）**
- **PFLASH burst = 8 页 = 256 字节**
- **DFLASH burst = 4 页 = 32 字节**

这意味着：
**PFLASH 适合高吞吐程序存储，DFLASH 更适合小粒度配置/数据写入。**

------

## 3. TC3xx 相比老 AURIX 改了什么

这一节其实是在告诉你：为什么这个章节看起来这么复杂，因为 TC3xx 的 Flash 架构升级了。

文档列出的关键变化包括：

- PFLASH 读取改成了专用高性能接口 **PFI**
- Flash 结构和容量重新设计
- **HSM PCODE** 支持增强
- **CFS 从别处移到了 DFLASH**
- **Erase Counter** 被放到各个 PFLASH 里
- 所有 Flash/UCB/CFS/寄存器有各自独立地址空间
- DFLASH 支持 **Complement Sensing**
- 引入 **Dual UCB** 概念
- 加入 **SOTA（Software Update Over the Air）** 支持
- PFLASH 只保留 **safe ECC**，取消旧式 legacy ECC

这段的真正含义是：

**TC3xx 的 NVM 不再只是“能读能写的 Flash”，而是一个面向安全、启动配置、在线升级、HSM 隔离设计出来的复杂子系统。**

------

## 4. PFLASH、DFLASH、UCB、CFS 分别扮演什么角色

------

### 4.1 PFLASH：程序代码和常量

PFLASH 主要用途有两个：

1. 存放程序代码和只读常量
2. 实现擦除计数器（Erase Counters）

PFLASH bank 按容量不同，逻辑扇区范围不同：

- 1MB bank：S0~S63
- 2MB bank：S0~S127
- 3MB bank：S0~S191

每个逻辑扇区是 **16KB**。

这说明在 PFLASH 里，**最常见的擦除粒度是逻辑扇区**，不是 page。

------

### 4.2 PF0 下方区域还能承载 TP 和 HSM PCODE

文档第 6、7 页这部分很关键，也是很多人第一次看会 confused 的地方。

PF0 的低地址区域，不只是普通代码区，它还可能被分配给：

- **TP（Tuning Protection / Secure Watchdog）**
- **HSM PCODE**
- 普通 CPU CODE & DATA

如果启用了 TP，那么：

- **PF0 S0** 专门给 TP 用，不能再配置为 HSM exclusive
- **PF0 S1~S7** 可做 TP extended memory
- **PF0 S8~S39** 可做 HSM PCODE（并行 TP + HSM 的配置）

第 6 页的图（Figure 57）画得很直观：左边是并行 TP+HSM，右边是纯 HSM 模式。
它想表达的是：

**PF0 前 40 个 16KB 逻辑扇区不是普通意义上的“随便放程序”，而是安全/启动相关资源池。**

而且为了支持 SOTA，另一个 PFLASH bank 的 S0~S39 也能做类似配置。

------

### 4.3 Erase Counter：记录擦除了什么

每个 PFLASH bank 里有一个专门的 **Erase Counter** 逻辑扇区，大小 **16KB**。
它是 **应用可读、应用不可写** 的，用来自动记录每次执行 `Erase Logical Sector Range` 时擦除了哪些逻辑扇区。

它的记录项包含：

- marker
- 起始逻辑扇区地址
- 擦除扇区数量

而且分成：

- **低优先级区域**
- **高优先级区域**

是否进高优先级区域，由 **UCB_ECPRIO** 配置决定。

这背后的工程意义是：

**系统不仅允许擦写，还在“记账”擦写历史，用于寿命管理、诊断和安全审计。**

------

### 4.4 DFLASH：EEPROM 仿真和配置

DFLASH 的主要任务有三类：

- 做 EEPROM 仿真
- 保存 UCB
- 保存 CFS（系统配置）

DFLASH0 包含：

- EEPROM 区
- UCB 区
- CFS 区

DFLASH1 在带 HSM 的器件里，通常给 HSM 做 EEPROM 仿真，并与应用隔离。

DFLASH 的一个重点是它支持两种模式：

- **Single-ended**
- **Complement sensing**

而且 **DFLASH0 与 DFLASH1 必须工作在同一种模式**。

------

### 4.5 EEPROM 仿真不是“直接往某个地址反复写”

文档花了不少篇幅讲 EEPROM emulation，真正想说的是：

**DFLASH 用来仿 EEPROM 时，软件必须做磨损均衡和断电恢复设计，不能把它当普通 RAM 用。**

它要求 EEPROM 仿真算法具备：

- 把写操作分散到更大区域，提升寿命
- 保证各单元擦写次数尽量均匀（wear leveling）
- 能回收旧数据
- 遇到中断/掉电后，启动时能恢复有效数据

文档尤其强调 **wordline failure**：

ECC 很擅长处理 bit/bitline 错误，但对整条 wordline 失效无能为力。
所以它建议的鲁棒写入流程是：

1. 编程前先把同一 wordline 上其他有效页备份到 SRAM
2. 编程新页
3. 对新页和备份页做比较
4. 如果失败或 PVER，则把这些数据迁移到另一条 wordline
5. 最多重试 2 次，再失败就认为超出规格条件

这个部分非常实战。它是在告诉你：

**DFLASH 做 EEPROM 仿真，软件策略和硬件 ECC 同样重要。**

------

## 5. 安全和可靠性：这是全文最核心的思想

这篇文档最大的价值，不是“教你发命令擦 Flash”，而是告诉你：

**Flash 操作必须在安全和功能安全框架下进行。**

------

### 5.1 两层保护：Security Layer + Safety Layer

文档一开始就强调，NVM 微架构同时包含：

- **Security layer**
- **Safety layer**

其中：

**Security layer** 主要处理：

- 读保护
- 写保护
- 密码保护
- HSM 独占等

**Safety layer** 主要处理：

- 主机访问控制
- ENDINIT 保护
- ECC 校验
- PFI 局部锁步
- 读路径完整性监控
- 向 SMU 上报错误/告警

------

### 5.2 Safety ENDINIT：PFLASH 修改前必须解保护

文档反复强调：

**所有会修改 PFLASH 内容的命令序列都受 Safety ENDINIT 保护。**

如果没先去掉 ENDINIT 就去改 PFLASH，会产生 **PROER（Protection Error）**。

而为了支持 SOTA，如果启用了 `SWAPEN`，DMU 会对当前不用于执行代码的那些“inactive PFLASH”移除 Safety ENDINIT 限制，方便后台更新。

这说明：

**TC3xx 支持在线升级，但不是“随便写”，而是严格限定只能改不在执行的那部分程序区。**

------

### 5.3 PFLASH ECC 和 DFLASH ECC 是不一样的

这是非常重要的区别。

#### PFLASH ECC

PFLASH 的 ECC 是 **safe ECC**，计算覆盖：

- 256-bit 数据
- 地址位

因此它不仅能发现数据错误，还能发现寻址错误。文档明确写出：

- 纠正 1-bit / 2-bit 错误
- 检测全部 1/2/3-bit 错误
- 检测 >99% 白噪声错误向量
- 检测 all-0 / all-1 异常
- 检测 addressing errors

但它有个副作用：

**擦除后的 PFLASH 区域不能靠普通读判断“是否全 0”，因为会出现 ECC 错误。**
所以文档后面专门提供 `Verify Erased` 命令，而不是让你直接读。

#### DFLASH ECC

DFLASH 的 ECC 只覆盖 **64-bit 数据**，不覆盖地址。
因此：

- 不能检测“读对了数据但地址错了”这类错误
- Single-ended 模式下，擦除区域可读出 ECC 正确的 0
- Complement sensing 模式下，擦除态可能读出任意结果甚至错误纠正结果

所以你可以理解成：

- **PFLASH ECC 更偏安全执行**
- **DFLASH ECC 更偏数据可靠性，但不负责地址保护**

------

### 5.4 告警体系很完整

文档列出大量会报给 SMU 的告警：

- SBAB/DBAB/MBAB/ZBAB 满
- SBER / DBER
- NVMCERR
- PFI/EDC/ECC checker 错误
- FLASHCON 参数非法
- PFLASH 读路径监视错误
- SRI 地址阶段/写数据阶段 ECC 错误

这意味着 NVM 子系统不是“静默出错”，而是设计成了 **可观测、可诊断、可联动安全管理单元** 的系统。

------

## 6. DMU：整篇文档的操作核心

如果说前面讲的是“存储结构”，那 DMU 章节讲的就是“怎么动它”。

DMU 的职责包括：

- 读 DFLASH0 / DFLASH1 / UCB / CFS
- 执行所有 PFLASH/DFLASH 操作
- 作为 Flash 的安全层
- 根据 UCB 的配置执行密码保护和访问控制
- 对 DFLASH1 做 Host/HSM 双通道管理
- 做 HSM exclusivity 检查

所以可以把 DMU 理解为：

**Flash 的“总调度器 + 安全门卫 + 命令解释器”。**

------

## 7. 读访问是怎么工作的

### 7.1 PFLASH 读

PFLASH 通过 PFI 提供高速路径。
本地 CPU 通过 DPI 点对点访问，支持 cacheable / non-cacheable 访问方式。

### 7.2 DFLASH 读

DFLASH 读只能是：

- **SRI 单次传输**
- **non-cacheable 地址空间**

如果是 block transfer，会直接 bus error。

这很好理解：
**PFLASH 追求性能，DFLASH 追求可控和正确性。**

------

### 7.3 读等待周期必须正确配置

PFLASH 读等待周期配置在 `HF_PWAIT`，DFLASH 读等待周期配置在 `HF_DWAIT`。文档给了具体算例：

- PFLASH 例子：`tPF=30ns, tPFECC=10ns, fSRI=200MHz`
  - `RFLASH = 5`
  - `RECC = 1`
- DFLASH 例子：`tDF=100ns, tDFECC=20ns, fFSI=100MHz`
  - `HF_DWAIT.RFLASH = 9`
  - `HF_DWAIT.ECC = 1`

这个部分的工程意义很强：

**Flash 时序不是自动适配的，时钟一变，wait cycle 也必须跟着改。**

------

## 8. Flash 操作不是“写内存”，而是“发命令序列”

文档里最重要的实践思想之一就是：

**除 memory-mapped read 以外，所有 Flash 操作都通过命令序列完成。**

写访问的解释方式是：

- 写 PFLASH 地址范围：**拒绝，bus error**
- 写 DFLASH0/1 地址范围：**被解释成命令周期**

也就是说，CPU 不是在“往 Flash 地址里写数据”，而是在“往命令接口里按特定顺序喂控制字”。

------

## 9. Host 和 HSM 两套命令解释器

DMU 里有两个 Command Sequence Interpreter：

### Host CSI

- 可操作 PFLASH、DFLASH0、UCB，以及在非 HSM-exclusive 条件下的 DFLASH1
- 通过 DFLASH0 地址段解码命令
- 任何片上 bus master 都可使用

### HSM CSI

- 仅在 DFLASH1 是 HSM-exclusive 时操作 DFLASH1
- 通过 DFLASH1 地址段解码命令
- 只有具有 H 访问属性的主机可用

这部分的本质是：

**TC3xx 的 DFLASH1 可以被硬件机制切成“安全世界自己的 Flash”。**

------

## 10. Page Mode：所有编程动作的起点

DMU 要编程时，必须先进入 **Page Mode**。
Page Mode 的作用是：

- 让装配缓冲区（assembly buffer）进入可装载状态
- 后续 `Load Page` 把数据送进去
- 最后 `Write Page` / `Write Burst` 真正发起写入

这就是为什么文档后面所有编程流程都长得像：

1. Enter Page Mode
2. Load Page
3. Write Page / Burst

------

## 11. 命令序列是全文最“硬核”的部分

文档给了完整命令总表，包括：

- Reset to Read
- Enter Page Mode
- Load Page
- Write Page
- Write Page Once
- Write Burst
- Write Burst Once
- Replace Logical Sector
- Verify Erased Page
- Verify Erased WL
- Verify Erased Logical Sector Range
- Erase Logical Sector Range
- Resume NVM Operation
- Disable Protection
- Resume Protection
- Clear Status

这一组命令基本构成了所有 Flash 操作的“指令集”。

------

## 12. 这些命令各自的实际意义

------

### 12.1 Reset to Read

把命令解释器恢复到初始状态，并退出 Page Mode。
如果前面命令发错了，靠它“清场”。

------

### 12.2 Load Page

把数据装到 page assembly buffer。
要求同一页的传输宽度一致，要么全 32 位，要么全 64 位；否则会报 `SQER`。
如果装太多，溢出部分会被丢弃，后续写命令报错。

------

### 12.3 Write Page

编程一页数据。
执行后会退出 Page Mode，并把对应 bank 标记为 BUSY。
如果没先进入 Page Mode，或者地址不对，会报 `SQER`；如果该页有保护，报 `PROER`；如果编程过程出错，报 `PVER`。

------

### 12.4 Write Page Once / Write Burst Once

这两个“Once”版本的意思是：

**写之前先检查页/页组是否处于擦除态。**

如果不是擦除态，就报 `EVER`。
而且在 write-once 保护区域，只允许用 Once 版本。

------

### 12.5 Write Burst

用来做高吞吐编程：

- PFLASH 一次 8 页 / 256B
- DFLASH 一次 4 页 / 32B

如果要做高速写入，优先考虑它。

------

### 12.6 Erase Logical Sector Range

擦除一段逻辑扇区。
但限制很多，例如：

- 范围必须在同一 physical sector 内
- PFLASH 范围不能超过 512KB
- DFLASH 范围不能超过 256KB addressable memory
- UCB 一次最多只能擦 1 个
- 地址必须对齐
- 不能是受保护区域
- global read protection 开启时也不行

这说明 TC3xx 的擦除操作被约束得非常严，不允许“随意跨区乱擦”。

------

### 12.7 Verify Erased 系列

这是为了弥补“PFLASH 不能通过普通读判断是否已擦除”的问题。

它提供：

- Verify Erased Page
- Verify Erased WL
- Verify Erased Logical Sector Range

尤其在 complement sensing 的 DFLASH 下，verify 的语义其实是：

**检查它是否“空白到足以重新编程”**，而不一定意味着普通意义上的“全 0 可读”。

------

### 12.8 Replace Logical Sector

这是很高级的功能。
当某个 PFLASH 逻辑扇区在擦写中出现硬故障时，可以把它重映射到冗余扇区。成功后会更新 `UCB_REDSEC`。

这个功能表明：

**TC3xx 不只是“发现坏块”，而是支持受控的坏块替换。**

------

### 12.9 Disable Protection / Resume Protection

通过提供正确密码，临时关闭某个 UCB/Flash 的保护；
再通过 Resume Protection 恢复。
如果密码错一次，直到下次 application reset 前，后续 Disable Protection 都会失败并报 `PROER`。

这个机制明显是为了防暴力尝试。

------

### 12.10 Clear Status

清操作和错误标志。
但文档特别提醒：**清标志不等于解决根因。**

------

## 13. 错误码背后的含义

文档里的几类错误要分清：

- **SQER**：命令序列错误，通常是命令顺序、地址、对齐、状态机状态不对
- **PROER**：保护错误，常见于密码错、写保护未解除、ENDINIT 没去掉
- **PVER**：编程校验错误
- **EVER**：擦除校验错误
- **OPER**：更严重的操作错误，往往意味着当前操作被中止，通常需要 reset 后再继续

工程上你可以这么理解：

- `SQER`：你“流程写错了”
- `PROER`：你“权限不够”
- `PVER/EVER`：硬件执行了，但“结果不理想”
- `OPER`：更偏“硬件/运行条件异常”，要慎重处理

------

## 14. Suspend / Resume：长操作可中断，但规则很多

文档专门用了图和多页文字来讲 suspend/resume，说明这是实战高频点。

可被 suspend 的操作包括：

- Write Page / Write Page Once
- Write Burst / Write Burst Once
- Erase Logical Sector Range
- Verify Erased 系列
- Replace Logical Sector

但有几个特别重要的约束：

1. Host 和 HSM 各自最多只能挂起 1 个操作
2. 被挂起的 program/erase 目标区域状态是 **undefined**
3. Resume 时参数必须和原命令完全一致，否则报 `SQER`
4. 从开始/恢复到发 suspend 请求之间，最好至少让操作跑 **~3ms**，推荐平均 **~10ms**（100MHz FSI 条件下）

第 40 页那张图（Figure 60）就是在解释：

- REQ 拉高后，DMU 不会立刻停
- 它要等操作走到可中断点
- 然后 BUSY 清掉，SPND 置位
- Resume 之后再继续执行

这意味着 suspend 不是“抢占式暂停”，而是 **受控停顿**。

------

## 15. Host 和 HSM 竞争时，FSI 还会做时间片调度

当 Host 和 HSM 同时要做 Flash 操作时，FSI 会用时间片：

- CPU 时间片：**50 ms**
- HSM 时间片：**5 ms**

所以并发时：

- CPU 的 DFLASH 操作可能被 HSM 插队延迟 5ms
- HSM 的 DFLASH 操作可能被 CPU 延迟 50ms
- HSM 擦除在时间片冲突下会显著变慢，文档甚至说可放大到约 15 倍

这个点很关键，因为很多人调试 Flash 超时问题时，没意识到是 **Host/HSM 竞争 + time slice** 导致 BUSY 拉长。

------

## 16. 编程和擦除的“推荐工程做法”

这部分非常有价值，几乎就是“官方经验”。

文档建议：

- 修改 PFLASH 前先去掉 Safety ENDINIT
- 执行 PFLASH 编程/擦除的代码，不要从同一个 PFLASH 中运行
- 命令周期要发到 **non-cached 地址空间**
- 切换时钟分频时，只允许读 Flash，不允许做其他 Flash 操作
- 切换时钟前后必须确保 wait cycle 足够
- 启动后的默认 wait cycle 只够 100MHz 场景，不是万能值

还有一个很容易忽视的细节：

### 32-bit Load Page 时要插入 DSYNC

因为 CPU 的 store buffer 可能把两个 32-bit 写合并成 64-bit 写，导致命令顺序失真。
所以文档要求：

**每个 32-bit `ST` 之后插 `DSYNC`**，或者插一个寄存器读也行。

这是真正的“踩坑经验”。

------

## 17. 文档甚至给了“防御式流程模板”

它给了编程/擦除的推荐顺序。

### 编程流程大意

1. Clear Status
2. Enter Page Mode
3. DSYNC
4. 等 PAGE 标志
5. 多次 Load Page
6. Write Page
7. DSYNC
8. 等 PROG 标志
9. 等 BUSY 清零
10. 检查 PVER / OPER
11. 建议再校验数据内容

### 擦除流程大意

1. Clear Status
2. Erase Logical Sector Range
3. DSYNC
4. 等 ERASE 标志
5. 等 BUSY 清零
6. 检查 PVER / EVER / OPER

这部分很适合作为你自己写 Flash driver 的骨架。

------

## 18. reset / 掉电期间做 Flash 操作会怎样

文档专门警告：

**如果 program / erase 过程中发生 reset 或掉电，目标区域会进入 undefined state。**

可能出现：

- 看起来像全 0
- 看起来像旧数据
- 看起来像垃圾数据
- 数据随工作条件变化而变化
- 由于 ECC 纠正，读到的内容甚至更“迷惑”

所以不要指望“读回来看看像不像正常”就判断操作有没有中断。
文档明确说，**单纯靠读 Flash，甚至靠 ECC，本身都不是可靠的中断检测手段。**

------

## 19. 后半部分还讲了什么

你现在看到的重点主要在总览和 DMU。
但这篇文档后面还有很重要的内容：

### 19.1 PFI（Program Flash Interface）

PFI 负责 PFLASH 到 CPU 的高速读路径。它有两条路径：

- **Demand path**
- **Prefetch path**

读出的数据既有未经纠错的，也有经 ECC 纠错后的，PFI 到 CPU 的传输还带 ECC sideband 保护。

这说明 PFLASH 的高性能读，不只是“快”，而是“快且带安全校验”。

### 19.2 PFI 寄存器

PFI 章节还描述了 ECCR / ECCS 等寄存器：

- ECCR 保存最近一次 PFLASH 读取的 ECC checksum
- ECCS 捕获最近一次 PFLASH 读取发现的 ECC 错误

### 19.3 UCB 详细内容

后半部分还有大量 UCB 表，讲不同 UCB 的内容和访问保护，比如：

- `UCB_BMHDx_ORIG / COPY`：Boot Mode Header，给 SSW 评估启动模式用
- `UCB_USER`：含芯片唯一标识 UID，记录 lot/date/wafer/x-y 坐标等信息
- `UCB_SSW`：IFX 生产时写入的只读数据
- `UCB_DBG`、`UCB_PFLASH`、`UCB_DFLASH` 等：用于各种读写保护和密码保护机制

所以这篇文档不是只给“操作方法”，也给了 **芯片安全配置数据结构**。

------

## 20. 你可以把整篇文档浓缩成这几个结论

### 第一层结论：它讲的是“存储子系统架构”

不是某个单点寄存器，而是从架构到命令、从性能到安全、从启动到在线升级的完整设计。

### 第二层结论：PFLASH 和 DFLASH 分工完全不同

- PFLASH：高性能代码存储 + 安全 ECC + PFI
- DFLASH：EEPROM 仿真 + UCB/CFS + 小粒度写入 + HSM 数据区

### 第三层结论：Flash 不是“像 RAM 一样写”

所有写/擦/校验都走命令序列，而且要经过状态机、保护检查、BUSY 管理、错误反馈。

### 第四层结论：安全和功能安全是主线

ENDINIT、密码、UCB、HSM 独占、ECC、SMU 告警、读路径监控，这些不是附加功能，而是系统核心。

### 第五层结论：实战上最容易踩坑的是

- 没去 ENDINIT 就改 PFLASH
- 命令发到 cacheable 地址
- 32-bit Load Page 不加 DSYNC
- 忽略 complement sensing 的语义差异
- suspend/resume 参数不一致
- 并发 Host/HSM 时间片导致超时估算错误
- reset 中断 Flash 操作后还试图用“直接读”判断状态

------

如果你愿意，我下一步可以继续按这三种方式之一往下讲：**按章节逐页精讲、专讲 DMU 命令序列、或者专讲 UCB/保护机制。**

下面我按你的 13 个点，一个一个用“**先说人话，再说文档里的准确定义**”的方式讲。

------

## 1. 层级结构：Bank / Sector / Page / Wordline

这几个词最容易混。你可以先把 Flash 想成一本大仓库账本：

- **Bank**：一整栋仓库里的一个分库
- **Sector**：分库里的大区块
- **Wordline**：大区块里的“一整行存储单元”
- **Page**：最小写入包

文档里的正式定义是：

- 一个 Flash module 里可以有多个 **bank**
- 一个 bank 里的 NVM 再分成 **physical sector** 和 **logical sector**
- **logical sector** 是可以一次擦除的单位
- **page** 是最小可编程单位
- **wordline** 是一组更大的对齐字节区域

尺寸上，文档给得很明确：

- **PFLASH page = 32B**
- **DFLASH page = 8B**
- **PFLASH wordline = 1024B**
- **DFLASH wordline = 512B（single-ended）/ 256B（complement sensing）**
- **PFLASH burst = 8 pages = 256B**
- **DFLASH burst = 4 pages = 32B**

再结合 PFLASH 的 sector 结构看，会更清楚：PFLASH 的逻辑 sector 是按 **16KB** 组织的，例如 1MB bank 有 S0~S63，2MB bank 有 S0~S127，3MB bank 有 S0~S191

### 你真正要记住的是

- **写**：按 **page**
- **擦**：按 **logical sector**
- **底层物理组织**：按 **wordline**
- **更高一层并发/隔离单位**：按 **bank**

------

## 2. HSM PCODE

先说人话：

**HSM PCODE 就是给 HSM（硬件安全模块）运行的程序代码区。**

它不是普通 CPU 程序区，而是安全域自己的代码存储区域。文档说，PF0 的低地址逻辑 sector 可以被配置成支持 **TP** 和 **HSM Program Code (PCODE)**

进一步地：

- PF0 的 **S0~S39** 可以配置成 **HSM_exclusive**
- 这样就可以把 HSM 的程序代码放在这些 sector 里
- 这些 HSM PCODE sector 还可以被配置成 **HSM locked forever**

如果要同时支持 TP 和 HSM，布局是：

- **S0**：TP 专用
- **S1~S7**：TP extended memory
- **S8~S39**：HSM PCODE
- 而且 TP、HSM PCODE、CPU 区间必须连续

### 直观理解

普通 CPU 像“主系统”，HSM 像“保险柜里的独立小电脑”。
**HSM PCODE 就是这个“小电脑”的固件代码。**

------

## 3. CFS

**CFS = Configuration Sector（配置扇区）**

它是系统配置数据区，但**用户不能直接访问**。文档在总览里就写了：CFS 用来存 system set-up data，且 **not accessible by the user**

它放在 DFLASH0 里，DFLASH0 包含三块内容：

- EEPROM 区
- UCB 区
- CFS 区

CFS 自己也是按 sector 组织的，比如 CFS0 到 CFS15，每个 4KB

### 你可以把它理解成

- **UCB** 更像“安全/启动/保护配置块”
- **CFS** 更像“系统内部配置块”

它们都在 DFLASH 体系里，但 **CFS 更偏系统内部使用，不给应用直接碰**。

------

## 4. Complement Sensing

这个概念确实很绕。

先说结论：

**Complement Sensing 是 DFLASH 的一种读取/存储解释模式。它不是普通的“直接读 0/1”，而是依赖互补数据对来判断内容。**

文档说：

- DFLASH0 和 DFLASH1 必须工作在同一种模式：**single-ended** 或 **complement sensing**
- 在 **single-ended** 模式下，擦除区读出来是 ECC 正确的 0
- 但在 **complement sensing** 模式下，擦除态是 `(erased/erased)` 这种配对，不符合“互补数据结构”，所以读出来可能是**任意结果**，甚至可能被 ECC 做出“错误纠正”或报多比特错误

文档还特别提醒：

在 complement sensing 模式下，`Verify Erased` 命令的含义不是“这块一定全 0 可读”，而是：

**这些 cell 的状态已经足够空白，可以在不预擦除的前提下重新编程。**
甚至某些合法的 complement 数据，经过一定历史操作后，也可能“看起来像 erased”

### 直观理解

- **Single-ended**：像普通判断电压高低
- **Complement sensing**：像看一对互补信号是不是匹配

所以 complement sensing 模式下，**“空白”和“读出来全 0”不是一回事**。

------

## 5. Erase Counters

这个非常像“Flash 擦除日志”。

文档说，每个 PFLASH bank 里都有一个专门的 **16KB Erase Counter logical sector**，应用里它是**只读**的

每次执行 `Erase Logical Sector Range`，FSI 都会自动记录：

- 被擦除的起始逻辑 sector 地址
- 擦除了多少个 sector

它写成一个 **256-bit log entry**

而且还分成：

- **Low priority area**
- **High priority area**

是否记录到高优先级区，由 `UCB_ECPRIO` 指定某些逻辑 sector 的优先级来决定

### 它有什么用

不是用来存用户数据，而是用来：

- 记录擦除历史
- 做寿命/维护分析
- 做诊断或审计

### 一个细节

它通过 marker 判断最后一条有效记录。读到 marker 为 0 的 entry，后面的 entry 就是空的；空 entry 可能会触发 ECC 多比特失败

------

## 6. TP（Tuning Protection / Secure Watchdog）

TP 在文档里也叫 **Secure Watchdog**。文档在总览里写得很直白：

它用于保护用户软件和数据，避免受到 **maltuning data** 的影响

同时在 BROM 特性里又说，TP code 用来实现 Secure Watchdog

### 你可以怎么理解

它不是普通“看门狗计时器”那么简单，而是更偏：

- 保护校准/调参相关数据
- 防止错误调参破坏系统行为
- 借助安全机制约束对某些关键参数或行为的修改

在 PFLASH 布局上，如果启用 TP：

- **PF0 S0** 必须拿来做 TP 的特定用途
- **PF0 S1~S7** 可做 TP 扩展区
- 然后再往上给 HSM PCODE 用

所以 TP 本质上是 **安全/保护机制 + 一块专用存储布局**。

------

## 7. EEPROM 仿真

这是 DFLASH 最典型的用途之一。

因为很多 MCU 没有真正独立的 EEPROM，所以会用 DFLASH 去“模拟 EEPROM 行为”。文档对 EEPROM emulation 的定义是：

- 把写访问分散到更大范围 Flash 中，提升有效寿命
- 保证各个 Flash cell 的擦写次数尽量均衡，也就是 **wear leveling**
- 管理新旧数据，让 stale data 可以被回收擦除
- 还必须能抵抗中断/掉电，启动后恢复有效数据

### 重点：它不是“固定地址反复改写”

EEPROM 仿真算法会像日志系统一样：

- 写新版本到新位置
- 用标记/序号判断哪个版本最新
- 等合适时机回收旧区域

### 为什么要这么做

因为 Flash 不能像 RAM 那样任意改一个字节：

- 写入有 page 粒度
- 擦除有 sector 粒度
- 寿命有限

所以 EEPROM 仿真的核心，其实是**软件管理策略**。

------

## 8. DFLASH

这个是总架构里的另一大主角。

文档说，DFLASH 是用来：

- 模拟 EEPROM
- 存储数据
- 分成两个 bank
- 读访问比 PFLASH 慢
- 同时还包含 UCB 和 CFS 区域

DFLASH 特性部分又进一步说：

- **DFLASH0**：通常给应用做 EEPROM 仿真，还包括 UCB 和 CFS
- **DFLASH1**：在带 HSM 的器件里，常给 HSM 做 EEPROM 仿真，并与应用隔离

它的访问特征也和 PFLASH 不一样：

- DFLASH 读是 **64-bit unique read access**
- page 编程单位是 **8 byte**
- burst 是 **32 byte**
- 支持 program/erase/suspend/resume 等命令
- ECC 用的是 **TECQED** 算法

### 你可以简单记

- **PFLASH**：放代码，追求高性能读取
- **DFLASH**：放可更新数据/配置，追求可靠写入和仿 EEPROM

------

## 9. ENDINIT

这个是 AURIX 里非常关键的“写保护门锁”。

文档说，所有会修改 PFLASH 内容的命令序列，都受 **Safety ENDINIT** 保护；如果没先移除这个保护，就去访问 PFLASH 做修改，会报 **PROER**

另外，文档也把它列成 NVM 的功能安全措施之一：
**Safety ENDINIT protection of relevant DMU configuration registers and command sequences modifying PFLASH content**

并且在 Flash 操作建议里再次强调：

**执行 PFLASH program/erase/user commands 之前，要先 disable Safety ENDINIT**

### 你可以把 ENDINIT 理解成

“**关键寄存器和关键 Flash 操作前的保险锁**”。

正常运行时这把锁是关着的。
要改关键内容，必须先按受控流程把锁打开，改完再关回去。

------

## 10. ECC

ECC 就是 **Error Correcting Code，纠错码**。

它的作用是：
**当 Flash 存储或读路径里出现 bit 错误时，能检测甚至纠正错误。**

### PFLASH ECC

PFLASH 的 ECC 很强，文档说它覆盖：

- **256 data bits**
- **地址位**

所以它不仅能发现数据出错，还能发现“地址错了但读到了别处的正确数据”这种问题

能力包括：

- 纠正 1-bit 和 2-bit 错误
- 检测 100% 的 1/2/3-bit 错误
- 检测 >99% 白噪声错误向量
- 检测全 0 / 全 1 异常
- 检测 addressing errors

但副作用是：

**擦除后的 PFLASH 不能靠直接读来判断“是不是全 0”**，因为 safe ECC 会把这种 all-0 当成异常，直接读会报 ECC 错误

### DFLASH ECC

DFLASH ECC 只覆盖 **64 data bits**，不覆盖地址，所以：

- 能保护数据
- **不能检测 addressing fault**

### 你可以简单理解

- **PFLASH ECC 更偏安全执行**
- **DFLASH ECC 更偏数据可靠性**

------

## 11. PFlash / DFlash 读等待周期

这个本质上是 **Flash 太慢，CPU 太快，所以要插等待拍数**。

文档说，PFLASH 和 DFLASH 的读路径都要根据时钟和器件延迟配置 wait cycles：

- PFLASH 用 `HF_PWAIT`
- DFLASH 用 `HF_DWAIT`

### PFLASH

PFLASH 的等待是按 **fSRI** 算的，分成两部分：

- `RFLASH`：Flash 读本体等待
- `RECC`：ECC 处理等待

例子：
`tPF = 30ns, tPFECC = 10ns, fSRI = 200MHz`
得到：

- `HF_PWAIT.RFLASH = 5`
- `HF_PWAIT.RECC = 1`

### DFLASH

DFLASH 的等待按 **fFSI** 算：

- `RFLASH`
- `ECC`

例子：
`tDF = 100ns, tDFECC = 20ns, fFSI = 100MHz`
得到：

- `HF_DWAIT.RFLASH = 9`
- `HF_DWAIT.ECC = 1`

### 一个非常重要的坑

切时钟分频时，必须先把 wait cycle 配对，不然 Flash 读时序会不对。文档明确说：

- PFlash wait-cycles 按 `fFSI2`
- DFlash wait-cycles 按 `fFSI`
- 上电后的默认值通常只够启动时 100MHz 场景，不是通用值

------

## 12. Flash 发命令序列

这是 AURIX Flash 操作最核心的思想：

**除了普通 memory-mapped read 以外，Flash 的编程、擦除、校验都不是“像内存那样直接写地址”，而是发送命令序列。**

文档原话很清楚：

- 所有 Flash 操作（除了 memory-mapped reads）都通过 **command sequences**
- 写 PFLASH 地址范围会直接 **bus error**
- 写 DFLASH0 / DFLASH1 地址范围会被解释成 **command cycles**

也就是说：

你不是在“往 Flash 地址里塞数据”，
而是在“往命令接口打一串规定顺序的控制码”。

文档还定义了命令序列的形式：

```
ST addr, data
```

参数里常见的有：

- `PA`：page 地址
- `WA`：wordline 地址
- `SA`：sector 地址
- `WD`：要写入的数据
- `xxnn`：sector 数量等

命令表里包括：

- Reset to Read
- Enter Page Mode
- Load Page
- Write Page
- Write Burst
- Verify Erased
- Erase Logical Sector Range
- Disable Protection
- Resume Protection 等

### 最容易踩坑的点

命令周期必须发到 **non-cached address range**，否则可能被 cache/store buffer 打乱顺序

------

## 13. Page Mode

**Page Mode 就是“准备写页数据”的预备状态。**

文档说，一个 Command Sequence Interpreter 必须先进入 Page Mode，才能：

- 把写数据装进 **Assembly Buffer (ASB)**
- 然后发起真正的写命令，把数据编程进 PFLASH/DFLASH

### 进入 Page Mode 后发生什么

`Enter Page Mode` 会让对应的 PFLASH 或 DFLASH assembly buffer 进入 Page Mode

同时：

- buffer 的写指针清零
- buffer 之前的内容保留
- 当后续命令完成后，Page Mode 退出
- Host 侧可通过 `PFPAGE` / `DFPAGE` 标志看是否已进入 Page Mode

### 接下来是 Load Page

`Load Page` 把数据逐步装入 assembly buffer，并推进写指针。
而且同一页的 Load Page 传输宽度必须一致，要么全 32-bit，要么全 64-bit，否则会报 SQER

### 最后才是 Write Page

`Write Page` 用前面 `Load Page` 放进 buffer 的内容，真正触发编程：

- 执行后退出 Page Mode
- 对应 bank 置 BUSY
- 如果没先在 Page Mode，就会报 SQER

### 所以 Page Mode 的本质

它相当于一个“装弹”阶段：

1. `Enter Page Mode`
2. `Load Page`
3. `Write Page`

文档给的防御式编程序列也是这么做的：
先 `Clear Status`，再 `Enter Page Mode`，等 PAGE 标志，再多次 `Load Page`，然后 `Write Page`

------

## 一句话总复盘

你这 13 个概念里，最核心的主线其实只有三条：

**第一条：结构层级**

- Bank 是大分区
- Sector 是擦除单位
- Page 是写入单位
- Wordline 是底层物理组织

**第二条：职责分工**

- PFLASH 放代码，偏高性能和安全执行
- DFLASH 放数据/配置，偏 EEPROM 仿真和安全配置

**第三条：操作方式**

- Flash 不是像 RAM 那样直接写
- 必须走命令序列
- 编程前先 Page Mode
- 改 PFLASH 前先处理 ENDINIT
- 读路径还要正确配置 wait cycle 和 ECC

如果你愿意，我下一条可以专门给你画一个“**Bank / Sector / Wordline / Page 的层级图**”，这样这几个概念会一下子通。