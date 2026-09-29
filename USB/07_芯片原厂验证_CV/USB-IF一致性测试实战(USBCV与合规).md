# USB-IF 一致性测试实战 (USBCV 与合规认证)

> **定位**：芯片原厂软硬件产品走向商用前，必须通过的 USB 官方协会（USB-IF）金牌认证。CV 工程师负责通过官方自动化测试集（USBCV）并对 Fail 项进行驱动层/固件层整改。

---

## 1. 核心测试体系与工具链
* **USBCV (USB Command Verifier)**：USB-IF 官方发布的自动化测试工具（运行于 Windows 专有 Host 控制器），覆盖协议层、描述符合法性、控制传输边界。
* **USB3CV**：针对 USB 3.x SuperSpeed 的专用验证套件。
* **电气一致性测试 (Electrical Compliance)**：使用高带宽示波器配合官方夹具，测试眼图、抖动（Jitter）、压摆率（Slew Rate）。

---

## 2. USBCV 核心测试章节与覆盖重点

### 2.1 描述符合法性测试 (Chapter 9 Tests)
* **测试内容**：
  * 设备描述符、配置描述符、接口描述符、端点描述符的字段长度与编码合法性；
  * `bMaxPacketSize0` 是否符合速度规范（全速必须为 8/16/32/64，高速固定为 64，超速固定为 512）；
  * 字符串描述符语言 ID (`0x0409`) 与 UNICODE 编码；
  * 各端点描述符声明的属性与实际控制器支持的端点类型是否绝对一致。

### 2.2 控制传输异常边界测试 (Control Transfer Tests)
* **Zero Length Packet (ZLP)**：当传输长度恰好是最大包长整数倍时，是否正确返回 0 长度包结束传输；
* **STALL 条件触发**：收到不支持的标准请求时（如非法的 `GET_DESCRIPTOR`），端点 0 必须立即响应 **Protocol STALL**；
* **清除挂起 (`CLEAR_FEATURE(ENDPOINT_HALT)`)**：被 STALL 的数据端点在收到清除命令后，必须正确复位 Data Toggle 为 0。

### 2.3 链路与电源管理测试 (Link & Power Tests)
* **挂起与远程唤醒测试**：
  * Host 发出挂起信号 3ms 后，Device 总电流必须降至规范限制以下（如 < 2.5mA）；
  * 若声明支持 Remote Wakeup，验证是否能通过驱动拉低 D- 持续产生 Resume 唤醒脉冲。
* **USB 3.x U1/U2 退出时延测试**：
  * 验证从 U1/U2 退回 U0 的延迟是否小于设备描述符中声明的 `bU1DevExitLat` 和 `wU2DevExitLat`。

---

## 3. 芯片原厂驱动常见 Fail 项与整改策略

| 常见 Fail 测试项 | 现象 / 报错信息 | 根本原因 | 原厂驱动整改策略 |
| :--- | :--- | :--- | :--- |
| **Ch9 Invalid Request Test** | 测试报 `Device did not STALL on unsupported request` | 驱动对未知 Setup 报文回复了 ACK 或超时忽略 | 在 `composite_setup()` 中针对未识别请求显式调用 `usb_ep_set_halt(gadget->ep0)` |
| **MaxPacketSize Mismatch** | `Descriptor verification failed: wMaxPacketSize invalid` | 驱动在高低速切换时未动态刷新端点描述符 | 确保在 `set_alt()` 或连接速度变更回调中重新初始化端点配置 |
| **Suspend Current Violation** | `Device consumed more than 2.5mA in Suspend` | 挂起时部分片上外设或模拟 PHY 未断电门控 | 在 Suspend 回调中主动配置 PHY 进入低功耗模式并关闭无关 PLL |
| **U1/U2 Exit Timeout** | `Link state machine failed to transition to U0 within limit` | 控制器收到唤醒 LFPS 到恢复时钟时间过长 | 调大驱动上报的时延容忍值，或优化硬件 PLL 快速锁定参数 |
