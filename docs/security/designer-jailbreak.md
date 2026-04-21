# Designer Jailbreak

Bypasses Lutron Designer's online authentication and feature flag requirements for offline/low-connectivity use. Enables the hidden Feature Flag Overrides screen that Lutron uses internally for testing.

## Quick Start

Apply 3 IL patches to `Lutron.Gulliver.QuantumResi.dll`, 2 patches to `Lutron.Gulliver.FeatureFlagServiceProvider.dll`, and 3 patches to `Lutron.Gulliver.Infrastructure.dll`. Rollout connectivity is NOT required. Flag overrides persist to `featureOverrides.json`.

The three Infrastructure.dll patches replace the hosts-file block + `ServicesConfig.json` URL-list hack described in [`designer-auth-bypass.md`](designer-auth-bypass.md) — Designer stays **fully online** (engraving preview/submission via PStore Apim, firmware update checks, LEAP cloud proxy all work), the myLutron server can't revoke the session, and every product channel (RA3/HWQS/Vive/Ketra/DTDT) is unlocked regardless of what the real account authorizes.

### Prerequisites

Take ownership of the MSIX app directory (one-time):

```powershell
$dir = "C:\Program Files\WindowsApps\LutronElectronics.LutronDesigner26.2.0.113_26.2.0.113_x86__hb4qhwkzq4pcy\QuantumResi"
takeown /f $dir /r /d Y
icacls $dir /grant "$env:USERNAME`:F" /t /c
Copy-Item "$dir\Lutron.Gulliver.QuantumResi.dll" "$dir\Lutron.Gulliver.QuantumResi.dll.original"
```

### Apply Patches

Close Designer before patching. All patches target `Lutron.Gulliver.QuantumResi.dll` (v26.2.0.113, 28,911,104 bytes).

#### Patch 1: Menu item visibility bypass

**Class:** `LutronMenuItem`
**Method:** `get_IsAvailable` (RVA `0x2c1c00`)
**Purpose:** Calls `IsFeatureFlagEnabled` on each menu item's type and hides the item if the associated feature flag is disabled. This patch makes all Tools menu items visible regardless of flag state.

```
File offset 0x2bfe31:
Before: 2C 7D        brfalse.s +125
After:  26 00        pop; nop
```

#### Patch 2: Navigation bypass

**Class:** `ShellViewModel`
**Method:** `ShowFeatureFlagOverrides` (RVA `0x2cdd64`)
**Purpose:** Checks `IsEnabled(FeatureFlagType.EnableFeatureFlagOverride)` before navigating to the override view. Patch removes the check.

```
File offset 0x2cbf70:
Before: 1F 42 28 B0 03 00 0A 2C 1B   ldc.i4.s 66; call IsEnabled; brfalse.s +27
After:  00 00 00 00 00 00 00 00 00   9x nop
```

#### Patch 3: LoadFlags filter bypass

**Class:** `FeatureFlagOverrideViewModel`
**Method:** `<LoadFlags>b__46_0` (RVA `0x2d8698`)
**Purpose:** The Where filter checks `get_FeatureFlagOverrideService` for each flag and excludes flags when the service is null (which it always is when `EnableFeatureFlagOverride` is false). Patch makes the filter always return true so all 262 flags populate the UI.

```
File offset 0x2d6898:
Before: 4E           tiny header (19-byte body)
After:  0A 17 2A     tiny header (2-byte body): ldc.i4.1; ret
```

### FeatureFlagServiceProvider.dll Patches

These patches make the `FeatureFlagOverrideService` always initialize, regardless of whether `EnableFeatureFlagOverride` is true in Rollout. With the service active, the real Get/Set/Clear methods work and overrides persist to `featureOverrides.json`.

#### Patch A: Setup — bypass EnableFeatureFlagOverride check

**Class:** `CloudBeesFeatureFlagService`
**Method:** `Setup` (RVA `0x21dc`)
**Purpose:** The Setup method creates the `CloudBeesFlagOverrideService` only when `IsFeatureFlagEnabled(EnableFeatureFlagOverride)` returns true. Since this flag is gated behind internal target group `613284d4ad80af9adc351d99`, the service never initializes for external users. This patch consumes the check result without branching.

```
File offset 0x43d:
Before: 2C 21        brfalse.s +33 (skip service creation)
After:  26 00        pop; nop (always create service)
```

#### Patch B: ResetOverrideService — bypass same check

**Class:** `CloudBeesFeatureFlagService`
**Method:** `ResetOverrideService` (RVA `0x230c`)
**Purpose:** Same conditional pattern as Setup. Without this patch, the service cannot be re-created after a reset.

```
File offset 0x515:
Before: 2C 21        brfalse.s +33
After:  26 00        pop; nop
```

Also clear the `CLR_STRONGNAMESIGNED` flag in the CLR header (offset 0x218: clear bit 3) to prevent assembly load failure.

### Infrastructure.dll Patches

Three patches on `Lutron.Gulliver.Infrastructure.dll` keep Designer online + unlocked without needing a forged credential file:

#### Patch I1: Wrap `AttemptAuthentication` so the real OAuth flow runs and de-auth is impossible

**Class:** `Lutron.Gulliver.Infrastructure.myLutronService.UserManager`
**Method:** `AttemptAuthentication(bool attemptTokenAuthentication)`
**Purpose:** This is the only method that hits `myLutronServiceManager.Instance` — it calls `RefreshToken(true)` for token refresh (startup) or `AuthenticateCode(user)` for the login-button OAuth path. Its return value feeds `AuthenticatefromSSO`, which flips `User.IsAuthenticated` based on the result.

A pure `return 1` stub would block the real OAuth call — breaking Bearer-authed endpoints (PStoreService/engraving Apim/etc.) that read `user.SecurityToken` at request time. Instead we wrap the original body in try/catch and always coerce the return to `AuthenticationSuccess`:

```
try {
    if (!attemptTokenAuthentication) myLutronServiceManager.Instance.AuthenticateCode(user);
    else                              myLutronServiceManager.Instance.RefreshToken(true);
} catch {}
return AuthenticationResult.AuthenticationSuccess;  // 1
```

The real call still executes — its side-effect (`UserManager.Instance.SetCurrentUser(...)` with a live `SecurityToken`) populates the bin on successful login. Any failure (offline, expired, revoked, 401, 4xx, anything) is swallowed and reported as success, so de-auth is unreachable.

Implementation lives in `PatchAttemptAuthenticationTryCatch` (`tools/dll-patcher/DllPatcher/Program.cs`), which extracts the `get_Instance` / `RefreshToken` / `AuthenticateCode` method refs + the `user` field ref from the original IL before rewriting the body.

**Intentionally NOT patched:** `User.ResetProperties(User)`. Earlier iterations stubbed it to no-op as belt-and-suspenders against de-auth, but it's called by `UserManager.AuthenticateNewUser` to clear `username`/`Code`/`CodeVerifier`/`IsAuthenticated`/`channels` before the login-button OAuth flow. Stubbing it leaves the post-logout `@Guest@` dummy username in place, which causes `AuthenticatefromSSO` to short-circuit via `IsCurrentUserADummyUser()` and never open the browser. The I1 try/catch alone is sufficient.

#### Patch I2: `User.get_ChannelTypes` → `ChannelTypes.All` (0x1FFFFFFF)

**Class:** `Lutron.Gulliver.Infrastructure.myLutronService.User`
**Method:** `get_ChannelTypes()`
**Purpose:** The server-returned channel bitmask is ignored. Every `[Flags]` bit is set, so the user effectively has every product channel (RA3, HWQS, Vive, Ketra, DTDT beta flags, currency codes, etc.) regardless of what the real myLutron account is authorized for. Downstream gates like project-open compat, toolbox model visibility, and link-type filtering all see the max bitfield.

```
New body: ldc.i4 0x1FFFFFFF; ret
```

The setter remains untouched — `SetUserChannels` still runs as normal (so `user.Channels` the string list is still populated for display); readers just always see `All`.

#### Patch I3: `User.get_AllChannelTypes` → `ChannelTypes.All` (0x1FFFFFFF)

**Class:** `Lutron.Gulliver.Infrastructure.myLutronService.User`
**Method:** `get_AllChannelTypes()`
**Purpose:** Mirror of I2 for the "all channels ever seen" getter — some call sites read this instead of `ChannelTypes`.

```
New body: ldc.i4 0x1FFFFFFF; ret
```

All three patches are applied by `tools/dll-patcher/` (search for the `--- Infrastructure.dll ---` block in `Program.cs`).

#### Previous patches 4-11 (REMOVED)

Earlier versions stubbed out 8 methods in QuantumResi.dll (`GetOriginalValueInternal`, `SetOverrideInternal`, `ClearOverrideInternal`, `GetHasOverrideInternal`) to avoid null-reference crashes when the override service was null. With the service now properly initialized by patches A and B, these stubs are no longer needed. The real methods handle persistence correctly.

## Architecture

### Authentication

Designer authenticates via myLutron service endpoints. Auth is bypassed offline by forging the `%APPDATA%\Lutron\Common\LutronData.bin` credential file with all 28 Display-named `ChannelTypes` strings — see `tools/auth-bypass/Program.cs` and `docs/security/designer-auth-bypass.md` for the full channel reference and file format. No internet or real Lutron credentials required.

### Feature Flags (Rollout/Rox SDK)

Designer uses the Rox SDK (Rollout.io, now CloudBees) for 660 remote feature flags. The SDK:

1. Fetches config from `conf.rollout.io` on startup
2. Caches to `CloudbeesFM/5ff344cfea0fccbee8632f63/configuration.json`
3. **Cryptographically signs** the cache (`signature_v0` field, RSA with pinned cert)
4. Evaluates flag conditions (target groups, percentages, version checks)

Config modification is not viable because the signature cannot be forged. The binary patches above bypass the flag system entirely at the IL level.

#### Rollout domains

```
conf.rollout.io                         configuration fetch
x-api.rollout.io                        device API
statestore.rollout.io                   state persistence
analytic.rollout.io                     analytics
push.rollout.io                         SSE push updates
rox-conf.cloudbees.io                   CloudBees config (mirror)
api.cloudbees.io                        CloudBees API (mirror)
rox-state.cloudbees.io                  CloudBees state (mirror)
fm-analytics.cloudbees.io               CloudBees analytics
sdk-notification-service.cloudbees.io   CloudBees notifications
```

These do NOT need to be blocked for the binary patches to work. The flag checks are bypassed regardless of SDK connectivity.

#### Rox SDK internals (rox-core.dll)

- `XSignatureVerifier.Verify` (RVA `0x3c04`) — RSA signature verification
- `NoOpSignatureVerifier.Verify` (RVA `0x3bd6`) — built-in no-op (2 bytes: `17 2A`)
- `ROXCertificateBase64` — pinned self-signed cert (expired Aug 2024, offset `0x1ac5e`)
- `DisableSignatureVerification` — property on `RoxOptionsBuilder`
- Application ID: `5ff344cfea0fccbee8632f63`

#### Cached config structure

```json
{
  "data": "{\"platform\":\".NETclient\",\"application\":\"5ff344cfea0fccbee8632f63\",\"experiments\":[...]}",
  "signature_v0": "TSmm1yVA/7Re7z...",
  "signed_date": "2026-04-12T07:00:53.288Z"
}
```

The `data` field is a JSON string containing 660 experiments with deployment conditions. Each condition uses a Roxx expression language (`ifThen`, `isInTargetGroup`, `semverGte`, `b64d`, etc.).

### Key Enums

#### FeatureFlagType (FeatureFlagServiceProvider.dll, 262 values)

Key values:
- 58 (0x3A): `EnableDemoModeShortcut` — always true
- 66 (0x42): `EnableFeatureFlagOverride` — gates the override UI, internal target group `613284d4ad80af9adc351d99`

#### ToolsMenuItemType (separate enum, NOT FeatureFlagType)

`CreateToolsMenu` (RVA `0x2d0ff8`) creates menu items using ToolsMenuItemType values. Each type maps internally to a FeatureFlagType for visibility. The menu item constructors:
- Token `0x0600a576` — separator/simple item
- Token `0x0600a5a1` — command item with bound command

### Feature Flag Override UI

- **View:** `Lutron.Gulliver.QuantumResi.Main.FeatureFlagOverride.FeatureFlagOverrideView`
- **ViewModel:** `FeatureFlagOverrideViewModel` — loads flags into three DataGrid sections (boolean, string, enum)
- **Override service:** `IFeatureFlagOverrideService` — reads/writes `featureOverrides.json`
- **Access:** Tools menu > Feature Flag Overrides (after patches applied)

### MSIX App Directory

```
C:\Program Files\WindowsApps\LutronElectronics.LutronDesigner26.2.0.113_26.2.0.113_x86__hb4qhwkzq4pcy\QuantumResi\
```

MSIX does NOT validate DLL integrity after `takeown` — patched DLLs load normally.

## What's Working

- **Override persistence** — toggling flags in the UI persists to `featureOverrides.json` via the `CloudBeesFlagOverrideService`. Survives Designer restarts.
- **Original values** — the "Original Value" column shows real Rollout default values.
- **All 262 flags** — the filter bypass (patch 3) shows all flags including `EnableFeatureFlagOverride` itself.

## Approaches That Failed

- **SSLKEYLOGFILE** — Neither the env var nor the SChannel registry key produced key logs. The MSIX AppContainer sandbox likely prevents write access, and .NET Core's `SslStream` may not honor the env var in this context.
- **Config patching** — Modifying the cached Rollout config and blocking domains to prevent refresh. The SDK validates the `signature_v0` RSA signature and rejects modified configs, falling back to defaults.
- **rox-core.dll signature bypass** — Patching `XSignatureVerifier.Verify` to return true. The patch was confirmed on disk but the SDK still rejected modified configs, possibly due to additional validation or the config being parsed before the verifier runs.
- **Changing ToolsMenuItemType enum values** — Changing menu item type IDs from 65/66 to 58 in `CreateToolsMenu`. This changed the menu item TYPE (making them display as "ReportPackageSeparator") rather than changing the flag gate.

## Version Info

- Designer: 26.2.0.113 (MSIX package `LutronElectronics.LutronDesigner26.2.0.113`)
- Rox SDK: 6.0.1 (.NET 6, from PDB path `rox-core.pdb`)
- Runtime: .NET Core (coreclr.dll, x86)
- VM: Windows 11 24H2 (build 26100), UTM shared network at 192.168.64.4
