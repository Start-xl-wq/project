## 1. 概览

ZC 通道承载 402 MB/s 的核心在于**两端都保持多笔传输在途**，让 DMA 引擎永不空转：

- Device 侧：**3 笔 request 在途**（受 DWC3 TRB ring 约束）
- Host 侧：**4 个 URB 在途**（受回调时延约束）
- wire 上是裸 payload 字节流，无 header，Host 侧靠**字节计数收敛**判定收齐。

前提：本次传输的元信息（传输 id、总字节数、SG 颗粒）已由 SERVICE 通道**前置协商**完成，故 ZC wire 可彻底裸奔。

---

## 2. 端到端时序

Text

Device业务 Device驱动 USB线(SS) Host驱动 Host业务

```
│                │                                    │              │

│                │        ── 传输前经 SERVICE 协商 id/总长/颗粒 ──      │

│                │                                    │              │

│ zc_submit      │                                    │ 预挂4个URB    │

│  (id,len,sg)   │                                    │◄─ zc_recv ───┤

├───────────────►│ 建 SG request                      │   (id,sg)    │

│                │ 拆成 request[0..2]                 │              │

│                ├─ ep_queue req0 ═══ burst16 ═══════►│ URB0 complete│

│                ├─ ep_queue req1 ═══ burst16 ═══════►│ URB1 complete│

│                ├─ ep_queue req2 ═══ burst16 ═══════►│ URB2 complete│

│                │  ↑ 3笔在途, DMA不空转   ↓            │  ↓字节计数++  │

│                │◄ req0 giveback ─ 补 req3 ─────────►│ URB0 重提交   │

│                │◄ req1 giveback ─ 补 req4 ─────────►│ URB1 重提交   │

│                │        ……持续流水……                 │  ……          │

│                │◄ 末笔 giveback                     │ 计数==总长     │

│◄── done 回调 ──┤                                    ├── recv 回调─► │

│  (id, status)  │                                    │  (id,收齐)   │
```

---

## 3. Device 侧：提交与流水补充

### 3.1 zc_submit 入口

业务模块调用 `zc_submit(id, len, sg)`，传入一段 scatter-gather buffer。驱动不拷贝 payload，直接把 SG 挂到 USB request 的 `sg / num_sgs` 上，交由 DWC3 做 SG-DMA。

### 3.2 拆分与在途窗口

一次大传输拆成多笔 request，**始终保持 3 笔在途**：

Text

初始: queue req0, req1, req2 (在途=3)

```
   ┌─────────────────────────────┐
```

每当: │ reqN complete(giveback) │

```
   │   → 若还有剩余数据            │

   │   → 立即 queue req(N+3)      │  维持在途=3

   └─────────────────────────────┘
```

末尾: 剩余数据不足则不再补, 在途逐笔收敛到 0

3 笔是 DWC3 TRB ring 深度与 SG 段数共同决定的经验窗口：太少则 giveback 到补队之间出现空窗，太多则 ring 溢出。

### 3.3 giveback 在原子上下文

request 完成回调（giveback）运行在**原子上下文**，不能睡眠。因此补队逻辑必须是纯非阻塞操作：只做 `usb_ep_queue`（本身可在原子上下文调用），不做任何分配/等待。若需要分配新 request，从**预分配的 request 池**取，不在回调里 kmalloc。

### 3.4 完成通知

末笔 giveback 后，驱动通过 `done 回调`通知业务模块本次 `id` 传输完成及最终 status。

---

## 4. Host 侧：预挂 URB 与字节计数收敛

### 4.1 zc_recv 入口

Host 业务调用 `zc_recv(id, sg)`，把接收 SG buffer 交给驱动。驱动**预先挂 4 个 URB**到 ZC IN 端点，全部挂到 anchor。

### 4.2 为什么是 4 个而非 3 个

Host 侧在途窗口受 URB complete 回调调度时延约束，比 Device 侧多 1 个作缓冲：

Text

Device 3笔 ── wire ── Host 4个URB

在途窗口不对称: Host 多 1 个吸收回调调度抖动, 防接收侧空窗

### 4.3 字节计数收敛

wire 上无 header、无边界信号，Host 靠**累计收到的字节数**判定收齐：

Text

每个 URB complete:

```
received += urb->actual_length

把该 URB 数据落到 SG 对应偏移

if received < total:

    重提交该 URB (维持在途=4)

else:

    停止重提交, 触发 recv 回调(id, 收齐)
```

- **total 来自 SERVICE 前置协商**，这是裸字节流能收敛的前提。
- 收齐后剩余在途 URB 自然回落，无需额外信号。

### 4.4 短包与末尾对齐

末笔数据长度通常不是 MPS 整数倍，会产生一个短包（short packet），xHCI 据此正常结束该 URB。字节计数在短包上正好补齐到 total，收敛点与短包点一致。

---

## 5. 两侧在途窗口对照

||Device 侧|Host 侧|
|---|---|---|
|在途单位|request|URB|
|在途数量|3|4|
|约束来源|DWC3 TRB ring 深度|complete 回调调度时延|
|补队时机|giveback 回调内|complete 回调内|
|补队上下文|原子（不能睡眠）|原子（不能睡眠）|
|收敛判据|剩余数据耗尽|字节计数 == total|

---

## 6. 流水为什么能压满带宽

Text

朴素单笔: queue ─ 传 ─ giveback ─(空窗)─ queue ─ 传 ─ ……

```
                          ↑ DMA 空转, 带宽掉到 ~200 MB/s
```

3/4 笔流水: ═══════════════════════════════════════

```
         传输连续无缝, giveback 与下一笔传输重叠

                          ↑ DMA 不空转, 配 burst=16 → 402 MB/s
```

关键三点叠加：

1. **多笔在途**：giveback 与下一笔传输时间重叠，消除软件空窗。
2. **SG-DMA 直达**：payload 零拷贝，无 CPU 搬运瓶颈。
3. **SS burst=16**：单次链路握手连发 16 包，摊薄协议开销。

---