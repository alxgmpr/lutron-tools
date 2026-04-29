# OTA Full-TX Orchestration Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** Build the full PowPak OTA orchestration two ways — firmware-side `CCA_CMD_OTA_FULL_TX` and host-side `tools/cca/ota-tx.ts` — both consuming the existing `cca_ota_tx.h` builders (PR #46), so we have an end-game-fast path (firmware) and a fast-iteration path (host) for Phase 2b of the PowPak RMJ→LMJ conversion attack.

**Architecture:**
- **Track 1 (Firmware)**: New stream-protocol commands upload an LDF body to a static SRAM buffer (~110 KB in RAM_D1, gitignored gen target). New shell command `cca ota-tx <subnet> <serial>` triggers `exec_ota_full_tx` which walks the buffer with `OtaChunkIter`, emitting BeginTransfer + 3,300× TransferData + ChangeAddrOff at the captured 75ms cadence.
- **Track 2 (Host)**: Pure TypeScript driver. New `lib/cca-ota-tx-builder.ts` mirrors the C++ byte layout; tests against captured-on-air ground truth. Driver `tools/cca/ota-tx.ts` reads LDF, builds packets, sends via `STREAM_CMD_TX_RAW_CCA = 0x01` UDP datagrams. Supports `--dry-run` (build + log packets only).

**Tech Stack:** STM32 C++ (FreeRTOS), TypeScript (Node tsx), live-capture JSONL ground truth.

---

## Track 2: Host-side `tools/cca/ota-tx.ts` (do first — pure TS, fastest to verify)

### Task A1: TS-side packet builder module

**Files:**
- Create: `lib/cca-ota-tx-builder.ts`
- Test: `test/cca-ota-tx-builder.test.ts`

The TS module mirrors `firmware/src/cca/cca_ota_tx.h` exactly. Same byte layouts, same return values, same validation. Each test asserts byte-for-byte equality against the captured-on-air ground truth (subset of `data/captures/cca-ota-20260428-190439.packets.jsonl`).

- [ ] **Step A1.1: Write failing test for `buildBeginTransfer`**

`test/cca-ota-tx-builder.test.ts`:

```typescript
import { describe, test } from "node:test";
import assert from "node:assert/strict";
import { buildBeginTransfer } from "../lib/cca-ota-tx-builder";

// Captured ground truth: 92 01 a1 ef fd 00 21 0e 00 06 fe 80 20 fe 06 00 02 20 00 00 00 1f
// (sequence byte [1] = 0x00 in builder output — TDMA engine sets it)
const EXPECT_BEGIN_TRANSFER = new Uint8Array([
  0x92, 0x00, 0xa1, 0xef, 0xfd, 0x00, 0x21, 0x0e,
  0x00, 0x06, 0xfe, 0x80, 0x20, 0xfe, 0x06, 0x00,
  0x02, 0x20, 0x00, 0x00, 0x00, 0x1f,
]);

describe("buildBeginTransfer", () => {
  test("matches captured DVRF-6L OTA ground truth", () => {
    const pkt = buildBeginTransfer(0xeffd, 0x06fe8020);
    assert.deepEqual(pkt, EXPECT_BEGIN_TRANSFER);
    assert.equal(pkt.length, 22);
  });

  test("encodes target serial big-endian at bytes 9..12", () => {
    const pkt = buildBeginTransfer(0xffff, 0xdeadbeef);
    assert.equal(pkt[9], 0xde);
    assert.equal(pkt[10], 0xad);
    assert.equal(pkt[11], 0xbe);
    assert.equal(pkt[12], 0xef);
  });

  test("encodes subnet big-endian at bytes 3..4", () => {
    const pkt = buildBeginTransfer(0xfffe, 0x12345678);
    assert.equal(pkt[3], 0xff);
    assert.equal(pkt[4], 0xfe);
  });
});
```

- [ ] **Step A1.2: Run test to verify it fails**

Run: `npx tsx --test test/cca-ota-tx-builder.test.ts`
Expected: FAIL, `buildBeginTransfer` not exported.

- [ ] **Step A1.3: Implement minimal `buildBeginTransfer`**

`lib/cca-ota-tx-builder.ts`:

```typescript
/**
 * Synth-OTA-TX builders — TS mirror of firmware/src/cca/cca_ota_tx.h.
 *
 * Emits on-air OTA packets that match captured Caseta REP2 → DVRF-6L OTA
 * traffic byte-for-byte. Packets are pre-CRC; CRC-16/0xCA0F is added by
 * the Nucleo's N81 framer downstream.
 *
 * See docs/firmware-re/cca-ota-live-capture.md for the wire protocol.
 */

const QS_PROTO_RADIO_TX = 0x21;

const OTA_SUB_BEGIN_TRANSFER = 0x00;
const OTA_SUB_CHANGE_ADDRESS_OFF = 0x01;
const OTA_SUB_TRANSFER_DATA = 0x02;

const OTA_TYPE_BEGIN_TRANSFER = 0x92;
const OTA_TYPE_CHANGE_ADDR_OFF = 0x91;

const OTA_BODY_LEN_SIG_BEGIN = 0x0e;
const OTA_BODY_LEN_SIG_CHADDR = 0x0c;
const OTA_BODY_LEN_SIG_LONG = 0x2b;

export const OTA_CHUNK_SIZE = 31;
export const OTA_PAGE_SIZE = 0x10000;

function writeHeader(
  pkt: Uint8Array,
  typeByte: number,
  bodyLenSig: number,
  subnet: number,
  targetSerial: number,
): void {
  pkt[0] = typeByte;
  pkt[1] = 0x00;
  pkt[2] = 0xa1;
  pkt[3] = (subnet >> 8) & 0xff;
  pkt[4] = subnet & 0xff;
  pkt[5] = 0x00;
  pkt[6] = QS_PROTO_RADIO_TX;
  pkt[7] = bodyLenSig;
  pkt[8] = 0x00;
  pkt[9] = (targetSerial >>> 24) & 0xff;
  pkt[10] = (targetSerial >>> 16) & 0xff;
  pkt[11] = (targetSerial >>> 8) & 0xff;
  pkt[12] = targetSerial & 0xff;
  pkt[13] = 0xfe;
}

export function buildBeginTransfer(subnet: number, targetSerial: number): Uint8Array {
  const pkt = new Uint8Array(22);
  writeHeader(pkt, OTA_TYPE_BEGIN_TRANSFER, OTA_BODY_LEN_SIG_BEGIN, subnet, targetSerial);
  pkt[14] = 0x06;
  pkt[15] = OTA_SUB_BEGIN_TRANSFER;
  pkt[16] = 0x02;
  pkt[17] = 0x20;
  pkt[18] = 0x00;
  pkt[19] = 0x00;
  pkt[20] = 0x00;
  pkt[21] = 0x1f;
  return pkt;
}
```

- [ ] **Step A1.4: Run test to verify it passes**

Run: `npx tsx --test test/cca-ota-tx-builder.test.ts`
Expected: PASS, all 3 tests.

- [ ] **Step A1.5: Add tests for `buildChangeAddressOffset` (RED)**

Add to `test/cca-ota-tx-builder.test.ts`:

```typescript
import { buildBeginTransfer, buildChangeAddressOffset } from "../lib/cca-ota-tx-builder";

// Captured: 91 01 a1 ef fd 00 21 0c 00 06 fe 80 20 fe 06 01 00 01 00 02 cc cc
const EXPECT_CHANGE_ADDR = new Uint8Array([
  0x91, 0x00, 0xa1, 0xef, 0xfd, 0x00, 0x21, 0x0c,
  0x00, 0x06, 0xfe, 0x80, 0x20, 0xfe, 0x06, 0x01,
  0x00, 0x01, 0x00, 0x02, 0xcc, 0xcc,
]);

describe("buildChangeAddressOffset", () => {
  test("matches captured ground truth", () => {
    const pkt = buildChangeAddressOffset(0xeffd, 0x06fe8020, 0x0001, 0x0002);
    assert.deepEqual(pkt, EXPECT_CHANGE_ADDR);
  });

  test("page indices encoded big-endian at bytes 16..19", () => {
    const pkt = buildChangeAddressOffset(0xffff, 0x12345678, 0x00ab, 0x00cd);
    assert.equal(pkt[16], 0x00);
    assert.equal(pkt[17], 0xab);
    assert.equal(pkt[18], 0x00);
    assert.equal(pkt[19], 0xcd);
  });

  test("padding bytes 20..21 are 0xCC", () => {
    const pkt = buildChangeAddressOffset(0xffff, 0x12345678, 0, 1);
    assert.equal(pkt[20], 0xcc);
    assert.equal(pkt[21], 0xcc);
  });
});
```

Run: FAIL.

- [ ] **Step A1.6: Implement `buildChangeAddressOffset` (GREEN)**

```typescript
export function buildChangeAddressOffset(
  subnet: number,
  targetSerial: number,
  prevPage: number,
  nextPage: number,
): Uint8Array {
  const pkt = new Uint8Array(22).fill(0xcc);
  writeHeader(pkt, OTA_TYPE_CHANGE_ADDR_OFF, OTA_BODY_LEN_SIG_CHADDR, subnet, targetSerial);
  pkt[14] = 0x06;
  pkt[15] = OTA_SUB_CHANGE_ADDRESS_OFF;
  pkt[16] = (prevPage >> 8) & 0xff;
  pkt[17] = prevPage & 0xff;
  pkt[18] = (nextPage >> 8) & 0xff;
  pkt[19] = nextPage & 0xff;
  return pkt;
}
```

Run: PASS.

- [ ] **Step A1.7: Add tests for `buildTransferData` (RED)**

```typescript
import { buildBeginTransfer, buildChangeAddressOffset, buildTransferData } from "../lib/cca-ota-tx-builder";

// Header bytes 0..19 of a captured TransferData (sub_counter=0x23, addrLo=0x49FD).
const EXPECT_TRANSFER_DATA_HEADER = new Uint8Array([
  0xb3, 0x00, 0xa1, 0xef, 0xfd, 0x00, 0x21, 0x2b,
  0x00, 0x06, 0xfe, 0x80, 0x20, 0xfe, 0x06, 0x02,
  0x23, 0x49, 0xfd, 0x1f,
]);

describe("buildTransferData", () => {
  test("header bytes 0..19 match captured ground truth", () => {
    const chunk = new Uint8Array(31);
    for (let i = 0; i < 31; i++) chunk[i] = 0xaa + i;
    const pkt = buildTransferData(0xb3, 0xeffd, 0x06fe8020, 0x23, 0x49fd, chunk);
    assert.deepEqual(pkt.slice(0, 20), EXPECT_TRANSFER_DATA_HEADER);
    assert.equal(pkt.length, 51);
  });

  test("chunk payload is verbatim at bytes 20..50", () => {
    const chunk = new Uint8Array(31);
    for (let i = 0; i < 31; i++) chunk[i] = 0x55 + i;
    const pkt = buildTransferData(0xb2, 0xeffd, 0x06fe8020, 0x10, 0x0000, chunk);
    assert.deepEqual(pkt.slice(20, 51), chunk);
  });

  test("rejects non-B1/B2/B3 carriers", () => {
    const chunk = new Uint8Array(31);
    assert.throws(() => buildTransferData(0xa1, 0xffff, 0x12345678, 0, 0, chunk));
    assert.throws(() => buildTransferData(0xb0, 0xffff, 0x12345678, 0, 0, chunk));
    assert.throws(() => buildTransferData(0xb4, 0xffff, 0x12345678, 0, 0, chunk));
  });

  test("rejects wrong chunk length", () => {
    assert.throws(() => buildTransferData(0xb2, 0xffff, 0x12345678, 0, 0, new Uint8Array(30)));
    assert.throws(() => buildTransferData(0xb2, 0xffff, 0x12345678, 0, 0, new Uint8Array(32)));
  });
});
```

Run: FAIL.

- [ ] **Step A1.8: Implement `buildTransferData` (GREEN)**

```typescript
export function buildTransferData(
  carrierType: number,
  subnet: number,
  targetSerial: number,
  subCounter: number,
  addrLo: number,
  chunk: Uint8Array,
): Uint8Array {
  if (carrierType < 0xb1 || carrierType > 0xb3) {
    throw new RangeError(`carrier type 0x${carrierType.toString(16)} not in B1..B3`);
  }
  if (chunk.length !== OTA_CHUNK_SIZE) {
    throw new RangeError(`chunk length ${chunk.length} != ${OTA_CHUNK_SIZE}`);
  }
  const pkt = new Uint8Array(51);
  writeHeader(pkt, carrierType, OTA_BODY_LEN_SIG_LONG, subnet, targetSerial);
  pkt[14] = 0x06;
  pkt[15] = OTA_SUB_TRANSFER_DATA;
  pkt[16] = subCounter;
  pkt[17] = (addrLo >> 8) & 0xff;
  pkt[18] = addrLo & 0xff;
  pkt[19] = OTA_CHUNK_SIZE;
  pkt.set(chunk, 20);
  return pkt;
}
```

Run: PASS.

- [ ] **Step A1.9: Add OtaChunkIter class tests (RED)**

```typescript
import { OtaChunkIter } from "../lib/cca-ota-tx-builder";

describe("OtaChunkIter", () => {
  test("emits chunks of exactly 31 bytes, advances addrLo by 31", () => {
    const body = new Uint8Array(100);
    for (let i = 0; i < 100; i++) body[i] = i;
    const it = new OtaChunkIter(body);
    assert.equal(it.addrLo, 0);
    const c1 = it.fill();
    assert.equal(c1.length, 31);
    assert.equal(c1[0], 0);
    assert.equal(c1[30], 30);
    it.advance();
    assert.equal(it.addrLo, 31);
    const c2 = it.fill();
    assert.equal(c2[0], 31);
  });

  test("signals page wrap at 64KB boundary", () => {
    const body = new Uint8Array(0x12000);
    const it = new OtaChunkIter(body);
    it.addrLo = 0xffe3;
    const wrapped = it.advance();
    assert.equal(wrapped, true);
    assert.equal(it.page, 1);
    assert.equal(it.addrLo, 2);
  });

  test("pads short final chunk with 0x00", () => {
    const body = new Uint8Array(40);
    for (let i = 0; i < 40; i++) body[i] = 0x10 + i;
    const it = new OtaChunkIter(body);
    it.advance(); // skip first chunk
    const c = it.fill();
    assert.equal(c[0], 0x10 + 31);
    assert.equal(c[8], 0x10 + 39);
    assert.equal(c[9], 0x00);
    assert.equal(c[30], 0x00);
  });
});
```

