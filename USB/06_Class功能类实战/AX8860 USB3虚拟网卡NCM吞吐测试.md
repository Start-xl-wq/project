- 当前状态：NCM（TCP，Windows 10 主机）已测 RPS、qmult、jumbo（MTU 7986）、NTB_OUT 和 NTB_IN（均为 16 KB 到 64512）的多种组合，各配置的实测带宽见第 1 节汇总表，未测的格子标“未测”。jumbo 下上行最高 3.20、下行 1 流最高 3.64、下行 4 流最高 3.19 Gbit/s，三个最高值出现在不同配置。稳定性长跑、并发、其他主机、RNDIS 尚未测试
- 最近更新：2026-10-29

## 1. 当前结论

以下均为 TCP，单位 Gbit/s，数值取 Windows 端 iperf3 汇总行（receiver），主机为 Windows 10，第 2 行例外（见表后说明）。同一格里的多个数是多次测量。RPS 指板端 `rx-0/rps_cpus`，开为 `fffc`，运行时设置，重启丢失。NTB_OUT 指设备上报给主机的 `dwNtbOutMaxSize`（上行方向），NTB_IN 指 `dwNtbInMaxSize`（下行方向），10-28 及之前固定为 16 KB；jumbo 指 `max_segment_size` 8000 加 MTU 7986。表中“未测”表示没有测量。

| # | MTU | NTB_OUT | NTB_IN | RPS | 上行 1 流 | 上行 4 流 | 下行 1 流 | 下行 4 流 | 来源 |
|---|---|---|---|---|---|---|---|---|---|
| 1 | 1500 | 16 KB | 16 KB | 关 | 2.16 到 2.22 | 2.10 到 2.20 | 1.89 到 1.91 | 1.30 | 10-27，4.6 |
| 2 | 1500 | 16 KB | 16 KB | 开 | 2.29 | 2.29 | 1.86 | 1.90 到 2.14 | 10-22、10-23，板端计数，4.2、4.4 |
| 3 | 7986 | 16 KB | 16 KB | 关 | 2.44 | 未测 | 未测 | 2.47 | 10-28，4.9 |
| 4 | 7986 | 16 KB | 16 KB | 开 | 2.40 | 未测 | 未测 | 3.16 / 3.19 / 3.16 / 3.15 | 10-28，4.9；10-29，4.13 |
| 5 | 7986 | 32 KB | 16 KB | 开 | 2.89 / 2.89 | 2.88 | 3.09 | 3.12 | 10-28，4.10 |
| 6 | 7986 | 64512 | 16 KB | 开 | 3.20 / 3.20 / 3.20 | 3.19 / 3.20 | 3.13 / 3.15 | 2.94 / 2.92 / 2.95 / 2.96 / 2.94 | 10-28，4.11；10-29，4.13、4.14 |
| 7 | 1500 | 64512 | 16 KB | 开 | 2.68 | 2.68 | 1.86 | 2.08 | 10-28，4.12 |
| 8 | 7986 | 43008 | 16 KB | 开 | 2.98 | 未测 | 未测 | 3.01 | 10-29，4.13 |
| 9 | 7986 | 49152 | 16 KB | 开 | 3.11 | 未测 | 未测 | 2.96 | 10-29，4.13 |
| 10 | 7986 | 64512 | 32 KB | 开 | 3.20 | 未测 | 3.42 / 3.48 | 2.67 / 2.62 | 10-29，4.14 |
| 11 | 7986 | 64512 | 49152 | 开 | 未测 | 未测 | 3.58 | 2.86 | 10-29，4.14 |
| 12 | 7986 | 64512 | 64512 | 开 | 3.20 | 未测 | 3.64 | 2.94 | 10-29，4.14 |
| 13 | 1500 | 64512 | 64512 | 开 | 2.68 | 未测 | 1.97 | 2.22 | 10-29，4.15 |
| 14 | 1500 | 16 KB | 16 KB | 开 | 2.22 | 未测 | 1.88 | 2.06 | 10-29，4.15（Windows 汇总值，与第 2 行同配置） |

