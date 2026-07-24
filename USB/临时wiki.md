# 算力棒 USB 驱动设计 - 流控层

## 一、流控层的作用

聚合层解决了**粒度匹配**（URB 利用率、吞吐），但没解决**速率匹配**——当发送方持续快于接收方消费时会出问题。流控层的职责就是让**发送速率跟随接收方的实际消费速率**。

先厘清一个前提：USB bulk 有硬件级背压。接收方 RX ring 无空闲 iocb 时，host controller 回 NAK，发送方 AIO 提交阻塞——**数据不会丢，但会阻塞**。所以流控要解决的不是丢包，而是这个"阻塞"带来的两个问题：

1. **吞吐掉底**：接收方消费跟不上，发送方长期被 NAK 硬阻塞在传输层，这个阻塞发生在底层，不可观测、不可控。
2. **死锁**：双向同时阻塞在写，谁也推进不了。

**为什么聚合层的双缓冲不够？** 聚合层的双缓冲 + 深 RX ring 只能吸收**短时抖动**。当发送方**持续**快于接收方消费时，ring 再深也会被填满并阻塞回发送方。

**结论**：治标靠深 ring，治本必须靠应用级 credit。**pkt 和 msg 两条通道都需要 credit**——两条通道都有各自的 RX ring，都会被灌满阻塞，只是粒度和批量策略不同。

---

## 二、整体位置

流控层夹在应用层和聚合层之间，对 pkt / msg 两条通道各维护一套 credit。

```
┌──────────────────────────────────────────────────────────────┐
│                          应用层                                │
│  send_pkt/recv_pkt/release_pkt   send_msg/recv_msg/release_msg│
└────┬───────────▲────────┬──────────┬───────────▲────────┬─────┘
     │TX         │RX      │release    │TX         │RX      │release
     ▼           │        ▼           ▼           │        ▼
┌──────────────────────────────────────────────────────────────┐
│                       流控层                                   │
│   pkt_tx_flow / pkt_rx_flow    msg_tx_flow / msg_rx_flow      │
│   credit 够才放行，否则挂起；攒够 release 批量回传 credit       │
└────┬───────────▲────────────────────┬───────────▲────────┬────┘
     │           │                    │           │        │
     ▼           │                    ▼           │        ▼
┌──────────────────────────────────────────────────────────────┐
│                       帧聚合层                                 │
│   pkt TX（聚合）  pkt RX（拆帧）      msg TX（直透） msg RX      │
└──────────────────────────────────────────────────────────────┘
```

---

## 三、credit 机制

credit 是"接收方准备好的接收 buffer 数量"的令牌。pkt 和 msg 都用同一套模型，只是粒度不同：

| | pkt credit | msg credit |
|---|---|---|
| credit 单位 | 1 slab（1MB） | 1 msg block（4KB） |
| 初始 credit | 32~64 | 4~8 |
| 扣减时机 | flush 一个 slab → credit-- | 提交一个 msg → credit-- |
| 补充时机 | 应用真正消费完 slab，`release_pkt(n)` | 应用真正消费完 msg，`release_msg(n)` |
| 回传批量 refill_batch | 16 个攒一次 | 2~4 个攒一次（msg 稀疏，批量要小） |

```
发送方                                   接收方
 │── 初始 credit（我准备了N个接收buffer）─────────→│
 │←──────── data ─────────────────────────────────│  credit-- 
 │←──────── data ─────────────────────────────────│  credit-- 
 │   ...（接收方真正消费掉一批后）                   │
 │←────── credit 补充 +batch ──────────────────────│  credit += batch
 │←──────── data ─────────────────────────────────│  继续发
```

通用规则：

- **发送侧**：每发一个单元，credit 减 1。`credit <= 低水位` 时**在流控层的 send 入口挂起**（或返回 EAGAIN），**绝不把数据继续灌到传输层去被 USB NAK 硬阻塞**——让背压发生在可控、可观测的逻辑层。
- **接收侧**：绑定"**真正消费完**"回传 credit，不是"收到"就回。只有绑定消费完成，credit 才真实反映接收方的处理能力。

---

## 四、credit 报文的通道与"免流控"约束（关键）

所有方向的 credit 补充报文，**统一走 msg 通道回传**（4KB，低延迟）。这带来一个必须处理的循环依赖：

> **msg 通道自己也有 credit，而 credit 报文又是一条 msg——如果 credit 报文也消耗 msg credit，msg credit 耗尽时就再也发不出任何 credit，形成自我死锁。**

死锁链：

