# AX8860 / AX525 新人指引：代码拉取、编译及 HAPS 烧录

## 1. 文档说明

本文用于指导新人完成 AX8860、AX525 项目的以下操作：

1. 在线查看项目代码
2. 拉取代码
3. 编译 Kernel 和 Boot Wrapper
4. 准备并映射烧录产物
5. 预约 HAPS 调试环境
6. 下载 Bitfile
7. 使用 Trace32 加载内核并进行调试

作为**外设驱动组**，重点关注以下目录：

```text
kernel/linux
build
```

---

## 2. 在线查看代码

各项目代码可通过以下地址在线查看：

```text
http://10.126.12.106:8080/source/xref/
```

---

## 3. 拉取代码

> 建议 AX8860 和 AX525 分别创建独立的代码目录，避免不同项目代码相互影响。

### 3.1 拉取 AX525 代码

```bash
mkdir -p ~/ax525
cd ~/ax525

repo init \
    -u git@git-ext.axera-tech.com:megsoc-bsp/manifest.git \
    -b ax525_trunk \
    --no-clone-bundle

repo sync -c --no-tags
repo forall -c git lfs pull
repo start ax525_trunk --all
```

也可以合并执行：

```bash
repo init -u git@git-ext.axera-tech.com:megsoc-bsp/manifest.git -b ax525_trunk --no-clone-bundle
repo sync -c --no-tags && repo forall -c git lfs pull && repo start ax525_trunk --all
```

### 3.2 拉取 AX8860 代码

```bash
mkdir -p ~/ax8860
cd ~/ax8860

repo init \
    -u git@git-ext.axera-tech.com:megsoc-bsp/manifest.git \
    -b ax8860_trunk \
    --no-clone-bundle

repo sync -c --no-tags
repo forall -c git lfs pull
repo start ax8860_trunk --all
```

也可以合并执行：

```bash
repo init -u git@git-ext.axera-tech.com:megsoc-bsp/manifest.git -b ax8860_trunk --no-clone-bundle
repo sync -c --no-tags && repo forall -c git lfs pull && repo start ax8860_trunk --all
```

### 3.3 重点目录

代码拉取完成后，外设驱动开发主要关注：

```text
<代码根目录>/kernel/linux
<代码根目录>/build
```

---

## 4. 编译

### 4.1 查看支持的编译项目

进入 Kernel 目录：

```bash
cd <代码根目录>/kernel/linux
```

执行：

```bash
make plist
```

该命令会列出当前代码支持的编译项目，即可用的 `p=` 参数。

常见示例：

```text
AX8860_haps
AX525_emmc
```

> 实际支持的平台请以 `make plist` 输出为准。

---

### 4.2 编译 Kernel

#### AX8860 HAPS 平台

```bash
cd <AX8860代码根目录>/kernel/linux
make p=AX8860_haps all install -j32
```

#### AX525 eMMC 平台

```bash
cd <AX525代码根目录>/kernel/linux
make p=AX525_emmc all install -j32
```

其他平台只需替换 `p=` 后的项目名称：

```bash
make p=<项目名称> all install -j32
```

参数说明：

| 参数         | 说明                       |
| ---------- | ------------------------ |
| `p=<项目名称>` | 指定目标平台                   |
| `all`      | 执行完整编译                   |
| `install`  | 将产物安装到输出目录               |
| `-j32`     | 使用 32 个并行任务编译，可根据服务器资源调整 |
|            |                          |

---

### 4.3 编译 Boot Wrapper

进入 Boot Wrapper 目录：

```bash
cd <代码根目录>/boot/boot-wrapper
```

使用与 Kernel 相同的平台参数进行编译。

例如 AX8860 HAPS：

```bash
make p=AX8860_haps all install -j32
```

例如 AX525 eMMC：

```bash
make p=AX525_emmc all install -j32
```

---

### 4.4 检查编译产物

编译完成后，会在代码根目录的 `build` 目录下生成 `out` 目录：

```text
<代码根目录>/build/out/
```

AX8860 HAPS 的常用产物目录示例：

```text
<代码根目录>/build/out/AX8860_haps_glibc/images/
```

其中 Trace32 启动脚本通常位于：

```text
<代码根目录>/build/out/AX8860_haps_glibc/images/cmm/
```

常用启动脚本：

```text
start_kernel_ax8860.cmm
```

> 不同平台、不同 libc 配置下，实际目录名称可能不同，请以 `build/out/` 下生成的内容为准。

到此，代码编译阶段结束。

---

## 5. 烧录前准备

### 5.1 为什么需要复制编译产物

开发代码通常位于个人目录，例如：

```text
/home/<用户名>/ax8860/
/home/<用户名>/ax525/
```

但调试使用的是远程 Windows 跳板机。当前服务器只对以下目录开放共享权限：

```text
/home/public/
```

个人开发目录 `/home/<用户名>/` 无法直接映射到 Windows 跳板机。

因此，需要将编译产物复制到 `/home/public/` 下的个人目录，再在跳板机上挂载该共享目录。

