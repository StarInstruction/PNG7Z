可以把这三个模块理解成一条分层链路：

**ComM（Communication Manager） → Nm（Network Management Interface） → CanNm（CAN Network Management）**

它们的关系不是“并列模块”，而是**上层做通信策略，中间层做统一适配，底层做 CAN 总线上的 NM 协议执行**。

### 1. 三个模块各自负责什么

**ComM** 是最上层的“通信资源管理者”。它负责协调不同唤醒事件、接收多个用户的通信请求、管理每个通道的通信状态机，并简化底层通信栈（尤其是网络管理）的使用。也就是说，**它决定“这个 ECU 现在要不要通信、要不要保持总线唤醒”** 。

**Nm** 是中间层的“总线无关适配层”。文档里写得很明确：Nm 位于 **ComM 和各个总线专用 BusNm（例如 CanNm）之间**，把 ComM 对网络的调用转发给对应 BusNm，再把 BusNm 的回调转发回 ComM；此外它还可以做多网络同步关断的协调 。

**CanNm** 是最下层的“CAN 专用 NM 执行者”。它只和 **Nm Interface** 交互，真正实现 CAN 上的网络管理算法，并通过 **AUTOSAR CAN Interface** 发送/接收 NM 报文 。

------

### 2. 静态架构上，它们怎么连

如果画成一条主链路，大致就是：

**Application / RTE / EcuM / Dcm 等**
→ **ComM**
→ **Nm**
→ **CanNm**
→ **CanIf**
→ **CAN 总线**

其中有两个关键点：

第一，**应用通常通过 ComM 提请求**，而不是直接去操作 CanNm。
第二，**CanNm 不直接面向应用，它没有 RTE service port**，所以它更像一个协议执行内核，主要由 Nm 驱动和接收回调 。

------

### 3. 动态上最核心的调用链：从“我要通信”到“总线被唤醒”

这个流程最能说明三者关系。

#### 3.1 应用先找 ComM

当上层请求 `COMM_FULL_COMMUNICATION` 时，ComM 先要求 **BusSM** 进入 Full Communication；等 BusSM 确认已经进入 Full Communication 之后，ComM 再去请求 BusNM 主动或被动启动网络管理 。

这里文档虽然写的是 “BusNM”，但在你这组三份文档对应的 CAN 场景下，**实际链路就是 ComM → Nm → CanNm**，因为 Nm 正是把 ComM 的请求转发到对应 BusNm（这里就是 CanNm）的适配层 。

#### 3.2 Nm 把请求转给 CanNm

Nm 的职责就是把诸如 `Nm_NetworkRequest / Nm_NetworkRelease / Nm_PassiveStartUp` 这类调用转到对应的 BusNm 上。
所以在 CAN 通道上，**Nm 收到来自 ComM 的网络请求后，会落到 CanNm 上执行** 。

#### 3.3 CanNm 真正在 CAN 上发 NM 报文

CanNm 被请求后，会离开 Bus-Sleep，进入 Network Mode/Repeat Message 等状态；它通过 `CanIf_Transmit` 发 NM 报文，CAN Interface 再通过 `CanNm_TxConfirmation` / `CanNm_RxIndication` 回调给 CanNm 。

同时，CanNm 在模式变化时会通知 Nm：

- 进入 Bus-Sleep：`Nm_BusSleepMode()`
- 进入 Network Mode：`Nm_NetworkMode()`
- 离开 Network Mode：`Nm_PrepareBusSleepMode()`

这些都是 CanNm 向 Nm 上报的状态变化 。

------

### 4. 回调方向正好反过来：CanNm → Nm → ComM

这条反向链也很重要，因为它说明 **ComM 不需要理解 CAN 协议细节，但它能知道网络状态**。

Nm 文档明确列出了它会调用 ComM 的这些回调：

- `ComM_Nm_NetworkStartIndication`
- `ComM_Nm_RestartIndication`
- `ComM_Nm_NetworkMode`
- `ComM_Nm_BusSleepMode`
- `ComM_Nm_PrepareBusSleepMode`

也就是说：

- **CanNm 先通知 Nm**
- **Nm 再把这些事件转发给 ComM**

例如：

- 当总线睡眠时收到 NM 报文，Nm 会把 `Nm_NetworkStartIndication` 转发给 ComM；ComM 收到后，会在下一个 `ComM_MainFunction` 中启动对应网络 。
- 当 Nm 进入 `Prepare Bus-Sleep`，ComM 会把对应 Bus State Manager 切到 **Silent Communication**，自己也进入 Silent Communication 。
- 当 Nm 最终进入 `Bus-Sleep`，ComM 会把对应 Bus State Manager 切到 **No Communication**，自己也进入 No Communication 。

所以从行为上看，**CanNm 决定总线 NM 协议状态，Nm 负责“翻译和转发”，ComM 根据这些状态去驱动整个 ECU 的通信模式**。

------

### 5. 为什么中间一定要有 Nm，不能 ComM 直接连 CanNm？

因为 **ComM 是总线无关的**，它不能写死只懂 CAN。
Nm 的存在，就是为了把 ComM 和 CAN/LIN/FlexRay/UDP 等不同 BusNm 隔离开。Nm 文档也明确说它支持 generic BusNm，以及多种标准 BusNm；这就是它的抽象层价值 。

换句话说：

- **ComM 只关心“我要不要通信”**
- **Nm 负责把这个意图路由到正确的总线 NM**
- **CanNm 负责“在 CAN 上具体怎么做到”**

------

### 6. 初始化顺序也能说明依赖关系

文档里给出的初始化顺序很有代表性：

- **CanNm** 要在 **CanIf 初始化之后、Nm 初始化之前** 初始化
- **ComM** 要在 **BusSM 和 Nm 初始化之后** 再初始化

所以依赖顺序基本就是：

**CanIf / BusSM → CanNm → Nm → ComM**

这和前面的架构关系完全一致。

------

### 7. 进阶一点：三者在“同步休眠/协调关断”里的关系

如果 ECU 有多个网络，Nm 还能提供协调算法，让多个网络同步关断；它会持续保持所有网络唤醒，直到所有网络都 ready-to-sleep，再统一开始 shutdown 。

而在 CAN 侧，CanNm 提供 `CanNm_SetSleepReadyBit()`，通过 NM 报文里的 Sleep Ready Bit 把“这个网络是否 ready to sleep”传播出去，这正是对 Nm 协调功能的底层支撑 。

所以在协调休眠场景里又能看出三层分工：

- **ComM**：管理 ECU/通道通信状态
- **Nm**：做多网络协调算法
- **CanNm**：通过 CAN NM 报文把协调状态发到总线上

------

### 8. 一句话总结

这三者的关系可以概括成：

**ComM 决策，Nm 转发/协调，CanNm 执行。**

再具体一点就是：

- **ComM** 负责“是否需要通信”的策略管理
- **Nm** 负责“把策略映射到具体总线 NM，并把底层状态回送上来”
- **CanNm** 负责“在 CAN 上发/收 NM 报文，维护 CAN NM 状态机”

如果你愿意，我可以下一步把这三者画成一张“启动流程图 + 休眠流程图”的中文时序图。





这三个接口最容易混淆的地方在于：**前两个是“向下发起请求”**，最后一个是**“向上报告事件”**。

先给你一句最直白的结论：

- **Nm_PassiveStartUp**：把 NM **叫醒**，但语义上是“**被动启动**”
- **Nm_NetworkRequest**：明确声明“**本 ECU 现在真的要通信**”
- **Nm_NetworkStartIndication**：不是请求，而是通知“**我在 Bus Sleep 时收到了别人的 NM 报文，网络已经被别的节点启动了**”

------

## 1. 从调用方向上看，根本不是一类东西

Nm 模块本身就是 **ComM 和 BusNm（例如 CanNm）之间的适配层**：ComM 往下调 Nm，Nm 再转给 CanNm；而 CanNm 的回调又通过 Nm 往上转给 ComM 。

所以：

- **Nm_PassiveStartUp / Nm_NetworkRequest**：通常是 **ComM → Nm → CanNm**
- **Nm_NetworkStartIndication**：通常是 **CanNm → Nm → ComM**

这就是它们的第一层区别。

------

## 2. 分别看这三个接口的官方语义

### A. `Nm_PassiveStartUp`

Nm 文档写得很明确：它是“**requests a passive start-up of the network management**”，Nm 会去调用对应 BusNm 的 passive start-up 函数 。
但如果这个网络是 **coordinated network**，那它不会走 passive start-up，而会改成调用对应 BusNm 的 `NetworkRequest` 。

落到 CanNm 上，这个接口的含义是：

- 从 **Bus Sleep / Prepare Bus Sleep**
- 切到 **Network Mode 的 Repeat Message**
- 如果当前不在这两个睡眠相关状态，调用无效并返回 `E_NOT_OK`

再结合 CanNm 的行为说明：当通过 `CanNm_PassiveStartUp` 或 `CanNm_NetworkRequest` 进入 `Repeat Message` 后，若这是**被动请求**，`Repeat Message` 结束后下一个状态会是 **Ready Sleep**，不是长期维持 Normal Operation 。

**所以它的本质是：**
“先把 NM 启起来、跟上网络节奏，但我不一定表达‘我要一直占着总线通信’。”

------

### B. `Nm_NetworkRequest`

Nm 文档对它的定义是：
“**requests the network and the bus communication**”，Nm 会调用对应 BusNm 的 `NetworkRequest` 。

CanNm 对应定义更直接：
“**Request the network, since ECU needs to communicate on the bus**” 。

