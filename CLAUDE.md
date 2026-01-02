# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## Project Overview

**Lutron Tools** - A comprehensive toolkit for hacking Lutron lighting control systems:

1. **Database Tools** (`db/`) - Extract and modify RadioRA3/Homeworks project files
2. **RF Tools** (`rf/`) - Clear Connect Type A protocol analysis and transmission
3. **Designer Proxy** (`proxy/`) - Unlock Homeworks Programming in Lutron Designer

## Repository Structure

```
lutron-tools/
├── db/                     # Database extraction tools
│   ├── lutron-tool.py      # Main CLI for .ra3/.hw files
│   └── extract_lutron_db.py
├── rf/                     # RF/Clear Connect tools
│   ├── lutron_cca/         # Python protocol library
│   ├── esphome/            # ESP32 CC1101 component
│   └── analysis/           # Protocol analysis scripts
├── proxy/                  # Designer API proxy
│   ├── server.js           # Node.js proxy server
│   └── package.json
├── docs/                   # Documentation
├── research/               # Reverse engineering notes
└── data/                   # Captures, projects, certs (gitignored)
```

## Database Tools (db/)

Extract and repack Lutron project files:

```bash
# Extract project file
python3 db/lutron-tool.py extract "project.hw" [output_dir]

# Pack database back to project
python3 db/lutron-tool.py pack Project.mdf output.lut --template original.lut

# Show file information
python3 db/lutron-tool.py info <file>
```

**File Format:**
```
.ra3 or .hw (ZIP Archive)
  └── <uuid>.lut (MTF Backup)
      └── Project.mdf + Project_log.ldf (SQL Server Database)
```

**Database Version:** Lutron Designer 25.10 uses SQL Server 2022 RTM (version 957). See `docs/DATABASE_EDITING.md` for compatibility.

## RF Tools (rf/)

### ESP32 CC1101 RF Transmitter

**Connection:**
- IP: `10.1.4.59` (always)
- API: ESPHome native API (port 6053, encrypted)

**ESPHome Commands:**
```bash
cd rf/esphome
esphome run pico-proxy-cc1101.yaml --no-logs  # Compile and upload
esphome compile pico-proxy-cc1101.yaml        # Compile only
esphome logs pico-proxy-cc1101.yaml           # Stream logs (blocking)
```

**Control via Python:**
```bash
python3 rf/esphome/esp32_controller.py list                # List buttons
python3 rf/esphome/esp32_controller.py press rf-on         # Press button
python3 rf/esphome/esp32_controller.py serve --port 8080   # Start web UI
```

**Button Aliases:**
- `rf-on`, `rf-off`, `rf-raise`, `rf-lower`, `rf-favorite` (Pico 05851117)
- `level-0` through `level-100` (AF902C00)
- `bridge-0`, `bridge-50`, `bridge-100` (06fdeff4)
- `beacon`, `beacon-5s`, `beacon-91`, `beacon-93` (Pairing beacons)

**Requirements:**
```bash
pip3 install aioesphomeapi flask
```

### RF Analysis Library (rf/lutron_cca/)

Python library for Clear Connect Type A protocol:

```bash
python3 -m lutron_cca capture -d 10           # Capture 10s
python3 -m lutron_cca decode capture.cu8      # Decode packets
python3 -m lutron_cca analyze capture.cu8     # Detailed analysis
```

**SDR Capture with Button Press:**
```bash
npx -y concurrently --success first \
  "timeout 8 rtl_sdr -f 433602844 -s 2000000 -g 40 capture.cu8" \
  "sleep 2 && python3 rf/esphome/esp32_controller.py press rf-on && sleep 5"
```

## Designer Proxy (proxy/)

Unlocks Homeworks Programming in Lutron Designer:

```bash
cd proxy
npm install
npm start  # Runs on port 3000
```

Configure your system to route Lutron API traffic through the proxy. See `proxy/README.md` for details.

## Documentation

All documentation is in `docs/`:
- `DATABASE_EDITING.md` - SQL Server integration guide
- `CCA_PROTOCOL.md` - Clear Connect Type A protocol spec
- `PACKET_ANALYSIS.md` - RF packet structure
- `KNOWN_ISSUES.md` - Current limitations

## ESPHome CLI Notes

- Always use `--no-logs` with `esphome run` to avoid hanging
- `esphome logs` is blocking - use with timeout
- Never pass `--device` flag (it's in the YAML)
- Working directory: `rf/esphome/`
