我先把这篇文档放到一个清晰框架里讲。

这份文档本质上不是“使用手册”，而是 **AURIX TC3xx 的 HSM（Hardware Security Module，硬件安全模块）目标规格书**。它定义了这个安全子系统的 **硬件组成、地址空间、寄存器、加密能力、启动机制、桥接通信、调试控制**，以及它和主机系统之间的边界。文档版本是 **Revision 2.0，日期 2016-03-04**；相较前一版，它明确增加/更新了 **Hash 模块、PKC 模块、看门狗、96 KB 本地 RAM、User OS Code Check 流程等**。

------

## 一、先理解这份文档在讲什么

官方在前言里说得很直接：这本书就是为了描述 HSM 的 **架构与功能**，内容按章节覆盖 Core System、TRNG、AES、Hash、PKC、Timer、Watchdog、Bridge、Memories、Firmware Architecture、System Debug。

从应用角度，HSM 主要服务这些场景：

- Secure boot
- 标定数据保护
- 传感器安全通信
- Authentication
- Secure flash load
- Immobilizer
- Secure log
- Secure debug authentication

也就是说，它不是单纯“加密外设”，而是 **整车/控制器安全根的一部分**。

------

## 二、整体架构：这不是一个单一加密核，而是一个“小型安全计算机”

第 12 页的架构图（Figure 1-1）最值得先看。图里能看到：HSM 不是一个 AES 单元，而是一个带 **32-bit CPU、Boot ROM、Local RAM、Unified Cache、TRNG、AES、Hash、PKC、Timer、Watchdog、Bridge** 的独立安全子系统，并通过桥接逻辑与主机 TriCore、Flash、RAM、传感器和外设交互。

文档对功能子模块的概括也很完整，HSM 包含：

- ARM Cortex-M3 CPU
- 4 KB 统一指令/数据 Cache
- Boot ROM
- 96 KB RAM
- AES 模块
- Hash 模块
- PKC 公钥加速模块
- TRNG
- 两个 16-bit 定时器
- 带 checkpoint 功能的 watchdog
- HSM bridge，用于和主机系统通信与隔离

**一句话概括架构定位：**
HSM = **独立 CPU + 安全存储/启动链 + 对称/哈希/公钥硬件加速 + 与主机受控互联的桥接防火墙**。

------

## 三、这份文档的主线其实有两条

### 主线 1：安全计算能力

也就是 AES、Hash、PKC、TRNG、定时器、看门狗这些模块本身能做什么。

### 主线 2：安全控制链

也就是 HSM 怎么启动、怎么加载配置、怎么和主机交换信息、怎么控制调试、怎么限制访问、怎么保证启动后进入安全状态。

真正读懂这份文档，要把这两条线结合起来看。

------

## 四、核心系统：HSM 先是一个 Cortex-M3 系统

Core System 章节说明，HSM 的核心是 **ARM Cortex-M3**，配有 **NVIC、SysTick、MPU、Unified Cache**。

文档还给出一些很“工程化”的实现细节：

- 处理器基于 ARM Cortex-M3 r2p0。
- `SEV` 指令不能用来和主机同步，和主机同步必须走 bridge communication registers。
- SysTick 以 HSM system clock 为核心时钟，给了基于 100 MHz SPB 的标定值。
- HSM 字节序是 **little endian**。

这说明 HSM 不是“只能跑固定硬件流程”的协处理器，而是可以运行安全固件逻辑的独立控制器。

------

## 五、地址空间与访问模型：这是理解 HSM 的基础

文档给了完整 memory map。几个最重要的点是：

- `0000_0000H - 0000_0FFFH` 是 **HSM Boot ROM**
- `2000_0000H - 2001_7FFFH` 是 **96 KB HSM Local RAM**
- `AF40_0E00H - AF40_0FFFH` 是 **HSM Config Area**
- `AFC0_0000H - AFC1_FFFFH` 是 **HSM Data Flash 1**

而 HSM 外设区也很清晰：

- `E800_0000H` 附近是 AES/Hash/PKC
- `EC00_0000H` 是 Timer
- `EC00_0100H` 是 Watchdog
- `EC00_0200H` 是 TRNG
- `F004_0000H - F005_FFFFH` 是 HSM bridge

更关键的是访问语义：

- 普通 memory 一般通过 **unified cache**
- 外设寄存器 **直接访问，不走 cache**
- 与主机共享 RAM 交换数据时，需要用 cache clean 机制
- 某些私有地址范围只允许 privilege mode
- SFR 访问推荐 32-bit 对齐、按寄存器宽度访问，否则可能有未定义行为

