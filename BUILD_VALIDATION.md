# Public-Link Development and Reproducible Builds

[Developer kit](README.md) · [Hardware guide](DEVELOPMENT.md) · [GPIO and LEDs](GPIO.md) · [Recovery](RECOVERY.md)

This page closes the gap between a hardware description and a buildable independent application. Give an AI coding assistant the repository URL, ask it to read all five documents, and describe the application you want. The assistant must create a new project; this documentation repository does not contain manufacturer firmware source or an application binary.

## What the public interface is sufficient for

An independent application can implement the 10 shared-ADC pads, mapped LED feedback, the single encoder, USB MIDI, basic I2S audio output and software selection of the existing App1. A minimal controller does not need the manufacturer's sound engine, private headers, sound banks, USB descriptors or calibration code.

The product can support original synthesizers as well as sampling instruments. **App1 accepts only application `.bin` and sound-bank `.sf2` files.** Procedural synthesis needs no sample bank. A sampler using the existing uploader must use SF2; raw WAV/MP3 samples must be packaged into SF2 first. Do not depend on ZIP or filesystem-image uploads. The audit's independent tone generator does not test an SF2 parser or bank upload.

Power, charging and battery-level management remain independent hardware functions. Do not generate application battery-management or power-latch code.

Compile verification does not establish correct sensing thresholds, velocity response, physical LED locations, USB enumeration, DAC output or recovery on an actual production device. Preserve that distinction in any AI-generated result.

## Create a complete independent project

Ask the assistant to deliver a directory such as:

```text
MyBeats/
  MyBeats.ino
  partitions.csv
  README.md
```

The primary sketch filename must match its directory name. Additional headers and source files are the independent application's own implementation. Do not request files from the official firmware project.

The application's README should describe its controls, chosen MIDI mapping, App1 entry, dependency versions, build command, output filename and hardware tests still required. For earlier devices, state the risk of an unbootable App0 explicitly.

## Install public build dependencies

