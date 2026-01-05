# CLAUDE.md

## Database Tools (`db/`)

```bash
python3 db/lutron-tool.py extract "project.hw"
python3 db/lutron-tool.py pack Project.mdf output.lut --template original.lut
python3 db/lutron-tool.py info <file>
```

File format: `.ra3`/`.hw` (ZIP) → `<uuid>.lut` (MTF) → `Project.mdf` (SQL Server 2022)

## ESP32 RF Controller (`rf/`)

```bash
python3 rf/esp32_controller.py list
python3 rf/esp32_controller.py press <button>
python3 rf/esp32_controller.py send <device_id> <button_code>
python3 rf/esp32_controller.py level <source_id> <target_id> <0-100>
python3 rf/esp32_controller.py pair <device_id>
python3 rf/esp32_controller.py serve --port 8080
```

ESP32 at `10.1.4.59:6053`. ESPHome config in `rf/esphome/`.

```bash
cd rf/esphome && esphome run pico-proxy-cc1101.yaml --no-logs
```

## RF Analysis (`rf/lutron_cca/`)

```bash
python3 -m lutron_cca capture -d 10
python3 -m lutron_cca decode capture.cu8
python3 -m lutron_cca analyze capture.cu8
python3 -m lutron_cca live
python3 -m lutron_cca devices
```

## RF Capture Tool (`rf/capture.py`)

```bash
python3 rf/capture.py <name>           # Capture RTL-SDR + ESPHome logs
python3 rf/capture.py <name> --no-sdr  # ESPHome logs only
python3 rf/capture.py <name> --no-logs # RTL-SDR only
```

Press Enter to stop. Files saved to `rf/captures/<name>.cu8` and `<name>.log`.

## Designer Proxy (`proxy/`)

```bash
cd proxy && npm install && npm start
```

## Rules

NEVER use an emoji. Ever.

NEVER run RTL-SDR captures or ESPHome log captures directly. ALWAYS ask the user to run the capture tool themselves and provide the output. The user is in control of all RF captures.