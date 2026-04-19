# Designer → Processor Activation PKI

How Lutron Designer provisions the per-project "SubSystem CA" on a fresh RA3
processor, and whether that bootstrap can be replicated by a third party.

Reversed from `Lutron Designer 26.0.2.100` DLLs, 2026-04-19.

**TL;DR.**

- The project SubSystem CA is **locally generated inside Designer** (BouncyCastle
  ECDSA P-384, self-signed, CN=`Lutron Project SubSystem Certificate Authority`).
  No Lutron cloud involvement, no hardware root of trust, no cross-project
  hierarchy.
- Designer installs it on the processor over a **pre-authenticated "LAP" channel
  on port 8083**, authenticated by **`residential_local_access.pfx`** — a generic
  client cert that ships in every Designer MSIX with an empty password.
- Consequence: any attacker on the same L2/L3 network as a **fresh (or
  factory-reset) RA3 processor**, in possession of a copy of Designer, can
  install their own SubSystem CA and then speak IPL:8902.
- There is **no cross-project PKI weakness**. Stealing one project's SubSystem CA
  does not help against another project — each project's CA is a self-signed root.

---

## 1. What the Processor Actually Trusts

```
┌─────────────────────────────────────────────┐
│  Per-project, locally-generated CA          │
│  CN=Lutron Project SubSystem Certificate    │
│     Authority                               │
│  ECDSA P-384 / SHA-384, self-signed         │
│  Not chained to any Lutron root             │
│  Stored in the .ra3 project file and        │
│  (on the Designer workstation) cert_v2.pfx  │
└─────────────┬───────────────────────────────┘
              │ signs
              ▼
       Client certs for:
         • Designer ↔ IPL:8902   (CertificateType.SubLoobK)
         • processor self-cert on the same mTLS stack
```

Source: `Lutron.Gulliver.Infrastructure.dll!CommunicationFramework.CertificateHelpers`:

```csharp
// X509CertificateIssuerInfo — the defaults the builder uses unless overridden
public X509CertificateIssuerInfo()
    : this("Lutron Project SubSystem Certificate Authority",
           "Lutron Electronics Co., Inc.", "Coopersburg", "PA", "US",
           DateTime.UtcNow.AddYears(100))
```

```csharp
// X509CertificateBuilder.GenerateCertificateUsingECDSAlgorithm
AsymmetricCipherKeyPair kp = GenerateECKeyPair(keyLength);           // 384
ISignatureFactory sig = new Asn1SignatureFactory(ECDsaWithSha384Algo, kp.Private);
var cert = GenerateCertificate(issuerInfo, kp, sig);                 // self-signed
var derKey = PrivateKeyInfoFactory.CreatePrivateKeyInfo(kp.Private).GetDerEncoded();
return new KeyValuePair<X509Certificate2, byte[]>(cert, derKey);
```

The "subject == issuer" equality in `GenerateCertificate` confirms this is a
root CA, not an intermediate. There is **no signature by any Lutron-controlled
key**. Every project is its own PKI island.

Two custom extensions appear under Lutron's Private Enterprise Number arc
`1.3.6.1.4.1.40073.1.*`:

| OID                        | Symbol                                     | Use |
|----------------------------|--------------------------------------------|-----|
| `1.3.6.1.4.1.40073.1.9`    | `WhiteListVerificationRequiredExtensionID` | `CertificateType.SubLoobK` cert: marks the processor-signed leaf |
| `1.3.6.1.4.1.40073.1.10`   | `WhiteListSignExtendedKeyUsageID`          | Added to CA + `SubCloudSub` certs; paired with `IdKPClientAuth` |

`CertificateType` only has two values: `SubLoobK` and `SubCloudSub`. The former
is for the local activation flow described here; the latter is for Lutron's
**cloud escalation** path (out of scope for IPL/LEAP; see §7).

## 2. The Three Ports and the Three PKIs

| Port | Protocol | Server cert chain | Client cert expected | Established via |
|------|----------|-------------------|----------------------|-----------------|
| 8081 | LEAP (JSON) | `radioRa3-products` tree | project SubSystem CA after activation | LEAP client |
| 8083 | **LAP** (JSON, activation) | `radioRa3-products` tree | `residential_local_access.pfx` (shipped) | Designer `ProcessorLapClient` |
| 8902 | IPL (binary) | `radioRa3-products` tree | project SubSystem CA | `tools/ipl-cmd.ts` once CA is in hand |

