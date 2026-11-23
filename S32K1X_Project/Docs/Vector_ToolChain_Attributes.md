# CANdb++ / Vector Tool Chain Attributes 说明

本文说明 CANdb++（Vector CANdb++ Admin）中 **Vector Tool Chain Attributes** 所包含的典型属性，重点解释每个属性的：

- **意思**：属性表达什么信息
- **用途**：哪个工具、哪个阶段会使用它
- **原理**：它通过什么机制影响交互层（IL）、传输协议（TP）、网络管理（NM）或 CANoe/CANalyzer 行为

> 范围说明：本文只解释属性，不修改 `Comfort_CAN.dbc`。  
> 不同 CANdb++、CANoe、CANalyzer 版本、不同 OEM 属性集以及不同 IL 类型，属性集合和枚举值可能略有差异；实际项目应以对应版本的属性定义和 OEM 规范为准。

---

## 1. 总体理解

### 1.1 什么是 Vector Tool Chain Attributes

Vector Tool Chain Attributes 不是 CAN 总线物理层或 CAN 帧格式的一部分，而是 **Vector 工具链使用的用户自定义属性**。它们通常由 OEM 或项目在 DBC 中定义，用来描述：

- 报文何时发送、周期多长、是否重复
- 信号变化是否触发报文、非活跃值是什么
- 接收超时如何判断
- 诊断和 TP 使用哪些 CAN ID、寻址方式、流控参数
- 网络管理报文如何分配 ID
- CANoe/CANalyzer 如何加载节点、仿真节点和定时器

这些属性主要影响：

```text
CANdb++ 数据库管理
CANoe / CANalyzer 的仿真、IL、TP、NM 行为
CAPL 生成器/交互层代码生成
诊断配置
网络管理配置
```

ECU 固件本身不会因为 DBC 里多了一个属性就自动实现对应行为；DBC 属性只是“描述和配置”，ECU 软件必须按相同规则实现。

### 1.2 在 DBC 中的存储原理

DBC 通过三类语句保存这些属性：

```text
BA_DEF_     定义属性名称、对象类型、值类型和取值范围
BA_DEF_DEF_ 定义属性默认值
BA_         给某个网络、节点、报文或信号赋属性值
```

示意：

```text
BA_DEF_ BO_ "GenMsgCycleTime" INT 0 65535;
BA_DEF_DEF_ "GenMsgCycleTime" 0;
BA_ "GenMsgCycleTime" BO_ 291 100;
```

含义是：为报文对象定义一个整数属性 `GenMsgCycleTime`，默认值为 0，并把 ID 为 291 的报文周期设置为 100 ms。

**注意：** CANdb++ 对 DBC 段顺序比较敏感。通常应保证：

```text
BA_DEF_ / BA_DEF_DEF_ / BA_
```

位于 `VAL_` 之前，否则 CANdb++ 可能报语法错误。

### 1.3 对象类型说明

本文中的对象类型含义：

| 对象类型 | 含义 |
|---|---|
| 网络（Network） | 整个 CAN 网段/数据库级配置 |
| 节点（Node） | ECU 或仿真节点 |
| 报文（Message） | 一条 CAN 报文 |
| 信号（Signal） | 报文中的一个信号 |
| 节点映射的 Rx 信号 | 某节点接收并映射的信号 |
| `<_ECU>` 后缀 | 当属性按接收 ECU 区分时，在属性名后加接收节点名 |

---

## 2. 一般属性（General）

### 2.1 Manufacturer

- **对象类型**：网络
- **值类型**：字符串
- **默认值**：`""`
- **意思**：指定该网络所属的 OEM/车厂。
- **用途**：用于数据库管理、导出、项目筛选和 OEM 规范区分。
- **原理**：属于元数据，不直接影响报文发送；Vector 工具和脚本可按 OEM 识别数据库来源或选择对应规则。

### 2.2 VersionYear

- **对象类型**：网络
- **值类型**：整数
- **默认值**：0
- **有效值**：0...99
- **意思**：网络发布版本的年份，按 BCD 编码填写。
- **用途**：版本管理和发布标识。
- **原理**：工具读取后作为版本信息保存/显示；BCD 编码意味着例如 2024 年通常写成 BCD 形式的 `0x24` 对应数值，而不是普通十进制 2024。

### 2.3 VersionMonth

- **对象类型**：网络
- **值类型**：整数
- **默认值**：1
- **有效值**：0...12
- **意思**：网络发布版本的月份，BCD 编码。
- **用途**：版本管理。
- **原理**：与 `VersionYear` 一起构成发布日期的月部分；不参与总线调度。

### 2.4 VersionWeek

- **对象类型**：网络
- **值类型**：整数
- **默认值**：1
- **有效值**：0...52
- **意思**：网络发布版本的周数，BCD 编码。
- **用途**：按周发布/版本追踪。
- **原理**：作为版本元数据保存，不改变通信行为。

### 2.5 VersionDay

- **对象类型**：网络
- **值类型**：整数
- **默认值**：1
- **有效值**：0...31
- **意思**：网络发布版本的日，BCD 编码。
- **用途**：版本管理。
- **原理**：与年、月、周共同组成发布版本日期。

### 2.6 VersionNumber

- **对象类型**：网络
- **值类型**：整数
- **默认值**：0
- **有效值**：>= 0
- **意思**：网络发布版本号，BCD 编码。
- **用途**：区分同一网络的不同发布版本。
- **原理**：版本元数据；工具可用它判断数据库新旧或生成版本标识。

---

## 3. 交互层属性（Interaction Layer）

交互层属性描述报文和信号的发送/接收行为。CANoe 的 IL 或 CAPL 生成器会根据这些属性生成周期发送、事件发送、超时处理等逻辑。

### 3.1 ILUsed

- **对象类型**：节点
- **值类型**：枚举
- **默认值**：No
- **有效值**：No、Yes（枚举顺序 1=No，2=Yes）
- **意思**：该节点是否使用交互层。
- **用途**：告诉 CANoe 是否为该节点启用 IL 相关的发送/接收处理。
- **原理**：设为 Yes 后，IL 生成器会把该节点纳入 IL 逻辑，例如自动发送、超时监控和信号更新；设为 No 时通常只做普通报文收发或由用户脚本自行处理。

### 3.2 GenMsgILSupport

