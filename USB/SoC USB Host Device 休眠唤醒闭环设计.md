
## 1. 背景

平台支持 USB Host 和 USB Device/Gadget 两种角色，涉及低功耗休眠和唤醒。

前提：

```text
1. SoC 低功耗时，不关闭 USB PHY + USB controller 电源。
2. USB clock 可以关闭。
3. USB clock 关闭后，普通 USB 中断不可用。
4. USB clock 关闭后，只有独立 wakeup interrupt 可用。
5. wakeup interrupt 检测 DP/DN 电平变化。
6. wakeup interrupt 触发后不能通过普通 clear 清除，只能 mask/disable。
7. wakeup interrupt 的触发状态，需要等下一次 USB suspend 后才会消失。
```

因此总体设计为：

```text
进入 suspend：
    enable wakeup interrupt
    close USB clk

发生 DP/DN 变化：
    wakeup interrupt 触发
    mask wakeup interrupt
    open USB clk
    恢复 USB controller 普通中断
    交给 Host/Device 状态机继续处理

再次进入 suspend：
    重新 enable wakeup interrupt
```

---

# 2. 核心原则

## 2.1 wakeup interrupt 只负责唤醒

wakeup interrupt 只能说明：

```text
DP/DN 电平发生变化
```

不能直接区分：

```text
Host 模式：
    device remote wakeup
    device disconnect
    device connect

Device 模式：
    host resume
    cable unplug
    host connect
```

所以 wakeup irq 中不要做最终状态判断，只做：

```text
mask wakeup irq
触发 resume
恢复 USB clk
```

后续状态由：

```text
Host:
    xHCI/root hub port change/change_state

Device:
    DWC3/gadget event
    VBUS IO
```

继续判断。

---

## 2.2 USB clk 关闭后，只能依赖 wakeup irq 或 VBUS IO

USB clk 关闭后，以下中断都不可依赖：

```text
USB controller interrupt
xHCI interrupt
DWC3 device event interrupt
port change interrupt
```

可用唤醒源只有：

```text
1. wakeup interrupt：检测 DP/DN 变化
2. VBUS IO interrupt：Device 模式检测插拔
3. 系统主动 resume
```

---

## 2.3 wakeup interrupt 触发后必须先 mask

因为 wakeup irq 触发后不能直接 clear，只能 mask。

所以 wakeup irq handler 第一动作必须是：

```c
disable_irq_nosync(wakeup_irq);
```

否则可能导致：

```text
1. 中断风暴
2. resume 过程重复进入
3. clk 未稳定时重复访问 USB controller
4. 状态机重入
```

---

# 3. 全局状态机

建议统一抽象为以下状态：

```text
ACTIVE
    USB clk on
    wakeup irq disabled
    normal USB irq enabled
    USB controller/xHCI/gadget 正常工作

SUSPENDING
    USB core/xHCI/gadget 正在进入 suspend
    准备关闭 USB clk

SUSPENDED
    USB clk off
    wakeup irq enabled
    normal USB irq unavailable
    USB PHY + controller power 保持

RESUMING
    wakeup irq disabled
    USB clk on
    controller 恢复
    normal USB irq 恢复

RECOVERING
    根据恢复后的真实硬件状态处理：
        remote wakeup
        resume
        connect
        disconnect
        enumeration
```

状态转换：

```text
ACTIVE
  -> SUSPENDING
  -> SUSPENDED
  -> RESUMING
  -> RECOVERING
  -> ACTIVE / ACTIVE(no device/no host)
```

---

# 4. 通用 suspend 动作

无论 Host 还是 Device，底层 suspend 动作保持统一语义。

## 4.1 suspend 做什么

```text
1. 配置 wakeup detect
2. enable wakeup interrupt
3. disable/mask 普通 USB 中断
4. 关闭 USB clk
5. 标记 suspended
```

推荐顺序：

```text
axera_suspend
    -> config wakeup detect
    -> enable wakeup irq
    -> disable normal USB irq
    -> close USB clk
    -> suspended = true
```

伪代码：

```c
static int axera_usb_suspend(struct axera_usb *axera)
{
    if (axera->suspended)
        return 0;

    axera_config_wakeup_detect(axera, true);

    enable_irq(axera->wakeup_irq);

    axera_disable_normal_usb_irq(axera);

    clk_disable_unprepare(axera->usb_clk);

    axera->suspended = true;
    axera->resuming = false;

    return 0;
}
```

---

# 5. 通用 resume 动作

## 5.1 resume 做什么

