# USB 3.x 休眠与唤醒逻辑

## 1. 概述

本文主要梳理 USB 3.x 的以下流程：

- 设备插入与拔出检测
- U1/U2 链路低功耗
- U3 Suspend
- Host Resume
- Device Remote Wake

链路训练过程不做深入展开，仅简化描述为：

```text
Rx.Detect
    -> Polling / LFPS
    -> Equalization
    -> Configuration
    -> U0
```

---

## 2. USB 3.x 主要链路状态

| 状态 | 说明 |
|---|---|
| U0 | 正常工作状态，可以进行数据传输 |
| U1 | 浅度链路低功耗状态，恢复延迟较小 |
| U2 | 深度链路低功耗状态，恢复延迟大于 U1 |
| U3 | 最深的链路低功耗状态，通常对应 USB Suspend |
| SS.Inactive | 链路异常或训练失败状态，不属于正常低功耗状态 |

> **说明：** U1/U2 是链路级低功耗状态，不等同于 USB Function Suspend。U3 通常与 USB Suspend 对应，但链路状态和设备功能状态在概念上仍有区别。

---

## 3. 插入检测

USB 3.x 的插入检测需要区分 Device 侧和 Host 侧。

### 3.1 Device 侧

Device 主要通过 `VBUS Valid` 判断 Host 是否存在。

基本流程如下：

```text
VBUS Valid
    -> Device 判断 Host 已连接
    -> 使能 SuperSpeed PHY
    -> 打开 Rx Termination
    -> 等待 Host 发起 Rx.Detect / Polling
```

VBUS 表示总线上存在 Host，同时决定 Device 是否允许连接到总线。

### 3.2 Host 侧

Host 是 VBUS 的提供者，因此不能通过 VBUS 判断 Device 是否插入。

Host 主要通过 PHY 的 `Rx.Detect` 检测对端的 Receiver Termination。

基本流程如下：

```text
Host 执行 Rx.Detect
    -> 检测到 Device 的 Rx Termination
    -> 进入 Polling
    -> 完成链路训练
    -> 进入 U0
```

### 3.3 Type-C 场景

在 USB Type-C 场景下，物理插入还可以通过 `CC Attach` 检测。

`CC Attach/Detach` 与 USB 3.x PHY 的 `Rx.Detect` 属于不同层面的机制：

- CC 用于连接器和端口层面的连接检测；
- Rx.Detect 用于 SuperSpeed 链路层面的对端检测。

---

## 4. 拔出检测

拔出检测不能简单概括为同时依赖 PHY Detect 和 VBUS。Host 与 Device 的检测方式存在差异。

### 4.1 Device 侧

Device 可以通过以下信息判断连接断开：

- VBUS 消失；
- Type-C 场景下发生 CC Detach；
- PHY 或链路状态异常；
- 对端终端消失。

其中，`VBUS Loss` 是 Device 判断 Host 或 Hub 已不存在的重要依据。

### 4.2 Host 侧

由于 Host 自身提供 VBUS，因此 VBUS 通常不能作为 Host 判断 Device 拔出的依据。

Host 主要通过以下信息识别拔出：

- PHY 检测到 Receiver Termination 消失；
- 链路错误或 Recovery 失败；
- Hub 或 Port 状态变化；
- Type-C 场景下发生 CC Detach。

### 4.3 U3 状态下的拔出检测

U3 状态下没有正常的高速数据活动，因此不能依赖普通数据包判断连接是否存在。

系统需要保留必要的低功耗检测能力，并结合以下信息识别拔出：

- PHY 低功耗连接检测；
- Port 状态变化；
- VBUS 状态；
- Type-C CC 状态。

Host 和 Device 的具体检测依据不同，不能统一归纳为必须同时依赖 PHY Detect 和 VBUS。

---

## 5. U1/U2 链路低功耗

U1/U2 是 USB 3.x 的链路级低功耗状态，主要用于在链路空闲时降低 PHY 功耗。

### 5.1 进入 U1/U2

链路双方都可以发起进入 U1/U2 的请求。

基本流程如下：

```text
链路处于 U0
    -> 链路空闲
    -> 一端发送 LGO_U1 或 LGO_U2
    -> 对端接受请求
    -> 双方进入 U1 或 U2
```

对端可以根据自身状态、延迟要求或当前事务选择接受或拒绝。

### 5.2 软件与硬件分工

驱动通常负责配置：

- 是否允许进入 U1/U2；
- U1/U2 Exit Latency；
- 链路空闲定时器；
- 链路电源管理策略；
- Device 或 Port 的相关能力。

配置完成后，通常由控制器硬件自动管理状态切换：

```text
Driver 配置 LPM Policy 和 Idle Timer
    -> Bus 空闲达到设定时间
    -> Controller 自动发送 LGO_U1/U2
    -> 对端接受
    -> 进入 U1/U2
```

正常运行过程中，软件通常不需要逐次参与 U1/U2 的进入和退出。

