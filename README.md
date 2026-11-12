# Hamulight L2547-1 / L2564 – Radio Protocol and ESPHome Integration

**Reverse engineering of the 433 MHz radio protocol, control of multiple installations, and integration into Home Assistant**

**Author:** Atilla Kati  
**Documentation date:** November 11, 2026  
**Status:** Tested with transmission, reception, ON/OFF state tracking, and dimming

## 1. Summary

The original Hamulight **L2547-1** remote control communicates wirelessly with the **L2564** LED driver. The proprietary protocol was analyzed using a Nooelec NESDR SMArt v5 and Universal Radio Hacker (URH), then successfully reproduced using a **diymore ESP32 LoRa V4.3 (ESP32-S3)** with an external **EBYTE CC1101**. The nine existing 12 V LED spotlights and the LED driver remain unchanged.

**Successfully tested on the first installation:**

- Switching ON and OFF via ESPHome/CC1101.
- Dimming up and down.
- Cycling through the four brightness levels **25%, 50%, 75%, and 100%**.
- Receiving and decoding the original remote control's ON/OFF radio telegrams.
- Updating the ON/OFF state in ESPHome and Home Assistant when the original remote control is used.

**Extension to two installations:** The latest comparative recording using two L2547-1 remote controls shows the same frame structure and ON command, but **different device-specific identifier fields**. The transmitter and decoder have been prepared conceptually, and in a partial YAML configuration, to handle two separate installations.

Neither **genuine status acknowledgements from the LED driver** nor an **absolute brightness value transmitted over the air** have been demonstrated so far. The displayed switching state reflects the last command received or sent locally, not a state acknowledged by the driver.

## 2. Hardware and Wiring

| CC1101 | Function | ESP32-S3 |
|---|---|---|
| GND | Ground | GND |
| VCC | 3.3 V | 3V3 |
| GDO0 | TX data line | GPIO4 |
| CSN | SPI chip select | GPIO41 |
| SCK | SPI clock | GPIO38 |
| MOSI | SPI MOSI | GPIO39 |
| MISO | SPI MISO | GPIO40 |
| GDO2 | RX data line | GPIO6 |

### Radio and SDR Parameters

| Parameter | Determined or configured value |
|---|---|
| CC1101 carrier frequency | 433.881 MHz (ESPHome configuration) |
| Modulation | ASK/OOK |
| Symbol encoding | NRZ, MSB first |
| Symbol rate | Approximately 30,000 symbols/s |
| Symbol duration | Approximately 33.3 µs |
| Receiver filter bandwidth | 203 kHz |
| TX/RX switching | `cc1101.begin_tx` and `cc1101.begin_rx` |
| SDR recording | URH, center frequency 433.500 MHz, 2 MS/s, ASK/NRZ, approximately 67 samples/symbol |

The CC1101 operates in **half-duplex** mode. It cannot receive and transmit simultaneously using the same radio transceiver.

```yaml
cc1101:
  spi_id: led_spi
  cs_pin: GPIO41
  frequency: 433.881MHz
  modulation_type: ASK/OOK
  symbol_rate: 30000
  filter_bandwidth: 203kHz
  rx_attenuation: 0dB
  output_power: 10
```

Configuration of the receiver and transmitter:

```yaml
remote_receiver:
  pin: GPIO6
  dump: []
  filter: 10us
  idle: 5ms
  use_dma: true
  rmt_symbols: 1024
  receive_symbols: 1024
  buffer_size: 16kB
  on_raw:
    then:
      - script.execute:
          id: hamulight_receive
          # On ESP32-S3, x is std::vector<long>; int[] expects std::vector<int>.
          raw: !lambda 'return std::vector<int>(x.begin(), x.end());'

remote_transmitter:
  id: hamulight_tx
  pin: GPIO4
  carrier_duty_percent: 100%

  on_transmit:
    then:
      - logger.log:
          format: "HAMULIGHT: CC1101 wechselt in TX"
          level: WARN
          tag: hamulight
      - cc1101.begin_tx

  on_complete:
    then:
      - cc1101.begin_rx
      - logger.log:
          format: "HAMULIGHT: CC1101 zurueck in RX"
          level: WARN
          tag: hamulight
```

## 3. Signal Analysis and Timing

The following two figures are taken from the uploaded measurement documentation. The IQ amplitudes shown are relative and have not been calibrated for signal power.

![Figure 1: Repeated radio telegrams from the original remote control](images/signal_01.png)

*Fig. 1 – Original remote control: telegrams repeated over time.*

![Figure 2: Reconstructed transmission using ESP32 and CC1101](images/signal_02.png)

*Fig. 2 – Reconstructed transmission using ESP32/CC1101.*

