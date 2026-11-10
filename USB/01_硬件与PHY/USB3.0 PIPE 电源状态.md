# USB3.0 PIPE 电源状态

Wiki：https://wiki.aixin-chip.com/pages/viewpage.action?pageId=270493254

链路层配套：[USB2.0 &&3.0 电源管理状态](https://wiki.aixin-chip.com/pages/viewpage.action?pageId=258590609)。

## 1. P 状态属于哪个层级

P0/P1/P2/P3 是 PIPE 接口定义的 PHY 电源状态，可以归入 USB 物理层的电源管理。PIPE 是控制器（MAC）与 PHY 之间的接口；P 状态是芯片内部接口的工作模式，不是线缆上双方交换的四种链路协议状态。

链路层的 U0/U1/U2/U3 说明链路能否传输、怎样进入低功耗及怎样恢复；控制器通过 PIPE 请求 PHY 进入相应 P 状态，PHY 据此执行内部省电操作。

配套阅读：[USB2.0 &&3.0 电源管理状态](https://wiki.aixin-chip.com/pages/viewpage.action?pageId=258590609)。

```text
链路层配套：[USB2.0 &&3.0 电源管理状态](https://wiki.aixin-chip.com/pages/viewpage.action?pageId=258590609)。
                   ↓
PIPE：控制器通过 PowerDown 请求 PHY 切换 P 状态
                   ↓
PHY：执行省电操作，通过 PhyStatus 反馈切换完成
```

控制器自身逻辑、SoC 时钟和电源域还可能另行省电，所以整颗 USB 子系统的功耗变化并非全部来自 PHY。

## 2. P0/P1/P2/P3 对比

以下针对 USB3 的传统 PIPE 并行接口；不同 PIPE 版本、接口架构及 PHY 实现需要结合实际手册确认。

| 状态 | 常见链路对应 | 正常高速数据收发 | PCLK | 电源行为及约束 |
| --- | --- | --- | --- | --- |
| P0 | U0，以及需要活动 PHY 的训练、恢复等状态 | 支持 | 运行 | PHY 内部时钟处于工作状态 |
| P1 | U1 | 暂停 | 必须运行 | 在收发空闲时进入，采用能满足快速退出要求的省电措施 |
| P2 | U2；也可用于 Rx.Detect、SS.Inactive | 暂停 | 必须运行 | 可关闭部分内部时钟；收发通道须空闲，完成反馈前 PCLK 和规定的直流共模须稳定 |
| P3 | U3、SS.Disabled | 暂停 | PHY 输出模式下关闭 | 并行接口进入异步模式，可关闭部分内部时钟；必须保留当前链路状态需要的检测和恢复能力 |

PCLK 是 PIPE 并行接口时钟，不能与 PHY 的 reference clock 或内部 PLL 输出时钟混为一谈。P3 关闭 PCLK 不代表可以直接关闭 reference clock，更不代表 PHY 整体断电。

## 3. P1 相比 P0 关掉什么

规范限定的是接口行为和恢复时序，并没有给所有厂商规定一份相同的 PLL、CDR、TX、RX 开关表。

- P1 保持 PCLK，可以门控部分内部逻辑或降低部分电路功耗，但必须满足快速退出要求。
- P2 允许关闭部分内部时钟，具体 PLL、CDR 和模拟模块怎样处理，由 PHY 实现及恢复时限决定。
- P3 可进一步省电；用于 U3 时仍需支持合法的唤醒和退出检测，不能按“所有电路全关”理解。

因此不能仅凭 P1/P2/P3 编号断言“PLL 开/关”。AX8860 ComboPHY 各状态实际关闭哪些模块，需要另查该 PHY 的实现资料。

## 4. 什么情况下进入

| 场景 | 链路层行为 | PHY 常见动作 |
| --- | --- | --- |
| 正常传输 | 保持 U0 | 保持 P0 |
| 短暂空闲，U1 已使能且进入请求获接受 | 进入 U1 | 控制器请求 P1 |
| 满足 U2 策略及协议进入条件 | 进入 U2 | 控制器请求 P2 |
| 主机安排端口挂起 | 进入 U3 | 控制器请求 P3 |
| 链路关闭或执行接收端检测 | SS.Disabled 或 Rx.Detect 等 | 根据对应状态使用 P3 或 P2 等 |

没有数据只是进入 U1/U2 的一个前提，还取决于软件使能、控制器策略与对端响应。U3 来自挂起流程，不是单纯把空闲计时继续延长。

P 状态由控制器请求，PHY 执行并反馈；不能把 PHY 理解成独立决定 USB 链路进入哪个 U 状态。

## 5. 切换握手与恢复

控制器改变 PowerDown 后，必须等待 PHY 完成反馈，再开始后续操作或新的电源切换。传统 USB PIPE 中，P0/P1/P2 之间通过单周期 PhyStatus 反馈完成。

P3 涉及 PCLK 停止，握手需要专门处理：

- PCLK 由 PHY 输出：进入时在停钟前拉高 PhyStatus，完全停钟后拉低；退出时拉高 PhyStatus，保持到 PCLK 稳定后再结束反馈。
- PCLK 由外部输入：PHY 通过规定的 PCLK 周期反馈可停钟或已恢复工作，不能套用输出模式的波形。

典型退出过程是：检测合法 LFPS/退出条件 → 恢复所需 PHY 电路和时钟 → 完成 PIPE 切换握手 → 完成链路恢复 → 返回 U0。LFPS 是低频周期信号，不是正常高速数据包；正常数据路径暂停不代表退出检测也停止。

状态也不必依次经过 P0 → P1 → P2 → P3。控制器可以按协议和 PIPE 版本允许的转换，直接请求 P0 → P2 或 P0 → P3。

## 6. Ux 与 Px 怎样对应

正常稳定低功耗场景下，常见配对是 U0/P0、U1/P1、U2/P2、U3/P3。它们在不同层级相互配合，但不能据此当作同一个状态：

1. P2/P3 还服务于部分非 U 状态，不能从 Px 唯一反推出 Ux。
2. PIPE 允许满足 USB 基础规范要求的 PHY 专用状态；实际映射要看控制器与 PHY 配置。
3. 转换期间请求与完成存在时间差，瞬间读取的 U/P 状态编号可能不同。

验证时应分别检查：链路是否进入目标 Ux、PHY 是否完成目标 Px 切换、实际时钟/功耗是否符合预期、退出后通信是否恢复。一个配置位已使能，不等于已进入对应状态；只观察到 Ux，也不足以证明预期的硬件省电措施全部生效。

## 7. 参考资料

- [Intel PIPE Architecture Specification Revision 7.1](https://cdrdv2-public.intel.com/643108/643108_PIPE_Arch_Spec_Rev_7_1.pdf#page=119)，§8.3.2 Power Management – USB Mode，印刷页 119–120。该规范同时涵盖 PCIe、USB4 等模式，本文采用 USB Mode 的定义。
- [Intel USB3 PIPE 接口规范](https://www.intel.com/content/dam/doc/white-paper/usb3-phy-interface-pci-express-paper.pdf)，USB 电源管理章节，可用于对照传统 USB3 PIPE 接口。
- [USB2.0 &&3.0 电源管理状态](https://wiki.aixin-chip.com/pages/viewpage.action?pageId=258590609)。
