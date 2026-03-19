下面我按“**这篇文档到底在讲什么 → 启动链路怎么走 → 各模块分别干什么 → 工程上怎么用它**”的顺序，给你做一个系统、详细、偏工程视角的讲解。

这份文档是 **AURIX™ TC3xx Platform Firmware** 的用户视角说明，版本是 **User’s Manual V2.0.0 / Firmware V1.1.0.1.18 / 2021-02**。它讲的不是你的应用程序，而是**芯片上电或复位后，BootROM 里的平台固件如何把芯片带起来、选启动模式、做自检、必要时进入 Bootloader，然后再跳到用户代码**。

------

## 一、先用一句话概括这篇文档

这篇文档描述的是：**TC3xx 上电/复位后，CPU0 先执行 Startup Software（SSW）；SSW 决定从哪里启动、是否跑 ABM/BSL/LBIST、是否启用 lockstep、是否做 SOTA/SWAP；然后可选进入 Checker Software（CHSW）做安全相关检查；最后才跳到用户代码。** 固件还内建了 Bootloader、关断处理器、以及调试用的 PSFDM。

------

## 二、文档整体结构与模块关系

从用户视角，这个平台固件由 5 类东西组成：

1. **Startup Software（SSW）**，包含 **SOTA** 支持
2. **Checker Software（CHSW）**
3. **Bootstrap Loaders（BSL）**
4. **Shutdown request handler**
5. **Power Supply Friendly Debug Monitor（PSFDM）**

其中最核心的是前两个：

- **SSW**：负责“把系统带起来，并决定怎么启动”
- **CHSW**：负责“检查 SSW 做完后，系统是否处于正确且安全的状态”

可以把它理解成一条启动流水线：

**复位 → SSW → 选启动模式（Flash / ABM / BSL）→ 可选 CHSW → 跳转用户代码**

文档在第 14 页的 **Figure 21** 也画了这个总流程：
Firmware START → Flash Rampup → Chip configuration → RAM initialization → SOTA configuration → BMHD 评估 → 可选 LBIST → Lockstep configuration → 执行启动模式 → CHSW → Debug/HARR → ESR0 handling → Jump to User Code。

------

## 三、SSW 是什么：它是整个启动链的“总调度器”

SSW 是**复位后第一个执行的软件**，而且**只在 CPU0 上执行**；其他 CPU 启动时都先被保持在 Halt 状态，后面再由用户软件决定是否启动。SSW 的入口在 **BootROM**，CPU0 的复位 PC 就指向这里；SSW 的最后一条关键动作是**跳转到第一条用户指令**。

SSW 决策时依赖的信息来源有四类：

- Flash 中预先写好的配置
- 某些专用寄存器或存储位
- 当前复位类型
- 外部配置引脚状态（可选）

这四类信息，就是后面 BMHD、ABM、HWCFG 引脚、HARR、调试口密码、SOTA 等所有机制的基础。

------

## 四、复位类型不同，SSW 的行为也不同

文档先把不同复位场景下，芯片初始状态讲清楚了，因为这直接影响启动路径。

### 1）Cold power-on reset

这是“真正冷启动”，也就是从没上电到上电。此时寄存器处于初始态、Flash 仍在 reset 状态、RAM 内容不确定、时钟系统初始。因为系统是“最原始状态”，所以这类启动流程最长、做的事情最多。

### 2）System reset / Warm power-on

对 SSW 来说，**system reset 基本按 warm power-on 处理**。这类复位时，受影响寄存器回初始态，Flash 也要重新带起来，但 RAM 通常保留复位前内容。

### 3）Application reset

这是最快的复位。Flash 仍可读，很多外围和 RAM 状态会保留，所以它不是“从零启动”，而是“轻量重进 SSW”。

### 4）时钟差异

- Power-on / system reset 下：
  fSRI=fCPU0=fFSI=fBACK≈100MHz，
  fSPB=fSTM≈50MHz，
  PLL/VCO 处于 power-down。
- Application reset 下：**时钟保持复位前状态**，不会自动回默认。

这点非常重要，因为后面像 **ESR0 延时、CAN BSL、某些检查项** 都依赖“当前时钟是不是默认状态”。

------

## 五、先看启动模式：这份文档真正的核心，是“怎么选启动路径”

文档把平台启动模式归纳成三大类：

### 1）Internal Start

最普通：从内部 PFlash 启动，入口地址由 **BMHDx.STAD** 指定。

### 2）Bootloader Modes

进入 Bootloader，把外部主机下载的代码/数据放入 **CPU0_PSPR**，随后从 PSPR 首地址执行。支持：

- **ASC Bootloader**
- **Generic Bootloader**（可自动在 ASC/CAN 间选择）

### 3）ABM（Alternate Boot Mode）

它也是“从用户指定地址启动”，但条件更严格：不仅要有启动地址，还要对 **ABM Header 和代码区域做完整性校验**。如果校验失败，某些情况下可以自动回退到 BSL。

------

## 六、BMHD：启动决策的根配置

### 1）BMHD 在哪里

TC3xx 里定义了 **4 组 Boot Mode Header（BMHD0~BMHD3）**，都在 **DFLASH 的 UCB 区**。每组都有一份 **Original** 和一份 **Copy**。

### 2）BMHD 里有什么

BMHD 里最关键的是这些字段：

- **BMI**：Boot Mode Index，里面定义启动方式
  - `PINDIS`：是否允许引脚参与模式选择
  - `HWCFG[3:1]`：模式编码
    - `111` = Internal Flash
    - `110` = ABM
    - `100` = Generic BSL
    - `011` = ASC BSL
  - `LSENA0..3`：CPU0~CPU3 lockstep 使能
  - `LBISTENA`：是否让 SSW 触发 LBIST
  - `CHSWENA`：是否允许 CHSW 执行，**101 表示禁用 CHSW**
- **BMHDID**：Header 标识，必须是 `B359H`
- **STAD**：启动地址，必须在 PFLASH 内、且 word aligned
- **CRCBMHD / CRCBMHD_N**：Header CRC 与反码

### 3）一个很容易忽略的关键点

文档特别强调：**STAD 必须始终是“合法的 PFLASH 字对齐地址”**，哪怕当前模式是 BSL、哪怕是通过引脚选出的模式，这个要求依然成立。
也就是说，**“反正我走 BSL，STAD 随便填”是错误的**。

这在板级 bring-up 中是非常常见的坑。

------

## 七、SSW 如何评估 BMHD：真正的启动判定流程

BMHD 评估是整篇文档的主干。可以概括成 6 个阶段：

### 阶段 1：初始化内部状态

SSW 先把这些标志清空：

- 还没找到可用 BMI
- 还没找到可用 BOOT_CFG
- 还没启用引脚选模

### 阶段 2：检查当前 UCB_BMHDx 状态

只有当前 BMHD 的状态是 **CONFIRMED** 或 **UNLOCKED**，才继续。否则这个 BMHD 直接判无效。

### 阶段 3：检查 Header 合法性

依次检查：

- BMHDID 是否正确
- BMI 编码是否合法
- STAD 是否是合法 PFLASH 地址
- CRCBMHD / CRCBMHD_N 是否匹配

### 阶段 4：决定“按引脚”还是“按 BMI”

只有同时满足这些条件，才允许“引脚决定模式”：

- `BMI.PINDIS=0`
- 全局 Boot Mode Lock 未激活（`BML=00B`）
- `HWCFG3` 采样为低，允许配置引脚生效

否则只能按 BMHD 里的 BMI 来选模式。

### 阶段 5：如果选中的是 ABM，就进入 ABM 评估

如果不是 ABM：

- 如果是 BSL，则 `BOOT_ADDR = CPU0_PSPR_begin`
- 如果是 Internal Start，则 `BOOT_ADDR = BMHDx.STAD`

### 阶段 6：保存结果到 STMEM1 / STMEM2，并配置 lockstep

SSW 会把：

- `BOOT_CFG`
- `BMHD_INDEX`
- `BMHD_COPY`
- `BOOT_PIN`

写到 **SCU_STMEM1**；把最终 `BOOT_ADDR` 写到 **SCU_STMEM2**；然后按 BMI 配置 `SCU_LCLCON0/1` 的 lockstep 控制位，最后置位 `BMI_VALID` 和 `BOOTMODE_CONFIGURED`。

### Original / Copy 的真实含义

流程图看起来像是“Original 和 Copy 都会走一遍”，但文档说明：**现实里同一时刻只能有一份（Original 或 Copy）处于 CONFIRMED/UNLOCKED 状态**。所以很多时候 Copy 并不会真的参与评估，只是逻辑上有这条后备路径。

> 第 8 页的 **Figure 17** 把这条 BMHD 评估链画得很直观，适合你对照理解。

------