The **server cert** trust anchors are the five product roots shipped in
`QuantumResi/BinDirectory/CertificateStore/`:

```
radioRa3_products.crt
homeworksqs_products.crt
athena_products.crt
myroom_products.crt
quantum_products.crt
```

`LapConnectionHelper.VerifyLAPConnection` only checks the processor's cert
chain against the product root for the current product type (with an
`AllowDevelopmentProcessorCertificate` config flag that additionally trusts
`*-products-dev` roots). **No per-processor pinning.**

```csharp
// LapConnectionHelper.RootCertificateNamesByProduct (RA3 branch)
case ProductType.RadioRA2:
    span[0] = "CN=radioRa3-products, O=\"Lutron Electronics Co., Inc.\", " +
              "L=Coopersburg, S=Pennsylvania, C=US";
```

This aligns with the existing finding in [ipl.md](../protocols/ipl.md) that
IPL:8902 server certs chain up to `radioRa3_products.crt`. **The same roots
are used on 8081 and 8083.**

## 3. The LAP Bootstrap: How the CA Gets to the Processor

### 3.1 The shared client cert

The port-8083 bootstrap channel is authenticated by a **shared cert that ships
in every Designer install**:

```csharp
// LapConnectionHelper.LAPCertificateFileName (RA3 / QuantumResi branch)
case ProductType.QuantumResi:
    return "residential_local_access.pfx";
```

```csharp
private static X509Certificate2? GetApplicationLAPCertificate() {
    return new X509Certificate2(
        Path.Combine(GulliverCoreConfiguration.Instance.CertificateFolderPath,
                     LAPCertificateFileName),
        string.Empty);   // <-- blank PFX password
}
```

`residential_local_access.pfx` is signed by `CN=Lutron Designer Certificate
Authority` and is bit-for-bit identical across every Designer workstation. It
is extractable from the MSIX (`QuantumResi/BinDirectory/CertificateStore/`) with
zero effort and opens with an empty password. Treat it as **public**.

### 3.2 `ClaimProcessor` (initial activation)

`ProcessorModelView.ClaimProcessor` drives the flow (`Lutron.Gulliver.ModelViews.dll`):

```csharp
using ProcessorLapClient processorLapClient =
    LapConnectionHelper.CreateLAPConnection(
        discoveredProcessorModelView.IPAddress,   // from mDNS/multicast discovery
        VerifyLAPConnection,                      // trusts radioRa3-products
        cancellationTokenSource.Token);
// CreateLAPConnection with no cert arg defaults to residential_local_access.pfx
```

Once the TLS handshake on :8083 is up, Designer generates the CA (idempotent —
only runs if `SubSystemCertificateV2` is null on the ProcessorSystem domain
object) and hands it off to the LAP client:

```csharp
// ProcessorModelView.ClaimProcessorOnLap
AssignedProcessorSystemModelView.ProcessorSystem.GenerateSubSystemCertificateV2();
Tuple<bool, X509Certificate2> t = lapClient.ClaimProcessor(
    SubSystemCertificate,       // CA cert
    Processor.SubSystemPrivateKey, // CA private key bytes (stays in Designer,
                                   // not transmitted — used locally to sign)
    cancellationToken);
if (t.Item1) { LoobKey = t.Item2; OnClaimingSucceeded(); ... }
```

`ProcessorLapClient.ClaimProcessor` implements a CSR-exchange "trust replacement":

1. **`SendStartTrustReplacementCommand()`** — processor returns a CSR containing
   its own keypair's public key, plus an `Href` for the update step:

   ```json
   Body.TrustReplacementSession = { "LOOBK": { "CSR": "<pem-csr>" }, "Href": "..." }
   ```

2. Designer signs the CSR **locally** with the SubSystem CA key:

   ```csharp
   var signed = X509CertificateBuilder.SignCertificate(
       subsystemCertificate.IssuerName,
       startTrustReplacementCommandResponse.Body.TrustReplacementSession.LOOBK.CSR,
       subSystemPrivateKey,
       CertificateType.SubLoobK);     // adds OID 1.3.6.1.4.1.40073.1.9 + KeyUsage=0xA6
   ```

