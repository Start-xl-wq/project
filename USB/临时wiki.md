# 算力棒 USB 驱动设计 - 聚合层

## 一、聚合层的作用

聚合层夹在应用层和传输层之间，核心职责是**做粒度转换**——解决传输层粒度（URB block）和应用层粒度（逻辑包）不匹配的问题。

传输层的工作单元是固定大小的 URB block（1MB），AIO 深队列要发挥吞吐优势，必须让每个 block 尽量填满。在 benchmark 场景下数据自造、永远填得满，所以没问题；但真实业务里应用层送来的是**大小不固定的逻辑包**（4KB 的控制指令 ~ 数百 KB 的数据帧）。如果每个逻辑包直接对应一次 AIO 提交，会出现：

- URB 利用率极低（1MB 的 block 只装了 4KB）
- 频繁提交小数据，USB 协议头开销占比高，带宽浪费在协议层
- AIO 深队列的吞吐优势完全失效，因为每个 URB 都是半空的

聚合层通过两个方向的操作弥合这个矛盾：

- **TX 方向：聚合**。把多个小逻辑包拼进同一个 slab，填满 1MB 再提交，最大化 URB 利用率。
- **RX 方向：拆解**。把收到的 1MB block 按 TLV 顺序还原成一个个逻辑包，交回应用层。

同时对**延迟敏感的控制类小包（msg）**保留直透路径，不做聚合，兼顾低延迟。

---

## 二、整体架构

在应用层和传输层之间加入**帧聚合层（Frame Aggregation Layer）**，TX 方向聚合，RX 方向拆解。

> 注：本文只讲聚合层。端到端的速率匹配（credit 背压）见《算力棒 USB 驱动设计 - 流控层》。

```
┌──────────────────────────────────────────────────────────────┐
│                          应用层                                │
│      send_msg(buf,len)        send_pkt(buf,len)               │
│      recv_msg(buf,len)        recv_pkt(buf,len)               │
└────────┬──────────▲───────────────────┬──────────▲────────────┘
         │TX        │RX                 │TX        │RX
         ▼          │                   ▼          │
┌──────────────────────────────────────────────────────────────┐
│                       帧聚合层                                 │
│   msg TX（直透）  msg RX（直透）  pkt TX（聚合）  pkt RX（拆帧）│
└────────┬──────────▲───────────────────┬──────────▲────────────┘
         │          │                   │          │
         ▼          │                   ▼          │
┌──────────────────────────────────────────────────────────────┐
│                        传输层                                  │
│        msg endpoint (ep1)           pkt endpoint (ep2)        │
│        AIO depth=8, block=4KB       AIO depth=64, block=1MB   │
└──────────────────────────────────────────────────────────────┘
                            │
                       USB 总线（SuperSpeed）
```

---

## 三、传输层架构

传输层维护两条独立的 AIO 通道，分别对应两个 functionfs endpoint，TX/RX 方向各自维护独立的 iocb 队列。

```
传输层内部结构：
  msg endpoint (ep1)                    pkt endpoint (ep2)
  ┌──────────────────────────┐          ┌──────────────────────────────┐
  │ block size = 4KB         │          │ block size = 1MB             │
  │ AIO queue depth = 8      │          │ AIO queue depth = 32~64      │
  │                          │          │                              │
  │  TX iocb ring (depth=8)  │          │  TX iocb ring (depth=32~64)  │
  │  [iocb][iocb]...[8]      │          │  [iocb][iocb]...[64]         │
  │   ↓ 每个对应一次小URB提交 │          │   ↓ 每个对应一个完整slab提交  │
  │                          │          │                              │
  │  RX iocb ring (depth=8)  │          │  RX iocb ring (depth=32~64)  │
  │  [iocb][iocb]...[8]      │          │  [iocb][iocb]...[64]         │
  │   ↓ 预挂起，等host下发    │          │   ↓ 预挂起，等host下发1MB block│
  └────────────┬─────────────┘          └──────────────┬───────────────┘
               │                                       │
               └──────────── io_context_t ─────────────┘
                             （共享 AIO 上下文，
                              统一 io_getevents 轮询）
```

两条通道的设计差异：

