# ESPHome DHT22 ESP-01S

ESPHome firmware for an ESP-01S with a DHT22 temperature/humidity sensor. OTA updates are done manually from the terminal with `esphome run`.

## Project structure

- `dht22_esp01.yml` — ESPHome configuration
- `.gitignore` — excludes `*.bin` from git
- The compiled firmware lives only on the build machine; it is not committed or released.

## Release workflow (publishing a new firmware version)

1. Bump `version` in `dht22_esp01.yml` under `esphome.project` (e.g. `"1.1.1"`)
2. Compile and push the new firmware over Wi-Fi (OTA) from the terminal:
   `esphome run dht22_esp01.yml --device <device-ip>`
   (e.g. `esphome run dht22_esp01.yml --device 10.0.10.48`)
3. Commit and push the updated YAML.

Use `esphome run` (compile + upload), NOT `esphome upload` — the latter only re-sends the
last compiled binary and silently ignores YAML changes. The first flash must be done over
USB (see README); after that all updates go over Wi-Fi via `esphome run`.

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
Assistant or the web interface; it survives reboots, no reflash needed. Do not give each
device a unique `friendly_name` in this shared config — that's what Location is for.

## DANGER: `restore_from_flash` BRICKS this ESP-01 — confirmed, do not use it

`esp8266: restore_from_flash: true` **hard-bricks this ESP-01.** This is confirmed in
isolation, not a guess:

- v1.1.1 added `restore_from_flash: true` + an `on_boot` flash write together → both bricked
  devices boot-looped and stayed dead even after a cold power cycle (USB reflash required).
- v1.1.3 re-tested `restore_from_flash` **alone** (no `on_boot`), with `safe_mode` added as a
  net, on a single device (10.0.10.47). It bricked again: completely off the network (no ping,
  no port 8266, no web). `safe_mode` did NOT help — the crash happens during flash-preference
  init, before safe_mode brings up Wi-Fi/OTA, so it can't recover. USB reflash required.
- v1.1.4 reverted to the proven-good config (no `restore_from_flash`, no `safe_mode`).

Consequence: `restore_value` data (offsets) lives in RTC memory and **resets to `initial_value`
(0) after every OTA reboot**. This is unavoidable on this hardware — just re-enter the offsets
after an update. Do NOT try to "fix" it with `restore_from_flash`; it does not work here.

`safe_mode` is also not a safety net for early-boot crashes — it only helps for crashes that
happen after Wi-Fi/OTA are up.

General rule: anything that runs at boot or touches flash (`restore_from_flash`, `on_boot`,
`preferences`, `globals`) must be tested on ONE device, confirmed back online, before touching
any others. Never push such a change to multiple devices untested.

## Important limitations

**ESP-01S only has 1MB of flash.** OTA requires ~50% free space for the new image (~470KB max per firmware slot). Do not add large components without checking binary size. `web_server` is already present and relatively heavy — remove it if the binary no longer fits.

**ESP8266 cannot validate SSL certificates and cannot handle large TLS records.** Fetching anything over HTTPS from a server with large TLS records (e.g. GitHub) is unreliable on this platform. This is a platform limitation, not a bug.

## GitHub repo

`hugo-leij/esphome-dht22-esp01`