这句话非常关键，因为它把 `NetworkRequest` 和 `PassiveStartUp` 的区别说透了：

- `PassiveStartUp` 强调的是 **启动 NM**
- `NetworkRequest` 强调的是 **本 ECU 需要总线通信**

另外，CanNm 文档还说明：当 CAN NM 已经处于 **Network Mode** 时，如果上层再次调用 `CanNm_NetworkRequest`，CanNm 可以重新切到 **Repeat Message**，从而触发立即发送 NM 报文（例如用于 PN 信息尽快上总线） 。

**所以它的本质是：**
“我不仅要醒，而且我要明确占用网络、保持通信。”

------

### C. `Nm_NetworkStartIndication`

这个接口完全不是“请求”。
Nm 文档写的是：当 **在 `Bus Sleep` 状态下收到了 NM 报文**，说明网络中有一些节点已经重启并进入了 `Network Mode`，这个通知会被转发给 ComM 。

ComM 侧对应的描述也很明确：
`ComM_Nm_NetworkStartIndication` 用来通知 COMM，**这次 restart 是因为 Nm 在 Bus-Sleep 模式下收到了一个 NM message**；ComM 会先把这个事件存起来，再在下一次 `ComM_MainFunction` 中启动对应网络 。

CanNm 状态图也支持这一点：
在 `Bus Sleep` 下，如果发生 **Rx Nm Msg**，CanNm 会通知 `NetworkStart Indication`；而 `Network requested / PassiveStartUp / Rx Nm Msg` 都可以使它进入 NetworkMode 并启动 NM 传输 。

**所以它的本质是：**
“不是我主动请求的，而是我发现别人已经把网络叫醒了，于是我往上报一个‘网络已启动’事件。”

------

## 3. 最核心的区别：谁主动、谁被动、谁是通知

你可以这样记：

### `Nm_PassiveStartUp`

**我方主动发起，但语义是被动启动。**
重点在“把 NM 从睡眠拉起来”，不强调“我要持续保持总线活跃”。在 CanNm 里，典型效果是进入 `Repeat Message`，之后如果只是被动请求，会转去 `Ready Sleep` 。

### `Nm_NetworkRequest`

**我方主动发起，而且语义是主动占用网络。**
重点在“本 ECU 现在确实需要 bus communication”，因此它比 PassiveStartUp 更强 。

### `Nm_NetworkStartIndication`

**不是我方请求，而是底层上报外部事实。**
重点在“Bus Sleep 时收到别人的 NM 报文”，说明其他节点已经启动了网络，于是通知 ComM 跟进处理 。

------

## 4. 放到实际场景里就很清楚了

### 场景 1：我这边只是想跟着网络起来

例如 ECU 因某个唤醒流程，需要先进入 NM，但不想表达“我要一直保持通信”。

这时更像是用 **`Nm_PassiveStartUp`**。
它会把 CanNm 从 Bus Sleep/Prepare Bus Sleep 拉到 Network Mode/Repeat Message 。

------

### 场景 2：我这边应用真要发报文、做诊断、通信

这时应该用 **`Nm_NetworkRequest`**。
因为它的语义就是“ECU needs to communicate on the bus” 。

------

### 场景 3：我本来已经睡了，但总线上别人先醒了

这时不是 ComM 去调启动接口，而是底层收到 NM 报文后触发 **`Nm_NetworkStartIndication`**，再一路上报给 ComM 。

------

## 5. 一个容易踩坑的点：`PassiveStartUp` 不等于“别人启动了网络”

很多人会把这两个混在一起：

- **`Nm_PassiveStartUp`**：是**我主动调用的 API**
- **`Nm_NetworkStartIndication`**：是**底层回调通知我，别人已经启动了网络**

两者都可能导致最终“我这边也进入 Network Mode”，但**触发源完全不同**：

- 前者：**本地软件主动发起**
- 后者：**外部网络事件被动触发**

------

## 6. 一张对照表

| 接口                        | 性质     | 调用方向      | 触发原因                       | 核心语义              |
| --------------------------- | -------- | ------------- | ------------------------------ | --------------------- |
| `Nm_PassiveStartUp`         | 服务接口 | 上层→Nm→BusNm | 本地想把 NM 拉起               | 被动启动 NM           |
| `Nm_NetworkRequest`         | 服务接口 | 上层→Nm→BusNm | 本地 ECU 确实要通信            | 主动请求网络/保持通信 |
| `Nm_NetworkStartIndication` | 回调通知 | BusNm→Nm→ComM | Bus Sleep 时收到别人的 NM 报文 | 网络已被外部节点启动  |

------

## 7. 最后给你一个最好记的口诀

- **PassiveStartUp**：**我把 NM 叫醒，但我不一定声明自己要一直占着网络**
- **NetworkRequest**：**我要通信，请把网络保持活跃**
- **NetworkStartIndication**：**不是我叫醒的，是别人先醒了，我这里只是收到通知**

如果你愿意，我下一步可以继续帮你把这三个接口放进 **ComM 状态机 + CanNm 状态机** 里，画成一条完整时序：
**ComM_RequestComMode → Nm_PassiveStartUp / Nm_NetworkRequest → CanNm 状态切换 → Nm_NetworkStartIndication 回调**。





这两个接口虽然名字都像“通知”，但**语义方向正好相反**：

- **`Nm_NetworkStartIndication`**：表示**网络被唤醒/启动了**
- **`Nm_RemoteSleepIndication`**：表示**别的节点基本都准备睡了**

而且它们有一个共同点：二者都属于 **Nm 的 callback**，是由 BusNm/CanNm 往上通知 Nm 的，不是 ComM 往下调用的服务接口。Nm 文档里 5.4 章就说明了，这一章描述的是“由其他模块调用、Nm 实现的回调函数” 。

------

## 1. `Nm_NetworkStartIndication` 是“网络启动通知”

Nm 文档定义得很直接：

> 当在 **Bus Sleep** 状态收到了一个 NM message，说明网络里有些节点已经重启并进入了 **Network Mode**；这个通知会转发给 ComM。

所以它的触发条件非常明确：

- **当前本节点/该网络在 Bus Sleep**
- **这时收到了 NM 报文**
- 于是判定：**外部节点已经把网络重新拉起来了**

### 它表达的含义

不是“大家要睡了”，而是相反，表示：

**“网络已经重新活跃起来了，你上层要准备进入通信流程。”**

------

## 2. `Nm_RemoteSleepIndication` 是“远端睡眠准备通知”

Nm 文档对它的定义是：

> 网络管理检测到**网络上所有其他节点都已经 ready to sleep**。这个通知可以选择性转发到上层。

CanNm 文档把这个场景解释得更清楚：

> 为了同步网络，有时需要知道是否**已经没有其他网络节点再需要 bus communication**；这就叫 **Remote Sleep Indication**。其开始由 `Nm_RemoteSleepIndication()` 指示；如果之后在 **Normal Operation** 或 **Ready Sleep** 又收到了 NM message，就会调用 `Nm_RemoteSleepCancellation()`。

所以它的核心不是“已经睡着”，而是：

**“我检测到别的节点都不再需要通信，已经进入可睡眠趋势了。”**

注意这里很关键的一点：

- `RemoteSleepIndication` ≠ 已经 Bus Sleep
- 它只是表示 **远端节点 ready to sleep / no more bus communication required**

------

## 3. 最本质的区别：一个是“醒”，一个是“快睡了”

你可以这样对比：

| 接口                        | 含义         | 触发时机                            | 说明                                               |
| --------------------------- | ------------ | ----------------------------------- | -------------------------------------------------- |
| `Nm_NetworkStartIndication` | 网络启动了   | **Bus Sleep** 状态下收到 NM 报文    | 说明别的节点已经重启并进入 Network Mode            |
| `Nm_RemoteSleepIndication`  | 远端准备睡了 | 检测到**其他节点都 ready to sleep** | 说明远端已基本不再需要通信，但还不是最终 Bus Sleep |

所以从系统含义上讲：

- **`NetworkStartIndication` = 唤醒方向事件**
- **`RemoteSleepIndication` = 休眠方向事件**

------

## 4. 从状态机角度看，更容易理解

### `Nm_NetworkStartIndication`

它发生在 **Bus Sleep → Network 活跃** 的边界上。
因为文档明确说：**Bus Sleep 收到 NM message**，说明一些节点已经 restart 并进入 `Network Mode` 。

也就是：

**休眠中 → 被别人叫醒了**

------

### `Nm_RemoteSleepIndication`

它发生在 **Normal Operation / Ready Sleep → 更接近睡眠** 的过程中。
CanNm 文档说得很明确：如果进入 remote sleep indication 后，在 **Normal Operation** 或 **Ready Sleep** 又收到 NM message，就要触发 `RemoteSleepCancellation` 。

这说明 `RemoteSleepIndication` 典型发生在：

- 网络还没真正睡着
- 但已经观察到“远端都准备休眠了”

也就是：

**通信逐步收敛 → 接近可休眠**

------

## 5. 对 ComM / 上层的意义也不同

### `Nm_NetworkStartIndication`

这个通知会**转发给 ComM** ，意义是：

**“上层注意，网络被外部节点重新启动了。”**

所以它通常会驱动 ComM 往 Full Communication 方向处理。

### `Nm_RemoteSleepIndication`

它主要表达的是：

**“远端节点已经没有通信需求了，协调关断/睡眠判断可以往前推进。”**

如果是多 BusNm/协调网络场景，Nm 还会聚合 remote sleep 状态；文档明确写到，多 BusNm 时，只有当最后一个 BusNm 也指示了 Remote Sleep，才会上报给上层 。

