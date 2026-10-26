- 当前状态：NCM 已完成默认参数基线和 RPS 实验，正在继续调优；RNDIS 尚未测试
- 最近更新：2026-10-23

## 1. 当前结论

| 协议 | 方向 | 默认参数 | 开启 RPS | 说明 |
|---|---|---|---|---|
| NCM | 上行：主机 → 板端，单流 | 2.22~2.29 Gbit/s | 2.29 Gbit/s | RPS 无收益 |
| NCM | 上行：主机 → 板端，4 流 | 2.21~2.24 Gbit/s | 2.29 Gbit/s | RPS 无收益 |
| NCM | 下行：板端 → 主机，单流 | 1.76~1.91 Gbit/s | 1.86 Gbit/s | 受 iperf3 发送线程单核限制 |
| NCM | 下行：板端 → 主机，4 流 | 1.23~1.28 Gbit/s | 1.90~2.14 Gbit/s | RPS 有效，提升约 50~65% |
| RNDIS | 上行 / 下行 | 未测 | - | - |

默认参数一列为 2026-10-22、10-23 两轮测试的范围。

要点：

1. 默认配置下，USB 中断、NCM 封包/解包、收发软中断和 TCP 处理全部在 **CPU1** 上执行。上行和 4 流下行时 CPU1 空闲只有 0~2%，其余 15 个核大多空闲。
2. **4 流下行**的瓶颈已确认是 CPU1 上的收包软中断（ACK 处理）。开启 RPS 后吞吐提升约 50~65%，CPU1 空闲回到 40% 以上。
3. **单流下行**开启 RPS 后，瓶颈转到板端 iperf3 发送线程所在的核（sys 88%，idle 0%），即单个 TCP 发送进程的拷贝开销。
4. **上行**在 RPS 开/关、单流/4 流、qmult 5/80、IRQ 放在 CPU1/CPU4 的各种组合下，都稳定在 2.22~2.29 Gbit/s（约 19 万包/秒）。开启 RPS 后板端没有任何核跑满，USB 请求队列深度也已排除。上行限制很可能在主机侧，下一步验证。

## 2. 测试环境

| 项目 | 内容 |
|---|---|
| 板端 | AX8860 EVB（eMMC），16 × Cortex-A55，Linux 6.18.33，glibc，DDR 3200 |
| 固件 | 完整 SDK 固件 + USB3/I2C1 DTB；boot 分区开启 `CONFIG_USB_CONFIGFS_NCM=y`（RNDIS 原本已开启）；soc 分区包含 `/soc/scripts/usb-ncm.sh`、`/soc/scripts/usb-rndis.sh` |
| USB 链路 | UDC `2e1f0000.dwc3`，USB3 SuperSpeed（5 Gbps，Gen1） |
| Gadget | configfs，接口 `ncm0` / `rndis0`；NCM 默认参数为 qmult 5（10 个 USB 请求）、NTB 16 KB、MTU 1500 |
| 主机 | Windows 10，NCM 驱动 UsbNcm Host Device `10.0.19041.1`（`usbncm.inf`，Microsoft，手动选择） |
| 工具 | iperf3；板端为服务端 `192.168.77.2`，Windows 为客户端 |

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

### 3.2 吞吐测试

```powershell
iperf3 -c 192.168.77.2 -t 60 -O 3 -P 1
iperf3 -c 192.168.77.2 -t 60 -O 3 -P 1 -R
iperf3 -c 192.168.77.2 -t 60 -O 3 -P 4
iperf3 -c 192.168.77.2 -t 60 -O 3 -P 4 -R
```

不带 `-R` 为上行（Windows 发、板端收），带 `-R` 为下行（板端发、Windows 收）。

### 3.3 板端 CPU 采样

iperf3 服务端必须放在后台运行，同一个串口 shell 里才能同时采样。先定义函数：