Run: FAIL.

- [ ] **Step A1.10: Implement OtaChunkIter (GREEN)**

```typescript
export class OtaChunkIter {
  cursor = 0;
  addrLo = 0;
  page = 0;
  subCounter = 0;

  constructor(public readonly body: Uint8Array) {}

  done(): boolean {
    return this.cursor >= this.body.length;
  }

  fill(): Uint8Array {
    const chunk = new Uint8Array(OTA_CHUNK_SIZE);
    const remaining = Math.max(0, this.body.length - this.cursor);
    const n = Math.min(remaining, OTA_CHUNK_SIZE);
    if (n > 0) chunk.set(this.body.subarray(this.cursor, this.cursor + n), 0);
    return chunk;
  }

  advance(): boolean {
    this.cursor += OTA_CHUNK_SIZE;
    this.subCounter = (this.subCounter + 1) & 0x3f;
    const next = this.addrLo + OTA_CHUNK_SIZE;
    if (next >= OTA_PAGE_SIZE) {
      this.page += 1;
      this.addrLo = next - OTA_PAGE_SIZE;
      return true;
    }
    this.addrLo = next;
    return false;
  }
}
```

Run: PASS.

- [ ] **Step A1.11: Commit Track 2 builder + tests**

```bash
git add lib/cca-ota-tx-builder.ts test/cca-ota-tx-builder.test.ts
git commit -m "feat(cca): TS mirror of cca_ota_tx.h builders + chunk iterator"
```

### Task A2: LDF body extractor in TS

**Files:**
- Create: `lib/ldf.ts`
- Test: `test/ldf.test.ts`

The driver needs to strip the 0x80-byte LDF header. We could shell out to the existing Python (`tools/firmware/ldf-extract.py`), but a pure-TS implementation is simpler and lets the driver work without Python.

- [ ] **Step A2.1: Write failing test**

`test/ldf.test.ts`:

```typescript
import { describe, test } from "node:test";
import assert from "node:assert/strict";
import { stripLdfHeader, LDF_HEADER_LEN } from "../lib/ldf";

describe("stripLdfHeader", () => {
  test("LDF_HEADER_LEN is 0x80", () => {
    assert.equal(LDF_HEADER_LEN, 0x80);
  });

  test("returns body bytes after the 0x80-byte header", () => {
    const file = new Uint8Array(0x80 + 100);
    for (let i = 0; i < file.length; i++) file[i] = i & 0xff;
    const body = stripLdfHeader(file);
    assert.equal(body.length, 100);
    assert.equal(body[0], 0x80);
    assert.equal(body[99], (0x80 + 99) & 0xff);
  });

  test("throws on file shorter than header", () => {
    assert.throws(() => stripLdfHeader(new Uint8Array(0x70)));
  });
});
```

Run: FAIL.

- [ ] **Step A2.2: Implement `stripLdfHeader` (GREEN)**

`lib/ldf.ts`:

```typescript
export const LDF_HEADER_LEN = 0x80;

export function stripLdfHeader(file: Uint8Array): Uint8Array {
  if (file.length < LDF_HEADER_LEN) {
    throw new Error(`LDF file too short: ${file.length} < ${LDF_HEADER_LEN}`);
  }
  return file.subarray(LDF_HEADER_LEN);
}
```

Run: PASS.

- [ ] **Step A2.3: Commit**

```bash
git add lib/ldf.ts test/ldf.test.ts
git commit -m "feat(ldf): pure-TS LDF header strip helper"
```

### Task A3: Driver `tools/cca/ota-tx.ts`

**Files:**
- Create: `tools/cca/ota-tx.ts`
- Modify: `package.json` (add `cca:ota-tx` script)

The driver reads the LDF, strips header, walks chunks via `OtaChunkIter`, builds packets with the Track 2 builders, and either logs them (`--dry-run`) or sends them to the Nucleo via `STREAM_CMD_TX_RAW_CCA = 0x01`.

**Design**: 3-arm carrier rotation matches firmware (B1/B2/B3 cycle); 75 ms cadence between packets; emits BeginTransfer once at start, ChangeAddrOff at every 64 KB page wrap, TransferData for each chunk.

- [ ] **Step A3.1: Implement driver as a small composable function (no test for the I/O wrapper itself)**

`tools/cca/ota-tx.ts`:

```typescript
#!/usr/bin/env npx tsx

/**
 * Host-side PowPak OTA driver — streams an LDF body to a target device via
 * the Nucleo's CC1101, using STREAM_CMD_TX_RAW_CCA UDP datagrams.
 *
 * Mirrors the firmware-side `cca ota-tx` orchestration but in TypeScript,
 * for fast iteration on packet format during Phase 2 debugging. See
 * docs/firmware-re/powpak-conversion-attack.md.
 *
 * Usage:
 *   npx tsx tools/cca/ota-tx.ts \
 *     --ldf data/firmware-re/powpak/PowPakRelay434_1-49.bin \
 *     --subnet 0xffff --serial 00BC2107 \
 *     --host 10.1.1.114 [--dry-run]
 */

import { readFileSync } from "node:fs";
import { createSocket } from "node:dgram";
import { setTimeout as sleep } from "node:timers/promises";
import {
  buildBeginTransfer,
  buildChangeAddressOffset,
  buildTransferData,
  OtaChunkIter,
} from "../../lib/cca-ota-tx-builder";
import { stripLdfHeader, LDF_HEADER_LEN } from "../../lib/ldf";
import { config } from "../../lib/config";

const STREAM_CMD_TX_RAW_CCA = 0x01;
const STREAM_CMD_KEEPALIVE = 0x00;
const PORT = 9433;
const TX_INTERVAL_MS = 75;

function getArg(name: string): string | undefined {
  const i = process.argv.indexOf(name);
  return i !== -1 ? process.argv[i + 1] : undefined;
}
function hasFlag(name: string): boolean {
  return process.argv.includes(name);
}

async function main(): Promise<void> {
  const ldfPath = getArg("--ldf");
  const subnetStr = getArg("--subnet");
  const serialStr = getArg("--serial");
  const host = getArg("--host") ?? config.openBridge;
  const dryRun = hasFlag("--dry-run");

  if (!ldfPath || !subnetStr || !serialStr) {
    console.error("Usage: npx tsx tools/cca/ota-tx.ts --ldf <path> --subnet <hex> --serial <hex> [--host <ip>] [--dry-run]");
    process.exit(1);
  }

  const subnet = parseInt(subnetStr.replace(/^0x/, ""), 16);
  const targetSerial = parseInt(serialStr.replace(/^0x/, ""), 16);

  const file = readFileSync(ldfPath);
  const body = stripLdfHeader(new Uint8Array(file.buffer, file.byteOffset, file.byteLength));
  console.log(`[ota-tx] LDF: ${file.length} bytes -> body ${body.length} bytes (header stripped)`);

  const sock = dryRun ? null : createSocket("udp4");
  if (sock) {
    await new Promise<void>((r) => sock.bind(0, () => r()));
    // Register as stream client
    sendStream(sock, host, STREAM_CMD_KEEPALIVE, new Uint8Array(0));
    await sleep(100);
  }

  const sendPacket = async (pkt: Uint8Array, label: string): Promise<void> => {
    if (dryRun) {
      const hex = Array.from(pkt).map((b) => b.toString(16).padStart(2, "0")).join(" ");
      console.log(`[dry] ${label.padEnd(14)} ${hex}`);
    } else {
      sendStream(sock!, host, STREAM_CMD_TX_RAW_CCA, pkt);
    }
    await sleep(TX_INTERVAL_MS);
  };

  // BeginTransfer (1×)
  await sendPacket(buildBeginTransfer(subnet, targetSerial), "BeginTransfer");

  // TransferData stream + ChangeAddrOff at page boundaries
  const carriers = [0xb1, 0xb2, 0xb3];
  const it = new OtaChunkIter(body);
  let chunkCount = 0;
  while (!it.done()) {
    const carrier = carriers[chunkCount % 3];
    const chunk = it.fill();
    const td = buildTransferData(carrier, subnet, targetSerial, it.subCounter, it.addrLo, chunk);
    await sendPacket(td, `TransferData[${chunkCount}]`);
    const wrapped = it.advance();
    if (wrapped && !it.done()) {
      const cao = buildChangeAddressOffset(subnet, targetSerial, it.page - 1, it.page);
      await sendPacket(cao, "ChangeAddrOff");
    }
    chunkCount++;
    if (chunkCount % 100 === 0) {
      console.log(`[ota-tx] sent ${chunkCount} chunks (page=${it.page})`);
    }
  }

  console.log(`[ota-tx] complete: ${chunkCount} chunks, ${it.page} page wraps`);
  if (sock) sock.close();
}

function sendStream(sock: import("node:dgram").Socket, host: string, cmd: number, data: Uint8Array): void {
  const frame = Buffer.alloc(2 + data.length);
  frame[0] = cmd;
  frame[1] = data.length;
  if (data.length > 0) Buffer.from(data).copy(frame, 2);
  sock.send(frame, 0, frame.length, PORT, host);
}

main().catch((e) => {
  console.error(e);
  process.exit(1);
});
```