这部分透露的设计思想是：
**HSM 不只是功能模块堆砌，而是严格区分了“缓存内存访问”和“外设寄存器访问”，并通过 MPU 和地址空间保证隔离。**

------

## 六、密码学能力：HSM 的“算力核心”是什么

### 1）AES 模块

HSM 提供高性能 AES-128 单元，并带本地密钥/上下文存储：

- 本地可存 8 个 128-bit key
- 其中 2 个是不可修改、只可使用的 “use-only” key
- 可存 5 组 context
- 支持模式：ECB、CBC、CTR、OFB、GCM、XTS

所以它不是“每次把密钥明文放进寄存器算一次”的普通加速器，而是带 **本地 key storage** 的安全 AES 引擎。

### 2）Hash 模块

支持：

- MD-5
- SHA-1
- SHA-256
- SHA-224（通过 SHA-256 加一些软件支持导出）

这意味着它既能做一般完整性校验，也能为签名流程提供 hash 前处理。

### 3）PKC 模块

这是这版文档的一大重点。PKC 支持：

- 最多 256-bit operand
- 椭圆曲线 over `F(p)` 和 `F(2m)`
- 模加、模减、模乘、模约减、模除、模逆、乘法
- ECC 点加、点倍乘、标量乘
- ECDSA 生成与验证

从目录还能看到，这版还覆盖了：

- Curve25519
- Ed25519
- X coordinate recovery
- SMULT25519
- SMULTED25519
- CHECKVALID
- ECDSASIG / ECDSAVER 等命令级接口

**这说明 HSM 的定位已经不是只做 SHE 风格对称密钥安全，而是能够承担更现代的公钥认证、签名验证任务。**

------

## 七、TRNG、Timer、Watchdog：安全系统的“运行基础设施”

从功能模块列表可知，HSM 有：

- TRNG
- 两个 16-bit 定时器
- 带 checkpoint 功能的 watchdog

其中 watchdog 的 checkpoint 模式很有代表性：
它不是只防“超时死机”，还可以要求软件按正确顺序写入递增服务值，用来做 **硬件辅助的流程控制检查**。

所以这里的 watchdog 更偏安全控制，而不只是可靠性看门狗。

------

## 八、Bridge 模块：HSM 与主机系统之间真正的“边界层”

这一章是全文里最容易被低估、但工程上最关键的部分。

文档说得很明确：bridge 的主要任务，是把 HSM 子系统连接到 host system，并承担双向交互与隔离；**所有 HSM 与 host 的互动都经过 bridge**。

几个要点非常重要：

### 1）权限并不对称

- HSM 原则上对 host system 有很强的访问能力
- host 对 HSM 内部资源的访问则是受限的
- 正常情况下 host 不能随便访问 HSM 内部 memory/peripheral
- 只有部分通信寄存器可由 host 使用，或在 debug/test 特殊条件下开放

### 2）Bridge 既是互联，也是“防火墙”

文档直接把它称为 “firewall functionality”，用来保护 HSM 内部不被其他 master 随便访问。

### 3）通信不靠共享内存自旋锁，而靠通信寄存器

因为 **不支持 atomic read-modify-write**，所以不能靠普通内存做可靠 mutex/semaphore；需要使用 communication registers 来同步共享资源访问。

### 4）支持状态寄存器、标志位和中断驱动同步

桥接通信寄存器允许 host 与 HSM 交换 32-bit 状态，还能通过 flag + interrupt enable 实现中断驱动同步。

### 5）有单次访问 host memory 的窗口

文档描述了 `SAHBASE + SAHMEM` 这套单次访问窗口机制，用于不经 cache 地访问 host address range；但也警告：
**同一 host 资源如果一边走 cache、一边走 SAHMEM，会造成不一致**。并且这个地址转换机制对 Core MPU 不可见，需要额外保护。

这部分非常关键，因为很多 HSM/Host 协同问题，最后都落在这里：
**缓存一致性、同步手段、窗口地址转换、权限控制。**

第 218 页的桥接框图（Figure 9-1）从视觉上也能看出 bridge 里包含 communication unit、single access window、debug、error unit、clock divider、sensor/reset 等部件，是一个完整的安全互联系统，而不是一组零散寄存器。图旁文字还说明了颜色含义：不同颜色区分 HSM-only、Host-only、双方可见的模块。

------

