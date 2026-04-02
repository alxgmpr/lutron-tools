# RE Directory Reorganization

Consolidate scattered reverse engineering artifacts into a structured `re/` directory, rename RE docs with a `re-` prefix, and clean up stale top-level directories.

## Directory Structure

### `re/` layout after reorg

```
re/
├── processors/
│   ├── phoenix-boot.screenlog       ← ph/screenlog.0
│   ├── caseta-boot.screenlog        ← caseta-uart/screenlog.0 copy
│   ├── ra2-select-boot.screenlog    ← rr-sel-uart/screenlog.0
│   └── vive-boot.log                ← ./vive-boot.log
├── app/                             (unchanged — iOS framework extraction)
└── designer/                        (unchanged — Ghidra projects, firmware images)
```

### What stays where it is

| Path | Reason |
|------|--------|
| `captures/` | Live working directory, referenced by `nucleo.ts`, `cca-capture.ts`, `cca-owt-analyze.ts` |
| `data/vive-hub/` | Extracted binaries — copyrighted, must stay gitignored in `data/` |
| `docs/claude-context/` | Serves a different purpose (Claude operational context) |
| Root `.pem` files | Used by LEAP tools via `lib/env.ts` |

## Gitignore Changes

Replace blanket `re/` ignore with targeted ignores for binary-heavy subdirs:

```diff
-re/
+re/app/
+re/designer/
```

Override the global `*.log` rule for RE processor captures:

```diff
+# RE processor captures (override *.log ignore)
+!re/processors/*.log
```

## RE Doc Renames

Files in `docs/` that are primarily reverse engineering writeups get a `re-` prefix:

| Current name | New name |
|---|---|
| `vive-hub-re.md` | `re-vive-hub.md` |
| `phoenix-rootfs-analysis.md` | `re-phoenix-rootfs.md` |
| `coproc-firmware-re.md` | `re-coproc-firmware.md` |
| `firmware-cdn-re.md` | `re-firmware-cdn.md` |
| `qsm-firmware-re.md` | `re-qsm-firmware.md` |
| `rr-sel-rep2-re.md` | `re-rr-sel-rep2.md` |
| `pd-3pcl-firmware-extraction.md` | `re-pd-3pcl-firmware.md` |
| `ble-commissioning-re.md` | `re-ble-commissioning.md` |
| `wink-hub-firmware-findings.md` | `re-wink-hub-firmware.md` |
| `leap-apk-discovery.md` | `re-leap-apk-discovery.md` |
| `leap-apk-surfaces.md` | `re-leap-apk-surfaces.md` |
| `lutron-pki.md` | `re-lutron-pki.md` |

Files **not** renamed (protocol specs, design docs, operational docs): `cca-protocol.md`, `ccx-coap-protocol.md`, `CCX.md`, `qslink-protocol.md`, `ipl-protocol.md`, `leap-routes.md`, `leap-api-exploration.md`, `dimming-curves.md`, `ra3-system.md`, `bridge-state-spec.md`, `designer-db-enumeration.md`, `cloud-leap-proxy.md`, `firmware-update-infra.md`, `stm32-toolchain-and-flashing.md`, and all others not listed above.

## Code Path Updates

None required. No TypeScript code references any of the paths being moved or renamed. The only code-referenced path (`captures/cca-sessions/`) is unchanged.

`docs/qsm-firmware-re.md` (becoming `docs/re-qsm-firmware.md`) references `re/designer/ghidra_project3/qsm_final` — that path is unchanged.

## Cleanup

| Action | Target |
|--------|--------|
| Delete | `ph/` (empty after move) |
| Delete | `caseta-uart/` (empty after move) |
| Delete | `rr-sel-uart/` (empty after move) |
| Delete | `src/` (empty, gitignored, stale) |
| Delete | `tmp/` (stale binary blobs, gitignored) |

## Tracked Files Summary

After reorg, `re/processors/` contains 4 tracked text files (UART boot captures). All other `re/` content (`re/app/`, `re/designer/`) remains gitignored.