## 八、ABM 到底是什么：比普通 Flash 启动更“讲证据”的启动方式

ABM 不是简单“跳转到另一个地址”，而是“**带头部和代码完整性验证的替代启动**”。

### 1）ABM Header 位置

ABM Header 可以放在 **PFLASH 内任意 word-aligned 地址**，它的起始地址来自“选中了 ABM 的那个 BMHD 的 STAD”。

### 2）ABM Header 内容

ABMHD 包含：

- `STADABM`：ABM 模式下的代码入口
- `ABMHDID`：必须是 `FA7C B359H`
- `CHKSTART / CHKEND`：待校验代码区范围
- `CRCRANGE / CRCRANGE_N`
- `CRCABMHD / CRCABMHD_N`

### 3）ABM 评估步骤

SSW 会依次检查：

1. `ABMHDID`
2. `STADABM` 是否合法
3. ABM Header 的 CRC
4. `CHKSTART..CHKEND` 代码区的 CRC
   全部通过后，才把 `BOOT_ADDR = STADABM`，ABM 才真正生效。

### 4）ABM 失败时怎么办

如果 ABM 失败，而且它是**由引脚选出来的 ABM**，并且发生在**最后一个 BMHD（BMHD3）**评估场景下，那么固件会根据引脚配置回退到：

- ASC BSL，或
- Generic BSL
  并把 `BOOT_ADDR` 改成 `CPU0_PSPR_begin`。

### 5）引脚和启动模式映射

文档给出 HWCFG[5:4] 的引脚模式表：

- `11` → Internal start from Flash
- `10` → ABM，失败回退 Generic BSL
- `01` → ABM，失败回退 ASC BSL
- `00` → Generic BSL

这说明：**引脚不仅能决定“是不是 ABM”，还能决定“ABM 失败后怎么兜底”**。

------

## 九、如果没有有效 BMHD，会发生什么

这是文档里非常实用的一部分，因为很多板子第一次 bring-up 都会卡在这里。

### 情况 1：没有有效 BMHD，但允许 default configuration

如果：

- HSM boot 被禁用
- Boot Mode Lock 未激活

那么 SSW 仍然可以给你一条生路：

- 初始化 security-sensitive RAM
- 若没请求 Halt After Reset，则进入 **Generic BSL**
- 若请求了 HARR，则准备 Internal Start 但实际会优先停在 HARR
- 把启动结果保存到 STMEM1 / STMEM2
- 禁用 CPU0 lockstep（因为没找到有效 BMI）

这本质上是：**即使 BMHD 全坏，只要默认配置允许，芯片仍能进入一种可恢复、可下载程序的状态。**

### 情况 2：默认配置也不允许

如果 default configuration 被禁用，那么 SSW **连启动模式都选不出来**。此时它会：

- 检查是否允许 debug access
- 若允许则打开调试接口
- 然后进入 **endless NOP loop**

这就意味着：**芯片不会进入用户代码，也不会进入可用的 BSL，而是卡死在 BootROM 里等你调试。**

------

## 十、SSW 主流程里还有哪些关键动作

除了“选启动模式”，SSW 还负责很多系统级动作。

### 1）RAM 初始化

RAM 初始化配置来自 UCB_DFLASH，最终装载到 `DMU_HF_PROCONRAM`。
它会把选中的 RAM 清零并生成正确 ECC，以避免访问时出现不可纠正 ECC 错误。支持：

- 冷/暖上电可选
- 按 CPU 单独配置
- 按 LMU 单独配置
  但处于 standby 供电的 `CPU0_DLMU` / `CPU1_DLMU` 在唤醒后不会被初始化。

### 2）Secure Boot 信息供给

如果芯片带 HSM，则 SSW 会把**当前模式、复位类型、主选启动模式、实际采取的启动模式**提供给 HSM 参与后续安全启动流程。

### 3）LBIST

若有效 BMI 里 `LBISTENA=1`，SSW 会触发 LBIST。
LBIST 完成后会导致一次内部 reset；下一次 SSW 重进时，如果 `LBISTDONE=1`，就不会重复触发。
注意：**SSW 不判断 LBIST 成功失败，结果要由应用软件自己读取寄存器判断。**

### 4）Lockstep 配置

只在 **cold power-on** 下做：

- 如果找到有效 BMI，就按 `BMI.LSENAn` 去配置 CPU0~CPU3 lockstep
- 否则至少会把 CPU0 lockstep 禁掉

### 5）Debug 系统处理

SSW 会检查外部调试器是否通过 `COMDATA` 写入了固定魔数 `0x76D6E24A` 发起调试解锁请求；若是，则再接收后续 8 个 word，组成 256-bit 调试密码交给 DMU 校验。
如果密码正确，就解锁调试；否则继续走普通路径。

若没有调试密码请求，SSW 会把 `UNIQUE_CHIP_ID_32BIT` 写入 COMDATA 供工具识别产品型号；它**不是单片唯一 ID**。如果需要真正唯一 ID，要去读 UCB_USER。

然后它还会看 Flash read protection 是否打开：

- 打开 → 调试接口维持锁定
- 没打开 → 可解锁调试
  并且当 `DBGIFLCK=1` 且开启 Flash 读保护时，必须密码正确才给调试。

### 6）Halt After Reset（HARR）

如果**调试访问被允许**，且**请求了 HARR**，SSW 会在第一条用户指令处布一个断点，并启用 OCDS。

### 7）ESR0 处理

如果 `ESR0CNT != FFFH` 且 ESR0 引脚当前是 open-drain reset output，SSW 会：

- 延时释放 ESR0
- 延时值由 `ESR0CNT` 决定
- 最后等待 ESR0 真正拉高，再跳用户代码
  这个延时是用 `STM0` 做的，所以如果 application reset 后用户代码已经改过默认时钟/定时器配置，这个延时可能就不准了。

### 8）结束 SSW

SSW 最后会：

- 解锁调试（如果适用）
- 跳到第一条用户指令 `STADD`
  另外在某些 ED 条件下，不是跳普通 STADD，而是跳到 EMEM 里的 prolog code。

------

## 十一、CHSW 是什么：它不是“启动代码”，而是“启动后的检查员”

CHSW 的目的不是启动，而是**验证 SSW 结束后，设备配置和安全相关准备是否正确**。

### CHSW 何时才会执行

不是每次启动都跑。必须同时满足：

1. **SSW 的 bootmode evaluation 被认为是正确的**
   CHSW 会把 SSW 写到 STMEM 里的启动信息，与 DFLASH/UCB 中的 BMHD 信息、HWCFG 引脚状态、相关启动因素重新比对。
2. **确实找到了有效 BMI 并用于启动选择**
   如果根本没有有效 BMHD/BMI，但系统通过兜底路径继续运行，那么 CHSW 后半段不会执行。
3. **BMI 没有禁用 CHSW**
   即 `CHSWENA != 101B`。

你可以把它理解成：**CHSW 的前提是“启动路径本身必须先有资格被检查”。**

------

## 十二、CHSW 检查什么

文档把 CHSW 检查分成几类，而且不同复位类型会触发不同子集。

### 1）任何复位都可能检查

- CHSW 总体状态
- Bootmode 选择
- SWAP 配置
- Shutdown request handler 入口
- GETH / GETH1
- RIF0 / RIF1
- CPUx_FLASHCON1

### 2）仅 cold power-on

- EVR trimming #1 / #2
- Converter control block trimming

### 3）cold + warm power-on

- PLL trimming
- FM trimming
- EMEM ECC 相关两类检查

### 4）system reset + cold/warm power-on

- CHIPID
- CCUCON0
- PFLASH wait states

### 5）复位类型识别检查

- CPOR
- WPOR
- SYSR
- APPR

另外文档提醒：很多检查只在芯片真正实现了对应模块时才有意义，比如 EMEM、RIF、GETH。

------

## 十三、CHSW 结果怎么看：看 STMEM3 ~ STMEM6

这是实际调试中最关键的用法。

CHSW 不会给你一个“总分”，而是把每个检查项的状态拆散写进四个寄存器：

- **SCU_STMEM3**：这个检查“开始了”
- **SCU_STMEM4**：这个检查“失败了”
- **SCU_STMEM5**：这个检查“通过了”
- **SCU_STMEM6**：这个检查“结束了”

文档还给了一个状态判定表：

- `STMEM3.*_CS`：未执行=0，PASS=1，FAIL=1
- `STMEM4.*_CF`：未执行=0，PASS=0，FAIL=1
- `STMEM5.*_CP`：未执行=0，PASS=1，FAIL=0
- `STMEM6.*_CE`：未执行=0，PASS=1，FAIL=1

所以工程上最常用的判断方式是：

**某项检查是否执行？** 看 `*_CS` / `*_CE`
**是否失败？** 看 `*_CF`
**是否通过？** 看 `*_CP`

