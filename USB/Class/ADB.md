# USB Gadget ADB 快速验证指南

本文档用于嵌入式 Linux 板卡枚举为 **ADB (Android Debug Bridge)** 后的快速连通性与双向传输验证。

---

## 1. Device 端确认与服务启动

确保 Gadget 已绑定 ADB function（通常使用 FunctionFS 挂载）：

```bash
# 1. 确认 FunctionFS 挂载点与设备节点就绪
ls -l /dev/usb-ffs/adb/
# 预期可见 ep0, ep1, ep2

# 2. 启动 adbd 守护进程（若系统未自动启动）
adbd &

# 3. 检查进程是否存在
ps | grep adbd
```

---

## 2. Host 端识别验证

通过 USB 线连接 PC，在 Host 端终端执行：

```bash
# 检查设备是否在线
adb devices
```
* **正常状态**：显示设备序列号及 `device`。
* **异常状态**：
  * 显示 `unauthorized`：在 Device 端确认授权弹窗或拷贝 Host 公钥到 `/data/misc/adb/adb_keys`。
  * 显示 `offline`：执行 `adb kill-server && adb start-server`。

---

## 3. 双向功能与传输验证

### 3.1 终端交互验证 (Device Shell)
```bash
# Host 端直接进入板卡 Shell
adb shell
```

### 3.2 文件双向传输压测验证

#### 方向 1：Host -> Device (Push)
```bash
# Host 端生成测试大文件并推送
dd if=/dev/urandom of=test_host.bin bs=1M count=100
adb push test_host.bin /tmp/

# 校验 MD5
md5sum test_host.bin
adb shell md5sum /tmp/test_host.bin
```

#### 方向 2：Device -> Host (Pull)
```bash
# Device 端生成测试文件
adb shell "dd if=/dev/urandom of=/tmp/test_dev.bin bs=1M count=100"

# Host 端拉取并校验
adb pull /tmp/test_dev.bin .
adb shell md5sum /tmp/test_dev.bin
md5sum test_dev.bin
```

---

## 4. 常见排查点

1. **`adb devices` 为空**：
   * Host 端检查 `lsusb`（Linux）或设备管理器，确认 VID/PID 是否被识别。
   * Linux Host 需配置 udev 规则添加当前 VID/PID 权限：`/etc/udev/rules.d/51-android.rules`。
2. **`adbd: cannot open /dev/usb-ffs/adb/ep0`**：
   * FunctionFS 尚未正确挂载或 Gadget 配置尚未完成。
