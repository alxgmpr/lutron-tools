# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## What This Project Is

Lutron reverse-engineering toolkit: RF transceiver, protocol analyzer, and control interface for Lutron lighting systems (RadioRA3, Homeworks QSX, Caseta, Vive). Covers three transport layers:
- **CCA** (Clear Connect Type A): 433 MHz FSK via STM32H723+CC1101
- **CCX** (Clear Connect Type X): Thread/802.15.4 via nRF52840 (NCP on STM32)
- **LEAP**: JSON/TLS processor API (port 8081)

## Architecture

```
  CC1101 (CCA)  ──┐                              ┌── Interactive shell (USART3/ST-LINK VCP)
                   ├── STM32H723 (FreeRTOS) ──────┤
  nRF52840 (CCX) ─┘   TCP :9433 (lwIP)           └── TCP stream → Bun CLI (cli/nucleo.ts)
```

### Key Directories

| Directory | Runtime | Purpose |
|-----------|---------|---------|
| `firmware/` | STM32 C/C++ (FreeRTOS, lwIP) | Radio drivers, protocol engine, TCP stream, shell |
| `cli/nucleo.ts` | Bun | Primary UI — TCP client, interactive TUI, packet decoder |
| `cli/tui/` | Bun | Terminal UI components (VT100 scroll regions, line editor, table) |
| `tools/` | Bun/TypeScript, Python | CLI utilities (LEAP, CCX, CCA, codegen, analyzers) |
| `protocol/` | YAML + generated TS/C | CCA/CCX protocol definitions (single source of truth) |
| `ccx/` | TypeScript | CCX protocol encoder/decoder/config |
| `lib/` | TypeScript | Shared libraries (env loader, IEEE 802.15.4, Thread crypto) |
| `ldproxy/` | Node.js | Designer auth proxy — injects channel strings to unlock product types |
| `data/` | — | LEAP dumps, device maps, Designer DB exports |

### Protocol Definitions

`protocol/cca.yaml` is the single source of truth for CCA packet structure. `tools/codegen.ts` generates `protocol/generated/typescript/protocol.ts` from it. `protocol/protocol-ui.ts` is the runtime module used by the CLI for packet identification and field parsing.

`protocol/ccx.yaml` defines CCX (Thread) message types. `ccx/` has its own encoder/decoder/types/constants.

### Firmware

FreeRTOS tasks: CCA RX/TX (`cca_task`), CCX NCP (`ccx_task`), TCP stream (`stream`), shell (`shell`).

Key firmware modules:
- `firmware/src/cca/` — CC1101 driver, N81 codec, CRC-16, packet encoder, pairing engine
- `firmware/src/ccx/` — Spinel/HDLC NCP driver, CBOR encoder, IPv6/UDP TX, SMP DFU
- `firmware/src/net/` — lwIP Ethernet, TCP stream server (binary framing, multi-client)
- `firmware/src/shell/` — UART interactive shell with line editing and history
- `firmware/src/storage/` — Flash persistence (device IDs, Thread params)

### CLI

`cli/nucleo.ts` connects to Nucleo over TCP:9433. Key abstractions:
- **Screen** (`cli/tui/screen.ts`) — VT100-based TUI with fixed header/status/input and scrollable packet table region
- **LineEditor** (`cli/tui/line-editor.ts`) — raw mode stdin with history, tab completion
- **PacketTable** (`cli/tui/table.ts`) — in-memory packet buffer with visible-line computation
- Packet identification via `identifyPacket()` from `protocol/protocol-ui.ts`

## Commands