- **对象类型**：报文
- **值类型**：枚举
- **默认值**：No
- **有效值**：No、Yes
- **意思**：该报文是否由交互层处理。
- **用途**：标记哪些报文参与 IL 的发送/接收逻辑。
- **原理**：IL 生成器只对标记为 Yes 的报文生成 IL 相关代码或调度逻辑；No 的报文通常由其他模块或手工 CAPL 处理。

### 3.3 GenMsgSendType

- **对象类型**：报文
- **值类型**：枚举
- **默认值**：取决于 IL 类型
- **有效值**：取决于 IL 类型；常见值包括 Cyclic、Spontaneous、IfActive、Event 等
- **意思**：定义报文级发送类型。
- **用途**：决定报文是周期发送、事件触发、激活时发送还是其他模式。
- **原理**：IL 将“报文发送类型”和“报文内各信号的发送类型”组合起来，决定最终发送行为。例如报文为 Cyclic 时按 `GenMsgCycleTime` 周期发送；报文为 IfActive 时还要看信号是否处于激活状态。

### 3.4 GenSigSendType

- **对象类型**：信号
- **值类型**：枚举
- **默认值**：取决于 IL 类型
- **有效值**：取决于 IL 类型；常见值包括 Cyclic、OnChange、OnWrite、IfActive 等
- **意思**：定义信号级发送类型。
- **用途**：描述信号变化、写入或激活时如何触发所在报文。
- **原理**：信号发送类型会影响报文发送触发条件、重复次数和快速周期。例如 OnChange 表示信号值变化时触发报文；IfActive 表示信号不等于非活跃值时才触发/维持快速发送。

### 3.5 GenMsgDelayTime

- **对象类型**：报文
- **值类型**：整数
- **单位**：ms
- **默认值**：0
- **有效值**：>= 0
- **意思**：两次报文发送之间的最小时间间隔。
- **用途**：限制报文发送频率，避免事件密集时总线负载过高。
- **原理**：IL 调度器在发送报文前检查距离上次发送的时间；未达到最小间隔时会等待或抑制发送。

### 3.6 GenMsgCycleTime

- **对象类型**：报文
- **值类型**：整数
- **单位**：ms
- **默认值**：0
- **有效值**：>= 0
- **意思**：周期报文的固定发送周期。
- **用途**：为 Cyclic 报文提供周期发送参数。
- **原理**：IL 使用定时器按该周期触发发送；值 0 通常表示不使用固定周期或由其他机制触发。

### 3.7 GenMsgCycleTimeFast

- **对象类型**：报文
- **值类型**：整数
- **单位**：ms
- **默认值**：0
- **有效值**：>= 0
- **意思**：高速/快速发送阶段的周期。
- **用途**：用于 IfActive 等模式：当报文内任一信号处于激活状态时，提高发送频率。
- **原理**：IL 根据信号是否激活在正常周期和快速周期之间切换；快速周期通常明显小于 `GenMsgCycleTime`。

### 3.8 GenMsgCycleTimeActive

- **对象类型**：报文
- **值类型**：整数
- **单位**：ms
- **默认值**：0
- **有效值**：>= 0
- **意思**：CAPL 生成器交互层中的“激活状态周期”。
- **用途**：与 `GenMsgCycleTimeFast` 类似，但面向 CAPL 生成器的 IL。
- **原理**：CAPL IL 在报文处于激活阶段时使用该周期发送；具体优先级由对应 IL 实现决定。

### 3.9 GenMsgStartDelayTime

- **对象类型**：报文
- **值类型**：整数
- **单位**：ms
- **默认值**：0
- **有效值**：>= 0
- **意思**：系统启动后到该报文第一次发送的延迟时间。
- **用途**：避免上电时所有报文同时发送造成启动突发。
- **原理**：IL 启动后先等待该时间，再启动该报文的周期/发送逻辑。

### 3.10 GenMsgNrOfRepetition（GenMsgNrOfRepetitions）

- **对象类型**：报文
- **值类型**：整数
- **默认值**：0
- **有效值**：>= 0
- **意思**：报文需要重复发送时的重复次数。
- **用途**：用于事件型/变化型发送，例如信号变化后连续发送若干次。
- **原理**：报文是否重复以及重复次数由报文内信号的 `GenSigSendType` 决定；该属性提供重复次数。不同工具版本中属性名可能是 `GenMsgNrOfRepetition` 或 `GenMsgNrOfRepetitions`。

### 3.11 GenSigInactiveValue

- **对象类型**：信号
- **值类型**：整数
- **默认值**：0
- **意思**：信号的“非激活值”，按信号原始值（raw value）填写。
- **用途**：配合 IfActive 发送模式判断信号是否处于非激活状态。
- **原理**：当信号发送类型为 IfActive 时，如果信号值不等于 `GenSigInactiveValue`，则认为信号激活，其所在报文可以按 `GenMsgCycleTimeFast` 快速周期发送；等于非激活值时通常回到正常周期或停止快速发送。

### 3.12 GenSigStartValue

- **对象类型**：信号
- **值类型**：整数
- **默认值**：0
- **意思**：信号启动/初始值，按信号原始值填写。
- **用途**：系统启动后、应用尚未第一次设置该信号之前，用该值作为初始发送值。
- **原理**：IL 用该值初始化信号；应用写入新值后，新值覆盖初始值。它不是物理值，而是未应用 factor/offset 前的 raw value。

### 3.13 GenSigTimeoutTime（GenSigTimeoutTime<_ECU>）

- **对象类型**：节点映射的 Rx 信号
- **值类型**：整数
- **单位**：ms
- **默认值**：0
- **有效值**：>= 0
- **意思**：接收该信号所在报文的超时时间。
- **用途**：监控接收方是否在指定时间内收到包含该信号的报文。
- **原理**：接收节点为信号设置超时定时器；报文超时未收到时触发超时事件，具体动作由所用 IL 决定（例如置默认值、报警、调用回调）。加上 `<_ECU>` 后缀可按接收 ECU 区分不同超时值。

### 3.14 GenSigTimeoutMsg（GenSigTimeoutMsg<_ECU>）

