# Recon: RD-R7150 (Shimano 105 Di2, 12-speed)

Sources: nRF Connect on iOS and the ESP32 firmware logs, November 2026.
Device-specific identifiers (MAC, serial number) are redacted.

## Advertising
| Field | Value |
|-------|-------|
| Local Name | `RDR7150 xxx` (suffix = last 3 characters of the serial number) |
| Manufacturer Data | `4a 04 10 00` → company ID `0x044A` (little-endian, Shimano) + payload `10 00` |
| Service UUIDs | `000018FF-5348-494D-414E-4F5F424C4500`, `000018EF-5348-494D-414E-4F5F424C4500` |
| Connectable | yes |
| PHY | primary 1M (legacy advertising) |
| MAC | random static (two most significant bits = `11`); stable so far |
| RSSI | -70 dBm (phone), -77…-90 dBm (ESP32 a few metres away) |

Notes:
- UUID base `xxxxxxxx-5348-494D-414E-4F5F424C4500` is ASCII "SHIMANO_BLE".
- The manufacturer payload is only 2 bytes (`10 00`) and stayed constant at 80 %, so it is
  not the battery level. Possibly flags/state.
- A Linux laptop (`bluetoothctl`) did not see the RD at all in 4 scans, while the phone and
  the ESP32 did. Cause unknown.

## GATT (connection WITHOUT pairing works)
| Service | Characteristics | Notes |
|---------|-----------------|-------|
| `180A` Device Information | `2A29` Manufacturer, `2A25` Serial, `2A26` FW revision | readable |
| **`180F` Battery Service** | **`2A19` Battery Level** (has CCCD → notify) | readable, 1 byte, % |
| `18EF` (Shimano) | `2AC0`–`2AC4`, all with CCCD | not explored |
| `18FE` (Shimano) | `2AE2`, `2AE3` | not explored |
| `18FF` (Shimano) | `2AF3`–`2AFF`; CCCD on `2AF3`, `2AF9`, `2AFB`, `2AFD` | not explored |

## Readings
- `2A19` = `0x50` = **80 %** (E-Tube Project showed "full" at the same time).
- `2A29` = `SHIMANO`, `2A25` = serial number, `2A26` = `4.1.0.0`
  (4-part version; Shimano's website uses 3 parts, e.g. `4.4.0`).
- Connection to all 4 values: ~4 s, mostly service discovery.
- The first connection attempt sometimes times out (status 133), especially while WiFi is
  (re)connecting on the shared radio; the next attempt succeeds.

## Wake / sleep behaviour
- The RD advertises for **~15 min** after last use, then goes silent (measured with
  `binary_sensor.di2_awake` in HA; that sensor turns off 60 s after the last packet).
- Advertising interval grows over time: every ~0.1–0.5 s at first, every several seconds later.
- Active scanning (scan requests) does **not** keep the RD awake.
- A sleeping RD cannot be queried: its radio is off.

## Open questions
- [ ] Is `2A19` the main BT-DN300 battery? Compare with E-Tube as the battery drains.
- [ ] Does manufacturer payload `10 00` change (shifting, charging, low battery)?
- [ ] Does the RD advertise while charging?
- [ ] Can Garmin / E-Tube connect while the ESP32 is connected?
- [ ] Do other 12-speed RDs (R8150, R9250, MTB Di2) expose the same `180F/2A19`?
