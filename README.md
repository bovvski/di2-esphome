# di2-bridge

ESPHome firmware for an ESP32 that reads the battery level of a **Shimano 12-speed Di2** rear
derailleur and shows it in Home Assistant. No phone, no pairing.

Tested with 105 Di2 (RD-R7150). Not affiliated with or endorsed by Shimano.

## Features

- Battery level (%) read every time the derailleur wakes up (shift a gear), then every 5 min
  while it stays awake (~15 min).
- `Di2 awake` sensor: see when the derailleur is active or asleep.
- Diagnostics: firmware version, serial number, manufacturer, name, RSSI.
- Read-only, no pairing, short connections only (the ESP32 never stays connected).

## Flash

Requirements: an ESP32 with BLE, Docker (or the `esphome` CLI).

```sh
cd esphome
cp secrets.yaml.example secrets.yaml   # fill in WiFi, api_key, ota_password
```

**First flash (USB):**

```sh
docker run --rm -it --network host -v "$PWD":/config --device /dev/ttyUSB0 \
  ghcr.io/esphome/esphome run di2.yaml
```

Shift a gear, find `RD seen: ... mac=XX:XX:...` in the log, put the MAC into `di2_mac` in
`secrets.yaml` and flash again.

**Updates (OTA over WiFi):**

```sh
docker run --rm -it --network host -v "$PWD":/config \
  ghcr.io/esphome/esphome run di2.yaml --device <ESP32-IP>
```

## Add to Home Assistant

Settings → Devices & services → **di2-bridge** is discovered automatically → Configure →
enter `api_key` from `secrets.yaml`. (Or: Add integration → ESPHome → ESP32 IP, port 6053.)

## Roadmap

- Low battery notification (< 20 %).
- New derailleur firmware notification.

## License

[PolyForm Noncommercial 1.0.0](LICENSE.md): free for personal and other noncommercial use;
commercial use, including selling devices with this firmware, is not permitted without
permission. ESPHome itself is a separate project under its own licenses.
