# AX525 NOR U-Boot U 盘升级：架构与实现全解析

  

> 适用范围：AX525 正式 U-Boot（非 FDL2）中的 `usb_storage_update` 命令。

> 代码基线：`u-boot-2020.04/cmd/axera/usb_stor_update/` + `u-boot-2020.04/cmd/axera/update/`。

> 所有 `file:line` 引用均以本仓库当前代码为准。

  

---

  

## 一、定位：两条独立的链

  

AX525 上有两套完全独立的固件路径，先分清楚：

  

```

启动链(日常)   BootROM ──► SPL ──► U-Boot(raw) ──► Kernel

升级链(下载)   BootROM ──► FDL1 ──► FDL2 ──► 私有协议 ──► Flash

```

  

`usb_storage_update` **不属于上面任何一条**——它是**正式 U-Boot 内置的一条“就地升级”命令**：

设备已经正常启动到 U-Boot，插一个 U 盘，U-Boot 自己当 **USB Host** 去读 U 盘、把镜像刷进板载 Flash。

  

它与 FDL2 下载升级的区别：

  

| 维度 | FDL2 下载 | `usb_storage_update` |

|---|---|---|

| 谁主动 | PC 端工具推 | 设备端 U-Boot 主动拉 |

| 传输 | USB/UART 私有协议 | USB Host + FAT 文件读取 |

| 运行环境 | 独立的 FDL2 程序 | 正式 U-Boot 命令 |

| 数据源 | PC 上的 axp/镜像 | U 盘上的 XML + 镜像文件 |

  

**设备角色：**

  

```

   AX525 (USB Host)  ◄──── USB ────►  U盘 (Mass Storage Device)

        │                                 │

   xHCI + DWC3 驱动                   DOS/MBR + FAT32

        │                                 │

   枚举为 blk "usb" 0                AX525_nor.xml + 各镜像文件

```

  

---

  

## 二、分层架构

  

整条命令可拆成 6 层，自上而下：

  

```

┌─────────────────────────────────────────────────────────────┐

│ L6 命令层     do_usb_stor_update()          编排整个流程       │

├─────────────────────────────────────────────────────────────┤

│ L5 数据源层   usb start → blk_get_dev → fat_register_device   │

│               USB Host(xHCI/DWC3) + FAT 文件系统               │

├─────────────────────────────────────────────────────────────┤

│ L4 解析层     update_parse_xml()                              │

│               ├ get_part_info_rawdata  分区表(name/size/unit) │

│               └ get_part_image_name_xml 镜像清单(<Img><File>) │

├─────────────────────────────────────────────────────────────┤

│ L3 编排层     usb_stor_update_save_storage()                  │

│               分块读文件 / sparse 判定 / 逐分区驱动写入        │

├─────────────────────────────────────────────────────────────┤

│ L2 存储抽象层 switch(storage_sel)  ★ 单点决策 ★               │

│               to_storage / raw_erase / get_part_info / 容量   │

├─────────────────────────────────────────────────────────────┤

│ L1 介质驱动层 NOR:MTD(nor0)  NAND:MTD+坏块  eMMC:blk_dwrite    │

└─────────────────────────────────────────────────────────────┘

```

  

**架构精髓在 L2**：L3~L6 完全不关心底层是 NOR/NAND/eMMC，所有介质差异被 `storage_sel`

一个枚举收敛到几个 `switch` 里。换介质 = 换一个枚举值，上层零改动。

  

---

  

## 三、一次完整升级的数据流

  

