# U-Boot USB DFU：下载文件到 RAM

以 AX525 原项目的 DWC3 Device 配置为例，使用 Windows `dfu-util` 向 U-Boot 的 RAM 后端下载文件。本文保留原项目验证参数，未重新上板验证；其他平台需替换控制器驱动、设备身份与可用 RAM 地址。

```text
Windows dfu-util → USB Device / U-Boot DFU → 暂存缓冲 → 指定 RAM 区域
```

此例只写 RAM，不会自动烧写 Flash，也不会自动执行下载的文件。

## 1. U-Boot 配置

AX525 原项目使用下列配置；配置项是否存在，以目标 U-Boot 版本为准。

```text
CONFIG_USB=y
CONFIG_DM_USB=y
CONFIG_USB_GADGET=y
CONFIG_DM_USB_GADGET=y
CONFIG_USB_GADGET_DOWNLOAD=y

CONFIG_USB_GADGET_MANUFACTURER="U-Boot"
CONFIG_USB_GADGET_VENDOR_NUM=0x0525
CONFIG_USB_GADGET_PRODUCT_NUM=0xa4a5
CONFIG_USB_GADGET_VBUS_DRAW=2

CONFIG_USB_DWC3=y
CONFIG_USB_DWC3_GADGET=y
CONFIG_USB_DWC3_AXERA=y

CONFIG_CMD_DFU=y
CONFIG_DFU=y
CONFIG_DFU_OVER_USB=y
CONFIG_DFU_RAM=y
```

其中 `CONFIG_USB_DWC3_AXERA` 是平台 glue，VID/PID 为原项目示例，产品配置使用自身分配的设备身份。DTS 中对应 DWC3 节点必须启用 Device 模式：

```dts
&dwc3 {
    dr_mode = "peripheral";
    status = "okay";
};
```

编译并运行包含这些配置的 U-Boot，在串口执行 `help dfu` 确认命令可用。

## 2. 准备目标区域

以下是原项目验证的 RAM 参数：

| 参数 | 示例值 |
| --- | --- |
| 目标起始地址 | `0x41100000` |
| 目标区域大小 | `0x10000`（64 KiB） |
| DFU 暂存缓冲大小 | `0x10000`（64 KiB） |

操作会覆盖指定 RAM 区域。使用前确认它不与 U-Boot、堆栈、堆或其他有效数据重叠；本例下载文件不得超过 64 KiB。不要把该地址视为其他板卡的通用空闲地址。

## 3. 板端启动 DFU

若已停在旧的 DFU 会话中，先按 `Ctrl+C` 返回 U-Boot 命令行，再逐条执行：

```text
setenv dfu_bufsiz 0x10000
setenv dfu_alt_info "kernel ram 0x41100000 0x10000"
printenv dfu_bufsiz
printenv dfu_alt_info
dfu 0 ram 0
```

`kernel` 只是 alt 的名称，不表示这里能容纳完整内核镜像。`dfu_bufsiz` 是暂存缓冲大小，与 `dfu_alt_info` 中的目标区域大小分别配置；原平台默认缓冲过大时会分配失败。

**预期效果：**U-Boot 等待 PC 传输，PC 能发现 DFU 设备。修改环境变量后要重新运行 `dfu`，不需要执行 `saveenv`。

## 4. Windows 安装驱动并下载

准备 [Zadig](https://zadig.akeo.ie/) 和 [dfu-util](https://dfu-util.sourceforge.net/releases/)。板端保持 DFU 会话：

1. 打开 Zadig，选择 `Options → List All Devices`。
2. 通过 VID/PID 确认目标 DFU 设备，选择 `WinUSB` 并安装；只为目标设备操作。
3. 在 Windows 命令行列出设备，确认 alt 0 对应本例的 RAM 目标：

```bat
dfu-util.exe -l
```

4. 下载准备好的、不超过 64 KiB 的测试文件：

```bat
dfu-util.exe -a 0 -D firmware1.bin
```

**预期效果：**工具报告传输完成，板端无 DFU 写入错误。按 `Ctrl+C` 回到 U-Boot 后，可查看 RAM 起始处：

```text
md.b 0x41100000 40
```

这里显示起始 64 字节，用于与文件头比对，不代表已经验证整个文件的一致性。

## 5. 重复下载与排查

再次传输前，按原项目验证方式退出并重新启动 `dfu 0 ram 0`，再执行：

```bat
dfu-util.exe -a 0 -D firmware2.bin
```

同一 alt 的后续下载覆盖同一段 RAM；如需保留前一文件，应先规划不同目标区域。

| 现象 | 检查项 |
| --- | --- |
| PC 找不到设备 | U-Boot 是否运行 DFU、DWC3 是否为 peripheral、设备口及数据线是否正确 |
| 能枚举但工具打不开 | Windows 驱动是否为目标设备的 WinUSB，VID/PID 是否匹配 |
| 内存分配失败 | `dfu_bufsiz` 与 U-Boot malloc 空间是否匹配 |
| 下载失败或超出目标区域 | 文件大小、目标地址、区域大小以及 U-Boot 串口错误 |

需要详细工具输出时使用 `dfu-util.exe -a 0 -D firmware1.bin -v`，同时保留 U-Boot 串口信息。

MMC、NAND、MTD 等后端需要各自的配置与 `dfu_alt_info` 格式，本例不验证其烧写流程。格式参见 [U-Boot DFU 文档](https://docs.u-boot.org/en/latest/usage/dfu.html)，工具参数参见 [dfu-util 手册](https://dfu-util.sourceforge.net/dfu-util.1.html)。