```bash
# TypeScript tools (all run directly with Bun, no build step)
bun run cli/nucleo.ts               # Connect to Nucleo interactive shell
bun run tools/codegen.ts            # Regenerate protocol.ts from cca.yaml
bun run tools/ccx-sniffer.ts --live # Sniff Thread traffic
bun run tools/leap-dump.ts          # Dump LEAP device hierarchy

# NPM script shortcuts
bun run codegen        # Regenerate protocol definitions
bun run ccx:sniff      # Sniff Thread traffic
bun run ccx:send       # Send CCX commands
bun run leap:dump      # Dump LEAP hierarchy
bun run lint           # Biome linter (check)
bun run lint:fix       # Biome auto-fix
bun run format         # Biome formatter

# Firmware
cd firmware && cmake -B build -DCMAKE_TOOLCHAIN_FILE=cmake/arm-none-eabi.cmake && make -C build -j8
cd firmware && make flash      # OpenOCD + ST-LINK (NEVER use st-flash)
cd firmware && make test       # C++ unit tests (CRC, N81, decoder, CBOR)
cd firmware && make monitor    # Serial terminal to ST-LINK VCP (115200)
cd firmware && make format     # clang-format
cd firmware && make lint       # cppcheck
```

## Code Patterns

### TypeScript Tool Structure

All tools follow this pattern: `#!/usr/bin/env bun` shebang, JSDoc header, imports, then `main()`. No CLI arg parsing library — tools use manual `process.argv.slice(2)` with helpers:
```typescript
const getArg = (name: string) => { const i = args.indexOf(name); return i !== -1 ? args[i + 1] : undefined; };
const hasFlag = (name: string) => args.includes(name);
```

Environment values load via `import { RA3_HOST } from "../lib/env"` (reads `.env` from project root, environment variables take precedence).

### CCX CBOR Encoding

`ccx/encoder.ts` uses a **custom minimal CBOR encoder** (not cbor-x) because CCX requires integer-keyed maps and `cbor-x` would stringify object keys. The decoder (`ccx/decoder.ts`) uses `cbor-x` since decoding doesn't have this issue. Messages are CBOR arrays `[msgType, body]` where body uses integer keys from `BodyKey` enum.

CCX sequence numbers persist to `.ccx-seq` file in project root to survive restarts.

### LEAP Client

`tools/leap-client.ts` provides `LeapConnection` class wrapping TLS with request/response pairing via auto-incrementing client tags (`lt-1`, `lt-2`, ...). Messages are newline-delimited JSON. Constructor takes an options object: `new LeapConnection({ host, certName })`.

The client auto-detects RA3 vs Caseta by probing the `/zone` endpoint — RA3 uses area-walk (`/area/{id}/associatedzone`), Caseta exposes zones directly.

Certificate paths are auto-resolved: tries `lutron-{certName}-{cert|key|ca}.pem` in the project root.

### CCX Config Loading

`ccx/config.ts` merges all `data/leap-*.json` files to build zone/device/serial/preset lookup tables. This means LEAP dump data drives CCX tool behavior (zone name resolution, device identification).

## Key Protocol Concepts

**OUTPUT vs DEVICE** is the fundamental architectural split in Lutron:
- **OUTPUT** = zone/load control with level + fade + delay (CCA format 0x0E, CCX type 0)
- **DEVICE** = component control like button presses (CCA pico packets, CCX type 1)

This split explains why pico set-level has no fade control — it uses the DEVICE path which lacks a fade field.

**CCA packet lengths**: type byte 0x80-0x9F = 24 bytes (22 data + 2 CRC), type 0xA0+ = 53 bytes (51 data + 2 CRC). Long packet padding uses 0x00.

**Level encoding**: `level16 = percent * 0xFEFF / 100` (shared across CCA and CCX).

**Fade encoding**: quarter-seconds (`byte = seconds * 4`), used in CCA format 0x0E byte 19 and CCX command key 3.

## Linting

Biome handles both linting and formatting. Config in `biome.json`:
- Scope: `cli/`, `tools/`, `ccx/`, `protocol/` (TypeScript only)
- 2-space indent, spaces not tabs
- Relaxed rules: `noExplicitAny` off, `noNonNullAssertion` off, `useTemplate` off, `useNodejsImportProtocol` off

## Environment Notes

- RTL-SDR captures use 2 MHz sample rate: `rtl_sdr -f 433602844 -s 2000000 -g 40 <output.bin>`
- CC1101 variable-length mode only captures packets matching its configured sync/length — use RTL-SDR to see everything
- Nucleo shell is on USART3 (ST-LINK VCP) — connect via serial terminal or use CLI text passthrough
