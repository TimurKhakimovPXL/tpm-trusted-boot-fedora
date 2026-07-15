---
tags:
  - trusted-boot
  - forward-sealing
  - tpm
  - luks
  - systemd-measure
module: 2
status: stable
---

# Module 2 — Secure OS Updates with Forward Sealing

## Purpose of This Module

Module 1 established that the TPM records a cryptographic fingerprint of what booted. This module addresses the problem that fingerprint changes with every OS update — and explains how to authorize future system states without requiring manual re-enrollment of the LUKS volume after each update.

> [!NOTE]
> **Module 2 in One Line**
> Replace static PCR binding with signature-based authorization so that approved OS updates can unlock the disk without breaking encryption.

---

## 1. The Problem: Static PCR Sealing Breaks on Updates

### 1.1 Where the LUKS Key Comes From

When a LUKS2 volume is created with `cryptsetup luksFormat`, Linux generates a 256-bit symmetric master key. This key is stored **encrypted** inside the LUKS2 header. Every keyslot in the header holds a different encrypted copy of the same master key, unlocked by different credentials (passphrase, TPM, FIDO2, etc.).

In a TPM-backed setup, the TPM holds a secret that can decrypt one of those keyslots — but only under a specific policy.

### 1.2 The Static Sealing Model

The naive approach seals the LUKS key to a fixed PCR value:

```
TPM → Release key only if PCR11 == <stored hash>
```

This creates a strict equality check. The stored hash is the PCR 11 value recorded at enrollment time, corresponding to the UKI that was running then.

**The fatal flaw:** Any legitimate OS update — new kernel version, updated initramfs, changed cmdline — produces a different UKI. A different UKI produces a different PCR 11. The equality check fails. The TPM refuses to release the key. **The disk becomes permanently inaccessible.**

Static sealing makes security and maintainability mutually exclusive.

---

## 2. The Solution: Forward Sealing

### 2.1 Core Concept

Forward sealing replaces the fixed-value equality check with **signature-based authorization**:

> Instead of trusting a single PCR value, the TPM trusts any PCR value that has been **cryptographically approved** by a designated signing authority.

The signing authority is the CI/CD pipeline that builds and releases OS images. It holds the policy private key. Any PCR value signed by that key is considered an approved system state.

### 2.2 The Roles of systemd-measure and systemd-stub

These two tools are frequently confused. Their roles are complementary and non-interchangeable.

| Tool | When | What it does |
|------|------|-------------|
| `systemd-measure` | Build time (offline) | **Predicts** the PCR 11 value a given UKI will produce at runtime |
| `systemd-stub` | Boot time (on device) | **Proves** the actual PCR 11 value by performing the real measurements |

Forward sealing works because these two values must match. `systemd-measure` computes what PCR 11 *will be*. `systemd-stub` produces what PCR 11 *is*. If the UKI was not tampered with, they are identical.

> [!IMPORTANT]
> **Why Both Are Needed**
> `systemd-measure` alone cannot be trusted — it runs offline and could be fed arbitrary inputs. `systemd-stub` alone cannot authorize future states — it only measures what is currently booting.
>
> Forward sealing requires both: `systemd-measure` predicts and signs authorized future states; `systemd-stub` proves the running state is one of them.

---

## 3. Forward Sealing Workflow

### 3.1 Build-Time Process (CI/CD Pipeline)

This runs in the CI environment for every new OS image release.

```
Step 1: Build new UKI
        ukify build --linux=... --initrd=... --cmdline=... --output=linux.efi

Step 2: Predict PCR 11
        systemd-measure calculate \
          --linux=vmlinuz \
          --initrd=initrd.img \
          --osrel=/etc/os-release \
          --cmdline=/etc/kernel/cmdline \
          --pcrpkey=tpm2-pcr-public-key.pem

Step 3: Sign the predicted PCR value
        systemd-measure sign \
          --linux=vmlinuz \
          --initrd=initrd.img \
          --private-key=policy-private-key.pem \
          --public-key=tpm2-pcr-public-key.pem \
          --output=tpm2-pcr-signature.json

Step 4: Embed or ship the signature
        Option A: Embed .pcrsig section into the UKI at build time
        Option B: Ship tpm2-pcr-signature.json to /usr/lib/systemd/
```

> [!NOTE]
> **Signature Storage Options**
> Embedding the `.pcrsig` inside the UKI keeps the signature self-contained and is preferred for standalone deployments. Shipping it as an external JSON file (`/usr/lib/systemd/tpm2-pcr-signature.json`) is useful when signatures are managed separately from images, such as in fleet deployments with rolling updates.

### 3.2 Boot-Time Process (On the Device)

```
1.  UEFI verifies UKI PE signature         → Secure Boot gate
2.  systemd-stub hashes each UKI section   → Two extends per section (name + content)
3.  systemd-stub extends hashes into PCR 11 → TPM records actual state
4.  systemd-cryptsetup reads LUKS2 header  → Finds TPM2 keyslot
5.  systemd-cryptsetup reads .pcrsig       → From UKI or /usr/lib/systemd/
6.  Signature verified with policy pubkey  → Was this PCR state approved?
7.  TPM policy session created             → Session bound to PCR constraints
8.  TPM releases sealed secret             → Only if PCRs match signed prediction
9.  Secret passed to dm-crypt              → Master key decrypted
10. Disk mapped: /dev/sda2 → /dev/mapper/cryptroot
```

