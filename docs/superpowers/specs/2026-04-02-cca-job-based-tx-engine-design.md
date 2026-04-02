# CCA Job-Based TX Engine

**GLAB-64** | 2026-04-02 | Status: Design approved

## Problem

The CCA radio stack is ~75% complete but has interconnected architectural gaps:

1. **Blocking TX** — `cca_cmd_execute()` blocks the CCA task for up to 5s during pairing sequences. RX stops entirely during TX bursts.
2. **No universal TX function** — each command type has custom burst logic with hardcoded delays. Adding new packet types requires writing new builder code.
3. **TDMA bypass** — the immediate TX queue (`cca_tx_immediate_queue`) fires packets without slot discipline, stomping other devices' slots during pairing.

These three gaps are the same problem: TX doesn't go through the TDMA engine. The fix is a single architectural change.

## Decision: Device Type Correction (0x2F rule) — Deferred

The QSM firmware's 0x2F subtraction rule is an internal dispatch index for routing packets to handler functions inside the HCS08 MCU. It maps 256 possible type bytes down to 24 handler table entries. Our decoder dispatches on raw type bytes directly (0x80, 0x88, etc.) and this works correctly. Adding the correction would be protocol-complete but provides no functional benefit. Deferred.

## Architecture

### Current flow (blocking)

```
shell/stream command
  → cca_cmd_enqueue(CcaCmdItem)
  → CCA task dequeues
  → cca_cmd_execute() blocks:
      stop RX → build packet → encode → transmit → delay → repeat × N → restart RX
```

### New flow (non-blocking)

```
shell/stream command
  → cca_cmd_enqueue(CcaCmdItem)
  → CCA task dequeues
  → cca_cmd_to_jobs(CcaCmdItem) → TdmaJobGroup
  → cca_tdma_submit_group(group)  [returns immediately]

CCA task main loop (every 2ms):
  → cca_check_rx()                [process incoming packets]
  → cca_tdma_poll()               [fire at most 1 TX packet if slot is open]
  → dequeue next command if any
```

RX is never stopped for more than ~2ms (one packet encode + transmit). Multiple commands interleave naturally through TDMA slot scheduling.

## Data Structures

### TdmaPacket — single TX packet ready to fire

```c
struct TdmaPacket {
    uint8_t data[53];     // raw payload (pre-CRC, pre-8N1)
    uint8_t len;          // 22 (short) or 51 (long)
    uint8_t type_rotate;  // 0=fixed type byte, 1=rotate 81/82/83 per retransmit
};
```

### TdmaPhase — one burst within a command

```c
struct TdmaPhase {
    TdmaPacket packet;
    uint8_t retransmits;     // frames to repeat (5=normal, 10=pairing)
    uint16_t post_delay_ms;  // delay after phase completes before next phase (0=immediate)
};
```

### TdmaJobGroup — a complete command decomposed into phases

```c
#define TDMA_MAX_PHASES 8
#define TDMA_MAX_GROUPS 4

struct TdmaJobGroup {
    TdmaPhase phases[TDMA_MAX_PHASES];
    uint8_t phase_count;

    // Managed by engine:
    uint8_t slot;               // assigned TDMA slot
    uint8_t current_phase;      // 0..phase_count-1
    uint8_t current_retransmit; // 0..retransmits-1
    uint32_t next_fire_ms;      // tick at which next packet fires
    bool active;                // slot in use

    void (*on_complete)(void* ctx);  // optional completion callback
    void* ctx;
};
```

### Constants

- `TDMA_MAX_PHASES = 8` — button press needs 2, pairing needs up to 6
- `TDMA_MAX_GROUPS = 4` — max concurrent commands (pairing + level + beacon + one spare)
- Frame period: 75ms (8 slots × ~9.375ms each)
- Retransmit counts: 5 (normal), 7 (button), 10 (pairing/scene), 20 (bridge level)

## Command Decomposition

Each command type becomes a builder function returning a `TdmaJobGroup`.

### Button press → 2 phases

```
Phase 0: SHORT_A packet, 7 retransmits, post_delay=0ms
Phase 1: LONG_A packet, 7 retransmits, post_delay=0ms
```

Total: 14 packets across ~1050ms, non-blocking. Same slot both phases.

### Bridge level → 1 phase

```
Phase 0: SET_LEVEL (type_rotate=1), 20 retransmits, post_delay=0ms
```

Total: 20 packets across ~1500ms. Type rotates 81→82→83 per retransmit.

### Beacon → 1 phase

```
Phase 0: BEACON_91 packet, 5 retransmits, post_delay=0ms
```

### Pairing (bridge pair) → 3 phases

```
Phase 0: B9 beacon, 10 retransmits, post_delay=500ms
Phase 1: B0 announce, 10 retransmits, post_delay=200ms
Phase 2: config packet, 5 retransmits, post_delay=0ms
```

### Raw command → 1 phase

```
Phase 0: user-supplied packet, user-supplied retransmit count, post_delay=0ms
```

## TDMA Engine Changes

### New in `cca_tdma.cpp`

The engine currently manages individual `CcaTdmaJob` structs. Replace with:

**Active group array:** `TdmaJobGroup groups_[TDMA_MAX_GROUPS]` — up to 4 concurrent job groups.

**`cca_tdma_submit_group(const TdmaJobGroup* group) → bool`**
1. Find a free slot in `groups_[]`
2. Copy the group, assign a TDMA slot (pick unoccupied from frame map)
3. Set `current_phase = 0`, `current_retransmit = 0`
4. Compute `next_fire_ms` based on slot position in current frame
5. Mark `active = true`, return success

