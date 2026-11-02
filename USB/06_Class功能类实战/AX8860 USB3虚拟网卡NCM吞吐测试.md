- 当前状态：NCM 虚拟网卡的 TCP 吞吐，主机为 Windows 10。USB3 默认配置上行 1 流 2.16~2.22、下行 1 流 1.89~1.91 Gbit/s；调优后上行 1 流最高 3.20、下行 1 流最高 3.69、下行 4 流最高 3.19 Gbit/s，三个最高值在不同配置里。USB2（high-speed）各方向 332~374 Mbit/s。稳定性测了一夜（NTB 63 KB / 63 KB）和约 60 小时（NTB_OUT 16 KB / NTB_IN 63 KB），吞吐、链路、内存没有下降趋势，两个配置的 dmesg 里都有未定位的 NTB 解析报错。其他主机、RNDIS 未测试。各配置的数值和配置项说明见第 1 节
- 最近更新：2026-11-02

## 1. 当前结论

范围：AX8860 EVB 的 USB 虚拟网卡（NCM），主机为 Windows 10，测的都是 TCP 吞吐（iperf3）。NCM 的 UDP 只有参考数据（4.7），RNDIS 未测试。

### 1.1 结论

- 默认配置（USB3，MTU 1500，NTB 16 KB，RPS 关）：上行 1 流 2.16~2.22 Gbit/s，下行 1 流 1.89~1.91，下行 4 流 1.30。
- 调优后 USB3 最高：上行 1 流 3.20，下行 1 流 3.69，下行 4 流 3.19 Gbit/s，三个最高值不在同一个配置里。
- 提升来源（实测）：RPS 提高下行 4 流；NTB_OUT 加大提高上行；NTB_IN 加大提高下行 1 流；MTU 由 1500 改为 7986（jumbo）整体提高。
- 取舍：jumbo 下 NTB_OUT 由 16 KB 加到 63 KB，上行提高，下行 1 流小幅提高，下行 4 流降低约 6%（3.15 → 2.96）；NTB_IN 加大只提高下行 1 流，不影响上行。
- 4 流与 1 流：上行两者相差不超过 0.03（默认配置的区间除外）；下行在 jumbo 且 NTB_OUT 不小于 42 KB 时 4 流低于 1 流。NTB_OUT 在 48 KB 以上时，已测的上行和下行 1 流都不低于 3.09，只有下行 4 流不到 3.0（2.62~2.96）。
- 主机 MTU 保持 1500 时，NTB_OUT 与 NTB_IN 都加到 63 KB 也有提升：上行 1 流 +20.7%，下行 1 流 +4.8%，下行 4 流 +7.8%。
- 稳定性：jumbo 下 NTB 63 KB / 63 KB 过夜 11 小时、NTB_OUT 16 KB / NTB_IN 63 KB 连续约 60 小时，吞吐、链路、内存无下降趋势；两个配置的 dmesg 都有成对的 NTB 解析报错（后者累计 11232 次），原因未定位，NTB_OUT 63 KB 不是必要条件；11-02 的短测里 jumbo 的各配置（含 NTB 均为 16 KB）都有新增，MTU 1500 和 USB2 的区间合计 1 次（4.16、4.18、4.19）。
- USB2（high-speed，MTU 1500）：332~374 Mbit/s。
- 局限：只测了 Windows 10 一台主机；除表 1 第 15 行（72 轮）外，每格 1 到 5 次测量，没有多轮统计。

### 1.2 表 1：USB3（SuperSpeed）NCM，TCP

速率单位 Gbit/s，MTU 单位字节，NTB 大小单位 KB。NA 表示没有数据。每一列的含义见 1.4。

| # | MTU（字节） | NTB_OUT（KB） | NTB_IN（KB） | RPS | 上行 1 流（Gbit/s） | 上行 4 流（Gbit/s） | 下行 1 流（Gbit/s） | 下行 4 流（Gbit/s） | 来源 |
|---|---|---|---|---|---|---|---|---|---|
| 1 | 1500 | 16 | 16 | 关 | 2.16~2.22 | 2.10~2.20 | 1.89~1.91 | 1.30 | 10-27，4.6 |
| 2 | 1500 | 16 | 16 | 开 | 2.29 | 2.29 | 1.86 | 1.90~2.14 | 10-22、10-23，板端计数，4.2、4.4 |
| 3 | 7986 | 16 | 16 | 关 | 2.44 | 2.42 | 3.01 | 2.47 | 10-28，4.9；11-02，4.19 |
| 4 | 7986 | 16 | 16 | 开 | 2.40 | 2.42 | 3.02 | 3.15~3.19 | 10-28，4.9；10-29，4.13；11-02，4.19 |
| 5 | 7986 | 32 | 16 | 开 | 2.89 | 2.88 | 3.09 | 3.12 | 10-28，4.10 |
| 6 | 7986 | 63 | 16 | 开 | 3.20 | 3.19~3.20 | 3.13~3.15 | 2.92~2.96 | 10-28，4.11；10-29，4.13、4.14 |
| 7 | 1500 | 63 | 16 | 开 | 2.68 | 2.68 | 1.86 | 2.08 | 10-28，4.12 |
| 8 | 7986 | 42 | 16 | 开 | 2.98 | NA | 3.13 | 3.01 | 10-29，4.13；11-02，4.19 |
| 9 | 7986 | 48 | 16 | 开 | 3.11 | 3.09 | 3.12 | 2.96 | 10-29，4.13；11-02，4.19 |
| 10 | 7986 | 63 | 32 | 开 | 3.20 | 3.19 | 3.42~3.48 | 2.62~2.67 | 10-29，4.14；11-02，4.19 |
| 11 | 7986 | 63 | 48 | 开 | NA | 3.20 | 3.58 | 2.86 | 10-29，4.14；11-02，4.19 |
| 12 | 7986 | 63 | 63 | 开 | 3.20 | 3.19 | 3.64 | 2.94 | 10-29，4.14；11-02，4.19 |
| 13 | 1500 | 63 | 63 | 开 | 2.68 | 2.67 | 1.97 | 2.22 | 10-29，4.15；11-02，4.19 |
| 14 | 1500 | 16 | 16 | 开 | 2.22 | 2.21 | 1.88 | 2.06 | 10-29，4.15（Windows 汇总值，与第 2 行同配置）；11-02，4.19 |
| 15 | 7986 | 16 | 63 | 开 | 2.36~2.43 | 2.42 | 3.57~3.69 | 2.95~3.12 | 10-30 到 11-02，72 轮，4.18；11-02，4.19 |

### 1.3 表 2：USB2（high-speed）NCM，TCP

速率单位 Mbit/s，其余口径同表 1，每格 1 次。

| # | 链路 | MTU（字节） | NTB_OUT（KB） | NTB_IN（KB） | RPS | 上行 1 流（Mbit/s） | 上行 4 流（Mbit/s） | 下行 1 流（Mbit/s） | 下行 4 流（Mbit/s） | 来源 |
|---|---|---|---|---|---|---|---|---|---|---|
| U1 | high-speed | 1500 | 16 | 16 | 开 | 336 | 332 | 374 | 371 | 10-30，4.17；11-02，4.19 |
| U2 | high-speed | 1500 | 63 | 63 | 开 | 349 | 350 | 369 | 369 | 10-30，4.17；11-02，4.19 |

- 10-30 在同一块板、同一固件上用 configfs 把 gadget 限到 high-speed（3.9），不是 USB2 版 DTS。
- 只测了 MTU 1500。high-speed 下的 jumbo（MTU 7986）没测，不是不能改（见 1.4 的 MTU）。
- USB3 同配置（表 1 第 14 行）折成同一单位为 2220 / 1880 / 2060 Mbit/s。

### 1.4 配置项说明

**上行与下行（以主机为基准）**

- 上行：主机（Windows）发、板子收，iperf3 客户端不带 `-R`。下行：板子发、主机收，带 `-R`。
- 对应 NCM 的两个方向：上行用 NTB_OUT（主机发给设备的 NTB），下行用 NTB_IN（设备发给主机的 NTB）。
- 表里的速率取 Windows 端 iperf3 输出里的 receiver 行，也就是接收端实际收到的速率（上行是板子收到的，下行是 Windows 收到的）。板端 `ncm0` 的计数另作交叉核对，一般比 Windows 汇总值略高（见 1.5）。

**单流与 4 流**

- iperf3 的 `-P N` 同时开 N 条 TCP 连接，表里 4 流的速率是四条之和（`[SUM]`）。
- 分开测是因为瓶颈不同（实测）：下行 1 流受板端 iperf3 发送线程所在的核限制（该核 sys 约 88% 到 90%、idle 0%）；默认配置下 4 流下行受 CPU1 的收包软中断限制（CPU1 idle 1% 到 2%），开 RPS 后缓解。
- 上行 1 流和 4 流的数值很接近：默认配置的区间最多相差约 0.1，其余各行相差不超过 0.03。

**MTU（含 jumbo）**

- 是什么：网络接口一次能发的最大 IP 包长度，不含 14 字节以太网头。默认 1500；jumbo 指 7986。
- 怎么改：板端 `ncm0` 的 MTU 要和设备端 NCM 的 `max_segment_size` 一起改。`max_segment_size` 是设备上报给主机的最大以太网帧长度，默认 1514（= 1500 + 14），源码上限 8000，对应 MTU 7986（= 8000 - 14）。按 `f_ncm.c`，`ncm0` 的 MTU 只在第一次创建网卡时按 `max_segment_size - 14` 初始化，之后改 `max_segment_size` 不会自动改 MTU，所以要再手动 `ip link set ncm0 mtu`；设备端收到长于 `max_segment_size` 的数据报会丢弃。步骤见 3.2。
- 上限 8000 的来源：`f_ncm.c` 的注释说，段长大于等于 8192 时，16 KB 的 NTB 聚合不了两个包，所以限制在 8000；注释里没有提 USB 速度，USB2 下也能设（high-speed 下没测）。
- 改了有什么影响（实测，Windows 10）：NTB_OUT 固定 63 KB 时，MTU 由 1500 改为 7986，上行 1 流 2.68 → 3.20，上行 4 流 2.68 → 3.19~3.20，下行 1 流 1.86 → 3.13，下行 4 流 2.08 → 2.92~2.95 Gbit/s。NTB_OUT 为 16 KB 时，上行包速率由约 19 万降到约 3.8 万包/秒（5 倍），吞吐只涨约 10%（2.16~2.22 → 2.40~2.44）。
- 主机是否感知或被改变：板端写入 8000 并重新枚举后，Windows 的 `NlMtu` 自己变成了 7986，没有手动设置，机制没有查清。板端重启后 `max_segment_size` 回到 1514，Windows 保持 1500。Linux、Win11、macOS 是否也这样没测；两端 MTU 不一致时的表现没测。`max_segment_size` 和 MTU 都是运行时设置，重启后丢失。

**NTB_OUT 与 NTB_IN**

- 是什么：NCM 把多个以太网帧打包成一个 NTB（NCM Transfer Block）经 USB 批量端点传输。设备在枚举时向主机报两个上限：`dwNtbOutMaxSize`（主机发给设备的 NTB，上行）和 `dwNtbInMaxSize`（设备发给主机的 NTB，下行），默认都是 16 KB。
- 怎么改：NCM function 的 configfs 属性 `ntb_out_size`（2048 到 65535）和 `ntb_in_size`（16384 到 65535，只允许调大），本地补丁，未提交。表里 1 KB = 1024 字节，上限 65535 字节，所以最大取 63 KB（64512）；42 KB 即 43008、48 KB 即 49152。步骤见 3.3、3.7。
- 改了有什么影响（实测，Windows 10）：
  - NTB_OUT 加大，上行升高：jumbo 下 16 / 32 / 42 / 48 / 63 KB 为 2.40 / 2.89 / 2.98 / 3.11 / 3.20 Gbit/s；MTU 1500 下 63 KB 为 2.68（16 KB 为 2.16~2.22）。jumbo 下下行 4 流随之下降后趋平：3.15~3.19 / 3.12 / 3.01 / 2.96 / 2.92~2.96；同一次开机内只改 `ntb_out_size`，16 KB 为 3.15、63 KB 为 2.96（约 -6.0%），原因没有定位。下行 1 流：MTU 1500 下基本不变（1.86~1.88），jumbo 下小幅上升（16 KB 3.02，32 KB 3.09，42 KB 以上 3.12~3.15，4.19）。
  - NTB_IN 加大，下行 1 流升高：jumbo 且 NTB_OUT 63 KB 时，16 / 32 / 48 / 63 KB 为 3.15 / 3.42~3.48 / 3.58 / 3.64；上行 1 流不变（3.20）；下行 4 流不单调，2.94 / 2.62~2.67 / 2.86 / 2.94，32 KB 偏低并复现。
  - 板端每个下行 tx 计数的平均字节数约等于 NTB_IN，说明 Windows 10 采纳了设备上报的 IN 大小。
- 主机是否感知：感知，这两个值是设备在枚举时报给主机的；Windows 10 的 UsbNcm 在板端重新枚举后采用新值。Linux、Win11、macOS 是否采用没测。
- 代价：设备端每个发送 NTB 按 `ntb_in_size` 整块 `alloc_skb(GFP_ATOMIC)`，NTB_IN 63 KB 时是 64 KB 级的连续分配，失败会计入 `tx_dropped`。10-29 夜里 11 小时（63 KB / 63 KB）660 个样本 `tx_dropped` 全为 0，只测了这一个配置（4.16）。

**RPS**

- 是什么：Linux 的 Receive Packet Steering，只管“收到的包”的协议栈处理在哪个核上做。这棵内核的 `u_ether.c` 在接收完成后直接调用 `netif_rx` 把包交给协议栈（文件里没有 workqueue 或 tasklet）；RPS 打开时，按流哈希从 `rps_cpus` 里选一个 CPU，把包放进它的 backlog 队列，再用 IPI 唤醒那个 CPU 做协议栈处理（软中断）。RPS 关闭时（值为 0，默认）在收中断的那个核上处理（`Documentation/networking/scaling.rst`）。
- 所以它基本只是把这段软中断分走：USB 中断本身和 NCM 解包（在 `netif_rx` 之前）仍在原来的核上，发送路径也没有被它直接分发。下行时板子收到的是 ACK，ACK 的处理被分到其他核；ACK 触发的 TCP 发包和 NCM 封包也跟着在那个核上执行，这点是推断，和实测的 CPU1 idle 回升一致，没有单独验证。
- 配置：`/sys/class/net/ncm0/queues/rx-0/rps_cpus`，值是 CPU 位图，命令见 3.6。表里关 = 0；开 = `fffc`，对应 CPU2 到 CPU15，不含 CPU0 和 CPU1。内核文档建议中断频率高时把中断所在的核从掩码里去掉；默认配置下 USB 中断在 CPU1（10-22、10-23 采样）。选这个掩码的依据没有记录。
- 从 0 改成 `fffc` 的变化（实测）：4 流下行明显提升。MTU 1500 下 1.23~1.30 → 1.90~2.14（后者为板端计数），同时 CPU1 idle 由 1% 到 2% 回到 42% 到 47%（4.2、4.4）；jumbo 加 16 KB 下 2.47 → 3.15~3.19，CPU1 idle 由 1% 变 13%（4.9）。上行没有作用，单流下行也没有（jumbo 16 KB：上行 4 流关、开都是 2.42，下行 1 流 3.01、3.02，4.19）。
- 主机是否感知：不感知，这是板端内核的本地设置，不经过 USB。运行时设置，板子重启后丢失。

### 1.5 数据说明

- 格内写法：NA 表示没有数据（没测，或测了但输出没有贴回，见 4.19）；“a~b”是多次测量的最小值到最大值，只测 1 次或几次结果相同的写一个数；除第 15 行外每格 1 到 5 次，没有多轮统计；10-23 同配置两次曾相差约 11%。第 15 行是 72 轮长跑的最小~最大（每相 600 秒，不带 `-O`），与其他行的 20 秒、`-O 3` 不同。
- 表 1 第 2 行的数值来自板端 `ncm0` 计数，不是 Windows 汇总值。10-28 同口径对比，MTU 1500 下板端计数比 Windows 汇总值高：上行约 2.7% 到 3.0%，下行约 4.8% 到 5.1%；jumbo 下相差不到 2%。所以第 2 行不能与其他行直接相减。第 14 行与第 2 行同配置，是 Windows 汇总值，口径与其他行一致。
- 第 1 行和第 2 行不是同一天、同一固件，线长也不同。
- 10-28 同日有几条被中途打断的单次数据，RPS 状态未记录，没有放进表。
- 第 5 到 15 行没有测 RPS 关。
- 10-29 的数据（第 4、6 行里的 10-29 值，以及第 8 到 14 行）的 NTB_OUT、NTB_IN 由板端 `ntbset` 设置；NTB_IN 在 10-28 及之前固定为 16 KB。
- 命令、配置值和第 4 节各日的原始记录仍保留当时的字节数写法（例如 64512）。
- 主机只测了 Windows 10 的 UsbNcm。Linux `cdc_ncm`、Win11、macOS 没测，它们是否跟随设备上报的 NTB 大小、是否自动调整 MTU，都没验证。

