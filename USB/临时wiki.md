# AX USB XFER — Device 端驱动对外接口说明

  

> 面向业务层调用者。本文描述 `ax_usb_xfer_device.ko` 暴露给上层的全部 EXPORT 接口、参数语义与调用流程。

>

> - 头文件：`include/ax_usb_xfer_device.h`（接口）、`include/ax_usb_xfer.h`（source/dest 结构）、`include/ax_usb_xfer_proto.h`（常量）。

> - 所有符号为 `EXPORT_SYMBOL_GPL`，调用模块必须声明 GPL 兼容 license。

> - Device 侧为**单实例**：接口不带 handle，内部按当前在线 gadget 自动路由。

  

---

  

## 1. 接口总览

  

Device 端对业务层暴露 **7 个函数** + **1 个回调结构体**，按用途分四组：

  

| 分组 | 接口 | 作用 |

|---|---|---|

| **注册/注销** | `ax_usb_xfer_device_register_client` | 注册回调（收 frame + 链路状态），全局单 client |

| | `ax_usb_xfer_device_unregister_client` | 注销回调，返回后保证回调不再触发 |

| **回调** | `struct ax_usb_xfer_device_client_ops` | 业务层实现：`frame_received` / `link_changed` |

| **大块数据（异步 SG）** | `ax_usb_xfer_device_submit` | **D2H 发送**：Device→Host 提交一笔 source SG |

| | `ax_usb_xfer_device_recv` | **H2D 接收**：为 Host→Device 预投一块 dest SG |

| **小消息（同步拷贝）** | `ax_usb_xfer_device_send_frame` | 发 CTRL/DBG/copy DATA 帧，返回即完成拷贝 |

| **状态查询** | `ax_usb_xfer_device_is_online` | 查询当前链路是否在线 |

  

### 1.1 两种数据语义（先理解再用）

  

| | frame 通路 | xfer 通路 |

|---|---|---|

| 接口 | `send_frame` / `frame_received` 回调 | `submit` / `recv` |

| 语义 | **同步拷贝** | **异步零拷贝借用** |

| buffer 所有权 | 函数返回后调用方可立即释放 | driver 借用 SG 直到 `complete` 回调 |

| 适用 | 小消息（CTRL/DBG）、控制协商、小 DATA | 大块 SG 传输（可达 MiB 级） |

| wire 是否带 header | 是（32B frame header） | 否（纯 payload 字节流） |

  

> **关键**：`submit`/`recv` 提交成功后，在 `complete` 回调触发前，**不得释放或改写 SG**；`send_frame` 则拷贝完即返回，无此约束。两者不可混用。

  

### 1.2 共享数据结构

  

```c

/* include/ax_usb_xfer.h —— source（D2H 发送）与 dest（H2D 接收）同构 */

struct ax_usb_xfer_source {          /* submit 用 */

    struct scatterlist *sg;          /* 业务数据 SG，调用方提供并持有 */

    u32                 nents;        /* SG 段数（上限 4096） */

    u64                 total_len;    /* 本笔总字节数（与对端协商一致） */

    void (*release)(void *priv);                        /* 释放回调 */

    void (*complete)(void *priv, int status, u64 actual); /* 完成回调 */

    void  *priv;                      /* 透传给回调的业务上下文 */

};

  

struct ax_usb_xfer_dest { /* recv 用；字段与 source 完全一致 */ };

```

  

---

  

## 2. 接口详解

  

### 2.1 `ax_usb_xfer_device_register_client`

  

```c

struct ax_usb_xfer_device_client_ops {

    int  (*frame_received)(void *priv, u32 type, u32 port,

                           const void *payload, u32 payload_len);

    void (*link_changed)(void *priv, bool online);

};

  

int ax_usb_xfer_device_register_client(

        const struct ax_usb_xfer_device_client_ops *ops, void *priv);

```

  

**作用**：注册业务层唯一 client，接收 frame 通路数据与链路状态变化。

  

**参数**：

- `ops`：回调表，两个回调**都不能为空**（否则 `-EINVAL`）。

- `priv`：业务上下文，原样透传给两个回调的第一个参数。

  

