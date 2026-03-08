---
name: ccx-capture
description: Capture and analyze CCX (Thread/802.15.4) traffic from the Lutron network using the nRF 802154 sniffer. Use when asked to "capture ccx", "sniff thread", "record ccx traffic", "capture coap", or "start sniffer".
metadata:
  author: alexgompper
  version: "1.0.0"
  argument-hint: [duration-seconds] [label]
user_invocable: true
---

# CCX Traffic Capture

Capture decrypted Thread/802.15.4 traffic from the Lutron network using the nRF 802154 USB sniffer dongle.

## Prerequisites

- nRF 802154 Sniffer dongle plugged in (shows as `/dev/cu.usbmodem*` with product "nRF 802154 Sniffer")
- Wireshark installed at `/Applications/Wireshark.app` (provides tshark)
- Thread decryption keys configured in `~/Library/Application Support/Wireshark/ieee802154_keys`
- Channel 25 saved in Wireshark preferences (set via GUI once)

## How It Works

1. **Detect sniffer** — find the nRF 802154 sniffer serial port via `mcp__serial__list_serial_ports` or `tshark -D`
2. **Start capture** — run tshark in background writing to `captures/` directory
3. **User performs actions** (e.g., Designer transfer, button presses, app control)
4. **Stop capture** — Ctrl+C or duration timeout
5. **Analyze** — decode the pcapng with Thread decryption + CCX CBOR parsing

## Capture Commands

### Start a timed capture (recommended for Designer transfers)
```bash
tshark -i /dev/cu.usbmodem201401 \
  -w captures/ccx-<label>.pcapng \
  -a duration:<seconds> \
  -q
```

### Start an indefinite capture (stop with Ctrl+C)
```bash
tshark -i /dev/cu.usbmodem201401 \
  -w captures/ccx-<label>.pcapng \
  -q
```

### Quick capture with live packet count
```bash
tshark -i /dev/cu.usbmodem201401 \
  -w captures/ccx-<label>.pcapng \
  -a duration:<seconds>
```

## Analysis Commands

### Summary stats
```bash
tshark -r captures/<file>.pcapng -q -z io,stat,0
```

### View all decrypted CCX traffic (UDP port 9190)
```bash
tshark -r captures/<file>.pcapng -Y "udp.port == 9190" \
  -T fields -e frame.time_relative -e ipv6.src -e ipv6.dst -e udp.payload
```

### View CoAP traffic (programming/config)
```bash
tshark -r captures/<file>.pcapng -Y "coap" \
  -T fields -e frame.time_relative -e ipv6.src -e ipv6.dst \
  -e coap.type -e coap.code -e coap.mid -e coap.payload
```

### View CoAP with full details
```bash
tshark -r captures/<file>.pcapng -Y "coap" -V | head -200
```

### Decode CCX CBOR payloads using our tool
```bash
bun run tools/ccx-sniffer.ts --file captures/<file>.pcapng
```

### Detailed CCX analysis
```bash
bun run tools/ccx-analyzer.ts --file captures/<file>.pcapng types
bun run tools/ccx-analyzer.ts --file captures/<file>.pcapng timeline
bun run tools/ccx-analyzer.ts --file captures/<file>.pcapng devices
```

## Key Facts

- **Channel**: 25 (saved in Wireshark GUI preferences)
- **Thread master key**: in ieee802154_keys UAT (seq 0-5 pre-configured)
- **CCX UDP port**: 9190 (multicast to ff03::1, CBOR-encoded)
- **CoAP port**: 5683 (unicast, used for programming/config)
- **Processor address**: fd00::ff:fe00:2c0c (RA3 processor)
- **Capture dir**: `captures/` in project root
- **Naming convention**: `ccx-<descriptive-label>.pcapng`

## Typical Workflows

### Designer Transfer Capture
1. Start capture: `tshark -i /dev/cu.usbmodem201401 -w captures/ccx-designer-transfer.pcapng -q`
2. Open Designer, connect to processor, do full transfer
3. Wait for transfer to complete
4. Press Ctrl+C to stop capture
5. Analyze CoAP: `tshark -r captures/ccx-designer-transfer.pcapng -Y "coap"`

### Button Press Capture
1. Start capture with duration: `tshark -i /dev/cu.usbmodem201401 -w captures/ccx-button-test.pcapng -a duration:30 -q`
2. Press buttons on keypads during the 30 seconds
3. Decode: `bun run tools/ccx-sniffer.ts --file captures/ccx-button-test.pcapng`

### Device State Capture
1. Start capture, trigger lights via app/keypad, stop
2. Look for DEVICE_REPORT (type 27) and STATUS (type 41) messages
3. `bun run tools/ccx-analyzer.ts --file captures/<file>.pcapng devices`

## Troubleshooting

- **No packets**: Check sniffer is detected (`tshark -D | grep nrf`), verify channel 25 in Wireshark GUI
- **Encrypted payloads**: Thread keys missing or wrong seq — check `~/Library/Application Support/Wireshark/ieee802154_keys`
- **No CoAP**: CoAP is unicast between processor and devices — sniffer must be in radio range of both
- **Sniffer not found**: Port changes on replug — re-detect with `tshark -D`