说明：

- 第 2 行的数值来自板端 `ncm0` 计数，不是 Windows 汇总值。10-28 同口径对比，MTU 1500 下板端计数比 Windows 汇总值高：上行约 2.7% 到 3.0%，下行约 4.8% 到 5.1%；jumbo 下相差不到 2%。所以第 2 行不能与其他行直接相减。
- 第 1 行和第 2 行不是同一天、同一固件，线长也不同。
- 标“未测”的格子没有数据。10-28 同日有几条被中途打断的单次数据，RPS 状态未记录，没有放进表。
- 第 5 到 14 行没有测 RPS 关。
- 10-29 的数据（第 4、6 行里的 10-29 值，以及第 8 到 14 行）的 NTB_OUT、NTB_IN 由板端 `ntbset` 设置；NTB_IN 在 10-28 及之前固定为 16 KB。第 14 行与第 2 行同配置：第 2 行是板端计数，第 14 行是 Windows 汇总值，口径与其他行一致。
- 每格 1 到 5 次测量，没有做多轮统计；10-23 同配置两次曾相差约 11%。
- NCM UDP 见 4.7（仅供参考），RNDIS 未测。

要点：

1. 默认配置（MTU 1500，NTB_OUT 16 KB）下，USB 中断、NCM 封包/解包、收发软中断和 TCP 处理都在 **CPU1** 上执行。上行和 4 流下行时 CPU1 空闲只有 0 到 2%，其余核大多空闲（10-22、10-23 采样）。
2. **4 流下行**的瓶颈是 CPU1 上的收包软中断。开启 RPS 后，MTU 1500 下从 1.23 到 1.30 提升到 1.90 到 2.14（后者为板端计数），jumbo 加 16 KB 下从 2.47 提升到 3.16 / 3.19 / 3.16。RPS 对上行没有作用。
3. **单流下行**的限制在板端 iperf3 发送线程所在的核（MTU 1500 下 cpu8 sys 88%、idle 0%；jumbo 加 64512 下 cpu9 sys 90%、idle 0%）。
4. **jumbo**：NTB_OUT 固定为 64512 时，MTU 由 1500 改为 7986，上行 1 流 2.68 变 3.20，上行 4 流 2.68 变 3.19 到 3.20，下行 1 流 1.86 变 3.13，下行 4 流 2.08 变 2.92 到 2.95。NTB_OUT 为 16 KB 时，上行包速率由约 19 万降到约 3.8 万包/秒（5 倍），吞吐只涨约 10%（2.16 到 2.22 变 2.40 到 2.44），说明上行不是每包开销限制。
5. **NTB_OUT** 加大，上行随之上升：jumbo 下 16 KB / 32 KB / 43008 / 49152 / 64512 为 2.40 / 2.89 / 2.98 / 3.11 / 3.20；MTU 1500 下 64512 为 2.68（16 KB 为 2.16 到 2.22），不需要主机改 MTU。下行单流不变。**jumbo 下下行 4 流随 NTB_OUT 增大而下降后趋平**：同样五档依次为 3.15 到 3.19 / 3.12 / 3.01 / 2.96 / 2.92 到 2.96，同时板端 cpu1 idle 为 10% 到 13% / 11% / 6% / 5% / 3% 到 5%。10-29 在同一次开机内只改 `ntb_out_size`，16384 为 3.15、64512 为 2.96（约 -6.0%），确认下降由 NTB_OUT 引起（4.13）。原因没有定位。
6. **NTB_IN**（10-29，jumbo，NTB_OUT 固定 64512）加大，下行 1 流随之上升：16 KB / 32 KB / 49152 / 64512 为 3.15 / 3.42 到 3.48 / 3.58 / 3.64，上行 1 流不变（3.20）。下行 4 流不单调：2.94 / 2.67 与 2.62 / 2.86 / 2.94，32 KB 偏低并复现。板端每个下行 tx 计数的平均字节数约等于 NTB_IN（下行 4 流：16,054 / 32,070 / 48,086 / 64,099 B），说明 Windows 10 采纳了设备上报的 IN 大小（4.14）。
7. 各配置的取舍（Windows 10，实测）：jumbo 加 NTB_OUT 64512 上行最高（3.20），下行 4 流为 2.92 到 2.96；jumbo 加 NTB_OUT 16 KB 下行 4 流最高（3.15 到 3.19），上行为 2.40；加大 NTB_IN 提高下行 1 流，不影响上行；MTU 1500 下增大 NTB 同样有提升（上行 2.22 到 2.68，下行 1 流 1.88 到 1.97，下行 4 流 2.06 到 2.22），绝对值低于 jumbo。上行和下行哪个更重要取决于业务，汇总表按配置给出实测值，不预设取舍。
8. 主机只测了 Windows 10 的 UsbNcm。Linux `cdc_ncm`、Win11、macOS 没测，它们是否跟随设备上报的 NTB 大小、是否自动调整 MTU，都没验证。

