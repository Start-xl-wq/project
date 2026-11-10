# USB2.0 UTMI 电源管理

Wiki：https://wiki.aixin-chip.com/pages/viewpage.action?pageId=270493416

链路层配套：[USB2.0 &&3.0 电源管理状态](https://wiki.aixin-chip.com/pages/viewpage.action?pageId=258590609)。

## 1. 结论与层级

USB2 的 UTMI/UTMI+ 有 PHY 电源管理控制，最基本的是正常工作与 Suspend，但没有与 USB3 PIPE P0/P1/P2/P3 相同的标准四级编号。USB2 L0/L1/L2/L3 是链路/总线电源状态，不能给它们机械配上 P0/P1/P2/P3。

UTMI 是控制器与 USB2 PHY 的接口；UTMI+ 对原接口扩展，ULPI 是另一种低引脚数接口，三者不应混称。LPM L1 的支持及 Sleep 信号还须结合后续扩展、PHY 型号和控制器集成。

## 2. 原始 UTMI：SuspendM

| SuspendM | 含义 | PHY 行为 |
| --- | --- | --- |
| 1 | 不挂起、正常工作 | 正常收发工作模式 |
| 0 | 请求 Suspend，低有效 | 降低功耗，保留 Suspend/Resume 所需功能 |

`M` 表示低有效的命名约定。SuspendM 由控制器/SIE 驱动，PHY 执行请求；它不是总线上的协议报文，也不是 PHY 自主决定 L 状态的接口。

挂起期间可停止不需要的时钟和电路，但仍需保持必要的线状态检测及连接指示。Intel UTMI 的设备挂起流程要求适当设置 TermSelect 等控制以维持所需的连接终端；不能把 SuspendM 单独理解成可以撤销 pull-up 或彻底断电。

## 3. CLK 与 LineState

UTMI 的 CLK 用于控制器与 PHY 的并行数据传输，典型为 8-bit 的 60 MHz 或 16-bit 的 30 MHz；它不等于 PHY 外部参考时钟。

Suspend 中允许停掉工作时钟，恢复要等待时钟达到可用条件。LineState 反映 D+/D− 单端接收器的状态；原始 UTMI 在没有可用 CLK 时提供组合反馈，时钟可用后再同步。因此时钟暂停期间仍能保留必要的 Resume/Reset 检测路径。

低功耗不是完全断电：如果连检测和恢复所需的电路也关闭，便不能按正常 Suspend/Resume 流程保持连接并返回工作态。

## 4. L0/L1/L2/L3 如何落到 PHY

| USB2 状态 | PHY 控制的理解 | 不能据此断言 |
| --- | --- | --- |
| L0 | 正常工作；SuspendM 通常不有效 | PHY 内部毫无省电优化 |
| L1 | 使用支持 LPM 的控制器/PHY 的 Sleep 或 L1 Suspend 集成方式 | 原始 UTMI 已定义统一 L1 控制或固定停钟方式 |
| L2 | 控制器完成 Suspend 条件识别后请求 PHY 挂起 | PHY 整体掉电或连接状态丢失 |
| L3 | 断开/关闭/掉电场景，结合平台电源控制处理 | 只拉低 SuspendM 就等于 L3 |

L1 恢复时限与传统 L2 不同，不能不检查恢复能力就给两者使用相同的 PHY 深度。

## 5. DWC3 的 Sleep/Suspend 集成示例

部分 DWC3/USB2 PHY 集成使用 `utmi_sleep_n`、`utmi_suspend_n`、`utmi_l1_suspend_n`，PHY 侧还有 `SLEEPM`、`SUSPENDM` 或等效输入。具体连接并非一套通用于所有芯片的固定映射。

例如 AMD 的 DWC3 集成说明区分持续保留 FREECLK 与允许停钟的 PHY 连接方式，涉及 `GUSB2PHYCFG.U2_FREECLK_EXISTS` 与 `GUCTL1.OVRLD_L1_SUSP_COM`。这说明 L1 与 L2 的控制路径、PLL 和时钟保留方式取决于集成配置。

该示例不能直接当作 AX8860 接线结论。AX8860 须进一步核对实际 PHY 手册、连线和寄存器定义。`snps,dis-u2-freeclk-exists-quirk` 中的 u2 指 USB2，不能理解为 SuperSpeed 链路 U2。

## 6. 进入和恢复流程

```text
控制器识别 L2 Suspend，或接受 L1 LPM
             ↓
按实际接口请求 PHY Suspend/Sleep
             ↓
PHY 进入低功耗，保留必要检测
             ↓
检测合法 Resume/Reset 等条件
             ↓
恢复 PHY、等待时钟可用，按对应协议继续处理
```

Resume、Reset、断开是不同事件，不能都视为“无须重新枚举的唤醒”。USB2 使用 D+/D− 线信号，不采用 USB3 LFPS；原始 UTMI 也没有 PIPE PowerDown + PhyStatus 那套 P 状态完成握手。

## 7. 验证方法与参考

分别确认控制器 L 状态、Suspend/Sleep 请求、真实 PHY 电源状态、工作时钟以及恢复后通信；不能仅凭一个 quirk 或使能位推断已降低功耗。

- [Intel UTMI Specification](https://www.intel.com/content/dam/www/public/us/en/documents/technical-specifications/usb2-transceiver-macrocell-interface-specification.pdf)：§4.1 接口信号、Suspend/Resume 和时钟恢复章节。
- [Microchip UTMI+ 接口说明](https://onlinedocs.microchip.com/oxy/GUID-199548F4-607C-436B-80C7-E4F280C1CAD2-en-US-1/GUID-72160C1C-93F7-4B78-BB3B-B9E1C3A0B117.html)：控制器/PHY 接口实例。
- [AMD GUCTL1 USB3_XHCI](https://docs.amd.com/r/en-US/ug1087-zynq-ultrascale-registers/GUCTL1-USB3_XHCI-Register)：Sleep/Suspend 和 FREECLK 集成示例。
- [USB3.0 PIPE 电源状态](https://wiki.aixin-chip.com/pages/viewpage.action?pageId=270493254)：对照 USB3 PHY 电源管理。
