# 状态
## Detect 
• Quiet
• Active
## Polling
• Active
• Configuration
• Compliance
## Configuration
• Linkwidth.Start
• Linkwidth.Accept
• Lanenum.wait
• Complete
• Idle
## Recovery
• RcvrLock
• Speed
• Equalization(Gen3)
## L0
# #过程
## 上电->Gen1
### 1. Detect
目的：用于确认对端是否存在。
方法： 在lane上做了接收端终端检测，通过特定的电气激励/检测手段，判断对端是否提供了预期的终端特性。
状态： 该阶段通常处于Idle（电气空闲）和短暂探测之前切换。
常见失败点： 走线断/焊接问题/连错lane
### 2. Polling
目的： 
    ◦ 双方交互TS1/TS2
    ◦ RX做CDR Lock / Bit Lock / Symbol Lock
    ◦ 多Lane 时做Lane-to-Lane deskew / 对齐
    ◦ 交换能力信息（支持速率，lane数，是否支持Gen3）
方法： 通过TS1 / TS2交互，有特定格式协议
常见失败点： 
    ◦ CDR锁不住（抖动大，眼图差）
    ◦ Lane间skew太大导致多lane对齐失败
    ◦ 对端能力配置不匹配（一端强制Gen3，一端只支持Gen2）
### 3. Configuration
目的： 
    ◦ Link Width协商：比如从x16掉成x8
    ◦ 确认lane极性等
    ◦ 从idle到complete，准备进入L0
方法： 通过TS1/TS2交互
常见失败点： 
    ◦ lane不稳定，确认失败
    ◦ lane映射错误
### 4. L0
## Gen1->Gen2
### 1. L0
准备要切换到recover
### 2. Recover.Speed
目的：切速
过程：
    ◦ 交互训练，双方达成一致，开始切速.
    ◦ 位置短暂的idle，给切速留窗口.
    ◦ Speed 从2.5GT/s->5GT/s
### 3. Recover.RcvrLock
目的：让接收端在5GT/s下重新获得稳定解码能力，几乎等价于Pollng做的事情
过程：
    ◦ CDR Lock
    ◦ 字对齐
    ◦ 多lane对齐
    ◦ 持续接收训练序列并验证稳定
### 4. Recover.idle/wait/configuartion
作用: 确认稳定，准备回归L0.
## Gen2->Gen3
和Gen1->Gen2的过程基本一致，多了个EQ的过程。
添加EQ的原因是5GT/s->8GT/s，几乎翻倍，速率过快，信号衰弱和误码比较严重，需要eq来均衡。
### 问题
## 1. 如果在Configuration阶段，Polling训练的配置不稳定，会怎么办？
    ◦ 有可能在Configuration阶段，降低确认配置，比如x8->x4.
    ◦ 也有可能回归Polling阶段，降到x4后，再次训练，然后来到Configuration阶段确认。
## 2. Gen1->Gen2和Gen2->Gen3的Lock过程有些等价于Polling？
是的，做的事情几乎是可以等价的，因为切换的过程速率变了，多lane对齐，CDR Lock等都需要重新做。
## 3. 从Gen1->Gen2和Gen2->Gen3是否提速，由sw还是hw决定？
多数情况下，当链路第一次起来后（Gen1）, 会按照内部策略自动尝试升到双方共同支持的最高速率（Gen2/Gen3），失败就退回。
少数情况下，软件可以通过配置寄存器的方式，决定是否提速。
具体情况取决于平台实现。
## 4. 提速的过程能否Gen1->Gen3，跳过Gen2？
可以的，但是多数平台的实现策略通常更加稳妥，一般不会跳过Gen2.
当然直接跳过也是可以的，因为中间也有Recover训练过程。