---
tags:
  - trusted-boot
  - uki
  - tpm
  - secure-boot
  - measurement
module: 1
status: stable
---

# Module 1: The Unified Kernel Image (UKI) and Measurement

## Purpose of This Module

This module establishes **system identity**. Before a disk can be unlocked based on system state, the system must produce a deterministic, tamper-evident representation of itself. That representation is the UKI, and the mechanism that records it is the TPM.

> [!NOTE]
> **Module 1 in One Line**
> Bundle the entire boot environment into one signed, measurable unit so the TPM can produce a cryptographic fingerprint of exactly what booted.

---

## 1. Foundational Concepts

### 1.1 The Kernel

The kernel is the core of the OS. It manages hardware directly and runs in privileged kernel space, responsible for:

- CPU scheduling and memory management
- Disk, network, and device access
- Process isolation and security enforcement

The kernel alone cannot boot a system. It needs a temporary environment to prepare storage before the real root filesystem can be mounted.

### 1.2 The initramfs (Initial RAM Filesystem)

The initramfs is a small, temporary filesystem loaded into RAM at boot. It provides the **early userspace** that the kernel needs to bootstrap itself into the real system.

**Why it is needed:** At the point the kernel starts, it cannot yet access:

- LUKS-encrypted disks
- LVM or RAID volumes
- Network-attached storage
- Storage requiring special drivers

The initramfs provides the tools and context to handle all of this.

**What it does, in order:**

1. Kernel loads initramfs into RAM and mounts it as `/`
2. `init` runs inside the initramfs environment
3. `init` loads required drivers, unlocks LUKS volumes, assembles logical volumes, and locates the real root filesystem
4. Control transfers to the real OS via `switch_root` / `pivot_root`
5. initramfs unmounts and is discarded from memory

> [!IMPORTANT]
> **Common Misconception**
> Neither the kernel nor the initramfs itself unlocks the disk. The initramfs **loads the userspace tools** (`systemd-cryptsetup`, `cryptsetup`) that perform the unlock. The initramfs is the environment; the tools inside it do the work.

**Correct assumption:** The kernel has the drivers but not the map. The initramfs provides the map: where root lives, how to unlock it, and which device to trust.

---

## 2. The Unified Kernel Image (UKI)

### 2.1 What It Is

In traditional setups, the kernel, initramfs, and kernel command line are stored as **separate files** on the ESP. Each can be modified independently, which creates attack surface.

A **UKI** is a single UEFI-executable PE binary that bundles all boot components into one atomic object:

| Section | Content |
|---------|---------|
| `.linux` | Kernel image (mandatory) |
| `.initrd` | Initramfs |
| `.cmdline` | Kernel command line |
| `.osrel` | `/etc/os-release` metadata |
| `.ucode` | CPU microcode (if included) |
| `.pcrpkey` | PEM public key for PCR policy (optional) |
| `.pcrsig` | Signed PCR prediction JSON (optional) |

**Typical location:** `/EFI/Linux/linux.efi`

**Built with:** `ukify`, `dracut`, or `mkosi`

> [!NOTE]
> **The Atomic Guarantee**
> A UKI is all-or-nothing. You cannot modify the initramfs without invalidating the kernel's signature. You cannot change the cmdline without producing a different TPM measurement. Every component is cryptographically bound to every other.

### 2.2 Classical Boot vs UKI Boot

**Classical boot: fragmented trust:**

```
Bootloader → Kernel      (separate file, can be replaced)
           → Initramfs   (separate file, can be replaced)
           → Cmdline     (separate config, can be modified)
```

Attack surface: any component can be modified, redirected, or replaced independently. An attacker who replaces the initramfs can intercept disk decryption, implant backdoors, or redirect the root filesystem, and Secure Boot cannot detect this if only the kernel is signed.

**UKI boot: unified trust:**

```
UEFI → Signed UKI → Verified + Measured
```

Everything boots together, or nothing boots.

---

## 3. The Boot and Measurement Flow

### 3.1 Step-by-Step Boot Sequence

```
1. UEFI loads /EFI/Linux/linux.efi
2. Secure Boot verifies the UKI's PE signature against the db certificate
3. systemd-stub executes (embedded mini-loader inside the UKI)
4. systemd-stub measures each UKI section into TPM PCR 11
5. Kernel starts
6. Embedded initramfs runs
7. systemd initializes userspace
```

### 3.2 systemd-stub

`systemd-stub` is the PE stub embedded inside every UKI. It is the root of measurement: the component that bridges UEFI execution and TPM recording.

**Responsibilities:**

- Parses and validates the embedded kernel command line
- Measures each UKI PE section into the TPM
- Measures boot phase markers (enter-initrd, leave-initrd, ready) into PCR 11
- Hands off execution to the kernel

> [!IMPORTANT]
> **PCR 11 Measurement Detail**
> For each PE section, systemd-stub performs **two** TPM extends:
> 1. The section **name** in ASCII (with trailing NUL) → PCR 11
> 2. The section **binary content** → PCR 11
>
> This means PCR 11 is sensitive to both the content and the naming of sections. Use `ukify` strictly: it implements the UAPI.5 spec and produces measurements that `systemd-measure` can predict accurately.

**Runtime execution chain:**

