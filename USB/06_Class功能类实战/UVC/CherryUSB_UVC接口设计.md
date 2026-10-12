# CherryUSB UVC 接口设计

本文说明 AX615 RT-Thread 项目中的**自研 CherryUSB 分支**：UVC Pump 接口和 DWC3 device controller port。以下 `usbd_video_pump_*`、request 排队模型及板级文件描述的是该分支，不能假定 CherryUSB upstream 已提供这些接口。

适用场景为 DWC3 工作在 USB 2.0 HS device 模式、静态 MJPEG UVC demo。本文根据原项目设计记录整理，未重新审计对应实现或执行板级验证。上板步骤见 [RTT CherryUSB UVC 操作指南](RTT_CherryUSB_UVC操作指南.md)，Linux Gadget 基础用法见 [UVC](../UVC.md)。

## 1. 从逐包写入到整帧提交

原项目旧路径一次只有一个 transfer 在途，应用维护帧偏移、UVC payload header、FID/EOF、ZLP 和逐包续传。ISO 对续传及时性敏感，应用调度延迟可能造成 service interval 空缺。

旧接口保留在 `class/video/usbd_video.h`：

```c
bool usbd_video_stream_split_transfer(uint8_t busid, uint8_t ep);
int  usbd_video_stream_start_write(uint8_t busid, uint8_t ep, uint8_t *ep_buf,
                                   uint8_t *stream_buf, uint32_t stream_len, bool do_copy);
```

新增 Pump 接口让应用提交整帧，类内部负责拆分 payload、填写 header、ZLP、暂停与多 request 排队。旧接口可以保留用于原有调用路径。

## 2. Pump API 与初始化

```c
struct usbd_video_pump_cfg {
    struct usbd_request *reqs;        /* request 池 */
    uint8_t **payloads;               /* 每个 request 的 payload 缓冲 */
    uint32_t nreq;                    /* request 数量 */
    uint32_t payload_size;            /* 单个 payload 字节数 */
    struct usbd_video_frame *frames;  /* 帧环 */
    uint32_t nframe;                  /* 帧环深度 */
    uint8_t ep_type;                  /* USB_ENDPOINT_TYPE_BULK / _ISOCHRONOUS */
};

int  usbd_video_pump_init(uint8_t busid, uint8_t ep, const struct usbd_video_pump_cfg *cfg);
int  usbd_video_pump_submit(uint8_t busid, uint8_t ep, uint8_t *frame, uint32_t len);
void usbd_video_pump_frame_done(uint8_t busid, uint8_t ep, uint8_t *frame, int status);
```

| 字段 | 约束 |
| --- | --- |
| `reqs` | 长度不少于 `nreq` 的 request 池，demo 使用 8 个 |
| `payloads` | 每个 request 对应独立 payload 缓冲 |
| `payload_size` | 与 `usbd_video_init_intf()` 的 `dwMaxPayloadTransferSize` 一致 |
| `frames`、`nframe` | 帧环及深度，demo 深度为 4 |
| `ep_type` | `USB_ENDPOINT_TYPE_BULK` 或 `USB_ENDPOINT_TYPE_ISOCHRONOUS` |

`cfg` 结构体会被拷贝，指向的池、指针数组和缓冲不会被拷贝，必须在整个 streaming 期间有效，通常使用静态存储。DMA 相关缓冲还需满足平台的布局、对齐和 cache 维护要求，不能只依赖 `static`。

初始化顺序如下：

1. 分配 request 池、每个 request 的 payload 缓冲及帧环，填好 `cfg`。
2. `usbd_desc_register()` 注册描述符，`usbd_add_interface()` 注册 UVC 接口。
3. 调用 `usbd_video_pump_init()`；端点由类内部注册，不再由应用重复 `usbd_add_endpoint()`。
4. 调用 `usbd_initialize()`。

