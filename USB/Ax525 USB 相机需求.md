
## 1. BSP 职责

负责 **USB Gadget / UVC / UAC 驱动侧能力**。

### 1.1 USB Device / Gadget 基础能力

```text
1. USB Device 模式基础功能；
2. Gadget configfs 配置；
3. UDC 绑定和枚举；
4. VID/PID、字符串描述符、配置描述符；
5. USB2.0 High Speed 下的基础传输能力。
```

---

### 1.2 UVC 驱动侧能力

```text
1. UVC Gadget function；
2. UVC descriptor 配置；
3. VideoControl interface；
4. VideoStreaming interface；
5. Streaming endpoint；
6. UVC Probe / Commit 流程；
7. UVC event/control 处理；
8. UVC 数据通路。
```

---

### 1.3 UVC 格式支持框架

需要在 USB 驱动侧支持或预留以下格式描述能力：

```text
1. YUY2 / YUYV；
2. MJPEG；
3. H264；
4. H265。
```

说明：

```text
YUY2/MJPEG 属于常见 UVC 格式，驱动侧需要支持 descriptor 和 streaming 框架。

H264/H265 驱动侧负责 UVC 传输框架和 descriptor 能力，不负责编码器和码流生成。
```

---

### 1.4 XU 扩展控制

```text
1. 支持 UVC Extension Unit descriptor；
2. 支持 Vendor GUID；
3. 支持 XU GET/SET 控制通路；
4. 提供基础 selector 框架；
5. 支持应用层处理具体 XU 命令。
```

说明：

```text
负责 XU 通路和框架；
具体业务命令由应用/AE/客户定义。
```

---

### 1.5 UAC 驱动侧能力

USB Audio

```text
1. UAC Gadget function；
2. UAC1 descriptor；
3. PCM 音频数据通路；
4. Capture：Device -> Host；
5. Playback：Host -> Device；
6. 基础音频参数配置。
```

默认建议：

```text
UAC1
16bit
16kHz
mono
capture + playback
```

---

### 1.6 UVC1.0 / UVC1.1 支持

```text
1. UVC1.1 descriptor 模板；
2. 如需要兼容老主控，再提供 UVC1.0 compatibility descriptor；
3. UVC1.0 建议只做基础 YUY2/MJPEG 能力。
```

---

### 1.7 基础验证支持

提供基础验证方式，例如：

```text
1. YUY2 彩条出图；
2. MJPEG 文件送帧；
3. H264/H265 测试码流送帧；
4. UAC PCM/正弦波验证；
5. XU GET/SET 验证；
6. Host 侧基础测试说明。
```


---

# 2. App 职责

```text
1. 完整 UVC 应用；
2. Camera 应用框架；
3. Sensor 驱动；
4. ISP pipeline；
5. H264/H265 编码器集成；
6. VENC 配置；
7. SPS/PPS/VPS；
8. GOP、码率、I 帧控制；
9. JPEG/H264/H265 真实码流生产；
10. 3A 算法，包括 AEC/NS/AGC；
11. Host 端播放器或 SDK；
12. Windows/macOS/Android App 适配；
13. 客户验收环境全覆盖；
14. 双路视频业务逻辑；
15. 产品级性能和稳定性承诺。
```

---
