# DWC3 UVC 重新开流瞬间绿屏

本文记录 AX615 原项目的一次修复：重新开流时复用了残留 TRB/HWO 状态的 ring，导致首段视频数据错位。以下现象与验证结果来自源项目记录，本次文档整理未重新上板验证。

## 适用环境

- AX615 NOR 开发板，`AX615_fastnor` 配置，RT-Thread LTS v4.1.x。
- 移植的 CherryUSB 与自研 DWC3 device port，控制器工作于 USB 2.0 HS。
- 静态 MJPEG bulk UVC demo：`video_static_mjpeg_pump_template.c`，`VIDEO_USE_BULK=1`。
- Windows + PotPlayer，设备 `cherryusb uvc demo`，格式 `640*480 29.96fps`。

这是一例特定 port 的描述符生命周期问题，不代表上游 CherryUSB 或所有 UVC 绿屏的共同根因。

## 触发步骤与现象

1. 上电并等待 Windows 完成枚举。
2. 在 PotPlayer 中打开设备，首次出图正常。
3. 点击“停止”，再重新打开设备。

原项目记录：重新打开的一瞬间，约 2/3 屏幕变绿，约 1 秒后自行恢复；首次打开没有此现象。

## 原项目定位根因

DWC3 的 TRB ring 在重新打开时保留了上一轮的 TRB 内容及 HWO（硬件所有权）状态。控制器重新使用 ring 后，把陈旧 TRB 当作有效描述符执行，导致视频数据错位。

定位重点是重启传输前的 ring 初始化与 DMA 可见性。仅重置软件队列或索引，不能消除 DMA 内存中的旧描述符。

## 修复顺序

原项目在 `dwc3_ring_init()` 中补齐以下顺序：

1. 在重新交给控制器前，通过 `memset` 清零待初始化的 TRB ring。
2. 调用 `usb_dcache_flush()`，将清零结果同步到 DMA 可见的内存。
3. 调用 `dwc3_axera_dmb()`，完成所需的内存屏障。
4. 再按正常初始化与提交流程将 ring 交给控制器。

上述操作应位于 ring 可安全重新初始化的阶段，不能直接覆盖仍由控制器访问的 TRB。传输结束与回收顺序另见[停止拔插后不出流](DWC3_UVC停止拔插后不出流.md)。本文描述修复逻辑，不提供缺少实际 ring 地址、长度与上下文的可直接粘贴补丁。

## 回归步骤与预期结果

1. 用相同板卡、固件配置和 Windows 播放器验证首次开流，确认能正常出图。
2. 反复执行“出流 → 停止 → 重新打开”，重点观察重新打开的第一秒。
3. 补充检查出流中的暂停/继续，以及直接拔插后的重新枚举与开流。

预期：重新开流不再出现瞬间绿屏，画面持续正常，串口仍有响应。源项目记录该问题已修复，停流重开、暂停/继续及拔插回归均能正常出流；原记录没有给出循环次数，本文不补填次数或成功率。

## 配套文档

- [RTT CherryUSB UVC 操作指南](../06_Class功能类实战/UVC/RTT_CherryUSB_UVC操作指南.md)：环境准备、编译烧录和 demo 操作。
- [CherryUSB UVC 接口设计](../06_Class功能类实战/UVC/CherryUSB_UVC接口设计.md)：request、TRB group 与缓冲所有权。
