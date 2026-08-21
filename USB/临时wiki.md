# AX USB XFER — 驱动对外接口说明

  

> 面向业务层：本文介绍 USB XFER 驱动 Host / Device 两侧对外提供的接口，便于业务层理解与调用。

>

> - 接口声明：`include/ax_usb_xfer_host.h`、`include/ax_usb_xfer_device.h`

> - 数据结构：`include/ax_usb_xfer.h`（`ax_usb_xfer_source` / `ax_usb_xfer_dest`）

> - 常量定义：`include/ax_usb_xfer_proto.h`（port 保留值、pipe type、能力位等）

> - 驱动源码：`usb/host/ax_usb_xfer_host.c`、`usb/device/ax_usb_xfer_device.c`

  

## 目录

  

- 第 1 节：公共概念（两侧通用，务必先读）

- 第 2 节：Device 端接口

- 第 3 节：Host 端接口

- 第 4 节：交互调用图（H2D / D2H / 异常）

- 第 5 节：调用速查

- 第 6 节：完整调用示例（worked example，可照抄的 C 代码）

  

---

  

## 1. 公共概念

  

### 1.1 Host 与 Device 的角色差异

  

- **Device 侧为单实例**：接口不带句柄，驱动内部按当前在线 gadget 自动路由。

- **Host 侧为多实例**：每个已枚举的 Device 对应一个 `struct ax_usb_xfer_host *` 句柄；所有收发接口需传入该句柄。句柄经 `host_added` 回调下发、`host_removed` 回调失效。

- **方向命名相对本端**：`submit` 恒为“本端发送”，`recv` 恒为“本端接收”。因此：

  - H2D（Host→Device）= Host `submit` + Device `recv`

  - D2H（Device→Host）= Device `submit` + Host `recv`

  

### 1.2 两种数据语义

  

驱动提供两条独立通路，语义完全不同，不可混用。

  

**frame 通路**（`send_frame` / `frame_received`）

  

- 语义：同步拷贝

- buffer 所有权：函数返回后即可释放

- 适用：小消息（CTRL/DBG）、控制协商、小 DATA

- 传输边界：由 32 字节帧头界定

  

**xfer 通路**（`submit` / `recv`）

  

- 语义：异步零拷贝借用

- buffer 所有权：驱动借用 SG 直到 `complete` 回调

- 适用：大块 SG 传输（可达 MiB 级）

- 传输边界：靠预先协商的 `total_len` 收敛

  

关键约束：`submit` / `recv` 提交成功后，在 `complete` 回调触发前不得释放或改写 SG；`send_frame` 拷贝完即返回，无此约束。

  

### 1.3 数据结构

  

```c

/* include/ax_usb_xfer.h

 * source（发送）与 dest（接收）字段完全一致 */

struct ax_usb_xfer_source {

        struct scatterlist *sg;          /* 业务数据 SG，调用方提供并持有 */

        u32                 nents;        /* SG 段数，上限 4096 */

        u64                 total_len;    /* 本笔总字节数，与对端协商一致 */

        void (*release)(void *priv);

        void (*complete)(void *priv, int status, u64 actual);

        void  *priv;                      /* 透传给回调的业务上下文 */

};

  

struct ax_usb_xfer_dest {

        /* 字段与 ax_usb_xfer_source 相同 */

};

```

  

### 1.4 port 与 transfer_id

  

二者正交，不可互相替代：

  

- `port` = “这是给谁的”，业务路由地址（哪个业务流）。

- `transfer_id` = “这是哪一笔”，传输实例身份（该业务流的哪一次搬运）。

  

同一个 port 上先后有多笔传输，各笔 `transfer_id` 不同：

  

```text

port=5, transfer_id=100    第一笔

port=5, transfer_id=101    第二笔

```

  

数据面不读这两个值：xfer endpoint 上只有纯 payload，收敛靠 endpoint + `total_len`。`port` / `transfer_id` 只存在于接口参数与异常 ABORT 控制消息中，正常收发不上总线。

  

保留 port（业务须避开）：

  

- `AX_USB_XFER_DYNAMIC_PORT = 0xffffffff`

- `AX_USB_XFER_SERVICE_PORT = 0xfffffffe`

  

#### transfer_id 由 Host 集中分配

  

驱动只把 `transfer_id` 当入参：谁分配、怎么协商，驱动不参与（它只用 id 建表、去重、ABORT 定位）。业务层约定 **Host 作为唯一分配者**，两个方向共用同一 id 空间，天然保证唯一、且不会 H2D/D2H 撞号：

  

- **H2D（Host 发）**：Host 自分配自用 —— 分配 id → OFFER 直接带 id → Device `recv(id)` 预投 → ACK → Host `submit(id)`。

- **D2H（Device 发）**：Device 先请求 —— Device 发 OFFER（带业务侧临时 `local_tag` + port + total_len，不带 id）→ Host 分配 id → Host 先 `recv(id)` 预投 → ACK 回带 `(local_tag, id)` → Device 用 `local_tag` 认领 id 后 `submit(id)`。

  

要点：① OFFER 必须带 `port` 和 `total_len`（接收端据此备 dest buffer）；② 接收端 `recv` 预投必须在 ACK 之前，ACK 语义即“我已就绪，用这个 id”；③ D2H 的 `local_tag` 仅用于 Device 匹配自己哪一个 OFFER 的 ACK，只在控制消息里，不上 xfer wire；④ 分配的 id 应绑当前 link generation，重连后旧 id 作废。

  

### 1.5 完成回调 complete(priv, status, actual)

  

`submit` / `recv` / `cancel` 共用同一套完成语义（两侧一致）：

  

```text

 0             成功，actual == total_len

-ECANCELED     被 cancel 主动取消

-ECONNABORTED  收到对端 ABORT

-ETIMEDOUT     5 秒超时

-EPROTO        短读 / 协议错（接收侧）

-ESHUTDOWN     链路断开

```

  

`actual` 为实际传输字节数：成功时等于 `total_len`，异常时为已传字节。回调顺序固定为先 `complete` 后 `release`：

  

