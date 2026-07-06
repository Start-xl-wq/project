# DWC3 device/gadget 

如果你现在开始学 **DWC3 device 模式**，建议不要一上来就看 `ep0.c` 或 TRB 细节，而是按下面顺序看。

核心顺序：

```text
1. drivers/usb/dwc3/core.c
2. drivers/usb/dwc3/gadget.c
3. drivers/usb/dwc3/ep0.c
4. drivers/usb/dwc3/core.h / gadget.h
5. drivers/usb/gadget/composite.c
6. drivers/usb/gadget/configfs.c
7. drivers/usb/gadget/function/
```

对于 DWC3 自身来说，最重要的是：

```text
drivers/usb/dwc3/core.c
drivers/usb/dwc3/gadget.c
drivers/usb/dwc3/ep0.c
```

---

# 1. 先看 `drivers/usb/dwc3/core.c`

## 这个文件在做什么

`core.c` 是 DWC3 控制器的总入口。

它负责把 DWC3 这个硬件 IP 初始化起来，然后根据 `dr_mode` 决定当前工作在：

```text
host
device/peripheral
otg/drd
```

对于 device 学习来说，`core.c` 的作用是：

```text
先把 DWC3 硬件初始化好
然后调用 dwc3_gadget_init()
把 DWC3 注册成一个 UDC
```

---

## 重点看什么函数

优先看：

```c
dwc3_probe()
dwc3_get_properties()
dwc3_core_init()
dwc3_core_soft_reset()
dwc3_phy_setup()
dwc3_event_buffers_setup()
dwc3_core_init_mode()
dwc3_remove()
```

device 相关重点看：

```c
dwc3_core_init_mode()
```

里面会根据模式调用：

```c
dwc3_gadget_init(dwc);
```

---

## 大致流程

```text
dwc3_probe()
    ↓
解析 DTS 属性
    ↓
获取寄存器、IRQ、PHY、clock、reset 等资源
    ↓
读取 DWC3 IP 版本和硬件参数
    ↓
dwc3_core_init()
    ↓
DWC3 core soft reset
    ↓
PHY 初始化
    ↓
event buffer 初始化
    ↓
dwc3_core_init_mode()
    ↓
如果是 peripheral/device 模式
    ↓
dwc3_gadget_init()
```

---

## 你看 `core.c` 时要抓住什么

你不需要一开始把所有 global register 都看懂。

第一遍只需要搞清楚：

```text
DWC3 是怎么 probe 的
DWC3 是怎么读取 DTS 的
DWC3 是怎么判断 device/host/otg 模式的
DWC3 是什么时候进入 gadget 初始化的
DWC3 的 IRQ/event buffer 是怎么准备的
```

重点关注这些概念：

- `dr_mode`
- `maximum-speed`
- `usb2-phy`
- `usb3-phy`
- `event buffer`
- `GSNPSID`
- `GHWPARAMS`
- `revision`
- `dwc3_core_init_mode()`

---

# 2. 第二个看 `drivers/usb/dwc3/gadget.c`

## 这个文件在做什么

`gadget.c` 是 DWC3 的 **UDC driver 主体**。

也就是说：

```text
DWC3 device 模式的大部分逻辑都在 gadget.c
```

它负责把 Linux gadget framework 的动作，转换成 DWC3 硬件操作。

比如：

```text
gadget framework 要 enable endpoint
    ↓
gadget.c 发送 DWC3 endpoint command

gadget framework 要 queue usb_request
    ↓
gadget.c 准备 TRB

DWC3 硬件产生 event
    ↓
gadget.c 解析 event
    ↓
回调 request complete
```

---

## 这个文件最重要

如果你只问 DWC3 device 应该重点看哪个文件：

```text
drivers/usb/dwc3/gadget.c
```

它就是 DWC3 device 模式主线。

---

## 重点看什么函数

### 初始化相关

```c
dwc3_gadget_init()
dwc3_gadget_free_endpoints()
dwc3_gadget_init_endpoints()
dwc3_gadget_start()
dwc3_gadget_stop()
```

它们负责：

```text
创建 gadget 设备
初始化 endpoint 对象
注册 UDC
和 gadget framework 建立连接
```

---

### pullup/connect 相关

```c
dwc3_gadget_pullup()
dwc3_gadget_run_stop()
```

它们负责：

```text
控制 device 是否对 host 可见
```

简单理解：

