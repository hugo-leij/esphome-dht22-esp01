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
- 🔁 OTA (Over-the-Air) updates via ESPHome
- 🛠️ Captive portal for easy Wi-Fi setup
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
| Restart Device | Button |

---

## 📄 ESPHome YAML

```yaml
substitutions:
  device_base_name: "dht22-esp01"
  device_friendly_name: "DHT22 ESP-01"
  setup_ap_ssid: "DHT22 ESP01 Setup"

esphome:
  name: ${device_base_name}
  name_add_mac_suffix: true
  friendly_name: ${device_friendly_name}

esp8266:
  board: esp01_1m

logger:

api:
  reboot_timeout: 0s

ota:
  - platform: esphome

web_server:
  version: 2
  port: 80

wifi:
  fast_connect: true
  power_save_mode: none
  ap:
    ssid: ${setup_ap_ssid}

captive_portal:

number:
  - platform: template
    name: "Temperature Offset"
    id: temperature_offset
    min_value: -10
    max_value: 10
    step: 0.1
    mode: box
    unit_of_measurement: "°C"
    optimistic: true
    restore_value: true
    initial_value: 0.0
    entity_category: config

  - platform: template
    name: "Humidity Offset"
    id: humidity_offset
    min_value: -20
    max_value: 20
    step: 0.1
    mode: box
    unit_of_measurement: "%"
    optimistic: true
    restore_value: true
    initial_value: 0.0
    entity_category: config

sensor:
  - platform: dht
    pin: GPIO2
    model: DHT22
    temperature:
      name: "Temperature"
      id: temperature
      device_class: temperature
      state_class: measurement
      filters:
        - lambda: return x + id(temperature_offset).state;
    humidity:
      name: "Humidity"
      id: humidity
      device_class: humidity
      state_class: measurement
      filters:
        - lambda: return x + id(humidity_offset).state;
    update_interval: 60s

  - platform: wifi_signal
    name: "WiFi RSSI"
    update_interval: 60s
    icon: "mdi:wifi"

  - platform: uptime
    name: "Uptime"
    update_interval: 60s
    icon: "mdi:timer-outline"

binary_sensor:
  - platform: status
    name: "Status"

text_sensor:
  - platform: wifi_info
    ip_address:
      name: "IP"
      entity_category: diagnostic
      icon: "mdi:ip-network"
    ssid:
      name: "SSID"
      entity_category: diagnostic
      icon: "mdi:wifi-settings"

button:
  - platform: restart
    name: "Restart device"
    icon: "mdi:restart"
```

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
