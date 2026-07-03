
---

# 3. 我这边建议的默认规格理解

## 3.1 YUV 默认采用 YUY2/YUYV

PM 已确认 YUV 没有具体格式要求，因此我理解 sample 或基础验证中可以默认采用：

```text
YUY2 / YUYV
```

原因：

```text
1. YUY2/YUYV 是 UVC 中最常见的未压缩 YUV 格式；
2. Host 兼容性较好；
3. 不依赖 JPEG/H264/H265 编码器；
4. 适合作为 UVC 基础链路验证；
5. 便于区分 USB/UVC 问题和视频编码问题。
```

基础规格可以先按：

```text
YUY2 640x480@30
YUY2 640x360@30
```

说明：

```text
通常标准写法是 YUY2/YUYV，不是 YUV2。
```

---

## 3.2 UAC 默认采用 UAC1 基础双讲规格

PM 已确认 UAC 没有具体参数要求，因此我理解 USB Audio 基础验证可以默认采用：

```text
UAC1
PCM
16bit
16kHz
mono
capture + playback
```

也就是：

```text
Device -> Host：Microphone capture
Host -> Device：Speaker playback
```

原因：

```text
1. UAC1 兼容性较好；
2. 16kHz/16bit/mono 适合语音双讲；
3. 数据量小，适合基础链路验证；
4. 实现复杂度较低；
5. 后续如果客户需要，可再扩展 48kHz、stereo 等规格。
```

关于 3A，我的理解是：

```text
UAC 驱动只负责 USB Audio 数据通道；
AEC/NS/AGC 等 3A 算法不属于 USB 驱动本身；
如果需要，可以预留 3A stub，由音频/算法/AE 后续接入。
```

---

## 3.3 MJPEG/H264/H265 都需要，但职责需要拆分

PM 已确认：

```text
H265 / H264 / MJPEG 全部需要。
```

我这边理解需要拆成两类看。

---

### 3.3.1 MJPEG

MJPEG 是 UVC 中比较常见的压缩格式，和 UVC 驱动关系比较直接。

USB/UVC 驱动侧可以支持：

```text
1. MJPEG format descriptor；
2. MJPEG frame descriptor；
3. frame interval 配置；
4. Probe/Commit 基础流程；
5. streaming endpoint；
6. UVC event/control 基础处理。
```

应用侧需要负责：

```text
1. 提供 JPEG 帧数据；
2. 按 Host 选择的分辨率和帧率送帧；
3. 后续接真实 camera 或 JPEG 编码器。
```

我理解 MJPEG 可以作为比较明确的 UVC 格式支持项。

---

### 3.3.2 H264/H265

H264/H265 也需要支持，但我理解这部分不能简单等同于 USB 驱动工作。

H264/H265 涉及：

```text
1. 编码器集成；
2. VENC pipeline；
3. SPS/PPS/VPS；
4. GOP；
5. 码率控制；
6. I 帧控制；
7. Host 解码兼容；
8. App 是否支持对应格式。
```

这些内容更偏应用、多媒体或 AE 后续适配，不属于 USB UVC 驱动本身。

USB 驱动侧需要提供的是：

```text
1. H264/H265 format descriptor 框架；
2. 标准 GUID 或自定义 GUID 的配置能力；
3. Probe/Commit 基础协商通路；
4. UVC streaming endpoint；
5. userspace 到 gadget driver 的数据通路；
6. 基础 payload/framing 支持；
7. 帧边界传输的基本机制。
```

所以我的理解是：

```text
H264/H265 可以通过 UVC 通道传输；
但 H264/H265 码流生产、编码器配置、码率/GOP/SPS/PPS/VPS、Host 解码兼容等，不属于 USB 驱动侧主责。
```

---

# 4. H264/H265/MJPEG 与 USB UVC 驱动职责关系

这一点我认为需要重点说明，避免后续把完整视频应用能力全部归到 USB 驱动侧。

## 4.1 职责拆分表