Install [Arduino CLI](https://arduino.github.io/arduino-cli/1.5/installation/) and make `arduino-cli` available in your terminal. The audit used CLI 1.5.1, ESP32 Core 3.3.2 and Adafruit NeoPixel 1.15.1. The Core and library versions below are pinned; they are a tested baseline, not a request to install the newest platform.

Run these commands individually and stop if any command fails:

```text
arduino-cli version
arduino-cli core update-index --additional-urls https://espressif.github.io/arduino-esp32/package_esp32_index.json
arduino-cli core install esp32:esp32@3.3.2 --additional-urls https://espressif.github.io/arduino-esp32/package_esp32_index.json
arduino-cli lib update-index
arduino-cli lib install "Adafruit NeoPixel@1.15.1"
arduino-cli core list
arduino-cli lib list
```

For the minimal USB MIDI / I2S target, these are the relevant public headers:

| Interface | Header | Provider |
| --- | --- | --- |
| GPIO / ADC / encoder polling | `Arduino.h` | ESP32 Core |
| Addressable LEDs | `Adafruit_NeoPixel.h` | Adafruit NeoPixel 1.15.1 |
| Native USB device | `USB.h` | ESP32 Core |
| Native USB MIDI | `USBMIDI.h` | ESP32 Core |
| I2S output | `ESP_I2S.h` | ESP32 Core |
| Partition lookup | `esp_partition.h` | Core's bundled ESP-IDF |
| App1 boot selection | `esp_ota_ops.h` | Core's bundled ESP-IDF |

Core 3.3.2 supplies native USB MIDI and I2S APIs. A minimal application using those APIs does not require a private USB layer, external MIDI library, BLE stack, display library or touch-controller library. Install additional public dependencies only if your own features require them. BLE and SF2 applications need their own dependency and compatibility validation.

## Use the complete partition CSV

Place the following **layout data** in `MyBeats/partitions.csv`. Keep the offsets, sizes, types and flags unchanged for this layout:

```csv
# Name,Type,SubType,Offset,Size,Flags
nvs,data,nvs,0x9000,0x5000,
otadata,data,ota,0xE000,0x2000,
app0,app,ota_0,0x10000,0x200000,encrypted
app1,app,ota_1,0x210000,0x100000,encrypted
spiffs,data,spiffs,0x310000,0xBF0000,encrypted
coredump,data,coredump,0xF00000,0x10000,
```

This CSV is a compilation input, not a request to flash a new partition table. Preserve the device's installed App1, bootloader, partition table and security configuration. The `encrypted` flag describes the existing layout; it does not instruct users to burn eFuses or encrypt application files manually.

## Compile without a connected device

From the directory containing `MyBeats`, run:

```text
arduino-cli compile --fqbn "esp32:esp32:esp32s3:FlashSize=16M,PSRAM=opi,PartitionScheme=custom,USBMode=default,CDCOnBoot=default,MSCOnBoot=default,DFUOnBoot=default" --warnings all --build-path ./build --output-dir ./out ./MyBeats
```

The exact long option string is intentional: it selects ESP32-S3, 16 MB Flash, OPI PSRAM, the custom partition CSV and TinyUSB with automatic CDC/MSC/DFU disabled. These option names refer to ESP32 Core 3.3.2. Refer to [Arduino CLI compile options](https://arduino.github.io/arduino-cli/1.5/commands/arduino-cli_compile/) if using a different CLI release.

Compilation requires no attached device. This command builds files and does not upload them. Use an isolated sketchbook/library configuration if auditing that your implementation has no dependency on another local firmware project.

On Windows, the CLI may be bundled with Arduino IDE even if it is not on PATH. A tool-extraction error involving a DLL or temporary directory is a host-environment error; use a writable temporary directory and the normal permissions required by the official toolchain. Do not change the device's Flash security settings to resolve a host build error.

## Identify and check the application artifact

For the project name above, the App0 candidate is **`out/MyBeats.ino.bin`**. Do not substitute `.ino.merged.bin`, `.ino.bootloader.bin`, `.ino.partitions.bin`, an ELF, a MAP file or a filesystem image.

**Check the file itself against 2,097,152 bytes (0x200000).** The platform's generic custom-partition maximum or an overall 16 MB Flash figure is not a sufficient App0 capacity check. A successful compiler exit does not remove this requirement.

If Python 3 is available, run:

```text
python -c "from pathlib import Path; p=Path('out/MyBeats.ino.bin'); n=p.stat().st_size; assert 0 < n <= 0x200000, n; print('Application bytes:', n)"
```

Also inspect `build/partitions.csv` or decode the generated partition binary to confirm App0 is at 0x10000 with size 0x200000 and App1 is at 0x210000 with size 0x100000. Record the application SHA-256 and exact toolchain versions with your build.

## Implementation details an AI must choose explicitly

| Area | Public interface contract | Independent implementation / device validation |
| --- | --- | --- |
| ADC | GPIO6 is shared; only the selected drive line is HIGH; all others are high impedance | ADC resolution/attenuation, settling time, idle baseline, pressure polarity, noise rejection, trigger/release thresholds, velocity curve and retrigger behavior |
| LED | GPIO2, GRB / 800 kHz; scan-to-pixel map is 8,9,5,1,2,0,3,4,7,6 | Low initial brightness and a one-pixel-at-a-time physical check; keep unused pixels off |
| Encoder | GPIO1 / GPIO3 / GPIO38, switch active LOW | Debouncing, detent direction, click behavior and a documented recovery operation |
| USB MIDI | Native USB, TinyUSB mode, public `USBMIDI` API | Define the MIDI note/channel mapping; initialize the MIDI interface before starting USB; verify host enumeration and messages |
| I2S | BCLK40, WS42, DOUT41; 44.1 kHz / stereo / 16-bit | Public `I2SClass::setPins` order is BCLK, WS, DOUT, DIN, MCLK; unused input and MCLK may be disabled for this three-signal test; verify DAC format and actual output on hardware |
| App1 | Locate existing ota_1; set boot partition; restart only after success | An early-running control handler, stop output, handle missing/invalid App1 without rebooting, and physically test recovery |

Initial sensing constants in an AI-generated test application are **test assumptions**, not factory calibration. Do not invent official ADC thresholds, a factory velocity algorithm or undocumented Bluetooth commands. If a feature needs additional product-specific hardware or protocol information, identify the missing specification rather than silently inventing it.

## Build-audit result

On 2026-10-19, an independent application was written using only the public documentation snapshot at commit `b2535ee65a5e5982ee64c9c05fe0a06ac74e3976` and public SDK/library interfaces. Hardware constants and the partition CSV were generated from those public tables. No official firmware project source was read for this experiment or linked into the application.

| Check | Observed result |
| --- | --- |
| Build environment | Windows; Arduino CLI 1.5.1; ESP32 Core 3.3.2; Adafruit NeoPixel 1.15.1 |
| Independently implemented features | Ten-pad ADC scanning, mapped LEDs, single encoder, USB MIDI, a simple procedural I2S tone generator and App1-selection operation |
| Compile / link | Passed, exit code 0 |
| Application BIN | 392,944 bytes; below the 2,097,152-byte App0 limit |
| Image target / integrity | ESP32-S3 chip ID 9; appended image SHA-256 verified |
| Generated partitions | App0 0x10000 / 0x200000; App1 0x210000 / 0x100000; all six layout entries present |
| Dependency audit | 107 generated dependency files checked; no dependencies on either private Fingersonic product project |
| LED library provenance | Seven relevant source/header files matched a freshly downloaded official NeoPixel 1.15.1 package |
| Warnings | Upstream Core / NeoPixel warnings were present with all warnings enabled; the build was not warning-free |
| Physical device / flashing | Not performed |
| BLE / SF2 sampling or upload | Not tested; the test synthesizer generates tones without a sample bank |

Application SHA-256 for the local audit artifact:

`260d9d240b92d91a34325efda79e23a05cdda7c05df41ad045add3ff5991c80d`

This is evidence that the public interface is sufficient to create and compile a new application with the tested scope. It is not a ready-to-flash product release or proof of physical compatibility. The experimental application source, BIN, ELF and system images remain local and are not published in this repository.

The audit exposed documentation gaps now addressed on this page: exact dependency commands, a complete partition CSV, built-in USB/I2S header selection, the complete build command, application-artifact identification, a separate 2 MiB check and explicit separation of interface facts from ADC calibration assumptions. The uploader's BIN/SF2-only constraint is now explicit as well.

Before claiming a working hardware release, test pad idle/press readings and polarity, thresholds and retrigger behavior, each physical LED, encoder direction, USB enumeration, I2S output, and App1 entry on the intended hardware batch. SF2 applications also require a verified uploader size limit, uploaded-bank location/discovery contract, preset mapping and a compatible independent SF2 engine. Those details were not inferred from private source or established by this compilation test.