- **对象类型**：节点映射的 Rx 信号
- **值类型**：十六进制
- **单位**：CAN 报文 ID
- **默认值**：0
- **有效值**：0x00...0x7FF
- **意思**：用于监视/刷新该信号超时的报文 ID。
- **用途**：当接收节点收到该 ID 的报文时，认为对应信号仍然有效，不触发超时。
- **原理**：超时监控不仅看原报文，也可由另一条“心跳/状态”报文刷新；超时时间本身由 `GenSigTimeoutTime<_ECU>` 定义。

### 3.15 GenMsgFastOnStart

- **对象类型**：报文
- **值类型**：整数
- **单位**：ms
- **默认值**：0
- **有效值**：>= 0
- **意思**：IL 启动后，以快速周期发送周期报文的持续时间。
- **用途**：启动阶段快速发送，便于其他节点尽快获取状态。
- **原理**：只有在普通周期 `GenMsgCycleTime` 和快速周期 `GenMsgCycleTimeFast` 都大于 0 时才有效；在该时间窗口内使用快速周期，之后恢复普通周期。

---

## 4. 传输协议与诊断属性（Transport Protocol / Diagnostics）

这些属性主要用于 ISO 15765-2（CAN TP）以及诊断报文的配置。

### 4.1 TpBaseAddress

- **对象类型**：网络
- **值类型**：十六进制
- **单位**：TP 报文 CAN ID
- **默认值**：OEM 相关
- **有效值**：0x00...0x7FF
- **意思**：TP 报文 CAN ID 的基地址，仅用于扩展寻址模式。
- **用途**：由基地址推导网络内 TP 报文的 ID 分配。
- **原理**：TP 层按寻址模式和节点地址/基地址计算实际 CAN ID，保证同一网络内 TP 通信 ID 不冲突。

### 4.2 TpNodeBaseAddress

- **对象类型**：节点
- **值类型**：十六进制
- **单位**：TP 报文 CAN ID
- **默认值**：OEM 相关
- **有效值**：0x00...0x7FF
- **意思**：节点级 TP 报文基地址，仅用于扩展寻址模式。
- **用途**：当不同节点使用不同的 TP 地址基址时使用。
- **原理**：节点 TP 层用该基址计算自己的发送/接收 ID；与网络级 `TpBaseAddress` 配合或覆盖。

### 4.3 TpTxIdentifier

- **对象类型**：节点
- **值类型**：十六进制
- **单位**：TP 报文 CAN ID
- **默认值**：OEM 相关
- **有效值**：0x00...0x7FF
- **意思**：节点发送 TP 报文使用的 CAN ID（标准/11 位混合寻址）。
- **用途**：直接指定节点 TP 发送 ID。
- **原理**：TP 发送时使用该 ID 作为 CAN 标识符；适用于标准寻址和 11 位混合寻址。

### 4.4 TpRxIdentifier

- **对象类型**：节点
- **值类型**：十六进制
- **单位**：TP 报文 CAN ID
- **默认值**：OEM 相关
- **有效值**：0x00...0x7FF
- **意思**：节点接收 TP 报文使用的 CAN ID。
- **用途**：直接指定节点 TP 接收 ID。
- **原理**：TP 接收过滤/匹配时使用该 ID；与 `TpRxMask` 配合识别有效 TP 帧。

### 4.5 TpRxMask

- **对象类型**：节点
- **值类型**：十六进制
- **单位**：TP 报文 CAN ID
- **默认值**：OEM 相关
- **有效值**：0x00...0x7FF
- **意思**：接收 TP 报文 ID 的掩码。
- **用途**：用于从 CAN ID 中匹配出 TP 报文。
- **原理**：接收过滤通常按 `(received_id & TpRxMask) == (TpRxIdentifier & TpRxMask)` 之类的方式匹配；掩码决定哪些位参与比较。

### 4.6 TpCanBus

- **对象类型**：节点
- **值类型**：整数
- **默认值**：1
- **有效值**：1 = CAN 通道 1；2 = CAN 通道 2
- **意思**：节点 TP 使用的 CAN 通道。
- **用途**：多通道 CANoe 配置中指定 TP 报文走哪个通道。
- **原理**：TP 层把发送/接收绑定到指定 CAN 通道，避免跨通道误匹配。

### 4.7 TpTxAdrMode

- **对象类型**：节点
- **值类型**：整数
- **默认值**：0
- **有效值**：0 = 物理寻址；1 = 功能寻址
- **意思**：节点在 NormalFixed/Mixed 寻址模式下使用物理寻址还是功能寻址。
- **用途**：决定 TP 发送报文的目标地址类型。
- **原理**：物理寻址面向单个目标 ECU；功能寻址面向一组 ECU；TP 用不同地址字段/ID 规则实现。

### 4.8 TpAddressExtension

- **对象类型**：节点
- **值类型**：整数
- **默认值**：0
- **有效值**：>= 0
- **意思**：11 位混合寻址模式下的地址扩展值。
- **用途**：扩展地址字段，区分同一 CAN ID 下的不同目标。
- **原理**：TP 帧中携带地址扩展字节，接收方按扩展地址过滤。

### 4.9 TpSTMin

- **对象类型**：节点
- **值类型**：整数
- **单位**：ms
- **默认值**：0
- **有效值**：>= 0
- **意思**：该节点要求的最小分段间隔时间（Separation Time Minimum）。
- **用途**：控制连续帧（CF）之间的最小发送间隔。
- **原理**：接收方通过流控帧（FC）告诉发送方自己的 STmin；发送方按该时间间隔发送连续帧，防止接收缓冲溢出。

### 4.10 TpBlockSize

- **对象类型**：节点
- **值类型**：整数
- **单位**：字节
- **默认值**：0
- **有效值**：>= 0
- **意思**：该节点的块大小（Block Size）。
- **用途**：控制发送方在等待下一次流控帧前可以连续发送多少帧。
- **原理**：BS=0 通常表示不再需要流控等待，一次发完；BS>0 表示每发送 BS 个连续帧后等待新的 FC。

### 4.11 TpAddressingMode

- **对象类型**：节点
- **值类型**：整数
- **默认值**：0
- **有效值**：0...4
- **意思**：节点的 TP 寻址模式。
- **用途**：选择 TP 报文 ID 和地址字段的计算方式。
- **原理**：
  - `0`：Normal addressing，普通寻址
  - `1`：Extended addressing，扩展寻址
  - `2`：Normal fixed addressing，普通固定寻址
  - `3`：Mixed addressing，混合寻址
  - `4`：11-bit mixed addressing，11 位混合寻址

