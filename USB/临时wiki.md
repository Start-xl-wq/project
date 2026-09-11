# UVC MJPEG 多请求流式发送（pump）接口说明

  

面向 UVC 应用开发，基于例程 `video_static_mjpeg_pump_template.c`。

  

## 0. 参考文件

  

cherryusb 根目录：

  

```

kernel/rtos/rt-thread-lts-v4.1.x/components/drivers/usb/cherryusb/

```

  

（`kernel/rtos_sdk/rt-thread-lts-v4.1.x/components/drivers/usb/cherryusb/` 下有一份完全相同的副本，两份内容一致）

  

| 内容 | 路径（相对 cherryusb 根目录） |

| --- | --- |

| 例程 | `demo/video_static_mjpeg_pump_template.c` |

| 接口声明（**编译时需要加入 include 路径**） | `class/video/usbd_video.h`（第 35–87 行） |

| 接口实现 | `class/video/usbd_video.c`（第 905–1136 行） |

| 依赖的类型定义 | `class/video/usb_video.h`（描述符宏、`struct usbd_request` 等） |

  

---

  

## 1. 这个接口解决什么问题

  

旧接口 `usbd_video_stream_start_write()` 一次只让**一个 transfer 在飞**：应用交出一帧后，必须靠端点完成回调里调用 `usbd_video_stream_split_transfer()` 把下一段喂进去。等速（isochronous）端点对这一点很敏感 —— **某个 service interval 上没有准备好的 transfer，这个 interval 的数据就直接丢了，USB 不会重传**。而从"完成中断"到"重新组好一个 payload"的往返时间，塞不进一个 125µs 的微帧。

  

pump 接口把这件事挪进类内部：应用只递交**整帧**，拆帧、写 UVC payload header、以及在帧与帧之间的空隙里保持端点有包可发，全部由类完成。

  

| | 旧接口 `usbd_video_stream_start_write` | pump 接口 `usbd_video_pump_*` |

| --- | --- | --- |

| 应用交出 | 单帧 + 自己反复调 `split_transfer` | 整帧，一次调用 |

| 同时在飞的请求 | 1 个 | `nreq` 个（例程 8 个） |

| 帧间空隙 | 应用自己保证 | ISO 自动补 0 长度 payload 占位 |

| 端点注册 | 应用 `usbd_add_endpoint()` | 类内部注册，应用不要注册 |

| 完成通知 | `ep_cb(nbytes)`，按 payload | `usbd_video_pump_frame_done()`，按整帧 |

| 帧 ID（frameIdentifier） | 类内 `stream_frameid` | 类内 `stream_frameid`（每帧翻转） |

| 缓冲区生命周期 | 应用自己管 | 类管，靠 `frame_done` 归还 |

  

旧接口保留未动，两条路可共存。

  

---

  

## 2. 应用需要实现的回调

  

三个都是 `__WEAK`，应用同名实现即覆盖。

  

```c

/* 主机把 VS 接口切到 alternate setting 1（开始取流）/ 切回 0（停止）时调用 */

void usbd_video_open (uint8_t busid, uint8_t intf);

void usbd_video_close(uint8_t busid, uint8_t intf);

  

/* 一帧发完、缓冲区可以复用。不是每个 payload 一次，是每帧一次 */

void usbd_video_pump_frame_done(uint8_t busid, uint8_t ep, uint8_t *frame, int status);

  

/* 通用 USB 设备事件，传给 usbd_initialize() */

static void usbd_event_handler(uint8_t busid, uint8_t event);

```

  

### 运行上下文（重要）

  

当前工程 **没有** 定义 `CONFIG_USBDEV_VIDEO_PUMP_THREAD`（`bsp/ax615/rtconfig.h` 中无此宏），因此 pump 走的是**非线程**分支：

  

- `usbd_video_pump_run()` 直接在**端点完成回调 / 中断上下文**里执行（`usbd_video.c:1009`）

- 所以 `usbd_video_pump_frame_done()`、`usbd_video_open()`、`usbd_video_close()` **都在中断上下文被调用**

- 这三个回调里**不要阻塞、不要 `usb_osal_msleep`、不要做重活**。需要唤醒生产者线程，就给信号量 / 置标志位

- `usbd_video_open` / `close` 是从 `video_notify_handler` 的 `USBD_EVENT_SET_INTERFACE` 分支里调的（`usbd_video.c:709-722`），同样是中断上下文

  