```text

complete(priv, status, actual)   先：报告结果与实传字节，此时 SG 仍有效，可安全读取

release(priv)                    后：释放/unpin/unmap SG 与 priv

```

  

### 1.6 调用契约

  

以下由业务层保证，驱动不替兜底：

  

1. `transfer_id` 由业务层维护、**Host 集中分配**（见 1.4）：非零、本端收发两表内唯一、两端必须一致。驱动校验非零与去重，但无法校验两端一致，一致性由 OFFER/ACK 协议保证。

2. 同方向单笔串行：xfer 数据面是纯字节流，同一方向同一时刻只能有一笔在传，否则字节交错损坏。多笔请排队。

3. 异步 SG 在 `complete` 前保持稳定：不释放、不改写、不解除 pin / map。

4. H2D 时序：须 Device 先 `recv` 预投，Host 再 `submit`，避免 OUT 数据到达时无人接收。

5. Host 句柄有效性：只在 `host_added` 之后、`host_removed` 之前使用 `host`。

  

### 1.7 链路状态：上下线、挂起，通知 + 查询

  

驱动区分两类状态，两侧对称，业务层可“推”（回调）也可“拉”（查询）：

  

- **online / offline**：是否已枚举。上下线经 `link_changed(online)` 回调通知（必选回调）。

- **suspended**：online 但被总线挂起（可 resume 恢复，与下线不同）。变化经 `suspend_changed(suspended)` 回调通知（**可选回调**，NULL 则不回调）。

  

查询接口（推拉并存，不注册回调也能主动查）：

  

```text

is_online()      是否已枚举

is_suspended()   online && 已挂起

is_ready()       online && !suspended —— 业务“能否发起 xfer”看这个

```

  

说明：① `suspend_changed` 可选，业务层不实现也不影响；② `register_client` 不补投挂起态——新注册 client 想知道当前是否挂起，直接查 `is_suspended()`；③ 挂起期间驱动不硬拒 `submit`/`recv`（入口仍只 gate online），业务层按 `is_ready()` 自行判断是否发起。

  

---

  

---

  

## 2. Device 端接口

  

Device 端提供 9 个函数 + 1 个回调结构体：

  

**注册 / 注销**

  

- `ax_usb_xfer_device_register_client` — 注册回调（收 frame + 链路状态），全局单 client

- `ax_usb_xfer_device_unregister_client` — 注销回调，返回后保证回调不再触发

  

**回调**

  

- `struct ax_usb_xfer_device_client_ops` — 业务层实现：`frame_received` / `link_changed` / `suspend_changed`（可选）

  

**大块数据（异步 SG）**

  

- `ax_usb_xfer_device_submit` — D2H 发送：Device 向 Host 提交一笔 source SG

- `ax_usb_xfer_device_recv` — H2D 接收：为 Host 来的数据预投一块 dest SG

  

**小消息（同步拷贝）**

  

- `ax_usb_xfer_device_send_frame` — 发 CTRL/DBG/copy DATA 帧，返回即完成拷贝

  

**状态查询**

  

- `ax_usb_xfer_device_is_online` — 查询当前链路是否在线

- `ax_usb_xfer_device_is_suspended` — 查询是否总线挂起（online 且已挂起）

- `ax_usb_xfer_device_is_ready` — 查询是否可发起 xfer（online && !suspended）

  

### 2.1 ax_usb_xfer_device_register_client

  

```c

struct ax_usb_xfer_device_client_ops {

        int  (*frame_received)(void *priv, u32 type, u32 port,

                               const void *payload, u32 payload_len);

        void (*link_changed)(void *priv, bool online);

        /* 可选：总线 suspend/resume 时通知，NULL 则不回调 */

        void (*suspend_changed)(void *priv, bool suspended);

};

  

int ax_usb_xfer_device_register_client(

        const struct ax_usb_xfer_device_client_ops *ops, void *priv);

```

  

作用：注册业务层唯一 client，接收 frame 通路数据与链路状态变化。

  

参数：

  

- `ops`：回调表，两个回调都不能为空，否则返回 `-EINVAL`。

- `priv`：业务上下文，原样透传给回调第一个参数。

  

回调语义：

  

- `frame_received(priv, type, port, payload, len)`：收到 frame 通路完整帧时触发。`payload` 是借用指针，仅回调期间有效，需留用必须自行拷贝。注意 xfer 大块数据不走此回调。

- `link_changed(priv, online)`：链路上线 / 下线通知。`online=false` 后所有在途 xfer 会被终止并回调 `complete(-ESHUTDOWN)`。

  

返回：`0` 成功；`-EINVAL` 回调缺失；`-EBUSY` 已有 client（单 client 模型）；`-ENOMEM`。

  

### 2.2 ax_usb_xfer_device_unregister_client

  

```c

void ax_usb_xfer_device_unregister_client(

        const struct ax_usb_xfer_device_client_ops *ops, void *priv);

```

  

作用：注销 client。`ops` 与 `priv` 须与注册时一致才生效。

  

保证：函数返回后 `frame_received` / `link_changed` 保证不再被调用，可安全释放 `priv` 指向的资源。

  

### 2.3 ax_usb_xfer_device_recv（H2D 接收）

  

```c

int ax_usb_xfer_device_recv(u32 transfer_id, u32 port,

                            const struct ax_usb_xfer_dest *dest);

```

  

作用：Device 作为接收端，为一笔 H2D 传输预投接收窗口，等待 Host 发来的 payload 落入 `dest->sg`。

  

参数：

  

- `transfer_id`：本笔传输标识，非零，且与 Host 侧协商一致。

- `port`：业务路由端口（见 1.4），须避开保留值。

- `dest`：接收目标，`sg` / `nents` / `total_len` / `release` 必填。

  

返回码：

  

