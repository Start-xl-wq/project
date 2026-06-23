
## 0. 背景定位
 
主要工作不是重新实现 USB 协议栈，而是让 Synopsys DWC3 USB 控制器 IP 在本 SoC 上正确跑起来。

日常重点：

- SoC glue driver
- DTS / binding
- clock / reset / power domain
- regulator
- USB2/USB3 PHY
- Type-C / role switch / VBUS / ID
- runtime PM / system suspend / wakeup
- SoC-specific wrapper register
- quirks / errata workaround

通用层一般不轻易修改，但必须理解：

- `drivers/usb/dwc3/core.c`
- `drivers/usb/dwc3/gadget.c`
- `drivers/usb/dwc3/ep0.c`
- `drivers/usb/dwc3/host.c`
- `drivers/usb/host/xhci*.c`
- `drivers/usb/core/`
- `drivers/usb/gadget/`

---

# 1. 总体认识

## 1.1 DWC3 是什么

DWC3 是 Synopsys 的 USB3.x Dual-Role Device Controller IP，同时支持 USB2。

Linux 中 DWC3 分为：

```text
SoC glue / wrapper 层
    ↓
DWC3 core
    ↓
host mode                 gadget/device mode
    ↓                          ↓
xHCI                      DWC3 gadget/ep0
    ↓                          ↓
USB core                  gadget framework
    ↓                          ↓
USB class driver          gadget function driver