- [ ] **Step A3.2: Smoke-test the driver in dry-run mode against the real LDF**

Run: `npx tsx tools/cca/ota-tx.ts --ldf data/firmware-re/powpak/PowPakRelay434_1-49.bin --subnet 0xffff --serial 00BC2107 --dry-run | head -5`
Expected: First line `[ota-tx] LDF: 100008 bytes -> body 99880 bytes ...`, then BeginTransfer hex line matching captured byte pattern (subnet=ffff overrides).

- [ ] **Step A3.3: Add npm script**

Edit `package.json` `scripts` section, add: `"cca:ota-tx": "npx tsx tools/cca/ota-tx.ts"`.

- [ ] **Step A3.4: Run tests + lint + typecheck**

Run: `npm run lint && npm run typecheck && npm test --silent 2>&1 | tail -20`
Expected: all green.

- [ ] **Step A3.5: Commit Track 2 driver**

```bash
git add tools/cca/ota-tx.ts package.json
git commit -m "feat(cca): host-side OTA TX driver (tools/cca/ota-tx.ts) with --dry-run"
```

### Task A4: Open Track 2 PR

- [ ] **Step A4.1: Push branch and open PR**

```bash
git push -u origin claude/dazzling-ritchie-7921f0
gh pr create --title "feat(cca): host-side OTA TX driver + TS builders" --body "<see template below>"
```

PR body template:
```
## Summary
- New `lib/cca-ota-tx-builder.ts` — TS mirror of `firmware/src/cca/cca_ota_tx.h` (BeginTransfer, ChangeAddressOffset, TransferData, OtaChunkIter), TDD against captured-on-air ground truth.
- New `tools/cca/ota-tx.ts` — host-side PowPak OTA driver. Reads LDF, builds packets, streams them via `STREAM_CMD_TX_RAW_CCA` UDP datagrams. Supports `--dry-run`.
- Track 2 of the Phase 2b plan; firmware-side track is independent and ships separately.

## Test plan
- [x] `npx tsx --test test/cca-ota-tx-builder.test.ts` — byte-for-byte captured ground truth match
- [x] `npm run lint && npm run typecheck`
- [x] Dry-run against `PowPakRelay434_1-49.bin` produces expected packet sequence
- [ ] Live-fire against sacrificial RMJ (deferred — explicitly out of scope for this PR)
```

---

## Track 1: Firmware-side `CCA_CMD_OTA_FULL_TX`

### Task B1: Static LDF body buffer + upload commands

**Files:**
- Create: `firmware/src/cca/cca_ota_session.cpp`
- Create: `firmware/src/cca/cca_ota_session.h`
- Modify: `firmware/src/net/stream.h` (add stream cmd codes)
- Modify: `firmware/src/net/stream.cpp` (handle the new commands)

The stream protocol's `STREAM_CMD_TEXT` only delivers 255 bytes per datagram, so a 102 KB LDF body needs a multi-chunk binary upload. We add three new codes:

- `STREAM_CMD_OTA_UPLOAD_START = 0x18` — `[u32 body_len LE]` — clear buffer + set expected total
- `STREAM_CMD_OTA_UPLOAD_CHUNK = 0x19` — `[u16 offset BE][bytes...]` — write at offset
- `STREAM_CMD_OTA_UPLOAD_END = 0x1A` — `[]` — verify length (used to commit / log)

The body buffer is a static array in `.bss` (RAM_D1, 320 KB available).

- [ ] **Step B1.1: Write failing tests for cca_ota_session module**

`firmware/tests/test_ota_session.cpp`:

