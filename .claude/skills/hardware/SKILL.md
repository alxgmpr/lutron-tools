---
name: hardware
description: Physical hardware topology and wiring for the Lutron reverse-engineering lab. Use this skill PROACTIVELY whenever discussing hardware, serial ports, USB devices, flashing, DFU, or any physical connection. CRITICAL — prevents confusing which device is which.
metadata:
  author: alexgompper
  version: "2.0.0"
user_invocable: false
---

# Hardware Topology — MUST READ BEFORE ANY HARDWARE DISCUSSION

## Physical Devices

### 1. STM32 Nucleo H723ZG (the "Nucleo")
- **What**: Main development board, ARM Cortex-M7 @ 550 MHz, runs FreeRTOS + lwIP
- **Connection to Mac**: USB (ST-LINK) → `/dev/tty.usbmodem*` (serial shell at 115200 baud)
- **Connection to Mac**: Ethernet → `10.0.0.3` (TCP/UDP port 9433 for CLI + stream)
- **Has attached**: CC1101 (SPI3) and nRF52840 dongle (USART2)
- **Flash method**: `cd firmware && make flash` (OpenOCD via ST-LINK). NEVER use `st-flash`.
- **Reset**: `reboot` shell command or `make reset` from host

### 2. nRF52840 Dongle — NCP (soldered to Nucleo)
- **What**: 802.15.4 radio for Thread/CCX, runs OpenThread NCP firmware
- **4 wires to Nucleo**: VCC, GND, TX (P0.20 → STM32 PD6), RX (P0.24 ← STM32 PD5)
- **UART**: USART2 at 460800 baud, Spinel/HDLC protocol, NO hardware flow control
- **Connection to Mac**: **NONE** — USB port exists on dongle but is NOT plugged into the Mac
- **The dongle's USB port is ONLY used for DFU recovery** (flashing NCP firmware)
- **DFU bootloader**: Red pulsing LED = stuck in bootloader, NOT running NCP firmware
- **Normal operation**: No red LED, communicates over UART only
- **EUI-64**: `F4:CE:36:70:D6:82:E5:33`
- **NCP firmware source**: `~/lutron-tools/src/ot-nrf528xx/` (see NCP Firmware section below)
- **Pre-built DFU zip**: `firmware/ncp/ot-ncp-ftd-dfu.zip` (ready to flash, known working)

### 3. nRF52840 Dongle — Sniffer/RCP (separate, USB to Mac)
- **What**: Second nRF52840 dongle, used for either 802.15.4 sniffing (Wireshark) or OpenThread RCP (ot-daemon)
- **Connection to Mac**: USB → `/dev/cu.usbmodem*` or `/dev/tty.usbmodem*`
- **NOT connected to the Nucleo** — this is a standalone USB dongle
- **Firmware options**:
  - **Sniffer mode** (`/nrf-sniffer` skill): Nordic 802.15.4 sniffer → Wireshark extcap
  - **RCP mode** (`/nrf-ot` skill): OpenThread RCP → ot-daemon CLI
- **DFU**: Press reset button → red LED pulses → flash via `nrfutil nrf5sdk-tools dfu usb-serial`

## CRITICAL DISTINCTIONS

| | NCP Dongle (on Nucleo) | Sniffer/RCP Dongle (USB) |
|---|---|---|
| **Connected to** | STM32 via UART (4 wires soldered) | Mac via USB |
| **USB to Mac?** | **NO** (only for DFU recovery) | **YES** |
| **Serial port on Mac** | None (goes through Nucleo) | `/dev/cu.usbmodem*` |
| **Firmware** | OpenThread NCP | Sniffer or RCP |
| **Purpose** | Thread mesh member, TX/RX | Passive capture or CLI |
| **Red LED = DFU** | Must plug USB to Mac to fix | Press reset, flash via USB |

## Serial Port Guide

| Port | Device |
|------|--------|
| `/dev/tty.usbmodem*` (when only Nucleo connected) | STM32 ST-LINK VCP (shell) |
| `/dev/cu.usbmodem*` (when sniffer dongle plugged in) | nRF52840 sniffer/RCP dongle |
| If TWO usbmodem ports appear | One is ST-LINK, other is sniffer dongle |