## 2. 测试环境

| 项目 | 内容 |
|---|---|
| 板端 | AX8860 EVB（eMMC），16 × Cortex-A55，Linux 6.18.33，glibc，DDR 3200 |
| 固件 | 完整 SDK 固件 + USB3/I2C1 DTB；boot 分区开启 `CONFIG_USB_CONFIGFS_NCM=y`（RNDIS 原本已开启）；soc 分区包含 `/soc/scripts/usb-ncm.sh`、`/soc/scripts/usb-rndis.sh` |
| USB 链路 | UDC `2e1f0000.dwc3`，USB3 SuperSpeed（5 Gbps，Gen1） |
| Gadget | configfs，接口 `ncm0` / `rndis0`；NCM 默认参数为 qmult 5（10 个 USB 请求）、NTB IN/OUT 各 16 KB、`max_segment_size` 1514、MTU 1500；10-28、10-29 的改动见 3.2、3.3 |
| 主机 | Windows 10（build 19044），NCM 驱动 UsbNcm Host Device `10.0.19041.1`（`usbncm.inf`，Microsoft，手动选择）；Intel USB 3.20 可扩展主机控制器（`DEV_7EC0`），NCM 设备直接挂在其根集线器下；18 个逻辑处理器；网卡名“以太网 5”，地址 192.168.77.3 |
| 工具 | iperf3；板端 3.20（服务端 `192.168.77.2`），Windows 为 Cygwin 版 3.22（客户端） |
| 线缆 | 10-22、10-23 线长未记录；10-27 使用 0.25 m 线；10-28、10-29 未记录 |

测试固件：

- 10-22 的基线是完整 SDK 固件加 USB3/I2C1 DTB；之后的测试只替换 boot 分区（`boot_signed.bin`）。
- 引导链（SPL、FDL1、FDL2、U-Boot）使用 `DDR_FREQ=3200`；U-Boot 的 pinmux 把 GPIO0_A10/A11 设为 I2C1（SGM7220 使用）。
- 内核和 DTB：EVB 构建，USB3 使能，保留 `ref-pad-clk-en`，SGM7220 方向不取反。
- 内核改动只涉及 NCM function：10-28 把 `f_ncm.c` 里的 `NTB_OUT_SIZE` 常量依次改为 16384、32768、64512 并重编；10-29 起 `ntb_out_size`、`ntb_in_size` 改成 configfs 属性（见 3.3），先后用到只带 `ntb_out_size` 的构建和再加 `ntb_in_size` 的构建。
- 构建后核对：`f_ncm.o` 和 `Image` 里 `dwNtbOutMaxSize` 的字节图案与目标值一致（16384、32768、64512 互斥命中 1 次）；带 `ntb_in_size` 的构建里 `f_ncm.o` 和 `Image` 有该属性字符串，ncm 相关无编译警告；板上读回的 `ntb_out_size`、`ntb_in_size` 与 `ntbset` 设置一致。
- 上述内核改动只在本地，没有提交。

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

