# Caseta Bridge as MRF2 Cross-Flash Vehicle — Feasibility

Investigation results from 2026-04-30 on whether the Caseta Pro bridge can be subverted to drive an OTA against an MRF2-3PD-1 with HQR/HWQS firmware, sidestepping the unknowns in our direct-drive [`tools/cca/ota-tx.ts`](../../tools/cca/ota-tx.ts) `--mcu hcs08` path.

**TL;DR:** The bridge IS subvertible (no pair crypto, runtime override flag exists, on-wire transmission is byte-raw with a 31-byte skip). But it doesn't *help* compared to direct drive — the on-device unknown (does MRF2's HCS08 bootloader accept plaintext LDF or require encrypted PFF?) is silicon-side and unaffected by which TX path drives the wire. Caseta-route ends in the same gate as direct-drive. Recommendation: validate plaintext-LDF acceptance against a sacrificial PowPak under direct drive first.

## Caseta lutron-core has a runtime override flag for DC enforcement

`ActivationDeviceClassChecksOverrideFlag` is a real, named, runtime-toggleable feature flag in both Phoenix RA3 and Caseta Pro `lutron-core`. When set true in the JSON config, it disables the family/product DeviceClass mismatch check during pair activation for both CCA and CCX devices.

### Binary anatomy (Phoenix RA3, `lutron-core-26.02.15f000`, 26 MB ARM ELF)

| Address | Symbol/string | Notes |
|---|---|---|
| `0x1554698` | string `"ActivationDeviceClassChecksOverrideFlag"` | only one xref |
| `0x9f63dc` | `sub_9f63dc` (loader) | reads flag from config tree via `sub_1484f80(&result, config_tree, "ActivationDeviceClassChecksOverrideFlag")` |
| `0x198a06c` | global byte (presence) | set to 1 if flag is present and bool-typed |
| `0x198a06d` | global byte (value) | the actual bool |
| `0x9f63dc → arg1+0x7d4 / 0x7d5` | writer offsets | the loader stores presence/value at these struct offsets (which alias `0x198a06c/d`) |

Read sites — both gated on `data_198a06c != 0 && data_198a06d != 0`:

- `sub_187264` at `0x188ab0` and `0x188b40` — CCA pair-time check; bypass logs `"Disabled device class checks while activating CCA device: %s, %s (%s)"` (string at `0x14a95f0`)
- `sub_1e6938` at `0x1e6bc8` and `0x1e6bd4` — CCX pair-time check; bypass logs `"Disabled device class checks while activating CCX device: %s, %s (%s)"` (string at `0x14af400`)

Both checks compare a masked DC: `(DC & 0xFFFF0000) | 0x0101`. Only the upper 16 bits (family + product) matter; the lower 16 bits (variant) are masked off. The "expected" DC comes from a SQLite Device row via vtable `*(*(arg2 + 0xc) + 0x5c)` (so this is the *re-addressing* path, where the bridge already has an enrollment).

### Caseta confirmation

The same string and same bypass logs are present in the Caseta Pro `lutron-core` (pulled from `10.1.1.37:/usr/sbin/lutron-core`, 11.2 MB ARM ELF). Verified by `strings | grep ActivationDeviceClassChecksOverrideFlag`. Unified codebase claim from [caseta-cca-ota.md](caseta-cca-ota.md) holds.

### Config injection point

`/etc/lutron.d/lutron.conf` on the bridge (top-level JSON, 128 lines, sibling to `SqliteDebugEnabled`, `WatchdogLogReportingEnabled`, etc.). Config tree is loaded by `sub_9ffc30` at startup. Add `"ActivationDeviceClassChecksOverrideFlag": true` and restart `lutron-core` (`monit restart lutron-core` or equivalent).

## The bridge ships file bytes raw, skipping the first 31 bytes

Empirical finding from cross-correlating the captured Caseta REP2 OTA against known PFFs.

