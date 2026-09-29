# SDIO / SD 驱动能力 (Driver Strength) 与阻抗匹配

在高速 SD/SDIO 接口（UHS-I 模式）设计与调试中，Host 与 Card 之间的信号完整性（SI）极大取决于输出驱动能力（Driver Strength）与阻抗匹配。

---

## 1. 驱动类型与输出阻抗定义 (Driver Strength Types)

根据 SD 规范（SD Physical Layer Specification / UHS-I），定义了以下四种驱动能力类型：

| 驱动类型 | 典型标称阻抗 | 阻抗范围 | 驱动倍率 (相对标准) | 适用速率模式 / 典型场景 |
| :--- | :--- | :--- | :--- | :--- |
| **Type B** (默认/基准) | **50 Ω** | 33 ~ 66 Ω | **1.0x** (标称基准) | 默认模式、SDR12、SDR25 |
| **Type A** (弱驱动) | **33 Ω** 或 0.75x | 22 ~ 44 Ω | **1.5x** | 信号过冲较大、走线较短时使用 |
| **Type C** (中驱动) | **66 Ω** 或 1.5x | 44 ~ 88 Ω | **0.75x** | SDR50、DDR50 (轻负载) |
| **Type D** (强驱动) | **25 Ω / 20 Ω** | 16 ~ 24 Ω | **2.0x** (最强驱动) | **SDR104** (高速、大容性负载、走线较长) |

> **注**：在 SD 规范术语中，Type B 为 1.0x 标准驱动（50Ω）。Type D 具有最低的内阻（约 20~25Ω），能输出最大的驱动电流并应对高达 208MHz (SDR104) 下的走线电容负载。

---

## 2. 硬件与信号调优场景

* **何时调大驱动能力 (如切至 Type D / 强驱动)**：
  * 高速模式（SDR104 @ 208MHz）下信号上升沿过缓、眼图闭合。
  * 示波器量测 CLK/CMD/DATA 边沿太缓，导致建立/保持时间违例（CRC 报错）。
  * 板级走线过长或串接了较大的寄生电容。
* **何时调小驱动能力 (如切至 Type A / Type C)**：
  * 走线短但信号过冲（Overshoot）或下冲（Undershoot）严重，引起回铃（Ringing）或 EMI 辐射超标。

---

## 3. 驱动能力选择与配置机制

### 3.1 协商流程 (CMD6 切换机制)
在 UHS-I 模式下，Host 通过 **CMD6 (SWITCH_FUNC)** 读取卡端能力并下发设置：
1. **Mode 0 (Check Function)**：Host 发送 CMD6 探测 Card 是否支持指定 Group 的驱动类型（Driver Strength 位于 Group 3）。
2. **Mode 1 (Set Function)**：Host 发送 CMD6 正式切换 Card 端的驱动类型。

### 3.2 Linux 内核驱动中的映射
在 Linux 内核 MMC 子系统 (`drivers/mmc/core/sd.c`) 中定义：
```c
#define SD_DRIVER_TYPE_B    0x01
#define SD_DRIVER_TYPE_A    0x02
#define SD_DRIVER_TYPE_C    0x04
#define SD_DRIVER_TYPE_D    0x08
```
设备树 (DTS) 节点中通常通过 `drive-strength` 属性配置 SoC 控制器引脚的驱动强度（如 4mA / 8mA / 12mA）。

---

## 4. 参考文献
* *SD Specifications Part 1 Physical Layer Specification (Version 3.01+)*
* Linux 内核 `drivers/mmc/` 核心与主机控制器驱动源码