链路参考：板子 host 模式下接高速 U 盘，裸读 430 MB/s（约 3.44 Gbit/s，实测）。该测试只覆盖板端接收方向。

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

条件：`NTB_OUT_SIZE=32768`，`rps_cpus=fffc`，MTU 7986，`max_segment_size=8000`，super-speed（每个 `.cpu` 第二行回显确认）；`-t 20 -O 3`；手动测，每组 1 次。

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

条件：`NTB_OUT_SIZE=64512`，`rps_cpus=fffc`，MTU 7986，`max_segment_size=8000`；`/proc/version` 构建时间 10-28 16:02:06；`-t 20 -O 3`；手动测。烧录后第一次读回 `max_segment_size` 是 1514，重设为 8000 并重新枚举后再测。

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

条件：同一次开机，接在 4.11 之后，没有重新烧录；板端 `max_segment_size` 改回 1514、`ncm0` MTU 1500，`rps_cpus=fffc`；每个 `.cpu` 第二行回显 `mtu=1500 max_segment_size=1514`；Windows 的 `NlMtu` 没有回显，板端上行每个包 1,500 B，说明主机按 1500 发送；`-t 20 -O 3`；手动测，每组 1 次。

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

条件：固件 `boot_signed_ntbcfg.bin`；每组前用 `ntbset` 切换 `ntb_out_size`；`max_segment_size=8000`、MTU 7986，`rps_cpus=fffc`，super-speed（每个 `.cpu` 第二行回显）；`-t 20 -O 3`；手动测，每格 1 次；同一次开机，板端 uptime 606 到 1865 s。

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

条件：固件 `boot_signed_ntbinout.bin`；每组前用 `ntbset 64512 8000 7986 <ntb_in>` 切换；`rps_cpus=fffc`，super-speed；`-t 20 -O 3`；手动测，每格 1 到 2 次；与 4.15 同一次开机，板端 uptime 765 到 6915 s。

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

## 5. 分析