| 模块 | USB 驱动侧 | 应用/多媒体/AE |
|---|---|---|
| USB Gadget 枚举 | 负责 | 不负责 |
| UVC function | 负责 | 不负责 |
| UVC descriptor 框架 | 负责 | 配合填写格式能力 |
| YUY2 descriptor | 负责 | 使用 |
| MJPEG descriptor | 负责 | 使用 |
| H264/H265 descriptor 框架 | 负责 | 配合具体格式定义 |
| endpoint/streaming 通道 | 负责 | 使用 |
| Probe/Commit 基础流程 | 负责 | 根据协商结果调整码流 |
| YUY2 测试图数据 | 可提供最小验证方式 | 后续产品化 |
| MJPEG JPEG 帧数据 | 不主责 | 负责提供 |
| H264/H265 编码 | 不负责 | 负责 |
| SPS/PPS/VPS/GOP/码率 | 不负责 | 负责 |
| sensor/ISP/VENC | 不负责 | 负责 |
| 3A 算法 | 不负责 | 音频/算法侧负责 |
| Host 解码 App | 不负责 | AE/客户/应用负责 |
| 客户环境适配 | 提供基础支持 | AE 主责 |

---

## 4.2 我的理解结论

```text
MJPEG 是 UVC 常见格式，USB 驱动侧可以提供较完整的 descriptor 和 streaming 支持。

H264/H265 也可以纳入 UVC 传输能力，但编码器集成、码流生产、Host 端解码兼容不属于 USB 驱动侧。

USB 驱动侧负责把 UVC 通道和基础框架打通，应用/多媒体侧负责把真实视频数据按协商结果送进来。
```

---

# 5. USB 驱动负责人职责边界

## 5.1 USB 驱动侧应负责

```text
1. USB Device/Gadget 基础能力；
2. configfs gadget sample；
3. UVC gadget function；
4. UVC1.1 descriptor 模板；
5. UVC1.0 compatibility descriptor 模板；
6. YUY2/MJPEG/H264/H265 format descriptor 框架；
7. UVC streaming endpoint；
8. Probe/Commit 基础流程；
9. XU descriptor 和 GET/SET 通路；
10. UAC1 gadget descriptor 和基础双向通道；
11. Sample setup/cleanup 脚本；
12. Host 侧基础测试说明。
```

---

## 5.2 USB 驱动侧可协助提供最小验证 demo，但不主责产品化应用

从 USB 驱动角度，我可以协助提供一些最小验证方式，用于证明 USB/UVC/UAC 链路是通的，例如：

```text
1. YUY2 彩条送帧 demo；
2. MJPEG 文件循环送帧 demo；
3. H264/H265 测试码流送帧 demo；
4. UAC 正弦波/PCM 文件 demo；
5. XU GET/SET 测试 demo。
```

但这些 demo 的定位是：

```text
简单验证链路；
便于 AE 或应用侧后续参考；
不等同于完整产品级应用；
不承诺性能、画质、稳定性和所有 Host 兼容性。
```

以下内容不建议作为 USB 驱动侧主责：

```text
1. 完整 camera 应用；
2. 真实 sensor pipeline；
3. ISP 调参；
4. H264/H265 编码器集成；
5. 码率/GOP/SPS/PPS/VPS 控制；
6. 3A 算法；
7. Host 端播放器/SDK；
8. 客户验收环境全覆盖；
9. 双路视频业务逻辑；
10. 多媒体 pipeline 产品化。
```

---

# 6. 应用 Sample 的提供思路

虽然应用 sample 的完整交付不属于我作为 USB 驱动工程师的主责，但如果项目需要验证链路，我这边可以提供或配合提供一些比较简单的思路。

定位是：

```text
目前比较简单；
基本满足需求验证；
不做产品化；
后续由 AE/应用/多媒体侧继续扩展。
```

---

## 6.1 YUY2 应用 sample 思路

目标：

```text
验证 UVC YUY2 基础出图。
```

实现方式：