举例：

- `BMSEL_CF=1` → 启动模式选择检查失败
- `PWAIT_CP=1` → PFLASH wait states 检查通过
- `CHSWGEN_CE=1` → CHSW 已经执行结束

这些位的详细定义从 STMEM3 一直到 STMEM6 都展开列出来了。

------

## 十四、Bootstrap Loader：平台内置的“救机/下载程序”通道

### 总体概念

BSL 的作用是：**通过外部接口把代码下载到 CPU0_PSPR，然后从那里运行。**
文档明确说：进入 BSL 后，协议会一直等到完整下载结束，**没有 timeout 自动退出**；想重新开始只能 reset。

------

## 十五、ASC BSL：最简单的串口下载方式

ASC Bootloader 的流程很简单：

1. 配置 RxD/TxD 引脚
2. 根据主机发来的 **zero byte** 计算波特率
3. 初始化 ASCLIN（8 data + 1 stop）
4. 发一个 `D5H` 应答字节
5. 开接收器
6. 等待接收 **128 字节**
7. 把这 128 字节作为 32 个 word 写到 `CPU0_PSPR @ C0000000H`
8. SSW 退出后，从 `C0000000H` 执行用户代码

引脚上：

- ASC Bootloader 模式：`ASCLIN0`, Rx=P15.3, Tx=P15.2
- Generic BSL 里的 ASC 协议：Rx=P14.1, Tx=P14.0

文档还特别提醒：支持 half-duplex 不代表把 Rx/Tx 直接短接，通常还是要借助物理层电路。

------

## 十六、CAN BSL：更复杂，但工程上更常用

### 1）基本特性

CAN BSL 通过 CAN 把代码下载到 `CPU0_PSPR`。它是**主引导器**，不要和第三方工具厂商卖的 secondary bootloader 混淆。
特性包括：

- 支持 Classical CAN
- 在 **Configured OSC** 场景下支持 CAN FD
- 初始 ID 固定为 `0x555`
- 要接收的数据帧数可编程

### 2）最大关键点：CAN BSL 能力取决于 OSC 是否配置

#### OSC 未配置

如果 `OSCCFG=0`，CAN BSL 使用内部 back-up clock：

- `MCANDIV=5`
- `CLKSELMCAN=01B`
- MCAN 异步部分时钟是 20MHz
  这时**只支持较少波特率**。

#### OSC 已配置

如果 `OSCCFG=1`，用户必须预先把 UCB_DFLASH / PROCONDF 配成匹配板子晶振的参数。此时：

- 使用外部 XTAL
- `CLKSELMCAN=10B`
- 支持更完整的波特率范围和 CAN FD

#### 一个很重要的限制

如果设备在 **OSC 未配置** 的场景下通过 CAN BSL 启动过，那么之后再靠 **application reset** 继续重进 CAN BSL 是**不可行**的。
要支持这种用法，必须先把 OSC 配好。

### 3）支持能力总结

- **未配置 OSC**：
  只支持 Classical CAN，采样点 60%，支持 100/125/250/500 Kbit/s。
- **已配置 OSC**：
  支持 Classical CAN + CAN FD，采样点 80%，波特率/FOSC 组合见 Table 55。

### 4）CAN BSL 的三阶段流程

文档第 26 页 Figure 23 画得很清楚，分成 3 阶段：

#### Phase 1：Initialization & Ack #1

主机反复发送 **Init Message 1**。
芯片尝试理解报文，自动识别波特率；如果识别成功：

- 配置 MCAN 到该波特率
- 应答 Acknowledgement Message 1
  否则会发错误帧 `0xAAA`，然后终止，等待 reset。

#### Phase 2：Initialization & Ack #2

收到 Ack1 后，主机可发 **Init Message 2**：

- Classical CAN：可选，用来给更精确的波特率
- CAN FD：必需，用来下发 `NBTP / DBTP / DBPM`
  若参数正确，芯片返回 Ack2，表示已准备好收数据。

#### Phase 3：Data Loading

主机开始发送 Programming messages，芯片把数据存到 `CPU0_PSPR @ C0000000H`。数据收满后，CAN BSL 返回主 SSW 流程，最终从 `C0000000H` 运行。

------

## 十七、SOTA / SWAP：双 Bank 启动切换能力

SSW 在 power-on 和 system reset 后，还会去看 `UCB_SWAP_ORIG/COPY`，决定是否启用 **SWAP** 功能，也就是切换 PFLASH Bank。
如果找到有效 SOTA 配置，它会：

- 禁止对 PFLASH 的 direct access（设 `CPUx_FLASHCON4.DDIS=1`）
- 在 `SRU_SWAPCTRL.ADDRCFG` 里激活 Region A 或 Region B

这就是 OTA 场景下常见的 **A/B Bank 切换**基础。

------

## 十八、Shutdown request handler：温和下电，而不是猛地停机

文档说，任何 **warm reset request** 到来时，所有 active CPU 都会无条件跳到 shutdown handler。这个 handler 不能被中断或 trap 打断，它会通过受控的功耗下降过程，避免电流突变。

处理逻辑是：

- 先根据 `CORE_ID` 跳到每个 CPU 自己的处理分支
- 执行 average-power loop
- 等待 `SCU_RSTCON2.TOUTyy=1`
- 最后执行 `WAIT`

不同 CPU 到达被动状态的时间不同：

- CPU2 / CPU5：20µs
- CPU1 / CPU4：40µs
- CPU0 / CPU3：60µs

------

## 十九、PSFDM：为了调试时不把供电冲垮

PSFDM 是 BootROM 中的一个独立例程，**不在正常启动流程里执行**。它是给调试器用的“友好型 debug trap handler”，目的有两个：

- 避免多个 CPU 同时 halt 导致电流突降，从而引起 EVR 过冲/欠冲
- 避免多个 CPU 同时放开 halt 时电流猛增

调试器应把 debug trap vector 指到 `AFFF FC80H`。进入后，PSFDM 在一个循环里混合执行：

- `MADD.Q`（高功耗）
- `NOP`（低功耗）
  直到 `CBS_TLS.TL2=0` 才退出。然后调试器再通过释放 halt + 拉 OTGS Trigger Line 2，让 CPU 们同步返回用户代码。

这个模块体现了 AURIX 在多核调试和电源完整性上的设计思路。

------

## 二十、STMEM1 / STMEM2：启动诊断时最先看的寄存器

### STMEM1：告诉你“SSW 最终选了什么”

`SCU_STMEM1` 里有这些关键字段：

- `BMI_VALID`
- `BOOT_PIN`
- `BMHD_COPY`
- `BMHD_INDEX`
- `BOOT_CFG`
- `SCRDIS`
- `BOOTMODE_CONFIGURED`
- `HARREQ`
- `SWAP_CFG`
- `SWAP_TARGET`
- `SWAP_DW_INDEX`

`BOOT_CFG` 编码尤其重要：

- `111` = Internal Start from Flash
- `110` = ABM, Generic BSL on fail
- `101` = ABM, ASC BSL on fail
- `100` = Generic BSL
- `011` = ASC BSL

### STMEM2：告诉你“最终要跳哪里”

`SCU_STMEM2.BOOT_ADDR[31:2]` 表示**SSW 最终确定的第一条用户指令地址**，而且天然是 word aligned。

------

## 二十一、从工程角度看，这篇文档最重要的 10 个结论

### 1）TC3xx 的启动不是“固定从 Flash 跳转”

它是一个多阶段决策系统，BMHD、引脚、ABM、BSL、SOTA、HARR 都会影响最后路径。

### 2）BMHD 是第一优先级配置中心

你不理解 BMHD，就基本不可能真正理解 TC3xx 的启动。

### 3）STAD 不能乱填

哪怕你只想用 BSL，`STAD` 也必须是合法 PFLASH 地址。

### 4）ABM 不是简单跳转，而是“带 CRC 验证的启动”

适合安全升级、冗余镜像、受控替代入口。

### 5）没有有效 BMHD 不一定死机

在 default configuration 允许时，SSW 仍可能把设备拉进 Generic BSL，给你恢复入口。

### 6）CHSW 不是总会运行

没有 valid BMI，或 BMI 禁用了 CHSW，后面的检查就不会做。

### 7）调试 bring-up 时最有价值的是 STMEM1 / STMEM2 / STMEM3~6

它们就是固件留给应用和工程师的“启动黑匣子”。

### 8）CAN BSL 的可用性高度依赖 OSC 配置

未配置 OSC 时能力受限，且 application reset 重入 CAN BSL 不可行。

### 9）应用软件不要假设 reset 后 RAM/PSPR 一定保持

SSW 可能覆盖 CPU0 DSPR 前 8KB、CPU0 PSPR 前 1KB。

