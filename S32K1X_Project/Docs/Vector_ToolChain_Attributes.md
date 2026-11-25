# CANdb++ / Vector Tool Chain Attributes 说明

本文说明 CANdb++ 中 **Vector Tool Chain Attributes** 的典型属性，并按“先通俗、后深入”的方式解释：

- 属性是什么意思
- 用在哪里
- 原理是什么
- 多个枚举值分别代表什么
- 哪些属性会互相影响或组合影响报文/信号发送
- 每个属性都有一个通俗例子

> 本文只描述属性，不修改 DBC 文件。  
> 不同 CANdb++/CANoe 版本、IL 类型、OEM 属性包可能有不同枚举值；实际以 DBC 中的 `BA_DEF_` 定义为准。

---

## 1. 阅读基础

### 1.1 属性在 DBC 中如何保存

Vector 工具链属性本质上是 DBC 的用户自定义属性，通过三类语句保存：

```text
BA_DEF_     定义属性名、对象类型、值类型和枚举/范围
BA_DEF_DEF_ 定义默认值
BA_         给具体网络、节点、报文或信号赋值
```

例如：

```text
BA_DEF_ BO_ "GenMsgCycleTime" INT 0 65535;
BA_DEF_DEF_ "GenMsgCycleTime" 0;
BA_ "GenMsgCycleTime" BO_ 291 100;
```

表示：为报文定义整数属性 `GenMsgCycleTime`，默认值为 0，并把 ID 291 的报文周期设为 100 ms。

**注意：** CANdb++ 对 DBC 段顺序较敏感，通常应保证 `BA_DEF_` / `BA_DEF_DEF_` / `BA_` 位于 `VAL_` 之前。

### 1.2 对象类型

| 对象类型 | 含义 |
|---|---|
| 网络（Network） | 整个 CAN 网段/数据库 |
| 节点（Node） | ECU 或仿真节点 |
| 报文（Message） | 一条 CAN 报文 |
| 信号（Signal） | 报文中的一个信号 |
| 节点映射 Rx 信号 | 某节点接收并映射的信号 |
| `<_ECU>` 后缀 | 按接收 ECU 区分属性时使用 |

### 1.3 属性分层

| 层次 | 作用 |
|---|---|
| 网络级 | 波特率、网络名、TP 基地址、NM 基地址、协议类型 |
| 节点级 | 是否使用 IL、是否参与 NM、TP 地址、节点定时器 |
| 报文级 | 发送类型、周期、延迟、重复次数、诊断/TP 标记 |
| 信号级 | 信号发送类型、初值、非激活值、信号超时 |

---

## 2. 一般/网络属性（General）

| 属性 | 对象 / 值 | 含义 | 用途 | 原理 | 通俗例子 |
|---|---|---|---|---|---|
| `Manufacturer` | 网络 / 字符串 | 网络所属 OEM/车厂 | 数据库管理、导出、OEM 规范区分 | 元数据，工具可按 OEM 选择规则或筛选数据库 | 写 `"OEM_A"`，表示该网络属于 OEM_A |
| `Baudrate` | 网络 / 整数，bit/s | CAN 网络波特率 | CANoe/CANalyzer 配置通道时序 | 所有节点必须使用一致波特率，否则出现位错误/无法通信 | `500000` 表示 500 kbit/s；`250000` 表示 250 kbit/s |
| `DBName` | 网络 / 字符串 | 数据库/网络名称 | 保存独立于文件名的网络名，允许在 CANdb++ 中改名 | 存在 `DBName` 时，CANdb++/CANoe 使用它作为网络/数据库名；不存在时常用文件名 | `DBName = "CAN1"`，即使文件叫 `Comfort_CAN.dbc`，Network 名称也显示 `CAN1` |
| `VersionYear` | 网络 / 整数 0...99 | 发布年份，BCD 编码 | 版本管理 | 工具按 BCD 保存/显示年份，不参与通信 | `0x24` 表示 2024 年（BCD） |
| `VersionMonth` | 网络 / 整数 0...12 | 发布月份，BCD 编码 | 版本管理 | 与年份一起组成版本日期 | `0x12` 表示 12 月（BCD） |
| `VersionWeek` | 网络 / 整数 0...52 | 发布周数，BCD 编码 | 按周追踪版本 | 版本元数据，不改变总线行为 | `0x15` 表示第 15 周（BCD） |
| `VersionDay` | 网络 / 整数 0...31 | 发布日，BCD 编码 | 版本管理 | 与年/月/周组成发布日期 | `0x09` 表示 9 日（BCD） |
| `VersionNumber` | 网络 / 整数 >= 0 | 版本号，BCD 编码 | 区分同一网络的不同版本 | 版本元数据，可用于比较数据库新旧 | `0x03` 表示第 3 版（BCD） |

> `Baudrate` 与 `BaudRate` 是不同属性名，DBC 属性名区分大小写，实际以项目定义的拼写为准。

---

## 3. 交互层属性（Interaction Layer）

### 3.1 什么是交互层，启用和不启用有什么区别

**交互层（Interaction Layer，IL）** 是应用逻辑和 CAN 报文收发之间的一层自动化软件。

通俗地说：

> 启用 IL，就像请了一个“自动发货助手”：你只告诉它“灯光状态变了”，它按 DBC 规则自动打包、定时发送、重复发送、判断超时。  
> 不启用 IL，就是你自己当调度员：定时器、信号变化判断、发送、重发、超时全部自己写。

| 工作 | 不启用 IL | 启用 IL |
|---|---|---|
| 何时发送 | 自己写定时器/事件 | 按 `GenMsgSendType`、`GenSigSendType` 自动决定 |
| 周期发送 | 自己维护定时器 | `GenMsgCycleTime` 自动调度 |
| 信号变化触发 | 自己比较新旧值 | `OnChange` 等信号发送类型自动判断 |
| 快速/激活发送 | 自己判断激活状态 | `GenSigInactiveValue` + 快速周期自动切换 |
| 初值/非激活值 | 自己初始化 | `GenSigStartValue`、`GenSigInactiveValue` |
| 接收超时 | 自己启动/刷新定时器 | `GenSigTimeoutTime`、`GenSigTimeoutMsg` |
| 代码量 | 多 | 少，大部分行为由 DBC 属性描述 |