```text
1. disable/mask wakeup interrupt
2. 打开 USB clk
3. 恢复 controller/glue 状态
4. enable 普通 USB 中断
5. 标记 active
6. 后续交给 Host/Device 状态机处理
```

推荐顺序：

```text
axera_resume
    -> disable wakeup irq
    -> open USB clk
    -> restore controller
    -> enable normal USB irq
    -> suspended = false
```

伪代码：

```c
static int axera_usb_resume(struct axera_usb *axera)
{
    if (!axera->suspended)
        return 0;

    disable_irq_nosync(axera->wakeup_irq);

    clk_prepare_enable(axera->usb_clk);

    axera_restore_usb_context(axera);

    axera_enable_normal_usb_irq(axera);

    axera->suspended = false;
    axera->resuming = false;

    return 0;
}
```

---

# 6. wakeup interrupt 处理

## 6.1 wakeup irq 只做唤醒入口

```text
wakeup irq
    -> mask wakeup irq
    -> 防重入
    -> 调度 resume
```

伪代码：

```c
static irqreturn_t axera_usb_wakeup_irq(int irq, void *data)
{
    struct axera_usb *axera = data;

    disable_irq_nosync(irq);

    if (axera->resuming)
        return IRQ_HANDLED;

    axera->resuming = true;

    schedule_work(&axera->resume_work);

    return IRQ_HANDLED;
}
```

resume work：

```c
static void axera_usb_resume_work(struct work_struct *work)
{
    struct axera_usb *axera =
        container_of(work, struct axera_usb, resume_work);

    axera_usb_resume(axera);

    /*
     * 不在这里直接判断 remote wakeup/connect/disconnect。
     * 后续交给 Host/Device 自身中断和状态机。
     */
}
```

---

# 7. Host 模式状态转换

## 7.1 Host 已枚举，主动 suspend

```text
ACTIVE(host, enumerated)
    |
    | host_xhci_suspend
    v
SUSPENDING
    |
    | axera_suspend
    |   enable wakeup irq
    |   close USB clk
    v
SUSPENDED(host, enumerated)
```

---

## 7.2 Host 已枚举，Host 主动 resume

```text
SUSPENDED(host, enumerated)
    |
    | host_xhci_resume
    v
RESUMING
    |
    | axera_resume
    |   disable wakeup irq
    |   open USB clk
    v
ACTIVE(host, enumerated)
```

说明：

```text
Host 主动 resume 不依赖 DP/DN 变化，因此不会触发 wakeup irq。
```

---

## 7.3 Host 已枚举，Device remote wakeup

```text
SUSPENDED(host, enumerated)
    |
    | device 发送 remote wakeup/resume signaling
    | DP/DN 变化
    v
wakeup irq
    |
    | mask wakeup irq
    v
RESUMING
    |
    | axera_resume
    |   open USB clk
    v
RECOVERING
    |
    | xHCI/root hub change_state
    | USB core recover
    v
ACTIVE(host, enumerated)
```

---

## 7.4 Host 已枚举，Device 拔出

```text
SUSPENDED(host, enumerated)
    |
    | device 拔出
    | DP/DN 变化
    v
wakeup irq
    |
    | mask wakeup irq
    v
RESUMING
    |
    | axera_resume
    |   open USB clk
    v
RECOVERING
    |
    | xHCI/root hub change_state
    | port status = disconnect
    v
ACTIVE(host, no device)
```

---

## 7.5 Host 未枚举，Device 插入

```text
SUSPENDED(host, no device)
    |
    | device 插入
    | DP/DN 变化
    v
wakeup irq
    |
    | mask wakeup irq
    v
RESUMING
    |
    | axera_resume
    |   open USB clk
    v
RECOVERING
    |
    | xHCI/root hub port connect change
    | USB enumeration
    v
ACTIVE(host, enumerated)
```

---

# 8. Device 模式状态转换

## 8.1 Device 未枚举，进入 suspend

```text
ACTIVE(device, no host/no enumeration)
    |
    | device idle suspend
    v
SUSPENDING
    |
    | axera_suspend
    |   enable wakeup irq
    |   close USB clk
    v
SUSPENDED(device, not enumerated)
```

---

## 8.2 Device 未枚举，插入 Host

```text
SUSPENDED(device, not enumerated)
    |
    | 插入 Host
    | VBUS valid 或 DP/DN 变化
    v
wakeup irq / VBUS irq
    |
    | mask wakeup irq
    v
RESUMING
    |
    | axera_resume
    |   open USB clk
    v
RECOVERING
    |
    | UDC ready
    | Host reset
    | USB enumeration
    v
ACTIVE(device, enumerated)
```

