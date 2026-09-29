# DWC3 DMA 描述符环 (TRB) 与数据流转机制

> **定位**：Synopsys DesignWare USB3 (DWC3) 控制器驱动开发与数据面性能调优的核心。原厂驱动排查“发包卡死”、“内存踩踏”、“DMA FIFO 溢出”必看。

---

## 1. 什么是 TRB (Transfer Request Block)？
在 DWC3 控制器中，软件（驱动）不直接向硬件 FIFO 搬移数据，而是构建一个由 **TRB (传输请求块)** 组成的环形缓冲区（TRB Ring）。控制器内部的 DMA 引擎会自动解析 TRB，并发起 AXI 总线突发操作完成数据读写。

一个标准 TRB 占用 **16 字节 (4 个 32-bit 字)**：
```text
┌────────────────────────────────────────────────────────┐
│ Buffer Pointer Low (物理地址低 32 位)                     │
├────────────────────────────────────────────────────────┤
│ Buffer Pointer High (物理地址高 32 位)                    │
├────────────────────────────────────────────────────────┤
│ Size (传输字节数, 24-bit) | PCM (包计数值)                 │
├────────────────────────────────────────────────────────┤
│ Control / Status (HWO, LST, CHN, IOC, TRB Type 等控制标志) │
└────────────────────────────────────────────────────────┘
```

---

## 2. 关键控制位与标志 (TRB Control Bits)
* **HWO (Hardware Owns)**：
  * `1`：归硬件所有。驱动填好数据后置 1，告知 DWC3 DMA 可以读取该 TRB 并传输。
  * `0`：归软件所有。DMA 传输完成或发生错误时，硬件自动将 HWO 置 0。
* **LST (Last)**：标识当前 TRB 是否是该笔传输的最后一个数据块。
* **CHN (Chain)**：是否与下一个 TRB 链接（用于支持非连续分散物理内存 Scatter-Gather 链表）。
* **IOC (Interrupt On Completion)**：传输完成时是否向 CPU 产生硬件中断。
* **ISP (Interrupt on Short Packet)**：收到短包（Short Packet）时是否强行触发中断并截断后续传输。
* **TRB Type**：
  * `Normal` (标准数据搬运)
  * `Link` (环形回环跳转指针，用于把环首尾相连)
  * `Control Setup` / `Control Status` (控制传输专用)

---

## 3. 端点数据流转全过程 (以 Device Bulk OUT 为例)

1. **软件分配与排队 (Software Prepare)**：
   * 上层驱动调用 `usb_ep_queue(ep, req)`；
   * DWC3 驱动从当前端点的 `trb_pool` 中取出一个空闲 TRB；
   * 填入 `req->dma` 物理地址、`req->length` 字节数；
   * 置位 `HWO=1`、`CHN=0`、`IOC=1`；
   * 执行内存屏障 `dma_wmb()` 确保写回物理内存。
2. **触发硬件启动 (DepStartNewCfg / UpdateTransfer)**：
   * 向对应端点的物理寄存器写入启动命令：`dwc3_send_gadget_ep_cmd(DEPCMD_STARTTRANSFER)`；
   * 控制器解析命令，内部 DMA 引擎读取该 TRB。
3. **数据搬运与总线交互 (DMA Transfer)**：
   * 外部 Host 发送 Data Packet 到达控制器 RX FIFO；
   * 控制器 DMA 引擎发起 AXI Write 突发，将 FIFO 内的数据按照 TRB 填写的物理地址直接刷入系统 DDR 内存。
4. **中断产生与回收 (Interrupt & Recycle)**：
   * 传输完成，硬件将 TRB 的 `HWO` 标志清 0，并将状态写入 Event Buffer，触发系统中断；
   * CPU 中断下半部读取 Event Buffer，确认传输成功，调用 `req->complete(ep, req)` 通知业务层。

---

## 4. 芯片原厂 CV 重点关注与异常排查
- [ ] **AXI 4KB 边界跨越**：单次 TRB 请求是否跨越了 AXI 4KB 物理地址边界？部分旧版 DWC3 IP 对跨 4K 边界不支持，会直接触发 DMA Error 甚至总线死锁。
- [ ] **TRB 环满/枯竭 (Starvation)**：如果上层排队慢，硬件端点跑完最后一个 TRB 且未及时更新，硬件会进入 NAK 状态；需重点验证短包处理与重新启动时序。
- [ ] **Cache 一致性**：TRB 环所在内存必须使用 `dma_alloc_coherent()` 分配一致性非 Cache 内存，避免 CPU Cache 与 DMA 读写不一致。