**两个开关：**

```text
ILUsed         = 节点总开关
GenMsgILSupport = 单条报文开关
```

| `ILUsed`（节点） | `GenMsgILSupport`（报文） | 结果 |
|---|---|---|
| No | No | 完全手动处理 |
| Yes | No | 节点启用 IL，但这条报文不交给 IL |
| No | Yes | 报文标记了 IL，但节点未启用，通常不会自动处理 |
| Yes | Yes | 这条报文由 IL 自动处理 |

### 3.2 节点/报文启用属性

| 属性 | 对象 / 值 | 含义 | 用途 | 原理 | 通俗例子 |
|---|---|---|---|---|---|
| `ILUsed` | 节点 / 枚举 No、Yes | 节点是否使用交互层 | 决定是否为该节点启用 IL | `No` 不纳入 IL；`Yes` 纳入 IL 生成范围 | 仿真 BCM 节点时设 `Yes`，让 IL 自动处理灯光报文 |
| `GenMsgILSupport` | 报文 / 枚举 No、Yes | 报文是否交给 IL | 标记哪些报文参与 IL 发送/接收 | `No` 时即使节点启用 IL，该报文也不自动处理 | `BCM1_1` 设为 `Yes`；一条裸测试报文设为 `No` |
| `IlTxTimeout` | 报文 / 整数 ms，默认 0 | 交互层发送超时时间 | 发现发送失败、无确认或总线异常 | 发送请求后启动定时器；超时未完成/未确认则触发超时处理；0 通常表示关闭 | `IlTxTimeout = 100`：100 ms 内没有发送确认就报发送超时 |

### 3.3 报文级发送类型：`GenMsgSendType`

`GenMsgSendType` 决定“这条报文整体怎么发”。常见枚举值如下，实际以 `BA_DEF_` 为准。

#### `Cyclic`

- **通俗**：像闹钟，每隔固定时间自动发一次。
- **深入**：IL 使用 `GenMsgCycleTime` 的周期定时器发送。
- **例子**：`GenMsgSendType=Cyclic`、`GenMsgCycleTime=100`，则 `BCM1_2` 每 100 ms 发送一次。

#### `Spontaneous`

- **通俗**：没有固定闹钟，只有事件发生才发。
- **深入**：由信号变化、应用写入或显式发送请求触发，可配合 `GenMsgDelayTime`、`GenMsgNrOfRepetition`。
- **例子**：按下灯光开关后发送一帧 `BCM1_1`，平时不发。

#### `CyclicIfActive`

- **通俗**：有信号激活时才按周期发，没激活就不发或停发。
- **深入**：IL 根据信号发送类型和 `GenSigInactiveValue` 判断激活状态；激活时按周期发送，非激活时按 IL 规则停止或降速。
- **例子**：`LightCmd != 0` 时 `BCM1_1` 每 100 ms 发；`LightCmd == 0` 时停止周期发送。

#### `CyclicAndSpontaneous`

- **通俗**：既有周期闹钟，也能在事件发生时临时插播。
- **深入**：周期部分按 `GenMsgCycleTime`，事件部分由信号/应用触发；事件发送可受最小间隔和重复次数限制。
- **例子**：`BCM1_2` 每 500 ms 发一次状态；灯光状态突变时立即额外发一帧。

#### `CyclicIfActiveAndSpontaneous`

- **通俗**：激活时定时发，同时事件发生时也能立即发。
- **深入**：结合“激活判断”和“事件触发”：激活状态决定是否维持周期发送，事件决定是否额外发送。
- **例子**：灯光开启时每 100 ms 发状态，关闭瞬间立即补发一帧“Off”。

#### `NoMsgSendType` / OEM 扩展值

- **通俗**：不指定标准 IL 发送方式，或使用 OEM 自定义方式。
- **深入**：IL 不按标准周期/事件模式自动发送，通常需要 CAPL/应用手动发送；OEM 扩展值由该 OEM 的 IL 实现定义。
- **例子**：特殊测试报文设为 `NoMsgSendType`，完全由 CAPL 的 `output()` 控制。

### 3.4 信号级发送类型：`GenSigSendType`

`GenSigSendType` 决定“这个信号怎样触发所在报文”。常见枚举值如下。

#### `Cyclic`

- **通俗**：这个信号按自己的周期发送。
- **深入**：信号级 IL 使用 `GenSigCycleTime`（或报文周期）作为节拍；报文实际周期由报文内多个信号和报文级发送类型共同决定。
- **例子**：`LightState` 设置 `Cyclic`、`GenSigCycleTime=100`，每 100 ms 发送一次状态。

#### `OnChange`

- **通俗**：值变了才发，没变不发。
- **深入**：IL 保存旧值并与新值比较，不同则触发发送；可配合重复次数和最小间隔。
- **例子**：`LightCmd` 从 0 变 1 时发送；一直为 1 时不重复发送。

#### `OnWrite`

- **通俗**：应用只要“写”这个信号就发，即使值没有变化。
- **深入**：触发条件是写入动作，不比较新旧值；适合需要刷新确认的场景。
- **例子**：应用再次写入 `LightCmd=1`，即使原来是 1，也发送一帧。

#### `IfActive`

- **通俗**：信号处于激活状态时才发；等于非激活值就不发或降速。
- **深入**：用 `GenSigInactiveValue` 判断激活；不等于该值时激活，可能使用快速周期；等于该值时回到非激活处理。
- **例子**：`GenSigInactiveValue=0`，`LightCmd=1` 时发送，`LightCmd=0` 时停止/降速。

#### `NoSigSendType`