## 九、启动与固件：这份文档最重要的“安全链路”

这是全书的核心之一。

### 1）BOS 是启动根

文档说明：

- Boot ROM 中存放 BOS（Boot System）
- 这是 Infineon 提供的唯一 HSM 软件组件
- HSM Config Area 中的配置数据是 **用 BOS-ROM 中的 mask-individual key 加密** 的
- 同时还带 **cryptographic checksum** 保护

这意味着系统安全链是：

**Boot ROM/BOS → 验证并解密 HSM Config Area → 配置密钥/调试/随机数/启动模式 → 进入用户代码**

### 2）BOS 的职责

BOS 负责：

- life cycle management after reset
- 锁定 debug 功能
- 更新安全设置（如 AES keys）
- 检查 BOS 配置数据一致性
- 初始化模块 SFR
- 解析 Testmode 请求
- 配置并启动 TRNG
- 清零 BOS 启动过程用过的 local RAM 区域

### 3）每次 reset 都先执行 BOS

HSM reset 后，硬件先 reset 寄存器，再从 BOS-ROM 的 vector table 初始化 SP 并跳到 reset vector，所以 **BOS 总是最先执行**。

### 4）BOS 区分 Testmode 和 Usermode

如果 `HT2HSMS` 内容有效，BOS 会读取 `HT2HSMS.REQ_MODE` 来区分 Testmode / Usermode；如果无效，则默认走 Usermode。

### 5）Boot Initialization 做了什么

Boot init 阶段最关键的动作是：

- 对 HSM Config Area 做校验
- 这个校验通过 AES 模块计算 16-byte hash 完成
- 使用的是 **CBC mode**
- key 和 IV 来自 BOS-ROM
- 若配置有效，再用 **ECB mode** 解密配置数据
- 然后加载 TRNG 配置并启动 TRNG

这里非常能体现安全设计：
**AES 不只是“业务加密模块”，还参与了平台自身的安全启动配置验证。**

### 6）AES key 装载与配置区加锁

之后，前两个 AES key 会从明文 HSM Config Area 复制到 AES 硬件中，并锁定 AES；HSM Config Area 随后被禁止继续读；debug locking 也会根据配置寄存器位更新。

这一步的意思是：
**启动后，密钥从“配置存储态”切换到“硬件受控态”，并立即收紧读取与调试权限。**

------

## 十、User Mode、Test Mode、Secure Boot 的运行方式

### 1）User-OS Code Check

BOS 会检查用户代码向量表，按 `BOOTSEL0~3` 定义的 sector 顺序寻找有效 vector table。这个流程图在第 273 页（Figure 11-6）里非常清楚。

### 2）User Mode 下 RAM 的处理

在 user mode 启动时，BOS 会占用 local RAM 最前面的 **384 bytes** 作为栈和变量；进入 User-OS 前，BOS 用过的 RAM 区会被清零；其余 local RAM 不动。

这是一种典型的“**启动后清痕**”安全处理。

### 3）Test Mode 的进入条件很苛刻

Test mode 只有在：

- HSM Config Area 未被 write protected
- host 通过 `HT2HSMS` 指示进入该模式
- host RAM 中提供正确的 mask-individual signature

时才可进入。所有 16 字节都必须匹配。执行完测试命令后，结果写回 host RAM，并更新 `HSM2HTS`，最后 HSM 进入 sleep。

这说明测试模式被严格限制，避免量产后被滥用为攻击入口。

### 4）Foreground / Background Secure Boot

用户代码阶段还支持两种 secure boot 协作策略：

- **background secure boot**：先置 `HSM2HTF.0`，主机用户代码先跑，安全检查并行进行
- **foreground secure boot**：先做安全检查，通过后才置 `HSM2HTF.0`，主机用户代码要等检查结束才启动；若检测到安全违规，则根本不启动 host user code

这个设计非常实用：
它允许系统在 **启动时延** 和 **安全强度** 之间选择策略。

------

## 十一、调试系统：可调试，但必须被 HSM 授权

第 12 章说明，HSM debug system 包含：

- FPB（硬件断点）
- DWT（观察点）
- ROM Table

提供的能力包括：

- 读 HSM boot ROM（有条件）
- 读写 local RAM
- 读写 core register
- 读写 bridge register
- 读写内部 memory-mapped 外设
- 6 个硬件 breakpoint
- 4 个 trigger/watchpoint

但它也明确说：

- **没有 ITM / ETM / TPIU**
- 也就是 **没有完整 trace capability**

