---
name: ccx-wiz-bridge
description: "CCX→WiZ bridge: captures Lutron Thread traffic and forwards level/scene/button commands to WiZ smart bulbs over UDP. Use when working on bridge code, warm dimming, WiZ integration, or pairing config."
metadata:
  author: alexgompper
  version: "1.0.0"
user_invocable: false
---

# CCX → WiZ Bridge

Captures Lutron CCX (Thread/802.15.4) traffic via nRF sniffer and forwards dimming/scene/button commands to WiZ smart bulbs over UDP.

## Key Files

| File | Purpose |
|------|---------|
| `bridge/main.ts` | Main bridge — tshark capture → CBOR decode → WiZ UDP |
| `tools/wiz-test.ts` | Manual WiZ bulb control (getPilot/setPilot) |
| `lib/warm-dim.ts` | B-spline warm dimming curves (brightness → Kelvin) |
| `config/ccx-bridge.example.json` | Config template with all options |
| `data/virtual-device.json` | Active config (single bulb: zone 5147 → 10.0.0.50) |

## Running

```bash
# Start bridge (requires nRF sniffer dongle — see /nrf sniffer)
npx tsx bridge/main.ts --serial                      # uses data/virtual-device.json
npx tsx bridge/main.ts --config config/my.json       # custom config
npx tsx bridge/main.ts --decrypt                     # native MAC decryption (no Wireshark keys needed)
npx tsx bridge/main.ts -v                            # verbose logging

# Test WiZ bulb directly
bun run tools/wiz-test.ts              # getPilot (query state)
bun run tools/wiz-test.ts 50           # set 50%
bun run tools/wiz-test.ts off          # turn off
bun run tools/wiz-test.ts --ip 10.0.0.51 75  # different bulb
```

**Runtime**: Bridge uses `npx tsx` (Node.js), not Bun — requires native AES-128-CCM for Thread decryption.

## Config Format

```json
{
  "pairings": [
    {
      "name": "Living Room Lamp",
      "lutron": {
        "zoneId": 1234,
        "serials": [123, 456],
        "presets": [100, 103, 106, 109]
      },
      "wiz": { "ip": "10.0.0.50", "port": 38899 },
      "warmDimming": true,
      "warmDimCurve": "default",
      "warmDimMin": 1800,
      "warmDimMax": 2800
    }
  ],
  "defaults": {
    "wizPort": 38899,
    "warmDimming": false,
    "warmDimCurve": "default",
    "wizDimScaling": true
  }
}
```

### Config Fields

- **`lutron.zoneId`** — CCX zone ID (from LEAP dump) for LEVEL_CONTROL matching
- **`lutron.serials`** — Dimmer serial numbers for DEVICE_REPORT matching (physical touch)
- **`lutron.presets`** — Pico preset IDs for BUTTON_PRESS matching
- **`wiz.ip`** — WiZ bulb IP address (UDP:38899)
- **`warmDimming`** — Enable brightness→CCT curve (adds `temp` to setPilot)
- **`warmDimCurve`** — Curve name: `default`, `halogen`, `finire2700`, `finire3000`
- **`wizDimScaling`** — Scale Lutron 1-100% → WiZ 10-100% (WiZ min dimming is ~10%)

## Message Types Handled

| CCX Message | Source | Matching |
|---|---|---|
| LEVEL_CONTROL | App, scenes, processor | `zoneId` from multicast |
| DEVICE_REPORT | Physical dimmer touch | `serial` from unicast |
| BUTTON_PRESS | Pico button press | `presetId` → zone lookup |
| DIM_HOLD / DIM_STEP | Pico raise/lower | Software ramp (21%/sec) |

## WiZ UDP Protocol

Port 38899, JSON over UDP. Key methods:

```json
// Query state
{"method": "getPilot", "params": {}}

// Set level (dimming 10-100, or state:false for off)
{"method": "setPilot", "params": {"state": true, "dimming": 75}}

// Set level + warm dim CCT
{"method": "setPilot", "params": {"state": true, "dimming": 75, "temp": 2200}}

// Turn off
{"method": "setPilot", "params": {"state": false}}
```

## Deduplication

The RA3 processor sends 6-7 copies of each CCX command. The bridge deduplicates with a 2000ms window using composite keys:
- `lc:${zoneId}:${sequence}` for LEVEL_CONTROL
- `dr:${serial}:${level}` for DEVICE_REPORT
- `bp:${zone}:${sequence}` for BUTTON_PRESS

## Warm Dimming

B-spline curves from Designer's SqlModelInfo.mdf map brightness% → color temperature (Kelvin):
- `default`: 1800K (dim) → 2800K (bright)
- `halogen`: 1798K → 2802K
- `finire2700`: 1784K → 2720K
- `finire3000`: 1794K → 3040K

API: `evalWarmDimCurve(curve, brightnessPercent)` → CCT in Kelvin.

## Dimming Ramp (Raise/Lower)

Pico raise/lower buttons trigger software-simulated ramps:
- Rate: 100% over 4.75 seconds (≈21%/sec)
- Updates sent every 50ms
- DIM_HOLD starts ramp, release stops it

## Architecture

```
nRF sniffer dongle (802.15.4 ch25)
  → tshark (pcap pipe or live capture)
  → bridge/main.ts (CBOR decode + dedup + zone matching)
  → WiZ bulb (UDP:38899 setPilot JSON)
```

The bridge also supports `--decrypt` mode which bypasses Wireshark's Thread decryption layer and decrypts 802.15.4 MAC frames natively using `lib/thread-crypto.ts`.