如果希望这些回调在线程里跑，定义下面三个宏并重新编译，类会自动创建名为 `uvc_pump` 的线程：

  

```c

CONFIG_USBDEV_VIDEO_PUMP_THREAD

CONFIG_USBDEV_VIDEO_PUMP_STACKSIZE   /* 线程栈大小 */

CONFIG_USBDEV_VIDEO_PUMP_PRIO        /* 线程优先级 */

```

  

### `usbd_video_close` 与总线事件

  

`usbd_event_handler` 里收到 `USBD_EVENT_CONFIGURED` / `USBD_EVENT_DISCONNECTED` 时，**应用自己要清掉本地的 streaming 标志**（例程 `:94-105`）。类内部的 `pump->streaming` 不会被复位事件清零，所以应用侧标志是生产者循环唯一的可靠闸门 —— 不过 `submit()` 内部还有 `usb_device_is_configured()` 兜底，最坏情况返回 `-USB_ERR_NOTCONN` 而不是发坏数据。

  

---

  

## 3. 初始化流程

  

严格按这个顺序（例程 `video_init()`，`:140-168`）：

  

```c

void video_init(uint8_t busid, uintptr_t reg_base)

{

    struct usbd_video_pump_cfg cfg = { 0 };

  

    usbd_desc_register(busid, video_descriptor);                    /* 1. 描述符 */

  

    usbd_add_interface(busid, usbd_video_init_intf(

        busid, &intf0, INTERVAL, MAX_FRAME_SIZE, MAX_PAYLOAD_SIZE)); /* 2. VC 接口 */

    usbd_add_interface(busid, usbd_video_init_intf(

        busid, &intf1, INTERVAL, MAX_FRAME_SIZE, MAX_PAYLOAD_SIZE)); /* 2. VS 接口 */

  

    for (unsigned i = 0; i < PUMP_NREQ; i++)                        /* 3. 填 cfg */

        pump_payload_ptr[i] = pump_payload[i];

    cfg.reqs         = pump_reqs;

    cfg.payloads     = pump_payload_ptr;

    cfg.nreq         = PUMP_NREQ;

    cfg.payload_size = MAX_PAYLOAD_SIZE;

    cfg.frames       = pump_frames;

    cfg.nframe       = PUMP_NFRAME;

    cfg.ep_type      = USB_ENDPOINT_TYPE_ISOCHRONOUS;

  

    if (usbd_video_pump_init(busid, VIDEO_IN_EP, &cfg) < 0) {       /* 4. pump */

        USB_LOG_ERR("video pump init failed\r\n");

        return;

    }

  

    usbd_initialize(busid, reg_base, usbd_event_handler);           /* 5. 启动 */

}

```

  

**`usbd_video_pump_init()` 必须在 `usbd_initialize()` 之前调用。**

  

### `usbd_video_init_intf()`

  

```c

struct usbd_interface *usbd_video_init_intf(uint8_t busid, struct usbd_interface *intf,

                                            uint32_t dwFrameInterval,        /* 单位 100ns，30fps -> 333333 */

                                            uint32_t dwMaxVideoFrameSize,    /* 最大单帧字节数 */

                                            uint32_t dwMaxPayloadTransferSize); /* 最大单个 payload */

```

  

这两个值会作为 PROBE/COMMIT 的默认值报给主机（`usbd_video.c:731-765`）。VC 和 VS 两个接口都要注册（例程注册了 `intf0` / `intf1`），它们共用同一份 probe/commit 状态，所以两处传**相同**的参数即可。返回值恒为 `intf`，不会失败。

  

---

  

## 4. `struct usbd_video_pump_cfg` 逐字段

  

```c

struct usbd_video_frame {

    uint8_t *buf;

    uint32_t len;

};

  

struct usbd_video_pump_cfg {

    struct usbd_request *reqs;      /* nreq 个请求描述符，由应用提供 */

    uint8_t            **payloads;  /* nreq 个 payload 缓冲指针，与 reqs 一一对应 */

    uint8_t              nreq;      /* 请求池深度 = 同时在飞的 payload 数 */

    uint32_t             payload_size; /* 单个 payload 字节数，**含** UVC header */

    struct usbd_video_frame *frames;/* nframe 个帧槽位，类内部使用 */

    uint8_t              nframe;    /* 帧队列深度 */

    uint8_t              ep_type;   /* ISOCHRONOUS 或 BULK，必须与描述符一致 */

};

```

  

