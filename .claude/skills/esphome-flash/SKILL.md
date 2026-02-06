---
name: esphome-flash
description: Compile and flash ESPHome firmware OTA to the CCA proxy device
disable-model-invocation: true
---

# ESPHome Flash (OTA)

Compile and flash the ESPHome firmware to the CCA proxy device over WiFi (OTA).

## Instructions

Run the following command from the repo root:

```bash
esphome run esphome/cca-proxy.yaml --device cca-proxy.local
```

Use a timeout of 600000ms (10 minutes) since compile + OTA upload can take ~90 seconds or more.

Report the result to the user: success or failure with relevant error output.
