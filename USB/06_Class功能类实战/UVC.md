# USB Gadget UVC 快速验证指南

本文档用于嵌入式 Linux 板卡枚举为 **UVC (USB 视频类摄像头)** 后的画面输出与推流验证。

---

## 1. 设备节点确认

### 1.1 Device 端确认
```bash
# 查看 UVC Gadget 视频输出节点
ls -l /dev/video*
```
> 通常存在两类节点：V4L2 采集卡节点（输入源）与 UVC Gadget 输出节点（输出端，驱动日志中通常标明为 `uvc gadget /dev/videoX`）。

### 1.2 Host 端确认
* **Linux PC**：
  ```bash
  dmesg | grep uvcvideo
  ls -l /dev/video* # 预期新识别出视频输入节点
  ```
* **Windows PC**：
  打开“设备管理器” -> “相机”，确认生成对应的 USB 摄像头设备（例如“UVC Camera”）。

---

## 2. 视频流推送验证

Device 端需有应用程序（如 `uvc-gadget` 或 demo 程序）向 `/dev/videoX` 写入图像帧。

### 2.1 Device 端推流 (以自带测试彩条/源为例)
```bash
# 运行 uvc 推流程序（将图像送入 UVC 输出端点）
# 示例：分辨率 1920x1080，格式 MJPEG / YUYV
uvc-gadget -v /dev/video0 -u /dev/video1 -s 1920x1080 -f 1 &
```

### 2.2 Host 端画面捕获与查看

#### 方案 A：Windows PC 查看
1. 打开 Windows 自带的 **“相机”** 应用。
2. 点击右上角“切换摄像头”图标，切换至刚插入的 USB 虚拟摄像头，观察是否能正常出图且无花屏/掉帧。

#### 方案 B：Linux PC 查看 (命令行快速抓图或预览)
```bash
# 方式 1：使用 mpv 或 ffplay 直接播放预览
ffplay -f v4l2 -video_size 1920x1080 /dev/video0

# 方式 2：使用 v4l2-ctl 抓取单帧
v4l2-ctl -d /dev/video0 --set-fmt-video=width=1920,height=1080,pixelformat=MJPG --stream-mmap --stream-count=1 --stream-to=frame.jpg
```

---

## 3. 常见排查点

1. **Host 打开相机黑屏或提示“相机未就绪”**：
   * **排查**：Device 端推流程序是否已正常启动且不断供帧；USB 传输带宽是否满足当前选定的未压缩格式（YUYV 高分辨率建议切换为 MJPEG 或 H.264）。
2. **画面撕裂或卡顿**：
   * **排查**：检查 UDC 控制器的 ISOC (等时传输) 或 Bulk 端点配置是否满足帧率带宽要求，重点核对 `wMaxPacketSize` 与 `bInterval`。
