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

> [!summary] The Problem in One Sentence
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

> [!note] What This Protects Against
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

> [!important] Version Requirement
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

## 3. Workflow 1 — Manual Package-Based Updates (dnf upgrade)

### 3.1 How kernel-install Works

`kernel-install` is the systemd tool that manages kernel and UKI installation. It runs automatically when a kernel RPM is installed or removed. It reads `/etc/kernel/install.conf` for layout configuration and executes all executable scripts found in `/etc/kernel/install.d/` in lexicographic order.

When `layout=uki` is set, `kernel-install` calls `ukify` to produce a new UKI and writes it to the ESP. The signing hook runs after this, as a final install step.

### 3.2 The Signing Hook

```bash
# /etc/kernel/install.d/99-tpm2-sign.install
#!/bin/bash

COMMAND="$1"          # add or remove
KERNEL_VERSION="$2"
ENTRY_DIR="$3"        # path to the ESP entry directory

# Only act on installation, not removal
[[ "$COMMAND" == "add" ]] || exit 0

UKI="${ENTRY_DIR}${KERNEL_VERSION}.efi"

# Exit cleanly if UKI does not exist yet
[[ -f "$UKI" ]] || exit 0

systemd-measure sign \
  --linux="/lib/modules/${KERNEL_VERSION}/vmlinuz" \
  --initrd="/boot/initramfs-${KERNEL_VERSION}.img" \
  --osrel="/etc/os-release" \
  --cmdline="/etc/kernel/cmdline" \
  --pcrpkey="/etc/systemd/tpm2-pcr-public-key.pem" \
  --private-key="/etc/systemd/tpm2-pcr-private-key.pem" \
  --output="/usr/lib/systemd/tpm2-pcr-signature.json"

echo "TPM2 PCR 11 signature updated for kernel ${KERNEL_VERSION}"
```

```bash
chmod +x /etc/kernel/install.d/99-tpm2-sign.install
```

Replace the `--private-key` line with the appropriate PKCS#11 or unseal invocation depending on the chosen key storage model.

**What happens on `sudo dnf upgrade` (kernel update):**

```
dnf installs new kernel RPM
  → kernel-install add <version> called automatically
    → dracut rebuilds initramfs for new kernel
    → ukify assembles new UKI
    → UKI written to ESP
    → 99-tpm2-sign.install executes
      → systemd-measure predicts new PCR 11
      → .pcrsig signed and written
  → reboot → new PCR 11 matches new .pcrsig → disk unlocks
```

For kernel updates, this is fully automated after initial setup.

### 3.3 The NVIDIA Driver Problem

An NVIDIA driver update does not install a new kernel package. It rebuilds the kernel module and triggers dracut to regenerate the initramfs, but does not call `kernel-install add`. The signing hook therefore does not fire. The UKI on the ESP still contains the old initramfs. The new initramfs is on disk but not yet part of the measured boot object.

**Result without intervention:** The system reboots with the old UKI, which does not contain the new NVIDIA driver in its initramfs. The disk unlocks (PCR 11 matches the old `.pcrsig`), but the new driver is not active at boot time.

**Result if the UKI is rebuilt without signing:** PCR 11 changes. No valid `.pcrsig` exists. Lockout.

The fix is to ensure that any initramfs rebuild triggers a UKI rebuild and re-sign. Use a DNF post-transaction plugin:

```python
# /etc/dnf/plugins/tpm2-resign.py
import libdnf5.plugin
import subprocess
import os

# Packages that rebuild the initramfs but do not install a new kernel
WATCHED_PACKAGES = {
    "nvidia-driver",
    "akmod-nvidia",
    "kmod-nvidia",
    "dracut",
}

class Tpm2ResignPlugin(libdnf5.plugin.IPlugin):
    def post_transaction(self, ctx):
        changed = {p.get_name() for p in ctx.get_transaction_packages()}
        if not changed & WATCHED_PACKAGES:
            return

        # Rebuild initramfs for all installed kernels
        subprocess.run(["dracut", "--regenerate-all", "--force"], check=True)

        # Trigger kernel-install for each kernel version
        # This fires the signing hook via install.d
        for version in os.listdir("/lib/modules"):
            vmlinuz = f"/lib/modules/{version}/vmlinuz"
            if os.path.exists(vmlinuz):
                subprocess.run(
                    ["kernel-install", "add", version, vmlinuz],
                    check=True
                )
```

**With this plugin in place, the full automated flow for an NVIDIA update is:**

```
sudo dnf upgrade nvidia-driver
  → DNF installs new NVIDIA packages
  → tpm2-resign.py post_transaction fires
    → dracut --regenerate-all rebuilds initramfs with new driver
    → kernel-install add called for each kernel version
      → ukify assembles new UKI with updated initramfs
      → 99-tpm2-sign.install fires
        → systemd-measure signs new PCR 11
        → .pcrsig written
  → reboot → disk unlocks → new NVIDIA driver active
```

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

Rollback protection (preventing revert to a vulnerable image) is enforced by removing the old `.pcrsig`. Without a valid signature, the old image cannot unlock the disk even if it boots. See [[04_Governance_Recovery_Lifecycle#4.7 Rollback and Version Control|Module 4 — Rollback and Version Control]].

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

> [!warning] Recovery Keyslot Is Mandatory
> If the signing hook fails, the HSM is lost, the signing service is unreachable, or anything breaks the `.pcrsig` chain — the recovery keyslot is the only way back in.
>
> Enroll a high-entropy recovery passphrase at provisioning time and store it in an offline vault or escrow system. Without it, any failure in the signing workflow results in a permanently locked disk.

```bash
# Add recovery passphrase at provisioning time
systemd-cryptenroll /dev/nvme0n1p3 --password
```

See [[04_Governance_Recovery_Lifecycle#4.4 Disaster Recovery|Module 4 — Disaster Recovery]] for full recovery procedures.

---

## Related Notes

- [[00_Introduction|Introduction — Architecture Overview]]
- [[02_Forward_Sealing|Module 2 — Forward Sealing]]
- [[03_Disk_Encryption_and_Policy_Enforcement|Module 3 — Disk Encryption and Policy Enforcement]]
- [[04_Governance_Recovery_Lifecycle|Module 4 — Governance, Recovery, and Lifecycle]]