## 2. 测试环境

| 项目 | 内容 |
|---|---|
| 板端 | AX8860 EVB（eMMC），16 × Cortex-A55，Linux 6.18.33，glibc，DDR 3200 |
| 固件 | 完整 SDK 固件 + USB3/I2C1 DTB；boot 分区开启 `CONFIG_USB_CONFIGFS_NCM=y`（RNDIS 原本已开启）；soc 分区包含 `/soc/scripts/usb-ncm.sh`、`/soc/scripts/usb-rndis.sh` |
| USB 链路 | UDC `2e1f0000.dwc3`，USB3 SuperSpeed（5 Gbps，Gen1）；10-30 的 USB2 测试在同一块板、同一固件上用 configfs 把 gadget 限到 high-speed（480 Mbps），不是 USB2 版 DTS（3.9） |
| Gadget | configfs，接口 `ncm0` / `rndis0`；NCM 默认参数为 qmult 5（10 个 USB 请求）、NTB IN/OUT 各 16 KB、`max_segment_size` 1514、MTU 1500；10-28、10-29 的改动见 3.2、3.3 |
| 主机 | Windows 10（build 19044），NCM 驱动 UsbNcm Host Device `10.0.19041.1`（`usbncm.inf`，Microsoft，手动选择）；Intel USB 3.20 可扩展主机控制器（`DEV_7EC0`），NCM 设备直接挂在其根集线器下；18 个逻辑处理器；网卡名“以太网 5”，地址 192.168.77.3 |
| 工具 | iperf3；板端 3.20（服务端 `192.168.77.2`），Windows 为 Cygwin 版 3.22（客户端） |
| 线缆 | 10-22、10-23 线长未记录；10-27 使用 0.25 m 线；10-28、10-29 未记录 |



说明：

- Windows 网卡属性显示的“3.8 Gbps”是 NCM gadget 在 SuperSpeed 下固定上报的值（`u_ether.h` 中 `gether_bitrate()` 返回 3,750,000,000），只表示设备按 USB3 枚举，不代表实测能力。
- Win10 自带 UsbNcm 驱动，但微软只对 Win11 正式支持 NCM。Win10 上需要在设备管理器里手动选择 “UsbNcm Host Device”。

## 3. 测试方法

### 3.1 启动 NCM

板端（使用调试 UART 串口操作，不要用 USB ACM 串口，因为下一步会停掉 ACM）：

```sh
/soc/scripts/usb-acm.sh stop
/soc/scripts/usb-ncm.sh start
cat /sys/class/udc/2e1f0000.dwc3/current_speed   # 应为 super-speed
ip addr add 192.168.77.2/24 dev ncm0
ip link set ncm0 up
iperf3 -s > /tmp/iperf-srv.log 2>&1 &
```

Windows 上把 USB 网卡设为同一网段的静态 IP，不填网关，先确认能 `ping 192.168.77.2`。

### 3.2 jumbo（max_segment_size 加 MTU）

先停 gadget、删 function 链接，再写值，否则写入返回 `EBUSY`：

```sh
G=/sys/kernel/config/usb_gadget/usb_ncm
/soc/scripts/usb-ncm.sh stop
rm $G/configs/c.1/ncm.usb0
echo 8000 > $G/functions/ncm.usb0/max_segment_size
cat $G/functions/ncm.usb0/max_segment_size       # 必须读回 8000
/soc/scripts/usb-ncm.sh start
ip link set ncm0 mtu 7986
ip addr add 192.168.77.2/24 dev ncm0
ip link set ncm0 up
```

Windows 端检查：

```powershell
Get-NetIPInterface -AddressFamily IPv4 | Format-Table InterfaceAlias,NlMtu
ping -f -l 7000 192.168.77.2
```

- 源码约束：`max_segment_size` 默认 1514，上限 8000（`f_ncm.c` 的 `MAX_DATAGRAM_SIZE`），对应 MTU 最大 7986。
- 板端写入 8000 并重新枚举后，Windows 的 `NlMtu` 读回 7986，没有手动设置；机制没有查清。重启后 `max_segment_size` 回到 1514，Windows 保持 MTU 1500，此时 `ping -f -l 7000` 报 “Packet needs to be fragmented but DF set”，所以每次烧录或重启后都要重做这一步。

### 3.3 修改 NTB_OUT 与 NTB_IN

`drivers/usb/gadget/function/f_ncm.c` 里的 `NTB_OUT_SIZE`（默认 16384）决定设备上报的 `dwNtbOutMaxSize`，同时是 rx 请求缓冲长度的下限（`u_ether.c` 的 `rx_submit`：`size = max(size, fixed_out_len)`）。10-28 依次改为 32768 和 64512，重编后烧录 boot 分区。NTB16 的块长度字段是 16 位（`ndp16_opts.block_length = 1`），上限 65535，64512 是 1024 的整数倍里不超过上限的最大值。

10-29 起这两个方向的大小改成 NCM function 的 configfs 属性（本地补丁，未提交），不需要重编就能切换：

- `ntb_out_size`：设备上报的 `dwNtbOutMaxSize`，同时是 rx 请求缓冲长度的下限，范围 2048 到 65535，默认 16384。
- `ntb_in_size`：设备上报的 `dwNtbInMaxSize`，也是设备发送 NTB 的大小上限，范围 16384 到 65535，只允许调大，默认 16384。`ncm_wrap_ntb` 每个新 NTB 按该值整块 `alloc_skb(GFP_ATOMIC)` 再拷入数据报，比一个最大 jumbo 帧还小会越界，所以下限取 16384。
- 按代码，function 已绑定时写入这两个属性返回 `EBUSY`（`max_segment_size` 的 `EBUSY` 已在 3.2 实测）。流程与 3.2 相同：stop、删链接、写值、读回、start，主机重新枚举后采纳新值。该流程封装成 3.7 里的 `ntbset`。

### 3.4 吞吐测试

```powershell
iperf3 -c 192.168.77.2 -t 20 -O 3 -P 1
iperf3 -c 192.168.77.2 -t 20 -O 3 -P 1 -R
iperf3 -c 192.168.77.2 -t 20 -O 3 -P 4
iperf3 -c 192.168.77.2 -t 20 -O 3 -P 4 -R
```

不带 `-R` 为上行（Windows 发、板端收），带 `-R` 为下行（板端发、Windows 收）。10-22 为 `-t 60`，10-27 为 `-t 40`，10-28 为 `-t 20`（总速率每个 1 秒区间很平）。`-R` 要紧贴写，中间有空格会变成上行。

### 3.5 板端 CPU 采样

`cpuwin` 采样窗口可传秒数；`exp` 先等流量出现再延迟采样，输出写文件，不依赖手速。iperf3 服务端必须放在后台运行，同一个串口 shell 里才能同时采样。

```sh
cpuwin() {
  n=${1:-20}
  grep '^cpu[0-9]' /proc/stat > /tmp/a; grep ncm0 /proc/net/dev > /tmp/na
  sleep $n
  grep '^cpu[0-9]' /proc/stat > /tmp/b; grep ncm0 /proc/net/dev > /tmp/nb
  awk 'NR==FNR{for(i=2;i<=8;i++)a[$1,i]=$i;next}
       {t=0;for(i=2;i<=8;i++){d[i]=$i-a[$1,i];t+=d[i]}
        if(t>0)printf "%-6s usr%3d%% sys%3d%% irq%3d%% sirq%3d%% idle%3d%%\n",$1,
        100*(d[2]+d[3])/t,100*d[4]/t,100*d[7]/t,100*d[8]/t,100*(d[5]+d[6])/t}' /tmp/a /tmp/b
  cat /tmp/na /tmp/nb
  sed 's/:/ /' /tmp/na > /tmp/na2; sed 's/:/ /' /tmp/nb > /tmp/nb2
  awk -v n=$n 'NR==FNR{r=$2;rp=$3;t=$10;tp=$11;next}
    {printf "ncm0 rx %.3f Gbit/s  tx %.3f Gbit/s (%ds)\n",($2-r)*8/n/1e9,($10-t)*8/n/1e9,n
     printf "avg B/pkt rx %.0f tx %.0f  pkt/s rx %.0f tx %.0f\n",($3-rp>0?($2-r)/($3-rp):0),($11-tp>0?($10-t)/($11-tp):0),($3-rp)/n,($11-tp)/n}' /tmp/na2 /tmp/nb2
}
exp() {
  tag=$1; w=${2:-10}; dl=${3:-4}
  mkdir -p /tmp/exp
  {
    echo "== $tag $(date '+%F %T') up=$(cut -d' ' -f1 /proc/uptime) window=${w}s delay=${dl}s"
    echo "rps_cpus=$(cat /sys/class/net/ncm0/queues/rx-0/rps_cpus 2>/dev/null) mtu=$(cat /sys/class/net/ncm0/mtu 2>/dev/null) max_segment_size=$(cat /sys/kernel/config/usb_gadget/usb_ncm/functions/ncm.usb0/max_segment_size 2>/dev/null) ntb_out_size=$(cat /sys/kernel/config/usb_gadget/usb_ncm/functions/ncm.usb0/ntb_out_size 2>/dev/null) ntb_in_size=$(cat /sys/kernel/config/usb_gadget/usb_ncm/functions/ncm.usb0/ntb_in_size 2>/dev/null) speed=$(cat /sys/class/udc/2e1f0000.dwc3/current_speed 2>/dev/null)"
    i=0; hit=0
    while [ $i -lt 120 ]; do
      a=$(( $(cat /sys/class/net/ncm0/statistics/rx_bytes) + $(cat /sys/class/net/ncm0/statistics/tx_bytes) ))
      sleep 1
      b=$(( $(cat /sys/class/net/ncm0/statistics/rx_bytes) + $(cat /sys/class/net/ncm0/statistics/tx_bytes) ))
      i=$((i+1))
      if [ $((b-a)) -gt 50000000 ]; then hit=1; break; fi
    done
    if [ $hit = 1 ]; then
      echo "traffic detected after ${i}s, sampling starts in ${dl}s"
      sleep $dl
      cpuwin $w
    else
      echo "no traffic within 120s, skipped"
    fi
  } > /tmp/exp/$tag.cpu 2>&1 &
}
```

每组先在板端执行 `exp <标签> 10 4`，再启动 Windows 端 iperf3；`exp` 最多等 120 秒，超时会写 “skipped”。每个 `.cpu` 文件的第二行记录 `rps_cpus`、`mtu`、`max_segment_size`、`speed`（10-29 起还有 `ntb_out_size`、`ntb_in_size`），用来核对该组的实际条件；10-29 起末尾多一行 `avg B/pkt`（每个计数的平均字节数和每秒个数，由 `/proc/net/dev` 的字节差除以计数差得到）。`ncm0` 两行计数的字节差除以窗口秒数，就是板端实际速率。

### 3.6 RPS

```sh
echo fffc > /sys/class/net/ncm0/queues/rx-0/rps_cpus   # 开
echo 0 > /sys/class/net/ncm0/queues/rx-0/rps_cpus      # 关
```

### 3.7 板端与 Windows 对测流程（10-29 起）

用 READY / DONE 握手把板端 CPU 采样和 Windows 的 iperf3 对齐，并用 `ntbset` 一条命令切换 NTB 大小和 MTU。板端在 3.5 的函数基础上增加 `spd`、`spdwait`、`ntbset`、`expc`（需要固件带 `ntb_out_size`、`ntb_in_size` 属性，见 3.3）；Windows 增加 `chk`、`run`。两边的函数各粘贴一次，板端重启后要重新粘贴。

板端：

```sh
G=/sys/kernel/config/usb_gadget/usb_ncm
F=$G/functions/ncm.usb0
spd() { cat /sys/class/udc/2e1f0000.dwc3/current_speed 2>/dev/null; }
spdwait() {
  k=0
  while [ $k -lt 10 ]; do
    s=$(spd)
    [ -n "$s" ] && [ "$s" != "UNKNOWN" ] && break
    sleep 1; k=$((k+1))
  done
}
# ntbset <ntb_out> <max_segment_size> <mtu> [ntb_in, default 16384]
ntbset() {
  ni=${4:-16384}
  /soc/scripts/usb-ncm.sh stop
  sleep 3
  rm -f $G/configs/c.1/ncm.usb0
  echo $1 > $F/ntb_out_size || { echo "FAIL: write ntb_out_size"; return 1; }
  echo $ni > $F/ntb_in_size || { echo "FAIL: write ntb_in_size"; return 1; }
  echo $2 > $F/max_segment_size || { echo "FAIL: write max_segment_size"; return 1; }
  [ "$(cat $F/ntb_out_size)" = "$1" ] && [ "$(cat $F/ntb_in_size)" = "$ni" ] && [ "$(cat $F/max_segment_size)" = "$2" ] || { echo "FAIL: readback mismatch"; return 1; }
  /soc/scripts/usb-ncm.sh start
  spdwait
  r=0
  while [ "$(spd)" != "super-speed" ] && [ $r -lt 2 ]; do
    r=$((r+1))
    echo "speed=$(spd), rebind try $r"
    printf '\n' > $G/UDC; sleep 3; echo 2e1f0000.dwc3 > $G/UDC
    spdwait
  done
  ip link set ncm0 mtu $3
  ip addr add 192.168.77.2/24 dev ncm0 2>/dev/null
  ip link set ncm0 up
  echo fffc > /sys/class/net/ncm0/queues/rx-0/rps_cpus
  echo "ntb_out=$(cat $F/ntb_out_size) ntb_in=$(cat $F/ntb_in_size) mss=$(cat $F/max_segment_size) mtu=$(cat /sys/class/net/ncm0/mtu) rps=$(cat /sys/class/net/ncm0/queues/rx-0/rps_cpus) speed=$(spd)"
  [ "$(spd)" = "super-speed" ] && return 0
  echo "NOT SUPER-SPEED: replug the cable, then run only the expc loop"
  dmesg | tail -15
  return 1
}
# expc <tag>: print READY, wait for the host run, print the .cpu result, wait until traffic stops, print DONE
expc() {
  tag=$1
  rm -f /tmp/exp/$tag.cpu
  echo "READY $tag: start the Windows run now (waits up to 120s for traffic)"
  exp $tag 10 4
  k=0
  while [ $k -lt 200 ]; do
    grep -q -e 'avg B/pkt' -e 'skipped' /tmp/exp/$tag.cpu 2>/dev/null && break
    sleep 1; k=$((k+1))
  done
  cat /tmp/exp/$tag.cpu
  z=0
  while [ $z -lt 2 ]; do
    a=$(( $(cat /sys/class/net/ncm0/statistics/rx_bytes) + $(cat /sys/class/net/ncm0/statistics/tx_bytes) ))
    sleep 1
    b=$(( $(cat /sys/class/net/ncm0/statistics/rx_bytes) + $(cat /sys/class/net/ncm0/statistics/tx_bytes) ))
    if [ $((b-a)) -lt 1000000 ]; then z=$((z+1)); else z=0; fi
  done
  echo "DONE $tag"
}
```

Windows：

```powershell
New-Item -ItemType Directory -Force C:\b | Out-Null
function chk($mtu){ $ix=(Get-NetIPAddress -IPAddress 192.168.77.3).InterfaceIndex; $m=(Get-NetIPInterface -InterfaceIndex $ix -AddressFamily IPv4).NlMtu; $p=Test-Connection 192.168.77.2 -Count 1 -Quiet; Write-Host "NlMtu=$m ping=$p want=$mtu"; return (($m -eq $mtu) -and $p) }
function run($t,$P,$R){ Write-Host "== $t"; $a=@('-c','192.168.77.2','-t','20','-O','3','-P',"$P"); if($R){$a+='-R'}; iperf3 @a | Tee-Object "C:\b\$t.txt" }
```

每个配置的步骤：

1. 板端 `ntbset <ntb_out> <mss> <mtu> <ntb_in>`（jumbo 的 `mss`、`mtu` 为 `8000 7986`，MTU 1500 为 `1514 1500`），状态行要有目标 `mtu` 和 `speed=super-speed`。
2. Windows `chk <mtu>`，要看到 `NlMtu=<mtu> ping=True`。
3. 板端 `for t in <标签...>; do expc $t; done`。
4. 板端打印第一个 `READY` 之后，Windows 执行 `if (chk <mtu>) { run <标签1> 1 $false; sleep 10; run <标签2> 1 $true; sleep 10; run <标签3> 4 $true }`。
5. 保存两端的完整输出。

`run` 的第三个参数 `$true` 为下行（函数自己加 `-R`），`$false` 为上行。板端 `expc` 在 120 秒内没检测到流量会写 “skipped”，该组板端数据无效，两边都要重跑。

### 3.8 过夜稳定性测试方法（10-29 起）

每晚只测一个配置，一次只改一个变量，整夜跑。Windows 循环测 up1 / dn1 / dn4 / dn8 / bidir（一路上行加一路下行同时跑，端口 5201 和 5202），每相 10 分钟，每相前检查 `NlMtu` 和 ping；板端每 60 秒记一行 `mon.log`，链路和 dmesg 事件记到 `events.log`。

判据：

