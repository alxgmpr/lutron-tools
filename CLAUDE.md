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

## Designer Proxy (`proxy/`)

```bash
cd proxy && npm install && npm start
```

NEVER use an emoji. Ever.