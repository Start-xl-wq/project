# Linux USB Gadget 核心架构与数据流转模型

> **定位**：Linux 内核 USB Device 侧软件架构的“承上启下枢纽”。上承各种应用功能（ACM/ADB/RNDIS/UVC），下接各大原厂控制器驱动（DWC3/ChipIdea/MUSB）。

---

## 1. 三层金字塔架构

Linux USB Gadget 驱动框架自顶向下分为三层：

```text
┌────────────────────────────────────────────────────────┐
│ 1. Function Layer (功能层 / 设备类驱动)                  │
│    - f_acm.c, f_adb.c, f_mass_storage.c, f_uvc.c 等    │
├────────────────────────────────────────────────────────┤
│ 2. Composite Layer (复合设备通用管理层 / ConfigFS)        │
│    - composite.c, configfs.c, ep0 控制传输分发与描述符拼装 │
├────────────────────────────────────────────────────────┤
│ 3. UDC Controller Driver (底层控制器硬件驱动)            │
│    - dwc3/gadget.c, udc-core.c (直接操作芯片寄存器与 DMA)│
└────────────────────────────────────────────────────────┘
```

---

## 2. 核心数据结构与关系

* **`struct usb_gadget`**：代表底层的物理控制器抽象（UDC），包含端点链表、当前工作速度、指向控制器设备树节点的指针。
* **`struct usb_ep`**：逻辑端点抽象（如 `ep1in`, `ep1out`），封装端点地址、最大包长 `maxpacket`、端点操作集 `ops->queue()`。
* **`struct usb_request`**：**数据传输的最小载体**。
  ```c
  struct usb_request {
      void            *buf;       // 传输缓冲区虚拟地址
      dma_addr_t      dma;        // 传输缓冲区物理/IOVA地址
      unsigned        length;     // 申请传输的字节数
      unsigned        actual;     // 实际完成传输的字节数
      int             status;     // 传输完成状态 (0=成功, -ECONNRESET等)
      void            (*complete)(struct usb_ep *ep, struct usb_request *req); // 完成回调
      void            *context;   // 上层私有上下文指针
  };
  ```
* **`struct usb_function`**：代表一个具体的 USB 功能实例（如一个 CDC 串口或一个虚拟网卡），提供 `bind()`, `set_alt()`, `disable()` 等状态机回调。

---

## 3. 数据流转与请求生命周期 (`usb_request`)

一个典型的下行收包（OUT）或上行发包（IN）数据流动流程：

```text
[功能层]               [UDC 核心层]             [硬件控制器/DWC3]
   │                         │                         │
   ├── 1. alloc_request() ───┼────────────────────────►│ (分配请求内存)
   ├── 2. usb_ep_queue() ────┼────────────────────────►│ (排队挂入硬件TRB环)
   │                         │                         │
   │                    (硬件开始传输...)                 │
   │                         │                         │
   │                         │◄── 3. 硬件DMA完成中断 ────┤ (DMA刷入DDR)
   │◄── 4. req->complete() ──┼─────────────────────────┤ (回调通知业务层)
   │                         │                         │
```

---

## 4. ConfigFS 动态配置机制
现代 Linux 普遍采用 ConfigFS 在用户态动态组织 USB Gadget，无需重新编译内核模块：
1. `mkdir /sys/kernel/config/usb_gadget/g1`：创建 Gadget 实例；
2. 写入 `idVendor`, `idProduct`, `bcdDevice`；
3. 创建字符串描述符（厂商、产品名称、序列号）；
4. 创建并挂载功能配置：`mkdir functions/acm.usb0`, `mkdir configs/c.1`；
5. 建立软链接将 function 绑定至 configuration；
6. 激活总线：`echo <udc_name> > UDC`，触发底层控制器上电使能 D+ 上拉电阻，Host 开始枚举。