- `tx_dropped`、`tx_errors`、`rx_errors` 全程不变；
- `events.log` 没有 `LINK`、`Call Trace`、分配失败、OOM；
- `MemAvailable` 和 buddy 高阶页没有持续下降；
- 各相吞吐不低于首轮约 5%；
- `summary.csv` 没有 FAIL、LINKDOWN；
- `nth=`（`Wrong NTH SIGN` 累计次数）不增长；增长时记下发生的相。

注意：

- `Wrong NDP SIGN`、`Wrong NTH SIGN` 发生时，`ncm_unwrap_ntb` 先 `skb_queue_purge` 再释放 skb 并返回 `-EINVAL`；`u_ether.c` 的 `rx_errors++` 在 `rx_complete` 的出队循环里，队列已清空，不会执行。所以 `rx_errors` 看不到这类丢弃，只能看 dmesg。
- dmesg 环形缓冲会回绕。10-29 夜里 `dmesg_hits`（dmesg 里匹配行的当前行数）约 23:07 写满后不再增长，之后的报错不能再用它统计（4.16）。所以开跑前先 `dmesg -c` 存档并清空，监控脚本里的 `nth=` 按内核时间戳累计，回绕后仍然计数。
- `nth=` 是 10-30 新增的，在宿主机用假 `dmesg` 冒烟过，并在约 60 小时的真板长跑（4.18）里用过：缓冲写满后 `dmesg_hits` 在 911~922 波动，`nth` 继续累计。
- 脚本开头会 `rm -rf /tmp/soak`，所以开跑前先把上一次的目录改名。
- 脚本结束时 `finish` 会 `killsrv`，杀掉板端所有 `iperf3 -s`。之后做别的测试前要先 `ps | grep '[i]perf3'`，没有就重启服务端（`iperf3 -s > /tmp/iperf-srv.log 2>&1 &`），否则 Windows 报 `Connection refused`（10-30 白天遇到过，原因是推断，见 4.17）。
- 板子之前被限到 high-speed（3.9 的 USB2 测试）时，必须用 3.9 的补丁版 `ntbset`：它每次都写 `max_speed`，默认写回 `super-speed`；3.7 里的原版 `ntbset` 不写 `max_speed`，会继续在 high-speed 下跑。

板端 UART，开跑前（上一次的日志和 dmesg 先存起来）：

```sh
mv /tmp/soak /tmp/soak-$(date +%m%d)
dmesg -c > /tmp/dmesg-$(date +%m%d).txt
type ntbset
```

`type ntbset` 报 not found 时，把 `ntbset` 函数（3.7）重新粘一遍。

Windows PowerShell，取当前时间：

```powershell
Get-Date -Format 'yyyy-MM-dd HH:mm:ss'
```

板端，把上面的时间填进 `date -s`（两边时钟要一致，后面才能把 dmesg 事件对应到 Windows 的相），然后设置配置并粘贴监控脚本。`ntbset` 的四个参数依次是 `ntb_out`、`max_segment_size`、MTU、`ntb_in`，下面是 4.16 的配置，换配置时只改这一行：

```sh
date -s "2026-10-30 HH:MM:SS"
ntbset 64512 8000 7986 64512
cat > /tmp/soak.sh <<'EOF'
# AX8860 NCM soak monitor, board side (busybox ash).
# Usage: sh /tmp/soak.sh <minutes> [interval_s]      e.g. sh /tmp/soak.sh 660 60
# Starts two iperf3 servers (ports 5201 and 5202, restarted if they exit) and appends one line per sample
# to $D/mon.log; link and dmesg events go to $D/events.log. Logs live in RAM and are lost on reboot.
# touch $D/STOP ends it early. Keep comments and messages ASCII (UART).
# nth= in mon.log is the running total of "Wrong NTH SIGN" lines in dmesg, counted by kernel timestamp,
# so it keeps counting after the dmesg ring buffer wraps (dmesg_hits does not).
M=${1:-660}; I=${2:-60}
D=${SOAKDIR:-/tmp/soak}
IF=${NCMIF:-ncm0}
SPD=${SPDF:-/sys/class/udc/2e1f0000.dwc3/current_speed}
F=${NCMFN:-/sys/kernel/config/usb_gadget/usb_ncm/functions/ncm.usb0}
SRV=${IPERF:-iperf3}
PAT='page allocation failure|Call Trace|WARNING|BUG:|Oops|Out of memory|oom-kill|soft lockup|rcu.*stall|hung task|dwc3|xhci|usb [0-9]|ncm|u_ether|gadget'

killsrv() { [ "$SRV" = iperf3 ] && kill $(ps | awk '/[i]perf3 -s/{print $1}') 2>/dev/null; }
ev() { [ "$(wc -c < $D/events.log)" -lt 1000000 ] && echo "$*" >> $D/events.log; }
evd() { [ "$(wc -c < $D/events.log)" -lt 1000000 ] && dmesg 2>/dev/null | tail -30 >> $D/events.log; }
evn() { [ "$(wc -c < $D/events.log)" -lt 1000000 ] && dmesg 2>/dev/null | grep 'Wrong NTH SIGN' | tail -3 >> $D/events.log; }

killsrv
rm -rf $D; mkdir -p $D; : > $D/events.log
LP=""
for p in 5201 5202; do
  ( while [ ! -f $D/STOP ]; do $SRV -s -p $p -i 0 > /dev/null 2>&1; sleep 1; done ) &
  LP="$LP $!"
done
finish() { kill $LP 2>/dev/null; killsrv; ev "$(date '+%F %T') END"; }
trap 'exit 0' INT TERM
trap finish EXIT

echo "# cfg ntb_out=$(cat $F/ntb_out_size 2>/dev/null) ntb_in=$(cat $F/ntb_in_size 2>/dev/null) mss=$(cat $F/max_segment_size 2>/dev/null) mtu=$(cat /sys/class/net/$IF/mtu 2>/dev/null) rps=$(cat /sys/class/net/$IF/queues/rx-0/rps_cpus 2>/dev/null) start=$(date '+%F %T') minutes=$M interval=${I}s" > $D/mon.log

N=$((M*60/I)); n=0; pb=-1; nl=0; nt=0
while [ $n -lt $N ] && [ ! -f $D/STOP ]; do
  ts=$(date '+%F %T')
  sp=$(cat $SPD 2>/dev/null)
  ca=$(cat /sys/class/net/$IF/carrier 2>/dev/null)
  nd=$(grep "$IF:" /proc/net/dev | sed 's/:/ /' | awk '{printf "rx_b=%s rx_p=%s rx_e=%s rx_d=%s tx_b=%s tx_p=%s tx_e=%s tx_d=%s",$2,$3,$4,$5,$10,$11,$12,$13}')
  mm=$(awk '/^MemAvailable/{a=$2}/^MemFree/{f=$2}END{printf "memavail_kB=%s memfree_kB=%s",a,f}' /proc/meminfo)
  bd=$(grep -E 'Normal|DMA' /proc/buddyinfo | sed 's/  */ /g' | tr '\n' ';')
  tp=$(cat /sys/class/thermal/thermal_zone0/temp 2>/dev/null)
  bad=$(dmesg 2>/dev/null | grep -c -i -E "$PAT")
  r=$(dmesg 2>/dev/null | awk -v L="$nl" '/Wrong NTH SIGN/{s=$0;sub(/^\[ */,"",s);sub(/\].*/,"",s);t=s+0;if(t>L+0){c++;if(t>m)m=t}}END{printf "%d %.6f\n",c+0,(m>L+0?m:L+0)}')
  [ -n "$r" ] || r="0 $nl"
  nn=${r% *}; nl=${r#* }; nt=$((nt+nn))
  sv=$(ps | grep -c "[i]perf3 -s")
  echo "$ts up=$(cut -d' ' -f1 /proc/uptime) speed=$sp carrier=$ca $nd $mm temp=$tp srv=$sv dmesg_hits=$bad nth=$nt buddy=\"$bd\"" >> $D/mon.log
  if [ "$sp" != "super-speed" ] || [ "$ca" != "1" ]; then ev "$ts LINK speed=$sp carrier=$ca"; fi
  if [ $pb -ge 0 ] && [ "$bad" != "$pb" ]; then ev "$ts DMESG hits $pb -> $bad"; evd; fi
  if [ "$nn" -gt 0 ]; then ev "$ts NTH +$nn total $nt"; evn; fi
  pb=$bad; n=$((n+1)); sleep $I
done
EOF
cat /sys/class/udc/2e1f0000.dwc3/current_speed
```

最后一行必须是 `super-speed`（`ntbset` 自己打印的速度不可信，见 3.9，所以这里单独读一次）；不是的话按 3.7 的办法重新插拔。

Windows，粘贴脚本，确认 MTU 和连通后开跑。`chk` 要回 `NlMtu=7986 ping=True` 才继续（MTU 1500 的配置把 7986 换成 1500）：

```powershell
# Windows-side soak loop for the AX8860 NCM link (PowerShell 5.1). Ran 12 rounds on Windows 10 (2026-10-29/30).
# Usage: soak <mtu> <phase_seconds> <hours>      e.g. soak 7986 600 10      dry run: soak 7986 20 0.06
# Cycle per round: up1, dn1, dn4, dn8, bidir (up and down at the same time, ports 5201 and 5202).
# Results: C:\b\soak-<time>\summary.csv plus one iperf3 output file per phase.
function chk($mtu){ $ix=(Get-NetIPAddress -IPAddress 192.168.77.3).InterfaceIndex; $m=(Get-NetIPInterface -InterfaceIndex $ix -AddressFamily IPv4).NlMtu; $p=Test-Connection 192.168.77.2 -Count 1 -Quiet; Write-Host "NlMtu=$m ping=$p want=$mtu"; return (($m -eq $mtu) -and $p) }
function prs($f){
$l=Select-String -Path $f -Pattern 'receiver' | Select-Object -Last 1
if(-not $l){ return 'FAIL,' }
$g=''; $r=''
if($l.Line -match '([0-9.]+) ([GMK])bits/sec'){ $v=[double]$matches[1]; if($matches[2] -eq 'M'){$v=$v/1000}; if($matches[2] -eq 'K'){$v=$v/1000000}; $g=('{0:N2}' -f $v) }
$s=Select-String -Path $f -Pattern 'sender' | Select-Object -Last 1
if($s -and ($s.Line -match 'bits/sec\s+(\d+)\s+sender')){ $r=$matches[1] }
return "$g,$r"
}
function soak($mtu,$sec,$hours){
if(-not ('W.P' -as [type])){ Add-Type -Namespace W -Name P -MemberDefinition '[DllImport("kernel32.dll")] public static extern uint SetThreadExecutionState(uint f);' }
$ip=(Get-Command iperf3).Source
$d='C:\b\soak-'+(Get-Date -Format 'yyyyMMdd-HHmm'); New-Item -ItemType Directory -Force $d | Out-Null
Write-Host "dir $d"
$sum="$d\summary.csv"; 'time,cycle,phase,gbps,retr,note' | Out-File $sum -Encoding ascii
$end=(Get-Date).AddHours($hours); $c=0
$ph=@(@('up1',1,$false),@('dn1',1,$true),@('dn4',4,$true),@('dn8',8,$true),@('bidir',0,$false))
[void][W.P]::SetThreadExecutionState(2147483649)
try {
while((Get-Date) -lt $end){
$c++
foreach($p in $ph){
if((Get-Date) -ge $end){break}
$n=$p[0]; $t=Get-Date -Format 'MM-dd HH:mm:ss'
$ok=$false
for($i=0;$i -lt 30 -and -not $ok;$i++){ $ok=chk $mtu; if(-not $ok){ sleep 10 } }
if(-not $ok){ "$t,$c,$n,,,LINKDOWN 5min" | Add-Content $sum; Write-Host "$t $n LINKDOWN"; continue }
if($n -eq 'bidir'){
$f1="$d\c$c-bidir-up.txt"; $f2="$d\c$c-bidir-dn.txt"
$p1=Start-Process $ip -ArgumentList "-c 192.168.77.2 -p 5201 -t $sec -i 10" -RedirectStandardOutput $f1 -NoNewWindow -PassThru
$p2=Start-Process $ip -ArgumentList "-c 192.168.77.2 -p 5202 -t $sec -i 10 -R" -RedirectStandardOutput $f2 -NoNewWindow -PassThru
$p1,$p2 | Wait-Process
$r1=prs $f1; $r2=prs $f2
"$t,$c,bidir-up,$r1," | Add-Content $sum
"$t,$c,bidir-dn,$r2," | Add-Content $sum
Write-Host "$t bidir up=$r1 dn=$r2"
} else {
$f="$d\c$c-$n.txt"; $a=@('-c','192.168.77.2','-t',"$sec",'-i','10','-P',"$($p[1])"); if($p[2]){$a+='-R'}
& $ip @a 2>&1 | Out-File $f -Encoding ascii
$r=prs $f
"$t,$c,$n,$r," | Add-Content $sum
Write-Host "$t $n $r"
}
sleep 5
}
}
} finally {
[void][W.P]::SetThreadExecutionState(2147483648)
"$(Get-Date -Format 'MM-dd HH:mm:ss'),$c,END,,," | Add-Content $sum
Write-Host "DONE $d"
}
}
```

```powershell
chk 7986
```

板端先启动监控，必须先于 Windows 的 `soak`：脚本开头会杀掉旧的 `iperf3 -s` 并重启 5201、5202 两个服务端，Windows 先起会在第一相连不上、记 FAIL（10-29 的实际顺序也是板端先）。板端分钟数要比 Windows 的小时数乘 60 多留一些，让它覆盖 Windows 的整个测试。括号里的 `trap '' HUP` 让串口会话断开时脚本不被杀（在宿主机用 dash 验证过：发 HUP 后继续记录，`touch /tmp/soak/STOP` 能正常退出并写 `END`；不加时被杀）。

```sh
(trap '' HUP; exec sh /tmp/soak.sh 660 60) > /dev/null 2>&1 &
```

再回 Windows 开跑：

```powershell
soak 7986 600 10
```

跑得久时（例如周末）：板端分钟数和 Windows 小时数一起加大（60 小时对应 Windows `soak 7986 600 60`、板端 3840 分钟）。`events.log` 有 1 MB 上限，报错密集时可能写满，之后的事件和 `END` 不再记录，这时以 `mon.log` 里的 `nth=` 为准。Windows 控制台里不要点击选中文字（快速编辑模式会暂停脚本，按 Esc 恢复），也不要让它睡眠、注销或被系统更新重启。

第二天早上，板端：

```sh
wc -l /tmp/soak/mon.log; head -1 /tmp/soak/mon.log; tail -2 /tmp/soak/events.log
awk 'NR>1&&(NR-2)%60==0{for(i=1;i<=NF;i++)if($i~/^(nth|dmesg_hits|memavail_kB|tx_d|tx_e|rx_e)=/)printf "%s ",$i; print $1,$2}' /tmp/soak/mon.log
grep -c ' LINK speed=' /tmp/soak/events.log
grep -n -i -E 'Call Trace|page allocation failure|Out of memory|oom-kill|BUG:|Oops|soft lockup|rcu.*stall|hung task' /tmp/soak/events.log | head -20
dmesg | head -2; cat /proc/uptime; date '+%F %T'
dmesg | grep -c 'Wrong NTH SIGN'
awk '/^#/{next}{for(i=1;i<=NF;i++){if($i~/^memavail_kB=/){split($i,a,"=");v=a[2]+0;if(!n||v<mn)mn=v;if(v>mx)mx=v;n++}}}END{print "memavail min",mn,"max",mx,"rows",n}' /tmp/soak/mon.log
awk 'NR>1&&(NR-2)%60==0{split($0,a,"buddy=");split(a[2],b," ");print $1,$2,"o7="b[12],"o8="b[13],"o9="b[14],"o10="b[15]}' /tmp/soak/mon.log
```

Windows：

```powershell
$d=(Get-ChildItem C:\b -Directory -Filter 'soak-*' | Sort-Object LastWriteTime | Select-Object -Last 1).FullName; Get-Content "$d\summary.csv"
```

### 3.9 USB2（high-speed）测试方法（10-30）

不重刷固件：固件仍是 USB3 版（SS PHY 使能），用 configfs 把 gadget 的最高速度限到 high-speed。

- `max_speed` 在 gadget 目录（`$G/max_speed`）下，只能在解绑之后写，已绑定时内核拒绝（`drivers/usb/gadget/configfs.c`）；绑定时 UDC 核心把它传给 dwc3，dwc3 在 `__dwc3_gadget_set_speed` 里按它设置 DCFG 的速度位。
- 写进去的值保留在 gadget 对象里，直到下一次写（读 `configfs.c` 的结论，没有专门验证），所以回到 USB3 要重新写 `super-speed`，补丁版 `ntbset` 每次都写。
- 这和烧 USB2 版 DTS（`maximum-speed = "high-speed"`，SS PHY 关闭）不完全等价：SS PHY 仍使能。协议层吞吐应该一样，没有验证。

3.7 的板端脚本有两处不适合 USB2，下面的补丁重新定义 `exp` 和 `ntbset`：