- **通俗**：不单独指定信号发送方式，由报文级类型或手动逻辑决定。
- **深入**：IL 不把该信号作为独立触发源；报文发送主要由 `GenMsgSendType`、其他信号或应用控制。
- **例子**：一个只随报文周期发送的校验信号设为 `NoSigSendType`。

#### `FastIfActive` / OEM 扩展值

- **通俗**：激活时用更快节奏发送。
- **深入**：属于 `IfActive` 的快速发送变体，常结合 `GenMsgCycleTimeFast`、`GenSigCycleTimeActive`；具体优先级由 IL/OEM 定义。
- **例子**：灯光激活后前 1 秒每 20 ms 发送，之后恢复 100 ms。

### 3.5 周期、延迟、重复与启动属性

| 属性 | 对象 / 值 | 含义 | 用途 | 原理 | 通俗例子 |
|---|---|---|---|---|---|
| `GenMsgCycleTime` | 报文 / 整数 ms | 正常周期 | 周期报文节拍 | IL 定时器按该值触发发送；0 通常表示不用固定周期 | `100`：每 100 ms 发送一次 |
| `GenMsgCycleTimeFast` | 报文 / 整数 ms | 快速周期 | 激活/快速阶段加快发送 | IL 在激活状态使用该周期；通常小于正常周期 | 正常 100 ms，快速 20 ms |
| `GenMsgCycleTimeActive` | 报文 / 整数 ms | CAPL IL 的激活周期 | CAPL 生成器 IL 的激活阶段发送 | 与 `GenMsgCycleTimeFast` 类似，但面向 CAPL IL | CAPL IL 中激活时每 50 ms 发送 |
| `GenSigCycleTime` | 信号 / 整数 ms | 信号级周期 | 信号级 IL 调度 | IL 以信号为单位按该周期发送；报文实际周期由多个信号组合决定 | `LightState` 每 100 ms 发送 |
| `GenSigCycleTimeActive` | 信号 / 整数 ms | 信号激活周期 | 信号级激活/快速发送 | 信号激活时使用该周期，配合 `GenSigInactiveValue` | `LightCmd` 激活时每 20 ms 发送 |
| `GenMsgDelayTime` | 报文 / 整数 ms | 两次发送最小间隔 | 防止事件密集导致总线过载 | IL 发送前检查距上次发送的时间，未达到间隔则等待/抑制 | 事件再多，也至少间隔 20 ms 才发下一帧 |
| `GenMsgStartDelayTime` | 报文 / 整数 ms | 启动后首次发送延迟 | 避免上电时所有报文同时发送 | IL 启动后先等待该时间，再开始该报文发送 | 上电后等 500 ms 再发第一帧 |
| `GenMsgFastOnStart` | 报文 / 整数 ms | 启动后快速发送持续时间 | 启动阶段快速发布状态 | 在窗口内使用快速周期，之后恢复普通周期；通常要求普通/快速周期都大于 0 | 启动后前 1 秒每 20 ms 发，之后每 100 ms 发 |
| `GenMsgNrOfRepetition` | 报文 / 整数 | 事件后重复次数 | 事件/变化报文连续发送 | 由信号发送类型决定是否需要重复，此属性给出重复次数 | 灯光变化后连续发送 3 帧，确保对方收到 |

### 3.6 信号初值与激活值

| 属性 | 对象 / 值 | 含义 | 用途 | 原理 | 通俗例子 |
|---|---|---|---|---|---|
| `GenSigStartValue` | 信号 / 整数（raw） | 信号初始值 | 应用第一次写值前使用 | IL 用该原始值初始化信号；应用写入后覆盖 | `LightState` 初值 `0`，上电默认 Off |
| `GenSigInactiveValue` | 信号 / 整数（raw） | 非激活值 | IfActive 判断阈值 | 信号值等于该值时视为非激活；不等于时视为激活 | `LightCmd` 非激活值 `0`，只有 1 才认为开灯 |

### 3.7 接收超时属性

| 属性 | 对象 / 值 | 含义 | 用途 | 原理 | 通俗例子 |
|---|---|---|---|---|---|
| `GenSigTimeoutTime` | 节点映射 Rx 信号 / 整数 ms | 接收超时时间 | 判断对方是否还在发送 | 接收方启动超时定时器；收到报文刷新，超时未收到则触发超时处理 | 500 ms 内没收到 `BCM1_2`，判定灯光状态超时 |
| `GenSigTimeoutMsg` | 节点映射 Rx 信号 / 十六进制 ID | 用于刷新超时的监视报文 ID | 用另一条心跳/状态报文证明信号仍有效 | 收到该 ID 报文时不触发超时；超时时间由 `GenSigTimeoutTime` 定义 | 收到 `0x456` 就认为 `LightState` 仍有效 |

### 3.8 IL 属性之间的组合与相互影响

#### 1. 报文级和信号级发送类型要组合看

```text
最终发送行为 = GenMsgSendType + GenSigSendType + 周期/延迟/重复参数
```

| 组合                                                            | 通俗理解         | 结果                       |
| ------------------------------------------------------------- | ------------ | ------------------------ |
| `GenMsgSendType=Cyclic` + 信号 `NoSigSendType`                  | 报文自己按闹钟发     | 按 `GenMsgCycleTime` 周期发送 |
| `GenMsgSendType=Spontaneous` + 信号 `OnChange`                  | 信号变化才发       | 变化触发一帧，可按重复次数重复          |
| `GenMsgSendType=CyclicIfActive` + 信号 `IfActive`               | 有激活信号才按周期发   | 激活时周期发送，非激活停止/降速         |
| `GenMsgSendType=CyclicAndSpontaneous` + 信号 `OnChange`         | 周期发 + 变化补发   | 周期继续，变化时额外发送             |
| `GenMsgSendType=CyclicIfActiveAndSpontaneous` + 信号 `IfActive` | 激活周期发 + 事件补发 | 激活时周期发，事件发生时额外发送         |