更关键的是，debug access 不是默认开的：

- 只有 HSM 软件设置 `DBGCTRL.HSM` 后，主机调试模块才能访问 HSM 调试窗口
- 文档强调必须 **strong authentication** 后再开放
- `DBGCTRL.HOST` 还允许 HSM 反过来控制 host debug access
- 客户软件负责安全控制 host 与 HSM 的调试开放策略

第 279 页的 debug block diagram 还能看出，调试相关模块和总线矩阵、PPB、ROM table 的连接关系，以及 host OCDS 通过 64 KB window 访问 HSM 内部资源的方式。

------

## 十二、这份文档最重要的安全思想

如果把整篇文档提炼成几个核心思想，我认为是这几条：

### 1）HSM 是独立安全执行环境，不是普通加密外设

它有 CPU、ROM、RAM、MPU、Cache、加密模块、随机数、定时器、看门狗。

### 2）安全不是只靠算法，还靠启动链与访问控制

真正的安全链来自：

- BOS in ROM
- 配置区加密与校验
- 启动时装载并锁定密钥
- 调试与配置区锁定
- 用户代码向量表检查
- Testmode 严格门控

### 3）Bridge 是安全边界核心

Host/HSM 协同的成败，不在“有没有共享内存”，而在：

- 通信寄存器如何同步
- cache/window 是否一致
- host 对 HSM 是否被正确限制
- debug 是否被 HSM 显式授权

### 4）它没有物理攻击防护，软件要补

文档明确说：
HSM **没有硬件级的物理攻击防护**；侧信道与故障攻击的对策需要软件实现。

这一点很关键，避免把 HSM 神化成“天然抗攻击黑盒”。

------

## 十三、如果你是工程师，应该怎么读这篇文档

### 1）如果你做启动/安全架构

重点看：

- 第 11 章 Firmware Architecture
- 第 9 章 Bridge
- 第 12 章 Debug
- 第 2 章 Core System

因为这些决定了 secure boot、权限、host/HSM 边界、debug 开闭。

### 2）如果你做密码库/驱动

重点看：

- 第 4 章 AES
- 第 5 章 Hash
- 第 6 章 PKC

尤其第 6 章既有算法级定义，也有命令级接口。

### 3）如果你做底层通信/集成

重点看：

- 第 9 章 Bridge
- 第 2 章 Address Map / Cache / MPU
- 第 11.3 / 11.4 Host-HSM 信息交换

因为实际集成常见问题都在这几章。

------

## 十四、最后给你一个“全篇摘要版”

这篇文档描述的是：
**AURIX TC3xx 内部的 HSM 是一个独立的安全子系统，它以 ARM Cortex-M3 为控制核心，配套 Boot ROM、Local RAM、Cache、TRNG、AES、Hash、PKC、Timer、Watchdog 和 Bridge。它通过 BOS 完成安全启动、配置校验、密钥装载、调试锁定和模式选择；通过 Bridge 与主机系统受控交互；通过 AES/Hash/PKC 提供对称、散列和公钥能力；通过 MPU、配置区锁、调试控制和受限测试模式来建立安全边界。**

如果你愿意，我下一步可以继续按下面两种方式之一展开：
**一种是“按章节逐章精讲”**，另一种是 **“从 secure boot / bridge / PKC 三条主线深挖”**。





好，我们先把这篇文档里的 **secure boot** 单独拎出来讲清楚。

先说结论：
这份规格里的 secure boot 不是“上电后直接验一遍主程序签名”这么简单，而是分成 **两层启动信任链**：

1. **HSM 自己先安全启动**：由 Boot ROM 里的 **BOS（Boot System）** 先运行，检查和解密 HSM 配置区，装载关键安全设置，锁定调试/配置访问，然后决定进入 Usermode 还是 Testmode。
2. **再由 HSM 用户代码参与主机侧 secure boot**：这一步可以做成 **background secure boot** 或 **foreground secure boot**，也就是“边放行主机边校验”或“校验通过后才放行主机”。

------

## 1. Secure boot 的“根”是谁？

在这份文档里，安全启动的根不是用户代码，而是 **BOS in Boot ROM**。

文档明确说：

- 所有启动必须用到的 BOS 功能都存放在 **HSM Boot ROM** 中；
- 这是 Infineon 提供的唯一 HSM 软件组件；
- HSM Config Area 中的配置数据是 **用 BOS-ROM 中的 mask-individual key 加密** 的；
- 同时还带 **cryptographic checksum** 保护。

