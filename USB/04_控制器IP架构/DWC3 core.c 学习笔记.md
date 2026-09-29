
### 1. `core.c` 的定位

`drivers/usb/dwc3/core.c` 是 DWC3 控制器的公共初始化层。

它不负责具体 USB 传输细节，也不负责 gadget function 逻辑。
它主要负责把 DWC3 controller 初始化到一个可工作的基础状态。

可以理解为：

```text
硬件上有一个 DWC3 controller
软件上用 struct dwc3 来描述和管理它
core.c 负责创建、填充、初始化这个 struct dwc3
```

---

## 2. `core.c` 主要做的事情

总体流程：

```text
dwc3_probe()
    ↓
分配 struct dwc3
    ↓
获取 platform/DTS 资源
    reg / irq
    ↓
映射寄存器空间
    ↓
读取 device property
    dr_mode / maximum-speed / quirks
    ↓
读取 DWC3 硬件寄存器
    revision / hwparams
    ↓
DWC3 core soft reset
    ↓
配置 global registers
    ↓
申请并配置 event buffer
    ↓
根据 dr_mode 进入 host/peripheral/otg 模式
```

---

## 3. `struct dwc3` 的作用

`struct dwc3` 是 DWC3 控制器在软件中的核心对象。

`core.c` 的很多工作，本质就是：

```text
分配 struct dwc3
填充 struct dwc3
根据 struct dwc3 初始化硬件
把 struct dwc3 交给后续 host/gadget/otg 模块使用
```

当前阶段重点关注这些成员：

```c
struct device       *dev;
void __iomem        *regs;
int                 irq_gadget;
enum usb_dr_mode    dr_mode;
enum usb_device_speed maximum_speed;
struct dwc3_event_buffer *ev_buf;
u32                 revision;
struct dwc3_hwparams hwparams;
```

含义大致是：

```text
dev             Linux device 对象
regs            DWC3 寄存器虚拟地址 base
irq_gadget      DWC3 中断号
dr_mode         host / peripheral / otg
maximum_speed   最大 USB 速度限制
ev_buf          event buffer
revision        DWC3 IP 版本
hwparams        DWC3 硬件能力参数
```

---

## 4. DTS/platform resource 和 `core.c` 的对应关系

设备树里常见属性：

```dts
usb@xxx {
    compatible = "snps,dwc3";
    reg = <...>;
    interrupts = <...>;
    dr_mode = "peripheral";
    maximum-speed = "high-speed";
};
```

对应到 `core.c`：

```text
reg
    ↓
platform_get_resource() / devm_platform_ioremap_resource()
    ↓
dwc->regs
    ↓
dwc3_readl() / dwc3_writel()

interrupts
    ↓
platform_get_irq()
    ↓
dwc->irq_xxx

dr_mode
    ↓
usb_get_dr_mode(dev)
    ↓
dwc->dr_mode

maximum-speed
    ↓
usb_get_maximum_speed(dev)
    ↓
dwc->maximum_speed
```

---

## 5. `reg` 的作用

DTS 里的：

```dts
reg = <...>;
```

描述 DWC3 controller 的寄存器物理地址范围。

`core.c` 中会把它映射成 CPU 可访问的虚拟地址：

```c
dwc->regs = devm_platform_ioremap_resource(pdev, 0);
```

之后访问寄存器时会用：

```c
dwc3_readl(dwc->regs, OFFSET);
dwc3_writel(dwc->regs, OFFSET, value);
```

所以可以理解为：

```text
DTS reg
    ↓
物理寄存器地址
    ↓
ioremap
    ↓
dwc->regs
    ↓
DWC3 寄存器读写 base
```

---

## 6. `interrupts` 的作用

DTS 里的：

```dts
interrupts = <...>;
```

描述 DWC3 controller 的中断号。

`core.c` 中通过 platform API 获取：

```c
irq = platform_get_irq(pdev, 0);
```

后续会用于 DWC3 event interrupt。

DWC3 的事件机制是：

```text
硬件产生事件
    ↓
硬件把 event 写入 event buffer
    ↓
触发 IRQ
    ↓
CPU 进入中断处理
    ↓
软件从 event buffer 解析具体事件
```

---

## 7. `dr_mode` 的作用

DTS 里的：

```dts
dr_mode = "peripheral";
```

常见取值：

```dts
dr_mode = "host";
dr_mode = "peripheral";
dr_mode = "otg";
```

`core.c` 中一般在 `dwc3_get_properties()` 里获取：

```c
dwc->dr_mode = usb_get_dr_mode(dev);
```

然后在 `dwc3_core_init_mode()` 里使用：

```c
switch (dwc->dr_mode) {
case USB_DR_MODE_PERIPHERAL:
    dwc3_gadget_init(dwc);
    break;

case USB_DR_MODE_HOST:
    dwc3_host_init(dwc);
    break;

case USB_DR_MODE_OTG:
    dwc3_drd_init(dwc);
    break;
}
```

所以：

```text
dr_mode 决定 DWC3 初始化后进入哪个 USB 角色
```

对于 device 设备：

```dts
dr_mode = "peripheral";
```

对应：

```text
dwc3_core_init_mode()
    ↓
dwc3_gadget_init()
```

---

## 8. `maximum-speed` 的作用

DTS 里的：

```dts
maximum-speed = "high-speed";
```

或者：

```dts
maximum-speed = "super-speed";
```

`core.c` 中通过：

