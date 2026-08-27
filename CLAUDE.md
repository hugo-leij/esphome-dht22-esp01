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
Assistant (the web interface is gone since 1.2.0); it survives soft reboots, no reflash needed. Do not give each
device a unique `friendly_name` in this shared config — that's what Location is for.

Known units (hostname suffix = last 3 MAC bytes):

| Hostname | MAC | Notes |
|---|---|---|
| `dht22-esp01-651a64` | `48:3f:da:65:1a:64` | recovered by a normal USB reflash |
| `dht22-esp01-07cdc1` | `2c:f4:32:07:cd:c1` | needed a full chip erase |
| `dht22-esp01-08fa66` | `a4:e5:7c:08:fa:66` | added 2026-08-27, first unit flashed with 1.2.0 |

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

### Un-bricking a device — try a normal reflash FIRST; full erase only as a last resort

Do **not** full-erase by default. Order of escalation when a device won't come online:

1. **Normal `esphome run` over USB first.** This is enough for most cases (e.g. a device that
   just needs the proven-good firmware back). It keeps the stored Wi-Fi credentials, so the
   device rejoins automatically with no captive-portal step. Confirmed sufficient on
   `48:3f:da:65:1a:64` — a full erase there would only have wiped its working credentials.
2. **Full chip erase only if a normal reflash still leaves it dead** (no AP, no HA, no serial
   logs even on its own power) — i.e. a genuine `restore_from_flash` brick.

Why a normal reflash isn't always enough for a real brick: `esphome run` only erases the
firmware region (`0x0`–`0x67fff`); the corrupt data `restore_from_flash` wrote to the **upper
flash sectors** (preferences, and likely the RF-cal/SDK sector) survives. The chip reads it at
boot, crashes before Wi-Fi/OTA come up → no AP, no HA, no serial logs — looks dead even though
the flash + hash verify "succeed". For that case, a **full chip erase** first, then a fresh
flash. Confirmed working on `2c:f4:32:07:cd:c1`:

```
# 1. Put the ESP-01 in the USB programmer (GPIO0 low / flash mode), then full-erase all 1MB:
/opt/homebrew/Cellar/esphome/<ver>/libexec/bin/python -m esptool \
  --chip esp8266 --port /dev/cu.usbserial-XXXX erase-flash
# 2. Flash fresh:
esphome run dht22_esp01.yml --device /dev/cu.usbserial-XXXX
```

Use esphome's bundled esptool (v5.x in its brew libexec venv) — the standalone
`~/.platformio/.../esptool.py` is an old v3.0 that fails to connect, and the system `python3`
lacks `pyserial`. After a full erase the stored Wi-Fi credentials are gone too, so the device
comes up on the **"DHT22 ESP01 Setup"** AP and must be re-provisioned via the captive portal.

### How devices get Wi-Fi credentials

The `wifi:` block has **no `ssid`/`password`** and there is no `secrets.yaml` — credentials are
**not** baked into the firmware. Each device is provisioned once via the captive portal AP
("DHT22 ESP01 Setup"), and the credentials are stored in flash. A normal `esphome run` reflash
(partial erase) preserves them, so a previously-provisioned device rejoins automatically after an
update. A brand-new device, or one after a **full** erase, has no credentials and must be
provisioned via the captive portal.

General rule: anything that runs at boot or touches flash (`restore_from_flash`, `on_boot`,
`preferences`, `globals`) must be tested on ONE device, confirmed back online, before touching
any others. Never push such a change to multiple devices untested.

## Devices going silent (1.2.0 — keep the config lean)

Symptom: sensors stop publishing to HA after hours/days; a power cycle sometimes doesn't
bring them back. Three things in the config made this worse and were removed/changed in 1.2.0:

- **`web_server` removed.** On the ESP8266 it's the heaviest component at *runtime*: the
  AsyncWebServer, JSON encoding of every state change, and one event-source connection per
  open browser tab all come out of the same ~50 KB heap that the API and Wi-Fi stack use.
  Heap exhaustion shows up as a device that is "on" but stops answering. Static RAM only
  drops ~0.7 KB (32428 → 31760 bytes) and flash ~35 KB (421601 → 386045); the real win is
  heap headroom, which the compiler doesn't report. Manage entities from HA instead.
  `captive_portal` is kept — it pulls in `web_server_base`, but only serves the tiny
  provisioning page while in AP mode.
- **`api: reboot_timeout: 0s` → `15min`.** `0s` disables ESPHome's own watchdog, so a device
  that lost the API connection would sit there forever. This is the single most likely reason
  a stuck unit never came back on its own.
- **`fast_connect: true` removed.** It skips the scan and re-associates with the stored BSSID.
  On a multi-AP/mesh network the device keeps hammering an AP that moved or rebooted instead
  of picking the strongest one. Only useful for hidden SSIDs.

`power_save_mode: none` is deliberately kept — it costs current but avoids the missed-packet
dropouts that `light` causes on ESP8266.

If devices still go silent after this, suspect **power**: the ESP-01 draws ~300 mA peaks on TX
and many DHT22/ESP-01 adapter boards have a marginal 3.3 V regulator. Add a 470 µF cap across
3V3/GND, or feed it from a decent supply. Also check the 4.7–10 kΩ pull-up on the DHT data line
(GPIO2) — GPIO2 is a boot-strap pin and must be high at boot.

## Important limitations

**ESP-01S only has 1MB of flash.** OTA requires ~50% free space for the new image (~470KB max per firmware slot). Do not add large components without checking binary size. Do not re-add `web_server` — see the section above.

**ESP8266 cannot validate SSL certificates and cannot handle large TLS records.** Fetching anything over HTTPS from a server with large TLS records (e.g. GitHub) is unreliable on this platform. This is a platform limitation, not a bug.

## GitHub repo

`hugo-leij/esphome-dht22-esp01`
