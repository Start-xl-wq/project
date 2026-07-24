
| 里程碑          | Action                             | 原理                                                                  | 完成目标                                               |
| ------------ | ---------------------------------- | ------------------------------------------------------------------- | -------------------------------------------------- |
| **M0 环境就绪**  | 跑 setup_gadget.sh，确认 UDC 绑定        | configfs gadget 必须先绑定 UDC，functionfs 才能被 host 枚举                    | dmesg 看到 gadget bind，无报错                           |
|              | host 确认枚举速度                        | USB3 SuperSpeed 才有 ~500 MB/s 物理上限，HS 只有 ~60 MB/s                    | lsusb -v 看到 bcdUSB 3.00                            |
|              | 两端编译运行不崩溃                          | 基础连通性验证，排除环境问题再谈带宽                                                  | device 打印 ENABLE，host 打印带宽数字                       |
| **M1 基线**    | mode=tx，默认参数跑通                     | 先跑通再调参，避免带宽低时不知道是 bug 还是参数问题                                        | 无 AIO/libusb 报错，稳定输出数字                             |
|              | 记录基线 block=512K, queue=8/16        | 有基线才能量化后续每步调参的收益                                                    | 基线 ≥ 200 MB/s                                      |
| **M2 调参**    | device queue-depth: 8→16→32        | AIO 队列越深，URB 空窗期越短；但过深会增加内存压力和调度开销，存在拐点                             | 找到带宽不再增长的拐点                                        |
|              | block-size: 512K→1M→2M             | 块越大，单次 URB 提交的数据量越多，协议头开销占比越低；但过大会增加延迟和内存占用                         | 找到最优 block size                                    |
|              | host queue-depth: 16→32→64         | host 侧 libusb transfer 队列必须足够深，否则 host 成为瓶颈而不是 device 或链路           | host 不成为瓶颈                                         |
|              | 检查 CPU 占用（top）                     | AIO 本身是异步的，但填充 payload 是同步 CPU 操作；CPU 打满会导致 io_submit 不及时，出现 URB 空窗 | CPU < 50%；若打满则改 payload 只在 init 时初始化，每次只更新头部 16 字节 |
|              | 单向稳定验证 60s                         | 短时峰值无意义，需要持续稳定                                                      | device→host ≥ 380 MB/s，无抖动                         |
| **M3 稳定性**   | host 开校验跑 5 分钟                     | seq 连续性校验能发现丢包、乱序、URB 截断等隐性问题                                       | verify err: 0，seq 无断层                              |
|              | 观察 stddev                          | 每秒带宽样本的标准差，反映传输是否平稳；stddev 大说明有周期性卡顿（常见原因：UDC 内部 URB 池耗尽、host 调度抖动） | stddev < 10 MB/s                                   |
|              | 检查 dmesg                           | xhci/dwc3 的 error/warning 是硬件层问题的直接信号，软件层调参无法解决                     | 无 xhci/dwc3 error/warning                          |
|              | --duration 300                     | 5 分钟平均值排除热身效应，反映真实稳态带宽                                              | 平均 ≥ 380 MB/s                                      |
| **M4 冲 400** | payload 只在 init 时初始化，每次只更新头部 16 字节 | 消除每块逐字节填充的 CPU 开销，让 AIO 提交更及时，减少 URB 空窗期                            | +5~10 MB/s                                         |
|              | taskset -c 绑核                      | 避免进程在核间迁移导致 cache miss 和调度延迟，AIO 回调更及时                              | +3~8 MB/s                                          |
|              | 调 dwc3 num_requests sysfs          | dwc3 driver 内部 URB 池深度默认较小，增大后 driver 层不会因池耗尽而停发                    | +5~15 MB/s（硬件相关）                                   |
|              | host --no-verify 跑极限               | 去掉 seq 检查的 CPU 开销，测出链路物理上限                                          | 峰值 ≥ 400 MB/s                                      |
|              | 换 Gen2 认证线（若未达标）                   | 物理层信号质量直接决定能否稳定跑 SS；劣质线会导致频繁 link recovery，带宽骤降                     | 排除物理层瓶颈                                            |
| **M5 双向+收尾** | --mode both 双向同跑                   | USB3 理论上全双工不互压，但部分 UDC 实现共享内部带宽，需实测验证                               | 两方向各自 ≥ 单向 60%                                     |
|              | 整理最优参数写入 README                    | 固化结论，可复现                                                            | 一行命令复现 400 MB/s                                    |