```text
pullup = 让 PC/Host 看到这个 USB device
```

如果 gadget 没有 pullup，PC 一般不会开始枚举。

---

### endpoint 相关

```c
dwc3_gadget_ep_enable()
dwc3_gadget_ep_disable()
dwc3_gadget_ep_queue()
dwc3_gadget_ep_dequeue()
```

它们对应 gadget framework 的 endpoint 操作。

例如 function driver 调用：

```c
usb_ep_enable()
usb_ep_queue()
```

最后会走到 DWC3 的：

```c
dwc3_gadget_ep_enable()
dwc3_gadget_ep_queue()
```

---

### TRB/传输相关

```c
dwc3_prepare_trbs()
dwc3_prepare_one_trb()
dwc3_send_gadget_ep_cmd()
dwc3_gadget_start_isoc_quirk()
```

它们负责：

```text
把 usb_request 转换成 DWC3 硬件认识的 TRB
然后通过 endpoint command 启动传输
```

---

### event/中断相关

```c
dwc3_gadget_interrupt()
dwc3_process_event_buf()
dwc3_process_event_entry()
dwc3_gadget_event_handler()
dwc3_endpoint_interrupt()
dwc3_endpoint_transfer_complete()
```

它们负责：

```text
处理 DWC3 硬件事件
比如 reset、connect、disconnect、setup packet、transfer complete
```

---

## `gadget.c` 第一遍怎么看

不要第一遍就钻每个 TRB bit。

建议先按这个主线看：

```text
dwc3_gadget_init()
    ↓
DWC3 怎么注册成 UDC

dwc3_gadget_pullup()
    ↓
DWC3 怎么连接到 host

dwc3_gadget_ep_enable()
    ↓
endpoint 怎么被 enable

dwc3_gadget_ep_queue()
    ↓
usb_request 怎么提交给 DWC3

dwc3_gadget_interrupt()
    ↓
DWC3 event 怎么被处理

dwc3_endpoint_transfer_complete()
    ↓
传输完成后怎么回调上层
```

---

# 3. 第三个看 `drivers/usb/dwc3/ep0.c`

## 这个文件在做什么

`ep0.c` 专门处理 **endpoint 0**。

EP0 是 USB 枚举的核心。

所有 USB device 枚举一开始都通过 EP0 进行，例如：

```text
GET_DESCRIPTOR
SET_ADDRESS
SET_CONFIGURATION
GET_CONFIGURATION
GET_STATUS
SET_FEATURE
CLEAR_FEATURE
```

所以：

```text
ep0.c = DWC3 device 模式枚举控制通道
```

---

## 为什么不要一开始就看 `ep0.c`

因为 `ep0.c` 依赖你先理解：

```text
DWC3 怎么注册 UDC
DWC3 怎么处理 event
DWC3 endpoint command 是什么
DWC3 usb_request 是怎么提交的
```

这些主要在 `gadget.c` 里。

所以顺序应该是：

```text
先 gadget.c 主框架
再 ep0.c 枚举细节
```

---

## 重点看什么函数

```c
dwc3_ep0_interrupt()
dwc3_ep0_handle_setup()
dwc3_ep0_delegate_req()
dwc3_ep0_set_config()
dwc3_ep0_set_address()
dwc3_ep0_start_trans()
dwc3_ep0_end_control_data()
```

---

## 大致流程

```text
PC/Host 发 SETUP packet
    ↓
DWC3 硬件产生 EP0 event
    ↓
dwc3_gadget_interrupt()
    ↓
dwc3_process_event_entry()
    ↓
发现是 EP0 event
    ↓
dwc3_ep0_interrupt()
    ↓
dwc3_ep0_handle_setup()
    ↓
判断 setup request 类型
    ↓
标准请求自己处理一部分
    ↓
其他请求交给 gadget/composite/function
```

---

## EP0 和 composite 的关系

很多标准请求或 class/vendor 请求，不是 DWC3 自己最终处理，而是交给 gadget framework。

关系大概是：

```text
DWC3 ep0.c
    ↓
gadget driver setup callback
    ↓
composite.c
    ↓
function driver
```

例如：

```text
Host 发 GET_DESCRIPTOR
    ↓
DWC3 ep0 收到 SETUP
    ↓
交给 composite.c
    ↓
composite 返回 device/config/interface/endpoint descriptor
    ↓
DWC3 ep0 把 descriptor 数据发回 host
```

---