```text
1. 打开 UVC gadget 对应 video node；
2. 响应 UVC events；
3. 根据 Host 选择的分辨率和帧率；
4. 生成彩条、灰阶、动态测试图；
5. 以 YUY2/YUYV 格式送给 UVC。
```

特点：

```text
1. 不依赖 camera sensor；
2. 不依赖 ISP；
3. 不依赖编码器；
4. 适合作为最基础 UVC 链路验证。
```

---

## 6.2 MJPEG 应用 sample 思路

目标：

```text
验证 UVC MJPEG 出流。
```

实现方式：

```text
1. 准备一组 JPEG 文件；
2. 按分辨率分类存放；
3. UVC 应用循环读取 JPEG 文件；
4. 根据 Host 选择的 MJPEG 分辨率送帧；
5. Host 侧用 Linux/Windows 工具打开验证。
```

特点：

```text
1. 不依赖 JPEG 编码器；
2. 实现简单；
3. 能基本满足 MJPEG 链路验证；
4. 后续 AE/应用侧可替换为真实 camera/JPEG encoder。
```

---

## 6.3 H264/H265 应用 sample 思路

目标：

```text
验证 H264/H265 码流可以通过 UVC 通道传输。
```

实现方式：

```text
1. 准备 H264/H265 测试码流文件；
2. UVC 应用按帧或按 NAL 读取测试码流；
3. 通过 UVC gadget streaming 通道送出；
4. Host 端使用指定工具接收；
5. 第一阶段不依赖真实 VENC。
```

定位：

```text
H264/H265 这类 sample 更适合作为 experimental/extension；
用于验证 USB/UVC 传输链路；
不等同于完整编码应用；
不承诺所有系统原生 Camera App 可直接识别。
```

后续如果要产品化，应由应用/多媒体侧接入：

```text
sensor -> ISP -> VENC -> H264/H265 -> UVC
```

---

## 6.4 UAC 应用 sample 思路

目标：

```text
验证 UAC 双向音频通道。
```

实现方式：

```text
1. Device -> Host capture：产生正弦波或读取 PCM 文件；
2. Host -> Device playback：接收 Host 播放数据并 dump 到文件；
3. 采用 UAC1 + 16kHz + 16bit + mono；
4. 预留 audio_process_3a_stub()。
```

3A stub 定位：

```text
audio_process_3a_stub() 当前只做 passthrough；
后续由音频/算法/AE 接入 AEC/NS/AGC。
```

---

# 7. 主要风险点

## 7.1 USB2.0 带宽风险

当前涉及的格式包括：

```text
YUY2
MJPEG
H264
H265
```

如果再叠加高分辨率、高帧率或双路视频，USB2.0 带宽会有明显压力。

风险包括：

```text
1. YUY2 只适合低分辨率；
2. MJPEG 高分辨率高帧率可能有带宽压力；
3. 双路同时出流风险更高；
4. H264/H265 带宽较友好，但 Host 兼容复杂；
5. 实际性能受 USB、编码器、内存带宽、Host App 共同影响。
```

我的建议是：

```text
USB 驱动侧可以验证基础传输链路；
但不应在驱动侧承诺所有分辨率/帧率组合满帧稳定。
```

---

## 7.2 H264/H265 兼容风险

H264/H265 需要重点注意 Host 侧兼容性。

风险包括：

```text
1. H264/H265 在 UVC 中需要明确封装方式；
2. H265 原生兼容性风险高于 H264；
3. Windows/macOS/Android 不同 App 支持差异较大；
4. 可能需要自定义 GUID；
5. 可能需要 Host 端专用工具或 SDK。
```

我的理解是：

```text
USB 驱动侧可以支持 UVC 传输框架；
但 Host App 是否能原生识别和解码，不应归为 USB 驱动问题。
```

---

## 7.3 UVC1.0 兼容风险

如果后续需要兼容只支持 UVC1.0 的老主控，例如 SSC/T21，需要注意：