#### 2. 正常周期、快速周期、非激活值的组合

典型 IfActive 逻辑：

```text
如果 GenSigSendType = IfActive：
    信号值 != GenSigInactiveValue  → 激活，使用快速周期
    信号值 == GenSigInactiveValue  → 非激活，停止快速发送/恢复普通发送
```

例子：

```text
GenMsgCycleTime = 100
GenMsgCycleTimeFast = 20
GenSigInactiveValue = 0
```

- `LightCmd=1`：每 20 ms 快速发送
- `LightCmd=0`：恢复 100 ms 或停止发送（取决于报文发送类型）

#### 3. 事件、最小间隔和重复次数的组合

```text
事件触发
  → 检查 GenMsgDelayTime（最小间隔）
  → 按 GenMsgNrOfRepetition 重复发送
  → 每次发送之间仍受最小间隔限制
```

例子：`OnChange` 触发，`GenMsgDelayTime=20`、`GenMsgNrOfRepetition=3`，则变化后连续发送 3 帧，帧间至少间隔 20 ms。

#### 4. 启动延迟和启动快速发送的组合

```text
系统启动
  → 等待 GenMsgStartDelayTime
  → GenMsgFastOnStart 窗口内使用快速周期
  → 窗口结束后恢复普通周期
```

例子：`GenMsgStartDelayTime=500`、`GenMsgFastOnStart=1000`、普通周期 100 ms、快速周期 20 ms：上电 500 ms 后开始发送，前 1 秒用 20 ms，之后用 100 ms。

#### 5. 接收超时和监视报文的组合

`GenSigTimeoutTime` 定义“多久没收到算超时”，`GenSigTimeoutMsg` 定义“收到哪条报文可以刷新超时”。

例子：`LightState` 超时 500 ms，监视报文 ID 为 `0x456`。收到 `0x456` 就刷新；超过 500 ms 没收到就触发超时。

#### 6. 一个完整的 IL 例子

```text
BCM1_1：LightCmd
  ILUsed = Yes
  GenMsgILSupport = Yes
  GenMsgSendType = CyclicIfActiveAndSpontaneous
  GenMsgCycleTime = 100
  GenMsgCycleTimeFast = 20
  GenSigSendType = IfActive
  GenSigInactiveValue = 0
  GenMsgNrOfRepetition = 3
  GenMsgDelayTime = 20

BCM1_2：LightState
  ILUsed = Yes
  GenMsgILSupport = Yes
  GenMsgSendType = Cyclic
  GenMsgCycleTime = 500
  GenSigStartValue = 0
```

通俗理解：

- `LightCmd=1` 时，`BCM1_1` 按 20 ms 快速发送；`LightCmd=0` 时停止/恢复。
- 灯光命令变化时，额外补发，且至少间隔 20 ms。
- `BCM1_2` 每 500 ms 报告一次灯光状态，上电初值为 Off。

---

## 4. 传输协议与诊断属性（TP / Diagnostics）

### 4.1 TP 属性详解

TP（Transport Protocol）用于发送超过 8 字节的数据，负责分段、流控和重组。

| 属性 | 对象 / 值 | 含义 | 用途 | 原理 | 通俗例子 |
|---|---|---|---|---|---|
| `TpBaseAddress` | 网络 / 十六进制 CAN ID | TP 报文基地址 | 扩展寻址模式推导 TP ID | 按基地址和寻址规则计算 TP 报文 ID | 基地址 `0x700`，节点地址偏移后得到对应 TP ID |
| `TpNodeBaseAddress` | 节点 / 十六进制 CAN ID | 节点级 TP 基地址 | 不同节点使用不同 TP 基址 | 节点 TP 层用该基址计算发送/接收 ID | 节点 A 用 `0x710`，节点 B 用 `0x720` |
| `TpTxIdentifier` | 节点 / 十六进制 CAN ID | TP 发送 ID | 直接指定节点 TP 发送 ID | TP 发送时使用该 CAN ID | `TpTxIdentifier=0x7E0` |
| `TpRxIdentifier` | 节点 / 十六进制 CAN ID | TP 接收 ID | 直接指定节点 TP 接收 ID | TP 接收过滤时使用该 ID | `TpRxIdentifier=0x7E8` |
| `TpRxMask` | 节点 / 十六进制 | 接收 ID 掩码 | 从 CAN ID 中匹配 TP 报文 | 按掩码比较 ID 的相关位 | `TpRxMask=0x7F8`，只比较高 11 位中的有效位 |
| `TpAddressExtension` | 节点 / 整数 | 地址扩展 | 11 位混合寻址 | TP 帧携带地址扩展字节，接收方按扩展地址过滤 | `TpAddressExtension=0xF1` |
| `TpSTMin` | 节点 / 整数 ms | 最小分段间隔 | 控制连续帧发送间隔 | 接收方通过流控帧告知发送方 STmin | `TpSTMin=10`：连续帧至少间隔 10 ms |
| `TpBlockSize` | 节点 / 整数 | 块大小 | 控制每批连续帧数量 | 发送 BS 个连续帧后等待新的流控帧 | `TpBlockSize=8`：发 8 帧后等 FC |
| `TpTargetAddress` | 节点 / 十六进制 0x00...0xFF | 目标地址 | 扩展寻址指定目标 | 发送方把目标地址写入 TP 地址字段 | `TpTargetAddress=0x10` |
| `TpAddressingMode` | 节点 / 整数 0...4 | TP 寻址模式 | 决定 ID/地址字段规则 | 不同模式对应不同 TP 地址计算方式 | 选择 `2` 使用 Normal Fixed 寻址 |
| `TpTxAdrMode` | 节点 / 整数 0、1 | 发送寻址类型 | NormalFixed/Mixed 下选择物理/功能寻址 | `0` 物理寻址面向单节点；`1` 功能寻址面向一组节点 | `TpTxAdrMode=1` 让多个 ECU 同时接收 |
| `TpUseFC` | 节点 / 整数 0、1 | 是否使用流控 | 决定 TP 是否使用 FC 帧 | `0` 不用 FC；`1` 用 FC 协商 BS/STmin | `TpUseFC=1` 启用流控 |
| `TpCanBus` | 节点 / 整数 1、2 | TP 使用的 CAN 通道 | 多通道环境选择通道 | 将 TP 收发绑定到指定 CAN 通道 | `TpCanBus=1` 表示走 CAN1 |

