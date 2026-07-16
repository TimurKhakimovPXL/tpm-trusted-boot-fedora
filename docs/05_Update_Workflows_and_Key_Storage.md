---
tags:
  - trusted-boot
  - forward-sealing
  - key-management
  - update-workflow
  - automation
module: 5
status: stable
---

# Module 5: Update Workflows and Signing Key Storage

## Purpose of This Module

Modules 1–4 define the architecture and its governance model. This module addresses the operational question that arises the moment someone runs `sudo dnf upgrade` on a production machine:

> How does a new UKI get signed before the next reboot, and where should the
> private key live?

Two distinct workflows exist depending on how the OS is managed. Both are covered here.

> [!NOTE]
> A rebuild that changes measured UKI content needs a valid policy signature
> for the resulting PCR 11 value before the next boot.

---

## 1. The Gap That Must Be Closed

When a kernel, initramfs, embedded command line or another measured section
changes, the resulting PCR 11 value changes. A rebuild of identical measured
content can produce a different UKI file hash while leaving PCR 11 unchanged.
Forward sealing requires the embedded `.pcrsig` to authorise the calculated PCR
11 state before the next reboot.

In a CI/CD pipeline this gap does not exist: signing is part of the build. In a manually managed system, something must fill that gap locally and automatically.

The validated solution is a local `kernel-install` hook that rebuilds the UKI
with native `ukify` PCR-policy and Secure Boot signing options. A separate
predictor uses `systemd-measure calculate` as an independent PCR 11 check. The
location and release policy of both private keys define the compromise boundary
of this arrangement.

---

## 2. Where the Private Key Can Live

Storing the policy private key as a plaintext file on the machine is the
weakest option and should be avoided in sensitive environments. The following
models have different operational and compromise boundaries; they are not a
simple ranking.

### 2.1 Plaintext on Disk (Baseline: Avoid in Sensitive Environments)

```
/etc/systemd/tpm2-pcr-private-key.pem   (chmod 600, root only)
```

Simple to set up. An attacker with root access or physical disk access can extract the key and sign arbitrary boot states. Acceptable only for personal workstations with no compliance requirements.

---

### 2.2 TPM-Sealed Private Key

The private key is stored on disk but encrypted by the TPM: bound to PCR 7 (Secure Boot policy). The key cannot be decrypted without the TPM that sealed it, so offline disk extraction yields nothing useful.

**Setup (once, at provisioning time):**

```bash
# Seal the private key into a TPM-encrypted credential
systemd-creds encrypt \
  --tpm2-device=auto \
  --tpm2-pcrs=7 \
  --name=tpm2-pcr-signing-key \
  /etc/systemd/tpm2-pcr-private-key.pem \
  /etc/systemd/tpm2-pcr-private-key.pem.enc

# Remove the plaintext key
shred -u /etc/systemd/tpm2-pcr-private-key.pem
```

**Usage in the signing hook:**

```bash
# Unseal at signing time
systemd-creds decrypt \
  --tpm2-device=auto \
  --name=tpm2-pcr-signing-key \
  /etc/systemd/tpm2-pcr-private-key.pem.enc \
  /tmp/tpm2-pcr-private-key-unsealed.pem

# Sign
systemd-measure sign \
  --private-key=/tmp/tpm2-pcr-private-key-unsealed.pem \
  ...

# Destroy plaintext immediately after use
shred -u /tmp/tpm2-pcr-private-key-unsealed.pem
```

> [!NOTE]
> **What This Protects Against**
> Offline disk theft: the encrypted key file is useless without the TPM. It does not protect against a fully compromised running OS, since the OS can request the unseal operation from the TPM while the system is running.

The validated hook also reads the Secure Boot private key from
`/etc/uefi-keys/db.key` as a root-only plaintext file. TPM-sealing the policy
key does not protect that separate key. A root compromise while the host is in
an approved state can therefore reach both local signing paths.

---

### 2.3 Hardware Security Module (HSM) or Smart Card

This is an alternative design, not an implementation shipped by this
repository. The validated hook decrypts a TPM-sealed PEM key and does not
contain PKCS#11 support.

The private key is generated inside the HSM and never leaves it. The signing operation is performed by the hardware itself; the OS only sends data to be signed and receives the signature back.

