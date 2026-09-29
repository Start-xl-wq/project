# AXI 总线错误响应 (SLVERR / DECERR) 与 CPU 死锁定位指南

> **定位**：芯片原厂 Bring-up 与实操排错圣经。当 CPU 执行一条读写指令就立即卡死（Hang 住）、或内核报 External Abort 时，如何通过总线协议级分析定位是哪个模块在捣鬼。

---

## 1. 经典故障现象与错误分类

### 1.1 现象 A：同步异常 (Synchronous External Abort)
* **表现**：CPU 执行到某条特定的汇编指令（如 `ldr r0, [r1]` 读寄存器）时，立即引发 Data Abort，堆栈和 PC 指针精准停在该行。
* **原因**：CPU 发起的 AXI 读事务返回了 `RRESP = SLVERR` 或 `DECERR`。因为 CPU 必须拿到读数据才能继续执行，因此该异常是严格“同步”的。

### 1.2 现象 B：异步异常 (Asynchronous External Abort / SError)
* **表现**：CPU 执行写指令 `str r0, [r1]`，指令早已执行完毕并跳过，几十或几百个时钟周期后内核突然 Panic 崩溃，打印的堆栈完全不是事发现场。
* **原因**：AXI 写操作是“缓冲型”的（Buffered Write）。CPU 写入片上写缓冲区（Write Buffer）后就认为指令已完成并继续执行下一条；当写数据最终到达 Slave 并触发 `BRESP = SLVERR/DECERR` 时，中断早已滞后，CPU 只能触发全局 `SError`。

### 1.3 现象 C：CPU 彻底挂死 (Bus Hang / Hard Lockup)
* **表现**：JTAG 连上发现 PC 停在一条简单的 `readl()` 指令上，时钟和中断均被阻塞，系统彻底失去响应。
* **原因**：**READY 握手信号丢失**。CPU 发出了 `ARVALID=1`，但目标外设的控制器因为自身状态机挂死、或者根本没有给外设供给总线时钟（Clock Gate 处于关闭状态），导致该外设**永远无法将 `ARREADY` 拉高**。CPU 总线接口一直在等待 `READY`，造成硬件死锁。

---

## 2. 芯片原厂故障定位“四步法”

```text
第 1 步：查时钟与电源门控 (Clock & Power Gating)
   └─ 目标外设的时钟是否已开启？外设上游复位是否已释放？
      (80% 的 Bus Hang 都是因为访问了关闭时钟的寄存器！)

第 2 步：核对芯片手册 Memory Map 物理基地址
   └─ 地址是否命中保留空洞？
      (命中空洞会导致 Interconnect 无法译码，报 DECERR)

第 3 步：使用 JTAG 探测总线接口寄存器
   └─ 读取 NoC / NIC-400 的错误状态捕获寄存器 (Error Logger)，
      查看记录的 Faulting Address、Master ID 与 Burst Type。

第 4 步：外设内部访问属性校验
   └─ 是否对外设 FIFO 进行了错误的 Burst 读写？
   └─ 是否以非对齐方式访问了只支持 32-bit 对齐的外设？
```

---

## 3. 防范措施与驱动加固规则
1. **访问外设寄存器前，严格遵从时序依赖**：
   * 必须遵循：`Power ON -> Clock Enable -> Deassert Reset -> Read/Write Registers`；
2. **写操作强行同步 (Barrier 与 Dummy Read)**：
   * 在需要确保配置生效的场景（如使能中断后），在 `writel()` 之后紧跟一个对同一寄存器的 `readl()`（利用读依赖强行排空写缓冲区），确保写响应完成。
