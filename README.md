# esphome-powerpal-ble

An ESPHome external component that reads a [Powerpal](https://www.powerpal.net) energy monitor over Bluetooth
Low Energy and, optionally, uploads the readings to the Powerpal cloud so the Powerpal app keeps its history.

The component descends from the `powerpal_ble` component originally written by WeekendWarrior1 and carried
in [muneeb1990/esphome@powerpal_ble](https://github.com/muneeb1990/esphome/tree/powerpal_ble). That fork is no
longer maintained and does not build on current ESPHome. This repository keeps it working.

This project is not affiliated with Powerpal.

## Status

Tested on an `esp32dev` board with the Arduino framework on ESPHome **2026.11.0**, with two gateways each
paired to its own Powerpal. Other boards, frameworks and ESPHome versions are untested. The fork this came from
built on ESPHome 2025.4 but does not build on 2026.11.

## Installation

```yaml
external_components:
  - source:
      type: git
      url: https://github.com/kanemari/esphome-powerpal-ble
      ref: v1.1.2
    components: [ powerpal_ble ]
    refresh: 1d
```

Pin `ref` to a tag. A tag never changes, so a push cannot alter firmware you have already tested. Use a branch
only on a test device.

## Example

```yaml
esp32:
  board: esp32dev
  framework:
    type: arduino

time:
  - platform: homeassistant
    id: homeassistant_time

ble_client:
  - mac_address: AA:BB:CC:DD:EE:FF   # the Powerpal's Bluetooth address, written without quotes
    id: powerpal

http_request:                        # only needed for the cloud upload
  id: powerpal_cloud_uploader
  verify_ssl: false

sensor:
  - platform: powerpal_ble
    ble_client_id: powerpal
    time_id: homeassistant_time
    pairing_code: 123456
    notification_interval: 1
    pulses_per_kwh: 3200
    cost_per_kwh: 0.25
    powerpal_device_id: "0004abcd"
    http_request_id: powerpal_cloud_uploader
    power:
      name: "Powerpal Power"
    daily_energy:
      name: "Powerpal Daily Energy"
    energy:
      name: "Powerpal Total Energy"
    battery_level:
      name: "Powerpal Battery"
```

## Options

| Option | Required | Description |
|---|---|---|
| `ble_client_id` | yes | The `ble_client` for the Powerpal. |
| `pairing_code` | yes | The Powerpal's pairing code, 1 to 999999. |
| `notification_interval` | yes | Minutes covered by each reading, 1 to 60. |
| `pulses_per_kwh` | yes | Pulses per kWh of your meter (the LED rate printed on it). |
| `cost_per_kwh` | with `http_request_id` | Price used for the `cost` sensor. |
| `time_id` | no | A time source. Without one, daily energy relies on the Powerpal's own clock. |
| `powerpal_device_id` | no | The Powerpal's 8-character hex id. Read from the device when not set. Quote it, because an id made only of digits is otherwise read as a number. |
| `powerpal_apikey` | no | The cloud API key. Read from the device when not set. |
| `http_request_id` | no | An `http_request` hub. Turns on the cloud upload. |

Sensors: `power` (W), `daily_energy` and `energy` (kWh), `battery_level` (%), and the raw per-reading values
`pulses`, `daily_pulses`, `watt_hours`, `cost` and `timestamp`. The raw sensors are ordinary ESPHome sensors,
which store a 32-bit float, so `timestamp` cannot hold a Unix time exactly (it is rounded to about two minutes).
The cloud upload does not use it.

## Cloud upload

With `http_request_id` set, readings are posted to
`https://readings.powerpal.net/api/v1/meter_reading/<device id>`, 15 readings per request, authorised with the
Powerpal's API key.

- **Exact timestamps.** Each reading keeps the time the Powerpal reported. A Powerpal that was out of range
  replays the readings it stored when it reconnects, and those are uploaded with their original times.
- **Retries.** A failed upload is tried again after 60 seconds. Up to 60 readings are held in RAM, and the oldest
  are dropped beyond that. Readings still in RAM are lost on a reboot.
- **`watt_hours`.** The uploaded value is `pulses * 1000 / pulses_per_kwh`. The `watt_hours` sensor keeps its
  older calculation and so does not match.
- **Cost.** The Powerpal cloud works out cost from the tariff on your Powerpal account. The `cost` field you send
  is not what ends up stored.
- **The API key.** It is read from the Powerpal during pairing and printed in the log at INFO level as
  `Powerpal apikey: ...`. Treat logs as sensitive.

## Things to know

- A Powerpal accepts **one Bluetooth connection at a time**. While a gateway holds it, the phone app cannot
  connect, and a second test device cannot run against the same Powerpal.
- The device id is not the Bluetooth address. The address (12 hex digits) goes in `ble_client`, and the device id
  (8 hex digits) identifies the Powerpal to the cloud.
- The Powerpal's name in a Bluetooth scan is `Powerpal <device id>`.
- Over-the-air updates replace only the app, not the bootloader or partition table. A device first flashed with
  an old ESPHome keeps them until it is flashed once over USB.

## Changes from the original fork

**v1.1.2** Pairing code is written only after authentication and service discovery have both finished, and is
retried up to 3 times (8 seconds apart) if the Powerpal does not confirm it, after which the link is dropped
so the BLE client reconnects. Before this, a slow link could issue the write during service discovery and never
get a confirmation, leaving the gateway connected but silent.

**v1.1.1** Corrected log format specifiers for 64-bit `time_t` and counters.

**v1.1.0** Cloud upload restored: exact timestamps, batches of 15, retry, API key option honoured, corrected
`watt_hours` in the upload.

**v1.0.0** Builds on ESPHome 2026.11: removed an unused `WiFi.h` include and updated `address_str()` use (it now
returns `const char*`).
