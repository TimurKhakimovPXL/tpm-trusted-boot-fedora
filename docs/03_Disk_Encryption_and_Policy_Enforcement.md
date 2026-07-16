---
tags:
  - trusted-boot
  - luks
  - tpm
  - disk-encryption
  - policy-enforcement
module: 3
status: stable
---

# Module 3: Disk Encryption and Policy Enforcement

## Purpose of This Module

Modules 1 and 2 establish what the system is (UKI identity) and what future states are approved (forward sealing). This module addresses enforcement: **who is permanently authorised to approve those states**, and how that authority is written into the disk itself.

> [!NOTE]
> **Module 3 in One Line**
> Enroll the policy public key into the LUKS2 header so the disk will only unlock for TPM sessions authorised by your CI/CD pipeline's signing key.

---

## 1. The Trust Anchor: Policy Public Key Enrollment

### 1.1 The Concept of "Welding"

Using `systemd-cryptenroll`, the policy public key is written into a TPM2 keyslot in the LUKS2 disk header. Once enrolled, the disk carries a permanent record of whose signature it will accept.

The disk now enforces the following rule:

> Unlock if and only if the TPM presents a valid policy session that was authorised by the private key counterpart of this specific public key.

No other credential: not a different key, not a manually crafted PCR value, not a replayed session: satisfies this condition.

### 1.2 Enrollment Command

```bash
# Deploy the public key to the system
install -m 644 tpm2-pcr-public-key.pem /etc/systemd/tpm2-pcr-public-key.pem

# Enroll the TPM2 keyslot with SPLIT policy:
#   PCR 7: statically bound to current Secure Boot policy state
#   PCR 11: signed-policy bound via the policy public key
systemd-cryptenroll /dev/nvme0n1p3 \
  --tpm2-device=auto \
  --tpm2-pcrs=7 \
  --tpm2-public-key=/etc/systemd/tpm2-pcr-public-key.pem \
  --tpm2-public-key-pcrs=11
```

> [!IMPORTANT]
> **Asymmetric PCR treatment**
> The two PCRs are bound differently because they have different change frequencies:
> - **PCR 7** is the Secure Boot policy value. It is stable across the system's lifetime; runtime changes indicate SB key rotation, which is itself a governance event requiring re-enrollment. Static binding is appropriate.
> - **PCR 11** changes when measured UKI content changes, for example after a
>   kernel, initramfs or embedded command-line change. Static binding would
>   require manual re-enrollment for each new measured state. Signed-policy
>   binding lets the policy key authorise those future PCR 11 values.

> [!WARNING]
> **`--tpm2-signature` is intentionally omitted**
> The kernel-install hook signs PCR 11 for the `enter-initrd` phase, which produces a value that does not match the post-`ready` runtime state. `systemd-cryptenroll --tpm2-signature` validates the signature against the **current** PCR state (post-`ready`), so passing it produces a false-negative dry-run failure (`Failed to unseal secret using TPM2: No such device or address`). Boot-time unlock works correctly because `systemd-cryptsetup` runs in initrd context, where the signed PCR 11 value is the one the TPM observes. The trade-off is that enrollment does not pre-validate that the next boot will TPM-unlock; this is mitigated by retaining the passphrase keyslot 0 as recovery anchor and validating end-to-end through a reboot test. See `06F` finding G.1.

> [!IMPORTANT]
> **PCR list must match the signature**
> `--tpm2-public-key-pcrs=` must exactly match the PCR list covered by the embedded `.pcrsig` entry. The hook's `--phases=enter-initrd` signature has `pcrs=[11]`; demanding `7+11` here has no matching signature entry and unseal fails. Combining `--tpm2-pcrs=7` (for the static PCR) with `--tpm2-public-key-pcrs=11` (for the signed PCR) is the only valid shape for this hook. See `06F` finding G.1.

---

## 2. Platform Configuration Registers (PCRs)

### 2.1 What PCRs Are

PCRs live inside the TPM chip: not on the disk. They store accumulated hash measurements of everything that has run since power-on.

