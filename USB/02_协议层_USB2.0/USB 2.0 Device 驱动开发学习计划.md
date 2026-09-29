## 第一阶段（第 1~2 周）

**目标：精通 PHY 握手 + 描述符体系 + 枚举流程 (Enumeration)**

---

### 第 1 周：彻底搞懂链路建立与电气协商

**你要搞清楚：** • 插入检测原理（D+ / D- 的上拉/下拉电阻机制） • NRZI 编码基础与位填充 (Bit Stuffing) • 基础总线状态：J 状态、K 状态、SE0 (Single-Ended Zero) • **Chirp Sequence 握手流程**（全速切高速的命脉） • Bus Reset、Suspend、Resume 的电气触发机制

**你必须能回答：** 一根 USB 线插上 PC，VBUS 电源上来后，底层 D+/D- 引脚电平发生了什么具体的时序变化，最终让总线稳定在 480Mbps 的 High-Speed 状态？

---

### 第 2 周：精通 Descriptors 与枚举逻辑（重点）

**你要搞清楚：** • Device Descriptor (设备描述符，含 VID/PID/MaxPacketSize0) • Configuration Descriptor (配置描述符) • Interface Descriptor (接口描述符，决定设备类) • Endpoint Descriptor (端点描述符，决定传输属性) • 控制传输 (Control Transfer) 的三阶段：Setup -> Data -> Status • 标准设备请求 (`GET_DESCRIPTOR`, `SET_ADDRESS`, `SET_CONFIGURATION`)

**你要做到：** 不看资料，能画出 USB 描述符的层级拓扑树。

**你必须能回答：** Windows 提示“无法识别的 USB 设备”时，在枚举的“黄金10步”里，通常是哪一步挂了？Host 发出 `SET_ADDRESS` 时你的硬件发生了什么？

## 第二阶段（第 3~5 周）

**目标：精通 Packet 协议 + 硬件 UDC 控制器 + 数据通道 (Data Path)**

---

### 第 3 周：精通总线事务与握手包解析

**你要搞清楚：** • 包 (Packet) 的基础结构：SYNC + PID + Payload + CRC + EOP • Token / Data / Handshake 包的区别与作用 • IN Transaction (输入事务) 完整流程 • OUT Transaction (输出事务) 完整流程 • **Data Toggle (DATA0/DATA1 数据翻转防丢包机制)** • NAK（未准备好）与 STALL（端点挂起/错误）的总线响应机制

**你必须能回答：** 当 Host 发来一个 IN Token，但你的 Device 内部 FIFO 还没准备好数据时，总线上发出了什么包？软件层需要做什么吗？

---

### 第 4~5 周：精通 UDC 硬件控制器与 DMA 机制

**你要搞清楚：** • USB Device Controller (UDC) 内部 FIFO 结构 (TX FIFO / RX FIFO) • UDC 中断系统 (全局 Bus 中断 vs 具体的 Endpoint 中断) • **DMA Descriptor (传输描述符环 / TRB / dTD) 机制** • 寄存器级排错：Endpoint 状态寄存器的读取与清零 • Ping-Pong Buffer / Double Buffering 硬件机制

**你必须能回答：** Host 想要读取一笔 4KB 的 Bulk 数据。从你的驱动把物理地址写进 UDC 控制器的 DMA 描述符开始，**到最后一次硬件向 Host 发出 ACK**，控制器内部发生了什么？

## 第三阶段（第 6~8 周）

**目标：精通 Linux UDC 驱动架构 + 带宽优化与性能调优**

---

### 第 6~7 周：精通 Linux Gadget 架构与数据流转

**你要搞清楚：** • 核心数据结构：`struct usb_gadget` / `struct usb_ep` • 数据载体：`struct usb_request` 的生命周期 • API 底层映射：`usb_ep_queue()` 如何把数据挂入 UDC 硬件 Ring • SG (Scatter-Gather) List 在 USB UDC 驱动中的展开与映射 • 中断下半部 (`usb_request->complete()` 回调) 机制的触发链路

**你必须能回答：** 上层应用连续下发了 5 个 `usb_request`，你的底层 UDC 驱动是怎么把它们排队挂在硬件 DMA Ring 上，并在完成时依次精准触发回调的？

---

### 第 8 周：成为数据通道带宽与时序专家

**你要搞清楚：** • SOF (Start of Frame) 与 High-Speed 下的 Microframe (125us) • Max Packet Size 限制 (例如：高速 Bulk 强制为 512B) • 协议微观开销分析 (Sync + PID + Header + CRC + EOP + 帧间间隙) • 硬件突发传输 (Burst) 与 AHB/AXI 总线带宽的匹配关系

**你必须会算：** • USB 2.0 High-Speed 标称 480Mbps（物理层 60MB/s）。 • 扣除 SOF、Token 令牌包、Handshake 握手包和空闲等待时间的协议开销。 • **推算你的 Device 在单 Endpoint 的 Bulk IN 传输下，极限有效 Payload 数据带宽到底能跑到多少 MB/s？**