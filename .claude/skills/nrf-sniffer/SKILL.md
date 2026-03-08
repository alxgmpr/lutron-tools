---
name: nrf-sniffer
description: Flash the nRF52840 USB dongle with Nordic 802.15.4 sniffer firmware for Wireshark packet capture. Use when asked to "switch to sniffer", "capture packets", "nrf sniffer mode", or "reflash sniffer".
metadata:
  author: alexgompper
  version: "1.0.0"
user_invocable: true
---

# nRF52840 → 802.15.4 Sniffer Mode

Flash the nRF52840 USB dongle with Nordic 802.15.4 sniffer firmware for capturing Thread/CCX traffic in Wireshark.

## Steps

### 1. Kill ot-daemon if running
If ot-daemon is running, the user must Ctrl+C it in its terminal first.

### 2. Put dongle in DFU bootloader mode
Press the **reset button** on the dongle. The LED will pulse red when in bootloader mode.
Wait for the user to confirm the dongle is in DFU mode.

### 3. Find the DFU serial port
```bash
ls /dev/tty.usbmodem*
```

### 4. Flash sniffer firmware
```bash
nrfutil nrf5sdk-tools dfu usb-serial \
  -pkg ~/Downloads/nRF-Sniffer-for-802.15.4/nrf802154_sniffer_nrf52840dongle_dfu.zip \
  -p <PORT>
```

### 5. Verify sniffer is detected
```bash
/Applications/Wireshark.app/Contents/MacOS/tshark -D 2>/dev/null | grep -i nrf
```
Or check serial ports:
```bash
ls /dev/cu.usbmodem*
```

### 6. Ready to capture
Start a capture (channel 25 must be set in Wireshark GUI preferences):
```bash
tshark -i /dev/cu.usbmodem<ID> -w captures/ccx-<label>.pcapng -q
```
Or with a time limit:
```bash
tshark -i /dev/cu.usbmodem<ID> -w captures/ccx-<label>.pcapng -a duration:<seconds> -q
```

## Key Facts
- **Sniffer firmware**: `~/Downloads/nRF-Sniffer-for-802.15.4/nrf802154_sniffer_nrf52840dongle_dfu.zip`
- **Channel**: 25 (configured in Wireshark GUI, not on command line)
- **Thread decryption keys**: `~/Library/Application Support/Wireshark/ieee802154_keys`
- **Capture dir**: `captures/` in project root

## Analysis After Capture
See the `/ccx-capture` skill for analysis commands, or:
```bash
# CoAP programming traffic
tshark -r captures/<file>.pcapng -d udp.port==5683,coap \
  -Y 'udp.port==5683 && coap.opt.uri_path_recon' \
  -T fields -e frame.time_relative -e ipv6.src -e ipv6.dst \
  -e coap.code -e coap.opt.uri_path_recon -e data

# Runtime CCX traffic
tshark -r captures/<file>.pcapng -Y 'udp.port==9190' \
  -T fields -e frame.time_relative -e ipv6.src -e ipv6.dst -e data
```
