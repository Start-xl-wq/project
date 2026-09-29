# USB Gadget RNDIS 快速验证指南

本文档用于嵌入式 Linux 板卡枚举为 **RNDIS (虚拟以太网卡)** 后的网络双向连通性与吞吐验证。

---

## 1. 节点与网卡确认

### 1.1 Device 端确认
```bash
# 检查是否生成虚拟网卡节点（通常为 usb0）
ifconfig -a | grep usb
ip link show usb0
```

### 1.2 Host 端确认
* **Linux PC**：
  ```bash
  dmesg | grep rndis_host
  ip link show # 预期出现 usb0 或 enx...
  ```
* **Windows PC**：
  打开“网络连接”或“设备管理器”，查看是否生成“远程 NDIS 兼容设备”或未知设备（若出现黄色感叹号需手动指定 Windows 自带 Microsoft RNDIS 驱动）。

---

## 2. IP 地址配置

配置 Device 与 Host 处于同一网段（以 `192.168.10.x` 为例）：

### 2.1 Device 端配置
```bash
ifconfig usb0 192.168.10.2 netmask 255.255.255.0 up
```

### 2.2 Host 端配置
* **Linux PC**：
  ```bash
  sudo ifconfig usb0 192.168.10.1 netmask 255.255.255.0 up
  ```
* **Windows PC**：
  在网卡属性的“Internet 协议版本 4 (TCP/IPv4)”中手动配置：
  * IP：`192.168.10.1`
  * 子网掩码：`255.255.255.0`

---

## 3. 双向通信与吞吐验证

### 3.1 基础双向 Ping 测试
* Device -> Host：
  ```bash
  ping -c 4 192.168.10.1
  ```
* Host -> Device：
  ```bash
  ping 192.168.10.2
  ```

### 3.2 双向网络带宽压测 (iperf3)

#### 方向 1：Device -> Host (上行)
1. **Host 端**启动服务端：
   ```bash
   iperf3 -s
   ```
2. **Device 端**启动客户端发包：
   ```bash
   iperf3 -c 192.168.10.1 -t 10 -i 1
   ```

#### 方向 2：Host -> Device (下行)
1. **Device 端**启动服务端：
   ```bash
   iperf3 -s &
   ```
2. **Host 端**发起下行打流：
   ```bash
   iperf3 -c 192.168.10.2 -t 10 -i 1
   ```

---

## 4. 常见排查点

1. **Windows 识别为未知设备（感叹号）**：
   * 在设备管理器中右键更新驱动 -> 浏览计算机以查找驱动程序 -> 从计算机的可用驱动程序列表中选择 -> 网络适配器 -> Microsoft -> Remote NDIS Compatible Device。
2. **Ping 不通**：
   * 检查防火墙是否拦截 ICMP（尤其是 Windows 防火墙）。
   * 确认双方 IP 是否在同一子网，且 `ifconfig` 状态为 `UP`。