3. **`SendCompleteTrustReplacementCommand(subsystemCert, signed, href)`** — posts
   both PEMs to the `Href`. Processor expects `204 No Content`.

4. **Legacy fallback**: if step 1 returns `400 Bad Request`, Designer falls back
   to `ClaimProcessorWithoutLOOBKey`, which just POSTs the CA to the fixed URL
   `/certpool/subsystem/certificate` with `RequestType=Create`, expecting
   `201 Created`. Older firmware.

After the call, the processor has the SubSystem CA installed as a trusted
client-auth anchor. From this point on, `openssl s_client --cert <subSystemCA-signed-leaf>`
works on port 8902.

### 3.3 Post-activation fixup

`ApplySecuitySettingsPipeline.ApplyLoobKeyAndMasterDeviceListSecuritySettingOnProcessor`
runs after the claim step and re-pushes the LOOB key if it's missing
(`ProcessorModelView.SetMissingLOOBKey`). The flow there is:

1. Open a LAP connection **without cert** first — `GetLoobKeyHref()` fetches
   `Body.Certificates[0].Href` from a `Read /certpool/subsystem/certificate` request
   (i.e. the processor tells the client where the LOOB update endpoint is).
2. Call `GenerateSubSystemCertificateV2()` (idempotent no-op if already present).
3. Open a second LAP connection **with `Processor.SubSystemCertificate`** as the
   client cert.
4. Call `SendLOOBKey(SubSystemCertificateV2, SubSystemPrivateKeyV2, loobKeyHref)`
   — same CSR-sign-then-install dance as `ClaimProcessor`, but targeted at the
   `loobKeyHref` URL from step 1.

Gated by feature flag `ApplyLoobkeyAndWhiteListDuringActivationAndTransfer` in
`FeatureFlagType` (Rollout.io).

In parallel, Designer pushes a **`MasterDeviceList`** — the per-project
whitelist of device serials — via `ProcessorModelView.UpdateMasterDeviceList`.
This is a defense against rogue devices, not a cert mechanism.

### 3.4 LAP request/response shape

All LAP traffic is newline-delimited JSON over TLS on :8083. `ProcessorRequestType`
mirrors the LEAP verb set: `Create`, `Read`, `Update`. Some observed URLs and status
codes:

| URL                                 | Verb     | Expected status | Purpose |
|-------------------------------------|----------|-----------------|---------|
| `/certpool/subsystem/certificate`   | `Read`   | 200 OK          | List existing subsystem certs; returns `Href` for LOOB update |
| `/certpool/subsystem/certificate`   | `Create` | 201 Created     | Legacy claim (raw CA PEM in body) |
| `/loobk/csr`                        | `Read`   | 200 OK          | Get processor CSR to sign |
| trust-replacement session `Href`    | `Update` | 204 No Content  | Install (CA + signed-leaf) pair |
| (processor-provided `loobKeyHref`)  | `Update` | 204 No Content  | Install LOOB key during post-activation repair |

Client tag literal seen in the code: `"123"` (no uniqueness enforced). Example:

```json
{
  "Header": {"RequestType": "Update", "ClientTag": "123", "Url": "..."},
  "Body": {
    "Certificate": {
      "SignedLOOBK": {"Certificate": "-----BEGIN CERTIFICATE----- ... -----END CERTIFICATE-----"},
      "Certificate":  "-----BEGIN CERTIFICATE----- ... CA PEM ... -----END CERTIFICATE-----"
    }
  }
}
```

(Yes, two fields named `Certificate` in the same object — that's in the
decompiled source. It's a quirk of how the anonymous-type projection runs; one
overwrites the other in the JSON serializer or the processor parses it
position-wise.)

## 4. Attacker Model

### 4.1 What an attacker needs

1. L2/L3 reachability to the processor's LAN IP on port 8083.
2. A copy of `residential_local_access.pfx` (trivial — unzip the Designer MSIX;
   empty PFX password).
3. The server cert chain returned by the processor must validate against
   `radioRa3-products` — but the attacker is the client, so this is a no-op for
   them (they don't need to validate anything; the processor validates the
   client cert).
4. A forged project SubSystem CA (generate one with `openssl ecparam` + `openssl
   req -x509`; Lutron's naming defaults aren't enforced on the processor side as
   far as we can see — only the client-auth EKU and signature are).

### 4.2 Preconditions on processor state