- `exp` 的流量检测阈值是 50 MB/s（400 Mbit/s），接近 high-speed 上报的速率 425.984 Mbit/s，USB2 下一直检测不到流量；补丁改成变量 `TH`，USB2 用 `TH=5000000`。
- `ntbset` 增加第 5 个参数（目标速度，默认 `super-speed`），写入 `max_speed`，重绑和结尾的判断都改成与它比较。

注意：

- `ntbset` 自己打印的速度不可信。第一次它连续三次读到 `super-speed`、报 NOT high-speed，随后手动 `cat current_speed` 是 `high-speed`；第二次（目标相同）一次读到 `high-speed`。解绑后 `current_speed` 是否保留旧值没有验证（dwc3 只在 disconnect 中断路径把 speed 置 UNKNOWN，`gadget.c:4092`）。速度以每个 `.cpu` 头里的 `speed=` 和 `ntbset` 之后手动 `cat current_speed` 为准。
- 板端 iperf3 服务端要先在监听（见 3.8 的注意）。
- 4 流下行在 10-30 的 63 KB 组里链式运行失败过一次，单独跑成功（4.17），建议 4 流下行单独跑。

板端 UART，改配置前先确认现有函数在，再贴补丁、设阈值、设置配置（`sleep 5` 是这里新加的，10-30 实测时没有等）：

```sh
type ntbset exp expc spdwait cpuwin
cat > /tmp/usb2_patch.sh <<'EOF'
# Patch for ncm_test.sh (skill ax8860-usb-throughput-test, scripts/board.sh): run the NCM test at high-speed (USB2).
# Redefines exp and ntbset. Paste after loading ncm_test.sh:   . /tmp/usb2_patch.sh
# Changes vs board.sh:
#   exp:    traffic-detect threshold comes from $TH (bytes/s, default 50000000). The default is close to the
#           USB2 high-speed bulk limit, so set TH=5000000 for USB2 runs.
#   ntbset: optional 5th argument = speed to run at (default super-speed). It is written to the gadget max_speed
#           while unbound, and the rebind loop and the final check compare against it instead of super-speed.
# Not verified on hardware yet. Keep comments and messages ASCII (UART).

# exp <tag> [window_s] [delay_s]: wait for traffic, then sample; result goes to /tmp/exp/<tag>.cpu
exp() {
  tag=$1; w=${2:-10}; dl=${3:-4}
  mkdir -p /tmp/exp
  {
    echo "== $tag $(date '+%F %T') up=$(cut -d' ' -f1 /proc/uptime) window=${w}s delay=${dl}s"
    echo "rps_cpus=$(cat /sys/class/net/ncm0/queues/rx-0/rps_cpus 2>/dev/null) mtu=$(cat /sys/class/net/ncm0/mtu 2>/dev/null) max_segment_size=$(cat $F/max_segment_size 2>/dev/null) ntb_out_size=$(cat $F/ntb_out_size 2>/dev/null) ntb_in_size=$(cat $F/ntb_in_size 2>/dev/null) speed=$(cat /sys/class/udc/2e1f0000.dwc3/current_speed 2>/dev/null)"
    i=0; hit=0
    while [ $i -lt 120 ]; do
      a=$(( $(cat /sys/class/net/ncm0/statistics/rx_bytes) + $(cat /sys/class/net/ncm0/statistics/tx_bytes) ))
      sleep 1
      b=$(( $(cat /sys/class/net/ncm0/statistics/rx_bytes) + $(cat /sys/class/net/ncm0/statistics/tx_bytes) ))
      i=$((i+1))
      if [ $((b-a)) -gt ${TH:-50000000} ]; then hit=1; break; fi
    done
    if [ $hit = 1 ]; then
      echo "traffic detected after ${i}s, sampling starts in ${dl}s"
      sleep $dl
      cpuwin $w
    else
      echo "no traffic within 120s, skipped"
    fi
  } > /tmp/exp/$tag.cpu 2>&1 &
}

# ntbset <ntb_out> <max_segment_size> <mtu> [ntb_in, default 16384] [speed, default super-speed]
ntbset() {
  ni=${4:-16384}; want=${5:-super-speed}
  /soc/scripts/usb-ncm.sh stop
  sleep 3
  rm -f $G/configs/c.1/ncm.usb0
  echo $want > $G/max_speed || { echo "FAIL: write max_speed"; return 1; }
  echo $1 > $F/ntb_out_size || { echo "FAIL: write ntb_out_size"; return 1; }
  echo $ni > $F/ntb_in_size || { echo "FAIL: write ntb_in_size"; return 1; }
  echo $2 > $F/max_segment_size || { echo "FAIL: write max_segment_size"; return 1; }
  [ "$(cat $G/max_speed)" = "$want" ] && [ "$(cat $F/ntb_out_size)" = "$1" ] && [ "$(cat $F/ntb_in_size)" = "$ni" ] && [ "$(cat $F/max_segment_size)" = "$2" ] || { echo "FAIL: readback mismatch"; return 1; }
  /soc/scripts/usb-ncm.sh start
  spdwait
  r=0
  while [ "$(spd)" != "$want" ] && [ $r -lt 2 ]; do
    r=$((r+1))
    echo "speed=$(spd), rebind try $r"
    printf '\n' > $G/UDC; sleep 3; echo 2e1f0000.dwc3 > $G/UDC
    spdwait
  done
  ip link set ncm0 mtu $3
  ip addr add 192.168.77.2/24 dev ncm0 2>/dev/null
  ip link set ncm0 up
  echo fffc > /sys/class/net/ncm0/queues/rx-0/rps_cpus
  echo "ntb_out=$(cat $F/ntb_out_size) ntb_in=$(cat $F/ntb_in_size) mss=$(cat $F/max_segment_size) mtu=$(cat /sys/class/net/ncm0/mtu) rps=$(cat /sys/class/net/ncm0/queues/rx-0/rps_cpus) speed=$(spd) want=$want"
  [ "$(spd)" = "$want" ] && return 0
  echo "NOT $want: check the cable and port (super-speed needs a replug), then run only the expc loop"
  dmesg | tail -15
  return 1
}
EOF
. /tmp/usb2_patch.sh
TH=5000000
ntbset 16384 1514 1500 16384 high-speed
sleep 5; cat /sys/class/udc/2e1f0000.dwc3/current_speed
kill $(ps | awk '/[i]perf3 -s/{print $1}') 2>/dev/null; iperf3 -s > /tmp/iperf-srv.log 2>&1 &
sleep 1; ps | grep '[i]perf3'
```

`type` 报 not found 时，把 3.7 的 `ncm_test.sh` 重新粘一遍。`cat` 的结果必须是 `high-speed`，不是的话先停下来。

Windows PowerShell，确认 MTU 和连通（`chk` 找不到就把 3.7 的 Windows 函数重新粘一遍）：

```powershell
chk 1500
```

板端：

```sh
for t in u2-m1500-n16-up1 u2-m1500-n16-dn1 u2-m1500-n16-dn4; do expc $t; done
```

Windows（板端打出第一个 READY 之后）：

```powershell
run u2-m1500-n16-up1 1 $false; sleep 10; run u2-m1500-n16-dn1 1 $true; sleep 10; run u2-m1500-n16-dn4 4 $true
```

换 NTB 63 KB 时把 `ntbset` 那一行改成 `ntbset 64512 1514 1500 64512 high-speed`，标签里的 `n16` 换成 `n63`。回到 USB3：`ntbset 16384 1514 1500 16384`（默认写回 `super-speed`），之后单独 `cat /sys/class/udc/2e1f0000.dwc3/current_speed` 确认；不是 `super-speed` 时按 3.7 的办法重新插拔。

## 4. 测试记录

### 4.1 2026-10-22 NCM 默认参数基线

参数：`-t 60 -O 3`，主机 Windows 10。

| # | 命令 | 方向 | 流数 | 带宽 | 传输量 | 重传 |
|---|---|---|---|---|---|---|
| 1 | `-P 1` | 上行 | 1 | **2.22 Gbit/s** | 15.5 GB | 0 |
| 2 | `-P 1 -R` | 下行 | 1 | **1.91 Gbit/s** | 13.4 GB | 0 |
| 3 | `-P 4` | 上行 | 4 | 2.21 Gbit/s | 15.4 GB | 0 |
| 4 | `-P 4 -R` | 下行 | 4 | 1.23 Gbit/s | 8.62 GB | 1 |

- 4 流上行，各流为 524 / 578 / 591 / 513 Mbit/s，分布均衡。
- 4 流下行，各流为 305 / 287 / 348 / 293 Mbit/s。
- 另一次中途中断的单流上行测试，在 26.7 秒时同样是 2.22 Gbit/s，结果可复现。
- 链路质量良好：全程只有 1 次重传，没有错误。

板端 CPU（20 秒采样）：

| | 上行 `-P 1` | 下行 `-P 4 -R` |
|---|---|---|
| 板端计数速率 | 2.28 Gbit/s | 1.31 Gbit/s |
| CPU1 | sys 44% + irq 4% + softirq 51%，**idle 0%** | sys 15% + irq 4% + softirq 77%，**idle 2%** |
| 其他核 | CPU2 sys 39%（iperf3 接收拷贝），CPU3 19%，其余基本空闲 | 基本全部空闲 |

- dwc3 中断（IRQ 53）全部落在 CPU1。
- `NET_RX` 软中断只出现在 CPU1，其他 15 个核都是 0。
- 下行时 USB 请求平均 15279 字节，接近 16 KB 的 NTB 上限。

### 4.2 2026-10-22 NCM 开启 RPS

设置：`echo fffc > /sys/class/net/ncm0/queues/rx-0/rps_cpus`，把协议栈处理分到 CPU2 到 15。测完恢复为 `0`。速率来自板端 ncm0 计数，Windows 端汇总待补。

| | 上行 `-P 1` | 下行 `-P 4 -R` |
|---|---|---|
| 板端计数速率 | 2.29 Gbit/s（不变） | **2.14 Gbit/s（+63%）** |
| CPU1 | sys 62% + irq 8% + softirq 12%，idle 16% | sys 35% + irq 10% + softirq 11%，idle 42% |
| 其他核 | CPU12 softirq 57%（协议栈），CPU8/9/11/13 sys 最高 17% | CPU3/5/7/14 softirq 各约 36 到 43%（ACK 处理） |

### 4.3 2026-10-23 NCM 默认参数复测（未开 RPS）

本轮原计划在 RPS 下测试，但 `rps_cpus` 实际为 `0`，因此作为默认参数复测记录。速率来自板端 ncm0 计数（20 秒窗口）。

| 组 | 方向 | 板端速率 | CPU1 | 其他核 | Windows 总 CPU |
|---|---|---|---|---|---|
| `-P 1` | 上行 | 2.29 Gbit/s | sirq 51%，idle 0% | CPU2/3 sys 约 30%（iperf3） | 43% |
| `-P 4` | 上行 | 2.24 Gbit/s | sirq 51%，idle 0% | CPU2 到 5 sys 7 到 18% | 47% |
| `-P 1 -R` | 下行 | 1.76 Gbit/s | sirq 35%，idle 30% | CPU8/9 sys 约 25% | 51% |
| `-P 4 -R` | 下行 | 1.28 Gbit/s | sirq 80%，idle 1% | 基本全部空闲 | 52% |

- 与 10-22 默认基线一致，结果可复现。
- 下行单流时 CPU1 仍有 30% 空闲，单流下行在默认参数下也不完全由 CPU1 决定。

### 4.4 2026-10-23 NCM 开启 RPS（四组完整）

设置：`echo fffc > /sys/class/net/ncm0/queues/rx-0/rps_cpus`，已读回确认。速率来自板端 ncm0 计数（20 秒窗口）。主机为 18 个逻辑处理器。

| 组 | 方向 | 板端速率 | CPU1 | 其他核 | Windows 总 CPU |
|---|---|---|---|---|---|
| `-P 1` | 上行 | 2.29 Gbit/s | sys 63% + irq 8% + sirq 10%，idle 18% | CPU5 sirq 58%（RPS），CPU8 sys 42%（iperf3） | 46% |
| `-P 4` | 上行 | 2.29 Gbit/s | sys 61% + irq 8% + sirq 10%，idle 18% | CPU4/5/7 sirq 12 到 28%，CPU8 sys 18% | - |
| `-P 1 -R` | 下行 | 1.86 Gbit/s | sys 31% + irq 8% + sirq 7%，idle 52% | **CPU8 sys 88%，idle 0%**（iperf3 发送） | - |
| `-P 4 -R` | 下行 | 1.90 Gbit/s | sys 34% + irq 8% + sirq 9%，idle 47% | CPU9/10/12 sirq 32 到 46%（ACK 处理） | - |

- 上行单流与 4 流完全相同（约 19 万包/秒，每包 1500 字节），板端没有任何核跑满。
- 下行单流时 iperf3 发送线程所在的 CPU8 跑满。
- 下行 4 流 1.90 Gbit/s，比 10-22 的 RPS 结果 2.14 Gbit/s 低约 11%，属于两次测试间波动，原因未查。
- 下行 USB 请求平均约 15263 字节，接近 16 KB NTB 上限。
- 测试期间 `rx_dropped` 只在 `-P 4 -R` 之前增加 1（74 → 75），采样窗口内不增加。

### 4.5 2026-10-23 NCM USB 请求队列深度（qmult）

条件：RPS 开启，dwc3 中断绑到 CPU4。qmult 在运行时修改：先 `stop`，删除 `configs/c.1/ncm.usb0` 链接，写入 `functions/ncm.usb0/qmult`，再 `start`。不删链接时写入会返回 EBUSY。

| qmult | 每方向 USB 请求数 | bulk OUT 端点 hwo=1 TRB | 上行 `-t 30 -O 3 -P 4` |
|---|---|---|---|
| 5（默认） | 10 | 11 | 2.22 / 2.23 Gbit/s |
| 80 | 160 | 161 | 2.23 / 2.22 Gbit/s |

- TRB 计数来自 dwc3 debugfs 的 `trb_ring`，在链路空闲时统计。计数比请求数多 1，多出的是环末尾的 link TRB。计数结果证明新的 qmult 在硬件上已经生效。
- 请求数增加 16 倍，上行吞吐不变。**USB OUT 请求队列深度不是上行瓶颈。**
- 另做过一次 qmult=10、`-t 6` 的短测：上行 2.23 Gbit/s，下行 4 流 2.02 Gbit/s，与 qmult=5 基本相同。
- 测试结束后已恢复 qmult=5。

### 4.6 2026-10-27 TCP 复测

条件：`-t 40 -O 3`，RPS 关闭（`rps_cpus=0000`），0.25 m 线，速率为 iperf3 报告。

| 方向 | 流数 | 结果（Gbit/s） |
|---|---|---|
| 上行 | 1 | 2.22 / 2.21 / 2.16（三次） |
| 下行 | 1 | 1.91 / 1.89 |
| 上行 | 4 | 2.20 / 2.10（SUM） |
| 下行 | 4 | 1.30（SUM） |

- 与 10-22、10-23 的默认参数结果一致。
- 上行 4 流没有叠加收益；下行 4 流比单流慢，原因见第 5 节第 1 条。

Windows 每核 CPU（18 个逻辑处理器，每秒采样）：

| 组 | 总占用 均值 / 峰值 | 最忙核 均值 | 最忙核 峰值 |
|---|---|---|---|
| 上行 1 流 | 18.1% / 27.9% | CPU1 33.3% | CPU6 78.4% |
| 下行 1 流 | 18.8% / 23.4% | CPU16 37.0% | CPU0 59.9% |
| 上行 4 流 | 16.7% / 32.3% | CPU7 35.3% | CPU7 84.6% |
| 下行 4 流 | 17.6% / 23.3% | CPU6 33.6% | CPU6 53.8% |

这轮没有采板端 CPU（板端 busybox 的 `top` 没有 `%Cpu` 行），10-28 起改用 `/proc/stat` 差值的 `cpuwin`（见 3.5）。

### 4.7 2026-10-27 UDP（仅供参考）

命令与原计划不一致，且都没带 `-l 1472`（用了默认 1460 字节数据报），数字只能定性看。

| 组 | 实际参数 | 方向 | 发送 | 接收 | 丢包 |
|---|---|---|---|---|---|
| U1 | `-u -b 1G` | 上行 | 1.00 | 0.77 | 23% |
| U2 | `-u -b 2G` | 上行 | 1.52 | 1.12 | 27% |
| U3 | `-u -b 2.5G` | 上行 | 1.18 | 1.04 | 12% |
| U4 | `-u -b 1G -R` | 下行 | 1.00 | 1.00 | 0.065% |
| U5 | `-u -b 2G -R` | 下行 | 1.35 | 1.34 | 0.33% |
| U6 | `-u -b 2.5G -R`（是否 `-P 4` 记录不一致） | 下行 | 1.34 | 1.34 | 0.25% |

- 速率单位 Gbit/s。PC 作发送端单流最高约 1.5，板子作发送端约 1.35，U6 板端采样时 cpu8 sys 83%、其余核空闲。
- 上行方向 1G 起就丢 12% 到 27%，下行方向不超过 0.33%。板端 UDP 计数（开机至今）`InErrors`、`RcvbufErrors` 均为 0，`ncm0` 的 `rx_dropped` 只有 21，所以上行丢包不是板端 UDP socket 缓冲区造成的。每组的 UDP 计数增量没有保存，丢在哪一段还没有定位。
- UDP 测不出链路上限，暂停，不作为调优方向。