```cpp
#include <cstdint>
#include <cstring>
#include <cstdio>

#include "cca_ota_session.h"

extern int test_pass_count;
extern int test_fail_count;
extern void test_registry_add(const char *name, void (*func)());

#define TEST(name) \
    static void test_##name(); \
    static struct test_reg_##name { \
        test_reg_##name() { test_registry_add(#name, test_##name); } \
    } test_reg_inst_##name; \
    static void test_##name()

#define ASSERT_EQ(a, b) do { \
    auto _a = (a); auto _b = (b); \
    if (_a != _b) { \
        printf("  FAIL: %s:%d: %s == %lld, expected %lld\n", \
               __FILE__, __LINE__, #a, (long long)_a, (long long)_b); \
        test_fail_count++; return; \
    } \
} while (0)

TEST(ota_session_starts_empty)
{
    cca_ota_session_reset();
    ASSERT_EQ(cca_ota_session_body_len(), 0u);
    ASSERT_EQ(cca_ota_session_expected_len(), 0u);
    ASSERT_EQ(cca_ota_session_complete(), false);
}

TEST(ota_session_start_sets_expected_len)
{
    cca_ota_session_reset();
    bool ok = cca_ota_session_start(102516);
    ASSERT_EQ(ok, true);
    ASSERT_EQ(cca_ota_session_expected_len(), 102516u);
    ASSERT_EQ(cca_ota_session_body_len(), 0u);
}

TEST(ota_session_rejects_oversize_start)
{
    cca_ota_session_reset();
    bool ok = cca_ota_session_start(200 * 1024);  // > 110KB cap
    ASSERT_EQ(ok, false);
}

TEST(ota_session_writes_chunks_at_offset)
{
    cca_ota_session_reset();
    cca_ota_session_start(64);
    uint8_t a[16]; for (int i=0;i<16;i++) a[i] = (uint8_t)(0x10 + i);
    uint8_t b[16]; for (int i=0;i<16;i++) b[i] = (uint8_t)(0x20 + i);
    bool r1 = cca_ota_session_write(0, a, 16);
    bool r2 = cca_ota_session_write(48, b, 16);
    ASSERT_EQ(r1, true);
    ASSERT_EQ(r2, true);
    ASSERT_EQ(cca_ota_session_body_len(), 64u);
    const uint8_t* body = cca_ota_session_body();
    ASSERT_EQ(body[0], 0x10);
    ASSERT_EQ(body[15], 0x1F);
    ASSERT_EQ(body[48], 0x20);
    ASSERT_EQ(body[63], 0x2F);
}

TEST(ota_session_rejects_write_past_expected_len)
{
    cca_ota_session_reset();
    cca_ota_session_start(64);
    uint8_t buf[16] = {0};
    bool r = cca_ota_session_write(56, buf, 16);  // would write to offset 71
    ASSERT_EQ(r, false);
}

TEST(ota_session_complete_when_filled)
{
    cca_ota_session_reset();
    cca_ota_session_start(32);
    uint8_t buf[16] = {0};
    cca_ota_session_write(0, buf, 16);
    ASSERT_EQ(cca_ota_session_complete(), false);
    cca_ota_session_write(16, buf, 16);
    ASSERT_EQ(cca_ota_session_complete(), true);
}
```

- [ ] **Step B1.2: Add test file to CMake**

Edit `firmware/tests/CMakeLists.txt`: add `test_ota_session.cpp` to the test_runner sources.

Run: `cd firmware && cmake -B build-host -DCMAKE_TOOLCHAIN_FILE=cmake/host.cmake && make -C build-host test_runner`

Run: `firmware/build-host/test_runner`
Expected: FAIL — module doesn't exist yet.

- [ ] **Step B1.3: Implement cca_ota_session (GREEN)**

`firmware/src/cca/cca_ota_session.h`:

```c
#ifndef CCA_OTA_SESSION_H
#define CCA_OTA_SESSION_H

#include <stdint.h>
#include <stddef.h>
#include <stdbool.h>

#ifdef __cplusplus
extern "C" {
#endif

/** Capacity of the static LDF body buffer (110 KB; LMJ body is ~102 KB). */
#define CCA_OTA_SESSION_CAPACITY (110 * 1024)

/** Reset session — clears expected_len and body_len. */
void cca_ota_session_reset(void);

/** Begin a session. Returns false if expected_len > capacity. */
bool cca_ota_session_start(uint32_t expected_len);

/** Write `len` bytes at `offset` into the body buffer. Returns false if
 *  offset+len would exceed expected_len. Updates body_len = max(prev, offset+len). */
bool cca_ota_session_write(uint32_t offset, const uint8_t* data, uint32_t len);

uint32_t cca_ota_session_expected_len(void);
uint32_t cca_ota_session_body_len(void);
bool cca_ota_session_complete(void);
const uint8_t* cca_ota_session_body(void);

#ifdef __cplusplus
}
#endif

#endif
```

`firmware/src/cca/cca_ota_session.cpp`:

```cpp
#include "cca_ota_session.h"
#include <cstring>

static uint8_t s_body[CCA_OTA_SESSION_CAPACITY];
static uint32_t s_expected_len = 0;
static uint32_t s_body_len = 0;

extern "C" {

void cca_ota_session_reset(void)
{
    s_expected_len = 0;
    s_body_len = 0;
}

bool cca_ota_session_start(uint32_t expected_len)
{
    if (expected_len > CCA_OTA_SESSION_CAPACITY) return false;
    s_expected_len = expected_len;
    s_body_len = 0;
    return true;
}

bool cca_ota_session_write(uint32_t offset, const uint8_t* data, uint32_t len)
{
    if (s_expected_len == 0) return false;
    if (offset + len > s_expected_len) return false;
    if (offset + len > CCA_OTA_SESSION_CAPACITY) return false;
    memcpy(s_body + offset, data, len);
    uint32_t end = offset + len;
    if (end > s_body_len) s_body_len = end;
    return true;
}

uint32_t cca_ota_session_expected_len(void) { return s_expected_len; }
uint32_t cca_ota_session_body_len(void) { return s_body_len; }
bool cca_ota_session_complete(void) { return s_expected_len > 0 && s_body_len == s_expected_len; }
const uint8_t* cca_ota_session_body(void) { return s_body; }

}
```

Run tests: PASS.

- [ ] **Step B1.4: Wire stream commands into stream.cpp**

Edit `firmware/src/net/stream.h`, add:

```c
#define STREAM_CMD_OTA_UPLOAD_START 0x18
#define STREAM_CMD_OTA_UPLOAD_CHUNK 0x19
#define STREAM_CMD_OTA_UPLOAD_END   0x1A
```

Edit `firmware/src/net/stream.cpp`, add include `#include "cca_ota_session.h"` and add cases to the `handle_rx_data` switch:

```cpp
case STREAM_CMD_OTA_UPLOAD_START: {
    if (data_len >= 4) {
        uint32_t expected = (uint32_t)buf[2] | ((uint32_t)buf[3] << 8) |
                            ((uint32_t)buf[4] << 16) | ((uint32_t)buf[5] << 24);
        bool ok = cca_ota_session_start(expected);
        printf("[stream] OTA upload start: %lu bytes (%s)\r\n",
               (unsigned long)expected, ok ? "ok" : "REJECTED");
    }
    break;
}
case STREAM_CMD_OTA_UPLOAD_CHUNK: {
    if (data_len >= 2) {
        uint16_t offset = ((uint16_t)buf[2] << 8) | buf[3];
        uint32_t scaled = (uint32_t)offset * 1024u;  // chunk offsets in KB
        bool ok = cca_ota_session_write(scaled, buf + 4, data_len - 2);
        if (!ok) printf("[stream] OTA upload chunk REJECTED at %lu\r\n", (unsigned long)scaled);
    }
    break;
}
case STREAM_CMD_OTA_UPLOAD_END: {
    printf("[stream] OTA upload end: %lu/%lu bytes (%s)\r\n",
           (unsigned long)cca_ota_session_body_len(),
           (unsigned long)cca_ota_session_expected_len(),
           cca_ota_session_complete() ? "complete" : "INCOMPLETE");
    break;
}
```

NOTE: 16-bit offset in KB scales to 64 MB max — overkill for 110 KB but lets us avoid 4-byte offset encoding in datagram payload. Keep `data_len ≤ 247` so the datagram fits the existing 1-byte length field (255 - 2 cmd/len - 4 cmd_payload_header = 249).

Wait — re-check. The stream framing has `[CMD:1][LEN:1][DATA:N]`. data_len byte caps DATA at 255. We need 4 header bytes (2 for offset BE + reserved) inside DATA, leaving 251 bytes per chunk. Choose 240 bytes per chunk for round numbers.

