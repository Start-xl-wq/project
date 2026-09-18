# CherryUSB UVC 接口设计说明

  

> 状态：接口改造与内部实现说明（本文回答“怎么改的”）。

> 配套文档：[RTT CherryUSB UVC 操作指南](https://wiki.aixin-chip.com/pages/viewpage.action?pageId=263698842)（怎么跑起来：硬件环境、编译烧录、demo 用法、稳定性结论）。

> 参考例程：`components/drivers/usb/cherryusb/demo/video_static_mjpeg_pump_template.c`，本文代码示例均基于该文件。

  

## 1. 背景与要解决的问题

  

- 旧接口一次只允许一个 transfer 在飞：应用必须自己把一帧拆成多个 payload，每个 payload 完成后才能发起下一次写入；ISO 端点稍有延迟就会漏掉 service interval。

- 应用还要自己处理 UVC payload header（`bHeaderLength`、`frameIdentifier`、`endOfFrame`）、整包时的 ZLP、无帧时的占位与暂停等细节。

- Pump 接口把这些全部收进类内部：应用只提交**整帧**，类内部完成拆帧、补 header、补零包、暂停与多请求并发。

- 旧接口仍然保留，可与 Pump 共存（`class/video/usbd_video.h` 中 `usbd_video_stream_split_transfer()` / `usbd_video_stream_start_write()` 未删除）。

  

## 2. 改造前：旧接口与痛点

  

旧接口签名（`class/video/usbd_video.h`）：

  

```c

bool usbd_video_stream_split_transfer(uint8_t busid, uint8_t ep);

int  usbd_video_stream_start_write(uint8_t busid, uint8_t ep, uint8_t *ep_buf,

                                   uint8_t *stream_buf, uint32_t stream_len, bool do_copy);

```

  

| 维度 | 旧接口 |

| --- | --- |

| 在途 transfer | 一次一个，完成回调后才能续下一个 |

| 拆帧 | 应用自己做，并维护帧内偏移 |

| UVC header / FID / EOF | 应用自己填 |

| ZLP、空帧占位 | 应用自己处理 |

| ISO service interval | 依赖应用及时续流，容易漏 |

| 多请求并发 | 无，吞吐受每包往返限制 |

  

## 3. 改造后：新接口

  

### 3.1 API

  

```c

struct usbd_video_pump_cfg {

    struct usbd_request *reqs;        /* request 池 */

    uint8_t **payloads;               /* 每个 request 的 payload 缓冲 */

    uint32_t nreq;                    /* request 数量 */

    uint32_t payload_size;            /* 单个 payload 字节数 */

    struct usbd_video_frame *frames;  /* 帧环 */

    uint32_t nframe;                  /* 帧环深度 */

    uint8_t ep_type;                  /* USB_ENDPOINT_TYPE_BULK / _ISOCHRONOUS */

};

  

int  usbd_video_pump_init(uint8_t busid, uint8_t ep, const struct usbd_video_pump_cfg *cfg);

int  usbd_video_pump_submit(uint8_t busid, uint8_t ep, uint8_t *frame, uint32_t len);

void usbd_video_pump_frame_done(uint8_t busid, uint8_t ep, uint8_t *frame, int status);

```

  

`usbd_video_pump_frame_done()` 在类内是 `__WEAK`，由应用覆盖实现。

  

### 3.2 `pump_cfg` 字段

  

| 字段 | 说明 |

| --- | --- |

| `reqs` | request 池数组，长度不小于 `nreq` |

| `payloads` | 每个 request 对应的 payload 缓冲指针数组 |

| `nreq` | request 数量（demo 用 8） |

| `payload_size` | 单个 payload 字节数，需与 `usbd_video_init_intf()` 的 `dwMaxPayloadTransferSize` 保持一致 |

| `frames` | 帧环数组，应用提交的帧先进这里 |

| `nframe` | 帧环深度（demo 用 4） |

| `ep_type` | `USB_ENDPOINT_TYPE_BULK` 或 `USB_ENDPOINT_TYPE_ISOCHRONOUS` |

  

注意：`cfg` 结构体本身会被拷贝，但 `reqs` / `payloads` / `frames` 指向的数组**不会**被拷贝，必须在整个 streaming 期间保持有效（通常定义为 static）。

  

### 3.3 `submit()` 返回值与所有权

  

| 返回值 | 含义 |

| --- | --- |

| `0` | 帧已入队；缓冲所有权移交 DMA，`frame_done` 之前应用不得再写 |

| `-USB_ERR_INVAL` | 参数或状态无效：`frame == NULL`、`len == 0`、ep 不匹配、pump 未初始化成功 |

| `-USB_ERR_NOTCONN` | 主机未 `SET_INTERFACE(alt=1)` 或设备未配置 |

| `-USB_ERR_BUSY` | 帧队列满，稍后重试 |

  

所有权三阶段：应用可写 → `submit()` 返回 0 后 DMA 只读 → `frame_done()` 回调后应用可写。**只有返回 0 才移交所有权。**

  

### 3.4 回调

  

- `usbd_video_open(busid, intf)` / `usbd_video_close(busid, intf)`：流开/关通知。

- `usbd_video_pump_frame_done(busid, ep, frame, status)`：**整帧**发送完成时调用一次（不是每个 payload 一次）；`frame` 就是 `submit()` 传入的指针。

- 回调默认在**中断上下文**执行，不要阻塞；定义 `CONFIG_USBDEV_VIDEO_PUMP_THREAD` 后改为在 pump 线程执行。

  

### 3.5 调用流程对照

  

| 阶段 | 改造前（旧接口） | 改造后（Pump） |

| --- | --- | --- |

| 初始化 | 注册描述符/接口/端点，应用自己管理 payload buffer | 注册描述符/接口后调用 `usbd_video_pump_init()`（必须在 `usbd_initialize()` 之前）；类内部自己注册端点，应用不要再 `usbd_add_endpoint()` |

| 启动流 | 主机 `SET_INTERFACE(alt=1)`，应用开始逐包写 | 主机 `SET_INTERFACE(alt=1)` 触发 `usbd_video_open()`，应用随后提交整帧 |

| 发送 | 应用拆帧，逐个 payload 调 `usbd_video_stream_start_write()` | `usbd_video_pump_submit(frame, len)` |

| 完成通知 | 每个 payload 一次回调，应用续下一个 | 整帧一次 `usbd_video_pump_frame_done()` |

| 停止流 | 主机 `SET_INTERFACE(alt=0)`，应用自行停止 | `SET_INTERFACE(alt=0)` 后类内部停流，并把未完成的帧以 `-USB_ERR_CANCEL` 归还应用 |

| 复位 / 断开 | 应用自行处理 | 类内部清理队列，所有已提交帧都会被归还 |

  

## 4. 用法示例

  

### 4.1 初始化顺序

  

```c

static struct usbd_request pump_reqs[PUMP_NREQ];

static uint8_t pump_payload[PUMP_NREQ][MAX_PAYLOAD_SIZE];

static uint8_t *pump_payload_ptr[PUMP_NREQ];

static struct usbd_video_frame pump_frames[PUMP_NFRAME];

  

struct usbd_video_pump_cfg cfg = { 0 };

  

for (unsigned i = 0; i < PUMP_NREQ; i++) {

    pump_payload_ptr[i] = pump_payload[i];

}

cfg.reqs         = pump_reqs;

cfg.payloads     = pump_payload_ptr;

cfg.nreq         = PUMP_NREQ;

cfg.payload_size = MAX_PAYLOAD_SIZE;

cfg.frames       = pump_frames;

cfg.nframe       = PUMP_NFRAME;

cfg.ep_type      = USB_ENDPOINT_TYPE_BULK;   /* 或 USB_ENDPOINT_TYPE_ISOCHRONOUS */

  

usbd_desc_register(busid, video_descriptor);

usbd_add_interface(busid, usbd_video_init_intf(busid, 0, VIDEO_IN_EP, VIDEO_INT_EP, ...));

usbd_add_interface(busid, usbd_video_init_intf(busid, 1, VIDEO_IN_EP, VIDEO_INT_EP, ...));

usbd_video_pump_init(busid, VIDEO_IN_EP, &cfg);   /* 必须在 usbd_initialize() 之前 */

usbd_initialize(busid, reg_base, usbd_event_handler);

```

  

要点：

  

1. `usbd_video_pump_init()` 必须在 `usbd_initialize()` 之前调用；

2. `reqs` / `payloads` / `frames` 指向的数组必须是 static，streaming 期间一直有效；

3. 端点由类内部注册，应用不要再调用 `usbd_add_endpoint()`。

  

### 4.2 生产者线程写法

  

```c

static volatile bool frame_in_use[PUMP_NFRAME];

  

void usbd_video_pump_frame_done(uint8_t busid, uint8_t ep, uint8_t *frame, int status)

{

    (void)busid; (void)ep; (void)status;

    for (unsigned i = 0; i < PUMP_NFRAME; i++) {

        if (frame_buffer[i] == frame) {

            frame_in_use[i] = false;

            break;

        }

    }

}

  

static void video_test(uint8_t busid)

{

    uint8_t slot = 0;

  

    while (1) {

        if (!streaming) {

            msleep(1);

            continue;

        }

        if (frame_in_use[slot]) {

            msleep(1);

            continue;

        }

        memcpy(frame_buffer[slot], mjpeg_data, mjpeg_len);

        frame_in_use[slot] = true;

        if (usbd_video_pump_submit(busid, VIDEO_IN_EP, frame_buffer[slot], mjpeg_len) < 0) {

            frame_in_use[slot] = false;

            msleep(1);

            continue;

        }

        slot = (uint8_t)((slot + 1U) % PUMP_NFRAME);

        msleep(1000 / CAM_FPS);

    }

}

```

  

## 5. UVC 内部怎么改的

  

1. **静态 request 池 + free 链表**：`usbd_video_pump_init()` 把 `cfg.reqs` 里的每个 request 串成 free 链表（借用 `req->context`），`req->complete` 指向类内部的完成回调；

2. **帧环**：`frames` 数组用 `frame_head` / `frame_tail` / `frame_count` 管理，`submit()` 只入环，不直接动 DMA；

3. **pump 主循环** `usbd_video_pump_run()`：从 free 链表取 request → 有帧就编码成 payload → `usbd_ep_queue()` 提交 → 没有 free request 就退出（“Everything is in flight”）；

4. **编码/提交分离** `usbd_video_encode()` / `usbd_video_commit()`：`encode` 只填 payload（UVC header + 帧数据切片），`commit` 才推进 `frame_offset`；只有 request 被控制器接受后才 commit，避免失败时在流里留下空洞；

5. **整帧完成通知**：`commit` 发现 `frame_offset >= frame->len` 时翻转 `stream_frameid`、`frame_head` 前进，并**只调一次** `usbd_video_pump_frame_done()`；

6. **停流/取消** `usbd_video_pump_flush()`：`stop_pending` 时把帧环里未发完的帧逐个以 `-USB_ERR_CANCEL` 归还应用；在途 request 由完成回调归还到 free 链表；

7. **回调上下文**：默认在中断上下文完成，定义 `CONFIG_USBDEV_VIDEO_PUMP_THREAD` 后由名为 `uvc_pump` 的线程处理，信号量在完成回调里 `give`。

  

## 6. DWC3 port 与 DCD 怎么改的

  

### 6.1 为什么要新写一个 port

  

CherryUSB 上游没有 AX615 这套 DWC3（Synopsys DesignWare USB3，此处作 USB 2.0 HS device）的 device controller 驱动，因此参考 Linux v6.6 `drivers/usb/dwc3` 精简重写，只保留 device 模式所需功能。

  

### 6.2 port 的组成

  

| 文件 | 作用 |

| --- | --- |

| `port/dwc3/usb_dc_dwc3.c` | DCD 主体：初始化/去初始化、控制器与事件配置、中断处理、EP0 控制路径、queue endpoint 的 TRB ring、请求队列、fault 恢复 |

| `port/dwc3/usb_dwc3_reg.h` | 寄存器偏移与位域；硬件 TRB 16 字节、普通事件 4 字节 |

| `port/dwc3/usb_dwc3_param.h` | 容量与轮询参数 |

| `port/dwc3/usb_glue_axera.c` | AX615 glue：controller base `0x08000000`、延时、临界区、cache/DMA hooks |

  

### 6.3 请求模型（与旧 DCD 最大的区别）

  

- 每个端点维护 pending / started / cancelled / faulted / done 五条链表；

- **每个 request 独占一个 64 字节的 TRB group**（每端点 16 组，最多 3 个 data TRB + 1 个 link TRB），CPU 不会写到控制器正在更新的 cache line；

- 一次 `STARTTRANSFER` 可以挂多个 request，控制器逐个用 `XFERINPROGRESS` / `XFERCOMPLETE` 上报。

  

### 6.4 取消与关闭

  

- `usbd_ep_dequeue()` / `usbd_ep_close()` 把 request 从 started 移到 cancelled，并下发 `ENDTRANSFER | CMDIOC`；

- 等 `EPCMDCMPLT` 确认控制器不再访问 TRB 之后才回收 request 并回调，避免提前复用缓冲；

- reset / disconnect 新增 `dwc3_release_active_transfer()`，在 `dwc3_abort_queue()` 之前直接写 `DEPCMD` 释放控制器侧 transfer resource。这里刻意不走 `dwc3_ep_command()`：总线复位时命令可能返回错误，不能因此触发 `dwc3_transfer_fault()` 把控制器整个拆掉。

  

### 6.5 两处真机问题的修复

  

1. **端点 STALL 时序**：主机对仍有在途传输的端点下发 `SET_FEATURE` / `CLEAR_FEATURE(HALT)`（Windows 点“停止”发的就是 `CLEAR_FEATURE(HALT)`）时，不能直接下发 `SETSTALL` / `CLEARSTALL`。引入 `stall_pending`：先发 `ENDTRANSFER`，在完成回调里回收 request，**之后再**下发 stall 命令。

2. **复位/断开释放传输资源**：reset / disconnect 原先只清理软件队列，控制器侧 transfer resource 仍被占用，重新枚举后传输起不来；改为先 `dwc3_release_active_transfer()` 再 `dwc3_abort_queue()`（见 6.4）。

  

这两处修复在主机侧用 stub 寄存器模型做了“先红后绿”的回归用例。

  

### 6.6 与 Linux 参考的差异

  

- Linux 用 `dwc3_stop_active_transfer()`，本 port 用等价的 `dwc3_release_active_transfer()`，并保持 fault-free；

- Linux UVC 的 SG 优化尚未实现：bulk zero-copy 目前只有设计稿 `uvc-sg-dma-bulk-design.md`，未实施。

  

## 7. 当前实现状态与带宽

  

- demo 默认走 **bulk IN**（`VIDEO_USE_BULK=1`）：上板验证约 **44–45 MB/s**（USB 2.0 HS bulk 接近上限；经 USB HUB 约 41 MB/s）；

- **isochronous** 变体（`VIDEO_USE_BULK=0`）也已上板验证：配合 `VIDEO_BENCH=1` 不限速推流，带宽稳定约 **22 MB/s**；正常 640×480@30fps 配置本身流量较小，需用 BENCH 方式才能把链路打满；

- 已知限制：协议层 unsupported SETUP 的恢复尚未闭环；bulk zero-copy（SG-DMA）仅有设计稿；

- 硬件环境、烧录底包、稳定性测试与 PotPlayer 演示步骤见操作指南：[RTT CherryUSB UVC 操作指南](https://wiki.aixin-chip.com/pages/viewpage.action?pageId=263698842)。