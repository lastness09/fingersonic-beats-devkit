# Fingersonic Beats Hardware and Third-Party Firmware Guide

Updated: 2026-10-19 · Platform: ESP32-S3 · Controls: 10 pressure-sensitive pads and one rotary encoder

Turn Beats into your own drum machine, sampler, sequencer, MIDI controller or wireless remote. Your App0 can define its own pad behavior, lighting, MIDI messages and audio engine. This documentation follows the interface-first approach of the [Guitar Developer Kit](https://github.com/lastness09/fingersonic-guitar-devkit), using the Beats hardware assignments.

**This repository contains development documentation only. It does not contain official firmware source or a ready-to-build manufacturer application.** Create your own application using public SDKs and libraries.

## Develop with an AI coding assistant

Share this repository URL with your assistant and use the following task brief, replacing the feature request with your own idea:

> Build an independent Fingersonic Beats App0 firmware project for the features I request. First read README.md, DEVELOPMENT.md, GPIO.md, RECOVERY.md and BUILD_VALIDATION.md from this developer kit. Use public SDKs and libraries; no official manufacturer firmware source is supplied or required. Target ESP32-S3, the published 16 MB partition layout and the 2 MiB App0 slot. Implement 10 shared-ADC pads, the single ENCODER2 knob, and the exact pad-to-LED map. Power, charging and battery-level management are hardware-managed and need no application implementation. Implement and document a way to select the existing App1 using the public OTA APIs before adding complex features. Respect the production-date recovery limits. Generate a complete new source project with pinned dependencies, matching build settings, a build command, an application-only BIN, documented controls and a validation checklist. Keep the installed App1, bootloader, partition table and security configuration intact. Identify any hardware assumptions or tests that still require a physical device. Do not claim the result is official firmware or that it has passed hardware tests unless those tests were actually performed.

The assistant should produce its own application code for the user. That does not require publishing the manufacturer's source or adding code to this documentation repository. Feature choices, audio engines and MIDI behavior can be designed independently; the hardware tables and recovery requirements remain the interface contract.

For a first application, request pad-to-USB/BLE-MIDI output with local LED feedback and a documented App1 gesture. Add a sampler, sequencer or I2S sound engine after verifying the input and recovery paths. If the assistant has no device access, it can check build results and static configuration but must leave ADC response, encoder behavior, LED positions, audio and recovery as explicit physical-device checks.

## Contents

- [1. Development boundary](#1-development-boundary)
- [2. Hardware and toolchain](#2-hardware-and-toolchain)
- [3. Arduino setup](#3-arduino-setup)
- [4. Partitions and application size](#4-partitions-and-application-size)
- [5. Hardware initialization](#5-hardware-initialization)
- [6. App1 entry and application design](#6-app1-entry-and-application-design)
- [7. Building, uploading and validation](#7-building-uploading-and-validation)
- [8. Troubleshooting](#8-troubleshooting)
- [9. Publishing your firmware](#9-publishing-your-firmware)

## 1. Development boundary

| Target | Developer operation |
| --- | --- |
| App0 | Build your own application BIN, then replace the main application through the supported Beats uploader |
| App1 | Enter and use the preinstalled uploader; preserve its firmware |
| Bootloader | Preserve the installed image and applicable production-batch recovery behavior |
| Partition table | Match the existing layout for compilation; preserve the device's partition table |
| Boot-selection records | Use public OTA APIs and the supported uploader; do not directly erase system records |
| Sound bank | The existing App1 uploader accepts `.sf2`; build any sampling instrument around an SF2-compatible engine |

Musical features and app compatibility are optional. You may use your own audio engine or implement only USB/BLE MIDI. Recovery access is essential, especially on earlier hardware. Read the [production-date recovery rules](RECOVERY.md) before installing a custom App0.

The existing uploader supports **`.bin` applications and `.sf2` sound banks only**. A synthesizer can create sounds procedurally without a sound bank. A sampler using App1 must load SF2 banks; prepare recorded samples inside an SF2 file before upload. WAV/MP3 files, ZIP archives, JSON resources and filesystem images are not accepted upload formats for this workflow.

An AI designing a sampler should choose an independent public SF2 engine or write its own compatible implementation, document supported SF2 features and presets, and validate bank size and memory use on the actual hardware. Do not assume the filesystem partition size is the usable SF2 size limit, invent a fixed uploaded filename or copy the manufacturer's synth/parser code. The exact uploader limit and bank-discovery contract must be confirmed before claiming end-to-end sampler compatibility.

## 2. Hardware and toolchain

| Component | Current project baseline |
| --- | --- |
| MCU | ESP32-S3 |
| Flash layout | 16 MB; 2 MiB App0 slot |
| PSRAM setting | OPI; confirm installed capacity for your hardware revision |
| Pads | 10 drive GPIOs sharing one ADC sense input |
| Encoder | One physical knob, named ENCODER2; GPIO1 / GPIO3 / GPIO38 |
| LEDs | GPIO2; GRB, 800 kHz; pad pixels 0–9 use a nonsequential scan-to-LED map |
| Audio | I2S output, 44.1 kHz stereo, 16-bit PCM |
| Power / charging / battery level | Managed independently by dedicated hardware; no App0 implementation required |
| Arduino ESP32 Core | 3.3.2, matching the inspected local development installation |

See the [complete GPIO and LED tables](GPIO.md). Hardware settings do not specify the complete bill of materials or guarantee every custom application has been tested on a device.

## 3. Arduino setup

Use Arduino IDE 2.x or Arduino CLI. Add the official Espressif board-manager URL:

`https://espressif.github.io/arduino-esp32/package_esp32_index.json`

Install the Espressif esp32 platform and select **ESP32S3 Dev Module**. For an initial build, match these settings to the existing product configuration:

| Setting | Value |
| --- | --- |
| ESP32 Core | 3.3.2 reference baseline |
| Flash Size | 16 MB |
| PSRAM | OPI PSRAM |
| Partition scheme | Custom, matching the published slot boundaries |
| USB Mode | USB-OTG (TinyUSB), if your application uses native USB MIDI |
| USB CDC On Boot | Disabled |
| USB Firmware MSC On Boot | Disabled |
| USB DFU On Boot | Disabled |

Create a new sketch for your own implementation. Configure its partition CSV to match the table below; keep the CSV alongside the sketch. Export the compiled application BIN. Only the application `.ino.bin` belongs in an App0 update, not a merged image, bootloader or partition image.

Choose public libraries according to your features:

| Feature | Public upstream |
| --- | --- |
| Arduino platform | [Espressif Arduino ESP32](https://github.com/espressif/arduino-esp32) |
| Command-line builds | [Arduino CLI](https://arduino.github.io/arduino-cli/) |
| Addressable LEDs | [Adafruit NeoPixel](https://github.com/adafruit/Adafruit_NeoPixel) |
| MIDI messages | [FortySevenEffects Arduino MIDI](https://github.com/FortySevenEffects/arduino_midi_library) |
| BLE MIDI | [Arduino BLE-MIDI](https://github.com/lathoub/Arduino-BLE-MIDI) |

Lock the versions you use and retain required third-party notices. Guitar-specific display and touch-controller libraries are not Beats requirements. For an ESP-IDF implementation, use its public GPIO, ADC, I2S and OTA APIs and validate the same hardware interfaces and slot boundaries.

For a reproducible Arduino CLI project, use the [complete installation commands, partition CSV and tested build recipe](BUILD_VALIDATION.md). With the 3.3.2 reference platform, the FQBN is `esp32:esp32:esp32s3:FlashSize=16M,PSRAM=opi,PartitionScheme=custom,USBMode=default,CDCOnBoot=default,MSCOnBoot=default,DFUOnBoot=default`. Check the resulting application file separately against the 2 MiB App0 limit; the platform's generic custom-partition maximum is not a sufficient capacity check. Review the recovery requirements before using an uploader.

## 4. Partitions and application size

| Partition | Type / subtype | Offset | Size | Current layout flag |
| --- | --- | --- | --- | --- |
| nvs | data / nvs | 0x9000 | 0x5000 | — |
| otadata | data / ota | 0xE000 | 0x2000 | — |
| app0 | app / ota_0 | 0x10000 | 0x200000 (2 MiB) | encrypted |
| app1 | app / ota_1 | 0x210000 | 0x100000 (1 MiB) | encrypted |
| spiffs | data / spiffs | 0x310000 | 0xBF0000 | encrypted |
| coredump | data / coredump | 0xF00000 | 0x10000 | — |

The current application mounts the `spiffs`-named data partition as LittleFS. Partition labels and filesystem implementations are separate concepts.

The App0 binary must fit within **2 MiB**. These boundaries describe the existing layout for application development; they are not serial flashing instructions. Actual production encryption and security settings must be confirmed for the device revision. Preserve App1, the bootloader, partition table and security settings. Do not erase all Flash, burn eFuses or substitute a Guitar image.

## 5. Hardware initialization

**Power, charging and battery-level management are handled by dedicated hardware independently of the application.** Custom App0 firmware does not need a battery ADC routine, charge-state GPIO polling, charging control or software power-latch maintenance. Do not copy those features from the Guitar guide: its power-related interfaces do not apply to Beats. This kit does not specify an application battery-telemetry API.

Initialize the recovery controls early. Then configure the encoder inputs, keep all pad-drive lines at high impedance, and initialize GPIO6 as the common ADC input without pulls. Initialize LEDs with the correct byte order and apply the [pad-to-LED mapping](GPIO.md#pad-to-led-mapping).

In scan order, pads map to physical pixels **8, 9, 5, 1, 2, 0, 3, 4, 7, 6**. Using scan index directly as the LED index will light the wrong pads. Check this mapping at low brightness before integrating MIDI follow lights or an audio engine.

Develop and validate your own pad baseline, noise rejection, velocity response and retrigger handling. Keep the scan loop responsive while servicing encoder input, MIDI and audio. This guide does not provide the official detection or calibration algorithm.

## 6. App1 entry and application design

Provide at least one clear operation that selects the preinstalled App1 and restarts. A physical gesture can remain available even if audio or sound-bank loading fails, provided App0 reaches the gesture handler. You may add a menu action or a Bluetooth command implemented by your own firmware and client.

Use `esp_partition_find_first` to locate `ESP_PARTITION_SUBTYPE_APP_OTA_1`, check `esp_ota_set_boot_partition` for success, and only then call `esp_restart`. See [the complete recovery sequence](RECOVERY.md#switching-to-the-preinstalled-app1).

Earlier devices require a running App0 to trigger this transition. Bluetooth and software gestures cannot rescue an App0 that cannot start. Future-batch forced entry is a separate manufacturer-specified feature; do not assume your software entry provides the same guarantee.

## 7. Building, uploading and validation

1. Confirm the production batch and verify that official firmware can enter App1.
2. Establish a minimal application on a development board or a test unit with a reliable recovery path.
3. Implement and test App1 entry before adding audio or complex application logic.
4. Verify the single knob, each pad and its mapped LED, then add MIDI or I2S output incrementally.
5. Export only your App0 application BIN, confirm it fits the slot, and use an uploader explicitly compatible with Beats.
6. Test several power cycles, USB enumeration, PSRAM behavior, pad response and App1 entry.
7. Restore a known compatible official Beats application using the supported process to verify recovery.

Find supported tools through [Official Beats resources](https://fingersonic.com/products/fingersonic-beats/). This guide does not assert that Beats App1 exposes the same HTTP endpoints, authentication or responses as Guitar. Do not copy Guitar upload requests without verifying compatibility.

A new independent test application was written from the public documentation and public SDK interfaces. See [build verification](BUILD_VALIDATION.md#build-audit-result) for its compilation results. Official firmware source was not used. No device flashing or physical-device validation was performed.

## 8. Troubleshooting

| Symptom | Check |
| --- | --- |
| Wrong pad lights | Use mapped physical LED indices rather than scan or internal key indices |
| All pad readings interfere | Only one drive line may be HIGH; all others should be high impedance |
| Encoder switch inverted | It is active LOW with pull-up |
| USB MIDI does not enumerate | Check native USB wiring, USB-OTG mode and your application's USB implementation |
| App0 is too large | Reduce it to fit the 2 MiB slot; do not expand into App1 |
| App1 operation does nothing | Check the target partition exists and boot-selection API calls succeed |
| Earlier hardware no longer boots App0 | Normal software recovery may be unavailable; contact official support |

## 9. Publishing your firmware

Describe the supported hardware revision, controls, build environment, application size and App1 entry. State the earlier-device bricking risk and identify tested recovery steps. Respect dependency licenses and distribute only files you are authorized to publish.

This official developer kit remains documentation only. It does not grant access to, or permission to redistribute, private manufacturer firmware source.

An AI-generated project should be delivered with a complete independent build, dependency versions and licenses, a partition-layout check, a 2 MiB size check, controls, the App1-entry operation and a clear list of tests actually run. The user chooses whether and how to distribute their own independent project; the manufacturer's private firmware remains outside that project.
