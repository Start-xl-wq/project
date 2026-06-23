# DWC3/USB 学习路线

## 1. 定位

重点不是重写 USB 协议栈，而是让 DWC3 USB 控制器在本 SoC 上正确工作。

日常重点：

- DTS / binding
- SoC glue driver
- clock / reset / power domain
- regulator
- USB2 PHY / USB3 PHY
- Type-C / role switch / VBUS / ID
- runtime PM / system suspend / wakeup
- SoC wrapper register
- quirks / errata workaround

通用层一般不优先修改，但需要理解：

- DWC3 core
- DWC3 gadget
- DWC3 ep0
- xHCI
- USB core
- gadget framework

---

## 2. Linux USB/DWC3 分层

```text
SoC glue / wrapper
    ↓
DWC3 core
    ↓
host mode                 device/gadget mode
    ↓                          ↓
xHCI                      DWC3 gadget / ep0
    ↓                          ↓
USB core                  gadget framework
    ↓                          ↓
class driver              function driver
```

硬件视角：

```text
connector / Type-C / VBUS / ID
    ↓
USB2 PHY / USB3 PHY
    ↓
SoC wrapper
    ↓
DWC3 controller
    ↓
DMA / interrupt / memory
```

---

## 3. 推荐学习顺序

整体路线：

```text
1. 先看本 SoC 的 DTS + glue driver
2. 看 DWC3 core probe/init/mode 选择
3. 学 USB2.0 基本枚举流程
4. 学 device/gadget EP0 枚举
5. 学 host/xHCI root hub 和枚举路径
6. 学 DWC3 gadget 非 EP0 传输和 event/TRB
7. 学 xHCI URB/TRB/event ring
8. 学 USB3 PHY/link/Type-C/role switch
9. 学 PM/wakeup/suspend/resume
```

一句话：

```text
先 glue + core，
再 USB2 枚举，
再 device/host 两条路径，
最后 USB3、Type-C、PM。
```

---

## 4. 工作收益优先级

对原厂适配最有价值的顺序：

```text
DTS/glue/PHY/clock/reset/power
    ↓
dr_mode/role/VBUS
    ↓
DWC3 core probe/init
    ↓
USB2 枚举问题定位
    ↓
USB3/Type-C 问题定位
    ↓
xHCI/gadget 传输细节
```

精力分配建议：

```text
SoC 集成层：50%
DWC3 core：20%
device/gadget/EP0：15%
host/xHCI：15%
```

如果项目偏 device，增加 gadget/EP0 比例。
如果项目偏 host，增加 xHCI/host 比例。

---

## 5. 第一阶段：SoC 集成层

这是原厂驱动工程师最常修改的部分，应该最先看。

重点内容：

- compatible 怎么匹配
- reg 资源怎么获取
- interrupts 怎么配置
- clock 怎么 enable
- reset 怎么 deassert
- power domain 怎么打开
- regulator 怎么控制
- USB2/USB3 PHY 怎么关联
- dr_mode 怎么决定 host/device/otg
- maximum-speed 怎么限制速度
- Type-C / role switch 怎么接入
- VBUS 由谁控制
- wrapper register 需要配置什么
- runtime PM 怎么处理
- suspend/resume 怎么恢复
- 是否需要 SoC-specific quirk

常见 DTS 结构：

```dts
usb_wrapper: usb@xxxx0000 {
    compatible = "vendor,soc-dwc3";
    reg = <...>;
    clocks = <...>;
    resets = <...>;
    power-domains = <...>;

    dwc3: usb@yyyy0000 {
        compatible = "snps,dwc3";
        reg = <...>;
        interrupts = <...>;

        dr_mode = "otg";
        maximum-speed = "super-speed";

        phys = <&usb2phy>, <&usb3phy>;
        phy-names = "usb2-phy", "usb3-phy";

        snps,dis_u2_susphy_quirk;
        snps,dis_u3_susphy_quirk;
    };
};
```

看 DTS 时要能回答：

- DWC3 core 的寄存器基地址在哪里？
- DWC3 IRQ 是哪个？
- USB2 PHY 是哪个？
- USB3 PHY 是哪个？
- 当前是 host、peripheral 还是 otg？
- maximum-speed 是什么？
- VBUS 由谁控制？
- Type-C controller 是谁？
- role switch 从哪里来？
- clock/reset/power domain 是否完整？
- 有没有 wrapper register？
- 有没有 SoC-specific quirk？