# 4. 同时配合看 `drivers/usb/dwc3/core.h`

## 这个文件在做什么

`core.h` 是 DWC3 内部核心数据结构和寄存器定义。

你看 `gadget.c` 和 `ep0.c` 时会经常遇到：

```c
struct dwc3
struct dwc3_ep
struct dwc3_request
struct dwc3_event_buffer
struct dwc3_trb
```

这些结构体大多在 `core.h` 里。

---

## 重点结构体

```c
struct dwc3
```

表示整个 DWC3 控制器。

里面有：

```text
寄存器基地址
IRQ
PHY
event buffer
gadget 对象
endpoint 数组
当前 speed
当前 role/mode
quirks
```

---

```c
struct dwc3_ep
```

表示一个 DWC3 endpoint。

里面有：

```text
endpoint number
endpoint name
endpoint direction
TRB ring
request list
endpoint flags
resource index
```

---

```c
struct dwc3_request
```

表示 DWC3 内部封装后的 usb_request。

关系是：

```text
struct usb_request
    ↓
struct dwc3_request
    ↓
TRB
    ↓
DWC3 hardware
```

---

```c
struct dwc3_trb
```

表示 DWC3 硬件传输描述符。

DWC3 不是直接理解 `usb_request`，而是理解 TRB。

---

## 第一遍重点

第一遍不用背寄存器宏。

优先搞清楚这些结构关系：

```text
struct dwc3       = 一个 DWC3 控制器
struct dwc3_ep    = 一个 endpoint
struct usb_ep     = gadget 框架看到的 endpoint
struct usb_request = gadget 框架提交的数据请求
struct dwc3_request = DWC3 内部封装的 request
struct dwc3_trb   = 硬件真正执行的传输描述符
```

---

# 5. 再看 `drivers/usb/gadget/composite.c`

## 这个文件在做什么

`composite.c` 不属于 DWC3，但 device 学习必须看。

它是 Linux USB gadget 的通用组合设备框架。

它负责：

```text
把多个 USB function 组合成一个 USB device
处理 descriptor
处理 configuration
处理 interface
处理 SET_CONFIGURATION
处理 function bind/enable/disable
```

例如一个设备可以同时有：

```text
ADB
MTP
RNDIS
ACM
HID
Mass Storage
```

这些 function 最后通过 composite 框架组合成一个 USB 设备。

---

## DWC3 和 composite 的关系

```text
DWC3 gadget.c
    ↓
提供 UDC 硬件能力

composite.c
    ↓
提供 USB device 逻辑和 descriptor

function driver
    ↓
提供具体功能，比如 adb/rndis/mass_storage
```

DWC3 不关心自己是 ADB 还是 U 盘。

DWC3 只负责：

```text
收 SETUP
发 descriptor 数据
enable endpoint
queue request
传输完成回调
```

具体 descriptor 和功能由 composite/function 提供。

---

# 6. 再看 `drivers/usb/gadget/configfs.c`

## 这个文件在做什么

`configfs.c` 负责通过 `/sys/kernel/config/usb_gadget/` 动态创建 USB gadget。

也就是常见的这种操作：

```bash
mkdir /sys/kernel/config/usb_gadget/g1
cd /sys/kernel/config/usb_gadget/g1

echo 0x18d1 > idVendor
echo 0x4ee7 > idProduct

mkdir strings/0x409
echo "0123456789" > strings/0x409/serialnumber
echo "Vendor" > strings/0x409/manufacturer
echo "Device" > strings/0x409/product

mkdir configs/c.1
mkdir functions/ffs.adb
ln -s functions/ffs.adb configs/c.1/

echo <udc_name> > UDC
```

最后这句：

```bash
echo <udc_name> > UDC
```

就是把 gadget 绑定到 DWC3 UDC 上。

---

## 这个文件和 DWC3 的关系

```text
configfs 创建 gadget
    ↓
composite 组织 descriptor/function
    ↓
echo UDC 触发 bind
    ↓
DWC3 gadget driver 开始工作
```

所以如果你调试 device 模式，经常要同时看：

```text
/sys/kernel/config/usb_gadget/
和
/sys/class/udc/
```

---

# 7. 最后看 `drivers/usb/gadget/function/`

## 这个目录在做什么

这个目录下是各种 USB gadget function。

例如：

