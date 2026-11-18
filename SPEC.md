# di2-esphome — SPEC v0.2

Goal: an ESP32 reads the battery level of a Shimano 12-speed Di2 rear derailleur whenever
the derailleur wakes up, and reports it to Home Assistant.

Reference system: Shimano 105 Di2 (R7100). Findings are in [docs/recon.md](docs/recon.md).

---

## 1. System overview

- **RD-R7150 (rear derailleur)** is the wireless unit: BLE + ANT radio, receives the shifters.
  **The ESP32 talks to the RD only.**
- **Main battery BT-DN300** sits in the seatpost/frame, wired to the derailleurs. The RD
  reports its level.
- **Shifters ST-R7170** are wireless, CR1632 coin cells. Out of scope for now.
- The RD advertises over BLE for ~15 min after it is used (shifting), then sleeps with the
  radio off.

## 2. Approach (decided)

Options considered, simplest first:

| Option | Description | Result |
|--------|-------------|--------|
| A | Read battery passively from advertising, no connection | ✗ manufacturer data is only `10 00`, not the battery |
| **B** | **Short GATT connection without pairing, read `180F/2A19`, disconnect** | **✓ works, chosen** |
| C | GATT connection with bonding (passkey) | not needed |
| D | ANT+ (needs nRF52 or ANT stick) | not needed |

Rule: **read-only.** Never write to any Shimano characteristic (settings, firmware update
mode).

## 3. Firmware (ESPHome, `esphome/di2.yaml`)

### Hardware
- Any ESP32 with BLE (tested: ESP32-D0WD-V3 DevKit, CP2102). USB powered, placed within a
  few metres of where the bike is parked.

### Functional requirements
- F1. Continuous BLE scan (active).
- F2. Match the RD by MAC address (from `secrets.yaml`).
- F3. Detect a new wake period: RD seen after >60 s of silence.
- F4. On each wake: connect → read `2A19`, `2A29`, `2A25`, `2A26` → disconnect.
  Repeat every 5 min while the RD keeps advertising.
- F5. Serial log: RD seen (every 30 s, with RSSI and manufacturer data), every value read.
- F6. Keep the last known values; the RD being offline most of the time is normal.

### Non-functional requirements
- N1. Never hold a permanent connection to the RD.
- N2. No writes to the RD's GATT.
- N3. WiFi credentials, API key, OTA password and MAC live in `secrets.yaml`, not in the repo.
- N4. No reboot when WiFi or Home Assistant is unavailable.

### Home Assistant entities
| Entity | Source |
|--------|--------|
| `Di2 battery` (%) | `180F` / `2A19` |
| `Di2 awake` (connectivity) | RD advertised within the last 60 s |
| `Di2 RSSI` (diagnostic) | advertising |
| `Di2 name` (diagnostic) | advertising |
| `Di2 manufacturer` / `serial number` / `firmware` (diagnostic) | `180A` / `2A29`, `2A25`, `2A26` |

## 4. Roadmap

- [ ] **Low battery notification**: HA automation when `Di2 battery` drops below 20 %.
- [ ] **New firmware notification**: compare `Di2 firmware` with the latest RD firmware.
  - Shimano lists versions at `bike.shimano.com/.../e-tube/project/firmware/detail.html`
    (latest found for RD-R7150: 4.4.0, 2026-08-19).
  - The page is behind bot protection (plain `curl` gets "Access Denied"); do not try to
    bypass it. Options: find a public JSON endpoint the page uses, or keep the latest
    version in an `input_text` updated by hand.
  - The RD reports `4.1.0.0` (4 parts), Shimano lists `4.4.0` (3 parts): normalise first.
- [ ] Discover the RD automatically (pairing mode) instead of a hard-coded MAC.
- [ ] Verify other 12-speed RDs (R8150, R9250, MTB Di2).
- [ ] Check coexistence with Garmin / E-Tube while the ESP32 is connected.
- [ ] Battery-powered standalone device with a display (see below).

### Standalone device (idea)
- Small display (e-paper preferred: shows the value without power), 2 buttons, buzzer
  at < 20 %, LiPo battery.
- Battery life needs a low-power BLE chip (nRF52840: ~5 mA while scanning vs ~60–100 mA on
  ESP32). With a ~15 min advertising window, a few seconds of scanning every 5 min is enough
  to catch every wake-up.
- Home Assistant without WiFi: broadcast the value as BTHome advertising.
- Requires new firmware (Zephyr / nRF Connect SDK or Arduino + Bluefruit); the protocol part
  is small and already known.