| | msg endpoint | pkt endpoint |
|---|---|---|
| block size | 4KB | 1MB |
| AIO 队列深度 | 4~8 | 32~64 |
| 设计目标 | 低延迟，接受低利用率 | 高吞吐，URB 必须填满 |
| TX 聚合策略 | 无，直接提交 | slab 聚合后提交 |
| RX 拆帧策略 | 无，直接透传 | TLV 拆帧，seq 校验 |

msg / pkt 分成两个独立 endpoint，除了粒度差异，也为后续流控层预留了独立通道（credit 报文借道 msg 通道），此处不展开。

---

## 四、TX 方向：帧聚合层

### 4.1 msg TX 通道（直透）

msg 是控制类小包，延迟敏感，不聚合。

```
send_msg(buf, len)
    │
    ▼
直接写入 4KB block（不足补零）
    │
    ▼
立即提交 AIO iocb 到 msg endpoint TX ring
    │
    ▼
io_getevents 等待完成（或异步回调）
```

### 4.2 pkt TX 通道（slab 聚合）

pkt 是数据类包，吞吐敏感，聚合到 slab 填满再提交。

**pkt_hdr 结构**（引入 flags 支持定界与分片）：

```
struct pkt_hdr {
    u32 magic;    // 帧起始魔数，用于 RX 校验
    u32 seq;      // 递增序号，用于丢包检测
    u32 length;   // 本帧 payload 长度
    u32 flags;    // 见下表
};

flags 取值：
    FLAG_NORMAL      普通完整帧
    FLAG_PADDING     补齐帧（slab 末尾填充，必在末尾）
    FLAG_FRAG_BEGIN  超大包的首个分片
    FLAG_FRAG_CONT   超大包的中间分片
    FLAG_FRAG_END    超大包的末个分片
```

**slab 内部布局（1MB）**：

```
┌──────────┬───────────┬──────────┬───────────┬──────────────┐
│ pkt_hdr  │  payload  │ pkt_hdr  │  payload  │  PADDING帧    │
│(magic+   │  (变长)   │(magic+   │  (变长)   │ (补齐末尾)     │
│ seq+len+ │           │ seq+len+ │           │              │
│ flags)   │           │ flags)   │           │              │
└──────────┴───────────┴──────────┴───────────┴──────────────┘
↑ write_pos 从左向右推进                        ↑ flush时补齐到1MB
```

#### 4.2.1 slab 边界处理（关键）

设 slab 净容量 `CAP = 1MB`，当前剩余 `remain = CAP - write_pos`，一个逻辑包占用 `need = sizeof(pkt_hdr) + len`。分三种情况：

```
send_pkt(buf, len)：
   need = sizeof(pkt_hdr) + len

   ┌── case 1: need <= remain ───────────────────────────────┐
   │   直接写入当前 slab，write_pos += need                    │
   └─────────────────────────────────────────────────────────┘

   ┌── case 2: need > remain 且 need <= CAP ─────────────────┐
   │   （典型：当前 slab 已用 900K，下一包 150K，塞不下）        │
   │   1. 当前 slab 末尾补 PADDING 帧，补齐到 1MB              │
   │   2. flush 当前 slab（提交 AIO），swap 到新 slab          │
   │   3. 整包写入新 slab 起始位置                             │
   │   —— 原则：不把一个"能装进单 slab 的包"拆开跨 slab，       │
   │      保证接收侧单 slab 内 TLV 可独立解析                   │
   └─────────────────────────────────────────────────────────┘

   ┌── case 3: need > CAP（单包比整个 slab 还大）──────────────┐
   │   必须分片（fragmentation）：                             │
   │   1. 若当前 slab 有残留数据，先补 PADDING 并 flush         │
   │   2. 从新 slab 起，把该逻辑包切成若干分片：                 │
   │        第一片: flags=FRAG_BEGIN                          │
   │        中间片: flags=FRAG_CONT                           │
   │        末尾片: flags=FRAG_END                            │
   │      同一逻辑包各分片用连续 seq + FRAG 标志识别归属，       │
   │      每片独占（或近似独占）一个 slab                        │
   │   3. 逐片 flush，接收侧按 FRAG_BEGIN..FRAG_END 重组        │
   └─────────────────────────────────────────────────────────┘
```

要点：

