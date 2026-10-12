# RTT CherryUSB UVC 操作指南

本文针对 AX615 NOR 开发板、`AX615_fastnor` 配置及项目自研的 CherryUSB DWC3 port，说明静态 MJPEG UVC demo 的编译、出流和停止/拔插验证。该项目基于 RT-Thread LTS v4.1.x，将 CherryUSB device 栈移植到 `kernel/rtos`；本文配置和 Pump 接口不代表 CherryUSB 上游版本。

本文数据和稳定性结论均为**原项目记录**，不是本仓库重新验证的结果。通用 Linux Gadget 验证见 [UVC](../UVC.md)，实现说明见 [CherryUSB UVC 接口设计](CherryUSB_UVC接口设计.md)。

## 1. 准备环境

| 项目 | 配置 |
| --- | --- |
| 开发板 | AX615 NOR，`AX615_fastnor` |
| 工程 | 含该移植实现的 SDK 工程，RTOS 源码在 `kernel/rtos` |
| 编译环境 | Linux，使用 SDK 配套工具链和构建环境 |
| 串口 | CH340 接 RTOS 专属 UART2，`921600`、8N1 |
| USB 主机 | Windows，PotPlayer |
| 出流格式 | 设备 `cherryusb uvc demo`，`640*480 29.96fps` |

原项目开发板的串口丝印有误：标注 `CTS` 的引脚实际为 UART2 TX，标注 `RTS` 的引脚实际为 UART2 RX。应以实际板级信号为准，板端 TX/RX 与 CH340 的 RXD/TXD 交叉连接并共地。

## 2. 编译与烧录

首次在工程根目录的 `build/` 下全量构建以生成依赖；直接编译 RTOS 可能缺依赖：

```sh
cd build
make p=AX615_fastnor all install -j120
```

完成首次构建且未清理 `out` 后，后续仅修改 RTOS 时，在工程根目录进入 `kernel/rtos` 编译：

```sh
cd kernel/rtos
make p=AX615_fastnor all install -j32
```

产物相对工程根目录为：

```text
build/out/AX615_fastnor_uclibc/images/rtos_signed.bin
```

使用与该 RTOS 版本配套的基础镜像，按 SDK 烧录流程更新 `rtos_signed.bin` 后复位。本仓库不包含基础镜像、烧录工具或完整 SDK；该指南不能替代平台镜像配套说明。

### 移植配置检查

- `bsp/ax615/rtconfig.h` 启用该项目的 CherryUSB device、HS、AXERA DWC3、video 和 template-video 配置。
- `bsp/ax615/applications/usb_probe.c` 通过 `INIT_APP_EXPORT` 初始化 USB，调用 `video_init()`，创建 `uvc_pump` 线程运行 `video_test()`。
- 保留 `bsp/ax615/link.lds` 的 `.noncacheable` 输出段。该平台堆从 `__bss_end` 开始，项目把该段放在 `__bss_start..__bss_end` 内，避免 DMA 缓冲作为孤儿段落入堆区域，并使其纳入 BSS 清零。

`.noncacheable` 的段名和链接布局本身不会关闭 cache；内存属性由 MMU 页表决定。`ALIGN(64)` 用于 cache line 对齐。更多说明见接口设计文档。

## 3. 选择 Bulk 或 ISO demo

参考文件：

```text
components/drivers/usb/cherryusb/demo/video_static_mjpeg_pump_template.c
```

| 宏 | 值 | 用途 |
| --- | --- | --- |
| `VIDEO_USE_BULK` | `1`，默认 | Bulk IN |
| `VIDEO_USE_BULK` | `0` | Isochronous IN |
| `VIDEO_BENCH` | `1`，默认 | 不限速推流，每 512 帧打印带宽 |
| `VIDEO_BENCH` | `0` | 按 `CAM_FPS` 推流，默认 30 fps |

按需修改这些宏后重新编译、烧录。`MAX_PAYLOAD_SIZE`、`PUMP_NREQ`、`PUMP_NFRAME`、`VIDEO_IN_EP` 等参数应与该 demo 的描述符和 Pump 配置配套，不能只修改一处。

同目录还有 `video_static_mjpeg_template.c`、`video_static_yuyv_template.c`、`video_static_h264_template.c`、`audio_*_template.c` 等模板；原项目板级配置只使能 template-video，本文验证范围是静态 MJPEG Pump demo。

## 4. 枚举与打开画面

1. 开发板断电，接好 UART2 和 USB 设备口，再上电。
2. 串口检查初始化日志，原项目示例为 `[CherryUSB] dwc3 reg_base=0x08000000 ...`。
3. 在 Windows 设备管理器或 PotPlayer 设备列表确认 `cherryusb uvc demo`。
4. 打开 PotPlayer，按 `Alt+D`，选择“摄像头”。
5. 选择设备 `cherryusb uvc demo`，格式选 `640*480 29.96fps`，打开/播放并确认出图。
6. 测带宽时设置 `VIDEO_BENCH=1`，观察串口输出：

```text
BULKBW <n> frames, window <ms> ms -> <x> MB/s
```

## 5. 原项目吞吐记录

测试条件为 AX615、Windows 主机、静态 MJPEG demo；下表是不限速推流的记录，不是任意主机、线缆或真实摄像头数据源都能达到的保证值。

| 配置 | 连接方式 | 原项目记录 |
| --- | --- | --- |
| `VIDEO_USE_BULK=1`、`VIDEO_BENCH=1` | 开发板直连主机 | 约 44–45 MB/s |
| Bulk | 经 USB HUB | 约 41 MB/s |
| `VIDEO_USE_BULK=0`、`VIDEO_BENCH=1` | 原项目 ISO 测试 | 约 22 MB/s |

正常 `640×480@30fps` 推流的业务流量较小，不能用该配置下的吞吐判断链路峰值。原记录未给出完整主机型号、HUB 型号和重复测试统计，跨环境对比需重新测量。

## 6. 停止、暂停与拔插验证

原项目在 Windows + 静态 MJPEG Bulk UVC 下记录了以下结果；复现时应同时观察画面、重新枚举和串口是否继续响应。

| 操作 | 原项目修复后的结果 |
| --- | --- |
| 出流中点“停止”，再打开 | 正常出流 |
| 出流中暂停，再继续 | 正常出流 |
| 出流中直接拔出，再插回 | 重枚举后正常出流 |
| 出流 → 停止 → 拔出 → 插回 → 打开，反复执行 | 正常出流，控制台不再挂死 |

原项目修复过两类问题：重新开流时旧 TRB/HWO 状态导致短暂绿屏，以及停止/拔插时在途传输和 transfer resource 未正确回收导致不出流。定位与修复细节见：

- [DWC3 UVC 重新开流绿屏](../../08_经典问题定位与案例/DWC3_UVC重新开流绿屏.md)
- [DWC3 UVC 停止拔插后不出流](../../08_经典问题定位与案例/DWC3_UVC停止拔插后不出流.md)
