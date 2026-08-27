## 1. 背景与目标

产品为 USB 算力棒，通过 USB3.0 接口与主机交换数据。核心指标：

| 项      | 值                             |
| ------ | ----------------------------- |
| 接口     | USB3.0 (SuperSpeed)           |
| 理论带宽   | 5 Gbps                        |
| 目标有效带宽 | ≥ 400 MB/s（D2H 大块数据）          |
| 传输模型   | device ↔ host 大块数据高吞吐 + 控制/协商 |

算力棒工作在 USB device（gadget） 模式，主机侧为 host 模式，主机初期采用 Linux SoC。**D2H 实测已达 402 MB/s。**

与用户态方案（f_fs + libusb）不同，本驱动采用**内核态实现**：两端均为自研内核驱动，向内核业务模块直接暴露 symbol API，业务模块无需经过用户态即可高速收发数据。

---

## 2. 系统架构

### 2.1 端到端数据链路

Text

```
     device 端(算力棒)                    host 端(Linux SoC)

┌──────────────────────┐            ┌──────────────────────┐

│  内核业务模块          │            │  内核业务模块          │

│  zc_submit/send_frame│            │  zc_recv/recv_frame  │

└──────────┬───────────┘            └───────────┬──────────┘

           │ symbol API                         │ symbol API

┌──────────┴───────────┐            ┌───────────┴──────────┐

│  算力棒驱动 (gadget)  │            │  算力棒驱动(usb_driver)│

└──────────┬───────────┘            └───────────┬──────────┘

           │                                    │

  composite core (内核)                  USB core (内核, 自带)

           │                                    │

      udc 驱动                            xhci host 驱动(自带)

           │                                    │

═══════════╧════════════ USB3.0 物理线 ═════════╧═══════════

 两端均为: 内核业务模块 + 自研内核驱动 + 内核 USB 通路
```

### 2.2 分层结构

驱动两端结构对称，各自分为两层，再向上通过客户端接口暴露给业务模块：

Text

┌────────────────────────────────────────────────────────────────┐

│ 客户端接口                                                                                                                                                                                                                      │

│ zc_submit / send_frame / register_client / is_online                                                                                                                                                       │

├────────────────────────────────────────────────────────────────┤

│ 通道层（数据面）                                                                                                                                                                                                           │

│ ┌────────────────┬────────────────┬────────────────────────┐             │

│ │ SERVICE 通道                                    │ DATA 通道                                         │ ZC 通道                                                                          │            │

│ │ 控制/协商/ABORT                             │ 常规可靠帧                                        │ zero-copy 大数据流                                                      │             │

│ │ 32B header,copy                               │ 32B header,copy                               │ 裸 payload,SG-DMA直达                                              │             │

│ └────────────────┴────────────────┴────────────────────────┘             │

├────────────────────────────────────────────────────────────────┤

│ 端点 / 生命周期层                                                                                                                                                                                                           │

│ 6 端点布局 · 速率适配(FS/HS/SS) · 原子上下文异步化                                                                                                                                                    │

│ 资源回收 · online 状态维护                                                                                                                                                                                             │

└────────────────────────────────────────────────────────────────┘

```
                          │

                   USB 总线 (SuperSpeed)
```

### 2.3 各层职责

|层|Device 侧|Host 侧|职责|
|---|---|---|---|
|客户端接口|symbol API + 回调|symbol API + 回调|业务模块唯一入口，不感知下层实现|
|通道层（数据面）|三通道收发|三通道收发|按流量语义分三条独立通道，互不阻塞|
|端点/生命周期层|6 端点 + 生命周期|6 端点 + URB 管理|端点布局、速率适配、资源回收、online 维护|
|USB 通路（现成）|composite / udc|USB core / xhci|内核自带，非自研|

---

## 3. 三条通道

数据面按流量语义横向切成三条并列通道，彼此独立收发、互不阻塞：

|通道|端点方向|wire 格式|拷贝语义|用途|
|---|---|---|---|---|
|**SERVICE**|IN/OUT|32B header + payload|copy|CTRL / DBG / CAPS 协商 / ABORT|
|**DATA**|IN/OUT|32B header + payload|copy|常规可靠帧|
|**ZC**|IN/OUT|裸 payload 字节流|zero-copy|大数据流，SG-DMA 直达，402 MB/s|

- **SERVICE 与 DATA 为帧通道**：wire 带 32B header，自描述结构，接收端按 header 重组帧。
- **ZC 为数据面核心**：wire 纯 payload、无结构，业务 SG buffer 直接 DMA，靠字节计数收敛，承载 400 MB/s 目标。
- **通道分离的意义**：ABORT 等控制信令走 SERVICE，即使 ZC 满负荷传输也能即时到达，不被大流量阻塞。

---

## 4. 端点布局

单一 vendor-specific interface，`bNumEndpoints = 6`，三对成对布置：

|端点|wMaxPacketSize (SS)|bMaxBurst (SS)|说明|
|---|---|---|---|
|SERVICE IN/OUT|1024|0|控制面，时延敏感|
|DATA IN/OUT|1024|0|copy 语义帧|
|ZC IN|1024|**15**（16 pkt/burst）|D2H 带宽核心|
|ZC OUT|1024|0|H2D 仅声明能力|

- Device 侧提供 FS/HS/SS 三套描述符，由 composite core 按协商速率自动挑选。
- Host 侧 probe 时按发现顺序绑定端点，并缓存 ZC IN 的 MPS 供分片对齐。
- **ZC IN burst=16 是把 D2H 从约 300 MB/s 拉到 402 MB/s 的关键。**

---

## 5. 两端角色对照

||Device 侧（gadget）|Host 侧（usb_driver）|
|---|---|---|
|挂接|composite core|USB core / xHCI|
|链路建立|set_alt → workqueue 使能端点|probe 发现端点|
|资源回收|usb_ep_disable 强制 giveback|anchor 统一 kill URB|
|ZC 在途窗口|3 笔 request|4 个 URB|
|生命周期难点|回调处于原子上下文，需异步化|URB 生命周期管理|

- **Device 侧**：composite 回调运行在原子上下文，端点使能等可睡眠操作投递到 workqueue 执行，回调本身立即返回。
- **Host 侧**：所有在途 URB 挂到 anchor，disconnect 时逐 anchor kill，统一回收、杜绝泄漏。

---

## 6. 关键设计取舍

|取舍|原因|
|---|---|
|数据面分三通道而非单一通道|控制信令与大流量隔离，ABORT 不被 ZC 阻塞|
|ZC 用裸字节流 + 字节计数收敛|有效载荷 100%，收敛确定性，不依赖 wire 边界信号|
|ZC 元信息经 SERVICE 前置协商|数据面得以彻底裸奔，无自描述开销|
|ZC 多请求流水（Device 3 / Host 4 在途）|DMA 引擎不空转，消除软件空窗|
|ZC IN 配置 SS burst=16|减少链路握手，压满带宽|
|Device 回调只投递、workqueue 干重活|规避原子上下文不能睡眠的限制|

---

## 7. 实测带宽

|场景|带宽|
|---|---|
|D2H，64 KiB SG，无 burst|311 MB/s|
|D2H，64 KiB SG，burst=16|**402 MB/s**|
|D2H，4 KiB 极端碎片 SG|248 MB/s|

常规场景 402 MB/s 达标；4 KiB 碎片 SG 下段数暴增导致退化至 248 MB/s。

---