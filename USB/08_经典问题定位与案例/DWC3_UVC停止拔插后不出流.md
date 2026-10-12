# DWC3 UVC 停止、拔插后不出流

本文记录 AX615 原项目的两处叠加问题：在途传输尚未结束时执行 `CLEARSTALL`，以及 reset/disconnect 时未释放控制器侧 transfer resource。以下现象与验证结果来自源项目记录，本次文档整理未重新上板验证。

## 适用环境

- AX615 NOR 开发板，`AX615_fastnor` 配置，RT-Thread LTS v4.1.x。
- 移植的 CherryUSB 与自研 DWC3 device port，控制器工作于 USB 2.0 HS。
- 静态 MJPEG bulk UVC demo：`video_static_mjpeg_pump_template.c`，`VIDEO_USE_BULK=1`。
- Windows + PotPlayer，设备 `cherryusb uvc demo`，格式 `640*480 29.96fps`。

## 触发步骤与现象

1. 上电，等待 Windows 枚举后打开 UVC 设备，确认正常出流。
2. 在播放器中点击“停止”。
3. 拔出 USB，再插回，等待重新枚举。
4. 重新打开设备，并重复上述操作。

原项目记录：该操作序列下第一次能出流、第二次不出流，串口控制台也停止输出，未观察到 HardFault。

## 原项目定位根因

两处问题共同影响端点的传输生命周期：

| 路径 | 原项目问题 | 影响 |
|---|---|---|
| 清除端点 halt | 该项目观察到 Windows 播放器点击“停止”后出现 `CLEAR_FEATURE(HALT)`；端点仍有在途 transfer，驱动却直接下发 `CLEARSTALL` | stall 命令与在途传输发生冲突，需要先结束传输 |
| reset/disconnect | 只清理软件请求队列，没有发 `ENDTRANSFER` 释放控制器侧 transfer resource | 原项目中 resource 跨总线复位仍被占用，重新枚举后无法重新启动传输 |

这里的 `CLEAR_FEATURE(HALT)` 是该项目测试环境中观察到的行为，不能把所有 Windows 应用或所有 UVC 停流都概括为同一种控制请求。其他环境应结合实际控制传输与端点事件定位。

## 修复顺序

### 在途传输期间处理 stall 请求

1. 引入 `stall_pending`，记录尚不能立即执行的 stall 操作。
2. 如果端点仍有在途传输，先发 `ENDTRANSFER`，暂缓 `SET/CLEARSTALL`。
3. 等控制器确认结束，再回收 request 并处理完成回调，随后执行待处理的 stall 命令。

原项目正常取消/关闭路径使用 `ENDTRANSFER | CMDIOC`，等待 `EPCMDCMPLT` 确认控制器不再访问 TRB 后才回收 request。不能提前复用请求或其 DMA 缓冲。

### reset/disconnect 释放硬件资源

1. 新增 `dwc3_release_active_transfer()`，先结束活动传输并释放控制器侧 transfer resource。
2. 再调用 `dwc3_abort_queue()` 清理软件请求队列。
3. 完成请求取消与缓冲归还后，再进入后续重新配置与传输流程。

原项目的 reset/disconnect 释放路径直接写 `DEPCMD`，没有复用普通 `dwc3_ep_command()`：总线复位期间命令可能返回错误，不能因普通命令错误处理触发 `dwc3_transfer_fault()`，进而拆掉整个控制器。该特殊处理依赖原 port 的实现，不应脱离对应控制器状态机直接照搬。

## 回归步骤与预期结果

在相同 Windows 静态 MJPEG bulk 环境分别执行：

| 操作 | 预期结果 |
|---|---|
| 出流 → 停止 → 重新打开 | 正常恢复出流 |
| 出流 → 暂停 → 继续 | 正常恢复出流 |
| 出流中直接拔出 → 插回 | 重新枚举后可正常打开 |
| 出流 → 停止 → 拔出 → 插回 → 打开，反复执行 | 每次均可恢复出流，串口控制台不再停止响应 |

源项目记录上述回归均通过，停止后反复拔插不再挂死。原记录没有给出循环次数与持续时间，因此该结论仅表示已记录场景通过，不代表无限次或长期稳定性证明。

## 配套文档

- [RTT CherryUSB UVC 操作指南](../06_Class功能类实战/UVC/RTT_CherryUSB_UVC操作指南.md)：环境准备、编译烧录和 demo 操作。
- [CherryUSB UVC 接口设计](../06_Class功能类实战/UVC/CherryUSB_UVC接口设计.md)：DWC3 port 的取消、关闭与请求回收流程。
- [重新开流瞬间绿屏](DWC3_UVC重新开流绿屏.md)：重新使用 TRB ring 前的初始化与 DMA 同步。