Suitable hardware: YubiKey 5, Nitrokey HSM, or enterprise HSMs (Thales Luna, AWS CloudHSM).

**Signing via PKCS#11:**

```bash
systemd-measure sign \
  --linux=/lib/modules/${KERNEL_VERSION}/vmlinuz \
  --initrd=/boot/initramfs-${KERNEL_VERSION}.img \
  --osrel=/etc/os-release \
  --cmdline=/etc/kernel/cmdline \
  --pcrpkey=/etc/systemd/tpm2-pcr-public-key.pem \
  --private-key-source=provider:pkcs11 \
  --private-key="pkcs11:token=MyHSM;object=tpm2-pcr-key" \
  --output=/usr/lib/systemd/tpm2-pcr-signature.json
```

> [!IMPORTANT]
> **Version Requirement**
> `--private-key-source=` is available from systemd v256 onwards. Verify your distribution ships a sufficient version before relying on PKCS#11 signing in the hook.

An HSM does not automatically protect signing from a compromised operating
system. A logged-in root process may still invoke the token unless the token
requires an independent PIN, touch, quorum or approval for each operation.

---

### 2.4 Remote Signing Service

This is also an architectural option rather than part of the validated local
implementation.

The private key lives on a centralised signing server. The machine sends only the predicted PCR hash; the service returns the signature. The key never touches the endpoint machine.

```
Machine                              Signing Service
  │                                        │
  ├── systemd-measure calculate ─────────► │
  │   (sends PCR hash, not the key)        │
  │                                        ├── validates request
  │                                        ├── enforces policy
  │                                        ├── signs with private key
  │                                        ├── logs signing event
  │ ◄────────────────── .pcrsig ───────────┤
  │                                        │
  ├── writes .pcrsig locally               │
  └── reboot proceeds normally             │
```

The service can enforce additional controls before signing:

- Verify the requesting machine's identity (mTLS client certificate)
- Require MFA from the administrator performing the update
- Reject requests outside maintenance windows
- Log every signing event centrally with full audit trail

A remote service provides a separate security boundary only when it validates
the requested artifact or measurement against independent policy. A service
that signs any PCR value submitted by an authenticated endpoint remains
vulnerable to a compromised endpoint.

---

### 2.5 Comparison

| Storage Model | Key Leaves Machine | Offline Disk Attack | Compromised OS | Suitable For |
|--------------|-------------------|--------------------:|----------------|--------------|
| Plaintext on disk | N/A: it is on disk | ❌ Exposed | ❌ Exposed | Personal workstations only |
| TPM-sealed on disk | No | ✅ Protected | ❌ Exposed | Single machines, moderate sensitivity |
| HSM / Smart card | Never | ✅ Protected | Conditional on token policy | Deliberately authorised local signing |
| Remote signing service | Never | ✅ Protected | Conditional on server-side validation | Centrally governed fleets |

---

## 3. Workflow 1: Validated Package-Based Updates with DNF5

The lab uses the byte-pinned hook, action rules, deciders and helpers shipped in
this repository. These replace the earlier generic hook and fixed package watch
list.

### 3.1 Components

| Component | Role |
|---|---|
| `hooks/80-tpm2-sign.install` | Rebuilds and signs UKIs |
| `dnf-actions/50-tboot-posttrans.actions` | Runs the boot-input decider after every completed host transaction |
| `dnf-actions/tboot-dnf-posttrans` | Computes the boot-input manifest and decides whether drift exists |
| `dnf-actions/tboot-dnf-helper` | Rebuilds initramfs images and invokes `kernel-install` |
| `dnf-actions/tboot-predict-pcr11` | Independently calculates and stores expected PCR 11 |
| `dnf-actions/60-tboot-sbloader.actions` | Watches inbound `systemd-boot-unsigned` transactions |
| `dnf-actions/tboot-sbloader-posttrans` | Decides whether the signed loader state is converged |
| `dnf-actions/tboot-sbloader-helper` | Signs and safely propagates systemd-boot to the ESP |

The action files are consumed by `libdnf5-plugin-actions`. Deciders determine
whether work is required; helpers perform the privileged mutations.

### 3.2 UKI Signing Hook

`80-tpm2-sign.install` runs during `kernel-install add` for the UKI layout. It
does not modify an already signed UKI with `objcopy`, and it does not rely on
`ukify --join-pcrsig`. Instead, it rebuilds the staged image in one native
`ukify build` operation using:

- the kernel, initramfs, command line and operating-system metadata;
- the PCR public key and TPM-unsealed RSA private policy key;
- `--phases=enter-initrd` and the SHA-256 PCR bank; and
- the Secure Boot `db` key and certificate.

The hook checks the required sections, embedded PCR policy, Secure Boot
signature, signer count and PE/COFF section layout before replacing the staged
UKI. Only then can `90-uki-copy.install` copy the file to the ESP.

### 3.3 Boot-Input Drift Detection

The broad action rule runs after every completed host transaction because
package names alone do not reliably identify changes to measured boot inputs.

The decider computes a deterministic manifest covering fourteen classes of
boot-relevant input, including installed kernels and modules, initramfs and
dracut configuration, firmware, early-boot systemd and udev inputs, storage and
cryptsetup tooling, the kernel command line, and the public trust artifacts.
It compares that manifest with a protected baseline:

- Equal manifest: no trusted-boot rebuild is needed.
- Drift: record the unsafe state and invoke the helper.
- Missing baseline in normal mode: fail closed instead of silently creating a
  new trust baseline.

### 3.4 Drift-Rebuild Flow

```text
DNF transaction completes
  → libdnf5 action invokes tboot-dnf-posttrans
    → compute and compare boot-input manifest
      → unchanged: exit without rebuilding
      → drift:
          write UNSAFE-TO-REBOOT sentinel
          invoke tboot-dnf-helper
            → rebuild initramfs for each installed kernel
            → invoke kernel-install add
              → 80-tpm2-sign.install rebuilds and signs each UKI
            → independently predict and store expected PCR 11
          advance the baseline atomically
          clear the sentinel
```

The helper does not sign UKIs itself. It runs the steps that trigger the signing
hook, then independently refreshes the expected PCR 11 value. The decider
advances the baseline only after the helper succeeds.

### 3.5 Failure Semantics

The DNF action executes in `post_transaction`. At that point the RPM database
and package transaction have already committed. A later signing failure cannot
roll those package changes back.

A failed post-transaction action has these results:

- DNF reports the post-transaction failure;
- the baseline is not advanced;
- the `UNSAFE-TO-REBOOT` sentinel remains present; and
- the machine must not reboot until the boot artifacts are repaired.

The sentinel warns the operator but does not block a reboot. This workflow also
does not roll back committed RPM changes.

### 3.6 systemd-boot Loader Updates

The UKI chain is not sufficient on its own because the
`systemd-boot-unsigned` package can replace the bootloader source binary. A
second action chain handles that package separately:

```text
systemd-boot-unsigned inbound transaction
  → 60-tboot-sbloader.actions
    → tboot-sbloader-posttrans compares the loader manifest
      → drift or non-converged shape
        → tboot-sbloader-helper
          → sign the source binary
          → test propagation on a throwaway loopback ESP
          → require both scratch targets to match the signed source
          → propagate to the real ESP
          → verify convergence
        → advance the baseline and clear the sentinel
```

The loopback-ESP checkpoint is mandatory. It proves that
`bootctl --no-variables install` selects and copies the `.efi.signed` file to
both loader targets before the real ESP is touched.

### 3.7 Operational Invariants

- Only `80-tpm2-sign.install` signs UKIs.
- Deciders decide; helpers mutate.
- A missing production baseline is an error, never an implicit prime.
- Mutation paths mark the machine unsafe to reboot before changing boot state.
- Baselines advance only after successful convergence.
- The passphrase recovery keyslot remains available throughout validation.
- PCR 11 prediction is refreshed without re-enrolling the LUKS keyslot.

The full installation and validation sequence is in
`06B_Golden_Path_Rebuild_Runbook.md`. Failure modes and rejected designs are in
`06F_Diagnostic_Findings_Catalog.md`.

---

## 4. Workflow 2: Image-Based Updates (bootc / OSTree)

### 4.1 How Image-Based Updates Work

In an image-based model the OS is never mutated in place. The entire OS: kernel, drivers, initramfs, and UKI, is built as an atomic image in CI. `bootc pull` stages the new image alongside the current one. On reboot, the system activates the new image. The old image is retained as the rollback target.