------

## 6. 一句最好记的区分

- **`Nm_NetworkStartIndication`**：
  **我在睡着时收到了 NM 报文，所以说明网络被别人唤醒了。**
- **`Nm_RemoteSleepIndication`**：
  **我观察到别的节点都不再需要通信了，所以说明网络在向睡眠收敛。**

------

## 7. 再压缩成一句话

**`NetworkStartIndication` 看的是“有人醒了”；`RemoteSleepIndication` 看的是“别人都快睡了”。**

如果你愿意，我可以继续把这两个接口放进 **CanNm 状态机时序** 里，画成一条“唤醒路径”和“一条休眠路径”的对照图。





在 **Partial Network（PN/PNC）** 场景里，这三个模块不是平均分工，而是明显分层：

- **ComM**：管理 **PNC 状态机**、决定哪些 **channel** 该被拉起、是否做 **gateway 路由**
- **Nm**：不是 PN 业务的核心处理者，它是 **ComM 和 CanNm 之间的总线无关适配层**，负责把 ComM 的请求往下转给 CanNm，再把 CanNm 的状态变化往上转给 ComM
- **CanNm**：真正处理 **CAN NM 报文里的 PN 信息**，包括 **CRI bit、PN filter、EIRA/ERA 聚合、PN 请求传播**；并且 CanNm 只和 **Nm Interface** 交互

所以在 PN 场景下，最准确的理解是：

**CanNm 负责“看懂并发送 PN 信息”，ComM 负责“根据 PN 信息决定 PNC/Channel 怎么动”，Nm 负责“把 ComM 和 CanNm 串起来”。**

------

## 1）先把 PN 场景里的“主语”分清

### ComM 关心的是 **PNC**

ComM 文档明确说，PN 场景下它有一套 **per-PNC 的状态机**；它通过两类 bit vector 和其他 ECU 交换 PNC 状态：

- **EIRA**
- **ERA**

并且 **ERA 只有在启用了 PNC Gateway，且该 PNC 是 coordinated（映射到多个 ComM channel）时才会被评估**

也就是说，**ComM 看的不是某一帧 CAN NM 本身，而是“这个 PNC 当前对 ECU/通道意味着什么”**。

### CanNm 关心的是 **NM 报文里的 PN 信息**

CanNm 文档说明：

- NM 报文里有 **CRI bit / Cluster Request Information**
- CanNm 会对收到的 PN 信息做 **filter**
- 然后做两种聚合：
  - **EIRA**：全 ECU 维度，外部+内部请求聚合
  - **ERA**：按网络维度，只聚合外部请求

所以 **CanNm 是 PN 信息的“采集器和发送器”**。

### Nm 在 PN 场景里主要是“桥”

Nm 的基本职责就是：
**ComM → Nm → BusNm（这里是 CanNm）**，以及 **CanNm → Nm → ComM**

------

# 2）场景一：其他 ECU 在 CAN 上发来了 PN 请求，这三个模块怎么联动

这是 PN 最典型的外部激活路径。

## 第一步：CanNm 收到 NM 报文，先判断这条 PN 请求对本 ECU 有没有意义

CanNm 的 PN filter 逻辑是：

- 如果 **CRI bit = 0**，这条 NM 报文对 PN 来说不相关
- 如果 **CRI bit = 1**，CanNm 解析 user data 里的 CRI 内容
- 再和本 ECU 配置的 **PN filter mask** 比较
- 至少有一个相关 bit 匹配，这条 NM 才算“relevant for the ECU”

所以 **PN 请求的第一道筛选发生在 CanNm**，不是 ComM，也不是 Nm。

## 第二步：CanNm 把相关 PN 请求聚合成 EIRA / ERA

CanNm 对 PN 请求的聚合规则是：

- **EIRA**：把收到的和自己发出的 PN 请求，跨所有网络聚合成全 ECU 状态
- **ERA**：按每个网络单独聚合收到的外部 PN 请求
- 当某个 cluster 首次被请求时，会存储并启动 timer；请求重复到来则 timer 重启；timer 超时则删除该请求

也就是说，**CanNm 并不只是“看到一帧就完事”，而是把 PN 请求保持成带超时语义的状态**。

## 第三步：CanNm 把 EIRA/ERA 的变化送到 Com 栈

文档写得很明确：
**只要 EIRA 或 ERA 发生变化，CanNm 就会更新 Com 里的对应 I-PDU，并调用 `PduR_CanNmRxIndication()`**

这里很关键：

**PN 信息从 CAN 总线进入 ECU 后，不是直接“CanNm 通知 ComM：某个 PNC=1”**
而是：

**CanNm → PduR/Com（更新 EIRA/ERA 信号）→ ComM 的 Com 回调**

## 第四步：ComM 收到 EIRA/ERA 信号变化，推进 PNC 状态机

ComM 提供了生成的 `ComM_ComCbk_<SignalName>` 回调。它的功能就是：

> 当用于传输 partial network channel request information 的 ComSignal 发生变化时通知 ComM；这个信号就是对应的 **EIRA_RX 或 ERA ComSignal**

而在 ComM 的 PNC 状态机里：

- 当 `ComM_COMCbk()` 检测到 **PNC bit within EIRA = 1**
- 或满足 **ERA 相关条件**
- PNC 会离开 `PNC_NO_COMMUNICATION`
- 并进一步触发 `Channel Full Communication Request()`

所以外部 PN 请求的完整链路可以写成：

**CAN NM 报文 → CanNm 解析 CRI/PN filter → 生成/更新 EIRA/ERA → ComM_ComCbk → ComM PNC 状态机进入 REQUESTED/FULL → 请求相关 channel 唤醒**

------

# 3）场景二：本 ECU 自己的用户请求某个 PNC，这三个模块怎么联动

这条路径和上面正好相反，是 **本地请求向总线扩散**。

## 第一步：应用/上层用户请求 PNC

ComM 的 PNC 状态机表明：

- `ComM_RequestComMode()` 对 PNC user 的 FullCom 请求
- 会让 PNC 从 `PNC_NO_COMMUNICATION` 进入活跃态
- 并执行 `Com_SendSignal()` 与 `Channel Full Communication Request()`

这说明在 PN 场景里，**ComM 不是直接去改 CanNm user data，而是先从自己的 PNC/Channel 逻辑出发**。

## 第二步：ComM 决定要不要把 channel 拉到 Full Communication

ComM 的总通信控制规则是：

- 当请求 `COMM_FULL_COMMUNICATION` 时
- 先让 **BusSM** 切到 Full Communication
- **BusSM 确认后**，ComM 再去请求 BusNM 主动/被动启动

在 CAN 场景下，这里的 BusNM 实际链路就是：

**ComM → Nm → CanNm**

因为 Nm 就是那层适配器

## 第三步：新的 PN 请求要尽快出现在总线上

CanNm 文档专门写了两种“新 PN 内部请求要立刻上总线”的机制：

### 方案 A：通过 Com 的 Transmission on Change

如果 NM user data 通过 Com 设置，并且 signal 配成 on-change immediate transmit，user data 变化就会触发额外一次 NM message 发送。这个方案要求 `Pn Handle Multiple Network Requests = OFF`

### 方案 B：通过再次 `CanNm_NetworkRequest`

当 CAN NM 已经在 `Network Mode` 时，如果上层再次调用 `CanNm_NetworkRequest`，CanNm 会重新进入 `Repeat Message`，从而**立即发送 NM message，并在之后以更快周期连发几帧**。这个机制要求：

- `Immediate Nm Transmission` 已启用
- PN feature 已启用
- `Pn Handle Multiple Network Requests = ON`

这就是 PN 场景里非常关键的一点：

**ComM 决定“我要这个 PNC”，CanNm 负责“怎么把这个新的 PN bit 尽快广播出去”。**

------

# 4）场景三：PN Gateway 路由时，这三个模块怎么分工

这里 **ComM 是绝对主角**。

ComM 文档明确说：

> ComM 负责 coordinated partial networks 的 gateway behavior，也就是把一个 channel 上收到的 PNC activation request 路由到其他 channel；这种路由是通过发送 **EIRA TX signals** 完成的

并且文档还强调：

> **只有 ComM 可以写 TX EIRA 信号，应用不允许自己写**

所以在 gateway 场景里：

- **CanNm** 负责把各 channel 上的外部 PN 请求变成 ERA/EIRA 输入
- **ComM** 决定这些请求该不该、以及如何路由到其他 channel
- **Nm/CanNm** 再负责把被唤醒的 channel 真正拉起来并发 NM 报文

## 三种 gateway type 的差异

### 1. 请求从 PASSIVE channel 收到

如果 ERA=1 是从 **PASSIVE** gateway channel 收到：

- 请求**不镜像回本 channel**
- 也**不路由到其他 PASSIVE channel**
- 只路由到 **ACTIVE channel**
- PNC 目标状态是 `COMM_PNC_REQUESTED`
- 协调 channel 中：
  - PASSIVE channel 的目标状态是 `COMM_FULL_COM_READY_SLEEP`
  - ACTIVE channel 的目标状态是 `COMM_FULL_COM_NETWORK_REQUESTED`

### 2. 请求从 ACTIVE channel 收到

如果 ERA=1 是从 **ACTIVE** gateway channel 收到：

- 请求会被**镜像回本 channel**
- 并路由到**所有其他 coordinated channels**
- 无论 PASSIVE 还是 ACTIVE，目标状态都变成 `COMM_FULL_COM_NETWORK_REQUESTED`

### 3. 请求从 gateway type NONE 的 channel 收到