### 4.12 TpTargetAddress

- **对象类型**：节点
- **值类型**：十六进制
- **默认值**：0
- **有效值**：0x00...0xFF
- **意思**：扩展寻址模式下节点的目标地址。
- **用途**：指定 TP 报文要发送到的目标地址。
- **原理**：发送方把目标地址写入 TP 地址字段，接收方按地址匹配。

### 4.13 TpUseFC

- **对象类型**：节点
- **值类型**：整数
- **默认值**：0
- **有效值**：0 = 不使用流控；1 = 使用流控
- **意思**：是否使用流控帧（Flow Control）。
- **用途**：决定 TP 通信是否启用接收方流控机制。
- **原理**：使用 FC 时，接收方通过 FC 帧告知发送方 BS 和 STmin；不使用 FC 时通常按固定/默认规则发送。

### 4.14 DiagRequest

- **对象类型**：报文
- **值类型**：枚举
- **默认值**：No
- **有效值**：No、Yes
- **意思**：该报文是否用于诊断请求。
- **用途**：诊断配置、诊断控制台和 TP 层识别请求方向。
- **原理**：工具按该标记把报文归入诊断请求通道，并结合诊断地址/TP 参数处理。

### 4.15 DiagResponse

- **对象类型**：报文
- **值类型**：枚举
- **默认值**：No
- **有效值**：No、Yes
- **意思**：该报文是否用于诊断响应。
- **用途**：诊断配置和响应匹配。
- **原理**：工具按该标记识别响应报文，并与请求进行关联/显示。

### 4.16 DiagStationAddress

- **对象类型**：节点
- **值类型**：十六进制
- **默认值**：OEM 相关
- **有效值**：0x00...0xFF
- **意思**：节点的诊断地址。
- **用途**：诊断寻址、TP 诊断连接配置。
- **原理**：诊断报文/TP 地址字段使用该地址标识目标 ECU；不同 OEM 可能把它映射到物理或功能地址。

---

## 5. 网络管理属性（Network Management）

这些属性用于 OSEK/Vector 网络管理（NM），描述 NM 报文 ID 分配、节点参与关系和睡眠/唤醒行为。

### 5.1 NmType

- **对象类型**：网络
- **值类型**：字符串
- **默认值**：无
- **意思**：网络使用的网络管理类型。
- **用途**：选择 NM 实现/规范，例如 Vector NM、OSEK NM 等。
- **原理**：CANoe 根据该类型启用对应的 NM 状态机、报文解析和脚本生成逻辑。

### 5.2 NmBaseAddress

- **对象类型**：网络
- **值类型**：十六进制
- **单位**：CAN 报文 ID
- **默认值**：OEM 相关；Vector 默认常见为 `0x400`
- **有效值**：0x00...0x7FF
- **意思**：第一条网络管理报文的 CAN ID 基地址。
- **用途**：与节点 NM 地址相加，得到各节点 NM 报文 ID。
- **原理**：NM 报文 ID 通常按 `NmBaseAddress + NmStationAddress` 计算；基地址确定 ID 区域起点。

### 5.3 NmMessageCount

- **对象类型**：网络
- **值类型**：整数
- **默认值**：OEM 相关；Vector 默认常见为 128
- **有效值**：1...255
- **意思**：网络管理报文可使用的 CAN ID 数量/预留 ID 数。
- **用途**：确定 NM ID 范围，防止与管理报文或其他报文冲突。
- **原理**：从 `NmBaseAddress` 开始预留该数量的 ID，用于不同节点的 NM 报文。

### 5.4 NmNode

- **对象类型**：节点
- **值类型**：枚举
- **默认值**：No
- **有效值**：No、Yes
- **意思**：该节点是否参与网络管理。
- **用途**：标记哪些节点需要 NM 状态机。
- **原理**：设为 Yes 后，CANoe 为该节点启用 NM 发送、接收、睡眠和唤醒逻辑。

### 5.5 NmMessage

- **对象类型**：报文
- **值类型**：枚举
- **默认值**：No
- **有效值**：No、Yes
- **意思**：该报文是否作为网络管理报文。
- **用途**：让工具识别哪些报文属于 NM。
- **原理**：NM 层按该标记解析报文内容、维护 NM 状态，并把它与节点 NM 地址关联。

### 5.6 NmStationAddress

- **对象类型**：节点
- **值类型**：十六进制
- **默认值**：0
- **有效值**：>= 0x00
- **意思**：节点的网络管理地址。
- **用途**：用于计算该节点 NM 报文 ID。
- **原理**：

  ```text
  NM message ID = NmBaseAddress + NmStationAddress
  ```

  例如 `NmStationAddress = 18`、`NmBaseAddress = 0x400`，则 NM 报文 ID 为 `0x412`。

### 5.7 NmCAN

- **对象类型**：节点
- **值类型**：整数
- **默认值**：0
- **有效值**：1 = CAN 通道 1；2 = CAN 通道 2
- **意思**：节点 NM 使用的 CAN 通道。
- **用途**：多通道环境下指定 NM 报文走哪个通道。
- **原理**：NM 发送/接收绑定到指定通道；该属性主要用于旧版 CANoe 或兼容模式。

### 5.8 GenNWMSleepTime

- **对象类型**：网络
- **值类型**：整数
- **单位**：ms
- **默认值**：0
- **有效值**：>= 0
- **意思**：网络级等待 SleepRequest 的时间。
- **用途**：当所有节点使用相同睡眠等待时间时统一配置。
- **原理**：NM 状态机在进入睡眠前等待该时间；若节点有独立配置，则节点级属性可覆盖。

### 5.9 GenNodSleepTime

> 属性名按 Vector 文档原文记录（`GenNodSleepTime`），部分数据库/OEM 规范中可能看到不同拼写。

- **对象类型**：节点
- **值类型**：整数
- **单位**：ms
- **默认值**：0
- **有效值**：>= 0
- **意思**：节点级等待 SleepRequest 的时间。
- **用途**：当不同节点需要不同睡眠等待时间时使用。
- **原理**：该值大于 0 时，对该节点覆盖网络级 `GenNWMSleepTime`。

### 5.10 NWM-WakeupAllowed