Actually let me revise: use byte-offset directly with 3-byte BE, leaves 252 bytes per chunk for data. But simpler is KB-offset (uint16) with 1024-byte chunks, but 1024 > 255 datagram cap. So back to byte-offset: 3-byte BE offset (24 bits = 16 MB max), 1-byte reserved, leaves 251 bytes per chunk.

Let me simplify: **fixed 240-byte chunks**, 16-bit chunk index. Chunk 0 → offset 0, chunk 1 → offset 240, ..., chunk N → offset N*240. 110 KB = 469 chunks; index 16-bit covers up to 256K chunks. Datagram payload = 2 (chunk index BE) + 240 (data) = 242 bytes, fits in u8 LEN.

Re-revised step B1.4 stream handler:

```cpp
case STREAM_CMD_OTA_UPLOAD_CHUNK: {
    if (data_len >= 2) {
        uint16_t chunk_idx = ((uint16_t)buf[2] << 8) | buf[3];
        uint32_t offset = (uint32_t)chunk_idx * 240u;
        bool ok = cca_ota_session_write(offset, buf + 4, data_len - 2);
        if (!ok) printf("[stream] OTA upload chunk %u REJECTED\r\n", chunk_idx);
    }
    break;
}
```

- [ ] **Step B1.5: Add cca_ota_session.cpp to firmware CMakeLists**

Edit `firmware/CMakeLists.txt`: ensure `src/cca/cca_ota_session.cpp` is in the source list (or covered by a glob).

- [ ] **Step B1.6: Run firmware unit tests**

Run: `cd firmware && make test`
Expected: all tests pass (existing 154 + 6 new = 160).

- [ ] **Step B1.7: Build ARM firmware**

Run: `cd firmware && cmake -B build -DCMAKE_TOOLCHAIN_FILE=cmake/arm-none-eabi.cmake && make -C build -j8`
Expected: clean build.

- [ ] **Step B1.8: Commit**

```bash
git add firmware/src/cca/cca_ota_session.{h,cpp} firmware/src/net/stream.{h,cpp} firmware/tests/test_ota_session.cpp firmware/tests/CMakeLists.txt firmware/CMakeLists.txt
git commit -m "feat(cca): static LDF body buffer + stream upload commands"
```

### Task B2: Firmware orchestrator + shell command

**Files:**
- Modify: `firmware/src/cca/cca_commands.h` (add `CCA_CMD_OTA_FULL_TX = 0x1E`)
- Modify: `firmware/src/cca/cca_pairing.cpp` (add `exec_ota_full_tx`)
- Modify: `firmware/src/shell/shell.cpp` (add `cca ota-tx <subnet> <serial>` parser)

The orchestrator:
1. Validates session is complete (`cca_ota_session_complete()`).
2. Sends BeginTransfer once.
3. For each chunk via `OtaChunkIter`: builds TransferData (cycling carriers B1/B2/B3), TX, vTaskDelay(75).
4. On wrap signal: sends ChangeAddrOff before next TransferData.
5. Logs progress every 100 chunks via `stream_broadcast_text`.

- [ ] **Step B2.1: Write failing test for the orchestration sequence**

`firmware/tests/test_ota_full_tx.cpp` — verify the orchestrator produces the right packet sequence for a synthetic 100-byte body. Use a captured TX buffer to check the order of packet types emitted.

We test the orchestrator's packet sequence by extracting it into a reusable helper:

```cpp
#include "cca_ota_tx.h"
#include "cca_ota_session.h"

extern int test_pass_count;
extern int test_fail_count;
extern void test_registry_add(const char *name, void (*func)());
#define TEST(name) ...   /* same macro as before */

/* The orchestrator emits packets via a callback so it can be tested without
 * a TX engine. We add `ota_full_tx_walk(callback, subnet, serial)` to
 * cca_ota_tx.h that walks the session buffer, invoking callback for each
 * packet (BeginTransfer, TransferData..., ChangeAddrOff between pages). */

struct PacketLog {
    uint8_t type;
    uint8_t body_len_sig;
    size_t  count;
};
static PacketLog test_log[64];
static size_t test_log_n = 0;

static void test_callback(const uint8_t* pkt, size_t len, void* /*ctx*/) {
    if (test_log_n < 64) {
        test_log[test_log_n].type = pkt[0];
        test_log[test_log_n].body_len_sig = pkt[7];
        test_log_n++;
    }
}

TEST(ota_full_tx_walk_emits_begin_then_chunks)
{
    cca_ota_session_reset();
    cca_ota_session_start(62);  // 2 chunks of 31 bytes
    uint8_t fw[62] = {0};
    cca_ota_session_write(0, fw, 62);

    test_log_n = 0;
    cca_ota_full_tx_walk(0xeffd, 0x06fe8020, test_callback, nullptr);

    /* Expected: 1 BeginTransfer (0x92) + 2 TransferData (0xB1..B3, body_len=0x2B). */
    /* No page wrap (62 bytes < 64KB) -> no ChangeAddrOff. */
    ASSERT_EQ(test_log_n, 3u);
    ASSERT_EQ(test_log[0].type, 0x92u);
    ASSERT_EQ(test_log[0].body_len_sig, 0x0Eu);
    ASSERT_EQ(test_log[1].body_len_sig, 0x2Bu);
    ASSERT_EQ(test_log[2].body_len_sig, 0x2Bu);
}

TEST(ota_full_tx_walk_emits_change_addr_at_page_wrap)
{
    cca_ota_session_reset();
    /* 64 KB + 31 bytes -> first 64K/31 ≈ 2114 chunks fill page 0,
     * the wrap happens, then 1 more TransferData from page 1.
     * We don't need exact counts, just verify ChangeAddrOff appears
     * exactly once between TransferData groups. */
    uint32_t total = 0x10000u + 31u;  /* page wrap forces exactly 1 ChangeAddrOff */
    cca_ota_session_start(total);
    uint8_t buf[31] = {0};
    for (uint32_t off = 0; off < total; off += 31) {
        uint32_t n = (off + 31 <= total) ? 31 : (total - off);
        cca_ota_session_write(off, buf, n);
    }

    test_log_n = 0;
    cca_ota_full_tx_walk(0xeffd, 0x06fe8020, test_callback, nullptr);

    /* Find at least one ChangeAddrOff (type 0x91, body_len_sig 0x0C). */
    int change_addr_count = 0;
    for (size_t i = 0; i < test_log_n; i++) {
        if (test_log[i].type == 0x91 && test_log[i].body_len_sig == 0x0C) change_addr_count++;
    }
    ASSERT_EQ(change_addr_count, 1);
}

TEST(ota_full_tx_walk_carriers_cycle_b1_b2_b3)
{
    cca_ota_session_reset();
    cca_ota_session_start(31 * 6);  // 6 chunks
    uint8_t buf[31 * 6] = {0};
    cca_ota_session_write(0, buf, 31 * 6);

    test_log_n = 0;
    cca_ota_full_tx_walk(0xeffd, 0x06fe8020, test_callback, nullptr);

    /* Expected sequence: Begin, TD(B1), TD(B2), TD(B3), TD(B1), TD(B2), TD(B3) */
    ASSERT_EQ(test_log_n, 7u);
    ASSERT_EQ(test_log[1].type, 0xB1u);
    ASSERT_EQ(test_log[2].type, 0xB2u);
    ASSERT_EQ(test_log[3].type, 0xB3u);
    ASSERT_EQ(test_log[4].type, 0xB1u);
    ASSERT_EQ(test_log[5].type, 0xB2u);
    ASSERT_EQ(test_log[6].type, 0xB3u);
}
```

