## 1. 什么是 USB 虚拟网卡

USB 虚拟网卡指设备通过一根 USB 线，在主机上模拟出一块以太网卡。主机和设备之间可以像普通网线一样配 IP、ping、传文件，常用于调试、固件升级和板卡与 PC 之间的数据传输。

设备端（例如 AX8860）以 USB Device 身份工作，通过 Linux 的 USB gadget 框架提供网卡功能；主机端（Windows、Linux、macOS）加载对应驱动后，就会多出一块网络适配器。

实现 USB 虚拟网卡的协议有多种，最常见的是 **RNDIS** 和 **NCM**。

## 2. RNDIS

RNDIS（Remote Network Driver Interface Specification）由微软定义，本质是把 Windows 网卡驱动接口 NDIS 的消息搬到 USB 上传输。

- **出身**：微软私有协议，不是 USB 标准。
- **兼容性**：Windows 很早就内置了 RNDIS 驱动，老系统也能用；Linux 也有现成驱动；macOS 原生不支持。
- **传输方式**：每个以太网帧前面加一个 RNDIS 消息头，通常一次 USB 传输只装一个帧。
- **连接状态**：主机通过查询消息获取设备的连接状态和速率。
- **现状**：实现简单、兼容性好，但效率一般。微软已不再推荐在新产品中使用。

## 3. NCM

NCM（Network Control Model）是 USB-IF 制定的 CDC（通信设备类）标准协议，可以看作 CDC ECM 的升级版。

- **出身**：USB 官方标准，各操作系统按标准实现驱动。
- **兼容性**：Linux 和 macOS 原生支持；Windows 11 内置驱动；Windows 10 有驱动但官方支持不完善，可能需要手动选择驱动。
- **传输方式**：把多个以太网帧打包成一个 NTB（NCM Transfer Block），一次 USB 传输携带多个帧，减少 USB 事务和中断次数。
- **连接状态**：设备主动通过中断端点向主机发送“已连接/断开”和速率通知。
- **现状**：效率更高，是目前推荐的方案。

## 4. 对比

| 对比项 | RNDIS | NCM |
|---|---|---|
| 协议归属 | 微软私有 | USB-IF 标准（CDC） |
| Windows | 老版本起即内置 | Windows 11 内置；Windows 10 支持有限 |
| Linux | 支持 | 支持 |
| macOS | 原生不支持 | 支持 |
| 数据封装 | 每帧加消息头，通常一次传输一帧 | 多帧打包成 NTB，一次传输多帧 |
| 传输效率 | 一般，小包多时开销大 | 较高，适合大流量 |
| 连接状态 | 主机主动查询 | 设备主动上报 |
| 推荐程度 | 兼容老 Windows 时使用 | 新项目首选 |

## 5. 怎么选

- **主机以 Windows 11、Linux 或 macOS 为主**：优先选 NCM，吞吐更高，也是标准协议。
- **需要兼容老版本 Windows、追求即插即用**：可以选 RNDIS。
- **两者都要**：设备端可以分别提供 NCM 和 RNDIS，按主机环境切换使用。

需要注意，同一个 USB 控制器同一时刻只能运行一种 gadget 功能，切换前要先停掉当前功能。

## 6. 在 AX8860 上使用

AX8860_emmc 工程同时提供了 NCM 和 RNDIS 两种虚拟网卡，通过启动脚本切换：

```sh
/soc/scripts/usb-acm.sh stop      # 先停掉默认的 ACM 串口
/soc/scripts/usb-ncm.sh start     # 或 /soc/scripts/usb-rndis.sh start
```

启动后板端会出现 `ncm0` 或 `rndis0` 网卡，配置 IP 后即可与主机通信。NCM 实测吞吐见同目录下的《AX8860 USB3虚拟网卡NCM吞吐测试》。

## 7. 小结

RNDIS 胜在兼容老 Windows，NCM 胜在标准化和传输效率。新项目建议以 NCM 为主，有老 Windows 兼容需求时再提供 RNDIS。
