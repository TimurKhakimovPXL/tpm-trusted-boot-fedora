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

# Module 5 — Update Workflows and Signing Key Storage

## Purpose of This Module

Modules 1–4 define the architecture and its governance model. This module addresses the operational question that arises the moment someone runs `sudo dnf upgrade` on a production machine:

> How does a new UKI get signed so the system does not lock itself out on the next reboot — and where should the private key live to make that safe?

Two distinct workflows exist depending on how the OS is managed. Both are covered here.

> [!NOTE]
> **The Problem in One Sentence**
> Every UKI rebuild produces a new PCR 11. Without a valid `.pcrsig` for that new value, the TPM refuses to release the disk key on the next boot.

---

## 1. The Gap That Must Be Closed

When a new UKI is produced — by a kernel update, an initramfs rebuild after a driver update, or a cmdline change — PCR 11 changes. Forward sealing requires a `.pcrsig` that authorises that new PCR value before the next reboot occurs.

In a CI/CD pipeline this gap does not exist: signing is part of the build. In a manually managed system, something must fill that gap locally and automatically.

The solution is a **local signing hook** that runs `systemd-measure` every time `kernel-install` produces a new UKI. The private key used by that hook is what determines the security level of the entire arrangement.

---

## 2. Where the Private Key Can Live

Storing the private key as a plaintext file on the machine is the weakest option and should be avoided in sensitive environments. The following alternatives are ordered by increasing security strength.

### 2.1 Plaintext on Disk (Baseline — Avoid in Sensitive Environments)

```
/etc/systemd/tpm2-pcr-private-key.pem   (chmod 600, root only)
```

Simple to set up. An attacker with root access or physical disk access can extract the key and sign arbitrary boot states. Acceptable only for personal workstations with no compliance requirements.

---

### 2.2 TPM-Sealed Private Key

The private key is stored on disk but encrypted by the TPM — bound to PCR 7 (Secure Boot policy). The key cannot be decrypted without the TPM that sealed it, so offline disk extraction yields nothing useful.

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
> Offline disk theft — the encrypted key file is useless without the TPM. It does not protect against a fully compromised running OS, since the OS can request the unseal operation from the TPM while the system is running.

---

### 2.3 Hardware Security Module (HSM) or Smart Card

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

**Operational implication:** The HSM must be physically present when `dnf upgrade` runs. Without it, the signing step fails. For sensitive environments, this is a deliberate control — a new boot state cannot be authorised without physical admin presence.

---

### 2.4 Remote Signing Service

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

This is the recommended model for enterprise fleets. It is also the architecture that a CI/CD pipeline implements — the pipeline is the signing service, triggered by a merge rather than an upgrade event.

---

### 2.5 Comparison

| Storage Model | Key Leaves Machine | Offline Disk Attack | Compromised OS | Suitable For |
|--------------|-------------------|--------------------:|----------------|--------------|
| Plaintext on disk | N/A — it is on disk | ❌ Exposed | ❌ Exposed | Personal workstations only |
| TPM-sealed on disk | No | ✅ Protected | ❌ Exposed | Single machines, moderate sensitivity |
| HSM / Smart card | Never | ✅ Protected | ✅ Protected | High-sensitivity, requires physical presence |
| Remote signing service | Never | ✅ Protected | ✅ Protected | Enterprise fleets, full audit trail |

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

## 4. Workflow 2 — Image-Based Updates (bootc / OSTree)

### 4.1 How Image-Based Updates Work

In an image-based model the OS is never mutated in place. The entire OS — kernel, drivers, initramfs, and UKI — is built as an atomic image in CI. `bootc pull` stages the new image alongside the current one. On reboot, the system activates the new image. The old image is retained as the rollback target.

```
CI pipeline builds new OS image
  → kernel + NVIDIA driver + initramfs bundled into image
  → UKI extracted and signed during image build
  → .pcrsig embedded in image or shipped alongside it
  → bootc pull stages new image on the machine
  → Reboot activates new deployment
    → New UKI boots → new PCR 11 → matches .pcrsig → disk unlocks
```

There is no gap between update and signing because they are the same atomic operation. The machine never runs an unsigned UKI.

### 4.2 Rollback

Because the previous image is retained on the ESP, its `.pcrsig` also remains valid. Rolling back to the previous image produces the previous PCR 11, which still has a matching signature. No re-enrollment is needed.

```
Current image  → .pcrsig A → valid → unlocks
Rollback image → .pcrsig B → valid → also unlocks
```

Rollback protection (preventing revert to a vulnerable image) is enforced by removing the old `.pcrsig`. Without a valid signature, the old image cannot unlock the disk even if it boots. See [Module 4 — Rollback and Version Control](04_Governance_Recovery_Lifecycle.md).

---

## 5. Workflow Comparison

| Aspect | Manual (dnf upgrade) | Image-Based (bootc) |
|--------|---------------------|---------------------|
| Who signs | Local hook on the machine | CI/CD pipeline at build time |
| Private key location | On machine (various options) | In CI secrets |
| Kernel update automated | ✅ Yes, via kernel-install hook | ✅ Yes, part of image build |
| Driver update automated | ⚠️ Requires DNF plugin too | ✅ Yes, always included in image |
| Rollback signing | Manual — retain old .pcrsig | Built in via bootc |
| Audit trail | Local logs only | Full CI pipeline audit log |
| Suitable for | Workstations, single machines | Servers, fleets, immutable OS |

---

## 6. Recommended Architecture for Sensitive Environments

Sensitive environments that require manual updates but cannot accept a pipeline have one additional constraint: **a new boot state should not be authorisable without deliberate physical admin action**.

An HSM achieves this naturally:

```
┌──────────────────────────────────────────────────────┐
│  Sensitive Environment — Manual Update Architecture  │
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
│    → dnf upgrade runs                               │
│    → hook fires, signing fails (no HSM)              │
│    → .pcrsig not updated                             │
│    → system falls back to previous UKI               │
│    → or halts if no fallback entry exists            │
└──────────────────────────────────────────────────────┘
```

The requirement for physical presence is a feature in sensitive environments: no new boot state can be authorised remotely or without the administrator being physically present with the signing token. This satisfies audit requirements and prevents remote compromise of the signing process.

---

## 7. The One Rule That Applies to All Workflows

> [!WARNING]
> **Recovery Keyslot Is Mandatory**
> If the signing hook fails, the HSM is lost, the signing service is unreachable, or anything breaks the `.pcrsig` chain — the recovery keyslot is the only way back in.
>
> Enroll a high-entropy recovery passphrase at provisioning time and store it in an offline vault or escrow system. Without it, any failure in the signing workflow results in a permanently locked disk.

```bash
# Add recovery passphrase at provisioning time
systemd-cryptenroll /dev/nvme0n1p3 --password
```

See [Module 4 — Disaster Recovery](04_Governance_Recovery_Lifecycle.md) for full recovery procedures.

---

## Related Notes

- [Introduction — Architecture Overview](00_Introduction.md)
- [Module 2 — Forward Sealing](02_Forward_Sealing.md)
- [Module 3 — Disk Encryption and Policy Enforcement](03_Disk_Encryption_and_Policy_Enforcement.md)
- [Module 4 — Governance, Recovery, and Lifecycle](04_Governance_Recovery_Lifecycle.md)