```
UKI → systemd-stub → kernel → initramfs → switch_root → systemd
```

---

## 4. TPM PCR Measurements

### 4.1 What PCRs Are

Platform Configuration Registers (PCRs) are registers inside the TPM chip that accumulate measurements of the boot process. They cannot be written to directly: only **extended**:

$$PCR_{new} = HASH(PCR_{old} \mathbin\| NewData)$$

This hash-chaining means PCR values are append-only digests of all prior measurements. They reset only on TPM reset or power-on.

> [!TIP]
> **assumption**
> Think of the TPM as a flight recorder (black box). It passively records everything that happens from power-on. It does not verify or approve anything: it only records. The policy engine reads the recording and decides.

### 4.2 PCR Registry (UAPI.7)

The Linux TPM PCR Registry defines which component measures into which register on systemd-based systems:

| PCR | Semantic Name | Measures |
|-----|--------------|---------|
| 0 | `platform-code` | Core firmware (UEFI) |
| 1 | `platform-config` | BIOS settings, hardware identity |
| 2–3 |: | Option ROM and device firmware |
| 4 | `boot-loader-code` | Bootloader binaries; systemd-stub also writes here |
| 5 | `boot-loader-config` | `loader.conf`, partition table |
| 7 | `secure-boot-policy` | Secure Boot state, DB/DBX/PK/KEK certificates |
| 9 | `kernel-initrd` | Kernel-measured initrds (volatile: avoid for direct binding) |
| 11 | `kernel-boot` | UKI sections + boot phase markers |
| 12 | `kernel-config` | Command lines, credentials |
| 13 | `sysexts` | System extension images |
| 14 | `shim-policy` | Shim MOK keys (zeroed when not using shim) |
| 15 | `system-identity` | Machine ID, filesystem UUIDs, LUKS metadata |

**PCRs used in this project's policy:** PCR 7 (Secure Boot state) and PCR 11 (UKI identity).

> [!NOTE]
> **Shim Elimination Advantage**
> This project uses systemd-boot directly without shim. As a result, PCR 14 (`shim-policy`) remains at its initial zero value and is stable: one fewer variable in the trust equation.

> [!WARNING]
> **PCR Stability Considerations**
> - PCR 0, 2: change with firmware updates. Do not use for direct long-lived binding.
> - PCR 9: highly volatile on systems with frequent dracut regeneration. Avoid for direct binding.
> - PCR 7: stable until Secure Boot key rotation. Good anchor.
> - PCR 11: changes when an update changes measured UKI content. Forward
>   sealing handles those new states (Module 2).

### 4.3 Secure Boot vs TPM: Distinct Responsibilities

These two mechanisms are often confused. They are complementary, not redundant.

| | Secure Boot | TPM |
|--|-------------|-----|
| **Purpose** | Verification | Measurement |
| **Mechanism** | Signature + certificate check | Hash extension into registers |
| **Question answered** | "Is this binary trusted to run?" | "What actually ran, and in what order?" |
| **Acts on** | Pre-execution | During and after execution |
| **Enforces** | What is allowed to boot | What state the system is in |

> [!IMPORTANT]
> **The Combined Guarantee**
> Secure Boot ensures only your signed UKI can execute. TPM records exactly what that UKI contained. Together they mean: the disk only unlocks when your specific, unmodified, signed software ran.

---

## 5. Policy-Based Key Release (PLR)

UKI measurements are the foundation for gating access to the LUKS encryption key:

```
UKI boots
  │
  ▼
systemd-stub measures sections → PCR 11
  │
  ▼
PCR 11 value represents the exact UKI that ran
  │
  ▼
Policy engine checks, is this PCR state approved?
  │
  ├─ Yes → TPM releases sealed secret → disk unlocks
  └─ No  → disk remains inaccessible
```

If anything in the UKI was modified: kernel, initramfs, cmdline, or any other section: PCR 11 will differ from the approved value, and the key will not be released.

### 5.1 What Happens if the initramfs Is Compromised?

A compromised initramfs is dangerous precisely because it runs inside the trusted boot boundary: as root, before any security mechanisms are active.

**Without UKI:** An attacker who replaces the initramfs can intercept disk decryption, redirect the root filesystem, disable policy enforcement, and implant persistent backdoors. If the attacker also signs the modified initramfs, Secure Boot accepts it. The TPM measures it. Policies match. Secrets are released. The system is compromised but appears trusted.

**With UKI:** The initramfs is sealed inside the signed binary. Replacing it invalidates the Secure Boot signature and changes PCR 11. The system cannot boot with the modified image, and the disk key is never released.

---

## 6. Summary

```
systemd-stub   → measures UKI sections into PCR 11
TPM            → stores the resulting PCR values (passively)
Secure Boot    → verifies the UKI signature before execution
Policy Engine  → reads PCRs, decides whether to release the disk key
```

The UKI is not just a packaging convenience. It is the mechanism that makes the system's identity deterministic and tamper-evident: the prerequisite for everything in Modules 2, 3, and 4.

---

## Related Notes

- [Introduction: Architecture Overview](00_Introduction.md)
- [Module 2: Forward Sealing](02_Forward_Sealing.md)
- [Module 3: Disk Encryption and Policy Enforcement](03_Disk_Encryption_and_Policy_Enforcement.md)