---

## 6. 第二阶段：DWC3 core 初始化

重点文件：

```text
drivers/usb/dwc3/core.c
drivers/usb/dwc3/core.h
```

重点函数：

```c
dwc3_probe()
dwc3_get_properties()
dwc3_core_init()
dwc3_core_soft_reset()
dwc3_phy_setup()
dwc3_event_buffers_setup()
dwc3_core_init_mode()
dwc3_suspend()
dwc3_resume()
```

初始化流程：

```text
dwc3_probe()
    ↓
解析 DTS / platform data
    ↓
获取 resource / irq
    ↓
获取 USB2 PHY / USB3 PHY
    ↓
读取 DWC3 revision / GHWPARAMS
    ↓
core soft reset
    ↓
PHY setup
    ↓
配置 global register
    ↓
分配 event buffer
    ↓
根据 dr_mode 初始化 host/gadget/drd
    ↓
注册 PM
```

这一阶段要理解：

- `dr_mode`
- `maximum-speed`
- DWC3 revision
- `GSNPSID`
- `GHWPARAMS`
- event buffer
- global register
- core soft reset
- USB2 PHY / USB3 PHY
- quirks
- runtime PM
- system suspend/resume

---

## 7. 第三阶段：USB2.0 基础枚举

先学 USB2.0，不要一开始就钻 USB3.0。

USB2 是理解 USB 枚举和基本传输的基础。

需要掌握：

- VBUS
- D+ / D-
- pull-up / pull-down
- full-speed / high-speed
- chirp
- reset
- setup packet
- control transfer
- device descriptor
- configuration descriptor
- interface descriptor
- endpoint descriptor
- SET_ADDRESS
- SET_CONFIGURATION
- bulk / interrupt / isochronous endpoint

Host 视角枚举流程：

```text
设备插入
    ↓
host 检测 connect
    ↓
host reset port
    ↓
获取 device descriptor
    ↓
set address
    ↓
获取 configuration descriptor
    ↓
set configuration
    ↓
匹配 interface/class driver
    ↓
设备可用
```

Device/Gadget 视角枚举流程：

```text
PC host reset
    ↓
DWC3 收到 reset/connect done
    ↓
EP0 收到 SETUP packet
    ↓
gadget/composite 返回 descriptor
    ↓
处理 SET_ADDRESS
    ↓
处理 SET_CONFIGURATION
    ↓
function enable
    ↓
非 EP0 endpoint enable
    ↓
设备可用
```

---

## 8. 第四阶段：Device/Gadget 路径

建议先看 device/gadget 的 EP0 枚举，因为更容易理解 USB 协议本质。

学习顺序：

```text
configfs gadget
    ↓
composite framework
    ↓
UDC bind
    ↓
DWC3 gadget init
    ↓
EP0 枚举
    ↓
非 EP0 endpoint 传输
```

重点文件：

```text
drivers/usb/dwc3/gadget.c
drivers/usb/dwc3/ep0.c
drivers/usb/gadget/composite.c
drivers/usb/gadget/configfs.c
drivers/usb/gadget/function/
```

重点函数：

```c
dwc3_gadget_init()
dwc3_gadget_start()
dwc3_gadget_pullup()
dwc3_gadget_run_stop()
dwc3_ep0_interrupt()
dwc3_ep0_handle_setup()
dwc3_gadget_ep_enable()
dwc3_gadget_ep_queue()
dwc3_prepare_trbs()
dwc3_send_gadget_ep_cmd()
dwc3_endpoint_transfer_complete()
```

关键调用链：

```text
gadget function/configfs 创建
    ↓
gadget bind 到 UDC
    ↓
PC 插入
    ↓
DWC3 connect
    ↓
host reset
    ↓
EP0 收到 SETUP
    ↓
composite 返回 descriptor
    ↓
SET_ADDRESS
    ↓
SET_CONFIGURATION
    ↓
bulk/interrupt/iso endpoint enable
    ↓
usb_request queue
    ↓
DWC3 TRB
    ↓
endpoint event
    ↓
request complete
```

核心概念：

- UDC
- gadget driver
- function driver
- configfs
- composite framework
- EP0
- setup packet
- usb_request
- endpoint
- TRB
- event buffer
- pullup
- connect/disconnect