- **对象类型**：信号
- **值类型**：枚举
- **默认值**：No
- **有效值**：No、Yes
- **意思**：该信号是否允许影响网络管理唤醒。
- **用途**：对于不应该触发 NM 唤醒的信号设为 No。
- **原理**：NM 层在信号更新时检查该属性；No 的信号即使变化也不会引起唤醒/影响 NM 状态。

---

## 6. 工具特定属性（Tool-specific）

这些属性主要影响 CANoe/CANalyzer 的仿真和测量行为，不直接改变 CAN 帧格式。

### 6.1 NodeLayerModules

- **对象类型**：节点
- **值类型**：字符串
- **默认值**：`""`
- **工具**：CANoe
- **意思**：CANoe 为该节点加载的节点层 DLL 列表。
- **用途**：启用 OSEK TP、OSEK NM 等节点层模块。
- **原理**：多个模块用逗号分隔，例如：

  ```text
  OSEK_TP.DLL, OSEKNM.DLL
  ```

  CANoe 启动节点时按列表加载对应 DLL，从而扩展节点的协议能力。

### 6.2 BusType

- **对象类型**：网络
- **值类型**：字符串
- **默认值**：`""`
- **工具**：CANoe、CANalyzer
- **意思**：网络类型，例如 CAN、LIN 等。
- **用途**：让工具知道当前数据库对应哪种总线。
- **原理**：CANoe/CANalyzer 根据 BusType 选择通道、驱动、协议解析器和显示方式。

### 6.3 ProtocolType

- **对象类型**：网络
- **值类型**：字符串
- **默认值**：取决于总线选项
- **有效值**：J1939、CANopen、AFDX、ARINC825、CANaerospace 等
- **工具**：CANoe、CANalyzer
- **意思**：网络使用的上层协议类型。
- **用途**：激活对应的协议解析和功能。
- **原理**：工具加载相应协议选项/插件；需要安装对应选项并具备有效 License。

### 6.4 ECU

- **对象类型**：节点
- **值类型**：字符串
- **默认值**：`""`
- **工具**：CANoe
- **意思**：节点所属的 ECU 名称。
- **用途**：当同一个 ECU 在多个总线上有多个节点时，用该属性把它们归为同一 ECU。
- **原理**：CANoe 以 ECU 为单位组织节点/接口；该属性用于多总线 ECU 的逻辑分组。

### 6.5 CANoeStartDelay

- **对象类型**：节点
- **值类型**：整数
- **单位**：ms
- **默认值**：0
- **工具**：CANoe
- **意思**：测量开始后，该节点保持完全被动的持续时间。
- **用途**：模拟节点启动延迟，避免测量一开始就参与通信。
- **原理**：在该时间内节点不响应外部影响，也不主动发送；时间到后才进入正常仿真状态。

### 6.6 CANoeDrift

- **对象类型**：节点
- **值类型**：整数
- **单位**：ms
- **默认值**：0
- **工具**：CANoe
- **意思**：节点定时器的漂移比例/修正量。
- **用途**：模拟不同 ECU 的时钟偏差，使周期发送产生可控漂移。
- **原理**：CANoe 按该值延长或缩短节点定时器周期，从而改变报文实际发送节拍。

### 6.7 CANoeJitterMin

- **对象类型**：节点
- **值类型**：整数
- **单位**：ms
- **默认值**：0
- **工具**：CANoe
- **意思**：节点定时器抖动区间的下限。
- **用途**：与 `CANoeJitterMax` 一起模拟周期抖动。
- **原理**：定时器实际触发时间在 `[CANoeJitterMin, CANoeJitterMax]` 区间内均匀分布，从而产生非固定周期。

### 6.8 CANoeJitterMax

- **对象类型**：节点
- **值类型**：整数
- **单位**：ms
- **默认值**：0
- **工具**：CANoe
- **意思**：节点定时器抖动区间的上限。
- **用途**：与 `CANoeJitterMin` 配合模拟发送抖动。
- **原理**：CANoe 在上下限之间按均匀分布随机化定时器触发时刻；上下限相等或为 0 时通常表示无抖动。

---

## 7. 关键组合规则

### 7.1 报文发送类型和信号发送类型共同决定行为

`GenMsgSendType` 和 `GenSigSendType` 不是互相独立的简单开关，而是组合生效：

- 报文级决定基础模式（周期、事件、激活等）
- 信号级决定触发条件（变化、写入、激活等）
- `GenMsgCycleTime` / `GenMsgCycleTimeFast` 提供周期参数
- `GenMsgNrOfRepetition` 提供重复次数
- `GenSigInactiveValue` 提供 IfActive 的判断阈值

因此不要只看单个属性判断发送行为，必须结合报文内所有信号。

### 7.2 原始值 vs 物理值

以下属性使用**原始值（raw value）**：

```text
GenSigInactiveValue
GenSigStartValue
```

不要直接填物理值。物理值需要按 DBC 信号定义换算：

```text
raw = (physical - offset) / factor
```

### 7.3 版本字段使用 BCD

`VersionYear`、`VersionMonth`、`VersionWeek`、`VersionDay`、`VersionNumber` 使用 BCD 编码。填写时不要按普通十进制直接理解。

### 7.4 TP 属性是配套使用的

TP 配置通常不是单个属性决定，而是组合使用：

- `TpAddressingMode` 决定寻址方式
- `TpBaseAddress` / `TpNodeBaseAddress` 决定基地址
- `TpTxIdentifier` / `TpRxIdentifier` / `TpRxMask` 决定显式 ID 和过滤
- `TpSTMin` / `TpBlockSize` / `TpUseFC` 决定流控行为
- `TpTargetAddress` / `TpAddressExtension` 决定扩展地址

修改其中一个属性时，必须检查整组 TP 配置是否仍然一致。

### 7.5 NM 报文 ID 的计算

典型计算关系：

```text
NM message ID = NmBaseAddress + NmStationAddress
```

`NmMessageCount` 用于确定预留的 NM ID 范围，防止与普通应用报文冲突。

---

## 8. 使用建议

