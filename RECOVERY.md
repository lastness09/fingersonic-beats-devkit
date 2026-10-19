# Fingersonic Beats Flashing and Recovery

[Developer kit](README.md) · [Development guide](DEVELOPMENT.md) · [GPIO and LED mapping](GPIO.md)

## Production-date recovery rules

| Production batch | Entry to flashing mode | If App0 cannot run |
| --- | --- | --- |
| After January 1, 2027 | With the device off, hold down the single knob on the right and power on | Forced entry is available according to the manufacturer's batch specification |
| Before January 1, 2027 | Running App0 must select App1 and restart | App0-dependent entry is unavailable; custom firmware can brick the device |

The future-batch behavior above is a manufacturer-provided specification, not a hardware test performed for this release. The stated rule uses **after** January 1, 2027; confirm the revision of any device manufactured on January 1 itself. Purchase date and firmware-update date do not determine the production batch.

On earlier devices running official firmware, **hold the first pad on the left (HH OPEN), then quickly press and release the single knob five times within approximately two seconds**. App0 handles this gesture, selects App1 and restarts. It is not an application-independent hardware recovery gesture.

**If custom App0 firmware cannot start on an earlier device, the device may become a brick that cannot be recovered through normal user operations. The developer assumes the risk of installing custom firmware.** Do not promise that Bluetooth, USB or a hidden software gesture can recover an application that cannot run.

## Provide an App1 entry in your firmware

Preserve at least one physical operation and optionally add a Bluetooth or menu entry. Initialize it early enough that it does not depend on sound-bank loading, audio tasks or the full user interface.

Possible designs include:

- The official-style gesture: hold HH OPEN and press the knob five times.
- A startup combination checked during the earliest App0 initialization stage.
- An **Enter update mode** menu action with a confirmation step.
- A Bluetooth command defined and implemented by your own firmware and client.

A Bluetooth command here is a **custom firmware design suggestion**, not a published official upgrade protocol. Both the application and its client must agree on the command, connection checks, acknowledgment and protection against accidental activation. Do not claim that the official app implements it or reuse an unconfirmed official MIDI CC.

All software entry points require App0 to execute their handlers. Early initialization improves availability when later features fail, but it does not provide recovery from an App0 that cannot start.

## Switching to the preinstalled App1

Use Espressif's public OTA APIs. The operation selects an existing application; it does not install or replace App1.

1. Locate the existing application partition with `esp_partition_find_first`, type `ESP_PARTITION_TYPE_APP`, subtype `ESP_PARTITION_SUBTYPE_APP_OTA_1`.
2. If App1 is not found, report an error. Do not change boot selection or restart.
3. Notify the user, stop active notes and audio output, and complete necessary state saves.
4. Call `esp_ota_set_boot_partition` with the App1 partition.
5. Check the return value. Only call `esp_restart` after `ESP_OK`; otherwise report the failure and keep the current application running.

Reference: [Espressif ESP32-S3 OTA API](https://docs.espressif.com/projects/esp-idf/en/stable/esp32s3/api-reference/system/ota.html). The device's existing security policy must allow the installed App1 to boot.

## Return to official Beats firmware

Use [Official Beats resources](https://fingersonic.com/products/fingersonic-beats/) to obtain a tool and recovery file explicitly intended for the installed Beats hardware. Enter the preinstalled App1 using an operation supported by your current firmware and hardware batch, then follow that tool's instructions.

Do not use Guitar application binaries, merged images, bootloaders or partition images as Beats recovery files. The developer kit does not distribute an official Beats firmware binary or private source.

## Before installing custom firmware

- Confirm the production batch and the available recovery path.
- Verify the application is a Beats App0 build and fits within its 2 MiB slot.
- Test the transition from your minimal custom App0 to the existing App1 before adding complex features.
- Obtain a compatible official recovery file and supported uploader in advance.
- Preserve App1, the bootloader, partition table and device security settings; do not erase all Flash.
- On earlier devices, accept that an application-startup failure may leave no normal user recovery path.
