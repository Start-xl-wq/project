## 1. 与 ZC 的本质区别

||ZC 通道|SERVICE / DATA 通道|
|---|---|---|
|wire 格式|裸 payload 字节流|32B header + payload|
|拷贝语义|zero-copy，SG-DMA 直达|copy，进出 driver buffer|
|收敛判据|字节计数 == 协商总长|header 自带长度，一帧即一个单位|
|在途流水|3/4 笔|单笔（无需流水）|
|前置协商|需要（元信息走 SERVICE）|不需要（header 自描述）|
|用途|大数据流|控制 / 协商 / 小帧|

一句话：**帧通道每次收发就是一个自包含的帧，header 说清楚这一帧多长、是什么，无需任何前置状态。** 所以实现简单，不追求带宽。

---

## 2. wire 格式：32B header + payload

Text

┌──────────────────────────────┬─────────────────────┐

│                       32B header                                                                    │               payload                                                  │

├──────────────────────────────┼─────────────────────┤

│ magic             |   type                   |   flags                  |    len                 │             业务数据                                                 │

│                         (对齐/校验字段, 共32字节)                                          │              (len 字节)                                                │

└──────────────────────────────┴─────────────────────┘

```
    自描述, 接收端读 header 即知整帧边界
```

- `type`：区分 CTRL / DBG / CAPS / ABORT 等（SERVICE 通道），或 DATA 帧类型。
- `len`：payload 字节数，接收端据此知道整帧多长。
- header 与 payload 在同一笔传输里连续发出，接收端一次收进 driver buffer 后就地解析。

---

## 3. 发送时序（以 device → host 为例）

Text

业务模块                                              Driver                                                   USB线               对端Driver                  对端业务

│ send_frame                                       │                                                                                      │                             │

│ (type,buf,len)                                    │                                                                                      │                             │

├───────────────►│ 取 driver txbuf                                                              │ 预挂 rx URB         │

│                                                          │ 填 32B header                                                              │                             │

│                                                          │ memcpy payload → txbuf                                            │                             │

│                                                          ├─ ep_queue (header+payload一笔) ───►             │ complete             │

│                                                          │◄─ giveback ─                                                           ├─ 解析 header     │

│◄── done ──────────┤                                                                                      ├─ memcpy出       │

│                                                         │                                                                                      ├── recv 回调─►│

**发送三步，无流水：**

1. 从 driver 的发送 buffer 取一块，填好 32B header（type / len / flags）。
2. `memcpy` 业务 payload 进 buffer（copy 语义，这里就是与 ZC 最大的不同）。
3. `usb_ep_queue` 一笔发出（header 与 payload 连续）。giveback 后通知业务 done。

因为帧通道流量小、不追吞吐，**单笔发送即可，不做多笔在途流水**。

---

## 4. 接收时序

Text

对端Driver 预挂 rx request/URB, 等待入帧

```
    │

    ▼
```

一笔传输到达 (header + payload)

```
    │

    ├─ 读 header.magic 校验

    ├─ 读 header.type   分派

    ├─ 读 header.len    取 payload

    ├─ memcpy payload → 业务 buffer (或直接回调传指针)

    ├─ 触发 recv 回调(type, buf, len)

    │

    └─ 重新预挂 rx request/URB, 等下一帧
```

**接收要点：**

- 接收端**常驻预挂**收 request/URB，收到一帧、解析、回调、再补挂，形成常开的收帧环。
- 一笔传输 = 一个完整帧（header + payload 一起到），无需跨笔拼接。
- 按 `header.type` 分派：SERVICE 通道据此区分 CTRL / CAPS / ABORT 等控制语义。

---

## 5. 为什么帧通道能这么简单

|简化点|原因|
|---|---|
|无前置协商|header 自描述，每帧自包含，接收端无需预知长度|
|无字节计数收敛|`header.len` 直接给出边界，一笔到齐|
|无多笔流水|流量小、不追带宽，单笔够用|
|copy 语义|帧小，memcpy 开销可忽略，换取业务 buffer 生命周期解耦|

帧通道用**一点点拷贝开销和自描述 header**，换来了极简的实现和零协商状态。ZC 则相反：牺牲实现复杂度（前置协商 + 字节计数 + 多笔流水 + SG-DMA），换取 402 MB/s。两者定位互补。

---

## 6. SERVICE 与 DATA 的差异

两条通道机制完全相同，仅**用途和端点不同**：

||SERVICE 通道|DATA 通道|
|---|---|---|
|端点|SERVICE IN/OUT|DATA IN/OUT|
|header.type|CTRL / DBG / CAPS / ABORT|常规数据帧类型|
|时延要求|敏感（控制信令）|一般|
|关键作用|承载 ZC 前置协商、ABORT，满负荷时仍可达|常规可靠小帧收发|

SERVICE 独占一对端点，正是为了让控制信令**不被 DATA 或 ZC 的大流量阻塞** —— 这是三通道分离的核心价值。

---

## 7. 三通道数据面小结

Text

SERVICE ┐

```
    ├─ 32B header + copy + 单笔 ── 简单, 自描述, 时延敏感
```

DATA ─┘

ZC ─────── 裸字节流 + SG-DMA + 3/4笔流水 ── 复杂, 高带宽

- **帧通道（SERVICE/DATA）**：header 自描述，一笔一帧，copy 语义，实现极简。
- **ZC 通道**：裸奔 + 流水 + 直达 DMA，实现复杂，专供 402 MB/s。
- 三者共享端点/生命周期层，各走独立端点对，互不阻塞。