Run: FAIL — `cca_ota_full_tx_walk` not declared.

- [ ] **Step B2.2: Add `cca_ota_full_tx_walk` to cca_ota_tx.h (GREEN)**

Add to `firmware/src/cca/cca_ota_tx.h` after the `OtaChunkIter` definition:

```cpp
typedef void (*OtaTxPacketCallback)(const uint8_t* pkt, size_t len, void* ctx);

/* Walks the LDF body in cca_ota_session, invoking `cb` for each on-air
 * packet (BeginTransfer, TransferData× chunks, ChangeAddrOff at page
 * boundaries). Pure orchestration — no TX side effects. The caller chooses
 * how to deliver each packet (TDMA queue, test logger, etc.). */
inline void cca_ota_full_tx_walk(uint16_t subnet, uint32_t target_serial,
                                 OtaTxPacketCallback cb, void* ctx)
{
    /* 1× BeginTransfer */
    uint8_t pkt22[22];
    cca_ota_build_begin_transfer(pkt22, subnet, target_serial);
    cb(pkt22, 22, ctx);

    /* TransferData stream + ChangeAddrOff at page wraps */
    OtaChunkIter it;
    extern uint32_t cca_ota_session_body_len(void);
    extern const uint8_t* cca_ota_session_body(void);
    cca_ota_chunk_iter_init(&it, cca_ota_session_body(), cca_ota_session_body_len());

    static const uint8_t carriers[3] = {0xB1, 0xB2, 0xB3};
    uint32_t chunk_count = 0;
    while (!cca_ota_chunk_iter_done(&it)) {
        uint8_t carrier = carriers[chunk_count % 3];
        uint8_t chunk[OTA_CHUNK_SIZE];
        cca_ota_chunk_iter_fill(&it, chunk);
        uint8_t pkt51[51];
        cca_ota_build_transfer_data(pkt51, carrier, subnet, target_serial,
                                    it.sub_counter, it.addr_lo, chunk, OTA_CHUNK_SIZE);
        cb(pkt51, 51, ctx);
        bool wrapped = cca_ota_chunk_iter_advance(&it);
        if (wrapped && !cca_ota_chunk_iter_done(&it)) {
            cca_ota_build_change_addr_offset(pkt22, subnet, target_serial,
                                             (uint16_t)(it.page - 1), (uint16_t)it.page);
            cb(pkt22, 22, ctx);
        }
        chunk_count++;
    }
}
```

Run host tests: PASS.

- [ ] **Step B2.3: Add `CCA_CMD_OTA_FULL_TX = 0x1E` enum**

Edit `firmware/src/cca/cca_commands.h`: add `CCA_CMD_OTA_FULL_TX = 0x1E,` after the existing OTA enum.

- [ ] **Step B2.4: Implement `exec_ota_full_tx` in cca_pairing.cpp**

Add to `firmware/src/cca/cca_pairing.cpp` (after `exec_ota_begin`):

```cpp
struct OtaTxCtx {
    uint32_t count;
    uint8_t  seq;
};

static void ota_full_tx_send(const uint8_t* pkt, size_t len, void* ctx_v)
{
    OtaTxCtx* ctx = (OtaTxCtx*)ctx_v;
    /* Place sequence byte on the wire — TDMA engine writes it again,
     * but the placeholder above (0x00) is what the builder leaves in. */
    uint8_t buf[64];
    if (len > sizeof(buf)) return;
    memcpy(buf, pkt, len);
    buf[1] = ctx->seq;
    cc1101_stop_rx();
    transmit_one(buf, len);
    cc1101_start_rx();
    ctx->count++;
    ctx->seq = (ctx->seq + 6) & 0xFF;
    if (ctx->seq >= 0x48) ctx->seq = 0x01;
    /* Progress logging every 100 packets via stream_broadcast_text */
    if ((ctx->count % 100) == 0) {
        char line[64];
        int n = snprintf(line, sizeof(line), "[cca] ota-full-tx progress: %lu packets",
                         (unsigned long)ctx->count);
        stream_broadcast_text(line, (size_t)n);
    }
    vTaskDelay(pdMS_TO_TICKS(75));
}

static void exec_ota_full_tx(uint16_t subnet, uint32_t target_serial)
{
    if (!cca_ota_session_complete()) {
        printf("[cca] CMD ota_full_tx — session not complete (%lu/%lu bytes); upload first\r\n",
               (unsigned long)cca_ota_session_body_len(),
               (unsigned long)cca_ota_session_expected_len());
        return;
    }
    printf("[cca] CMD ota_full_tx serial=%08X subnet=%04X body=%lu bytes\r\n",
           (unsigned)target_serial, subnet, (unsigned long)cca_ota_session_body_len());

    OtaTxCtx ctx = {0, 0x01};
    cca_ota_full_tx_walk(subnet, target_serial, ota_full_tx_send, &ctx);

    printf("[cca] CMD ota_full_tx complete (%lu pkts)\r\n", (unsigned long)ctx.count);
}
```

Add the dispatch case to `cca_pairing_execute`:

```cpp
case CCA_CMD_OTA_FULL_TX:
    exec_ota_full_tx((uint16_t)((item->raw_payload[0] << 8) | item->raw_payload[1]),
                     item->device_id);
    break;
```

- [ ] **Step B2.5: Add shell command parser**

Edit `firmware/src/shell/shell.cpp`, add after the `ota-begin` parser:

```cpp
/* cca ota-tx <subnet_hex> <serial_hex>
 * Full-OTA transmit using the LDF body uploaded via the stream protocol. */
if (strncmp(arg, "ota-tx ", 7) == 0) {
    char* p;
    uint16_t subnet = (uint16_t)strtoul(arg + 7, &p, 16);
    if (*p != ' ') {
        printf("Usage: cca ota-tx <subnet_hex> <target_serial_hex>\r\n");
        return;
    }
    uint32_t serial = (uint32_t)strtoul(p + 1, NULL, 16);

    CcaCmdItem item = {};
    item.cmd = CCA_CMD_OTA_FULL_TX;
    item.device_id = serial;
    item.raw_payload[0] = (subnet >> 8) & 0xFF;
    item.raw_payload[1] = subnet & 0xFF;
    if (cca_cmd_enqueue(&item)) {
        printf("OTA full-tx queued (subnet=%04X serial=%08X)\r\n", subnet, (unsigned)serial);
    } else {
        printf("Command queue full!\r\n");
    }
    return;
}
```

Update help text:
```
printf("  cca ota-tx <subnet> <serial> — full-OTA TX from uploaded LDF body\r\n");
```

- [ ] **Step B2.6: Run all firmware unit tests**

Run: `cd firmware && make test`
Expected: 160+ tests pass.

- [ ] **Step B2.7: ARM build clean**