### 5.3 退出 U1/U2

当任意一方有数据需要发送时，可以发起退出。

```text
U1/U2
    -> 任意一端发起 LFPS
    -> PHY 恢复
    -> Recovery
    -> U0
```

U1 保留的 PHY 功能更多，因此退出速度更快；U2 关闭的模块更多，因此恢复时间更长。

---

## 6. U3 Suspend

U3 是 USB 3.x 最深的链路低功耗状态，通常与 USB Suspend 对应。

需要区分两个概念：

- **U3**：USB 3.x 链路状态；
- **Function Suspend**：USB 设备或功能层面的状态。

两者密切相关，但并非完全相同。

### 6.1 进入 U3

U3 只能由链路的 Downstream Port 发起。

在 Host 直连 Device 的场景下，通常由 Host 或 Root Port 发起：

```text
Host / Downstream Port
    -> 发送 LGO_U3
    -> Device / Upstream Port 接受
    -> 双方进入 U3
```

Device 不能自行决定让整条链路进入 U3。

### 6.2 U3 下的 PHY 状态

进入 U3 后，PHY 可以关闭大部分高功耗模块，例如：

- 高速发送电路；
- 高速接收电路；
- CDR；
- 均衡模块；
- PLL；
- 大部分数字链路逻辑。

具体关闭范围取决于控制器和 PHY 的实现。

同时，必须保留必要的低功耗检测能力，包括：

- LFPS 检测；
- 唤醒逻辑；
- 连接状态变化检测；
- 必要的 VBUS、CC 或 Port Event 检测。

因此，U3 下可以关闭 PHY 的大部分功能，但不能关闭 LFPS 唤醒和连接事件检测所依赖的 Always-On 电路。

---

## 7. Host Resume

当 Host 需要恢复链路时，由 Downstream Port 发起 U3 Exit。

基本流程如下：

```text
U3
    -> Host 发送 LFPS
    -> Device 检测到 LFPS
    -> Device 恢复 PLL、CDR、Rx、Tx 等 PHY 模块
    -> 双方完成 U3 Exit 相关握手
    -> 进入 Recovery
    -> 恢复到 U0
```

Device 在检测到 LFPS 后恢复 PHY，但不会单方面直接进入 U0。链路双方仍需完成对应的退出和恢复流程。

---

## 8. Device Remote Wake

Remote Wake 是由 Device 侧发起的链路唤醒。

基本流程如下：

```text
U3
    -> Device 产生远程唤醒事件
    -> Device 发送 LFPS
    -> Host 或 Hub 检测到 LFPS
    -> Host 或 Hub 恢复 PHY 和链路逻辑
    -> 双方进入 Recovery
    -> 返回 U0
    -> Host 软件处理 Remote Wake 事件
```

Device 发起 Remote Wake 通常需要满足以下条件：

- Device 支持 Remote Wake；
- Host 已允许 Device 使用 Remote Wake；
- 当前 Device 或 Function 处于允许唤醒的 Suspend 状态；
- 唤醒信号满足规范要求的时序。

如果链路中存在 USB Hub，Remote Wake 会通过 Hub 逐级向上游传播，而不是直接到达 Root Port。

---

## 9. 流程汇总

### 9.1 插入与链路初始化

#### Device 侧

```text
VBUS Valid
    -> 使能 PHY
    -> 打开 Rx Termination
    -> 等待 Host 发起链路检测
```

#### Host 侧

```text
Rx.Detect
    -> Polling / LFPS
    -> Equalization
    -> Configuration
    -> U0
```

### 9.2 U1/U2 进入与退出

#### 进入 U1/U2

```text
U0
    -> 链路空闲
    -> 任意一端发送 LGO_U1/U2
    -> 对端接受
    -> U1/U2
```

#### 退出 U1/U2

```text
U1/U2
    -> 任意一端产生数据发送需求
    -> 发起 LFPS
    -> Recovery
    -> U0
```

### 9.3 U3 Suspend 与 Host Resume

#### 进入 U3

```text
Host / Downstream Port
    -> 发送 LGO_U3
    -> Device 接受
    -> U3
    -> PHY 大部分模块关闭
    -> 保留 LFPS 和连接事件检测
```

#### Host Resume

```text
U3
    -> Host 发送 LFPS
    -> Device 恢复 PHY
    -> Recovery
    -> U0
```

### 9.4 Device Remote Wake

```text
U3
    -> Device 产生远程唤醒事件
    -> Device 在获得授权后发送 LFPS
    -> Host / Hub 唤醒
    -> Recovery
    -> U0
```

### 9.5 拔出检测

#### Device 侧

```text
VBUS Loss
    / CC Detach
    / PHY 或链路状态变化
        -> 判断连接断开
```

#### Host 侧

```text
Receiver Termination 消失
    / CC Detach
    / PHY 或链路状态变化
        -> 判断 Device 拔出
```