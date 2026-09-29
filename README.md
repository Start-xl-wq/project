# 嵌入式底层与总线协议实战笔记库

本笔记库归档了在芯片 Bring-up、芯片原厂驱动开发与系统级芯片验证 (CV / Post-Silicon Validation) 以及总线协议栈调优中的实战经验、核心规范与操作指南。

---

## 📚 知识目录与模块导航

### 1. [USB 体系（驱动与原厂 CV 全景）](./USB/)

* **01_硬件与PHY**：
  * [micro-USB OTG ID脚与角色说明](./USB/01_硬件与PHY/micro-USB%20OTG%20ID脚与角色说明.md)
  * [Type-C CC 逻辑与 PD 协商基础](./USB/01_硬件与PHY/Type-C_CC逻辑与PD协商基础.md)
  * [USB2 PHY 眼图调试及调优](./USB/01_硬件与PHY/USB2%20PHY眼图调试及调优.md)
  * [USB3 PHY 眼图概念及调优](./USB/01_硬件与PHY/USB3%20PHY眼图概念及调优.md)
  * [PIPE 与 UTMI+ 控制器与 PHY 接口规范](./USB/01_硬件与PHY/PIPE与UTMI+控制器接口规范.md)
* **02_协议层_USB2.0**：
  * [物理链路建立与电气协商](./USB/02_协议层_USB2.0/物理链路建立与电气协商.md)
  * [枚举与描述符体系](./USB/02_协议层_USB2.0/枚举与描述符体系.md)
  * [USB Suspend / Resume / Wakeup 场景说明](./USB/02_协议层_USB2.0/USB%20Suspend%20Resume%20Wakeup%20场景说明.md)
  * [USB 2.0 Device 驱动开发学习计划](./USB/02_协议层_USB2.0/USB%202.0%20Device%20驱动开发学习计划.md)
* **03_协议层_USB3.x**：
  * [LTSSM 链路训练状态机深度解析](./USB/03_协议层_USB3.x/LTSSM链路状态机深度解析.md)
  * [USB 3.x 休眠与唤醒逻辑](./USB/03_协议层_USB3.x/USB%203.x%20休眠与唤醒逻辑.md)
  * [USB 2.0 & USB 3.x 链路电源状态 (L0-L3 / U0-U3)](./USB/03_协议层_USB3.x/USB%202.0%20&%20USB%203.x%20链路电源状态.md)
* **04_控制器IP架构**：
  * [DWC3 core.c 学习笔记](./USB/04_控制器IP架构/DWC3%20core.c%20学习笔记.md)
  * [USB DWC3 学习路线笔记](./USB/04_控制器IP架构/USB%20DWC3%20学习路线笔记.md)
  * [DWC3 DMA 描述符环 (TRB) 与数据流转](./USB/04_控制器IP架构/DWC3_DMA描述符环(TRB)与数据流转.md)
* **05_Linux内核驱动栈**：
  * [Linux USB Gadget 核心架构与数据流转模型](./USB/05_Linux内核驱动栈/Linux_USB_Gadget核心架构与数据流转模型.md)
* **06_Class功能类实战（快速验证操作）**：
  * [ACM (虚拟串口)](./USB/06_Class功能类实战/ACM.md)
  * [ADB (Android 调试桥)](./USB/06_Class功能类实战/ADB.md)
  * [HID (虚拟键盘/鼠标)](./USB/06_Class功能类实战/HID.md)
  * [MSC (标准 U 盘/大容量存储)](./USB/06_Class功能类实战/MSC.md)
  * [UAS (USB Attached SCSI)](./USB/06_Class功能类实战/UAS.md)
  * [RNDIS (虚拟以太网卡)](./USB/06_Class功能类实战/RNDIS.md)
  * [UVC (摄像头推流)](./USB/06_Class功能类实战/UVC.md)
  * [算力棒架构与时序全集](./USB/06_Class功能类实战/算力棒/)
* **07_芯片原厂验证_CV**：
  * [芯片 Bring-up 点亮 Checklist (USB 专栏)](./USB/07_芯片原厂验证_CV/芯片Bring-up点亮Checklist.md)
  * [USB-IF 一致性测试实战 (USBCV 与合规认证)](./USB/07_芯片原厂验证_CV/USB-IF一致性测试实战(USBCV与合规).md)
  * [USB 协议分析仪 (LeCroy / Ellisys) 抓包与定位指南](./USB/07_芯片原厂验证_CV/协议分析仪(LeCroy_Ellisys)抓包指南.md)
