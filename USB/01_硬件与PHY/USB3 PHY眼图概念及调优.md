## 一、眼图判读标准（Mask 模板）

### 1.1 眼图组成

|组成|是什么|作用|
|---|---|---|
|波形轨迹带（Trace）|大量 UI 波形对齐叠加形成的轨迹|反映 TX 实际信号|
|Eye Mask|规范定义的禁止进入区域|判定最小眼高、眼宽|
|电压限制|最大差分输出范围|判定摆幅、过冲和下冲|
|Eye Height|垂直方向开口|反映电压裕量|
|Eye Width|水平方向开口|反映时间和抖动裕量|
|Crossover|P/N 翻转交叉区域|反映抖动、ISI 和差分对称性|
||||

USB3 Gen1 的基本参数：

```text
线路速率：5Gbps
编码方式：8b/10b
1 UI：200ps
信号：SSTX+ / SSTX-
```

USB3 的 UI 很小，因此参考时钟抖动、反射、通道损耗和 P/N 路径不对称都会明显影响眼图。

---

### 1.2 USB3 Gen1 使用什么 Mask

USB3 Gen1 应选择测试仪器或合规软件中的：

```text
SuperSpeed USB 5Gbps
USB 3.0 / USB 3.2 Gen1
Transmitter Eye Diagram Mask
```

不能选择：

```text
USB2 High-Speed 480Mbps Mask
USB 3.2 Gen2 10Gbps Mask
PCIe 5GT/s Mask
SATA 6Gbps Mask
```

USB3 Gen1 的眼图判定主要包括：

1. **Eye Opening Mask**  
    波形不能进入眼图中心禁区，用于判断最小眼高和最小眼宽。
    
2. **Differential Voltage Limit**  
    波形不能超过最大差分电压限制，用于判断 TX 摆幅、过冲和下冲。
    
3. **Jitter Limit**  
    需要结合时钟恢复和规定 BER 下的抖动结果判断，不能只看静态 Mask 截图。
    

示意如下：

```text
差分电压上限
────────────────────────────────

          /----------------\
         /                  \
        /                    \
       /      Eye Mask        \
      <        禁止区           >
       \                      /
        \                    /
         \                  /
          \----------------/

────────────────────────────────
差分电压下限
```

判定规则：

```text
波形冲出上下电压限制       -> Fail
波形进入中心 Eye Mask      -> Fail
眼高或眼宽低于要求         -> Fail
抖动超过对应项目限制       -> Fail
```

USB3 Mask 的横坐标以 UI 为单位，纵坐标以差分电压为单位：

```text
Gen1：1 UI = 200ps
Gen2：1 UI = 100ps
```

---

### 1.3 测试点与参考面

USB3 的 Eye Mask 必须和测试参考面配套使用。

常见测试点：

|测试点|位置|主要用途|
|---|---|---|
|TP1|TX 输出侧、近端参考面|检查 PHY/连接器发送质量|
|TP4|Compliance Channel 后的接收侧参考面|检查通道损耗后的接收眼图|
|Near-end|DUT 连接器附近|暴露过冲、幅度过大、阻抗失配|
|Far-end|经过规定通道或线缆后|暴露损耗、ISI、眼高/眼宽不足|

---

### 1.4 USB3 Mask 坐标与软件加载

USB3 Gen1 Mask 的正式顶点坐标、参考接收机、时钟恢复、滤波和抖动算法，应由以下内容共同决定：

- USB 3.x Electrical Specification
- USB-IF Electrical Compliance Test Specification
- 测试仪器对应版本的 USB3 Compliance Application
- 当前选择的 TP1、TP4、Host、Device 和通道模型

---

### 1.5 USB3 Compliance Pattern

USB3 PHY TX 测试不是使用 USB2 的 `test_packet`，而是进入 SuperSpeed Compliance Mode，发送 Compliance Pattern。

常见码型：

|Pattern|主要用途|
|---|---|
|CP0|加扰数据码型，常用于眼图和抖动测试|
|CP1|固定数据特征，辅助抖动和数据相关分析|
|CP2～CP8|用于不同电气参数、周期性码型或调试项目|

不同规范版本和仪器可能对具体测试项目选择不同 CP，正式测试以合规软件提示为准。

USB3 Compliance Mode 通常通过以下方式进入：

- USB3 Compliance Fixture
- Polling.Compliance 状态
- LFPS/Ping.LFPS 切换 Compliance Pattern
- 控制器或 PHY 专用测试寄存器
- 平台私有 debugfs、sysfs 或测试工具

USB3 没有所有平台通用的：