```text

 0           预投成功，后续经 dest->complete 通知结果

-EINVAL      参数为空 / transfer_id == 0 / port 为保留值

-EMSGSIZE    dest->nents > 4096

-ENOTCONN    链路不在线

-EOPNOTSUPP  对端未声明 CAP_H2D（Host 侧尚未就绪支持 H2D）

-ENOSPC      在途 xfer 已达上限（8 笔）

-EEXIST      该 transfer_id 在本端 TX/RX 任一表已在册

-EAGAIN      提交过程中链路重连（generation 变化），可重试

-ENOMEM      分配失败

```

  

buffer 生命周期：见 1.5（先 `release` 后 `complete`）。

  

### 2.4 ax_usb_xfer_device_submit（D2H 发送）

  

```c

int ax_usb_xfer_device_submit(u32 transfer_id, u32 port,

                              const struct ax_usb_xfer_source *source);

```

  

作用：Device 作为发送端，向 Host 提交一笔 D2H 大块传输。长度取自 `source->total_len`，驱动内部自动选择单包 / 大包 rolling-window 路径，业务层无需区分。

  

参数：与 `recv` 相同，方向相反（`source` 提供待发数据 SG）。

  

返回码：

  

```text

 0           成功，后续经 source->complete 通知结果

-EINVAL      参数非法 / transfer_id == 0 / port 为保留值

-ENOTCONN    链路不在线

-ENOSPC      在途 xfer 已达上限

-EEXIST      该 transfer_id 已在册

-ENOMEM      分配失败

```

  

D2H 为既有能力（两端已声明 BIDIR），此接口不检查 CAP。提交失败（返回非 0）时 source 所有权不转移，回调不触发，调用方自行释放。

  

### 2.5 ax_usb_xfer_device_send_frame（小消息同步发送）

  

```c

int ax_usb_xfer_device_send_frame(u32 type, u32 port,

                                  const void *buf, u32 len);

```

  

作用：发送一帧 CTRL / DBG / copy DATA。返回即完成拷贝，`buf` 可立即释放，这是它与 `submit` 的根本区别。

  

参数：

  

- `type`：`AX_USB_XFER_PIPE_TYPE_CTRL(0)` / `_DATA(1)` / `_DBG(2)`。

- `port`：业务端口，须避开 `DYNAMIC_PORT`。

- `buf` / `len`：待发数据。长度上限按 type：CTRL / DBG 各 1024 字节，DATA 约 2 MiB。

  

返回：`0` 成功；`-EINVAL`（参数非法）；`-EMSGSIZE`（超上限）；`-ENOSPC`（发送池满，不写半帧）；`-ENOTCONN`（不在线）。

  

业务层的 OFFER / ACK 协商消息即用本接口发送（`type=CTRL` + 业务 port），对端从 `frame_received` 收。

  

### 2.6 ax_usb_xfer_device_is_online / is_suspended / is_ready

  

```c

bool ax_usb_xfer_device_is_online(void);     /* 是否已枚举在线 */

bool ax_usb_xfer_device_is_suspended(void);  /* online && 已总线挂起 */

bool ax_usb_xfer_device_is_ready(void);      /* online && !suspended */

```

  

作用：查询链路状态，轻量，可随时调用。三态语义见 1.7。业务层“能否发起 xfer”查 `is_ready()` 即可，无需自行组合 online 与 suspended。挂起（`is_suspended` 为 true）可经 resume 恢复，与下线（`is_online` 为 false）不同。

  

---

  

## 3. Host 端接口

  

Host 端提供 9 个函数 + 1 个回调结构体。与 Device 端主要差异：接口带 `host` 句柄、多 `host_added` / `host_removed` 两个回调、额外提供 `cancel`、`send_frame` 返回已发送字节数。

  

**注册 / 注销**

  

- `ax_usb_xfer_host_register_client` — 注册回调（收 frame + 链路 + 句柄增删），全局单 client

- `ax_usb_xfer_host_unregister_client` — 注销回调

  

**回调**

  

- `struct ax_usb_xfer_host_client_ops` — 业务层实现：`frame_received` / `link_changed` / `host_added` / `host_removed` / `suspend_changed`（可选）

  

**大块数据（异步 SG）**

  

- `ax_usb_xfer_host_submit` — H2D 发送：Host 向 Device 提交一笔 source SG

- `ax_usb_xfer_host_recv` — D2H 接收：为 Device 来的数据预投一块 dest SG

- `ax_usb_xfer_host_cancel` — 取消一笔已提交的 submit 或 recv

  

**小消息（同步拷贝）**

  

- `ax_usb_xfer_host_send_frame` — 发 CTRL/DBG/copy DATA 帧，返回即完成拷贝

  

**句柄 / 状态查询**

  

- `ax_usb_xfer_host_find_by_id` / `ax_usb_xfer_host_put` / `ax_usb_xfer_host_id` — 按稳定 id 取带引用的 host、释放、取 id

- `ax_usb_xfer_host_is_online` / `is_suspended` / `is_ready` — 三态查询

  

### 3.1 ax_usb_xfer_host_register_client

  

```c

struct ax_usb_xfer_host_client_ops {

        int  (*frame_received)(void *priv, struct ax_usb_xfer_host *host,

                               u32 type, u32 port,

                               const void *payload, u32 payload_len);

        void (*link_changed)(void *priv, struct ax_usb_xfer_host *host,

                             bool online);

        void (*host_added)(void *priv, struct ax_usb_xfer_host *host);

        void (*host_removed)(void *priv, struct ax_usb_xfer_host *host);

        /* 可选：总线 suspend/resume 时通知，NULL 则不回调 */

        void (*suspend_changed)(void *priv, struct ax_usb_xfer_host *host,

                                bool suspended);

};

  

int ax_usb_xfer_host_register_client(

        const struct ax_usb_xfer_host_client_ops *ops, void *priv);

```

  

作用：注册业务层唯一 client，接收 frame 通路数据、链路状态、以及 Device 句柄的增删。

  

参数：

  

- `ops`：回调表，前四个回调（`frame_received`/`link_changed`/`host_added`/`host_removed`）不能为空，否则返回 `-EINVAL`；`suspend_changed` 可选（可为 NULL）。