### 10）很多“启动异常”其实是设计使然，不是 Bug

比如：

- CHSW 没跑：可能是 BMI 无效或 CHSW 被禁
- ESR0 延时不准：可能是 application reset 后时钟已被改
- CAN BSL 不支持某波特率：可能是 OSC 未配置
- 无法进入应用：可能是 default configuration 被禁，结果卡在 endless NOP loop

------

## 二十二、如果你现在要拿这篇文档做实际调试，建议按这个顺序看

### 第一步：先看 STMEM1 / STMEM2

确认：

- 有没有 `BMI_VALID`
- `BOOT_CFG` 是什么
- `BOOT_PIN` 有没有参与
- `BOOT_ADDR` 最终是多少

### 第二步：如果模式不对，回头查 BMHD

检查：

- BMHDID
- BMI 编码
- STAD 是否合法
- CRC 是否正确
- UCB 状态是不是 CONFIRMED / UNLOCKED

### 第三步：如果是 ABM 失败

检查：

- ABMHDID
- STADABM
- Header CRC
- CHKSTART/CHKEND 代码区 CRC

### 第四步：如果启动后行为不稳，再看 CHSW

重点看：

- `BMSEL_CF / BMSEL_CP`
- `PWAIT_CF / PWAIT_CP`
- `CCUCON0_CF / CCUCON0_CP`
- `FLASHCON1_CF / FLASHCON1_CP`
- `EVRTx` / `PLLTRIM` / `FMTRIM` 等相关项

### 第五步：如果你走 BSL

- ASC：看引脚、波特率、D5h 握手
- CAN：先确认 OSC 配置，再看波特率支持和 Init Msg1/2 流程

------

## 二十三、最后给你一个“中文脑图式总结”

这篇文档其实可以浓缩成下面这张“逻辑图”：

**复位发生**
→ **SSW 在 CPU0 上从 BootROM 开始执行**
→ 初始化 Flash / 芯片配置 / RAM / SOTA
→ 顺序评估 BMHD0~3
→ 根据 BMHD + 引脚决定：
  - Internal Start
  - ABM
  - ASC / Generic BSL
  - 或无可用模式
→ 如配置要求，执行 LBIST / Lockstep / Debug/HARR / ESR0
→ 若满足条件，执行 CHSW
→ 最后根据 `BOOT_ADDR` 跳转用户代码。
其中所有决策结果和检查结果，都通过 **STMEM1~6** 暴露给你。

------

下一条我可以继续把 **BMHD / ABMHD / STMEM1~6** 整理成一张中文对照表，再给你配一个“最小可启动配置”示例。





那我继续，把你最关心的这几块整理成“**能直接拿来调试**”的版本。

## 1. BMHD 中文对照表

BMHD 是 SSW 选启动模式的核心结构。TC3xx 有 4 组 BMHD（BMHD0~3），每组都有 Original 和 Copy，放在 DFLASH 的 UCB 里。SSW 会按顺序评估它们。

### BMHD 里的关键字段

`BMI`
这是 16 位的 Boot Mode Index，里面最关键的子字段有：

- `PINDIS[0]`：是否允许用 HWCFG 引脚参与模式选择；0=允许，1=禁止。
- `HWCFG[3:1]`：启动模式编码；`111`=Internal Start，`110`=ABM，`100`=Generic BSL，`011`=ASC BSL。
- `LSENA0..3[7:4]`：CPU0~CPU3 的 lockstep 使能。
- `LBISTENA[8]`：是否让 SSW 在冷上电后触发 LBIST。
- `CHSWENA[11:9]`：CHSW 是否执行；其中 `101B` 明确表示“禁用 CHSW”。
- `reserved[15:12]`：保留位，必须配 0。

`BMHDID`
必须是 `B359H`，否则这个 BMHD 直接无效。

`STAD`
32 位启动地址。

- Internal Start 时，它是用户代码入口。
- ABM 时，它不是代码入口，而是 **ABM Header 地址**。
- BSL 时这个地址不参与实际启动，但**仍然必须是合法的、字对齐的 PFLASH 地址**。这点非常重要。

`CRCBMHD / CRCBMHD_N`
BMHD 自身的 CRC 和反码。SSW 会计算并比对它们。

------

## 2. SSW 是怎么判定 BMHD 有效的

SSW 对单个 BMHD 的基本检查顺序是：

1. 先看这个 UCB_BMHDx 的状态是不是 `CONFIRMED` 或 `UNLOCKED`。
2. 检查 `BMHDID`。
3. 检查 `BMI` 是否合法。
4. 检查 `STAD` 是否是合法、字对齐的 PFLASH 地址。
5. 计算 `CRCBMHD / CRCBMHD_N`。
6. 若允许引脚选模，再看 `HWCFG3` 和 `HWCFG[5:4]`；否则按 BMI 选模。
7. 如果最终模式是 ABM，再进入 ABM 评估。
8. 成功后把结果写到 `STMEM1 / STMEM2`，并置 `BMI_VALID` 和 `BOOTMODE_CONFIGURED`。

------

## 3. ABMHD 中文对照表

ABM 不是“跳到另一个地址”这么简单，而是“**带头部和代码完整性校验**”的启动。ABM Header 可以放在 PFLASH 里任意字对齐位置，它的起始地址来自 BMHD 里的 `STAD`。

### ABMHD 字段

`STADABM`
ABM 模式下真正的用户代码入口。必须在完整 PFLASH 范围内、字对齐。

`ABMHDID`
必须是 `FA7C B359H`。

`CHKSTART / CHKEND`
要做 CRC 校验的代码区起止地址。注意二者必须都在 cached 段，或者都在 non-cached 段，**不能混用**。

`CRCRANGE / CRCRANGE_N`
代码区 CRC 及其反码。

`CRCABMHD / CRCABMHD_N`
ABM Header 自身的 CRC 及其反码。

### ABM 通过条件

SSW 会依次检查：

- `ABMHDID`
- `STADABM`
- `CRCABMHD / CRCABMHD_N`
- `CHKSTART..CHKEND` 对应代码区的 `CRCRANGE / CRCRANGE_N`

全部通过后，才把 `BOOT_ADDR` 设为 `STADABM`。

如果 ABM 失败，而且它是**由引脚选出来**、并且发生在最后一个 BMHD 的评估路径上，固件可以按 `HWCFG[5:4]` 回退到 ASC BSL 或 Generic BSL。

------

## 4. STMEM1~6 中文对照表

这些寄存器就是平台固件给你的“启动黑匣子”。

### STMEM1：SSW 最终“选了什么”

`SCU_STMEM1` 记录启动结果，最重要的字段有：

- `BMI_VALID[1]`：有没有找到有效 BMHD/BMI。
- `BOOT_PIN[2]`：本次是否是按引脚选的模式。
- `BMHD_COPY[3]`：用的是 Original 还是 Copy。
- `BMHD_INDEX[5:4]`：用的是 BMHD0/1/2/3 哪一个。
- `BOOT_CFG[8:6]`：SSW 最终采取的启动模式；`111`=Internal Start，`110`=ABM(失败转 Generic BSL)，`101`=ABM(失败转 ASC BSL)，`100`=Generic BSL，`011`=ASC BSL。
- `SCRDIS[9]`：SCR 是否因启动失败被 SSW 禁掉。
- `BOOTMODE_CONFIGURED[10]`：这次 SSW 是否成功选出启动模式。
- `HARREQ[14]`：是否收到了 Halt-After-Reset 请求。
- `SWAP_CFG / SWAP_TARGET / SWAP_DW_INDEX`：SOTA/SWAP 相关结果。

### STMEM2：SSW 最终“准备跳到哪里”

`SCU_STMEM2.BOOT_ADDR[31:2]` 表示 SSW 结束后要跳转的第一条用户指令地址。文档明确说它就是“the first user-code instruction, taken after SSW”，而且天然按字对齐。

### STMEM3~6：CHSW 检查状态

CHSW 用 4 个寄存器记录“每项检查的生命周期”：

- `STMEM3.*_CS`：check started，检查已开始。
- `STMEM4.*_CF`：check failed，检查失败。
- `STMEM5.*_CP`：check passed，检查通过。
- `STMEM6.*_CE`：check ended，检查结束。

文档给了总判定表：

- 未执行：`CS=0, CF=0, CP=0, CE=0`
- 通过：`CS=1, CF=0, CP=1, CE=1`
- 失败：`CS=1, CF=1, CP=0, CE=1`

### STMEM3~6 中常见检查位

位号在 STMEM3~6 中是一致的，最常用的是：