### 4.2 TP 枚举值逐一解释

#### `TpAddressingMode`

| 值 | 通俗理解 | 深入原理 | 例子 |
|---:|---|---|---|
| `0` | 普通寻址 | Normal addressing：按普通 CAN ID 和固定字段区分 TP 连接 | 普通 11 位 CAN ID 的诊断 TP |
| `1` | 扩展寻址 | Extended addressing：TP 帧中增加地址扩展字节 | 同一 CAN ID 下用扩展地址区分不同 ECU |
| `2` | 普通固定寻址 | Normal fixed addressing：使用固定 ID 映射规则 | 固定 `0x7E0/0x7E8` 诊断对 |
| `3` | 混合寻址 | Mixed addressing：结合普通寻址和地址扩展 | 需要同时用 CAN ID 和地址字段区分目标 |
| `4` | 11 位混合寻址 | 11-bit mixed addressing：11 位 ID 下的混合规则 | 兼容 11 位 ID 的混合地址场景 |

#### `TpTxAdrMode`

| 值 | 通俗理解 | 深入原理 | 例子 |
|---:|---|---|---|
| `0` | 点对点发送 | Physical addressing：目标是一个具体 ECU | 只给发动机 ECU 发诊断请求 |
| `1` | 群发/功能发送 | Functional addressing：目标是一组 ECU | 广播“所有 ECU 进入诊断会话” |

#### `TpUseFC`

| 值 | 通俗理解 | 深入原理 | 例子 |
|---:|---|---|---|
| `0` | 不用流控 | 不使用 Flow Control 帧 | 固定小数据，接收方不需要协商流量 |
| `1` | 使用流控 | 接收方通过 FC 告知 Block Size 和 STmin | 大文件刷写时启用流控避免溢出 |

#### `TpCanBus`

| 值 | 通俗理解 | 深入原理 | 例子 |
|---:|---|---|---|
| `1` | 走 CAN 通道 1 | TP 收发绑定 CAN1 | 诊断走 CAN1 |
| `2` | 走 CAN 通道 2 | TP 收发绑定 CAN2 | 诊断走 CAN2 |

### 4.3 诊断属性详解

| 属性 | 对象 / 值 | 含义 | 用途 | 原理 | 通俗例子 |
|---|---|---|---|---|---|
| `DiagRequest` | 报文 / 枚举 No、Yes | 是否为诊断请求报文 | 诊断方向识别 | 诊断层按请求方向处理并与响应关联 | `0x7E0` 设为 `Yes` |
| `DiagResponse` | 报文 / 枚举 No、Yes | 是否为诊断响应报文 | 诊断方向识别 | 诊断层按响应方向处理并配对请求 | `0x7E8` 设为 `Yes` |
| `DiagState` | 报文 / 枚举 No、Yes | 是否为诊断状态/诊断 TP 报文 | 诊断状态和 TP 分类 | 被纳入诊断连接和 TP 状态管理 | 一条诊断状态报文设为 `Yes` |
| `DiagConnection` | 报文 / 字符串或项目定义枚举 | 诊断连接/通道名称 | 多诊断链路区分 | 诊断运行时按连接名选择 TP、地址和回调 | `DiagConnection="UDS_CAN1"` |
| `DiagUUDTResponse` | 报文 / 枚举 No、Yes | 是否为 UUDT 非请求响应 | 识别主动诊断响应 | 不依赖请求上下文，交给 UUDT 接收逻辑 | 主动故障上报报文设为 `Yes` |
| `DiagStationAddress` | 节点 / 十六进制 0x00...0xFF | 节点诊断地址 | 诊断寻址 | 诊断报文/TP 地址字段使用该地址标识 ECU | `0x10` 表示该 ECU 的诊断地址 |

### 4.4 TP/诊断属性组合与相互影响

#### 1. 寻址模式 + 基地址 + ID + 掩码

```text
TpAddressingMode
  → 决定使用哪种 ID/地址计算规则
TpBaseAddress / TpNodeBaseAddress
  → 提供 ID 基址
TpTxIdentifier / TpRxIdentifier
  → 直接指定 ID（部分模式）
TpRxMask
  → 接收时匹配有效位
TpTargetAddress / TpAddressExtension
  → 扩展寻址的目标和扩展地址
```

例子：`TpAddressingMode=1`（扩展寻址）时，`TpAddressExtension` 和 `TpTargetAddress` 才有意义；如果选普通寻址却配置扩展地址，可能不会生效。

#### 2. STmin + BlockSize + UseFC

```text
TpUseFC=1
  → 使用流控
TpBlockSize
  → 每批连续帧数量
TpSTMin
  → 连续帧最小间隔
```

例子：`TpBlockSize=8`、`TpSTMin=10`，接收方每收 8 帧发一次 FC，要求发送方帧间至少等 10 ms。

#### 3. 诊断标记 + 连接 + 地址

`DiagRequest`/`DiagResponse`/`DiagState`/`DiagUUDTResponse` 决定报文在诊断层的角色；`DiagConnection` 决定它属于哪条诊断链路；`DiagStationAddress` 决定目标 ECU 地址。

例子：`0x7E0` 设 `DiagRequest=Yes`、`0x7E8` 设 `DiagResponse=Yes`，两者都绑定同一个 `DiagConnection="UDS_CAN1"`，再配合 `DiagStationAddress` 定位 ECU。

---

## 5. 网络管理属性（Network Management）

### 5.1 NM 属性详解