```sh
cpuwin() {
  grep '^cpu[0-9]' /proc/stat > /tmp/a; grep ncm0 /proc/net/dev > /tmp/na
  sleep 20
  grep '^cpu[0-9]' /proc/stat > /tmp/b; grep ncm0 /proc/net/dev > /tmp/nb
  awk 'NR==FNR{for(i=2;i<=8;i++)a[$1,i]=$i;next}
       {t=0;for(i=2;i<=8;i++){d[i]=$i-a[$1,i];t+=d[i]}
        if(t>0)printf "%-6s usr%3d%% sys%3d%% irq%3d%% sirq%3d%% idle%3d%%\n",$1,
        100*(d[2]+d[3])/t,100*d[4]/t,100*d[7]/t,100*d[8]/t,100*(d[5]+d[6])/t}' /tmp/a /tmp/b
  cat /tmp/na /tmp/nb
}
```

Windows 端执行 `iperf3 -c 192.168.77.2 -t 40 ...`，大约 5 秒后在板端执行 `cpuwin`，让 20 秒采样窗口完整落在测试期间。ncm0 两行计数的字节差除以 20 秒，就是板端实际速率。

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

设置：`echo fffc > /sys/class/net/ncm0/queues/rx-0/rps_cpus`，把协议栈处理分到 CPU2~15。测完恢复为 `0`。速率来自板端 ncm0 计数，Windows 端汇总待补。

| | 上行 `-P 1` | 下行 `-P 4 -R` |
|---|---|---|
| 板端计数速率 | 2.29 Gbit/s（不变） | **2.14 Gbit/s（+63%）** |
| CPU1 | sys 62% + irq 8% + softirq 12%，idle 16% | sys 35% + irq 10% + softirq 11%，idle 42% |
| 其他核 | CPU12 softirq 57%（协议栈），CPU8/9/11/13 sys 最高 17% | CPU3/5/7/14 softirq 各约 36~43%（ACK 处理） |

### 4.3 2026-10-23 NCM 默认参数复测（未开 RPS）

本轮原计划在 RPS 下测试，但 `rps_cpus` 实际为 `0`，因此作为默认参数复测记录。速率来自板端 ncm0 计数（20 秒窗口）。

| 组 | 方向 | 板端速率 | CPU1 | 其他核 | Windows 总 CPU |
|---|---|---|---|---|---|
| `-P 1` | 上行 | 2.29 Gbit/s | sirq 51%，idle 0% | CPU2/3 sys 约 30%（iperf3） | 43% |
| `-P 4` | 上行 | 2.24 Gbit/s | sirq 51%，idle 0% | CPU2~5 sys 7~18% | 47% |
| `-P 1 -R` | 下行 | 1.76 Gbit/s | sirq 35%，idle 30% | CPU8/9 sys 约 25% | 51% |
| `-P 4 -R` | 下行 | 1.28 Gbit/s | sirq 80%，idle 1% | 基本全部空闲 | 52% |

- 与 10-22 默认基线一致，结果可复现。
- 下行单流时 CPU1 仍有 30% 空闲，单流下行在默认参数下也不完全由 CPU1 决定。

### 4.4 2026-10-23 NCM 开启 RPS（四组完整）

设置：`echo fffc > /sys/class/net/ncm0/queues/rx-0/rps_cpus`，已读回确认。速率来自板端 ncm0 计数（20 秒窗口）。主机为 18 个逻辑处理器。

| 组 | 方向 | 板端速率 | CPU1 | 其他核 | Windows 总 CPU |
|---|---|---|---|---|---|
| `-P 1` | 上行 | 2.29 Gbit/s | sys 63% + irq 8% + sirq 10%，idle 18% | CPU5 sirq 58%（RPS），CPU8 sys 42%（iperf3） | 46% |
| `-P 4` | 上行 | 2.29 Gbit/s | sys 61% + irq 8% + sirq 10%，idle 18% | CPU4/5/7 sirq 12~28%，CPU8 sys 18% | - |
| `-P 1 -R` | 下行 | 1.86 Gbit/s | sys 31% + irq 8% + sirq 7%，idle 52% | **CPU8 sys 88%，idle 0%**（iperf3 发送） | - |
| `-P 4 -R` | 下行 | 1.90 Gbit/s | sys 34% + irq 8% + sirq 9%，idle 47% | CPU9/10/12 sirq 32~46%（ACK 处理） | - |

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