- **case 2 是最常见的边界情况**：宁可牺牲当前 slab 末尾一点利用率（补 PADDING），也要保证每个普通包完整落在单个 slab 内，接收侧拆帧最简单，且每个 slab 可独立解析。
- **case 3 才允许跨 slab**：只有单包大于 slab 容量时才启用分片，用 FRAG 标志显式标记，接收侧维护重组缓冲区还原。
- PADDING 补齐时，若剩余空间不足以容纳一个 `pkt_hdr`，直接补零到末尾（接收侧扫描到不足一个头长度即视为 slab 结束）。

#### 4.2.2 三个 flush 触发条件

```
send_pkt(buf, len)
    │
    ▼
写入 slab[current]（按 4.2.1 决定是否先切 slab）
    │
    ├─── write_pos >= 1MB ──────────────→ 【size 触发】保证URB利用率最优
    │
    ├─── 距第一个包入队 > 1ms ──────────→ 【timeout 触发】数据稀疏时保证延迟上界
    │
    └─── 上层调用 send_pkt_flush() ─────→ 【flush 触发】语义边界（一帧/一次推理结束）
```

#### 4.2.3 双缓冲轮转

```
正常状态：
  slab[0]  ←── 帧聚合层正在写入（current）
  slab[1]  ←── 传输层 AIO 飞行中（inflight）

触发 flush 后：
  1. 末尾补 PADDING 帧，补齐到 1MB
  2. 提交 slab[0] 到传输层 TX AIO ring
  3. swap：slab[1] → current，slab[0] → inflight
  4. 重置 timerfd

唯一阻塞点：
  新 slab 又写满，但 inflight 还未完成
      │
      ▼
  等待 io_getevents 完成通知，再 swap
```

> 说明：这里的阻塞是聚合层的**本地**背压，只保证 inflight slab 完成后再复用 buffer，不感知对端消费进度。端到端速率匹配由流控层负责。

---

## 五、RX 方向：帧聚合层

### 5.1 msg RX 通道（直透）

```
传输层 msg endpoint RX ring 收到 4KB block
    │
    ▼
帧聚合层直接截取有效长度（去掉补零）
    │
    ▼
回调 recv_msg(buf, len) 交给应用层
    │
    ▼
立即向 RX ring 补挂一个新 iocb，保持队列满深度
```

### 5.2 pkt RX 通道（TLV 拆帧）

传输层每次收到一个完整的 1MB block，帧聚合层负责从中还原所有逻辑包。

```
传输层 pkt endpoint RX ring 收到 1MB block
    │
    ▼
帧聚合层按 TLV 顺序扫描
    │
    ├─── 剩余空间 < sizeof(pkt_hdr) → 视为 slab 结束，扫描结束
    │
    ├─── 读 pkt_hdr，校验 magic
    │         │
    │         ├─ magic 不匹配 → 丢弃整个 block，记录错误，继续下一个 block
    │         └─ magic 匹配 ↓
    │
    ├─── 校验 seq 连续性
    │         │
    │         ├─ seq 跳变 → 记录丢包计数，继续处理（不阻塞）
    │         └─ seq 正常 ↓
    │
    ├─── flags == PADDING → 跳过，扫描结束（PADDING 必然在末尾）
    │
    ├─── flags 含 FRAG_* → 进入分片重组：
    │         FRAG_BEGIN → 新建重组缓冲，拷入本片 payload
    │         FRAG_CONT  → 追加到重组缓冲
    │         FRAG_END   → 追加后整包完成，回调 recv_pkt(重组buf, 总长)
    │
    └─── FLAG_NORMAL → 按 length 截取 payload，回调 recv_pkt(buf, len)
                  read_pos += sizeof(pkt_hdr) + length，继续扫描
```

**RX 双缓冲**：

```
RX 方向同样维护双缓冲，与 TX 对称：
  slab[0]  ←── 传输层正在填充（inflight，挂在 RX AIO ring）
  slab[1]  ←── 帧聚合层正在拆帧（current）

收到完成通知后：
  1. swap：slab[0] → current（开始拆帧），slab[1] → inflight（立即重新挂起）
  2. 拆帧和下一个 URB 接收并行，不阻塞吞吐
```

---

> 下篇：《算力棒 USB 驱动设计 - 流控层》——讲 credit 背压、pkt/msg 独立流控通道、TX/RX 独立流控。