```
CI pipeline builds new OS image
  → kernel + NVIDIA driver + initramfs bundled into image
  → UKI extracted and signed during image build
  → .pcrsig and .pcrpkey embedded in the UKI
  → bootc pull stages new image on the machine
  → Reboot activates new deployment
    → New UKI boots → new PCR 11 → matches .pcrsig → disk unlocks
```

There is no gap between update and signing because they are the same atomic operation. The machine never runs an unsigned UKI.

### 4.2 Rollback

Because the previous signed UKI is retained on the ESP, its embedded policy
remains valid. Rolling back to that UKI reproduces its authorised PCR 11 state.
No re-enrollment is needed while its policy key remains enrolled.

```
Current image  → .pcrsig A → valid → unlocks
Rollback image → .pcrsig B → valid → also unlocks
```

To revoke the old deployment, remove its UKI from every bootable location and
verify that no boot-loader or firmware entry still selects it. Removing a
standalone signature file does not revoke a signature embedded in a UKI. See
[Module 4: Rollback and Version Control](04_Governance_Recovery_Lifecycle.md).

---

## 5. Workflow Comparison

| Aspect | Manual (dnf upgrade) | Image-Based (bootc) |
|--------|---------------------|---------------------|
| Who signs | Local hook on the machine | CI/CD pipeline at build time |
| Private key location | On machine (various options) | In CI secrets |
| Kernel update automated | ✅ Yes, via kernel-install hook | ✅ Yes, part of image build |
| Driver update automated | ⚠️ Requires DNF plugin too | ✅ Yes, always included in image |
| Rollback signing | Retain or remove complete signed UKIs | Retain or remove complete signed deployments |
| Audit trail | Local logs only | Full CI pipeline audit log |
| Suitable for | Workstations, single machines | Servers, fleets, immutable OS |

---

## 6. External Signer Design Considerations

An HSM or remote service can add an approval boundary, but the shipped hook
does not implement either integration. A future design would need to define the
signer interface, authentication, request validation, timeout behavior and
recovery path. For an HSM, it must also define whether every signature requires
touch, PIN or another action that a compromised host cannot silently satisfy.

The intended flow would be:

```
┌──────────────────────────────────────────────────────┐
│  Sensitive Environment: Manual Update Architecture  │
│                                                      │
│  Private key:   HSM (YubiKey / Nitrokey)             │
│  Signing:       kernel-install hook + DNF plugin     │
│  Trigger:       HSM must be physically present       │
│  Recovery:      Passphrase in offline vault          │
│                                                      │
│  Update flow:                                        │
│    Admin inserts HSM                                 │
│    → sudo dnf upgrade                               │
│    → hook fires, HSM signs new PCR prediction        │
│    → HSM removed                                     │
│    → reboot → disk unlocks                           │
│                                                      │
│  Without HSM present:                                │
│    → dnf upgrade commits package changes             │
│    → post-transaction signing fails                  │
│    → unsafe-reboot sentinel remains                  │
│    → operator repairs the chain before reboot        │
└──────────────────────────────────────────────────────┘
```

Physical presence is a meaningful control only if the token enforces it for
each signing operation. Possession of an HSM alone does not establish that
property.

---

## 7. Recovery Requirement for Every Workflow

> [!WARNING]
> **Recovery Keyslot Is Mandatory**
> If the signing hook fails, the HSM is lost, the signing service is unreachable, or anything breaks the `.pcrsig` chain: the recovery keyslot is the only way back in.
>
> Enroll a high-entropy recovery passphrase at provisioning time and store it in an offline vault or escrow system. Without it, any failure in the signing workflow results in a permanently locked disk.

```bash
# Add recovery passphrase at provisioning time
systemd-cryptenroll /dev/nvme0n1p3 --password
```

See [Module 4: Disaster Recovery](04_Governance_Recovery_Lifecycle.md) for full recovery procedures.

---

## Related Notes

- [Introduction: Architecture Overview](00_Introduction.md)
- [Module 2: Forward Sealing](02_Forward_Sealing.md)
- [Module 3: Disk Encryption and Policy Enforcement](03_Disk_Encryption_and_Policy_Enforcement.md)
- [Module 4: Governance, Recovery, and Lifecycle](04_Governance_Recovery_Lifecycle.md)