---

## 8.3 Device 已枚举，Host 主动 suspend

```text
ACTIVE(device, enumerated)
    |
    | Host bus suspend
    | Device PHY/controller 检测 suspend
    v
SUSPENDING
    |
    | axera_suspend
    |   enable wakeup irq
    |   close USB clk
    v
SUSPENDED(device, enumerated)
```

---

## 8.4 Device 已枚举，Host 主动 resume

```text
SUSPENDED(device, enumerated)
    |
    | Host resume signaling
    | DP/DN 变化
    v
wakeup irq / PHY resume detect
    |
    | mask wakeup irq
    v
RESUMING
    |
    | axera_resume
    |   open USB clk
    v
RECOVERING
    |
    | DWC3/gadget resume event
    | function resume
    v
ACTIVE(device, enumerated)
```

---

## 8.5 Device 已枚举，拔出

```text
SUSPENDED(device, enumerated)
    |
    | cable unplug
    | VBUS invalid
    v
VBUS irq
    |
    | mask wakeup irq
    v
RESUMING
    |
    | axera_resume
    |   open USB clk
    v
RECOVERING
    |
    | gadget disconnect
    | stop endpoints
    | clear configuration
    v
ACTIVE(device, no host)
```

如果拔出时同时触发 DP/DN wakeup：

```text
VBUS irq 和 wakeup irq 都可能进入 resume 路径。
需要 resuming 标志防重入。
```

---

# 9. Host/Device 场景汇总

## 9.1 Host 模式

| 场景 | 当前状态 | 触发源 | resume 前动作 | resume 后事件 | 最终状态 |
|---|---|---|---|---|---|
| Host 主动 suspend | ACTIVE, enumerated | host_xhci_suspend | enable wakeup irq, close clk | 无 | SUSPENDED |
| Host 主动 resume | SUSPENDED, enumerated | host_xhci_resume | disable wakeup irq | 无 | ACTIVE |
| Device remote wakeup | SUSPENDED, enumerated | wakeup irq | mask wakeup irq | change_state/recover | ACTIVE |
| Device 拔出 | SUSPENDED, enumerated | wakeup irq | mask wakeup irq | change_state/disconnect | ACTIVE(no device) |
| Device 插入 | SUSPENDED, no device | wakeup irq | mask wakeup irq | port connect/enumeration | ACTIVE(enumerated) |

---

## 9.2 Device 模式

| 场景 | 当前状态 | 触发源 | resume 前动作 | resume 后事件 | 最终状态 |
|---|---|---|---|---|---|
| 未枚举 suspend | ACTIVE, no host | idle suspend | enable wakeup irq, close clk | 无 | SUSPENDED |
| 插入 Host | SUSPENDED, not enumerated | wakeup irq/VBUS irq | mask wakeup irq | reset/enumeration | ACTIVE(enumerated) |
| Host bus suspend | ACTIVE, enumerated | bus suspend | enable wakeup irq, close clk | 无 | SUSPENDED |
| Host resume | SUSPENDED, enumerated | wakeup irq/PHY resume | mask wakeup irq | gadget resume | ACTIVE(enumerated) |
| 拔出 | SUSPENDED, enumerated | VBUS irq | mask wakeup irq | gadget disconnect | ACTIVE(no host) |

---

# 10. 关键时序要求

## 10.1 suspend 时序

```text
config wakeup detect
    -> enable wakeup irq
        -> disable normal USB irq
            -> close USB clk
                -> enter SUSPENDED
```

不要先关 clk 再配置 wakeup。

---

## 10.2 resume 时序

```text
wakeup irq / VBUS irq / active resume
    -> disable wakeup irq
        -> open USB clk
            -> restore controller
                -> enable normal USB irq
                    -> enter RECOVERING
```

不要先开普通 USB 中断再处理 wakeup irq mask。

---

## 10.3 wakeup irq 只在 suspend path enable

```text
enable wakeup irq:
    only in axera_suspend

disable wakeup irq:
    in wakeup irq handler
    in axera_resume
```

resume 完成后不要立即重新 enable wakeup irq。

因为 wakeup irq 触发状态可能需要等下一次 suspend 后才消失。

---

## 10.4 resume 后交给正常 USB 状态机

resume 后不在 wakeup irq 中判断具体原因。

Host 模式：

```text
open clk
    -> xHCI/root hub port change
        -> remote wakeup recover
        -> connect
        -> disconnect
```