```bash
echo test_packet > .../testmode
```

该命令更常见于 USB2 Test Packet。USB3 应查看当前 DWC3、xHCI、PHY 驱动或芯片平台的测试接口。

---

### 1.6 近端与远端必须结合判断

|测试点|主要暴露|调整倾向|
|---|---|---|
|近端|摆幅过大、过冲、振铃、边沿过快|倾向“减”|
|远端|通道损耗、ISI、眼高/眼宽不足|倾向“加均衡”|
|RX 端|RX EQ、终端、CDR 和信号检测问题|调 RX 参数|

推荐顺序：

```text
先解决近端过冲、振铃和幅度超限
再解决远端眼高、眼宽和接收误码
```

近端已经过冲时继续增加 TX swing 或 TX de-emphasis，通常会使问题更严重。

---

## 二、违规现象 → 病因 → 药方总表

核心逻辑：先看波形在哪里违规，再决定调哪类寄存器。

|#|违规现象|本质|主要病因|第一药方|第二药方|
|---|---|---|---|---|---|
|1|冲出上/下电压限制|幅度过大或过冲|TX swing 大、边沿快、去加重强|`TXSWING`↓|`TXSLEW`放缓、`TXDEEMPH`↓|
|2|上下顶进 Eye Mask|眼高不足|TX 幅度低、通道损耗、均衡不足|`TXSWING`↑|远端调 `TXDEEMPH`、`RXEQ`|
|3|左右挤进 Eye Mask|眼宽不足|抖动、ISI、反射|查 `REFCLK/PLL`|调 `TXDEEMPH`、终端和通道|
|4|近端正常、远端闭眼|高频损耗|走线、连接器、线缆损耗|`TXDEEMPH`↑|`RXCTLE/RXEQ`↑|
|5|过冲、下冲明显|边沿过快或失配|TX slew、输出阻抗、通道不连续|`TXSLEW`放缓|调 `TXTERM/TXIMP`|
|6|周期性振铃|反射|阻抗失配、过孔、stub、连接器|调 `TXTERM/TXIMP`|`TXSLEW`放缓|
|7|波形发毛、轨迹变厚|噪声或抖动|电源、参考时钟、串扰|查电源和 refclk|查 PLL、SSC 和布局|
|8|Crossover 上下偏移|差分/共模不对称|P/N 驱动、偏置、布局不对称|`TXCROSSOVER/TXCM`|查 P/N 和 AC 电容|
|9|Crossover 左右扩散|抖动或延迟不对称|PLL、P/N skew、反射|查 `REFCLK/PLL`|调终端、均衡|
|10|跳变后幅度恢复慢|高频损耗或 EQ 不足|通道损耗、post-cursor 不足|`TXPOSTCURSOR`↑|`RXCTLE`↑|
|11|短线正常、长线失败|通道补偿不足|TX/RX EQ 太小|`TXDEEMPH`↑|`RXEQ`↑|
|12|长线正常、短线失败|过补偿|TX/RX EQ 太强|`TXDEEMPH`↓|`RXEQ`↓|
|13|眼图正常但误码高|RX/CDR 问题|RX EQ、CDR、SSC、电源噪声|调 `RXEQ/CDR`|查时钟和供电|
|14|无法进入 U0|链路训练问题|RX termination、LFPS、检测门限|查 `RXTERM/LFPS`|查 TX/RX 连接和 polarity|
|15|频繁进入 Recovery|接收裕量不足|RX EQ、CDR、通道损耗|调 `RXEQ/CDR`|查 refclk、SSC、通道|
|16|随机降为 USB2|SuperSpeed 链路失效|LFPS、RX detect、误码、供电|查链路状态|调检测门限和 RX EQ|

### 三个关键判断

#### 上下方向有两种相反的问题

```text
冲出外部电压限制 = 幅度或过冲太大 -> 减
顶进中心 Eye Mask = 眼高不足       -> 加
```

#### 去加重是双刃剑

```text
远端高频损耗、ISI -> 增大去加重
近端过冲、短通道过补偿 -> 减小去加重
```

#### 左右闭眼优先查抖动和 ISI

```text
眼宽不足 -> 先查 refclk、PLL、SSC、反射和通道
```

不要优先用 `TXSWING` 处理纯眼宽问题。

---

## 三、常见 PHY 寄存器详解

> 字段数值增大不一定代表物理量增大。例如某些 PHY 中 `TXTERM` 数值增大代表阻抗减小。修改前必须确认字段编码。

---

### 3.1 TX 主摆幅

#### `TXSWING` / `TXAMPLITUDE` / `TXVREF`

