# CLAUDE.md

## Database Tools (`db/`)

```bash
python3 db/lutron-tool.py extract "project.hw"
python3 db/lutron-tool.py pack Project.mdf output.lut --template original.lut
python3 db/lutron-tool.py info <file>
```

File format: `.ra3`/`.hw` (ZIP) → `<uuid>.lut` (MTF) → `Project.mdf` (SQL Server 2022)

## CCA Playground (`rf/`)

### Unified Server
```bash
cca serve                    # Start unified server (frontend + backend)
cca serve -p 3000 -b 8080    # Custom ports
```

Opens web UI at http://localhost:3000 with real-time packet display.

### CCA CLI
```bash
cca decode <file.log>        # Decode ESPHome log file
cca decode "88 00 8D E6..."  # Decode hex packet
cca live                     # Live packet stream from ESP32
cca live --json              # JSON output for scripting
cca crc "88 00 8D E6..."     # Calculate CRC-16
cca info                     # Protocol reference
```

### Python API
```python
import cca
packet = cca.decode("88 00 8D E6 95 05 21 04...")
print(packet.device_id, packet.button, packet.crc_valid)
crc = cca.calc_crc(bytes.fromhex("88008DE6950521"))
```

### ESP32 Controller (low-level)
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

## Designer Proxy (`proxy/`)

```bash
cd proxy && npm install && npm start
```

## Rules

NEVER use an emoji. Ever.

NEVER run RTL-SDR captures or ESPHome log captures directly. ALWAYS ask the user to run the capture tool themselves and provide the output. The user is in control of all RF captures.