| 属性 | 对象 / 值 | 含义 | 用途 | 原理 | 通俗例子 |
|---|---|---|---|---|---|
| `NmType` | 网络 / 字符串 | 网络管理类型 | 选择 NM 实现/规范 | 工具按类型启用对应 NM 状态机 | `NmType="Vector"` |
| `NmBaseAddress` | 网络 / 十六进制 CAN ID | NM 报文基地址 | 计算各节点 NM ID | 通常 `NM ID = NmBaseAddress + NmStationAddress` | 基地址 `0x400` |
| `NmMessageCount` | 网络 / 整数 1...255 | NM ID 数量/预留范围 | 避免与普通报文冲突 | 从基地址开始预留指定数量 ID | `NmMessageCount=128` 预留 `0x400...0x47F` |
| `NmNode` | 节点 / 枚举 No、Yes | 是否参与 NM | 标记需要 NM 状态机的节点 | `Yes` 启用 NM 发送/接收/睡眠/唤醒 | BCM 设为 `Yes` |
| `NmMessage` | 报文 / 枚举 No、Yes | 是否为 NM 报文 | 让 NM 层识别 NM 报文 | `Yes` 时按 NM 规则解析和发送 | `0x412` 设为 `Yes` |
| `NmStationAddress` | 节点 / 十六进制 | 节点 NM 地址 | 计算 NM 报文 ID | `NM ID = NmBaseAddress + NmStationAddress` | 地址 `0x12`、基址 `0x400` → ID `0x412` |
| `NmCAN` | 节点 / 整数 1、2 | NM 使用的 CAN 通道 | 多通道选择 | 绑定 NM 收发通道；多用于旧版/兼容模式 | `NmCAN=1` 表示 CAN1 |
| `GenNWMSleepTime` | 网络 / 整数 ms | 网络级睡眠等待时间 | 统一所有节点等待时间 | NM 进入睡眠前等待该时间 | 网络统一等待 2000 ms |
| `GenNodSleepTime` | 节点 / 整数 ms | 节点级睡眠等待时间 | 不同节点不同等待时间 | 大于 0 时覆盖网络级 `GenNWMSleepTime` | BCM 单独等待 1500 ms |
| `NWM-WakeupAllowed` | 信号 / 枚举 No、Yes | 信号是否允许影响 NM 唤醒 | 控制哪些信号参与唤醒 | `No` 时信号变化不影响 NM 唤醒 | 普通状态信号设为 `No` |

### 5.2 AUTOSAR NM 属性（NmAsr*）

| 属性 | 对象 / 值 | 含义 | 用途 | 原理 | 通俗例子 |
|---|---|---|---|---|---|
| `NmAsrBaseAddress` | 网络 / 十六进制 CAN ID | AUTOSAR NM 报文基地址 | 确定 NM ID 区域起点 | 按基地址和节点标识/配置计算 NM ID | `NmAsrBaseAddress=0x500` |
| `NmAsrMessageCount` | 网络 / 整数 | AUTOSAR NM ID 数量 | 预留 NM ID 范围 | 从基地址开始预留指定数量 ID | `NmAsrMessageCount=32` |
| `NmAsrNode` | 节点 / 枚举 No、Yes | 是否参与 AUTOSAR NM | 启用 AUTOSAR NM 状态机 | `Yes` 时处理 Repeat Message、Ready Sleep、Bus Sleep | BCM 设为 `Yes` |
| `NmAsrNodeIdentifier` | 节点 / 整数或十六进制 | AUTOSAR NM 节点标识 | 标识节点、参与 NM 报文内容/ID | 写入 NM PDU 的 Node ID 字段，部分配置用于 ID 计算 | `NmAsrNodeIdentifier=0x21` |
| `NmAsrMessage` | 报文 / 枚举 No、Yes | 是否为 AUTOSAR NM 报文 | 识别 ASR NM 报文 | `Yes` 时按 AUTOSAR NM 周期/状态规则处理 | `0x521` 设为 `Yes` |
| `NmAsrCanMsgCycleTime` | 网络 / 整数 ms | 正常 NM 周期 | 配置 NM 正常发送节拍 | NM 状态机在正常阶段按该周期发送 | `NmAsrCanMsgCycleTime=500` |
| `NmAsrCanMsgReducedTime` | 网络 / 整数 ms | 缩短的 NM 周期 | 快速 NM 发送阶段 | 在 Repeat Message/唤醒阶段用较短周期替代正常周期 | `NmAsrCanMsgReducedTime=100` |
| `NmAsrCanMsgCycleOffset` | 网络 / 整数 ms | NM 周期相位偏移 | 错开节点发送时刻 | 定时器先等待偏移再按周期发送 | `NmAsrCanMsgCycleOffset=50` |
| `NmAsrRepeatMessageTime` | 网络 / 整数 ms | Repeat Message 状态时长 | 控制唤醒后快速发送持续时间 | 定时器到期后进入 Normal/Ready Sleep | `NmAsrRepeatMessageTime=1000` |
| `NmAsrTimeoutTime` | 网络 / 整数 ms | NM 超时时间 | 判断网络是否还活跃 | 超过该时间未收到 NM 报文则离开 Network Mode | `NmAsrTimeoutTime=2000` |
| `NmAsrWaitBusSleepTime` | 网络 / 整数 ms | 等待 Bus Sleep 时间 | 让总线安静后睡眠 | Prepare Bus Sleep 等待该时间后进入 Bus Sleep | `NmAsrWaitBusSleepTime=1500` |

### 5.3 NM 属性组合与相互影响

#### 1. NM 地址和 ID 范围

```text
NM message ID = NmBaseAddress + NmStationAddress
```

例子：`NmBaseAddress=0x400`、`NmStationAddress=0x12`，则 NM ID 为 `0x412`；`NmMessageCount` 则规定从基地址开始预留多少 ID。

#### 2. 节点开关 + 报文开关

