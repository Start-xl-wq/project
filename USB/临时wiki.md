# 算力棒 USB 驱动设计 - 传输层

## 一、传输层的作用

传输层是整个驱动栈的地基，向上层（聚合层）提供一个**异步、深队列、双向**的 block 收发接口，向下对接 Linux functionfs 的 endpoint 文件。它的核心职责就一件事：

> **把固定大小的 block（URB）以尽可能高的并发喂给 USB 硬件，用队列深度掩盖单次传输的往返延迟，跑满 SuperSpeed 带宽。**

它**不关心** block 里装的是什么（TLV、slab、credit 报文都一样对待），**不做**聚合/拆帧（那是聚合层），**不做**速率匹配（那是流控层）。它只做一件事：**可靠、高并发地搬运定长 block**。

为什么必须异步深队列？USB 每次传输都有协议往返开销（令牌包、握手包、总线调度间隙）。如果同步收发——发一个 block、等完成、再发下一个——总线在等待期间空闲，带宽利用率极低。用 AIO 一次性挂起几十个 iocb，让 host controller 流水线式连续调度，才能填满总线。

---

## 二、技术选型：functionfs + AIO

**functionfs（FunctionFS）**：Linux gadget 子系统提供的用户态 USB function 接口。通过读写 `/dev/ffs/ep1`、`/dev/ffs/ep2` 这样的 endpoint 文件收发数据，descriptor 和 string 在初始化时一次性写入 `ep0`。选它的原因：驱动逻辑全在用户态，无需写内核 gadget 驱动，调试、迭代快。

**Linux AIO（libaio，io_submit/io_getevents）**：对 functionfs 的 endpoint fd 提交异步 I/O。选它而不是同步 read/write 或 epoll 的原因：

- functionfs 的 endpoint fd 对 `O_NONBLOCK` + epoll 支持不完整，且 epoll 只解决"何时可读写"，解决不了"一次挂起多个传输"。
- AIO 天然支持**深队列**：`io_submit` 一次提交多个 iocb，内核逐个转成 URB 排入 host controller，完成后通过 `io_getevents` 批量收割。这正好匹配"深队列掩盖延迟"的需求。

```
用户态                          内核态                    硬件
 │                               │                        │
 │─ io_submit(N iocb) ──────────→│                        │
 │                               │─ 转成 N 个 URB ────────→│ host controller
 │                               │                        │ 流水线连续调度
 │                               │←──── URB 完成 ──────────│
 │←─ io_getevents(批量收割) ─────│                        │
```

---

## 三、整体结构

传输层维护两条独立的通道，分别对应两个 functionfs endpoint。每条通道内部 TX / RX 方向各自维护独立的 iocb ring。所有通道共享一个 `io_context_t`，用一个线程统一 `io_getevents` 收割。

```
                       io_context_t（共享 AIO 上下文）
                                   │
        ┌──────────────────────────┴──────────────────────────┐
        │                                                      │
  msg endpoint (ep1)                              pkt endpoint (ep2)
  ┌──────────────────────────┐                  ┌──────────────────────────────┐
  │ block size = 4KB         │                  │ block size = 1MB             │
  │                          │                  │                              │
  │  TX iocb ring (depth=8)  │                  │  TX iocb ring (depth=32~64)  │
  │  [iocb][iocb]...[8]      │                  │  [iocb][iocb]...[64]         │
  │   ↓ 每个 = 一次小 URB     │                  │   ↓ 每个 = 一个完整 slab      │
  │                          │                  │                              │
  │  RX iocb ring (depth=8)  │                  │  RX iocb ring (depth=32~64)  │
  │  [iocb][iocb]...[8]      │                  │  [iocb][iocb]...[64]         │
  │   ↓ 预挂起，等 host 下发   │                  │   ↓ 预挂起，等 host 下发1MB    │
  └──────────────────────────┘                  └──────────────────────────────┘
```

两条通道的参数差异（由上层需求决定，传输层只按配置执行）：

