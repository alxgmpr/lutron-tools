# Designer Auth Bypass

Fully offline authentication bypass for Lutron Designer. No Lutron account required. Works by forging the encrypted credential file and forcing the app into offline mode, where it auto-authenticates any non-guest user.

Combines with the [DLL jailbreak](designer-jailbreak.md) (IL patches for feature flags and the 26.2 channel-compat gate — see also [designer-26.2-channel-fix](../infrastructure/designer-26.2-channel-fix.md)) for full offline operation. This bypass handles authentication only.

## How It Works

### Auth Architecture (3 layers)

| Layer | Gate | Bypass |
|---|---|---|
| Windows Principal | `EnableAuthentication` config key | Already `True` but only rejects Windows guest users |
| myLutron OAuth SSO | Browser-based login → token refresh | Forged credential file + forced offline mode |
| Channel Authorization | `User.ChannelTypes` bitfield | All channels baked into forged file |

### The Offline Fallback

When `GetInternetStatus()` returns false, `IsUserAuthenticated()` takes the offline path:

```csharp
// LoginViewModel.IsUserAuthenticated() — offline branch
UserManager.AuthenticateWithoutInternetAccess(null);
// null → !IsCurrentUserADummyUser() → true (for any non-"@Guest@" user)
```

The only check is whether the username is literally `"@Guest@"` with code `"$Guest$"`. Any other username passes.

### Credential File

**Path:** `%APPDATA%\Lutron\Common\LutronData.bin`

**Encryption:**
- Algorithm: AES-256-CBC
- Key derivation: `PasswordDeriveBytes("UserInformation", "Ivan Medvedev")`
  - Salt bytes: `[73, 118, 97, 110, 32, 77, 101, 100, 118, 101, 100, 101, 118]`
  - Key: 32 bytes from PBKDF1, IV: 16 bytes from PBKDF1
- Format: Base64(AES(UTF-16LE(JSON)))

**Machine ID:** `Win32_Processor.ProcessorId` + `Win32_BIOS.SerialNumber` (WMI). VMs typically return `"0000000000000000"` + `""`.

**JSON schema:**
```json
{
  "Username": "any@email.com",
  "SecurityToken": "fake",
  "RefreshToken": "fake",
  "Machineid": "<CPU_ID + BIOS_SERIAL>",
  "Firstname": "Offline",
  "Lastname": "User",
  "Role": "",
  "Location": "",
  "CustomRole": "",
  "Channels": 499909119,
  "AccountNumber": "",
  "AuthorizedChannels": [
    "RadioRA 3 All", "LDB", "DesVive", "DesmyRoom", "DesQuantum",
    "DesAll", "CommmyRoomLegacy", "LDBlockUsageTrac", "LDAlpha",
    "LDBeta", "OQT", "DCurrUSD", "DCurrCAD", "DCurrGBP", "DCurrEUR",
    "DCurrINR", "DCurrJPY", "DCurrBRL", "DCurrMXN", "DCurrCNY",
    "CommQuantum", "CommmyRoom", "CommAll", "QTT", "DLSI",
    "PIDHW013", "Design Ketra Legacy",
    "Beta_LutronDesignerPlus_DTDTPhaseOne",
    "Beta_LutronDesignerPlus_DTDT_OQT_Hybrid"
  ],
  "Roles": ["Standard"],
  "Permissions": ["User.View"],
  "Code": "",
  "CodeVerifier": "",
  "LutronSellingCompany": "",
  "ShipToNumber": "0",
  "UserGuid": "<any-guid>",
  "UserReferenceId": "",
  "UserRef": ""
}
```

### Internet Status Check

`RemoteServiceUtilities.GetInternetStatus()` pings URLs from `InternetConnectivityURLList` in `ServicesConfig.json`. Default list includes yahoo.com, google.com, etc. Must be patched to only contain blocked URLs so the check returns false.

**ServicesConfig.json location:**
`%LOCALAPPDATA%\Packages\LutronElectronics.LutronDesigner*\LocalCache\Local\Lutron Designer <version>\ServicesConfig.json`

