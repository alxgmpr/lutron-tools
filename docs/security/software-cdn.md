# Lutron Software & Firmware CDN Inventory

Date: 2026-04-11

Complete inventory of all publicly accessible Lutron software downloads and device firmware
discovered through CDN enumeration. No authentication required for any of these URLs.

## 1. RadioRA 2 Programming Software

S3 bucket: `RadioRA2`
Pattern: `https://s3.amazonaws.com/RadioRA2/RadioRA+2+{version}-full.exe`

The standalone RA2 commissioning application (Windows). Used to discover, commission,
and program RadioRA 2 Main Repeaters. Separate from Lutron Designer.

| Version | URL | Notes |
|---------|-----|-------|
| 7.0 | `https://s3.amazonaws.com/RadioRA2/RadioRA+2+7.0-full.exe` | Earliest available |
| 7.1 | `https://s3.amazonaws.com/RadioRA2/RadioRA+2+7.1-full.exe` | |
| 7.5 | `https://s3.amazonaws.com/RadioRA2/RadioRA+2+7.5-full.exe` | |
| 8.0 | `https://s3.amazonaws.com/RadioRA2/RadioRA+2+8.0-full.exe` | |
| 8.1 | `https://s3.amazonaws.com/RadioRA2/RadioRA+2+8.1-full.exe` | |
| 8.2 | `https://s3.amazonaws.com/RadioRA2/RadioRA+2+8.2-full.exe` | |
| 9.0 | `https://s3.amazonaws.com/RadioRA2/RadioRA+2+9.0-full.exe` | Pre-TCP, multicast only |
| 9.1 | `https://s3.amazonaws.com/RadioRA2/RadioRA+2+9.1-full.exe` | |
| 9.2 | `https://s3.amazonaws.com/RadioRA2/RadioRA+2+9.2-full.exe` | |
| 9.3 | `https://s3.amazonaws.com/RadioRA2/RadioRA+2+9.3-full.exe` | |
| 10.0 | `https://s3.amazonaws.com/RadioRA2/RadioRA+2+10.0-full.exe` | |
| 10.1 | `https://s3.amazonaws.com/RadioRA2/RadioRA+2+10.1-full.exe` | |
| 10.2 | `https://s3.amazonaws.com/RadioRA2/RadioRA+2+10.2-full.exe` | |
| 10.4 | `https://s3.amazonaws.com/RadioRA2/RadioRA+2+10.4-full.exe` | |
| 10.5 | `https://s3.amazonaws.com/RadioRA2/RadioRA+2+10.5-full.exe` | |
| 10.6 | `https://s3.amazonaws.com/RadioRA2/RadioRA+2+10.6-full.exe` | |
| 10.7 | `https://s3.amazonaws.com/RadioRA2/RadioRA+2+10.7-full.exe` | Listed on archive page |
| 11.0 | `https://s3.amazonaws.com/RadioRA2/RadioRA+2+11.0-full.exe` | TCP commissioning added for RA2 |
| 11.6.0 | `https://s3.amazonaws.com/RadioRA2/RadioRA+2+11.6.0-full.exe` | Listed on archive page |
| 12.10.0 | `https://s3.amazonaws.com/RadioRA2/RadioRA+2+12.10.0-full.exe` | Latest, listed on archive page |

Archive page: `https://designer-installers.iot.lutron.io/releases/RA2-archive.html`

## 2. Lutron Designer (Legacy — HWQS/Quantum/Vive/myRoom)

S3 bucket: `LutronDesigner`
Pattern: `https://s3.amazonaws.com/LutronDesigner/{version}/Lutron+Designer+{version}.exe`

The old Lutron Designer for HomeWorks QS, Quantum, Vive, and myRoom. NOT for RadioRA 2.
NOT for RA3/HWQSX (those use the modern Designer below). Requires myLutron auth
(see `tools/auth-bypass/` for offline credential forging) for full product access.

Auth endpoint: `designeruat.lutron.com/myLutron/myLutron.svc/Authenticate`

| Version | URL | Status |
|---------|-----|--------|
| 10.0 | `https://s3.amazonaws.com/LutronDesigner/10.0/Lutron+Designer+10.0.exe` | 200 |
| 11.0 | `https://s3.amazonaws.com/LutronDesigner/11.0/Lutron+Designer+11.0.exe` | 200 |
| 11.1 | `https://s3.amazonaws.com/LutronDesigner/11.1/Lutron+Designer+11.1.exe` | 200 |
| 11.2 | `https://s3.amazonaws.com/LutronDesigner/11.2/Lutron+Designer+11.2.exe` | 200 |
| 11.3 | `https://s3.amazonaws.com/LutronDesigner/11.3/Lutron+Designer+11.3.exe` | 200 |

Versions 12.x+: 403 (access restricted).

## 3. Lutron Designer (Modern — RA3/HWQSX/Athena)

CDN: `designer-installers.iot.lutron.io`
Pattern (MSIX): `https://designer-installers.iot.lutron.io/releases/Lutron+Designer+FixedVersion+{version}.msix`
Pattern (EXE): `https://designer-installers.iot.lutron.io/releases/Lutron+Designer+{version}.exe`

Archive page: `https://designer-installers.iot.lutron.io/releases/archive.html`

Auth endpoint: `designer-relay.lutron.com/myLutron/myLutron.svc/AuthenticateCode`