- `priv`：业务上下文，原样透传给回调第一个参数。

  

回调语义：

  

- `frame_received(priv, host, type, port, payload, len)`：收到 frame 通路完整帧时触发，`host` 标识来自哪个 Device。`payload` 是借用指针，仅回调期间有效。xfer 大块数据不走此回调。

- `link_changed(priv, host, online)`：该 `host` 的链路上线 / 下线通知。

- `host_added(priv, host)`：一个 Device 枚举成功，下发其句柄。业务层须保存此 `host` 用于后续收发。注册时若已有在线 Device，会逐个补投一次。

- `host_removed(priv, host)`：该 Device 断开，句柄即将失效，业务层须停止使用。

- `suspend_changed(priv, host, suspended)`（可选）：该 `host` 总线挂起 / 恢复通知，见 1.7。

  

返回：`0` 成功；`-EINVAL` 回调缺失；`-EBUSY` 已有 client。

  

### 3.2 ax_usb_xfer_host_unregister_client

  

```c

void ax_usb_xfer_host_unregister_client(

        const struct ax_usb_xfer_host_client_ops *ops, void *priv);

```

  

作用：注销 client。`ops` 与 `priv` 须与注册时一致才生效。

  

### 3.3 ax_usb_xfer_host_submit（H2D 发送）

  

```c

int ax_usb_xfer_host_submit(struct ax_usb_xfer_host *host,

                            u32 transfer_id, u32 port,

                            const struct ax_usb_xfer_source *src);

```

  

作用：Host 作为发送端，向指定 Device 提交一笔 H2D 大块传输，数据经 `xfer_out` bulk OUT 发出。长度取自 `src->total_len`。

  

参数：

  

- `host`：目标 Device 句柄（来自 `host_added`）。

- `transfer_id`：本笔传输标识，非零，且与 Device 侧协商一致。

- `port`：业务路由端口，须避开保留值。

- `src`：待发数据，`sg` / `nents` / `total_len` / `complete` / `release` 必填。

  

返回码：

  

```text

 0           成功，后续经 src->complete 通知结果

-EINVAL      参数为空 / transfer_id == 0 / port 为保留值

-ENOTCONN    链路不在线

-EOPNOTSUPP  对端未声明 CAP_H2D（Device 侧无 preposted receiver，

             此时发数据会 NAK 堆积；须等两端 CAP_H2D 就绪）

-EEXIST      该 transfer_id 在本 host 已在册

-ENOMEM      分配失败

```

  

前置条件：H2D 要求 Device 先 `recv` 预投，Host 再 `submit`。调用前须完成 OFFER/ACK 握手（见第 4 节），确认 Device 就绪。buffer 生命周期见 1.5。

  

### 3.4 ax_usb_xfer_host_recv（D2H 接收）

  

```c

int ax_usb_xfer_host_recv(struct ax_usb_xfer_host *host,

                          u32 transfer_id, u32 port,

                          const struct ax_usb_xfer_dest *dest);

```

  

作用：Host 作为接收端，为一笔 D2H 传输预投接收窗口，数据经 `xfer_in` bulk IN 落入 `dest->sg`。

  

参数：与 `submit` 相同，方向相反（`dest` 提供接收目标 SG）。

  

返回码：

  

```text

 0           预投成功，后续经 dest->complete 通知结果

-EINVAL      参数为空 / transfer_id == 0 / port 为保留值

-ENOTCONN    链路不在线

-EOPNOTSUPP  D2H 接收能力未就绪（xfer_rx_enabled 未置位）

-EEXIST      该 transfer_id 在本 host 已在册

-ENOMEM      分配失败

```

  

buffer 生命周期见 1.5。

  

### 3.5 ax_usb_xfer_host_cancel

  

```c

int ax_usb_xfer_host_cancel(struct ax_usb_xfer_host *host, u32 transfer_id);

```

  

作用：主动取消一笔已提交的 xfer。驱动在本 host 的发送表与接收表中按 `transfer_id` 查找，命中即终止：kill 在途 URB、向对端发 ABORT 通知、最终回调 `complete(-ECANCELED, actual)`。

  

返回：`0` 命中并已发起取消；`-ENOENT` 未找到该 `transfer_id`；`-EINVAL` `host` 为空。

  

说明：取消是收敛的发起动作，实际释放仍通过 `complete` 回调异步完成。

  

### 3.6 ax_usb_xfer_host_send_frame（小消息同步发送）

  

```c

ssize_t ax_usb_xfer_host_send_frame(struct ax_usb_xfer_host *host,

                                    u32 type, u32 port,

                                    const void *buf, u32 len);

```

  

作用：向指定 Device 发送一帧 CTRL / DBG / copy DATA。返回即完成拷贝，`buf` 可立即释放。

  

参数：

  

- `host`：目标 Device 句柄。

- `type`：`AX_USB_XFER_PIPE_TYPE_CTRL(0)` / `_DATA(1)` / `_DBG(2)`。

- `port`：业务端口，须避开 `DYNAMIC_PORT`。

- `buf` / `len`：待发数据。长度上限按 type：CTRL / DBG 各 1024 字节，DATA 约 2 MiB。

  

返回：成功返回已发送字节数 `len`（`ssize_t` 正值）；失败返回负错误码（`-EINVAL` / `-EMSGSIZE` / `-ENOSPC` / `-ENOTCONN`）。

  

> 注意：Host 的 `send_frame` 成功返回字节数，Device 的 `send_frame` 成功返回 0，判断成功时须区分。

  

### 3.7 句柄查找与状态查询

  

```c

struct ax_usb_xfer_host *ax_usb_xfer_host_find_by_id(u32 id); /* 返回带引用的 host */

void ax_usb_xfer_host_put(struct ax_usb_xfer_host *host);     /* 释放 find_by_id 的引用 */

u32  ax_usb_xfer_host_id(struct ax_usb_xfer_host *host);      /* (busnum<<16)|devnum */

  

bool ax_usb_xfer_host_is_online(struct ax_usb_xfer_host *host);

bool ax_usb_xfer_host_is_suspended(struct ax_usb_xfer_host *host);  /* online && 挂起 */

bool ax_usb_xfer_host_is_ready(struct ax_usb_xfer_host *host);      /* online && !suspended */

```

  