| 字段 | 谁填 | 说明 |

| --- | --- | --- |

| `reqs` | 应用 | `static struct usbd_request reqs[nreq];` 内容不必初始化，`pump_init` 会 `memset` 并设好 `buf` 与 `complete` |

| `payloads` | 应用 | `payloads[i]` 指向一块**至少 `payload_size` 字节**的缓冲，不能为 NULL |

| `nreq` | 应用 | 建议 ≥ 4；越大越能扛调度抖动，代价是内存 |

| `payload_size` | 应用 | 必须 `> 12`（UVC header 长度），且与端点 `wMaxPacketSize` / `dwMaxPayloadTransferSize` 自洽 |

| `frames` | 应用 | 只需提供数组，内容由类的 `submit()` 填写，**应用不要碰** |

| `nframe` | 应用 | 帧队列深度，决定 `submit()` 能被接受几帧 |

| `ep_type` | 应用 | ISO → 无帧可发时补 0 长度包占住 interval；Bulk → 直接暂停等下一帧 |

  

### `usbd_video_pump_init()` 的校验规则

  

返回 `0` 成功，负 errno 失败（`usbd_video.c:1027-1075`）：

  

- `cfg` / `cfg->reqs` / `cfg->payloads` / `cfg->frames` 任一为 NULL → `-USB_ERR_INVAL`

- `nreq == 0` 或 `nframe == 0` → `-USB_ERR_INVAL`

- `payload_size <= 12` → `-USB_ERR_INVAL`

- `ep_type` 既不是 ISO 也不是 BULK → `-USB_ERR_INVAL`

- `payloads[i] == NULL` → `-USB_ERR_INVAL`

  

另外：**类自己会注册这个端点**，应用不要再对它调用 `usbd_add_endpoint()`（例程 `:158-162` 的注释）。也因此不存在"应用忘记续流"导致流停掉的路径。

  

---

  

## 5. `usbd_video_pump_submit()` —— 递交一帧

  

```c

int usbd_video_pump_submit(uint8_t busid, uint8_t ep, uint8_t *frame, uint32_t len);

```

  

立即返回，帧在后台发送。检查顺序与返回值（`usbd_video.c:1077-1106`）：

  

| 返回 | 条件 |

| --- | --- |

| `0` | 帧已入队 |

| `-USB_ERR_INVAL` | `frame == NULL` 或 `len == 0`；`ep` 与 `pump_init` 时不一致；`pump_init` 未成功 |

| `-USB_ERR_NOTCONN` | 主机尚未 `SET_INTERFACE(alt=1)`（未开始取流）或设备未被配置 |

| `-USB_ERR_BUSY` | 帧队列满（`frame_count >= nframe`），即主机消费不过来 |

  

### 缓冲区所有权（关键）

  

```

应用可写  ──submit() 成功──►  类/控制器只读  ──frame_done()──►  应用可写

```

  

- **`submit()` 返回 0 之后，到 `frame_done()` 报告这一帧之前，`frame` 指向的内存应用不可写**。这是 DMA 正在读的数据，提前覆盖会花屏或撕裂

- 只有返回 `0` 才移交所有权。返回 `-USB_ERR_BUSY` / `-USB_ERR_INVAL` 时所有权还在应用手里，可以立刻重试或丢弃

- `len` **没有上限校验**，类比不对照 `dwMaxVideoFrameSize` 或描述符里的 `MAX_FRAME_SIZE`。超长帧照样会被发出去，是否合法由应用自己保证

  

### 队列深度怎么选

  

- `nframe` 决定 `submit()` 能被接受几帧。例程取 2，配合"两个帧缓冲轮转 + `-USB_ERR_BUSY` 时跳过本次采集"的策略

- `nreq` 决定同时在飞的 payload 数。HS 下 8 × 3072B ÷ 约 24.6MB/s ≈ **1ms** 的抗抖动窗口，也就是被调度晚 1ms 也不会丢 interval

  

---

  

## 6. 内存与缓冲区要求

  

### payload 缓冲（必须）

  

```c

USB_NOCACHE_RAM_SECTION USB_MEM_ALIGNX uint8_t pump_payload[PUMP_NREQ][MAX_PAYLOAD_SIZE];

```

  

- **必须非 cache（或按平台方式做过 cache 维护）+ cache line 对齐**

- 不能与其它数据共享同一条 cache line，否则 cache 回写会踩坏相邻数据