---

### 5.2 创建个人共享目录

建议在 `/home/public/` 下创建以用户名命名的目录：

```bash
mkdir -p /home/public/<用户名>
```

建议进一步区分项目及版本：

```bash
mkdir -p /home/public/<用户名>/ax8860
mkdir -p /home/public/<用户名>/ax525
```

---

### 5.3 复制编译产物

#### AX8860 示例

```bash
cp -a <AX8860代码根目录>/build/out/AX8860_haps_glibc \
    /home/public/<用户名>/ax8860/
```

也可以复制整个 `out` 目录：

```bash
cp -a <AX8860代码根目录>/build/out \
    /home/public/<用户名>/ax8860/
```

#### AX525 示例

```bash
cp -a <AX525代码根目录>/build/out \
    /home/public/<用户名>/ax525/
```

复制完成后，检查文件是否完整：

```bash
ls -lh /home/public/<用户名>/ax8860/
```

> 建议每次复制到带日期或版本号的目录，避免覆盖之前可用的产物。例如：
>
> ```text
> /home/public/<用户名>/ax8860/2025xxxx/
> ```

---

## 6. 预约烧录机器

### 6.1 预约入口

HAPS 机器预约地址：

```text
https://cmdb.aixin-chip.com/crs/
```

### 6.2 预约注意事项

- 每次最多预约 **1 小时**
- 当前预约时间段结束后，才能发起新的预约
- 请根据项目选择对应的 AX8860 或 AX525 资源
- 预约前确认所选机器是否为 BSP 专用资源
- 请记录预约信息中的：
  - 远程 Windows 跳板机 IP
  - 登录用户名和密码
  - HAPS 主板/子板资源
  - 串口信息
  - 其他资源备注

### 6.3 资源分配表

AX8860 / AX525 的 BSP 专用机器、远程跳板机 IP、密码及 HAPS 子板资源等信息，请查看以下 Wiki：

```text
https://wiki.aixin-chip.com/pages/viewpage.action?pageId=224112078
```

```text
https://wiki.aixin-chip.com/pages/viewpage.action?pageId=224125767
```

> 具体使用哪一个页面，请根据项目和当前资源分配情况确认。

---

## 7. 连接远程跳板机

### 7.1 准备 VNC 软件

通过软件中心安装或加载 VNC 客户端。

### 7.2 连接跳板机

在预约时间段内，根据预约信息填写：

- 跳板机 IP
- 用户名
- 密码

通过 VNC 连接到对应的 Windows 跳板机。

> 只能在有效预约时间段内使用对应机器。建议提前准备好代码产物，避免占用预约时间进行编译或大文件复制。

---

## 8. 映射编译产物到 Windows

在远程 Windows 跳板机上，将 Linux 服务器的共享目录映射为网络驱动器。

需要映射的 Linux 目录为：

```text
/home/public/<用户名>/
```

映射完成后，应能在 Windows 资源管理器中看到编译产物，例如：

```text
AX8860_haps_glibc/images/
```

重点确认以下内容可正常访问：

```text
images/
images/cmm/
images/cmm/start_kernel_ax8860.cmm
```

> 网络驱动器的具体地址、共享名及认证方式请参考团队现有配置。如果不清楚，可咨询同组同事或机器资源维护人员。

---

## 9. 使用 HAPS 软件加载 Bitfile

### 9.1 打开 HAPS 软件

在远程 Windows 跳板机上打开对应的 HAPS 管理软件。

如预约资源中指定了 HAPS 编号，例如 HAPS 100，请确认当前连接和操作的是对应设备，避免影响其他使用者。

### 9.2 选择 Bitfile

在 HAPS 软件中，通过 **Bitfile 路径**选择对应项目的 Bitfile 文件或目录。

> 如果不清楚 Bitfile 的位置或应该选择哪个版本，请咨询朱学亮。

### 9.3 下载 Bitfile

选择 Bitfile 后，点击：

```text
Load All
```

等待下载完成。

当执行状态显示：

```text
就绪
```

说明 Bitfile 下载完成。

> 加载过程中不要关闭 HAPS 软件、断开 VNC 或执行复位操作。

---

## 10. HAPS 复位

如果调试过程中需要复位 HAPS，点击：

```text
Reset HAPS
```

等待执行状态重新显示：

```text
就绪
```

此时表示 HAPS 复位完成。

复位后，通常需要根据实际情况重新通过 Trace32 加载并启动内核。

---

## 11. 使用 Trace32 连接 HAPS

### 11.1 确认编译产物

首先确认编译产物已经挂载到 Windows 路径，或者已完整复制到 Windows 本地。

AX8860 HAPS 的目录示例：

```text
build/out/AX8860_haps_glibc/images/
```

需要重点确认启动脚本存在：

```text
images/cmm/start_kernel_ax8860.cmm
```

### 11.2 打开 Trace32

在远程 Windows 跳板机上打开 Trace32 软件。

### 11.3 运行启动脚本

在 Trace32 中选择：