```c
dwc->maximum_speed = usb_get_maximum_speed(dev);
```

获取。

它表示软件允许该控制器使用的最大 USB 速度。

例如：

```dts
maximum-speed = "high-speed";
```

表示即使硬件支持 USB3，也只允许最高工作到 USB2 high-speed。

常见取值：

```text
"low-speed"
"full-speed"
"high-speed"
"super-speed"
"super-speed-plus"
```

---

## 9. 从寄存器读取硬件信息

`core.c` 不只是从 DTS 读取软件配置，还会通过：

```c
dwc3_readl(dwc->regs, OFFSET);
```

从 DWC3 global registers 读取硬件自身信息。

常见寄存器：

```c
DWC3_GSNPSID
DWC3_GHWPARAMS0
DWC3_GHWPARAMS1
...
DWC3_GHWPARAMS8
```

这些寄存器提供：

```text
DWC3 IP 版本
硬件配置参数
endpoint 数量
FIFO/RAM 能力
支持的硬件特性
```

典型流程：

```text
dwc->regs + DWC3_GSNPSID
    ↓
读取 IP ID / revision
    ↓
保存到 dwc->revision

dwc->regs + DWC3_GHWPARAMSx
    ↓
读取硬件能力
    ↓
保存到 dwc->hwparams
```

注意区分：

```text
DTS 属性：软件/平台配置
硬件寄存器：DWC3 IP 自身能力
```

---

## 10. core reset 和 global register 配置

`core.c` 不只是读取信息，还会真正初始化 DWC3 硬件。

主要包括：

```text
DWC3 core soft reset
配置 global control registers
根据 revision 处理一些 workaround
设置某些全局控制 bit
```

常见相关寄存器：

```c
DWC3_GCTL
DWC3_DCTL
```

可以理解为：

```text
让 DWC3 core 从 reset/未知状态进入可工作的基础状态
```

如果平台不需要软件配置 PHY，那么 PHY 相关代码第一轮可以暂时跳过，但 core reset 和 global 配置仍然是 `core.c` 的重点。

---

## 11. event buffer 的作用

event buffer 是 DWC3 事件机制的核心。

DWC3 硬件发生事件时，不是单纯靠一个状态寄存器告诉软件，而是：

```text
把 event 数据 DMA 写入 event buffer
然后触发中断
```

所以 `core.c` 需要：

```text
申请 event buffer
得到 CPU virtual address 和 DMA address
把 DMA address 写给 DWC3 硬件
```

概念流程：

```text
dma_alloc_coherent()
    ↓
evt->buf      CPU 访问地址
evt->dma      硬件 DMA 地址
    ↓
写入 DWC3 event buffer 地址寄存器
    ↓
DWC3 后续将事件写入这块 buffer
```

相关寄存器通常包括：

```c
DWC3_GEVNTADRLO
DWC3_GEVNTADRHI
DWC3_GEVNTSIZ
DWC3_GEVNTCOUNT
```

---

## 12. event buffer 和 interrupt 的关系

DWC3 event 流程：

```text
USB 事件发生
    比如 reset/connect/disconnect/endpoint event
        ↓
DWC3 硬件生成 event
        ↓
硬件 DMA 写 event buffer
        ↓
硬件触发 interrupt
        ↓
软件中断处理函数运行
        ↓
软件读取 event buffer
        ↓
解析 event 类型
        ↓
分发给 gadget/host/otg 具体逻辑
```

所以：

```text
interrupts 提供 IRQ
event buffer 提供事件内容存储区
两者一起构成 DWC3 事件处理机制
```

---

## 13. `dwc3_core_init_mode()`

core 初始化完成后，最后要根据 `dr_mode` 进入具体模式。

逻辑是：

```text
dwc3_core_init_mode()
    ↓
检查 dwc->dr_mode
    ↓
host       -> dwc3_host_init()
peripheral -> dwc3_gadget_init()
otg        -> dwc3_drd_init()
```

对于当前 device-only 设备，重点是：

```text
dr_mode = peripheral
    ↓
dwc3_gadget_init()
```

第一轮看 `core.c` 时，看到这里即可，暂时不展开 `gadget.c`。

---

## 14. 当前阶段可以先跳过的内容

由于你们的软件不需要配置 PHY，DTS 也比较简单，第一轮看 `core.c` 可以暂时不深入：

```text
PHY 初始化细节
ULPI
复杂 power management
runtime PM
debugfs
各种 quirk 的具体含义
OTG/DRD role switch
不同 DWC3 revision workaround 细节
endpoint/TRB 细节
```

但要知道它们属于：

```text
平台适配
电源管理
兼容性处理
高级功能
```

---

## 15. 对当前理解的总结

目前对 `core.c` 的理解可以总结为：

```text
core.c 是 DWC3 控制器的公共初始化代码。

它在 probe 中：
1. 分配并初始化 struct dwc3
2. 从 DTS/platform resource 获取 reg、irq
3. ioremap DWC3 寄存器空间
4. 从 device property 获取 dr_mode、maximum-speed 等配置
5. 从 DWC3 寄存器读取 revision、hwparams 等硬件信息
6. 对 DWC3 core 做 soft reset 和 global register 配置
7. 分配并配置 event buffer
8. 根据 dr_mode 调用 host/gadget/otg 初始化函数
```

一句话：

```text
core.c 负责把 DWC3 硬件 core 初始化好，并根据 dr_mode 把控制权交给对应的角色模块。
```