Device 模式：

```text
open clk
    -> DWC3/gadget event 或 VBUS IO
        -> resume
        -> reset
        -> enumeration
        -> disconnect
```

---

## 10.5 防重入

需要防止以下路径重复 resume：

```text
1. wakeup irq 重复触发
2. wakeup irq 和 VBUS irq 同时触发
3. wakeup irq 和系统主动 resume 同时发生
4. resume 尚未完成时再次进入 resume
```

建议使用状态位：

```c
if (axera->resuming)
    return;

axera->resuming = true;
```

或者 atomic bit：

```c
if (test_and_set_bit(AXERA_USB_RESUMING, &axera->flags))
    return;
```

---

# 11. 代码框架

## 11.1 数据结构

```c
struct axera_usb {
    struct device *dev;

    int wakeup_irq;
    int vbus_irq;

    struct clk *usb_clk;

    struct work_struct resume_work;

    bool suspended;
    bool resuming;

    enum axera_usb_role role;
};
```

---

## 11.2 suspend

```c
static int axera_usb_suspend(struct axera_usb *axera)
{
    if (axera->suspended)
        return 0;

    axera_config_wakeup_detect(axera, true);

    enable_irq(axera->wakeup_irq);

    axera_disable_normal_usb_irq(axera);

    clk_disable_unprepare(axera->usb_clk);

    axera->suspended = true;
    axera->resuming = false;

    return 0;
}
```

---

## 11.3 resume

```c
static int axera_usb_resume(struct axera_usb *axera)
{
    if (!axera->suspended)
        return 0;

    disable_irq_nosync(axera->wakeup_irq);

    clk_prepare_enable(axera->usb_clk);

    axera_restore_usb_context(axera);

    axera_enable_normal_usb_irq(axera);

    axera->suspended = false;
    axera->resuming = false;

    return 0;
}
```

---

## 11.4 wakeup irq

```c
static irqreturn_t axera_usb_wakeup_irq(int irq, void *data)
{
    struct axera_usb *axera = data;

    disable_irq_nosync(irq);

    if (axera->resuming)
        return IRQ_HANDLED;

    axera->resuming = true;

    schedule_work(&axera->resume_work);

    return IRQ_HANDLED;
}
```

---

## 11.5 VBUS irq

```c
static irqreturn_t axera_usb_vbus_irq(int irq, void *data)
{
    struct axera_usb *axera = data;

    /*
     * Device 模式下，VBUS 变化可能表示插入或拔出。
     * 如果当前 USB clk 已关闭，需要先恢复 clk。
     */

    disable_irq_nosync(axera->wakeup_irq);

    if (axera->resuming)
        return IRQ_HANDLED;

    axera->resuming = true;

    schedule_work(&axera->resume_work);

    return IRQ_HANDLED;
}
```

---

## 11.6 resume work

```c
static void axera_usb_resume_work(struct work_struct *work)
{
    struct axera_usb *axera =
        container_of(work, struct axera_usb, resume_work);

    axera_usb_resume(axera);

    /*
     * Host:
     *   等待 xHCI/root hub port change
     *
     * Device:
     *   等待 DWC3/gadget event 或 VBUS 状态处理
     */
}
```

---

# 12. 最终闭环

完整闭环可以总结为：

```text
suspend:
    USB 状态机确认进入低功耗
    -> axera_suspend
    -> enable wakeup irq
    -> close USB clk
    -> SUSPENDED

wakeup:
    DP/DN change 或 VBUS change
    -> wakeup irq / VBUS irq
    -> mask wakeup irq
    -> schedule resume

resume:
    axera_resume
    -> open USB clk
    -> restore controller
    -> enable normal USB irq
    -> RECOVERING

recover:
    Host:
        xHCI/root hub change_state 判断 remote wakeup/connect/disconnect

    Device:
        DWC3/gadget/VBUS 判断 resume/reset/enumeration/disconnect

next suspend:
    再次 enable wakeup irq
    -> close USB clk
```

核心结论：

```text
1. wakeup irq 只负责唤醒，不负责判断 USB 最终状态。
2. USB clk off 后，普通 USB 中断不可用。
3. wakeup irq 触发后必须先 mask。
4. resume 后不要立即重新 enable wakeup irq。
5. Host 模式恢复后靠 xHCI/root hub change_state 闭环。
6. Device 模式恢复后靠 DWC3/gadget/VBUS 闭环。
7. 下一次 suspend 时重新 enable wakeup irq。
```