| Characteristic | Observation |
|---|---|
| Complete usable bit sequence | 208 bits |
| Telegram duration | Approximately 6.93 ms |
| Interval between telegram starts | Approximately 8.03 ms |
| Telegrams per command group | 8 identical repetitions |
| ON/OFF button held down | New group approximately every 167 ms, with incremented counter |

Earlier verification tests independently confirmed the transmission timing using synthetic 400 µs and 33 µs pulses.

## 4. Reconstructed Telegram Format

### 4.1 Frame

| Bit position | Length | Content |
|---|---:|---|
| 0–79 | 80 bits | Preamble `0xAA` × 10 |
| 80–95 | 16 bits | Synchronization word `0x2D 0xD4` |
| 96–207 | 112 bits | 14 masked payload bytes (MSB first) |

### 4.2 Abstract Payload Structure

The **14 bytes before XOR masking** have the following structure in the analyzed messages:

```text
[HEADER_0] [HEADER_1] [COUNTER_BASE]
[DEVICE_ID_HIGH] [DEVICE_ID_LOW]
[PROTOCOL_FIELD] [COMMAND]
[RESERVED_0] [RESERVED_1] [RESERVED_2]
[RESERVED_3] [RESERVED_4] [RESERVED_5]
[CHECKSUM]
```

The non-device-specific constants in the basic format are:

- `HEADER_0 = 0xAA`, `HEADER_1 = 0x02`, `COUNTER_BASE = 0x00`.
- `PROTOCOL_FIELD = 0x01`; the six reserved bytes are `0x00` in the telegrams examined.
- The **two `DEVICE_ID_*` bytes** distinguish the installations. Their actual values are intentionally omitted from this documentation.
- The checksum byte is calculated as a bitwise XOR of the **first 13 unmasked payload bytes**. The checksum therefore **also depends on the respective device identifier**; a fixed formula such as `CRC = constant XOR COMMAND` would be incorrect.
- All **14 data bytes, including the checksum**, are then XOR-masked byte by byte using the same **8-bit telegram counter**. The counter can be derived from the third transmitted payload byte because its original, unmasked value is `0x00`.

The semantic interpretation was derived by comparing URH recordings and through practical tests; it is **not** based on a published manufacturer specification.

### 4.3 Decoded Commands

| Command | Function | Validation on installation 1 |
|---|---|---|
| `0x01` | Switch ON | Successfully tested |
| `0x02` | Switch OFF | Successfully tested |
| `0x10` | Next brightness level | Successfully tested |
| `0x20` | Dim up | Successfully tested |
| `0x21` | Dim down | Successfully tested |

**Operating behavior:** The driver must first be switched ON; only then do dimming commands and brightness-level changes take effect. The middle button on the remote control always transmits the same level-change command, rather than four distinct commands for absolute brightness levels.

## 5. Comparison of Two Remote Controls and Installations

The additional URH recording `Hamulight_2xOn_2xL2547-1.complex16s`, together with its associated `.proto` file, contains **four short ON button presses from two remote controls**. Eight identical telegrams were observed for each button press.

| Characteristic | Installation A | Installation B |
|---|---|---|
| Radio frame | Same 208-bit format | Same 208-bit format |
| Preamble and sync | Same | Same |
| ON command | `0x01` | `0x01` |
| Device identifier | Distinct, not disclosed | Distinct, not disclosed |
| Counter | Increments independently for each remote | Increments independently for each remote |
| Checksum | Identifier-dependent, valid XOR format | Identifier-dependent, valid XOR format |
| Practical ESP32 transmission test | Successful | Pending |

The analysis suggests that one shared CC1101 can address both installations using their identifier fields.

## 6. ESPHome Architecture for Two Installations

### 6.1 Transmission

The existing `hamulight_send` script accepts the parameters `device`, `command`, and `groups`:

```yaml
script:
  - id: hamulight_send
    mode: queued
    parameters:
      device: int
      command: int
      groups: int
    # The generator and transmission actions remain part of the local ESPHome configuration.
```

At runtime, the **locally configured device identifier** corresponding to the selected installation is inserted. The encoder constructs a valid 208-bit frame, calculates the identifier-dependent XOR checksum, masks the payload with the counter, and transmits each group eight times.

Separate counter variables are planned for the two installations (`hamulight_counter_1` and `hamulight_counter_2`). The existing shared transmission module is serialized using a queued execution mode (`mode: queued`). During dimming, several groups are transmitted with incrementing counters at intervals on the order of 167 ms.

### 6.2 Reception

`remote_receiver.on_raw` passes the received RMT pulse durations to `hamulight_receive`. An explicit vector type conversion was required on the ESP32-S3 used:

```yaml
remote_receiver:
  # Existing pin/RMT configuration
  on_raw:
    then:
      - script.execute:
          id: hamulight_receive
          raw: !lambda |-
            return std::vector<int>(x.begin(), x.end());
```