如果 ERA=1 是从 **gateway type NONE** 的 channel 收到：

- 请求被忽略，不存到内部 ERA
- 不回写本 channel
- 不路由到其他 channel
- 不会设置到 EIRA TX signal
- 但这种 channel 仍然会处理通过 **EIRA Rx** 收到的请求

这一段能看出：

**PN gateway 路由策略是 ComM 决定的；CanNm 不负责决定“转不转发到别的 channel”。**

------

# 5）场景四：为什么 PN 场景里 ComM 还需要 Nm 的 `StateChangeNotification`

这是很多人最容易忽略的点。

表面上看，PN 信息主要走的是：

**CanNm → EIRA/ERA → ComM_ComCbk**

但 PN 还涉及一个很关键的功能：

## PNC to Channel Routing Limitation

ComM 支持 **Pnc to Channel Routing Limitation**，也就是在运行时选择性限制某个 PNC 对某个 channel 的路由。文档写得很清楚：

- 这个 feature 允许用 `ComM_LimitPncToChannelRouting()` 限制某个 channel
- 被限制的 channel 不会因为别的 channel 上收到的 PNC activation request 而被唤醒
- 于是 gateway 不会把 PNC 请求继续路由到这个受限 channel

## Routing Limitation 不是静态的，它跟 Nm 状态绑定

ComM 里定义了 channel 上 Routing Limitation 的三种状态：

- **Disabled**
- **Partly disabled**
- **Enabled**

并且规则里明确写到：

- Disabled 时，ComM 会保持 channel 唤醒并路由请求；Nm 会在 gateway 发出的 NM message 里把对应 PNC vector bit 置 1
- Partly disabled 时，ComM 不保持 channel 唤醒，但仍会路由请求；Nm 仍会在发出的 NM message 里置位
- Enabled 时，ComM 不保持 channel 唤醒，也不会再有 gateway 发出的 NM message

而 `ComM_Nm_StateChangeNotification()` 的作用就是：

> 当 Nm state 改变时通知 ComM；**Pnc Routing Limitation state** 会根据 Nm 是否进入/离开 `NM_STATE_REPEAT_MESSAGE` 来更新

这说明：

**Nm 在 PN 里虽然不解析 PNC，但它通过“NM 状态变化”影响 ComM 的路由限制策略。**

## 这条链路怎么接起来

- CanNm 状态变化时，会调用 `Nm_StateChangeNotification()`
- Nm 可以把这个通知转给上层配置的 callback
- PN routing limitation 场景下，这个 callback 要配置成 `ComM_Nm_StateChangeNotification`，而且文档要求在 Nm Interface 和相关 BusNm 里都打开 `State Change Ind Enabled`

所以这部分的真实关系是：

**CanNm 提供 NM 状态变化 → Nm 转发 → ComM 用它来控制 PN 路由限制**

------

# 6）场景五：PN 请求释放、PNC 进入睡眠时，三者如何联动

这里要把 **PNC 释放** 和 **channel 物理睡眠** 分开看。

## A. PNC 逻辑上先收敛

在 ComM 的 PNC 状态机里，当：

- `ComM_COMCbk()` 发现 **PNC bit within EIRA = 0**
- 或本地 PNC user 释放请求

ComM 会启动 `ComMPncPrepareSleepTimer`，进入准备睡眠路径

而且 ComM 特别提醒：

> `ComMPncPrepareSleepTimer` 必须在 Network Management 离开 Ready Sleep 之前先到期；
> 对 CAN 来说，关键窗口和 `CanNm Timeout Time`、`CanNm PNC Reset Time` 相关

这说明 **PNC 先是“逻辑上没人再要了”，然后才是底层总线真的进入睡眠**。

## B. CanNm 侧的 PN 请求会因 timer 超时而被删除

CanNm 对 EIRA/ERA 里的每个 cluster request 都是带 timer 的：

- 首次置位时启动 timer
- 重复请求就重启 timer
- timer 到期时删除请求

所以从 PN 信息传播角度看，**CanNm 负责“让 PN 请求自动老化消失”**。

## C. Channel 物理睡眠仍然走普通 NM 链路

当 channel 侧最终准备休眠时：

- CanNm 进入 Prepare Bus Sleep / Bus Sleep 会先通知 Nm，再由 Nm 转给 ComM
- ComM 收到 `ComM_Nm_PrepareBusSleep` 后，把对应 BusSM 切到 Silent Communication；收到 `ComM_Nm_BusSleepMode` 后，再切到 No Communication

所以 PN 场景下休眠链路其实分两层：

- **PNC 层**：ComM 根据 EIRA/ERA 和用户请求决定这个 PNC 该不该退
- **Channel/NM 层**：Nm/CanNm 负责把对应总线真的退到 Prepare Bus Sleep / Bus Sleep

------

# 7）把整条 PN 交互压成两条主链

## 主链 1：外部 PN 请求进来