**回调语义**：

  

| 回调 | 触发时机 | 注意 |

|---|---|---|

| `frame_received(priv, type, port, payload, len)` | 收到 **frame 通路**完整帧（CTRL/DBG/copy DATA） | `payload` 是**借用指针，仅回调期间有效**，需留用必须自行拷贝。**xfer 大块数据不走此回调** |

| `link_changed(priv, online)` | 链路上线 / 下线 | `online=false` 后所有在途 xfer 会被终止并回调 `complete(-ESHUTDOWN)` |

  

**返回**：`0` 成功；`-EINVAL` 回调缺失；`-EBUSY` 已有 client（单 client 模型）；`-ENOMEM`。

  

---

  

### 2.2 `ax_usb_xfer_device_unregister_client`

  

```c

void ax_usb_xfer_device_unregister_client(

        const struct ax_usb_xfer_device_client_ops *ops, void *priv);

```

  

**作用**：注销 client。`ops` 与 `priv` 须与注册时一致才生效。

  

**保证**：内部 `synchronize_srcu`，函数返回后 `frame_received` / `link_changed` 保证不再被调用，可安全释放 `priv` 指向的资源。

  

---

  

### 2.3 `ax_usb_xfer_device_recv` —— H2D 接收（重点）

  

```c

int ax_usb_xfer_device_recv(u32 port, u32 transfer_id,

                            const struct ax_usb_xfer_dest *dest);

```

  

**作用**：Device 作为**接收端**，为一笔 H2D 传输**预投接收窗口**，等待 Host 发来的 payload 落入 `dest->sg`。

  

**参数**：

- `port`：业务路由端口（见 §4 概念说明），须避开保留值。

- `transfer_id`：本笔传输标识，非零，且与 Host 侧协商一致（见 §4）。

- `dest`：接收目标。`sg`/`nents`/`total_len`/`release` 必填；`total_len` 为本笔总长。

  

**返回码**：

  

| 返回 | 条件 |

|---|---|

| `0` | 预投成功，后续经 `dest->complete` 通知结果 |

| `-EINVAL` | 参数为空 / `transfer_id == 0` / `port` 为保留值 |

| `-EMSGSIZE` | `dest->nents > 4096` |

| `-ENOTCONN` | 链路不在线 |

| `-EOPNOTSUPP` | **对端未声明 CAP_H2D**（Host 侧尚未就绪支持 H2D） |

| `-ENOSPC` | 在途 xfer 已达上限（8 笔） |

| `-EEXIST` | 该 `transfer_id` 在本端 TX/RX 任一表已在册 |

| `-EAGAIN` | 提交过程中链路重连（generation 变化），重试即可 |

| `-ENOMEM` | 分配失败 |

  

**buffer 生命周期**：返回 `0` 后 driver 借用 `dest->sg`，直到回调（顺序固定）：

  

```text

release(priv)                    ← 先：可在此 unpin / unmap SG

complete(priv, status, actual)   ← 后：报告结果与实收字节

```

  

---

  

### 2.4 `ax_usb_xfer_device_submit` —— D2H 发送

  

```c

int ax_usb_xfer_device_submit(u32 port, u32 transfer_id,

                              const struct ax_usb_xfer_source *source);

```

  

**作用**：Device 作为**发送端**，向 Host 提交一笔 D2H 大块传输。长度取自 `source->total_len`，driver 内部自动选择单包 / 大包 rolling-window 路径，业务层无需区分。

  

**参数**：同 `recv`，方向相反（`source` 提供待发数据 SG）。

  

**返回码**：`0` 成功；`-EINVAL`（参数 / `transfer_id==0` / 保留 port）；`-ENOTCONN`（不在线）；以及内部 `-ENOSPC` / `-EEXIST` / `-ENOMEM`。

> D2H 为既有能力（两端已声明 BIDIR），此接口**不检查 CAP**。

  

**buffer 生命周期**：与 `recv` 一致（`release` → `complete`）。提交失败（返回非 0）时 source 所有权**不转移**，回调不触发，调用方自行释放。

  

---

  

