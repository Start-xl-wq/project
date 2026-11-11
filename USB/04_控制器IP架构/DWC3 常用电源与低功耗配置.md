# DWC3 常用电源与低功耗配置

Wiki：https://wiki.aixin-chip.com/pages/viewpage.action?pageId=270496156

配套阅读：[USB2.0 &&3.0 电源管理状态](https://wiki.aixin-chip.com/pages/viewpage.action?pageId=258590609)、[USB2.0 UTMI 电源管理](https://wiki.aixin-chip.com/pages/viewpage.action?pageId=270493416)、[USB3.0 PIPE 电源状态](https://wiki.aixin-chip.com/pages/viewpage.action?pageId=270493254)。

本文按控制器协议、PHY 接口、事件通知和参考时钟四个层次整理 DWC3 常用配置。寄存器偏移使用 DWC3 相对寄存器窗口偏移，具体位定义需结合控制器 IP revision 和 PHY 集成手册确认。

## 常用配置索引

| 分类 | 配置 | 作用 |
| --- | --- | --- |
| USB2 协议能力 | `snps,usb2-lpm-disable` / `snps,usb2-gadget-lpm-disable` | 分别禁止 Host 硬件 LPM 和 Device LPM 能力；不禁止传统 L2 Suspend |
| USB2 Device 能力 | `DCFG.LPM_CAP` | 允许 Device 响应 USB2 LPM 请求 |
| USB2 PHY | `GUSB2PHYCFG.SUSPHY` / `ENBLSLPM` | 分别控制 PHY 挂起和 L1 省电信号传递 |
| USB2 时钟 | `GUSB2PHYCFG.U2_FREECLK_EXISTS` | 声明真实的自由运行时钟连接 |
| L1 退出 | `GUCTL1.DEV_L1_EXIT_BY_HW` | 在适用条件下自动发起 L1 远程唤醒 |
| L1 深度 | `DCTL.HIRDTHRES` | 根据 HIRD 选择 L1 PHY 省电控制路径 |
| LPM 响应 | `DCTL.LPM_NYET_THRES` | LPM erratum 机制下的 BESL 响应阈值 |
| USB3 权限 | `DCTL.INITU1/2ENA` / `ACCEPTU1/2ENA` | 主动发起与接受 U1/U2 请求 |
| PIPE 切换 | `REQP1P2P3` / `DELAYP1TRANS` / `DelayP1P2P3` | 是否省略、何时发起 PHY 电源切换 |
| PIPE 接收 | `GUSB3PIPECTL.LFPSFILTER` | P0 中对 LFPS 指示进行资格过滤 |
| U2 与 P3 | `GUSB3PIPECTL.U2P3ok` / `GUCTL1.P3_IN_U2` | 依 IP 版本配置链路与 PHY 电源映射 |
| 事件 | `DEV_DECOUPLE_L1L2_EVT` / `DEVTEN` | 分离、使能或屏蔽状态通知 |
| Host 调度 | `PARKMODE_DISABLE_SS/HS` | park 调度兼容控制，与链路省电无关 |
| 全局与帧计时 | `GCTL` / `GUCTL.REFCLKPER` / `GFLADJ` | 角色、核心控制、参考时钟和定时补偿 |

布尔 DTS 属性写上表示 true；带 `disable` 的属性通常表示禁用某功能。但不写属性不一定会把寄存器位清零：有些初始化只在属性存在时置位，最终值还取决于复位值和其他代码路径。

## 1. USB2 LPM 能力

### 1.1 `DCFG.LPM_CAP`

`DCFG[22]` 决定 USB2 Device 是否具备响应 Host LPM Token 的协议能力：

* `0`：不接受 LPM 请求，不能通过 LPM 进入 L1；
* `1`：允许控制器根据当前 FIFO、端点和协议条件返回 ACK、NYET 或 STALL。

置位不等于强制进入 L1。L1 仍由 Host 发起，Device 只决定是否接受本次请求。`snps,usb2-gadget-lpm-disable` 会使该能力关闭。

### 1.2 USB2 PHY 省电

`GUSB2PHYCFG.SUSPHY`（bit 6）用于在链路满足挂起条件时允许 USB2 PHY 进入挂起模式，常见于传统 USB2 L2 Suspend。它不决定链路是否进入 L2，也不等于整颗 PHY 断电。

`GUSB2PHYCFG.ENBLSLPM`（bit 8）允许控制器在 USB2 L1 期间把 `utmi_sleep_n` 和 `utmi_l1_suspend_n` 这类控制器到 PHY 的硬件省电信号传给外部 PHY。它影响 PHY 是否执行对应的 L1 省电动作，不是 LPM 能力开关。

`GUSB2PHYCFG.U2_FREECLK_EXISTS`（bit 30）声明 USB2 PHY 是否提供自由运行时钟，必须与实际 UTMI 时钟连接一致。

## 2. USB3 Device U1/U2

`DCTL.INITU1ENA` 和 `DCTL.INITU2ENA` 分别允许 USB3 Device 主动发起 U1/U2；`DCTL.ACCEPTU1ENA` 和 `DCTL.ACCEPTU2ENA` 分别允许 Device 接受 Host 发来的 U1/U2 请求。

这些位由软件根据 USB control request（例如 `SetFeature(U1_ENABLE/U2_ENABLE)`）和运行策略配置，之后每次 U0 与 U1/U2 之间的链路握手由硬件 LTSSM 完成，不需要软件逐次参与。USB Reset 会清除主动发起位。置位只是允许发起请求，仍须对端接受，不能单方面强制进入。

U1/U2 的进入使用 Link Commands（例如 LGO_U1/LGO_U2 与相应接受/拒绝握手）；LFPS 用于低功耗退出等过程，不能把进入描述成 LFPS 握手。

## 3. USB2 L1 的 HIRD 阈值

`DCTL.HIRDTHRES`（bit 28:24）是软件配置的比较门限；控制器将 LPM Token 中的 HIRD 与门限比较，决定 USB2 L1 期间向 PHY 输出哪一种省电控制信号。HIRD 是 Host Initiated Resume Duration，描述 Host 发起恢复时的信令时长，不表示预计睡眠多久。它不是 L1 空闲超时，也不直接指定 PHY 电压。采用 BESL 的机制需按对应规范和 IP 文档解释，不能直接套用 HIRD 的线性时间关系。

当启用深度 L1 判断时，HIRD 达到配置的低四位阈值，可使 `utmi_l1_suspend_n` 有效；未达到阈值时通常使用较浅的 `utmi_sleep_n` 路径。具体 PHY 对这两个信号的时钟和电路处理由 PHY 实现决定。SuperSpeed 操作时该字段应为 0。

| `HIRDTHRES[4]` | 比较结果 | L1 中的控制路径 |
| --- | --- | --- |
| 0 | 任意 | `utmi_sleep_n` |
| 1 | HIRD 小于 `[3:0]` | `utmi_sleep_n` |
| 1 | HIRD 大于等于 `[3:0]` | `utmi_l1_suspend_n`，允许更深的低功耗 |

这两个 `_n` 信号低电平有效，属于控制器到 PHY 的内部信号；还须由 `ENBLSLPM` 允许传递。例如低四位为 8、bit 4 为 1 时，收到 HIRD 编码 6 选择 sleep 路径，收到编码 10 选择 l1_suspend 路径。这里比较的是编码，不表示睡眠 6 或 10 微秒。

`LPM_NYET_THRES` 是另一个字段：启用对应 LPM erratum 时，本地 3.30b 手册描述需要满足 FIFO 条件且 Token 中 BESL 不大于此门限才 ACK。它影响是否接受 LPM，不应与选择 PHY 省电深度的 HIRD 门限混淆；该区域与其他响应位存在版本/配置复用。

## 4. L1 退出和事件

`GUCTL1.DEV_L1_EXIT_BY_HW`（bit 24）允许 USB2 Device 在 L1 中已有 Bulk/Interrupt 数据收发条件时，由硬件自动发送远程唤醒信号并退出 L1。退出完成后链路回到活动状态，Host 再调度实际传输。

自动唤醒仍需当次 LPM 允许 Device remote wakeup，并满足控制器端点就绪条件。应用层有数据不等于数据已经排到硬件端点。此功能不适用于 Control/Isochronous；L1 hibernation 另有限制。当前通用驱动按 IP 版本判断设置 bit 24，没有独立 DTS 开关。

`GUCTL1.DEV_DECOUPLE_L1L2_EVT`（bit 31）将 USB2 L1 挂起/恢复事件与 L2/U3 挂起/恢复事件分开，使 `DEVTEN` 可以单独屏蔽频繁的 L1 事件。它只改变事件通知，不禁止 L1 进入或退出。

L1 恢复示意：Device 端点就绪 → 在获得许可时发起恢复信令 → Host 接续恢复 → 恢复信令完成、链路回到 L0 → Host 发 IN token → Device 返回数据。发出唤醒只表示开始退出，不等于已回到 L0。

## 5. USB3 PIPE 与 PHY 电源切换

`GUSB3PIPECTL` 用于配置 USB3 控制器与 PIPE PHY 的配合：

* `REQP1P2P3`（bit 24）：U0 进入 U1/U2/U3 时始终请求对应 P1/P2/P3，即使发生立即退出也不省略请求；
* `DELAYP1TRANS`（bit 18）和 `DEP1P2P3`（bit 21:19）：等待 PIPE 接收状态或指定错误计数后再切换 PHY 电源状态；
* `LFPSFILTER`（bit 9）：在 P0 下结合 `RxElecIdle`/`RxValid` 过滤 LFPS 接收指示；是否开启应由 PHY 集成要求或已确认的误报问题决定；
* `U2P3OK`（bit 29）：在具体 IP/PHY 支持时允许特定 U2 或 SS.Inactive 场景使用 P3，必须核对对应版本手册和 PHY 退出延迟。

U1/U2/U3 是链路状态，P1/P2/P3 是 PIPE PHY 电源状态，两者由控制器请求和 PHY 完成反馈配合，并非固定一一等价。

补充边界：

* `REQP1P2P3` 不是 P1→P2→P3 依次切换；它控制是否省略进入对应 Px 的请求。
* `LFPSFILTER=1` 时，在 P0 中只有 `RxElecIdle=0` 且 `RxValid=0` 才允许接收 LFPS 指示；这些条件本身并不构成 LFPS 检测器。它不是模拟滤波，也不是省电总开关。
* `SUSPENDENABLE/SUSPHY`（bit 17）允许满足条件时挂起 USB3 PHY；不表示写入后立即进入 U3。
* `DisRxDetP3`（bit 28）要求在需要接收器检测时先由 P3 切到 P2，再检测并返回。
* 本地 3.30b 手册对 bit 29 只明确描述 SS.Inactive 的 P2/P3 选择，同时在 `GUCTL1.P3_IN_U2[25]` 明确描述 U2 用 P3；其他 DWC3 集成资料对 bit 29 又包含 U2。应核对目标 IP，不能混写为一个无条件通用开关。
* 深度省电必须满足 PHY LFPS 能力、时钟和退出时延，不能只因想省电就开启 P3 映射。

## 6. 事件和 Host 调度

`DEVTEN` 控制 Device 事件是否生成。解耦后，`L1SUSPEN` 和 `L1WKUPEVTEN` 可独立控制 L1 事件；`U3L2L1SUSPEN` 和 `WKUPEVTEN` 则用于 U3/L2 事件。事件屏蔽只减少软件通知，不阻止硬件状态机运行。

| `DEVTEN` 位 | 未解耦 | 解耦后 |
| --- | --- | --- |
| bit 6 | U3/L2/L1 挂起 | U3/L2 挂起 |
| bit 4 | U3/L2/L1 恢复检测 | U3/L2 恢复检测 |
| bit 8 | 不作为独立 L1 挂起控制 | L1 挂起 |
| bit 14 | 不作为独立 L1 恢复控制 | L1 恢复检测 |
| bit 3 | 通用链路状态变化 | 通用链路状态变化 |

bit 号不是事件 type。`LINK_STATUS_CHANGE`（type 3）和专门的 `SUSPEND`（type 6）是不同事件，前者也可以携带 USB2 L2/USB3 U3 状态，两者并不互斥。当前较新版本默认不持续开启通用状态变化事件，保留专门挂起事件；软件远程唤醒时可临时开启通用状态变化以观察恢复。

USB3 U1/U2 可以通过通用链路状态变化事件观察，目前接口没有各自的专用使能。正常运行通常由硬件自主处理进出，验证时需要另行设计观察方式。不能由此推断 USB 协议不允许这些状态触发中断，或推断 L1 专用事件是错误设计。

`GUCTL1.PARKMODE_DISABLE_SS/HS` 仅用于 Host 调度兼容性。HS 位置 1 会关闭 park 多包流水线，改为较保守的单包调度，可能改善特定慢设备连续 NAK 时的性能；它与 USB 链路低功耗无关。

SS 位在本地手册中明确用于调试；不能直接套用 HS 调度包数和 NAK 行为，更不能认为置位普遍提高性能。

## 7. 全局、参考时钟和帧计时

`GCTL`（`0xc110`）负责角色、核心复位、时钟门控和 hibernation 等全局行为；`GUCTL.REFCLKPER`（`0xc12c` bit 31:22）描述参考时钟周期；`GFLADJ`（`0xc630`）提供帧长度和参考时钟补偿。三者应与 DTS 中的 `ref` 时钟及通用 DWC3 参考时钟配置配套使用，不能只修改其中一个字段。

## 8. 验证原则

验证时分别确认：协议是否允许进入目标状态、控制器是否发起状态请求、PHY 是否完成目标电源切换、事件是否被使能并上报，以及恢复后的枚举和传输是否正常。读取配置位只能证明配置已写入，不能单独证明链路或 PHY 已实际进入目标状态。

本文是配置机制说明，不代表已完成 AX8860 的 L1/L2/U1/U2/U3 板端验证。不建议运行中直接改写准静态位；端点命令前临时恢复 PHY 时钟等操作应遵循通用驱动流程。

## 9. 参考与源码定位

* Synopsys DWC_usb3 Programming Guide 3.30b，§1.2.5 GCTL（原 PDF 第 58 页）、§1.2.8 GUCTL1（第 74 页起）、§1.2.12 GUCTL（第 90 页）、§1.2.45 GUSB2PHYCFG（第 163 页起）、§1.2.49 GUSB3PIPECTL（第 181 页起）、§1.2.64 GFLADJ（第 210 页）、§1.3.1 DCFG（第 217 页）、§1.3.2 DCTL（第 221 页起）、§1.3.3 DEVTEN。资料版本不代表目标 IP 版本。
* 当前工程 Linux 6.18.33：`drivers/usb/dwc3/core.c` 中属性读取、PHY 设置、参考时钟及 GUCTL1 初始化；`gadget.c` 中 LPM 能力、端点命令与事件处理；`ep0.c` 中 U1/U2 Feature 处理；`core.h` 中寄存器和事件编码。
* `Documentation/devicetree/bindings/usb/snps,dwc3-common.yaml` 与 `snps,dwc3.yaml`：属性角色和含义。
