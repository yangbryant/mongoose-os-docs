# mos 工具说明

## 用 `--port` 选项

`mos` 工具用 `--port` 参数指定端口连接设备, 默认设置为 `auto`, 这意味着 `mos` 自动检测设备的串口端口. 你也可以手动指定端口值. 它可以是串口设备, 例如 在Windows下 `--port COM3`, 或者在Linux下 `--port /dev/ttyUSB0`.
也可以设置 `--port` 为网络节点, 而不是串口端口. 设备监听 串口, Websocket, 和 MQTT传输的命令(除非被禁用). 因此, 通过 Websocket `--port ws://IP_ADDR/rpc` 方式或者通过 MQTT协议 `--port mqtt://MQTT_SERVER/DEVICE_ID/rpc` 方式连接远程设备. 这能够使 `mos` 工具用作远程设备管理工具.

## 使用环境变量设置默认选项值

通过环境变量 `MOS_FLAGNAME` 可以覆盖 `mos` 参数的默认值. 例如, 在 Mac/Linux 上导入 `MOS_PORT` 变量, 将其放在 `~/.profile` 文件中, 这样可以设置 `--port` 参数的默认值.

```
export MOS_PORT=YOUR_SERIAL_PORT  # E.g. /dev/ttyUSB0
```

## 开发板布局

在某些情况下, 例如, 如果你用 ESP8266 架构模块, 而不是开发板, 你需要在刷固件和开机引导状态时操作额外的切换模块的步骤.下表提供一个汇总:

| Platform           | Wiring Notes                                           |
| ------------------ |--------------------------------------------------------|
| bare bones ESP8266 |  flash via UART:  `GPIO15 LOW, GPIO0 LOW, GPIO2 HIGH`<br> boot from flash: `GPIO15 LOW, GPIO0 HIGH, GPIO2 HIGH`<br> boot from SD: `GPIO15 HIGH` |
| bare bones ESP32 |  flash via UART:  `GPIO0 LOW`<br> boot from flash: `GPIO0 HIGH`|
| CC3200 launchpad   | connect J8 to SOP2 (see [guide](http://energia.nu/cc3200guide/))  |

## 版本控制

`mos` 工具能通过 Web UI 或者命令行 `mos update` 自行更新. `mos` 工具版本也影响固件构建: 固件构建时用到的库与 `mos` 版本对应. 有如下三种方式保持更新:

- 固定到特定版本, 例如 `mos update 1.18`. 这是最稳定的方法, 在这种情况下不会有任何改变.
- 固定到"release"渠道, `mos update release`. 这是默认的方法吗一般 1-2 周更新一次.
- 固定到"latest"渠道, `mos update latest`. 获取到最新的更新, 但有时会遇到破坏.
