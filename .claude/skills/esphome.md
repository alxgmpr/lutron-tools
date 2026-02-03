# `/esphome` - ESP32/CC1101 Embedded Development

Develop and deploy the ESP32 firmware that handles CC1101 radio communication for CCA packet capture and transmission.

## Usage

```
/esphome              # Show status and available commands
/esphome flash        # Compile and flash via OTA
/esphome logs         # View live ESP32 logs
/esphome compile      # Compile without flashing
/esphome clean        # Clean build artifacts
```

---

## Quick Start

Based on the arguments provided, take the appropriate action:

### No arguments
Show the current status:
1. Check if ESPHome is installed
2. Show the device configuration summary
3. List available commands

### "flash"
Compile and flash the firmware via OTA:
```bash
esphome run esphome/cca-proxy.yaml --device cca-proxy.local --no-logs
```

### "logs"
View live ESP32 logs:
```bash
esphome logs esphome/cca-proxy.yaml --device cca-proxy.local
```

### "compile"
Compile without flashing (useful to check for errors):
```bash
esphome compile esphome/cca-proxy.yaml
```

### "clean"
Clean build artifacts:
```bash
rm -rf esphome/.esphome/build/cca-proxy
```

---

## Proactive Usage

Use this skill automatically when:
- User asks to modify ESP32/CC1101 code
- User mentions flashing, OTA, or firmware
- User wants to view device logs
- User is debugging radio communication issues
- Editing files in `esphome/` directory

---

## Architecture

```
ESP32 (cca-proxy) <-- SPI --> CC1101 Radio <-- RF --> CCA Devices
         |
         +-- UDP --> Backend Server (port 5000)
```

**Key Components:**
- **CC1101 Radio Driver** - Handles packet encoding, CRC, sync words
- **UDP Relay** - Streams packets to backend with minimal latency
- **Packet TX** - Transmits packets on command from backend

---

## Critical Files

| File | Purpose |
|------|---------|
| `esphome/cca-proxy.yaml` | Main ESPHome configuration |
| `esphome/custom_components/cc1101_cca/cc1101_cca.cpp` | CC1101 driver implementation |
| `esphome/custom_components/cc1101_cca/cc1101_cca.h` | CC1101 driver header |
| `esphome/udp_stream.h` | UDP packet relay |

---

## CC1101 Knowledge Base

### Radio Configuration
- **Frequency:** 433.92 MHz (CCA band)
- **Modulation:** 2-FSK
- **Data Rate:** ~38.4 kbps
- **Sync Word:** CCA-specific preamble

### Packet Format (over the air)
```
[Preamble] [Sync Word] [Length] [Payload] [CRC16]
```

### Common Issues
- **No packets received:** Check SPI wiring, GDO pins
- **CRC errors:** Sync word or frequency drift
- **TX not working:** Check GDO0/GDO2 pin configuration
- **OTA fails:** Device not on network, wrong hostname

---

## Debugging Tips

1. **Check device is online:**
   ```bash
   ping cca-proxy.local
   ```

2. **View raw SPI traffic:** Add debug logging in `cc1101_cca.cpp`

3. **Check UDP relay:** Verify backend receives packets on port 5000

4. **Monitor RSSI:** Strong signal = -60 to -30 dBm, weak = < -80 dBm

---

## Protocol Reference

Load CCA protocol details from:
- `protocol/cca.yaml` - Packet type definitions
- `docs/cca-pairing-protocol.md` - Pairing sequence documentation