`NmNode=Yes` 只表示节点参与 NM；还需要 `NmMessage=Yes` 标记具体 NM 报文，二者组合后 NM 层才知道“这个节点用哪条报文做 NM”。

#### 3. 睡眠时间覆盖关系

`GenNodSleepTime > 0` 时覆盖网络级 `GenNWMSleepTime`；等于 0 时使用网络级配置。

例子：网络统一 2000 ms，但 BCM 需要 1500 ms，则 `GenNodSleepTime=1500`。

#### 4. AUTOSAR NM 定时器组合

```text
唤醒/启动
  → Repeat Message 状态：NmAsrRepeatMessageTime + NmAsrCanMsgReducedTime
  → 正常阶段：NmAsrCanMsgCycleTime + NmAsrCanMsgCycleOffset
  → 超时：NmAsrTimeoutTime
  → 进入睡眠：NmAsrWaitBusSleepTime
```

这些定时器共同决定 NM 状态机的节奏；只改其中一个可能导致状态迁移时间不符合 OEM 规范。

---

## 6. 工具特定属性（Tool-specific）

| 属性 | 对象 / 值 | 含义 | 用途 | 原理 | 通俗例子 |
|---|---|---|---|---|---|
| `NodeLayerModules` | 节点 / 字符串 | CANoe 加载的节点层 DLL 列表 | 启用 TP/NM 等节点层模块 | 逗号分隔，CANoe 启动节点时加载 | `"OSEK_TP.DLL, OSEKNM.DLL"` |
| `BusType` | 网络 / 字符串 | 总线类型 | 选择通道和协议解析器 | 工具按类型选择 CAN/LIN/FlexRay/Ethernet 解析 | `"CAN"` 表示 CAN 总线 |
| `ProtocolType` | 网络 / 字符串 | 上层协议类型 | 激活 J1939/CANopen 等协议 | 工具加载对应协议选项/插件 | `"J1939"` 启用 J1939 解析 |
| `ECU` | 节点 / 字符串 | 节点所属 ECU | 多总线 ECU 的分组 | CANoe 以 ECU 为单位组织节点/接口 | 同一 ECU 的两个节点都填 `"BCM_ECU"` |
| `CANoeStartDelay` | 节点 / 整数 ms | 测量开始后节点被动时间 | 模拟节点启动延迟 | 时间内不响应也不主动发送 | 上电后前 1000 ms 不参与通信 |
| `CANoeDrift` | 节点 / 整数 | 定时器漂移修正 | 模拟 ECU 时钟偏差 | 按该值延长/缩短节点定时器周期 | 非零值让周期逐渐偏快/偏慢 |
| `CANoeJitterMin` | 节点 / 整数 ms | 抖动下限 | 模拟周期抖动 | 定时器触发时间在上下限间分布 | 抖动范围下限 0 ms |
| `CANoeJitterMax` | 节点 / 整数 ms | 抖动上限 | 模拟周期抖动 | 与下限共同决定随机抖动区间 | 抖动范围上限 10 ms |

### 6.1 `ProtocolType` 常见值

| 值 | 通俗理解 | 深入原理 | 例子 |
|---|---|---|---|
| `J1939` | 商用车 J1939 | 启用 J1939 地址声明、参数组解析 | 发动机/车身 J1939 报文 |
| `CANopen` | CANopen 协议 | 启用对象字典、PDO/SDO 解析 | 工业 CANopen 设备 |
| `AFDX` | 航空 AFDX | 启用虚拟链路和航空总线解析 | 航空电子网络 |
| `ARINC825` | 航空 ARINC825 | 启用 ARINC825 解析 | 航空 CAN 网络 |
| `CANaerospace` | 航空 CANaerospace | 启用 CANaerospace 解析 | 航空传感器网络 |

> 这些协议功能通常需要安装对应 CANoe/CANalyzer 选项并具备有效 License。

### 6.2 `BusType` 常见值

| 值 | 通俗理解 | 深入原理 | 例子 |
|---|---|---|---|
| `CAN` | 普通 CAN | 选择 CAN 通道、驱动和报文解析 | 本项目的 CAN 网络 |
| `LIN` | LIN 总线 | 选择 LIN 通道和调度/解析 | 车门 LIN 子网 |
| `FlexRay` | FlexRay | 选择 FlexRay 通道、时隙和帧解析 | 底盘 FlexRay 网络 |
| `Ethernet` | 车载以太网 | 选择以太网通道和协议解析 | 车载以太网骨干 |

### 6.3 关于 `BusType already exists in the database and wasn't imported`

这不是属性名，而是 CANdb++ 导入属性定义时的提示：

```text
BusType already exists in the database and wasn't imported
```

含义：当前 DBC 已有 `BusType` 属性，导入新的属性定义时跳过同名属性，避免重复定义。

处理建议：

- 不需要重复创建 `BusType`。
- 检查现有 `BusType` 的对象类型和值类型是否与导入源一致。
- 如果类型不兼容，先确认是否修改/删除旧定义，再重新导入。
- 同名但对象类型或值类型不同，可能导致赋值失败或 CANdb++ 报错。

---

## 7. 跨类别属性组合与相互影响

### 7.1 IL 与 TP 的边界

- IL 负责应用信号的发送/接收行为。
- TP 负责大于 8 字节数据的分段、流控和重组。
- 诊断 TP 报文通常由 `DiagRequest`/`DiagResponse`/`DiagState` 标记，不应再当作普通 IL 应用报文处理。
- 同一条报文不要同时承担普通 IL 信号发送和诊断 TP 分段，否则发送时序和超时容易冲突。

### 7.2 IL 与 NM 的边界

- IL 管应用报文。
- NM 管网络睡眠、唤醒和网络状态。
- `NmMessage=Yes` 或 `NmAsrMessage=Yes` 的报文通常不应由普通 IL 自动发送。
- `NWM-WakeupAllowed=No` 可防止普通状态信号误触发 NM 唤醒。

### 7.3 总线类型、协议类型与 IL/TP/NM