- `1 CHSWGEN`：CHSW 总体状态
- `2 BMSEL`：Bootmode selection
- `3 SWAP`：SWAP 配置
- `4/5/6/7`：CPOR/WPOR/SYSR/APPR
- `11 RSTCON3`
- `12 CHIPID`
- `13 CCUCON0`
- `14 PLLTRIM`
- `15 PWAIT`
- `16/17 EMEMT1/EMEMT2`
- `20 CONVCTRL`
- `21/22/23/24 GETH/RIF0/RIF1/GETH1`
- `29 FLASHCON1`
- `31 FMTRIM`

另外，`STMEM3` 里 `CHSWGEN_CS=1` 表示 CHSW 已启动，`BMSEL_CS=1` 表示 Bootmode selection 检查已启动。
`STMEM4` 里 `CHSWGEN_CF=1` 表示 CHSW 失败，`BMSEL_CF=1` 表示 Bootmode selection 检查失败。
`STMEM5` 里 `CHSWGEN_CP=1`、`BMSEL_CP=1` 表示通过。
`STMEM6` 里 `CHSWGEN_CE=1`、`BMSEL_CE=1` 表示这两项检查已经执行结束。

------

## 5. 一个最小可启动配置示例：先把“稳定从 PFLASH 启起来”

下面这个是**工程上最朴素、最容易 bring-up** 的思路。它不是文档原样给出的现成模板，而是严格按文档字段规则整理出的“最小配置清单”。

### 目标

让芯片最简单地走 **Internal Start from Flash**。

### 建议配置

第一，选择一个 BMHD，例如 `BMHD0_ORIG`。
因为 SSW 会顺序评估 BMHD0~3，找到第一个有效的就用。

第二，把 `BMI.HWCFG` 配成 `111B`。
这表示 Internal Start。

第三，把 `PINDIS` 先配成 `1`。
这样启动模式只由 BMI 决定，不受板上 HWCFG 引脚影响，bring-up 最稳。文档定义上，`1` 就是禁用引脚选模。

第四，`LSENA0..3` 先都配 `0`。
先不引入 lockstep 复杂性。文档允许 0=disabled。

第五，`LBISTENA=0`。
先不要让 SSW 冷启动时主动触发 LBIST。

第六，`CHSWENA` 有两种做法：

- **最简 bring-up**：配成 `101B`，先禁 CHSW。
- **想保留启动后检查**：配成非 `101B`。
  因为文档明确写了 `101B` 会禁用 CHSW。

第七，`reserved[15:12]=0`。
这是硬要求。

第八，`BMHDID=B359H`。

第九，`STAD=你的应用入口地址`。
这个地址必须满足两条：

- 在完整 PFLASH 范围内
- 字对齐
  文档特别强调，这个要求**无论是否走 BSL、无论是否由引脚选模，都始终成立**。

第十，正确生成 `CRCBMHD` 和 `CRCBMHD_N`。
否则 BMHD 会被 SSW 判无效。

第十一，确保这个 UCB_BMHD 的状态是 `CONFIRMED` 或 `UNLOCKED`。
否则 SSW 根本不会采用它。

------

## 6. 这个“最小配置”启动成功后，你应该看到什么

复位后，理论上你最先该检查：

### `STMEM1`

应该看到：

- `BMI_VALID=1`
- `BOOT_PIN=0`（如果你用了 `PINDIS=1` 且不是引脚选模）
- `BMHD_INDEX=0`（如果你用的是 BMHD0）
- `BOOT_CFG=111B`
- `BOOTMODE_CONFIGURED=1`

### `STMEM2`

应该等于你的 `STAD`。因为 Internal Start 时，SSW 会把 `BMHDx.STAD` 装进 `BOOT_ADDR`。

------

## 7. 两个很容易踩的坑

### 坑 1：以为 BSL/ABM 时 STAD 可以乱写

不行。文档明确说，`STAD` 必须始终是合法、字对齐的 PFLASH 地址，即便当前模式是 BSL，它“没有效果”，这个约束也仍然成立。

### 坑 2：把 `0xA000A020` 当成通用应用入口

不建议。这个地址在文档里是“**没有有效 BMHD、并且 default configuration 也不允许正常选模**”时，SSW 安装进去的一个特殊 fallback 地址，它不是通用的 Internal Start 示例入口。

------

## 8. 你现在可以把它理解成一句话

**最小可启动系统 = 一个状态有效的 BMHD + 正确的 BMHDID + 合法的 STAD + 正确 CRC + 合法 BMI。**
只要这几件事对了，SSW 就会把结果写进 `STMEM1/2`；如果还开了 CHSW，就再把检查结果写进 `STMEM3~6`。

下一步我可以继续给你做一版：**“BMHD/ABMHD 的伪代码结构体 + 启动排错流程图（看寄存器一步步定位）”**。



好，下面我把它整理成你真正能拿去做 bring-up / 调试的版本：

# 一、BMHD / ABMHD 的“伪代码结构体”

先说明一下：下面不是官方头文件，而是**按文档字段语义重组的逻辑结构**，目的是帮助你在工程里理解“SSW 到底在检查什么”。BMHD 有 4 组，位于 DFLASH 的 UCB 中，每组都有 `ORIG` 和 `COPY`。

## 1）BMHD 逻辑结构

```c
typedef struct
{
    // 16-bit BMI
    struct {
        uint16_t PINDIS   : 1;  // 0=允许 HWCFG 引脚选模，1=禁止
        uint16_t HWCFG    : 3;  // 111=Internal Start, 110=ABM, 100=Generic BSL, 011=ASC BSL
        uint16_t LSENA0   : 1;  // CPU0 lockstep
        uint16_t LSENA1   : 1;  // CPU1 lockstep
        uint16_t LSENA2   : 1;  // CPU2 lockstep
        uint16_t LSENA3   : 1;  // CPU3 lockstep
        uint16_t LBISTENA : 1;  // 1=冷上电时由 SSW 触发 LBIST
        uint16_t CHSWENA  : 3;  // 101b=禁用 CHSW，其它值=允许 CHSW
        uint16_t reserved : 4;  // 必须为 0
    } BMI;

    uint16_t BMHDID;      // 必须 = 0xB359
    uint32_t STAD;        // Internal Start: 用户入口；ABM: ABM Header 地址；始终必须是合法 PFLASH 字对齐地址
    uint32_t CRCBMHD;     // BMHD CRC
    uint32_t CRCBMHD_N;   // BMHD CRC 反码
} BMHD_t;
```

这几个字段的定义，文档给得很明确：
`PINDIS` 控制是否允许 HWCFG 引脚参与模式选择；`HWCFG` 定义启动模式；`LSENA0..3` 控制 CPU0~3 的 lockstep；`LBISTENA` 控制冷上电时是否跑 LBIST；`CHSWENA=101B` 表示禁用 CHSW；`BMHDID` 必须是 `B359H`；`STAD` 必须始终位于 PFLASH 且字对齐。

其中最容易踩坑的一条是：

> **`STAD` 的合法性约束始终成立**，哪怕你当前选的是 BSL，哪怕模式是通过引脚选出来的，也一样要满足“PFLASH 内、word-aligned”。

------

## 2）ABMHD 逻辑结构

ABM Header 放在 PFLASH 任意字对齐位置，它的首地址来自“那个选中 ABM 的 BMHD 的 `STAD`”。`CHKSTART` 和 `CHKEND` 必须同属 cached 段或同属 non-cached 段，不能混用。

```c
typedef struct
{
    uint32_t STADABM;      // ABM 真正用户入口，必须在 PFLASH 内且字对齐
    uint32_t ABMHDID;      // 必须 = 0xFA7CB359
    uint32_t CHKSTART;     // 待校验代码起始地址
    uint32_t CHKEND;       // 待校验代码结束地址
    uint32_t CRCRANGE;     // [CHKSTART..CHKEND] 的 CRC
    uint32_t CRCRANGE_N;   // 上述 CRC 的反码
    uint32_t CRCABMHD;     // ABM Header 自身 CRC
    uint32_t CRCABMHD_N;   // 上述 CRC 的反码
} ABMHD_t;
```

文档定义：
`STADABM` 是 ABM 模式下真正执行的入口；`ABMHDID` 必须是 `FA7C B359H`；`CHKSTART/CHKEND` 指定待检查代码区；`CRCRANGE`/`CRCRANGE_N` 对应代码区完整性；`CRCABMHD`/`CRCABMHD_N` 对应 Header 自身完整性。

------

# 二、把 SSW 的 BMHD 判定过程改写成“伪代码”

这段最重要，因为它就是你调试时脑子里该跑的流程。

