# USB Gadget UAS / BOT (U盘/存储) 快速验证指南

本文档用于嵌入式 Linux 板卡枚举为 **UAS (USB Attached SCSI) / Mass Storage (BOT)** 存储设备后的挂载、读写与速度验证。

---

## 1. 节点确认

### 1.1 Device 端确认
```bash
# 确认用于共享的后端介质文件或块设备是否存在
ls -lh /dev/mmcblk0p* /tmp/disk.img 2>/dev/null
```

### 1.2 Host 端确认
* **Linux PC**：
  ```bash
  dmesg | grep -E "uas|usb-storage|sd "
  lsblk # 确认新增的磁盘节点（如 /dev/sdb）
  ```
  * 若驱动打印 `uas` 表明工作在 **UAS 模式**；
  * 若打印 `usb-storage` 表明降级在传统 **BOT (Bulk-Only Transport) 模式**。
* **Windows PC**：
  “此电脑”中自动弹出对应可移动磁盘盘符。

---

## 2. 读写与数据一致性验证

### 2.1 基础文件写入与读取 (以 Linux Host 为例)

```bash
# 1. 挂载到 Host 本地目录
sudo mkdir -p /mnt/usb_dev
sudo mount /dev/sdb1 /mnt/usb_dev

# 2. Host 写入数据
echo "UAS test data" | sudo tee /mnt/usb_dev/test.txt
sync

# 3. Host 读取数据
cat /mnt/usb_dev/test.txt

# 4. 卸载
sudo umount /mnt/usb_dev
```

---

## 3. 双向大文件吞吐性能压测

以裸块设备方式读写（**注意：避免在挂载状态下操作，以防文件系统损坏**）：

### 3.1 Host -> Device (写压测)
```bash
# 在 Host 端向设备写入 1GB 顺序数据
sudo dd if=/dev/zero of=/dev/sdb bs=1M count=1024 oflag=direct status=progress
```

### 3.2 Device -> Host (读压测)
```bash
# 从设备读取 1GB 数据到 Host /dev/null
sudo dd if=/dev/sdb of=/dev/null bs=1M count=1024 iflag=direct status=progress
```

### 3.3 数据一致性校验 (MD5)
```bash
# 生成 500MB 随机文件写入
dd if=/dev/urandom of=tx_data.bin bs=1M count=500
sudo dd if=tx_data.bin of=/dev/sdb bs=1M count=500 oflag=direct

# 读出并校验
sudo dd if=/dev/sdb of=rx_data.bin bs=1M count=500 iflag=direct
md5sum tx_data.bin rx_data.bin
```

---

## 4. 常见排查点

1. **Host 无法识别盘符或报错 I/O Error**：
   * **排查**：Device 端作为后端的介质文件（backing storage）是否被多个实体同时以写模式挂载导致冲突；确保 Device 侧未在本地 mount 读写该同一块设备。
2. **UAS 协商失败自动退回 BOT**：
   * **排查**：Host 或 Device 控制器/PHY 是否工作在 USB 3.0 SuperSpeed 模式。UAS 在 USB 2.0 下部分系统不启用。