This is the unknown that separates "interesting" from "critical." The
trust-replacement endpoint gates behavior by processor state, but the Designer
code reveals the gate only indirectly:

- `SendStartTrustReplacementCommand()` returning `200 OK` → processor is willing
  to replace its current subsystem trust root. Happens on fresh units and
  probably after a factory-reset.
- `400 Bad Request` → processor is already claimed and refuses rotation on this
  endpoint. Designer's response is to fall back to the legacy
  `/certpool/subsystem/certificate Create` endpoint, which means **that endpoint
  evidently accepts a second CA PEM even when one is already installed**, or the
  fallback would be pointless. It's unclear whether the processor replaces, adds,
  or ignores — this needs live testing.

The pragmatic attack path:

- **Fresh / factory-reset unit**: attacker wins outright. Install a chosen CA,
  generate a matching client cert, talk IPL.
- **Already-activated unit**: the legacy-fallback POST to
  `/certpool/subsystem/certificate` is the candidate endpoint for takeover. Needs
  validation on hardware (see §6).
- **Physical access**: the factory-reset button on the RA3 makes the "already
  activated" case irrelevant for anyone with unescorted physical access.

### 4.3 What the attacker cannot do (cross-project isolation)

A stolen `cert_v2.pfx` from project A is **useless** against project B:

- Project B's processor has project B's CA installed as its sole client-auth
  anchor. A leaf cert signed by project A's CA will fail validation.
- There is no parent CA signing both. `GenerateCertificate()` makes the cert
  self-signed (issuer DN = subject DN, signed with its own private key).
- The `X509CertificateIssuerInfo` defaults produce the **same DN** for every
  project, but DNs are not secrets; what matters is the signing key, which is
  fresh per project.

So concerns about "exfiltrate one project's CA, own all RA3 processors
everywhere" are **unfounded**. The concern is specifically about the bootstrap
channel.

### 4.4 No pinning, no challenge-response

Observations that reduce the work factor for an attacker:

- Server cert validation is chain-to-product-root only (§2). No processor-level
  pinning during the LAP handshake — a substitution attack (e.g., a second RA3
  the attacker owns, physically interposed) would still authenticate as long as
  its server cert chains to `radioRa3-products`.
- The CSR/sign dance proves the processor owns the keypair it's handing you but
  does **not** challenge the client in any way. There's no nonce, no signed
  timestamp from the client, nothing binding the signed leaf to a specific
  client identity. Any caller with the shared PFX can drive the flow.
- Serial numbers for certs come from `new BigInteger(sizeInBits, new Random())`
  (not `SecureRandom`) — 10 bits wide in `X509CertificateIssuerInfo.SerialNumber`.
  Cryptographically meaningless (private keys are generated with `SecureRandom`)
  but confirms this code predates modern hygiene.

## 5. Replication in a Third-Party Tool

All the pieces to activate an RA3 processor from outside Designer are present
in the decompiled code and are well-scoped:

**Required cert material** (can be prepared once, reused):

```bash
# Extract from any Designer MSIX
unzip Lutron\ Designer\ *.msix QuantumResi/BinDirectory/CertificateStore/residential_local_access.pfx
openssl pkcs12 -in residential_local_access.pfx -out rla.pem -nodes -passin pass: -legacy

# Generate your own subsystem CA (anything ECDSA; Lutron uses P-384)
openssl ecparam -genkey -name secp384r1 -out my_ca_key.pem
openssl req -x509 -new -key my_ca_key.pem -sha384 -days 36500 -out my_ca_cert.pem \
    -subj "/CN=Lutron Project SubSystem Certificate Authority/O=Lutron Electronics Co., Inc./L=Coopersburg/ST=PA/C=US" \
    -addext "basicConstraints=critical,CA:TRUE,pathlen:5" \
    -addext "extendedKeyUsage=clientAuth,1.3.6.1.4.1.40073.1.10"
```

**Protocol loop** (per processor):

1. TLS connect to `<processor-ip>:8083` with `rla.pem` as the client cert.
   Verify processor's server cert against `radioRa3_products.crt`.
2. POST newline-delimited JSON: `{"Header":{"RequestType":"Read", "ClientTag":"1",
   "Url":"/certpool/subsystem/certificate"}}`. Read back `Body.Certificates[0]`.