这意味着它的信任根链条是：

**Boot ROM / BOS → 验证 HSM Config Area → 解密配置 → 装载安全设置/密钥 → 决定后续启动路径**

这就是为什么文档把 BOS 直接叫作 **secure boot system**。

------

## 2. BOS 到底负责什么？

BOS 的职责很关键，文档列得很清楚，包括：

- reset 后的生命周期管理
- 锁定 debug 功能
- 更新安全设置（例如 AES keys）
- 检查 BOS 配置数据一致性
- 初始化部分模块 SFR
- 判断是否进入 Testmode
- 配置并启动 TRNG
- 在进入 Usermode 前，把启动过程中使用过的 local RAM 区域清零
- 失效启动过程中用过的 cache / register file 等安全敏感信息
- 重新检查关键安全位
- 用 MPU 做栈溢出保护
- 对 HSM Config Area 提供 read lock / write lock 机制

这里最重要的一点是：
**secure boot 在这份文档里不只是“验签”，而是“启动期间建立安全状态”**。

------

## 3. HSM secure boot 的启动流程

### 第一步：上电/复位后，先执行 BOS

文档说得非常明确：每次 HSM reset 后，硬件会先 reset 寄存器，再读取 BOS-ROM 里的 vector table，初始化 SP，然后跳到 reset vector，所以 **BOS 总是第一个被执行**。

### 第二步：BOS 做 Boot Initialization

这个阶段是 secure boot 的核心检查点。文档说明：

- 先配置中断和异常处理；
- 然后检查 **HSM Config Area 的 cryptographic checksum**；
- 具体做法是：对加密的 HSM 配置数据，从 offset `0x10` 开始，用 **AES 模块计算 16-byte hash**；
- 这里 AES 运行在 **CBC mode**，其 key 和 IV 来自 BOS-ROM；
- 如果 checksum 无效，就只用硬件复位默认值启动 TRNG，并设置状态码；
- 如果 checksum 有效，则再用 **ECB mode** 用 BOS-ROM 里的 mask-individual key 解密配置数据；
- 然后加载 TRNG 配置并启动 TRNG。

这一步很值得注意：
**AES 模块在 secure boot 里既用于“完整性检查”，也用于“配置解密”**。
也就是说，AES 不是业务功能，而是平台自身启动信任链的一部分。

------

## 4. 配置区为什么这么重要？

因为 HSM 的很多关键安全参数都在这里，比如模块配置、AES 安全密钥等。文档写明这些信息保存在 Flash 中 HSM 专属的 Config Area，而且它是加密并带校验的。

而且 BOS 对这个区域有两层锁：

- **read lock**：在 Usermode 启动早期就会开启
- **write lock**：在最终完成配置编程后开启，并且会 **阻断 Testmode**

这背后的安全思想是：

- 启动前，它是“受保护配置源”
- 启动后，它不应该再被当成“可随意读取的明文配置存储”

------

## 5. 启动成功后，BOS 还会做什么“收口动作”？

文档说明，在 boot initialization 后，AES key 不需要暂存到 local RAM；然后 **HSM Config Area 会被锁定禁止继续读取**；接着 BOS 会读取 `DMU_SP_PROCONHSM` 里的调试相关位，更新 `DBGCTRL`，完成 debug locking。

另外，在进入用户代码前，BOS 还会：

- 清除 BOS bits
- 销毁 confidential data
- 把启动时用过的 RAM 区域清零
- 必要时关闭 ROM 可见性
- 清空错误状态后再进入用户代码。

所以 secure boot 的“最后一步”不是“跳转”，而是 **清痕 + 锁定 + 缩小攻击面**。

------

## 6. Usermode 怎么判定“可不可以启动用户代码”？

这一步相当于 **第二道门**。

BOS 在进入 Usermode 后，会执行 **User-OS Code Check**。文档的 Figure 11-6 表明，它会按照 `BOOTSEL0~3` 指定的扇区顺序，依次检查 vector table 是否有效；只要找到有效的用户代码向量表就 PASS，否则 FAIL。

这意味着：

- HSM 用户代码不是“固定一个地址死跳”
- 而是按配置定义的 boot sector 顺序去找有效启动映像

这本质上也是 secure boot 的一部分，只不过它这里校验的是 **“用户启动入口是否合法/有效”**，而不是配置区校验。

------

## 7. Host 和 HSM 在 secure boot 里怎么配合？

文档专门定义了两个通信寄存器：