- 这些缓冲由 DMA 直接读取，`pump_init` 只把指针存起来，不做任何拷贝或重定位

- 例程把"怎么分配、什么属性"留给应用，因为对齐与 cache 属性是平台决策（`:70-78` 的注释）

  

### 帧缓冲

  

类内部是 `usb_memcpy()` 从 `frame->buf` 拷到 payload（`usbd_video.c:934`），即**帧缓冲只被 CPU 读**。所以：

  

- 如果帧是 CPU 采集/编码写进去的 → 普通内存即可

- 如果帧由 **ISP / 硬件编码器 / DMA 写** → 同样需要非 cache + 对齐，否则 CPU 读到的可能是旧数据

  

例程统一加了 `USB_NOCACHE_RAM_SECTION USB_MEM_ALIGNX`（`:87`），跟随这个写法最省事。

  

### 生命周期

  

`cfg` 结构体本身会被**拷贝**进类内（`pump->cfg = *cfg`），但 **`cfg` 指向的所有数组和缓冲区不会被拷贝**。它们必须在整个 streaming 期间保持有效 —— 例程用 `static` 数组正是这个原因。

  

---

  

## 7. 描述符与端点约束

  

### 端点类型必须自洽

  

`cfg.ep_type` 必须和描述符里的端点属性一致：

  

| ep_type | 端点描述符属性 | 无帧可发时的行为 |

| --- | --- | --- |

| `USB_ENDPOINT_TYPE_ISOCHRONOUS` | `0x05` | 提交 0 长度 payload 占住 service interval（`:978-980`） |

| `USB_ENDPOINT_TYPE_BULK` | `0x02` | 暂停发，等下一帧（`:981-984`） |

  

例程用的是 ISO：`USB_ENDPOINT_DESCRIPTOR_INIT(VIDEO_IN_EP, 0x05, VIDEO_PACKET_SIZE, 0x01)`。

  

### `payload_size` 与端点包大小的关系（HS）

  

```c

#define MAX_PAYLOAD_SIZE  3072                                          /* 3 个 transaction */

#define VIDEO_PACKET_SIZE (unsigned int)(((MAX_PAYLOAD_SIZE / 3)) | (0x02 << 11))

/*                              = 1024 字节/微帧      | wMaxPacketSize[12:11] = 2 → 3 transactions */

```

  

`wMaxPacketSize` 的 bit[12:11] 表示**每微帧的 transaction 数**：

  

- 1 个 transaction/微帧 → 上限 1024B/125µs ≈ **8.2 MB/s**（大帧不够用）

- 3 个 transaction/微帧 → 约 **24.6 MB/s**

  

Full Speed 分支是单 transaction、1020 字节/1ms 间隔。

  

### 几个容易踩的点

  

1. **`dwMaxPayloadTransferSize` 不会回灌到 pump**。主机在 PROBE/COMMIT 里协商出的值只更新 `probe`/`commit` 结构，pump 始终按 `cfg.payload_size` 切分。所以传给 `usbd_video_init_intf()` 的 `dwMaxPayloadTransferSize` 应与 `cfg.payload_size` 保持一致，不要指望运行时被主机改小。

2. **`VIDEO_INT_EP (0x83)` 实际没被用到**。描述符用的是 `VIDEO_VC_NOEP_DESCRIPTOR_INIT`，它产出的 VC 接口 `bNumEndpoints = 0`，宏内部也没有引用传入的端点地址。别以为设备上报了中断端点。

3. **`open` / `close` 不区分接口号**。`video_notify_handler` 收到任何 `SET_INTERFACE` 都按 alt 值分发（`usbd_video.c:709-722`）。正常取流顺序下（先给控制接口设 alt 0，再给 VS 接口设 alt 1）不会出问题，但如果主机在取流过程中对控制接口做 `SET_INTERFACE(alt=0)`，会触发一次多余的 `close()`。应用侧对 `close()` 做幂等处理即可。

4. **Bulk 端点需要 ZLP**。代码对 bulk 请求设了 `req->zero = 1`，由控制器在必要时补零长包（`:987`），应用不用管。

  

---

  

## 8. 生产者线程写法

  

例程 `video_test()`（`:171-199`）的标准模式 —— 两个帧缓冲轮转：

  