参考例程是 `components/drivers/usb/cherryusb/demo/video_static_mjpeg_pump_template.c`，完整参数应以该分支例程及头文件为准。

## 3. 帧缓冲所有权与回调

| `usbd_video_pump_submit()` 返回值 | 含义及应用动作 |
| --- | --- |
| `0` | 帧已入队，应用停止修改该帧，等待对应 `frame_done` |
| `-USB_ERR_INVAL` | 参数或状态无效，例如空指针、零长度、端点不匹配或未成功初始化 |
| `-USB_ERR_NOTCONN` | 原接口约定为设备未配置或主机尚未 `SET_INTERFACE(alt=1)` |
| `-USB_ERR_BUSY` | 帧队列满，保留所有权，稍后重试 |

只有返回 `0` 才转移所有权。应用侧约定是：可写 → 提交成功后禁止改写 → `usbd_video_pump_frame_done()` 归还后才可复用。提交失败不能等待本次提交的完成回调。

`usbd_video_pump_frame_done()` 为弱定义，由应用覆盖；回调携带原始 `frame` 指针，按整帧通知，不按 payload 通知。`usbd_video_open()` / `usbd_video_close()` 通知流的开启与关闭。原记录说明默认回调在中断上下文执行，不应阻塞；定义 `CONFIG_USBDEV_VIDEO_PUMP_THREAD` 后由 Pump 线程处理。

### 必须核对的完成语义

原设计记录存在两种不同表述：接口部分写“整帧发送完成”，内部实现部分却写“最后一个 payload 被控制器接受后，`commit` 推进至帧尾即调用 `frame_done`”。**提交到控制器不等于总线传输完成，本文不将两者视为已经一致。**

当前记录明确为 copy 路径，即原始 frame 数据先复制到 request 的 payload 缓冲。如果原始帧的全部数据已经复制到独立缓冲，原始 frame 的归还可能早于 USB 发送结束；这仍需核对具体分支实现和 API 约定。尤其不能据此提前改写或回收仍在途的 payload 缓冲。接入实际视频源前，应确认 `frame_done` 代表“原始帧可复用”还是“整帧已发送”，以及取消与错误路径是否保持同样约定。

## 4. Pump 内部排队与取消

原项目记录的主要流程：

```text
应用 submit(frame)
  → 帧环
  → Pump 取空闲 request
  → encode：UVC header + 帧切片复制到 payload
  → usbd_ep_queue：交给 DCD
  → 成功接受后 commit：推进帧内偏移
  → request 完成回调：归还空闲 request，继续 Pump
```

- `usbd_video_pump_init()` 将 request 池串成 free 链表，借用 `req->context`，完成函数指向类内部回调。
- `frames` 使用 `frame_head`、`frame_tail`、`frame_count` 管理；`submit()` 只入环，不直接操作 DMA。
- `usbd_video_pump_run()` 从 free 链表取 request 并排队；没有空闲 request 时退出，等待完成事件续传。
- `usbd_video_encode()` 只填 payload，`usbd_video_commit()` 在控制器接受 request 后才推进偏移，避免提交失败跳过数据。
- 原记录称帧尾 commit 会翻转 `stream_frameid`、前移 `frame_head` 并通知一次 `frame_done`；其完成时机应按上一节核对。
- `usbd_video_pump_flush()` 在 `stop_pending` 时将帧环中未完成的帧以 `-USB_ERR_CANCEL` 归还；在途 request 通过完成回调回到 free 链表。

接口记录将 `SET_INTERFACE(alt=0)` 作为停流处理入口，并约定 reset/disconnect 时清理队列、归还已提交帧。不同主机与传输模式的实际停止请求应以抓包为准，不能只凭播放器“停止”按钮推断总线请求。

## 5. 自研 DWC3 port

该项目为 AX615 的 DWC3 新增 device port，参考 Linux v6.6 `drivers/usb/dwc3` 精简实现，保留 device 模式所需功能。这是项目当时的移植选择，不代表对当前 upstream 支持范围的判断。