**Capture:** `data/captures/cca-ota-20260428-190439.packets.jsonl` — 3731 `06 02` TransferData chunks, 31 bytes per chunk, against a Vogelkop dimmer / DVRF-6L target.

**Method:** Extract chunk[0]'s 31 firmware bytes and search across the entire local PFF corpus.

```python
chunk[0] bytes 20-50 = 8337364070a13ce543cd67a661d6d7b67187ddc2922e90ef6eb80402dd3745
HIT @ 0x1f in dvrf6l-v3.021.pff (= 07911506_v3.021_VogelkopDimmerAppCaseta.pff, 187060 bytes)
```

The on-wire TransferData chunks match the PFF byte-for-byte starting at offset `0x1f` (decimal 31). The bridge **does not transform, encrypt, or otherwise modify** the file before transmission. It just skips the first 31 bytes (which are PFF Major + Minor + start-of-signature blob — bridge-side metadata only) and ships everything else through the standard `06 02` TransferData wire opcode.

This is a load-bearing finding: it means the bridge would deliver any bytes we drop in the manifest path *unchanged* (modulo the 31-byte skip).

## PFF body is encrypted, LDF body is plaintext, no overlap exists

Entropy comparison across our firmware corpus:

| File | MCU target | Body entropy | Verdict |
|---|---|---|---|
| `HWQS_3PD_3.08.LDF` | HCS08 | **6.53** | Plaintext compiled HCS08 (instructions visible) |
| `cca-basenji-app-release-002.026...pff` | EFR32 | **7.94** | Encrypted |
| `07910242_v2.05_CasetaDimmerApp.pff` (PD-6WCL) | EFR32 (or HCS08?) | **7.95** | Encrypted |

7.94+ bits/byte over multi-KB regions is essentially indistinguishable from random — both PFFs are wrapping ciphertext (likely AES). The 6.53 entropy of the LDF is what compiled code looks like (instruction byte patterns repeat).

**The PFF/LDF format split is along *product line*, not MCU**:
- Caseta + Phoenix bridge OTA → encrypted PFF (BASENJI/eagle-owl/bananaquit/Vogelkop/PD-6WCL)
- Designer (HW-CCA) → plaintext LDF (HQRD/HQRT/HWQS/MRF2/PowPak/RR-2 Maestro)

**No device exists in both formats.** Designer's MSIX has 45 LDF files and zero PFFs. Caseta's manifest has 15 PFFs and zero LDFs. Direct decrypt-vs-plaintext comparison is therefore not possible from our corpus alone — would need either a PFF decryption key (device-side, undisclosed) or a captured wire stream of a known-LDF firmware delivery.

## Caseta DC enrollment gates (both must permit)

Two gates between "MRF2 announces over the air" and "bridge accepts and OTAs":

### Gate 1: DeviceClassInfo (in `/var/db/lutron-db.sqlite`)

```sql
SELECT printf('0x%08X', DeviceClass), ModelNumber, Description, LeapDeviceTypeID
FROM DeviceClassInfo
WHERE printf('%08X', DeviceClass) LIKE '04%';
-- 52 rows, families 0x0401, 0x0405, 0x0407, 0x040C, 0x040E, 0x040F, 0x0413,
-- 0x0414, 0x0415, 0x042A, 0x0432, 0x0433, 0x0434, 0x0435, 0x0441, 0x0442,
-- 0x0446, 0x0447, 0x0448, 0x0449, 0x044A, 0x044B, 0x044C, 0x044D, 0x044E,
-- 0x044F, 0x0450, 0x0451, 0x0452, 0x0454, 0x0455, 0x0456, 0x0457, 0x0458,
-- 0x0459, 0x045A, 0x045B, 0x045C, 0x045D, 0x0460, 0x0461, 0x0462, 0x0463,
-- 0x0464, 0x0465, 0x0466, 0x0468
```

