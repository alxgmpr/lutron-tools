# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## What This Project Is

Lutron reverse-engineering toolkit: RF transceiver, protocol analyzer, and control interface for Lutron lighting systems (RadioRA3, Homeworks QSX, Caseta, Vive). Covers three transport layers:
- **CCA** (Clear Connect Type A): 433 MHz FSK via STM32H723+CC1101
- **CCX** (Clear Connect Type X): Thread/802.15.4 via nRF52840 (NCP on STM32)
- **LEAP**: JSON/TLS processor API (port 8081, read-only)

## Architecture

STM32H723 Nucleo is the primary platform. It runs dual radios (CC1101 for CCA, nRF52840 for CCX), Ethernet (lwIP), and FreeRTOS. The host CLI (`cli/nucleo.ts`) connects over UDP for packet streaming and commands.

```
  CC1101 (CCA)  ──┐                              ┌── Interactive shell (USART3/ST-LINK VCP)
                   ├── STM32H723 (FreeRTOS) ──────┤
  nRF52840 (CCX) ─┘   TCP :9433 (lwIP)           └── TCP stream → Bun CLI (cli/nucleo.ts)
```

### Directory Layout

| Directory          | Runtime                      | Purpose                                                        |
| ------------------ | ---------------------------- | -------------------------------------------------------------- |
| `firmware/`        | STM32 C/C++ (FreeRTOS, lwIP) | Radio drivers, protocol engine, TCP stream, shell              |
| `cli/nucleo.ts`    | Bun                          | Primary UI — TCP client, interactive shell, packet decoder     |
| `tools/`           | Bun/TypeScript, Python       | CLI utilities (RTL-SDR decoder, LEAP dump, codegen, analyzers) |
| `protocol/`        | YAML + generated TS          | CCA/CCX protocol definitions (single source of truth)          |
| `ccx/`             | TypeScript                   | CCX protocol encoder/decoder/config                            |
| `captures/`        | —                            | RF capture files (RTL-SDR .bin, session logs)                  |
| `docs/`            | —                            | Protocol documentation (start with `lutron-rf-overview.md`)    |

### Protocol Definitions

`protocol/cca.yaml` is the single source of truth for CCA packet structure. `tools/codegen.ts` generates `protocol/generated/typescript/protocol.ts` from it. `protocol/protocol-ui.ts` is the runtime module used by the CLI for packet identification and field parsing.

`protocol/ccx.yaml` defines CCX (Thread) message types. `ccx/` has its own encoder/decoder/types/constants.

### Firmware

STM32H723 Nucleo firmware in `firmware/`. FreeRTOS tasks: CCA RX/TX (`cca_task`), CCX NCP (`ccx_task`), TCP stream (`stream`), shell (`shell`). Build with STM32CubeIDE or `make` (ARM GCC).

Key firmware modules:
- `firmware/src/cca/` — CC1101 driver, N81 codec, CRC-16, packet encoder, pairing engine, command builders
- `firmware/src/ccx/` — Spinel/HDLC NCP driver, CBOR encoder, IPv6/UDP TX, SMP DFU
- `firmware/src/net/` — lwIP Ethernet, TCP stream server (binary framing, multi-client)
- `firmware/src/shell/` — UART interactive shell with line editing and history
- `firmware/src/storage/` — Flash persistence (device IDs, Thread params)

### CLI

`cli/nucleo.ts` is the primary user interface. Connects to Nucleo over TCP:9433. Features:
- Interactive shell with all CCA/CCX commands
- Live packet display with protocol decoding (`identifyPacket()`)
- Text passthrough to STM32 shell (prefix with `!` or use `pass` command)
- Status query, packet recording to CSV
- Auto-reconnect on disconnect

## Commands

```bash
# CLI — connect to Nucleo and interact
bun run cli/nucleo.ts

# Codegen — regenerate protocol.ts from cca.yaml
bun run tools/codegen.ts

# RTL-SDR decode — analyze raw RF captures
bun run tools/rtlsdr-cca-decode.ts --rate 2000000 <capture.bin>

# CCX tools
bun run tools/ccx-send.ts --help
bun run tools/ccx-sniffer.ts --live --relay

# LEAP system dump
bun run tools/leap-dump.ts

# RTL-SDR capture
rtl_sdr -f 433602844 -s 2000000 -g 40 <output.bin>
```

## Key Protocol Concepts

**OUTPUT vs DEVICE** is the fundamental architectural split in Lutron:
- **OUTPUT** = zone/load control with level + fade + delay (CCA format 0x0E, CCX type 0)
- **DEVICE** = component control like button presses (CCA pico packets, CCX type 1)

This split explains why pico set-level has no fade control — it uses the DEVICE path which lacks a fade field.

**CCA packet lengths**: type byte 0x80-0x9F = 24 bytes (22 data + 2 CRC), type 0xA0+ = 53 bytes (51 data + 2 CRC). Long packet padding uses 0x00.

**Level encoding**: `level16 = percent * 0xFEFF / 100` (shared across CCA and CCX).

**Fade encoding**: quarter-seconds (`byte = seconds * 4`), used in CCA format 0x0E byte 19 and CCX command key 3.

## Environment Notes

- RTL-SDR captures use 2 MHz sample rate: `rtl_sdr -f 433602844 -s 2000000 -g 40 <output.bin>`
- CC1101 variable-length mode only captures packets matching its configured sync/length — use RTL-SDR to see everything
- Nucleo shell is on USART3 (ST-LINK VCP) — connect via serial terminal or use CLI text passthrough