**The NCP dongle does NOT appear as a serial port on the Mac** — it talks to the STM32 over UART, and you interact with it through the Nucleo's shell commands (`ot`, `ccx`, `spinel`).

## Common Mistakes to AVOID

1. **DO NOT** assume `/dev/tty.usbmodem*` is the nRF52840 — it's almost always the STM32 ST-LINK
2. **DO NOT** try to flash the NCP dongle via Mac serial commands — it's not connected via USB
3. **DO NOT** confuse the two nRF52840 dongles — one is soldered to the Nucleo (NCP), the other is a separate USB device (sniffer/RCP)
4. **DO NOT** use `nrfutil` to flash the NCP dongle unless its USB is physically plugged into the Mac
5. **DO NOT** send `ot reset` from the shell — it has been removed, but NEVER re-add it
6. **DO NOT** rebuild the NCP firmware without using the EXACT build commands from `firmware/PROJECT.md`
7. **DO NOT** use the `script/build` wrapper in ot-nrf528xx — use the direct cmake invocation

## NCP Firmware — Build & Recovery

### Pre-built (use this first)
A known-working DFU zip is saved at `firmware/ncp/ot-ncp-ftd-dfu.zip`.

### Recovering NCP from DFU Bootloader (Red Pulsing LED)
1. **Plug the NCP dongle's USB into the Mac** — appears as `/dev/tty.usbmodemXXXX`
2. **Flash**: `nrfutil nrf5sdk-tools dfu usb-serial -pkg firmware/ncp/ot-ncp-ftd-dfu.zip -p /dev/tty.usbmodemXXXX`
3. **Unplug dongle USB from Mac**
4. **Reboot the Nucleo** (`reboot` command or power cycle)
5. **Wait ~45s** for NCP probe + Thread join + router promotion

### Building from source (only if needed)
Source: `~/lutron-tools/src/ot-nrf528xx/`

Key config in `src/nrf52840/transport-config.h` (must be modified from defaults):
```c
#define UART_HWFC_ENABLED 0     // disabled (default is 1)
#define UART_PIN_TX 20          // P0.20 (default is 6 = LED1!)
#define UART_PIN_RX 24          // P0.24 (default is 8 = LED2!)
```

Build commands (EXACT — do not change):
```bash
export PATH="/Applications/ArmGNUToolchain/15.2.rel1/arm-none-eabi/bin:$PATH"
cd ~/lutron-tools/src/ot-nrf528xx
cmake -B build -GNinja \
  -DCMAKE_TOOLCHAIN_FILE=src/nrf52840/arm-none-eabi.cmake \
  -DCMAKE_BUILD_TYPE=Release \
  -DOT_APP_NCP=ON -DOT_APP_CLI=OFF -DOT_APP_RCP=OFF \
  -DOT_BOOTLOADER=USB -DNRF_PLATFORM=nrf52840 -DOT_PLATFORM=external \
  -DOT_UART_BAUDRATE=460800
ninja -C build ot-ncp-ftd

# Convert + package
arm-none-eabi-objcopy -O ihex build/bin/ot-ncp-ftd /tmp/ot-ncp-ftd-dfu.hex
nrfutil nrf5sdk-tools pkg generate \
  --hw-version 52 --sd-req 0x00 \
  --application /tmp/ot-ncp-ftd-dfu.hex \
  --application-version 7 \
  /tmp/ot-ncp-ftd-dfu.zip
```

**CRITICAL build notes:**
- Must use `-DOT_BOOTLOADER=USB` — this selects the correct linker script (app at 0x1000, after MBR)
- Using `-DOT_BOOTLOADER=UART` or no bootloader flag = app at 0x0000 = **WILL NOT BOOT** on PCA10059
- Pins MUST be overridden in transport-config.h (defaults are P0.06/P0.08 which are LEDs)
- The `script/build` wrapper does NOT work for our config — use cmake directly
