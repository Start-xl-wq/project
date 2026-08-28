# Axera 平台 USB DFU 升级使用指南

> **文档定位**：USB 驱动工程师 → AE 工程师  
> **提供内容**：DFU RAM 下载 Sample（已验证）  
> **平台**：Axera 芯片 · DWC3 USB 控制器 · U-Boot  
> **工具**：dfu-util（Windows）

---

## ⚠️ 声明

```  
本文档仅提供 DFU RAM 下载参考实现(Sample)。  
NAND/MMC/MTD 等介质不提供,请参考 RAM 实现自行移植。  
USB 驱动工程师仅保证 USB Device + DFU 链路可用。  
```

---

## 1. 原理简介

DFU 是 USB 标准固件升级协议。设备 U-Boot 作为 **USB Device** 被 PC 识别，PC 端 `dfu-util` 通过 USB 把文件下发到设备：

```  
PC (dfu-util) ──USB──► [DFU 暂存缓冲区(malloc分配)] ──► 目标介质(RAM)  
```

- 数据先进入暂存缓冲区（大小由 `dfu_bufsiz` 决定），再写入目标介质  
- **默认 U-Boot 的 USB 是 Host 模式，DFU 必须跑在 Device 模式，因此必须改配置和 DTS**

---

## 2. U-Boot 配置（defconfig）

在板级 defconfig 中增加：

```kconfig  
# USB Gadget (Device 模式)  
CONFIG_USB_GADGET=y  
CONFIG_USB_GADGET_DWC3=y  
CONFIG_USB_GADGET_DOWNLOAD=y  
CONFIG_USB_GADGET_MANUFACTURER="Axera"  
CONFIG_USB_GADGET_VENDOR_NUM=0x1234 # 按实际 VID/PID  
CONFIG_USB_GADGET_PRODUCT_NUM=0x5678

# DFU 功能  
CONFIG_DFU=y  
CONFIG_CMD_DFU=y  
CONFIG_DFU_RAM=y # ★ RAM Sample 必需  
CONFIG_USB_FUNCTION_DFU=y

# 缓冲区:默认 8MB 太大会分配失败,必须调小  
CONFIG_SYS_DFU_DATA_BUF_SIZE=0x10000  
```

> 其他介质（供参考，本 Sample 不开）：`CONFIG_DFU_MMC` / `CONFIG_DFU_NAND` / `CONFIG_DFU_MTD`

---

## 3. DTS 修改（关键）

默认 `dr_mode` 为 host，**必须改为 peripheral**，否则 PC 完全识别不到设备：

```dts  
&dwc3 {  
dr_mode = "peripheral"; /* host → peripheral */  
status = "okay";  
};  
```

---

## 4. Windows 驱动安装（Zadig）

设备进入 Device 模式后，Windows 默认无驱动，需用 **Zadig** 安装一次：

1. 板端 U-Boot 执行 `dfu 0 ram 0`（此时设备已枚举）  
2. 打开 Zadig → **Options → List All Devices**  
3. 选中 DFU 设备（VID/PID 对应 defconfig 中配置）  
4. 驱动选择 **WinUSB** → 点击 **Install Driver**  
5. 安装一次即可，之后无需重复

---

## 5. 使用步骤

### 本平台已验证参数

| 参数 | 值 |  
|------|-----|  
| 目标地址 | `0x41100000`（实测唯一可写，其他地址写入无效） |  
| 区域大小 | `0x10000`（64KB，**文件不得超过此大小**） |  
| 缓冲区 | `0x10000` |

### 步骤 1：U-Boot 端（逐条执行）

```bash  
Ctrl + C # 退出可能残留的 dfu  
setenv dfu_bufsiz 0x10000  
setenv dfu_alt_info "kernel ram 0x41100000 0x10000"  
dfu 0 ram 0 # 阻塞等待 PC  
```

> 注意：`dfu` 只在启动时读一次环境变量，改完变量必须重新运行 `dfu`。

### 步骤 2：Windows 端下发固件（U-Boot 阻塞后立即执行）

```cmd  
dfu-util.exe -a 0 -D firmware.bin  
```

下载完成，U-Boot 串口打印成功信息即结束。报错一律看 **U-Boot 串口**。

### 其他常用命令

```cmd  
dfu-util.exe -l :: 列出 DFU 设备  
```

```bash  
mw.l 0x41100000 0x12345678 # 验证地址可写性  
md.l 0x41100000 1 # 读回 12345678 即有效  
```

---

## 6. 移植其他介质（参考 RAM）

介质写入与 USB 链路无关，USB 部分**无需改动**，只需两步：

1. defconfig 开启对应宏：`CONFIG_DFU_MMC` / `CONFIG_DFU_NAND` / `CONFIG_DFU_MTD`  
2. 按介质格式构造 `dfu_alt_info` 并换用对应 `dfu` 命令，例如：

```bash  
# MMC 示例(格式: 名称 mmc <dev>:<part>)  
setenv dfu_alt_info "kernel mmc 0:1"  
dfu 0 mmc 0  
```

具体介质调试由 AE 自行完成。

---

## 附录：完整操作流程速查

```bash  
# ===== U-Boot 端 =====  
setenv dfu_bufsiz 0x10000  
setenv dfu_alt_info "kernel ram 0x41100000 0x10000"  
dfu 0 ram 0  
```

```cmd  
:: ===== Windows 端(首次需 Zadig 装 WinUSB 驱动) =====  
dfu-util.exe -a 0 -D firmware.bin  
```