---

## 9. 第五阶段：Host/xHCI 路径

DWC3 在 host 模式下通常交给 xHCI 处理传输。

关系：

```text
DWC3 host.c
    ↓
创建/注册 xHCI platform device
    ↓
xhci-plat.c
    ↓
xhci.c / xhci-ring.c / xhci-mem.c
```

学习顺序：

```text
DWC3 host init
    ↓
xHCI platform probe
    ↓
HCD 注册
    ↓
root hub 注册
    ↓
port connect
    ↓
USB core 枚举
    ↓
class driver bind
    ↓
URB 传输
```

重点文件：

```text
drivers/usb/dwc3/host.c
drivers/usb/host/xhci-plat.c
drivers/usb/host/xhci.c
drivers/usb/host/xhci-ring.c
drivers/usb/host/xhci-mem.c
drivers/usb/core/hub.c
drivers/usb/core/message.c
drivers/usb/core/driver.c
```

重点函数：

```c
dwc3_host_init()
xhci_plat_probe()
xhci_gen_setup()
xhci_run()
usb_add_hcd()
hub_event()
usb_new_device()
usb_get_device_descriptor()
usb_set_configuration()
usb_probe_interface()
usb_submit_urb()
xhci_urb_enqueue()
```

关键调用链：

```text
DWC3 切到 host
    ↓
dwc3_host_init()
    ↓
创建 xHCI platform device
    ↓
xhci_plat_probe()
    ↓
usb_add_hcd()
    ↓
root hub 出现
    ↓
hub_event()
    ↓
检测 port connect
    ↓
port reset
    ↓
usb_new_device()
    ↓
GET_DESCRIPTOR
    ↓
SET_ADDRESS
    ↓
SET_CONFIGURATION
    ↓
usb_probe_interface()
    ↓
class driver bind
    ↓
usb_submit_urb()
    ↓
xhci_urb_enqueue()
    ↓
queue TRB
    ↓
ring doorbell
    ↓
event ring
    ↓
urb complete
```

核心概念：

- HCD
- root hub
- hub driver
- port status
- USB device
- USB interface
- USB class driver
- URB
- endpoint
- xHCI command ring
- xHCI transfer ring
- xHCI event ring
- doorbell
- TRB

---

## 10. 第六阶段：USB3.0 / Type-C / PHY

USB3.0 在 USB2 基础上增加了链路层和 PHY 复杂度。

重点概念：

- SuperSpeed
- USB3 PHY
- PIPE
- LFPS
- link training
- U0 / U1 / U2 / U3
- SuperSpeed descriptor
- endpoint companion descriptor
- LPM
- equalization
- de-emphasis
- lane polarity
- lane swap

常见问题：

- 只能识别成 high-speed，不能 super-speed
- USB3 插入无反应，但 USB2 正常
- Type-C 一个方向正常，另一个方向不正常
- SuperSpeed link training 失败
- suspend/resume 后 SS 不恢复
- USB3 速率不稳定

常见原因：

- USB3 PHY 没 ready
- PIPE clock 问题
- Type-C orientation mux 问题
- lane swap / polarity 问题
- PHY tuning 问题
- maximum-speed 配置错误
- GUSB3PIPECTL 配置问题
- 电源/时钟/复位时序问题
- 板级 SI 问题

Type-C/role switch 路径：

```text
Type-C controller / TCPC / PD
    ↓
role/orientation/vbus event
    ↓
usb_role_switch_set_role()
    ↓
DWC3 DRD role switch
    ↓
dwc3_set_mode()
    ↓
host/gadget start/stop
```

---

## 11. 第七阶段：PM / wakeup / suspend/resume

USB 量产项目中，PM 问题很常见。

常见场景：

- runtime suspend 后插拔不识别
- system suspend 后无法唤醒
- resume 后 USB2/USB3 不恢复
- device 模式 remote wakeup 不工作
- host 模式外设唤醒不工作
- Type-C attach wake 不工作

涉及层次：

- DWC3 core PM
- SoC glue PM
- PHY PM
- Type-C/PD PM
- power domain
- clock gating
- reset state
- wakeup IRQ
- runtime PM
- system suspend/resume

学习重点：

