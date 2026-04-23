# 1) 链路层的职责
## (1) 可靠传输：ACK/NAK + 重放（Replay）
* 对每个发送出去的 TLP（更准确：Data Link 层的帧），链路层用 **序号（Sequence Number）**标记。
* 接收端校验通过后发 ACK DLLP；校验失败/丢包发 NAK DLLP（或超时没 ACK）。
* 发送端把“未被 ACK 的包”放在 Replay Buffer，必要时重传（replay）。
你作为驱动最常遇到的现象：
* 链路速率不变但吞吐下降/抖动 → 可能 replay 增多（误码/边界）
* AER correctable 上升、并伴随 retrain/降速 → 链路质量不足导致频繁重传
## (2) 流控：Credit（FC DLLP）
* 接收端告诉发送端：“我还有多少缓冲能接 posted/NP/Cpl 的 header/data”
* 发送端没 credit 就必须停（stall），直到收到新的 FC update DLLP
驱动侧可解释的现象：
* 大量并发 MemRd 导致 completion 回来慢 → 可能 CplD credit 成瓶颈
* 某些 workload（小包高频 doorbell/完成）吞吐不如预期 → 包太碎 + credit 频繁耗尽
## (3) 链路层错误检测/上报：LCRC 等
* 链路层对接收到的帧做校验（LCRC），错了就 NAK → replay
* 统计错误/触发上层错误报告（配合 AER/PCIe 设备状态）
---
# 2) DLLP 在链路层里常见的几类
* ACK/NAK DLLP：告诉对端“你发的序号 X 我收到了/没收好”
* Flow Control Update DLLP：更新 credit（PH/PD/NPH/NPD/CplH/CplD）
* （还有电源管理/厂商等 DLLP，驱动通常不直接碰）
你可以把 DLLP 当成链路层内部的“运输控制包”，对驱动透明，但会决定性能和稳定性。
---
# 3) 驱动能看到什么、看不到什么？
## 3.1 标准 config space 里，你能侧面看到的链路层信息
你通常从 PCIe Capability 和 AER 间接判断链路层问题：
* LnkSta：当前 speed/width（链路是否降级是最强信号）
* AER correctable / uncorrectable 计数（错误趋势）
* Device Status/错误报告相关位（具体平台差异）
但要注意：这些是“结果/症状”，不是 DLLP/ACK/credit 的实时细节。
## 3.2 真正的“链路层细节”（ACK/NAK/replay/credit）在哪里？
* 协议分析仪：能直接看到 FC DLLP、ACK/NAK、replay
* 控制器/PHY 私有寄存器：很多 SoC/IP 会提供：
    * replay counter、NAK counter、ACK timeout
    * FC stall 次数、credit 水位
    * LCRC error counter
        这些对排障非常关键，但不属于 PCI 标准寄存器。
---
# 4) 驱动工程师需要掌握的“链路层 → 现象”对照表
## A) replay 频繁（NAK 多）
现象
* 吞吐下降/延迟抖动
* 有时伴随 AER correctable 增长
* 严重时触发 retrain/降速（回退 Gen2）
典型根因
* Gen3 信号裕量不足（ISI/抖动/噪声）
* 省电状态切换干扰（ASPM/L1SS）
驱动能做的
* A/B：禁用 ASPM/L1SS、改变 retrain 时机
* 必要时限制到 Gen2 作为稳定性策略
## B) credit 成瓶颈（FC stall）
现象
* 链路没降速但吞吐“上不去”
* 小包/高频 doorbell + completion workload 更明显
* 读请求并发高时 completion 回来慢（CplD credit 紧）
典型根因
* 对端缓冲/credit 配置较小
* 请求粒度太碎（MRRS/MPS、小 IO）
驱动能做的
* 增大 batch（减少 doorbell 次数）
* 合理队列深度/并发
* 调整 MRRS/MPS（通常由 OS 设置，驱动谨慎介入）
* 中断合并/减少 completion 打断频率（MSI-X 合并策略）
---
## 5) 驱动“间接影响链路层”的几种杠杆
* 请求并发度：发多少 outstanding MemRd（队列深度越深，completion/credit压力越大）
* 请求粒度：MRRS/MPS、descriptor 合并、批处理策略
* 中断策略：MSI-X 合并/节流，避免“包太碎 + 中断太多”把系统压垮
* 省电策略：ASPM/L1SS 会影响链路稳定性，间接影响 replay
* 错误恢复策略：检测到错误趋势后，是否触发 retrain、是否降速保底
---
# 6) 应该形成的链路层“驱动心智模型”
1. TLP 是想干的事（读写/完成/doorbell/MSI）
2. 链路层保证 TLP 可靠送达：ACK/NAK + replay
3. 链路层控制发送速率：credit 流控
4. 在驱动里主要靠 现象 + 计数器做推断，并用 策略（并发/粒度/省电/重训/降速）做收敛