作用：

  

- **多 device 查找**：`find_by_id` 按稳定 id `(busnum<<16)|devnum` 查在线 host，命中返回带引用的句柄，用完必须 `put`（只能在可睡眠上下文调用）。收到 `host_removed` 后须释放之前 `find_by_id` 得到的全部引用。`id` 取自 `ax_usb_xfer_host_id(host)`，可在 `host_added`/`host_removed` 之间关联同一 host。

- **状态查询**：三态语义见 1.7，带 `host` 句柄（多实例，各 host 独立）。业务“能否发起 xfer”查 `is_ready()`。

  

---

  

## 4. 交互调用图

  

图为函数调用视角：业务层按什么顺序调哪个接口、回调如何串接。OFFER / ACK 均走 frame 通路（`send_frame` CTRL），数据走 xfer 通路。**`transfer_id` 一律由 Host 分配**（见 1.4）：H2D 时 Host 自分配，D2H 时 Host 应 Device 请求而分配、经 ACK 回传。

  

### 4.1 H2D 调用图（Host 发 / Device 收）

  

```text

Host 业务层                          Device 业务层

-----------                          -------------

register_client(ops)                 register_client(ops)

   收 host_added(host)                    |

        |                                 |

   等 link online                    等 link online

        |                                 |

Host 分配 transfer_id                     |

        |                                 |

ax_usb_xfer_host_send_frame -- OFFER ---> frame_received 回调

  (host, CTRL, OFFER{id,port,len})              |

        |                         ax_usb_xfer_device_recv(id, port, dest)

        |                                 |    先预投，再回 ACK

frame_received 回调 <---- ACK ----- ax_usb_xfer_device_send_frame(CTRL, ACK{id})

        |                                 |

ax_usb_xfer_host_submit(host,id,port,src) |

        |                                 |

   [数据经 xfer_out 传输]                  |

        |                                 |

src->complete(status,actual)         dest->complete(status,actual)

src->release(priv)                   dest->release(priv)

```

  

### 4.2 D2H 调用图（Device 发 / Host 收）

  

发送端在 Device（`submit`），接收端在 Host（`recv`），数据走 `xfer_in`。与 H2D 不同：id 仍由 Host 分配，Device 先发 OFFER 请求，Host 分配后经 ACK 回传 id。

  

```text

Device 业务层                        Host 业务层

-------------                        -----------

register_client(ops)                 register_client(ops)

        |                               收 host_added(host)

   等 link online                    等 link online

        |                                 |

生成本地 local_tag                        |

        |                                 |

ax_usb_xfer_device_send_frame - OFFER --> frame_received 回调

  (CTRL, OFFER{local_tag,port,len})             |

        |                            Host 分配 transfer_id

        |                            ax_usb_xfer_host_recv(host, id, port, dest)

        |                                 |    先预投，再回 ACK

frame_received 回调 <- ACK{local_tag,id} - ax_usb_xfer_host_send_frame(host, CTRL, ACK)

        |                                 |

用 local_tag 认领 id                      |

ax_usb_xfer_device_submit(id,port,source) |

        |                                 |

   [数据经 xfer_in 传输]                   |

        |                                 |

source->complete(status,actual)      dest->complete(status,actual)

source->release(priv)                dest->release(priv)

```

  

### 4.3 异常收敛图（任一方向通用）

  

```text

某端 timeout / IO error / cancel

        |

        +--> 本地：终止本端 xfer 对象、kill/dequeue 在途请求

        |

        +--> service endpoint 发 ABORT{transfer_id, status}

                     |

                     v

             对端 frame_received --> 按 transfer_id 在收发两表定位那一笔

                     |

                     +--> 终止对应 xfer 对象

                     +--> kill / dequeue 其在途请求

                     +--> complete(status, actual) + release(priv)

```

  

Host 侧还可通过 `ax_usb_xfer_host_cancel(host, transfer_id)` 主动发起上述收敛。ABORT 靠 `transfer_id` 指认要清理哪一笔，这是它在异常路径中不可替代的作用。

  

---

  

## 5. 调用速查

  

### 5.1 Device 端

  

```text

初始化:   register_client(ops, priv)  然后等 link_changed(online=true)

  

H2D 收:   收到 OFFER（frame_received） -> 备 dest SG

          ax_usb_xfer_device_recv(id, port, dest)          返回 0

          ax_usb_xfer_device_send_frame(CTRL, port, ACK)

          等 dest->complete(status, actual)

  

D2H 发:   生成 local_tag，发 OFFER(local_tag, port, total_len)

          收到 ACK(local_tag, id)，用 local_tag 认领 id

          ax_usb_xfer_device_submit(id, port, source)

          等 source->complete(status, actual)

  

小消息:   ax_usb_xfer_device_send_frame(CTRL/DBG, port, buf, len)   同步

  

下线:     ax_usb_xfer_device_unregister_client(ops, priv)

```

  

### 5.2 Host 端

  

```text

初始化:   register_client(ops, priv)

          收 host_added(host) 保存句柄

          等 link_changed(online=true)

  

H2D 发:   Host 分配 transfer_id

          ax_usb_xfer_host_send_frame(host, CTRL, port, OFFER{id})

          收到 ACK（frame_received）

          ax_usb_xfer_host_submit(host, id, port, src)

          等 src->complete(status, actual)

  

D2H 收:   收到 OFFER(local_tag)（frame_received） -> Host 分配 id -> 备 dest SG

          ax_usb_xfer_host_recv(host, id, port, dest)      返回 0（先预投）

          ax_usb_xfer_host_send_frame(host, CTRL, port, ACK{local_tag,id})

          等 dest->complete(status, actual)

  

取消:     ax_usb_xfer_host_cancel(host, transfer_id)

  

小消息:   ax_usb_xfer_host_send_frame(host, CTRL/DBG, port, buf, len)   同步

  

下线:     ax_usb_xfer_host_unregister_client(ops, priv)

```

  