```

 do_usb_stor_update (usb_storage_update.c:925)

     │

 ┌───▼─────────────────────────────────────────┐

 │ ① usb start → blk_get_dev("usb",0)          │  L5 拿到 U 盘块设备

 │    fat_register_device(dev, 1 或 0)          │      挂成 FAT

 └───┬─────────────────────────────────────────┘

     │

 ┌───▼─────────────────────────────────────────┐

 │ ② update_parse_part_info → update_parse_xml │  L4 读 AX525_nor.xml

 │    fat_read_file("AX525_nor.xml")            │

 │    ├ 分区表 → name + size(×unit)             │      得到分区链表

 │    ├ auto-resize 末分区 = 容量 - 已用         │      (update_part_info)

 │    └ 每分区 → 对应镜像文件名 / "none"         │

 └───┬─────────────────────────────────────────┘

     │

 ┌───▼─────────────────────────────────────────┐

 │ ③ usb_stor_update_bin_check                  │  校验：文件存在

 │    fat_exists + fat_size ≤ part_size         │       + 不超分区

 └───┬─────────────────────────────────────────┘

     │

 ┌───▼─────────────────────────────────────────┐

 │ ④ usb_stor_update_save_storage  逐分区：     │  L3 编排

 │    update_parts_info → 拼 mtdparts/bootargs  │

 │    分块 fat_read_file(10MB/块)               │

 │    is_sparse_image?                          │

 │      ├ sparse → erase + write_sparse_img     │

 │      └ raw    → usb_stor_update_to_storage ──┼──► ⑤

 └───┬─────────────────────────────────────────┘

     │

 ┌───▼─────────────────────────────────────────┐

 │ ⑤ to_storage: update_verify_image(验签)      │  L2→L1

 │    switch(storage_sel):                      │

 │      NOR  → to_spinor : mtd_erase+mtd_write  │

 │      NAND → to_spinand: +跳坏块              │

 │      eMMC → blk_dwrite + hwpart              │

 └───┬─────────────────────────────────────────┘

     │

 ┌───▼─────────────────────────────────────────┐

 │ ⑥ env usbupdate=finish → set_reboot_mode     │  收尾

 │    (按 boot_type 写复位启动源) → reboot       │

 └──────────────────────────────────────────────┘

```

  

---

  

## 四、函数调用关系图（详细，带 file:line）

  

缩进 = 调用层级；`[F]` 标注该函数定义处；`→` 表示跨文件跳转；`{...}` 是条件分支。

  

