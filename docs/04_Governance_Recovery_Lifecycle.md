---
tags:
  - trusted-boot
  - governance
  - key-management
  - recovery
  - lifecycle
module: 4
status: stable
---

# Module 4 — Governance, Recovery, and Lifecycle Control

## Purpose of This Module

Modules 1–3 establish a technically sound, hardware-backed trusted boot chain. A system built on those three modules enforces trust. This module defines how an **organisation governs** that trust over time.

Without Module 4, the design is secure but brittle: a single compromised key, replaced motherboard, or unplanned firmware update can cause permanent loss of access.

Module 4 addresses:

- How authority is structured and delegated
- How keys are rotated without downtime
- How compromise is detected and contained
- How recovery works when the TPM path fails
- How the system is safely decommissioned

> [!NOTE]
> **The Core Principle**
> The system no longer depends on trust in people. It depends on cryptographic authority with governed lifecycle control.

---

## 4.1 Trust Authority Model

### 4.1.1 Trust Anchor

The **policy public key** enrolled in the LUKS2 header is the root of authority for disk unlock. Only PCR signatures produced by its corresponding private key are accepted.

This means:

> The entity controlling the private key controls the machine.

Key governance is therefore equivalent to system governance. The security of the entire architecture reduces to the security of the private signing key.

### 4.1.2 Signing Authority Hierarchy

The private key must never reside on developer machines, never be committed to version control, and never be reused manually outside the CI pipeline.

Recommended hierarchy:

```
Offline Root Key        (optional — long-term, air-gapped)
        │
        ▼
CI/CD Signing Key       (operational — used per UKI build)
        │
        ▼
UKI PCR Signatures      (per-image artefacts)
```

The offline root key signs the CI key. The CI key signs PCR policies. If the CI key is compromised, it can be rotated without touching the offline root. If you choose not to use a root key hierarchy, the CI key becomes the trust anchor directly.

### 4.1.3 Independent Security Layers

Secure Boot and the TPM policy key are independent controls. A compromise of one does not imply total failure:

| Scenario | Secure Boot | TPM Policy | Outcome |
|----------|-------------|------------|---------|
| Normal operation | ✅ Validates UKI signature | ✅ Validates PCR signature | Disk unlocks |
| Policy key compromised | ✅ Still validates UKI | ❌ Attacker can sign PCR policies | Rotate policy key immediately |
| Secure Boot key compromised | ❌ Attacker can sign UKIs | ✅ Still validates PCR policy | Rotate Secure Boot db key; PCR 7 changes |
| Both compromised | ❌ | ❌ | Full incident response required |

Defence-in-depth means one failure does not equal total compromise.

---

## 4.2 Key Lifecycle Management

### 4.2.1 Key Rotation Triggers

Key rotation is required when any of the following occur:

- Signing key is suspected or confirmed compromised
- CI/CD environment is rebuilt or migrated
- Organisation changes ownership or personnel with key access
- Scheduled rotation policy interval is reached (e.g. annually)

### 4.2.2 Zero-Downtime Rotation with Dual Keyslots

LUKS2 supports multiple keyslots simultaneously. Use this capability to rotate keys without a window of inaccessibility.

**Procedure:**

```
Phase 1 — Prepare new key
  1. Generate new keypair (new policy private key + new policy public key)
  2. Enroll new public key into a second LUKS2 TPM2 keyslot (Slot B)
     systemd-cryptenroll /dev/nvme0n1p3 \
       --tpm2-device=auto \
       --tpm2-public-key=new-pcr-public-key.pem \
       --tpm2-public-key-pcrs=7+11 \
       --tpm2-signature=new-tpm2-pcr-signature.json
  3. Verify Slot B unlocks successfully before proceeding

Phase 2 — Transition
  4. Update CI/CD pipeline to sign with new private key
  5. Re-sign current approved UKI with new key; ship updated .pcrsig

Phase 3 — Remove old key
  6. Confirm all hosts unlock via Slot B
  7. Remove old keyslot (Slot A)
     systemd-cryptenroll /dev/nvme0n1p3 --wipe-slot=<old-slot-number>
```

