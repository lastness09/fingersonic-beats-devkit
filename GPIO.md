# Fingersonic Beats GPIO and LED Mapping

[Developer kit](README.md) · [Development guide](DEVELOPMENT.md) · [Flashing and recovery](RECOVERY.md)

GPIO numbers below are ESP32-S3 signal numbers, not connector positions or package pin numbers. Beats has one physical knob and 10 pressure-sensitive pads. Legacy definitions for other controls in the internal project do not describe extra controls on this product.

## Power and battery management

Power, charging and battery-level management are handled by dedicated hardware independently of App0. No application battery sampling, charge-state polling, charging control or power-hold GPIO implementation is required. Do not import the Guitar product's power and battery GPIO assignments into Beats firmware. An application battery-telemetry API is not specified by this kit.

## Single rotary encoder

The only physical encoder is named `ENCODER2` in the hardware configuration.

| Signal | GPIO | Direction and behavior |
| --- | ---: | --- |
| CLK / A | 1 | Encoder input with pull-up |
| DT / B | 3 | Encoder input with pull-up |
| SW / push switch | 38 | Pull-up input; pressing connects to ground, active LOW |

The encoder common terminal and switch common terminal connect to GND. GPIO3 is also a strapping pin on ESP32-S3; preserve the original board wiring and consult the chip documentation before changing the surrounding circuit.

## Shared ADC pad interface

The common sense input is **GPIO6 (ADC1_CH5)**. Scan the 10 drive lines one at a time. Drive only the selected line HIGH, keep all other drive lines as high-impedance inputs, sample the common ADC input after settling, then return the selected line to high impedance. Configure GPIO6 as an input without pull-up or pull-down.

These are not 10 independent ADC inputs. A scan index, an internal key index, an LED chain index and a MIDI note number identify different things.

| Scan index | Pad | Drive GPIO | Internal key index | Physical LED chain index | Follow-light MIDI note |
| ---: | --- | ---: | ---: | ---: | ---: |
| 0 | Open Hi-Hat / HH OPEN, first pad on the left | 4 | 10 | 8 | 46 |
| 1 | Closed Hi-Hat | 7 | 6 | 9 | 42 |
| 2 | Crash | 8 | 13 | 5 | 49 |
| 3 | Snare | 9 | 2 | 1 | 38 |
| 4 | Tom1 | 10 | 11 | 2 | 48 |
| 5 | Kick | 12 | 0 | 0 | 36 |
| 6 | Tom2 | 14 | 9 | 3 | 45 |
| 7 | Floor Tom | 15 | 7 | 4 | 43 |
| 8 | Ride | 17 | 17 | 7 | 53 |
| 9 | Crash2 | 18 | 21 | 6 | 57 |

The note column describes the current follow-light mapping and pad identity, not every possible performance mode's MIDI output. Tom1 uses note 48; the current follow-light receiver also accepts legacy note 47 for that pad. Custom firmware may choose its own output mapping.

## Pad-to-LED mapping

**The pad scan order is not the LED wire order.** In scan order 0–9, the LED chain indices are **8, 9, 5, 1, 2, 0, 3, 4, 7, 6**. All LED indices are zero-based: index 0 is the first addressable pixel on the data chain, not a printed pad number.

| Physical LED chain index | Pad | Scan index | Internal key index |
| ---: | --- | ---: | ---: |
| 0 | Kick | 5 | 0 |
| 1 | Snare | 3 | 2 |
| 2 | Tom1 | 4 | 11 |
| 3 | Tom2 | 6 | 9 |
| 4 | Floor Tom | 7 | 7 |
| 5 | Crash | 2 | 13 |
| 6 | Crash2 | 9 | 21 |
| 7 | Ride | 8 | 17 |
| 8 | Open Hi-Hat | 0 | 10 |
| 9 | Closed Hi-Hat | 1 | 6 |

For example, scan index 0 reads HH OPEN through drive GPIO4 and common ADC GPIO6. Its internal key index is 10, but its light is **pixel 8**, not pixel 0 or pixel 10. Apply this mapping consistently to local pad feedback, MIDI follow lights and level displays. If you maintain a per-pixel color table, use the mapped physical LED index to select the pad's color.

| LED interface setting | Current configuration |
| --- | --- |
| Data GPIO | 2 |
| Protocol | NeoPixel / WS2812-compatible, 800 kHz |
| Byte order | GRB |
| Driver buffer length | 24 pixels |
| Pad feedback indices | 0–9, using the mapping above |

The current driver allocates 24 pixels, while drum-pad feedback uses indices 0–9. Current pad-oriented displays clear indices 10–23. The buffer length is not evidence of 24 visible pad LEDs or 24 playing pads. Keep unused indices off during initial testing.

## I2S audio and USB

| Signal | GPIO | Direction / connection |
| --- | ---: | --- |
| I2S BCLK | 40 | Bit-clock output to DAC |
| I2S DATA OUT | 41 | MCU output to DAC data input |
| I2S LRCLK / WS | 42 | Word-clock output to DAC |
| USB D− | 19 | ESP32-S3 native USB |
| USB D+ | 20 | ESP32-S3 native USB |

The current audio configuration is 44.1 kHz, stereo, 16-bit PCM, with no I2S data input. Preserve the original USB wiring and do not repurpose USB pins as ordinary GPIO when using USB.

## Bring-up recommendations

1. Start with all drive lines at high impedance and all LEDs off.
2. Verify the single encoder's rotation and active-LOW switch.
3. At low brightness, light pixels 0–9 individually and check them against the LED table.
4. Scan one pad at a time, then verify every pad's ADC-to-LED pairing.
5. Add your own baseline tracking, noise rejection, velocity curve and retrigger handling before integrating audio or MIDI.

GPIO uses 3.3 V logic; do not apply 5 V to an ESP32-S3 input. Consult component specifications and the actual PCB revision before modifying electrical connections. A signal omitted from this table is not automatically available for expansion. This guide publishes the interface mapping, not the official sensing or calibration implementation.
