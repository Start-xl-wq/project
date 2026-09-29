# PIPE 与 UTMI+ 控制器与 PHY 接口规范

> **定位**：芯片原厂软硬件联合调试的关键边界。当物理链路不通、状态机卡死时，判断是“数字控制器问题”还是“模拟 PHY 问题”的核心分水岭。

---

## 1. 为什么需要接口规范？
* **数字控制器 (Controller)**：如 Synopsys DWC3、Cadence IP，负责高层协议、DMA、TRB 管理、事务调度。
* **模拟收发器 (PHY)**：负责高速模拟电平驱动、端接电阻、CDR 时钟数据恢复、串并转换。
* **桥梁**：
  * **UTMI / UTMI+**：USB 2.0 数字控制器与 USB 2.0 PHY 之间的并行接口规范。
  * **PIPE (PHY Interface for the PCI Express and USB SuperSpeed)**：USB 3.x / PCIe 数字控制器与 SerDes PHY 之间的行业标准接口。

---

## 2. UTMI+ (USB 2.0) 核心信号线与交互
* **时钟信号**：`CLK` (通常为 60MHz @ 8-bit 或 30MHz @ 16-bit)。
* **数据总线**：`DataIn[7:0]` (Rx), `DataOut[7:0]` (Tx)。
* **收发控制**：
  * `TxValid` / `TxReady`：控制器向 PHY 请求发包并流控。
  * `RxValid` / `RxActive`：PHY 检测到总线有包并向控制器推送。
* **低功耗与状态控制**：
  * `SuspendM`：控制 PHY 进入休眠模式；
  * `OpMode[1:0]`：正常操作、不产生 Sync/EOP 的非驱动模式、禁用位填充等测试模式。
  * `XceiverSelect` / `TermSelect`：全速与高速收发器切换、终端电阻切换（Chirp 握手关键信号）。

---

## 3. PIPE (USB 3.x SuperSpeed) 核心信号线与交互
* **时钟与位宽**：如 125MHz/250MHz 时钟，配合 16-bit 或 32-bit 并行数据。
* **收发通路**：`TxData`, `TxDataK` (指示是否为控制 K 码), `RxData`, `RxDataK`。
* **状态机与电源管理**：
  * `PowerDown[1:0]`：控制器要求 PHY 进入 P0 (正常), P1 (浅休眠), P2, P3 (深休眠)。
  * `PhyStatus`：PHY 反馈状态切换完成的指示脉冲。
  * `TxDetectRx_Loopback`：指示 PHY 发起模拟接收端探测（Rx.Detect 阶段）。
  * `RxStatus[2:0]`：接收端状态反馈（如检测到 8b/10b 解码错误、溢出、Disparity 错误等）。

---

## 4. 芯片 Bring-up 与 CV 排查核心思路
1. **查时钟与复位**：数字控制器是否把复位释放给了 PHY？PHY 输出给控制器的参考时钟（如 UTMI 60MHz / PIPE PCLK）是否起振？
2. **查状态机响应**：控制器下发 `PowerDown=P0` 后，PHY 的 `PhyStatus` 是否有高电平脉冲回响？如果无回响，证明 PHY 内部 PLL 或状态机挂死。
3. **查测试环回 (Loopback)**：
   * **近端数字环回 (Internal Loopback)**：数据不经物理引脚，在 PIPE 接口直接折返，用于排除硬件电路问题、快速验证数字控制器与驱动。
