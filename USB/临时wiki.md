![矢量智能对象](assets/image1.png)

|  |
| --- |
| **文档版本：V1.0** |
| **发布日期：2026/07/23** |

<div align="center"><h1>目 录</h1></div>

- [目 录](#目-录)
- [前 言](#前-言)
- [1 eFuse bit说明](#1-efuse-bit说明)
  - [1.1 eFuse说明](#11-efuse说明)
  - [1.2 私有区域](#12-私有区域)
  - [1.3 Anti rollback区域](#13-anti-rollback区域)
  - [1.4 Bondoption区域](#14-bondoption区域)
  - [1.5 Public key hash区域](#15-public-key-hash区域)
  - [1.6 AKS Key区域](#16-aks-key区域)
  - [1.7 BLK LOCK区域](#17-blk-lock区域)
  - [1.8 CE区域](#18-ce区域)
  - [1.9 用户区域](#19-用户区域)
- [2 eFuse接口说明](#2-efuse接口说明)
  - [2.1 Main域用户接口](#21-main域用户接口)
    - [2.1.1 API简介](#211-api简介)
    - [2.1.2 API定义](#212-api定义)
      - [AX_EFUSE_Init](#ax_efuse_init)
      - [AX_EFUSE_Deinit](#ax_efuse_deinit)
      - [AX_EFUSE_Read](#ax_efuse_read)
      - [AX_EFUSE_Write](#ax_efuse_write)
      - [AX_EFUSE_Lock](#ax_efuse_lock)
    - [2.1.3 数据结构](#213-数据结构)
    - [2.1.4 错误码](#214-错误码)
  - [2.2 Main域TEE接口](#22-main域tee接口)
    - [2.2.1 API简介](#221-api简介)
    - [2.2.2 API定义](#222-api定义)
      - [TEE_Efuse_Init](#tee_efuse_init)
      - [TEE_Efuse_Deinit](#tee_efuse_deinit)
      - [TEE_Efuse_Read](#tee_efuse_read)
      - [TEE_Efuse_Write](#tee_efuse_write)
      - [TEE_Efuse_Lock](#tee_efuse_lock)
    - [2.2.3 数据结构](#223-数据结构)
    - [2.2.4 错误码](#224-错误码)
- [3 eFuse工具说明](#3-efuse工具说明)
  - [3.1 sample_efuse](#31-sample_efuse)
    - [3.1.1 工具介绍](#311-工具介绍)
    - [3.1.2 工具使用示例](#312-工具使用示例)
  - [3.2 sample_efuse_secure](#32-sample_efuse_secure)
    - [3.2.1 工具介绍](#321-工具介绍)
    - [3.2.2 工具使用示例](#322-工具使用示例)
  - [3.3 U-Boot eFuse命令](#33-u-boot-efuse命令)
    - [3.3.1 工具介绍](#331-工具介绍)
    - [3.3.2 工具使用示例](#332-工具使用示例)
  - [3.4 U-Boot eFuse Key烧写命令](#34-u-boot-efuse-key烧写命令)
    - [3.4.1 工具介绍](#341-工具介绍)
    - [3.4.2 工具使用说明及示例](#342-工具使用说明及示例)
      - [Public Key Hash烧写](#public-key-hash烧写)
      - [EFEK烧写](#efek烧写)

**权利声明**

爱芯元智半导体有限公司或其许可人保留一切权利。

非经权利人书面许可，任何单位和个人不得擅自摘抄、复制本文档内容的部分或全部，并不得以任何形式传播。

**注意**

您购买的产品、服务或特性等应受商业合同和条款的约束，本文档中描述的全部或部分产品、服务或特性可能不在您的购买或使用范围之内。除非商业合同另有约定，本公司对本文档内容不做任何明示或默示的声明或保证。

由于产品版本升级或其他原因，本文档内容会不定期进行更新。除非另有约定，本文档仅作为使用指导，本文档中的所有陈述、信息和建议不构成任何明示或暗示的担保。

<div align="center"><h1>前 言</h1></div>

本文档旨在指导您使用AXHELIX的eFuse。

## 适用产品

AXHELIX

## 适读人群

- 软件开发工程师

- 技术支持工程师

## 符号与格式定义

| 符号/格式 | 说明 |
| --- | --- |
| xxx | 表示您可以执行的命令行。 |
| 斜体 | 表示变量。如，“安装目录/AXHELIX_SDK_Vx.x.x/build目录”中的“安装目录”是一个变量，由您的实际环境决定。 |
| 说明/备注： | 表示您在使用产品的过程中，我们向您说明的事项。 |
| 注意： | 表示您在使用产品的过程中，我们需要您注意的事项。 |

<div align="right"><h1>1 eFuse bit说明</h1></div>

## 1.1 eFuse说明

AXHELIX包含efuse0和efuse1两个eFuse控制器，每个控制器包含64个32bit blk，共2Kbit。部分eFuse区域已规划使用，用户不可随意烧写。eFuse0对应blk0~blk63，eFuse1对应blk64~blk127。

## 1.2 私有区域

efuse0 blk0~blk6这部分区域为爱芯芯片出厂预烧写区域，普通用户禁止烧写。其中芯片的unique ID就烧写在efuse0 blk0~blk1区域，共64bit。在Linux下可以通过以下命令获取unique ID：

```bash
cat /proc/ax_proc/hwinfo/uid
```

## 1.3 Anti rollback区域

efuse1 blk9~blk16这部分区域规划为anti rollback区域，用于给用户烧写各个启动镜像的基础版本。如果使能anti rollback功能，启动时会校验当前程序版本和eFuse中基础版本比较。每个eFuse blk对应一个启动镜像版本。

| Blk | rollback version image |
| --- | --- |
| efuse1 blk9 | spl version |
| efuse1 blk10 | atf version |
| efuse1 blk11 | optee version |
| efuse1 blk12 | uboot version |
| efuse1 blk13 | kernel version |
| efuse1 blk14 | rootfs version |

## 1.4 Bondoption区域

eFuse中的efuse0 blk8为bondoption0，efuse1 blk8为bondoption1，两个blk的位定义不同，应按下表配置。

bondoption0：

| Bit | bond name | 状态/作用 | 备注 |
| --- | --- | --- | --- |
| 0 | bondopt_cpfd_npu1_intcnt_npu0_pp_disable_uart0 | 0：不关闭；1：关闭UART0 | 为0时由sec_glb.uart0_dont_use控制；为1时UART0不可用 |
| 1 | bondopt_sw_disable_uart0_download | 0：不限制；1：软件ROM不支持UART0下载 | 软件读取使用 |
| 2 | bondopt_disable_usb | 0：不关闭；1：关闭USB | 为0时由sec_glb.usb_dont_use控制；为1时整个USB IP不可用 |
| 3 | bondopt_sw_disable_usb_download | 0：不限制；1：软件ROM不支持USB下载 | 软件读取使用 |
| 4 | bondopt_sw_disable_pcie_download | 0：不限制；1：软件ROM不支持PCIe下载 | 软件读取使用 |
| 5 | bondopt_sw_rsa2048_rsa3072 | 0：RSA2048；1：RSA3072 | 选择RSA密钥长度 |
| 6 | bondopt_sw_usb2phy_retune | 0：使用GLB默认值；1：使用eFuse retune值 | 控制efuse0 blk6中csr_usb2_tune的使用 |
| 7 | bondopt_sec | 1：开启安全访问 | a*prot[1]==0时，安全访问，可访问sec_glb；为1时，非安全访问，不可访问sec_glb |
| 8 | bondopt_sw_pcie_init_print_disable | 1：不打印PCIe初始化部分信息 | 软件配置 |
| 9 | bondopt_secboot | 1：enable security boot | 置1后进入安全启动流程 |
| 11 | bondopt_disable_jtag | 1：JTAG不可使用 | 为0时由sec_glb.jtag_dont_use控制；为1时debug JTAG不可用 |
| 12 | bondopt_enable_mem_repair | 1：使能repair | 全局repair控制 |
| 24~29 | Reserved | 保留 | 不应配置 |
| 30 | bondopt_sw_pcie_gen5phy_cfg_change | 1：修改PCIe Gen5 PHY参数 | 软件读取使用 |
| 31 | bondopt_sw_pcie_combphy_cfg_change | 1：修改PCIe comb PHY参数 | 软件读取使用 |

## 1.5 Public Key Hash区域

efuse0 blk15~blk22该区域用于存储安全启动RSA public key的SHA256值，确保Secure boot时校验使用的Public key不被篡改，共256bit。

## 1.6 AKS Key区域

efuse0 blk23~blk30为AKS Key区域，共256bit。该区域用于保存镜像AES解密流程使用的EFEK相关材料。

## 1.7 BLK LOCK区域

eFuse bit只能从0写成1，并且只能写1次。如需要把整个blk锁住，需要通过写入lock bit实现。每个blk都有单独一个bit用于lock，lock bit写1后，其对应的blk中任何bit无法修改。

- efuse0 blk31中bit0~bit30用于锁住efuse0 blk0~blk30。
- efuse1 blk62中bit0~bit31用于锁住efuse1 blk0~blk31。
- efuse1 blk63中bit0~bit29用于锁住efuse1 blk32~blk61。

## 1.8 CE区域

efuse0 blk32~blk63一共1024bit，为CE专用区域，普通用户禁止通过通用eFuse接口操作，需要调用CE接口。

## 1.9 用户区域

其他区域，efuse0 blk9~blk14及efuse1 blk23~blk61一共1440bit用户根据产品需求使用。

<div align="right"><h1>2 eFuse接口说明</h1></div>

## 2.1 Main域用户接口

### 2.1.1 API简介

eFuse模块Main域提供的API接口如下：

- AX_EFUSE_Init：初始化eFuse相关的软硬件资源。
- AX_EFUSE_Deinit：释放eFuse相关的软硬件资源。
- AX_EFUSE_Read：读取eFuse blk内容。
- AX_EFUSE_Write：写入eFuse blk内容。
- AX_EFUSE_Lock：锁定eFuse blk，锁定后不可再写入。

### 2.1.2 API定义

#### AX_EFUSE_Init

**【描述】**

初始化eFuse模块的软硬件资源。

**【语法】**

```c
AX_S32 AX_EFUSE_Init(void);
```

**【参数】**

无

**【返回值】**

| 返回值 | 描述 |
| --- | --- |
| 非0 | 失败 |
| 0 | 成功 |

**【需求】**

- 头文件：ax_efuse_api.h

- 库文件：libax_efuse.so

**【注意】**

无

**【举例】**

参考msp/sample/efuse/sample_efuse.c

**【相关主题】**

无

#### AX_EFUSE_Deinit

**【描述】**

释放eFuse模块的软硬件资源。

**【语法】**

```c
AX_S32 AX_EFUSE_Deinit(void);
```

**【参数】**

无

**【返回值】**

| 返回值 | 描述 |
| --- | --- |
| 非0 | 失败 |
| 0 | 成功 |

**【需求】**

- 头文件：ax_efuse_api.h

- 库文件：libax_efuse.so

**【注意】**

无

**【举例】**

参考msp/sample/efuse/sample_efuse.c

**【相关主题】**

无

#### AX_EFUSE_Read

**【描述】**

读取eFuse对应blk的值。

**【语法】**

```c
AX_S32 AX_EFUSE_Read(AX_S32 blk, AX_S32 *data);
```

**【参数】**

| 参数名称 | 描述 | 输入/输出 |
| --- | --- | --- |
| blk | eFuse全局blk序号0~127，其中blk0~blk63对应efuse0，blk64~blk127对应efuse1；区域使用限制见“eFuse bit说明” | 输入 |
| data | 用于保存读取到的eFuse值 | 输出 |

**【返回值】**

| 返回值 | 描述 |
| --- | --- |
| 非0 | 失败 |
| 0 | 成功 |

**【需求】**

- 头文件：ax_efuse_api.h
- 库文件：libax_efuse.so

**【注意】**

blk的编码范围仅表示当前接口实现可接收的编号范围。efuse0 blk0~blk6为出厂私有区域，普通用户应通过特定系统接口读取所需信息；efuse0 blk32~blk63为CE专用区域，普通用户禁止通过该接口操作。

**【举例】**

参考msp/sample/efuse/sample_efuse.c

**【相关主题】**

无

#### AX_EFUSE_Write

**【描述】**

写入eFuse对应blk的值。

**【语法】**

```c
AX_S32 AX_EFUSE_Write(AX_S32 blk, AX_S32 data);
```

**【参数】**

| 参数名称 | 描述 | 输入/输出 |
| --- | --- | --- |
| blk | 接口实现可接收的eFuse全局blk序号为0~30和32~125；blk31、blk126和blk127为lock blk，不支持通过该接口直接写入；区域使用限制见“eFuse bit说明” | 输入 |
| data | 需要写入的32bit eFuse值 | 输入 |

**【返回值】**

| 返回值 | 描述 |
| --- | --- |
| AX_SUCCESS | 成功 |
| AX_ERR_EFUSE_ILLEGAL_PARAM | blk超出范围或为不支持直接写入的lock blk |
| AX_ERR_EFUSE_SYS_NOTREADY | Linux NVMEM设备未准备就绪 |
| AX_ERR_EFUSE_WRITE_FAIL | eFuse写入失败 |
| AX_ERR_EFUSE_ACCESS_DENIED | 调用进程无写权限或缺少CAP_SYS_RAWIO capability |

**【需求】**

- 头文件：ax_efuse_api.h
- 库文件：libax_efuse.so

**【注意】**

eFuse写入为不可逆操作，已写为1的bit不能重复写入。efuse0 blk0~blk6为出厂私有区域，efuse0 blk32~blk63为CE专用区域，普通用户禁止通过该接口烧写。该接口通过Linux NVMEM设备/sys/bus/nvmem/devices/ax-efuse0/nvmem执行写操作，调用进程需要具备文件写权限和CAP_SYS_RAWIO capability。

**【举例】**

参考msp/sample/efuse/sample_efuse.c

**【相关主题】**

无

#### AX_EFUSE_Lock

**【描述】**

锁定eFuse blk。锁定后该blk不可再写入，其中尚未写为1的bit也无法继续写入。

**【语法】**

```c
AX_S32 AX_EFUSE_Lock(AX_S32 blk);
```

**【参数】**

| 参数名称 | 描述 | 输入/输出 |
| --- | --- | --- |
| blk | 接口实现可锁定的eFuse全局blk序号为0~30和64~125；区域使用限制见“eFuse bit说明” | 输入 |

**【返回值】**

| 返回值 | 描述 |
| --- | --- |
| AX_SUCCESS | 成功 |
| AX_ERR_EFUSE_ILLEGAL_PARAM | blk不是支持的锁定目标 |
| AX_ERR_EFUSE_SYS_NOTREADY | Linux NVMEM设备未准备就绪 |
| AX_ERR_EFUSE_LOCK_FAIL | eFuse blk锁定失败 |
| AX_ERR_EFUSE_ACCESS_DENIED | 调用进程无写权限或缺少CAP_SYS_RAWIO capability |

**【需求】**

- 头文件：ax_efuse_api.h
- 库文件：libax_efuse.so

**【注意】**

eFuse blk锁定操作不可回退。efuse0 blk0~blk6为出厂私有区域，普通用户禁止通过该接口锁定。efuse0 blk31、blk32~blk63以及efuse1对应的全局blk126~blk127不支持作为锁定目标。

**【举例】**

参考msp/sample/efuse/sample_efuse.c

**【相关主题】**

无

### 2.1.3 数据结构

无

### 2.1.4 错误码

eFuse API 错误码如下：

| 错误代码 | 宏定义 | 描述 |
| --- | --- | --- |
| 0x8005000a | AX_ERR_EFUSE_ILLEGAL_PARAM | blk参数超出范围 |
| 0x80050010 | AX_ERR_EFUSE_SYS_NOTREADY | 系统未准备就绪 |
| 0x80050080 | AX_ERR_EFUSE_READ_FAIL | eFuse读失败 |
| 0x80050081 | AX_ERR_EFUSE_WRITE_FAIL | eFuse写失败 |
| 0x80050082 | AX_ERR_EFUSE_MMAP_FAIL | eFuse mmap失败 |
| 0x80050083 | AX_ERR_EFUSE_LOCK_FAIL | eFuse锁定失败 |
| 0x80050084 | AX_ERR_EFUSE_NOT_SUPPORT | 当前配置不支持该操作 |
| 0x80050085 | AX_ERR_EFUSE_ACCESS_DENIED | eFuse操作被拒绝 |

## 2.2 Main域TEE接口

### 2.2.1 API简介

eFuse模块Main域TEE中提供的API接口如下：

- TEE_Efuse_Init：初始化eFuse相关的软硬件资源。
- TEE_Efuse_Deinit：释放eFuse相关的软硬件资源。
- TEE_Efuse_Read：读取eFuse blk内容。
- TEE_Efuse_Write：写入eFuse blk内容。
- TEE_Efuse_Lock：锁定eFuse blk，锁定后不可再写入。

### 2.2.2 API定义

#### TEE_Efuse_Init

**【描述】**

初始化TEE中的eFuse模块。

**【语法】**

```c
TEE_Result TEE_Efuse_Init(void);
```

**【参数】**

无

**【返回值】**

| 返回值 | 描述 |
| --- | --- |
| 非0 | 失败 |
| 0 | 成功 |

**【需求】**

- 头文件：tee_internal_api.h

**【注意】**

无

**【举例】**

参考msp/sample/optee/optee_efuse/ta/optee_efuse_ta.c

**【相关主题】**

无

#### TEE_Efuse_Deinit

**【描述】**

释放TEE中的eFuse模块资源。

**【语法】**

```c
TEE_Result TEE_Efuse_Deinit(void);
```

**【参数】**

无

**【返回值】**

| 返回值 | 描述 |
| --- | --- |
| 非0 | 失败 |
| 0 | 成功 |

**【需求】**

- 头文件：tee_internal_api.h

**【注意】**

无

**【举例】**

参考msp/sample/optee/optee_efuse/ta/optee_efuse_ta.c

**【相关主题】**

无

#### TEE_Efuse_Read

**【描述】**

TEE中读取eFuse对应blk的值。

**【语法】**

```c
TEE_Result TEE_Efuse_Read(uint32_t blk, uint32_t *data);
```

**【参数】**

| 参数名称 | 描述 | 输入/输出 |
| --- | --- | --- |
| blk | eFuse全局blk序号0~127，其中blk0~blk63对应efuse0，blk64~blk127对应efuse1；区域使用限制见“eFuse bit说明” | 输入 |
| data | 用于保存读取到的eFuse值 | 输出 |

**【返回值】**

| 返回值 | 描述 |
| --- | --- |
| 非0 | 失败 |
| 0 | 成功 |

**【需求】**

- 头文件：tee_internal_api.h

**【注意】**

blk的编码范围仅表示当前接口实现可接收的编号范围。efuse0 blk0~blk6为出厂私有区域，普通用户应通过特定系统接口读取所需信息；efuse0 blk32~blk63为CE专用区域，普通用户禁止通过该接口操作。

**【举例】**

参考msp/sample/optee/optee_efuse/ta/optee_efuse_ta.c

**【相关主题】**

无

#### TEE_Efuse_Write

**【描述】**

TEE中写入eFuse对应blk的值。

**【语法】**

```c
TEE_Result TEE_Efuse_Write(uint32_t blk, uint32_t data);
```

**【参数】**

| 参数名称 | 描述 | 输入/输出 |
| --- | --- | --- |
| blk | 接口实现可接收的eFuse全局blk序号为0~30和32~125；blk31、blk126和blk127为lock blk，不支持通过该接口直接写入；区域使用限制见“eFuse bit说明” | 输入 |
| data | 需要写入的eFuse值 | 输入 |

**【返回值】**

| 返回值 | 描述 |
| --- | --- |
| 非0 | 失败 |
| 0 | 成功 |

**【需求】**

- 头文件：tee_internal_api.h

**【注意】**

eFuse写入为不可逆操作，已写为1的bit不能重复写入。efuse0 blk0~blk6为出厂私有区域，efuse0 blk32~blk63为CE专用区域，普通用户禁止通过该接口烧写。

**【举例】**

参考msp/sample/optee/optee_efuse/ta/optee_efuse_ta.c

**【相关主题】**

无

#### TEE_Efuse_Lock

**【描述】**

TEE中锁定eFuse blk。锁定后该blk不可再写入，其中尚未写为1的bit也无法继续写入。

**【语法】**

```c
TEE_Result TEE_Efuse_Lock(uint32_t blk);
```

**【参数】**

| 参数名称 | 描述 | 输入/输出 |
| --- | --- | --- |
| blk | 接口实现可锁定的eFuse全局blk序号为0~30和64~125；区域使用限制见“eFuse bit说明” | 输入 |

**【返回值】**

| 返回值 | 描述 |
| --- | --- |
| 非0 | 失败 |
| 0 | 成功 |

**【需求】**

- 头文件：tee_internal_api.h

**【注意】**

eFuse blk锁定操作不可回退。efuse0 blk0~blk6为出厂私有区域，普通用户禁止通过该接口锁定。efuse0 blk31、blk32~blk63以及efuse1对应的全局blk126~blk127不支持作为锁定目标。

**【举例】**

参考msp/sample/optee/optee_efuse/ta/optee_efuse_ta.c

**【相关主题】**

无


### 2.2.3 数据结构

无

### 2.2.4 错误码

eFuse TEE API成功时返回TEE_SUCCESS，失败时直接返回底层eFuse驱动状态。由于接口返回类型为TEE_Result，负数状态以对应的32bit无符号值表示。

| 错误代码 | 底层状态 | 描述 |
| --- | --- | --- |
| 0xffffffff | AX_EFUSE_MEMMAP_FAIL | eFuse寄存器映射失败 |
| 0xfffffffe | AX_EFUSE_INIT_FAIL | eFuse初始化失败 |
| 0xfffffffd | AX_EFUSE_PARAM_FAIL | 参数无效、未初始化或blk不支持该操作 |
| 0xfffffffc | AX_EFUSE_READ_FAIL | eFuse读取失败 |
| 0xfffffffb | AX_EFUSE_READ_WRITE | eFuse写入等待超时 |
| 0xfffffffa | AX_EFUSE_LOCKED | eFuse blk已锁定 |
| 0xfffffff9 | AX_EFUSE_WRITE_TWICE | eFuse bit被重复写入 |
| 0xfffffff8 | AX_EFUSE_FAULT | eFuse硬件检测到写入故障 |
| 0xfffffff7 | AX_EFUSE_NOT_SUPPORT | 当前配置不支持该操作 |

<div align="right"><h1>3 eFuse工具说明</h1></div>

## 3.1 sample_efuse

### 3.1.1 工具介绍

sample_efuse是eFuse功能测试工具，支持读取、写入和锁定指定blk，dump全部128个blk，以及读取芯片unique ID。

工具源码位于msp/sample/efuse目录。写入和锁定操作均不可逆，工具执行写入或锁定前不会二次确认，必须提前核对blk和写入数据。通用sample_efuse禁止通过写blk8新增使能secure boot bit9或secure system bit7。

运行帮助信息：

```text
/tmp/test # ./sample_efuse
./sample_efuse <type> <blk> [data]
type:
        -r: read efuse blk
        -w: write efuse blk
        -l: lock efuse blk, can't write
        -d: dump efuse
        -c: get chip uid from efuse
example:
        ./sample_efuse -r 10
        ./sample_efuse -w 10 0x12345678
        ./sample_efuse -l 10
        ./sample_efuse -d
        ./sample_efuse -c
```

### 3.1.2 工具使用示例

#### eFuse读blk

读取eFuse全局blk8。

```text
/tmp/test # ./sample_efuse -r 8
blk8: 0x0
```

#### eFuse写blk

向eFuse全局blk91写入0xa5a5a5a5。

```text
/tmp/test # ./sample_efuse -w 91 0xa5a5a5a5
write blk91: 0xa5a5a5a5
```

#### eFuse blk上锁

锁定eFuse全局blk91。锁定后该blk不可再次写入。

```text
/tmp/test # ./sample_efuse -l 91
lock blk91
```

#### eFuse dump操作

dump全部128个eFuse全局blk的内容。

```text
/tmp/test # ./sample_efuse -d
CASE_DATA:EFUSE:block00 = 0x00000000
CASE_DATA:EFUSE:block01 = 0x00000000
CASE_DATA:EFUSE:block02 = 0x00000000
CASE_DATA:EFUSE:block03 = 0x00000000
CASE_DATA:EFUSE:block04 = 0x5a5a5a5a
CASE_DATA:EFUSE:block05 = 0x00000000
……
```

## 3.2 sample_efuse_secure

### 3.2.1 工具介绍

sample_efuse_secure用于生成并烧写Public Key Hash和EFEK。Public Key Hash写入efuse0 blk15~blk22，EFEK写入efuse0 blk23~blk30；真正写入时，每个blk依次执行写入、回读校验和锁定，任一步失败即停止后续操作。

工具源码位于msp/sample/efuse_secure目录，生成的可执行程序名称为sample_efuse_secure。工具依赖cipher驱动，运行前需要确认/dev/ax_cipher存在。将EFUSE_WRITE_ENABLE设为1执行烧写和锁定操作时，还需要确认Linux NVMEM设备/sys/bus/nvmem/devices/ax-efuse0/nvmem存在，并使用具备CAP_SYS_RAWIO capability的root用户运行。

运行帮助信息：

```text
/tmp/test # ./sample_efuse_secure
./sample_efuse_secure pubkey <2048/3072>
example: write RSA pubkey hash to PKH blk
        ./sample_efuse_secure pubkey 2048
        ./sample_efuse_secure pubkey 3072
./sample_efuse_secure aeskey
example: write efek to EFEK blk
        ./sample_efuse_secure aeskey
```

### 3.2.2 工具使用示例

sample_efuse_secure通过EFUSE_WRITE_ENABLE控制实际烧写。设为0时，pubkey命令仍会计算并打印Public Key Hash；aeskey命令会跳过HUK固化、EFEK计算和eFuse写入，因此不会打印EFEK。确认数据无误后，可将msp/sample/efuse_secure/efuse_secure_tool.c中的EFUSE_WRITE_ENABLE设为1并重新编译。

```c
#define EFUSE_WRITE_ENABLE      1
```

#### Public key hash烧写

通过pubkey参数选择RSA 2048或RSA 3072公钥。工具使用源码中预置的模数和指数计算SHA256，并将结果作为Public Key Hash。AXHELIX仅使用一组Public Key Hash区域，不支持通过key索引选择多个PKH区域。使用RSA 3072时，真正烧写完成后还会同时配置RSA 3072选择位。

**【举例】**

生成并烧写Public Key Hash区域。

```text
/tmp/test # ./sample_efuse_secure pubkey 2048
RSA2048 key is:
2b be 06 52 a2 f2 42 93 e3 33 84 d4 24 75 c5 15 
66 b6 06 57 a7 f7 47 98 e8 38 89 d9 29 7a ca 1a 
6b bb 0b 5c ac fc 4c 9d ed 3d 8e de 2e 7f cf 1f 
70 c0 10 61 b1 01 52 a2 f2 42 93 e3 33 84 d4 24 
75 c5 15 66 b6 06 57 a7 f7 47 98 e8 38 89 d9 29 
7a ca 1a 6b bb 0b 5c ac fc 4c 9d ed 3d 8e de 2e 
7f cf 1f 70 c0 10 61 b1 01 52 a2 f2 42 93 e3 33 
84 d4 24 75 c5 15 66 b6 06 57 a7 f7 47 98 e8 38 
d9 1b c8 77 27 d7 86 36 e6 95 45 f5 a4 54 04 b4 
63 13 c3 72 22 d2 81 31 e1 90 40 f0 9f 4f ff ae 
5e 0e be 6d 1d cd 7c 2c dc 8b 3b eb 9a 4a fa a9 
59 09 b9 68 18 c8 77 27 d7 86 36 e6 95 45 f5 a4 
54 04 b4 63 13 c3 72 22 d2 81 31 e1 90 40 f0 9f 
4f ff ae 5e 0e be 6d 1d cd 7c 2c dc 8b 3b eb 9a 
4a fa a9 59 09 b9 68 18 c8 77 27 d7 86 36 e6 95 
45 f5 a4 54 04 b4 63 13 c3 72 22 d2 81 31 e1 90 
01 00 01 00 
RSA2048 hash value is:
d7a64381, c31c0a09, d760ba23, 1d75f4fc
063b97fe, 8697d6bf, bdf4f380, 164797c5
```

使能写入后，sample_efuse_secure会对Public Key Hash区域的8个blk逐个执行写入、回读校验和锁定；选择RSA 3072时会同时配置RSA 3072选择位。

#### EFEK烧写

通过aeskey参数生成并烧写EFEK。工具使用HUK对源码中配置的32字节FEK进行加密，得到写入eFuse的EFEK。如需修改FEK内容，应修改efuse_secure_tool.c中的aes_key配置。

```c
static const char *aes_key =
    "0000000000000000000000000000000000000000000000000000000000000000";
```

**【举例】**

```text
/tmp/test # ./sample_efuse_secure aeskey
efek is:
0xec63d1ae 0x1ab7e574 0xb847f9dd 0x4350d8f5 0xec63d1ae 0x1ab7e574 0xb847f9dd 0x4350d8f5 
```

使能写入后，sample_efuse_secure会对EFEK区域的8个blk逐个执行写入、回读校验和锁定。

注意：

工具生成EFEK前会固化随机HUK，并使用该HUK加密FEK。HUK和EFEK必须在同一颗芯片上生成和使用，HUK固化后不可更改。

## 3.3 U-Boot eFuse命令

### 3.3.1 工具介绍

U-Boot提供ax_efuse命令，用于读取、写入、锁定和连续dump eFuse全局blk。该命令随CONFIG_CMD_AXERA_CIPHER编译，配置项默认不使能；使能AXERA_SECURE_BOOT时会同时选择该配置。

- 注意：eFuse写入和锁定均为不可逆操作。执行write或lock时，工具会要求用户输入y进行确认。锁定仅支持全局blk0~blk30和blk64~blk125。

命令格式如下：

```text
AXERA-UBOOT=>ax_efuse 
ax_efuse - Axera efuse debug tool

Usage:
ax_efuse read  <blk> [count]   - read one or consecutive blks (blk: 0..127)
ax_efuse dump  <start> [count]  - read consecutive blks (count <= 128)
ax_efuse write <blk> <hexval>   - program one blk, IRREVERSIBLE
ax_efuse lock  <blk>            - lock one blk, IRREVERSIBLE

blk 0..63 map to efuse0, blk 64..127 map to efuse1.
lock is not supported on blk 31, 32..63, 126 and 127.
```

| 命令 | 说明 |
| --- | --- |
| read <blk> [count] | 从指定blk开始读取；count可选，默认为1。blk和count均按十进制解析，读取范围不能超过全局blk127。 |
| dump <start> [count] | 从start开始连续读取blk；count可选，默认为1，最大为128。 |
| write <blk> <hexval> | 向指定blk写入32bit数据。blk按十进制解析，value按十六进制解析。 |
| lock <blk> | 锁定指定blk。锁定后该blk不可再次烧写。 |

### 3.3.2 工具使用示例

读取blk14内容：

```text
AXERA-UBOOT=>ax_efuse read 14
blk 14: 0x00000000
```

连续读取blk14开始的4个blk：

```text
AXERA-UBOOT=>ax_efuse read 14 4
blk 14: 0x00000000
blk 15: 0x00000000
blk 16: 0x00000000
blk 17: 0x00000000
```

向blk91写入数据。命令执行后需要按提示输入y确认：

```text
AXERA-UBOOT=>ax_efuse write 91 0xa5a5a5a5
about to write blk 91 = 0xa5a5a5a5
Writing efuse blk 91 is IRREVERSIBLE. Continue? <y/N> y
blk 91 written
```

锁定blk91。命令执行后需要按提示输入yes确认：

```text
AXERA-UBOOT=>ax_efuse lock 91                                       
Locking efuse blk 91 is IRREVERSIBLE. Continue? <y/N> y
blk 91 locked
```

dump blk90开始的4个blk：

```text
AXERA-UBOOT=>ax_efuse dump 90 4
blk  90: 0x00000000
blk  91: 0xa5a5a5a5
blk  92: 0x00000000
blk  93: 0x00000000
```

## 3.4 U-Boot eFuse Key烧写命令

### 3.4.1 工具介绍

U-Boot提供efuse_secure_tool命令，用于生成并烧写Public Key Hash和EFEK。该命令与ax_efuse命令一同随CONFIG_CMD_AXERA_CIPHER编译。

工具只支持pubkey和aeskey两个子命令，使用efuse_secure_tool.c中预置的RSA公钥数据和FEK，不支持从外部内存地址或文件读取key。

为了防止误烧写，EFUSE_WRITE_ENABLE默认设置为0，此时工具只打印提示信息。确认数据无误后，可将efuse_secure_tool.c中的EFUSE_WRITE_ENABLE修改为1并重新编译U-Boot。开启写入后，每个key blk写入后会立即锁定。

命令格式如下：

```text
AXERA-UBOOT=>efuse_secure_tool 
efuse_secure_tool - Efuse secure tool

Usage:
efuse_secure_tool pubkey <2048|3072> - write RSA pubkey hash to PKH blk
efuse_secure_tool aeskey            - write efek to EFEK blk
```

| 命令 | 说明 |
| --- | --- |
| pubkey <2048 \| 3072> | 使用源码中对应长度的RSA模数和指数计算SHA256，并处理Public Key Hash区域。 |
| aeskey | 使用HUK加密源码中配置的32字节FEK，生成并处理EFEK区域。 |

### 3.4.2 工具使用说明及示例

执行pubkey子命令时，工具根据参数选择RSA 2048或RSA 3072公钥数据，计算Public Key Hash。使能写入后，工具将结果写入并锁定efuse0 blk15~blk22；选择RSA 3072时还会同时配置RSA 3072选择位。

#### Public Key Hash烧写

```text
AXERA-UBOOT=>efuse_secure_tool pubkey 2048                                                                   
RSA2048 key is:
2b be 06 52 a2 f2 42 93 e3 33 84 d4 24 75 c5 15 
66 b6 06 57 a7 f7 47 98 e8 38 89 d9 29 7a ca 1a 
6b bb 0b 5c ac fc 4c 9d ed 3d 8e de 2e 7f cf 1f 
70 c0 10 61 b1 01 52 a2 f2 42 93 e3 33 84 d4 24 
75 c5 15 66 b6 06 57 a7 f7 47 98 e8 38 89 d9 29 
7a ca 1a 6b bb 0b 5c ac fc 4c 9d ed 3d 8e de 2e 
7f cf 1f 70 c0 10 61 b1 01 52 a2 f2 42 93 e3 33 
84 d4 24 75 c5 15 66 b6 06 57 a7 f7 47 98 e8 38 
d9 1b c8 77 27 d7 86 36 e6 95 45 f5 a4 54 04 b4 
63 13 c3 72 22 d2 81 31 e1 90 40 f0 9f 4f ff ae 
5e 0e be 6d 1d cd 7c 2c dc 8b 3b eb 9a 4a fa a9 
59 09 b9 68 18 c8 77 27 d7 86 36 e6 95 45 f5 a4 
54 04 b4 63 13 c3 72 22 d2 81 31 e1 90 40 f0 9f 
4f ff ae 5e 0e be 6d 1d cd 7c 2c dc 8b 3b eb 9a 
4a fa a9 59 09 b9 68 18 c8 77 27 d7 86 36 e6 95 
45 f5 a4 54 04 b4 63 13 c3 72 22 d2 81 31 e1 90 
01 00 01 00 
RSA2048 hash value is:
d7a64381, c31c0a09, d760ba23, 1d75f4fc
063b97fe, 8697d6bf, bdf4f380, 164797c5
```

执行aeskey子命令时，工具先查询HUK是否已经固化。若尚未固化，则生成并固化随机HUK；若已经固化，则复用现有HUK。随后工具使用HUK加密源码中配置的FEK，生成EFEK。使能写入后，工具将EFEK写入并锁定efuse0 blk23~blk30。

#### EFEK烧写

```text
AXERA-UBOOT=>efuse_secure_tool aeskey
fail: ret: 0, result: 8d000000
huk not provisioned yet, provisioning a random one
efek is:
0x304eeb23 0xc271c48f 0xf929121c 0xefd3d588 0x304eeb23 0xc271c48f 0xf929121c 0xefd3d588 
```

HUK固化后不可更改，EFEK必须在目标芯片上使用该芯片的HUK生成。修改FEK或RSA公钥数据后，应先在禁止写入的配置下核对工具打印结果，再使能写入。