### 4.8 2026-10-27 环境事实

板端：

- iperf3 3.20；`rps_cpus=0000`；`tx_queue_len` 1000。
- dwc3 中断 IRQ 54 累计约 1026 万次，几乎全在 CPU1（CPU0 为 50）。
- `softnet_stat` 只有 CPU1 在处理收包（processed 59,131,864，dropped 0，time_squeeze 0）。
- `ncm0`：`rx_errors` 0，`tx_dropped` 0，`rx_dropped` 在 16 到 21 之间。
- `tc qdisc show` 报 `Operation not supported`：有效 `.config` 里 `# CONFIG_NET_SCHED is not set`。`CONFIG_RPS`、`XPS`、`NET_FLOW_LIMIT`、`NET_RX_BUSY_POLL` 均为 y。
- 内核没有开 `CONFIG_FTRACE`，板上没有 dwc3 的 trace 事件；`CONFIG_DEBUG_FS=y`。

主机：

- Windows 10（build 19044），Cygwin iperf3 3.22，网卡“以太网 5”（UsbNcm Host Device），地址 192.168.77.3。
- NCM 设备的父节点链：USB Root Hub (USB 3.0)，再到 Intel USB 3.20 可扩展主机控制器（`DEV_7EC0`）。主机另有第二颗 xHCI（`DEV_7E7D`），没有对比过。
- 网卡显示的 3.8 Gbps 是固定上报值，不是实测。

链路参考：板子 host 模式下接高速 U 盘，裸读 430 MB/s（约 3.44 Gbit/s，用户实测）。该测试只覆盖板端接收方向。

jumbo：

- 源码约束（`f_ncm.c`、`u_ether.c`）：`max_segment_size` 默认 1514、上限 8000，对应 MTU 最大 7986；改值前要先 `stop`、删 function 链接，否则返回 `-EBUSY`；MTU 只在 netdev 首次注册时按它设置，之后要手动 `ip link set ncm0 mtu`。
- 10-27 的结果：板端还没改，Windows 的 `netsh interface ipv4 set subinterface ... mtu=7986` 返回“找不到元素”，`Get-NetIPInterface` 复核仍是 1500。
- 10-28 的结果：板端把 `max_segment_size` 改成 8000 并重启 gadget 后，Windows 的 `NlMtu` 读回 7986；`Set-NetIPInterface` 在没有管理员权限时报“拒绝访问”，没有手动设置过。机制没有查清（见 3.2）。

另外 10-27 用开发板当 host 做对照：开发板的 xHCI 持续报 `usb2-port1: Cannot enable`，SS 没起来，没有得到数据。

### 4.9 2026-10-28 jumbo 与 RPS（NTB_OUT 16 KB）

条件：板端 `max_segment_size=8000`、MTU 7986，Windows 的 `NlMtu` 读回 7986；`-t 20 -O 3`，板端 `exp` 10 秒窗口；每组 1 次；内核为 10-27 14:36 构建（推断）。

| 组 | RPS | Windows | 板端 ncm0（10 秒） | 每个计数 | CPU1（sys/irq/sirq，idle） | 其他较忙的核 |
|---|---|---|---|---|---|---|
| 上行 1 流 | 关 | 2.44 | rx 2.451 | 7,982 B，3.84 万个/秒 | 40/8/48，idle 3% | cpu2、cpu3 sys 约 18% 到 19% |
| 下行 4 流 | 关 | 2.47 | tx 2.487 | 16,055 B，1.94 万个/秒 | 22/4/71，idle 1% | 其余核 idle 92% 以上 |
| 上行 1 流 | 开 | 2.40 | rx 2.446 | 7,981 B，3.83 万个/秒 | 47/8/9，idle 34% | cpu14 sirq 48%，cpu8 sys 31% |
| 下行 4 流 | 开 | 3.16 | tx 3.191 | 16,055 B，2.48 万个/秒 | 38/7/40，idle 13% | cpu8、11、14、15 sirq 约 20% |

- 下行 4 流（RPS 开）另有两次复测，同一次开机、同配置：Windows 3.19（板端 tx 3.231，cpu1 idle 13%）和 3.16（板端采样因 `exp` 等待超时跳过），连同表中的 3.16 共三次 3.16 / 3.19 / 3.16，重传 0。
- 上行 RPS 关/开为 2.44 / 2.40，开 RPS 后 cpu1 idle 由 3% 变 34%，吞吐不变。
- 下行 4 流 RPS 关/开为 2.47 / 3.16。
- 上行包速率约 3.8 万包/秒（MTU 1500 时约 19 万，见 4.4）。
- Windows 与板端速率相差不到 2%。

### 4.10 2026-10-28 NTB_OUT 32 KB（jumbo）

条件：`NTB_OUT_SIZE=32768`，`rps_cpus=fffc`，MTU 7986，`max_segment_size=8000`，super-speed（每个 `.cpu` 第二行回显确认）；`-t 20 -O 3`；用户手动测，每组 1 次。

| 组 | Windows | 板端 ncm0（10 秒） | 每个计数 | CPU1 idle | 其他较忙的核 |
|---|---|---|---|---|---|
| 上行 1 流 a | 2.89 | rx 2.906 | 7,986 B，4.55 万个/秒 | 42% | cpu3 sirq 34% |
| 上行 1 流 b | 2.89 | rx 2.909 | 7,982 B，4.56 万个/秒 | 42% | cpu11 sirq 48%，cpu12 sys 34% |
| 上行 4 流 | 2.88 | rx 2.897 | 7,981 B，4.54 万个/秒 | 40% | cpu10 sirq 25% |
| 下行 4 流 | 3.12 | tx 3.145 | 16,054 B，2.45 万个/秒 | 11% | cpu8 sirq 33%，cpu11 sirq 28% |
| 下行 1 流 | 3.09 | tx 3.127 | 15,662 B，2.50 万个/秒 | 28% | cpu8 sys 69%、idle 23%（发送线程） |

- 上行对 16 KB（2.40 到 2.44）约 +18% 到 +20%，两次单流一致；下行 4 流 3.12 对 16 KB 三次均值 3.17 为 -1.6%，在已见波动内，只有 1 次。
- 上行 cpu1 idle 约 40% 到 42%（16 KB 时 3% 到 34%）。

### 4.11 2026-10-28 NTB_OUT 64512（jumbo）

条件：`NTB_OUT_SIZE=64512`，`rps_cpus=fffc`，MTU 7986，`max_segment_size=8000`；`/proc/version` 构建时间 10-28 16:02:06；`-t 20 -O 3`；用户手动测。烧录后第一次读回 `max_segment_size` 是 1514，重设为 8000 并重新枚举后再测。

| 组 | Windows | 板端 ncm0（10 秒） | 每个计数 | CPU1 idle | 其他较忙的核 |
|---|---|---|---|---|---|
| 上行 1 流 a | 3.20 | rx 3.223 | 7,981 B，5.05 万个/秒 | 46% | cpu11 sirq 48% |
| 上行 1 流 b | 3.20 | rx 3.225 | 7,984 B，5.05 万个/秒 | 45% | cpu10 sirq 49% |
| 上行 4 流 a | 3.19 | rx 3.212 | 7,940 B，5.06 万个/秒 | 45% | cpu12 sirq 22% |
| 上行 4 流 b | 3.20 | rx 3.218 | 7,979 B，5.04 万个/秒 | 45% | cpu15 sirq 38% |
| 下行 4 流 a | 2.94 | tx 2.995 | 16,053 B，2.33 万个/秒 | 5% | cpu5、7、14、15 sirq 15% 到 20% |
| 下行 4 流 b | 2.92 | tx 2.954 | 16,053 B，2.30 万个/秒 | 4% | cpu5、13、14、15 sirq 15% 到 18% |
| 下行 4 流 c | 2.95 | tx 2.956 | 16,054 B，2.30 万个/秒 | 3% | cpu3、10、11、15 sirq 14% 到 17% |
| 下行 1 流 | 3.13 | tx 3.172 | 15,641 B，2.53 万个/秒 | 21% | cpu9 sys 90%、idle 0%（发送线程） |

- 上行对 32 KB（2.88 到 2.89）约 +10.7%，对 16 KB（2.40 到 2.44）约 +31% 到 +33%，超过 3.0 约 6.7%。板端计数与 Windows 汇总值相差不到 1%。
- 上行 4 流 b 最初标成了下行（命令里 `-R` 写成了 `- R`，中间有空格，输出里没有 “Reverse mode”），实际是第二次上行 4 流。
- 下行 4 流三次 2.94 / 2.92 / 2.95，极差 0.03；16 KB 三次 3.16 / 3.19 / 3.16，极差也是 0.03。均值相差约 -7.4%，下降能复现。下行 1 流没有变差（3.13 对 32 KB 的 3.09），限制在发送线程所在的核。

jumbo 下下行 4 流随 NTB_OUT 的变化（RPS 开）：

| NTB_OUT | Windows | 板端 tx | tx 个数/秒 | CPU1 idle |
|---|---|---|---|---|
| 16 KB | 3.16 / 3.19 / 3.16 | 3.191 / 3.231 | 2.48 万 / 2.52 万 | 13% |
| 32 KB | 3.12 | 3.145 | 2.45 万 | 11% |
| 64512 | 2.94 / 2.92 / 2.95 | 2.995 / 2.954 / 2.956 | 2.33 万 / 2.30 万 / 2.30 万 | 5% / 4% / 3% |

### 4.12 2026-10-28 MTU 1500 加 NTB_OUT 64512

条件：同一次开机，接在 4.11 之后，没有重新烧录；板端 `max_segment_size` 改回 1514、`ncm0` MTU 1500，`rps_cpus=fffc`；每个 `.cpu` 第二行回显 `mtu=1500 max_segment_size=1514`；Windows 的 `NlMtu` 没有回显，板端上行每个包 1,500 B，说明主机按 1500 发送；`-t 20 -O 3`；用户手动测，每组 1 次。

| 组 | Windows | 板端 ncm0（10 秒） | 每个计数 | CPU1 idle | 其他较忙的核 |
|---|---|---|---|---|---|
| 上行 1 流 | 2.68 | rx 2.760 | 1,500 B，23.0 万包/秒 | 13% | cpu13 sirq 64% |
| 上行 4 流 | 2.68 | rx 2.753 | 1,499 B，23.0 万包/秒 | 11% | cpu4 sirq 20%，cpu14 sys 20% |
| 下行 1 流 | 1.86 | tx 1.954 | 15,247 B，1.60 万个/秒 | 44% | cpu8 sys 88%、idle 0%（发送线程） |
| 下行 4 流 | 2.08 | tx 2.181 | 15,280 B，1.78 万个/秒 | 32% | cpu5、9、11、13 sirq 30% 到 44% |

- 上行 2.68 对 10-27 的 MTU 1500、16 KB、RPS 关（2.16 到 2.22）约 +21% 到 +24%，不需要主机改 MTU；距 3.0 差 10.7%。此时上行 cpu1 idle 只有 11% 到 13%，包速率约 23 万包/秒。
- 下行 1 流 1.86 对 10-27 的 1.89 到 1.91 差 -1.6% 到 -2.6%，在已见波动内。下行 1 流 Windows 的区间速率前约 5 秒为 1.91 到 1.93，之后约 1.82 到 1.85，原因没查。
- 下行 4 流 2.08：同口径（MTU 1500、16 KB、RPS 开）只有 10-22、10-23 的板端计数 1.90 到 2.14，不是同日同固件的对照，没有得出结论。
- 板端比 Windows 高：上行 2.7% 到 3.0%，下行 4.8% 到 5.1%。上行的差与协议头一致（板端按 IP 包长计，iperf3 只计 TCP 负载：2.760 乘 1460/1500 得 2.686，Windows 为 2.68）；下行的差含 NCM 头，没有逐项拆。

NTB_OUT 为 64512 时 MTU 的影响（Windows 值）：

| 方向 | MTU 1500 | MTU 7986 |
|---|---|---|
| 上行 1 流 | 2.68 | 3.20 |
| 上行 4 流 | 2.68 | 3.19 到 3.20 |
| 下行 1 流 | 1.86 | 3.13 |
| 下行 4 流 | 2.08 | 2.92 到 2.95 |

### 4.13 2026-10-29 NTB_OUT 可配置固件（jumbo，NTB_IN 16 KB）

条件：固件 `boot_signed_ntbcfg.bin`；每组前用 `ntbset` 切换 `ntb_out_size`；`max_segment_size=8000`、MTU 7986，`rps_cpus=fffc`，super-speed（每个 `.cpu` 第二行回显）；`-t 20 -O 3`；用户手动测，每格 1 次；同一次开机，板端 uptime 606 到 1865 s。

| 组 | NTB_OUT | 方向 | Windows | 板端 ncm0（10 秒） | CPU1 idle | 其他较忙的核 |
|---|---|---|---|---|---|---|
| A1 | 16384 | 下行 4 流 | 3.15 | tx 3.169 | 10% | cpu15 sirq 39%，cpu13 sirq 21% |
| A2a | 64512 | 上行 1 流 | 3.20 | rx 3.223 | 46% | cpu10 sirq 48%，cpu8 sys 30% |
| A2b | 64512 | 下行 4 流 | 2.96 | tx 3.008 | 5% | cpu4、12、14、15 sirq 16% 到 19% |
| A3a | 49152 | 上行 1 流 | 3.11 | rx 3.130 | 43% | cpu3 sirq 35%，cpu4 sys 18% |
| A3b | 49152 | 下行 4 流 | 2.96 | tx 2.981 | 5% | cpu3、7、9、12 sirq 14% 到 19% |
| A4a | 43008 | 上行 1 流 | 2.98 | rx 2.999 | 42% | cpu5 sirq 37% |
| A4b | 43008 | 下行 4 流 | 3.01 | tx 3.058 | 6% | cpu4、9、10、15 sirq 17% 到 19% |

jumbo 下按 NTB_OUT 汇总（RPS 开，NTB_IN 16 KB，Windows 值）：

| NTB_OUT | 上行 1 流 | 下行 4 流 | 下行 4 流 CPU1 idle |
|---|---|---|---|
| 16 KB | 2.40 | 3.16 / 3.19 / 3.16 / 3.15 | 10% 到 13% |
| 32 KB | 2.89 | 3.12 | 11% |
| 43008 | 2.98 | 3.01 | 6% |
| 49152 | 3.11 | 2.96 | 5% |
| 64512 | 3.20 | 2.94 / 2.92 / 2.95 / 2.96 | 3% 到 5% |

- 同一次开机、同一固件、MTU 和 RPS 不变，只改 `ntb_out_size`：下行 4 流 16384 为 3.15、64512 为 2.96（约 -6.0%），确认下降由 NTB_OUT 引起，而不是两次构建或开机的差异。
- 上行随 NTB_OUT 单调上升；下行 4 流先降后趋平。32 KB 那一行来自 10-28（不同次开机，单次）。
- 板端比 Windows 高 0.6% 到 1.6%。
- 标签核对：A2a 的板端文件名写成了 `k16j-up1`，实际是 64512 上行；A1 的板端文件被 A2b 的同名文件覆盖，A1 的板端内容来自覆盖前贴回的文本；A2b、A3b、A4b 的 Windows 贴回没有 `run` 命令行，标签未核实。条件以 `.cpu` 第二行为准。`ntbset` 状态行和 Windows 的 `NlMtu` 没有贴回。

### 4.14 2026-10-29 NTB_IN 扫描（jumbo，NTB_OUT 64512）

条件：固件 `boot_signed_ntbinout.bin`；每组前用 `ntbset 64512 8000 7986 <ntb_in>` 切换；`rps_cpus=fffc`，super-speed；`-t 20 -O 3`；用户手动测，每格 1 到 2 次；与 4.15 同一次开机，板端 uptime 765 到 6915 s。

下行 1 流：

| NTB_IN | Windows | 板端 tx | CPU1 idle | 发送线程所在核 | 每个 tx 计数 | tx 计数/秒 |
|---|---|---|---|---|---|---|
| 16384 | 3.15 | 3.185 | 22% | cpu8 sys 91% | 15,651 B | 2.54 万 |
| 32768 | 3.42 | 3.460 | 50% | cpu8 sys 91% | 31,636 B | 1.37 万 |
| 32768 | 3.48 | 3.521 | 50% | cpu11 sys 75% 加 cpu8 sys 15% | 31,740 B | 1.39 万 |
| 49152 | 3.58 | 3.615 | 62% | cpu8 sys 90% | 48,048 B | 0.94 万 |
| 64512 | 3.64 | 3.680 | 70% | cpu10 sys 60% 加 cpu8 sys 30% | 64,052 B | 0.72 万 |

下行 4 流：

