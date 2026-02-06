---
name: esphome-compile
description: Compile ESPHome firmware for the CCA proxy device
disable-model-invocation: true
---

# ESPHome Compile

Compile the ESPHome firmware for the CCA proxy ESP32 device.

## Instructions

Run the following command from the repo root:

```bash
esphome compile esphome/cca-proxy.yaml
```

Use a timeout of 300000ms (5 minutes) since compilation can take ~60 seconds or more.

Report the result to the user: success or failure with relevant error output.