```c

#define PUMP_NFRAME 2

USB_NOCACHE_RAM_SECTION USB_MEM_ALIGNX uint8_t frame_buffer[2][32 * 1024];

static volatile bool frame_in_use[2];

static volatile bool streaming;

  

void video_test(uint8_t busid)

{

    unsigned slot = 0;

  

    while (1) {

        if (!streaming) {                    /* 主机还没取流 */

            usb_osal_msleep(1);

            continue;

        }

        if (frame_in_use[slot]) {            /* 两个缓冲都还在总线上 */

            usb_osal_msleep(1);              /* 跳过本次采集，别覆盖 DMA 正在读的数据 */

            continue;

        }

  

        /* ▼ 真实应用在这里把一帧填进 frame_buffer[slot]（采集/编码/DMA） */

        usb_memcpy(frame_buffer[slot], cherryusb_mjpeg, sizeof(cherryusb_mjpeg));

  

        frame_in_use[slot] = true;           /* 先置位，再提交 */

        if (usbd_video_pump_submit(busid, VIDEO_IN_EP, frame_buffer[slot],

                                   sizeof(cherryusb_mjpeg)) < 0) {

            frame_in_use[slot] = false;      /* 队列满 / 流已停：丢弃本帧，下轮重试 */

            usb_osal_msleep(1);

            continue;

        }

        slot ^= 1U;                          /* 切换缓冲 */

        usb_osal_msleep(1000 / CAM_FPS);     /* 30fps 节拍 */

    }

}

  

void usbd_video_pump_frame_done(uint8_t busid, uint8_t ep, uint8_t *frame, int status)

{

    for (unsigned i = 0; i < 2; i++) {

        if (frame_buffer[i] == frame) {      /* 靠指针认领是哪个槽位 */

            frame_in_use[i] = false;

            break;

        }

    }

}

```

  

要点：

  

- `frame_in_use` 必须在 `submit()` **之前**置位，否则 `frame_done` 可能在置位前就把标志清了

- `frame_done` 的参数就是当初 `submit` 传进去的 `frame` 指针，靠它反查槽位

- 不要指望 ISO 丢帧能重传 —— 只能靠 `nreq` 深度和补零包把 interval 填住

- `status` 为 0 表示整帧发完；负数表示被取消或被控制器打断（当前实现只在成功路径回调，`status` 恒为 0）

  

---

  

## 9. 排查速查

  

| 现象 | 可能原因 |

| --- | --- |

| `submit()` 一直返回 `-USB_ERR_NOTCONN` | 主机没做 `SET_INTERFACE(alt=1)`；或设备未配置（枚举失败） |

| `submit()` 频繁返回 `-USB_ERR_BUSY` | 主机取流速度跟不上生产速度 → 提高 `nframe`，或降低帧率/分辨率 |

| 画面花屏、撕裂 | payload 缓冲没配成非 cache，或没做 cache line 对齐；或帧缓冲在 `frame_done` 之前被改写 |

| 画面卡顿、周期性丢帧 | `nreq` 太浅，扛不住调度抖动；或生产者线程优先级太低、节拍不稳 |

| `pump_init` 返回 `-USB_ERR_INVAL` | `payload_size <= 12`；`nreq`/`nframe` 为 0；某条 `payloads[i]` 为 NULL；`ep_type` 写错 |

| 主机看不到流 | 端点描述符属性与 `ep_type` 不符（ISO 要 `0x05`，Bulk 要 `0x02`）；`wMaxPacketSize` 的 transaction 数不够 8.2MB/s |

  

---

  

## 10. 例程参数一览（HS 分支，640×480@30）

  

```c

#define VIDEO_IN_EP   0x81          /* 流端点 */

#define VIDEO_INT_EP  0x83          /* 未实际使用，见 §7.2 */

  

#define MAX_PAYLOAD_SIZE  3072      /* 3 transaction × 1024B */

#define VIDEO_PACKET_SIZE (1024 | (0x02 << 11))

  

#define WIDTH  640

#define HEIGHT 480

#define CAM_FPS 30

#define INTERVAL       (10000000 / CAM_FPS)              /* 333333，单位 100ns */

#define MIN_BIT_RATE   (640 * 480 * 16 * 30)

#define MAX_BIT_RATE   (640 * 480 * 16 * 30)

#define MAX_FRAME_SIZE (640 * 480 * 2)                   /* 614400 */

  

#define PUMP_NREQ   8               /* 请求池：约 1ms 抗抖动 */

#define PUMP_NFRAME 2               /* 帧队列深度 */

```