* **08_经典问题定位与案例**：
  * [USB Device 插拔状态更新异常分析](./USB/08_经典问题定位与案例/usb%20device%20插拔状态更新异常.md)

---

### 2. [PCIe 体系（Bring-up 与总线）](./PCIe/)
* **物理层**：[物理层信号](./PCIe/物理层/PCIe%20基础之物理层-信号.md)、[LTSSM 链路建立与训练](./PCIe/物理层/PCIe%20基础之物理层-链路建立和训练%20LTSSM.md)
* **链路层**：[基础概念](./PCIe/链路层/PCIe%20基础之链路层%20基础概念.md)、[DLLP 报文](./PCIe/链路层/PCIe%20基础之链路层%20DLLP.md)、[流量控制 Flow Control](./PCIe/链路层/PCIe%20基础之链路层%20流控.md)
* **事务层**：[TLP 格式](./PCIe/事务层/PCIe%20基础之事务层%20TLP%20格式.md)、[TLP 机制](./PCIe/事务层/PCIe%20基础之事务层%20TLP%20机制.md)、[TLP 性能与开销](./PCIe/事务层/PCIe%20基础之事务层%20TLP%20性能.md)、[总线枚举](./PCIe/事务层/PCIe%20基础之事务层-枚举.md)
* **DataPass & 性能**：[数据交互模型](./PCIe/DataPass/PCIe%20基础之DataPass%20数据交互.md)、[带宽建模与瓶颈分析](./PCIe/DataPass/PCIe%20基础之Datapass%20带宽建模.md)
* **芯片 Bring-up 实战系列 (一 至 八)**：
  1. [链路训练与硬件三板斧](./PCIe/实战进阶/芯片%20Bring-up%20实战笔记%20(一)：PCIe%20点亮与链路训练.md)
  2. [BAR 探测与 MMIO 本质](./PCIe/实战进阶/芯片%20Bring-up%20实战笔记%20(二)：PCIe%20BAR%20探测与%20MMIO%20的本质.md)
  3. [I/O 事件通知机制](./PCIe/实战进阶/芯片%20Bring-up%20实战笔记%20(三)：PCIe%20I_O%20事件通知机制.md)
  4. [iATU 地址映射与工作机制](./PCIe/实战进阶/芯片%20Bring-up%20实战笔记%20(四)：PCIe%20iATU%20地址映射与工作机制.md)
  5. [ASPM 链路电源管理](./PCIe/实战进阶/芯片%20Bring-up%20实战笔记%20(五)：PCIe%20ASPM电源管理.md)
  6. [AER 错误报告机制](./PCIe/实战进阶/芯片%20Bring-up%20实战笔记%20(六)：PCIe%20AER错误机制.md)
  7. [BAR 与 DMA 底层流转](./PCIe/实战进阶/芯片%20Bring-up%20实战笔记%20(七)：PCIe%20BAR与DMA的底层流转.md)
  8. [MMU / IOMMU / iATU 地址翻译对比](./PCIe/实战进阶/芯片%20Bring-up%20实战笔记%20(八)：PCIe地址翻译%20MMU%20%20IOMMU%20iATU.md)

---

### 3. [AMBA 片上总线](./AMBA/)
* **架构总览**：[三大总线 (AXI / AHB / APB) 极简对比](./AMBA/三大总线（AXI%20AHB%20APB）极简对比.md)
* **APB**：[慢速外设生存指南](./AMBA/APB/总线生存指南：慢速外设的“单行窄巷”.md)
* **AHB**：[信号线与读写交互时序](./AMBA/AHB/总线基础原理：信号线与读写交互.md)
* **AXI**：
  * [核心笔记（一）：基础架构、物理通道与寻址规范](./AMBA/AXI/核心笔记（一）：基础架构、物理通道与寻址规范.md)
  * [核心笔记（二）：突发传输 (Burst Transfer) 与底层时序规范](./AMBA/AXI/核心笔记（二）：%20突发传输%20(Burst%20Transfer)%20与底层时序规范.md)

---

### 4. [MMC 存储与扩展总线](./MMC/)
* **eMMC**：[引导工作模式 (Original & Alternative Boot)](./MMC/eMMC/eMMC%20引导工作模式.md)
* **SDIO**：[驱动能力 (Driver Strength Type A~D) 与阻抗匹配](./MMC/SDIO/SDIO%20驱动能力.md)
* **综合对比**：[MMC / SD / SDIO 枚举初始化时钟对比与 I/O 驱动模式 (OD vs PP)](./MMC/MMC枚举初始化时钟对比.md)
* **SPEC 规格书**：`sdips_gravityxr_sdio_emmc_host_iip_databook.pdf`