Run: `cd firmware && cmake -B build -DCMAKE_TOOLCHAIN_FILE=cmake/arm-none-eabi.cmake && make -C build -j8`
Expected: clean.

- [ ] **Step B2.8: Commit**

```bash
git add firmware/
git commit -m "feat(cca): firmware-side OTA full-TX orchestrator (CCA_CMD_OTA_FULL_TX)"
```

### Task B3: Driver to upload LDF body via stream protocol

**Files:**
- Create: `tools/cca/ota-upload.ts` (or add `--upload-firmware` flag to `tools/cca/ota-tx.ts`)

To use the firmware-side path end-to-end, host code needs to upload the LDF body first via `STREAM_CMD_OTA_UPLOAD_*`. We add this as a separate tool to keep concerns clean — the host-side track 2 is independent of this.

(Decision: keep this as a separate small tool — `tools/cca/ota-upload.ts` — so the existing `tools/cca/ota-tx.ts` (Track 2) doesn't depend on the new firmware commands.)

- [ ] **Step B3.1: Implement upload tool**

```typescript
#!/usr/bin/env npx tsx
/**
 * Upload an LDF body to the Nucleo's OTA session buffer via stream protocol.
 *
 * Usage:
 *   npx tsx tools/cca/ota-upload.ts --ldf <path> [--host <ip>]
 *
 * After upload completes, run `cca ota-tx <subnet> <serial>` on the Nucleo
 * to start the full-OTA transmit. See docs/firmware-re/powpak-conversion-attack.md.
 */

import { readFileSync } from "node:fs";
import { createSocket } from "node:dgram";
import { setTimeout as sleep } from "node:timers/promises";
import { stripLdfHeader } from "../../lib/ldf";
import { config } from "../../lib/config";

const STREAM_CMD_KEEPALIVE = 0x00;
const STREAM_CMD_OTA_UPLOAD_START = 0x18;
const STREAM_CMD_OTA_UPLOAD_CHUNK = 0x19;
const STREAM_CMD_OTA_UPLOAD_END = 0x1a;
const PORT = 9433;
const CHUNK_SIZE = 240;

function getArg(name: string): string | undefined {
  const i = process.argv.indexOf(name);
  return i !== -1 ? process.argv[i + 1] : undefined;
}

async function main(): Promise<void> {
  const ldfPath = getArg("--ldf");
  const host = getArg("--host") ?? config.openBridge;
  if (!ldfPath) {
    console.error("Usage: npx tsx tools/cca/ota-upload.ts --ldf <path> [--host <ip>]");
    process.exit(1);
  }
  const file = readFileSync(ldfPath);
  const body = stripLdfHeader(new Uint8Array(file.buffer, file.byteOffset, file.byteLength));
  console.log(`[ota-upload] body: ${body.length} bytes`);

  const sock = createSocket("udp4");
  await new Promise<void>((r) => sock.bind(0, () => r()));

  const send = (cmd: number, data: Uint8Array): void => {
    const frame = Buffer.alloc(2 + data.length);
    frame[0] = cmd;
    frame[1] = data.length;
    Buffer.from(data).copy(frame, 2);
    sock.send(frame, 0, frame.length, PORT, host);
  };

  send(STREAM_CMD_KEEPALIVE, new Uint8Array(0));
  await sleep(100);

  // START
  const startData = new Uint8Array(4);
  new DataView(startData.buffer).setUint32(0, body.length, true);
  send(STREAM_CMD_OTA_UPLOAD_START, startData);
  await sleep(50);

  // CHUNKS
  for (let chunkIdx = 0, off = 0; off < body.length; chunkIdx++, off += CHUNK_SIZE) {
    const slice = body.subarray(off, Math.min(off + CHUNK_SIZE, body.length));
    const data = new Uint8Array(2 + slice.length);
    data[0] = (chunkIdx >> 8) & 0xff;
    data[1] = chunkIdx & 0xff;
    data.set(slice, 2);
    send(STREAM_CMD_OTA_UPLOAD_CHUNK, data);
    if (chunkIdx % 10 === 9) await sleep(5); // brief breather every 10 chunks
  }
  await sleep(100);

  // END
  send(STREAM_CMD_OTA_UPLOAD_END, new Uint8Array(0));
  await sleep(200);
  sock.close();
  console.log(`[ota-upload] uploaded ${body.length} bytes in ${Math.ceil(body.length / CHUNK_SIZE)} chunks`);
}

main().catch((e) => { console.error(e); process.exit(1); });
```

- [ ] **Step B3.2: Add npm script + commit**

```bash
git add tools/cca/ota-upload.ts package.json
git commit -m "feat(cca): host tool to upload LDF body to Nucleo OTA session"
```

### Task B4: Open Track 1 PR

- [ ] **Step B4.1: Push & PR**

```bash
gh pr create --title "feat(cca): firmware-side OTA full-TX orchestrator" --body "..."
```

PR body:
```
## Summary
- Static LDF body buffer + stream protocol upload commands (STREAM_CMD_OTA_UPLOAD_{START,CHUNK,END}).
- New `cca_ota_full_tx_walk` orchestrator in cca_ota_tx.h (pure, callback-based; testable without TX engine).
- Shell command `cca ota-tx <subnet> <serial>` + `CCA_CMD_OTA_FULL_TX = 0x1E`.
- Host tool `tools/cca/ota-upload.ts` to upload an LDF body to the Nucleo before running `cca ota-tx`.
- Track 1 of the Phase 2b plan; host-side track is independent and ships separately.

## Test plan
- [x] `firmware/build-host/test_runner` — 6 new tests for cca_ota_session, 3 new tests for cca_ota_full_tx_walk all pass
- [x] ARM build clean
- [x] `npm run lint && npm run typecheck`
- [ ] Live-fire against sacrificial RMJ (deferred — explicitly out of scope for this PR)
```

---

## Task C: Update docs and final commit

- [ ] **Step C1: Update docs/firmware-re/powpak-conversion-attack.md §"Phase 2b"**

Mark Phase 2b orchestration as built (firmware-side and host-side). Note: not yet hardware-tested — that's a separate plan that requires a sacrificial RMJ.

```bash
git commit -m "docs(powpak): mark Phase 2b orchestration as built (firmware + host)"
```

---

## Self-Review

**Spec coverage:**
- Track 1 firmware-side `CCA_CMD_OTA_FULL_TX`: covered by tasks B1-B2 (session buffer + upload + orchestrator + shell wiring + tests).
- Track 2 host-side `tools/cca/ota-tx.ts`: covered by tasks A1-A3 (TS builder + LDF strip + driver + npm script).
- TDD all builder code: covered (tests precede implementation in every task).
- No git hook bypass: implicit in default git commit flow.
- No TX to live hardware: explicit in PR test plans; both tracks support `--dry-run` or callback-based tests.
- Use venv for Python: no Python is needed for either track.

**Type consistency:**
- All TS imports/exports match: `buildBeginTransfer`, `buildChangeAddressOffset`, `buildTransferData`, `OtaChunkIter`, `stripLdfHeader`, `LDF_HEADER_LEN`.
- Firmware names match: `cca_ota_session_*` family, `cca_ota_full_tx_walk`, `CCA_CMD_OTA_FULL_TX`.

**Placeholder scan:** none.

---