1. **先确认 IL 类型**：不同 IL（CANoe IL、CAPL 生成器 IL、OEM 自定义 IL）支持的属性集合和枚举值不同。
2. **不要随意添加未定义属性**：DBC 中通过 `BA_` 使用的属性必须先用 `BA_DEF_` 定义，否则 CANdb++ 会报 `Undefined attribute`。
3. **保持属性定义顺序正确**：`BA_DEF_` / `BA_DEF_DEF_` / `BA_` 应位于 `VAL_` 之前，避免 CANdb++ 语法错误。
4. **保持 DBC 与 ECU 实现一致**：DBC 属性只是配置描述，ECU 固件必须实现相同的周期、超时、TP、NM 行为。
5. **注意属性名大小写和 OEM 差异**：属性名通常区分大小写；部分 OEM 会扩展或改名，例如 `GenMsgNrOfRepetition` / `GenMsgNrOfRepetitions`。

---

## 9. 快速索引

| 分类 | 属性 |
|---|---|
| 一般 | `Manufacturer`、`Baudrate`、`DBName`、`VersionYear`、`VersionMonth`、`VersionWeek`、`VersionDay`、`VersionNumber` |
| 交互层 | `ILUsed`、`IlTxTimeout`、`GenMsgILSupport`、`GenMsgSendType`、`GenSigSendType`、`GenMsgDelayTime`、`GenMsgCycleTime`、`GenMsgCycleTimeFast`、`GenMsgCycleTimeActive`、`GenMsgStartDelayTime`、`GenMsgNrOfRepetition(s)`、`GenSigCycleTime`、`GenSigCycleTimeActive`、`GenSigInactiveValue`、`GenSigStartValue`、`GenSigTimeoutTime`、`GenSigTimeoutMsg`、`GenMsgFastOnStart` |
| TP/诊断 | `TpBaseAddress`、`TpNodeBaseAddress`、`TpTxIdentifier`、`TpRxIdentifier`、`TpRxMask`、`TpCanBus`、`TpTxAdrMode`、`TpAddressExtension`、`TpSTMin`、`TpBlockSize`、`TpAddressingMode`、`TpTargetAddress`、`TpUseFC`、`DiagRequest`、`DiagResponse`、`DiagState`、`DiagConnection`、`DiagUUDTResponse`、`DiagStationAddress` |
| 网络管理 | `NmType`、`NmBaseAddress`、`NmMessageCount`、`NmNode`、`NmMessage`、`NmStationAddress`、`NmCAN`、`GenNWMSleepTime`、`GenNodSleepTime`、`NWM-WakeupAllowed`、`NmAsrBaseAddress`、`NmAsrMessageCount`、`NmAsrNode`、`NmAsrNodeIdentifier`、`NmAsrMessage`、`NmAsrCanMsgCycleTime`、`NmAsrCanMsgReducedTime`、`NmAsrCanMsgCycleOffset`、`NmAsrRepeatMessageTime`、`NmAsrTimeoutTime`、`NmAsrWaitBusSleepTime` |
| 工具特定 | `NodeLayerModules`、`BusType`、`ProtocolType`、`ECU`、`CANoeStartDelay`、`CANoeDrift`、`CANoeJitterMin`、`CANoeJitterMax` |

---

## 10. 补充属性（本次核对后新增）

> 说明：以下属性中，`Baudrate`、`DBName` 来自 CANdb++ 帮助和 Vector 工具链通用属性；`NmAsr*`、`IlTxTimeout`、`GenSigCycleTime`、`GenSigCycleTimeActive`、`DiagState`、`DiagConnection`、`DiagUUDTResponse` 属于 Vector AUTOSAR NM、IL、诊断扩展属性的常见命名。不同 CANdb++/CANoe 版本和 OEM 属性包的对象类型、值类型可能略有差异，最终应以实际 DBC 中的 `BA_DEF_` 定义为准。

### 10.1 一般/网络补充属性

#### Baudrate

- **对象类型**：网络（Network）
- **值类型**：整数
- **单位**：bit/s
- **默认值**：常见为空或 0，具体由项目定义
- **意思**：CAN 网络的波特率，例如 500000 表示 500 kbit/s。
- **用途**：CANoe/CANalyzer 配置通道波特率、采样点、同步跳转宽度等总线时序参数。
- **原理**：CAN 控制器按该波特率对总线采样；所有节点必须使用一致的波特率和时序参数，否则会出现位错误、错误帧或无法通信。该属性不改变 CAN 帧内容，只影响物理层/链路层时序配置。
- **命名注意**：DBC 属性名区分大小写。常见写法有 `Baudrate` 和 `BaudRate`，以项目实际 `BA_DEF_` 为准；二者不是同一个属性名。

#### DBName

- **对象类型**：网络（Network）
- **值类型**：字符串
- **默认值**：`""`
- **意思**：数据库/网络名称。
- **用途**：让 DBC 数据库保存一个独立于文件名的网络名称，并允许在 CANdb++ 的 Network 对话框中修改网络名。
- **原理**：如果 DBC 中存在网络级字符串属性 `DBName`，CANdb++ 会把网络名称与 `DBName` 关联并保存；CANoe/CANalyzer 也使用 `DBName` 作为数据库名称。若不存在 `DBName`，CANoe/CANalyzer 通常使用 DBC 文件名作为数据库名称，CANdb++ 中修改网络名可能无法保存。
- **与 MDC 的区别**：在 `.mdc` 原生数据库中该属性没有实际意义；它主要用于 DBC 文件。

### 10.2 交互层补充属性

#### IlTxTimeout

- **对象类型**：报文（Message，常见定义）
- **值类型**：整数
- **单位**：ms
- **默认值**：0
- **有效值**：>= 0
- **意思**：交互层发送超时时间。
- **用途**：监控报文发送是否在指定时间内完成/获得确认，用于发现发送失败或总线异常。
- **原理**：IL 在发起报文发送时启动定时器；如果在 `IlTxTimeout` 内没有收到预期的发送确认或完成事件，则触发发送超时处理。值为 0 通常表示不启用发送超时监控。

#### GenSigCycleTime

- **对象类型**：信号（Signal）
- **值类型**：整数
- **单位**：ms
- **默认值**：0
- **有效值**：>= 0
- **意思**：信号级周期发送时间。
- **用途**：在信号级 IL 调度中定义信号的周期发送节拍。
- **原理**：IL 以信号为调度单位时，按 `GenSigCycleTime` 决定该信号的发送周期；报文实际发送周期可能由报文内多个信号的周期和 `GenMsgSendType`/`GenSigSendType` 共同决定。CANdb++ 在启用用户自定义发送类型属性时，可能不允许直接编辑该 Vector 工具链系统属性。