控制 USB3 TX 差分主摆幅。

主要影响：

- Eye Height
- 最大差分电压
- 远端接收裕量
- 过冲和 EMI

调节方向：

```text
近端、远端眼高都不足 -> ↑
近端冲出电压限制     -> ↓
过冲、下冲明显       -> 谨慎降低
```

注意：

- 近端正常、远端不足，不一定要增加 TX swing。
- 如果是通道高频损耗，优先调 TX de-emphasis。
- TX swing 过大可能增加反射和共模噪声。

---

### 3.2 TX 去加重与发送均衡

#### `TXDEEMPH` / `TXDEEMPHASIS` / `TXEQ`

控制发送端去加重强度，用于补偿通道高频损耗。

调节方向：

```text
远端眼高不足、ISI 明显 -> ↑
远端眼宽因 ISI 变窄   -> 适当 ↑
近端过冲、振铃        -> ↓
短通道过补偿          -> ↓
```

USB3 Gen1 常见去加重配置会以 dB 或离散档位表示，例如：

```text
No de-emphasis
约 -3.5dB
约 -6dB
```

具体档位以 PHY 定义为准。

#### `TXMAINCURSOR`

控制主游标幅度。

影响：

- 主脉冲幅度
- 整体 Eye Height
- 与 pre/post-cursor 的幅度分配

#### `TXPOSTCURSOR`

控制跳变后符号的补偿。

适用情况：

```text
远端跳变后收敛慢
后游标 ISI 明显
长通道高频损耗
```

#### `TXPRECURSOR`

控制跳变前符号的补偿，部分 USB3 PHY 支持。

适用情况：

```text
前游标 ISI 明显
接收端眼图左右不对称
通道模型需要前向补偿
```

注意：

- `pre-cursor`、`main-cursor`、`post-cursor` 通常相互影响。
- 增加 EQ 后可能需要重新调整主摆幅。
- 字段可能采用符号数或补码，不能直接按数值大小判断方向。

---

### 3.3 TX 边沿速率

#### `TXSLEW` / `TXRISETUNE` / `TXFALLTUNE`

控制 TX 上升沿和下降沿速度。

调节方向：

```text
过冲、下冲、振铃、EMI 大 -> 放缓
边沿过慢、交叉区过宽     -> 加快
```

边沿过快：

- 激发过孔、连接器和 stub 反射
- 过冲、下冲加重
- 串扰和 EMI 增大

边沿过慢：

- 眼宽减小
- Crossover 轨迹变厚
- 高频分量不足
- ISI 增加

如果 PHY 分别提供 rise/fall 配置，还需要检查两个方向是否对称。

---

### 3.4 TX 输出阻抗

#### `TXTERM` / `TXIMP` / `TXDRVIMP` / `TXRESCODE`

控制 TX 输出阻抗或源端终端。

适用问题：

```text
周期性振铃
反射分支明显
过冲后多次回摆
不同线缆下波形差异很大
```

调试方法：

1. 从 PHY 推荐默认值开始。
2. 向相邻档位单步调整。
3. 比较振铃幅度和衰减时间。
4. 同时观察近端和远端。
5. 必要时结合 TDR 或 S 参数确认。

阻抗问题不能只靠降低 TX swing 掩盖。

---

### 3.5 TX 共模与 Crossover

#### `TXCROSSOVER` / `TXCM` / `TXCMV` / `TXVCOM`

控制发送共模电压或交叉点。

适用问题：

```text
正、负摆幅不对称
Crossover 上下偏移
眼图上下不对称
共模噪声明显
```

同时检查：

- P/N 走线长度与结构
- P/N 过孔数量
- AC 耦合电容容值和焊盘
- 连接器 P/N 结构
- PHY 模拟电源和偏置
- 差分到共模转换

如果 P/N 板级结构不对称，寄存器只能做有限补偿。

---

### 3.6 RX CTLE 与均衡

#### `RXEQ` / `RXCTLE` / `RXHFEQ`

控制 RX 高频增益，补偿通道高频损耗。

调节方向：

```text
长线、高损耗、远端高频塌陷 -> ↑
短线、高频噪声被放大       -> ↓
```

均衡不足：

- 长线误码
- 频繁 Recovery
- CDR 锁定不稳定
- 短线正常、长线失败

均衡过强：

- 高频噪声放大
- 短线过补偿
- CDR 输入抖动增大
- 短线反而失败

#### `RXAFEGAIN` / `RXVGA`

控制 RX 模拟前端或可变增益放大器。

适用问题：