```text
1. Linux4.9 默认 UVC Gadget 可能更偏 UVC1.1；
2. UVC1.0 不是简单修改 bcdUVC 就一定兼容；
3. 老主控可能对 descriptor、endpoint、format 比较敏感；
4. PC 能识别不代表 SSC/T21 一定能识别；
5. H264/H265 在 UVC1.0 下兼容风险更高。
```

我的建议是：

```text
如果确实要求 UVC1.0，USB 驱动侧可以提供 compatibility descriptor 框架；
但建议能力裁剪为 YUY2/MJPEG 基础分辨率；
H264/H265 和双路不建议放到 UVC1.0 基础兼容范围内。
```

---

## 7.4 职责边界风险

如果不提前明确，下面这些内容容易被误认为都是 USB 驱动侧负责：

```text
1. H264/H265 编码器集成；
2. camera pipeline；
3. sensor/ISP；
4. 码率/GOP/SPS/PPS/VPS；
5. 3A；
6. Host App；
7. 多 OS 客户验收；
8. 双路业务逻辑；
9. 产品级应用稳定性。
```

我的建议是：

```text
USB 驱动侧负责 UVC/UAC Gadget、descriptor、endpoint、control、streaming 框架和基础验证；
应用、多媒体、音频算法、Host App、客户验收适配由对应团队或 AE 承接。
```

---

# 8. 我这边的汇报结论

## 8.1 当前理解

```text
1. YUV 没有特殊要求，我理解默认采用 YUY2/YUYV 即可；
2. UAC 没有特殊要求，我理解默认采用 UAC1 + 16kHz + 16bit + mono 即可；
3. MJPEG/H264/H265 都需要，但三者职责不同；
4. MJPEG 与 UVC 驱动关系较直接，可以作为常见 UVC 格式支持；
5. H264/H265 需要 UVC 通道支持，但编码和 Host 解码兼容不属于 USB 驱动主责。
```

---

## 8.2 我这边可以负责的内容

```text
1. USB Device/Gadget 基础能力；
2. UVC gadget function；
3. UVC1.1 descriptor 模板；
4. UVC1.0 compatibility descriptor 模板；
5. YUY2/MJPEG/H264/H265 format descriptor 框架；
6. UVC streaming endpoint；
7. Probe/Commit 基础流程；
8. XU descriptor 和 GET/SET 通路；
9. UAC1 gadget descriptor 和基础双向通道；
10. 基础 setup/cleanup 脚本；
11. Host 侧基础测试说明。
```

---

## 8.3 不建议归到我这边主责的内容

```text
1. 完整 UVC 应用开发；
2. H264/H265 编码器集成；
3. sensor/ISP/VENC pipeline；
4. 码率/GOP/SPS/PPS/VPS 控制；
5. 3A 算法；
6. Host 端播放器/SDK；
7. 客户验收环境全覆盖；
8. 双路视频业务逻辑和性能承诺；
9. 产品级稳定性和兼容性验收。
```

---

## 8.4 应用 sample 的态度

```text
如果项目需要基础验证，我可以从 USB 驱动角度提供或配合提供简单 demo 思路：

1. YUY2 彩条；
2. MJPEG 文件循环；
3. H264/H265 测试码流送帧；
4. UAC PCM/正弦波；
5. XU GET/SET 测试。

这些 demo 主要用于验证 USB/UVC/UAC 链路，目前比较简单，基本满足需求验证，不作为完整产品级应用交付。
```

---

## 8.5 最终结论

```text
我这边建议把 USB 驱动侧工作边界明确在 Gadget/UVC/UAC 驱动能力、descriptor、endpoint、control、streaming 框架和基础验证上。

YUY2、MJPEG、H264、H265、UAC、XU 这些能力，从 USB 通道和描述符角度我可以支持或提供框架。

但 H264/H265 编码器、真实 camera pipeline、3A、Host App、客户环境适配、双路业务逻辑和产品化应用，不应作为 USB 驱动侧主责，需要由应用、多媒体、音频算法或 AE 后续承接。
```