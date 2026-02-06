---
name: esphome-logs
description: Stream logs from the CCA proxy ESP32 device
disable-model-invocation: true
---

# ESPHome Logs

Stream live logs from the CCA proxy device over WiFi.

## Instructions

Run the following command from the repo root in the background:

```bash
esphome logs esphome/cca-proxy.yaml --device cca-proxy.local
```

Run this command in the background so the user can continue working while logs stream. Use a timeout of 600000ms (10 minutes).

If the user provides additional arguments (e.g. filtering), append them to the command.

Report that logs are streaming and remind the user they can stop the background task when done.
