# Type-C CC 逻辑与 PD 协商基础

> **定位**：芯片原厂在 Bring-up Type-C 接口板卡或调试外挂 TCPC/TCPM 芯片时的物理层/协议层基础。

---

## 1. Type-C 核心引脚分布与作用
* **CC1 / CC2 (Configuration Channel)**：用于角色检测、正反插判断、供电能力广播与 USB PD 报文通信。
* **VBUS / GND**：电源与地。默认供电为 5V/Vsafe0V。
* **D+ / D-**：USB 2.0 差分信号（正反面各一组并联）。
* **TX1/RX1, TX2/RX2**：USB 3.x SuperSpeed 高速差分对（根据正反插选择使用其中一组或两组并联）。
* **SBU1 / SBU2 (Sideband Use)**：辅助边带信号（通常用于 DP Alt-Mode 等备用模式）。

---

## 2. 角色定义与 CC 阻抗检测机制

| 角色 | 含义 | CC 引脚特征 | 供电行为 |
| :--- | :--- | :--- | :--- |
| **DFP (Downstream Facing Port)** | 主机端口 (Host) | 通过 **Rp (Pull-up)** 上拉到 3.3V/5V | 供电方 (Source) |
| **UFP (Upstream Facing Port)** | 设备端口 (Device) | 通过 **Rd (5.1kΩ Pull-down)** 下拉到 GND | 受电方 (Sink) |
| **DRP (Dual Role Port)** | 双角色端口 (OTG/可主可从) | 内部开关周期性切换 **Rp / Rd** | 依协商而定 |

* **正反插检测**：线缆内部只引出一根 CC 线（另一根对应为 VCONN 供电或悬空）。当 DFP 检测到某一根 CC（如 CC1）被 Rd 拉低，而另一根悬空时，即判定为正插；反之则为反插，芯片内部交叉开关（Mux）自动倒换高速通道。

---

## 3. 供电能力广播 (Broadcast Current)
DFP 通过配置 Rp 的阻抗阻值（或电流源大小），向 UFP 广播当前端口可提供的电流量：
* **默认 USB 供电 (Default)**：5V @ 500mA (USB 2.0) 或 900mA (USB 3.0)。
* **1.5A 广播**：5V @ 1.5A。
* **3.0A 广播**：5V @ 3.0A。

---

## 4. USB PD (Power Delivery) 协议简介
* **通信载波**：在选定的单根 CC 线上采用 **BMC (Biphase Mark Coding)** 编码进行半双工通信。
* **协商报文**：
  1. Source 发送 `Source_Capabilities`（广播支持的电压/电流档位，如 5V/9V/12V/15V/20V）；
  2. Sink 响应 `Request` 申请期望档位；
  3. Source 回复 `Accept` 并触发电源芯片升降压；
  4. 最终发送 `PS_RDY` 确认电压平稳就绪。

---

## 5. 芯片原厂 CV 重点关注项
- [ ] DRP 状态机切换占空比与时序（Toggle 周期通常要求符合规范 50ms~100ms）；
- [ ] Type-C 模拟开关 Mux 在正反插切换后的差分信号损耗与眼图退化；
- [ ] 掉电态（Dead Battery）时，UFP 端硬件是否有硬连线的 Rd 保证能被 Host 识别供电。