| NTB_IN | Windows | 板端 tx | CPU1 sys / irq / sirq，idle | tx 计数/秒 |
|---|---|---|---|---|
| 16384 | 2.94 | 2.973 | 41 / 6 / 46，4% | 2.32 万 |
| 32768 | 2.67 | 2.644 | 21 / 4 / 71，2% | 1.03 万 |
| 32768 | 2.62 | 2.647 | 20 / 4 / 74，1% | 1.03 万 |
| 49152 | 2.86 | 2.869 | 17 / 3 / 77，1% | 0.75 万 |
| 64512 | 2.94 | 2.974 | 14 / 3 / 78，2% | 0.58 万 |

上行 1 流：

| NTB_IN | Windows | 板端 rx | CPU1 idle |
|---|---|---|---|
| 16384 | 3.20 | 3.223 | 46%（4.13 的 A2a） |
| 32768 | 3.20 | 3.220 | 48% |
| 49152 | 未测 | 未测 | 未测 |
| 64512 | 3.20 | 3.219 | 48% |

- 下行 1 流随 NTB_IN 单调上升，16 KB 到 64512 为 3.15 到 3.64（约 +15.6%）；16 KB 的 3.15 与 10-28 同口径的 3.13 一致。每个下行 tx 计数的平均字节数约等于 NTB_IN，tx 计数速率相应下降。每个 tx 计数对应一个 NTB 是推断：平均值大于单包上限 7,986 B，且与 2、4、6、8 个 8000 B 帧加头部相符。
- 下行 4 流不单调：32768 的 2.67、2.62 偏低且复现，16384 与 64512 相同。四档里 CPU1 都接近跑满（idle 1% 到 4%），tx 计数速率降到约四分之一，CPU1 的 sys 由 41% 降到 14%、sirq 由 46% 升到 78%，原因没有定位。32768 下四条流不均（641 / 553 / 661 / 814 与 743 / 729 / 388 / 761 Mbit/s）。
- 下行 1 流发送线程所在的核会迁移，把几个核的 sys 加起来都约 90%；3.64 不能当作链路上限。
- 板端 `/proc/net/dev` 发送方向的 drop 列在所有 `.cpu` 前后都是 0；接收方向的 drop 列在每个 10 秒窗口内不变，在 `ntbset` 重新枚举后增加（增量不固定）。
- 板端与 Windows 相差：下行 -1.0% 到 +1.2%（32768 下行 4 流第一次板端 2.644 低于 Windows 的 2.67），上行约 +0.6%。
- 标签核对：标签 `o64i64-dn1` 的板端读回 `ntb_in_size=32768`，该结果（3.48）归入 32768；标签 `o64i49-dn1` 没有板端文件，按顺序推断也是 32768，其 Windows 结果 3.42 与另一次 32768 的 3.42 相同，归入 32768 一档；49152 和 64512 的下行 1 流由 `o64i49-dn1b`、`o64i64-dn1b` 测得。NTB_IN 64512 的上行 1 流第一次尝试板端 120 秒内没检测到流量，弃用，重跑为 3.20。下行 4 流的 Windows 贴回多数没有 `run` 命令行，标签未核实。`o64i64-dn1b` 那组结束时板端有两个 `exp` 后台任务，`cpuwin` 共用固定的临时文件，该组 CPU 百分比不能完全排除受影响，板端 tx 与 Windows 的比值（1.011）与其他组一致。

### 4.15 2026-10-29 MTU 1500 加 NTB 大小

条件：接在 4.14 之后，同一次开机，板端 `max_segment_size=1514`、MTU 1500，`rps_cpus=fffc`，super-speed；每组前 `ntbset` 后 Windows `chk 1500` 回传 `NlMtu=1500 ping=True`；`-t 20 -O 3`；每格 1 次。配置 B 为 `ntb_out=64512`、`ntb_in=64512`，配置 C 为 `ntb_out=16384`、`ntb_in=16384`（固件默认）。

| 组 | Windows | 板端 ncm0（10 秒） | 每个计数 | CPU1 idle | 其他较忙的核 |
|---|---|---|---|---|---|
| B 上行 1 流 | 2.68 | rx 2.759 | 1,500 B，23.0 万包/秒 | 11% | cpu7 sirq 62%，cpu8 sys 60% |
| B 下行 1 流 | 1.97 | tx 2.059 | 47,169 B，0.55 万个/秒 | 51% | cpu8 sys 87%、idle 0% |
| B 下行 4 流 | 2.22 | tx 2.340 | 47,266 B，0.62 万个/秒 | 35% | cpu5、7、8、12 sirq 33% 到 40% |
| C 上行 1 流 | 2.22 | rx 2.282 | 1,500 B，19.0 万包/秒 | 18% | cpu7 sirq 55%，cpu8、9 sys 25% |
| C 下行 1 流 | 1.88 | tx 1.977 | 15,245 B，1.62 万个/秒 | 50% | cpu8 sys 88%、idle 0% |
| C 下行 4 流 | 2.06 | tx 2.166 | 15,280 B，1.77 万个/秒 | 42% | cpu3 sirq 54%，cpu4 sirq 64%，cpu9 sirq 33% |

- B 对 C：上行 1 流约 +20.7%，下行 1 流约 +4.8%，下行 4 流约 +7.8%。B 对 10-28 同口径（NTB_OUT 64512、NTB_IN 16 KB，4.12）的 2.68 / 1.86 / 2.08：上行相同，下行 1 流约 +5.9%，下行 4 流约 +6.7%。
- 默认配置 C 的上行 2.22、下行 1 流 1.88，与 10-27 的 2.16 到 2.22、1.89 到 1.91 一致；下行 4 流 2.06 是 RPS 开、默认 NTB 下第一个 Windows 口径的数。
- B 的每个下行 tx 计数平均约 47 KB，没到 NTB_IN 64512（约 64 KB）。代码里 `TX_MAX_NUM_DPE` 为 32（每个 NTB 最多约 31 个 1514 B 数据报，约 47 KB），数值相符，这是推断，没有单独验证。
- 下行 1 流在 B、C 都是 cpu8 sys 87% 到 88%、idle 0%。上行 CPU1 忙度 89%（B）和 82%（C），包速率 23.0 万和 19.0 万/秒，与 4.12 一致，仍是未验证的候选瓶颈。
- C 的下行 4 流四条流不均（613 / 650 / 394 / 402 Mbit/s），B 相对均匀（571 / 531 / 528 / 594 Mbit/s）。
- 板端比 Windows 高：上行 2.8% 到 2.9%，下行 4.5% 到 5.4%。
- `ntbset` 状态行没有贴回，不清楚自动重绑是否触发，也不清楚 Windows 的 MTU 是自己回到 1500 还是手动改过。

### 4.16 2026-10-29 到 10-30 过夜稳定性（jumbo，NTB_OUT 与 NTB_IN 均为 63 KB）

条件：`ntbset 64512 8000 7986 64512`，`rps_cpus=fffc`，MTU 7986，super-speed；板端 `sh /tmp/soak.sh 660 60`，10-29 19:49:39 起，10-30 06:50:35 结束（`events.log` 的 `END`）；Windows `soak 7986 600 10`，19:50:06 起，每相 600 秒，不带 `-O`，12 轮，末轮 bidir 05:45:39 起，05:55:45 记 `END`；每相前 `chk` 检查 `NlMtu` 与 ping。方法见 3.8。用户手动测，原始数据在本地记录里。

Windows（`summary.csv`，12 轮共 60 个测量相，没有 FAIL，没有 LINKDOWN）：

| 相 | 轮数 | 首轮 | 最小 | 最大 | 平均 | 最小相对首轮 |
|---|---|---|---|---|---|---|
| 上行 1 流 | 12 | 3.20 | 3.18 | 3.20 | 3.19 | -0.6% |
| 下行 1 流 | 12 | 3.60 | 3.54 | 3.69 | 3.64 | -1.7% |
| 下行 4 流 | 12 | 2.92 | 2.88 | 2.93 | 2.90 | -1.4% |
| 下行 8 流 | 12 | 2.86 | 2.81 | 2.87 | 2.84 | -1.7% |
| 同时跑：上行 | 12 | 3.23 | 3.20 | 3.25 | 3.23 | -0.9% |
| 同时跑：下行 | 12 | 1.55 | 1.55 | 2.96 | 1.92 | 波动大，无单调趋势 |

- 单向各相与 4.14 的同配置值一致（上行 1 流 3.20、下行 1 流 3.64、下行 4 流 2.94），下行 4 流这一夜略低（2.88 到 2.93）。
- 同时跑的下行重传按轮次依次为 24、16、75、15、26、18、23、24、24、30、56、24，与下行吞吐没有明显对应关系，原因没查。下行相的重传列在单向下行里都是 0；上行相 Windows 是发送端，不报重传。
- dn1 最低的 3.54 出现在第 9 轮，下一轮回到 3.61。

板端（`mon.log` 660 行，每 60 秒一行）：

- `tx_dropped`、`tx_errors`、`rx_errors` 660 行全为 0；每小时样本均为 super-speed、carrier=1；`events.log` 里 `LINK speed=` 行数为 0（该文件有 1 MB 上限，`END` 行写在同一个保护下，说明没到上限）。
- `MemAvailable` 最小 15995108、最大 16161244 kB（约 1% 波动），每小时样本没有下降趋势；温度 44 到 47 ℃。
- buddy（DMA zone）order 10 每小时 3904 到 3917；order 7 到 9 有波动，不是单调下降。最低的两个样本（10-30 00:50：order 7 到 9 为 23 / 3 / 4，05:50：6 / 4 / 3）都落在同时跑的相内，00:50 的下一小时回升到 54 / 17 / 5。
- 10-30 10:07 核对：当前 dmesg 和 `events.log` 对 `Call Trace`、`page allocation failure`、`Out of memory`、`oom-kill`、`BUG:`、`Oops`、`soft lockup`、`rcu stall`、`hung task` 均无命中。覆盖不全：dmesg 在 10-30 02:40 以前已回绕，`events.log` 只在 `dmesg_hits` 变化时转储 30 行。

dmesg 里的 NTB 解析报错（`Wrong NDP SIGN`、`Wrong NTH SIGN`）：

- 来自 `ncm_unwrap_ntb`：`Wrong NTH SIGN` 是 skb 开头不是 NTH 签名，`Wrong NDP SIGN` 是 NTH 通过后 `ndp_index` 处不是 NDP 签名。每次 NDP 报错后紧跟一次 NTH 报错，间隔不到 0.1 ms（309.407329 与 309.407387，25618.846109 与 25618.846162）。NTH 报错记录的 `skblen` 有 7500、9836、46764、27244、40620、11884、20076，都小于 63 KB；已看到的 6 条 HEAD（skb 开头 32 字节）都不以 `NCMH` 开头。
- 板端 `rx_errors` 一直是 0，原因见 3.8：出错路径清空队列后返回，计数不会增加。
- `dmesg_hits` 从 16 涨到 920（+904 行，约 452 对，是估计）发生在 21:21 到 23:07，按分钟对应到 Windows 的相（板端与 Windows 时钟没有单独核对，两边起始时刻相差 27 秒，对应有约一个采样的不确定）：第 2 轮同时跑 +58 行，第 3 轮 up1 +250，第 3 轮同时跑 +572（含起点附近 22:11:51 的 +50，不含则 +522），第 4 轮同时跑 +24。第 1 轮、第 2 轮 up1 和第 4 轮 up1 为 0。
- 约 23:07 起 `dmesg_hits` 停在 919 到 920，是缓冲写满，之后的报错不能再用它统计。10-30 10:07 板端核对时，dmesg 最老一条在 uptime 25618.85 s（`/proc/uptime` 为 52395.98 s，`date` 为 10-30 10:07:01，推出开机约 10-29 19:33:45，最老一条对应 10-30 02:40:43）。23:07 到 02:40 之间的报错已被覆盖，第 5 到 8 轮是否有报错未知，全夜总次数无法统计。
- 存活的 02:40:43 到 05:14:46（uptime 34861.33 s）有 `Wrong NTH SIGN` 460 条、`Wrong NDP SIGN` 460 条，次数相等。按 600 秒分桶对应 Windows 的相：

| 桶起点（uptime 秒） | NTH 数 | 墙钟 | 所在的相 |
|---|---|---|---|
| 25200 | 26 | 02:33:45 | 第 9 轮 up1（缓冲从 02:40:43 起，前 7 分钟已被覆盖） |
| 27600 | 22 | 03:13:45 | 第 9 轮 dn8 末尾 29 秒，第 9 轮同时跑 |
| 28200 | 129 | 03:23:45 | 第 9 轮同时跑 35 秒，第 10 轮 up1 |
| 28800 | 2 | 03:33:45 | 第 10 轮 up1 末尾 41 秒，dn1 |
| 30600 | 25 | 04:03:45 | 第 10 轮 dn8 57 秒，同时跑 |
| 31200 | 123 | 04:13:45 | 第 10 轮同时跑 63 秒，第 11 轮 up1 |
| 31800 | 9 | 04:23:45 | 第 11 轮 up1 末尾 68 秒，dn1 |
| 34200 | 110 | 05:03:45 | 第 11 轮同时跑 91 秒，第 12 轮 up1 |
| 34800 | 14 | 05:13:45 | 第 12 轮 up1 末尾 97 秒，dn1 |

  其余桶（02:43 到 03:13 的 dn1、dn4、dn8，03:43 到 04:03，04:33 到 05:03，05:23 以后）都是 0。存活窗口内 dn1、dn4、dn8 相里一条都没有；第 9 到 12 轮的 up1 相都有，同时跑的相第 9、10 轮有，第 11 轮同时跑（04:55 到 05:03 的桶为 0）和第 12 轮同时跑没有。
- 报错时段的上行吞吐没有变化（up1 3.19 到 3.20）。Windows 作为上行发送端不报重传，丢弃对 TCP 的影响看不出来。
- 此前的实验没有抓过 dmesg，不知道默认 16 KB 或 MTU 1500 下是否也有，也不知道是否与 `ntb_out=63 KB` 有关。后来的 4.18（NTB_OUT 16 KB）同样出现，63 KB 不是必要条件。
- 推断（未证实）：可能是一个 NTB 被分在两个 OUT 请求里，前一个请求里 NDP 的位置超出实际收到的数据，后一个是剩余部分、不带 NTH。依据只有：两类报错次数相等、间隔很短、`skblen` 都小于 63 KB，以及 `ncm_unwrap_ntb` 解析第一个 NTB 前只拿 `block_len` 与 `ntb_max` 比较，没有与 `skb->len` 比较。要证实需要在两处报错点打印 `skb->len`、`block_len`、`ndp_index`，要重编重刷。

### 4.17 2026-10-30 USB2（high-speed）NCM

条件：EVB，USB3 固件，不重刷；补丁版 `ntbset <ntb_out> 1514 1500 <ntb_in> high-speed`（3.9），MTU 1500，`max_segment_size` 1514，`rps_cpus=fffc`，板端 `TH=5000000`；`-t 20 -O 3`，各 1 次；每个 `.cpu` 头都是 `speed=high-speed`、`mtu=1500`。用户手动测，板端 uptime 69682 到 72346 s（10-30 14:55 到 15:40）。单位 Mbit/s。

| 组 | 方向 | Windows receiver | 板端 ncm0 | 板端每个计数 | CPU1 |
|---|---|---|---|---|---|
| 16 KB / 16 KB | 上行 1 流 | 336 | rx 345 | 1,496 B，2.88 万包/秒 | idle 83% |
| 16 KB / 16 KB | 下行 1 流 | 374 | tx 392 | 15,280 B，0.32 万个/秒 | sys 18% sirq 18% idle 59% |
| 16 KB / 16 KB | 下行 4 流 | 371 | tx 389 | 15,280 B，0.32 万个/秒 | sys 19% sirq 19% idle 57% |
| 63 KB / 63 KB | 上行 1 流 | 349 | rx 359 | 1,500 B，2.99 万包/秒 | idle 86% |
| 63 KB / 63 KB | 下行 1 流 | 369 | tx 386 | 47,284 B，0.10 万个/秒 | sys 17% sirq 17% idle 63% |
| 63 KB / 63 KB | 下行 4 流 | 369 | tx 386 | 47,284 B，0.10 万个/秒 | sys 17% sirq 17% idle 62% |

- 63 KB 对 16 KB：上行 1 流 +3.9%（336 到 349），下行 1 流 -1.3%（374 到 369），下行 4 流 -0.5%（371 到 369）。每秒区间值很稳：上行 333 到 338 对 348 到 350，下行 1 流 373 到 375 对 369 到 370（两组不重叠）；下行 4 流 370 到 373 对 367 到 371（有重叠）。每格 1 次，没有重复。
- 板端比 Windows 高：上行 +2.7%、+2.9%，下行 +4.6% 到 +4.9%，与 MTU 1500 下 USB3 的差距相近（4.15）。
- 板端没有 CPU 瓶颈：最忙的核 idle 57% 到 86%，下行 1 流的发送线程所在核没有出现（USB3 下是 cpu8 sys 88%）。
- 内核对 high-speed 上报的速率是 425,984,000 bit/s（`u_ether.h` 的 `gether_bitrate`）。板端下行计数 386 到 392、上行 345 到 359 Mbit/s，约为它的 91% 到 92% 和 81% 到 84%，只是和一个常数比，总线上实际能否达到没有验证。
- USB2 下行高于上行（374 对 336，369 对 349），USB3 同配置上行高于下行 1 流（2.22 对 1.88 Gbit/s）；USB2 下 4 流与 1 流相近，USB3 下 4 流高于 1 流。NTB 63 KB 在 USB2 下的变化（上行 +3.9%，下行 1 流 -1.3%）小于 USB3 同样改动（4.15：+20.7%、+4.8%）。原因没有验证。