If signature validation fails at step 6, or PCR values do not match at step 8, the disk remains inaccessible.

---

## 4. systemd-cryptsetup — The Enforcement Bridge

`systemd-cryptsetup` is the systemd component that manages encrypted disk unlock during early boot. It is the integration point between TPM, LUKS, policy, and the kernel.

**Timeline:**

```
Kernel
  → initramfs
    → systemd starts (early userspace)
      → systemd-cryptsetup@cryptroot.service runs
        → disk unlock
          → switch_root to real OS
```

**Trigger:** `/etc/crypttab` entry:

```
cryptroot UUID=<luks-uuid> none tpm2-device=auto
```

When systemd sees this entry, it starts `systemd-cryptsetup@cryptroot.service`.

**What it does:**

| Step | Action |
|------|--------|
| 1. Request | Read LUKS2 header, locate TPM2 keyslot |
| 2. Challenge | Ask TPM to start a policy session |
| 3. Proof | Present the `.pcrsig` signature to the TPM |
| 4. Verification | TPM checks: was this signature made by the policy private key? Do the PCR values match current state? |
| 5. Release | If both checks pass, TPM releases the sealed secret |
| 6. Unlock | Secret passed to dm-crypt; disk mapped |

> [!IMPORTANT]
> **systemd-cryptsetup Is the Bridge**
> Without it, the TPM cannot unlock LUKS. Without it, forward sealing has no runtime enforcement. It connects every component in the chain: TPM policy, LUKS keyslot, PCR signature, and kernel dm-crypt.

---

## 5. Component Responsibility Model

```
systemd-stub        → measures  (extends real PCR values at boot)
TPM                 → stores    (accumulates and holds PCR state)
systemd-measure     → predicts  (computes expected PCR values at build time)
systemd-cryptsetup  → enforces  (validates signature, manages unlock)
cryptsetup          → decrypts  (operates on the LUKS layer)
kernel dm-crypt     → maps      (exposes /dev/mapper/cryptroot)
```

---

## 6. Implementation Requirements

### 6.1 Predictable PCR 11 State

`systemd-measure` predicts PCR 11 starting from an all-zero initial state. If anything extends PCR 11 **before** systemd-stub measures the UKI, the predicted and actual values will diverge and unlock will fail.

Ensure no firmware or bootloader component extends PCR 11 prematurely.

### 6.2 Consistent Hash Algorithm

All components — measurement, sealing, and unlocking — must use the same PCR bank. Use SHA-256 throughout. Mixing SHA-1 and SHA-256 banks across components will cause mismatches.

### 6.3 Boot Phase Markers

`systemd-pcrphase` extends boot phase markers into PCR 11 at runtime: `enter-initrd`, `leave-initrd`, `sysinit`, `ready`. These are part of the measured state. `systemd-measure` accounts for these automatically when invoked correctly — but be aware that PCR 11 at disk-unlock time is not solely the UKI hash. It also contains phase markers from earlier in the boot.

### 6.4 Enrollment Safety Check

When enrolling a LUKS TPM2 slot with `--tpm2-signature`, `systemd-cryptenroll` validates that the provided signature actually works against the current PCRs **before** writing the keyslot. This prevents bricking the volume with a misconfigured or mismatched signature. Never skip this step or bypass it.

---

## 7. Security Guarantees

| Guarantee | Mechanism |
|-----------|-----------|
| **Update safety** | New UKI with valid `.pcrsig` unlocks without re-enrollment |
| **Integrity control** | Only signed, authorized PCR states are accepted |
| **Tamper protection** | Modified kernel, initramfs, or cmdline produce a different PCR 11 |
| **Rollback prevention** | Old `.pcrsig` files can be removed to deny access to deprecated states |

---

## 8. Known Limitations and Risks

> [!WARNING]
> **TPM Auto-Unlock and Physical Access**
> If the disk unlocks from TPM with no user interaction, a powered-on device with physical access gives an attacker a decrypted disk. For high-value targets, add `--tpm2-with-pin=yes` to require a PIN in addition to PCR validation.

> [!WARNING]
> **Bus Interposer Attacks**
> Discrete TPM chips connected via SPI or LPC buses can be subject to hardware interposer attacks that inject fake PCR extends or replay measurement streams. The Linux kernel mitigates this with null-seed HMAC session chaining, but a determined attacker with hardware access and sufficient resources remains a residual risk. Treat TPM-based boot security as one layer of defence-in-depth, not the sole control.

---

## Related Notes

- [Introduction — Architecture Overview](00_Introduction.md)
- [Module 1 — UKI and Measurement](01_Unified_Kernel_Image.md)
- [Module 3 — Disk Encryption and Policy Enforcement](03_Disk_Encryption_and_Policy_Enforcement.md)
- [Module 4 — Governance, Recovery, and Lifecycle](04_Governance_Recovery_Lifecycle.md)
