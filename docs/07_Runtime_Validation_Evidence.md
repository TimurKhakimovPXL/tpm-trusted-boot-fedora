<!--
PUBLIC RELEASE VERSION - sanitized for publication.
Machine-id and filesystem/LUKS UUIDs are format-valid EXAMPLE values, not the
original lab identifiers. PCR values, policy-key digests and file hashes are
genuine measurements from the validated build and are public values by design.
-->

# Appendix 07: Runtime Validation Evidence

> Captured on the test rig (VM 500, `tboot-lab`) after a clean reboot on 2026-05-28.
> The blocks are the actual console output; long key and blob dumps are shortened
> with `[...]`. This appendix is a frozen snapshot of the state at capture: a later
> full `dnf update` including a kernel version upgrade (2026-06-19) advanced PCR 11
> beyond the value shown here without breaking TPM2 unlock — see the README.

## 1. Validated state at capture

### 1.1 Boot identity (`bootctl status`, abridged)

```text
System:
      Firmware: UEFI 2.70 (Proxmox distribution of EDK II 1.00)
   Secure Boot: enabled (user)
  TPM2 Support: yes
  Measured UKI: yes
  Boot into FW: supported

Current Boot Loader:
       Product: systemd-boot 258.7-1.fc43
 Current Entry: 0123456789abcdef0123456789abcdef-6.19.14-200.fc43.x86_64.efi
 Default Entry: 0123456789abcdef0123456789abcdef-6.19.14-200.fc43.x86_64.efi

Current Stub:
       Product: systemd-stub 258.7-1.fc43

Default Boot Loader Entry:
         type: Boot Loader Specification Type #2 (UKI, .efi)
           id: 0123456789abcdef0123456789abcdef-6.19.14-200.fc43.x86_64.efi
       source: /boot/efi/EFI/Linux/0123456789abcdef0123456789abcdef-6.19.14-200.fc43.x86_64.efi
      options: root=UUID=aaaaaaaa-bbbb-cccc-dddd-eeeeeeeeeeee ro rd.luks.uuid=luks-11111111-2222-3333-4444-555555555555 quiet
```

Secure Boot is in user mode under the custom keys, the TPM measures the UKI
(`Measured UKI: yes`), and the active default entry is the hook-generated,
machine-id-prefixed UKI as Type #2 (UKI, `.efi`).

### 1.2 Runtime PCRs against the stored prediction

```text
runtime  PCR 7 : BF8878BD8B70654924BD39CA71D770F556F03558AC200BAD46A9CEB5A2B596BD
runtime  PCR 11: 28E66CE224C9BB0711059470D77103A5AB5A5C1674C03D1BA1563C2FEE52C90F
expected PCR 11: 28E66CE224C9BB0711059470D77103A5AB5A5C1674C03D1BA1563C2FEE52C90F
```

Runtime PCR 11 matches the stored prediction exactly. PCR 7 matches the
Secure Boot state baseline.

### 1.3 LUKS keyslots and TPM2 token (`luksDump`, abridged)

```text
$ systemd-cryptenroll /dev/sda3
SLOT TYPE
   0 password
   1 tpm2

$ cryptsetup luksDump /dev/sda3
UUID:           11111111-2222-3333-4444-555555555555
Keyslots:
  0: luks2   PBKDF: argon2id        (passphrase, recovery anchor)
  1: luks2   PBKDF: pbkdf2 / sha512 (TPM2)
Tokens:
  0: systemd-tpm2
        tpm2-hash-pcrs:   7
        tpm2-pcr-bank:    sha256
        tpm2-pubkey:      [PEM key, abridged]
        tpm2-pubkey-pcrs: 11
        tpm2-primary-alg: ecc
        tpm2-pin:         false
        tpm2-srk:         true
        tpm2-policy-hash: [abridged]
        tpm2-blob:        [abridged]
        Keyslot:    1
```

