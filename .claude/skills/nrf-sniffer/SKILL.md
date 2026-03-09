---
name: nrf-sniffer
description: Flash the nRF52840 USB dongle with Nordic 802.15.4 sniffer firmware for Wireshark packet capture. Use when asked to "switch to sniffer", "capture packets", "nrf sniffer mode", or "reflash sniffer".
metadata:
  author: alexgompper
  version: "2.0.0"
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

The DFU zip must be generated from the hex in the cloned repo:
```bash
# Clone repo if not present
ls ~/Downloads/nRF-Sniffer-for-802.15.4/nrf802154_sniffer/nrf802154_sniffer_nrf52840dongle.hex || \
  git clone https://github.com/NordicSemiconductor/nRF-Sniffer-for-802.15.4.git ~/Downloads/nRF-Sniffer-for-802.15.4

# Generate DFU zip from hex (only needed once)
ls ~/Downloads/nRF-Sniffer-for-802.15.4/nrf802154_sniffer_nrf52840dongle_dfu.zip || \
  nrfutil nrf5sdk-tools pkg generate --hw-version 52 --sd-req 0x00 --application-version 1 \
    --application ~/Downloads/nRF-Sniffer-for-802.15.4/nrf802154_sniffer/nrf802154_sniffer_nrf52840dongle.hex \
    ~/Downloads/nRF-Sniffer-for-802.15.4/nrf802154_sniffer_nrf52840dongle_dfu.zip

# Flash
nrfutil nrf5sdk-tools dfu usb-serial \
  -pkg ~/Downloads/nRF-Sniffer-for-802.15.4/nrf802154_sniffer_nrf52840dongle_dfu.zip \
  -p <PORT>
```

### 5. Set up Python venv for extcap (only needed once)
```bash
ls ~/Downloads/nRF-Sniffer-for-802.15.4/venv/bin/activate || \
  (cd ~/Downloads/nRF-Sniffer-for-802.15.4 && python3 -m venv venv && source venv/bin/activate && pip install pyserial)
```

The Wireshark extcap script at `~/Library/Application Support/Wireshark/extcap/nrf802154_sniffer` sources this venv.

### 6. Verify sniffer is detected
```bash
/Applications/Wireshark.app/Contents/MacOS/tshark -D 2>/dev/null | grep -i nrf
```
Or check serial ports:
```bash
ls /dev/cu.usbmodem*
```

### 7. Start capture
Channel 25 must be set in Wireshark GUI preferences (extcap config).
```bash
# Indefinite capture (Ctrl+C to stop)
tshark -i /dev/cu.usbmodem<ID> -w captures/ccx-<label>.pcapng -q

# With time limit (seconds)
tshark -i /dev/cu.usbmodem<ID> -w captures/ccx-<label>.pcapng -a duration:<seconds> -q
```

**IMPORTANT:** Use long durations for transfers — a full Designer transfer takes 10+ minutes. Use `-a duration:900` (15 min) or longer.

## Key Facts
- **Sniffer firmware source**: `https://github.com/NordicSemiconductor/nRF-Sniffer-for-802.15.4` (clone, generate DFU zip from hex)
- **Channel**: 25 (configured in Wireshark GUI extcap preferences, not CLI)
- **Thread decryption keys**: `~/Library/Application Support/Wireshark/ieee802154_keys`
- **Capture dir**: `captures/` in project root
- **Extcap script**: `~/Library/Application Support/Wireshark/extcap/nrf802154_sniffer`
- **Python venv**: `~/Downloads/nRF-Sniffer-for-802.15.4/venv/`

## Analysis After Capture
See the `/ccx-capture` skill for analysis commands, or:
```bash
# CoAP programming traffic (port 5683)
tshark -r captures/<file>.pcapng -d udp.port==5683,coap \
  -Y 'udp.port==5683 && coap.opt.uri_path_recon' \
  -T fields -e frame.time_relative -e ipv6.src -e ipv6.dst \
  -e coap.code -e coap.opt.uri_path_recon -e data

# CCX multicast traffic (port 9190)
tshark -r captures/<file>.pcapng -Y 'udp.port==9190' \
  -T fields -e frame.time_relative -e ipv6.src -e ipv6.dst -e data

# TMF traffic (port 61631)
tshark -r captures/<file>.pcapng -Y 'udp.port==61631' \
  -T fields -e frame.time_relative -e ipv6.src -e ipv6.dst -e data

# All unique IPv6 addresses
tshark -r captures/<file>.pcapng -T fields -e ipv6.src -e ipv6.dst | \
  tr '\t' '\n' | sort -u | grep -v '^$'
```
