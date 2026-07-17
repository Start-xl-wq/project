# USB Suspend / Resume / Wakeup 场景说明

## 1. 术语定义

### 1.1 Runtime Suspend

`runtime suspend` 是 USB 空闲时进入的局部低功耗状态，不代表整个系统休眠。

进入流程：

- 保留 USB controller 配置和链路上下文；
- 保持必要的 DP/DM 电气状态；
- 开启 wakeup 中断；
- 关闭 USB clock；
- 不 reset USB controller。

恢复 USB clock 后，可继续使用原有链路和配置，通常不需要重新枚举。

### 1.2 Suspend（系统休眠）

`suspend` 是整个系统进入休眠时调用的 USB suspend 流程。

进入流程：

- 开启 wakeup 中断；
- 关闭 USB clock；
- reset USB controller。

USB controller reset 后，原有配置和链路上下文丢失。系统 resume 时需要重新初始化 USB controller，并根据当前连接状态重新建立链路。

### 1.3 两者区别

| 项目 | Runtime Suspend | Suspend（系统休眠） |
|---|---|---|
| 触发条件 | USB 空闲 | 整个系统休眠 |
| 影响范围 | USB 局部低功耗 | 系统级低功耗 |
| USB clock | 关闭 | 关闭 |
| Wakeup 中断 | 开启 | 开启 |
| USB controller reset | 否 | 是 |
| Controller 配置 | 保留 | 丢失 |
| 链路上下文 | 保留 | 丢失 |
| Resume 后处理 | 恢复 clock，继续原链路 | 重新初始化，必要时重新枚举 |

---

## 2. USB Clock 关闭后的行为

USB clock 关闭后，USB controller 的正常协议处理逻辑停止工作：

- 正常 USB 中断无法触发；
- 无法处理 USB 协议包；
- 无法进行控制、批量、中断或等时传输；
- 不能依赖普通 USB 中断唤醒系统；
- 只有独立的 wakeup 检测逻辑可以继续工作。

Wakeup 中断只用于唤醒 USB controller 或系统，不负责处理 USB 协议。

---

## 3. Wakeup 唤醒逻辑

Wakeup 逻辑独立于 USB clock，通过检测 DP/DM 电平或线状态变化产生 wakeup 中断。

基本流程：

1. 进入低功耗前开启 wakeup 中断；
2. 关闭 USB clock；
3. Wakeup 模块持续检测 DP/DM；
4. DP/DM 电平或线状态发生变化；
5. 触发 wakeup 中断；
6. 恢复 USB 电源域和 USB clock；
7. 恢复或重新初始化 USB controller；
8. 重新读取 DP/DM 和端口状态；
9. 执行 resume、attach、disconnect 或重新枚举。

可能触发 wakeup 的线状态变化包括：

- Device 发起 remote wakeup，主动拉 K；
- Host 发出 resume signaling；
- Device 插入，上拉出现；
- Device 拔出或掉电，上拉消失;
- Host 发送sof;

具体支持哪些唤醒条件，以芯片 wakeup 模块能力和寄存器配置为准。

### 3.1 Wakeup 中断与普通 USB 中断

| 项目 | 普通 USB 中断 | Wakeup 中断 |
|---|---|---|
| 是否依赖 USB clock | 是 | 否 |
| USB clock 关闭后能否触发 | 否 | 是 |
| 触发来源 | USB 协议及 controller 事件 | DP/DM 电平或线状态变化 |
| 是否处理 USB 传输 | 是 | 否 |
| 主要用途 | 协议和数据处理 | 唤醒 controller 或系统 |

Wakeup 中断仅表示 DP/DM 发生了需要关注的变化。具体事件类型需要在 USB clock 恢复后，根据当前线状态进一步判断。

---

# 4. Device 模式场景

## 4.1 不发生物理拔出

初始状态：host 与 device 保持连接，双方均处于 runtime suspend。

### 场景 1：Host 主动 Resume

1. Host 主动退出 runtime suspend；
2. Host 恢复 USB 总线；
3. Device 检测到 resume signaling；
4. Device 退出 runtime suspend；
5. 双方继续使用原有配置通信。

连接和配置未丢失，不需要重新枚举。

### 场景 2：Device 有数据上行

1. Device 有数据需要上传；
2. Device 主动退出 runtime suspend；
3. Device 通过拉 K 发起 remote wakeup；
4. Host 的 wakeup 模块检测到 DP/DM 变化；
5. Host 退出 runtime suspend并恢复 USB clock；
6. 链路恢复后，device 上传数据。

该场景要求 host 已允许 device remote wakeup。

### 场景 3：Device 进入系统 Suspend

1. Device 进入系统 suspend；
2. USB clock 关闭，USB controller被 reset；
3. DP/DM 上拉丢失；
4. Host 检测到上拉消失，判定 device 断连。