> [!TIP]
> **Scoreboard Analogy**
> Think of PCRs as a security scoreboard. From the moment the power button is pressed, each component that executes adds its hash to the running total. By the time disk unlock is requested, the scoreboard contains a cryptographic fingerprint of the entire boot sequence.

**Extension formula:**

$$PCR_{new} = \text{HASH}(PCR_{old} \mathbin\| \text{NewData})$$

Values can only be extended, never overwritten. They reset only on TPM reset or power cycle.

### 2.2 PCRs Used in This Project

| PCR | Measures | Stability |
|-----|----------|-----------|
| **7** | Secure Boot state: DB, DBX, PK, KEK certificates | Stable until key rotation |
| **11** | UKI sections + boot phase markers | Changes when measured content or phase inputs change |

By the time `systemd-cryptsetup` requests the disk key, PCR 7 and PCR 11 together encode, which Secure Boot policy is active, and exactly which UKI ran.

> [!NOTE]
> **PCR 15: System Identity (Conditional Addition)**
> PCR 15 (`system-identity`) measures machine ID, root filesystem UUID, and LUKS metadata. Including it in the policy provides protection against filesystem confusion attacks: where an attacker swaps GPT partitions or duplicates UUIDs to trick the unlock mechanism.
>
> In the split-policy model used by this project, PCR 15 must be **deliberately assigned to either the static PCR policy path or the signed-policy path**, not both:
> - Adding it via `--tpm2-pcrs=` (alongside PCR 7) binds the keyslot to the current PCR 15 value statically: fine as long as the bound system identity doesn't change.
> - Adding it via `--tpm2-public-key-pcrs=` (alongside PCR 11) requires the UKI / `.pcrsig` generation path to **also sign for PCR 15**. The validated hook in this project signs only PCR 11; adding PCR 15 to the signed-policy list without updating the hook causes enrollment-time policy lookup or boot-time unlock to fail.
>
> For high-assurance deployments, the static-policy form is the lower-risk addition.

---

## 3. The LUKS2 Disk Header

### 3.1 Structure

The LUKS2 header is a metadata area at the very beginning of the encrypted partition (e.g. `/dev/nvme0n1p3`), before any encrypted data.

It contains:

- **Keyslots**: each holds an encrypted copy of the master key, protected by a different credential
- **Tokens**: metadata linking keyslots to external unlock mechanisms (TPM, FIDO2, etc.)
- **Configuration**: cipher, hash algorithm, key size

### 3.2 PCRs vs Keyslots: Two Separate Things

This distinction is frequently confused.

| | PCRs | Keyslots |
|--|------|----------|
| **Location** | Inside the TPM chip | Inside the LUKS2 header on disk |
| **Contents** | Hash measurements of the boot process | Encrypted copies of the master key |
| **Role** | Security journal: records what ran | Lock: protects the key |
| **Controlled by** | TPM hardware | `cryptsetup` / `systemd-cryptenroll` |

They work together: the keyslot stores the key, and the PCRs determine whether the TPM is willing to release the credential needed to open that keyslot.

### 3.3 The Master Key

The LUKS2 master key is the actual symmetric key that encrypts and decrypts all data on the volume.

- It is generated once at `cryptsetup luksFormat` time (256-bit symmetric)
- It never leaves the kernel in plaintext during normal operation
- Every keyslot holds the same master key, wrapped differently
- The TPM2 keyslot wraps it with a secret that the TPM only releases under a valid policy session

---

## 4. The Unlock Sequence at Boot

When `systemd-cryptsetup@cryptroot.service` runs during initramfs, the following exchange occurs between it, the disk, and the TPM:

```
1. REQUEST
   systemd-cryptsetup reads the LUKS2 header
   → Finds a TPM2 keyslot with an enrolled policy public key

2. CHALLENGE
   systemd-cryptsetup asks the TPM to start a policy session

3. PROOF
   systemd-cryptsetup presents the .pcrsig to the TPM
   → "Here is the signed authorisation for the current boot state"

4. VERIFICATION
   TPM evaluates the split-policy session:
   a) PCR 7 matches the value statically bound at enrollment time
      (the Secure Boot policy state captured then must equal now)
   b) PCR 11 satisfies a policy authorised by the enrolled public key
      and the UKI-provided .pcrsig
   (PCR 7 is not signed. Only PCR 11 is. The two PCRs are bound by
    different mechanisms: see §1.2 for the architectural reason.)

5. RELEASE
   Both checks pass → TPM releases the sealed secret
   → systemd-cryptsetup uses the secret to unwrap the master key
   → Master key passed to dm-crypt

6. MAPPING
   Kernel maps: /dev/nvme0n1p3 → /dev/mapper/cryptroot
   → System continues boot on decrypted volume
```

