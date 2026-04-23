# 1 先给结论：MPS 和 MRRS 分别限制什么？
## MPS（Max Payload Size）
限制的是：单个 TLP 能携带的最大 payload（最关键影响对象是）：
* MemWr 的 payload 上限
* Completion with Data（CplD）的 payload 上限
直觉：

> MPS 决定“对端一次最多能回/能收多少数据”。

## MRRS（Max Read Request Size）
限制的是：单个 MemRd Request 允许请求的最大读取长度。
直觉：

> MRRS 决定“我一次最多能要多少数据”。

两者组合决定了在抓包里看到的形态：
* MRRS 大：device 一次 MemRd 要得多（请求少）
* MPS 大：host 每条 CplD 回得多（返回包少）
* 反之就会拆成很多小包 → 吞吐下降、开销变大
---
# 2 它们如何导致“拆包”（你前面(2)看到的多条 CplD）
以 device DMA read 读取 host memory 为例：
1. device 发 MemRd，请求长度 ≤ MRRS
2. host 需要回 CplD，但每条 CplD 的数据量 ≤ MPS
3. 所以：
* 若 MRRS > MPS：一个 MemRd 往往对应 多条 CplD（按 MPS 分段）
* 若 MRRS <= MPS：一个 MemRd 可能只需要 一条 CplD（不分段）
此外还有一个常见拆包原因（你做性能调优必须知道）：
* 4KB 边界：读请求一般不跨 4KB（很多实现/规则会把跨界请求拆开）
    * 这会让“看起来 MRRS 很大”但仍然拆成更多请求/返回
---
# 3 它们为什么影响性能？（驱动工程师能用的解释）
关键原因：TLP header/协议开销是“按包计”的
* 每个 TLP 都有 header、链路层开销、流控处理、交换/路由开销
* 包越碎，同样的数据要发更多包 → 有效吞吐下降
你可以用一个直觉图：
* 大 MPS/MRRS：少量大包 → header 占比低 → 吞吐高
* 小 MPS/MRRS：大量小包 → header 占比高 → 吞吐低
---
# 4 在 config space 里哪里看/改？（最关键的“驱动落地点”）
在 PCIe Capability（不是 extended）里常用两处：
## A) Device Control（DevCtl）里的 Max Payload Size（MPS）
* 这是“本设备愿意接收的最大 payload”
* 注意：链路两端最终能用多大，取决于两端都能接受（以及 RC/交换结构限制）
## B) Device Control（DevCtl）里的 Max Read Request Size（MRRS）
* 这是“本设备发起 MemRd 时单次最大请求长度”
* 设备作为 requester（DMA read 发起者）会遵守这个上限

> 常见误区： MPS 主要影响“我收包/对端回包”（尤其 Completion） MRRS 主要影响“我发读请求一次能要多大”
---
# 5 一个非常实用的“怎么看抓包推断 MPS/MRRS”
## 推断 MRRS（看 MemRd）
* 观察 device 发出的 MemRd request 的 Length 分布：
    * 总是 128B/256B/512B 附近的上限，很可能就是 MRRS 限制
    * 如果从不超过 256B，八成 MRRS=256B（或被 4KB 边界/实现策略限制）
## 推断 MPS（看 CplD / MemWr）
* 观察 host 回的 CplD payload 大小：
    * 若每条 CplD 最大只到 128B/256B/512B，那常常就是 MPS 上限
* 观察 device 写 completion ring（MemWr）是否也按某个 payload 上限分段（一般 completion entry 小，不明显）
---
# 6 驱动能不能“随便调大”MPS/MRRS？要小心
## MRRS：相对安全、经常可调（但也要看平台）
* 调大 MRRS 通常能减少 MemRd 数量（提升 DMA read 效率）
* 但受限于 RC/交换结构/设备能力
* 有些平台/固件会固定或限制它
## MPS：更敏感（因为影响“我能接多大包”）
* 把 MPS 调得比系统/设备链路实际能支持的更大，会带来错误或回退
* 实际链路可能受：
    * Root Port 限制
    * Switch 限制
    * IOMMU/RC 实现策略
    * 设备内部缓冲能力限制
> 工程建议（驱动侧常用策略）： 遵循 OS/固件枚举后的设置通常最稳 只有在明确知道平台策略且做过验证时，才在驱动里强改
---
# 7 Gen3 性能调优里，MPS/MRRS 典型取值和现象
常见取值（经验，不是硬规定）：
* MPS：128B / 256B（很多系统默认 128 或 256）
* MRRS：256B / 512B / 1024B（很多高性能设备倾向更大）
典型现象：
* MRRS 太小：device 读大 buffer 时 MemRd 请求非常多，CplD 也非常碎 → 吞吐掉
* MPS 太小：即便 MRRS 很大，host 回 CplD 仍然每条很小 → Completion 数量多 → 吞吐掉
* 跨 4KB 边界频繁：请求被拆开，包数上升
---
# 8 最该记住的 3 句口诀
1. MRRS 控制“我一次读请求能要多大”（MemRd 的长度上限）
2. MPS 控制“单包最多带多少数据”（MemWr/CplD 的 payload 上限）
3. 包越碎 header 越多，吞吐越差（拆包就是性能杀手）