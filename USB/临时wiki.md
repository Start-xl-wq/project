  

| 分组 | 接口 | 作用 |

|---|---|---|

| 注册/注销 | `ax_usb_xfer_device_register_client` | 注册回调（收 frame + 链路状态），全局单 client |

| 注册/注销 | `ax_usb_xfer_device_unregister_client` | 注销回调，返回后保证回调不再触发 |

| 回调 | `struct ax_usb_xfer_device_client_ops` | 业务层实现：`frame_received` / `link_changed` |

| 大块数据（异步 SG） | `ax_usb_xfer_device_submit` | D2H 发送：Device 向 Host 提交一笔 source SG |

| 大块数据（异步 SG） | `ax_usb_xfer_device_recv` | H2D 接收：为 Host 来的数据预投一块 dest SG |

| 小消息（同步拷贝） | `ax_usb_xfer_device_send_frame` | 发 CTRL/DBG/copy DATA 帧，返回即完成拷贝 |

| 状态查询 | `ax_usb_xfer_device_is_online` | 查询当前链路是否在线 |