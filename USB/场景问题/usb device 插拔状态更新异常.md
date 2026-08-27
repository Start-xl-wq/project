## 1. 背景

当前方案使用 Micro USB 接口，系统为 USB 自供电设计，不依赖外部 VBUS 给系统供电。

USB 角色通过 Micro USB 的 ID/IO 进行判断：

- 对端为 Host 时，本端作为 Device；
    
- 对端为 Device 时，本端作为 Host。
    

---

## 2. 作为 Device 时的插拔检测

### 2.1 Host 插入检测

当前作为 Device 时，Host 插入是可以检测到的，方式是：

- Host 接入后会发送 USB Reset 信号；
    
- Device 侧通过检测到该 Reset 信号，判断 Host 已接入并建立连接。
    

流程如下：

```
Host 插入
  ↓
Host 发送 USB Reset
  ↓
Device 检测到 Reset
  ↓
Device 判断连接建立
```

---

### 2.2 Host 拔出检测问题

当前作为 Device 时，Host 拔出无法直接检测，主要原因是：

- 现有硬件缺少专门用于检测 VBUS 状态变化的 GPIO；
    
- 当前USB 2.0 PHY（芯动）不支持检测，无法可靠感知拔出动作；
    
- 若直接让 GPIO 去驱动 DWC3 的断开逻辑，会和当前 DWC3 驱动架构耦合较深；
    
- 这种方式需要较多修改 DWC3 状态机和事件处理流程，软件改动会比较大。
    

---

## 3. 为什么不能直接让 GPIO 触发 DWC3 逻辑

这里的关键点是：GPIO 只能感知电平变化，但 DWC3 需要的是符合其驱动模型的 disconnect 事件。

如果由 GPIO 直接去控制 DWC3 断开流程，会带来以下问题：

### 3.1 与 DWC3 架构耦合过深

DWC3 驱动通常依赖其自身的状态机、事件队列和中断机制来处理连接/断开。

如果 GPIO 直接插入 DWC3 断开路径，相当于绕过了 DWC3 原有的事件入口，会让软件逻辑变得分散，不利于维护。

### 3.2 软件改动较大

直接 GPIO 触发 DWC3，通常意味着要补充：

- GPIO 中断处理；
    
- USB 连接状态同步；
    
- DWC3 内部状态清理；
    
- disconnect 场景下的边界处理。
    

这些内容会增加对现有驱动框架的改动量。

### 3.3 不利于复用现有硬件通知流程

DWC3 本身已经具备完整的 disconnect 处理流程。

如果直接从 GPIO 介入，就等于绕开 PHY/DWC3 的既有通知机制，无法很好复用现有框架。

---

## 4. 当前采用的软件处理方式

我们当前已经采用的方式是：

1. GPIO 检测到 Host 拔出；
    
2. 软件配置 USB PHY 的某个寄存器；
    
3. 由 PHY 触发一次 disconnect event；
    
4. DWC3 接收到该 disconnect 事件后，沿用其现有硬件通知流程完成断开处理。
    

流程如下：

```
GPIO 检测到 VBUS 拔出
  ↓
软件配置 USB PHY 寄存器
  ↓
PHY 触发 disconnect event
  ↓
DWC3 接收到断开通知
  ↓
进入现有 disconnect 处理流程
```

### 这样做的原因

- GPIO 只负责“检测”；
    
- PHY 负责“上报事件”；
    
- DWC3 继续走原有断开流程；
    
- 软件只需要做少量适配，改动非常小。
    

### 4.1 软件核心实现

![](https://wiki.aixin-chip.com/download/thumbnails/253464737/image2026-6-25_17-26-1.png?version=1&modificationDate=1782379546226&api=v2 "贺端矗 > usb device 插拔检测分析 > image2026-6-25_17-26-1.png")

---

## 5. 硬件修改方案

为支持 Host 拔出检测，硬件需要做如下修改。

### 5.1 在 VBUS 上增加 GPIO 检测

需要在 VBUS 上增加一路 GPIO，用于检测外部 VBUS 电平变化。

目的：

- 识别 Host 是否插入；
    
- 识别 Host 是否拔出；
    
- 为软件触发 PHY disconnect 提供依据。
    

---

### 5.2 处理 DP/DN 倒灌导致的 VBUS 残压问题

当前硬件设计下，即使 Host 拔出，仍可能存在：

- DP/DN 倒灌电压；
    
- 使得 VBUS 电压无法立即拉低；
    
- VBUS 可能维持在约 2.5V 左右。
    

这会导致 GPIO 无法准确判断 Host 已经拔出。

因此需要修改对应的 分压电阻 设计，使得：

- DP/DN 倒灌对 VBUS 的影响减小；
    
- Host 拔出后，VBUS 能够更快、更可靠地回到低电平；
    
- GPIO 能正确识别 VBUS 失效状态。
    

---

## 6. 硬件设计  
![](https://wiki.aixin-chip.com/download/attachments/253464737/image2026-6-25_16-29-45.png?version=1&modificationDate=1782376170434&api=v2 "贺端矗 > usb device 插拔检测分析 > image2026-6-25_16-29-45.png")