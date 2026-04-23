# Overview Offset
![在这里插入图片描述](https://i-blog.csdnimg.cn/direct/95439a3ef1ea47eabe134e61ddca3a41.png)
# Basic Configuration
## Read Vendor/Device id
Offset：0x00, 0x02.
Mean: 
    ◦ check vendor id不能为0x00 / 0xffffffff. 
    ◦ 表示为某个厂家的某个型号，比如inter 82599芯片，用于进行芯片级区分
## Read Rev, sub/base code
Offset：0x8，0x0a，0x0b
Mean：
    ◦ 记录版本号
    ◦ get sub code
    ◦ get base code
## Read Hdr type
Offset: 0x0e
Mean：
    ◦ get hdr，check 是typ0/1。
    ◦ 高bit 代表是否为mul funx。
## Read Int pin/line
Offset：0x3c，0x3d
Mean：
    ◦ 中断号, 使用哪个中断线INT（A~D）#。
    ◦ PCIE弃用，使用msi-x。
## Read Command
Offset: 0x04
Mean: 
    ◦ bit0 I/O space 访问，pcie不使用
    ◦ bit1 mem space 访问，pcie使用
    ◦ bit2 允许device作为bus master发起业务。 if枚举正常，buf dma不动，可能跟该bit相关。
    ◦ 其他均是pci历史遗留，pcie均不使用
## CFG BAR
### Read Hdr type
通过tpye类型，确认bar数量，type0即ep，为6，type1即bridge, 为2
**从bar0循环读取，offset += 0x04**
Offset: 0x10
Mean: 
    ◦ bit0 1为io space，0为mem space
    ◦ bit1~2 00h为32bit，10h为64bit
    ◦ bit3 是否可预取，大多数为0，不可预取
    ◦ bit4~31 base addr
地址计算逻辑：
size = ~(readvalue & 0xfffffff0) + 1
基地址需要host分配一段物理地址后，把起始地址写进去
#### 为什么size必须是2次幂？
ip check逻辑是 addr&mask == base，而不是check 地址范围。（对硬件友好）
假设size为24k = 0x6000，base addr = 0x80000000，
场景1： addr = 0x80002000，经过check逻辑计算后，pass.
场景2： addr = 0x80001000, 经过check逻辑计算后，fail.
#### 写入bar0
获取到size后，host分配一块内存，基于bit3决定是否需要可预取，把起始地址写入到bar0.
#### cfg command
基于bit0，决定是否使能io / mem space 访问，然后去配置command寄存器。
## CFG Rom addr
偏移： 0x30
Mean: pcie设备通常不用，用于存放固件的，biso使用，非必要实现。 该bar默认不启用，通过使能位启用。普通bar默认不启用，通过command(0x04) bit1 进行全局使能。
## CFG Cache line/Lattimer
Offset: 0x0c/0x0d
Mean: 
    ◦ 前者是cache line , 0x08表示32 byte，0x10表示64 byte.
    ◦ 后者是 Lattimer , 代表总线占用时间上限，用于限制单个设备在共享总线的占用时间。
    ◦ 上述两个寄存器，pcie均不使用，可直接忽略。
## Read sub vendor/dev id
Offset： 0x2c, 0x2d
Mean：前面的dev/vendor id进行芯片级区分，用于区分是哪家芯片。sub dev/vendor是进行板级区分，同样的芯片不同的板级定制
# Capability Configuration
## The method of reading cap
    ◦ 读取status(0x06)，check bit4，确认是否支持cap list。
    ◦ 根据hdr type决定读取cap point 的地址（0x34）
    ◦ 从0x34开始挨个读取cap list
    ◦ cap的第一个字节是cap id，可以根据id判断是否是需要的cap
    ◦ 如果是，则可以继续向下获取到该cap所有定义。
    ◦ 如果不是，或者想遍历，每一个cap的第二个字节，存放的是下一个cap point。以cap id等于0为结尾。
## PME
该cap是PCIE必须要实现的。
    ◦ 查询cap list中是否有 PME（0x01)
    ◦ check pme version ，不支持pme ver 大于3.
    ◦ check 是否支持 任意一种power status.
    ◦ disable pme and clean pme status，后续会启用.
## MSI
### MSI
    ◦ 查询cap list中是否有 MSI（0x05)
    ◦ check msi 是否支持，如果支持暂时disable，后续会启用
### MSIX
    ◦ 查询cap list中是否有 MSIX（0x11)
    ◦ check msix 是否支持，如果支持暂时disable，后续会启用
### PCIE
    ◦ 查询cap list中是否有 PCIE（0x10)
## CFG SIZE
    ◦ cfg space size为0x100
## PCIE
Base Offset: 0x100
## ARI
Offset:  0x24
Mean：
    ◦ check bit5 是否支持路由转发
    ◦ 交换机和root必须支持，其他设备可以不支持
## Device cap
Offset: 0x02
    ◦ 记录设备类型，是端点，桥后者复合等
    ◦ cap version必须大于2