**CAN 总线 NM 报文**
→ **CanNm** 解析 CRI bit / PN filter
→ **CanNm** 更新 EIRA/ERA，并通过 `PduR_CanNmRxIndication()` 更新 Com 信号
→ **ComM_ComCbk_`** 收到 EIRA/ERA 变化
→ **ComM** 推进 PNC 状态机、决定要唤醒哪些 channel
→ **ComM → Nm → CanNm** 请求相关 channel 进入通信

## 主链 2：本地用户请求某个 PNC

**Application / SWC**
→ `ComM_RequestComMode()`
→ **ComM** 让 PNC 进入 REQUESTED，并发起 `Channel Full Communication Request()` / `Com_SendSignal()`
→ **ComM → Nm → CanNm** 启动相关 channel 的 NM
→ **CanNm** 通过 immediate transmit 或再次 `NetworkRequest`，尽快把新的 PN bit 广播到总线

------

# 8）一句最核心的结论

在 Partial Network 场景里：

- **ComM 负责“PNC 语义”**：谁请求了、该唤醒哪些 channel、要不要做 gateway 路由、是否受 Routing Limitation 限制
- **CanNm 负责“PN 报文语义”**：CRI bit、PN filter、EIRA/ERA 聚合、PN bit 上总线
- **Nm 负责“控制通路和状态回路”**：ComM 的 channel 请求往下送给 CanNm，CanNm 的 NM 状态变化再回送给 ComM

所以你可以把它记成：

**PN 的“内容处理”在 CanNm，PN 的“策略决策”在 ComM，PN 的“模块连接”靠 Nm。**

你要的话，我下一步可以直接给你画一张 **PN 外部请求时序图** 和一张 **PN 本地请求+网关转发时序图**。





这些词确实容易绕。它们很多都不是“新功能”，而是在描述 **PN（Partial Network）里不同层次的对象、位图和处理规则**。
我先把它们翻成大白话，再把它们串起来。

------

## 先用一句话建立整体图景

你可以把 **整车网络** 想成一栋楼，**PNC** 就像楼里一组“可以一起开灯/关灯的房间”。
某个 ECU 通过 CAN NM 报文说：“我现在需要 1 号房间和 3 号房间亮起来。” 这份“我要哪些房间亮”的信息，就放在 **CRI** 里。
**CanNm** 先判断这份请求和自己有没有关系（看 **PN filter mask**），然后把有效请求整理成 **EIRA / ERA**；
**ComM** 再根据 EIRA / ERA 推进 **每个 PNC 自己的状态机**，决定要不要把某些 channel 唤醒，或者把请求转发到别的 channel。
这些状态机是 **per-PNC** 的；如果一个 PNC 涉及多个 channel，它就是 **coordinated**，此时还可能用到 **PNC Gateway**。这一整套行为在文档里有明确描述 。

------

## 1）PNC 是什么

**PNC = Partial Network Cluster**。
它不是一根真实的总线，也不是一个物理 ECU，而是一个**逻辑分组**：把一组“应该一起被唤醒/一起允许通信”的功能或节点，定义成一个 Partial Network Cluster。ComM 文档写明，PNC 有自己的状态机，而且**这个状态机是“per partial network”存在的**，也就是“每个 PNC 各管各的” 。

文档还说，每个 PNC 在 EIRA/ERA 这种 bit vector 里都占用**固定的 bit 位置**，这个位置由 **PNC ID** 定义；某个 PNC 的 bit = 1 就表示 active，bit = 0 表示 inactive 。

### 大白话理解

PNC 就像一个“逻辑开关组”。
例如：

- PNC 5：车门相关
- PNC 7：座椅相关
- PNC 9：空调相关

是不是这几个例子，文档没给具体业务含义，但机制上就是这种“按组控制”。

------

## 2）CRI 是什么

**CRI = Cluster Request Information**。CanNm 的缩写表里写的是 **Partial Network Cluster Request Information** 。

在 CanNm 的 NM 报文里，Control Bit Vector 里有一个 **Cluster Request Information Bit**，这个 bit 就是给 Partial Networking 用的；如果这个 bit 被置位，说明这条 NM 报文里带了 PN 请求信息 。

同时，CanNm 文档还说：

- CRI 的具体位置和长度可配置
- **CRI 里的每一位表示一个 cluster**
- 哪个 cluster 的 bit = 1，就表示这个 cluster 被请求了

### 大白话理解

CRI 就是一张“**我要哪些 PNC**”的请求表。
比如某帧 CRI 是：

- bit0 = 1
- bit1 = 0
- bit2 = 1

那就表示“我要 PNC0 和 PNC2，不要 PNC1”。

------

## 3）PN filter mask 是什么

CanNm 不会盲目接受所有 PN 请求。
文档说得很清楚：不是每个 cluster 都和当前 ECU 有关，所以会对收到的 CRI 内容应用一个 **可配置的 PN filter mask**。如果某个 cluster 对本 ECU 不 relevant，就把对应 mask bit 配成 0；**只有当收到的 PN 信息和本 ECU 的 filter mask 至少有一位匹配时，这条 NM 报文才对本 ECU relevant** 。

### 大白话理解

它就是一张“**我关心哪些 PNC**”的过滤表。

例子：

- 收到的 CRI：`10110000`
- 本 ECU 的 PN filter mask：`00110000`

两者按位一对，发现中间有至少一个共同的 1，说明“这条请求里有我关心的 PNC”，于是这条消息对我有效。

如果完全对不上，就说明“别人请求的那些 PNC 跟我无关”，那我可以忽略它。

------

## 4）EIRA / ERA 是什么

CanNm 说它会用两种略有不同的算法去**聚合 PN 请求**：

- **EIRA = External Internal Requests Aggregated**
- **ERA = External Requests Aggregated**

### 先理解“聚合”是什么意思

这里的“聚合”不是简单说“收到一帧就记一下”，而是把多个来源、多次出现、带超时语义的请求，整理成一个更稳定的状态。
文档说：

- 某个 cluster 第一次被请求时，会存储这个请求并启动 timer
- 如果后面又重复收到请求，timer 重新开始
- timer 到期后，这个请求被删除

所以聚合的本质是：

**把一连串分散的 PN 报文，整理成“当前到底哪些 PNC 还在被请求”的状态表。**

------

## 5）EIRA 是什么

文档定义非常直接：

- **external（收到的）**
- 加上 **internal（自己发出的）**
- **跨所有网络/所有 channel**
- 聚合成一个 combined state，叫 **EIRA**
- 因此，EIRA 表示的是：**整个 ECU 维度上，哪些 partial networks 当前是 active 的**

### 你问的那句话怎么理解

> “把收到的和自己发出的 PN 请求，跨所有网络聚合成全 ECU 状态”

这句话其实就是 EIRA 的定义。

拆开看：

- **收到的**：别人 ECU 发来的 PN 请求
- **自己发出的**：本 ECU 自己也可能请求某些 PNC
- **跨所有网络**：如果这个 ECU 有多个网络/多个 channel，都一起统计
- **全 ECU 状态**：最后得到一张“对这个 ECU 来说，目前哪些 PNC 活跃”的总表

### 大白话例子

假设 ECU 有两个 CAN channel：

- CAN1 上别人请求了 PNC2
- CAN2 上本 ECU 自己请求了 PNC5

那 EIRA 看到的结果就是：

- PNC2 = 1
- PNC5 = 1

它不关心这两个请求分别来自哪个 channel，只关心**整个 ECU 当前有哪些 PNC 被激活**。

------

## 6）ERA 是什么

ERA 的定义是：

- **只看 external（收到的）**
- **按每个网络分别保存**
- 因此 ERA 表示的是：**在某个具体网络上，哪些 PNC 是被其他 ECU 请求的，因此出于外部需求必须保持 active**

文档还特别强调：**每个 network 都有自己的 ERA I-PDU** 。

### 你问的那句话怎么理解

> “按每个网络单独聚合收到的外部 PN 请求”

这句话就是 ERA。

拆开看：

- **按每个网络单独**：不是全 ECU 合并，而是 CAN1 一份、CAN2 一份
- **收到的外部**：只看别人 ECU 发来的请求，不算我自己发出去的
- **聚合**：多帧、多次请求、带超时地整理成当前状态

### 大白话例子

还是 ECU 有两个 channel：

- CAN1 上别人请求了 PNC2
- CAN2 上别人请求了 PNC5

那 ERA 会是：

- ERA(CAN1)：PNC2 = 1
- ERA(CAN2)：PNC5 = 1

它保留了“**请求来自哪个网络**”这个信息。

------

## 7）EIRA 和 ERA 的区别，一次说透

最容易混的是这两个。

### EIRA

看的是：

- 收到的 + 自己发出的
- 全 ECU 角度
- 不区分具体哪个网络

也就是：
**“整个 ECU 现在到底有哪些 PNC 在活跃。”**

### ERA

看的是：

- 只看收到的外部请求
- 每个网络各自保存
- 强调“外部网络需求来自哪条网络”

也就是：
**“在这条具体网络上，其他 ECU 要我保持哪些 PNC 活跃。”**

### 最好记的口诀

- **EIRA**：全局总表
- **ERA**：分网络外部请求表

------

## 8）per-PNC 是什么意思

这个词不是某个新模块，它只是说：

**某个行为/状态机/配置，是“按每个 PNC 单独存在”的。**

ComM 文档明确说，PNC 状态机 **exists per partial network** 。

### 大白话理解

不是“整个 ECU 只有一个 PNC 状态”，而是：

- PNC1 有自己的状态
- PNC2 有自己的状态
- PNC3 也有自己的状态

一个 PNC 可以在 READY_SLEEP，另一个 PNC 同时还在 REQUESTED。它们互相独立。

------

## 9）PNC Gateway 是什么

ComM 文档写得非常直接：

**ComM 负责 coordinated partial networks 的 gateway behavior**，也就是把一个 channel 上收到的 PNC activation request，**路由到其他 channel**；这个路由是通过发送 **EIRA TX signals** 完成的 。

### 大白话理解

假设同一个 PNC 同时涉及两条总线：

- 这个请求先在 CAN-A 上收到了
- 但为了让 CAN-B 上相关节点也醒来
- ECU 需要把这个请求“转发”到 CAN-B

这就是 **PNC Gateway**。

所以 Gateway 的本质不是普通 IP 路由那种“转发报文”，而是：

**把一个 PNC 的激活需求，从一个 channel 传播到其他相关 channel。**

------

## 10）coordinated 是什么意思

这个词在这三份文档里其实有两层含义，容易混。

### 在 ComM 的 PNC 场景里

ComM 明确说：ERA 只有在 PNC Gateway 打开、而且这个 PNC 是 **coordinated** 时才会评估；这里的 coordinated 指的是：**这个 PNC 被分配到了多个 ComM channel** 。

所以在 PN 语境里，经常说的“coordinated PNC”基本就是：

**一个 PNC 关联了多个 channel，需要跨 channel 协同。**

### 在 Nm 的协调器语境里

Nm 里 coordinated network 的意思是：某个 network 属于一个 coordinator cluster，多个 network 要一起同步关断。Nm 还区分 active coordinated / passive coordinated 。

### 你当前最需要记住的

在你现在问的 PN 场景里，**coordinated 先按 ComM 这层理解就够了**：

**一个 PNC 不是只映射一条 channel，而是映射了多条 channel。**

------

## 11）“把收到的和自己发出的 PN 请求，跨所有网络聚合成全 ECU 状态”到底在说什么

这句话其实就是在解释 **EIRA**，不是另一个新概念。
CanNm 文档原话就是：external 和 internal 的 PN requests，**over all networks (channels)** 聚合成 combined state，这个 state 包含了**整个 ECU 上哪些 partial networks 是 active** 。

### 换成最简单的话

它不是问“请求从哪条总线来的”，而是在问：

**“站在这个 ECU 整体角度，现在有哪些 PNC 被认为是活跃的？”**

------

## 12）“按每个网络单独聚合收到的外部 PN 请求”到底在说什么

这句话就是在解释 **ERA**。
文档说：ERA 只聚合 external（received）requests，并且是 **per network** 保存，所以它表示的是：**每个网络上，哪些 PNC 是被别的 ECU 请求的** 。

### 换成最简单的话

它不是全局总表，而是分账本：

- CAN1 一本
- CAN2 一本
- 每本只记“别人要我开的 PNC”

------

## 13）把这些概念串成一条真实处理链

你把它理解成下面这条链，就不容易乱了：

### 第一步：别人发来一条带 PN 信息的 NM 报文

报文里如果 **CRI bit = 1**，就表示这里带了 Partial Networking 请求信息 。

### 第二步：CanNm 解析 CRI 内容

CRI 里的每个 bit 对应一个 cluster / PNC 请求 。

### 第三步：CanNm 用 PN filter mask 过滤

只保留和本 ECU 相关的 PNC 请求 。

### 第四步：CanNm 做 EIRA / ERA 聚合

- EIRA：全 ECU，总表
- ERA：每网络，外部请求表
  并且每个请求都带 timer，重复请求重启 timer，超时后删除 。

### 第五步：ComM 读取这些位图，推进每个 PNC 的状态机

ComM 的 PNC 状态机是 **per-PNC** 的，而且可以根据 EIRA/ERA 的 bit 变化做状态跳转 。

### 第六步：如果这个 PNC 是 coordinated 的

ComM 可能启用 **PNC Gateway**，把在一个 channel 上收到的请求路由到别的 channel，方法是发送 EIRA TX signal 。

------

## 14）最后给你一套“超简版记忆法”

### 对象类

- **PNC**：一个逻辑上的局部网络组
- **CRI**：报文里“请求哪些 PNC”的位图内容
- **PN filter mask**：本 ECU “关心哪些 PNC”的过滤条件

### 状态类

- **per-PNC**：每个 PNC 各有自己的状态机
- **coordinated**：这个 PNC 映射到了多个 channel，需要协同

### 聚合类

- **EIRA**：全 ECU 总表，收到的 + 自己发出的
- **ERA**：每网络分表，只看收到的外部请求
- **EIRA/ERA 聚合**：把零散报文整理成稳定状态，并且带超时老化

### 路由类

- **PNC Gateway**：把某条 channel 上的 PNC 激活请求传播到别的相关 channel

------

## 15）一句话总收束

你可以这样记：

**CRI 是“报文里说了什么”，PN filter mask 是“我关心什么”，EIRA/ERA 是“我整理后的结果”，per-PNC 是“每个 PNC 各自管理”，coordinated/PNC Gateway 是“一个 PNC 跨多个 channel 时怎么联动”。**

如果你愿意，我下一步可以直接给你画一张 **“PN 报文进入 ECU 后，CRI → filter → EIRA/ERA → ComM PNC 状态机 → Gateway 转发”** 的中文流程图。



这里的“映射”，你先不要理解成代码里的指针关系，**它本质上是“这个 PNC 需要作用到哪些通信通道上”**。

### 先说结论

**“一个 PNC 不是只映射一条 channel，而是映射了多条 channel”** 的意思就是：

> 某个 **PNC（一个逻辑上的局部网络功能组）**，不是只和一条 ComM channel 绑定，
> 而是同时和 **多个 ComM channel** 关联。
> 所以这个 PNC 一旦被请求，ComM 可能要同时处理多条 channel 的唤醒、保持、转发和休眠。

文档里把这种情况直接叫做 **coordinated PNC**，定义就是：
**“assigned to more than one COMM channel”**，也就是“被分配到不止一条 COMM channel” 。

------

## 1）channel 在这里是什么

在 ComM 里，**channel** 不是“某个报文”，而是 **一条被 ComM 管理的通信通道**。
ComM 对“每个 channel”都有独立状态机，文档明确说它实现了 **for every channel** 的 channel state machine 。
所以你可以把 channel 粗略理解成：

- 一条 CAN 网络
- 或一条 Ethernet 通道
- 或 ComM 配置中的一条系统通信通道

------

## 2）PNC 映射到 channel，是什么意思

文档里明确提到配置参数 **`ComMChannelPerPnc`**，并且说：

- 每个 PNC 至少要分配给 **一个 channel**
- 非 coordinated 的 PNC 可以改分配到另一个 channel
- coordinated 的 PNC 可以分配到其他 channels

所以“PNC 映射到 channel”本质上就是：

> 在配置里规定：这个 PNC 由哪些 channel 来承载、传播和执行它的通信需求。

### 用最简单的话说

如果：

- **PNC-A → Channel 1**

那就是这个 PNC 只跟 Channel 1 有关。

如果：

- **PNC-A → Channel 1 + Channel 2 + Channel 3**

那就是这个 PNC 不是单通道的，而是一个跨多个 channel 的 PNC。

------

## 3）为什么一个 PNC 会映射多条 channel

因为 **一个功能需求可能不是只靠一条总线完成**。

比如你可以想象这样一种逻辑场景：

- 某个功能在 **CAN1** 上有一部分节点
- 在 **CAN2** 上还有另一部分节点
- 这两边都属于同一个“局部功能组”

那这个功能组对应的 PNC，就不能只挂在一条 channel 上，而要同时关联多条 channel。
这时它就成了文档里的 **coordinated PNC** 。

------

## 4）“映射多条 channel”以后，会发生什么

这才是这个概念真正重要的地方。

### 情况 A：只映射一条 channel

假设：

- PNC 5 只映射到 Channel A

那么当 PNC 5 被请求时，ComM 只需要处理 Channel A。

这就是**非 coordinated PNC** 的直观理解。

### 情况 B：映射多条 channel

假设：

- PNC 5 映射到 Channel A 和 Channel B

那么当 PNC 5 被请求时，ComM 不能只考虑 A，还要考虑 B。
这时一个 channel 上收到的 PNC 激活请求，可能要被**路由到其他 channel**。文档明确说：

> 对于 coordinated partial networks，ComM 负责 gateway behavior，也就是把 **一个 channel 上的 PNC activation request 路由到其他 channels** 。

这就是“多 channel 映射”真正带来的复杂性。

------

## 5）为什么这会叫 coordinated

因为它已经不是“一个 PNC 管一条线”的简单关系了，而是：

> **一个 PNC 的状态，需要多个 channel 协同一致地处理。**

文档里也说明：

- ERA 只在 **PNC Gateway 打开**
- 且这个 PNC 是 **coordinated**
- 也就是这个 PNC 被分配到 **多于一条 COMM channel** 时才评估 。

所以 coordinated 的本质就是：

**这个 PNC 的行为要跨多个 channel 协调。**

------

## 6）给你一个最直观的例子

假设 ECU 上有两条通道：

- **Channel 1 = CAN动力总线**
- **Channel 2 = CAN车身总线**

再假设：

- **PNC-X** 表示“某个跨域功能组”

### 方案 1：PNC-X 只映射 Channel 1

那就表示：

- 这个功能组只和 Channel 1 有关
- PNC-X 激活时，只需要 Channel 1 参与
- Channel 2 不用管

### 方案 2：PNC-X 同时映射 Channel 1 和 Channel 2

那就表示：

- 这个功能组跨了两条通道
- 只在 Channel 1 上收到请求还不够
- ComM 可能要把请求继续路由到 Channel 2
- 两条 channel 的状态要配合处理

这就是“一个 PNC 映射多条 channel”的直观含义。

------

## 7）为什么文档里还说“不同 channel 上的 PNC vector 可以不同”

因为并不是所有 PNC 都映射到所有 channel。
文档明确提醒：

> Nm message 里的 PNC vector 内容会考虑 **Partial Networks to channels** 的映射关系，
> 所以如果某些 PNC 并没有映射到所有 channel，**不同 channel 上的 PNC vector 内容可以不同** 。

这句话反过来也印证了“映射”是什么意思：

- 哪个 PNC 映射到哪个 channel
- 就决定了这个 channel 上该不该带这个 PNC 的信息

------

## 8）一句话把它彻底讲透

**PNC 是逻辑功能组，channel 是实际通信通道。**

所以：

- **PNC 映射一条 channel**
  = 这个功能组只靠这一条通道处理
- **PNC 映射多条 channel**
  = 这个功能组跨多个通道，需要 ComM 在这些通道之间做协同、路由和状态管理
  = 这就是 **coordinated PNC**

------

## 9）你可以先这样记

**不是“一个 PNC 属于几条线”这么抽象，**
而是：

**“这个 PNC 被激活时，需要哪些 channel 一起参与通信。”**

这就是“PNC 映射到多条 channel”的真正含义。

你要的话，我下一步可以直接给你画一个 **“单 channel PNC”和“多 channel coordinated PNC”** 的对比图。





在 **Partial Network（PN）** 场景下，**NM PDU 的核心处理不是“收到一帧就直接唤醒”**，而是：

**先由 CanNm 解析 NM PDU 里的 PN 信息 → 过滤 → 聚合成 EIRA/ERA → 通过 Com/PduR 通知上层 → ComM 再按 PNC 状态机决定哪些 channel 要被请求、路由或释放。**
其中 **Nm** 主要负责把 **ComM 和 CanNm 串起来**：ComM 往下的网络请求通过 Nm 转给 CanNm，CanNm 的回调再通过 Nm 往上转给 ComM 。

我按 **接收流程（Rx）** 和 **发送流程（Tx）** 给你展开。

------

# 1. 先看 NM PDU 里，PN 相关信息放在哪

CanNm 文档说明：

- NM PDU 的 **Control Bit Vector** 里有一个 **CRI bit**
- 这个 bit 专门用于 **Partial Networking**
- 只有这个 bit 置位，才表示这条 NM 报文里带了 PN 请求信息

同时，CanNm 还说明：

- CRI 的具体内容放在 **NM user data** 里
- 每个 bit 对应一个 cluster / PNC 请求

所以在 PN 场景下，一条 NM PDU 里和 PN 最关键的两部分就是：

- **CBV 里的 CRI bit**：先告诉你“这帧有没有 PN 信息”
- **User Data 里的 CRI 内容**：再告诉你“请求了哪些 PNC”

------

# 2. 接收路径：收到 NM PDU 后怎么处理

## 第一步：CanNm 先判断 PN 过滤功能是否已经启用

CanNm 文档说明，PN 接收过滤不是上电就一直开的：

- PN filter 在初始化后默认是关闭的
- 需要调用 `CanNm_ConfirmPnAvailability()` 后才激活
- 如果 channel 重新启动，而没有再次调用这个接口，filter 会再次失效

所以第一层前提是：

**不是所有收到的 NM PDU 都会马上走 PN 过滤算法。**

------

## 第二步：先检查 CRI bit

过滤算法的第一步非常直接：

- 如果 **CRI bit = 0**，这条 NM message **对 PN 来说不 relevant**
- 如果 **CRI bit = 1**，CanNm 才继续解析 CRI 内容

也就是说：

**没有 CRI bit，就没有 PN 处理。**

------

## 第三步：用 PN filter mask 判断这条 PN 请求和本 ECU 有没有关系

CanNm 会把收到的 CRI 内容和本 ECU 的 **PN filter mask** 做比较：

- CRI 里的每一位表示一个 cluster
- 不是每个 cluster 都和当前 ECU 相关
- 因此会用可配置的 **PN filter mask** 把无关 cluster 屏蔽掉
- 只有当收到的 PN 信息与 filter mask **至少有一个 bit 匹配** 时，这条 NM 报文才被认为 **relevant for the ECU**

### 结果分两种

如果 **不 relevant**：

- 若配置了 `All Nm Messages Keep Awake = true`，仍然会执行标准 NM 接收处理
- 否则这条报文直接忽略

如果 **relevant**：

- CanNm 先执行标准 NM 接收处理
- 然后把过滤后的 PN 内容继续送进 PN 算法

所以这里要注意：

**PN 处理是在“标准 NM 接收”之外叠加的一层，而不是完全替代标准 NM 行为。**

------

## 第四步：CanNm 对 PN 请求做 EIRA / ERA 聚合

收到的 PN 请求并不会直接一帧一帧交给 ComM，而是先由 CanNm 聚合成两个更稳定的状态：

### EIRA

CanNm 把：

- 外部收到的 PN 请求
- 本 ECU 自己发出的 PN 请求

在 **所有网络 / 所有 channel 维度上合并**，得到 **EIRA**。
这个状态表示：**整个 ECU 当前哪些 partial networks 是 active**

### ERA

CanNm 把：

- 只来自外部的 PN 请求

按 **每个网络分别聚合**，得到 **ERA**。
这个状态表示：**在某个具体网络上，哪些 PNC 是被其他 ECU 请求的，因此必须保持 active**

### 还有超时保持机制

CanNm 不是“看到 bit=1 就永远记住”，而是：

- 某个 cluster 第一次被请求时，存储请求并启动 timer
- 重复请求到来时，timer 重新开始
- timer 到期后，这个请求被删除

所以聚合的本质是：

**把零散的 NM PDU 请求，整理成带老化超时的 PNC 活跃状态。**

------

## 第五步：EIRA / ERA 变化后，CanNm 通过 Com/PduR 往上报告

文档写得很清楚：

- 只要 EIRA 或 ERA 发生变化
- CanNm 就会更新 Com 中对应的 **EIRA / ERA I-PDU**
- 并调用 `PduR_CanNmRxIndication()`

这一步很关键，因为很多人会误以为：

> CanNm 会直接告诉 ComM “某个 PNC = 1”

其实不是。真正链路是：

**CanNm 更新 EIRA/ERA I-PDU → Com/PduR 分发信号变化 → ComM 的信号回调被触发**

------

## 第六步：ComM 通过 `ComM_ComCbk_<SignalName>` 接收 EIRA/ERA 变化

ComM 文档明确说：

`ComM_ComCbk_<SignalName>()` 的作用是：

- 当用于传输 partial network channel request information 的 **ComSignal** 发生变化时通知 ComM
- 这个 `SignalName` 就是对应的 **EIRA_RX 或 ERA ComSignal**

所以从模块交互看：

**NM PDU 的 PN 信息，不是通过 Nm 回调进入 ComM 的，而是通过 Com 信号变化进入 ComM 的。**

------

## 第七步：ComM 根据 EIRA/ERA 推进 per-PNC 状态机

ComM 的 PNC 状态机是 **per partial network** 的，也就是每个 PNC 各有一套状态机；ComM 使用 **EIRA 和 ERA 两类 bit vector** 和其他 ECU 交换 PNC 状态，ERA 只在 PNC Gateway 打开且 PNC 是 coordinated 时评估 。

状态机里关键触发条件是：

- 当 `ComM_COMCbk()` 检测到 **PNC bit within EIRA = 1**
- 或在 gateway 条件下 **PNC bit within ERA = 1**
- ComM 会触发 `Com_SendSignal()` 和 `Channel Full Communication Request()`，把相关 channel 拉起来

而当 EIRA bit 变回 0 时，ComM 会启动 `ComMPncPrepareSleepTimer`，准备让 PNC 走向休眠

所以接收路径的最终效果是：

**NM PDU → CanNm 识别出相关 PNC → ComM 把这些 PNC 对应的 channel 请求起来。**

------

# 3. Nm 在接收路径里扮演什么角色

在 **PN 信息内容处理** 上，Nm 不是主角。
Nm 的主职责是：

- **ComM 的网络请求往下转给 CanNm**
- **CanNm 的 NM 状态回调往上转给 ComM**

所以在 **NM PDU 的 PN 内容解析** 这件事上，链路主要是：

**CanNm → Com/PduR → ComM**

而在 **channel 进入 FullCom / RepeatMessage / PrepareBusSleep** 这种总线模式变化上，链路还是：

**CanNm → Nm → ComM**

这两条链同时存在，但处理的东西不一样。

------

# 4. 发送路径：本 ECU 要把 PN 请求放进 NM PDU 时怎么处理

接收路径讲完，再看发送路径。

## 第一步：CanNm 在 NM PDU 中带上 CRI bit

文档说明：

- 只要某个 channel 开启了 Partial Networking feature
- CanNm 就会把 **CRI bit（CBV bit6）置 1**

所以从发送角度看：

**一旦该 channel 支持 PN，CanNm 发出的 NM PDU 就会明确告诉别人：这帧带了 PN 信息。**

------

## 第二步：当本 ECU 内部新请求了某个 PN，NM User Data 里的对应 bit 会被置位

CanNm 文档说得很直接：

- 当 **a new PN is internally requested**
- 对应 bit 会写进 **NM message user data**
- 并且这个更新必须尽快在总线上可见

这一步通常不是应用直接改 CanNm 内部结构，而是 **ComM/Com/用户数据配置** 联动起来完成。

------

## 第三步：为了让新的 PN 请求尽快上总线，有两种机制

### 机制 A：Com 信号变化立即发送

如果 NM user data 通过 Com 设置，并配置成 **Transmission on Change**：

- 信号变化会触发一次额外的 NM 报文发送
- 这样新的 PN bit 很快就能被发出去
- 这个机制要求 `Pn Handle Multiple Network Requests = OFF`

### 机制 B：再次请求网络，进入 Repeat Message

如果 CanNm 已经在 Network Mode，而上层再次调用 `CanNm_NetworkRequest()`：

- CanNm 会回到 **Repeat Message**
- 立即发送 NM message
- 然后以更快的周期再发送几帧
- 这个机制要求 PN feature 启用，并且 `Immediate Nm Transmission` 启用，同时 `Pn Handle Multiple Network Requests = ON`

------

# 5. 发送路径里 ComM / Nm / CanNm 怎么配合

这一段最容易跟“PN 内容处理”混到一起。

## 本地请求的控制链

当某个 PNC 被本地用户请求后，ComM 会根据自己的 PNC 状态机：

- 触发 `Com_SendSignal()`
- 并发起 `Channel Full Communication Request()`

然后在网络控制层面：

- **ComM 的请求通过 Nm 往下**
- **Nm 再调用 CanNm**
- CanNm 通过 `CanNm_NetworkRequest` / Repeat Message / Immediate Tx，让更新后的 NM PDU 尽快发上总线

所以发送路径要分成两层理解：

### 内容层

**哪些 PN bit 要写进 NM PDU**
这主要由 **ComM + Com 用户数据 + CanNm PN 功能**决定。

### 总线发送层

**这些更新后的 NM PDU 何时立刻发出去**
这主要由 **CanNm 的 Repeat Message / Immediate Tx 机制**决定。

------

# 6. 如果是 coordinated PNC，还会有 Gateway 路由

如果某个 PNC 映射了多条 channel，ComM 还要做 **PNC Gateway**。
文档明确写到：

- ComM 负责 coordinated partial networks 的 gateway behavior
- 也就是把一个 channel 上收到的 PNC activation request 路由到其他 channel
- 这个路由通过发送 **EIRA TX signals** 完成

这意味着在 gateway 场景里，NM PDU 的处理流程会变成：

**某条 channel 收到 NM PDU → CanNm 算出 ERA/EIRA → ComM 判断这是 coordinated PNC → ComM 通过 EIRA TX 把请求传播到其他 channel → 对应 channel 再通过 Nm/CanNm 把新的 PN 信息发到各自总线。**

------

# 7. 用一条完整时序把流程串起来

## 接收方向

1. **CanIf/CanNm 收到 NM PDU**
2. CanNm 检查 **CRI bit**
3. 若 PN filter 已启用，则用 **PN filter mask** 过滤相关 cluster
4. relevant 的 PN 信息进入 EIRA/ERA 聚合，并启动/刷新定时器
5. EIRA/ERA 变化后，CanNm 更新 Com 中的 EIRA/ERA I-PDU，并调用 `PduR_CanNmRxIndication()`
6. Com 信号变化触发 `ComM_ComCbk_<SignalName>()`，把 EIRA_RX / ERA 变化通知给 ComM
7. ComM 的 per-PNC 状态机根据 EIRA/ERA 位变化，执行 `Channel Full Communication Request()`、`Com_SendSignal()` 或准备休眠

## 发送方向

1. 本 ECU 某个 PNC 被本地请求
2. ComM 推进 PNC 状态机，并触发 `Com_SendSignal()` / Channel 请求
3. 网络请求通过 **ComM → Nm → CanNm** 下发
4. CanNm 在 NM PDU 中携带 **CRI bit + PN user data**
5. 若需要快速上总线，则通过：
   - Com 的 on-change 发送，或
   - `CanNm_NetworkRequest` 触发 Repeat Message + Immediate Tx

------

# 8. 最后压缩成一句话

在 Partial Network 场景下，**NM PDU 的主要处理流程**就是：

**CanNm 负责从 NM PDU 中识别并过滤 PN 请求，聚合成 EIRA/ERA；ComM 通过 Com 回调拿到这些聚合结果，再按 per-PNC 状态机决定唤醒、路由或释放哪些 channel；Nm 则负责把 ComM 的网络控制请求转给 CanNm，并把总线 NM 状态回送给 ComM。**

你要的话，我下一步可以把这个流程直接画成一张
**“NM PDU Rx 路径图”** 和 **“NM PDU Tx 路径图”**。







如果**没有开启 PN（Partial Networking）功能**，那 **NM PDU 就不再走 PN 那套“CRI 解析 → PN 过滤 → EIRA/ERA 聚合 → ComM 按 PNC 状态机处理”** 的链路，而是回到 **标准的 CanNm 状态机处理流程**。CanNm 文档也明确说了：PN 过滤算法只有在 PN 相关机制激活后才使用，**否则就按“usual way of receiving messages”处理** 。

你可以把它理解成一句话：

**没开 PN 时，NM PDU 只用于标准网络管理：维持/唤醒整个 channel 的 Network Mode、刷新 NM 超时、驱动 Repeat Message / Normal Operation / Ready Sleep / Prepare Bus-Sleep / Bus-Sleep 这些状态切换。**

------

## 1. 先说结论：没有 PN 时，哪些东西“不发生”

没有开启 PN 时：

- **不会做 CRI 内容解析**
- **不会做 PN filter mask 匹配**
- **不会生成 EIRA / ERA**
- **不会通过额外的 EIRA/ERA I-PDU 去通知 ComM**
- **不会走 per-PNC 的局部网络唤醒/释放逻辑**

因为这些都属于 3.18 Partial Networking 的附加算法，而标准 CanNm 本身的主流程是独立存在的 。

另外，Control Bit Vector 里的各个 bit 都是“按功能可选”的；**如果某功能没启用，对应 bit 值就是 0**。也就是说，PN 没用时，PN 对应的 **CRI bit 不参与正常处理** 。
唯一例外是 Vector 提到一个配置：即使 PN 关闭，也可以强制发送 CRI bit（`Cri Bit Always Enabled`），但这只是“bit 被发出来”，**不代表 PN 算法真的启用** 。

------

## 2. 标准 NM PDU 的接收流程

### 第一步：CanIf 收到 NM 报文，回调给 CanNm

CanNm 的标准收发入口很清楚：

- 发送通过 `CanIf_Transmit`
- 接收由 CanIf 调 `CanNm_RxIndication()`
- 发送成功后由 CanIf 调 `CanNm_TxConfirmation()`

所以接收链起点就是：

**CanIf → CanNm_RxIndication**

------

### 第二步：CanNm 按标准 NM PDU 布局解析报文

默认 NM PDU 布局是：

- Byte0：Source Node Identifier
- Byte1：Control Bit Vector
- 后面：User Data

也就是说，CanNm 收到一帧后，首先还是会按标准 NM 报文结构去识别：

- 谁发的（Node ID）
- 控制位状态（CBV）
- 用户数据

但这时候**不会进入 PN 专用算法**。

------

### 第三步：通知 Nm Interface 收到了一帧 NM PDU

CanNm 文档明确写到，收到 NM message 后，会通过可选接口通知 Nm：

- `Nm_PduRxIndication()`
- 如果同一 channel 上有多个 BusNm，也可能是 BusNm-specific 的接收通知

所以标准接收路径里，有一条很直接的通知链：

**CanNm_RxIndication → Nm_PduRxIndication**

而 Nm 本身是 **ComM 和各 BusNm 之间的总线无关适配层**，负责把调用在 ComM 和 CanNm 之间转发 。

------

### 第四步：根据当前状态，刷新超时或触发状态迁移

没有 PN 时，NM PDU 的核心作用就是**驱动标准 NM 状态机**。

CanNm 的标准状态机是：

- **Bus-Sleep Mode**
- **Prepare Bus-Sleep Mode**
- **Network Mode**
  - Repeat Message
  - Normal Operation
  - Ready Sleep

#### 场景 A：当前在 Bus-Sleep，收到 NM PDU

如果在 **Bus-Sleep Mode** 收到 NM 报文：

- CanNm 会调用 `Nm_NetworkStartIndication()`
- 这表示网络上已经有节点重新开始通信了
- Nm 再把这个通知转给 ComM

也就是说：

**Bus Sleep 下收到 NM PDU = 网络被别人唤醒了**

------

#### 场景 B：当前在 Prepare Bus-Sleep，收到 NM PDU

如果当前在 **Prepare Bus-Sleep Mode**，成功收到 NM 报文：

- 会离开 Prepare Bus-Sleep
- 回到 Network Mode

而 Repeat Message 状态的进入条件里，也明确包括：

- 在 Prepare Bus-Sleep 收到 NM message

所以可以理解为：

**PBS 收到 NM PDU → 重新拉回网络活动状态，一般进入 Repeat Message。**

------

#### 场景 C：当前已经在 Network Mode

在 **Network Mode** 里，NM PDU 的作用主要是：

- 刷新 NM Timeout Timer
- 维持网络活跃状态
- 在特定功能启用时，触发额外行为（比如 Node Detection）

状态图里明确写到：

- **NM message received or transmitted / Restart NM Timeout Timer**

这说明在 Normal Operation / Ready Sleep 中，收到一帧 NM 报文最核心的动作就是：

**“说明总线上还有节点活着”，所以重启超时计时器，不让本节点过早睡眠。**

------

### 第五步：如果启用了 Node Detection，Repeat Message Bit 也会起作用

这不属于 PN，而是标准 CanNm 的另一个功能。

如果启用了 **Node Detection**：

- 某节点请求节点检测时，会在 NM PDU 里设置 **Repeat Message Bit**
- 其他节点收到这个 bit 后，会转入 **Repeat Message State**
- 请求方就可以收集当前活跃节点的 Source Node Identifier

所以没开 PN 时，CBV 里**仍然可能有别的 bit 生效**，比如：

- Repeat Message Request Bit
- Active Wake-up Bit
- Sleep Ready Bit

只是**不会有 PN 的那条算法链**。

------

## 3. 标准 NM PDU 的发送流程

没有 PN 时，发送流程也很标准。

### 第一步：上层请求网络

CanNm 的基本机制是：
**只有在节点需要通信时才发送 NM message；不需要通信时就停止发送，最后整个网络一起进入 bus-sleep** 。

如果本 ECU 需要通信：

- 上层通过 `CanNm_NetworkRequest()` 请求网络
- 这个接口由 Nm Interface 调用，Nm 再把请求转给具体 BusNm

在 ComM 侧，标准流程是：

- ComM 请求 Full Communication
- BusSM 先切到 Full Communication
- 然后 ComM 请求 BusNM 主动或被动启动

------

### 第二步：从 Bus-Sleep / Prepare Bus-Sleep 进入 Repeat Message

`CanNm_NetworkRequest()` 或 `CanNm_PassiveStartUp()` 会把状态拉到 **Repeat Message**：

- `CanNm_PassiveStartUp()`：从 Bus Sleep / Prepare Bus Sleep 启动到 Network Mode（Repeat Message）
- `CanNm_NetworkRequest()`：请求网络，表示 ECU 需要在总线上通信
- Bus Sleep 下请求网络会进入 Repeat Message

------

### 第三步：CanNm 周期发送标准 NM PDU

在发送上，CanNm 通过 `CanIf_Transmit()` 发 NM message 。

标准发送节奏是：

- 进入 Repeat Message 后
- 经过 `Msg Cycle Offset`
- 发送第一帧
- 之后按 `Msg Cycle Time` 周期发送

并且文档明确说：

- **只要状态在 Repeat Message 或 Normal Operation，NM message 就会持续发送**

这就是没有 PN 时最经典的 NM 报文发送模型。

------

### 第四步：网络释放后，进入 Ready Sleep / Prepare Bus-Sleep / Bus-Sleep

当应用不再需要通信时：

- 调 `CanNm_NetworkRelease()` 释放网络
- 在 Network Mode 中，如果本地不再请求，但别的节点还在发 NM message，会停留在 **Ready Sleep**
- 一段时间后若再也收不到 NM message，就退出 Network Mode，进入 **Prepare Bus-Sleep**
- Prepare Bus-Sleep 结束后进入 **Bus-Sleep**，并通过 `Nm_BusSleepMode()` 通知上层

另外，Com 只在 **Network Mode** 里活跃；进入 Prepare Bus-Sleep 时，应用报文发送就停止了 。

------

## 4. 所以“没开 PN”的 NM PDU，到底在做什么？

可以压缩成下面这条标准链路：

**Rx：**
CanIf 收到 NM PDU
→ `CanNm_RxIndication()`
→ 标准解析 Node ID / CBV / User Data
→ `Nm_PduRxIndication()` 通知 Nm
→ 根据当前状态刷新 NM Timeout 或触发 BusSleep / PrepareBusSleep / NetworkMode 的状态迁移
→ Nm 再把模式变化通知给 ComM

**Tx：**
ComM / 上层请求通信
→ Nm 转给 CanNm
→ `CanNm_NetworkRequest()` / `CanNm_PassiveStartUp()`
→ 进入 Repeat Message
→ CanNm 周期发送标准 NM PDU
→ 网络释放后逐步进入 Ready Sleep / Prepare Bus-Sleep / Bus-Sleep

------

## 5. 和开启 PN 时的本质区别

最本质的区别只有一句：

**不开 PN 时，NM PDU 只管理“整个 channel 是否保持唤醒”；开了 PN 时，NM PDU 还会额外管理“哪些 PNC/局部网络需要保持激活”。**

所以你前一个问题里那条链：

**CRI → PN filter → EIRA/ERA → ComM per-PNC 状态机**

在 **PN disabled** 的情况下，整体都没有；只剩下：

**标准 CanNm 状态机 + Nm/ComM 的 channel 级网络管理。**

你要的话，我可以下一条直接给你画一个
**“不开 PN 时 NM PDU 的标准 Rx/Tx 时序图”**，会比文字更直观。