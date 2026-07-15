---
tags:
  - trusted-boot
  - security
  - overview
module: 0
status: stable
---

# Trusted Boot Chain — Introduction

## What This Project Builds

This project implements a **hardware-backed trusted boot chain** for Linux systems. The goal is a self-verifying system that enforces integrity at boot time using hardware — requiring no trust in any individual person or software component.

The chain guarantees that disk encryption keys are only released when the system boots in a cryptographically verified, unmodified configuration. Any tampering with the kernel, initramfs, boot parameters, or Secure Boot policy causes the system to deny access to protected data.

> [!NOTE]
> **One-Sentence Definition**
> A system where the hardware itself decides whether the software is trustworthy enough to access the encrypted disk.

---

## The Four Modules

The architecture is structured as four progressive layers. Each module depends on the one before it.

```
Module 1 → Identity       (UKI + Measurement)
Module 2 → Authorization  (Forward Sealing)
Module 3 → Enforcement    (Policy Key in LUKS)
Module 4 → Governance     (Control Over Time)
```

| Module | Role | Core Question |
|--------|------|---------------|
| [Module 1](01_Unified_Kernel_Image.md) | Identity | What exactly booted? |
| [Module 2](02_Forward_Sealing.md) | Authorization | Is this boot state approved? |
| [Module 3](03_Disk_Encryption_and_Policy_Enforcement.md) | Enforcement | Who is allowed to authorize unlocking? |
| [Module 4](04_Governance_Recovery_Lifecycle.md) | Governance | Who controls the authority, and for how long? |

The project README describes a five-layer stack: layer 1 (the custom Secure Boot key setup) is a build-time prerequisite here, and layers 2 to 5 map to Modules 1 to 4.

> [!IMPORTANT]
> **The Distinction Between Modules 1–3 and Module 4**
> **Without Module 4:** The system enforces trust technically.
> **With Module 4:** The organisation governs trust operationally.
>
> A system without Module 4 is secure but brittle — it cannot survive key rotation, hardware failure, or personnel change.

---

## What This Prevents

- Booting a tampered kernel or initramfs
- Modifying kernel boot parameters to bypass security
- Cloning a disk and accessing it on different hardware
- Accessing encrypted data from a live USB or recovery environment
- Rolling back to a known-vulnerable system state (optional)

---

## What This Does Not Replace

- Application-layer security
- Network security controls
- Post-boot access control (PAM, sudo, SELinux)
- Physical security of the hardware itself

> [!WARNING]
> **Physical Access Threat Model**
> TPM-based disk unlock with no user interaction means a powered-on device with physical access reaches a decrypted disk. For high-value targets, combine with `--tpm2-with-pin` or a network-bound key (Tang/NBDE). See [Module 4](04_Governance_Recovery_Lifecycle.md).

---

## Trust Flow at a Glance

```
Firmware (UEFI)
  │  verifies signature
  ▼
Signed UKI                ← Secure Boot enforces this
  │  measures components
  ▼
TPM PCR Registers         ← Hardware records what ran
  │  evaluated against
  ▼
PCR Policy + Signature    ← Signed by CI/CD pipeline
  │  if approved
  ▼
LUKS Master Key Released  ← Disk decrypted
  │
  ▼
System boots
```

---

## Related Notes

- [Module 1 — UKI and Measurement](01_Unified_Kernel_Image.md)
- [Module 2 — Forward Sealing](02_Forward_Sealing.md)
- [Module 3 — Disk Encryption and Policy Enforcement](03_Disk_Encryption_and_Policy_Enforcement.md)
- [Module 4 — Governance, Recovery, and Lifecycle](04_Governance_Recovery_Lifecycle.md)