- `HT2HSMS`：Host → BOS
- `HSM2HTS`：BOS → Host

其中 `HT2HSMS` 很关键，里面包括：

- `REQ_MODE`：请求 Usermode 还是 Testmode
- `OS_ENTRY_ALLOWED`：Host 是否允许 User-OS entry
- `HAR_ENABLE`：是否请求 Halt After Reset
- `CONT_VALID`：这些内容是否有效

这表示 secure boot 并不是 HSM 单方面自顾自启动，它和主机启动序列之间有同步机制。

尤其 `OS_ENTRY_ALLOWED` 很重要。文档说，BOS 会等待这个位被置位，然后才允许进入 User-OS；随后如果请求了 HAR 且 debug mode 已启用，还会在第一条 User-OS 指令处触发 break。

也就是说：

**Host 负责“是否允许 HSM 用户代码开始跑”的节拍控制；HSM 负责“在进入用户代码前已经处于安全状态”。**

------

## 8. 启动失败时会怎样？

文档给出的行为非常“安全优先”：

- 如果 HSM Config hash 无效，或者
- AES key 没正确锁定，或者
- User code 不合法，

BOS 都会更新 `HSM2HTS` 的 `STATUS_CODE` 和 `ACT_MODE`，然后让 HSM 进入 **sleep mode**。

另外文档还说明：当启动过程中的 sanity check 失败时，BOS 会把详细错误信息写到 `HSM2HTS[15:0]`，并把 `ACT_MODE` 设为 `11B`，随后进入 sleep，让 host 去分析问题并决定恢复手段。

这说明 secure boot 的失败处理不是“继续凑合跑”，而是 **fail closed**。

------

## 9. 真正和“主机应用启动”相关的 secure boot：foreground vs background

这里是很多人最容易混淆的点。

文档在 11.4 里说，**foreground/background secure boot** 是由 **HSM user code** 决定的主机启动策略，而不是 BOS 本身那一层。具体有两种方式：

### background secure boot

HSM 用户代码会 **立即设置 `HSM2HTF.0`**，于是 host 的 SSW 会先跳到 host user code；与此同时，安全检查在后台并行进行。

### foreground secure boot

HSM 用户代码会 **先执行 secure boot checks**，只有检查通过、没有发现 security violation 时，才设置 `HSM2HTF.0`。因此 host user code 会被延后启动；如果发现安全违规，host user code **根本不会启动**。

另外，foreground secure boot 还能让 HSM user code 向 `HSM2HTS` 写一个 seed，供 SSW 生成噪声来扰乱功耗轮廓；噪声在 `HSM2HTF.0` 置位后结束。

------

## 10. 你可以把整个 secure boot 理解成两段

### 第一段：HSM 自举安全

由 BOS 完成，目标是：

- 验证配置区
- 解密配置
- 启动 TRNG
- 锁定配置区
- 锁定 debug
- 清除敏感启动痕迹
- 验证 HSM 用户代码入口是否合法

### 第二段：HSM 参与主机安全启动

由 HSM user code 完成，目标是：

- 决定主机代码是否立刻启动
- 或等待 secure boot checks 结束再放行
- 发现违规时直接阻止 host user code 启动

所以这份文档里的 secure boot 是：

**“先保证 HSM 自己可信，再由 HSM 去决定主机系统是否可信地继续启动。”**

------

## 11. 从工程角度，最该抓住的几个点

### 第一，BOS 是 Root of Trust

因为它在 Boot ROM 中，reset 后最先执行，而且持有配置校验/解密所需的 key/IV。

### 第二，Config Area 是 secure boot 的核心对象

它不是普通参数区，而是“受加密和校验保护的启动配置源”。

### 第三，secure boot 不只看“验不验”，还看“验完后是否锁住”

文档反复强调：

- read lock
- write lock
- debug lock
- 清理敏感数据
- 失败则 sleep。

### 第四，foreground / background 是系统策略选择

- background：启动快，但 host 代码和安全检查并行
- foreground：安全更强，未通过不放行 host user code

------

## 12. 一句话总结

这篇文档里的 secure boot 可以概括成：

**BOS 先基于 Boot ROM 建立 HSM 自身的可信启动环境，对配置区做加密完整性校验和解密、装载安全设置并锁定关键资源；随后 HSM 用户代码再以 foreground 或 background 的方式决定 host user code 是否被允许启动。**

下一步我建议直接继续讲 **foreground secure boot 和 background secure boot 的区别**，因为这部分最容易和 BOS 启动混在一起。