```text
RX 输入幅度偏低
长通道接收灵敏度不足
```

增益过高也可能放大噪声。

---

### 3.7 RX DFE 与自适应均衡

#### `RXDFE`

使用判决反馈方式消除后游标 ISI。

适用问题：

- 高损耗通道
- 后游标拖尾明显
- CTLE 单独调节无法满足 BER

#### `RXADAPT` / `RXEQADAPT`

控制 RX 自动均衡。

需要确认：

- 自适应是否启动
- 是否成功收敛
- 软件固定值是否覆盖自适应结果
- Recovery 后是否重新训练
- 不同线缆下收敛值是否合理

调试时可以分别测试：

```text
固定 RX EQ
自动 RX EQ
自动 EQ 收敛后锁定
```

---

### 3.8 RX 终端与接收检测

#### `RXTERM` / `RXTERMCTRL` / `RXRESCODE`

控制 RX 输入终端。

可能影响：

- 输入反射
- 接收幅度
- 共模转换
- 链路训练
- Receiver Detection

典型问题：

```text
TX 近端正常，RX 端振铃
无法稳定识别对端
链路训练失败
```

#### `RXDET` / `RXDETECT`

控制或校准 Receiver Detection。

适用问题：

- 检测不到对端终端
- 误判对端存在
- SuperSpeed 初始化失败
- 端口反复进行接收检测

---

### 3.9 RX Squelch、LOS 与门限

#### `RXSQUELCH` / `RXLOS` / `RXSIGDET`

控制有效信号检测门限。

门限过低：

```text
噪声被识别为有效信号
空闲时误触发
链路状态异常切换
```

门限过高：

```text
弱信号检测不到
长线频繁掉链
无法稳定进入 U0
```

该类寄存器通常不改变 TX 眼图，但会影响功能和链路稳定性。

---

### 3.10 LFPS 检测

#### `LFPSDET` / `LFPSVTH` / `LFPSWIDTH` / `LFPSRX`

控制 LFPS 幅度、周期或脉宽检测。

适用问题：

- 无法进入 Polling
- 无法进入 U0
- U1/U2/U3 切换异常
- Resume/Wakeup 异常
- 频繁 Recovery
- 高速 TX 眼图正常但链路仍失败

LFPS 是低频周期信号，必须与 5Gbps 数据眼图分开测试和判断。

---

### 3.11 参考时钟与 PLL

常见字段：

```text
REFCLKSEL
FSEL
PLLFREQ
PLLBW
PLLCP
PLLLOCK
```

眼宽不足或轨迹左右变厚时，优先检查：

- 参考时钟频率是否正确
- REFCLK 来源是否选择正确
- 参考时钟抖动
- PLL 是否稳定锁定
- PLL 带宽和 charge pump 配置
- 模拟电源纹波

典型现象：

```text
所有码型都出现类似水平扩散
眼高正常但眼宽不足
不同温度下抖动变化明显
```

此类问题不要优先调 TX swing。

---

### 3.12 SSC 与 CDR

#### `SSCEN` / `SSCRANGE` / `SSCMOD`

控制扩频时钟。

配置异常可能导致：

- 低频周期性抖动
- CDR 跟踪异常
- 对端兼容性问题
- 长时间压力测试掉链

#### `CDRBW` / `CDRGAIN` / `CDRLOCK`

控制 RX 时钟数据恢复。

CDR 带宽过低：

```text
无法跟踪低频频偏或 SSC
锁定慢
频繁失锁
```

CDR 带宽过高：

```text
高频抖动进入恢复时钟
噪声跟踪过多
BER 变差
```

---

## 四、不同情形下的寄存器调整方法

### 4.1 近端冲出上下电压限制

现象：

```text
TX 输出幅度过大
边沿处过冲或下冲
波形冲出外部电压边界
```

调整顺序：

```text
1. TXSLEW         放缓边沿
2. TXDEEMPH       减小去加重
3. TXSWING        降低主摆幅
4. TXTERM/TXIMP   优化输出阻抗
```

判断：

- 仅边沿瞬间越界：优先调 `TXSLEW` 和终端。
- 整个平台都过高：调 `TXSWING`。
- 只有跳变符号过高：优先减小 `TXDEEMPH`。

---

### 4.2 上下顶进 Eye Mask

现象：

```text
Eye Height 不足
上下波形进入中心禁区
```

近端和远端需要分开处理。

近端眼高不足：

```text
1. TXSWING ↑
2. 检查 PHY 模拟电源
3. 检查 TXTERM
```

近端正常、远端眼高不足：