Keyslot 0 is the passphrase recovery anchor (argon2id). Keyslot 1 is bound via
TPM2 token 0 to the split policy: PCR 7 static (`tpm2-hash-pcrs: 7`) and PCR 11
via the signed public key (`tpm2-pubkey-pcrs: 11`), in the sha256 bank, with the
SRK as primary key. Note that `tpm2-primary-alg: ecc` refers to the storage root
key; the *policy* key used for the PCR 11 signature is RSA.

### 1.4 Evidence that the TPM (not the passphrase) unlocked

```text
May 28 14:39:26 tboot ... Starting systemd-cryptsetup@luks-...-555555555555.service - Cryptography Setup ...
May 28 14:39:27 tboot ... Finished systemd-cryptsetup@luks-...-555555555555.service - Cryptography Setup ...
May 28 14:39:26 tboot ... systemd 258.7-1.fc43 running in system mode (... +TPM2 ... +LIBCRYPTSETUP +LIBCRYPTSETUP_PLUGINS ...)
```

The cryptsetup service started and completed early in boot with no passphrase
prompt; the operator typed nothing. This systemd version does not log an explicit
"unlocked with TPM2" line, so the evidence is threefold: the service completed
cleanly, systemd is built with TPM2 support (`+TPM2`), and runtime PCR 11 matched
the sealed prediction (§1.2). Together these show the TPM released the key.

### 1.5 Integrity of the booted UKI

```text
1187c78ce2435e0a5c27ed8f35e0fd653da3580ac7a2597b03a30bbfd5deb843  /boot/efi/EFI/Linux/0123456789abcdef0123456789abcdef-6.19.14-200.fc43.x86_64.efi
```

## 2. PCR 11 evolution across milestones

The PCR 11 value changed with every substantive change to the boot path. Each
time, the chain predicted and re-sealed the new value in advance, and each time
the runtime value after reboot matched the prediction exactly. That is the core
proof that forward sealing and the automation work: a change does not break
unlocking, because the new state is signed ahead of time.

```text
Phase 1 (Secure Boot + first UKI):                            42F096D23D0227A780C801C3670A7BF7E57C1583D9360E3D79B5D2AF544FE3CC
Module 3 (LUKS TPM2 enrollment):                              A03EB49C9335D721758AE8AA9C34DA87664C5CAF9FAD7C314C68C038B16435DD
State at capture (after systemd 258.7 + kernel/dracut updates): 28E66CE224C9BB0711059470D77103A5AB5A5C1674C03D1BA1563C2FEE52C90F
PCR 7 (baseline, unchanged):                                  BF8878BD8B70654924BD39CA71D770F556F03558AC200BAD46A9CEB5A2B596BD
```

Finding E.3 (`06F_Diagnostic_Findings_Catalog.md`): the UKI's sha256 changes even
when the measured content stays the same, due to the random RSA-PSS salt in
`.pcrsig` and a fresh `sbsign` timestamp. PCR 11 changes only when the measured
content itself changes, as with the transition to systemd 258.7.

## 3. Blocks B.2.2 and B.5: automation chains under a real transaction

Block B.2.2 exercised the first chain in a real transaction: the helper processed
two kernels (`6.17.1-300.fc43.x86_64` secondary and `6.19.14-200.fc43.x86_64`
booted) in 83 seconds and rebuilt, re-signed and re-sealed the UKIs via the
validated hook. After the reboot, runtime PCR 11 matched the prediction and LUKS
unlocked via the TPM without a passphrase prompt.

Block B.5 closed the second chain (systemd-boot loader) with the same staged gate
method, capped by a real reboot in which runtime PCR 11 matched the prediction
byte for byte. Anchor: `module5-b5-validated`. Citable evidence from B.2.4:
Gate 3 (reference run), Gate 4 (independent prediction, byte match), Gate 6A
(operator-confirmed clean reboot) and Gate 6B (runtime PCR 11 byte match).
