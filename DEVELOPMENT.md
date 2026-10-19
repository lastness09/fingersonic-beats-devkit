# 独立固件开发与环境搭建

[返回目录](README.md) · [GPIO](GPIO.md) · [恢复与 App1](RECOVERY.md)

本仓库只包含指南，不含可直接编译的官方工程。开发者应新建自己的应用，使用公开 SDK 和库实现功能；无需、也不能从此仓库获取官方固件源码。

## 硬件与工具基线

- MCU：ESP32-S3。
- 现有项目设置：16 MB Flash、OPI PSRAM；具体 PSRAM 容量须按实际硬件确认。
- Arduino IDE 2.x 或 Arduino CLI；也可自行使用 ESP-IDF。
- 现有本地开发环境安装的 Arduino ESP32 Core 为 3.3.2。它是参考基线，不是对所有自定义固件的兼容性保证。
- USB MIDI 使用 USB-OTG（TinyUSB），USB CDC On Boot、USB Firmware MSC On Boot、USB DFU On Boot 均关闭。
- LED 可选用 Adafruit NeoPixel；MIDI、蓝牙、音频等库按自己需要选择。Guitar 特有的触摸和显示屏依赖不应照搬到 Beats。

公开上游：[Arduino ESP32](https://github.com/espressif/arduino-esp32)、[Arduino CLI](https://arduino.github.io/arduino-cli/)、[Adafruit NeoPixel](https://github.com/adafruit/Adafruit_NeoPixel)、[Arduino MIDI](https://github.com/FortySevenEffects/arduino_midi_library)、[Arduino BLE-MIDI](https://github.com/lathoub/Arduino-BLE-MIDI)。锁定所选版本并遵守相应许可证。

在 Arduino 的附加开发板管理器网址中配置官方索引：

`https://espressif.github.io/arduino-esp32/package_esp32_index.json`

安装 Espressif 的 esp32 平台，选择 ESP32S3 Dev Module，核对 Flash、PSRAM、USB 和自定义分区设置。导出时只使用自己的应用 `.ino.bin`，不要将 merged 镜像、bootloader 或 partition 镜像作为 App0 上传。

## 现有分区边界（用于理解与编译）

| 分区 | 起始地址 | 大小 | 用途 |
| --- | --- | --- | --- |
| nvs | 0x9000 | 0x5000 | 系统配置 |
| otadata | 0xE000 | 0x2000 | 启动选择数据 |
| app0 / ota_0 | 0x10000 | 0x200000（2 MiB） | 主应用、自定义固件所在槽 |
| app1 / ota_1 | 0x210000 | 0x100000（1 MiB） | 预装上传与恢复应用 |
| spiffs | 0x310000 | 0xBF0000 | 文件分区，现有应用使用 LittleFS |
| coredump | 0xF00000 | 0x10000 | 崩溃转储 |

App0 成品必须适合 2 MiB 槽位。编译时按这些边界设置自己的分区布局，并保持与原设备一致；这些地址不是让用户通过串口写入整机镜像的指令。现有工程将 app0、app1 和文件分区标记为 encrypted，实际量产设备安全配置须由官方核验。

用户开发以替换 App0 为范围，保留原装 App1、bootloader、分区表及安全配置。不要擦除整片 Flash、烧写 eFuse 或尝试绕过加密。Guitar 的镜像与用户提供的 Guitar `.bin` 均不能作为 Beats 恢复镜像。

## 推荐开发顺序

1. 确认生产日期、现有官方固件可进入 App1，记录当前官方恢复流程。
2. 在开发板或有可靠恢复入口的测试机上建立最小应用，先实现并测试 App1 切换入口。
3. 接入唯一旋钮和低亮度 LED，再逐路接入 ADC 鼓盘；按 GPIO 表测试。
4. 添加 USB MIDI、BLE MIDI 或 I2S 音频，每增加一个功能重新检查恢复入口和启动行为。
5. 检查二进制大小、启动稳定性、PSRAM、USB 枚举和各鼓盘响应，再通过适配 Beats 的官方上传工具写入 App0。
6. 在多次重启后复测 App1 入口，并用适配 Beats 的官方文件验证恢复。

当前指南没有核实 Beats App1 的 HTTP 上传端点、鉴权和响应格式；不得把 Guitar 上传接口直接认定为 Beats 的接口。使用官网明确标注适配 Beats 的上传流程，未确认时先向官方咨询。

本次是文档发布，未编译自定义固件，也未执行设备烧录或实机测试。
