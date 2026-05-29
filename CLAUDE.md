# ESPHome DHT22 ESP-01S

ESPHome firmware voor een ESP-01S met DHT22 temperatuur/vochtsensor. OTA updates verlopen via Home Assistant dankzij de `update` component.

## Projectstructuur

- `dht22_esp01.yml` — ESPHome configuratie
- `manifest.json` — firmware manifest voor de update component (gehost via GitHub raw)
- `.gitignore` — sluit `*.bin` uit van git
- Gecompileerde firmware wordt als asset geüpload bij een GitHub Release

## Release-workflow (nieuwe firmwareversie uitbrengen)

1. Verhoog `version` in `dht22_esp01.yml` onder `esphome.project` (bijv. `"1.0.1"`)
2. Verhoog `version` in `manifest.json` naar dezelfde waarde
3. Pas de `path` URL in `manifest.json` aan naar de nieuwe release-tag (bijv. `v1.0.1`)
4. Compileer de firmware in ESPHome Dashboard → download de `.bin`
5. Commit en push de gewijzigde YAML en manifest
6. Maak een GitHub Release aan met de nieuwe tag: `gh release create v1.0.1 dht22-esp01.bin --title "v1.0.1"`
7. Home Assistant toont automatisch "update beschikbaar" → installeren

## Belangrijke beperkingen

**ESP-01S heeft slechts 1MB flash.** OTA vereist dat ~50% vrij is voor het nieuwe image (~470KB max per firmware slot). Voeg geen grote components toe zonder de binary size te controleren. `web_server` is al aanwezig en relatief zwaar — verwijder het als de binary niet meer past.

**ESP8266 kan geen SSL-certificaten valideren.** De `http_request` component staat daarom op `verify_ssl: false`. Dit is een platformbeperking, geen bug.

## GitHub repo

`hugo-leij/esphome-dht22-esp01`

Manifest URL: `https://raw.githubusercontent.com/hugo-leij/esphome-dht22-esp01/main/manifest.json`
