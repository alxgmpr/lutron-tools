---
name: nucleo
description: Context for working with the STM32 Nucleo H723ZG and its attached radios (CC1101 for CCA 433 MHz, nRF52840 for CCX Thread/802.15.4). Use this skill proactively whenever modifying firmware, CLI code, shell commands, or radio protocol handling. Also use when the user asks about building, flashing, debugging, or interacting with the Nucleo hardware.
metadata:
  author: alexgompper
  version: "1.0.0"
user_invocable: false
---

# STM32 Nucleo H723ZG — Hardware & Firmware Reference

## Hardware Setup

```
  CC1101 (SPI1)  ───┐                                 ┌── USART3 (ST-LINK VCP) → serial shell
  433 MHz CCA       ├── STM32H723ZG Nucleo-144 ───────┤
  nRF52840 (SPI6) ──┘   FreeRTOS + lwIP               ├── Ethernet (LAN8742A) → TCP/UDP :9433
  802.15.4 CCX           ARM Cortex-M7 @ 550 MHz       └── UDP stream → Bun CLI (cli/nucleo.ts)
```

- **CC1101**: 433 MHz FSK transceiver for CCA (Clear Connect Type A). SPI1, GDO0/GDO2 interrupt pins.
- **nRF52840**: 802.15.4 radio for CCX (Clear Connect Type X / Thread). SPI6 via Spinel/HDLC NCP protocol.
- **Ethernet**: LAN8742A PHY, lwIP TCP/IP stack. Nucleo IP: `10.1.1.114`.
- **USART3**: ST-LINK Virtual COM Port — interactive shell with line editing and history.

## Building & Flashing

**IMPORTANT: Never use `st-flash`. Always use `make flash` (OpenOCD).**

```bash
cd firmware

# Build (debug)
make build

# Build + flash via OpenOCD
make flash

# Release build
make release

# Reset without reflash
make reset
```

Build system: CMake with ARM GCC toolchain. Sources listed in `firmware/CMakeLists.txt`.
OpenOCD config: `firmware/openocd.cfg`.
Output ELF: `firmware/build/lutron-nucleo.elf`.

## FreeRTOS Tasks

| Task | File | Priority | Purpose |
|------|------|----------|---------|
| `cca_task` | `firmware/src/cca/cca_task.c` | 3 | CC1101 RX/TX, CCA packet processing, command queue |
| `ccx_task` | `firmware/src/ccx/ccx_task.cpp` | 3 | nRF52840 Spinel NCP, Thread join, CCX multicast + CoAP unicast |
| `stream_task` | `firmware/src/net/stream.c` | 2 | UDP packet stream to CLI clients (port 9433, binary framing) |
| `shell_task` | `firmware/src/shell/shell.cpp` | 1 | UART interactive shell |
| `eth_task` | `firmware/src/net/eth.c` | 2 | lwIP Ethernet, link polling |

## Shell Commands (UART + CLI passthrough)

### Top-level
| Command | Description |
|---------|-------------|
| `status` | CC1101 state, RSSI, packet counts, heap, uptime |
| `rx on\|off` | Enable/disable CCA RX |
| `tx <hex>` | Transmit raw CCA packet (hex bytes) |
| `cca [cmd]` | CCA radio commands |
| `ccx [cmd]` | CCX Thread commands |
| `ot [cmd]` | OpenThread NCP query/control |
| `spinel [cmd]` | Raw Spinel property access |
| `stream` | UDP stream status |
| `eth` | Ethernet PHY debug |
| `config` | Show stored flash settings |
| `save` | Save settings to flash |
| `reboot` | NVIC_SystemReset |