`0x0424` (MRF2-3PD-1's family) is **not** in the table. Adjacent families that ARE present:
- `0x04130201 MRF2-8S-DV-B` — Maestro Wireless 8A switch (proves Caseta supports MRF2 family at all)
- `0x04140101 RR-3PD-1-XX` — RadioRA 2 Plug-In Dimmer (closest functional analog to MRF2-3PD-1)
- `0x042A0101 RRD-6NA-XX` — adjacent family

Schema:
```
DeviceClassInfoID INTEGER PRIMARY KEY
DeviceInfoID INTEGER → DeviceInfo
ModelNumber TEXT
DeviceClass INTEGER UNIQUE
DeviceClassMask INTEGER
Description TEXT
ShortDescription TEXT
LeapDeviceTypeID INTEGER → LeapDeviceType
LeapDeviceEngravingKitTypeID INTEGER (nullable)
```

Trigger `CheckUniqueMaskedDeviceClass` aborts inserts whose masked DC overlaps an existing entry. `0x04240201` doesn't overlap any current entry, so insert is permitted.

### Gate 2: SupportedDevicesInfo (per-system-type matrix)

```
SystemTypeID = 1 (Smart Bridge), 915 rows total
4 rows per supported DC (per-processor compatibility matrix)
```

Schema:
```
SupportedDevicesInfoID INTEGER PRIMARY KEY
DeviceClassInfoID → DeviceClassInfo
SystemTypeID → SystemType  (=1 for Smart Bridge)
ProcessorDeviceClassInfoID → DeviceClassInfo
ChannelSetTypeID → ChannelSetType (nullable)
UNIQUE(SystemTypeID, ProcessorDeviceClassInfoID, DeviceClassInfoID, ChannelSetTypeID)
```

To enroll `0x04240201`, would need 4 insert rows mirroring the structure of the existing `0x04130201 MRF2-8S-DV-B` rows.

### Gate 3: device-firmware-manifest.json

`/opt/lutron/device_firmware/device-firmware-manifest.json` has 15 entries. Adding a new entry for `0x04240201`:

```json
{
  "DeviceClass": "0x04240201",
  "App": {
    "Path": "firmware/MRF2_HQR_PAYLOAD.pff",
    "TargetLocation": 0,
    "TargetLocationName": "CCA",
    "ImageType": 1,
    "Sha256Hash": "<sha256 of crafted payload file>",
    "DisplayRevision": "003.008.000r000",
    "Revision": {"Major": 3, "Minor": 8, "Patch": 0, "Label": 128},
    "MinimumRevisions": [],
    "EstimatedFastUploadTimeInSeconds": 1200
  }
}
```

## Hybrid-file craft for the cross-flash attempt

If we go this route, the file dropped at the manifest path needs to align with the bridge's "skip first 31 bytes" convention. LDF format from [`lib/ldf.ts`](../../lib/ldf.ts):

```
0x00–0x3F  : ASCII filename (64 bytes, NUL-padded)
0x40–0x7F  : metadata (16 BE32 fields)
0x80+      : plaintext compiled HCS08 image  ← what direct-drive ships
```

To deliver only the body (bytes `0x80..end`) over the bridge:

```
crafted_file = [31 bytes dummy] + LDF[0x80:end]
             = 31 + 109956 bytes = 109987 bytes (for HWQS_3PD_3.08.LDF)
```

Bridge skips 31 bytes → ships `LDF[0x80:end]` byte-for-byte = same payload our direct-drive code already streams via `06 02` TransferData. Recompute SHA-256 of the crafted file, write to manifest.

**However:** this only works if the MRF2's HCS08 bootloader accepts plaintext-LDF body bytes through the same `06 nn` dispatch the PowPak HCS08 bootloader does. Which is the same unknown direct-drive faces.

## Bootloader lineage hypothesis (not byte-confirmed)

| Bootloader | Lineage | Format expected | Evidence |
|---|---|---|---|
| PD-6WCL HCS08 (Caseta in-wall dimmer) | Caseta product line | Encrypted PFF (decryption-capable) | Caseta only ships encrypted PFFs; bridge ships them raw; device must decrypt |
| PowPak HCS08 (RMJ/RMJS) | HW-CCA product line | Plaintext LDF | RE'd at body addresses 0x52BF (sync detect) and 0x1A23 (dispatcher), per [powpak.md](powpak.md) — no decryption code in the OTA path |
| MRF2-3PD-1 HCS08 | HW-CCA product line (Maestro Wireless) | **Plaintext LDF (hypothesis)** | Same family lineage as PowPak; Designer ships only plaintext LDFs for HW devices |

**Same silicon, different bootloader implementations from different product teams.** Hypothesis is plausible but not byte-confirmed without an SWD/JTAG dump of either a PD-6WCL or an MRF2.

## Why the Caseta route doesn't actually help

The bridge route's appeal was: "let the bridge handle BeginTransfer payload + framing details + retry logic by-construction-correct". But:

1. The bridge's BeginTransfer payload is calibrated for *its* manifest entries (BASENJI/eagle-owl/Vogelkop/PD-6WCL). It would emit the same payload for our injected `0x04240201` entry. Whether that payload happens to be correct for MRF2's HCS08 dispatcher is the same question we'd ask under direct drive.

2. The bridge ships file bytes raw — we already proved this. Direct drive does the same. No advantage in OTA byte-correctness.

3. The bridge cannot help with the encryption/plaintext question because its own corpus has zero plaintext-LDF entries.

4. The setup cost is significant: DB row inserts (with FK satisfaction), manifest entry, override flag, SHA-256 recomputation, bridge service restart, pair handshake — all to deliver bytes the direct-drive code already delivers in one CLI command.

5. Under direct drive we already control timing, retries, and can probe non-destructively with `cca ota-poll` (firmware shell command). No bridge equivalent.

## Recommended path forward

**Step 1 (gating experiment):** acquire a sacrificial PowPak (RMJ or RMJS, HCS08, same dispatcher as MRF2). Direct-drive against it with `tools/cca/ota-tx.ts --mcu hcs08` using a known-good LDF for that device.

- **If it works** → MRF2 will accept plaintext LDF the same way; direct-drive is the route; the Caseta investigation in this doc is moot.
- **If it doesn't** → MRF2 bootloader probably wants encrypted PFF or different framing. We'd need device-side AES key recovery (SWD/JTAG on a chip) before any RF route works.

**Step 2 (only if step 1 succeeds and we want belt-and-suspenders):** validate the Caseta hybrid-file approach against the same sacrificial PowPak before risking the MRF2.

**Do not attempt MRF2 cross-flash before step 1 succeeds.** BeginTransfer is destructive on first packet; one bricked RMJ already happened (see [powpak-conversion-attack.md §"Brick incident"](powpak-conversion-attack.md)). The MRF2 in our possession is the conversion target — we don't have a second sacrificial MRF2.

## References

- [`cca-ota-hcs08.md`](cca-ota-hcs08.md) — direct-drive `--mcu hcs08` mode
- [`cca-ota-live-capture.md`](cca-ota-live-capture.md) — Caseta REP2 → DVRF-6L OTA capture (the source of the chunk-correlation analysis above)
- [`caseta-cca-ota.md`](caseta-cca-ota.md) — Caseta Pro bridge OTA orchestration (`platform_manager_wrapper.sh -p`, manifest format, IPC)
- [`powpak.md`](powpak.md) — PowPak HCS08 bootloader RE
- [`powpak-conversion-attack.md`](powpak-conversion-attack.md) — RMJ→LMJ conversion plan (sibling attack)
- [`../protocols/cca-pairing.md`](../protocols/cca-pairing.md) — pair handshake (no crypto, family/product DC gate)
- [`../../lib/ldf.ts`](../../lib/ldf.ts) — LDF format definition
- Memory: `reference-cca-ota-wire-protocol`, `reference-phoenix-hwcca-ota-prerequisite-271`, `feedback-caseta-no-hcs08-ota`