```

do_usb_stor_update()                                    [F] usb_storage_update.c:925

│

├─ run_command("usb start")                                 usb_storage_update.c:937

│     └─(USB Host 栈) xhci / dwc3-axera 枚举 U 盘

│

├─ blk_get_dev("usb", 0)                                    usb_storage_update.c:944

├─ fat_register_device(usb_stor_desc, 1 | 0)                usb_storage_update.c:951/954

│

├─ update_parse_part_info(&pbin_info)                   [F] axera_update.c:1828

│  └─ update_parse_xml(bin_info)                        [F] axera_update.c:1705

│     ├─ fat_exists("AX525_nor.xml")                        axera_update.c:1717

│     ├─ fat_size / fat_read_file(XML → part_data)          axera_update.c:1723/1735

│     ├─ get_part_info_rawdata(&pheader, part_data, size)   [F] axera_update.c:1464

│     │   ├─ 解析 <Partitions unit=X> → part_size 单位       axera_update.c:1487

│     │   ├─ 每分区: extract name / size → update_part_info  axera_update.c:1529/1549

│     │   └─ get_capacity_user()  (auto-resize=0xFFFFFFFF)  [F] axera_update.c:1402

│     │       └{NOR} uclass_get_device(UCLASS_SPI_FLASH)

│     │              → flash->size (16MB)                    axera_update.c:1443

│     ├─ (缺 spl/uboot 时) 补 SPL_MAX_SIZE / UBOOT_MAX_SIZE  axera_update.c:1766/1780

│     └─ get_part_image_name_xml(xml, part_name, file_name) [F] axera_update.c:1340

│         └─ 匹配 <Img><Block id="分区名"><File>xxx</File>   axera_update.c:1378/1389

│

├─ usb_stor_update_bin_check(pbin_info)                 [F] usb_storage_update.c:575

│   ├─ fat_exists(file_name)                                usb_storage_update.c:593

│   └─ fat_size(file_name) ≤ part_size                      usb_storage_update.c:599

│

├─ usb_stor_update_save_storage(pbin_info)              [F] usb_storage_update.c:751

│  ├─ update_parts_info(pheader)                        [F] axera_update.c:1619

│  │   └─ 依 storage_sel 选 BOOTARGS_* → 拼 mtdparts/bootargs → env_set

│  │      (NOR: BOOTARGS_SPINOR)                            axera_update.c:1661

│  │  【逐分区循环】

│  ├─ fat_size / fat_read_file(镜像 → pbuf, 10MB/块)        usb_storage_update.c:799/827

│  ├─ is_sparse_image(pbuf) ?                               usb_storage_update.c:834

│  │  ├─{sparse} sparse_info_init(&sparse, part_name)       usb_storage_update.c:844

│  │  │          common_get_part_info(part_name,&addr,&len) [F] axera_update.c:159  (L851)

│  │  │            └ switch(storage_sel) → {NOR} spi 分区表查询

│  │  │          common_raw_erase(part_name, addr, size)    [F] axera_update.c:1252 (L854)

│  │  │            └ switch(storage_sel) → {NOR} spi_nor_erase

│  │  │          write_sparse_img(&sparse,…, &point)        [F] sparse_img.c:100 (L863/904)

│  │  │            └ 按 chunk 解析稀疏格式, 跳全0块

│  │  └─{raw}   usb_stor_update_to_storage(&file, len_read) [F] usb_storage_update.c:399 (L892)

│  │            ├─ update_verify_image(part_name, pbuf)     [F] update_verify.c:22

│  │            │   └{安全镜像 & secure_boot}

│  │            │      public_key_verify()                   update_verify.c:42

│  │            │      cipher_aes_decrypto()                 update_verify.c:50

│  │            │      secure_verify()                       update_verify.c:59

│  │            │      cipher_aes_encrypto()                 update_verify.c:66

│  │            └─ switch(iram_misc_info->bootinfo.storage_sel)  usb_storage_update.c:411

│  │               ├─{EMMC} usb_stor_update_to_emmc()       [F] usb_storage_update.c:77

│  │               │        └ get_part_info / blk_dwrite (+hwpart 切换)

│  │               ├─{NAND} usb_stor_update_to_spinand()    [F] usb_storage_update.c:198

│  │               │        ├ usb_stor_spi_nand_protect_disable()   :157

│  │               │        ├ mtd_arg_off(MTD_DEV_TYPE_NAND)        :229

│  │               │        ├ mtd_erase (循环, 跳坏块)              :244

│  │               │        └ mtd_write                             :309

│  │               └─{NOR}  usb_stor_update_to_spinor()     [F] usb_storage_update.c:329

│  │                        ├ get_mtd_device_nm("nor0")            :348

│  │                        ├ uclass_get_device(UCLASS_SPI_FLASH)  :343

│  │                        ├ mtd_arg_off(…, MTDPARTS_SPINOR)      :357

│  │                        ├ mtd_erase (按 erasesize 整分区擦)     :365

│  │                        └ mtd_write (写镜像实际长度)            :386

│  └─ (循环尾) 若 sparse 未写完 → write_sparse_img(flush)    usb_storage_update.c:904

│

├─ env_set("usbupdate","finish") / env_save()               usb_storage_update.c:983/984

├─ set_reboot_mode_after_dl()                           [F] axera_update.c:1889

│   └─ switch(boot_type) → writel(TOP_CHIPMODE_GLB_SW_SET) 设复位启动源  axera_update.c:1895

└─ reboot()                                             [F] ax525.c:129

    └─ chip_rst_sw()  写 COMM_GLB_REG_ABORT_CFG → 芯片复位  ax525.c:114

  

失败路径：任一步 return 非0 → goto normal_boot → run_command_list("axera_boot")

                                                        usb_storage_update.c:993-996

```

  

> 关于 `storage_sel` 的来源（重要）：它**不是本命令探测**的，而是 **SPL 按启动 flash

> 类型写进共享 IRAM `0x3E00` 的 `struct boot_info`**，U-Boot 在 `get_dl_and_boot_info()`

> （`ax525.c`）读出。跳过 SPL 直接启动 U-Boot 时该区为 0(=EMMC)，需在 `magic` 无效时

> 按 `CONFIG_TARGET_AX525_*` 兜底填正确值。

  

---

  

## 五、三个核心抽象

  

### 抽象①：存储介质 = 一个枚举 `storage_sel`

  

`iram_misc_info->bootinfo.storage_sel`（`EMMC=0 / NAND=1 / NOR=2 / SD=3`）在**四处**分发：

  

| 关注点 | 函数 | file:line |