### CCA subcommands (`cca ...`)
| Command | Description |
|---------|-------------|
| `cca button <dev_id_hex> <name>` | Button press (on/off/fav/raise/lower/scene1-4) |
| `cca level <zone_hex> <target_hex> <%> [fade_qs]` | Bridge set-level |
| `cca broadcast <zone_hex> <%> [fade_qs]` | Broadcast level to all devices |
| `cca pico-level <dev_id_hex> <%>` | Pico level control |
| `cca state <dev_id_hex> <%>` | State report |
| `cca beacon <dev_id_hex> [dur]` | Discovery beacon |
| `cca unpair <zone_hex> <target_hex>` | Unpair device |
| `cca led <zone_hex> <target_hex> <0-3>` | LED config |
| `cca fade <zone_hex> <target_hex> <on_qs> <off_qs>` | Fade config |
| `cca trim <zone_hex> <target_hex> <hi%> <lo%>` | Trim config |
| `cca phase <zone_hex> <target_hex> <byte_hex>` | Phase config |
| `cca save-fav <dev_id_hex>` | Save favorite level |
| `cca vive-level <hub_hex> <zone_hex> <%> [fade]` | Vive set-level |
| `cca vive-raise/lower <hub_hex> <zone_hex>` | Vive dim |
| `cca vive-pair <hub_hex> <zone_hex> [dur]` | Vive pairing |
| `cca pair pico <dev_hex> [type] [dur]` | Pico pairing (5btn/2btn/4btn-rl/4btn-scene) |
| `cca pair bridge <id_hex> <target_hex> [dur]` | Bridge pairing |
| `cca identify <target_hex>` | Flash device LED |
| `cca query <target_hex>` | Query device component info |
| `cca tune ...` | CC1101 register/tuning debug |
| `cca log [on\|off]` | CCA RX UART log toggle |

### CCX subcommands (`ccx ...`)
| Command | Description |
|---------|-------------|
| `ccx` | Thread status (role, RX/TX counts) |
| `ccx on <zone>` | Send ON to zone (multicast) |
| `ccx off <zone>` | Send OFF to zone |
| `ccx level <zone> <0-100>` | Set level % (multicast) |
| `ccx scene <id>` | Recall scene |
| `ccx peers` | List known Thread peers (RLOC16 → serial) |
| `ccx promisc [on\|off]` | Promiscuous mode (raw 802.15.4 frames) |
| `ccx log [on\|off]` | CCX RX UART log toggle |
| `ccx coap preset <addr> <dev_id> <preset_id> <level%> [fade_s]` | CoAP preset programming |
| `ccx coap aha <addr> <k4> <k5>` | CoAP AHA LED brightness |
| `ccx coap get <addr> <uri_path>` | CoAP GET |
| `ccx coap put <addr> <uri_path> <hex_payload>` | CoAP PUT |
| `ccx coap post <addr> <uri_path> <hex_payload>` | CoAP POST |

### CCX address resolution (`<addr>` argument)
The CoAP commands accept addresses in three formats:
- Full IPv6: `fd0d:02ef:a82c:0000:abcd:1234:5678:9abc`
- RLOC shorthand: `rloc:2C0C` → builds full RLOC IPv6 from mesh-local prefix
- Serial lookup: `serial:72200096` → looks up RLOC from peer table

### OT subcommands (`ot ...`)
| Command | Description |
|---------|-------------|
| `ot` | Thread state, RLOC16, partition ID |
| `ot extaddr` | Get NCP EUI-64 |
| `ot eui64` | Get factory EUI-64 |
| `ot channel` | Get/set channel |
| `ot panid` | Get/set PAN ID |
| `ot masterkey` | Get/set network master key |
| `ot extpanid` | Get/set extended PAN ID |
| `ot rloc16` | Get RLOC16 |
| `ot leaderdata` | Get leader data |
| `ot netdata` | Get network data |
| `ot addrtable` | Get address table |
| `ot ifconfig [up\|down]` | Interface up/down |
| `ot thread [start\|stop]` | Thread start/stop |
| `ot reset` | NCP reset |

## CLI (Host Side)

```bash
# Connect to Nucleo
bun run cli/nucleo.ts              # uses NUCLEO_HOST env var
bun run cli/nucleo.ts 10.1.1.114   # direct IP

# Environment
NUCLEO_HOST=10.1.1.114             # default Nucleo IP
```

The CLI connects over UDP port 9433. Features:
- Interactive shell mirroring all firmware commands
- Live CCA/CCX packet display with protocol decoding
- Prefix `!` or use `pass` to send raw text to STM32 shell
- `record` / `stop` for CSV packet capture
- `status` queries firmware status blob
- Auto-reconnect on disconnect
- LEAP data integration for zone/device name lookups

