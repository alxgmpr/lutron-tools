---
name: cca-debug
description: "Debug CCA/CCX packet reception issues — systematic diagnosis when packets stop showing up, CC1101 register problems, firmware RX path tracing"
metadata:
  author: alex
  version: "1.0"
user_invocable: false
---

# Debugging CCA/CCX Packet Reception

Use this when packets stop showing up in the CLI, packet counts drop, or specific devices disappear.

## First Response: Triage

When the user reports "no packets" or "packets stopped", immediately gather diagnostics. Have them run `status` in the Nucleo shell:

```
status
```

This shows:
- **CCA rx/tx/drop** counts — are packets being decoded at all?
- **CC1101 overflow/runt** — is the FIFO overflowing?
- **IRQ count** — is GDO0 firing (radio detecting preambles)?
- **Sync hit/miss** — are sync words being found in accumulated data?
- **Ring in/dropped** — are bytes flowing from FIFO to accumulator?
- **Restart counts** — timeout, overflow, manual, packet
- **ISR latency** — is the task responding to interrupts promptly?
- **CCX rx/tx/joined** — is the other radio working? (isolates board vs radio issues)

### Decision Tree

```
CCX working, CCA rx=0?
  → CCA-specific issue (radio config, decoder, or task)

IRQ count > 0 but rx=0?
  → Radio detects preambles but packets aren't decoding
  → Check: ring in=0? → FIFO not being drained (task stuck or SPI issue)
  → Check: sync hit > 0 but rx=0? → Decoder rejecting everything
  → Check: drop > 0? → Packets decode but fail CRC (radio config issue)

IRQ count = 0?
  → Radio not detecting anything
  → CC1101 may be misconfigured or not in RX mode
  → Check cc1101_is_rx_active(), try `cca tune show`

Both CCA and CCX dead?
  → Board-level issue: power, Ethernet, task scheduler
```

## Common Causes

### 1. CC1101 Register Changes Killed RX

**This is the #1 cause.** CC1101 register changes can make the radio completely deaf with no obvious error. The symptoms are: IRQs still fire (barely), but zero decoded packets.

**Dangerous registers** (caused total RX failure when changed together):
- `MCSM2=0x74` — RX_TIME_RSSI=1 terminates reception on weak RSSI. **Most dangerous.**
- `FSCTRL1=0x0F` — IF frequency change (152→380kHz). Untested in isolation.
- `AGCCTRL0=0xFF` — Max AGC hysteresis. Causes near-field saturation (close devices fail, distant ones work). Reverted to 0x91.
- `PKTCTRL1=0x04` — APPEND_STATUS changes FIFO accumulation from 80→82 bytes. Added dead time.
- `MCSM1=0x33` — CCA_MODE=11 killed RX entirely. But MCSM1=0x0F (only RXOFF change) works fine.

**Known-good baseline registers:**
```
FSCTRL1=0x06  MCSM1=0x0F  MCSM0=0x18  AGCCTRL0=0x91
TEST2=0xAC    PKTCTRL1=0x00  PKTCTRL0=0x00  FIFOTHR=0x07
```

**Rule: NEVER batch CC1101 register changes.** Change ONE register, clean build (`rm -rf build`), flash, test pico RX, confirm working, commit. Then change the next.

### 2. Build Cache (Stale Object Files)

**Always nuke the build directory before building firmware:**
```bash
cd firmware && rm -rf build && cmake -B build -DCMAKE_TOOLCHAIN_FILE=cmake/arm-none-eabi.cmake && make -C build -j8
```

Incremental builds (`make -C build`) can link stale .o files that reference old struct layouts, wrong function signatures, or missing symbols. This causes silent runtime failures — the firmware boots, shell works, but CCA behavior is wrong.

### 3. CC1101 Needs Power Cycle

If firmware reverts don't fix RX, the CC1101 may be stuck in a bad analog state. The SRES strobe in `cc1101_init()` resets digital registers but not always the analog front-end.

**Fix:** Physically unplug the Nucleo USB cable, wait 5 seconds, plug back in. This is the only way to guarantee a full CC1101 power-on reset.

### 4. Task Crash in RX Callback

If `on_rx_packet()` crashes (e.g., decoder bug, stack overflow, null deref), the CCA FreeRTOS task dies silently. Symptoms:
- Shell still works (different task)
- CCX still works (different task)
- CCA rx=0, irq stops incrementing
- No UART output from CCA task

The CCA task has 2048 bytes of stack. The decoder allocates ~200 bytes per decode attempt (decoded[56] + tolerant[56] + tracked[56] + err arrays). Deep call chains can overflow.

### 5. Decoder Changes Breaking Packet Parsing

The decoder in `cca_decoder.h` runs in the RX hot path. New decoder code (e.g., additional fallback paths) adds CPU time per packet. If decode takes too long, the CC1101 FIFO overflows before the next `cc1101_check_rx()` call.

At 62.5kbps with a 64-byte FIFO, overflow happens in ~8ms. The decode + FIFO drain must complete within this window.

**Safe pattern for decoder changes:** Add new decode paths as FALLBACKS after existing paths, not replacing them. If the existing strict/tolerant decoders succeed, the new path never runs.

## Shell Diagnostic Commands

```
status              — full system overview (CCA + CCX + ETH + heap)
cca tune show       — current CC1101 register values
cca tune reg get XX — read specific CC1101 register (hex address)
cca log on/off      — enable/disable UART packet logging (bypasses TCP stream)
tdma status         — TDMA frame sync state, confidence, slot occupancy
tdma slots          — per-device slot assignments with stride/confidence
```

### Verifying CC1101 Is Alive

```
cca tune reg get 31  — read MARCSTATE (should be 0x0D = RX)
cca tune reg get 3B  — read RXBYTES (should fluctuate if receiving)
cca tune show        — dump all tunable params
```

If MARCSTATE is not 0x0D, the radio isn't in RX mode. Check if `cc1101_start_rx()` was called.

## CLI-Side Debugging

If firmware shows rx > 0 but CLI shows no packets:

1. **Check TCP connection**: CLI status bar should show "Connected" with client count
2. **Check stream**: Run `stream` in shell — shows connected clients and bytes sent
3. **Check filters**: CLI might have quiet mode on (`q` toggle) or protocol filter active
4. **Raw test**: `cca log on` in shell prints packets to UART directly — if these show but CLI doesn't, the issue is in the TCP stream or CLI parser

## Git Archaeology

When debugging a regression, find what changed:
```bash
# What changed in CCA code since last known-good?
git log --oneline -- firmware/src/cca/
git diff <good-commit>..HEAD -- firmware/src/cca/cc1101.c

# Specifically check CC1101 register values
git log -p -- firmware/src/cca/cc1101.c | grep -A1 -B1 "write_register"
```

## Incremental Testing Protocol

When making firmware changes that could affect RX:

1. **Baseline**: Confirm picos work before any changes
2. **One change at a time**: Make exactly one functional change
3. **Clean build**: `rm -rf build && cmake ... && make ...`
4. **Flash and test**: `make flash`, press picos, check `status` counters
5. **Commit if good**: Lock in the working state before the next change
6. **If broken**: `git checkout HEAD -- firmware/` to revert, rebuild, reflash, confirm recovery

Never make multiple changes between tests. Never assume a change is safe because it "shouldn't affect RX".