3. If the processor is fresh, call `StartTrustReplacement` (exact URL not
   included in the Designer source but discoverable live; likely
   `/trustreplacement` or `/loobk` under `/certpool/subsystem/`). Parse the
   returned CSR.
4. Sign the CSR with `my_ca_key.pem` using `openssl x509 -req -sha384
   -extensions sub_loobk` where `sub_loobk` adds OID `1.3.6.1.4.1.40073.1.9` as
   critical and `keyUsage = 0xA6` (digitalSignature | keyEncipherment |
   keyCertSign | cRLSign — bits 7,6,5,2,1 of the 1-byte 0xA6; matches
   `KeyUsage(166)` in the decompile).
5. POST `{"Header":{"RequestType":"Update","Url":"<href>"}, "Body":{"Certificate":
   {"SignedLOOBK":{"Certificate":"<signed-leaf-pem>"}, "Certificate":"<ca-pem>"}}}`.
6. Tear down LAP, reconnect to :8902 with a fresh leaf signed by `my_ca_cert.pem`
   — you now have IPL write access.

This is straightforward to encode as `tools/ipl-activate.ts` alongside
`tools/ipl-cmd.ts`. Cost: a few hundred lines of TypeScript using Node's `tls`
and `node-forge`.

## 6. Open Items / Next Live Tests

Things worth confirming on hardware (in order of interest):

1. **Does the `/certpool/subsystem/certificate Create` endpoint replace the
   existing CA on an already-activated processor?** This is the legacy-fallback
   path, and its behavior decides whether the attack works beyond fresh units.
   Test: claim a lab processor with Designer, snapshot its trust state, then
   POST a second CA via the legacy endpoint, check if IPL:8902 accepts leaves
   signed by the second CA.
2. **What URL does `StartTrustReplacementCommand` actually hit?** The Designer
   code stores the URL in a `CommandType.StartTrustReplacement` constant we
   haven't decompiled. Pull it from `Lutron.Services.Core.LAPFramework.dll!CommandType`
   or capture a Designer run with `mitmproxy` in front of the processor.
3. **Does the processor enforce uniqueness on the CA DN?** The defaults in
   `X509CertificateIssuerInfo` produce the identical DN across projects; if the
   processor treats DN as a key, a "claim over the top" from a different
   Designer would be rejected even on a fresh unit. Unlikely but cheap to check.
4. **Does factory-reset actually wipe the CA?** Should, but verify. If not,
   the physical-access path requires something stronger (firmware hold/reset).
5. **BLE activation (`Lutron.Gulliver.BLEActivation.dll`)**: uses a separate
   transport ("Bleap" — BLE Association Protocol) with its own `BleapEncryption`
   class. This is the commissioning path for processors without Ethernet yet
   connected. Worth a separate look — if the BLE path accepts a CA without
   Ethernet-reachable `residential_local_access.pfx`, the attacker model
   changes.

## 7. What `SubCloudSub` / Cloud Escalation Is (for reference)

Orthogonal to local activation: Designer also has a cloud-mediated path for
transferring cluster ownership between Lutron accounts.

```csharp
// Lutron.Gulliver.LutronCloudApiIntegration.dll
public async Task<SystemClusterCloudEscalationResponseDTO>
    RequestCSRForCloudEscalation(SystemClusterCloudEscalationDTO dto, CancellationToken ct)
    => await restClient.PostAsync(
           ApiConstants.BackendCloudBaseUrl + "/api/v2/provisioning/cluster/ownership-transfer",
           dto, ...);
```

Related DTO fields:

- `CertificateChain`
- `CertificateSigningRequest`
- `PemFormattedCSR`
- `PemFormattedCertificate`
- `SignedByCertificate`
- `SubsystemCertificate`
- `CloudSystemClusterCertificate`

This uses `CertificateType.SubCloudSub` (OID `1.3.6.1.4.1.40073.1.10` + client
auth EKU). It does not touch the local LAP/IPL trust path — only Lutron-hosted
cloud features (remote access, myLutron app, etc.). Out of scope for 8902.

## 8. Summary Answers to the Open Questions