### 2.5 完成回调 `complete(priv, status, actual)` 语义

  

`submit` / `recv` 两者共用同一套完成语义：

  

| status | 含义 | actual |

|---|---|---|

| `0` | 成功，`actual == total_len` | 实传字节数 |

| `-ECANCELED` | 被 `cancel` 主动取消 | 已传字节 |

| `-ECONNABORTED` | 收到对端 ABORT | 已传字节 |

| `-ETIMEDOUT` | 5 秒超时 | 已传字节 |

| `-EPROTO` | 短读 / 协议错（H2D 接收侧） | 已传字节 |

| `-ESHUTDOWN` | 链路断开 | 已传字节 |

  

---

  

### 2.6 `ax_usb_xfer_device_send_frame` —— 小消息同步发送

  

```c

int ax_usb_xfer_device_send_frame(u32 type, u32 port, const void *buf, u32 len);

```

  

**作用**：发送一帧 CTRL / DBG / copy DATA。**返回即完成拷贝**，`buf` 可立即释放 —— 这是它与 `submit` 的根本区别。

  

**参数**：

- `type`：`AX_USB_XFER_PIPE_TYPE_CTRL(0)` / `_DATA(1)` / `_DBG(2)`。

- `port`：业务端口（须避开 `DYNAMIC_PORT`）。

- `buf` / `len`：待发数据。长度上限按 type：CTRL/DBG 各 1024 字节，DATA 约 2 MiB。

  

**返回**：`0` 成功；`-EINVAL`（参数非法）；`-EMSGSIZE`（超上限）；`-ENOSPC`（发送池满，不写半帧）；`-ENOTCONN`（不在线）。

  

> **业务层的 OFFER/ACK 协商消息，就用本接口发**（`type=CTRL` + 业务 port），对端从 `frame_received` 收。

  

---

  

### 2.7 `ax_usb_xfer_device_is_online`

  

```c

bool ax_usb_xfer_device_is_online(void);

```

  

**作用**：查询当前 gadget 链路是否在线。轻量，可随时调用。返回 `true` 表示已枚举且可收发。

  

---

  

## 3. 契约（driver 不替业务层兜底的部分）

  

1. **`transfer_id` 由业务层维护**：非零、本端 TX/RX 两表内唯一、**H2D 两端必须一致**。driver 校验非零与 dedup，但**无法校验两端一致** —— 这是业务层协商责任。

2. **同方向单笔串行**：xfer 数据面是纯字节流，同一方向同一时刻只能有一笔在传，否则字节交错损坏。多笔请排队，勿并发提交同方向。

3. **异步 SG 在 `complete` 前保持稳定**：不释放、不改写、不解除 pin/map。

4. **H2D 时序**：须 Device 先 `recv` 预投，Host 再发数据 —— 这是 ACK 握手的意义，勿让 Host 抢跑。

  

---

  

## 4. 核心概念：port 与 transfer_id

  

二者正交，不可互相替代：

  

```text

port         = "这是给谁的"   —— 业务路由地址（哪个业务流）

transfer_id  = "这是哪一笔"   —— 传输实例身份（该业务流的哪一次搬运）

```

  

- **同一个 port 上先后有多笔传输**，各笔 `transfer_id` 不同：

  

  ```text

  port=5, transfer_id=100   ← 第一笔

  port=5, transfer_id=101   ← 第二笔

  ```

  

- **数据面不读这两个值**：xfer endpoint 上只有纯 payload，收敛靠 `endpoint + total_len`。`port`/`transfer_id` 只存在于接口参数与 SERVICE ABORT 控制消息中，正常收发不上总线。

- **保留 port**（业务须避开）：`AX_USB_XFER_DYNAMIC_PORT = 0xffffffff`、`AX_USB_XFER_SERVICE_PORT = 0xfffffffe`。

  

---

  

## 5. H2D + D2H 交互调用图

  

### 5.1 H2D（Host → Device）：业务层协商 + driver 搬运

  