```
msg credit 耗尽 → 想回传 credit 补充报文，但它自己也是 msg → 发不出去
→ 对端永远等不到 credit → 双方 msg 通道全部停摆 → 死锁
```

**解法：credit 报文是"免流控"的控制报文（out-of-band control）**，不受任何 credit 约束：

- credit 报文**不消耗** msg credit，也不占用普通 msg 的 RX ring 配额。
- 接收侧在 msg RX ring 中**预留固定数量的 iocb 专供控制报文**（如 depth 8 里留 2 个），保证 credit 报文永远有落脚点。
- 拆帧时按 msg 头部的类型字段区分：`TYPE_DATA`（普通 msg，走 msg_rx_flow 计入 credit）与 `TYPE_CREDIT`（控制报文，直接更新对应通道的发送方 credit 计数，不计消费、不回传）。

这样：**普通 pkt / 普通 msg 都受 credit 约束，唯独 credit 报文自身免流控**，打破循环依赖。

同时，pkt 与 msg 通道物理隔离（独立 endpoint）这一点，在开启流控后从"优化项"升级为"正确性必需项"：若 credit 报文和大数据挤在同一通道，会排在几十 MB 数据后面才到达，流控失效。聚合层的双 endpoint 设计正好满足这一前提。

---

## 五、TX 与 RX、pkt 与 msg 各自独立

发送/接收是独立方向，pkt/msg 是独立通道，**四者各维护一套独立 credit**（独立计数器、水位、回传逻辑），互不共享。用同一套对称代码实例化四个实例即可：

```
Device 侧四套 flow：
   pkt_tx_flow  → 发结果(IN)：flush slab → credit--；收 host 的 credit-msg → 补充
   pkt_rx_flow  → 收输入(OUT)：release_pkt(n) 攒够 → 回传 credit 给 host
   msg_tx_flow  → 发控制/状态：发 msg → credit--；收 host 的 credit-msg → 补充
   msg_rx_flow  → 收控制/命令：release_msg(n) 攒够 → 回传 credit 给 host
```

要点：

- **credit 报文流向与数据流相反**：谁是接收方，谁往回发 credit。
- credit-msg 用字段区分"补给哪条通道、哪个方向"（如 `{channel: pkt/msg, dir: tx/rx, delta: n}`）。多个方向的 credit 补充可以**合并进一条 credit-msg**，进一步减少控制报文数量。
- **某方向是否启用 credit**，取决于"发送方是否可能**持续**快于接收方消费"：
  - **pkt 发结果（device→host）**：结果量大，几乎一定要开。
  - **pkt 发输入（host→device）**：输入大到能撑爆 device 内存就要开（device 内存通常比 host 紧张）。
  - **msg 双向**：控制/命令流通常稀疏，但**突发**（如 host 批量下发一堆命令）也会灌满 device 的 msg RX ring，所以 msg 同样需要 credit 兜住突发。

---

## 六、与聚合层双缓冲的关系

流控层的 credit 背压和聚合层的双缓冲背压是**两个层次、互补**的机制：

| | 聚合层双缓冲阻塞 | 流控层 credit |
|---|---|---|
| 层次 | 本地背压 | 端到端背压 |
| 保证 | inflight buffer 完成后再复用 | 对端有空位、消费得过来才发 |
| 感知范围 | 只感知本端 AIO 完成 | 感知对端消费进度 |

两者叠加协作：

- credit 决定"**能不能发**"，双缓冲决定"**发出去后 buffer 怎么复用**"。
- credit 充足时，双缓冲全速轮转，跑满带宽。
- credit 耗尽时，在 send 入口挂起，双缓冲自然停在满状态，不会有数据继续下沉到传输层被硬阻塞。

---

## 七、端到端保障链（小结）

```
硬件层：USB bulk NAK ───────── 兜底不丢数据（但会阻塞）
传输层：pkt/msg RX ring ─────── 吸收短时抖动，完成回调里立即补挂 iocb
聚合层：RX 双缓冲 + TLV 拆帧 ── 拆帧与接收并行；大包分片重组
流控层：pkt/msg × tx/rx 四套 credit ─ 钳住发送速率 = 对端消费速率
                                     credit 报文免流控，走独立 msg 通道回传
架构上：pkt/msg 独立 endpoint ─ 保证 credit 送得到；破解双向死锁
```

一句话总结：

> **粒度匹配靠聚合层，速率匹配靠流控层；深 ring 治标、credit 治本；pkt/msg 通道独立保证 credit 送得到，pkt/msg × tx/rx 四套独立 credit 保证每条通道每个方向都各自受控；唯独 credit 报文自身免流控，打破循环依赖。**