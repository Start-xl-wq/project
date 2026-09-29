# USB Gadget ACM (CDC-ACM) 双向传输验证指南

本文档面向嵌入式 Linux 开发板作为 **USB Device** 并配置为 **CDC-ACM 虚拟串口** 后的实战使用与双向传输验证。

---

## 1. 前置准备与节点确认

在进行双向传输前，确保 USB Gadget 已成功使能（通过 ConfigFS 或平台 Gadget 脚本绑定 UDC 并连接到 Host 主机）。

### 1.1 Device 端节点确认
* 检查 Device 端生成的串口字符设备节点：
  ```bash
  ls -l /dev/ttyGS*
  ```
  正常情况下会出现 `/dev/ttyGS0`（若配置了多个 ACM 实例，则会有 `ttyGS1` 等）。

### 1.2 Host 端节点确认
* **Linux PC 作为 Host**：
  查看内核识别到的 ACM 设备：
  ```bash
  dmesg | grep cdc_acm
  ls -l /dev/ttyACM*
  ```
  通常生成 `/dev/ttyACM0`。
* **Windows PC 作为 Host**：
  打开“设备管理器” -> “端口 (COM 和 LPT)”，确认生成对应的 `USB 串行设备 (COMx)`。

---

## 2. 串口环境初始化配置

为了避免回显干扰、换行符自动转换（如 `\n` 转 `\r\n`）或控制字符截断影响原始数据，建议测试前将 Device 与 Host 串口均配置为 **Raw 模式**：

```bash
# 关闭回显并开启原始数据模式
stty -F /dev/ttyGS0 raw -echo cs8 115200
```

> **注意**：CDC-ACM 底层使用 USB Bulk 端点传输，波特率设置对于实际 USB 物理传输速率没有限制，但在串口应用层做参数协商时仍建议保持一致。

---

## 3. 双向数据传输验证

### 场景 A：单向验证（基础联通性）

#### 方向 1：Device -> Host (上行)
1. **Host 端**先启动监听：
   * **Windows**：打开串口调试助手（MobaXterm、PuTTY、SSCOM 等），选择对应的 `COMx`，打开串口。
   * **Linux PC**：
     ```bash
     cat /dev/ttyACM0
     ```
2. **Device 端**发送测试数据：
   ```bash
   echo "Hello from USB Device!" > /dev/ttyGS0
   ```
3. **验证结果**：Host 端调试助手或控制台应能实时看到打印内容。

---

#### 方向 2：Host -> Device (下行)
1. **Device 端**先启动监听：
   ```bash
   cat /dev/ttyGS0
   ```
2. **Host 端**发送数据：
   * **Windows**：在串口助手中勾选“加回车换行”，输入内容并点击“发送”。
   * **Linux PC**：
     ```bash
     echo "Hello from USB Host!" > /dev/ttyACM0
     ```
3. **验证结果**：Device 端的 `cat` 终端中实时输出接收到的内容。

---

### 场景 B：全双工连续回环压测（Loopback 自动化验证）

如果需要进行稳定性、抗丢包或吞吐压测，可以在 Device 端跑一个 **Echo 回环**，由 Host 端进行自动发收校验。

#### 1. Device 端开启回环转发
```bash
# 将收到的所有数据原路发回 Host
cat /dev/ttyGS0 > /dev/ttyGS0 &
```

#### 2. Host 端 (以 Linux PC 为例) 校验
```bash
# 1. 终端 A 持续监听接收并写入文件
cat /dev/ttyACM0 > rx.bin &

# 2. 终端 B 产生 10MB 随机数据并发送
dd if=/dev/urandom of=tx.bin bs=1M count=10
cat tx.bin > /dev/ttyACM0

# 3. 停止接收并对比 MD5 校验完整性
killall cat
md5sum tx.bin rx.bin
```

---

## 4. 常见问题与排查

1. **写 `/dev/ttyGS0` 进程卡死 (Hang 住)**：
   * **原因**：Host 端尚未打开串口（DTR/RTS 信号未就绪），或 PC 侧串口驱动未就绪，Gadget 驱动默认会阻塞在写等待。
   * **排查**：确保 PC 端串口工具已点击“打开连接”。
2. **接收端收到乱码或丢字符**：
   * **排查**：是否未配置 `stty ... raw -echo`，导致特殊 ASCII 控制字符被终端驱动吞掉或转换。
3. **节点 `/dev/ttyGS0` 不存在**：
   * **排查**：检查内核是否编译使能了 `CONFIG_USB_F_ACM` 或 `CONFIG_USB_G_SERIAL`，以及 ConfigFS 中 functions/acm.xxx 是否已链接到 configs 目录并绑定 UDC。
