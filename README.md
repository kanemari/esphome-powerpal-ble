# esphome-powerpal-ble

`powerpal_ble` external component for ESPHome, based on
[muneeb1990/esphome@powerpal_ble](https://github.com/muneeb1990/esphome/tree/powerpal_ble).

Changes from upstream (needed for ESPHome 2026.11.0):
- Removed unused `#include "WiFi.h"`.
- `address_str().c_str()` -> `address_str()` (it now returns `const char*`).

## Usage

```yaml
external_components:
  - source:
      type: git
      url: https://github.com/kanemari/esphome-powerpal-ble
      ref: v1.1.0
    components: [ powerpal_ble ]
    refresh: 1d
```

Pin `ref` to a tag for live devices; use a branch only for testing.
A Powerpal accepts one BLE connection at a time: never run a test device against the same Powerpal as a live gateway.

## Cloud upload (v1.1.0)

Restores the Powerpal cloud upload that was disabled in the original fork.
Add an `http_request:` hub to your config and reference it from the sensor:

```yaml
http_request:
  id: powerpal_cloud_uploader
  verify_ssl: false

sensor:
  - platform: powerpal_ble
    http_request_id: powerpal_cloud_uploader
    cost_per_kwh: 0.2211          # required with http_request_id
    # powerpal_apikey: XXXXXXXX-XXXX-XXXX-XXXX-XXXXXXXXXXXX   # optional, otherwise read from the Powerpal
```

Every 15 readings are POSTed to `https://readings.powerpal.net/api/v1/meter_reading/<device_id>`.
Failed uploads retry after 60 s; at most 60 readings are held in RAM.
The uploaded `watt_hours` is `pulses * 1000 / pulses_per_kwh`; the legacy `watt_hours` sensor is unchanged.

## Pairing fix (v1.1.2)

The pairing-code write is now sent only after both authentication and service discovery have completed,
and is retried (up to 3 times, 8 s apart) if the Powerpal does not confirm it, then the link is dropped
so the BLE client reconnects. Previously the write could be issued while service discovery was still
running, and a slow link never got a write confirmation.