`BusType` 决定工具使用哪种总线解析器；`ProtocolType` 激活上层协议；IL/TP/NM 属性只在该总线/协议环境中有意义。

例子：`BusType="CAN"`、`ProtocolType="J1939"` 时，IL/TP/NM 的行为可能受 J1939 选项影响；如果 BusType 设错，后续属性即使正确也无法按预期工作。

### 7.4 定时器属性的层次关系

| 属性 | 作用层次 | 与谁组合 | 例子 |
|---|---|---|---|
| `GenMsgStartDelayTime` | 报文启动 | `GenMsgFastOnStart`、周期 | 上电等 500 ms 后开始发 |
| `GenMsgCycleTime` | 正常周期 | `GenMsgSendType` | 每 100 ms 发 |
| `GenMsgCycleTimeFast` | 快速周期 | `GenSigInactiveValue`、`IfActive` | 激活时每 20 ms 发 |
| `GenMsgDelayTime` | 最小间隔 | `GenMsgNrOfRepetition` | 事件帧之间至少隔 20 ms |
| `GenSigCycleTime` | 信号周期 | 信号级 IL | 信号每 100 ms 发 |
| `GenSigCycleTimeActive` | 信号激活周期 | `GenSigInactiveValue` | 激活时每 20 ms 发 |
| `CANoeStartDelay` | 节点仿真启动 | 所有发送属性 | 测量开始后 1 s 内节点被动 |
| `CANoeDrift` / `CANoeJitterMin/Max` | 节点定时器仿真 | 周期/超时 | 让周期产生漂移和抖动 |

### 7.5 端到端组合例子

```text
网络：BusType=CAN，Baudrate=500000，DBName=CAN1
节点：BCM、LCU
IL：BCM 节点 ILUsed=Yes
报文 BCM1_1（LCU 发，BCM 收）：
  GenMsgILSupport=Yes
  GenMsgSendType=CyclicIfActive
  GenMsgCycleTime=100
  GenMsgCycleTimeFast=20
  GenSigSendType=IfActive
  GenSigInactiveValue=0
报文 BCM1_2（BCM 发，LCU 收）：
  GenMsgILSupport=Yes
  GenMsgSendType=Cyclic
  GenMsgCycleTime=500
  GenSigStartValue=0
接收监控：
  LCU 对 LightState 设置 GenSigTimeoutTime=500
  GenSigTimeoutMsg=0x456
```

通俗结果：

- `LCU` 的 `LightCmd=1` 时，`BCM1_1` 快速周期发送；`0` 时停止/降速。
- `BCM` 每 500 ms 报告 `LightState`，初值 Off。
- `LCU` 500 ms 没收到状态就判定超时。
- 网络名为 `CAN1`，波特率 500 kbit/s，总线类型 CAN。

---

## 8. 属性索引

| 分类 | 属性 |
|---|---|
| 一般/网络 | `Manufacturer`、`Baudrate`、`DBName`、`VersionYear`、`VersionMonth`、`VersionWeek`、`VersionDay`、`VersionNumber` |
| 交互层 | `ILUsed`、`IlTxTimeout`、`GenMsgILSupport`、`GenMsgSendType`、`GenSigSendType`、`GenMsgDelayTime`、`GenMsgCycleTime`、`GenMsgCycleTimeFast`、`GenMsgCycleTimeActive`、`GenMsgStartDelayTime`、`GenMsgNrOfRepetition`、`GenMsgFastOnStart`、`GenSigCycleTime`、`GenSigCycleTimeActive`、`GenSigStartValue`、`GenSigInactiveValue`、`GenSigTimeoutTime`、`GenSigTimeoutMsg` |
| TP | `TpBaseAddress`、`TpNodeBaseAddress`、`TpTxIdentifier`、`TpRxIdentifier`、`TpRxMask`、`TpAddressExtension`、`TpSTMin`、`TpBlockSize`、`TpTargetAddress`、`TpAddressingMode`、`TpTxAdrMode`、`TpUseFC`、`TpCanBus` |
| 诊断 | `DiagRequest`、`DiagResponse`、`DiagState`、`DiagConnection`、`DiagUUDTResponse`、`DiagStationAddress` |
| 网络管理 | `NmType`、`NmBaseAddress`、`NmMessageCount`、`NmNode`、`NmMessage`、`NmStationAddress`、`NmCAN`、`GenNWMSleepTime`、`GenNodSleepTime`、`NWM-WakeupAllowed` |
| AUTOSAR NM | `NmAsrBaseAddress`、`NmAsrMessageCount`、`NmAsrNode`、`NmAsrNodeIdentifier`、`NmAsrMessage`、`NmAsrCanMsgCycleTime`、`NmAsrCanMsgReducedTime`、`NmAsrCanMsgCycleOffset`、`NmAsrRepeatMessageTime`、`NmAsrTimeoutTime`、`NmAsrWaitBusSleepTime` |
| 工具特定 | `NodeLayerModules`、`BusType`、`ProtocolType`、`ECU`、`CANoeStartDelay`、`CANoeDrift`、`CANoeJitterMin`、`CANoeJitterMax` |

---

## 9. 参考依据

本文属性清单和基础定义参考 CANdb++ 3.1 SP4 自带帮助中的 **Vector Tool Chain Attributes** 页面，并结合 Vector 工具链中 IL、TP、NM、诊断的通用工作机制进行解释。

`DBName`、`Baudrate` 等属性行为参考 CANdb++ 帮助中关于 DBC 导出、网络名称和波特率的说明；`NmAsr*`、`IlTxTimeout`、`GenSigCycleTime`、`GenSigCycleTimeActive`、`DiagState`、`DiagConnection`、`DiagUUDTResponse` 等属性在不同 CANdb++/CANoe 版本和 OEM 属性包中可能存在定义差异。实际项目请以对应版本的属性定义、DBC 中的 `BA_DEF_` 以及 OEM 规范为准。
