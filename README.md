# Fingersonic Beats Developer Kit

Fingersonic Beats 公开开发指南。仅公开 GPIO、独立固件开发环境搭建方法及升级恢复注意事项，不提供官方固件源码、内部算法或系统镜像。

Beats 是 **10 个力度感应鼓盘、一个旋钮**的产品；唯一旋钮在内部工程中命名为 `ENCODER2`，这个名称不表示产品有两个旋钮。

## 开始开发

| 文档 | 内容 |
| --- | --- |
| [GPIO 与硬件接口](GPIO.md) | 唯一旋钮、10 路 ADC 激励、公共 ADC、LED 和 I2S |
| [环境搭建与独立固件](DEVELOPMENT.md) | Arduino、内存、USB、分区和开发步骤 |
| [烧录模式与恢复风险](RECOVERY.md) | 新旧生产批次、App0/App1 和自定义恢复入口 |

## 烧录模式：请先确认生产日期

- **2027 年 1 月 1 日之后生产的设备**：按住右侧唯一旋钮开机，可以强制进入烧录模式，即使 App0 无法运行。此规则按生产批次判断，不按购买时间、升级时间或当前日期判断。
- **此前生产的设备**：必须由正在运行的 App0 调用烧录模式。官方固件操作为：按住左边第一个按键（HH OPEN），在约 2 秒内快速连续按下并松开唯一旋钮 5 次。
- **旧设备风险**：如果你的自定义 App0 无法启动，就无法通过 App0 调用 App1，设备可能变砖。开发和刷入自定义固件的风险由开发者自行承担。不要把蓝牙入口当成 App0 崩溃后的硬件恢复保证。
- **2027 年 1 月 1 日当天生产的设备**：这里的批次规则未明确涵盖当天，请先向官方确认硬件版本，不应自行假设具有强制恢复功能。

建议自定义固件始终保留至少一种进入 App1 的办法，例如蓝牙调用、按键组合或菜单操作，并在更改音频和应用逻辑前先验证它。详见[恢复指南](RECOVERY.md)。

## 相关链接

- [Fingersonic 官网](https://fingersonic.com/)
- [Fingersonic Guitar Developer Kit](https://github.com/lastness09/fingersonic-guitar-devkit)
- [Beats Developer Kit](https://github.com/lastness09/fingersonic-beats-devkit)

Guitar 的开发资料可用于理解整体开发流程，但 Guitar 固件、GPIO 和二进制文件不可作为 Beats 固件使用。本文档不附带官方恢复固件下载；恢复文件必须使用官方明确标注适配 Beats 硬件的版本。

本指南中的 GPIO 来自当前 Beats 工程配置，生产批次恢复规则由厂商提供。尚未完成本文独立固件方案的实机验证。
