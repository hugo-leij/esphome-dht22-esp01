# ESPHome DHT22 ESP-01S

ESPHome firmware for an ESP-01S with a DHT22 temperature/humidity sensor. OTA updates are handled via Home Assistant using the `update` component.

## Project structure

- `dht22_esp01.yml` — ESPHome configuration
- `manifest.json` — firmware manifest for the update component (hosted via GitHub raw)
- `.gitignore` — excludes `*.bin` from git
- Compiled firmware is uploaded as an asset to a GitHub Release

## Release workflow (publishing a new firmware version)

1. Bump `version` in `dht22_esp01.yml` under `esphome.project` (e.g. `"1.0.1"`)
2. Bump `version` in `manifest.json` to the same value
3. Update the `path` URL in `manifest.json` to point to the new release tag (e.g. `v1.0.1`)
4. Compile the firmware in ESPHome Dashboard → download the `.bin`
5. Commit and push the updated YAML and manifest
6. Create a GitHub Release with the new tag: `gh release create v1.0.1 dht22-esp01.bin --title "v1.0.1"`
7. Home Assistant will automatically show "update available" → install

## Important limitations

**ESP-01S only has 1MB of flash.** OTA requires ~50% free space for the new image (~470KB max per firmware slot). Do not add large components without checking binary size. `web_server` is already present and relatively heavy — remove it if the binary no longer fits.

**ESP8266 cannot validate SSL certificates.** The `http_request` component therefore has `verify_ssl: false`. This is a platform limitation, not a bug.

## GitHub repo

`hugo-leij/esphome-dht22-esp01`

Manifest URL: `https://raw.githubusercontent.com/hugo-leij/esphome-dht22-esp01/main/manifest.json`