**`cca_tdma_poll()` — modified**
For each active group:
1. If `HAL_GetTick() < next_fire_ms` → skip (not time yet)
2. Stop RX, encode + transmit the current phase's packet, restart RX
3. Set sequence byte: `seq = slot + (current_retransmit * slot_count)`
4. If `type_rotate`: set type byte to `0x81 + (current_retransmit % 3)`
5. Increment `current_retransmit`
6. If retransmits exhausted for this phase:
   - Advance `current_phase`
   - If `post_delay_ms > 0`: set `next_fire_ms = now + post_delay_ms`
   - Else: set `next_fire_ms` to next slot window
7. If all phases done: call `on_complete`, mark `active = false`

At most one packet fires per poll cycle to keep RX interruption minimal.

**`cca_tdma_cancel_all()`** — mark all groups inactive. Used on mode changes or error recovery.

**`cca_tdma_is_idle() → bool`** — true when no active groups.

### Preserved (no changes)

- Frame sync logic (anchor tracking, stride histogram, confidence scoring)
- Device slot tracking and occupancy map
- Stale device eviction
- Telemetry reporting (`cca_tdma_get_state`, `cca_tdma_get_devices`)

### Removed

- `cca_tx_immediate_queue` — deleted entirely
- `cca_tx_send_immediate()` — deleted
- Individual `CcaTdmaJob` struct — replaced by `TdmaJobGroup`

## Command Layer Changes

### `cca_commands.h` — new builder API

```c
TdmaJobGroup cca_jobs_button(uint32_t device_id, uint8_t button);
TdmaJobGroup cca_jobs_level(uint32_t src, uint32_t dst, uint8_t pct, uint8_t fade_qs);
TdmaJobGroup cca_jobs_beacon(uint32_t zone_id, uint8_t type_byte);
TdmaJobGroup cca_jobs_pico_level(uint32_t device_id, uint8_t pct);
TdmaJobGroup cca_jobs_bridge_pair(uint32_t device_id, uint32_t target_id, uint8_t duration);
TdmaJobGroup cca_jobs_raw(const uint8_t* payload, uint8_t len, uint8_t retransmits);
```

Each builder uses the existing `cca_build_*` functions from `cca_tx_builder.h` to fill `TdmaPacket.data`.

### `cca_commands.cpp` — simplified

`cca_cmd_execute()` becomes a dispatcher that calls the appropriate `cca_jobs_*` builder then `cca_tdma_submit_group()`. No blocking loops, no `cca_timer_wait_until()`, no `cc1101_stop_rx()`/`cc1101_start_rx()` calls. The entire function becomes ~50 lines (switch on cmd type → build group → submit).

### `cca_task.cpp` — simplified main loop

```c
for (;;) {
    cca_check_rx();
    cca_tdma_poll();

    CcaCmdItem cmd;
    if (xQueueReceive(cmd_queue, &cmd, 0) == pdTRUE) {
        TdmaJobGroup group = cca_cmd_to_jobs(&cmd);
        if (!cca_tdma_submit_group(&group)) {
            printf("[cca] TX engine full, command dropped\r\n");
        }
    }

    vTaskDelay(pdMS_TO_TICKS(2));
}
```

## Format Coverage Expansion

Incremental — no architectural changes needed. Add format extraction to `parse_type_specific()` in `cca_decoder.h` as formats are encountered in captures.

Priority additions:
- ZONE (0x28) — 40-byte zone assignment, format byte at offset 6
- CCI formats (0x29-0x2C) — VCRX contact closure inputs
- Property set (0x64 fixed, 0x65 variable)

No speculative code. Only add formats we can validate against real packet captures.

## Testing

### Unit tests (firmware/tests/)

- `test_tdma_jobs.cpp` — job group lifecycle: submit, advance phases, complete
- `test_tdma_timing.cpp` — verify slot timing, retransmit spacing, post-delay behavior
- `test_cmd_builders.cpp` — verify each `cca_jobs_*` builder produces correct packet bytes and phase counts

### Integration (manual, with Nucleo + CC1101)

- Button press: verify 14 packets in 2 phases, correct type bytes, correct slot
- Bridge level: verify 20 packets with type rotation
- Pairing: verify multi-phase with post-delays, slot discipline maintained
- Concurrent commands: submit level + button simultaneously, verify interleaving
- RX during TX: verify packets are still received between TX fires

## Files Changed

| File | Change |
|------|--------|
| `firmware/src/cca/cca_tdma.h` | New structs (TdmaPacket, TdmaPhase, TdmaJobGroup), new API |
| `firmware/src/cca/cca_tdma.cpp` | Replace job engine with group-based engine |
| `firmware/src/cca/cca_commands.h` | Add `cca_jobs_*` builder declarations |
| `firmware/src/cca/cca_commands.cpp` | Replace blocking execute with job builders + submit |
| `firmware/src/cca/cca_task.cpp` | Simplify main loop, remove immediate queue |
| `firmware/tests/test_tdma.cpp` | New job group tests |
| `firmware/tests/test_main.cpp` | Register new test files |

## What's NOT in scope

- Sync serial mode (CC1101 PKTCTRL0 PKT_FORMAT=01) — packet mode works, defer
- Device type correction (0x2F rule) — internal QSM dispatch, not needed
- Completion callbacks wired to shell/stream — add later when needed
- Adaptive slot count (4/16/32/64 slot frames) — 8-slot is sufficient