- runtime_suspend / runtime_resume
- suspend / resume
- `wakeup-source`
- `device_may_wakeup()`
- `enable_irq_wake()`
- PHY power_on / power_off
- clock disable / enable 顺序
- power domain 关断条件
- role 切换时 PM 状态

---

## 12. 问题定位思路

### 12.1 Probe 阶段问题

现象：

- dmesg 没有 dwc3
- 没有 xhci
- 没有 UDC
- probe defer

优先查：

- DTS compatible
- reg / interrupts
- clock
- reset
- power-domain
- regulator
- PHY
- driver bind
- EPROBE_DEFER

常用命令：

```bash
dmesg | grep -iE "dwc3|xhci|usb|phy|typec|udc"
ls /sys/kernel/debug/devices_deferred
ls /sys/class/udc/
```

---

### 12.2 Host 模式不识别外设

先查：

- xHCI 有没有注册？
- root hub 有没有？
- VBUS 有没有 5V？
- PHY 有没有 connect event？
- Type-C role 是否 host？
- hub.c 有没有枚举日志？
- 设备类驱动有没有 bind？

常用命令：

```bash
dmesg -w
dmesg | grep -iE "xhci|usb|hub|new high-speed|new SuperSpeed"
lsusb
lsusb -t
cat /sys/kernel/debug/usb/devices
```

判断：

```text
连 root hub 都没有：
    多半是 DWC3 host/xHCI 没起，或 clock/reset/irq/resource 问题。

root hub 有，但插设备没反应：
    多半是 VBUS/PHY/Type-C role/port connect detect 问题。

能枚举但设备功能异常：
    再看 USB core/class driver/xHCI transfer。
```

---

### 12.3 Device/Gadget 模式 PC 不识别

先查：

- UDC 有没有注册？
- gadget function 有没有 bind 到 UDC？
- role 是否 peripheral？
- VBUS 是否 valid？
- DWC3 是否执行 pullup/connect？
- PC host 有没有 reset？
- EP0 setup 包有没有收到？

常用命令：

```bash
ls /sys/class/udc/
ls /sys/kernel/config/usb_gadget/
dmesg | grep -iE "dwc3|gadget|udc|ep0|configfs"
```

判断：

```text
/sys/class/udc/ 没有：
    多半是 DWC3 gadget 没初始化，或者 dr_mode/DTS/PHY/clock/reset 问题。

UDC 有，但 PC 不识别：
    多半查 gadget 是否 bind、VBUS/role、USB2 pullup、Type-C device role、PHY。

PC 发了 setup 但枚举失败：
    再看 ep0.c、composite.c、function descriptor。
```

---

## 13. 修改代码的原则

优先修改：

- DTS
- binding
- SoC glue driver
- PHY driver
- Type-C / role switch 相关
- regulator / clock / reset / power domain
- SoC wrapper register
- quirk

尽量不要一开始就改：

- `drivers/usb/dwc3/core.c`
- `drivers/usb/dwc3/gadget.c`
- `drivers/usb/dwc3/ep0.c`
- `drivers/usb/host/xhci.c`
- `drivers/usb/core/`
- `drivers/usb/gadget/`

原则：

```text
能通过 glue、DTS、PHY、quirk 解决的，不要改通用传输层。
```

只有明确属于以下情况才考虑改通用层：

- DWC3 IP version errata
- 现有 quirk 不覆盖本 SoC
- 上游已有类似 patch，可以复用或扩展
- 明确是 DWC3 core 逻辑和硬件行为不匹配
- DMA coherency / cache / IOMMU 特殊问题
- runtime PM / role switch 通用流程 bug
- xHCI controller 需要新增 quirk

---

## 14. 最终主线

完整学习主线：

```text
SoC 集成层
    ↓
DWC3 core 初始化
    ↓
USB2 枚举
    ↓
device/gadget EP0
    ↓
host/xHCI root hub 和枚举
    ↓
DWC3 gadget TRB/event
    ↓
xHCI URB/TRB/event ring
    ↓
USB3 PHY/link/Type-C
    ↓
PM/wakeup/suspend/resume
```

最终目标：

- 知道各层边界
- 能完成 SoC 适配
- 能判断问题属于 glue、PHY、role、DWC3 core、gadget EP0、xHCI 还是 USB core
- 能在必要时添加合适的 quirk/workaround
- 避免无意义地修改通用层