| Version | Type | Date |
|---------|------|------|
| 21.6.0.811 | EXE | Dec 2021 |
| 21.7.2.2567 | EXE | Feb 2022 |
| 21.8.0.3004 | EXE | Mar 2022 |
| 22.0.0.3699 | EXE | Apr 2022 |
| 22.1.1.5653 | EXE | Jun 2022 |
| 22.2.0.5227 | EXE | Jun 2022 |
| 22.3.0.6411 | EXE | Jul 2022 |
| 22.4.0.7531 | EXE | Sep 2022 |
| 22.5.1.8408 | EXE | Oct 2022 |
| 22.6.0.9046 | EXE | Nov 2022 |
| 22.7.0.9937 | EXE | Dec 2022 |
| 22.8.1.11089 | EXE | Feb 2023 |
| 23.0.0.11923 | EXE | Mar 2023 |
| 23.1.1.13631 | EXE | May 2023 |
| 23.2.2.14097 | EXE | Jun 2023 |
| 23.3.0.14620 | EXE | Jul 2023 |
| 23.4.0.15294 | EXE | Aug 2023 |
| 23.5.2.16449 | EXE | Sep 2023 |
| 23.6.1.17758 | EXE | Nov 2023 |
| 23.7.0.17651 | EXE | Nov 2023 |
| 23.8.0.18657 | EXE | Dec 2023 |
| 23.9.0.19674 | EXE | Jan 2024 |
| 24.0.0.20335 | EXE | Feb 2024 |
| 24.1.1.21847 | EXE | Apr 2024 |
| 24.2.1.22543 | EXE | May 2024 |
| 24.3.1.23696 | EXE | Jul 2024 |
| 24.4.1.24535 | EXE | Aug 2024 |
| 24.5.2.25388 | EXE | Sep 2024 |
| 24.6.0.23886 | EXE | Oct 2024 |
| 24.7.2.100 | EXE | Nov 2024 |
| 24.8.1.100 | EXE | Dec 2024 |
| 25.0.1.100 | EXE | Jan 2025 |
| 25.1.0.112 | EXE | Feb 2025 |
| 25.1.1.100 | EXE | Mar 2025 |
| 25.2.0.112 | EXE | Mar 2025 |
| 25.3.1.100 | EXE | Apr 2025 |
| 25.4.1.100 | EXE | Jun 2025 |
| 25.5.1.101 | EXE | Jul 2025 |
| 25.6.0.113 | EXE | Jul 2025 |
| 25.7.0.116 | EXE | Sep 2025 |
| 25.7.1.100 | EXE | Sep 2025 |
| 25.8.0.113 | EXE | Sep 2025 |
| 25.8.1.100 | EXE | Oct 2025 |
| 25.9.0.114 | MSIX | Nov 2025 |
| 25.9.1.101 | MSIX | Nov 2025 |
| 25.10.0.112 | MSIX | Dec 2025 |
| 26.0.0.110 | MSIX | Jan 2026 |
| 26.0.1.100 | MSIX | Feb 2026 |
| 26.0.2.100 | MSIX | Feb 2026 |
| 26.1.0.112 | MSIX | Mar 2026 |
| 26.1.1.100 | MSIX | Mar 2026 |
| 26.2.0.113 | MSIX | Apr 2026 |

## 4. Device Firmware (firmware-downloads.iot.lutron.io)

See `docs/security/firmware-cdn.md` for full details. Summary of available firmware:

### Caseta / RA2 Select (unencrypted .deb)
CDN: `firmware-downloads.iot.lutron.io/caseta-ra2select/final/`
329 versions from v08.00.06 through v08.28.02. All unencrypted.

### Caseta SmartBridge (unencrypted .deb)
S3: `caseta.s3.amazonaws.com/`
Version v02.08.00.

### Connect Bridge — HW/RA2 HomeKit bridge (unencrypted .deb)
CDN: `firmware-downloads.iot.lutron.io/connect/alpha/`

### Phoenix / RA3 / HWQSX (manifests only, .debs purged)
CDN: `firmware-downloads.iot.lutron.io/phoenix/final/`
Packages.gz manifests survive but actual .deb files are 404.

### Caseta Pro / lite-heron (manifests only, .debs purged)
CDN: `firmware-downloads.iot.lutron.io/lite-heron/final/`

### Vive Hub (encrypted .vive)
CDN: `firmware-downloads.iot.lutron.io/vive/Release/`
4 recent versions, AES-128-CBC encrypted.

### Vive Hub Prototype (unencrypted .deb, 2016)
S3: `s3.amazonaws.com/vive-hub/`

### Older QS Class (RadioRA 2, HWQS, Quantum) — NOT ON CDN
No firmware found on any CDN for the ColdFire MCF527x processors.
Firmware updates for these devices are pushed via the RadioRA 2 or
HWQS programming software over TFTP (port 69) + UDP 777.
The firmware binary is embedded in the programming software installer.

## 5. Lutron Documentation (assets.lutron.com)

| Document | URL |
|----------|-----|
| RA2/HWQS Networking Guide (App Note #731) | `https://assets.lutron.com/a/documents/residential_systems_networking_guide.pdf` |
| RA2 Main Repeater Firmware Recovery | `https://assets.lutron.com/a/documents/radiora2_main_repeater_firmware_update_recovery_process.pdf` |
| RA2 Main Repeater Installation | `https://assets.lutron.com/a/documents/044-159c.pdf` |
| RA2 Manual Setup Guide | `https://assets.lutron.com/a/documents/044-254b.pdf` |