```text

   Host 业务层            Host driver         │  Device driver         Device 业务层

                                              │

① 生成 transfer_id                            │

② send_frame(CTRL, port,  ───► xfer_out ──────┼──► frame_received ───►  ③ 收 OFFER

   OFFER{id,port,total_len})                  │      (CTRL)              备好 dest SG

                                              │                         ④ ax_usb_xfer_device_recv

                                              │                    ◄───    (port, id, dest)

                                              │                            预投 xfer_out 接收窗口

                                              │                         ⑤ send_frame(CTRL, port, ACK)

⑦ 收 ACK  ◄─────────────────  frame_received ◄┼──── xfer_out ◄──────────

   ax_usb_xfer_host_submit                    │

   (host, id, port, src)                      │

⑧ ══════ payload SG ═══════►  xfer_out ═══════┼═══► 预投的 OUT request 收数据

                                              │      actual += req->actual

                                              │      actual == total_len ?

                                              │                         ⑨ dest->complete(0, actual)

⑩ src->complete(0, actual) ◄─                 │                            release(priv)

```

  

**要点**：

- ①②⑤ 走 **frame 通路**（`send_frame` / `frame_received`），协商 `transfer_id`/`port`/`total_len`。

- ④ Device 先预投（`recv`），⑤ ACK 通知 Host 就绪，⑦ Host 才 `submit` —— **握手消除时序竞态**，避免 OUT 数据到达时无人接收导致 NAK。

- ⑧ 走 **xfer 通路**（`xfer_out` endpoint），wire 上纯 payload，靠 `actual == total_len` 收敛。

  

### 5.2 D2H（Device → Host）：方向对称

  

```text

   Device 业务层         Device driver        │  Host driver          Host 业务层

                                              │

① 生成 transfer_id                            │

② send_frame(CTRL, port, ───► service ────────┼──► frame_received ───► ③ 收 OFFER

   OFFER{id,port,total_len})                  │                        备好 dest SG

                                              │                        ④ ax_usb_xfer_host_recv

                                              │                   ◄───    (host, id, port, dest)

                                              │                           预投 xfer_in 接收窗口

⑦ 收 ACK ◄──── frame_received ◄───────────────┼─── service ◄────────── ⑤ send_frame(CTRL, port, ACK)

   ax_usb_xfer_device_submit                  │

   (port, id, source)                         │

⑧ ═══ payload SG ═══► xfer_in ════════════════┼═══► 预投的 IN URB 收数据

                                              │      actual == total_len ?

⑩ source->complete(0) ◄─                      │                        ⑨ dest->complete(0, actual)

```

  

**对称性**：D2H 与 H2D 完全镜像 —— 发送端 `submit`（提供 source），接收端 `recv`（提供 dest），协商与握手流程一致，仅方向与 endpoint（`xfer_in` vs `xfer_out`）不同。

  

### 5.3 异常收敛（任一方向通用）

  

```text

某端 timeout / IO error / cancel

   │

   ├─► 本地：终止本端 xfer 对象、kill/dequeue 在途请求

   │

   └─► service endpoint 发 ABORT{transfer_id, status}

          │

          ▼

       对端 frame_received → 按 transfer_id 在 TX/RX 表定位那一笔

          │

          ├─► 终止对应 xfer 对象

          ├─► dequeue / kill 其在途请求

          └─► complete(status, actual) + release(priv)

```

  

> ABORT 靠 `transfer_id` 指认"清理哪一笔"——这是它在异常路径中不可替代的作用。

  

---

  

## 6. 调用速查

  

```text

初始化:   register_client(ops, priv);  等 link_changed(online=true)

  

H2D 收:   [OFFER 到达 → frame_received] 备 dest SG

           ax_usb_xfer_device_recv(port, id, dest);   // 拿 0

           ax_usb_xfer_device_send_frame(CTRL, port, ACK);

           ... 等 dest->complete(status, actual)

  

D2H 发:   [OFFER 已发, ACK 到达]

           ax_usb_xfer_device_submit(port, id, source);

           ... 等 source->complete(status, actual)

  

小消息:   ax_usb_xfer_device_send_frame(CTRL/DBG, port, buf, len);  // 同步

  

下线:     unregister_client(ops, priv);   // 返回后回调保证停止

```