| 文件（相对 CherryUSB 目录） | 作用 |
| --- | --- |
| `port/dwc3/usb_dc_dwc3.c` | 初始化、事件/中断、EP0、TRB ring、请求队列和 fault 恢复 |
| `port/dwc3/usb_dwc3_reg.h` | 寄存器与位域；硬件 TRB 为 16 字节，普通事件为 4 字节 |
| `port/dwc3/usb_dwc3_param.h` | 容量和轮询参数 |
| `port/dwc3/usb_glue_axera.c` | AX615 glue，基址 `0x08000000`，延时、临界区及 cache/DMA hooks |

### 请求与 TRB 隔离

每个端点维护 `pending`、`started`、`cancelled`、`faulted`、`done` 五条链表。每个 request 独占一个 64 字节 TRB group，每端点 16 组，每组最多 3 个 data TRB 加 1 个 link TRB，目的是隔离不同 request 的 cache line，减少 CPU 与控制器并发更新同一 cache line 的风险。

原记录说明一次 `STARTTRANSFER` 可以挂多个 request，控制器通过 `XFERINPROGRESS` / `XFERCOMPLETE` 上报进度与完成。request 被排队接受时不能立即复用其 TRB 或 payload。

### 取消与关闭

`usbd_ep_dequeue()` / `usbd_ep_close()` 将 started request 移到 cancelled，并发送 `ENDTRANSFER | CMDIOC`；等待 `EPCMDCMPLT` 确认控制器停止访问 TRB 后，才回收 request 并执行回调。

reset/disconnect 路径中，`dwc3_release_active_transfer()` 先释放控制器侧 transfer resource，再调用 `dwc3_abort_queue()` 清理软件队列。原项目此处直接写 `DEPCMD`，刻意不走 `dwc3_ep_command()`：总线复位时的命令错误不能直接触发 `dwc3_transfer_fault()` 拆除整个控制器。

原项目还引入 `stall_pending`：若仍有在途传输，先 `ENDTRANSFER`，等完成处理回收请求后再执行 `SET/CLEARSTALL`。故障背景见 [停止拔插后不出流](../../08_经典问题定位与案例/DWC3_UVC停止拔插后不出流.md)。

## 6. Cache、链接布局与重开流

AX615 的堆从 `__bss_end` 开始。项目在 `bsp/ax615/link.lds` 中将 `.noncacheable` 放入 `__bss_start..__bss_end`，避免其成为与堆重叠的孤儿段，同时覆盖 BSS 清零。原项目 map 记录该段约 156 KB，末尾等于 `__bss_end`。

`USB_NOCACHE_RAM_SECTION` 只是段标记；是否 cacheable 由 MMU 页表决定。`ALIGN(64)` 便于按 cache line 维护 DMA 缓冲，不等价于自动保证 DMA 一致性。

原项目重新开流修复要求 `dwc3_ring_init()` 在复用 ring 前清零、`usb_dcache_flush()`、`dwc3_axera_dmb()`，再交给控制器，避免旧 TRB/HWO 被执行。详见 [重新开流绿屏](../../08_经典问题定位与案例/DWC3_UVC重新开流绿屏.md)。

## 7. 实现边界与原项目结果

当前记录中的数据路径仍有 frame → payload 的一次复制。Bulk SG-DMA zero-copy 只有设计稿 `uvc-sg-dma-bulk-design.md`，**尚未实施**，不能将本接口描述为零拷贝。

原项目静态 MJPEG 测试记录：Bulk 直连约 44–45 MB/s，经 HUB 约 41 MB/s；ISO 配合 `VIDEO_BENCH=1` 不限速推流约 22 MB/s。这些是原项目条件下的结果，未在本仓库重新验证。具体设置及停止/拔插检查步骤见 [操作指南](RTT_CherryUSB_UVC操作指南.md)。