```text
Run Script
```

然后运行：

```text
images/cmm/start_kernel_ax8860.cmm
```

等待脚本执行并完成内核加载。

> AX525 或其他平台应使用对应平台的 `.cmm` 脚本，具体文件名请以 `images/cmm/` 目录中的实际文件为准。

---

## 12. 串口调试

待 Trace32 完成内核加载并启动后，打开对应串口工具。

如果内核正常启动，应能在串口窗口中看到启动日志。进入系统后，即可通过串口输入 Linux 命令完成调试，例如：

```bash
uname -a
```

```bash
dmesg
```

```bash
lsmod
```

```bash
cat /proc/interrupts
```

```bash
devmem <address>
```

外设驱动常用日志查看方式：

```bash
dmesg -w
```

如果需要过滤特定驱动日志：

```bash
dmesg | grep -i <关键字>
```

---

## 13. 完整操作流程速查

```text
1. 选择 AX8860 或 AX525 项目
        ↓
2. repo init / repo sync 拉取代码
        ↓
3. 进入 kernel/linux 执行 make plist
        ↓
4. 编译 kernel/linux
        ↓
5. 编译 boot/boot-wrapper
        ↓
6. 检查 build/out 下的编译产物
        ↓
7. 将产物复制到 /home/public/<用户名>/
        ↓
8. 在 CRS 系统预约 HAPS 机器
        ↓
9. 通过 VNC 连接预约的 Windows 跳板机
        ↓
10. 映射 /home/public/<用户名>/ 网络目录
        ↓
11. 打开 HAPS 软件，选择并 Load All 下载 Bitfile
        ↓
12. 等待 HAPS 状态显示“就绪”
        ↓
13. 打开 Trace32
        ↓
14. 运行 images/cmm 下对应的启动脚本
        ↓
15. 等待内核加载
        ↓
16. 通过串口查看日志并进行调试
```

---

## 14. 常见问题

### 14.1 `make plist` 中没有目标平台

检查以下事项：

1. 当前是否位于正确的 `kernel/linux` 目录
2. 是否拉取了正确的项目分支
3. `repo sync` 是否完整成功
4. Git LFS 文件是否拉取成功

可重新执行：

```bash
repo sync -c --no-tags
repo forall -c git lfs pull
```

---

### 14.2 找不到 `build/out` 或产物目录

检查：

1. Kernel 是否编译成功
2. Boot Wrapper 是否编译成功
3. 编译命令是否带有 `install`
4. 是否使用了正确的 `p=` 参数
5. 编译日志中是否存在错误

示例：

```bash
make p=AX8860_haps all install -j32
```

---

### 14.3 Windows 无法访问个人开发目录

这是正常现象。当前只开放：

```text
/home/public/
```

请先将编译产物复制到：

```text
/home/public/<用户名>/
```

然后再从 Windows 跳板机映射该目录。

---

### 14.4 HAPS 加载 Bitfile 失败

建议检查：

1. 是否选择了正确项目、正确版本的 Bitfile
2. 是否操作了预约分配给自己的 HAPS 设备
3. HAPS 软件是否连接到目标设备
4. 当前设备是否被其他人占用
5. Bitfile 路径是否可访问
6. 是否需要先执行 `Reset HAPS`

如果仍无法确认 Bitfile，请咨询朱学亮。

---

### 14.5 Trace32 脚本执行失败

建议检查：

1. HAPS Bitfile 是否已经加载完成
2. HAPS 状态是否为“就绪”
3. 是否选择了对应芯片和平台的 `.cmm` 脚本
4. `images` 目录下的文件是否复制完整
5. 网络驱动器是否断开
6. 脚本中引用的文件路径是否有效
7. 是否需要先复位 HAPS 后再重新执行脚本

---

### 14.6 内核加载完成但串口没有输出

建议检查：

1. 串口工具是否连接到正确的 COM 口
2. 串口波特率等参数是否正确
3. Trace32 是否真正执行到启动阶段
4. HAPS 是否发生异常或被复位
5. 当前 Kernel、Boot Wrapper 和 Bitfile 是否匹配
6. 是否使用了对应项目的 `images` 产物

---

## 15. 新人首次操作建议

首次操作建议请熟悉流程的同事协助，重点确认以下匹配关系：

| 项目 | 需要保持一致的内容 |
|---|---|
| 芯片项目 | AX8860 或 AX525 |
| 编译平台 | 如 `AX8860_haps`、`AX525_emmc` |
| Bitfile | 与芯片及 HAPS 环境匹配 |
| Trace32 脚本 | 与芯片平台匹配 |
| 编译产物 | Kernel、Boot Wrapper 来自同一次或兼容版本编译 |
| HAPS 设备 | 与预约分配的设备一致 |

建议在预约前完成：

- 代码拉取
- Kernel 编译
- Boot Wrapper 编译
- 编译产物复制
- Bitfile 路径确认
- Trace32 脚本确认

这样可以将有限的 1 小时预约时间主要用于 HAPS 加载和实际调试。