---

  

## 6. 完整调用示例（worked example）

  

> 本节把第 4 节的时序落到真实函数调用上，给出可照抄的端到端 C 序列。**OFFER / ACK 的消息结构由业务层自定义**，下面仅为示例；驱动只把它当普通 CTRL frame 搬运。示例里 `axdemo_` 前缀均为业务侧代码，`ax_usb_xfer_` 前缀才是驱动导出接口。

  

### 6.1 示例约定

  

**自定义协商消息**（走 frame 通路 CTRL，业务层自行定义）：

  

```c

enum {

        AXDEMO_OFFER = 1,

        AXDEMO_ACK   = 2,

};

  

struct axdemo_offer {

        u32 msg;          /* AXDEMO_OFFER */

        u32 transfer_id;  /* H2D: Host 已分配; D2H: 0（待 Host 分配） */

        u32 local_tag;    /* D2H: Device 本地标识; H2D: 0 */

        u32 port;         /* 业务 port */

        u64 total_len;    /* 本笔字节数 */

};

  

struct axdemo_ack {

        u32 msg;          /* AXDEMO_ACK */

        u32 transfer_id;  /* 最终生效的 id（一律 Host 分配） */

        u32 local_tag;    /* D2H: 原样回带以匹配 OFFER; H2D: 0 */

        u32 port;

};

```

  

**SG 辅助**（业务自备，非驱动接口）：`axdemo_build_sg(total_len, &nents)` 按长度分配页并建好 `scatterlist` 表，`axdemo_free_sg(sg, nents)` 释放。发送方需在 SG 里填好待发数据，接收方给空 SG 等待落地。

  

**上下文约定**：`frame_received` 可能在原子上下文触发。`send_frame` 是同步拷贝，可在回调内直接调用；而 `recv` / `submit` 及其 SG 分配建议调度到工作队列执行。为聚焦调用顺序，下文把提交逻辑写成 `axdemo_arm_*()` 辅助函数，业务层可按需放进 workqueue。

  

**成功判据**：`complete(priv, status, actual)` 中 `status == 0` 即成功（此时 `actual == total_len`）；非 0 为异常，`actual` 是已传字节。回调顺序恒为先 `complete` 后 `release`（见 1.5）。

  

---

  

### 6.2 D2H xfer tx 通路：驱动内部调用链（rolling window）

  

只画 xfer 数据面：从 `ax_usb_xfer_device_submit` 入口深入驱动内部，展示一笔大块 D2H 发送如何被切成 chunk、经 rolling-window 三槽泵送到 DWC3、逐笔完成后补包收敛。此前的 OFFER/ACK 业务协商见 6.4，不在本图。

  

```text

只画 xfer 数据面。rolling window：submit 预分配 window 个 usb_request 作为固定槽

（默认 3），total_len 按 2MiB / 84 段 / mps 动态切成任意多个 chunk，由这几个槽

轮流复用承载——不是把数据切成 3 份。

  

［提交］

业务层 ax_usb_xfer_device_submit(transfer_id, port, source)      // 入口校验

  └─ ux_device_tx_submit(cs, port, id, total_len, source)

       · 校验对端 CAP_D2H

       · kzalloc xfer；window = clamp(ux_large_sw_window, 1, 3)

       · 预分配 window 个 req：slots[i].req = usb_ep_alloc_request()

       · 挂 large_xfers 链表、查重、ready = true

       · schedule_work(&xfer->pump_work)                     ──▶ ［泵送］

  

［泵送］ tx workqueue

ux_device_tx_pump_work           // 轮询 window 个槽，填满所有空闲槽（!active）

  for slot in slots[0 .. window-1]:

    if slot->active || submitted == total_len:  skip

    offset = xfer->submitted

    ├─ ux_large_chunk_span(xfer, offset, mps)           // 算这一笔 chunk 长度

    │    · 累计到 2 MiB 上限 或 84 段上限 即截断

    │    · 非末段 ALIGN_DOWN(span, mps)；末段按 remaining 精确 clamp

    │    ↳ return payload_len

    xfer->submitted += payload_len                      // 推进偏移（预扣）

    ├─ ux_device_tx_slot_build(slot, offset, payload_len)    // 建该 chunk 的 SG 视图

    │    · sg_alloc_table

    │    · 从 source SG [offset, offset+len) 切段填入 slot->sgt；slot->active = true

    ├─ 填 req->sg / num_sgs / length / context / complete

    │  slot->queued = true; xfer->outstanding++

    └─ usb_ep_queue(ep, slot->req, GFP_ATOMIC)          // 提交到 DWC3 底层

  ▶ 一轮最多提交 window 笔 → 多槽并发在途

       ┆ DWC3 逐笔把 payload 搬到 xfer_in（Host recv 侧对称接收）

  

［完成 / 补包］

ux_device_tx_complete(ep, req)         // 硬件回调，每笔完成一次

    xfer->outstanding--

    正常：xfer->actual += payload_len；pump = true

    出错/短传：terminal = true；status = err；send_abort

    ├─ ux_device_tx_slot_reset(slot)   // sg_free_table 释放本 chunk；槽置空闲（req 复用）

    └─ if pump: schedule_work(&xfer->pump_work)          ──▶ ［泵送］（空槽补下一 chunk）

       rolling window 滚动：谁完成谁补下一块，直到 submitted == total_len

  

［收尾］ submitted == total_len 且 outstanding == 0（或 timeout / IO / 断链）

  置 terminal，status = 0/err → schedule_work(&transport.reap_work)

  └─ ux_transport_reap_work → list_del(xfer->node) 摘链

       └─ ux_device_tx_free(xfer)

            · cancel timeout/pump work；usb_ep_free_request 各槽 req

            · source.complete(priv, status, actual)   ← 先（SG 仍有效，可读取）

            · source.release(priv)                    ← 后（unpin / free）

            · kfree(xfer)

```

  

