# USB Gadget Mass Storage (U盘) 快速验证指南

本文档用于嵌入式 Linux 板卡枚举为 **标准 USB 闪存盘 (Mass Storage / BOT)** 后的创建镜像、挂载与读写传输验证。

---

## 1. Device 端准备后端存储并绑定

USB Mass Storage 需要一个块设备或普通文件镜像作为底层物理介质。

### 1.1 创建虚拟 U 盘介质镜像 (以 512MB 为例)
```bash
# 1. 创建 512MB 虚拟磁盘镜像文件
dd if=/dev/zero of=/tmp/usb_disk.img bs=1M count=512

# 2. 格式化为 FAT32 文件系统 (Windows/Linux 通用)
mkfs.vfat -F 32 /tmp/usb_disk.img
```

### 1.2 挂载/绑定到 Gadget Mass Storage
* **若使用 ConfigFS 配置**：
  ```bash
  # 将镜像路径写入 LUN 的 file 属性
  echo "/tmp/usb_disk.img" > /sys/kernel/config/usb_gadget/g1/functions/mass_storage.usb0/lun.0/file
  ```
* **若使用传统 g_mass_storage 模块**：
  ```bash
  modprobe g_mass_storage file=/tmp/usb_disk.img removable=1
  ```

---

## 2. Host 端识别验证

通过 USB 连接 Host PC：

* **Windows PC**：
  * “此电脑”中自动弹出“可移动磁盘”盘符。
  * 可直接双击进入，格式化或存放文件。
* **Linux PC**：
  ```bash
  dmesg | grep -E "usb-storage|sd "
  lsblk # 查看生成的 /dev/sdX 节点
  ```

---

## 3. 双向数据传输与一致性验证

### 3.1 Host 侧读写验证 (以 Linux Host 为例)
```bash
# 1. 挂载 U 盘
sudo mkdir -p /mnt/udisk
sudo mount /dev/sdb1 /mnt/udisk

# 2. Host 写入大文件并刷盘
dd if=/dev/urandom of=/mnt/udisk/test_100m.bin bs=1M count=100
sync

# 3. 校验写入文件的 MD5
md5sum /mnt/udisk/test_100m.bin > host.md5

# 4. 卸载以确保数据完全落盘
sudo umount /mnt/udisk
```

### 3.2 Device 侧挂载与一致性核验
在 Host 卸载或未写入时，Device 端可回环挂载该镜像进行数据校验：

```bash
# 1. Device 端挂载镜像
mkdir -p /mnt/img_check
mount -o loop /tmp/usb_disk.img /mnt/img_check

# 2. 查看文件并计算 MD5
ls -lh /mnt/img_check/test_100m.bin
md5sum /mnt/img_check/test_100m.bin

# 3. 卸载检查目录
umount /mnt/img_check
```

---

## 4. 常见排查点与避坑指南

1. **绝对禁忌：双端同时以读写模式挂载同一个镜像**：
   * **原因**：FAT32/EXT4 等传统文件系统非集群文件系统，Host 和 Device 两端各自有 Page Cache，如果同时写入同一个物理镜像，会导致文件系统元数据破坏、数据丢损或乱码。
   * **规则**：若 Host 正在读写，Device 端切勿同时 `mount` 写；如果 Device 需要写，先卸载或设为只读 (`ro=1`)。
2. **Windows 提示“需要格式化磁盘才能使用”**：
   * **排查**：在 Linux 端用 `mkfs.vfat` 格式化时是否缺少分区表；直接通过 Windows 格式化该盘符一次即可正常读写。