If either check in step 4 fails: wrong key, tampered UKI, unapproved boot state: the TPM releases nothing. The disk remains encrypted and inaccessible.

---

## 5. The `/etc/crypttab` Entry

The unlock process is triggered by the `crypttab` entry for the volume:

```
# /etc/crypttab
cryptroot UUID=<luks-partition-uuid> none discard,tpm2-device=auto
```

When systemd encounters this during initramfs startup, it spawns `systemd-cryptsetup@cryptroot.service`, which executes the unlock sequence above.

> [!IMPORTANT]
> **The PCR policy lives in the LUKS2 token, not in crypttab**
> The PCR list is **not** duplicated in `/etc/crypttab`. The enrolled `systemd-tpm2` token in the LUKS2 header is the source of truth:
>
> - `tpm2-hash-pcrs: 7` (statically bound)
> - `tpm2-pubkey-pcrs: 11` (signed-policy bound)
> - `tpm2-pcr-bank: sha256`
>
> The crypttab option `tpm2-device=auto` is sufficient: it tells `systemd-cryptsetup` to look up the TPM2 token and use whatever PCR configuration was committed at enrollment time. Specifying `tpm2-pcrs=` or `tpm2-public-key-pcrs=` in crypttab would create a second source of truth that can drift from the LUKS token; the systemd convention is to keep the policy on the token where it lives with the keyslot it protects.

> [!NOTE]
> **Initramfs propagation**
> Edits to `/etc/crypttab` do not automatically reach the boot path. The validated kernel-install hook reads `/boot/initramfs-${KVER}.img` as the source of `--initrd=` for `ukify build`. `kernel-install add` under UKI layout does not refresh that file when the kernel is already installed. Operators must run `dracut --force /boot/initramfs-${KVER}.img ${KVER}` **before** `kernel-install add`, then verify the embedded UKI initrd contains the new option via `lsinitrd`. See `06F` finding G.2 and `06B` Step 24.

---

## 6. The Complete Theory Chain

This module is the third link in the chain that spans all four modules:

```
Module 1 → IDENTITY
           You have a signed, atomic UKI that produces a deterministic PCR 11

Module 2 → PREDICTION
           Your pipeline calculates and signs the PCR 11 state for that UKI

Module 3 → AUTHORITY
           You enroll the public key into the disk header
           → The disk now only unlocks for signatures from your pipeline

Module 4 → GOVERNANCE
           You define how that authority is managed, rotated, and revoked over time
```

The disk does not trust the system. It trusts the key. The key trusts the pipeline. The pipeline is governed by process.

---

## 7. Recovery Keyslot: Mandatory

> [!WARNING]
> **Always Maintain a Recovery Keyslot**
> A disk with only a TPM2 keyslot is a disk you can permanently lose access to. Firmware updates, TPM replacement, Secure Boot key rotation, and PCR policy changes can all cause TPM unlock to fail.
>
> Red Hat explicitly mandates a recovery passphrase for all TPM-backed LUKS configurations. Treat this as a hard requirement, not an optional extra.

```bash
# Add a high-entropy recovery passphrase to a separate keyslot
systemd-cryptenroll /dev/nvme0n1p3 --password
```

Store the recovery credential in an offline vault or escrow system. See [Module 4: Disaster Recovery](04_Governance_Recovery_Lifecycle.md) for full recovery procedures.

---

## Related Notes

- [Introduction: Architecture Overview](00_Introduction.md)
- [Module 1: UKI and Measurement](01_Unified_Kernel_Image.md)
- [Module 2: Forward Sealing](02_Forward_Sealing.md)
- [Module 4: Governance, Recovery, and Lifecycle](04_Governance_Recovery_Lifecycle.md)