数据质量与异常：

- 下行 4 流：
  - 16 KB 组：receiver 四条流 79.6 / 97.2 / 97.3 / 97.3，汇总 371；发送端汇总行 0.00-32.49 秒、重传 4 次，只用 receiver 行。
  - 63 KB 组第一次（和上行、下行 1 流链式运行）：Windows 报 `iperf3: error - control socket has closed unexpectedly`，每秒行只到 2.00-3.01，无汇总行；板端 READY 后 65 秒才检测到流量，窗口 tx 0.096 Gbit/s（推断只采到最后 2 到 3 秒）；作废。
  - 63 KB 组重跑（`dn4b`，服务端重启后单独跑）：receiver 四条流 95.2 / 90.2 / 91.7 / 91.9，汇总 369；发送端汇总行 0.00-57.51 秒、重传 8 次，只用 receiver 行。
  - 作废那一次之后读到的服务端日志尾部（4 条流）：第 56 到 59 秒每秒仍有 365 到 368 Mbit/s、重传 0，也就是服务端那一轮至少跑了 59 秒，而 Windows 贴回的输出只到第 3 秒，三处对不上，原因没查。
  - 4 流下行的发送端汇总行比 receiver 的 20 秒长很多（32.49 秒、57.51 秒），1 流下行没有这个现象（20.09、20.07 秒）；USB3 上没有记录这一项，不知道是否同样。
- 首次 Windows 运行 `Connection refused`：板端没有 iperf3 服务端（推断：10-29 夜里监控脚本结束时的 `killsrv` 杀掉了它，没有看到 `ps` 输出）；重启服务端后正常。
- `ntbset` 的速度自检不可信（3.9）。
- Windows 的 `NlMtu`、LinkSpeed 输出没有贴回；MTU 1500 以 `.cpu` 头的 `mtu=1500` 和上行平均 1,496 / 1,500 B 的包为据。
- `Wrong NTH SIGN`：10-30 10:07 的计数是 460，USB2 两组之前（14:55 之前的上行之后）为 459，最后一条时间戳 34861.326308 没变；63 KB 组的上行、下行 1 流和 `dn4b` 跑完后仍是 459，最后两条没变。USB2 两组的上行各约 23 秒都没有新增。USB3 上 63 KB 的 up1 相有的整相为 0，有的 10 分钟里 120 到 130 次，这点样本不能说明 USB2 下不出现。460 变 459 推断是环形缓冲继续回绕丢了最老的一条（没验证）。

### 4.18 2026-10-30 到 11-02 周末连续稳定性（jumbo，NTB_OUT 16 KB，NTB_IN 63 KB）

条件：补丁版 `ntbset 16384 8000 7986 64512`（3.9），`rps_cpus=fffc`，MTU 7986，super-speed（`mon.log` 首行 `ntb_out=16384 ntb_in=64512 mss=8000 mtu=7986 rps=fffc`）；板端 `sh /tmp/soak.sh 3840 60`，10-30 19:20:43 起；Windows `soak 7986 600 60`，19:21:15 起，每相 600 秒，不带 `-O`，11-02 07:25:04 记 `END`（60.06 小时）。第一个样本 `dmesg_hits=0`，与开跑前 `dmesg -c` 清空相符。方法见 3.8。用户手动测。与 4.16 只有 `ntb_out` 不同（63 KB）。

Windows（`summary.csv`：72 轮，428 行数据，没有 FAIL、LINKDOWN 和空值；第 72 轮只有 up1、dn1，Windows 到时结束）。速率单位 Gbit/s：

| 相 | 轮数 | 首轮 | 最小~最大 | 平均 | 前 10 轮平均 | 第 62 到 71 轮平均 | 最小相对首轮 |
|---|---|---|---|---|---|---|---|
| 上行 1 流 | 72 | 2.42 | 2.36~2.43 | 2.42 | 2.42 | 2.41 | -2.5% |
| 下行 1 流 | 72 | 3.58 | 3.57~3.69 | 3.65 | 3.63 | 3.66 | -0.3% |
| 下行 4 流 | 71 | 3.08 | 2.95~3.12 | 3.03 | 3.03 | 2.99 | -4.2% |
| 下行 8 流 | 71 | 2.94 | 2.87~2.97 | 2.91 | 2.91 | 2.92 | -2.4% |
| 同时跑：上行 | 71 | 2.64 | 2.01~2.71 | 2.62 | 2.57 | 2.67 | 不适用 |
| 同时跑：下行 | 71 | 3.18 | 2.44~3.33 | 3.12 | 3.08 | 3.16 | 不适用 |

- 单向各相没有低于首轮 95% 的行，最低是下行 4 流（-4.2%）；前 10 轮与第 62 到 71 轮的平均相差都在 1.2% 以内（上行 1 流 -0.4%，下行 1 流 +0.9%，下行 4 流 -1.2%，下行 8 流 +0.4%）。同时跑的相波动大，不套用 95% 判据。
- 重传（Windows 作为上行发送端不报重传）：下行 1 流 72 轮全为 0；下行 4 流 71 轮中 4 轮非 0（第 4 轮 44，其余各 1）；下行 8 流 71 轮中 3 轮非 0（最多 3）。
- 同时跑：下行重传 71 轮中 68 轮非 0（中位数 59，最大 353）。上行低于 2.5 的有 11 轮，其中 10 轮的下行为 3.25~3.33（另一轮 2.79）；重传不少于 150 的有 12 轮，下行 3.11~3.19、上行 2.58~2.70；下行低于 2.8 的有 5 轮（2.44~2.79）。原因没查。10-29 夜（NTB_OUT 63 KB）同时跑上行 3.20~3.25、下行 1.55~2.96。
- `summary.csv` 是我从用户贴回的终端文本转录的：周期结构用脚本核对过（71 个完整周期加第 72 轮 2 行，时间递增，周期间隔 3028~3030 秒），逐格数值没有独立核对。

板端（`mon.log` 3745 个样本，贴回每小时一行共 63 行）：

- 63 个每小时样本的 `rx_e`、`tx_e`、`tx_d` 全为 0（其余 3682 行没看）；`LINK speed=` 事件 0；`events.log` 对 `Call Trace`、`page allocation failure`、`Out of memory`、`oom-kill`、`BUG:`、`Oops`、`soft lockup`、`rcu stall`、`hung task` 无命中。覆盖不全：`events.log` 只在 `dmesg_hits` 变化和 `nth` 增长时写，`dmesg_hits` 约 22:21 起写满不再变化，之后只有 `nth` 增长时的 3 行摘要；当前 dmesg 没查这些关键字。
- `MemAvailable`：3745 行最小 16053448、最大 16151612 kB（波动 0.61%）；每小时样本的线性拟合斜率约 -206 kB/小时（62 小时共 -12.8 MB，约 0.08%），小于样本间的波动，看不出趋势。
- 板端 uptime 310754.64 s（约 86.3 小时），开机约 10-29 19:33，期间没有重启。贴回时（11-02 09:53）soak 还在跑（到 11:20 结束），`events.log` 末尾没有 `END`。`buddyinfo`、温度没有贴回。

NTB 解析报错（`Wrong NDP SIGN` / `Wrong NTH SIGN`）：

- `nth` 累计 11232 次。按小时增量：有流量的 60 个小时每小时 38~318 次（平均 187），Windows 结束（07:25）后的最后 2 小时为 0。第 1 小时 270 次，前 3 小时共 576 次（22:21）。
- 10-29 夜（NTB_OUT 63 KB）开始后约 1.5 小时 `dmesg_hits` 没变（没有报错），第一次变化在 21:21；这次 NTB_OUT 16 KB 一开始就有。两个配置的发生率不能比：10-29 夜 dmesg 回绕丢了 23:07 到 02:40 的记录。
- `nth` 的核对：缓冲写满之前 `dmesg_hits` 等于 `nth` 的 2 倍（20:20 为 540 对 270，21:20 为 896 对 448；每次报错 NDP、NTH 各一行）；22:21 起 `dmesg_hits` 在 911~922 波动（缓冲约容纳 456~461 对），`nth` 继续累计。累计总数没有独立核对。
- 当前 dmesg（11-02 09:53）有 456 条 `Wrong NTH SIGN`，最老一条 uptime 294698.37 s（按开机约 10-29 19:33:45 推算为 11-02 05:25:23），最后一条 301215.19 s（07:14:00），落在 Windows 第 72 轮 up1（07:04:53 起，600 秒）内；之后 `nth` 保持 11232（07:27、08:28、09:28 三个样本同值），第 72 轮 dn1（07:14:58 起）和 Windows 结束后都没有新增。这个对应依赖板端和 Windows 时钟一致，没有单独核对。
- 这次没有把增量分到各个相，所以 4.16 里"下行 1、4、8 流相内没有报错"的说法没有在这次数据上重新验证。
- 能确认的：NTB_OUT 16 KB 下同样出现，NTB_OUT 63 KB 不是必要条件。两次运行相同的条件是 jumbo、NTB_IN 63 KB、RPS 开、Windows 10 主机，哪个是必要条件没有验证；NTB 均为 16 KB、MTU 1500 下没测（4.17 的 USB2 两组上行各约 23 秒没有新增，样本太短）。

### 4.19 2026-11-02 补测表 1、表 2 的 NA 格子

条件：补丁版 `ntbset`（3.9），切换后 `sleep 5` 再读 `current_speed`（12 步里除表 2 的两步为 `high-speed`，其余都是 `super-speed`）；`rps_cpus=fffc`（表 1 第 3 行手动写 0）；Windows `-t 20 -O 3`，每项 1 次。开跑前收尾周末 soak、存档 dmesg 并 `dmesg -c` 清零、重启 `iperf3 -s`；之后每个板端块先打印 `Wrong NTH SIGN` 累计数。用户手动测。

Windows receiver 汇总（Gbit/s，表 2 为 Mbit/s）：

| 表中位置 | 配置 | 上行 4 流 | 下行 1 流 | 四条上行流 receiver（Mbit/s） |
|---|---|---|---|---|
| 表 1 第 12 行 | 7986，63 / 63，RPS 开 | 3.19 | 未跑 | 872 / 1070 / 563 / 688 |
| 表 1 第 11 行 | 7986，63 / 48，RPS 开 | 3.20 | 未跑 | 685 / 797 / 775 / 943 |
| 表 1 第 10 行 | 7986，63 / 32，RPS 开 | 3.19 | 未跑 | 794 / 836 / 774 / 787 |
| 表 1 第 15 行 | 7986，16 / 63，RPS 开 | 2.42 | 未跑 | 651 / 584 / 555 / 634 |
| 表 1 第 4 行 | 7986，16 / 16，RPS 开 | 2.42 | 3.02 | 617 / 637 / 621 / 548 |
| 表 1 第 3 行 | 7986，16 / 16，RPS 关 | 2.42 | 3.01 | 566 / 629 / 594 / 635 |
| 表 1 第 8 行 | 7986，42 / 16，RPS 开 | 没有贴回 | 3.13 | 没有贴回 |
| 表 1 第 9 行 | 7986，48 / 16，RPS 开 | 3.09 | 3.12 | 885 / 596 / 733 / 876 |
| 表 1 第 13 行 | 1500，63 / 63，RPS 开 | 2.67 | 未跑 | 686 / 661 / 668 / 654 |
| 表 1 第 14 行 | 1500，16 / 16，RPS 开 | 2.21 | 未跑 | 649 / 496 / 548 / 519 |
| 表 2 U2 | high-speed，1500，63 / 63 | 350 Mbit/s | 未跑 | 128 / 56.5 / 84.2 / 80.9 |
| 表 2 U1 | high-speed，1500，16 / 16 | 332 Mbit/s | 未跑 | 87.5 / 116 / 69.8 / 58.2 |

- 仍为 NA 的两格：表 1 第 11 行上行 1 流（`na-r11-up1`）和第 8 行上行 4 流（`na-r8-up4`）：贴回的输出里没有。Windows 的 `run` 会把输出存到 `C:\b\<标签>.txt`，这两个文件如果存在，可以取回 receiver 行。
- 上行 4 流与 1 流接近：已有两项都测过的各行相差不超过 0.03（默认配置的区间除外），最高配置的上行 4 流 3.19、1 流 3.20。
- 下行 1 流：jumbo 下 NTB_OUT 16 KB 为 3.02（RPS 关 3.01），32 KB 为 3.09（4.10），42 KB 以上为 3.12~3.15；MTU 1500 下 1.86~1.88。RPS 关、开对上行 4 流（2.42 对 2.42）和下行 1 流（3.01 对 3.02）都没有作用，只是 jumbo 16 KB 一个配置。
- 四条上行流不均匀：最慢的流是最快的 0.44~0.95 倍（USB3 最低是第 12 行的 563 对 1070，USB2 最低是 U2 的 56.5 对 128）。
- 全部是 1 次测量，没有重复。

`Wrong NTH SIGN` 累计数（读数在该步 `ntbset` 之前打印，下一步的读数减本步的读数 = 本步 `ntbset` 重新枚举加 Windows 运行期间的新增）：

| 区间 | 对应的步骤 | 新增 |
|---|---|---|
| 步骤 1、2 | 表 1 第 12、11 行，jumbo，上行 4 流 | 13（两步合计，步骤 2 开头的读数没有贴回） |
| 步骤 3 | 第 10 行，jumbo，63 / 32，上行 4 流 | 15 |
| 步骤 4 | 第 15 行，jumbo，16 / 63，上行 4 流 | 5 |
| 步骤 5 | 第 4 行，jumbo，16 / 16，上行 4 流加下行 1 流 | 5 |
| 步骤 6 | 第 3 行，jumbo，16 / 16，RPS 关，上行 4 流加下行 1 流 | 12 |
| 步骤 7 | 第 8 行，jumbo，42 / 16，下行 1 流（上行 4 流没有贴回） | 6 |
| 步骤 8 | 第 9 行，jumbo，48 / 16，上行 4 流加下行 1 流 | 10 |
| 步骤 9 | 第 13 行，MTU 1500，63 / 63，上行 4 流 | 0 |
| 步骤 10 | 第 14 行，MTU 1500，16 / 16，上行 4 流 | 1 |
| 步骤 11 | 表 2 U2，high-speed，63 / 63，上行 4 流 | 0 |
| 步骤 12 | 表 2 U1，high-speed，16 / 16，上行 4 流 | 没有读数 |

- 每步只有 20 秒到约 1 分钟，各 1 次，只能当粗略对比；步骤 5、6、8 里还有下行 1 流，没有拆开上行和下行。
- jumbo 的各区间都有新增（5~15 次，步骤 1、2 合计 13），包括 NTB_IN 为 16 KB 和 NTB 均为 16 KB 的步骤（5、6、7、8），所以 NTB_OUT、NTB_IN 的大小都不是必要条件。MTU 1500 和 USB2 的三个区间合计 1 次。
- jumbo 是否必要未定：样本短；而且 4.16 里 10-29 夜 63 KB / 63 KB 的前约 1.5 小时（含单流上行相和一路上行加一路下行同时跑的相）没有报错，这次是 4 流上行，没有同配置的 1 流上行对照。

## 5. 分析

