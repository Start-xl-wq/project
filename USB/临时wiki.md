# USB 电源管理状态 Wiki

> USB 2.0 与 USB 3.x 链路电源状态（Link Power Management）完全整理

---

## 目录

1. [概述](#1-概述)
2. [USB 2.0 链路状态（L-States）](#2-usb-20-链路状态l-states)
3. [USB 3.x 链路状态（U-States）](#3-usb-3x-链路状态u-states)
4. [PIPE PHY 电源状态](#4-pipe-phy-电源状态)
5. [状态对应关系](#5-状态对应关系)
6. [Wakeup 机制](#6-wakeup-机制)
7. [电流与时序约束](#7-电流与时序约束)
8. [Controller 时钟电源管理](#8-controller-时钟电源管理)
9. [软件实现要点](#9-软件实现要点)
10. [常见问题排查](#10-常见问题排查)

---

## 1. 概述

USB 通过分级的链路电源状态实现节能。核心原则：

- **状态越深，越省电，但退出延迟越大**
- **除物理断开或 VBUS 掉电外，所有低功耗状态都保持连接态，不需要重新枚举**
- **省电 vs 快速恢复是核心矛盾**，USB3 通过多档细分状态来平衡

| 版本 | 状态数 | 切换粒度 | PHY 类型 | Wakeup 检测 |
|---|---|---|---|---|
| USB 2.0 | L0/L1/L2/L3 | ms 级 | 模拟为主 | LineState 比较器 |
| USB 3.x | U0/U1/U2/U3 | μs 级 | SerDes | LFPS detector |

---

## 2. USB 2.0 链路状态（L-States）

### 2.1 状态定义

| 状态 | 名称 | 描述 | PLL | 连接保持 |
|---|---|---|---|---|
| **L0** | On (Active) | 正常工作，全速数据传输 | 开 | ✓ |
| **L1** | Sleep (LPM) | 轻度睡眠，快速唤醒 | 开 | ✓ |
| **L2** | Suspend | 深度睡眠，标准 suspend | 关 | ✓ |
| **L3** | Off | 断电/断开 | 关 | ✗ |

### 2.2 L1 (LPM) 详解

- **触发**：Host 通过 **LPM token 事务**（LPM + EXT token）发起
- **协商**：Device 可 ACK / NAK / STALL
- **退出延迟**：由 `HIRD` (Host Initiated Resume Duration) 字段指定，50 μs - 10 ms
- **退出机制**：K-state signaling
- **特点**：单一档位，仅 host 可发起

### 2.3 L2 (Suspend) 详解

- **触发**：总线 idle > 3 ms 自动进入
- **总线行为**：
  - HS device **revert 到 FS**（移除 HS termination）
  - D+ pull-up (1.5kΩ) **必须保持**（维持 connect 状态）
- **退出机制**：Host 发 K-state ≥ 20 ms（Resume signaling）
- **电流约束**：bus-powered device ≤ **2.5 mA**
- **Remote Wakeup**：Device 发 K-state (1-15 ms) 唤醒 host

### 2.4 L2 恢复时序（关键红线）

```
Host 发 K-state ≥ 20ms
    ↓
Device 必须在 20ms 内完成完整唤醒并准备好通信
    ↓
HS device 需完成 HS Chirp handshake
```

**20ms 是 suspend 深度的红线**：

| 恢复动作 | 典型耗时 |
|---|---|
| PLL 重锁 | 10-100 μs |
| XTAL 起振 | 1-5 ms |
| PHY 上电稳定 | ~ms |
| Controller 重新初始化 | ~ms |
| **总预算** | **< 20 ms** |

---

## 3. USB 3.x 链路状态（U-States）

### 3.1 状态定义

| 状态 | 名称 | 描述 | PLL | 退出延迟 | 发起方 |
|---|---|---|---|---|---|
| **U0** | Active | 正常工作 | 开 | — | — |
| **U1** | Standby (Fast Exit) | 浅睡眠，PLL 保留 | **开** | **<10 μs** | 双向 |
| **U2** | Standby (Slow Exit) | 深睡眠，PLL 关闭 | **关** | **<2 ms** | 双向 |
| **U3** | Suspend | 标准 suspend | 关 | ~ms | Host（device 可 remote wake）|

### 3.2 U1 详解

- **核心特点**：**PLL 保持锁定**（为了 <10 μs 快速退出）
- TX driver 关闭（进入 electrical idle）
- LFPS detector 必须开启
- **PIPE clock (250MHz) 保留**（否则 controller 无法感知 PHY 状态）
- 对应 PIPE 状态 **P0s**

### 3.3 U2 详解

- **核心特点**：**PLL 关闭**（深度省电）
- TX/RX 完全关闭
- LFPS detector 保留（AON 域）
- PIPE clock 停止
- Controller 寄存器必须 retention
- 对应 PIPE 状态 **P1**

### 3.4 U3 详解

- **核心特点**：类似 USB2 L2 的深度 suspend
- SS PHY 大部分模拟电路可关
- LFPS detector + VBUS detector 保留（AON）
- **连接保持依赖 USB2 pull-up**（SS 通道 U3 时电气 idle）
- 电流约束：bus-powered ≤ **2.5 mA**
- 对应 PIPE 状态 **P2/P3**

### 3.5 U1/U2 自动下沉机制（USB2 无）

```
U0 无传输
    ↓ U1 inactivity timer 到期（如 10 μs）
    发送 LGO_U1 → 对方 LAU (accept)
    ↓
进入 U1
    ↓ U2 inactivity timer 到期（如 256 μs）
    直接下沉
    ↓
进入 U2
```

- **双向发起**：device 和 host 都可发 `LGO_Ux`
- **链路级**（非事务级），link 空闲时自动省电

### 3.6 U 状态 Link Command

| 命令 | 含义 |
|---|---|
| `LGO_U1/U2/U3` | 请求进入 Ux |
| `LAU` | Link Accept U（接受）|
| `LXU` | Link Reject U（拒绝）|
| `LPMA` | Link Power Management ACK |

---

## 4. PIPE PHY 电源状态

PIPE (PHY Interface for PCIe and USB) 定义 PHY 侧电源状态，由 controller 通过 `PowerDown[]` 控制：

| PIPE State | 名称 | 对应 U-State | PHY 内部 |
|---|---|---|---|
| **P0** | Normal Operational | U0 | 全开：PLL/CDR/TX/RX |
| **P0s** | Low Recovery Latency | U0 idle / U1 | TX 关，PLL 保留 |
| **P1** | Longer Recovery Latency | U2 | TX/RX 关，**PLL 关** |
| **P2** | Lowest Power | U3 | 深度掉电，保留 LFPS det |
| **P3** | (PIPE 4.x+) | U3 深度 | 几乎完全掉电 |

### PIPE 信号线

```
Controller ──PowerDown[2:0]──► PHY      控制电源状态
Controller ──TxDetectRx/Loopback──► PHY
PHY ──PhyStatus──► Controller           状态确认
PHY ──RxElecIdle──► Controller          检测 electrical idle
```

---

## 5. 状态对应关系

### 5.1 总览映射

```
USB2                    USB3
─────                   ─────
L0 (Active)     ═══════ U0 (Active)          完全对应
L1 (LPM)        ═══╤═══ U1 (PLL 保留)         部分对应
                   └═══ U2 (PLL 关闭)         (USB3 细分两档)
L2 (Suspend)    ═══════ U3 (Suspend)          完全对应
L3 (Off)        ═══════ SS.Disabled/Inactive  部分对应
```

### 5.2 完整对比表

| 维度 | L0 | L1 | L2 | U0 | U1 | U2 | U3 |
|---|---|---|---|---|---|---|---|
| 活动状态 | Active | Idle | Suspend | Active | Idle浅 | Idle深 | Suspend |
| PLL | 开 | 开 | 关 | 开 | 开 | 关 | 关 |
| 退出延迟 | — | 50μs-10ms | >20ms | — | <10μs | <2ms | ~ms |
| 发起方 | — | Host | Host | — | 双向 | 双向 | Host |
| 连接保持 | ✓ | ✓ | ✓ | ✓ | ✓ | ✓ | ✓ |
| Address 保留 | ✓ | ✓ | ✓ | ✓ | ✓ | ✓ | ✓ |
| 需重新枚举 | ✗ | ✗ | ✗ | ✗ | ✗ | ✗ | ✗ |
| Runtime PM | — | ✓ | ✓ | — | ✓ | ✓ | ✓ |
| 电流上限(bus-powered) | 按config | 按config | 2.5mA | 按config | 按config | 按config | 2.5mA |

### 5.3 关键差异说明

**L1 ≠ 完全等价 U1/U2**：
- L1 是**单档**，U1/U2 是**两档**
- L1 只能 **host 发起**，U1/U2 是**双向**发起
- L1 是**事务级**（LPM token），U1/U2 是**链路级**（Link Command）
- U1/U2 有 **inactivity timer 自动下沉**机制，L1 没有

**U3 连接保持的特殊性**：
- SS 通道 U3 时电气 idle（无 LFPS）
- 连接靠 **USB2 PHY 的 D+ pull-up** 维持
- **pull-up 掉 → host 认为 disconnect → 需重新枚举**

---

## 6. Wakeup 机制

### 6.1 USB2 Wakeup

| 类型 | 机制 |
|---|---|
| Host 唤醒 device | K-state signaling (Resume) ≥ 20ms |
| Device remote wakeup | 发 K-state (1-15ms) |
| 检测器 | LineState 比较器（模拟）|
| 前置条件 | `SET_FEATURE(DEVICE_REMOTE_WAKEUP)` |

### 6.2 USB3 Wakeup

| 类型 | 机制 |
|---|---|
| Host 唤醒 device | LFPS U3_Wakeup burst |
| Device remote wakeup | 发 LFPS U3_Wakeup |
| 检测器 | **LFPS detector**（低频包络检测，10-50MHz）|
| 前置条件 | `SET_FEATURE(DEVICE_REMOTE_WAKEUP)` |
| 特有能力 | **Function Suspend**（function 级 PM）|

### 6.3 LFPS (Low Frequency Periodic Signaling)

- USB3 的**带外唤醒信号**
- 频率 10-50 MHz（相对 5Gbps 数据是低频）
- 独立于高速 SerDes，功耗极低（μA 级）
- 只做包络检测，不需要 CDR
- 功能等同 USB2 的 LineState detector

### 6.4 Wakeup 通知路径（通用）

```
Bus 上的 wakeup signaling (K-state / LFPS)
    ↓
PHY 检测器 (AON 域)
    ↓
Controller wakeup 逻辑 (AON 域)
    ↓
SoC PMU / 中断控制器
    ↓
恢复时钟 → 恢复电源 → 恢复 PHY → 恢复 Controller
    ↓
USB2: 恢复 SOF  /  USB3: LTSSM Recovery → U0
    ↓
继续 pending transfer（不重新枚举）
```

---

## 7. 电流与时序约束

### 7.1 电流上限

| 状态 | USB2 | USB3 |
|---|---|---|
| Unconfigured | 100 mA | 150 mA (SS) |
| Configured max | 500 mA | 900 mA (SS) |
| **Suspend (L2/U3)** | **2.5 mA** | **2.5 mA** |
| Low-power device suspend | 500 μA | — |
| Remote wakeup 额外 | +2.5 mA | +2.5 mA |

> **注意**：仅 bus-powered device 受此限制；self-powered 不受限但建议节能。

### 7.2 退出延迟约束

| 转换 | 约束 | Descriptor 字段 |
|---|---|---|
| USB2 L2 → L0 | Resume 后 20ms 内完成 | — |
| USB3 U1 → U0 | 推荐 <10 μs | `bU1DevExitLat` (0-10μs) |
| USB3 U2 → U0 | <2048 μs | `bU2DevExitLat` (0-2047μs) |
| USB3 U3 → U0 | 推荐 <10 ms | — |

> **报告太大 → host 不使能 U1/U2；报告太小 → 实际达不到会出错**

---

## 8. Controller 时钟电源管理

### 8.1 三种 Suspend 深度方案

| 方案       | 关闭内容                     | 恢复时间    | 功耗   | 场景                      |
| -------- | ------------------------ | ------- | ---- | ----------------------- |
| **A 轻量** | 时钟门控 + PLL               | μs 级    | ~1mA | autosuspend             |
| **B 中等** | PHY 大部分 + 时钟 + retention | 几十μs-ms | <mA  | 主流做法                    |
| **C 深度** | Controller 完全掉电          | ms 级    | μA 级 | system S3 / Hibernation |
|          |                          |         |      |                         |

### 8.2 Suspend 时各部分处理原则

| 部分 | 必须关 | 可以关 | 必须保留(AON) |
|---|---|---|---|
| PHY 高速电路 (SerDes/HS TX-RX) | ✓ | | |
| PLL (U2/U3/L2) | ✓ | | |
| 高速时钟 (PIPE 250MHz / 60MHz UTMI) | ✓ | | |
| RefClk / XTAL | | ✓ | |
| Controller 数据路径 | | ✓ | |
| AHB/AXI clock | | ✓ | |
| **Wakeup 检测器** (LineState/LFPS) | | | ✓ |
| **VBUS 检测器** | | | ✓ |
| **D+ pull-up** (保持 connect) | | | ✓ |
| **Wakeup 中断路径** | | | ✓ |
| Controller 寄存器 | | (retention) | 推荐保留 |

### 8.3 参考功耗数字（VBUS 5V）

**USB2 HS device**：
| 状态 | 电流 |
|---|---|
| L0 active | ~30 mA |
| L1 LPM | ~2 mA |
| L2 浅 suspend | ~1 mA |
| L2 深 suspend | ~200 μA |

**USB3 device**：
| 状态 | 电流 |
|---|---|
| U0 active (5Gbps) | ~150 mA |
| U1 | ~15 mA |
| U2 | ~5 mA |
| U3 浅 | ~1 mA |
| U3 深 | ~200 μA |
| U3 + Hibernation | ~50 μA |

---

## 9. 软件实现要点

### 9.1 DWC3 关键寄存器

```c
/* USB2 PHY 控制 */
GUSB2PHYCFG.SUSPHY       /* L2 时自动 suspend PHY */
GUSB2PHYCFG.ENBLSLPM     /* L1 LPM 时 PHY suspend */

/* USB3 PIPE 控制 */
GUSB3PIPECTL.SUSPEND_EN  /* 使能 SS PHY 自动 suspend */
GUSB3PIPECTL.DELAYP1TRANS

/* Device U1/U2 控制 */
DCTL.INITU1ENA/INITU2ENA   /* device 主动发起 LGO */
DCTL.ACCEPTU1ENA/ACCEPTU2ENA /* device 接受 host LGO */
DCTL.KEEP_CONNECT          /* hibernation 保持 connect */

/* Hibernation */
GCTL.GBLHIBERNATIONEN

/* 状态查询 */
DSTS.USBLNKST            /* 当前 link state */
```

### 9.2 Descriptor 正确报告 exit latency

```c
static struct usb_ss_cap_descriptor ss_cap = {
    .bU1devExitLat = 4,                    /* 4 μs */
    .wU2DevExitLat = cpu_to_le16(500),     /* 500 μs */
};
```

### 9.3 U1/U2 使能顺序

```
1. DCTL.ACCEPTU1ENA/ACCEPTU2ENA = 1   (接受 host request)
2. GUSB3PIPECTL.SUSPEND_EN = 1         (使能 PHY 自动 suspend)
3. 收到 SET_FEATURE(U1_ENABLE) 后置 INITU1ENA
```

### 9.4 Suspend/Resume 顺序

```
关闭顺序：Controller idle → PHY suspend → 关 PHY clock → 关 Controller clock
恢复顺序：反向（先开 clock 再访问寄存器，PHY 稳定后再 training）
```

### 9.5 Linux Wakeup 注册

```c
enable_irq_wake(irq);   /* 保证 wakeup 中断走 AON 路径 */
```

---

## 10. 常见问题排查

### 10.1 进不了低功耗状态

| 现象 | 可能原因 |
|---|---|
| Device 从不进 U1/U2 | Host 没发 SET_FEATURE(U1/U2_ENABLE) / exit latency 报告太大 / inactivity timer 太长 / EP 一直有传输 |
| Device 从不进 L1 | Host 不支持 LPM / device NAK LPM token |

### 10.2 Wakeup 失败

| 现象 | 可能原因 |
|---|---|
| U3/L2 后唤不醒 | LFPS/LineState detector 没供电 / wakeup 中断没路由到 AON / 没调 enable_irq_wake() / retention 电压不够寄存器丢失 |
| Remote wakeup 超时 | XTAL 起振太慢 / device wakeup latency 超窗口 |

### 10.3 Resume 后 hang

| 现象 | 可能原因 |
|---|---|
| Resume 后死机 | Controller 时钟没先开就访问寄存器 / PHY 没稳定就 training / LTSSM 从错误状态起始 / suspend/resume 顺序错 |

### 10.4 连接丢失（意外重新枚举）

| 现象 | 可能原因 |
|---|---|
| U3 后 host 报 disconnect | **USB2 D+ pull-up 掉了** / VBUS 检测异常 |

### 10.5 Compliance 测试失败

| 测试项 | 关注点 |
|---|---|
| EL_38 (suspend current) | 关得不够，超 2.5mA |
| EL_23 (U1/U2 timer) | timer 配置 / device 响应 LGO 太慢 |
