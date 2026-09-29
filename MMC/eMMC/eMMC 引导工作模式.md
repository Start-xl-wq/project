# eMMC 引导工作模式 (Boot Operation) 详解

---

## 1. 设备复位至 Pre-idle 状态
### 1.1 触发机制
1. **上电复位 (Power-on Reset)**：主机上电后，设备处于 Inactive 状态，进入 Pre-idle 状态。
2. **软件复位命令**：`GO_PRE_IDLE` 命令（`CMD0 + 0xF0F0F0F0`），设备被置入 Pre-idle 状态。
3. **硬件复位引脚**：拉低 `RST_n` 信号（低电平有效）。

### 1.2 电源与时序
* `VCC`：内部控制器和 Flash 存储单元电源。
* `VCCQ`：I/O 引脚接口电源。
* `RST_n`：硬件复位信号（低电平复位，低有效）。
* **上电时序要求**：`VCC` 必须早于或等于 `VCCQ` 上电；在电源稳定、`RST_n` 释放并保持高电平稳定至少 1ms 后，Host 才能开始发送时钟与命令。

---

## 2. 引导操作 (Original Boot Operation)
* **进入方式**：在复位释放且发送第一个命令之前，Host 将 `CMD` 线拉低保持至少 74 个时钟周期（Clock Cycles）。
* **数据准备**：从机（eMMC）检测到 CMD 持续拉低后识别为引导模式，开始内部准备引导数据（Boot Data）。
* **Boot ACK**：若使能了引导确认（Boot ACK），eMMC 必须在 50ms 内在 `DAT0` 线上向 Host 返回 ACK（二进制序列 `010`）。
* **数据流传输**：在 CMD 保持拉低 1s 内，从机通过 `DAT` 总线流式发送 Boot Partition 内的引导数据；Host 必须持续保持 CMD 为低直至读取完毕。
* **DDR 模式时序**：若配置为 DDR 引导模式，起始位（Start bit）、停止位（Stop bit）和 ACK 位均在时钟上升沿有效。
* **退出机制**：Host 将 `CMD` 线拉高即可终止引导模式；从机在 $N_{ST}$ 周期内停止传输并退出。

---

## 3. 备用引导操作 (Alternative Boot Operation)
* **进入方式**：Host 在 74 个 clock cycle 之后、且发送 CMD1 之前，发送带专用参数的 CMD0：
  ```text
  CMD0 + 0xFFFFFFFA
  ```
* **特点**：无需长拉低 CMD 线。从机接收到该命令后即识别为 Alternative Boot，并在 1s 内开始向 Host 发送引导数据。

---

## 4. 引导状态转移
![引导模式状态图](https://i-blog.csdnimg.cn/direct/5ce60f19f30641ccbc425548df9c1f02.png)

---

## 5. 引导总线与分区配置
* **ECSD 寄存器配置**：通过扩展 CSD 寄存器（`EXT_CSD`）中的配置位设置：
  * `BOOT_BUS_CONDITIONS`：配置引导阶段的总线位宽（x1 / x4 / x8）以及单边沿 (SDR) 或双边沿 (DDR)。
  * `BOOT_CONFIG`：选择从 Boot Partition 1 或 Boot Partition 2 引导，以及是否使能 Boot ACK。
* **引导分区写保护 (Write Protection)**：
  * **Power-on Write Protection**：单次上电有效，掉电重启后保护失效。
  * **Permanent Write Protection**：一次性永久写保护，防篡改。

