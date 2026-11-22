# S32K148 启动流程与链接文件（ICF）逐行详解

> 工程：`AUTOSAR_S32K1X_2/S32K1X_Project`
> 器件：NXP S32K148（Cortex-M4F，1.5 MB Flash，256 KB SRAM）
> 工具链：IAR Embedded Workbench for ARM（IAR C/C++ Compiler）
> 相关文件：
> - `StartUp_Code/startup_S32K148.S`（向量表 + 复位入口 + 默认异常处理）
> - `StartUp_Code/devices/startup.c`（`init_data_bss()`）
> - `StartUp_Code/devices/S32K148/startup/system_S32K148.c`（`SystemInit()`）
> - `StartUp_Code/devices/S32K148/startup/system_S32K148.h`（`DISABLE_WDOG` 等）
> - `Linker_Files/S32K148_256_flash.icf`（IAR 链接文件）
>
> 本文中的地址均取自当前工程 `Debug/List/S32K1X_Project.map` 与 `Debug/Exe/S32K1X_Project.out`（Debug 构建）。

---

## 目录

1. [总览：从上电到 main 的完整执行路径](#1-总览从上电到-main-的完整执行路径)
2. [向量表 `.intvec` 逐行详解](#2-向量表-intvec-逐行详解)
3. [Flash 配置字段 `FlashConfig`](#3-flash-配置字段-flashconfig)
4. [`Reset_Handler` 逐行详解](#4-reset_handler-逐行详解)
5. [`SystemInit()` 逐行详解](#5-systeminit-逐行详解)
6. [`init_data_bss()` 逐行详解](#6-init_data_bss-逐行详解)
7. [进入 `main()`](#7-进入-main)
8. [链接文件 `S32K148_256_flash.icf` 逐行详解](#8-链接文件-s32k148_256_flashicf-逐行详解)
9. [链接结果：实际内存布局](#9-链接结果实际内存布局)
10. [本文分析发现的关键问题与修复建议](#10-本文分析发现的关键问题与修复建议)
11. [附录：常用地址/符号速查](#11-附录常用地址符号速查)
12. [一句话总结](#12-一句话总结)

---

## 1. 总览：从上电到 `main` 的完整执行路径

### 1.1 Cortex-M4 硬件复位行为

S32K148 上电复位（POR）、引脚复位、看门狗复位、软件复位等发生后，Cortex-M4 内核会执行两个固定动作：

1. 从地址 `0x0000_0000` 读取 32 位值装入 **MSP（主栈指针）**；
2. 从地址 `0x0000_0004` 读取 32 位值装入 **PC（程序计数器）**，并从该地址开始取指。

当前镜像中：

| 地址 | 内容 | 含义 |
|---|---|---|
| `0x0000_0000` | `0x2001_F000` | 初始栈顶 = `sfe(CSTACK)` |
| `0x0000_0004` | `0x0000_1329` | `Reset_Handler`（Thumb 地址，实际代码在 `0x1328`） |

所以硬件复位后，**第一条被执行的用户代码是 `Reset_Handler`（`0x1328`）**，而不是 `main`。

> 说明：ELF 头里的 `Entry point` 是 `0x1561`（IAR 库的 `__iar_program_start`），但 Cortex-M 硬件复位**不使用** ELF entry，而是使用向量表第 2 项。只有某些调试器/加载器在特定设置下才会跳 ELF entry。真实硬件运行永远走向量表。

### 1.2 实际执行链（当前 Debug 构建）

```text
上电/复位
  │
  ├─ 硬件：SP = [0x0000_0000] = 0x2001_F000
  │        PC = [0x0000_0004] = 0x0000_1329 (Reset_Handler)
  │
  ├─ Reset_Handler (0x1328)
  │    ├─ CPSID I                    关全局中断
  │    ├─ R1..R12 清零
  │    ├─ [可选] SystemInit()        条件：未定义 __NO_SYSTEM_INIT
  │    │      ├─ FPU 使能 (CPACR)
  │    │      └─ 关闭看门狗 WDOG
  │    ├─ [可选] ECC RAM 初始化      条件：定义了 START_FROM_FLASH
  │    │      把 0x1FFE_0000 ~ 0x2001_EFFF 全部写 0
  │    ├─ SP = sfe(CSTACK) = 0x2001_F000
  │    ├─ init_data_bss()  (0x1058)
  │    │      ├─ 复制 .data / __CODE_ROM / .customSection
  │    │      ├─ 清 .bss
  │    │      └─ 复制向量表到 RAM，设置 VTOR
  │    ├─ CPSIE I                    开全局中断
  │    └─ BL main (0x1142)
  │
  └─ main()
       ├─ Det_Init()
       ├─ Mcu_Init(&McuModuleConfiguration)
       ├─ Mcu_InitClock(McuClockSettingConfig_0)
       ├─ 等待 PLL 锁定
       ├─ Mcu_DistributePllClock()
       ├─ Mcu_SetMode(McuModeSettingConf_0)
       ├─ Port_Init(&PortConfigSet)
       └─ for(;;) 翻转 PTE20/PTE21
```

### 1.3 参与启动的关键地址（Debug 构建）

| 符号 | 地址 | 说明 |
|---|---:|---|
| `__vector_table` / `__Vectors` | `0x0000_0000` | `.intvec`，256 个 32 位向量，共 0x400 字节 |
| `FlashConfig` | `0x0000_0400` | 16 字节 Flash 保护/安全配置 |
| 代码区起点 | `0x0000_0410` | `TEXT_region` 的 `m_text_start` |
| `init_data_bss` | `0x0000_1058` | C 函数 |
| `main` | `0x0000_1142` | 用户 `main` |
| `Reset_Handler` | `0x0000_1328` | 复位入口 |
| `__iar_data_init3` | `0x0000_13A8` | IAR 库：执行 init table（当前**未被调用**） |
| `?main` / `__cmain` | `0x0000_1414` | IAR 库启动（当前**未被调用**） |
| `__iar_program_start` | `0x0000_1560` | ELF entry（硬件复位**不使用**） |
| RW 初值源（init table） | `0x0000_1540` | 16 字节 |
| `RW`（已初始化数据） | `0x1FFE_0400` | 16 字节块 |
| `ZI`（零初始化） | `0x2000_0000` | 9 字节 |
| `CSTACK` | `0x2001_EC00`~`0x2001_EFFF` | 1 KB 栈，栈顶 `0x2001_F000` |
| RAM 向量表 | `0x1FFE_0000` | 0x400 字节 |

---

## 2. 向量表 `.intvec` 逐行详解

`startup_S32K148.S` 第 40~80 行附近的汇编指令先建立段与符号：

| 行 | 代码 | 作用 / 原理 |
|---:|---|---|
| 41 | `MODULE ?cstartup` | IAR 模块名，仅供调试/链接识别，不产生代码。 |
| 44 | `SECTION CSTACK:DATA:NOROOT(3)` | 声明一个名为 `CSTACK` 的段：`DATA` 表示数据段，`NOROOT` 表示即使未被引用也保留，`(3)` 表示 2^3 = 8 字节对齐（AAPCS 要求栈 8 字节对齐）。`sfe(CSTACK)` 即该段结束地址，用作初始 SP。 |
| 46 | `SECTION .intvec:CODE:ROOT(2)` | 声明向量表段 `.intvec`：`CODE` 表示只读代码段，`ROOT` 表示链接器必须保留（不能因为“未引用”而丢弃），`(2)` 表示 2^2 = 4 字节对齐（每个向量 4 字节）。 |
| 48 | `EXTERN main` | 声明外部符号 `main`，告诉汇编器该符号在别的目标文件里。 |
| 49 | `EXTERN SystemInit` | 声明外部符号 `SystemInit`。 |
| 50 | `EXTERN init_data_bss` | 声明外部符号 `init_data_bss`。 |
| 51 | `PUBLIC __vector_table` | 导出符号，供链接文件/调试器/其它模块使用。 |
| 52 | `PUBLIC __vector_table_0x1c` | 导出 0x1C 处标记，Flash 编程工具常用它定位向量表。 |
| 53 | `PUBLIC __Vectors` | 导出向量表起始别名。 |
| 54 | `PUBLIC __Vectors_End` | 导出向量表结束。 |
| 55 | `PUBLIC __Vectors_Size` | 导出向量表大小。 |
| 57 | `DATA` | 切换到数据区（只读常量也放在这里，最终由 `.intvec:CODE` 决定段属性）。 |

### 2.1 向量表逐项

> 规则：第 `n` 项位于偏移 `4*n`。`DCD` = Define Constant Double-word，即在此处放入一个 32 位常量（地址或数值）。
> 第 0 项是初始 SP，不是函数；第 1 项是复位入口；第 2~15 项是 Cortex-M4 内核异常；第 16 项起是外设中断 IRQ0、IRQ1……

**内核异常向量（0~15）：**

| 索引 | 偏移 | 写入符号 | 用途/说明 |
|---:|---:|---|---|
| 0 | 0x000 | `sfe(CSTACK)` | 初始主栈指针 MSP = CSTACK 段结束地址（`0x2001_F000`）。 |
| 1 | 0x004 | `Reset_Handler` | 复位入口。 |
| 2 | 0x008 | `NMI_Handler` | 不可屏蔽中断。当前实现为 `B .`（死循环）。 |
| 3 | 0x00C | `HardFault_Handler` | 硬件错误（总线错误升级、未定义异常等最终都到这里）。当前 `B .`。 |
| 4 | 0x010 | `MemManage_Handler` | MPU 访问违规。当前 `B .`。 |
| 5 | 0x014 | `BusFault_Handler` | 总线错误（非法地址、外设写保护等）。当前 `B .`。 |
| 6 | 0x018 | `UsageFault_Handler` | 用法错误（未定义指令、非对齐访问被禁止等）。当前 `B .`。 |
| 7 | 0x01C | `0` | 保留（Cortex-M4 保留向量）。 |
| 8 | 0x020 | `0` | 保留。 |
| 9 | 0x024 | `0` | 保留。 |
| 10 | 0x028 | `0` | 保留。 |
| 11 | 0x02C | `SVC_Handler` | 超级用户调用（SVCall）。当前 `B .`。 |
| 12 | 0x030 | `DebugMon_Handler` | 调试监控异常。当前 `B .`。 |
| 13 | 0x034 | `0` | 保留。 |
| 14 | 0x038 | `PendSV_Handler` | 可挂起系统调用（RTOS 常用）。当前 `B .`。 |
| 15 | 0x03C | `SysTick_Handler` | 系统节拍定时器中断。当前 `B .`。 |

**外设中断向量（IRQ0 ~ IRQ146，对应索引 16 ~ 162）：**

| IRQn | Index | Offset | Symbol | Description |
|---:|---:|---:|---|---|
| 0 | 16 | 0x040 | `DMA0_IRQHandler` | DMA channel 0 transfer complete |
| 1 | 17 | 0x044 | `DMA1_IRQHandler` | DMA channel 1 transfer complete |
| 2 | 18 | 0x048 | `DMA2_IRQHandler` | DMA channel 2 transfer complete |
| 3 | 19 | 0x04C | `DMA3_IRQHandler` | DMA channel 3 transfer complete |
| 4 | 20 | 0x050 | `DMA4_IRQHandler` | DMA channel 4 transfer complete |
| 5 | 21 | 0x054 | `DMA5_IRQHandler` | DMA channel 5 transfer complete |
| 6 | 22 | 0x058 | `DMA6_IRQHandler` | DMA channel 6 transfer complete |
| 7 | 23 | 0x05C | `DMA7_IRQHandler` | DMA channel 7 transfer complete |
| 8 | 24 | 0x060 | `DMA8_IRQHandler` | DMA channel 8 transfer complete |
| 9 | 25 | 0x064 | `DMA9_IRQHandler` | DMA channel 9 transfer complete |
| 10 | 26 | 0x068 | `DMA10_IRQHandler` | DMA channel 10 transfer complete |
| 11 | 27 | 0x06C | `DMA11_IRQHandler` | DMA channel 11 transfer complete |
| 12 | 28 | 0x070 | `DMA12_IRQHandler` | DMA channel 12 transfer complete |
| 13 | 29 | 0x074 | `DMA13_IRQHandler` | DMA channel 13 transfer complete |
| 14 | 30 | 0x078 | `DMA14_IRQHandler` | DMA channel 14 transfer complete |
| 15 | 31 | 0x07C | `DMA15_IRQHandler` | DMA channel 15 transfer complete |
| 16 | 32 | 0x080 | `DMA_Error_IRQHandler` | DMA error interrupt channels 0-15 |
| 17 | 33 | 0x084 | `MCM_IRQHandler` | FPU sources |
| 18 | 34 | 0x088 | `FTFC_IRQHandler` | FTFC Command complete |
| 19 | 35 | 0x08C | `Read_Collision_IRQHandler` | FTFC Read collision |
| 20 | 36 | 0x090 | `LVD_LVW_IRQHandler` | PMC Low voltage detect interrupt |
| 21 | 37 | 0x094 | `FTFC_Fault_IRQHandler` | FTFC Double bit fault detect |
| 22 | 38 | 0x098 | `WDOG_EWM_IRQHandler` | Single interrupt vector for WDOG and EWM |
| 23 | 39 | 0x09C | `RCM_IRQHandler` | RCM Asynchronous Interrupt |
| 24 | 40 | 0x0A0 | `LPI2C0_Master_IRQHandler` | LPI2C0 Master Interrupt |
| 25 | 41 | 0x0A4 | `LPI2C0_Slave_IRQHandler` | LPI2C0 Slave Interrupt |
| 26 | 42 | 0x0A8 | `LPSPI0_IRQHandler` | LPSPI0 Interrupt |
| 27 | 43 | 0x0AC | `LPSPI1_IRQHandler` | LPSPI1 Interrupt |
| 28 | 44 | 0x0B0 | `LPSPI2_IRQHandler` | LPSPI2 Interrupt |
| 29 | 45 | 0x0B4 | `LPI2C1_Master_IRQHandler` | LPI2C1 Master Interrupt |
| 30 | 46 | 0x0B8 | `LPI2C1_Slave_IRQHandler` | LPI2C1 Slave Interrupt |
| 31 | 47 | 0x0BC | `LPUART0_RxTx_IRQHandler` | LPUART0 Transmit / Receive Interrupt |
| 32 | 48 | 0x0C0 | `Reserved48_IRQHandler` | Reserved Interrupt 48 |
| 33 | 49 | 0x0C4 | `LPUART1_RxTx_IRQHandler` | LPUART1 Transmit / Receive  Interrupt |
| 34 | 50 | 0x0C8 | `Reserved50_IRQHandler` | Reserved Interrupt 50 |
| 35 | 51 | 0x0CC | `LPUART2_RxTx_IRQHandler` | LPUART2 Transmit / Receive  Interrupt |
| 36 | 52 | 0x0D0 | `Reserved52_IRQHandler` | Reserved Interrupt 52 |
| 37 | 53 | 0x0D4 | `Reserved53_IRQHandler` | Reserved Interrupt 53 |
| 38 | 54 | 0x0D8 | `Reserved54_IRQHandler` | Reserved Interrupt 54 |
| 39 | 55 | 0x0DC | `ADC0_IRQHandler` | ADC0 interrupt request. |
| 40 | 56 | 0x0E0 | `ADC1_IRQHandler` | ADC1 interrupt request. |
| 41 | 57 | 0x0E4 | `CMP0_IRQHandler` | CMP0 interrupt request |
| 42 | 58 | 0x0E8 | `Reserved58_IRQHandler` | Reserved Interrupt 58 |
| 43 | 59 | 0x0EC | `Reserved59_IRQHandler` | Reserved Interrupt 59 |
| 44 | 60 | 0x0F0 | `ERM_single_fault_IRQHandler` | ERM single bit error correction |
| 45 | 61 | 0x0F4 | `ERM_double_fault_IRQHandler` | ERM double bit error non-correctable |
| 46 | 62 | 0x0F8 | `RTC_IRQHandler` | RTC alarm interrupt |
| 47 | 63 | 0x0FC | `RTC_Seconds_IRQHandler` | RTC seconds interrupt |
| 48 | 64 | 0x100 | `LPIT0_Ch0_IRQHandler` | LPIT0 channel 0 overflow interrupt |
| 49 | 65 | 0x104 | `LPIT0_Ch1_IRQHandler` | LPIT0 channel 1 overflow interrupt |
| 50 | 66 | 0x108 | `LPIT0_Ch2_IRQHandler` | LPIT0 channel 2 overflow interrupt |
| 51 | 67 | 0x10C | `LPIT0_Ch3_IRQHandler` | LPIT0 channel 3 overflow interrupt |
| 52 | 68 | 0x110 | `PDB0_IRQHandler` | PDB0 interrupt |
| 53 | 69 | 0x114 | `Reserved69_IRQHandler` | Reserved Interrupt 69 |
| 54 | 70 | 0x118 | `Reserved70_IRQHandler` | Reserved Interrupt 70 |
| 55 | 71 | 0x11C | `SAI1_Tx_IRQHandler` | SAI1 Transmit Synchronous interupt (for interrupt controller) |
| 56 | 72 | 0x120 | `SAI1_Rx_IRQHandler` | SAI1 Receive Synchronous interupt (for interrupt controller) |
| 57 | 73 | 0x124 | `SCG_IRQHandler` | SCG bus interrupt request |
| 58 | 74 | 0x128 | `LPTMR0_IRQHandler` | LPTIMER interrupt request |
| 59 | 75 | 0x12C | `PORTA_IRQHandler` | Port A pin detect interrupt |
| 60 | 76 | 0x130 | `PORTB_IRQHandler` | Port B pin detect interrupt |
| 61 | 77 | 0x134 | `PORTC_IRQHandler` | Port C pin detect interrupt |
| 62 | 78 | 0x138 | `PORTD_IRQHandler` | Port D pin detect interrupt |
| 63 | 79 | 0x13C | `PORTE_IRQHandler` | Port E pin detect interrupt |
| 64 | 80 | 0x140 | `SWI_IRQHandler` | Software interrupt |
| 65 | 81 | 0x144 | `QSPI_IRQHandler` | QSPI All interrupts ORed output |
| 66 | 82 | 0x148 | `Reserved82_IRQHandler` | Reserved Interrupt 82 |
| 67 | 83 | 0x14C | `Reserved83_IRQHandler` | Reserved Interrupt 83 |
| 68 | 84 | 0x150 | `PDB1_IRQHandler` | PDB1 interrupt |
| 69 | 85 | 0x154 | `FLEXIO_IRQHandler` | FlexIO Interrupt |
| 70 | 86 | 0x158 | `SAI0_Tx_IRQHandler` | SAI0 Transmit Synchronous interrupt (for interrupt controller) |
| 71 | 87 | 0x15C | `SAI0_Rx_IRQHandler` | SAI0 Receive Synchronous interrupt (for interrupt controller) |
| 72 | 88 | 0x160 | `ENET_TIMER_IRQHandler` | ENET 1588 Timer Interrupt - synchronous |
| 73 | 89 | 0x164 | `ENET_TX_IRQHandler` | ENET Data transfer done |
| 74 | 90 | 0x168 | `ENET_RX_IRQHandler` | ENET Receive Buffer Done for Ring/Queue 0 |
| 75 | 91 | 0x16C | `ENET_ERR_IRQHandler` | ENET Payload receive error. |
| 76 | 92 | 0x170 | `ENET_STOP_IRQHandler` | ENET Graceful stop |
| 77 | 93 | 0x174 | `ENET_WAKE_IRQHandler` | ENET Wake from sleep. |
| 78 | 94 | 0x178 | `CAN0_ORed_IRQHandler` | CAN0 OR'ed [Bus Off OR Transmit Warning OR Receive Warning] |
| 79 | 95 | 0x17C | `CAN0_Error_IRQHandler` | CAN0 Interrupt indicating that errors were detected on the CAN bus |
| 80 | 96 | 0x180 | `CAN0_Wake_Up_IRQHandler` | CAN0 Interrupt asserted when Pretended Networking operation is enabled, and a valid message matches the selected filter criteria during Low Power mode |
| 81 | 97 | 0x184 | `CAN0_ORed_0_15_MB_IRQHandler` | CAN0 OR'ed Message buffer (0-15) |
| 82 | 98 | 0x188 | `CAN0_ORed_16_31_MB_IRQHandler` | CAN0 OR'ed Message buffer (16-31) |
| 83 | 99 | 0x18C | `Reserved99_IRQHandler` | Reserved Interrupt 99 |
| 84 | 100 | 0x190 | `Reserved100_IRQHandler` | Reserved Interrupt 100 |
| 85 | 101 | 0x194 | `CAN1_ORed_IRQHandler` | CAN1 OR'ed [Bus Off OR Transmit Warning OR Receive Warning] |
| 86 | 102 | 0x198 | `CAN1_Error_IRQHandler` | CAN1 Interrupt indicating that errors were detected on the CAN bus |
| 87 | 103 | 0x19C | `Reserved103_IRQHandler` | Reserved Interrupt 103 |
| 88 | 104 | 0x1A0 | `CAN1_ORed_0_15_MB_IRQHandler` | CAN1 OR'ed Interrupt for Message buffer (0-15) |
| 89 | 105 | 0x1A4 | `CAN1_ORed_16_31_MB_IRQHandler` | CAN1 OR'ed Interrupt for Message buffer (16-31) |
| 90 | 106 | 0x1A8 | `Reserved106_IRQHandler` | Reserved Interrupt 106 |
| 91 | 107 | 0x1AC | `Reserved107_IRQHandler` | Reserved Interrupt 107 |
| 92 | 108 | 0x1B0 | `CAN2_ORed_IRQHandler` | CAN2 OR'ed [Bus Off OR Transmit Warning OR Receive Warning] |
| 93 | 109 | 0x1B4 | `CAN2_Error_IRQHandler` | CAN2 Interrupt indicating that errors were detected on the CAN bus |
| 94 | 110 | 0x1B8 | `Reserved110_IRQHandler` | Reserved Interrupt 110 |
| 95 | 111 | 0x1BC | `CAN2_ORed_0_15_MB_IRQHandler` | CAN2 OR'ed Message buffer (0-15) |
| 96 | 112 | 0x1C0 | `CAN2_ORed_16_31_MB_IRQHandler` | CAN2 OR'ed Message buffer (16-31) |
| 97 | 113 | 0x1C4 | `Reserved113_IRQHandler` | Reserved Interrupt 113 |
| 98 | 114 | 0x1C8 | `Reserved114_IRQHandler` | Reserved Interrupt 114 |
| 99 | 115 | 0x1CC | `FTM0_Ch0_Ch1_IRQHandler` | FTM0 Channel 0 and 1 interrupt |
| 100 | 116 | 0x1D0 | `FTM0_Ch2_Ch3_IRQHandler` | FTM0 Channel 2 and 3 interrupt |
| 101 | 117 | 0x1D4 | `FTM0_Ch4_Ch5_IRQHandler` | FTM0 Channel 4 and 5 interrupt |
| 102 | 118 | 0x1D8 | `FTM0_Ch6_Ch7_IRQHandler` | FTM0 Channel 6 and 7 interrupt |
| 103 | 119 | 0x1DC | `FTM0_Fault_IRQHandler` | FTM0 Fault interrupt |
| 104 | 120 | 0x1E0 | `FTM0_Ovf_Reload_IRQHandler` | FTM0 Counter overflow and Reload interrupt |
| 105 | 121 | 0x1E4 | `FTM1_Ch0_Ch1_IRQHandler` | FTM1 Channel 0 and 1 interrupt |
| 106 | 122 | 0x1E8 | `FTM1_Ch2_Ch3_IRQHandler` | FTM1 Channel 2 and 3 interrupt |
| 107 | 123 | 0x1EC | `FTM1_Ch4_Ch5_IRQHandler` | FTM1 Channel 4 and 5 interrupt |
| 108 | 124 | 0x1F0 | `FTM1_Ch6_Ch7_IRQHandler` | FTM1 Channel 6 and 7 interrupt |
| 109 | 125 | 0x1F4 | `FTM1_Fault_IRQHandler` | FTM1 Fault interrupt |
| 110 | 126 | 0x1F8 | `FTM1_Ovf_Reload_IRQHandler` | FTM1 Counter overflow and Reload interrupt |
| 111 | 127 | 0x1FC | `FTM2_Ch0_Ch1_IRQHandler` | FTM2 Channel 0 and 1 interrupt |
| 112 | 128 | 0x200 | `FTM2_Ch2_Ch3_IRQHandler` | FTM2 Channel 2 and 3 interrupt |
| 113 | 129 | 0x204 | `FTM2_Ch4_Ch5_IRQHandler` | FTM2 Channel 4 and 5 interrupt |
| 114 | 130 | 0x208 | `FTM2_Ch6_Ch7_IRQHandler` | FTM2 Channel 6 and 7 interrupt |
| 115 | 131 | 0x20C | `FTM2_Fault_IRQHandler` | FTM2 Fault interrupt |
| 116 | 132 | 0x210 | `FTM2_Ovf_Reload_IRQHandler` | FTM2 Counter overflow and Reload interrupt |
| 117 | 133 | 0x214 | `FTM3_Ch0_Ch1_IRQHandler` | FTM3 Channel 0 and 1 interrupt |
| 118 | 134 | 0x218 | `FTM3_Ch2_Ch3_IRQHandler` | FTM3 Channel 2 and 3 interrupt |
| 119 | 135 | 0x21C | `FTM3_Ch4_Ch5_IRQHandler` | FTM3 Channel 4 and 5 interrupt |
| 120 | 136 | 0x220 | `FTM3_Ch6_Ch7_IRQHandler` | FTM3 Channel 6 and 7 interrupt |
| 121 | 137 | 0x224 | `FTM3_Fault_IRQHandler` | FTM3 Fault interrupt |
| 122 | 138 | 0x228 | `FTM3_Ovf_Reload_IRQHandler` | FTM3 Counter overflow and Reload interrupt |
| 123 | 139 | 0x22C | `FTM4_Ch0_Ch1_IRQHandler` | FTM4 Channel 0 and 1 interrupt |
| 124 | 140 | 0x230 | `FTM4_Ch2_Ch3_IRQHandler` | FTM4 Channel 2 and 3 interrupt |
| 125 | 141 | 0x234 | `FTM4_Ch4_Ch5_IRQHandler` | FTM4 Channel 4 and 5 interrupt |
| 126 | 142 | 0x238 | `FTM4_Ch6_Ch7_IRQHandler` | FTM4 Channel 6 and 7 interrupt |
| 127 | 143 | 0x23C | `FTM4_Fault_IRQHandler` | FTM4 Fault interrupt |
| 128 | 144 | 0x240 | `FTM4_Ovf_Reload_IRQHandler` | FTM4 Counter overflow and Reload interrupt |
| 129 | 145 | 0x244 | `FTM5_Ch0_Ch1_IRQHandler` | FTM5 Channel 0 and 1 interrupt |
| 130 | 146 | 0x248 | `FTM5_Ch2_Ch3_IRQHandler` | FTM5 Channel 2 and 3 interrupt |
| 131 | 147 | 0x24C | `FTM5_Ch4_Ch5_IRQHandler` | FTM5 Channel 4 and 5 interrupt |
| 132 | 148 | 0x250 | `FTM5_Ch6_Ch7_IRQHandler` | FTM5 Channel 6 and 7 interrupt |
| 133 | 149 | 0x254 | `FTM5_Fault_IRQHandler` | FTM5 Fault interrupt |
| 134 | 150 | 0x258 | `FTM5_Ovf_Reload_IRQHandler` | FTM5 Counter overflow and Reload interrupt |
| 135 | 151 | 0x25C | `FTM6_Ch0_Ch1_IRQHandler` | FTM6 Channel 0 and 1 interrupt |
| 136 | 152 | 0x260 | `FTM6_Ch2_Ch3_IRQHandler` | FTM6 Channel 2 and 3 interrupt |
| 137 | 153 | 0x264 | `FTM6_Ch4_Ch5_IRQHandler` | FTM6 Channel 4 and 5 interrupt |
| 138 | 154 | 0x268 | `FTM6_Ch6_Ch7_IRQHandler` | FTM6 Channel 6 and 7 interrupt |
| 139 | 155 | 0x26C | `FTM6_Fault_IRQHandler` | FTM6 Fault interrupt |
| 140 | 156 | 0x270 | `FTM6_Ovf_Reload_IRQHandler` | FTM6 Counter overflow and Reload interrupt |
| 141 | 157 | 0x274 | `FTM7_Ch0_Ch1_IRQHandler` | FTM7 Channel 0 and 1 interrupt |
| 142 | 158 | 0x278 | `FTM7_Ch2_Ch3_IRQHandler` | FTM7 Channel 2 and 3 interrupt |
| 143 | 159 | 0x27C | `FTM7_Ch4_Ch5_IRQHandler` | FTM7 Channel 4 and 5 interrupt |
| 144 | 160 | 0x280 | `FTM7_Ch6_Ch7_IRQHandler` | FTM7 Channel 6 and 7 interrupt |
| 145 | 161 | 0x284 | `FTM7_Fault_IRQHandler` | FTM7 Fault interrupt |
| 146 | 162 | 0x288 | `FTM7_Ovf_Reload_IRQHandler` | FTM7 Counter overflow and Reload interrupt |

**索引 163~255（保留/未实现）：**

| Index | Offset | Symbol | Description |
|---:|---:|---|---|
| 163 | 0x28C | `DefaultISR` | 保留/未实现中断（IRQ147）。 |
| 164~254 | 0x290~0x3F8 | `DefaultISR` | 保留/未实现中断（IRQ148~IRQ238），每条 `DCD` 都指向 `DefaultISR`。 |
| 255 | 0x3FC | `0xFFFFFFFF` | 用户 TRIM 值保留位（NXP 出厂/用户校准用）。 |

> 说明：
> - 索引 16 开始，`IRQn = Index - 16`，偏移 = `0x40 + 4*IRQn`。
> - 当前工程只用到 `SysTick` 之前的默认处理（全部 `B .`），以及极少数外设中断；所有未实现的中断都落到 `DefaultISR` 的 `B DefaultISR` 死循环。
> - 这就是为什么“跑飞”时如果进入未实现中断，调试器会看到 PC 停在某个 `DefaultISR`/`B .` 地址。要定位真正的故障，需要把 `HardFault_Handler`/`BusFault_Handler` 改成能记录 `CFSR/HFSR/BFAR` 的实现。

### 2.2 `__Vectors` / `__Vectors_Size`

```asm
__Vectors       EQU   __vector_table
__Vectors_Size  EQU   __Vectors_End - __Vectors
```

| 代码 | 作用 / 原理 |
|---|---|
| `__Vectors EQU __vector_table` | 给向量表起始地址起一个别名，供其它代码/调试器使用。 |
| `__Vectors_Size EQU __Vectors_End - __Vectors` | 向量表字节数 = 结束地址 − 起始地址。当前为 `0x400`（256 项 × 4 字节）。 |

### 2.3 默认异常处理与 `DefaultISR`

文件后半部分对每个中断都做了：

```asm
PUBWEAK DMA0_IRQHandler
...
SECTION .text:CODE:REORDER:NOROOT(1)
DMA0_IRQHandler
...
DefaultISR
        B DefaultISR
```

| 代码 | 作用 / 原理 |
|---|---|
| `PUBWEAK xxx` | 声明为“弱符号”。若用户工程里定义了同名函数（强符号），链接器用用户的；否则用这里的默认实现。 |
| `SECTION .text:CODE:REORDER:NOROOT(1)` | 代码段，`REORDER` 允许链接器重排，`NOROOT` 防止被丢弃，2 字节对齐（Thumb 指令最小对齐）。 |
| `label:` 连续写多个标签 | 多个中断名共享同一个地址，即它们都指向同一段默认代码。 |
| `B .` | 原地跳转（无限循环），表示“未处理”。 |
| `B DefaultISR` | 所有未实现的外设中断统一跳到 `DefaultISR` 死循环。 |

> 注意：这 200 多个 `PUBWEAK` 只是“占位”，不会给每个中断生成独立函数体；实际代码量很小。

---

## 3. Flash 配置字段 `FlashConfig`

`startup_S32K148.S` 中：

```asm
        SECTION FlashConfig:CODE
__FlashConfig
        DCD 0xFFFFFFFF    ; 8 bytes backdoor comparison key
        DCD 0xFFFFFFFF    ;
        DCD 0xFFFFFFFF    ; 4 bytes program flash protection bytes
        DCD 0xFFFF7FFE    ; FDPROT:FEPROT:FOPT:FSEC(0xFE = unsecured)
__FlashConfig_End
```

`.icf` 把 `FlashConfig` 段放在 `0x0000_0400 ~ 0x0000_040F`，正好是 S32K148 Flash 的 **Flash Configuration Field（FCF）**。

| 偏移 | 内容 | 作用 / 原理 |
|---:|---|---|
| 0x400~0x407 | 0xFFFFFFFF ×2 | 8 字节后门比较密钥（Backdoor Key）。全 1 表示不启用后门解锁。 |
| 0x408~0x40B | 0xFFFFFFFF | Flash 保护字节（FPROT），全 1 表示不保护。 |
| 0x40C~0x40F | `0xFFFF7FFE` | 依次为 `FDPROT/FEPROT/FOPT/FSEC` 四个字节。小端存放为 `FE 7F FF FF`：`FSEC=0xFE`（未加密/Unsecured）、`FOPT=0x7F`、`FEPROT=0xFF`、`FDPROT=0xFF`。 |

> 重要：如果把 `FSEC` 改成安全值（例如 `0xFC` 等），芯片会被锁住，调试器无法再连接，需要整片擦除。当前 `0xFE` 是“未加密”，正常调试。

---

## 4. `Reset_Handler` 逐行详解

源码（省略条件编译外的注释）：

```asm
        THUMB

        PUBWEAK Reset_Handler
        SECTION .text:CODE:REORDER:NOROOT(2)
Reset_Handler
        CPSID   I
        LDR     R1,=0
        LDR     R2,=0
        LDR     R3,=0
        LDR     R4,=0
        LDR     R5,=0
        LDR     R6,=0
        LDR     R7,=0
        MOV     R8,R7
        MOV     R9,R7
        MOV     R10,R7
        MOV     R11,R7
        MOV     R12,R7

#ifndef __NO_SYSTEM_INIT
        LDR     R0, =SystemInit
        BLX     R0
#endif

#ifdef START_FROM_FLASH
        IMPORT __RAM_START, __RAM_END
        LDR R1, =__RAM_START
        LDR R2, =__RAM_END
        SUBS    R2, R2, R1
        SUBS    R2, #1
        BLE .LC5
        MOVS    R0, #0
        MOVS    R3, #4
    .LC4:
        STR R0, [R1]
        ADD R1, R1, R3
        SUBS R2, #4
        BGE .LC4
    .LC5:
#endif

        LDR     R0, =sfe(CSTACK)
        MOV     R13,R0

        LDR     R0, =init_data_bss
        BLX     R0
        CPSIE   I

        BL      main

JumpToSelf
        B JumpToSelf
```

### 4.1 逐行说明

| 行 | 代码 | 作用 / 原理 |
|---:|---|---|
| `THUMB` | 伪指令 | 告诉汇编器后续使用 Thumb/Thumb-2 指令集。Cortex-M4 只支持 Thumb。 |
| `PUBWEAK Reset_Handler` | 伪指令 | 允许用户工程用强符号覆盖默认复位入口。 |
| `SECTION .text:CODE:REORDER:NOROOT(2)` | 伪指令 | 把后面的代码放入 `.text` 段，4 字节对齐。 |
| `Reset_Handler` | 标签 | 复位入口地址（当前 `0x1328`，向量表里写 `0x1329` 是 Thumb 位）。 |
| `CPSID I` | 指令 | 置 PRIMASK.I = 1，关闭所有可屏蔽中断。启动阶段堆栈/数据还未就绪，必须关中断，防止中断服务程序访问未初始化数据。 |
| `LDR R1,=0` … `LDR R7,=0` | 指令 | 把 R1~R7 清零。C 语言要求 `main` 开始时通用寄存器是确定值（防止脏数据影响调试/安全）。`LDR =0` 由汇编器展开成 `MOV` 或从字面池加载。 |
| `MOV R8,R7` … `MOV R12,R7` | 指令 | 用 R7=0 把 R8~R12 也清零。Thumb-2 的 `MOV` 可在高寄存器间搬移。 |
| `#ifndef __NO_SYSTEM_INIT` | 预处理 | 如果**没有**定义 `__NO_SYSTEM_INIT`，才调用 `SystemInit`。当前 Debug 配置已去掉该宏，Release 配置仍保留（见第 10 章）。 |
| `LDR R0, =SystemInit` | 指令 | 把 `SystemInit` 地址装入 R0。 |
| `BLX R0` | 指令 | 带状态切换的调用（ARM/Thumb 自动切换），并保存返回地址到 LR。 |
| `#ifdef START_FROM_FLASH` | 预处理 | 从 Flash 启动时才需要 ECC RAM 初始化。当前 C 编译器定义了 `START_FROM_FLASH`。 |
| `IMPORT __RAM_START, __RAM_END` | 伪指令 | 引用链接文件导出的 RAM 起止符号（见 ICF 第 73~74 行）。 |
| `LDR R1, =__RAM_START` | 指令 | R1 = `0x1FFE_0000`。 |
| `LDR R2, =__RAM_END` | 指令 | R2 = `0x2001_EFFF`。 |
| `SUBS R2, R2, R1` | 指令 | R2 = 区间长度 = `0x3EFFF`。 |
| `SUBS R2, #1` | 指令 | 长度减 1，便于后面按 4 字节递减判断结束。 |
| `BLE .LC5` | 指令 | 若长度 ≤ 0，跳过 ECC 初始化。 |
| `MOVS R0, #0` | 指令 | R0 = 0，表示“写 0”。 |
| `MOVS R3, #4` | 指令 | R3 = 4，地址步进。 |
| `.LC4:` | 标签 | 循环体入口。 |
| `STR R0, [R1]` | 指令 | 向当前 RAM 地址写 0。写 0 会同时生成正确的 ECC 校验位，避免上电后读未初始化 SRAM 触发 ECC 错误。 |
| `ADD R1, R1, R3` | 指令 | 地址 += 4。 |
| `SUBS R2, #4` | 指令 | 剩余长度 -= 4。 |
| `BGE .LC4` | 指令 | 若 ≥ 0 继续循环；否则结束。 |
| `.LC5:` | 标签 | 循环结束。 |
| `LDR R0, =sfe(CSTACK)` | 指令 | `sfe` = Section End，取 CSTACK 段结束地址 = `0x2001_F000`。 |
| `MOV R13,R0` | 指令 | 设置 SP（R13 = MSP）。虽然硬件复位已从向量表装入 SP，但这里再显式设置一遍，保证与链接脚本一致。 |
| `LDR R0, =init_data_bss` | 指令 | 取 `init_data_bss` 地址。 |
| `BLX R0` | 指令 | 调用 `init_data_bss()`。 |
| `CPSIE I` | 指令 | 清 PRIMASK.I，开全局中断。此时栈和数据已就绪。 |
| `BL main` | 指令 | 调用用户 `main`。注意：**直接调用用户 main，不经过 IAR 的 `?main/__cmain`**（这是后面问题的关键）。 |
| `JumpToSelf: B JumpToSelf` | 指令 | `main` 正常不应返回；若返回则原地死循环，防止跑飞。 |

### 4.2 当前构建的反汇编对照（Debug）

```text
00001328 <Reset_Handler>:
    1328: b672        cpsid   i
    132a: 2100        movs    r1, #0
    ...
    1342: 480a        ldr     r0, [pc, #40]   ; 0x13b1 -> SystemInit
    1344: 4780        blx     r0
    1346: 490a        ldr     r1, [pc, #40]   ; 0x1FFE0000
    1348: 4a0a        ldr     r2, [pc, #40]   ; 0x2001EFFF
    134a: 1a52        subs    r2, r2, r1
    134c: 3a01        subs    r2, #1
    134e: dd05        ble.n   135c
    1350: 2000        movs    r0, #0
    1352: 2304        movs    r3, #4
    1354: 6008        str     r0, [r1]        ; ECC 初始化
    ...
    135c: 4806        ldr     r0, [pc, #24]   ; 0x2001F000
    135e: 4685        mov     sp, r0
    1360: 4806        ldr     r0, [pc, #24]   ; 0x1059 -> init_data_bss
    1362: 4780        blx     r0
    1364: b662        cpsie   i
    1366: f7ff feec   bl      1142 <main>
```

> 注意：当前 Debug 构建里 `Reset_Handler` **调用 `SystemInit`（0x1342）**；而旧版本（定义 `__NO_SYSTEM_INIT` 时）这里没有这条 `blx`，直接进入 ECC 初始化。

---

## 5. `SystemInit()` 逐行详解

文件：`StartUp_Code/devices/S32K148/startup/system_S32K148.c`

### 5.1 文件级内容

```c
#include "device_registers.h"
#include "system_S32K148.h"
#include "stdbool.h"

uint32_t SystemCoreClock = DEFAULT_SYSTEM_CLOCK;
```

| 代码 | 作用 / 原理 |
|---|---|
| `#include "device_registers.h"` | 汇总包含 `S32K148.h`、`S32K148_features.h`、`s32_core_cm4.h`，得到所有外设寄存器定义与特性宏。 |
| `#include "system_S32K148.h"` | 包含 `DISABLE_WDOG`、`ICACHE_ENABLE`、`CPU_XTAL_CLK_HZ` 等配置宏。 |
| `#include "stdbool.h"` | 为了使用 `bool/true/false`（`SystemCoreClockUpdate` 里用到）。 |
| `uint32_t SystemCoreClock = DEFAULT_SYSTEM_CLOCK;` | 全局系统时钟频率变量，默认 `DEFAULT_SYSTEM_CLOCK`（通常 48 MHz）。MCAL/驱动有时用它计算超时。注意：本工程启动时**没有**调用 `SystemCoreClockUpdate()`，所以上电后它仍是默认值。 |

### 5.2 `SystemInit()` 函数

```c
void SystemInit(void)
{
#ifdef ENABLE_FPU
  S32_SCB->CPACR |= (S32_SCB_CPACR_CP10_MASK | S32_SCB_CPACR_CP11_MASK);
#ifdef ERRATA_E6940
  S32_SCB->FPCCR &= ~(S32_SCB_FPCCR_LSPEN_MASK);
#endif
#endif

#if (DISABLE_WDOG)
  WDOG->CNT = (uint32_t) FEATURE_WDOG_UNLOCK_VALUE;
  (void)WDOG->CNT;

  WDOG->CS = (uint32_t)( (1UL << WDOG_CS_CMD32EN_SHIFT)
                       | (FEATURE_WDOG_CLK_FROM_LPO << WDOG_CS_CLK_SHIFT)
                       | (0U << WDOG_CS_EN_SHIFT)
                       | (1U << WDOG_CS_UPDATE_SHIFT) );
  WDOG->TOVAL = (uint32_t)0xFFFF;
#endif

#if defined(I_CACHE) && (ICACHE_ENABLE == 1)
  LMEM->PCCCR = ...;
#endif

#if ENABLE_ALL_BUS_MASTERS_IN_MPU
  MPU->RGDAAC[0] |= ...;
#endif
}
```

| 代码 | 作用 / 原理 |
|---|---|
| `#ifdef ENABLE_FPU` | Cortex-M4F 的 FPU 默认是关闭的（CPACR 的 CP10/CP11=0）。若代码里有浮点指令，访问 FPU 会触发 UsageFault。`s32_core_cm4.h` 对 CM4F 内核定义了 `ENABLE_FPU`。 |
| `S32_SCB->CPACR |= (CP10_MASK \| CP11_MASK)` | 把 CP10/CP11 设为“全访问”（0b11），使能 FPU。SCB 基址 `0xE000_ED00`，CPACR 偏移 `0x88`。 |
| `#ifdef ERRATA_E6940` | 芯片勘误 e6940 的规避：关闭 FPU 惰性上下文保存。 |
| `S32_SCB->FPCCR &= ~LSPEN` | 清 `FPCCR.LSPEN`，避免异常压栈时出现已知问题。 |
| `#if (DISABLE_WDOG)` | `system_S32K148.h` 默认 `#define DISABLE_WDOG 1`，所以这段会编译。 |
| `WDOG->CNT = FEATURE_WDOG_UNLOCK_VALUE;` | 向 WDOG 计数器寄存器写入解锁码 `0xD928C520`（32 位解锁序列）。WDOG 的配置位是 write-once，必须先解锁才能改 `CS`。 |
| `(void)WDOG->CNT;` | 空读一次 CNT，制造总线同步，确保解锁写已完成（WDOG 要求命令之间有时序）。 |
| `WDOG->CS = ...` | 写控制/状态寄存器：<br>`CMD32EN=1`：支持 32 位刷新/解锁命令；<br>`CLK=LPO`：看门狗时钟选 LPO；<br>`EN=0`：**关闭看门狗**；<br>`UPDATE=1`：允许更新配置。 |
| `WDOG->TOVAL = 0xFFFF;` | 设置超时值（虽然已经关闭，仍给一个安全值）。 |
| `#if defined(I_CACHE) && (ICACHE_ENABLE == 1)` | 本工程 `ICACHE_ENABLE=0`，不编译。若开启，会配置 LMEM 代码缓存（invalidate + enable）。 |
| `#if ENABLE_ALL_BUS_MASTERS_IN_MPU` | 本工程该宏被注释掉（见 `system_S32K148.h`），不编译。若开启，会给 MPU 的 region 0 增加 ENET 总线主设备权限。 |

> **关键点：S32K148 的 WDOG 复位后默认是开启的**（`WDOG_CS` 复位值 `0x0000_2980`，`EN=1`），并且由 `LPO128K_CLK` 计数，`TOVAL=0x400`。RM 23.1.3 明确：
>
> > Following initial power on the watchdog is clocked by the LPO128K_CLK and will issue a timeout after 1024 cycles. This results in the Watchdog timeout being generated after approximately **8 ms**, which will force a MCU reset.
>
> 所以如果 `SystemInit()` 没被调用（例如定义了 `__NO_SYSTEM_INIT`），MCU 会在上电约 8 ms 后被看门狗复位。这正好落在启动 + `Mcu_Init` + 时钟初始化附近。

### 5.3 `SystemCoreClockUpdate()`（启动阶段未被调用）

| 代码 | 作用 / 原理 |
|---|---|
| `divider = ((SCG->CSR & DIVCORE_MASK) >> SHIFT) + 1` | 读当前系统时钟的 CORE 分频。 |
| `switch (SCS)` | 根据系统时钟源选择：SOSC/SIRC/FIRC/SPLL。 |
| `case 0x1: SCGOUTClock = CPU_XTAL_CLK_HZ;` | SOSC 直接作为系统时钟。 |
| `case 0x2: ... SIRCCFG.RANGE ...` | SIRC 高/低范围频率。 |
| `case 0x3: ... FIRCCFG.RANGE ...` | FIRC 频率（48/52/56/60 MHz）。 |
| `case 0x6: ...` | SPLL：`SOSC/(PREDIV+1) * (MULT+16) / 2`。 |
| `SystemCoreClock = SCGOUTClock / divider;` | 得到 CORE_CLK 频率。 |

> 本工程启动代码没有调用它；如果应用依赖 `SystemCoreClock`，需要在 `Mcu_InitClock()` 之后手动调用一次。

### 5.4 `SystemSoftwareReset()`（启动阶段未被调用）

通过写 `SCB->AIRCR` 的 `SYSRESETREQ` 位请求一次系统复位，属于应用层 API。

---

## 6. `init_data_bss()` 逐行详解

文件：`StartUp_Code/devices/startup.c`

### 6.1 静态变量与包含

```c
#include "startup.h"
#include <stdint.h>

static volatile uint32_t * const s_vectors[NUMBER_OF_CORES] = FEATURE_INTERRUPT_INT_VECTORS;
```

| 代码 | 作用 / 原理 |
|---|---|
| `#include "startup.h"` | 包含 `init_data_bss` 声明、`NUMBER_OF_CORES`、`GET_CORE_ID()` 等。 |
| `#include <stdint.h>` | 使用 `uint32_t/uint8_t`。 |
| `s_vectors` | 一个只读指针数组，元素指向“向量表寄存器”。S32K148 单核：`FEATURE_INTERRUPT_INT_VECTORS = {&S32_SCB->VTOR}`，即 `s_vectors[0] = &SCB->VTOR`（地址 `0xE000_ED08`）。后面用它设置 VTOR。 |

### 6.2 函数体逐行

```c
void init_data_bss(void)
{
    uint32_t n;
    uint8_t coreId;

#if !defined(__ARMCC_VERSION)
    uint8_t * data_ram;
    uint8_t * code_ram;
    uint8_t * bss_start;
    uint8_t * custom_ram;
    const uint8_t * data_rom, * data_rom_end;
    const uint8_t * code_rom, * code_rom_end;
    const uint8_t * bss_end;
    const uint8_t * custom_rom, * custom_rom_end;
#endif

#if defined(__ARMCC_VERSION)
    extern uint32_t __RAM_VECTOR_TABLE_SIZE;
    extern uint32_t __VECTOR_ROM;
    extern uint32_t __VECTOR_RAM;
#else
    extern uint32_t __RAM_VECTOR_TABLE_SIZE[];
    extern uint32_t __VECTOR_TABLE[];
    extern uint32_t __VECTOR_RAM[];
#endif

#if defined(__ICCARM__)
    data_ram        = __section_begin(".data");
    data_rom        = __section_begin(".data_init");
    data_rom_end    = __section_end(".data_init");

    #pragma section = "__CODE_ROM"
    #pragma section = "__CODE_RAM"
    code_ram        = __section_begin("__CODE_RAM");
    code_rom        = __section_begin("__CODE_ROM");
    code_rom_end    = __section_end("__CODE_ROM");

    bss_start       = __section_begin(".bss");
    bss_end         = __section_end(".bss");

    custom_ram      = __section_begin(".customSection");
    custom_rom      = __section_begin(".customSection_init");
    custom_rom_end  = __section_end(".customSection_init");
#elif ...
#endif

#if !defined(__ARMCC_VERSION)
    while (data_rom_end != data_rom) { *data_ram = *data_rom; data_ram++; data_rom++; }
    while (code_rom_end != code_rom) { *code_ram = *code_rom; code_ram++; code_rom++; }
    while (bss_end != bss_start)     { *bss_start = 0; bss_start++; }
    while (custom_rom_end != custom_rom) { *custom_ram = *custom_rom; custom_rom++; custom_ram++; }
#endif

    coreId = (uint8_t)GET_CORE_ID();
    if (__VECTOR_RAM != __VECTOR_TABLE)
    {
        for (n = 0; n < (((uint32_t)__RAM_VECTOR_TABLE_SIZE)/sizeof(uint32_t)); n++)
        {
            __VECTOR_RAM[n] = __VECTOR_TABLE[n];
        }
        *s_vectors[coreId] = (uint32_t)__VECTOR_RAM;
    }
    else
    {
        *s_vectors[coreId] = (uint32_t)__VECTOR_TABLE;
    }
}
```

| 代码 | 作用 / 原理 |
|---|---|
| `uint32_t n; uint8_t coreId;` | `n` 用于向量表复制循环；`coreId` 用于多核时选择对应的 VTOR。S32K148 单核，`GET_CORE_ID()` 恒为 0。 |
| `#if !defined(__ARMCC_VERSION)` | 针对 IAR/GCC/GHS 等：需要手动复制数据段。ARMCC 使用库自带初始化，不需要这些指针。 |
| `data_ram = __section_begin(".data");` | IAR 内建函数：取 `.data` 段起始地址（RAM 中已初始化变量）。当前工程该段为空，返回 0。 |
| `data_rom = __section_begin(".data_init");` | 取 `.data` 初值在 ROM 中的副本段起始地址。当前为空。 |
| `data_rom_end = __section_end(".data_init");` | 初值副本段结束地址。 |
| `#pragma section = "__CODE_ROM"` / `__CODE_RAM` | 告诉 IAR 后面要按名字访问这两个段。用于“需要在 RAM 中执行的函数”（`__ramfunc`/`.textrw`）。 |
| `code_ram = __section_begin("__CODE_RAM");` | RAM 中代码段起始地址。当前为 `0x1FFE_0410`（空）。 |
| `code_rom = __section_begin("__CODE_ROM");` | ROM 中代码副本起始地址。当前为 `0x15A6`（空）。 |
| `code_rom_end = __section_end("__CODE_ROM");` | 副本结束地址。 |
| `bss_start = __section_begin(".bss");` | `.bss` 起始地址。当前为 `0x2000_0004`。 |
| `bss_end = __section_end(".bss");` | `.bss` 结束地址。当前为 `0x2000_0009`。 |
| `custom_ram = __section_begin(".customSection");` | 用户自定义段（可放置绝对地址数据）在 RAM 中的起始。当前为空。 |
| `custom_rom = __section_begin(".customSection_init");` | 自定义段的 ROM 初值起始。 |
| `custom_rom_end = __section_end(".customSection_init");` | 自定义段初值结束。 |
| `while (data_rom_end != data_rom) { ... }` | 把 `.data_init` 的内容逐字节复制到 `.data`。这就是“已初始化全局变量”的初始化。当前为空循环。 |
| `while (code_rom_end != code_rom) { ... }` | 把需要在 RAM 中运行的代码复制到 RAM。当前为空循环。 |
| `while (bss_end != bss_start) { *bss_start = 0; ... }` | 把 `.bss` 清零（未初始化全局变量/静态变量）。当前清 `0x2000_0004~0x2000_0008`（5 字节，Det 的 4 个变量）。 |
| `while (custom_rom_end != custom_rom) { ... }` | 复制 `.customSection`。当前为空循环。 |
| `coreId = GET_CORE_ID();` | 取当前核编号（S32K148 恒 0）。 |
| `if (__VECTOR_RAM != __VECTOR_TABLE)` | 判断是否需要把向量表复制到 RAM。当前 `__VECTOR_TABLE=0x0`、`__VECTOR_RAM=0x1FFE_0000`，需要复制。 |
| `for (n=0; n < __RAM_VECTOR_TABLE_SIZE/4; n++) __VECTOR_RAM[n] = __VECTOR_TABLE[n];` | 把 0x400 字节（256 项）向量表从 Flash 复制到 RAM。这样运行时可以动态改中断向量（如 `INT_SYS_InstallHandler`）。 |
| `*s_vectors[coreId] = (uint32_t)__VECTOR_RAM;` | 写 `SCB->VTOR = 0x1FFE_0000`，让内核从 RAM 取向量。 |
| `else { *s_vectors[coreId] = (uint32_t)__VECTOR_TABLE; }` | 若定义了 `__flash_vector_table__`，`__VECTOR_RAM == __VECTOR_TABLE`，则直接把 VTOR 指向 Flash 向量表。 |

### 6.3 重要补充：它**没有**初始化 `.mcal_data` / `.mcal_bss`

`init_data_bss()` 只处理以下段：

- `.data` / `.data_init`
- `__CODE_ROM` / `__CODE_RAM`
- `.bss`
- `.customSection` / `.customSection_init`
- `.intvec`（向量表）

而 MCAL 的 `MemMap.h` 把“已初始化的驱动变量”放在了 **`.mcal_data`** 段，把“零初始化变量”放在了 **`.mcal_bss`** 段。当前链接结果：

```text
RW  (0x1FFE_0400, 0x10 字节)
  .mcal_data Mcu.o      0x1FFE_0400  4   -> Mcu_pConfigPtr = NULL
  .mcal_data Mcu_IPW.o  0x1FFE_0404  4   -> Mcu_pClockConfig = NULL
  .mcal_data Port.o     0x1FFE_0408  4   -> Port_pConfig = NULL
  .mcal_data Mcu.o      0x1FFE_040C  1   -> Mcu_eStatus = MCU_UNINIT (3)
ZI  (0x2000_0000, 0x9 字节)
  .mcal_bss Mcu.o       0x2000_0000  1   -> Mcu_au8ClockConfigIds[?]
  .mcal_bss Mcu.o       0x2000_0001  1   -> Mcu_au8ModeConfigIds[?]
  .bss Det.o            0x2000_0004  5
```

反汇编 `init_data_bss()` 可见它拿到的指针是：

```text
data_ram=0, data_rom=0, data_rom_end=0   -> .data 为空
code_ram=0x1FFE0410, code_rom=code_rom_end=0x15A6 -> __CODE_ROM 为空
bss_start=0x20000004, bss_end=0x20000009 -> 只清 .bss
custom_ram=custom_rom=custom_rom_end=0   -> .customSection 为空
```

也就是说：**`.mcal_data` 的初值（尤其是 `Mcu_eStatus = 3`）没有被写入**，上电后它保持 ECC 初始化写的 0。

> 谁本来应该初始化它？
> IAR 链接器根据 ICF 的 `initialize by copy { readwrite }` 生成了一张 **init table**（本工程在 Flash `0x1540`，内容 16 字节：`Mcu_pConfigPtr=NULL`、`Mcu_pClockConfig=NULL`、`Port_pConfig=NULL`、`Mcu_eStatus=3`）。
> 这张表由 IAR 库函数 **`__iar_data_init3()`** 执行；而 `__iar_data_init3()` 只被 `?main`/`__cmain` 调用，`?main` 只被 `__iar_program_start` 调用。
> 本工程的 `Reset_Handler` 是 **`BL init_data_bss` 然后直接 `BL main`**，既不经过 `__iar_program_start`，也不经过 `?main`，所以 **init table 永远不会被执行**。
>
> 这正是第 10 章要分析的“274 行跑飞”的直接原因链。

---

## 7. 进入 `main()`

`main` 地址 `0x1142`。反汇编开头：

```text
00001142 <main>:
    1142: b510        push    {r4, lr}
    1144: 4c13        ldr     r4, [pc, #76]   ; 0x1194 -> 超时计数器初值
    1146: f7ff fbf7   bl      938 <Det_Init>
    114a: 4813        ldr     r0, [pc, #76]   ; 0x1198 -> &McuModuleConfiguration
    114c: f7ff fb4f   bl      7ee <Mcu_Init>
    1150: 2000        movs    r0, #0
    1152: f7ff fb87   bl      864 <Mcu_InitClock>
    ...
```

| 代码 | 作用 / 原理 |
|---|---|
| `Det_Init()` | 初始化 DET（开发错误报告）模块。 |
| `Mcu_Init(&McuModuleConfiguration)` | 保存配置指针、初始化 RCM/PMC/SIM/SMC。内部会先调用 `Mcu_HLDChecksEntry(MCU_INIT_ID)` 检查驱动状态。 |
| `Mcu_InitClock(McuClockSettingConfig_0)` | 配置 SIM/SCG/PCC 时钟树。内部调用 `Mcu_Ipw_InitClock()`。 |
| 等待 PLL | 用有上限的循环等待 `Mcu_GetPllStatus() == MCU_PLL_LOCKED`。 |
| `Mcu_DistributePllClock()` | 把 PLL 分频到系统（本工程实际在 `Mcu_SCG_SystemClockInit()` 里已切到 SPLL）。 |
| `Mcu_SetMode(McuModeSettingConf_0)` | 配置/进入 RUN 模式。 |
| `Port_Init(&PortConfigSet)` | 配置引脚复用。 |
| `for(;;)` | 翻转 PTE20/PTE21，主循环。 |

> 注意：`main` 的第一条指令是 `push {r4, lr}`，会使用栈。因此 `Reset_Handler` 里“设置 SP”这一步必须在 `BL main` 之前完成；当前代码顺序是正确的。

---

## 8. 链接文件 `S32K148_256_flash.icf` 逐行详解

> ICF = IAR Configuration File，IAR 的链接脚本（类似 GCC 的 `.ld`）。
> 基本语法：
> - `define symbol X = 值;` 定义链接期常量；
> - `define memory mem with size = ...;` 定义一块可寻址空间；
> - `define region R = mem:[from A to B] | mem:[...];` 定义区域（可由多段组成）；
> - `define block B { ... };` 定义输出块；
> - `initialize by copy/manually/do not initialize` 控制启动时是否由库初始化；
> - `place in/at ... { ... }` 把段/块放到区域或绝对地址。

### 8.1 逐行对照表

| 行 | 代码 | 作用 / 原理 |
|---:|---|---|
| 1~25 | 注释 | 版权与文件说明，无链接作用。 |
| 27~31 | 注释 | 说明 `__flash_vector_table__` 的作用：定义该符号则向量表不复制到 RAM；否则调试器/`INT_SYS_InstallHandler` 可在 RAM 改向量。 |
| 32 | `define symbol __ram_vector_table_size__ = isdefinedsymbol(__flash_vector_table__) ? 0 : 0x00000400;` | 计算 RAM 向量表大小。未定义 `__flash_vector_table__` 时为 `0x400`（256 项 × 4 字节）。 |
| 33 | `define symbol __ram_vector_table_offset__ = isdefinedsymbol(__flash_vector_table__) ? 0 : 0x000003FF;` | 计算 RAM 向量表“结束偏移”（大小−1），用于第 47 行算出结束地址。 |
| 35 | `/* Flash */` | 注释。 |
| 36 | `define symbol m_interrupts_start = 0x00000000;` | 向量表/中断向量区起始地址（Flash 0x0）。 |
| 37 | `define symbol m_interrupts_end = 0x000003FF;` | 向量表区结束（0x400 字节）。 |
| 39 | `define symbol m_flash_config_start = 0x00000400;` | Flash 配置字段（FCF）起始。 |
| 40 | `define symbol m_flash_config_end = 0x0000040F;` | FCF 结束（16 字节）。 |
| 42 | `define symbol m_text_start = 0x00000410;` | 代码/常量区起始（跳过 FCF）。 |
| 43 | `define symbol m_text_end = 0x0017FFFF;` | 代码/常量区结束。S32K148 1.5 MB Flash = 0x0~0x17FFFF。 |
| 45 | `/* SRAM_L */` | 注释。S32K148 的 SRAM 分为 SRAM_L（低）和 SRAM_U（高）。 |
| 46 | `define symbol m_interrupts_ram_start = 0x1FFE0000;` | RAM 向量表起始地址（SRAM_L 起始）。 |
| 47 | `define symbol m_interrupts_ram_end = 0x1FFE0000 + __ram_vector_table_offset__;` | RAM 向量表结束地址（0x1FFE_03FF）。 |
| 49 | `define symbol m_data_start = m_interrupts_ram_start + __ram_vector_table_size__;` | 普通数据区起始 = RAM 向量表之后（0x1FFE_0400）。 |
| 50 | `define symbol m_data_end = 0x1FFFFFFF;` | SRAM_L 结束地址。SRAM_L 共 0x20000 = 128 KB。 |
| 52 | `/* SRAM_U */` | 注释。 |
| 53 | `define symbol m_data_2_start = 0x20000000;` | SRAM_U 起始（用于 ZI、HEAP 等）。 |
| 54 | `define symbol m_data_2_end = 0x2001EFFF;` | SRAM_U 使用到的结束地址（0x20000 中的 0x1F000 = 124 KB）。 |
| 56 | `/* Sizes */` | 注释。 |
| 57 | `if (isdefinedsymbol(__stack_size__)) {` | 若用户在链接选项里自定义了 `__stack_size__`，则用它。 |
| 58 | `define symbol __size_cstack__ = __stack_size__;` | 栈大小 = 用户值。 |
| 59 | `} else {` | 否则使用默认值。 |
| 60 | `define symbol __size_cstack__ = 0x00000400;` | 默认栈 1 KB。 |
| 61 | `}` | 条件结束。 |
| 63 | `if (isdefinedsymbol(__heap_size__)) {` | 堆大小同理。 |
| 64 | `define symbol __size_heap__ = __heap_size__;` | 用户自定义堆大小。 |
| 65 | `} else {` | 默认。 |
| 66 | `define symbol __size_heap__ = 0x00000400;` | 默认堆 1 KB。 |
| 67 | `}` | 条件结束。 |
| 69 | `define exported symbol __VECTOR_TABLE = m_interrupts_start;` | 导出向量表地址（0x0），供 C 代码/调试器使用。 |
| 70 | `define exported symbol __VECTOR_RAM = isdefinedsymbol(__flash_vector_table__) ? m_interrupts_start : m_interrupts_ram_start;` | 导出 RAM 向量表地址：未定义 `__flash_vector_table__` 时为 `0x1FFE_0000`。 |
| 71 | `define exported symbol __RAM_VECTOR_TABLE_SIZE = __ram_vector_table_size__;` | 导出向量表大小（0x400），`init_data_bss` 用它计算复制次数。 |
| 73 | `define exported symbol __RAM_START = m_interrupts_ram_start;` | 导出 RAM 起始（0x1FFE_0000），供 ECC 初始化循环使用。 |
| 74 | `define exported symbol __RAM_END = m_data_2_end;` | 导出 RAM 结束（0x2001_EFFF），供 ECC 初始化循环使用。 |
| 76 | `define memory mem with size = 4G;` | 定义整个 4 GB 地址空间，便于后面用 `mem:[from ... to ...]` 划分区域。 |
| 77 | `define region m_flash_config_region = mem:[from m_flash_config_start to m_flash_config_end];` | 区域：0x400~0x40F（FCF）。 |
| 78~79 | `define region TEXT_region = mem:[from m_interrupts_start to m_interrupts_end] \| mem:[from m_text_start to m_text_end];` | 代码区 = 向量表区 0x0~0x3FF **并上** 代码区 0x410~0x17FFFF。中间的 0x400~0x40F 留给 FCF，不属于 TEXT。 |
| 80 | `define region DATA_region = mem:[from m_data_start to m_data_end];` | 数据区（SRAM_L 中向量表之后的部分，0x1FFE_0400~0x1FFF_FFFF）。 |
| 81 | `define region DATA_region_2 = mem:[from m_data_2_start to m_data_2_end-__size_cstack__];` | SRAM_U 中除去栈之后的部分（ZI/HEAP）。 |
| 82 | `define region CSTACK_region = mem:[from m_data_2_end-__size_cstack__+1 to m_data_2_end];` | 栈区，位于 SRAM_U 最末尾 1 KB（0x2001_EC00~0x2001_EFFF）。 |
| 83 | `define region m_interrupts_ram_region = mem:[from m_interrupts_ram_start to m_interrupts_ram_end];` | RAM 向量表区（0x1FFE_0000~0x1FFE_03FF）。 |
| 86 | `define block CSTACK with alignment = 8, size = __size_cstack__ { };` | 定义名为 CSTACK 的空块，大小 1 KB，8 字节对齐。`{ }` 表示不收集任何输入段，只是一个“保留空间”的块。 |
| 87 | `define block HEAP with alignment = 8, size = __size_heap__ { };` | 定义 HEAP 块，1 KB，8 字节对齐。 |
| 88 | `define block RW { readwrite };` | 定义 RW 块，收集所有 `readwrite` 属性的段（本工程就是 `.mcal_data` 等已初始化数据）。 |
| 89 | `define block ZI { zi };` | 定义 ZI 块，收集所有 `zi` 属性的段（本工程 `.mcal_bss`）。 |
| 91~92 | 注释 | 说明 `.customSection` 的用途：用户可用 `__attribute__((section(".customSection")))` 把数据放到绝对地址。 |
| 93 | `define block customSectionBlock { section .customSection };` | 把 `.customSection` 单独定义成一个块。 |
| 95 | `define block __CODE_ROM { section .textrw_init };` | `__CODE_ROM` 块：RAM 执行代码的 ROM 初值段。 |
| 96 | `define block __CODE_RAM { section .textrw };` | `__CODE_RAM` 块：需要搬到 RAM 执行的代码段。 |
| 98 | `initialize by copy { readwrite };` | 告诉链接器：所有 `readwrite` 段需要“从 ROM 复制到 RAM”。这会生成 init table 条目（本工程 Copy 0x1540→0x1FFE0400）。 |
| 99 | `initialize manually { section .textrw };` | `.textrw` 由用户代码手动初始化（本工程 `init_data_bss` 里的 `__CODE_ROM` 复制）。 |
| 100 | `initialize manually { section .bss };` | `.bss` 由用户手动清零（`init_data_bss`）。 |
| 101 | `initialize manually { section .customSection };` | `.customSection` 由用户手动初始化。 |
| 102 | `initialize manually { section .data };` | `.data` 由用户手动初始化（`init_data_bss`）。 |
| 103 | `initialize manually { section __DLIB_PERTHREAD };` | IAR 线程局部存储段，手动初始化（本工程不用）。 |
| 104 | `do not initialize { section .noinit, section .bss, section .data, section __DLIB_PERTHREAD, section .customSection };` | 明确告诉链接器/库：这些段**不要**由自动初始化代码处理（避免与手动初始化重复）。注意 `.noinit` 保留原值（常用于保存复位原因）。 |
| 106 | `place at address mem: m_interrupts_start { readonly section .intvec };` | 把只读的 `.intvec` 段**绝对定位**到 0x0000_0000。这是硬件复位取向量的要求。 |
| 107 | `place in m_flash_config_region { section FlashConfig };` | 把 `FlashConfig` 段放到 0x400~0x40F。 |
| 108 | `place in TEXT_region { readonly };` | 把所有剩余的只读段（`.text`、`.rodata`、`.data_init`、`.mcal_const` 等）放到 TEXT 区（Flash）。 |
| 109 | `place in TEXT_region { block __CODE_ROM };` | 把 RAM 执行代码的初值副本放到 Flash。 |
| 110 | `place in DATA_region { block RW };` | 把 RW 块（`.mcal_data` 等）放到 SRAM_L 的 0x1FFE_0400 起。 |
| 111 | `place in DATA_region { block __CODE_RAM };` | 把需要在 RAM 执行的代码放到 SRAM_L（本工程为空）。 |
| 112 | `place in DATA_region_2 { first block customSectionBlock };` | 在 SRAM_U 区域**最前面**放置 `.customSection`。`first` 表示放在该区域已放置块的起始位置。 |
| 113 | `place in DATA_region_2 { block ZI };` | 放置 ZI 块（`.mcal_bss` 等）。 |
| 114 | `place in DATA_region_2 { last block HEAP };` | 在 SRAM_U 区域**最后**放置 HEAP（紧邻栈下方）。 |
| 115 | `place in CSTACK_region { block CSTACK };` | 把栈块放到 SRAM_U 末尾 1 KB。 |
| 116 | `place in m_interrupts_ram_region { section m_interrupts_ram };` | 把 `m_interrupts_ram` 段放到 0x1FFE_0000 起的 RAM 向量表区（本工程该段为空，复制由 `init_data_bss` 完成）。 |

### 8.2 关键设计点总结

1. **向量表必须在 0x0**：`place at address mem: m_interrupts_start { readonly section .intvec }`。
2. **FCF 必须在 0x400**：由 `FlashConfig` 段和 `m_flash_config_region` 保证。
3. **代码从 0x410 开始**：`TEXT_region` 的第二段。
4. **RAM 向量表在 0x1FFE_0000**：方便运行时改中断向量；`VTOR` 由 `init_data_bss` 设置。
5. **RW 数据在 0x1FFE_0400**：紧接 RAM 向量表之后。
6. **ZI 在 0x2000_0000 起**：SRAM_U 起始。
7. **栈在 SRAM_U 最末尾**：`0x2001_EC00~0x2001_EFFF`，栈顶 `0x2001_F000`。
8. **ECC 初始化范围**：`__RAM_START=0x1FFE_0000` 到 `__RAM_END=0x2001_EFFF`，覆盖 SRAM_L + SRAM_U 的实际使用范围。
9. **init 属性冲突**：ICF 同时写了 `initialize by copy { readwrite }` 和 `initialize manually { section .data }`，这是 NXP SDK 的常见写法；但**手动初始化函数只处理部分段**，导致 `.mcal_data`/`.mcal_bss` 被遗漏（见第 10 章）。

---

## 9. 链接结果：实际内存布局

当前 Debug 构建（来自 `.map`）：

```text
Flash:
  0x0000_0000 ~ 0x0000_03FF   .intvec         (向量表，1024 B)
  0x0000_0400 ~ 0x0000_040F   FlashConfig     (16 B，FCF)
  0x0000_0410 ~ 0x0000_15A5   代码/常量        (TEXT_region)
  0x0000_1540 ~ 0x0000_154F   RW init table   (16 B，但当前未被执行)
  0x0000_15A6 ~ 0x0017_FFFF   未使用

SRAM_L (0x1FFE_0000 ~ 0x1FFF_FFFF):
  0x1FFE_0000 ~ 0x1FFE_03FF   RAM 向量表      (0x400 B)
  0x1FFE_0400 ~ 0x1FFE_040F   RW (.mcal_data) (16 B)
  0x1FFE_0410 ~ 0x1FFF_FFFF   未使用

SRAM_U (0x2000_0000 ~ 0x2001_EFFF):
  0x2000_0000 ~ 0x2000_0001   .mcal_bss       (2 B)
  0x2000_0004 ~ 0x2000_0008   .bss            (5 B)
  0x2000_0009 ~ 0x2001_EBFF   未使用
  0x2001_EC00 ~ 0x2001_EFFF   CSTACK          (0x400 B)
  栈顶 = sfe(CSTACK) = 0x2001_F000
```

> 注意：S32K148 的 SRAM_U 物理上到 `0x2001_FFFF`，但 ICF 只用到 `0x2001_EFFF`（把 `0x2001_F000~0x2001_FFFF` 留空，通常与器件保留区/ECC 分区有关）。栈放在 `0x2001_EC00~0x2001_EFFF` 是安全的。

---

## 10. 本文分析发现的关键问题与修复建议

### 10.1 问题一：`__NO_SYSTEM_INIT` 导致 `SystemInit()` 不执行 → WDOG 8 ms 复位

**现象**：代码烧进去后跑飞，PC 停在 `Mcu_SIM_ClockConfig` 第 274 行附近。

**机制**：

1. `S32K1X_Project.ewp` 的 **汇编器**（`AARM → ADefines`）里定义了 `__NO_SYSTEM_INIT`。
2. `startup_S32K148.S` 用 `#ifndef __NO_SYSTEM_INIT` 包住 `BLX SystemInit`，所以 `SystemInit()` 不执行。
3. `SystemInit()` 里唯一关闭 WDOG 的代码不执行。
4. S32K148 的 WDOG 复位后默认开启，时钟 `LPO128K`，`TOVAL=0x400` → **约 8 ms** 后强制复位（RM 23.1.3）。
5. 启动 + `Mcu_Init` 的耗时正好在毫秒级，复位点容易落在时钟初始化函数里，调试器看起来就像“在 274 行跑飞”。

**当前状态**：

- Debug 配置（ewp 第 538~540 行）已去掉 `__NO_SYSTEM_INIT`，`SystemInit` 已在 `Reset_Handler` 中调用（反汇编 0x1342），WDOG 关闭（`WDOG->CS=0x2120`，`EN=0`）。
- **Release 配置（ewp 第 1600~1602 行）仍然保留 `__NO_SYSTEM_INIT`**，如果编译/下载的是 Release，问题依旧。

**修复**：两套配置（Debug/Release）都删除 `__NO_SYSTEM_INIT`，保留 `START_FROM_FLASH`。

### 10.2 问题二（当前 274 行跑飞的直接原因）：`.mcal_data` 未初始化 → `Mcu_Init` 被跳过 → 配置指针为 NULL/垃圾

这是本文对启动流程逐行分析后得到的、**与现象完全吻合**的根因链：

```text
① 上电 ECC 初始化把 0x1FFE_0000~0x2001_EFFF 全写 0
        ↓
② Reset_Handler 调用 init_data_bss()
   init_data_bss 只处理 .data/.bss/.customSection/__CODE_ROM/.intvec
   —— 不处理 .mcal_data / .mcal_bss
        ↓
③ IAR 的 init table（0x1540: Mcu_eStatus=3 等）由 __iar_data_init3 执行
   但 __iar_data_init3 只被 ?main/__cmain 调用
   而 Reset_Handler 是 BL init_data_bss 后直接 BL main
   —— __iar_data_init3 没有被调用
        ↓
④ Mcu_eStatus (0x1FFE_040C) 仍是 0，而不是 MCU_UNINIT(3)
        ↓
⑤ Mcu_Init():
   Mcu_HLDChecksEntry(MCU_INIT_ID) 读 Mcu_eStatus
   0 != 3 → 返回 E_NOT_OK（并 Det_ReportError）
   Mcu_Init 直接 return，没有执行 Mcu_pConfigPtr = pConfigPtr
        ↓
⑥ Mcu_pConfigPtr (0x1FFE_0400) 仍为 NULL
        ↓
⑦ Mcu_InitClock():
   Mcu_HLDChecksEntry(MCU_INITCLOCK_ID) 看到 status != 3 → 返回 E_OK
   Mcu_CheckInitClock() 读 [Mcu_pConfigPtr+8] = [0x8]（Flash 向量表 NMI 地址）
   ClockSetting(0) < 0x13A7 → 返回 E_OK
   然后：
     r1 = Mcu_pConfigPtr->Mcu_apClockConfig
        = *(uint32_t*)(0 + 16) = *(0x10) = 0x0000_1447（MemManage_Handler 向量）
     r0 = &ClockConfig[ Mcu_au8ClockConfigIds[0] ]
        = 0x1447 + 0 = 0x1447
        ↓
⑧ Mcu_Ipw_InitClock(0x1447):
   Mcu_pClockConfig = 0x1447
   SIM_pClockConfig = *(uint32_t*)(0x1447 + 4)
                    = 非对齐读取 0x144B
                    = 0x0280_00F3（由指令字节冒充的“指针”）
        ↓
⑨ Mcu_SIM_ClockConfig(0x0280_00F3):
   pClockConfigPtr->u32ClockGatingConfiguration
   = ldrb [0x0280_00F3 + 8] = ldrb [0x0280_00FB]
   0x0280_00FB 超出 1.5 MB Flash，属于非法地址
        ↓
   BusFault（精确总线错误），PC 停在 274 行
```

**证据**：

- `.map`：`Mcu_eStatus = 0x1FFE_040C`（`.mcal_data`），`Mcu_pConfigPtr = 0x1FFE_0400`（`.mcal_data`），`Mcu_ClockConfigPB0 = 0x1550`，`SIM_ClockConfigPB0_0 = 0x1588`。
- `Mcu_HLDChecksEntry` 反汇编（0x788）：`ldr r0,=0x1FFE040C; ldrb; cmp #3; beq ...`。
- `Mcu_Init` 反汇编（0x7EE）：检查失败后 `bne 0x862` 直接返回，跳过 `str r5,[r4]`（保存 `Mcu_pConfigPtr`）。
- `Mcu_InitClock` 反汇编（0x864）：`ldr r0,[&Mcu_pConfigPtr]; ldr r1,[r0,#16]; ... bl Mcu_Ipw_InitClock`。
- 向量表 `0x10` 处的值是 `0x0000_1447`（MemManage_Handler）；`0x1447+4 = 0x144B` 处是 `nop.w` 指令字节，组合成 `0x0280_00F3`。
- `Mcu_SIM_ClockConfig` 反汇编（0xA8A）第一条就是 `ldrb r1,[r0,#8]`，与 274 行 `pClockConfigPtr->u32ClockGatingConfiguration` 对应。

> 这也解释了为什么“改掉 `__NO_SYSTEM_INIT` 后还是在 274 行跑飞”：WDOG 不再是原因，但 `.mcal_data` 初始化缺失依然会让 `Mcu_Init` 静默失败。

### 10.3 修复方案

#### 方案 A（推荐）：让 IAR 的 init table 也执行

在 `init_data_bss()` 的末尾（或 `Reset_Handler` 中 `BL main` 之前）调用 IAR 库函数 `__iar_data_init3()`：

```c
/* startup.c */
extern void __iar_data_init3(void);   /* IAR 库：执行链接器生成的 init table */

void init_data_bss(void)
{
    /* ... 原有的 .data/.bss/... 处理 ... */

    /* 让 IAR 的 init table 生效（复制 .mcal_data、清 .mcal_bss） */
    __iar_data_init3();

    /* ... 原来的向量表复制/VTOR 设置 ... */
}
```

或者直接在 `Reset_Handler` 里把 `BL main` 改成 `BL __iar_program_start`（不推荐，因为会重复做一遍 C 库初始化；但可行）。

#### 方案 B（最小改动）：Reset_Handler 走 IAR 标准启动

```asm
        ; 原来：
        LDR     R0, =init_data_bss
        BLX     R0
        CPSIE   I
        BL      main

        ; 改为：
        LDR     R0, =__iar_program_start
        BLX     R0
```

`__iar_program_start` → `?main` → `__low_level_init` → `__iar_data_init3`（执行 init table）→ `main`。  
注意：这样 `init_data_bss` 仍会被调用（如果你保留它），会造成重复初始化；可以把 `init_data_bss` 的调用删掉，让库统一初始化。

#### 方案 C（不推荐，但可救急）：手动初始化 MCAL 段

在 `init_data_bss()` 里增加对 `.mcal_data` / `.mcal_data_init` / `.mcal_bss` 的处理：

```c
/* 示例：只处理 MCAL 段 */
uint8_t *mcal_ram  = __section_begin(".mcal_data");
uint8_t *mcal_rom  = __section_begin(".mcal_data_init");
uint8_t *mcal_end  = __section_end(".mcal_data_init");
while (mcal_end != mcal_rom) { *mcal_ram++ = *mcal_rom++; }

uint8_t *mcal_bss  = __section_begin(".mcal_bss");
uint8_t *mcal_bss_end = __section_end(".mcal_bss");
while (mcal_bss_end != mcal_bss) { *mcal_bss++ = 0; }
```

但 MCAL 的段名可能随 MemMap 配置变化（不同模块、不同 `MCU_START_SEC_*`），不如方案 A 稳。

#### 方案 D（仅用于验证问题）：确认 `Mcu_Init` 是否被跳过

在 `main()` 里 `Mcu_Init()` 之后，用调试器查看：

- `Mcu_eStatus`（`0x1FFE_040C`）是否等于 `3`；
- `Mcu_pConfigPtr`（`0x1FFE_0400`）是否等于 `&McuModuleConfiguration`（`0x150C`）。

如果 `Mcu_eStatus` 是 0、`Mcu_pConfigPtr` 是 0，就证实了本节的结论。

### 10.4 验证步骤（修复后）

1. 重新编译；
2. 在 `main` 开头（`Det_Init` 之前）查看 `Mcu_eStatus == 3`；
3. 单步执行 `Mcu_Init` 后查看 `Mcu_pConfigPtr == 0x150C`（`&McuModuleConfiguration`）；
4. 继续执行，`Mcu_InitClock` 传入的 `SIM_pClockConfig` 应为 `0x1588`（`SIM_ClockConfigPB0_0`）；
5. 若仍有异常，请把 `HardFault_Handler`/`BusFault_Handler` 改为记录 `SCB->CFSR/HFSR/BFAR/MMFAR` 的实现，而不是 `B .`。

### 10.5 其他注意点

1. **`SIM_LPOCLKS` 是 write-once 且仅在 POR/LVD 复位**（RM 11.3.1.4.2）。`Mcu_SIM_Init()` 会无条件写它；用调试器做非 POR 复位重复运行 `Mcu_Init` 时，第二次写可能被忽略（或产生总线错误，取决于实现）。建议用 `Mcu_GetResetReason()` 判断，只有 POR/LVD 才写，或读回比较。
2. **`SystemCoreClock` 未更新**：启动代码没有调用 `SystemCoreClockUpdate()`。如果应用依赖它，应在 `Mcu_InitClock()` 之后调用一次。
3. **`__NO_SYSTEM_INIT` 与 `START_FROM_FLASH` 只应加在 C 编译器还是汇编器？** 这两个宏只影响 `startup_S32K148.S`，应只加在 **Assembler**（AARM）的 Defined symbols 里；C 编译器不需要（`INIT_CLOCK`/`MCU_*` 是 MCAL 的配置宏）。
4. **栈大小**：当前 1 KB。MCAL + 调试打印（如果后续加 `printf`）可能不够，建议根据最坏调用深度调整 `__stack_size__`。
5. **`FlashConfig` 不要随意改**：`FSEC=0xFE` 是未加密；改成安全值会导致调试器无法连接。

---

## 11. 附录：常用地址/符号速查

### 11.1 内核与外设基址

| 名称 | 地址 |
|---|---:|
| Flash | `0x0000_0000` ~ `0x0017_FFFF` |
| Flash Configuration Field | `0x0000_0400` ~ `0x0000_040F` |
| SRAM_L | `0x1FFE_0000` ~ `0x1FFF_FFFF` |
| SRAM_U | `0x2000_0000` ~ `0x2001_EFFF`（ICF 使用范围） |
| SCB | `0xE000_ED00`（CPACR `+0x88`，VTOR `+0x08`，CFSR `+0x28`，HFSR `+0x2C`，MMFAR `+0x34`，BFAR `+0x38`） |
| WDOG | `0x4005_2000`（CS `+0x00`，CNT `+0x04`，TOVAL `+0x08`） |
| RCM | `0x4007_F000`（SRS `+0x08`，SSRS `+0x18`） |
| SIM | `0x4004_8000` |
| SCG | `0x4006_4000` |
| PORTA~PORTE | `0x4004_9000` 起 |

### 11.2 当前工程关键符号

| 符号 | 地址 | 说明 |
|---|---:|---|
| `__vector_table` | `0x0000_0000` | 向量表 |
| `Reset_Handler` | `0x0000_1328` | 复位入口 |
| `SystemInit` | `0x0000_13B0` | 系统初始化（WDOG 关闭） |
| `init_data_bss` | `0x0000_1058` | 数据段/向量表初始化 |
| `main` | `0x0000_1142` | 用户主函数 |
| `__iar_data_init3` | `0x0000_13A8` | IAR init table 执行函数（当前未被调用） |
| `?main` / `__cmain` | `0x0000_1414` | IAR 标准启动（当前未被调用） |
| `__iar_program_start` | `0x0000_1560` | ELF entry（硬件复位不走这里） |
| `Mcu_eStatus` | `0x1FFE_040C` | MCAL 驱动状态（初值应为 3） |
| `Mcu_pConfigPtr` | `0x1FFE_0400` | MCAL 配置指针 |
| `Mcu_pClockConfig` | `0x1FFE_0404` | MCAL 时钟配置指针 |
| `McuModuleConfiguration` | `0x0000_150C` | MCAL 配置数据 |
| `SIM_ClockConfigPB0_0` | `0x0000_1588` | SIM 时钟配置数据 |
| `Mcu_au8ClockConfigIds` | `0x2000_0000` | 时钟配置 ID 映射表 |
| `Mcu_au8ModeConfigIds` | `0x2000_0001` | 模式配置 ID 映射表 |
| `CSTACK` | `0x2001_EC00`~`0x2001_EFFF` | 栈 |

### 11.3 ICF 符号与内存关系（速记）

```text
__VECTOR_TABLE          = 0x0000_0000
__VECTOR_RAM            = 0x1FFE_0000
__RAM_VECTOR_TABLE_SIZE = 0x0000_0400
__RAM_START             = 0x1FFE_0000
__RAM_END               = 0x2001_EFFF
RW                      = 0x1FFE_0400 .. 0x1FFE_040F
ZI                      = 0x2000_0000 .. 0x2000_0008
CSTACK                  = 0x2001_EC00 .. 0x2001_EFFF
SP (sfe(CSTACK))        = 0x2001_F000
```

---

## 12. 一句话总结

- **进 `main` 之前**：硬件取 `[0x0]` 作 SP、`[0x4]` 作 PC → `Reset_Handler` 关中断、清寄存器、调 `SystemInit()`（若未被 `__NO_SYSTEM_INIT` 屏蔽）、做 ECC RAM 初始化、设 SP、调 `init_data_bss()` 复制/清零数据并搬向量表到 RAM、开中断、调 `main`。
- **链接文件**：把向量表钉在 `0x0`、FCF 钉在 `0x400`、代码放 Flash `0x410+`、RW 放 `0x1FFE_0400`、ZI 放 `0x2000_0000`、栈放 `0x2001_EC00~0x2001_EFFF`，并导出 `__VECTOR_TABLE/__VECTOR_RAM/__RAM_START/__RAM_END` 等符号给启动代码使用。
- **当前工程的两个启动级问题**：① `__NO_SYSTEM_INIT` 导致 WDOG 约 8 ms 复位（Debug 已修，Release 未修）；② 自定义启动只调用 `init_data_bss` + `main`，绕过了 IAR 的 `__iar_data_init3`，导致 `.mcal_data`（`Mcu_eStatus=3`）未初始化，`Mcu_Init` 静默失败、`Mcu_pConfigPtr` 为 NULL，最终在 `Mcu_SIM_ClockConfig` 第 274 行因垃圾配置指针触发 BusFault。

---

*本文由代码与当前构建产物静态分析生成；地址/数值以 `Debug/List/S32K1X_Project.map` 与 `Debug/Exe/S32K1X_Project.out` 为准。*