1. **4 流下行（MTU 1500）**：主机回来的 ACK 原本全部由 CPU1 接收，TCP 发包和 NCM 封包拷贝也在 CPU1 的软中断里执行，4 流时 CPU1 softirq 77% 到 80%、idle 1% 到 2%（4.1、4.3），单流时 softirq 35%、idle 30%。开启 RPS 后 ACK 处理分散到其他核，CPU1 idle 回到 42% 到 47%（4.2、4.4），吞吐提升约 50% 到 65%。所以默认参数下 4 流比单流慢，是所有流的收包处理挤在 CPU1 上。jumbo 加 16 KB 下同样如此：RPS 关 2.47、CPU1 idle 1%，RPS 开 3.16 到 3.19、CPU1 idle 13%（4.9）。
2. **单流下行**：限制在板端 iperf3 发送线程所在的核（4.4 的 CPU8、4.11 的 cpu9，sys 88% 到 90%、idle 0%），即单个 TCP 发送进程从用户态拷贝数据的开销。这是测试工具的单线程上限，不是 NCM 驱动的上限；实际业务若为多连接，可参考 4 流结果。
3. **上行（MTU 1500，NTB_OUT 16 KB）**：RPS 开/关、单流/4 流、qmult 5/80、IRQ 放在 CPU1/CPU4 的各种组合下，都稳定在 2.22 到 2.29 Gbit/s（约 19 万包/秒），板端没有核跑满，**已排除板端 CPU 和 USB 请求队列深度**。jumbo 使包速率降到 5 分之 1，吞吐只涨约 10%，**也排除每包开销是主要限制**。
4. **NTB_OUT 对上行**：加大后上行持续上升（jumbo 下 2.40 到 2.44、2.88 到 2.89、3.19 到 3.20；MTU 1500 下 2.16 到 2.22、2.68），Windows 10 的 UsbNcm 跟随了设备上报的值。jumbo 下上行时板端 cpu1 idle 随 NTB_OUT 增大而增大（16 KB 为 34%，32 KB 为 40% 到 42%，64512 为 45% 到 46%）。MTU 1500 加 64512 时 cpu1 idle 只有 11% 到 13%（约 23 万包/秒），是候选瓶颈，没到 100%，也没有 `perf` 可以拆开，未验证。
5. **NTB_OUT 对 jumbo 下的下行 4 流**：随 NTB_OUT 增大而下降后趋平（16 KB、32 KB、43008、49152、64512 依次为 3.15 到 3.19、3.12、3.01、2.96、2.92 到 2.96），同时 cpu1 idle 为 10% 到 13%、11%、6%、5%、3% 到 5%，64512 时 cpu1 接近跑满；10-29 同一次开机内只改 `ntb_out_size` 也得到 3.15 对 2.96（4.13）。下行 1 流小幅上升（3.02 → 3.13~3.15，4.19）。下降原因没有验证，有一条候选线索（代码和配置事实加推断）：
   - 下行时板端 rx 方向只收主机回来的 ACK（约 2.2 万个/秒，每个 40 B），但每个 rx 请求的缓冲长度是 `fixed_out_len`，也就是 `NTB_OUT_SIZE`（`u_ether.c` 的 `rx_submit`）；
   - gadget 层对每个请求按 `req->length` 做 `dma_map_single` 和 `dma_unmap_single`（`udc/core.c`）；
   - `axhelix.dtsi` 里 `dma-noncoherent` 只出现在 GIC 和 ITS 节点，axera dts 目录里没有 `dma-coherent`，没有确认 dwc3 的实际一致性；
   - 若不一致，每个 ACK 完成时的缓存维护量会随缓冲长度增大。32 KB 到 64512，下行 1 流和 4 流的 cpu1 忙度都多了约 7 个点，量级与此相符，但这不是证明。
6. **UDP**：iperf3 单流 UDP 两端都上不去（PC 约 1.5，板子约 1.35 Gbit/s），测不出链路能力（4.7）。
7. **主机侧**：Windows 10 的 UsbNcm 跟随了设备上报的 NTB 大小（上行随 NTB_OUT 上升）；在板端 `max_segment_size` 改成 8000 并重新枚举后 `NlMtu` 变成 7986。Windows 每核 CPU 在 10-27 的 16 KB 测试中最忙核峰值 78% 到 85%，未见持续满载；NTB_OUT 加大后没有重新采。Linux `cdc_ncm` 的行为没测（内核头文件里 TX/RX 的 NTB 最大值是 65536，默认 16384，是否自动使用设备上报值没有验证）。
8. **NTB_IN（jumbo，NTB_OUT 64512，4.14）**：下行 1 流随 NTB_IN 上升（3.15 到 3.64），上行 1 流不变（3.20），下行 4 流不单调（32768 偏低）。下行 1 流的发送线程所在核 sys 合计在各档都约 90%，吞吐随 NTB_IN 增大而上升，说明同样占用下每字节的开销在下降；原因没有验证。板端每个 tx 计数的平均字节数约等于 NTB_IN，说明 Windows 10 采纳了设备上报的 IN 大小。下行 4 流 CPU1 一直接近跑满，tx 计数速率降到约四分之一时 CPU1 的占用没有随之下降，只是 sys 与 sirq 的比例变了，原因没有定位。
9. **MTU 1500 下增大 NTB（4.15）**：NTB_OUT 与 NTB_IN 同为 64512 对默认 16 KB，上行 1 流约 +20.7%，下行 1 流约 +4.8%，下行 4 流约 +7.8%；绝对值低于 jumbo。下行每个 tx 计数平均约 47 KB，与 `TX_MAX_NUM_DPE`=32 的上限相符（推断）。上行 CPU1 忙度仍在 82% 到 89%，没有 `perf`，未验证。
10. **大 NTB 的内存分配**：`ncm_wrap_ntb` 每个新 NTB 按 `ntb_in_size` 整块 `alloc_skb(GFP_ATOMIC)`，NTB_IN 64512 时是 64 KB 级的连续分配，失败会计入 `tx_dropped`。10-29 的各组（每组约 23 秒）和 10-29 夜里 11 小时（4.16，63 KB / 63 KB）板端发送方向 drop 列都是 0（660 个样本），`MemAvailable` 波动约 1%，buddy order 10 稳定在 3904 到 3917；只测了这一个配置一夜，没有 16 KB 对照。
11. **NTB 解析报错**（4.16）：10-29 夜里 dmesg 出现成对的 `Wrong NDP SIGN` / `Wrong NTH SIGN`，`ncm_unwrap_ntb` 丢弃该 OUT 请求的 skb，`rx_errors` 不增加。能确认的：出现在含上行的相，存活窗口（10-30 02:40 到 05:14）内下行 1、4、8 流相里没有；上行吞吐没有变化。10-30 到 11-02 的约 60 小时（NTB_OUT 16 KB，NTB_IN 63 KB，4.18）同样出现，累计 11232 次，所以 NTB_OUT 63 KB 不是必要条件。11-02 的短测（4.19）里 NTB_IN 为 16 KB、NTB 均为 16 KB 的 jumbo 配置也有新增，所以 NTB_IN 63 KB 也不是必要条件；MTU 1500 的两个区间合计 1 次、USB2 为 0，jumbo 是否必要未定（样本短）。不能确认的：成因、jumbo 是否必要、两个配置的发生率比较、10-29 夜 23:07 到 02:40 的分布。成因只有一条未证实的推断（一个 NTB 被拆在两个 OUT 请求里），见 4.16。
12. **USB2（high-speed）**（10-30，MTU 1500，RPS 开，4.17）：NTB 16 KB / 16 KB 时上行 1 流 336、下行 1 流 374、下行 4 流 371 Mbit/s；NTB 63 KB / 63 KB 时 349 / 369 / 369。63 KB 对 16 KB：上行 +3.9%，下行 1 流 -1.3%，下行 4 流 -0.5%，每格 1 次。板端 cpu 不忙（最忙的核 idle 57% 到 86%）。USB2 下行高于上行，USB3 同配置相反；下行 4 流与 1 流相近（371 对 374，369 对 369），USB3 下 4 流高于 1 流。内核对 high-speed 上报的速率是 425.984 Mbit/s（`gether_bitrate`），板端下行计数 386 到 392 Mbit/s 约为它的 91% 到 92%，上行 345 到 359 Mbit/s 约为 81% 到 84%，只是和一个常数的比较，总线上能否达到没有验证。
13. **补测 NA 格子**（11-02，4.19）：补了 15 格，剩表 1 第 8 行上行 4 流和第 11 行上行 1 流（输出没有贴回）。上行 4 流与 1 流接近（相差不超过 0.03，默认配置的区间除外）；4 流与 1 流的差距在下行：jumbo 下 NTB_OUT 为 16 KB 和 32 KB 时下行 4 流略高于 1 流，42 KB 起低于 1 流（63 KB / 63 KB 为 2.94 对 3.64）。下行 1 流在 jumbo 下随 NTB_OUT 小幅上升（16 KB 3.02，32 KB 3.09，42 KB 以上 3.12~3.15），MTU 1500 下基本不变。RPS 对上行和下行 1 流都没有作用（jumbo 16 KB）。原因都没有验证。

## 6. 调优计划与进度

每次只改一个变量。

| # | 项目 | 方法 | 状态 |
|---|---|---|---|
| 1 | RPS | `rps_cpus=fffc` | 已测（10-22、10-23、10-28）：多流下行有效，上行无作用 |
| 2 | 加深 USB 请求队列 | qmult 5 → 10 → 80（保持 RPS），验证上行 | 已测（10-23）：TRB 确认生效，上行不变，**已排除** |
| 3 | 中断与进程绑核 | dwc3 中断和 iperf3 分开到不同核 | 部分完成（10-23）：qmult 实验中 dwc3 中断绑 CPU4，上行不变；iperf3 绑核未做 |
| 4 | jumbo | `max_segment_size` 8000，MTU 7986，需要主机侧配合 | 已测（10-28）：见 4.9、4.12 |
| 5 | NTB_OUT | 10-28 改 `f_ncm.c` 的 `NTB_OUT_SIZE` 重编烧录；10-29 起用 configfs 属性 `ntb_out_size` | 已测：16 KB / 32 KB / 43008 / 49152 / 64512（jumbo），MTU 1500 下 16 KB / 64512；改动只在本地 |
| 6 | 补齐配置矩阵 | 表 1、表 2 里标 NA 的格子；第 5 到 15 行的 RPS 关 | 部分完成（10-29：MTU 1500 同日对照、NTB_OUT 中间值；11-02：补了 15 个 NA 格子，4.19）；还剩表 1 第 8 行上行 4 流和第 11 行上行 1 流（输出没有贴回），第 5 到 15 行的 RPS 关未测 |
| 7 | 其他主机 | Win11、Linux `cdc_ncm`、macOS；原生 iperf3 替换 Cygwin 版；Windows 每核 CPU 在 NTB_OUT 加大后重采 | 待做；10-27 用开发板当 host 未成功（SS 没起来） |
| 8 | RNDIS 基线 | 按相同方法测试，与 NCM 对比 | 待做 |
| 9 | 纯 bulk 链路上限 | 板端 SourceSink（`CONFIG_USB_F_SS_LB`）加主机侧测试程序，或换 Linux 主机用 testusb | 待做 |
| 10 | UDP | iperf3 UDP 受工具限制 | 暂停，不作为调优方向 |
| 11 | NTB_IN | configfs 属性 `ntb_in_size`，`ntbset` 切换 | 已测（10-29）：jumbo 下 16 KB / 32 KB / 49152 / 64512，MTU 1500 下 16 KB / 64512；改动只在本地 |
| 12 | 稳定性长跑 | 大 NTB 配置整夜跑，看 `ncm0` 的 drop、dmesg、`/proc/buddyinfo`（方法见 3.8） | 部分完成：jumbo，NTB 63 KB / 63 KB 一夜（4.16）；NTB_OUT 16 KB / NTB_IN 63 KB 连续约 60 小时（4.18）；NTB 均为 16 KB、MTU 1500 未测 |
| 13 | 并发 | 上下行同时跑；流数超过 4 | 部分完成（10-29 夜）：下行 8 流、一路上行加一路下行；上行多流和更多流数未测 |
| 14 | cpu1 软中断拆解 | 取下行 4 流和 MTU 1500 上行时 `/proc/softirqs`、`/proc/interrupts` 的差值 | 待做 |
| 15 | 稳定性对照夜 | 只把 `ntb_out_size` 改回 16 KB，其余同 4.16，整夜跑；用带 `nth=` 的脚本，开跑前 `dmesg -c` 存档（3.8） | 已测（10-30 到 11-02）：连续约 60 小时，NTB 解析报错在 NTB_OUT 16 KB 下同样出现，见 4.18 |
| 16 | NTB 解析报错定位 | 对照结果见 4.18（NTB_OUT 16 KB 下同样有）；在 `ncm_unwrap_ntb` 两处报错点打印 `skb->len`、`block_len`、`ndp_index`，需要重编重刷 | 待做 |
| 17 | USB2（high-speed）NCM | configfs 限到 high-speed（3.9），MTU 1500，NTB 16 KB / 63 KB，上行 1 流、上行 4 流、下行 1 流、下行 4 流 | 已测（10-30、11-02）：见 4.17、4.19；jumbo、RNDIS 未测 |

修改 qmult 前需要先解绑并删除 function 链接，否则内核返回 EBUSY：

```sh
G=/sys/kernel/config/usb_gadget/usb_ncm
/soc/scripts/usb-ncm.sh stop
rm $G/configs/c.1/ncm.usb0
echo 20 > $G/functions/ncm.usb0/qmult
/soc/scripts/usb-ncm.sh start
```

RPS、`max_segment_size`、MTU 都是运行时设置，板子重启后失效，每次测试前要重新设置；10-29 起的 `ntb_out_size`、`ntb_in_size` 同样重启后回到 16384，可用 3.7 的 `ntbset` 一次设好。

## 7. 局限

- 只有单台主机（Windows 10），每个配置 1 到 3 次测量，没有做多轮统计；10-23 同配置两次曾相差约 11%。
- 表 1 第 2 行是板端 `ncm0` 计数，与其他行的 Windows 汇总值口径不同（第 14 行是同配置的 Windows 值）；标 NA 的格子（表 1 第 8 行上行 4 流、第 11 行上行 1 流）没有数据；第 5 到 15 行没有测 RPS 关。
- 10-28 的 NTB_OUT 测试中，部分组没有 `Tee-Object` 输出文件，Windows 侧只保留了汇总行；有几条被中途打断的数据没有放进表。
- NTB_OUT 增大时 jumbo 下的下行 4 流下降、NTB_IN 为 32 KB 时下行 4 流偏低，原因都未定位；没有 `perf`，无法把 cpu1 的占用拆开；NTB_OUT 为 32 KB 的下行 4 流只有 1 次。
- 没有直接观测到 Windows 实际发出的 NTB 大小，上行提升与 NTB_OUT 的对应关系来自吞吐和 cpu 数据的间接推断。
- 10-28 的 MTU 1500 加 64512 下行 4 流（2.08）没有同日同固件的 16 KB 对照；10-29 的 4.15 补了同一次开机的对照（2.22 对 2.06）。
- `NTB_OUT_SIZE`（10-28 改常量）和 `ntb_out_size`、`ntb_in_size`（10-29 改成 configfs 属性）的改动只在本地 `f_ncm.c`、`u_ncm.h`，没有提交；它们影响所有使用这个 gadget 的主机，其他主机（Win11、macOS、Linux）的兼容性、更大缓冲对内存的影响都没有测。
- Windows 的 MTU 为什么在板端改值后自己变成 7986，没有查清；10-28 的 MTU 1500 加 64512 那一轮没有读回 Windows 的 `NlMtu`；10-29 切到 MTU 1500 后 `chk` 读回 1500，但不清楚是自动回到 1500 还是手动改过。
- 尚未单独测试 gadget 模式下 USB 链路的纯传输极限（例如 bulk loopback）。
- 板端脚本已在真板上完成 NCM 启动和枚举。重复 start/stop、RNDIS 启动及其 Windows 驱动绑定尚未验证。
- 10-27 的 UDP 六组命令与计划不一致且缺 `-l 1472`，U6 的流数记录不一致，只能当定性参考；每组的 UDP 计数增量没有保存。
- 开发板当 host 的对照没有完成；主机的两颗 Intel xHCI 没有对比。
- 10-29 的数据每格只有 1 到 2 次；部分组没有贴回 `ntbset` 状态行和 Windows 的 `NlMtu`；多数 Windows 贴回没有 `run` 命令行，标签未逐一核实；板端文件名和条件不一致时，以 `.cpu` 第二行为准。
- 10-29 每次 `ntbset` 后 USB3 链路曾降为 high-speed，需要重新插拔，原因没查。`ntbset` 里的 stop 后等 3 秒和最多 2 次自动重绑是缓解尝试；之后有一次输出为 super-speed 且没有重绑行，只有这一个样本，不能说明有效。
- `cpuwin` 使用固定的 `/tmp/a`、`/tmp/b`、`/tmp/na`、`/tmp/nb` 临时文件，两个采样同时运行会互相覆盖。
- 稳定性只测了两个配置：jumbo 下 NTB 63 KB / 63 KB 一夜、NTB_OUT 16 KB / NTB_IN 63 KB 约 60 小时；NTB 均为 16 KB 和 MTU 1500 没测；并发只测了下行 8 流和一路上行加一路下行同时跑，没有测上行多流。
- 10-29 夜里 dmesg 约 23:07 写满后，23:07 到 02:40 之间的 NTB 解析报错没有保留，第 5 到 8 轮是否有报错未知，全夜总次数无法统计；板端与 Windows 时钟没有单独核对，报错与相的对应有约一个采样的不确定。
- 板端监控脚本里的 `nth=` 累计计数是 10-30 新增，在 4.18 的约 60 小时里跑过：缓冲写满之前 `dmesg_hits` 等于 `nth` 的 2 倍（540 对 270、896 对 448），写满之后 `dmesg_hits` 在 911~922 波动、`nth` 继续累计；累计总数没有独立核对。
- 同时跑的相里下行 1.55 到 2.96，波动大，原因没查。
- USB2（4.17）只测了两个配置（NTB 16 KB / 63 KB，MTU 1500），每格 1 次，jumbo、RNDIS 没测（上行 4 流 11-02 补测，4.19）；用 configfs 限速的 high-speed 与 USB2 版 DTS 不完全等价（SS PHY 仍使能），没有验证两者吞吐一致。
- USB2 的 4 流下行：发送端汇总行比 receiver 的 20 秒长很多（32.49 秒、57.51 秒），一次链式运行报 `control socket has closed unexpectedly` 作废，原因都没查；Windows 的 `NlMtu`、LinkSpeed 没有贴回。
- `ntbset` 自己打印的速度不可信（3.9）；解绑后 `current_speed` 是否保留旧值没有验证。