> [!WARNING]
> Never remove the old keyslot until the new one is verified working on the target system. Removing it prematurely causes permanent lockout.

The `--wipe-slot=tpm2` shorthand removes all existing TPM2 slots and can be used for atomic replacement in one operation when re-enrolling from scratch.

---

## 4.3 Compromise Response

### 4.3.1 Policy Private Key Compromise

**Immediate response:**

1. Disable the CI/CD pipeline — halt all signing operations
2. Generate a new keypair
3. Enroll new public key into the LUKS2 header (Slot B, following rotation procedure)
4. Re-sign the current approved UKI with the new private key
5. Remove the compromised keyslot

**If an attacker signed a malicious UKI before detection:** The malicious UKI would pass PCR policy validation (policy key was theirs). However, it would still need to pass Secure Boot signature validation. If the attacker does not also hold the Secure Boot `db` private key, the malicious UKI will not boot.

This is why Secure Boot key and policy key must be kept separately and should never be held by the same person.

### 4.3.2 CI/CD Runner Compromise

If the CI runner itself is compromised, assume the private key is exposed regardless of whether the key file was directly accessed.

Response:

1. Treat private key as compromised — begin rotation immediately
2. Rebuild the runner from a known-clean state
3. Audit the full artefact history produced by the compromised runner
4. Review all signing logs for anomalous entries

All signing operations must produce immutable, timestamped audit logs. Without these, artefact history cannot be verified.

---

## 4.4 Disaster Recovery

> [!WARNING]
> **Mandatory Recovery Keyslot**
> Every LUKS2 volume with a TPM2 keyslot must also have an independent recovery credential — a high-entropy passphrase or offline recovery key stored in a secure vault. Red Hat explicitly requires this for all TPM-backed LUKS configurations.
>
> A disk with only a TPM2 keyslot is a disk you can permanently lose access to. Firmware updates, Secure Boot key rotation, TPM replacement, or PCR policy drift can all cause TPM unlock to fail.

### 4.4.1 TPM Failure or Motherboard Replacement

When a motherboard is replaced, the TPM is also replaced. The new TPM has no knowledge of the old sealed secret.

**Recovery procedure:**

1. Boot into recovery mode or live environment
2. Unlock the disk using the recovery passphrase or offline recovery key
3. Re-enroll a new TPM2 keyslot using the new TPM:
   ```bash
   systemd-cryptenroll /dev/nvme0n1p3 \
     --wipe-slot=tpm2 \
     --tpm2-device=auto \
     --tpm2-public-key=/etc/systemd/tpm2-pcr-public-key.pem \
     --tpm2-public-key-pcrs=7+11 \
     --tpm2-signature=/usr/lib/systemd/tpm2-pcr-signature.json
   ```
4. Verify TPM unlock works with the new hardware
5. Retain or remove the recovery key according to policy

### 4.4.2 Disk Migration to New Hardware

Moving a LUKS2 disk to a different machine means the new TPM will have different PCR values (different firmware, different Secure Boot keys). TPM unlock will fail.

**Procedure:**

1. Unlock the disk on the new hardware using the recovery credential
2. Re-enroll the TPM2 keyslot for the new machine's TPM and PCR state
3. Verify unlock
4. Optionally remove the recovery key after confirming TPM path works

### 4.4.3 Recovery Runbook Reference

Every system should have a documented runbook covering:

- How to boot to an initramfs emergency shell
- How to unlock the disk manually with the recovery passphrase
- How to regenerate a UKI and `.pcrsig` for a changed PCR state
- How to re-enroll a TPM2 keyslot after hardware change

---

## 4.5 Administrative Control Model

### 4.5.1 Separation of Duties

No single person should hold complete control over the trust chain.