```text
1. TXDEEMPH / TXPOSTCURSOR ↑
2. RXCTLE / RXEQ ↑
3. 检查通道插入损耗
4. 检查连接器、过孔和线缆
```

---

### 4.3 左右挤进 Eye Mask

现象：

```text
Eye Width 不足
Crossover 区域左右变厚
眼高可能仍然正常
```

调整顺序：

```text
1. 检查 REFCLK 抖动
2. 检查 PLL 和 SSC
3. 检查 PHY 电源纹波
4. 检查 TXTERM 和通道反射
5. 调整 TXDEEMPH，减小 ISI
6. 检查 CDR 配置
```

不要优先调整：

```text
TXSWING
```

---

### 4.4 近端正常、远端闭眼

现象：

```text
TP1 或连接器附近通过
经过通道后眼高、眼宽明显下降
```

调整顺序：

```text
1. TXDEEMPH ↑
2. TXPOSTCURSOR ↑
3. RXCTLE / RXEQ ↑
4. RXDFE / RXADAPT 优化
5. 检查 PCB、连接器和线缆损耗
```

如果远端主要表现为振铃而不是平滑闭合，应先处理阻抗，不能直接增加均衡。

---

### 4.5 过冲、下冲或周期性振铃

现象：

```text
跳变后出现一次或多次回摆
振铃具有明显周期
```

调整顺序：

```text
1. TXTERM / TXIMP
2. TXSLEW 放缓
3. TXDEEMPH 减小
4. 检查过孔、stub、连接器和 AC 电容焊盘
```

如果振铃周期固定，通常更接近板级反射问题。

---

### 4.6 波形发毛、轨迹整体变厚

现象：

```text
高低电平和边沿都不稳定
轨迹没有单一明显反射周期
```

优先检查：

```text
PHY 模拟电源
PLL 电源
参考时钟
邻线串扰
地平面完整性
示波器探头和夹具
```

可能涉及：

```text
PLLBW
PLLCP
SSC
TXSLEW
TXSWING
```

不要把电源噪声问题单纯当作 TX 幅度问题。

---

### 4.7 Crossover 上下偏移

现象：

```text
眼图上下不对称
正负摆幅不同
交叉点偏离中心电压
```

调整：

```text
1. TXCROSSOVER / TXCM
2. 检查 TXSWING 和 TXTERM
3. 检查 P/N 物理对称性
4. 检查 AC 耦合电容
```

---

### 4.8 Crossover 左右扩散或偏移

现象：

```text
交叉区域水平扩散
上升沿和下降沿位置不一致
```

优先检查：

```text
REFCLK
PLL
P/N skew
TX rise/fall 配置
通道反射
```

可能调整：

```text
TXRISETUNE
TXFALLTUNE
TXDEEMPH
TXTERM
CDRBW
```

---

### 4.9 短线正常、长线失败

调整顺序：

```text
1. TXDEEMPH ↑
2. TXPOSTCURSOR ↑
3. RXCTLE / RXEQ ↑
4. 开启或优化 RXADAPT
5. 检查长通道插入损耗
```

避免同时把所有参数调到最大，否则短通道可能过补偿。

---

### 4.10 长线正常、短线失败

常见原因：

```text
TX de-emphasis 过强
RX CTLE 过强
RX gain 过高
CDR 配置不适合短通道
```

调整顺序：

```text
1. TXDEEMPH ↓
2. RXCTLE / RXEQ ↓
3. RXAFEGAIN ↓
4. 检查自动均衡是否错误收敛
```

---

### 4.11 眼图通过但频繁 Recovery

重点检查：

```text
RXEQ / RXCTLE
RXDFE / RXADAPT
CDRBW / CDRLOCK
SSC
RXTERM
RXSQUELCH
LFPSDET
```

同时检查：

- 两个传输方向是否都正常
- Host TX 通过不代表 Device TX 通过
- 眼图通过不代表接收容限通过
- 眼图通过不代表 LFPS 正常
- 是否只有特定线缆或对端失败

---

### 4.12 无法进入 SuperSpeed U0

检查顺序：

```text
1. SSTX/SSRX 是否连接正确
2. P/N polarity 是否正确或已配置翻转
3. AC 耦合电容位置是否正确
4. RX termination 是否开启
5. Receiver Detection 是否成功
6. LFPS 检测是否正常
7. REFCLK 和 PLL 是否锁定
8. RX squelch 是否过高
9. TX swing 是否过低
```

相关寄存器：

```text
RXTERM
RXDET
LFPSDET
LFPSVTH
RXSQUELCH
REFCLKSEL
PLLLOCK
TXSWING
```

---