```text
f_mass_storage.c   # U 盘
f_acm.c            # 串口
f_ecm.c            # USB 网卡 ECM
f_rndis.c          # USB 网卡 RNDIS
f_hid.c            # HID
f_fs.c             # FunctionFS，ADB 常用
f_uac1.c / f_uac2.c # USB Audio
f_uvc.c            # USB Camera
```

这些 function 负责具体 USB 功能。

DWC3 不知道自己传的是 ADB、RNDIS 还是 U 盘数据。
DWC3 只知道 endpoint 上有 request 要传。

---

# 推荐第一轮阅读路线

第一轮不要追求全部细节，看主线即可。

```text
drivers/usb/dwc3/core.c
    ↓
看 dwc3_probe()
看 dwc3_core_init_mode()
知道什么时候进 gadget

drivers/usb/dwc3/gadget.c
    ↓
看 dwc3_gadget_init()
看 dwc3_gadget_pullup()
看 dwc3_gadget_ep_enable()
看 dwc3_gadget_ep_queue()
看 dwc3_gadget_interrupt()

drivers/usb/dwc3/ep0.c
    ↓
看 dwc3_ep0_interrupt()
看 dwc3_ep0_handle_setup()
知道 SETUP 包怎么处理

drivers/usb/gadget/composite.c
    ↓
看 setup callback
看 descriptor 怎么返回
看 SET_CONFIGURATION 怎么 enable function

drivers/usb/gadget/configfs.c
    ↓
知道 echo UDC 后发生了什么
```

---

# device 模式主线调用关系

## 初始化阶段

```text
dwc3_probe()
    ↓
dwc3_core_init()
    ↓
dwc3_core_init_mode()
    ↓
dwc3_gadget_init()
    ↓
dwc3_gadget_init_endpoints()
    ↓
usb_add_gadget_udc()
    ↓
/sys/class/udc/ 出现 DWC3 UDC
```

---

## gadget 绑定阶段

```text
用户创建 configfs gadget
    ↓
echo <udc_name> > UDC
    ↓
gadget bind 到 DWC3 UDC
    ↓
dwc3_gadget_start()
    ↓
等待 pullup/connect
```

---

## 连接枚举阶段

```text
PC/Host 连接
    ↓
dwc3_gadget_pullup()
    ↓
dwc3_gadget_run_stop()
    ↓
DWC3 连接到 bus
    ↓
Host 发 USB reset
    ↓
DWC3 产生 reset event
    ↓
Host 发 SETUP packet
    ↓
dwc3_ep0_interrupt()
    ↓
dwc3_ep0_handle_setup()
    ↓
composite 返回 descriptor
    ↓
SET_ADDRESS
    ↓
SET_CONFIGURATION
    ↓
function enable
    ↓
非 EP0 endpoint enable
```

---

## 数据传输阶段

```text
function driver
    ↓
usb_ep_queue()
    ↓
dwc3_gadget_ep_queue()
    ↓
dwc3_prepare_trbs()
    ↓
dwc3_send_gadget_ep_cmd()
    ↓
DWC3 硬件传输
    ↓
DWC3 event
    ↓
dwc3_endpoint_interrupt()
    ↓
dwc3_endpoint_transfer_complete()
    ↓
usb_request complete callback
    ↓
function driver 收到完成通知
```

---

# 最建议你先啃的 3 个函数

如果你刚开始，不要同时看太多。

先看这三个：

```c
dwc3_probe()
dwc3_gadget_init()
dwc3_gadget_ep_queue()
```

它们分别代表：

```text
DWC3 控制器怎么起来
DWC3 怎么注册成 UDC
USB request 怎么真正交给硬件传输
```

然后再看：

```c
dwc3_ep0_handle_setup()
dwc3_endpoint_transfer_complete()
```

它们分别代表：

```text
枚举请求怎么处理
传输完成怎么回调上层
```

---

# 一句话总结

开始学 DWC3 device，文件顺序建议是：

```text
core.c      看 DWC3 怎么初始化、怎么进入 device 模式
gadget.c    看 DWC3 作为 UDC 怎么工作，这是 device 主文件
ep0.c       看 USB 枚举和控制传输
core.h      看 DWC3 内部结构体和 TRB/event 定义
composite.c 看 descriptor/function/configuration 怎么组织
configfs.c  看用户如何创建并绑定 gadget
function/   看具体 ADB/U盘/RNDIS/HID 等功能
```

其中最核心的是：

```text
drivers/usb/dwc3/gadget.c
```

它就是 DWC3 device/UDC 的主战场。