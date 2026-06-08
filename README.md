# 🌡️ DHT22 ESP-01S — ESPHome Temperature & Humidity Sensor

A minimal, Wi-Fi connected temperature and humidity sensor built with an **ESP-01S (ESP8266)** and a **DHT22** sensor, powered by [ESPHome](https://esphome.io) and integrating natively with **Home Assistant**.

---

## 📦 Hardware

| Component | Details | Buy |
|---|---|---|
| Microcontroller + Sensor | ESP-01S (ESP8266) + DHT22 (AM2302) — combo | [AliExpress](https://www.aliexpress.com/item/1005007048872208.html) |
| Programmer | USB-TTL adapter (set jumper to **3.3V** — never 5V!) | [Amazon NL](https://www.amazon.nl/dp/B07TXVRQ7V) |
| Wires | Dupont jumper wires | |
| Power (optional) | Separate 3.3V supply (ESP-01 is power-sensitive) | |

---

## ✨ Features

- 🌡️ Temperature measurement with configurable offset
- 💧 Humidity measurement with configurable offset
- 📶 Wi-Fi signal strength (RSSI) reporting
- ⏱️ Uptime sensor
- 🔌 Status binary sensor
- 🌐 Built-in web server (port 80)
- 🔁 OTA firmware updates over Wi-Fi via `esphome upload` (no USB required after initial flash)
- 🛠️ Captive portal for easy Wi-Fi setup
- 📍 Configurable **Location** label (stored on-device) to tell multiple units apart
- 🏠 Native Home Assistant API integration

---

## 🔌 Pinout — ESP-01S

| ESP-01S Pin | Function |
|---|---|
| VCC | 3.3V power |
| GND | Ground |
| TX | Serial transmit → RX on USB-TTL |
| RX | Serial receive → TX on USB-TTL |
| GPIO0 | Flash mode (pull to GND during boot) |
| CH_PD / EN | Enable (connect to 3.3V) |
| GPIO2 | DHT22 data pin |

---

## 🛠️ Flashing — Step by Step

### Step 1 — Install the YAML in ESPHome

1. Open **Home Assistant** and go to **ESPHome** (add-on or integration).
2. Click **+ New Device** → choose **Import from File** → upload [`dht22_esp01.yml`](./dht22_esp01.yml) from this repo.
3. Click **Save**.

### Step 2 — Download the firmware

1. Click **Install** → choose **Manual download**.
2. ESPHome will compile the firmware — this takes a minute or two.
3. Download the resulting `.bin` file to your computer.

### Step 3 — Connect the USB-TTL adapter

> ⚠️ **Important:** Make sure your USB-TTL adapter jumper is set to **3.3V**. Connecting 5V will damage the ESP-01S.

Wire your ESP-01S to the USB-TTL adapter as follows:

```
ESP-01S      USB-TTL
--------     --------
VCC      →   3.3V
GND      →   GND
TX       →   RX
RX       →   TX
EN       →   3.3V
GPIO0    →   GND     ← only during flashing!
```

### Step 4 — Enter flash mode

1. Connect **GPIO0 → GND** before powering on.
2. Plug in the USB-TTL adapter (this powers and resets the ESP-01S).
3. The device is now in flash mode.

### Step 5 — Flash via ESPHome Web

1. Go to **[web.esphome.io](https://web.esphome.io)** in Chrome or Edge.
2. Click **Connect** and select your USB-TTL serial port.
3. Click **Install** → upload the `.bin` file you downloaded in Step 2.
4. Wait for the flash to complete.

### Step 6 — Boot normally

1. **Disconnect GPIO0 from GND**.
2. Power-cycle the ESP-01S (unplug and replug USB-TTL).
3. The device will boot and broadcast a Wi-Fi AP called **`DHT22 ESP01 Setup`**.

### Step 7 — Configure Wi-Fi

1. Connect your phone or laptop to the **`DHT22 ESP01 Setup`** access point.
2. A captive portal will appear — enter your Wi-Fi credentials.
3. The device will connect to your network and appear in Home Assistant automatically.

---

## 🏠 Home Assistant

Once connected, the following entities will appear in Home Assistant:

| Entity | Type |
|---|---|
| Temperature | Sensor (°C) |
| Humidity | Sensor (%) |
| WiFi RSSI | Sensor (dBm) |
| Uptime | Sensor |
| Status | Binary Sensor |
| IP Address | Text Sensor (diagnostic) |
| SSID | Text Sensor (diagnostic) |
| Temperature Offset | Number (config) |
| Humidity Offset | Number (config) |
| Location | Text (config) |
| Restart Device | Button |

---

## 📍 Telling devices apart

All units share the same friendly name (`DHT22 ESP-01`), so with several on your network it's hard to know which physical sensor you're looking at. Each device has a **Location** text entity for this:

1. Open the device — in Home Assistant, or directly via its IP (web server on port 80).
2. Set **Location** to where the sensor lives, e.g. `Living room` or `Bedroom`.
3. The value is stored on the device and survives reboots — no reflash needed, and it moves with the device if you relocate it.

The Location appears at the top of the web interface, so opening an IP immediately tells you which room it is.

---

## 🔄 Firmware Updates

After the initial USB flash, all future updates are pushed over Wi-Fi from the terminal with `esphome upload` — no USB adapter or programmer needed.

### How to update

1. Bump `version` in `dht22_esp01.yml` under `esphome.project` (optional, for tracking).
2. From a machine with [ESPHome installed](https://esphome.io/guides/installing_esphome), run:
   ```bash
   esphome upload dht22_esp01.yml --device <device-ip>
   ```
   For example: `esphome upload dht22_esp01.yml --device 10.0.10.48`
3. ESPHome compiles the firmware locally and flashes it over Wi-Fi; the device reboots automatically.

> ℹ️ The **first flash must be done via USB** (see flashing instructions above). The Wi-Fi OTA mechanism (`ota: esphome`) is installed as part of that initial firmware.

> ⚠️ Home Assistant auto-update is **not** used. GitHub's TLS records are too large for the ESP8266's limited RAM (`BR_ERR_TOO_LARGE`), so manifest-based updates over HTTPS are unreliable on the ESP-01. Manual `esphome upload` over the local network is the supported path.

---

## 🎯 Calibration

The DHT22 sensor can have a small offset compared to the actual temperature or humidity. You can correct this directly in Home Assistant using the built-in offset controls — no reflashing needed.

### How to calibrate

1. Place a accurate reference thermometer/hygrometer next to the ESP-01S sensor and let both stabilize for at least 15 minutes.
2. In Home Assistant, go to the device and find the **Temperature Offset** and **Humidity Offset** number entities.
3. Compare the readings and enter the difference as the offset. For example, if the sensor reads 22.5°C but the reference shows 21.8°C, set the offset to `-0.7`.
4. The corrected value is applied in real time — no restart needed.

> 💡 A good reference device to use: [Bambulab Circular Digital Thermometer/Hygrometer](https://nl.aliexpress.com/item/1005009214032096.html) — compact, accurate, and battery-powered.

---

## ⚠️ Troubleshooting

| Problem | Solution |
|---|---|
| Device not detected in browser | Install CH340 or CP210x driver for your USB-TTL adapter |
| Flash fails | Check GPIO0 is connected to GND before power-on |
| No sensor readings | Verify DHT22 data wire is on GPIO2; check power supply stability |
| Device not showing in HA | Check Wi-Fi credentials via captive portal; ensure HA API is enabled |
| Wrong temperature/humidity | Use the offset sliders in Home Assistant to calibrate |

---

## 📝 License

MIT — free to use, modify, and share.