要点（校正常见误解，均以 `device/ax_usb_xfer_device.c` 源码为准）：

  

1. **三槽是滚动窗口，不是把数据切成 3 份**：`submit` 预分配 `window`（默认 3）个 `usb_request`；`total_len` 按 2MiB / 84 段 / mps 切成**任意多个 chunk**，三槽轮流复用。例如 10 MiB ≈ 切 5 笔 2 MiB chunk，由 3 槽分批消化，绝非只发 6 MiB。

2. **pump 轮询槽，chunk_span 算单笔**：遍历 `window` 个槽的是 `ux_device_tx_pump_work` 的 for 循环；`ux_large_chunk_span` 只对一个 `offset` 算一笔 chunk 的长度。二者别混。

3. **补包由 complete 驱动**：每笔 `complete` 里 `ux_device_tx_slot_reset` 腾出槽（`sg_free_table` + `active=false`，**req 复用**）并 `schedule_work(pump_work)`；pump 重跑发现空槽且 `submitted < total_len`，用 `offset = submitted` 处的下一 chunk 填充。`submitted == total_len` 后所有槽命中 skip，不再补，自然收敛。

4. **chunk 对齐**：非末段 `ALIGN_DOWN(span, mps)` 保证 wire 上中间无 short packet；末段按 `remaining` 精确 clamp，任意长度。故正常每笔 chunk 是 mps 整数倍（SS=1024 / HS=512），接近整 2 MiB。

5. **complete 先、release 后**：收尾统一在 `ux_device_tx_free`，与 1.5 一致。H2D rx 路径完全对称（`ux_device_rx_span` / `ux_device_rx_slot_build` / rx pump，遍历 dest SG、经 xfer_out 预投接收，方向相反）。

  

---

  

### 6.3 H2D：Host 发送，Device 接收

  

调用主线：Host 分配 id → OFFER → Device `recv` 预投 → ACK → Host `submit` → 双端 `complete`。

  

**Host 端（发送方，发起者）**

  

```c

/* 第 1 步：链路就绪后，Host 分配 id 并发 OFFER，请求接收方预投 */

static void host_start_h2d(struct ax_usb_xfer_host *host, u32 port, u64 len)

{

        struct axdemo_offer off = {

                .msg         = AXDEMO_OFFER,

                .transfer_id = axdemo_alloc_id(),   /* Host 集中分配，非零 */

                .local_tag   = 0,

                .port        = port,

                .total_len   = len,

        };

        /* send_frame 返回已发送字节数（ssize_t），< 0 为失败 */

        if (ax_usb_xfer_host_send_frame(host, AX_USB_XFER_PIPE_TYPE_CTRL,

                                        port, &off, sizeof(off)) < 0)

                return;                     /* 发送失败，按业务重试 */

        /* 业务记录 (transfer_id, len)，等 frame_received 收 ACK */

}

  

/* 第 2 步：收到 ACK 后再提交数据（ACK 语义 = 对端已 recv 预投就绪） */

static int host_frame_received(void *priv, struct ax_usb_xfer_host *host,

                               u32 type, u32 port,

                               const void *payload, u32 payload_len)

{

        const struct axdemo_ack *ack = payload;

  

        if (type != AX_USB_XFER_PIPE_TYPE_CTRL ||

            payload_len != sizeof(*ack) || ack->msg != AXDEMO_ACK)

                return -ENOENT;             /* 不是本 client 关心的帧 */

  

        axdemo_arm_submit(host, ack->transfer_id, ack->port);  /* 见第 3 步 */

        return 0;

}

  

/* 第 3 步：提交 source SG（建议在 workqueue 内执行） */

static void host_submit_complete(void *priv, int status, u64 actual)

{

        /* status==0 即成功；本端为发送方，不校验数据 */

}

static void host_submit_release(void *priv) { axdemo_free_sg(...); }

  

static void axdemo_arm_submit(struct ax_usb_xfer_host *host, u32 id, u32 port)

{

        u32 nents;

        u64 len = ...;                      /* 发起时已按 id 记录的本笔长度 */

        struct ax_usb_xfer_source src = {

                .sg        = axdemo_build_sg(len, &nents),  /* 已填好待发数据 */

                .nents     = nents,

                .total_len = len,

                .complete  = host_submit_complete,

                .release   = host_submit_release,

                .priv      = ...,

        };

        int ret = ax_usb_xfer_host_submit(host, id, port, &src);

        if (ret) {                          /* 提交失败：所有权未转移，自行释放 */

                axdemo_free_sg(src.sg, nents);

                return;

        }

        /* 成功：SG 交由驱动借用，直到 host_submit_complete/release */

}

```

  

**Device 端（接收方）**

  

```c

/* 收到 OFFER：先 recv 预投，再回 ACK —— 顺序不可颠倒 */

static int dev_frame_received(void *priv, u32 type, u32 port,

                              const void *payload, u32 payload_len)

{

        const struct axdemo_offer *off = payload;

  

        if (type != AX_USB_XFER_PIPE_TYPE_CTRL ||

            payload_len != sizeof(*off) || off->msg != AXDEMO_OFFER)

                return -ENOENT;

  

        axdemo_arm_recv(off->transfer_id, off->port, off->total_len);

        return 0;

}

  

static void dev_recv_complete(void *priv, int status, u64 actual)

{

        if (status == 0) {

                /* 成功，actual == total_len，此处消费落地的数据 */

        }

}

static void dev_recv_release(void *priv) { axdemo_free_sg(...); }

  

static void axdemo_arm_recv(u32 id, u32 port, u64 len)

{

        u32 nents;

        struct ax_usb_xfer_dest dest = {

                .sg        = axdemo_build_sg(len, &nents),   /* 空 SG 待接收 */

                .nents     = nents,

                .total_len = len,                            /* 必须等于 OFFER */

                .complete  = dev_recv_complete,

                .release   = dev_recv_release,

                .priv      = ...,

        };

        /* 先预投接收窗口 */

        if (ax_usb_xfer_device_recv(id, port, &dest)) {

                axdemo_free_sg(dest.sg, nents);

                return;

        }

        /* 预投成功后，才回 ACK 告知 Host “可以发了” */

        struct axdemo_ack ack = {

                .msg = AXDEMO_ACK, .transfer_id = id, .local_tag = 0, .port = port,

        };

        /* Device 的 send_frame 返回 int，成功为 0 */

        ax_usb_xfer_device_send_frame(AX_USB_XFER_PIPE_TYPE_CTRL,

                                      port, &ack, sizeof(ack));

}

```

  