| | msg endpoint (ep1) | pkt endpoint (ep2) |
|---|---|---|
| block size | 4KB | 1MB |
| AIO 队列深度 | 4~8 | 32~64 |
| 设计目标 | 低延迟，接受低利用率 | 高吞吐，队列越深越好 |
| 服务对象 | 控制/命令/credit 报文 | 大数据帧（slab） |

**为什么分成两个 endpoint**：粒度隔离（小包低延迟 vs 大包高吞吐互不干扰），且为流控层的 credit 报文预留独立通道（credit 报文借道 msg 通道，不被大数据阻塞）。这是上层正确性的物理前提。

---

## 四、TX 方向

上层（聚合层）把一个填好的 buffer 交给传输层，传输层封装成 iocb 提交。

```
上层调用 tx_submit(channel, buf, len)
    │
    ▼
从对应通道的 TX iocb ring 取一个空闲 iocb
    │
    ├── ring 无空闲 iocb ──→ 返回 EAGAIN / 阻塞
    │                        （由上层双缓冲 + 流控决定如何处理）
    ▼
io_prep_pwrite(iocb, ep_fd, buf, len, 0)
    │
    ▼
io_submit(ctx, 1, &iocb)  ──→ 内核转 URB 排入 host controller
    │
    ▼
（异步）io_getevents 收到该 iocb 完成事件
    │
    ▼
回调上层 tx_complete(iocb->buf)，归还 iocb 到 ring
```

要点：

- **TX 提交是纯异步的**：`io_submit` 立即返回，不等传输完成。上层可以连续提交多个 buffer 填满 ring depth，让 host controller 流水线调度。
- **iocb 与 buffer 的生命周期**：buffer 由上层（聚合层双缓冲）拥有，传输层只借用引用直到 `tx_complete`。完成前上层不得复用该 buffer。
- **ring 满即背压**：TX ring 无空闲 iocb 时返回 EAGAIN。这是传输层能感知的最底层背压——但它只反映"本端 AIO 未排空"，不反映对端消费进度，所以真正的端到端速率匹配必须靠流控层。

---

## 五、RX 方向

RX 与 TX 相反：传输层**主动预挂起**一批接收 iocb 到 ring，等 host 下发数据填充，完成后回调上层。

```
初始化：把 RX ring 全部 iocb 预挂起
    │
    ▼
for each iocb in RX ring:
    io_prep_pread(iocb, ep_fd, buf, block_size, 0)
    io_submit(ctx, 1, &iocb)   ──→ 等待 host 下发数据
    │
    ▼
（异步）host 下发一个 block，某 iocb 完成
    │
    ▼
io_getevents 收割，得到 (buf, 实际长度)
    │
    ▼
回调上层 rx_complete(channel, buf, actual_len)
    │
    ▼
★ 立即向 RX ring 补挂一个新 iocb ★
    （保持 ring 始终满深度，不让接收窗口出现空档）
```

**RX 的第一原则：ring 永远保持满深度。** 每收割一个完成事件，立刻补挂一个新 iocb。如果补挂不及时，RX ring 出现空闲窗口，host 下发的数据无处落脚，host controller 回 NAK，发送方被硬阻塞——这正是流控层要极力避免的不可控阻塞。传输层通过"收割即补挂"把这个窗口压到最小。

**buffer 从哪来**：RX buffer 同样由上层管理（聚合层 RX 双缓冲）。补挂 iocb 时，传输层向上层要一个空闲 buffer；若上层暂时没有空闲 buffer（拆帧慢），则该 iocb 暂缓补挂——这时才轮到上层双缓冲和流控 credit 发挥作用。

---

## 六、事件收割：统一的 io_getevents 循环

所有通道、所有方向的完成事件，由一个专用线程统一收割，避免多线程争抢 `io_context`。

