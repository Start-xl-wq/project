好的，已按你的要求去掉第 6、7、9、10 章，保留应用开发者最常用的接口、初始化和线程写法。

---

# UVC MJPEG Pump 接口速查（上层应用版）

> **参考例程源码**：`/home/public/heduanchu/share/615/Saturn_usb/video_static_mjpeg_pump_template.c`
> 本文所有代码示例均基于该文件。

## 1. 解决什么问题

旧接口一次只允许一个 transfer 在飞，应用自己拆帧、自己续流，ISO 端点容易漏 service interval。

Pump 接口：应用只交整帧，类内部完成拆帧、补 UVC header、补零包/暂停、多请求并发。旧接口仍可用。

## 2. 回调

```c
void usbd_video_open(uint8_t busid, uint8_t intf);   // 开始取流/停止时调用
void usbd_video_close(uint8_t busid, uint8_t intf);
void usbd_video_pump_frame_done(uint8_t busid, uint8_t ep, uint8_t *frame, int status); // 整帧完成
```

- 默认在**中断上下文**执行：不要阻塞、不要 sleep、不要做重活，只置标志/发信号量。
- 如定义 `CONFIG_USBDEV_VIDEO_PUMP_THREAD`，则回调在线程中执行。
- `frame_done` 的 `frame` 就是 `submit` 时传入的指针，靠它反查缓冲槽位。
- `frame_done` 每帧一次，不是每个 payload 一次。

## 3. 初始化顺序

```c
usbd_desc_register(busid, video_descriptor);
usbd_add_interface(busid, usbd_video_init_intf(busid, &intf0, INTERVAL, MAX_FRAME_SIZE, MAX_PAYLOAD_SIZE));
usbd_add_interface(busid, usbd_video_init_intf(busid, &intf1, INTERVAL, MAX_FRAME_SIZE, MAX_PAYLOAD_SIZE));

cfg.reqs = pump_reqs;
cfg.payloads = pump_payload_ptr;
cfg.nreq = PUMP_NREQ;
cfg.payload_size = MAX_PAYLOAD_SIZE;
cfg.frames = pump_frames;
cfg.nframe = PUMP_NFRAME;
cfg.ep_type = USB_ENDPOINT_TYPE_ISOCHRONOUS;

usbd_video_pump_init(busid, VIDEO_IN_EP, &cfg);
usbd_initialize(busid, reg_base, usbd_event_handler);
```

- `usbd_video_pump_init()` **必须**在 `usbd_initialize()` 之前。
- 类内部自己注册端点，**应用不要再调用 `usbd_add_endpoint()`**。

## 4. `pump_cfg` 字段

```c
struct usbd_video_pump_cfg {
    struct usbd_request *reqs;      // nreq 个请求描述符，pump_init 会初始化
    uint8_t **payloads;             // 每个指针至少 payload_size 字节，需 DMA 可读
    uint8_t nreq;                   // 同时在飞 payload 数，建议 >= 4
    uint32_t payload_size;          // 单个 payload 字节数，含 UVC header，必须 > 12
    struct usbd_video_frame *frames; // 帧槽位数组，类内部使用
    uint8_t nframe;                 // 帧队列深度，决定 submit 能收几帧
    uint8_t ep_type;                // ISO 或 BULK，必须和描述符一致
};
```

- `payload_size` 与 `usbd_video_init_intf()` 的 `dwMaxPayloadTransferSize` 保持一致。
- `cfg` 会被拷贝，但 `reqs/payloads/frames` 指向的数组不会被拷贝，必须整个 streaming 期间有效，通常用 `static`。

## 5. `submit()` 要点

```c
int usbd_video_pump_submit(uint8_t busid, uint8_t ep, uint8_t *frame, uint32_t len);
```

| 返回值 | 含义 |
| --- | --- |
| `0` | 帧已入队，缓冲区归 DMA，不可写 |
| `-USB_ERR_INVAL` | `frame==NULL`、`len==0`、ep 不匹配、pump_init 未成功 |
| `-USB_ERR_NOTCONN` | 主机未 `SET_INTERFACE(alt=1)` 或设备未配置 |
| `-USB_ERR_BUSY` | 帧队列满，主机消费不过来 |

- **只有返回 0 才移交所有权**；返回非 0 时缓冲仍在应用手里，可重试或丢弃。
- `len` 没有上限校验，应用自己保证不超描述符限制。
- 缓冲区所有权：应用可写 → `submit` 后类/DMA 只读 → `frame_done` 后应用可写。

## 6. 生产者线程关键写法

```c
while (1) {
    if (!streaming) { usb_osal_msleep(1); continue; }
    if (frame_in_use[slot]) { usb_osal_msleep(1); continue; }

    // 填充 frame_buffer[slot]
    frame_in_use[slot] = true;                  // 先置位
    if (usbd_video_pump_submit(busid, EP, frame_buffer[slot], len) < 0) {
        frame_in_use[slot] = false;             // 失败立刻清
        usb_osal_msleep(1);
        continue;
    }
    slot ^= 1;
    usb_osal_msleep(1000 / FPS);                // 节拍
}

void usbd_video_pump_frame_done(uint8_t busid, uint8_t ep, uint8_t *frame, int status) {
    for (int i = 0; i < N_FRAME; i++) {
        if (frame_buffer[i] == frame) {
            frame_in_use[i] = false;
            break;
        }
    }
}
```