|---|---|---|

| 容量（auto-resize 用） | `get_capacity_user` | axera_update.c:1402 |

| 分区 offset/size | `common_get_part_info` | axera_update.c:159 |

| 擦除 | `common_raw_erase` | axera_update.c:1252 |

| 写入分发 | `usb_stor_update_to_storage` | usb_storage_update.c:399 |

  

值来自 SPL → IRAM(0x3E00) → U-Boot。这是“换介质上层零改动”的关键，也是“绕过 SPL 调试”踩坑的根源。

  

### 抽象②：分区模型 = XML 分区表 → mtdparts

  

分区信息经过两次“翻译”：

  

```

AX525_nor.xml <Partitions unit=2>        ← 源：人写的元数据

   │  get_part_info_rawdata (axera_update.c:1464)

   ▼

update_part_info 链表 {name, size×unit}  ← 内存模型

   │  update_parts_info (axera_update.c:1619)

   ▼

"spi3.0:64K(spl),384K(uboot),..."        ← mtdparts 字符串

   │  写进 env + bootargs（给 kernel）

   ▼

mtd_arg_off(part_name,...)               ← 写入时反查 offset/size (usb_storage_update.c:357)

```


- `unit`：`0`=1MB `1`=512KB `2`=1KB `3`=1B，决定 size 数值单位（日志 `unit val "2"` 即 1KB）。

- **auto-resize**：末分区 size 填 `0xFFFFFFFF` → `capacity_user - 已用`，自动吃满剩余（opt 分区）。

- 同一份 mtdparts 三处复用：写入定位 / env / kernel cmdline —— 保证 U-Boot 与 kernel 对分区布局认知一致。


### 抽象③：镜像模型 = raw / sparse + 可选验签

  

```

                    ┌─ is_sparse_image()?  (usb_storage_update.c:834)

 fat_read_file ─────┤

   (10MB/块)        ├─ YES → common_raw_erase 整分区擦

                    │        write_sparse_img 按 chunk 解析(跳全0块) → 省空间/时间

                    └─ NO  → update_verify_image(验签) → 介质 erase+write

  

 update_verify_image (update_verify.c:22):

   仅对 spl/uboot/atf/optee/dtb/kernel 等安全镜像,

   且开了 CONFIG_AXERA_SECURE_BOOT 时验公钥/解密; 否则直接放过。

```

---
## 六、NOR 落地细节

  

`usb_stor_update_to_spinor()`（usb_storage_update.c:329）：

  

```c

mtd = get_mtd_device_nm("nor0");                 // 按名字拿 NOR 的 mtd_info

mtd_arg_off(part_name,…,MTDPARTS_SPINOR,size);   // 从 mtdparts 解析该分区 offset/size

while (remaining) mtd_erase(mtd, &erase_op);     // 按 erasesize 对齐, 整分区擦净

mtd_write(mtd, off, wr_len, &retlen, pfile->pbuf); // 只写镜像实际长度

```

  

对应日志每分区两行 `erased @off=… size=分区大小` / `write @off=… size=镜像大小` ——

**擦整分区、写实际数据**，符合 NOR “先擦后写、擦除粒度为 sector” 的约束。

  

---

  

## 七、关键数据结构

  

```c

// 分区 + 镜像的内存模型 (axera_update.h:14)

struct update_part_info {

    char part_name[32];   // spl/uboot/env/kernel/rootfs/customer/opt

    u64  part_size;       // 分区容量(字节) = size × unit

    char file_name[48];   // U 盘镜像名 或 "none"(跳过)

    u64  image_size;      // 此路径未用, 靠 fat_size 实读

    struct update_part_info *next;

};

// SPL → U-Boot 的握手信息 (共享 IRAM 0x3E00, boot_info.h)

struct boot_info {

    unsigned int  magic;        // 0x12345678, 校验 SPL 是否写过

    boot_mode_e   mode;         // NORMAL / USB_UPDATE / SD_UPDATE ...

    dl_channel_e  dl_channel;

    storage_sel_e storage_sel;  // ★ 介质选择的唯一来源 ★

    boot_type_e   boot_type;    // 复位启动源

    unsigned char is_sd_boot;

};

```

  

---