| Question | Answer |
|----------|--------|
| **Where does the per-project SubSystem CA come from?** | Generated locally by Designer (`X509CertificateBuilder.GenerateCertificateUsingECDSAlgorithm(384)`, BouncyCastle). Self-signed ECDSA P-384. No cloud, no HSM, no Lutron root. Stored in the project file and cached on the workstation as `cert_v2.pfx`. |
| **How does Designer push it?** | JSON-over-TLS on port **8083** (LAP). Authenticates with the shipped-and-public `residential_local_access.pfx`. Uses a CSR-exchange "trust replacement" to have the processor sign a leaf with its own key, then installs the signed leaf + CA PEM via an `Update` request. Legacy fallback posts the CA PEM directly to `/certpool/subsystem/certificate`. |
| **Is there an unauthenticated / shared-secret window?** | **Yes.** Port 8083 is authenticated by a secret that isn't one: `residential_local_access.pfx` is identical on every Designer install, empty-password, extractable from the MSIX. Combined with factory-reset or fresh units, this is an unauthenticated activation window in all but name. |
| **Is that window reachable on already-activated units?** | **Probably yes via the legacy fallback**, which posts a raw CA PEM to `/certpool/subsystem/certificate` with `RequestType=Create`. Needs live confirmation (§6.1). Definitely yes via factory-reset, which requires physical access. |
| **Cross-project PKI weakness?** | **No.** Each project's CA is a self-signed root with no shared parent. One project's compromise does not affect another. The weak shared material is `residential_local_access.pfx`, which is a bootstrap credential, not a trust anchor that survives activation. |
| **Can a third-party tool replicate activation?** | Yes, with a few hundred lines of TS/Python. All inputs (the shared PFX, the product root cert, the JSON request shapes, the signing algorithm + extensions) are recoverable from the Designer MSIX and decompiled source. |

## 9. Source Index

| Finding | DLL / Type / Method |
|---|---|
| Local CA generation | `Infrastructure.dll!X509CertificateBuilder.GenerateCertificateUsingECDSAlgorithm` |
| Default issuer DN | `Infrastructure.dll!X509CertificateIssuerInfo` (ctor) |
| CSR signing w/ SubLoobK EKU | `Infrastructure.dll!X509CertificateBuilder.SignCertificate` + `AddCustomExtensionsForCertificate` |
| Custom OID arc 40073.1.* | `Infrastructure.dll!X509CertificateBuilder` (const strings) |
| Bootstrap cert selection | `ModelViews.dll!LapConnectionHelper.LAPCertificateFileName` |
| Server cert chain validation | `ModelViews.dll!ProcessorModelView.VerifyLAPConnection` → `LapConnectionHelper.RootCertificateNamesByProduct` |
| LAP port default = 8083 | `Infrastructure.dll!GulliverConfiguration.DefaultLAPConnectionPort` |
| Claim flow (entry) | `QuantumResi.dll!ProcessorActivationStep.Execute` → `ModelViews.dll!ProcessorModelView.ClaimProcessor` → `ClaimProcessorOnLap` |
| Trust-replacement protocol | `LAPFramework.dll!ProcessorLapClient.ClaimProcessor` + `SendStartTrustReplacementCommand` + `SendCompleteTrustReplacementCommand` |
| Legacy fallback | `LAPFramework.dll!ProcessorLapClient.ClaimProcessorWithoutLOOBKey` + `LocalAssociationProtocolHelper.CreateClaimProcessorRequest` |
| Post-activation LOOB repair | `ModelViews.dll!ProcessorModelView.SetMissingLOOBKey` → `LAPFramework.dll!ProcessorLapClient.SendLOOBKey` / `GetLoobKeyHref` |
| Feature-flag gate | `QuantumResi.dll!ApplyLoobKeyAndMasterDeviceListSecuritySettingOnProcessor.CanExecute` (`FeatureFlagType.ApplyLoobkeyAndWhiteListDuringActivationAndTransfer`) |
| Multicast IP push fallback | `ModelViews.dll!ProcessorModelView.ClaimProcessorOnMulticast` (calls same `ClaimProcessorOnLap` afterward) |
| Cloud escalation (unrelated to IPL) | `LutronCloudApiIntegration.dll!SystemClusterAPIService.RequestCSRForCloudEscalation` → `/api/v2/provisioning/cluster/ownership-transfer` |

See [ipl.md](../protocols/ipl.md) for what to do with IPL:8902 once you have a
SubSystem CA — including `tools/ipl-cmd.ts`'s verified `GoToLevel` write path.