#### GenSigCycleTimeActive

- **对象类型**：信号（Signal）
- **值类型**：整数
- **单位**：ms
- **默认值**：0
- **有效值**：>= 0
- **意思**：信号在激活状态下的周期发送时间。
- **用途**：用于信号级 IL 的激活/快速发送阶段。
- **原理**：当信号处于激活状态时，IL 使用 `GenSigCycleTimeActive` 作为发送周期；与 `GenSigInactiveValue`、`GenSigSendType` 配合决定信号何时进入/退出激活状态。

### 10.3 诊断补充属性

#### DiagState

- **对象类型**：报文（Message）
- **值类型**：枚举
- **默认值**：No
- **有效值**：No、Yes
- **意思**：标识该报文属于诊断状态/诊断 TP 相关报文。
- **用途**：诊断配置和 ECU 导出时识别诊断 TP 报文；CANdb++ 导出 ECU 时，`DiagState`、`DiagRequest`、`DiagResponse` 中任一为 Yes 的报文会作为 TP 报文处理。
- **原理**：诊断层/TP 层按该标记把报文纳入诊断连接和传输协议的状态管理，而不是当作普通应用报文。

#### DiagConnection

- **对象类型**：报文（Message；部分 OEM 可能定义为网络或字符串属性）
- **值类型**：字符串或枚举，具体以属性定义为准
- **意思**：把报文绑定到某个诊断连接/诊断通道。
- **用途**：在存在多条诊断链路（例如多个 CAN 通道、多个诊断协议或多 ECU 诊断）时区分报文所属连接。
- **原理**：诊断运行时按连接名称选择对应的 TP 参数、诊断地址、ECU 上下文和回调；同一条物理报文可因连接不同而采用不同的诊断处理策略。

#### DiagUUDTResponse

- **对象类型**：报文（Message）
- **值类型**：枚举
- **默认值**：No
- **有效值**：No、Yes
- **意思**：标识该报文是 UUDT（Unsolicited Unacknowledged Data Transfer，非请求/无确认数据传输）诊断响应。
- **用途**：让诊断层识别没有对应诊断请求的主动诊断响应，并交给 UUDT 处理逻辑。
- **原理**：普通诊断响应通常与请求配对；UUDT 响应不依赖请求上下文，诊断层按报文/连接配置直接分发到 UUDT 接收处理。

### 10.4 AUTOSAR 网络管理补充属性（NmAsr*）

AUTOSAR NM 属性用于描述 AUTOSAR CanNm 状态机参数。不同 OEM 的属性包可能把部分参数定义在网络或节点上，本文按常见 Vector 定义说明。

#### NmAsrBaseAddress

- **对象类型**：网络（Network）
- **值类型**：十六进制
- **单位**：CAN 报文 ID
- **意思**：AUTOSAR NM 报文 CAN ID 的基地址。
- **用途**：确定 NM 报文 ID 区域的起点。
- **原理**：NM 层按基地址和节点标识/配置计算实际 NM 报文 ID；同一网络内所有节点的计算规则必须一致。

#### NmAsrMessageCount

- **对象类型**：网络（Network）
- **值类型**：整数
- **有效值**：>= 1
- **意思**：AUTOSAR NM 报文可用/预留的 CAN ID 数量。
- **用途**：确定 NM 报文 ID 范围，避免与普通应用报文冲突。
- **原理**：从 `NmAsrBaseAddress` 开始预留一定数量的 ID；实际使用数量由节点数和 OEM 分配规则决定。

#### NmAsrNode

- **对象类型**：节点（Node）
- **值类型**：枚举
- **默认值**：No
- **有效值**：No、Yes
- **意思**：该节点是否参与 AUTOSAR 网络管理。
- **用途**：标记需要运行 AUTOSAR NM 状态机的节点。
- **原理**：设为 Yes 后，CANoe 为该节点启用 AUTOSAR NM 的发送、接收、Repeat Message、Ready Sleep、Bus Sleep 等状态处理。

#### NmAsrNodeIdentifier

- **对象类型**：节点（Node）
- **值类型**：整数/十六进制（常见为 0...255）
- **意思**：AUTOSAR NM 节点标识（Node ID）。
- **用途**：在 NM 报文中标识节点，并参与 NM 报文 ID 或报文内容的计算。
- **原理**：AUTOSAR CanNm 使用 Node ID 区分网络中各节点；该标识通常写入 NM PDU 的 Source Node Identifier 字段，部分配置也用它推导 CAN ID。网络中必须唯一。

#### NmAsrMessage

- **对象类型**：报文（Message）
- **值类型**：枚举
- **默认值**：No
- **有效值**：No、Yes
- **意思**：该报文是否作为 AUTOSAR NM 报文。
- **用途**：让 NM 层识别哪些报文参与 AUTOSAR NM 状态机。
- **原理**：标记为 Yes 的报文按 NM 周期、Repeat Message、超时和睡眠规则处理；普通报文不参与 NM 状态迁移。

#### NmAsrCanMsgCycleTime

- **对象类型**：网络（Network）
- **值类型**：整数
- **单位**：ms
- **默认值**：0
- **有效值**：>= 0
- **意思**：AUTOSAR NM 报文在正常工作状态下的周期发送时间。
- **用途**：配置 NM 报文的正常周期。
- **原理**：NM 状态机在 Network Mode 的正常阶段按该周期发送 NM 报文；具体周期可受 Repeat Message、Reduced Time 等状态影响。

#### NmAsrCanMsgReducedTime

- **对象类型**：网络（Network）
- **值类型**：整数
- **单位**：ms
- **默认值**：0
- **有效值**：>= 0
- **意思**：AUTOSAR NM 报文缩短后的发送周期。
- **用途**：在需要加快 NM 报文发送的阶段（常见于 Repeat Message/唤醒后的快速发送阶段）使用较短周期。
- **原理**：NM 状态机在特定状态下用 `NmAsrCanMsgReducedTime` 替代正常 `NmAsrCanMsgCycleTime`，使节点更快地宣布网络状态、加快网络收敛。具体生效状态由 OEM NM 配置决定。

#### NmAsrCanMsgCycleOffset