| Role | Capability |
|------|------------|
| Developer | Build UKI artefacts |
| Maintainer | Approve and merge changes |
| CI System | Sign PCR values (holds operational private key) |
| Security Administrator | Rotate keys, manage keyslots |
| Recovery Officer | Hold and manage the recovery credential |

Separation of duties means that compromise of any single role does not compromise the entire system.

### 4.5.2 Protected Branch Enforcement

The signing pipeline should trigger only when:

- A commit is merged into a designated protected branch
- The merge required a review approval
- The pipeline runs in a protected, isolated environment

This prevents a developer from signing an arbitrary UKI by pushing directly to the signing trigger.

---

## 4.6 Audit and Logging

All signing operations must produce immutable, append-only log entries containing:

| Field | Purpose |
|-------|---------|
| Commit hash | Links signed artefact to source code |
| PCR value signed | Records which system state was authorised |
| Timestamp | Establishes timeline for forensic review |
| Signing key ID | Identifies which key performed the signing |

> [!IMPORTANT]
> If you cannot prove what was signed, by which key, and when — you do not have governance. You have the appearance of governance.

Audit logs must be stored outside the CI system so that a compromised runner cannot alter its own history.

---

## 4.7 Rollback and Version Control

Forward sealing supports rollback prevention. To deny access to a deprecated or vulnerable UKI version:

- Do not re-sign its PCR value (no new `.pcrsig` issued)
- Remove any existing `.pcrsig` for that version from the system
- Maintain a version whitelist — only currently signed PCR values are authorised

> [!NOTE]
> Rollback support (allowing older versions to boot) is achieved by keeping multiple valid `.pcrsig` files on the system — one per authorised UKI version. `systemd-cryptsetup` will try each signature until one matches the current PCR state. Remove a signature to revoke that version's access.

---

## 4.8 Decommissioning Procedure

When retiring a system, ensure the trust authority is fully revoked:

```
1. Remove all TPM2 keyslots from the LUKS2 header
   systemd-cryptenroll /dev/nvme0n1p3 --wipe-slot=tpm2

2. Remove recovery keyslots
   systemd-cryptenroll /dev/nvme0n1p3 --wipe-slot=<recovery-slot>

3. Wipe the LUKS2 header entirely
   cryptsetup erase /dev/nvme0n1p3

4. If the organisation is dissolving: securely destroy the policy private key
```

This prevents reused hardware from retaining any trust authority from its previous deployment.

---

## 4.9 System-Level Architecture

The four modules form a complete, layered architecture:

```
Module 1 → IDENTITY
           UKI bundles all boot components into a single signed, measurable unit
           TPM records the cryptographic fingerprint of what ran

Module 2 → AUTHORIZATION
           Forward sealing signs approved future PCR states
           Updates do not require re-enrollment

Module 3 → ENFORCEMENT
           Policy public key enrolled in LUKS2 header
           Disk only unlocks for TPM sessions authorised by the signing pipeline

Module 4 → GOVERNANCE
           Key lifecycle, compromise response, recovery, and decommissioning
           Organisation controls the authority, not individuals
```

---

## 4.10 Complete Architecture Guarantees

| Property | Mechanism |
|----------|-----------|
| Boot integrity | TPM measurement via systemd-stub |
| Update safety | Forward sealing with systemd-measure |
| Cryptographic enforcement | TPM2 keyslot in LUKS2 header |
| Controlled authority | Policy key governance |
| Safe recovery | Recovery keyslot + documented runbook |
| Secure evolution | Zero-downtime key rotation via dual keyslots |
| Auditability | Immutable signing logs per artefact |
| Decommissioning | Full keyslot and header wipe procedure |

---

## Related Notes

- [Introduction — Architecture Overview](00_Introduction.md)
- [Module 1 — UKI and Measurement](01_Unified_Kernel_Image.md)
- [Module 2 — Forward Sealing](02_Forward_Sealing.md)
- [Module 3 — Disk Encryption and Policy Enforcement](03_Disk_Encryption_and_Policy_Enforcement.md)