```c
bool eval_one_bmhd(BMHD_t* bmhd, bool is_copy, int index)
{
    // 1) UCB 状态必须是 CONFIRMED 或 UNLOCKED
    if (!ucb_state_ok(index, is_copy)) return false;

    // 2) BMHDID 必须正确
    if (bmhd->BMHDID != 0xB359) return false;

    // 3) BMI 必须合法（HWCFG 编码有效，未定义位为 0）
    if (!bmi_valid(bmhd->BMI)) return false;

    // 4) STAD 必须是合法 PFLASH 字对齐地址
    if (!pflash_word_aligned_addr_valid(bmhd->STAD)) return false;

    // 5) BMHD CRC / 反码必须通过
    if (!bmhd_crc_ok(bmhd)) return false;

    // 6) 决定模式来自 pins 还是 BMI
    if ((bmhd->BMI.PINDIS == 0) &&
        (DMU_HF_PROCONTP.BML == 0) &&
        (SCU_STSTAT.HWCFG3 == 0))
    {
        BOOT_PIN  = 1;
        BMHD_INDEX = index;
        BOOT_CFG = decode_hwcfg_pins(SCU_STSTAT.HWCFG54);
    }
    else
    {
        BOOT_PIN  = 0;
        BMHD_INDEX = index;
        BOOT_CFG = decode_bmi_mode(bmhd->BMI.HWCFG);
    }

    // 7) 若当前选择的是 ABM，转入 ABM 评估
    if (BOOT_CFG is ABM)
    {
        if (!eval_abm((ABMHD_t*)bmhd->STAD))
            return false;
    }
    else
    {
        // 8) 不是 ABM，则决定最终入口
        if (BOOT_CFG is BSL)
            BOOT_ADDR = CPU0_PSPR_begin;
        else
            BOOT_ADDR = bmhd->STAD;
    }

    // 9) 保存结果给 HSM / 应用查看
    SCU_STMEM1 <- BOOT_CFG, BMHD_INDEX, BMHD_COPY, BOOT_PIN;
    SCU_STMEM2 <- BOOT_ADDR;

    // 10) 按 BMI 配 lockstep
    install_lockstep_from_bmi(bmhd->BMI.LSENA0..3);

    // 11) 成功标志
    BMI_VALID = 1;
    BOOTMODE_CONFIGURED = 1;
    return true;
}
```

这不是我拍脑袋写的，而是按文档里的流程顺序压缩来的：
SSW 先检查 UCB_BMHDx 状态是否是 `CONFIRMED/UNLOCKED`，再检查 `BMHDID`、`BMI`、`STAD`、`CRCBMHD/CRCBMHD_N`；随后决定是按 pins 还是按 BMI 选模式；如果选到 ABM 就继续做 ABM 评估；否则直接把 `BOOT_ADDR` 设为 `CPU0_PSPR_begin` 或 `BMHDx.STAD`；最后把结果写入 `STMEM1/2`，并配置 lockstep，再置位 `BMI_VALID` 和 `BOOTMODE_CONFIGURED`。

还有一个常被忽略的现实细节：
逻辑上 SSW 会尝试 `ORIG -> COPY`，但文档说明，**实际同一时刻通常只有其中一个会处于 `CONFIRMED/UNLOCKED`**，所以 `COPY` 常常只是兜底路径。

------

# 三、ABM 判定过程的伪代码

```c
bool eval_abm(ABMHD_t* abmhd)
{
    // 1) Header ID
    if (abmhd->ABMHDID != 0xFA7CB359)
        goto fail;

    // 2) 真正入口地址
    if (!pflash_word_aligned_addr_valid(abmhd->STADABM))
        goto fail;

    // 3) ABM Header 自身 CRC
    if (!abm_header_crc_ok(abmhd))
        goto fail;

    // 4) 代码区 CRC
    if (!range_crc_ok(abmhd->CHKSTART, abmhd->CHKEND,
                      abmhd->CRCRANGE, abmhd->CRCRANGE_N))
        goto fail;

    // 5) 全通过，真正生效
    BOOT_ADDR = abmhd->STADABM;
    return true;

fail:
    // 仅当 ABM 是“由 pins 选中”且发生在最后一个 BMHD 上，
    // 才允许根据 HWCFG[5:4] 回退到 ASC BSL 或 Generic BSL
    if (BOOT_PIN && BMHD_INDEX == 3)
    {
        BOOT_CFG  = fallback_bsl_from_hwcfg();
        BOOT_ADDR = CPU0_PSPR_begin;
        return true;
    }
    return false;
}
```

这正对应文档的 ABM 流程：
先检查 `ABMHDID`，再检查 `STADABM`，再验 `CRCABMHD/CRCABMHD_N`，然后校验 `[CHKSTART..CHKEND]` 的 `CRCRANGE/CRCRANGE_N`；全通过才把 `BOOT_ADDR` 设为 `STADABM`。如果失败，并且满足“由 pins 选中 + 最后一个 BMHD”的条件，才会退到 BSL。

------

# 四、启动排错流程图：按寄存器一步步定位

这一部分最适合你拿着调试器现场看。

## Step 0：先记住两件事

第一，SSW 的启动主流程是：
**Flash/RAM/配置 → 评估 BMHD → 可选 LBIST → lockstep → 执行选中的启动模式 → 可选 CHSW → 调试/HARR → ESR0 → 跳用户代码。**

第二，固件专门把启动结果写进了 `SCU_STMEM1/2`，把 CHSW 结果写进了 `SCU_STMEM3..6`，所以**调试第一现场不是猜，而是先读 STMEM**。

------

## Step 1：先看 `SCU_STMEM1.BOOTMODE_CONFIGURED`

### 情况 A：`BOOTMODE_CONFIGURED = 1`

说明 SSW 已经成功选出了启动模式。`STMEM1` 里可以继续看：

- `BMI_VALID`
- `BOOT_PIN`
- `BMHD_COPY`
- `BMHD_INDEX`
- `BOOT_CFG`

### 情况 B：`BOOTMODE_CONFIGURED = 0`

说明 **SSW 没有基于 BMHD/pins 成功选出启动模式**。这种情况要立刻区分两条分支：

#### B1）“没有有效 BMHD，但 default configuration 还允许”

SSW 会走兜底路径：

- 初始化 security-sensitive RAM
- 若没 HARR，则装入 Generic BSL，`BOOT_ADDR=CPU0_PSPR_begin`
- 若有 HARR，则装入 Internal Start，但实际先停在 HARR
- 最后把结果写到 `STMEM1/2`，并关闭 CPU0 lockstep

#### B2）“连 default configuration 都不允许”

那就更严重：
SSW **根本选不出任何启动模式**，此时只会：

- 若允许调试，就开调试口
- 然后进入 endless NOP loop

所以工程上只要看到 `BOOTMODE_CONFIGURED=0`，你第一反应不是“应用跑飞了”，而是：

> **BootROM 还没把你真正送到应用入口。**

------

## Step 2：如果 `BMI_VALID = 0`

这表示没有找到有效 BMHD/BMI。文档对 `BMI_VALID` 的定义非常直接：
`0` = last SSW execution 没找到 valid BMHD/BMI；`1` = 找到了。

这时排查顺序应该是：

```text
先查 UCB_BMHDx 状态
  -> 是否 CONFIRMED / UNLOCKED
再查 BMHDID
  -> 是否 0xB359
再查 BMI
  -> HWCFG 编码是否合法，未定义位是否为 0
再查 STAD
  -> 是否 PFLASH 内、是否字对齐
再查 CRCBMHD / CRCBMHD_N
  -> 是否匹配
```

这正是 SSW 对 BMHD 的官方检查顺序。

------

## Step 3：如果 `BMI_VALID = 1`，再看 `BOOT_CFG`

`BOOT_CFG` 的编码在 `STMEM1` 中定义得很清楚：

- `111B` = Internal start from Flash
- `110B` = ABM，失败回 Generic BSL
- `101B` = ABM，失败回 ASC BSL
- `100B` = Generic BSL
- `011B` = ASC BSL

所以你可以直接这样判断：

### 1）`BOOT_CFG = 111`

当前是普通 Flash 启动。
此时 `STMEM2.BOOT_ADDR` 应该就是最终要执行的用户入口地址。文档定义 `BOOT_ADDR` 是 “the first user-code instruction, taken after SSW”。

### 2）`BOOT_CFG = 100` 或 `011`

当前进了 BSL。
这时 `BOOT_ADDR` 应该是 `CPU0_PSPR_begin`，不是你的 PFLASH 入口。SSW 规范就是这么做的。

### 3）`BOOT_CFG = 110` 或 `101`

当前选择了 ABM 路径。
这时不要盯着 BMHD 的 `STAD` 当成代码入口看，因为它此时是 **ABM Header 地址**，真正入口在 `ABMHD.STADABM`。

------

## Step 4：如果是 ABM 问题，就只盯这四项

ABM 失败时，排查最好不要发散，直接只看四样：

```text
ABMHDID
STADABM
CRCABMHD / CRCABMHD_N
CRCRANGE / CRCRANGE_N
```