此时从 host 视角看，device 已不存在，因此不存在 host 通过 USB 主动唤醒 device 的场景。

### 场景 4：Device 被其他唤醒源唤醒

1. Device 进入系统 suspend；
2. USB controller被 reset，DP/DM 上拉丢失；
3. Host 判定 device 断连；
4. Device 被其他唤醒源唤醒；
5. Device 重新初始化 USB controller；
6. Device 恢复 DP/DM 上拉；
7. 对 host 表现为一次新的设备插入；
8. Host 重新枚举 device。

Device 不需要关心此时 host 是否处于休眠状态。Host 是否支持被 USB attach 事件唤醒，属于 host 侧行为。

---

## 4.2 休眠期间拔出并重新插入

### 场景 1：Device 处于非断电态

Device 未掉电，DP/DM 检测能力保留。

1. 拔出前，host 与 device 处于连接状态；
2. 拔出后，device 进入 disconnect 状态；
3. Host 检测到上拉消失，判定断链；
4. Device 重新插入；
5. Host 检测到 attach；
6. 重新执行枚举。

### 场景 2：Device 处于断电态

Device 断电，DP/DM 同时掉电。

1. Device 进入断电状态时，DP/DM 上拉已经丢失；
2. Host 已判定 device 断连；
3. 此后执行物理拔出，不会产生新的有效状态变化；
4. 如果 device 仍以断电状态重新插入，DP/DM 上拉不会恢复；
5. Host 检测不到 attach，device 也无法仅通过 DP/DM 被唤醒。

该场景必须依赖以下唤醒源之一：

- VBUS；
- Type-C CC；
- GPIO；
- 其他外部唤醒源。

Device 被唤醒并初始化 USB controller后，恢复 DP/DM 上拉，host 才能检测到 attach 并开始枚举。

---

# 5. Host 模式场景

## 5.1 不发生物理拔出

初始状态：host 与 device 保持连接，双方均处于 runtime suspend。

### 场景 1：Host 主动 Runtime Resume

1. Host 主动退出 runtime suspend；
2. Host 恢复 USB clock 和总线；
3. Device 检测到 resume signaling；
4. Device 退出 runtime suspend；
5. 双方继续使用原有配置通信。

不需要重新枚举。

### 场景 2：Device 发起 Remote Wakeup

1. Device 因数据或其他事件需要恢复；
2. Device 主动拉 K；
3. Host 的 wakeup 模块检测到 DP/DM 变化；
4. Wakeup 中断触发；
5. Host 退出 runtime suspend；
6. Host 恢复 USB clock 和总线；
7. 双方继续使用原有连接和配置。

该场景要求 host 已允许 device remote wakeup。

### 场景 3：Host 从系统 Suspend 恢复

1. Host 进入系统 suspend；
2. USB clock 关闭，USB controller被 reset；
3. 原有 controller 配置和链路上下文丢失；
4. Host 被唤醒并执行系统 resume；
5. 重新初始化 USB controller；
6. 根据当前端口状态重新建立连接；
7. 重新枚举 device。

---

## 5.2 休眠期间拔出并重新插入

### 场景 1：Host 处于 Runtime Suspend

1. Host 与 device 保持连接；
2. Host 进入 runtime suspend；
3. Device 被拔出，DP/DM 线状态发生变化；
4. Wakeup 中断触发；
5. Host 恢复 USB clock；
6. Host 检测到断链；
7. Device 重新插入；
8. Host 检测到 attach 并重新枚举。

### 场景 2：Host 处于系统 Suspend

1. Host 与 device 保持物理连接；
2. Host 进入系统 suspend；
3. USB controller被 reset，原有配置丢失；
4. Device 被拔出，DP/DM 线状态发生变化；
5. Wakeup 中断触发，host 执行系统 resume；
6. Host 恢复 USB clock并重新初始化 USB controller；
7. Host 默认处于无设备连接状态；
8. Device 重新插入；
9. Host 检测到 attach 并重新枚举。

---

# 6. 结论

1. Runtime suspend 只关闭 USB clock，不 reset controller，原有配置和链路上下文保留。
2. 系统 suspend 会关闭 USB clock并 reset controller，原有配置和链路上下文丢失。
3. USB clock 关闭后，普通 USB 中断无法触发，USB 协议也无法继续处理。
4. 低功耗期间只能依靠 wakeup 模块检测 DP/DM 电平或线状态变化。
5. Wakeup 中断只负责唤醒；恢复 USB clock 后，才能确认实际的 USB 事件。
6. Runtime resume 后，如果连接未丢失，通常可以继续原链路，不需要重新枚举。
7. 系统 resume 后需要重新初始化 USB controller，并根据链路状态决定是否重新枚举。
8. Device 完全断电且 DP/DM 上拉无法恢复时，必须依赖 VBUS、Type-C CC 或其他外部唤醒源。