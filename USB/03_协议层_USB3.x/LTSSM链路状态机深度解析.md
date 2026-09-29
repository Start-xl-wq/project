# USB 3.x LTSSM 链路训练状态机深度解析

> **定位**：芯片原厂 Bring-up USB 3.0/3.1/3.2 物理链路、定位“协商降级为 USB 2.0”或“卡在 Polling”问题必须掌握的状态机核心。

---

## 1. LTSSM 总体架构全景
USB 3.x SuperSpeed 丢弃了 2.0 的广播轮询与 Chirp 机制，全面借鉴了 PCIe 的 **LTSSM (Link Training and Status State Machine)**。主要大状态包括：

```text
       SS.Disabled (关闭)
            │
            ▼
        Rx.Detect ◄────────┐
            │              │
            ▼              │
         Polling ──────────┤
            │              │
            ▼              │ (链路恶化/重训)
           U0 (正常工作) ───┤
         ▲ │   │           │
         │ ▼   ▼           │
       U1/U2  U3(挂起)     │
         │     │           │
         └─────┴─► Recovery ┘
```

---

## 2. 关键核心状态拆解

### 2.1 Rx.Detect (接收端探测)
* **本质**：模拟电路 RC 充放电检测。发送端（Tx）向物理线路上施加一个电压阶跃，通过检测走线上的充电时间常数，判断远端是否连接了 50Ω 的端接电阻。
* **原厂 CV 排错**：如果芯片始终卡在 Rx.Detect，100% 是硬件外围或 PHY 模拟端接故障（如 AC 耦合电容漏焊、走线开路、PHY 偏置电流未配对）。

### 2.2 Polling (轮询同步)
* **目标**：对齐比特时钟、实现符号定界、协商通道参数。
* **流程**：
  1. **Polling.LFPS**：互发低频周期信号（LFPS, Low Frequency Periodic Signaling），确认双方均已退出模拟空闲；
  2. **Polling.RxEQ**：接收端均衡器自适应调优；
  3. **Polling.Active**：互发连续的 **TS1 (Training Sequence 1)** 训练码字（包含大量跳变沿供 CDR 提取时钟）；
  4. **Polling.Configuration**：互发 **TS2** 确认配置，锁定成功后跳转到 U0。
* **原厂 CV 排错**：卡在 Polling 往往表明**信号完整性 (SI) 不达标**、眼图闭合、抖动过大或 CDR 无法 Lock，最终超时降退到 USB 2.0。

### 2.3 U0 (Active 正常传输态)
* 链路处于全速工作状态，允许收发 Header Packet、Data Packet、链路控制原语（Link Commands）。

### 2.4 Recovery (链路恢复)
* 当链路在 U0 状态下检测到连续错误（如重传超时、坏包率过高、或者从 U1/U2 低功耗状态唤醒）时进入 Recovery。
* 重新发送快速 TS1/TS2 序列恢复同步；若恢复失败达到重试上限，则链路彻底断开并复位到 Rx.Detect。

---

## 3. 常见寄存器观测与排查思路 (以 DWC3 为例)
* 读取 DWC3 链路状态寄存器 `DSTS.LinkState`：
  * `0x0`: U0 (正常工作)
  * `0x1`: U1
  * `0x2`: U2
  * `0x3`: U3
  * `0x4`: SS.Disabled
  * `0x5`: Rx.Detect
  * `0x7`: Polling
  * `0x8`: Recovery
* **现象**：插上后秒掉回 USB 2.0（或 `dmesg` 打印 `cannot reset link / fallback to high-speed`）：
  * 先确认硬件走线 AC 电容是否为标准 100nF；
  * 用寄存器固定链路在 Polling，用示波器抓取 LFPS 信号幅度与周期是否符合规范（10MHz ~ 50MHz）。