因为文档的 ABM 判定就是按这个顺序来的。

另外还要额外检查一个细节：
`CHKSTART` 和 `CHKEND` 必须在同一种访问段里，要么都 cached，要么都 non-cached，不能混。

------

## Step 5：如果“模式不对”，查 `BOOT_PIN` 和 `BMHD_INDEX`

`STMEM1` 里：

- `BOOT_PIN=1` 表示本次启动模式是由 HWCFG 引脚选出来的
- `BMHD_INDEX` 告诉你用的是 BMHD0/1/2/3 哪一个
- `BMHD_COPY` 告诉你用的是 ORIG 还是 COPY

所以常见误判是：

> “我明明把 BMI 配成 Internal Start，怎么还进 BSL/ABM？”

先别急，先看 `BOOT_PIN`。如果它是 1，说明这次不是 BMI 决定的，而是引脚决定的。SSW 只有在 `BMI.PINDIS=0`、`BML=00B`、并且 `HWCFG3` 采样为低时，才会真的允许引脚接管模式选择。

------

## Step 6：如果应用没起来，但你怀疑 CHSW

先别直接看应用，先判断 **CHSW 有没有资格运行**。

CHSW 只有同时满足这 3 条才会真正跑：

1. SSW 的 bootmode evaluation 被 CHSW 复核为正确
2. 这次启动确实找到了 valid BMI，并且它被用于选模
3. 该 BMI 没把 CHSW 禁掉，也就是 `CHSWENA != 101B`

所以：

- `BMI_VALID=0` 时，后续 CHSW 大概率根本不会跑
- `CHSWENA=101B` 时，CHSW 被显式禁用
- 你看到 “没有 CHSW 痕迹”，不一定是 Bug，很可能只是**前置条件不满足**

------

## Step 7：如果 CHSW 跑了，就看 `STMEM3..6`

CHSW 用 4 个寄存器表达“一个检查项的一生”：

- `STMEM3`：check started
- `STMEM4`：check failed
- `STMEM5`：check passed
- `STMEM6`：check finished

所以你可以把任意检查位按下面理解：

```text
CS=0 CF=0 CP=0 CE=0   -> 没执行
CS=1 CF=0 CP=1 CE=1   -> 执行并通过
CS=1 CF=1 CP=0 CE=1   -> 执行并失败
```

例如最常用的检查位有：

- `CHSWGEN`：CHSW 总体状态
- `BMSEL`：Bootmode selection
- `SWAP`：SWAP 配置
- `RSTCON3`：shutdown handler 入口
- `CHIPID`
- `CCUCON0`
- `PWAIT`
- `FLASHCON1`
  以及只在某些 reset 或某些产品上才有意义的 `EVRT1/2`、`CONVCTRL`、`GETH`、`RIF0/1` 等。

其中：

- `CHSWGEN_CS=1` 表示 CHSW 已开始
- `BMSEL_CS=1` 表示 Bootmode selection 检查已开始

而：

- `CHSWGEN_CF=1` 表示 CHSW 整体失败
- `BMSEL_CF=1` 表示 boot mode selection 检查失败

------

# 五、给你一个“最实用”的现场排错顺序

你真上板子时，我建议固定按这个顺序来：

```text
1. 读 SCU_STMEM1
   -> 看 BOOTMODE_CONFIGURED, BMI_VALID, BOOT_CFG, BOOT_PIN, BMHD_INDEX, BMHD_COPY

2. 读 SCU_STMEM2
   -> 看 BOOT_ADDR 到底是 PFLASH 入口，还是 CPU0_PSPR_begin

3. 如果 BMI_VALID=0
   -> 回查 BMHD：UCB 状态 / BMHDID / BMI / STAD / CRCBMHD

4. 如果 BOOT_CFG=ABM
   -> 回查 ABMHD：ABMHDID / STADABM / CRCABMHD / CRCRANGE

5. 如果 BOOT_PIN=1
   -> 查 HWCFG 引脚、电平锁存、以及 PINDIS/BML 条件

6. 如果期望 CHSW 但没看到结果
   -> 查 BMI_VALID、CHSWENA、以及 bootmode evaluation 是否成立

7. 如果 CHSW 失败
   -> 从 STMEM4 里优先找 *_CF=1 的第一个位
```

这套顺序几乎完全贴合固件自己的内部决策链，所以效率很高。对应依据分别是：
SSW 先做 BMHD 评估并把结果写入 `STMEM1/2`，之后才可能执行 CHSW；CHSW 再把检查生命周期写入 `STMEM3..6`。

------

# 六、最后给你一句工程化总结

你现在可以把整个 TC3xx 启动诊断浓缩成一句话：

> **先用 `STMEM1/2` 判断 SSW 到底“选了什么、跳到哪里”，再用 `STMEM3..6` 判断 CHSW “查了什么、哪项失败”；如果 SSW 连模式都没选出来，就回到 BMHD/ABMHD 的合法性和 CRC。**

如果你愿意，我下一条可以直接给你做一版 **“BMHD / ABMHD 初始化模板（接近可落地的 C 结构 + 字段填写示例）”**。





下面我把这些概念都用“**它是什么 → 在这篇文档里怎么用 → 你调试时怎么理解**”的方式讲清楚。

------

## 1）lockstep

**它是什么**
lockstep 可以理解成“**两个核同步做同一件事，并互相监督**”的安全机制。你可以把它想成“双人同时算同一道题，再比较答案”；如果结果不一致，就说明可能出现了硬件故障或执行异常。

**在这篇文档里怎么用**
文档里没有展开讲 lockstep 的底层原理，但明确说了：

- 只有**冷上电**时，SSW 才会做 lockstep 配置
- 如果启动时找到了有效 BMI，就把 `BMI.LSENAn` 的配置写进 `SCU_LCLCON0/1`
- 如果没找到有效 BMI，则至少把 `CPU0` 的 lockstep 关掉

另外，BMHD 里的 `LSENA0..3` 位就是在控制 CPU0~CPU3 的 lockstep 是否使能：0=关闭，1=开启。

**你调试时怎么理解**
看到 lockstep，你就把它理解成：

> “启动时顺手把安全冗余模式配好。”
> 它不是启动模式本身，而是**启动过程中附带做的一项安全配置**。

------

## 2）LBIST

**它是什么**
LBIST 是 **Logic Built-In Self Test**，也就是“**芯片内部逻辑自检**”。
通俗说，就是芯片上电后先给自己做一轮“体检”。

**在这篇文档里怎么用**
如果 BMHD 里 `LBISTENA=1`，SSW 会触发 LBIST。LBIST 做完后会引发一次**内部复位**，于是 SSW 会重新跑一遍；如果这次发现 `LBISTDONE=1`，就不会再次触发 LBIST，而是继续进入用户代码。文档也特别强调：**SSW 不判断 LBIST 是否通过**，真正结果要由应用软件去看 `SCU_LBISTCTRL3`。

BMHD 里这个位的定义也很明确：`LBISTENA=1` 表示冷上电时由 SSW 启动 LBIST。

**你调试时怎么理解**
看到 LBIST，不要把它当成“跑应用前的一个普通函数”。它更像：

> “上电后先自检一次，检完再重新启动流程。”

------

## 3）RAM 初始化

**它是什么**
RAM 初始化就是：**在启动时把选中的 RAM 清零，并把 ECC 位也一起写正确**。

**在这篇文档里怎么用**
文档明确说：SSW 支持 RAM 初始化，配置来自 `UCB_DFLASH`，启动时装到 `DMU_HF_PROCONRAM`。初始化会：

- 把选中的 RAM 填成全 0
- 同时安装正确的 ECC 位
  这样后续访问这些 RAM 时，不会因为 ECC 没准备好而报不可纠正错误。

而且它支持：

- 冷上电 / 暖上电可选
- 按 CPU 单独选
- 按 LMU 单独选

**你调试时怎么理解**
RAM 初始化的核心不是“为了清零好看”，而是：

> “把 RAM 和 ECC 状态整理成一个可安全访问的初始状态。”

------

## 4）Halt After Reset（HARR）

**它是什么**
HARR 就是“**复位后先别立刻跑用户程序，而是先停住，方便调试器接管**”。

**在这篇文档里怎么用**
文档里说，如果收到了 `CBS_OSTATE.HARR=1`，那 SSW 会把 `BOOT_CFG` 先装成 Internal Start，但**不会立即真的跳进应用运行**，而是优先执行 Halt After Reset。

后面 SSW 的具体处理是：

- 先确认调试访问整体上被允许
- 再确认确实请求了 HARR
- 然后在**第一条用户指令**处布一个 Break Before Make 断点
- 并启用 On-Chip Debug Support 系统

**你调试时怎么理解**
HARR 本质上就是：

