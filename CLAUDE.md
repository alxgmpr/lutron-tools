# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## Project Overview

A Python CLI tool for extracting and modifying Lutron lighting control system project files. It enables round-trip editing of RadioRA3 (.ra3) and Homeworks QS (.hw) project files by extracting their embedded SQL Server databases, allowing modifications, and repacking.

## File Format Hierarchy

```
.ra3 or .hw (ZIP Archive)
  └── <uuid>.lut (MTF Backup - Microsoft Tape Format)
      └── Project.mdf + Project_log.ldf (SQL Server Database)
```

## Commands

```bash
# Extract project file to .lut
python3 lutron-tool.py extract "project.hw" [output_dir]

# Extract .lut to database files
python3 lutron-tool.py extract "backup.lut" [output_dir]

# Pack database back to .lut (use --template for best results)
python3 lutron-tool.py pack Project.mdf output.lut --template original.lut

# Pack .lut back to project file (auto-detects .ra3 or .hw from metadata)
python3 lutron-tool.py pack backup.lut output

# Show file information
python3 lutron-tool.py info <file>
```

## Architecture

**lutron-tool.py** - Main CLI tool with all functionality:
- `extract_project_file()` - Unzips .ra3/.hw to get .lut, saves project type metadata
- `extract_lut_file()` - Parses MTF format, extracts SQL Server .mdf/.ldf starting at offset 0x4000
- `pack_to_lut()` - Creates .lut from .mdf, uses template to preserve MTF headers
- `pack_to_project()` - Zips .lut back to .ra3/.hw

**extract_lutron_db.py** - Standalone legacy extractor (superceded by lutron-tool.py)

## Key Technical Details

- Database signature: SQL Server pages start with `0x01 0x0f` at offset 0x4000
- Standard .mdf size: 36,700,160 bytes (35,840 KB) - sparse backups are zero-padded
- Metadata stored in `.lutron-metadata.json` to remember original project type
- Template-based packing preserves MTF header structure from original file
- Uses only Python standard library (no external dependencies)

## Critical: Database Version Compatibility

Lutron Designer 25.10 uses SQL Server database **version 957** (SQL Server 2022 RTM). Most SQL Server installations will upgrade the database when attached, causing Lutron Designer to reject the file.

**Solution:** Use SQL Server 2022 RTM in Docker:
```bash
docker run -e "ACCEPT_EULA=Y" -e "MSSQL_SA_PASSWORD=LutronPass123" \
  -p 1433:1433 --name sql2022rtm \
  -v "$(pwd)":/data \
  -d mcr.microsoft.com/mssql/server:2022-RTM-ubuntu-20.04
```

**Best Method:** Edit the live database while Lutron Designer has the project open:
1. Open project in Lutron Designer
2. Find LocalDB pipe: `Get-ChildItem "\\.\pipe\" | Where-Object {$_.Name -like "*LOCALDB*"}`
3. Connect in SSMS: `np:\\.\pipe\LOCALDB#XXXXX\tsql\query`
4. Make SQL changes, then close Lutron Designer normally

See `LUTRON_DATABASE_EDITING.md` for complete documentation.

## ESP32 CC1101 RF Transmitter

The ESP32 with CC1101 radio is used for Lutron Clear Connect Type A RF transmission.

**Connection Details:**
- IP Address: `10.1.4.59` (ALWAYS - never use any other IP)
- Hostname: `pico-trigger.local`
- Web Auth: `admin:claude`

**ESPHome Commands (NEVER use --device flag, it's in the YAML):**
```bash
esphome run pico-proxy-cc1101.yaml      # Compile and upload
esphome compile pico-proxy-cc1101.yaml  # Compile only
esphome logs pico-proxy-cc1101.yaml     # Stream logs (blocking)
```

**Trigger Buttons via HTTP:**
```bash
# Authentication required - NOTE: ESPHome uses DASHES not underscores in URLs!
curl -u admin:claude "http://10.1.4.59/button/BUTTON_ID/press"

# Available buttons (use dashes, not underscores):
curl -u admin:claude "http://10.1.4.59/button/rf-on/press"        # RF On (Pico)
curl -u admin:claude "http://10.1.4.59/button/rf-off/press"       # RF Off (Pico)
curl -u admin:claude "http://10.1.4.59/button/pair-esp32/press"   # PAIR ESP32 (unique ID)
curl -u admin:claude "http://10.1.4.59/button/pair--0xb9-/press"  # PAIR (0xB9) (real Pico ID)
curl -u admin:claude "http://10.1.4.59/button/test-pkt/press"     # TEST PKT
```

**Working Directory for ESPHome:**
```
/Users/alexgompper/lutron-db-tool/esphome/
```

**Using the ESPHome CLI:**
- use `esphome run` to compile and upload to the device
- `esphome run` should ALWAYS have `--no-logs` passed with it or we will hang forever
- `esphome logs` is interactive so it will STREAM logs. you cannot just use the command without a timeout.
- never pass `--device` to `esphome`

**SDR Capture**
Often we need to run `rtl_sdr` WHILE we trigger a CURL to the ESP to emit some RF. We can use `concurrently`, the npm/npx package to do this. Remember that rtl_sdr does not start INSTANTLY. So we should ALWAYS be careful to make sure we get a real capture. Log the timestamps that we run CURL commands so that we can validate this when we analyze.