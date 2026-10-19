# Fingersonic Beats Developer Kit

Build a drum machine, sampler, sequencer, MIDI controller or wireless remote with Fingersonic Beats. This public kit describes the hardware interfaces, development environment and recovery requirements for your own App0 firmware.

**[Development guide](DEVELOPMENT.md)** · **[GPIO and LED mapping](GPIO.md)** · **[Build verification](BUILD_VALIDATION.md)** · **[Flashing and recovery](RECOVERY.md)**

**[Official Beats resources](https://fingersonic.com/products/fingersonic-beats/)** · **[Developer Kits](https://fingersonic.com/developers/)**

Beats has **10 pressure-sensitive drum pads and one physical rotary encoder**. The single encoder is named `ENCODER2` in the hardware configuration; this name does not indicate a second physical knob.

## Explore the developer kit

| Document | Content |
| --- | --- |
| [Development guide](DEVELOPMENT.md) | ESP32-S3 platform, Arduino setup, USB, partitions and development workflow |
| [GPIO reference](GPIO.md) | Encoder, shared ADC, pad drive pins, I2S and USB |
| [Pad-to-LED mapping](GPIO.md#pad-to-led-mapping) | Actual LED chain indices, pad identities and follow-light note mapping |
| [Public-link build verification](BUILD_VALIDATION.md) | Complete dependency commands, partition CSV, build recipe, artifact checks and independent audit results |
| [Flashing and recovery](RECOVERY.md) | Production-date rules, official recovery gesture and custom App1 entry points |

## Develop with an AI coding assistant

Give your AI assistant this repository URL and describe the instrument or controller you want to build. The [development guide](DEVELOPMENT.md#develop-with-an-ai-coding-assistant) includes a copyable task brief. The [build verification guide](BUILD_VALIDATION.md) supplies the complete dependency commands, partition CSV, compilation recipe and artifact checks. An assistant should read all five documents before generating a new application.

This is an interface specification for **independent firmware development**, not a release of the official firmware. The assistant can create a new source project using public SDKs and libraries without receiving, copying or reconstructing private manufacturer implementation code.

Power, charging and battery-level management are handled by dedicated hardware independently of App0. Your custom application does not need to implement those functions or maintain a software power latch.

Download the documentation with GitHub's **Code → Download ZIP**, or clone this repository. The kit contains documentation only. It does not include manufacturer firmware source, private processing algorithms, application binaries or system images.

User application development targets **App0 (2 MiB)**. Keep the preinstalled App1 uploader, bootloader, partition table and security configuration intact. Use files and tools explicitly intended for Beats; Guitar firmware and GPIO assignments are not interchangeable with Beats.

## Synthesizers and samplers

Create an original synthesizer, drum synthesizer, sequencer, MIDI controller or other instrument. A synthesizer can generate its own sound and does not need an SF2 bank.

**The existing App1 uploader accepts only application `.bin` and sound-bank `.sf2` files.** A sampling instrument using this upload workflow must use **SF2** as its sample-bank format. Convert or package your own samples into SF2 before uploading. Do not design a workflow that depends on uploading standalone WAV/MP3 files, ZIP archives or filesystem images through App1. This transport restriction does not require the private official sound engine; an independent application can use its own SF2-compatible engine.

## Read before installing custom firmware

| Production date | Entry to flashing mode | If App0 cannot run |
| --- | --- | --- |
| After January 1, 2027 | Hold down the single knob on the right while powering on | Forced entry remains available, according to the manufacturer's batch specification |
| Before January 1, 2027 | A running App0 must select App1 and restart | Software entry is unavailable; an unbootable custom App0 can brick the device |

On earlier devices running official firmware, **hold the first pad on the left (HH OPEN), then quickly press and release the single knob five times within approximately two seconds**. This gesture is handled by App0, so custom firmware must implement its own entry if it needs to retain that operation.

**Developers assume the risk of installing custom firmware on earlier devices. If your custom App0 cannot start, the device may become a brick that you cannot recover through normal user operations.** A Bluetooth command, menu or software button combination only works while App0 can execute the code handling it.

Always preserve and test an App1 entry in your firmware, such as a physical gesture, menu action or your own Bluetooth command. See the [recovery guide](RECOVERY.md#provide-an-app1-entry-in-your-firmware) for the public OTA API sequence and its limits.

The manufacturer-supplied batch rule says **after** January 1, 2027. Devices manufactured on January 1 itself need separate hardware-revision confirmation. Production date, rather than purchase date or firmware-update date, determines the applicable batch. Future-batch recovery behavior has not been tested as part of this documentation release.

## Support and related projects

- [Fingersonic Lab](https://fingersonic.com/)
- [Official Beats resources](https://fingersonic.com/products/fingersonic-beats/)
- [Fingersonic Guitar Developer Kit](https://github.com/lastness09/fingersonic-guitar-devkit)
- [Support](mailto:support@fingersonic.net)

Hardware tables describe interfaces for independent implementations; no manufacturer source code is required. See the [public-link build audit](BUILD_VALIDATION.md#build-audit-result) for the independent compilation result and its limits. Device flashing and physical-device tests were not performed.