> “芯片复位后，在应用真正开始跑之前先帮你刹车。”
> 这对调试启动早期问题特别有用。

------

## 5）ESR0

**它是什么**
ESR0 是一个和**复位/错误信号输出**有关的引脚。
在这份文档里，它主要是在讲：**SSW 什么时候把 ESR0 释放、释放前等多久**。

**在这篇文档里怎么用**
如果满足两个条件：

- `ESR0CNT != FFFH`
- 并且 ESR0 引脚当前配置成 open-drain 的 reset output

那么 SSW 会：

- 通过 `SCU_ESROCFG.ARC=1` 去释放 ESR0
- 按 `ESR0CNT` 设定一个延时
- 在跳到第一条用户指令前，等待 ESR0 真正变高电平

这个延时是用 `STM0` 生成的；文档还提醒，如果是 application reset，且用户代码之前改过默认时钟/STM 设置，那么 ESR0 的实际延时就可能不准。

**你调试时怎么理解**
ESR0 可以理解成：

> “芯片对外表示‘我这次复位还没完全结束/现在可以放行了’的一个脚。”
> SSW 在跳用户代码前，会把这个释放过程处理好。

------

## 6）BSL

**它是什么**
BSL 是 **Bootstrap Loader**。
最通俗地说，就是：**芯片自己带的一个最小下载器/引导器**，允许你不靠 PFlash 现有程序，而是通过外部接口把一段程序先下载进 RAM 再运行。

**在这篇文档里怎么用**
文档说，Bootstrap Loaders 的作用是：

- 通过选定接口加载用户程序
- 把代码搬到 `CPU0_PSPR`
- 在退出 BootROM 后开始执行

同时文档还指出，一旦进入 BSL，通信流程必须按定义完整跑完，**不会靠 timeout 自动跳出**；想重新开始只能 reset。

`BOOT_CFG` 里也能看到 BSL 是启动模式之一：

- `100B` = Generic Bootstrap Loader
- `011B` = ASC Bootstrap Loader

**你调试时怎么理解**
BSL 就是芯片自带的“救援启动模式”。
当 Flash 里没有合适程序、或者你想下载一段临时代码验证时，它特别有用。

------

## 7）A/B Bank 切换

**它是什么**
这就是常说的 **A/B 镜像切换**：
PFlash 有两个可切换的 Bank，启动时决定“现在用 A 还是用 B”。

**在这篇文档里怎么用**
文档把它放在 SOTA 里讲：SSW 会读取 `UCB_SWAP_ORIG/COPY` 里的配置，启用 SWAP 功能，从而在 PFlash 的两个 Bank 之间切换。
如果找到有效 SOTA 配置，SSW 会：

- 禁用对 PFlash 的 direct access
- 再在 `SRU_SWAPCTRL.ADDRCFG` 里设成 Address region A active 或 B active

`STMEM1.SWAP_CFG` 也明确给出结果编码：

- `01B` = SWAP A configured（Bank A 有效，B 无效）
- `10B` = SWAP B configured（Bank B 有效，A 无效）

**你调试时怎么理解**
A/B Bank 切换就是：

> “开机时决定当前主用哪套 Flash 镜像。”
> 这正是 OTA 升级里常见的双镜像策略。

------

## 8）CAN BSL

**它是什么**
CAN BSL 就是：**通过 CAN 总线进行下载的 Bootstrap Loader**。

**在这篇文档里怎么用**
文档明确写到，CAN BSL 会把外部主机通过 CAN 发来的程序/数据，装到 **CPU0 Program Scratchpad RAM** 里。它是**primary bootloader**，不要和第三方工具链提供的 secondary bootloader 混淆。

它的主要特性包括：

- 支持 Classical CAN
- 也支持 CAN FD，但 **只有在 Configured OSC 时**
- 初始标识符固定为 `0x555`
- 接收的数据帧数可编程

它的流程分三段：

1. **Phase 1**：识别主机发来的 `Init Message 1`，自动猜 CAN 波特率；识别失败就发错误消息 `0xAAA` 并结束。
2. **Phase 2**：主机发 `Init Message 2`，给更精确的波特率，或者启用 CAN FD 所需参数。
3. **Data Loading**：开始收 Programming messages，把数据写到 `CPU0_PSPR` 的 `C0000000H` 起始处；收完后回到主 SSW 流程。

**你调试时怎么理解**
CAN BSL 就是：

> “不靠串口，而是用 CAN 把启动代码灌进片上 RAM，再跑起来。”
> 在车规 MCU 场景里，这很常见。

------

## 9）OSC

**它是什么**
这里的 OSC 指的是 **Oscillator Circuit**，也就是芯片的振荡器/外部晶振相关时钟源配置。

**在这篇文档里怎么用**
文档在 CAN BSL 的上下文里，重点区分了两种状态：

### 未配置 OSC

如果 `DMU_HF_PROCONDF.OSCCFG=0`，通常表示还是出厂默认状态，没有把外部晶振参数按你的板子配置进去。
这时 CAN BSL 会用**内部 back-up clock** 给 MCAN 供时钟，所以支持的波特率更少。

### 已配置 OSC

如果 `OSCCFG=1`，启动过程会把相关信息装进 `CCU_OSCCON`，使用接在 `XTAL1/2` 上的外部晶振/谐振器。这样 MCAN 可以拿到完整的外部时钟，支持更完整的 CAN BSL 波特率范围。

文档还强调：如果你想支持“power-on 进 CAN BSL，之后 application reset 还继续进 CAN BSL”，那么**未配置 OSC 的器件做不到**，得先把 OSC 配好。

**你调试时怎么理解**
OSC 在这里你就理解成：

> “CAN BSL 能不能跑全功能、支不支持更多波特率，关键看时钟是不是已经按你的板子配置好了。”

------

## 10）PSPR

**它是什么**
PSPR 是 **Program Scratchpad RAM**，也就是“**程序临时存放/执行用的片上 RAM**”。

**在这篇文档里怎么用**
文档多次明确说：

- BSL 会把下载下来的代码放到 `CPU0_PSPR`
- 然后用户代码从 `PSPR` 开头开始执行

ASC BSL 会把收到的 128 字节存到 `C0000000H` 起始的 `CPU0_PSPR`，退出 SSW 后也从这里开始执行。

CAN BSL 同样会把数据写到 `CPU0_PSPR` 的 `C0000000H` 起始处。

另外，文档提醒：启动过程本身可能覆盖 `CPU0 PSPR` 开头最多 1KB，所以**不能把必须跨复位保留的程序放在这块区域**。

**你调试时怎么理解**
PSPR 就是：

> “CPU0 的一块片上高速代码 RAM，Bootloader 常把临时下载的程序放这儿直接跑。”

------

## 11）CPU Halt

**它是什么**
CPU Halt 就是“**CPU 被暂停执行**”。

**在这篇文档里有两个语境**

### 语境 A：启动时其他 CPU 先别动

文档一开头就说，SSW 只在 `CPU0` 上执行，**其他 CPU 在 boot 期间都保持 Halt-state**，等用户软件以后再启动。

也就是说，启动初期是单核串行完成的，不是所有核一起跑。

### 语境 B：调试时 CPU 被调试器暂停

文档在 PSFDM 里说得更直接：当多个 CPU 被 OCDS 同时 halt 时，可能造成电流突变，所以 BootROM 里专门有一个 PSFDM 机制，用来**代替直接 halt**，减小电源冲击。

PSFDM 的意思是：发生 debug event 后，CPU 们先进入一个特殊 trap handler 循环；如果工具确实需要，也可以再单独 halt 某些 CPU。继续运行时，调试器再把这些 CPU 从 halt 里放出来，并让它们一起退出 PSFDM。

**你调试时怎么理解**
所以 CPU Halt 不是一个单一概念，它至少有两层意思：

- **启动阶段的 halt**：先只让 CPU0 跑，其他核先暂停
- **调试阶段的 halt**：被调试器停住，便于观察现场

------

## 最后给你一个“一句话总记忆法”

你可以先这样记：

- **lockstep**：安全冗余，同步双核监督
- **LBIST**：上电逻辑自检
- **RAM 初始化**：清零并补齐 ECC
- **HARR**：复位后先停住给调试器
- **ESR0**：复位释放相关引脚时序
- **BSL**：芯片自带下载启动器
- **A/B Bank 切换**：双镜像 Flash 启动切换
- **CAN BSL**：通过 CAN 下载到 PSPR 再执行
- **OSC**：时钟/晶振配置，决定 CAN BSL 能力
- **PSPR**：CPU0 的程序 scratchpad RAM
- **CPU Halt**：CPU 暂停，可能是启动时也可能是调试时

下一条我可以把这 11 个概念再整理成一张“**概念关系图**”，让你一眼看出它们在启动流程里的位置。

