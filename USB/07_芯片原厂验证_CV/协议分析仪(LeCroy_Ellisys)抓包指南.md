# USB 协议分析仪 (LeCroy / Ellisys) 原厂抓包与定位指南

> **定位**：芯片原厂解决“玄学死机”、“偶发掉盘”、“枚举超时”的终极武器。当软件日志输出已不可靠时，总线物理 Packet 抓包是唯一不可辩驳的铁证。

---

## 1. 抓包仪工作拓扑与连接规范

```text
[Host 主机] 
    │
[硬件分析仪 (如 LeCroy Voyager / Ellisys USB Explorer)] ── (以太网/USB3) ──► [分析仪上位机分析软件]
    │
[DUT: 芯片原厂被测板卡 (Device)]
```

* **原则 1：阻抗匹配与短线原则**：分析仪与 DUT 之间的连线必须尽量短（建议 < 30cm），避免串联测试夹具引起额外的信号眼图衰减。
* **原则 2：Trigger (触发条件) 精准配置**：
  * 高速数据传输时流量极大（几分钟内可吞吐数 GB 日志，导致抓包软件内存溢出）；
  * **切忌无条件全量录制**，必须使用硬件触发（Hardware Trigger）。

---

## 2. 常用硬件触发条件设定技巧

### 2.1 捕获枚举挂死 (Enumeration Failure)
* **Trigger 条件**：设置捕获 `Standard Device Request` 中的 `SET_ADDRESS` 或 `GET_DESCRIPTOR (Configuration)`。
* **后触发观察**：观察从机在收到 SETUP 令牌包后，是在 DATA 阶段返回了 `STALL`，还是持续回复 `NAK` 导致 Host 超时放弃。

### 2.2 捕获偶发总线错误 (CRC / Babble / Bit Stuffing)
* **Trigger 条件**：在分析仪面板勾选 `Error Trigger`：
  * `CRC5 / CRC16 Error`
  * `Bad Packet Framing`
  * `Babble Error` (设备发送数据超出了帧时间)
* **分析方向**：一旦触发，立刻前向查看前几个包的数据幅度和边沿，判断是板级瞬态电源噪声干扰还是 PHY 发包逻辑错位。

### 2.3 捕获长稳压测中的传输卡死 (DMA Hang)
* **Trigger 条件**：设置针对特定端点（如 Bulk IN Endpoint 1）的 `Continuous NAK Timeout`（持续 NAK 超过 100ms）。
* **分析方向**：说明外部 Host 疯狂发起 IN 令牌索取数据，但芯片内部的 DMA/TRB 并未将数据喂进 FIFO，直接锁定原厂驱动或 DMA 中断丢失。

---

## 3. 经典总线抓包波形分析法

| 报文特征 | 物理层总线现象 | 问题定性与责任划分 |
| :--- | :--- | :--- |
| **Token IN -> NAK -> Token IN -> NAK** | 持续返回 NAK | **正常背压**或**驱动断流**：从机内部数据尚未准备好，属于合法流控，但若长时间持续则为驱动死锁。 |
| **Token OUT -> STALL** | 明确返回 STALL | **协议错误**：从机主动宣告发生严重错误，驱动代码中主动调用了 `set_halt`。 |
| **Token OUT -> 无响应 (Timeout)** | 总线静默，Host 等不到任何 ACK/NAK/STALL | **硬件致命挂死**：从机 PHY 或控制器已死锁或未上电，未将包成功打上总线。 |
| **Chirp K -> Chirp K-J 翻转异常** | Reset 期间 K-J 对数不足 3 对 | **PHY 模拟问题**：高速握手失败，设备自动回退为 12Mbps 全速模式。 |