The reception script:

1. Converts the pulse durations into NRZ bits.
2. Searches for the preamble and synchronization word.
3. Reads the 14 payload bytes and removes the counter-based XOR mask.
4. Checks the basic format, reserved bytes, and XOR checksum.
5. Compares the **locally stored device identifier** with installation A or installation B.
6. Maps valid ON/OFF commands to the corresponding ESPHome state.

**Important:** Received telegrams are reflected in the state only by calling `publish_state()`; they must **not** be retransmitted. Otherwise, feedback loops and unnecessary radio traffic may occur.

### 6.3 State and Home Assistant

| Logical component | Installation A | Installation B |
|---|---|---|
| Transmission counter | `hamulight_counter_1` | `hamulight_counter_2` |
| ON/OFF state | `hamulight_power_1` | `hamulight_power_2` |
| Native light entity | `hamulight_light_1` | `hamulight_light_2` |
| Dim up/down | Separate actions | Separate actions |
| Next level | Separate button | Separate button |

For Home Assistant, a **native binary ESPHome light entity** without a brightness slider is appropriate for each installation. The Dim Up, Dim Down, and Next Level functions remain separate actions. The UI should hide or disable these actions when the light is off; the ESP32 should also avoid triggering dimming or level-change commands for lights that are off.

![Figure 3: Lights off](images/UI_Light_Off.png)

![Figure 4: Lights on](images/UI_Light_On.png)

The ESP32 tracks the binary light state based on **received or locally triggered ON/OFF commands**. **The LED driver does not actively acknowledge these commands.** Other transmitters, packet loss, radio collisions, power failures, or operation during a CC1101 transmission can cause the displayed state to become inconsistent with the actual state.

## 7. Validation and Open Issues

| Test item | Status |
|---|---|
| Installation A: ON/OFF via ESP32/CC1101 | **Confirmed** |
| Installation A: Dim up/down via ESP32 | **Confirmed** |
| Installation A: Cyclic brightness-level changes | **Confirmed** |
| Installation A: Original remote control decoded by RX | **Confirmed** |
| Installation A: ON/OFF state tracking in HA | **Confirmed** |
| Second remote control: protocol/identifier differences in SDR | **Analyzed** |
| Two installations: separate mapping of transmitter and decoder in ESPHome code | **Prepared; not yet fully tested in practice** |
| Installation B: ESP32 ON/OFF at LED driver | **Test pending** |
| Installation B: Dimming and level changes | **Test pending** |
| Native light entities for both installations operating together in HA | **Not yet confirmed as a complete system** |
| Actual driver feedback or absolute brightness | **Not demonstrated** |

### Further Investigation

- Counter rollover at `0xFF` and behavior after restart or power loss.
- Counter behavior when the ESP32 and both original remote controls are used concurrently.
- Unambiguous assignment of the physical remote controls and LED drivers during the later installation B test.
- Behavior when radio commands overlap in time, and radio range at the final installation location.
- No actual brightness feedback from the L2564 is known. The four levels are managed cyclically by the driver rather than being transmitted as absolute target values.

## 8. Measurement Data and Reproducibility

| Recording | Purpose |
|---|---|
| `hamulight_3x_on_01.complex16s`, `hamulight_3x_off_01.complex16s` | Initial ON/OFF analysis |
| `hamulight_OnOffOnOffOnOff.complex16s` and `.proto` | Alternating ON/OFF commands |
| `Testsignal_2.complex16s` | Timing reference/pulse test |
| `Testsignal_CustomOnOff.complex16s`, `Custom_OnOff.proto` | ON/OFF frames generated by the ESP32 |
| `Dimmtest_ON.complex16s` and `.proto` | Long ON button press / dim up |
| `Dimmtest_OFF.complex16s` and `.proto` | Long OFF button press / dim down |
| `Dimmtest_ON-OFF_short.complex16s` and `.proto` | Comparison of short button presses |
| `Hamulight_4_Stufen.complex16s` and `.proto` | Cyclic brightness levels |
| `Hamulight_2xOn_2xL2547-1.complex16s` and `.proto` | Comparison of two original remote controls |

## 9. AI Assistance

This project was developed and documented with assistance from **ChatGPT (OpenAI)**.

AI assistance was used primarily to analyze and interpret the captured radio telegrams, reconstruct the radio protocol, and implement the ESPHome components.

Hardware integration, measurement activities, practical functional tests, and validation of the results were carried out by the author.

## 10. Origin and Limitations

This documentation describes **reverse-engineering findings** and is not an official Hamulight protocol specification. The demonstrated operational results currently apply explicitly to installation A only.

---

**Author:** Atilla Kati  
**Devices:** Hamulight L2547-1 / L2564 · ESP32-S3 · CC1101 · ESPHome · Home Assistant  
**As of:** November 11, 2026