- **对象类型**：网络（Network）
- **值类型**：整数
- **单位**：ms
- **默认值**：0
- **有效值**：>= 0
- **意思**：NM 报文周期发送的相位偏移。
- **用途**：错开不同节点的 NM 发送时刻，降低总线瞬时负载和报文碰撞风险。
- **原理**：NM 定时器在启动时先等待偏移时间，再按周期发送；偏移量通常小于周期时间。

#### NmAsrRepeatMessageTime

- **对象类型**：网络（Network）
- **值类型**：整数
- **单位**：ms
- **默认值**：0
- **有效值**：>= 0
- **意思**：AUTOSAR NM Repeat Message 状态的持续时间。
- **用途**：控制节点在启动/唤醒后保持快速 NM 发送的时间。
- **原理**：进入 Repeat Message 状态后启动定时器，时间到后根据 NM 状态机迁移到 Normal Operation 或 Ready Sleep；期间通常使用较快的 NM 周期。

#### NmAsrTimeoutTime

- **对象类型**：网络（Network）
- **值类型**：整数
- **单位**：ms
- **默认值**：0
- **有效值**：>= 0
- **意思**：AUTOSAR NM 的超时时间（NM Timeout Timer）。
- **用途**：判断网络是否仍然活跃。
- **原理**：节点在 Network Mode 中启动 NM 超时定时器；每收到有效 NM 报文就刷新定时器。若超过 `NmAsrTimeoutTime` 未收到 NM 报文，则离开 Network Mode 并进入 Prepare Bus Sleep 等状态。

#### NmAsrWaitBusSleepTime

- **对象类型**：网络（Network）
- **值类型**：整数
- **单位**：ms
- **默认值**：0
- **有效值**：>= 0
- **意思**：AUTOSAR NM 等待进入 Bus Sleep 的时间。
- **用途**：在 Prepare Bus Sleep 状态下等待总线安静，确保所有节点都准备好睡眠。
- **原理**：NM 状态机在 Prepare Bus Sleep 中启动该定时器；时间到且没有唤醒事件后进入 Bus Sleep，停止 NM 发送并进入低功耗。

### 10.5 关于“BusType already exists in the database and wasn't imported”

这不是一个属性名，而是 CANdb++ 导入属性定义时的提示信息：

```text
BusType already exists in the database and wasn't imported
```

含义是：当前 DBC 中已经存在名为 `BusType` 的属性，因此导入新的属性定义时跳过了同名属性，避免重复定义。

处理原则：

- 不需要重复创建 `BusType`。
- 检查现有 `BusType` 的对象类型和值类型是否与导入源一致。
- 如果现有定义不兼容，应先确认是否删除/修改旧定义，再重新导入。
- 属性名相同但对象类型或值类型不同，会导致赋值失败或 CANdb++ 报错。

---

## 11. 本次属性核对清单

| 序号 | 属性 | 文档状态 |
|---:|---|---|
| 1 | `Manufacturer` | 原有，见 2.1 |
| 2 | `Baudrate` | **本次补充，见 10.1** |
| 3 | `DBName` | **本次补充，见 10.1** |
| 4 | `NmType` | 原有，见 5.1 |
| 5 | `NmAsrWaitBusSleepTime` | **本次补充，见 10.4** |
| 6 | `NmAsrTimeoutTime` | **本次补充，见 10.4** |
| 7 | `NmAsrRepeatMessageTime` | **本次补充，见 10.4** |
| 8 | `NmAsrMessageCount` | **本次补充，见 10.4** |
| 9 | `NmAsrCanMsgCycleTime` | **本次补充，见 10.4** |
| 10 | `NmAsrBaseAddress` | **本次补充，见 10.4** |
| 11 | `IlTxTimeout` | **本次补充，见 10.2** |
| 12 | `BusType` | 原有，见 6.2；导入提示见 10.5 |
| 13 | `NmAsrNodeIdentifier` | **本次补充，见 10.4** |
| 14 | `NmAsrNode` | **本次补充，见 10.4** |
| 15 | `NmAsrCanMsgReducedTime` | **本次补充，见 10.4** |
| 16 | `NmAsrCanMsgCycleOffset` | **本次补充，见 10.4** |
| 17 | `ILUsed` | 原有，见 3.1 |
| 18 | `DiagRequest` | 原有，见 4.14 |
| 19 | `DiagResponse` | 原有，见 4.15 |
| 20 | `DiagState` | **本次补充，见 10.3** |
| 21 | `DiagConnection` | **本次补充，见 10.3** |
| 22 | `GenMsgSendType` | 原有，见 3.3 |
| 23 | `GenMsgILSupport` | 原有，见 3.2 |
| 24 | `NmAsrMessage` | **本次补充，见 10.4** |
| 25 | `GenMsgDelayTime` | 原有，见 3.5 |
| 26 | `GenMsgNrOfRepetition` | 原有，见 3.10 |
| 27 | `GenMsgStartDelayTime` | 原有，见 3.9 |
| 28 | `DiagUUDTResponse` | **本次补充，见 10.3** |
| 29 | `GenMsgCycleTimeFast` | 原有，见 3.7 |
| 30 | `GenMsgCycleTime` | 原有，见 3.6 |
| 31 | `GenSigSendType` | 原有，见 3.4 |
| 32 | `GenSigCycleTime` | **本次补充，见 10.2** |
| 33 | `GenSigStartValue` | 原有，见 3.12 |
| 34 | `GenSigInactiveValue` | 原有，见 3.11 |
| 35 | `GenSigCycleTimeActive` | **本次补充，见 10.2** |
| 36 | `GenSigTimeoutTime` | 原有，见 3.13 |

---

## 12. 参考依据

本文属性清单和基础定义参考 CANdb++ 3.1 SP4 自带帮助中的 **Vector Tool Chain Attributes** 页面，并结合 Vector 工具链中 IL、TP、NM、诊断的通用工作机制进行解释。`DBName`、`Baudrate` 等属性行为参考 CANdb++ 帮助中关于 DBC 导出、网络名称和波特率的说明。

`NmAsr*`、`IlTxTimeout`、`GenSigCycleTime`、`GenSigCycleTimeActive`、`DiagState`、`DiagConnection`、`DiagUUDTResponse` 等属性在不同 CANdb++/CANoe 版本和 OEM 属性包中可能存在定义差异；实际项目请以对应版本的属性定义、DBC 中的 `BA_DEF_` 以及 OEM 规范为准。