## 5. 分析

1. **4 流下行**：主机回来的 ACK 原本全部由 CPU1 接收，TCP 发包和 NCM 封包拷贝也在 CPU1 的软中断里执行，导致 CPU1 满载。开启 RPS 后 ACK 处理分散到其他核，CPU1 空闲回到 40% 以上，吞吐提升约 50~65%。默认参数下 4 流比单流慢，也是这个原因：多条流的 ACK 处理全部压在 CPU1 上。
2. **单流下行**：开启 RPS 后瓶颈转移到板端 iperf3 发送线程（CPU8 sys 88%），即单个 TCP 发送进程从用户态拷贝数据的开销。这是测试工具的单线程上限，不是 NCM 驱动的上限；实际业务若为多连接，可参考 4 流结果。
3. **上行**：RPS 开/关、单流/4 流结果都在 2.22~2.29 Gbit/s，开启 RPS 后板端各核都有空闲。qmult 从 5 加到 80 后，bulk OUT 端点上挂着的请求从 10 个增加到 160 个，吞吐仍然不变，**已排除板端 CPU 和 USB 请求队列深度这两个原因**。剩下的候选都在主机侧：
   - Windows 发送侧 CPU：总占用 46%（18 个逻辑处理器），但总占用不能排除某一个核跑满；
   - Windows 端 iperf3 是 Cygwin 版（3.22），socket 经过一层模拟；
   - Win10 UsbNcm 驱动本身的限制，例如每次发送的 NTB 大小或处理开销。

## 6. 调优计划与进度

每次只改一个变量。

| # | 项目 | 方法 | 状态 |
|---|---|---|---|
| 1 | RPS | `rps_cpus=fffc` | 已测（10-22、10-23）：4 流下行 +50~65%，上行不变 |
| 2 | 补齐 RPS 数据 | RPS 下四组 | 已完成（10-23）；Windows 端 iperf3 汇总和每核占用仍待补 |
| 3 | 加深 USB 请求队列 | qmult 5 → 10 → 80（保持 RPS），验证上行 | 已测（10-23）：TRB 确认生效，上行不变，**已排除** |
| 3a | 主机侧定位上行瓶颈 | Windows 每核 CPU；原生 iperf3 替换 Cygwin 版；Linux 主机对比 | **下一步** |
| 4 | 中断与进程绑核 | dwc3 中断和 iperf3 分开到不同核 | 待做 |
| 5 | jumbo 帧 | 调大 `max_segment_size`，上限 8000，需要主机侧配合 | 待做 |
| 6 | NTB 缓冲 | 16 KB 写死在 `f_ncm.c`，加大需要改内核 | 待评估 |
| 7 | RNDIS 基线 | 按相同方法测试，与 NCM 对比 | 待做 |
| 8 | Win11 主机复测 | 排除 Win10 驱动的影响 | 视条件 |

修改 qmult 前需要先解绑并删除 function 链接，否则内核返回 EBUSY：

```sh
G=/sys/kernel/config/usb_gadget/usb_ncm
/soc/scripts/usb-ncm.sh stop
rm $G/configs/c.1/ncm.usb0
echo 20 > $G/functions/ncm.usb0/qmult
/soc/scripts/usb-ncm.sh start
```

RPS 只是运行时设置，板子重启后失效，每次测试前要重新设置。

## 7. 局限

- 目前只有单台主机、两轮测试（10-22、10-23），没有做多轮统计；同配置两轮间有约 10% 波动。
- 10-22 之后的速率来自板端 ncm0 计数，Windows 端 iperf3 汇总还没有核对。
- Windows CPU 目前只有总占用（18 个逻辑处理器），每个核的占用未采集，不能排除主机单核满载。
- 尚未单独测试 USB 链路的纯传输极限（例如 bulk loopback）。“链路仍有余量”是根据 CPU 已跑满推断出来的，没有直接测量。
- 板端脚本已在真板上完成 NCM 启动和枚举。重复 start/stop、RNDIS 启动及其 Windows 驱动绑定尚未验证。