## Key Firmware Source Files

| File | Purpose |
|------|---------|
| `firmware/src/cca/cca_task.c` | CCA RX/TX task, command queue, packet processing |
| `firmware/src/cca/cc1101.c` | CC1101 SPI driver, register config, RX/TX |
| `firmware/src/cca/n81_codec.c` | CCA N81 bit encoding/decoding |
| `firmware/src/cca/cca_crc.c` | CCA CRC-16 calculation |
| `firmware/src/cca/cca_commands.c` | CCA command builders (button, level, pair, config) |
| `firmware/src/ccx/ccx_task.cpp` | CCX task: Thread join, multicast TX, CoAP unicast, peer table |
| `firmware/src/ccx/coap.c` | CoAP message builder (GET/PUT/POST with CBOR payload) |
| `firmware/src/ccx/ipv6_udp.c` | IPv6/UDP packet construction, checksum |
| `firmware/src/ccx/ccx_msg.c` | CCX CBOR message encoder (LEVEL_CONTROL, SCENE_RECALL) |
| `firmware/src/ccx/ccx_cbor.c` | Minimal CBOR encoder |
| `firmware/src/ccx/smp_serial.c` | SMP DFU over Spinel (nRF firmware update) |
| `firmware/src/net/stream.c` | UDP stream server (binary framing, multi-client) |
| `firmware/src/net/eth.c` | lwIP Ethernet init, link management |
| `firmware/src/shell/shell.cpp` | UART shell with line editing, history, all commands |
| `firmware/src/storage/flash_store.c` | Flash persistence (device IDs, Thread credentials) |

## Protocol Encoding Quick Reference

### CCA
- Packet types: 0x80-0x9F = 24 bytes (22 data + 2 CRC), 0xA0+ = 53 bytes (51 data + 2 CRC)
- Level encoding: `level16 = percent * 0xFEFF / 100`
- Fade encoding: `byte = seconds * 4` (quarter-seconds)
- IDs are 32-bit hex (device_id, zone_id, target_id)

### CCX
- Multicast: UDP to `ff03::1` port 9190, CBOR payload
- LEVEL_CONTROL: `[0, { 0: {0: level, 3: fade, 4: delay}, 1: [16, zoneId], 5: seq }]`
- SCENE_RECALL: `[1, { 1: sceneId, 5: seq }]`
- CoAP unicast: port 5683, to device primary ML-EID
- AHA LED brightness: path `/cg/db/ct/c/AHA`, payload `[108, {4: <active>, 5: <inactive>}]`
- Level encoding same as CCA: `level16 = percent * 0xFEFF / 100`
- Zone type always 16 in LEVEL_CONTROL key 1

## Network

| Host | IP | Purpose |
|------|-----|---------|
| Nucleo | 10.1.1.114 | STM32 (TCP/UDP :9433) |
| RA3 Processor | 10.1.1.133 | LEAP API (:8081), Thread border router |
| Caseta | 10.1.9.3 | LEAP API (:8081) |
| Designer VM | 10.1.1.115 | SSH (alex/alex), Designer + SQL Server |

Thread mesh-local prefix: `fd0d:2ef:a82c:0::/64`
Thread channel: 25, PAN ID: 0x62EF

## Common Workflows

### Build and flash firmware change
```bash
cd firmware && make flash
```

### Test a CCA command
```bash
bun run cli/nucleo.ts
# In CLI: cca level <zone> <target> 50
```

### Test a CCX command
```bash
bun run cli/nucleo.ts
# In CLI: ccx level <zone> 75
```

### Send CoAP to a device
```bash
bun run cli/nucleo.ts
# In CLI: ccx coap aha fd0d:02ef:a82c:0000:XXXX:XXXX:XXXX:XXXX 150 20
```

### Check Thread network status
```bash
bun run cli/nucleo.ts
# In CLI: ccx
# In CLI: ot
# In CLI: ccx peers
```

### Debug CC1101 radio
```bash
bun run cli/nucleo.ts
# In CLI: status
# In CLI: cca tune show
# In CLI: cca tune reg get 0x04
```
