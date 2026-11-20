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
      ref: v1.0.0
    components: [ powerpal_ble ]
    refresh: 1d
```

Pin `ref` to a tag for live devices; use a branch only for testing.
A Powerpal accepts one BLE connection at a time: never run a test device against the same Powerpal as a live gateway.
