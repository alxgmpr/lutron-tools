---
name: nrf-ot
description: Flash the nRF52840 USB dongle with OpenThread RCP firmware and start ot-daemon for Thread CLI access. Use when asked to "switch to ot", "start openthread", "nrf ot mode", or "join thread network".
metadata:
  author: alexgompper
  version: "1.0.0"
user_invocable: true
---

# nRF52840 → OpenThread RCP Mode

Flash the nRF52840 USB sniffer dongle with OpenThread RCP firmware and start ot-daemon so we can join the Lutron Thread network and send/receive packets (CoAP, ping, etc).

## Steps

### 1. Put dongle in DFU bootloader mode
Press the **reset button** on the dongle. The LED will pulse red when in bootloader mode.
Wait for the user to confirm the dongle is in DFU mode.

### 2. Find the DFU serial port
```bash
ls /dev/tty.usbmodem*
```

### 3. Flash RCP firmware
The DFU package must be generated from the ELF first:
```bash
/Applications/ArmGNUToolchain/15.2.rel1/arm-none-eabi/bin/arm-none-eabi-objcopy -O ihex \
  ~/lutron-tools/src/ot-nrf528xx/build/nrf52840-usb/bin/ot-rcp /tmp/ot-rcp.hex

nrfutil nrf5sdk-tools pkg generate --hw-version 52 --sd-req 0x00 \
  --application /tmp/ot-rcp.hex --application-version 1 /tmp/ot-rcp-dfu.zip

nrfutil nrf5sdk-tools dfu usb-serial -pkg /tmp/ot-rcp-dfu.zip -p <PORT>
```

If the RCP firmware needs rebuilding:
```bash
cd ~/lutron-tools/src/ot-nrf528xx
PATH="/Applications/ArmGNUToolchain/15.2.rel1/arm-none-eabi/bin:$PATH" \
  OT_CMAKE_BUILD_DIR=build/nrf52840-usb ./script/build nrf52840 USB_trans -DOT_BOOTLOADER=USB
```

### 4. Find the new serial port (changes after flash)
```bash
ls /dev/tty.usbmodem*
```

### 5. Start ot-daemon
**User must run this in a separate terminal** (requires sudo for TUN interface):
```bash
sudo ~/bin/ot-daemon -I wpan0 -v 'spinel+hdlc+uart://<PORT>' --data-path /tmp/ot-data
```

### 6. Configure and join Thread network
**User must run in another terminal** with sudo:
```bash
sudo ~/bin/ot-ctl -I utun<N>
```
The interface name is printed by ot-daemon (e.g., "Thread interface: utun8"). Then in the ot-ctl shell:
```
dataset clear
dataset channel 25
dataset panid 0x62EF
dataset extpanid 0D02EFA82C989231
dataset networkkey 2009F0F102B4EEA86F31DC701D8E3D62
dataset commit active
ifconfig up
thread start
```
Wait ~5 seconds, then check: `state` — should show `router` or `child`.

### 7. Ready
Once joined, you can:
- Ping devices: `ping fd0d:2ef:a82c::ff:fe00:<rloc>`
- Send CoAP via udp6 transport: `NUCLEO_HOST= bun run tools/ccx-coap-send.ts ...`
- Discover devices: `ping ff03::1`

## Key Facts
- **Thread credentials**: channel 25, PAN ID 0x62EF, in .env file
- **ot-daemon binaries**: `~/bin/ot-daemon`, `~/bin/ot-ctl`
- **RCP firmware source**: `~/lutron-tools/src/ot-nrf528xx/`
- **Socket**: `/tmp/openthread-utun<N>.sock` (auto-created by ot-daemon)