## Setup

### Tool

```bash
cd tools/auth-bypass

# Generate forged credential file
dotnet run -- --machine-id "<CPU_ID><BIOS_SERIAL>" --output LutronData.bin

# Decrypt an existing credential file
dotnet run -- --dump /path/to/LutronData.bin

# Encrypt custom JSON
dotnet run -- --encrypt payload.json --output LutronData.bin
```

### Deploy (3 files to modify)

**1. Forged credential file:**
```
Copy LutronData.bin → %APPDATA%\Lutron\Common\LutronData.bin
```

**2. Patch ServicesConfig.json** — change `InternetConnectivityURLList` to contain only the blocked domain:
```json
"InternetConnectivityURLList": ["https://designer-relay.lutron.com/ping"]
```

Also redirect auth URL as backup:
```json
"MyLutronLoginServiceURL": "https://192.0.2.1/myLutron/myLutron.svc/"
```

**3. Block auth server in hosts file:**
```
# C:\Windows\System32\drivers\etc\hosts
127.0.0.1 designer-relay.lutron.com
127.0.0.1 mylutronservices.lutron.com
```

### Getting the Machine ID

```powershell
# On the target Windows machine
(Get-WmiObject Win32_Processor).ProcessorId
(Get-WmiObject Win32_BIOS).SerialNumber
# Machine ID = ProcessorId + SerialNumber concatenated
# VMs typically: "0000000000000000" + ""
```

### Undoing

```powershell
# Remove hosts entries
(Get-Content C:\Windows\System32\drivers\etc\hosts) -notmatch 'lutron' |
  Set-Content C:\Windows\System32\drivers\etc\hosts

# Delete forged credential file
Remove-Item "$env:APPDATA\Lutron\Common\LutronData.bin"

# Restore ServicesConfig.json (delete it; app recreates from embedded default on next launch)
```

## Key Code Paths (Lutron.Gulliver.Infrastructure.dll)

| Class | Method | Role |
|---|---|---|
| `UserManager` | `FetchUserDetail()` | Reads/decrypts `LutronData.bin`, validates machine ID |
| `UserManager` | `AuthenticateWithoutInternetAccess(null)` | Sets auth=true for non-guest users |
| `UserManager` | `SetUserChannels(user, channels)` | Parses channel strings → `ChannelTypes` bitfield |
| `UserManager` | `IsCurrentUserADummyUser()` | Only true for `@Guest@`/`$Guest$` |
| `User` | `SetAuthencticationStatus(bool)` | Single write path to `IsAuthenticated` |
| `Crypto` | `Decrypt(cipherText, "UserInformation")` | AES decrypt with hardcoded PBKDF1 key |
| `RemoteServiceUtilities` | `GetInternetStatus()` | Pings `InternetConnectivityURLList` URLs |
| `GulliverConfiguration` | `IsAuthenticationEnabled` | Reads `EnableAuthentication` app config key |

## Also Discovered

### No Service Rep / Internal Roles

There are no hidden service, technician, or internal employee roles in the codebase. `User.Roles` and `User.Permissions` come from the server but are never checked client-side. ALL feature gating is via the `ChannelTypes` bitfield.

`IsLutronUser` (email contains `@lutron.com`) only gates the Yellow GUI diagnostic banner on support files.

### Built-in Server-Down Fallback

Even without the offline bypass, `AuthenticatefromSSO(isFetchedLocally: true)` auto-promotes `ServerError`/`Timeout` to `AuthenticationSuccess`. This is the intended "server is down" resilience path and is what would naturally fire if Lutron went out of business with a valid cached credential file.

### ServerModeUser

`UserManager.CreateServerModeUser()` creates a pre-authenticated user with `CommQuantum` channel. Triggered when `IsOnPremModeEnabled=true` in config and internet is down. Limited to Quantum commissioning only.

### Hardcoded Telnet Credentials

`SystemUser` static constructor whitelists: `lutron/integration`, `admin/admin1`, `itsthebishop/""`.
