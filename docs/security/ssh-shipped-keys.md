# Pre-installed SSH authorized_keys in shipped firmware

> Finding: the RA3 phoenix firmware (`v26.01.13f000`) ships with two
> SSH public keys pre-installed in `authorized_keys` files. These are
> present on every deployed processor, creating a persistent SSH access
> path controlled by whoever holds the matching private keys.

## Accounts enabled for SSH

From `/etc/openssh/sshd_config`:

```
PasswordAuthentication no
AllowUsers root support ssh-credentials-transfer u_db-transfer-mngmt u_fwu u_dfp
```

Six accounts total. Four (`u_db-transfer-mngmt`, `u_fwu`, `u_dfp`,
`ssh-credentials-transfer`) are SFTP-chrooted via `ForceCommand internal-sftp`.
**`root` and `support` have full shell access** (support is SFTP-chrooted in
practice, but the config allows shell).

Key-only auth is enforced (no passwords).

## Pre-installed keys found in firmware image

### `/home/support/.ssh/authorized_keys`

```
ssh-rsa AAAAB3NzaC1yc2EAAAADAQABAAABAQDX9bEO...f9 abhat@PC0008690
```

Comment suggests a Lutron engineer workstation (`abhat@PC0008690`). The
`support` account is chrooted SFTP (for collecting diagnostic bundles),
but its key is present on every shipped RA3 processor.

### `/home/ssh-credentials-transfer/.ssh/authorized_keys`

```
ssh-rsa AAAAB3NzaC1yc2EAAAADAQABAAABAQDps8jF...t ssh-credentials-transfer
```

Used by internal processes to deliver SSH credentials (outbound direction
likely, to pre-provision new keys onto the processor). The service/user
holding the private key can shell into every RA3 as this account.

## Why this matters

1. **Single engineer workstation compromise → access to every RA3 unit
   globally.** Whoever has the `abhat` private key can SFTP in as `support`
   on any RA3 without further authentication. There's no per-device key
   uniqueness for these shipped keys.
2. **Key rotation requires a firmware update.** The public keys are baked
   into the rootfs tarball. Revoking a compromised key means pushing a new
   firmware release to all deployed units.
3. **Combined with other vulns (LEAP → root via sqlite_helper eval from
   [leap-sqlite-eval.md](leap-sqlite-eval.md)), an attacker with LEAP Admin
   can drop their own pubkey into `/root/.ssh/authorized_keys`** and gain
   durable shell access that survives LEAP re-pairing. The PoC in
   `phoenix-root.md` does exactly this.

## What the SFTP accounts are used for

Based on the chroot directories in `sshd_config` and the account names:

| User | Chroot | Purpose |
|---|---|---|
| `support` | `/var/sftp/support/tmp` | Collect diagnostic bundles / logs |
| `u_db-transfer-mngmt` | `/var/sftp/u_db-transfer-mngmt` | **Database transfer — this is where Designer uploads a new DB** |
| `ssh-credentials-transfer` | `/var/ssh-credentials-transfer` | Push credentials onto the processor |
| `u_fwu` | `/var/firmware/u_fwu` | Firmware update staging |
| `u_dfp` | `/var/firmware/u_dfp` | Device firmware package staging |

This answers an earlier open question — **Designer uploads the database
via SFTP as `u_db-transfer-mngmt`**. The `DatabaseTransferSession` CREATE
body in LEAP presumably carries the SFTP destination that the processor
uses as the CLIENT for the push. For DB extraction (processor → us), we'd
host an SFTP server and supply credentials in the CREATE body.

## Cross-reference

- [leap-sqlite-eval.md](leap-sqlite-eval.md) — once root via LEAP, drop a
  new pubkey to persist.
- [phoenix-root.md](phoenix-root.md) — original NTP-URL root RCE, PoC
  injects a pubkey as persistence.
- [leap-server.md](../firmware-re/leap-server.md) — Database transfer
  endpoint inventory.