1. **4 流下行（MTU 1500）**：主机回来的 ACK 原本全部由 CPU1 接收，TCP 发包和 NCM 封包拷贝也在 CPU1 的软中断里执行，4 流时 CPU1 softirq 77% 到 80%、idle 1% 到 2%（4.1、4.3），单流时 softirq 35%、idle 30%。开启 RPS 后 ACK 处理分散到其他核，CPU1 idle 回到 42% 到 47%（4.2、4.4），吞吐提升约 50% 到 65%。所以默认参数下 4 流比单流慢，是所有流的收包处理挤在 CPU1 上。jumbo 加 16 KB 下同样如此：RPS 关 2.47、CPU1 idle 1%，RPS 开 3.16 到 3.19、CPU1 idle 13%（4.9）。
2. **单流下行**：限制在板端 iperf3 发送线程所在的核（4.4 的 CPU8、4.11 的 cpu9，sys 88% 到 90%、idle 0%），即单个 TCP 发送进程从用户态拷贝数据的开销。这是测试工具的单线程上限，不是 NCM 驱动的上限；实际业务若为多连接，可参考 4 流结果。
3. **上行（MTU 1500，NTB_OUT 16 KB）**：RPS 开/关、单流/4 流、qmult 5/80、IRQ 放在 CPU1/CPU4 的各种组合下，都稳定在 2.22 到 2.29 Gbit/s（约 19 万包/秒），板端没有核跑满，**已排除板端 CPU 和 USB 请求队列深度**。jumbo 使包速率降到 5 分之 1，吞吐只涨约 10%，**也排除每包开销是主要限制**。
4. **NTB_OUT 对上行**：加大后上行持续上升（jumbo 下 2.40 到 2.44、2.88 到 2.89、3.19 到 3.20；MTU 1500 下 2.16 到 2.22、2.68），Windows 10 的 UsbNcm 跟随了设备上报的值。jumbo 下上行时板端 cpu1 idle 随 NTB_OUT 增大而增大（16 KB 为 34%，32 KB 为 40% 到 42%，64512 为 45% 到 46%）。MTU 1500 加 64512 时 cpu1 idle 只有 11% 到 13%（约 23 万包/秒），是候选瓶颈，没到 100%，也没有 `perf` 可以拆开，未验证。
5. **NTB_OUT 对 jumbo 下的下行 4 流**：随 NTB_OUT 增大而下降后趋平（16 KB、32 KB、43008、49152、64512 依次为 3.15 到 3.19、3.12、3.01、2.96、2.92 到 2.96），同时 cpu1 idle 为 10% 到 13%、11%、6%、5%、3% 到 5%，64512 时 cpu1 接近跑满；10-29 同一次开机内只改 `ntb_out_size` 也得到 3.15 对 2.96（4.13）。下行 1 流不受影响。下降原因没有验证，有一条候选线索（代码和配置事实加推断）：
   - 下行时板端 rx 方向只收主机回来的 ACK（约 2.2 万个/秒，每个 40 B），但每个 rx 请求的缓冲长度是 `fixed_out_len`，也就是 `NTB_OUT_SIZE`（`u_ether.c` 的 `rx_submit`）；
   - gadget 层对每个请求按 `req->length` 做 `dma_map_single` 和 `dma_unmap_single`（`udc/core.c`）；
   - `axhelix.dtsi` 里 `dma-noncoherent` 只出现在 GIC 和 ITS 节点，axera dts 目录里没有 `dma-coherent`，没有确认 dwc3 的实际一致性；
   - 若不一致，每个 ACK 完成时的缓存维护量会随缓冲长度增大。32 KB 到 64512，下行 1 流和 4 流的 cpu1 忙度都多了约 7 个点，量级与此相符，但这不是证明。
6. **UDP**：iperf3 单流 UDP 两端都上不去（PC 约 1.5，板子约 1.35 Gbit/s），测不出链路能力（4.7）。
7. **主机侧**：Windows 10 的 UsbNcm 跟随了设备上报的 NTB 大小（上行随 NTB_OUT 上升）；在板端 `max_segment_size` 改成 8000 并重新枚举后 `NlMtu` 变成 7986。Windows 每核 CPU 在 10-27 的 16 KB 测试中最忙核峰值 78% 到 85%，未见持续满载；NTB_OUT 加大后没有重新采。Linux `cdc_ncm` 的行为没测（内核头文件里 TX/RX 的 NTB 最大值是 65536，默认 16384，是否自动使用设备上报值没有验证）。
8. **NTB_IN（jumbo，NTB_OUT 64512，4.14）**：下行 1 流随 NTB_IN 上升（3.15 到 3.64），上行 1 流不变（3.20），下行 4 流不单调（32768 偏低）。下行 1 流的发送线程所在核 sys 合计在各档都约 90%，吞吐随 NTB_IN 增大而上升，说明同样占用下每字节的开销在下降；原因没有验证。板端每个 tx 计数的平均字节数约等于 NTB_IN，说明 Windows 10 采纳了设备上报的 IN 大小。下行 4 流 CPU1 一直接近跑满，tx 计数速率降到约四分之一时 CPU1 的占用没有随之下降，只是 sys 与 sirq 的比例变了，原因没有定位。
9. **MTU 1500 下增大 NTB（4.15）**：NTB_OUT 与 NTB_IN 同为 64512 对默认 16 KB，上行 1 流约 +20.7%，下行 1 流约 +4.8%，下行 4 流约 +7.8%；绝对值低于 jumbo。下行每个 tx 计数平均约 47 KB，与 `TX_MAX_NUM_DPE`=32 的上限相符（推断）。上行 CPU1 忙度仍在 82% 到 89%，没有 `perf`，未验证。
10. **大 NTB 的内存分配**：`ncm_wrap_ntb` 每个新 NTB 按 `ntb_in_size` 整块 `alloc_skb(GFP_ATOMIC)`，NTB_IN 64512 时是 64 KB 级的连续分配，失败会计入 `tx_dropped`。10-29 的各组（每组约 23 秒）板端发送方向 drop 列都是 0；长时间运行、内存碎片化时是否会失败，没有测。

