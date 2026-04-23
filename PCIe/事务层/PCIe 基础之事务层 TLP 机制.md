# 1 为什么 MemRd 必须有 Completion（而 MemWr 通常没有）
• MemWr（写）：写包自己带 payload，发出去就结束（posted write），一般不要求对端确认。
• MemRd（读）：读请求包不带数据，只是“请把地址 X 的 N 字节给我”。
    数据只能由对端（RC/内存子系统 或 device）用 Completion with Data（CplD） 回来。
所以 MemRd 的事务结构天然是两段：
1. MemRd Request（请求）：Addr + Length + Requester ID + Tag
2. CplD（返回）：Tag + Requester 信息 + Byte Count/Lower Address + Data
---
# 2 怎么把“返回的数据”匹配回“哪一次读请求”？——Requester ID + Tag
## MemRd 请求里你关心的匹配字段
* Requester ID（BDF）：发起读请求的功能号
* Tag：该请求的“事务编号”

> 直觉：Requester ID 表示“回给谁”，Tag 表示“回的是谁发的哪一单”。

## Completion 里带回哪些匹配信息？
### Completion（Cpl/CplD）会携带：
* Requester ID（指明这是回给哪个 requester 的）
* Tag（指明对应哪个 MemRd 请求）
* 以及一些“片段定位信息”（Byte Count / Lower Address，下面讲）
### 驱动工程师要记住的关键行为：
* 一个 device 可以同时发很多 MemRd（DMA read）并挂起等待返回
* Tag 让 device 能区分并发的多个读请求
* 只要 Tag 不冲突，Completion 可以交错返回也没关系
---
# 3 为什么一个 MemRd 可能收到“多条 CplD”？——拆包与分段返回
发一个 MemRd，Length 可能很大（比如读 512B、4KB）。
对端返回时通常受 Max Payload Size（MPS） 等限制，无法用一条 CplD 把全部数据塞回来，就会拆成多条。
这时 Completion Header 里两个字段很关键：
## A) Byte Count：这条 Completion“总共还剩/要返回多少字节”（概念上）
抓包工具常显示 Byte Count，你可以用它理解：
* 这条 CplD 覆盖的是原请求的哪一部分、返回了多少
## B) Lower Address：这条 CplD 对应原请求地址的低位偏移
当一次 MemRd 被拆成多条 CplD：
* 每条 CplD 都会用 Lower Address 指示“从原请求地址的哪个偏移开始返回”
* 再配合 Byte Count/Length 就能拼回完整数据

> 可以把 Lower Address 理解为：“这条返回数据在原请求中的起始位置”。

---
# 4 拆包的常见原因（你不用背细节，知道方向就行）
1. MPS 限制：CplD 单包 payload 有上限
2. MRRS 限制：Requester 的单次 MemRd request 也有限（更像“你一次最多能要多少”）
3. 边界限制：某些实现/规则下不能跨某些边界（比如 4KB 边界常见约束）
4. 系统/RC 实现策略：即使理论上能回一大包，也可能选择分多包回
这些会在(3)里详细讲（MPS/MRRS 是性能核心），你现在先记“拆包是常态”。
---
# 5 Completion 会乱序吗？——两层概念别混
这里需要分清两个“乱序”：
## (1) 同一个 MemRd 请求的多条 CplD：顺序通常可预测，但不要依赖
多数实现会按地址递增返回，但规范层面你不该把“必须按顺序到达”当成绝对保证。更安全的拼装方式是：
* 依赖 Tag + Lower Address/Byte Count 组合去重组，而不是假设顺序。
## (2) 多个不同 Tag 的 MemRd：Completion 可以交错返回
这是更常见的“乱序”来源：
* device 发了 Tag=5 读 descriptor，Tag=6 读 data
* host 可能先回 Tag=6 的某部分，再回 Tag=5（取决于内存/RC）
* 这完全正常，因为 Tag 就是为这种并发准备的

```go
驱动视角：你通常不直接处理这些 Tag，因为 DMA engine/endpoint 硬件会处理拼装；但你看协议分析仪时要能接受“交错返回”是正常现象。
```
---
# 6 把 MemRd ↔ CplD 套回你最关心的三条路径（超实用）
## A) device 读 descriptor（DMA read ring）
* device 发 MemRd（带 Tag）
* host 回 1~N 条 CplD（同 Tag，Lower Address 指示分段）
* device 组装出 descriptor
## B) device 读 TX data buffer
* 同理：MemRd + CplD（可能拆多包，尤其大包）
## C) host 读设备 BAR 寄存器（MMIO read）
* host 发 MemRd（Requester=host/RC）
* device 回 CplD（数据=寄存器值）
* 一般很小（1DW），不会拆包
---
# 7 你作为驱动工程师最该用它来解释什么现象？
1. DMA read 性能不如预期：可能 MRRS 太小、Completion 拆太碎（(3)会讲）
2. 协议分析仪里看到大量 CplD：不代表异常，多数是正常的拆包返回
3. 看到 Completion Status 非成功：才值得警惕（可能是 UR/CA 等错误）