要点：

  

- **先 recv 后 ACK**：ACK 一旦发出，Host 就会 `submit`，OUT 数据随即到达；若此时 Device 未预投，数据无人接收（契约 1.6 第 4 条）。

- Host `submit` 前必须已收到 ACK；返回 `-EOPNOTSUPP` 表示对端未声明 CAP_H2D，须等两端就绪。

  

---

  

### 6.4 D2H：Device 发送，Host 接收

  

与 H2D 的唯一区别在协商：id 仍由 Host 分配，但由 **Device 先发 OFFER 请求**，Host 分配后经 ACK 回带 id；Device 用 `local_tag` 匹配是自己哪一笔 OFFER 的 ACK。

  

**Device 端（发送方，发起者）**

  

```c

/* 第 1 步：生成 local_tag，发 OFFER（不带 id，请求 Host 分配） */

static void dev_start_d2h(u32 port, u64 len)

{

        struct axdemo_offer off = {

                .msg         = AXDEMO_OFFER,

                .transfer_id = 0,                 /* 待 Host 分配 */

                .local_tag   = axdemo_next_tag(), /* 本地唯一，用于匹配 ACK */

                .port        = port,

                .total_len   = len,

        };

        ax_usb_xfer_device_send_frame(AX_USB_XFER_PIPE_TYPE_CTRL,

                                      port, &off, sizeof(off));

        /* 业务记录 (local_tag, len)，等 ACK */

}

  

/* 第 2 步：收 ACK，用 local_tag 认领 id，再 submit */

static int dev_frame_received(void *priv, u32 type, u32 port,

                              const void *payload, u32 payload_len)

{

        const struct axdemo_ack *ack = payload;

  

        if (type != AX_USB_XFER_PIPE_TYPE_CTRL ||

            payload_len != sizeof(*ack) || ack->msg != AXDEMO_ACK)

                return -ENOENT;

        if (!axdemo_claim_tag(ack->local_tag))    /* 匹配自己发出的 OFFER */

                return -ENOENT;

  

        axdemo_arm_submit(ack->transfer_id, ack->port);  /* 见第 3 步 */

        return 0;

}

  

/* 第 3 步：提交 source（建议在 workqueue 内执行） */

static void dev_submit_complete(void *priv, int status, u64 actual)

{

        /* status==0 即成功；本端为发送方，不校验数据 */

}

static void dev_submit_release(void *priv) { axdemo_free_sg(...); }

  

static void axdemo_arm_submit(u32 id, u32 port)

{

        u32 nents;

        u64 len = ...;                      /* 发起时已按 local_tag 记录的本笔长度 */

        struct ax_usb_xfer_source src = {

                .sg        = axdemo_build_sg(len, &nents),  /* 已填好待发数据 */

                .nents     = nents,

                .total_len = len,

                .complete  = dev_submit_complete,

                .release   = dev_submit_release,

                .priv      = ...,

        };

        if (ax_usb_xfer_device_submit(id, port, &src))

                axdemo_free_sg(src.sg, nents);   /* 失败自行释放 */

}

```

  

**Host 端（接收方）**

  

```c

/* 收 OFFER：分配 id → recv 预投 → 回 ACK（带 local_tag + id） */

static int host_frame_received(void *priv, struct ax_usb_xfer_host *host,

                               u32 type, u32 port,

                               const void *payload, u32 payload_len)

{

        const struct axdemo_offer *off = payload;

  

        if (type != AX_USB_XFER_PIPE_TYPE_CTRL ||

            payload_len != sizeof(*off) || off->msg != AXDEMO_OFFER)

                return -ENOENT;

  

        axdemo_arm_recv(host, off->local_tag, off->port, off->total_len);

        return 0;

}

  

static void host_recv_complete(void *priv, int status, u64 actual)

{

        if (status == 0) {

                /* 成功，actual == total_len，消费数据 */

        }

}

static void host_recv_release(void *priv) { axdemo_free_sg(...); }

  

static void axdemo_arm_recv(struct ax_usb_xfer_host *host, u32 tag,

                            u32 port, u64 len)

{

        u32 nents;

        u32 id = axdemo_alloc_id();         /* Host 集中分配 */

        struct ax_usb_xfer_dest dest = {

                .sg        = axdemo_build_sg(len, &nents),

                .nents     = nents,

                .total_len = len,

                .complete  = host_recv_complete,

                .release   = host_recv_release,

                .priv      = ...,

        };

        /* 先预投 */

        if (ax_usb_xfer_host_recv(host, id, port, &dest)) {

                axdemo_free_sg(dest.sg, nents);

                return;

        }

        /* 预投成功后回 ACK，回带 local_tag 让 Device 认领这一笔 */

        struct axdemo_ack ack = {

                .msg = AXDEMO_ACK, .transfer_id = id,

                .local_tag = tag, .port = port,

        };

        ax_usb_xfer_host_send_frame(host, AX_USB_XFER_PIPE_TYPE_CTRL,

                                    port, &ack, sizeof(ack));

}

```

  

要点：

  

- **id 一律 Host 分配**：D2H 里 Device 只出 `local_tag`，真正的 `transfer_id` 由 Host 收到 OFFER 后分配、经 ACK 回带；Device 靠 `local_tag` 对上号。

- **先 recv 后 ACK**：同 H2D，接收窗口必须先于 ACK 就绪。

- `host` 句柄只在 `host_added` 之后、`host_removed` 之前使用（契约 1.6 第 5 条）。