## 6. 调优计划与进度

每次只改一个变量。

| # | 项目 | 方法 | 状态 |
|---|---|---|---|
| 1 | RPS | `rps_cpus=fffc` | 已测（10-22、10-23、10-28）：多流下行有效，上行无作用 |
| 2 | 加深 USB 请求队列 | qmult 5 → 10 → 80（保持 RPS），验证上行 | 已测（10-23）：TRB 确认生效，上行不变，**已排除** |
| 3 | 中断与进程绑核 | dwc3 中断和 iperf3 分开到不同核 | 部分完成（10-23）：qmult 实验中 dwc3 中断绑 CPU4，上行不变；iperf3 绑核未做 |
| 4 | jumbo | `max_segment_size` 8000，MTU 7986，需要主机侧配合 | 已测（10-28）：见 4.9、4.12 |
| 5 | NTB_OUT | 10-28 改 `f_ncm.c` 的 `NTB_OUT_SIZE` 重编烧录；10-29 起用 configfs 属性 `ntb_out_size` | 已测：16 KB / 32 KB / 43008 / 49152 / 64512（jumbo），MTU 1500 下 16 KB / 64512；改动只在本地 |
| 6 | 补齐配置矩阵 | 汇总表里标“未测”的格子（上行 4 流、部分下行 1 流）；第 5 到 14 行的 RPS 关 | 部分完成（10-29：MTU 1500 同日对照、NTB_OUT 中间值）；上行 4 流和 RPS 关的格子仍未测 |
| 7 | 其他主机 | Win11、Linux `cdc_ncm`、macOS；原生 iperf3 替换 Cygwin 版；Windows 每核 CPU 在 NTB_OUT 加大后重采 | 待做；10-27 用开发板当 host 未成功（SS 没起来） |
| 8 | RNDIS 基线 | 按相同方法测试，与 NCM 对比 | 待做 |
| 9 | 纯 bulk 链路上限 | 板端 SourceSink（`CONFIG_USB_F_SS_LB`）加主机侧测试程序，或换 Linux 主机用 testusb | 待做 |
| 10 | UDP | iperf3 UDP 受工具限制 | 暂停，不作为调优方向 |
| 11 | NTB_IN | configfs 属性 `ntb_in_size`，`ntbset` 切换 | 已测（10-29）：jumbo 下 16 KB / 32 KB / 49152 / 64512，MTU 1500 下 16 KB / 64512；改动只在本地 |
| 12 | 稳定性长跑 | 大 NTB 配置各跑 10 分钟以上，看 `ncm0` 的 drop、dmesg、`/proc/buddyinfo` | 待做 |
| 13 | 并发 | 上下行同时跑；流数超过 4 | 待做 |
| 14 | cpu1 软中断拆解 | 取下行 4 流和 MTU 1500 上行时 `/proc/softirqs`、`/proc/interrupts` 的差值 | 待做 |

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
- 汇总表第 2 行是板端 `ncm0` 计数，与其他行的 Windows 汇总值口径不同（第 14 行是同配置的 Windows 值）；标“未测”的格子（多数是上行 4 流和部分下行 1 流）没有数据；第 5 到 14 行没有测 RPS 关。
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
- 没有做长时间稳定性测试，也没有测上下行同时跑和 4 流以上的并发。
