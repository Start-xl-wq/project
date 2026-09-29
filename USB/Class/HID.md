# USB Gadget HID (键盘/鼠标) 快速验证指南

本文档用于嵌入式 Linux 板卡枚举为 **USB HID (人机接口设备，如虚拟键盘/鼠标)** 后的快速功能验证与输入注入。

---

## 1. 节点与设备确认

### 1.1 Device 端确认
检查内核 Gadget HID 驱动是否生成对应的字符设备节点：
```bash
ls -l /dev/hidg*
```
> 正常情况下会看到 `/dev/hidg0`（若配置了多个 HID 实例，则会有 `hidg1` 等）。

### 1.2 Host 端确认
* **Linux PC**：
  ```bash
  dmesg | grep -i hid
  ls /dev/input/by-id/ # 查看新增的虚拟输入设备
  ```
* **Windows PC**：
  打开“设备管理器” -> “人体学输入设备”或“键盘/鼠标”，查看是否识别出标准 HID 兼容设备。

---

## 2. 功能快速验证 (Device -> Host 数据注入)

### 2.1 键盘模拟验证 (Key Injection)
标准 HID 键盘报文长度为 8 字节：
* Byte 0: 修饰键 (Modifier keys: Ctrl/Shift/Alt/GUI)
* Byte 1: 保留位 (0x00)
* Byte 2~7: 按键键码 (Keycode 数组，支持同时按下 6 个键)

```bash
# 示例 1：在 Host 焦点窗口输入小写字母 'a' (Keycode 0x04) 并释放
# 1. 按下 'a'
echo -ne "\x00\x00\x04\x00\x00\x00\x00\x00" > /dev/hidg0
# 2. 释放按键 (全 0)
echo -ne "\x00\x00\x00\x00\x00\x00\x00\x00" > /dev/hidg0

# 示例 2：输入大写字母 'A' (Left Shift = 0x02 + 'a' = 0x04) 并释放
echo -ne "\x02\x00\x04\x00\x00\x00\x00\x00" > /dev/hidg0
echo -ne "\x00\x00\x00\x00\x00\x00\x00\x00" > /dev/hidg0

# 示例 3：按下回车键 Enter (Keycode 0x28) 并释放
echo -ne "\x00\x00\x28\x00\x00\x00\x00\x00" > /dev/hidg0
echo -ne "\x00\x00\x00\x00\x00\x00\x00\x00" > /dev/hidg0
```

> **验证方式**：在 Host 端打开记事本或文本框，Device 端执行上述命令，观察 Host 端是否自动打出对应字符。

---

### 2.2 鼠标模拟验证 (Mouse Movement)
标准相对坐标 HID 鼠标报文通常为 4 字节（或 3 字节）：
* Byte 0: 按键状态 (Bit 0: 左键, Bit 1: 右键, Bit 2: 中键)
* Byte 1: X 轴相对位移 (有符号 int8, -127 ~ 127)
* Byte 2: Y 轴相对位移 (有符号 int8, -127 ~ 127)
* Byte 3: 滚轮位移 (Wheel)

```bash
# 示例：鼠标向右移动 20 像素 (X=+20 -> 0x14)
echo -ne "\x00\x14\x00\x00" > /dev/hidg0

# 示例：鼠标左键单击 (按下 -> 释放)
echo -ne "\x01\x00\x00\x00" > /dev/hidg0
echo -ne "\x00\x00\x00\x00" > /dev/hidg0
```

---

## 3. 常见排查点

1. **写 `/dev/hidg0` 提示 `Device or resource busy` 或无反应**：
   * **排查**：确保每次按下后必须发送一个**全 0 报文**释放按键，否则 Host 会一直认为按键被持续按住。
2. **写节点卡死 (Hang)**：
   * **排查**：HID 基于中断端点 (Interrupt Endpoint)，若 Host 侧并未轮询该端点或未枚举完成，写操作会阻塞在内核队列中。
