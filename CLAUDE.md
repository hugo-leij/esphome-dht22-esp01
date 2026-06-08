# ESPHome DHT22 ESP-01S

ESPHome firmware for an ESP-01S with a DHT22 temperature/humidity sensor. OTA updates are done manually from the terminal with `esphome upload`.

## Project structure

- `dht22_esp01.yml` — ESPHome configuration
- `.gitignore` — excludes `*.bin` from git
- The compiled firmware lives only on the build machine; it is not committed or released.

## Release workflow (publishing a new firmware version)

1. Bump `version` in `dht22_esp01.yml` under `esphome.project` (e.g. `"1.0.2"`)
2. Push the new firmware over Wi-Fi (OTA) from the terminal:
   `esphome upload dht22_esp01.yml --device <device-ip>`
   (e.g. `esphome upload dht22_esp01.yml --device 10.0.10.48`)
3. Commit and push the updated YAML.

The first flash must be done over USB (see README). After that, all updates go over
Wi-Fi via `esphome upload`.

## Why no Home Assistant auto-update

We previously used the `update` + `http_request` components to pull a manifest/binary
from GitHub and let Home Assistant install updates. **This does not work on the ESP-01.**
GitHub's TLS records are too large for the ESP8266's RAM, so the fetch fails with
`BR_ERR_TOO_LARGE` even with `verify_ssl: false`. Those components were removed; updates
are now manual via the terminal.

## Identifying devices

All units share the same `device_friendly_name` (`DHT22 ESP-01`), with `name_add_mac_suffix`
making only the hostname unique. To tell physical sensors apart there's a `text` template
entity **Location** (`id: location`, `restore_value: true`) — set per device from Home
Assistant or the web interface; it's stored in flash and survives reboots, no reflash needed.
Do not give each device a unique `friendly_name` in this shared config — that's what Location
is for.

## Important limitations

**ESP-01S only has 1MB of flash.** OTA requires ~50% free space for the new image (~470KB max per firmware slot). Do not add large components without checking binary size. `web_server` is already present and relatively heavy — remove it if the binary no longer fits.

**ESP8266 cannot validate SSL certificates and cannot handle large TLS records.** Fetching anything over HTTPS from a server with large TLS records (e.g. GitHub) is unreliable on this platform. This is a platform limitation, not a bug.

## GitHub repo

`hugo-leij/esphome-dht22-esp01`
