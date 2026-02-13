# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## What This Project Is

Lutron reverse-engineering toolkit: RF transceiver, protocol analyzer, and control interface for Lutron lighting systems (RadioRA3, Homeworks QSX, Caseta, Vive). Covers three transport layers:
- **CCA** (Clear Connect Type A): 433 MHz FSK via ESP32+CC1101
- **CCX** (Clear Connect Type X): Thread/802.15.4 via nRF52840
- **LEAP**: JSON/TLS processor API (port 8081, read-only)

## Commands

```bash
# Development (starts backend + web + ccx-sniffer concurrently)
# Dev does everything, we don't need to run builds as this is not a production app yet
npm run dev

# Run web tests (vitest)
npm test

# ESPHome compile (ESP32 firmware)
esphome compile esphome/cca-proxy.yaml

# ESPHome flash OTA — ALWAYS use --no-logs with "run" (blocks forever otherwise)
esphome run --no-logs --device cca-proxy.local esphome/cca-proxy.yaml
# If you need logs after flashing (but use a timeout or limit blocking some other way)
esphome logs --device cca-proxy.local esphome/cca-proxy.yaml

# CLI tools (all use Bun)
bun run tools/ccx-send.ts --help
bun run tools/ccx-sniffer.ts --live --relay
bun run tools/rtlsdr-cca-decode.ts --rate 2000000 <capture.bin>
bun run tools/leap-dump.ts
```

## Architecture

ESP sends data via UDP to Bun backend, Bun backend exposes packets via SSE to frontend.
### Workspace Layout

**npm workspaces**: `web/` and `backend/` are npm workspaces; `tools/` and `ccx/` are standalone Bun scripts.

| Directory | Runtime | Purpose |
|-----------|---------|---------|
| `web/` | Vite + React 18 + Tailwind 4 + shadcn/ui | Packet viewer, device control UI |
| `backend/` | Bun | UDP receiver → SSE broadcaster, HTTP API |
| `esphome/custom_components/cc1101_cca/` | ESP32 C++ | CC1101 radio driver, CCA encode/decode |
| `tools/` | Bun/TypeScript | CLI utilities (RTL-SDR decoder, LEAP dump, analyzers) |
| `ccx/` | TypeScript | CCX protocol encoder/decoder/config |
| `protocol/` | TypeScript | Shared CCA protocol definitions |
| `db/` | Python | SQLite extraction from .ra3/.hw project files |
| `proxy/` | Node.js | Lutron Designer API proxy (feature unlocking) |
| `captures/` | — | RF capture files (RTL-SDR .bin, session logs) |
| `docs/` | — | Protocol documentation (start with `lutron-rf-overview.md`) |

### Shared Protocol Layer

`protocol/cca.yaml` is the single source of truth for CCA protocol definitions. `protocol/protocol-ui.ts` is the runtime module imported by both backend and frontend for packet identification, field parsing, and button name mapping.

`ccx/` has its own encoder/decoder/types/constants/config — separate from CCA.

### Frontend

React 18 + Vite 6, Tailwind CSS 4, shadcn/ui (New York style, zinc palette). Path alias `@/*` maps to `web/src/*`. API calls proxy through Vite dev server (`/api` → `localhost:5001`).

Key hooks: `usePacketStream` (SSE connection with 250ms batch flush), `useApi` (HTTP wrapper).

### Backend

Bun HTTP server on port 5001. Receives CCA packets on UDP:9433, CCX packets on UDP:9190. Broadcasts to connected SSE clients. Sends TX commands to ESP32 on UDP:9434. ESP32 host is auto-detected from incoming UDP source address.

### ESP32 Firmware

ESPHome custom component at `esphome/custom_components/cc1101_cca/`. Main config is `esphome/cca-proxy.yaml` (~680 lines of lambdas and service definitions). The C++ driver handles CC1101 SPI, N81 serial encoding, CRC-16, multi-sync RX, and all pairing/config/control TX sequences. Communication with backend is via `esphome/udp_stream.h` (JSON over UDP).

## Key Protocol Concepts

**OUTPUT vs DEVICE** is the fundamental architectural split in Lutron:
- **OUTPUT** = zone/load control with level + fade + delay (CCA format 0x0E, CCX type 0)
- **DEVICE** = component control like button presses (CCA pico packets, CCX type 1)

This split explains why pico set-level has no fade control — it uses the DEVICE path which lacks a fade field.

**CCA packet lengths**: type byte 0x80-0x9F = 24 bytes (22 data + 2 CRC), type 0xA0+ = 53 bytes (51 data + 2 CRC). All long packets must be padded with 0xCC (not 0x00).

**Level encoding**: `level16 = percent * 0xFEFF / 100` (shared across CCA and CCX).

**Fade encoding**: quarter-seconds (`byte = seconds * 4`), used in CCA format 0x0E byte 19 and CCX command key 3.

## Environment Notes

- Dev UI is always at `http://localhost:5173` — use the browser, not curl
- `esphome run` ALWAYS needs `--no-logs` — otherwise it streams logs forever and blocks the terminal
- RTL-SDR captures use 2 MHz sample rate: `rtl_sdr -f 433602844 -s 2000000 -g 40 <output.bin>`
- CC1101 variable-length mode only captures packets matching its configured sync/length — use RTL-SDR to see everything