```
event_loop（专用线程）:
    while running:
        n = io_getevents(ctx, min=1, max=BATCH, events[], timeout)
        for i in 0..n:
            iocb  = events[i].obj
            res   = events[i].res      # 实际传输字节数，或负错误码
            ch    = iocb->channel      # 反查属于哪条通道/方向

            if res < 0:
                → 错误处理（见第七节）
            elif iocb 是 TX:
                → tx_complete(ch, iocb->buf)；iocb 归还 ring
            else:  # RX
                → rx_complete(ch, iocb->buf, res)
                → 立即补挂新 RX iocb
```

要点：

- **批量收割**：`io_getevents` 一次可收割多个事件（`max=BATCH`），减少系统调用次数。
- **单线程收割 + 回调分发**：收割线程只做分发，重活（拆帧、拷贝）交回上层线程或线程池，避免阻塞收割循环拖慢整个流水线。
- **timeout 配合**：`io_getevents` 带超时，让收割线程能周期性处理定时任务（如 TX flush 的 timeout 触发信号）或响应退出请求。

---

## 七、异常处理

传输层直面硬件，必须处理这些 endpoint 层事件：

| 事件 | 现象 | 处理 |
|---|---|---|
| USB 断连 / 拔出 | endpoint fd 上 I/O 返回 `-ESHUTDOWN` / `-ECONNRESET` | 标记通道失效，取消所有在途 iocb，通知上层重连 |
| endpoint 未使能 | host 尚未 set_configuration，读写返回 `-ENODEV` | 等待 functionfs `ep0` 上报 `FUNCTIONFS_ENABLE` 事件后再启动收发 |
| 传输被 halt/stall | 某 iocb 返回错误 | 清 halt（`FUNCTIONFS_CLEARHALT`），重挂该 iocb |
| io_submit 返回 EAGAIN | AIO 上下文事件槽满 | 退避重试，或据此判定 ring 已满向上层背压 |
| 短包 (short packet) | RX `res < block_size` | 正常情况——把实际长度交给上层，由上层（聚合层）按 TLV 判断边界 |

**ep0 控制事件循环**：除了数据 endpoint，传输层还需监听 `ep0` 的 functionfs 事件（`BIND` / `UNBIND` / `ENABLE` / `DISABLE` / `SETUP`），驱动整个 function 的生命周期。数据通道的启停必须跟随 `ENABLE`/`DISABLE`，否则会对未就绪的 endpoint 提交 I/O 而报错。

---

## 八、初始化流程

```
1. 挂载 functionfs：mount -t functionfs stick /dev/ffs
2. 打开 ep0，写入 descriptors + strings
      → 定义两个 bulk endpoint：ep1(4KB msg)、ep2(1MB pkt)
3. 等待 ep0 上报 FUNCTIONFS_BIND / FUNCTIONFS_ENABLE
4. 打开 ep1、ep2 endpoint fd
5. io_setup(MAX_EVENTS, &ctx)  创建共享 AIO 上下文
6. 为每条通道分配 TX/RX iocb ring 及关联 buffer 池
7. RX ring 全部预挂起 iocb（进入接收就绪）
8. 启动 event_loop 收割线程
9. 向上层（聚合层）暴露 tx_submit / rx_complete 回调接口
```

---

## 九、与上层的接口契约（小结）

传输层向聚合层暴露的最小接口：

```
tx_submit(channel, buf, len)      → 异步提交发送，ring 满返回 EAGAIN
tx_complete(channel, buf)         ← 发送完成回调，buf 可复用
rx_complete(channel, buf, len)    ← 收到一个 block 回调（len 为实际长度）
rx_provide_buffer(channel)        → 上层提供空闲 RX buffer 供补挂 iocb
```

**职责边界一句话**：

> 传输层只保证"定长 block 高并发、可靠地上下总线，ring 尽量不空档"；至于 block 里怎么拼包/拆包（聚合层）、发多快才不撑爆对端（流控层），传输层一概不管——**它是一根又粗又直的管子，粗靠深队列，直靠不掺业务逻辑。**