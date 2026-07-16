<!--
PUBLIC RELEASE VERSION: sanitized for publication.
Machine-id, IP addresses (RFC 5737 documentation range), hostnames, and the
LUKS UUID are format-valid EXAMPLE values, not the original lab identifiers.
All PCR values, policy-key digests, and script hashes are genuine measurements
from the validated lab build and are safe to publish (public values by design).
-->

---
tags:
  - trusted-boot
  - golden-path
  - reproducible-rebuild
module: 6
category: golden-path
status: stable
companion-state: 00_Current_Project_State.md
companion-findings: 06F_Diagnostic_Findings_Catalog.md
companion-evidence:
  - 06_Lab_Setup_Runbook.md
  - 06_Lab_Setup_Runbook_continuation.md
  - 06_Lab_Setup_Runbook_continuation_v2_1_patches.md
  - 06_Lab_Setup_Runbook_continuation_v3.md
last-runbook-validation: 2026-05-28
last-operational-validation: 2026-06-19
current-source-regression: pending-live-run
---

# 06B: Golden Path Trusted Boot Lab Rebuild Runbook

**Purpose:** reference rebuild of the validated trusted boot lab.

**How to use this document:**

- Follow steps in order. Do not skip snapshots.
- Each step has a stop condition. If it triggers, halt and consult the linked finding in `06F_Diagnostic_Findings_Catalog.md` or the original evidence file.
- The gate sequence through Step 39 was validated against VM 500 (`tboot-lab`).
- For the why behind a step, the matching forensic finding, or the deprecation rule, see `06F_Diagnostic_Findings_Catalog.md`.
- For the chronological discovery context, the four `06_Lab_Setup_Runbook*.md` files are still around. They are kept while the migration to `06F` is being verified, and will be archived once the team agrees the migration is complete.

> [!important] What this document is not
> This document does not explain why each step is the way it is, what was tried before, or what failed during development. For that, see `06F_Diagnostic_Findings_Catalog.md` or the legacy evidence files. This document only contains the current validated path.

> [!important] Public reproduction boundary
> The repository contains the byte-pinned implementation and the validated gate
> sequence. It does not contain the live Proxmox configuration, snapshot lineage
> or current-state ledger used by the original lab. Replace references to
> `proxmox_technical_docs_v13.md` and `00_Current_Project_State.md` with
> equivalent site-specific values before running a gate. Do not treat example
> identifiers or historical hashes as values for a different machine.

> [!note] Validation after the recorded runbook session
> Steps 1 through 39 record the gate state as of 28 May 2026. On 19 June 2026,
> the same system completed an unrestricted `dnf update`, including a kernel
> version upgrade, and rebooted with automatic TPM unlock and matching runtime
> PCR 11. The raw transcript from that later update is not included. Statements
> such as "still gated" inside earlier steps describe the historical state at
> that gate and are superseded by this later operational validation.

> [!note] Helper scripts policy
> The commands in this runbook are the canonical rebuild path and are intentionally inlined for self-contained execution. The scripts under `scripts/` (`predict_pcr11_from_uki.sh`, `validate_dnf_kernel_reinstall.sh`, `show_trusted_boot_result.sh`) are optional convenience wrappers for repeated validation and ongoing operations after the rebuild procedure is understood. They are not required to execute the golden path. See `scripts/README.md` for usage.

> [!note] Where things live (routing model)
> This file is the operational flight checklist and the self-contained rebuild path. Supporting files own the explanation:
> - `06F_Diagnostic_Findings_Catalog.md`: failures, forbidden procedures, harness bugs, deprecations, forensic findings.
> - `05_Update_Workflows_and_Key_Storage.md`: architecture, update workflow, trust model, signing workflow.
> - `06C_Golden_Path_Operator_Notes.md`: operator guidance, methodology and rationale.
>
> Dangerous-mistake guards (what bricks or locks the system, when to stop, when rollback is required, which snapshot protects the current cliff edge) stay **here**, inline with the step they protect.

---

## Per-step format

Every step uses this structure. The bold-field header form below is the single conforming form: steps do not use `###`-heading fields and do not omit the header.

```
## Step N: Short operational title

**Goal:** Concise operational explanation. 2–5 lines maximum: enough to know what the step accomplishes and why it is ordered here, without theory. Deep rationale lives in 06C / 05 / 06F.
**Prerequisites:** Exact prior step/gate requirements (closed gates, rollback anchors, frozen shas this step depends on).
**Where to run:** Proxmox host pve-host / Fedora VM 500 as root / mixed.

### Commands
[bash; comments only where they prevent an operator mistake]

### Expected output
[concrete output or a `| Check | Expected result |` table]

### Stop condition
[each halt condition says what to do or where to triage]

### Snapshot point
[snapshot name + description template, or "None."]
```

**Expected-output rule:** prefer actual validated output or a check/result table over prose. Where a value is machine-specific use placeholders `<KVER>`, `<MID>`, `<sha256>`, `<PCR7>`, `<PCR11>`, `<snapshot-name>`. Never invent exact output; if it is not recoverable, mark it as a pattern with a TODO to recapture.

**Sha-pinned sources:** several steps inline a script via heredoc whose sha256 is a validated invariant (the hook, the B.2.2 helper, the PCR predictor, the B.2.3 decider, the B.4 helper, the B.4 decider). Those heredoc bodies are byte-exact: never edit them, including their comments.

---

# Phase 1: lab environment and custom Secure Boot ownership

## Step 1: Proxmox host prerequisites

**Goal:** host is configured per existing Proxmox infrastructure with LVM thin pool for VM storage.
**Prerequisites:** existing Proxmox VE 8.x; `proxmox_technical_docs_v13.md` host setup applied.
**Where to run:** Proxmox host `pve-host` as `root`.

### Commands

```bash
# Run on Proxmox host pve-host
pvesm status | grep vm-storage
ip -br addr show vmbr1
iptables -t raw -L PREROUTING | grep 'CT zone'
lvs --units g vm-storage/vm-thin
```

### Expected output

- `vm-storage  lvmthin  active  ...` (thin pool exists)
- `vmbr1  UP  192.0.2.1/24` (private bridge configured)
- An iptables rule like `CT zone 1` (conntrack zone fix applied, so the VM-level firewall coexists with NAT)
- Thin pool `Data%` < 75%

### Stop condition

Any of the above missing: halt. See `proxmox_technical_docs_v13.md` for host preparation. Do not proceed without the conntrack zone fix; without it, VM firewall will silently drop NAT traffic.

### Snapshot point

None, host-side prep, not VM state.

---

## Step 2: Create VM 500 (`tboot-lab`)

**Goal:** Empty VM with OVMF firmware in Setup Mode, TPM 2.0 attached, three disks (scsi0/efidisk0/tpmstate0).
**Prerequisites:** Step 1.
**Where to run:** Proxmox host `pve-host` as `root`.

### Commands

```bash
# Run on Proxmox host
qm create 500 \
  --name tboot-lab \
  --machine q35 \
  --bios ovmf \
  --efidisk0 vm-storage:1,efitype=4m,pre-enrolled-keys=0 \
  --tpmstate0 vm-storage:1,version=v2.0 \
  --scsi0 vm-storage:40,iothread=1 \
  --scsihw virtio-scsi-single \
  --cpu x86-64-v2-AES \
  --cores 2 \
  --memory 4096 \
  --balloon 0 \
  --net0 virtio,bridge=vmbr1 \
  --ide2 local:iso/Fedora-Server-dvd-x86_64-43-1.<n>.iso,media=cdrom \
  --boot order='ide2;scsi0;net0'
```

### Expected output

```bash
qm config 500
# Should show:
#   bios: ovmf
#   efidisk0: vm-storage:vm-500-disk-0,efitype=4m,pre-enrolled-keys=0,size=4M
#   tpmstate0: vm-storage:vm-500-disk-2,size=4M,version=v2.0
#   scsi0: vm-storage:vm-500-disk-1,iothread=1,size=40G
#   cpu: x86-64-v2-AES
```

### Stop condition

- `pre-enrolled-keys=1`. Setup Mode will not work; recreate efidisk with `pre-enrolled-keys=0`.
- `cpu: host`: kernel panic in Contabo's nested virtualisation; use `x86-64-v2-AES`. See `06F` finding A.2.
- No `tpmstate0`. TPM2 unseal will fail later; add the disk.

### Snapshot point

None yet.

---

## Step 3: Install Fedora Server 43 (minimal)

**Goal:** Fedora Server 43 installed with minimal package set, root password set, no automatic firmware enrollment.
**Prerequisites:** Step 2.
**Where to run:** Mixed: Fedora installer console, Proxmox host for ISO eject/reboot, then Fedora VM as `root` after installation.

### Commands

Boot the VM and run through the Fedora installer:

```
Installation Profile:    Fedora Server Edition (minimal)
Disk Layout:             Mandatory encrypted root layout
                         /boot/efi (vfat, 1 GB, ESP)
                         /boot (ext4, 1 GB, unencrypted)
                         / (xfs/ext4 inside LUKS2, remainder)
Swap:                    optional, but if present should not weaken the LUKS model
Root password:           set (record it)
User:                    optional admin user
Network:                 DHCP from vmbr1 (192.0.2.50–.150 range)
Timezone:                set
Software:                Fedora Server Edition (default)
```

After install completes, eject the ISO and reboot:

```bash
# Run on Proxmox host
qm set 500 --ide2 none,media=cdrom
qm reboot 500
```

Then update the system inside the VM:

```bash
# Run inside VM as root
dnf upgrade --refresh -y
reboot
```

### Expected output

- VM boots into Fedora 43 minimal
- `uname -r` reports a kernel in the 6.x range
- `cat /etc/os-release` reports `NAME="Fedora Linux"` and `VERSION_ID=43`
- `bootctl status` reports `Secure Boot: disabled (setup)` (Setup Mode active)

### Stop condition

- `Secure Boot: enabled (deployed)`: efidisk has factory keys; recreate with `pre-enrolled-keys=0`.
- Root is not backed by LUKS2, reinstall before continuing. Module 3 depends on LUKS2 and the golden path assumes it from the start.
- Network not working, check DHCP on vmbr1, conntrack zone fix per Step 1.

### Snapshot point

None yet. Step 4 is a continuation of base setup.

---

## Step 4: Install tooling and configure workspace

**Goal:** Install Phase 1–2 tooling; configure tmux for long-running sessions.
**Prerequisites:** Step 3.
**Where to run:** Fedora VM as `root`.

### Commands

```bash
# Inside VM as root
dnf install -y \
  systemd-ukify \
  sbsigntools \
  efitools \
  efibootmgr \
  binutils \
  pesign \
  tmux \
  ShellCheck

# tmux config for long-running sessions
sudo tee /root/.tmux.conf <<'EOF'
set-option -g mouse on
set-option -g history-limit 50000
EOF

# Project state/artifact directories used by later dynamic validation steps
install -d -m 0700 /root/tboot-lab/state /root/tboot-lab/artifacts /root/tboot-lab/failed-ukis
```

### Expected output

```bash
which ukify sbsign efi-updatevar efibootmgr objcopy pesign tmux shellcheck
# All present, in /usr/bin/ or /usr/sbin/

systemd-ukify --version | head -1
# systemd 258.x.fc43 or newer
```

### Stop condition

Any tool missing. `dnf install` again or check repo configuration.

### Snapshot point

None yet. Step 5 is the first snapshot.

---

## Step 5: First snapshot (`post-install`)

**Goal:** Capture the clean Fedora install state as a rollback target.
**Prerequisites:** Step 4 complete.
**Where to run:** Proxmox host `pve-host` as `root`.

### Commands

```bash
# Run on Proxmox host
qm snapshot 500 post-install \
  --description "Fedora 43 Server, fully updated, Phase 1-2 tooling installed (systemd-ukify, sbsigntools, efitools, efibootmgr, binutils, pesign, tmux, ShellCheck), OVMF in Setup Mode, no project trust-chain config applied yet"

qm listsnapshot 500
```

### Expected output

```
post-install                   Fedora 43 Server, fully updated...
You are here!
```

All three disks (`scsi0`, `efidisk0`, `tpmstate0`) captured.

### Stop condition

- `qm snapshot` exit non-zero: check thin pool capacity (`Data%` should be < 75%).

### Snapshot point

✅ `post-install` taken.

---

## Step 6: Generate PK/KEK/db keys

**Goal:** Generate custom Secure Boot key hierarchy in `/etc/uefi-keys/`. Keys remain on disk only; not yet enrolled in OVMF.
**Prerequisites:** Step 5.
**Where to run:** Fedora VM as `root` in tmux; snapshot command at the end runs on Proxmox host.

### Commands

```bash
# Inside VM as root, in tmux session 'tboot'
sudo -i
tmux new-session -s tboot

mkdir -p /etc/uefi-keys
chmod 0700 /etc/uefi-keys
chown root:root /etc/uefi-keys
cd /etc/uefi-keys

# Generate three keypairs (PK, KEK, db). RSA-2048, 10-year validity
for k in PK KEK db; do
  CN="tboot-lab Platform Key"
  [[ "$k" == "KEK" ]] && CN="tboot-lab Key Exchange Key"
  [[ "$k" == "db"  ]] && CN="tboot-lab Signature Database Key"

  openssl req -newkey rsa:2048 -nodes \
    -keyout "${k}.key" \
    -new -x509 -sha256 -days 3650 \
    -subj "/CN=${CN}/" \
    -out "${k}.crt"
done

# Convert each .crt to ESL format (UEFI signature list)
GUID="$(uuidgen)"
echo "$GUID" > GUID.txt

for k in PK KEK db; do
  cert-to-efi-sig-list -g "$GUID" "${k}.crt" "${k}.esl"
done

# Sign each ESL with the appropriate higher-tier key:
#   PK  signs PK   (self-signed)
#   PK  signs KEK
#   KEK signs db
sign-efi-sig-list -g "$GUID" -k PK.key  -c PK.crt  PK  PK.esl  PK.auth
sign-efi-sig-list -g "$GUID" -k PK.key  -c PK.crt  KEK KEK.esl KEK.auth
sign-efi-sig-list -g "$GUID" -k KEK.key -c KEK.crt db  db.esl  db.auth

ls -la /etc/uefi-keys/
```

### Expected output

13 files in `/etc/uefi-keys/`:
- `PK.key`, `PK.crt`, `PK.esl`, `PK.auth`
- `KEK.key`, `KEK.crt`, `KEK.esl`, `KEK.auth`
- `db.key`, `db.crt`, `db.esl`, `db.auth`
- `GUID.txt`

### Stop condition

- Any signing step fails, check that the previous step produced a valid `.crt` and `.esl`.
- `cert-to-efi-sig-list` not found. `dnf install efitools`.

### Snapshot point

```bash
# Run on Proxmox host
qm snapshot 500 pre-enrollment \
  --description "Phase 1 keys generated and signed. /etc/uefi-keys/ contains PK/KEK/db .key, .crt, .esl, .auth files. All auth files structurally verified. OVMF still in Setup Mode."
```

✅ `pre-enrollment` taken.

---

## Step 7: Configure kernel-install for UKI layout

**Goal:** Configure kernel-install to build Unified Kernel Images via `ukify`, with correct config that includes both Secure Boot signing and PCR signing slots.
**Prerequisites:** Step 6.
**Where to run:** Fedora VM as `root`.

> [!note] Why this differs from `06_Lab_Setup_Runbook.md` §12 Stage A
> The original §12 used a `[Signing]` section that ukify silently ignores (`06F` finding B.4). This step uses the corrected `[UKI]` form documented in `06_Lab_Setup_Runbook_continuation.md` §16.2 and §18.2, and adds `BOOT_ROOT=/boot/efi` per the v3 continuation §20.1. The `[PCRSignature:initial]` block is included from the start so that future hook activations don't have to reshape the config.

### Commands

```bash
# Inside VM as root

# /etc/kernel/install.conf, tells kernel-install to use UKI layout, BOOT_ROOT=/boot/efi
cat > /etc/kernel/install.conf <<'EOF'
layout=uki
uki_generator=ukify
BOOT_ROOT=/boot/efi
EOF

# /etc/kernel/cmdline, kernel command line for UKI .cmdline section
# Replace UUIDs with the actual values for your filesystem and LUKS volume
ROOT_UUID="$(findmnt -no UUID /)"
LUKS_UUID="$(blkid | awk -F'"' '/crypto_LUKS/ {print $2; exit}')"

if [[ -z "$ROOT_UUID" || -z "$LUKS_UUID" ]]; then
  echo "Missing ROOT_UUID or LUKS_UUID. The golden path requires LUKS2 root from Step 3." >&2
  exit 1
fi

cat > /etc/kernel/cmdline <<EOF
root=UUID=${ROOT_UUID} ro rd.luks.uuid=luks-${LUKS_UUID} quiet
EOF

# /etc/kernel/uki.conf, corrected form with [UKI] + [PCRSignature:initial]
cat > /etc/kernel/uki.conf <<'EOF'
[UKI]
Cmdline=@/etc/kernel/cmdline
OSRelease=@/etc/os-release
SecureBootPrivateKey=/etc/uefi-keys/db.key
SecureBootCertificate=/etc/uefi-keys/db.crt

[PCRSignature:initial]
PCRPublicKey=/etc/systemd/tpm2-pcr-public-key.pem
EOF

# Verify
cat /etc/kernel/install.conf
echo "---"
cat /etc/kernel/cmdline
echo "---"
cat /etc/kernel/uki.conf
```

### Expected output

- Three config files in `/etc/kernel/` with the contents above.
- `cmdline` contains both the actual root filesystem UUID and the LUKS UUID.

### Stop condition

- `findmnt` returns empty UUID: root filesystem not on a block device with a UUID; investigate disk layout.
- No LUKS UUID is found, reinstall or correct disk layout before continuing.
- `[Signing]` appears anywhere in `uki.conf`: that's the deprecated form (`06F` finding B.4); rewrite as shown.

### Snapshot point

None yet. Step 8 builds the UKI before snapshotting.

---

## Step 8: Build initial signed UKI (Phase 1 fallback)

**Goal:** Build a signed UKI for the running kernel and place it on the ESP at the bare-kver path. This UKI does not yet have `.pcrsig` (Block A and the hook produce that later); it serves as the Phase 1 fallback.
**Prerequisites:** Step 7.
**Where to run:** Fedora VM as `root`.

### Commands

```bash
# Inside VM as root
KVER="$(uname -r)"
mkdir -p /boot/efi/EFI/Linux

ukify build \
  --linux=/boot/vmlinuz-${KVER} \
  --initrd=/boot/initramfs-${KVER}.img \
  --cmdline=@/etc/kernel/cmdline \
  --os-release=@/etc/os-release \
  --secureboot-private-key=/etc/uefi-keys/db.key \
  --secureboot-certificate=/etc/uefi-keys/db.crt \
  --output=/boot/efi/EFI/Linux/${KVER}.efi

ls -la /boot/efi/EFI/Linux/${KVER}.efi
sbverify --cert /etc/uefi-keys/db.crt /boot/efi/EFI/Linux/${KVER}.efi
```

### Expected output

```
/boot/efi/EFI/Linux/<KVER>.efi present, ~50 MB
Signature verification OK
```

### Stop condition

- `sbverify` fails: check `/etc/uefi-keys/db.crt` is the cert that matches `db.key`.
- `ukify build` fails: read its stderr; usually missing input file (vmlinuz, initramfs) or unreadable key.

### Snapshot point

None yet, combined snapshot after Step 9.

---

## Step 9: Install and sign systemd-boot

**Goal:** Place signed `systemd-bootx64.efi` on the ESP as the primary boot loader; configure `BootOrder` to launch it first.
**Prerequisites:** Step 8.
**Where to run:** Fedora VM as `root`; use Proxmox host only for the snapshot command at the end.

### Commands

```bash
# Inside VM as root

# Disable Fedora's auto-update of systemd-boot, we manage it ourselves under Secure Boot
systemctl disable --now systemd-boot-update.service

# Install systemd-boot to ESP
bootctl install

# Sign the installed systemd-boot binaries
for b in /boot/efi/EFI/systemd/systemd-bootx64.efi /boot/efi/EFI/BOOT/BOOTX64.EFI; do
  sbsign --key /etc/uefi-keys/db.key \
         --cert /etc/uefi-keys/db.crt \
         --output "${b}.signed" "$b"
  mv -f "${b}.signed" "$b"
done

# Verify both signed binaries
sbverify --cert /etc/uefi-keys/db.crt /boot/efi/EFI/systemd/systemd-bootx64.efi
sbverify --cert /etc/uefi-keys/db.crt /boot/efi/EFI/BOOT/BOOTX64.EFI

# Configure systemd-boot loader
cat > /boot/efi/loader/loader.conf <<'EOF'
default *
timeout 5
console-mode auto
editor no
EOF

# Confirm or create EFI boot entry for systemd-boot
if ! efibootmgr | grep -qi 'Linux Boot Manager'; then
  DISK=/dev/sda
  PART=1
  efibootmgr --create     --disk "$DISK"     --part "$PART"     --label 'Linux Boot Manager'     --loader '\EFI\systemd\systemd-bootx64.efi'
fi

# Put Linux Boot Manager first in BootOrder if needed
efibootmgr
bootctl status
```

### Expected output

- Both `systemd-bootx64.efi` and `BOOTX64.EFI` present on ESP, signed (`Signature verification OK` for both)
- `bootctl status` shows the UKI from Step 8 as a discoverable Type #2 boot loader entry
- `efibootmgr` shows `Boot0002* Linux Boot Manager` (or similar) first in `BootOrder`

### Stop condition

- `sbverify` fails on either binary: re-run sbsign.
- `bootctl status` doesn't list the UKI: check the UKI is at `/boot/efi/EFI/Linux/${KVER}.efi`.

### Snapshot point

```bash
# Run on Proxmox host
qm snapshot 500 pre-enrollment-v2 \
  --description "Phase 1 attempt 2, Stages A-D complete: UKI built and signed at /boot/efi/EFI/Linux/${KVER}.efi (db.crt signed). systemd-boot installed and signed at /EFI/systemd/systemd-bootx64.efi and /EFI/BOOT/BOOTX64.EFI. loader.conf written. EFI BootOrder configured. systemd-boot-update.service disabled. /etc/uefi-keys/ intact. OVMF still in Setup Mode, no firmware enrollment yet. bootctl recognizes UKI as Default Boot Loader Entry Type #2."
```

✅ `pre-enrollment-v2` taken.

---

## Step 10: Test boot under Setup Mode (insurance)

**Goal:** verify the discovery chain (OVMF, systemd-boot, UKI) works *before* enrolling keys. Cheap insurance, per `06_Lab_Setup_Runbook.md` §16.4.
**Prerequisites:** Step 9.
**Where to run:** Fedora VM as `root` before and after reboot.

### Commands

```bash
# Inside VM as root
sync
sync
reboot
```

After reboot, log back in and verify:

```bash
bootctl status | grep -E 'Secure Boot|Current Entry|Default Entry'
```

### Expected output

```
Secure Boot: disabled (setup)        ← still in Setup Mode (no enrollment yet)
Current Entry: <KVER>.efi            ← UKI loaded successfully
Default Entry: <KVER>.efi
```

### Stop condition

- VM doesn't boot, reset via Proxmox; rollback to `pre-enrollment-v2` and triage.
- Boots but `Current Entry` is wrong. `BootOrder` issue; re-run efibootmgr.

### Snapshot point

None. Step 11 is the next snapshot.

---

## Step 11: Enroll PK/KEK/db in OVMF, reboot to enforcing

**Goal:** Write the three signed `.auth` blobs into OVMF's firmware variables. After this, OVMF transitions from Setup Mode to User Mode on next boot, and Secure Boot becomes enforcing under our keys.
**Prerequisites:** Step 10.
**Where to run:** Mixed: Fedora VM as `root` for enrollment/reboot/verification; Proxmox host for snapshots.

### Commands

```bash
# Inside VM as root
cd /etc/uefi-keys

# Phase 1: enroll db and KEK. These are reversible while still in Setup Mode.
efi-updatevar -f db.auth db
efi-updatevar -f KEK.auth KEK

# Verify db and KEK landed correctly. Setup Mode is still active here, so a
# bad enrollment can be rolled back by `efi-updatevar -d` or by Proxmox snapshot.
efi-readvar -v db
efi-readvar -v KEK
strings /sys/firmware/efi/efivars/db-* 2>/dev/null | grep tboot-lab
strings /sys/firmware/efi/efivars/KEK-* 2>/dev/null | grep tboot-lab

# Phase 2: enroll PK. This is the cliff edge. Writing PK exits Setup Mode and
# the NVRAM becomes append-only (further updates require auth blobs signed by
# the appropriate higher-tier key). Do this LAST, and only after the
# verification above has passed.
efi-updatevar -f PK.auth PK

# Final verification across all three slots
bootctl status | grep -i 'secure boot'
efi-readvar -v PK
efi-readvar -v KEK
efi-readvar -v db
```

> [!important] Why the verification gate sits between KEK and PK
> The PK write is what flips `SetupMode` from `01` to `00`. Once in User Mode, further updates require signed auth blobs and the recovery path is much narrower. Inserting the verification step *between* KEK and PK is what makes enrollment recoverable: if the auth blobs have a structural problem that only firmware notices on commit, you find out before the one-way door closes. See `06F` finding A.4 for why `efi-readvar`'s output looks empty under a careless grep, and the `strings` fallback against the raw efivar bytes.

### Expected output

- All three `efi-updatevar` invocations exit 0
- After the PK enrollment, OVMF's NVRAM holds our trust hierarchy

```bash
# Snapshot before rebooting (insurance, see 06_Lab_Setup_Runbook.md §16.4)
# Run on Proxmox host
qm snapshot 500 enrolled-pre-reboot \
  --description "Phase 1 attempt 2: PK/KEK/db enrolled in OVMF, Setup Mode exit pending. Trust hierarchy committed: PK 855B, KEK 863B, db 875B with our certs. Boot path: OVMF → Boot0002 (Linux Boot Manager) → signed systemd-boot → signed UKI. /etc/uefi-keys/ retains all 13 files. Secure Boot enforcement ARMED but not yet active (current boot pre-dates PK write). Recovery point if next reboot fails under enforcement."
```

Then reboot:

```bash
sync
reboot
```

After reboot:

```bash
bootctl status
cat /sys/class/tpm/tpm0/pcr-sha256/7
cat /sys/class/tpm/tpm0/pcr-sha256/11
```

### Expected output

```
Secure Boot: enabled (user)          ← enforcing under our keys
TPM2 Support: yes
Measured UKI: yes
Current Entry: <KVER>.efi
```

PCR 7 and PCR 11 will have specific values dependent on the rebuild. Record them for the next snapshot description.

### Stop condition

- `Secure Boot: disabled` after reboot: enrollment didn't take; rollback to `enrolled-pre-reboot` and re-run efi-updatevar.
- VM doesn't boot at all, fallback chain works (Phase 1 UKI is still db-signed, systemd-boot is db-signed); if it doesn't, rollback to `pre-enrollment-v2`.

### Snapshot point

```bash
# Run on Proxmox host, AFTER successful boot under enforcement
PCR7="$(ssh root@192.0.2.40 'cat /sys/class/tpm/tpm0/pcr-sha256/7')"
PCR11="$(ssh root@192.0.2.40 'cat /sys/class/tpm/tpm0/pcr-sha256/11')"

qm snapshot 500 phase-1-complete \
  --description "Phase 1 complete. Secure Boot enforcing under custom PK/KEK/db. Boot path: OVMF → Boot0002 (Linux Boot Manager) → signed systemd-boot → signed UKI. systemd-stub measured UKI sections into PCR 11. Reference PCR values: PCR 7 = ${PCR7}, PCR 11 = ${PCR11}. ESP UKI 53MB at /boot/efi/EFI/Linux/<KVER>.efi (Phase 1 manual UKI, no .pcrsig). systemd-boot-update.service disabled."
```

✅ `phase-1-complete` taken.

---

# Block A: Policy Keypair, TPM2 Sealing

## Step 12: Generate policy keypair and TPM-seal private key

**Goal:** Generate the RSA policy keypair that will sign PCR predictions, then seal the private key into a TPM2 credential so it only releases when PCR 7 matches the seal-time value.
**Prerequisites:** Step 11.
**Where to run:** Fedora VM as `root`; Proxmox host for the optional/required snapshots shown in comments.

### Commands

```bash
# Inside VM as root

# Generate the policy keypair
mkdir -p /etc/systemd
chmod 0755 /etc/systemd
openssl genrsa -out /etc/systemd/tpm2-pcr-private-key.pem 3072
openssl rsa -in /etc/systemd/tpm2-pcr-private-key.pem -pubout \
  -out /etc/systemd/tpm2-pcr-public-key.pem

chmod 0600 /etc/systemd/tpm2-pcr-private-key.pem
chmod 0644 /etc/systemd/tpm2-pcr-public-key.pem

# Capture pubkey sha256 (this is the architectural invariant for forward sealing)
sha256sum /etc/systemd/tpm2-pcr-public-key.pem
PRIVKEY_SHA="$(sha256sum /etc/systemd/tpm2-pcr-private-key.pem | awk '{print $1}')"

# Snapshot before sealing (in case sealing or shred goes wrong)
# Run on Proxmox host:
#   qm snapshot 500 module2-prelude-keys-plaintext \
#     --description "Stage A.1 complete. Policy keypair generated at /etc/systemd/tpm2-pcr-{private,public}-key.pem (RSA-3072). Private key plaintext on disk. No TPM sealing yet."
```

> [!warning] systemd-creds auto-discovery
> If `/etc/systemd/tpm2-pcr-public-key.pem` is present, `systemd-creds encrypt` silently treats the operation as if `--tpm2-public-key=...` were specified and embeds a forward-sealed PCR policy in the credential. That is not what we want here: the credential should be bound only to PCR 7 (Secure Boot policy), not forward-sealed. The workaround is to park the public key out of `/etc/systemd/` during encrypt, then restore it. See `06F` finding C.1, with original context in `06_Lab_Setup_Runbook_continuation.md` §16.7.

```bash
# Park public key during encrypt
mv /etc/systemd/tpm2-pcr-public-key.pem /tmp/pubkey-park.pem

# Seal the private key to PCR 7 (no forward-seal embedded)
systemd-creds encrypt \
  --tpm2-pcrs=7 \
  --name=tpm2-pcr-signing-key \
  /etc/systemd/tpm2-pcr-private-key.pem \
  /etc/systemd/tpm2-pcr-private-key.pem.enc

# Restore public key
mv /tmp/pubkey-park.pem /etc/systemd/tpm2-pcr-public-key.pem

ls -la /etc/systemd/tpm2-pcr-private-key.pem.enc
chmod 0600 /etc/systemd/tpm2-pcr-private-key.pem.enc

# Round-trip verify: decrypt and confirm sha matches plaintext
systemd-creds decrypt \
  --with-key=tpm2 \
  --name=tpm2-pcr-signing-key \
  /etc/systemd/tpm2-pcr-private-key.pem.enc \
  /tmp/decrypted.pem

DECRYPTED_SHA="$(sha256sum /tmp/decrypted.pem | awk '{print $1}')"
if [[ "$DECRYPTED_SHA" == "$PRIVKEY_SHA" ]]; then
  echo "✓ TPM2 unseal round-trip verified"
else
  echo "✗ Decrypted sha differs from plaintext. DO NOT shred plaintext"
  exit 1
fi

# Shred plaintext private key
shred -u /etc/systemd/tpm2-pcr-private-key.pem
shred -u /tmp/decrypted.pem

# Confirm only the encrypted form remains
ls -la /etc/systemd/tpm2-pcr-private-key.pem*
```

### Expected output

- `/etc/systemd/tpm2-pcr-private-key.pem.enc` exists, mode 0600, ~3.9 KB
- `/etc/systemd/tpm2-pcr-public-key.pem` exists, mode 0644, ~625 B
- `/etc/systemd/tpm2-pcr-private-key.pem` (plaintext) is **gone**
- Round-trip verification reports `✓`

### Stop condition

- Round-trip verification fails (`✗`). DO NOT shred the plaintext; investigate why decrypt produces different bytes (likely a sealing parameter mismatch).
- `systemd-creds encrypt` fails. `tpm2-tools` may not be installed, or TPM is busy/in failure mode.

### Snapshot point

```bash
# Run on Proxmox host
qm snapshot 500 module2-prelude-complete \
  --description "Block A complete. Policy keypair generated and TPM-sealed. Trust anchor: RSA-3072. Public key at /etc/systemd/tpm2-pcr-public-key.pem (sha256 <PUBKEY_SHA>). Private key TPM-sealed to PCR 7 at /etc/systemd/tpm2-pcr-private-key.pem.enc. Plaintext shredded. Round-trip verified. Seal bound to PCR 7 = <PCR7>. Note: A.2 sealing required parking the public key out of /etc/systemd/ during encrypt to prevent systemd-creds auto-discovery from forward-sealing the credential. See 06F finding C.1."
```

✅ `module2-prelude-complete` taken.

---

# Block B: Kernel-Install Signing Hook

## Step 13: Install dracut TPM2 userspace prerequisite

**Goal:** Add `tpm2-tools` package and `tpm2-tss` dracut module so the initramfs can perform TPM2 operations during boot. Required for the full phase chain (`enter-initrd → leave-initrd → sysinit → ready`) to extend PCR 11 correctly.
**Prerequisites:** Step 12.
**Where to run:** Fedora VM as `root`.

### Commands

```bash
# Inside VM as root

# Install tpm2-tools (provides tpm2_pcrread, tpm2_pcrextend, etc.)
dnf install -y tpm2-tools

# Configure dracut to include tpm2-tss in initramfs
cat > /etc/dracut.conf.d/90-tboot-tpm2.conf <<'EOF'
add_dracutmodules+=" tpm2-tss "
EOF

# Regenerate initramfs for the running kernel
KVER="$(uname -r)"
dracut --force /boot/initramfs-${KVER}.img ${KVER}

# Verify tpm2-tss userspace is in the new initramfs
lsinitrd /boot/initramfs-${KVER}.img | grep -E 'tpm2|libtss2' | head -5
```

### Expected output

- `tpm2-tools-5.x` installed
- `/etc/dracut.conf.d/90-tboot-tpm2.conf` written
- New initramfs built (~38 MB, larger than before due to TPM2 userspace)
- `lsinitrd` shows `tpm2-tss`, `libtss2-*`, `tpm2`, `tpm2_pcrread`, `tpm2_pcrextend` present

### Stop condition

- `dracut` fails: read stderr; module dependencies missing.
- `lsinitrd` doesn't show TPM2 binaries: config file syntax error or wrong dracut module name.

### Snapshot point

None yet. Step 14 covers hook lint, install, and snapshot together.

---

## Step 14: Write, lint, and dry-run-validate the hook

**Goal:** Write the validated v2.1.2 80-tpm2-sign hook to `/tmp`, ShellCheck it, run a synthetic dry-run against `60-ukify.install` output to confirm it produces a UKI with `.pcrsig` matching expectations.
**Prerequisites:** Step 13.
**Where to run:** Fedora VM as `root`.

> [!important] Source of the hook
> The hook source below is the v2.1.2 form: ukify-native PCR signing, atomic-replace stage 9, SC2317/SC2329 ShellCheck disable directive on `_cleanup`. Sha256 = `5857e51d5551e05a7ca384f71529ec2fef07f5a6dbe40df70c6bbe49d720947e`.
> Do not use the objcopy form from `06_Lab_Setup_Runbook_continuation.md` §18.3. See `06F` finding B.1 for why.

### Commands: write the hook

```bash
# Inside VM as root
tee /tmp/80-tpm2-sign.install <<'HOOK_EOF'
#!/bin/bash
#
# /etc/kernel/install.d/80-tpm2-sign.install
#
# Forward-seal kernel-install-built UKIs by rebuilding the staged UKI via
# `ukify build` with native PCR signing flags. Operates entirely on
# $KERNEL_INSTALL_STAGING_AREA and runs before 90-uki-copy.install: on any
# failure, the ESP is never touched.
#
# References:
#   Module 5 §2.2          (TPM-sealed signing key)
#   Module 5 §3.2          (kernel-install hook design: updated)
#   Runbook §16.1   (corr) (objcopy unsafe for PE+ construction)
#   Runbook §16.13  (corr) (objcopy --add-section produces firmware-rejected UKI)
#   Runbook §16.17         (ukify build native PCR signing canonical)
#   Runbook §16.18         (tpm2-tools required for dracut tpm2-tss)
#   Runbook §16.19         (initramfs TPM2 userspace required for full phase chain)
#   Runbook §20            (Block B.3 live execution log)
#   Runbook §21            (this hook architecture)
#

set -euo pipefail

#============================================================================
# Constants
#============================================================================

readonly TAG="80-tpm2-sign"
readonly SCRATCH="/run/tboot-sign"
readonly UKI_NAME="uki.efi"
readonly DB_KEY="/etc/uefi-keys/db.key"
readonly DB_CRT="/etc/uefi-keys/db.crt"
readonly PCR_PUB="/etc/systemd/tpm2-pcr-public-key.pem"
readonly PCR_ENC="/etc/systemd/tpm2-pcr-private-key.pem.enc"
readonly CRED_NAME="tpm2-pcr-signing-key"

# Section-VMA sanity threshold: any section at VMA >= this is a §16.13-style
# layout corruption indicator. ukify-built UKIs always have all sections in
# a contiguous low-GB range (typically < 0x180000000).
readonly VMA_SANITY_THRESHOLD="0x180000000"

#============================================================================
# Logging helpers
#============================================================================

log()  { logger -t "$TAG" -- "$*"; printf '[%s] %s\n' "$TAG" "$*" >&2; }
info() { log "info: $*"; }
err()  { log "error: $*"; }

#============================================================================
# Cleanup trap: always shred scratch contents on exit
#============================================================================

# shellcheck disable=SC2317,SC2329  # invoked via trap below
_cleanup() {
    local rc=$?
    if [[ -d "$SCRATCH" ]]; then
        find "$SCRATCH" -type f -exec shred -u {} + 2>/dev/null || true
        rm -rf "$SCRATCH" 2>/dev/null || true
    fi
    exit "$rc"
}
trap _cleanup EXIT HUP INT TERM

#============================================================================
# Stage 1: Gate: only run for kernel-install add of UKI layout
#============================================================================

COMMAND="${1:-}"
KERNEL_VERSION="${2:-}"

# Only act on 'add' (not 'remove')
[[ "$COMMAND" == "add" ]] || exit 0

# Only act when kernel-install layout is uki
[[ "${KERNEL_INSTALL_LAYOUT:-}" == "uki" ]] || exit 0

# Only act when ukify is the configured UKI generator
[[ "${KERNEL_INSTALL_UKI_GENERATOR:-ukify}" == "ukify" ]] || exit 0

# Skip if kernel-install was invoked with an external UKI image (KERNEL_INSTALL_IMAGE_TYPE=uki)
# That path does not produce a staged UKI for us to enrich.
[[ "${KERNEL_INSTALL_IMAGE_TYPE:-}" != "uki" ]] || exit 0

# kernel-install must have given us a staging area
: "${KERNEL_INSTALL_STAGING_AREA:?KERNEL_INSTALL_STAGING_AREA must be set}"

# kernel-install must have given us a kernel version
[[ -n "$KERNEL_VERSION" ]] || { err "KERNEL_VERSION (\$2) empty"; exit 1; }

# Required input artefacts must exist
for f in "$DB_KEY" "$DB_CRT" "$PCR_PUB" "$PCR_ENC"; do
    [[ -f "$f" ]] || { err "required file missing: $f"; exit 1; }
done

# Source artefacts ukify will read from must exist
for f in "/lib/modules/${KERNEL_VERSION}/vmlinuz" \
         "/boot/initramfs-${KERNEL_VERSION}.img" \
         "/etc/kernel/cmdline" \
         "/etc/os-release"; do
    [[ -f "$f" ]] || { err "required source artefact missing: $f"; exit 1; }
done

info "starting for kernel ${KERNEL_VERSION}"

#============================================================================
# Stage 2: Scratch directory (tmpfs, 0700, root-only)
#============================================================================

if [[ -d "$SCRATCH" ]]; then
    find "$SCRATCH" -type f -exec shred -u {} + 2>/dev/null || true
    rm -rf "$SCRATCH"
fi
install -d -m 0700 -o root -g root "$SCRATCH"

# Confirm we're on tmpfs (defence in depth: if /run/tboot-sign somehow lands
# on persistent storage, we abort before any key material is written there)
SCRATCH_FSTYPE="$(stat -f -c '%T' "$SCRATCH")"
if [[ "$SCRATCH_FSTYPE" != "tmpfs" ]]; then
    err "scratch dir $SCRATCH is on $SCRATCH_FSTYPE, not tmpfs: aborting"
    exit 1
fi

#============================================================================
# Stage 3: Locate staged UKI; sanity-check 60-ukify's output
#============================================================================

STAGED_UKI="${KERNEL_INSTALL_STAGING_AREA}/${UKI_NAME}"
[[ -f "$STAGED_UKI" ]] || { err "staged UKI missing: $STAGED_UKI"; exit 1; }
[[ -s "$STAGED_UKI" ]] || { err "staged UKI empty: $STAGED_UKI"; exit 1; }

if ! sbverify --cert "$DB_CRT" "$STAGED_UKI" >/dev/null 2>&1; then
    err "60-ukify produced unsigned UKI: uki.conf likely missing [UKI] SecureBoot* keys"
    exit 1
fi

STAGED_SIZE="$(stat -c%s "$STAGED_UKI")"
info "60-ukify output present and signed (${STAGED_SIZE} bytes): will rebuild with .pcrsig"

#============================================================================
# Stage 5: Unseal policy private key (no stage 4: no extraction needed)
#============================================================================

info "unsealing policy private key (TPM2)"

if ! systemd-creds decrypt \
        --with-key=tpm2 \
        --name="$CRED_NAME" \
        "$PCR_ENC" \
        "$SCRATCH/private-key.pem"
then
    err "systemd-creds decrypt failed (TPM2 policy mismatch or cred corrupt)"
    exit 1
fi

chmod 0600 "$SCRATCH/private-key.pem"

if ! openssl rsa -in "$SCRATCH/private-key.pem" -check -noout >/dev/null 2>&1; then
    err "unsealed credential is not a valid RSA private key"
    exit 1
fi

info "policy private key unsealed and validated"

#============================================================================
# Stage 7: Rebuild UKI via ukify-native PCR signing (NEW: replaces old 4/6/7/8)
#============================================================================
#
# Old hook (§18.3):
#   stage 4: extract sections (.linux, .initrd, .pcrpkey) from staged UKI
#   stage 6: systemd-measure sign against extracted sections → .pcrsig
#   stage 7: objcopy --add-section .pcrsig=… into staged UKI    ← invalid (§16.13)
#   stage 8: sbsign re-sign
#
# New hook: ukify build does all four atomically with correct PE+ layout.
#

info "rebuilding UKI via ukify build with native PCR signing"

if ! ukify build \
        --linux="/lib/modules/${KERNEL_VERSION}/vmlinuz" \
        --initrd="/boot/initramfs-${KERNEL_VERSION}.img" \
        --cmdline=@/etc/kernel/cmdline \
        --os-release=@/etc/os-release \
        --pcr-private-key="$SCRATCH/private-key.pem" \
        --pcr-public-key="$PCR_PUB" \
        --phases=enter-initrd \
        --pcr-banks=sha256 \
        --secureboot-private-key="$DB_KEY" \
        --secureboot-certificate="$DB_CRT" \
        --output="$SCRATCH/built.efi" \
        2>"$SCRATCH/ukify.stderr"
then
    err "ukify build failed"
    if [[ -s "$SCRATCH/ukify.stderr" ]]; then
        while IFS= read -r line; do
            err "  ukify: $line"
        done < "$SCRATCH/ukify.stderr"
    fi
    exit 1
fi

# Shred private key as soon as ukify is done with it
shred -u "$SCRATCH/private-key.pem"

BUILT_SIZE="$(stat -c%s "$SCRATCH/built.efi")"
info "ukify build succeeded (${BUILT_SIZE} bytes)"

#============================================================================
# Stage 9: Atomic replace within staging
#============================================================================
#
# 90-uki-copy.install will copy the staged UKI to the ESP. By replacing
# $STAGED_UKI here, we ensure 90-uki-copy delivers our enriched UKI, not
# 60-ukify's plain output.
#
# Two-step replace because $SCRATCH (tmpfs) and $KERNEL_INSTALL_STAGING_AREA
# may be on different filesystems:
#   step 1: install built.efi to STAGED_UKI.new in the staging dir (cross-fs
#            copy; non-atomic but to a sentinel name that 90-uki-copy ignores)
#   step 2: rename .new over the original within the staging dir (same fs,
#            atomic per POSIX rename(2))
#

STAGED_NEW="${STAGED_UKI}.new"
install -m 0600 -o root -g root "$SCRATCH/built.efi" "$STAGED_NEW"
mv -f "$STAGED_NEW" "$STAGED_UKI"

#============================================================================
# Stage 10: Belt-and-braces verification of the final staged UKI
#============================================================================
#
# These checks must all pass. If any fail, the hook exits non-zero,
# kernel-install aborts the transaction, and 90-uki-copy never runs:
# the ESP is never touched.
#

# Inspect on a tmpfs copy (per corrected §16.1: never run objcopy/objdump
# write paths against the source)
INSPECT_COPY="$SCRATCH/final-inspect.efi"
cp "$STAGED_UKI" "$INSPECT_COPY"

# 10.1: .pcrpkey and .pcrsig sections present
if ! objdump -h "$INSPECT_COPY" \
     | awk '$2==".pcrpkey"{p=1} $2==".pcrsig"{s=1} END{exit !(p && s)}'; then
    err "final UKI missing .pcrpkey or .pcrsig"
    exit 1
fi

# 10.2: Authenticode signature verifies against db.crt
if ! sbverify --cert "$DB_CRT" "$STAGED_UKI" >/dev/null 2>&1; then
    err "final UKI sbverify failed"
    exit 1
fi

# 10.3: Exactly one Authenticode signer (no double-signing, no missing sig)
sig_count="$(pesign --show-signature --in="$INSPECT_COPY" 2>/dev/null \
              | grep -c 'common name' || true)"
if (( sig_count != 1 )); then
    err "final UKI has ${sig_count} Authenticode signatures (expected 1)"
    exit 1
fi

# 10.4: Section-VMA sanity (catch §16.13-style layout corruption)
#
# Any section with VMA >= $VMA_SANITY_THRESHOLD indicates the kind of
# layout damage objcopy --add-section produced (.pcrsig at 0x200000000).
# ukify-built UKIs never produce VMAs in this range; if this fires, the
# build is suspect even if other checks passed.
if objdump -h "$INSPECT_COPY" \
   | awk -v threshold="$VMA_SANITY_THRESHOLD" '
       /^[[:space:]]+[0-9]+[[:space:]]+\./ {
         vma = strtonum("0x" $4)
         if (vma >= strtonum(threshold)) {
           printf "anomalous VMA: %s at 0x%s\n", $2, $4
           bad = 1
         }
       }
       END { exit !bad }
     '; then
    err "final UKI has section(s) with anomalous VMA (>= ${VMA_SANITY_THRESHOLD}): see Runbook §16.13"
    exit 1
fi

# 10.5: File size within sanity range (kernel UKIs are 30–80 MB; outside
# this is a strong signal something went wrong)
FINAL_SIZE="$(stat -c%s "$STAGED_UKI")"
if (( FINAL_SIZE < 20 * 1024 * 1024 || FINAL_SIZE > 200 * 1024 * 1024 )); then
    err "final UKI size ${FINAL_SIZE} bytes is outside sanity range (20MB–200MB)"
    exit 1
fi

shred -u "$INSPECT_COPY"

info "kernel ${KERNEL_VERSION}: ukify-native signed UKI ready (${FINAL_SIZE} bytes, ${sig_count} signature)"
exit 0
HOOK_EOF
```

### Commands: lint and verify checksum

```bash
shellcheck -S info /tmp/80-tpm2-sign.install
bash -n /tmp/80-tpm2-sign.install
echo "exit: $?"
sha256sum /tmp/80-tpm2-sign.install
wc -l /tmp/80-tpm2-sign.install
```

### Expected output (lint)

- `shellcheck -S info` exits 0 with no output
- `bash -n` exits 0 with no output
- `sha256sum` reports `5857e51d5551e05a7ca384f71529ec2fef07f5a6dbe40df70c6bbe49d720947e`
- `wc -l` reports 296 lines

### Stop condition (lint)

- ShellCheck SC2317/SC2329 fires on `_cleanup` body, your paste lost the `# shellcheck disable=SC2317,SC2329  # invoked via trap below` line above `_cleanup() {`. Fix and re-paste.
- sha256 doesn't match: paste introduced byte differences (smart quotes, line endings, trailing whitespace). Re-paste.

### Commands: synthetic dry-run before installing on the canonical path

The dry-run runs the hook against a tmpfs synthetic staging area exactly as `kernel-install` would, without touching the ESP, so a structurally invalid forward-sealed UKI is caught here, before the Step 15 cliff edge. Rationale (why syntactic lint is insufficient, what runtime properties this exercises) is in `06C`; the firmware-rejected-UKI failure mode is `06F` B.1.

```bash
# Inside VM as root
KVER="$(uname -r)"
MID="$(cat /etc/machine-id)"

# 1. Verify PCR 7 still matches Block A seal-time. If it has drifted,
#    the hook's stage 5 unseal will fail and there is no point running the dry-run.
PCR7_NOW="$(cat /sys/class/tpm/tpm0/pcr-sha256/7)"
echo "Current PCR 7: $PCR7_NOW"
echo "Expected PCR 7 (record from Step 12 snapshot description): <PCR7 from Block A>"
# If these do not match, halt. PCR 7 drift means Secure Boot policy changed since
# Block A. Investigate before running the hook.

# 2. Build a synthetic staging area in tmpfs that mimics what 60-ukify produces
FAKE_STAGING="$(mktemp -d -p /run -t tboot-dryrun-XXXXXX)"
mkdir -p "$FAKE_STAGING/${MID}/${KVER}"
STAGED_DIR="$FAKE_STAGING/${MID}/${KVER}"

# 3. Run 60-ukify directly to populate the staging area with a base (non-pcrsig) UKI.
#    This is what kernel-install would do automatically; here we drive it manually
#    so the synthetic staging area mirrors a real transaction.
KERNEL_IMAGE="/lib/modules/${KVER}/vmlinuz"
INITRD="/boot/initramfs-${KVER}.img"

env \
  KERNEL_INSTALL_LAYOUT=uki \
  KERNEL_INSTALL_UKI_GENERATOR=ukify \
  KERNEL_INSTALL_STAGING_AREA="$STAGED_DIR" \
  KERNEL_INSTALL_MACHINE_ID="$MID" \
  KERNEL_INSTALL_BOOT_ROOT=/boot/efi \
  bash /usr/lib/kernel/install.d/60-ukify.install add "$KVER" "$STAGED_DIR" "$KERNEL_IMAGE" "$INITRD"
echo "60-ukify exit: $?"

ls -la "$STAGED_DIR"
# Should contain uki.efi at this point. No .pcrsig section yet; that is our hook's job.
sbverify --cert /etc/uefi-keys/db.crt "$STAGED_DIR/uki.efi"
objdump -h "$STAGED_DIR/uki.efi" | awk '/^[[:space:]]+[0-9]+[[:space:]]+\./ {print $2}' | sort -u

# 4. Invoke the hook against the synthetic staging area
env \
  KERNEL_INSTALL_LAYOUT=uki \
  KERNEL_INSTALL_UKI_GENERATOR=ukify \
  KERNEL_INSTALL_STAGING_AREA="$STAGED_DIR" \
  KERNEL_INSTALL_MACHINE_ID="$MID" \
  KERNEL_INSTALL_BOOT_ROOT=/boot/efi \
  bash /tmp/80-tpm2-sign.install add "$KVER" "$STAGED_DIR" "$KERNEL_IMAGE" "$INITRD"
echo "hook exit: $?"

# 5. Inspect the output of the hook
DRY_UKI="$STAGED_DIR/uki.efi"

echo "--- size ---"
stat -c%s "$DRY_UKI"

echo "--- sbverify ---"
sbverify --cert /etc/uefi-keys/db.crt "$DRY_UKI"

echo "--- signer count (must be 1) ---"
pesign --show-signature --in="$DRY_UKI" 2>/dev/null | grep -c 'common name' || true

echo "--- sections (must include .pcrpkey and .pcrsig) ---"
objdump -h "$DRY_UKI" | awk '/^[[:space:]]+[0-9]+[[:space:]]+\./ {print "  " $2}'

echo "--- VMA sanity (no section above 0x180000000) ---"
objdump -h "$DRY_UKI" | awk '
  /^[[:space:]]+[0-9]+[[:space:]]+\./ {
    vma = strtonum("0x" $4)
    if (vma >= strtonum("0x180000000")) {
      printf "ANOMALOUS: %s at 0x%s\n", $2, $4
      bad = 1
    }
  }
  END {
    if (!bad) print "  all sections within sane range"
    exit bad
  }'

echo "--- .pcrsig contents ---"
TMPD="$(mktemp -d)"
cp "$DRY_UKI" "$TMPD/inspect.efi"
objcopy --dump-section ".pcrsig=$TMPD/pcrsig.json" "$TMPD/inspect.efi" "$TMPD/throwaway.efi"
python3 -c "
import json
d = json.load(open('$TMPD/pcrsig.json'))
for e in d['sha256']:
    print(f'  pcrs={e[\"pcrs\"]}')
    print(f'  pkfp={e[\"pkfp\"]}')
    print(f'  pol ={e[\"pol\"]}')
"
shred -u "$TMPD"/* && rm -rf "$TMPD"

# 6. Confirm the hook's trap cleaned up its scratch directory
ls /run/tboot-sign 2>&1 | head -1
# Expected: "ls: cannot access '/run/tboot-sign': No such file or directory"

# 7. Tear down the synthetic staging area
rm -rf "$FAKE_STAGING"
```

### Expected output (dry-run)

| Check | Pass criterion |
|---|---|
| `60-ukify` exit | 0 |
| `60-ukify` produced `uki.efi` | yes, signed, 12 sections (no `.pcrsig` yet) |
| Hook exit | 0 |
| Final UKI size | typically 50 to 80 MB |
| `sbverify` | OK |
| Signer count | 1 |
| Sections | 13 total, includes both `.pcrpkey` and `.pcrsig` |
| VMA sanity | "all sections within sane range" |
| `.pcrsig pcrs` | `[11]` |
| `.pcrsig pkfp` | sha256 of `/etc/systemd/tpm2-pcr-public-key.pem` (recorded in Step 12 snapshot description) |
| `.pcrsig pol` | non-empty hex string (this is the forward-seal policy digest the hook commits to) |
| `/run/tboot-sign` post-hook | absent |

The hook's `info:` log lines should appear in the journal under `journalctl -t 80-tpm2-sign --since '5 minutes ago'`. A clean run shows: `starting`, `60-ukify output present and signed`, `unsealing policy private key`, `policy private key unsealed and validated`, `rebuilding UKI via ukify build with native PCR signing`, `ukify build succeeded`, `ukify-native signed UKI ready`. No `error:` lines.

> [!tip] Optional section-by-section reference comparison
> If a known-good MID-prefixed UKI from an earlier rebuild is on the ESP, you can sha256-compare the dry-run output section-by-section as the strongest pre-transaction validation. The technique (and which sections legitimately differ) is in `06C`.

### Stop condition (dry-run)

- `60-ukify` exits non-zero: the staging area cannot be built, so the hook has nothing to enrich. Read its stderr; usually missing `vmlinuz` or `initramfs` paths, or a malformed `/etc/kernel/uki.conf`.
- Hook exits non-zero: read the journal (`journalctl -t 80-tpm2-sign --since '5 minutes ago'`) for which stage failed. Common causes: PCR 7 drifted from Block A seal-time (stage 5 unseal fails), input artefact missing (stage 1 gate fails), `ukify build` itself errored (stage 7 fails).
- VMA sanity reports `ANOMALOUS`: should be impossible with the v2.1.2 hook source, but if it fires, **do not install** the hook at the canonical path. Triage via `06F` finding B.1.
- Section count is not 13: either `60-ukify` didn't produce a valid base UKI (something missing from `uki.conf`), or the hook didn't add `.pcrsig` (look for `.pcrsig` parsing failures in the journal).
- `.pcrsig pcrs` is not `[11]`: the hook's `--phases=enter-initrd` is being overridden somehow. Verify the hook source against the documented sha.
- `/run/tboot-sign` persists after hook exit: the trap was killed (SIGKILL?) or the hook was invoked under shell options that bypass traps. Triage; this is unusual.

### Snapshot point

None yet. Step 15 installs and snapshots together.

---

## Step 15: Install hook at canonical path

**Goal:** Install the validated hook at `/etc/kernel/install.d/80-tpm2-sign.install` with mode 0755, owner root:root.
**Prerequisites:** Step 14 (lint + sha verified).
**Where to run:** Fedora VM as `root` for hook installation; Proxmox host for the snapshot command.

### Commands

```bash
# Inside VM as root
install -m 0755 -o root -g root \
  /tmp/80-tpm2-sign.install \
  /etc/kernel/install.d/80-tpm2-sign.install

# Verify
ls -la /etc/kernel/install.d/80-tpm2-sign.install
sha256sum /etc/kernel/install.d/80-tpm2-sign.install
file /etc/kernel/install.d/80-tpm2-sign.install

# Confirm hook ordering window
ls /etc/kernel/install.d /usr/lib/kernel/install.d 2>/dev/null \
  | grep -E '\.install$' | sort -u

# Cleanup /tmp source
shred -u /tmp/80-tpm2-sign.install
```

### Expected output

- Canonical path file: mode `-rwxr-xr-x`, owner `root root`, sha `5857e51d…20947e`
- `file` reports `Bourne-Again shell script, Unicode text, UTF-8 text executable`
- Hook ordering shows `60-ukify.install` → `80-tpm2-sign.install` → `90-uki-copy.install` (lex-sorted)
- `/tmp/80-tpm2-sign.install` shredded

### Stop condition

- Sha doesn't match after install, file system permissions or filesystem-specific issues; investigate.
- `80-tpm2-sign.install` not between `60-ukify.install` and `90-uki-copy.install` in lex order: wrong filename, rename.

### Snapshot point

```bash
# Run on Proxmox host
qm snapshot 500 module5-kernel-signing-hook-rewritten \
  --description "Hook at /etc/kernel/install.d/80-tpm2-sign.install (sha256 5857e51d5551e05a7ca384f71529ec2fef07f5a6dbe40df70c6bbe49d720947e, mode 0755, 11319 bytes, 296 lines). v2.1.2 form (ukify-native PCR signing + SC2317/SC2329 disable on _cleanup). Lint clean (ShellCheck 0.11.0 -S info, bash -n). Hook NOT YET exercised by real kernel-install/DNF transaction. Holds remain in effect (discipline-only): dnf upgrade kernel*, dnf reinstall kernel*, dnf upgrade systemd*."
```

✅ `module5-kernel-signing-hook-rewritten` taken.

---

> [!warning] Cliff edge crossed at Step 15
> The hook is now on the active kernel-install path. Any `kernel-install add` invocation runs it: DNF kernel updates, manual `kernel-install add`, post-install scriptlets. The discipline gate ("don't run dnf reinstall kernel* yet") was the only protection between Steps 13 and 16. Step 16 deliberately exercises the hook.

---

## Step 16: First hooked kernel reinstall (live transaction validation)

**Goal:** Run `dnf reinstall` against the running kernel to drive the hook through `%posttrans` → `kernel-install add` → `60-ukify` → `80-tpm2-sign` (our hook) → `90-uki-copy`. Verify the hook produces a forward-sealed UKI on the ESP.
**Prerequisites:** Step 15.
**Where to run:** Fedora VM as `root`.

### Commands: pre-flight forensic capture

```bash
# Inside VM as root
KVER="$(uname -r)"
MID="$(cat /etc/machine-id)"
NEW_HOOK_UKI="/boot/efi/EFI/Linux/${MID}-${KVER}.efi"
HOOK_PATH="/etc/kernel/install.d/80-tpm2-sign.install"

# Sanity gates
echo "Hook sha:    $(sha256sum "$HOOK_PATH" | awk '{print $1}')"
echo "Expected:    5857e51d5551e05a7ca384f71529ec2fef07f5a6dbe40df70c6bbe49d720947e"
echo "PCR 7 now:   $(cat /sys/class/tpm/tpm0/pcr-sha256/7)"
echo "PCR 7 ref:   <PCR7 from Step 11 snapshot description>"

# Capture pre-DNF state
PRE_DNF_TIME="$(date -Iseconds)"
journalctl --rotate
```

### Commands: live transaction (version-pinned to running kernel)

```bash
dnf reinstall -y \
  "kernel-core-${KVER}" \
  "kernel-modules-${KVER}" \
  "kernel-modules-core-${KVER}"
echo "dnf exit: $?"
```

> [!note] Why exclude `kernel-${KVER}` (the meta-package)
> Including it would trigger `kernel-install add` twice in the same transaction (once from `kernel-core` post-install, once from `kernel` meta-package post-install), firing the hook twice. The hook is idempotent so this isn't fatal, but B.3-style validation wants exactly one deterministic invocation. See `06F` finding E.2, with original context in `06_Lab_Setup_Runbook_continuation_v3.md` §16.20 and §20.9.

> [!tip] Re-running this validation later
> The full transaction-and-checks flow in this step is wrapped up as `scripts/validate_dnf_kernel_reinstall.sh` for ongoing operation, after the rebuild is done. The script runs the same `dnf reinstall` and the same post-flight checks, then exits non-zero on any failure. Use it during real kernel updates, or as a periodic health check. The inline form below is the canonical first-time path; the script is for repeated runs.

### Commands: verify hook journal

```bash
journalctl -t 80-tpm2-sign --since "$PRE_DNF_TIME" --no-pager

# Hook should fire exactly once with full info chain
ERR_COUNT="$(journalctl -t 80-tpm2-sign --since "$PRE_DNF_TIME" --no-pager | grep -c 'error:' || true)"
HOOK_START_COUNT="$(journalctl -t 80-tpm2-sign --since "$PRE_DNF_TIME" --no-pager \
                    | grep -c 'info: starting for kernel' || true)"

echo "ERR_COUNT=$ERR_COUNT (expected 0)"
echo "HOOK_START_COUNT=$HOOK_START_COUNT (expected 1)"

# Trap fired
ls -la /run/tboot-sign 2>/dev/null && echo "✗ trap did not fire" || echo "✓ trap fired"

# Hook integrity preserved
sha256sum "$HOOK_PATH"
# expected: 5857e51d5551e05a7ca384f71529ec2fef07f5a6dbe40df70c6bbe49d720947e
```

### Commands: verify ESP UKI

```bash
ls -la "$NEW_HOOK_UKI"

# Should be ~56 MB; should NOT exist before this DNF run
sbverify --cert /etc/uefi-keys/db.crt "$NEW_HOOK_UKI"

# Section list must include .pcrpkey AND .pcrsig
objdump -h "$NEW_HOOK_UKI" | awk '/^[[:space:]]+[0-9]+[[:space:]]+\./ {print $2}'

# Exactly one Authenticode signer
pesign --show-signature --in="$NEW_HOOK_UKI" 2>/dev/null | grep -c 'common name'

# Extract .pcrsig and inspect (read-only on a copy)
TMPD="$(mktemp -d)"
cp "$NEW_HOOK_UKI" "$TMPD/inspect.efi"
objcopy --dump-section ".pcrsig=$TMPD/pcrsig.json" "$TMPD/inspect.efi" "$TMPD/throwaway.efi"
python3 -c "
import json
d = json.load(open('$TMPD/pcrsig.json'))
for e in d['sha256']:
    print(f\"  pcrs={e['pcrs']}\")
    print(f\"  pkfp={e['pkfp']}\")
    print(f\"  pol ={e['pol']}\")
"
shred -u "$TMPD"/* && rm -rf "$TMPD"
```

### Expected output

| Check | Pass criterion |
|---|---|
| `dnf exit` | 0 |
| Hook journal | full info chain (starting → 60-ukify output → unsealing → unsealed → rebuilding → built → ready) |
| `ERR_COUNT` | 0 |
| `HOOK_START_COUNT` | 1 |
| `/run/tboot-sign` post-DNF | absent |
| Hook sha post-DNF | `5857e51d…20947e` (unchanged) |
| `${MID}-${KVER}.efi` | exists, ~56 MB |
| sbverify | OK |
| Section count | 13 (includes `.pcrpkey` and `.pcrsig`) |
| Signer count | 1 |
| `.pcrsig pkfp` | matches sha256 of policy public key (recorded in Step 12 snapshot description) |
| `.pcrsig pcrs` | `[11]` |

### Stop condition

- `dnf exit` non-zero: the kernel-install transaction aborted; investigate stderr. The ESP UKI from before the transaction is preserved (fail-safe).
- `ERR_COUNT > 0`: hook produced an error; read journalctl output to find which stage failed.
- `HOOK_START_COUNT != 1`: either hook didn't fire (check `KERNEL_INSTALL_LAYOUT=uki` in `/etc/kernel/install.conf`) or fired multiple times (you included the meta-package).
- ESP UKI not produced or sbverify fails. `90-uki-copy.install` failed downstream of our hook; rare.
- VMA sanity warning anywhere in journalctl, should be impossible with the v2.1.2 hook, but if it appears, do not boot.

### Snapshot point

None at this stage. Step 17 reboots and validates runtime; both will be snapshotted together in Step 18.

---

## Step 17: Reboot and runtime PCR 11 validation

**Goal:** Boot the new MID-prefixed UKI via one-shot, verify runtime PCR 11 matches the systemd-measure prediction, confirm full phase chain in journal.
**Prerequisites:** Step 16 success.
**Where to run:** Fedora VM as `root` before reboot and again after reconnecting.

### Commands: independent prediction (defense in depth)

```bash
# Inside VM as root
KVER="$(uname -r)"
MID="$(cat /etc/machine-id)"
NEW_HOOK_UKI="/boot/efi/EFI/Linux/${MID}-${KVER}.efi"
MEASURE="/usr/lib/systemd/systemd-measure"   # not on PATH; absolute path required

# Extract sections from the new UKI
WORK="$(mktemp -d)"
cp "$NEW_HOOK_UKI" "$WORK/uki.efi"
for sec in linux initrd cmdline osrel uname sbat pcrpkey; do
    objcopy --dump-section ".${sec}=$WORK/${sec}" "$WORK/uki.efi"
done

# Calculate full phase ladder and extract post-`ready` value
PRED_PCR11="$("$MEASURE" calculate \
  --linux="$WORK/linux" \
  --initrd="$WORK/initrd" \
  --cmdline="$WORK/cmdline" \
  --osrel="$WORK/osrel" \
  --uname="$WORK/uname" \
  --sbat="$WORK/sbat" \
  --pcrpkey="$WORK/pcrpkey" \
  --bank=sha256 \
  | grep -E '^11:sha256=' | tail -1 | cut -d= -f2)"

echo "Predicted post-ready PCR 11: $PRED_PCR11"
install -d -m 0700 /root/tboot-lab/state
printf '%s
' "$(echo "$PRED_PCR11" | tr 'a-f' 'A-F')" > /root/tboot-lab/state/expected-pcr11-after-hook-uki.txt
echo "Saved expected PCR 11 to /root/tboot-lab/state/expected-pcr11-after-hook-uki.txt"
shred -u "$WORK"/* && rm -rf "$WORK"
```

> [!note] Parser idiom
> `grep -E '^11:sha256=' | tail -1 | cut -d= -f2`. The `tail -1` is essential because `systemd-measure calculate` emits one `11:sha256=` line per phase boundary; the last one is the post-`ready` final value. See `06F` finding D.2.

> [!tip] One-line equivalent
> The same prediction is wrapped up as `scripts/predict_pcr11_from_uki.sh`. Run `predict_pcr11_from_uki.sh --store /boot/efi/EFI/Linux/${MID}-${KVER}.efi` to predict and persist the expected value to `/root/tboot-lab/state/expected-pcr11-after-hook-uki.txt` in one go. The inline commands above are kept for first-time pedagogy; the script is for repeated runs.

### Commands: set one-shot and reboot

```bash
HOOK_ID="${MID}-${KVER}.efi"
bootctl set-oneshot "$HOOK_ID"
bootctl status | grep -Ei 'current entry|default entry|one-shot|oneshot'
sync
reboot
```

### Commands: after reboot reconnects, validate

```bash
# Inside VM as root, after reboot
KVER="$(uname -r)"
MID="$(cat /etc/machine-id)"
EXPECTED_ENTRY="${MID}-${KVER}.efi"

bootctl status | grep -E 'Secure Boot|TPM2|Measured UKI|Current Entry|Default Entry'
CURRENT_ENTRY="$(bootctl status 2>/dev/null | awk -F': ' '/Current Entry:/ {print $2}' | xargs)"

echo "Expected entry: $EXPECTED_ENTRY"
echo "Actual entry:   $CURRENT_ENTRY"

# Runtime PCR 11
EXPECTED_PCR11="$(cat /root/tboot-lab/state/expected-pcr11-after-hook-uki.txt)"
RUNTIME_PCR11="$(cat /sys/class/tpm/tpm0/pcr-sha256/11)"
echo "Runtime PCR 11:  $RUNTIME_PCR11"
echo "Expected PCR 11: $EXPECTED_PCR11"
if [[ "${RUNTIME_PCR11^^}" == "${EXPECTED_PCR11^^}" ]]; then
  echo "✓ runtime PCR 11 matches predicted value"
else
  echo "✗ runtime PCR 11 mismatch" >&2
  exit 1
fi

# Full phase chain in journal
journalctl -b --no-pager \
  | grep -Ei 'Extended PCR index 11|systemd-pcrextend|pcrphase' \
  | tail -20
```

### Expected output

```
Secure Boot: enabled (user)
TPM2 Support: yes
Measured UKI: yes
Current Entry: <MID>-<KVER>.efi      ← hook UKI (MID-prefixed)
Default Entry: <KVER>.efi            ← Phase 1 fallback (unchanged)

Expected entry: matches Current Entry
Runtime PCR 11: same value (case-insensitive) as predicted

Journal phase chain:
  systemd-pcrextend[...]: Extended PCR index 11 with 'enter-initrd'
  systemd-pcrextend[...]: Extended PCR index 11 with 'leave-initrd'
  systemd-pcrextend[...]: Extended PCR index 11 with 'sysinit'
  systemd-pcrextend[...]: Extended PCR index 11 with 'ready'
  systemd-pcrextend[...]: Extended PCR index 15 with 'machine-id:<MID>'
```

### Stop condition

- VM doesn't boot. Proxmox console shows the OVMF firmware menu; select the Phase 1 default UKI to recover. Then triage why the hook UKI was firmware-rejected. With the v2.1.2 hook this should be impossible; if it happens, investigate using the `.loaderror` recovery procedure in `06_Lab_Setup_Runbook_continuation_v2_1_patches.md` §20.6.
- Boots but `Current Entry` is the bare-kver Phase 1 UKI. `bootctl set-oneshot` didn't take; possibly EFI variable write failed.
- Runtime PCR 11 ≠ predicted, environmental drift (kernel/initramfs/cmdline changed since UKI was built); rebuild UKI with current inputs and try again.
- Phase chain incomplete (missing one or more of enter-initrd / leave-initrd / sysinit / ready), initramfs lacks `tpm2-tss` or systemd-pcrphase services aren't active. Re-run Step 13 to ensure tpm2-tss is in initramfs.

### Snapshot point

✅ Step 18.

---

## Step 18: Snapshot the validated state (`module5-kernel-signing-hook-validated`)

**Goal:** Capture the validated state, the highest-value rollback target in the entire Module 5 work.
**Prerequisites:** Step 17 success.
**Where to run:** Proxmox host `pve-host` as `root`; commands SSH into the Fedora VM for state collection.

### Commands

```bash
# Run on Proxmox host
PCR7="$(ssh root@192.0.2.40 'cat /sys/class/tpm/tpm0/pcr-sha256/7')"
PCR11="$(ssh root@192.0.2.40 'cat /sys/class/tpm/tpm0/pcr-sha256/11')"
NEW_HOOK_UKI_SHA="$(ssh root@192.0.2.40 'sha256sum /boot/efi/EFI/Linux/$(cat /etc/machine-id)-$(uname -r).efi | awk "{print \$1}"')"

qm snapshot 500 module5-kernel-signing-hook-validated \
  --description "Block B.3 live validation complete. Rewritten ukify-native kernel-install hook validated end-to-end through real DNF transaction → reboot → runtime PCR 11 match. Hook at /etc/kernel/install.d/80-tpm2-sign.install (sha256 5857e51d5551e05a7ca384f71529ec2fef07f5a6dbe40df70c6bbe49d720947e). Booted UKI: /boot/efi/EFI/Linux/<MID>-<KVER>.efi (sha ${NEW_HOOK_UKI_SHA}). Runtime PCR 7 = ${PCR7} (Block A seal-time). Runtime PCR 11 = ${PCR11} (matches systemd-measure prediction post-ready). Full phase chain confirmed. DNF transaction: dnf reinstall -y kernel-core-<KVER> kernel-modules-<KVER> kernel-modules-core-<KVER>; explicitly excluded kernel meta-package. Phase 1 manual UKI preserved at bare-kver path. Forensic artefacts preserved. Block B Stage 3 done. Pending: B.2 dracut-only path, B.4 systemd-boot signing automation, B.5 systemd-boot reinstall validation, Module 3 LUKS enrollment."

qm listsnapshot 500
lvs --units g vm-storage/vm-thin
```

### Expected output

- Snapshot named `module5-kernel-signing-hook-validated` in tree, parent = `module5-kernel-signing-hook-rewritten`
- `qm listsnapshot` shows `current` below it
- `Data%` shifted by approximately 0 (CoW snapshots cost zero at creation)

### Stop condition

- `qm snapshot` exit non-zero: likely thin pool capacity issue.

### Snapshot point

✅ `module5-kernel-signing-hook-validated` taken. **This is the canonical rollback target** until B.4/B.5 produce a successor.

> [!tip] Ongoing health check
> From this point on, `scripts/show_trusted_boot_result.sh` is the quickest way to confirm the trust chain is intact. It prints `bootctl status`, the current MID-prefixed UKI's signature and section list, runtime PCR 7 and PCR 11, and the most recent PCR-phase journal evidence. If `predict_pcr11_from_uki.sh --store` was run earlier, it also compares runtime PCR 11 against the stored expected value. Useful after every reboot and after every kernel update.

---


## Step 19: Make MID-prefixed UKI the persistent default before Module 3

**Goal:** Convert the validated one-shot boot into the persistent default boot path. This prevents Module 3 from enrolling LUKS TPM2 policy against a UKI that is not actually used by default.
**Prerequisites:** Step 18 snapshot taken.
**Where to run:** Fedora VM as `root`; Proxmox host for the snapshot command at the end.
**Last validated:** 2026-05-10 on VM 500 (`tboot-lab`).

### Commands

```bash
# Inside VM as root
KVER="$(uname -r)"
MID="$(cat /etc/machine-id)"
HOOK_ID="${MID}-${KVER}.efi"

bootctl set-default "$HOOK_ID"
bootctl status 2>/dev/null | grep -E '^\s*(Current Entry:|Default Entry:|Secure Boot:|Measured UKI:)'
```

### Expected output

```
Default Entry: <MID>-<KVER>.efi
```

`Current Entry` should also be `<MID>-<KVER>.efi` if you are still on the validated boot from Step 17.

### Stop condition

- `Default Entry` remains the bare-kver Phase 1 fallback: do not start Module 3. Re-run `bootctl set-default` and check EFI variable write permissions.
- `bootctl list` does not show the MID-prefixed UKI. Step 16 did not produce the kernel-install path correctly.

### Reboot and confirm the default holds

A `bootctl set-default` write only proves the EFI variable updated. Confirm it survives a normal reboot before snapshotting.

```bash
sync
reboot

# After reboot, log back in:
bootctl status 2>/dev/null | grep -E '^\s*(Current Entry:|Default Entry:|Secure Boot:|Measured UKI:)'
# Both Current Entry and Default Entry must be ${MID}-${KVER}.efi.
```

### Regenerate and store the expected runtime PCR 11

Step 17's snapshot description carries the expected runtime PCR 11. After Step 19 reboot you should re-derive that value from the currently-booted UKI and store it under `/root/tboot-lab/state/` so Module 3 enrollment scripts can read a stable file instead of parsing snapshot descriptions.

```bash
# Inside VM as root
sudo bash <<'EOF'
set -eo pipefail

KVER="$(uname -r)"
MID="$(cat /etc/machine-id)"
UKI="/boot/efi/EFI/Linux/${MID}-${KVER}.efi"
MEASURE="/usr/lib/systemd/systemd-measure"

WORK="$(mktemp -d)"
trap 'shred -u "$WORK"/* 2>/dev/null || true; rm -rf "$WORK"' EXIT

echo "KVER=$KVER"
echo "MID=$MID"
echo "UKI=$UKI"
echo

cp "$UKI" "$WORK/uki.efi"

for sec in linux initrd cmdline osrel uname sbat pcrpkey; do
  objcopy --dump-section ".${sec}=$WORK/${sec}" "$WORK/uki.efi" "$WORK/throwaway-${sec}.efi" 2>/dev/null
done

PRED_PCR11="$("$MEASURE" calculate \
  --linux="$WORK/linux" \
  --initrd="$WORK/initrd" \
  --cmdline="$WORK/cmdline" \
  --osrel="$WORK/osrel" \
  --uname="$WORK/uname" \
  --sbat="$WORK/sbat" \
  --pcrpkey="$WORK/pcrpkey" \
  --bank=sha256 2>&1 \
  | grep -E '^11:sha256=' | tail -1 | cut -d= -f2 | tr 'a-f' 'A-F')"

RUNTIME_PCR11="$(cat /sys/class/tpm/tpm0/pcr-sha256/11 | tr 'a-f' 'A-F')"

echo "PRED_PCR11=$PRED_PCR11"
echo "RUNTIME_PCR11=$RUNTIME_PCR11"

if [[ "$PRED_PCR11" != "$RUNTIME_PCR11" ]]; then
  echo "PCR11 prediction mismatch; do not snapshot yet"
  exit 1
fi

install -d -m 0700 /root/tboot-lab/state
printf '%s\n' "$PRED_PCR11" > /root/tboot-lab/state/expected-pcr11-after-hook-uki.txt

echo
echo "stored=/root/tboot-lab/state/expected-pcr11-after-hook-uki.txt"
cat /root/tboot-lab/state/expected-pcr11-after-hook-uki.txt
EOF
```

> [!note] Why the heredoc and `sudo bash <<'EOF'` form
> Running `set -u` directly in an interactive root shell makes bash fatal on prompt-related variables that some Fedora prompt themes leave unset (`PROMPT_START` and similar). Wrapping the strict-mode block in `sudo bash <<'EOF' ... EOF` runs it in a clean non-interactive shell where these variables are not in scope. See `06F` finding F.4.

### Expected output (post-reboot validation)

- Both `Current Entry` and `Default Entry` are the MID-prefixed UKI.
- `PRED_PCR11` and `RUNTIME_PCR11` are byte-identical (the script exits non-zero otherwise).
- `/root/tboot-lab/state/expected-pcr11-after-hook-uki.txt` exists with mode `0700` directory, single uppercased SHA-256 line.

### Stop condition (post-reboot validation)

- `Default Entry` reverted to the Phase 1 bare-kver UKI after reboot: the EFI variable write did not persist. Investigate firmware variable storage and retry.
- `PRED_PCR11 != RUNTIME_PCR11`: do not snapshot. Either the UKI on disk differs from what booted (rare; would indicate ESP corruption between Step 17 and Step 19), or the prediction inputs were extracted incorrectly. Re-verify by hand before continuing.

### Snapshot point

> [!note] Snapshot-description values are reference values
> The machine-id, UKI sha, PCR 7, and PCR 11 values in the snapshot description below are reference values from the validated lab. On a fresh rebuild, substitute the values produced by that rebuild.

```bash
# Run on Proxmox host
qm snapshot 500 module5-default-entry-cleaned \
  --description "Module 5 Step 19 complete. Validated MID-prefixed hook-generated UKI set as persistent systemd-boot default and confirmed after normal reboot. Current/default entry: 0123456789abcdef0123456789abcdef-6.19.14-200.fc43.x86_64.efi. Runtime PCR 7 = BF8878BD8B70654924BD39CA71D770F556F03558AC200BAD46A9CEB5A2B596BD. Runtime PCR 11 = 6F2EA11CA4CEDF2F3D04D015942445A9F30B7ACE8E339B6F06342C728CBA2B0B, matching systemd-measure prediction stored at /root/tboot-lab/state/expected-pcr11-after-hook-uki.txt. Ready for Module 3 LUKS TPM2 enrollment. Pending: B.2 dracut-only path, B.4/B.5 systemd-boot signing automation."
```

✅ `module5-default-entry-cleaned` taken on this lab at 2026-05-10 18:44:52. This is the successor rollback target to `module5-kernel-signing-hook-validated` and the preferred starting point for Module 3.

---

# Module 3: LUKS2 TPM2 Keyslot Enrollment

Steps 20-25 were live-validated on this lab on 2026-05-21. Step 26 is documented procedure pending live execution. The seven steps below take the system from `module5-default-entry-cleaned` to a fully enrolled, reboot-validated TPM2 unlock chain.

Two findings emerged during validation and are documented in `06F`: **G.1** (split-policy enrollment is required; `--tpm2-signature` must be omitted because the hook signs the `enter-initrd` phase) and **G.2** (`kernel-install add` does not refresh `/boot/initramfs-${KVER}.img` under UKI layout, so `dracut --force` must run first when initramfs-content inputs change). The procedures below already incorporate both fixes.

---

## Step 20: Pre-flight verification and `.pcrsig` validation

**Goal:** Confirm the lab is in a clean baseline for Module 3, then extract and validate the policy signature embedded in the currently-booted UKI. No state changes; all checks read-only except the explicit passphrase test.
**Prerequisites:** Step 19 (`module5-default-entry-cleaned`). The currently-booted UKI must be the MID-prefixed hook-generated UKI.
**Where to run:** Fedora VM as `root`.
**Last validated:** 2026-05-21.

### Commands

```bash
bash <<'EOF'
set -euo pipefail

MID="$(cat /etc/machine-id)"
KVER="$(uname -r)"
UKI="/boot/efi/EFI/Linux/${MID}-${KVER}.efi"

# Architectural invariants (from 00_Current_Project_State.md)
EXPECTED_PKFP="81e8c4e74b044f9e66ec6498323cd7317e3999b675b6b1580c069db94bac6bc8"
EXPECTED_PCR11="6F2EA11CA4CEDF2F3D04D015942445A9F30B7ACE8E339B6F06342C728CBA2B0B"
VMA_THRESHOLD="0x180000000"    # 06F B.1
EXPECTED_SECTION_COUNT=13      # 06F B.1 op. consequences

STAGE_DIR="/run/tboot-pcrsig-module3"
rm -rf "$STAGE_DIR"
install -d -m 0700 "$STAGE_DIR"

echo "=== Pre-flight ==="
echo "UKI=$UKI"

# 1. systemd-cryptenroll available (need >= 252 for --tpm2-public-key)
systemd-cryptenroll --version | head -1

# 2. Policy public key on disk and parseable
ls -la /etc/systemd/ | grep -E 'tpm2-pcr|tpm2.*pub'
openssl pkey -pubin -in /etc/systemd/tpm2-pcr-public-key.pem -text -noout 2>&1 | head -5

# 3. Passphrase keyslot 0 still unlocks (06F F.5: </dev/tty required inside heredoc)
cryptsetup open --test-passphrase /dev/sda3 </dev/tty && echo "PASSPHRASE OK"

# 4. Current PCR state matches invariants
tpm2_pcrread sha256:7,11
echo "  expected PCR 11: $EXPECTED_PCR11"

# 5. UKI sections, count, VMA sanity (06F B.1)
SECTION_COUNT="$(objdump -h "$UKI" \
  | awk '/^[[:space:]]+[0-9]+[[:space:]]+\./ {print $2}' | wc -l)"
echo "section_count=$SECTION_COUNT (expected $EXPECTED_SECTION_COUNT)"
objdump -h "$UKI" | grep -E '\.pcrsig|\.pcrpkey'

objdump -h "$UKI" | awk -v threshold="$VMA_THRESHOLD" '
  /^[[:space:]]+[0-9]+[[:space:]]+\./ {
    vma = strtonum("0x" $4)
    if (vma >= strtonum(threshold)) {
      printf "  ANOMALOUS: %s at 0x%s\n", $2, $4; bad = 1
    }
  }
  END { if (!bad) print "  all sections within sane range"; exit bad }'

echo
echo "=== Extract .pcrsig from a COPY (06F B.1, B.3) ==="

cp "$UKI" "$STAGE_DIR/uki-copy.efi"
objcopy --dump-section ".pcrsig=$STAGE_DIR/tpm2-pcr-signature.json" \
        "$STAGE_DIR/uki-copy.efi" "$STAGE_DIR/throwaway.efi"

stat -c '%s bytes' "$STAGE_DIR/tpm2-pcr-signature.json"

python3 - <<'PYEOF'
import json, sys

path = "/run/tboot-pcrsig-module3/tpm2-pcr-signature.json"
expected_pkfp = "81e8c4e74b044f9e66ec6498323cd7317e3999b675b6b1580c069db94bac6bc8"

with open(path) as f:
    d = json.load(f)

ok = True
for e in d["sha256"]:
    print(f"  pcrs = {e['pcrs']}")
    print(f"  pkfp = {e['pkfp']}")
    print(f"  pol  = {e['pol']}")
    if e["pcrs"] != [11]:
        print(f"  FAIL: pcrs should be [11]"); ok = False
    if e["pkfp"] != expected_pkfp:
        print(f"  FAIL: pkfp mismatch"); ok = False

print("VERDICT:", "OK" if ok else "FAIL")
sys.exit(0 if ok else 1)
PYEOF

echo "JSON staged at: $STAGE_DIR/tpm2-pcr-signature.json"
EOF
```

### Expected output

| Check | Pass criterion |
|---|---|
| `systemd` version | >= 258 (for `--tpm2-public-key`) |
| Policy public key | RSA-3072, parseable |
| Passphrase test | `PASSPHRASE OK` |
| Current PCR 7 / PCR 11 | match `00_Current_Project_State.md` invariants |
| Section count | 13 |
| VMA sanity | all sections within sane range |
| `.pcrsig` size | ~700 bytes (RSA-3072 + metadata) |
| `pcrs` in JSON | `[11]` |
| `pkfp` in JSON | matches `81e8c4e7…6bc8` invariant |
| `pol` in JSON | matches `13be40c3…654be` invariant |
| Final verdict | `OK` |

### Stop condition

- Passphrase test fails: investigate before any header modification. A working keyslot 0 is the recovery anchor for the entire Module 3 sequence.
- PCR 11 runtime ≠ stored expected value: drift since `module5-default-entry-cleaned`; do not enroll.
- `pkfp` mismatch: the UKI was signed with a different policy key than the one currently on disk. Critical workflow bug; halt and triage.
- VMA anomaly: see `06F` B.1; do not proceed.

### Snapshot point

None. `module5-default-entry-cleaned` remains the rollback target through Step 23.

---

## Step 21: Pre-enrollment input validation

**Goal:** Confirm all enrollment inputs are accessible and the LUKS header is in the expected baseline state before any state-changing operation.
**Prerequisites:** Step 20.
**Where to run:** Fedora VM as `root`.
**Last validated:** 2026-05-21.

### Commands

```bash
bash <<'EOF'
set -euo pipefail
DEVICE="/dev/sda3"
PUBKEY="/etc/systemd/tpm2-pcr-public-key.pem"
JSON="/run/tboot-pcrsig-module3/tpm2-pcr-signature.json"
EXPECTED_PCR7="BF8878BD8B70654924BD39CA71D770F556F03558AC200BAD46A9CEB5A2B596BD"
EXPECTED_PCR11="6F2EA11CA4CEDF2F3D04D015942445A9F30B7ACE8E339B6F06342C728CBA2B0B"

echo "--- inputs ---"
ls -la "$JSON" "$PUBKEY"

echo "--- JSON parse ---"
python3 -m json.tool "$JSON" >/dev/null && echo "JSON OK"

echo "--- keyslots BEFORE enrollment (expect only slot 0 password) ---"
systemd-cryptenroll "$DEVICE"

echo
cryptsetup luksDump "$DEVICE" | awk '/^Keyslots:/,/^Digests:/'

echo "--- TPM2 device ---"
ls -la /dev/tpm0 /dev/tpmrm0

echo "--- PCR state ---"
tpm2_pcrread sha256:7,11

PCR7_NOW="$(cat /sys/class/tpm/tpm0/pcr-sha256/7 | tr 'a-f' 'A-F')"
PCR11_NOW="$(cat /sys/class/tpm/tpm0/pcr-sha256/11 | tr 'a-f' 'A-F')"
[[ "$PCR7_NOW" == "$EXPECTED_PCR7" ]]   || { echo "FAIL: PCR7 mismatch"; exit 1; }
[[ "$PCR11_NOW" == "$EXPECTED_PCR11" ]] || { echo "FAIL: PCR11 mismatch"; exit 1; }
echo "PCR 7 + 11 OK"
EOF
```

### Expected output

| Check | Pass criterion |
|---|---|
| JSON parse | OK |
| `systemd-cryptenroll` output | `SLOT 0 password` only |
| `luksDump` keyslots | `0: luks2` only; `Tokens:` empty |
| `/dev/tpm0` and `/dev/tpmrm0` | exist with sane permissions |
| PCR 7 match | invariant |
| PCR 11 match | invariant |

### Stop condition

- Token table non-empty: prior TPM2 enrollment present, must be removed (`--wipe-slot=tpm2`) before re-enrolling.
- PCR drift since Step 20: do not enroll.

### Snapshot point

None.

---

## Step 22: TPM2 keyslot enrollment with split PCR policy

**Goal:** Write a TPM2 keyslot to the LUKS2 header with PCR 7 statically bound and PCR 11 bound to signed policy via the policy public key.
**Prerequisites:** Step 21.
**Where to run:** Fedora VM as `root`.
**Last validated:** 2026-05-21.

> [!important] Split policy, not combined
> The natural impulse is `--tpm2-public-key-pcrs=7+11` to bind both PCRs via signed policy. This **does not work**: the hook signs only PCR 11 (`.pcrsig pcrs=[11]`, see Step 20), so a signed binding that includes PCR 7 has no signature entry to match against.
>
> The correct form binds PCR 7 statically because it represents stable Secure
> Boot policy state, and PCR 11 through signed policy because measured UKI
> content can change during updates. See `06F` finding G.1 for the failure mode
> if this is mis-specified.

> [!warning] `--tpm2-signature` is intentionally omitted
> `systemd-cryptenroll --tpm2-signature=<json>` validates the signature against the **current** PCR 11 state before writing the keyslot. The hook signs the **enter-initrd** phase value of PCR 11, not the post-`ready` runtime value. Validation against current PCR 11 always fails outside initrd context, producing `Failed to unseal secret using TPM2: No such device or address`. See `06F` finding G.1.
>
> Skip the dry-run validation; the reboot test in Step 26 is the real end-to-end check.

### Commands

```bash
bash <<'EOF'
set -euo pipefail
DEVICE="/dev/sda3"
PUBKEY="/etc/systemd/tpm2-pcr-public-key.pem"

echo "=== TPM2 enrollment (split policy, no --tpm2-signature) ==="
echo "  PCR 7  : static binding to current Secure Boot policy"
echo "  PCR 11 : signed-policy binding (signature read from UKI at unlock time)"

[[ -f "$PUBKEY" ]] || { echo "FAIL: missing public key"; exit 1; }

systemd-cryptenroll "$DEVICE" \
  --tpm2-device=auto \
  --tpm2-pcrs=7 \
  --tpm2-public-key="$PUBKEY" \
  --tpm2-public-key-pcrs=11 \
  </dev/tty

echo "Enrollment exited successfully."
EOF
```

### Expected output

```
🔐 Please enter current passphrase for disk /dev/sda3: ********
New TPM2 token enrolled as key slot 1.
```

(Exact wording may differ across systemd versions; a clean exit with `key slot 1` is the success signal.)

### Stop condition

- `Failed to unseal secret using TPM2: No such device or address`: either `--tpm2-signature` was passed (do not pass it; see G.1) or the PCR policy shape does not match the embedded `.pcrsig` (the only safe combination is split policy as shown above).
- `Failed to set up TPM2 device`: TPM not reachable; investigate `/dev/tpm0` access.
- Wrong passphrase: re-run.

LUKS header writes are atomic; a failed enrollment cannot leave the header in a half-written state.

### Snapshot point

None yet. Snapshot is taken in Step 26 after the full Module 3 chain validates.

---

## Step 23: Post-enrollment verification

**Goal:** Confirm the new keyslot and token are in the LUKS header with the expected metadata, and that the passphrase keyslot still works. Read-only.
**Prerequisites:** Step 22.
**Where to run:** Fedora VM as `root`.
**Last validated:** 2026-05-21.

### Commands

```bash
bash <<'EOF'
set -euo pipefail
DEVICE="/dev/sda3"

echo "--- systemd-cryptenroll view ---"
systemd-cryptenroll "$DEVICE"

echo
echo "--- LUKS keyslots ---"
cryptsetup luksDump "$DEVICE" \
  | awk '/^Keyslots:/,/^Tokens:/' \
  | grep -E '^[[:space:]]+[0-9]+:|^Keyslots:|^Tokens:|Priority|Cipher:' \
  | head -30

echo
echo "--- Tokens (TPM2 metadata) ---"
cryptsetup luksDump "$DEVICE" | awk '/^Tokens:/,/^Digests:/'

echo
echo "--- passphrase keyslot 0 test ---"
cryptsetup open --test-passphrase --key-slot 0 "$DEVICE" </dev/tty \
  && echo "PASSPHRASE OK (keyslot 0 intact)"

echo
echo "--- dev-mapper unaffected ---"
lsblk -p -o NAME,TYPE,FSTYPE,MOUNTPOINTS /dev/sda3
EOF
```

### Expected output

| Check | Pass criterion |
|---|---|
| `systemd-cryptenroll` | `SLOT 0 password`, `SLOT 1 tpm2` |
| `luksDump` keyslots | `0: luks2` and `1: luks2`, both `aes-xts-plain64` |
| Token 0 type | `systemd-tpm2`, `Keyslot: 1` |
| `tpm2-hash-pcrs` | `7` |
| `tpm2-pubkey-pcrs` | `11` |
| `tpm2-pcr-bank` | `sha256` |
| `tpm2-pubkey` | embedded PEM matches `/etc/systemd/tpm2-pcr-public-key.pem` |
| `tpm2-policy-hash` | committed (32 bytes hex) |
| Passphrase keyslot 0 test | `PASSPHRASE OK` |
| `/dev/mapper/luks-…` | still mounted at `/` |

### Stop condition

- Token wired to wrong keyslot: header is malformed; halt and triage.
- Embedded `tpm2-pubkey` does not match `/etc/systemd/tpm2-pcr-public-key.pem`: wrong key was enrolled; `--wipe-slot=tpm2` and re-enroll with the correct key path.
- Passphrase keyslot test fails: **critical**. Halt all work. Do not reboot. Recovery anchor is gone.

### Snapshot point

None yet.

---

## Step 24: Update crypttab, refresh initramfs, rebuild UKI

**Goal:** Add `tpm2-device=auto` to `/etc/crypttab`, regenerate `/boot/initramfs-${KVER}.img` so the new option is present, then trigger a fresh UKI build via `kernel-install add` so the option propagates into the running boot image.
**Prerequisites:** Step 23.
**Where to run:** Fedora VM as `root`.
**Last validated:** 2026-05-21.

> [!important] Two-step propagation is required
> `kernel-install add` under `KERNEL_INSTALL_LAYOUT=uki` does not refresh `/boot/initramfs-${KVER}.img` when the kernel package is not being installed. The validated 80-tpm2-sign hook reads `/boot/initramfs-${KVER}.img` verbatim as `--initrd=` to `ukify build`. Edits to `/etc/crypttab` therefore require an explicit `dracut --force` before `kernel-install add`, otherwise the UKI is built around a stale initramfs and the crypttab edit never reaches the boot path. See `06F` finding G.2.

### Commands

#### Stage A: edit `/etc/crypttab`

```bash
bash <<'EOF'
set -euo pipefail
CRYPTTAB="/etc/crypttab"

echo "--- before ---"
cat "$CRYPTTAB"

BACKUP="${CRYPTTAB}.pre-module3.$(date +%Y%m%d-%H%M%S)"
cp -a "$CRYPTTAB" "$BACKUP"
echo "Backup: $BACKUP"

# Strict pattern: matches our current line shape only
sed -i 's|\(^luks-[0-9a-f-]\+ UUID=[0-9a-f-]\+ none\) discard$|\1 discard,tpm2-device=auto|' "$CRYPTTAB"

echo "--- after ---"
cat "$CRYPTTAB"
diff "$BACKUP" "$CRYPTTAB" || true

grep -q ',tpm2-device=auto$' "$CRYPTTAB" || { echo "FAIL: sed pattern did not match"; exit 1; }
EOF
```

#### Stage B: refresh `/boot/initramfs` and verify

```bash
bash <<'EOF'
set -euo pipefail
KVER="$(uname -r)"
INITRD="/boot/initramfs-${KVER}.img"

dracut --force "$INITRD" "$KVER"

CRYPTTAB_IN_INITRD="$(lsinitrd "$INITRD" -f /etc/crypttab 2>/dev/null || true)"
printf '%s\n' "$CRYPTTAB_IN_INITRD"

printf '%s\n' "$CRYPTTAB_IN_INITRD" | grep -q 'tpm2-device=auto' \
  || { echo "FAIL: refreshed initramfs missing tpm2-device=auto"; exit 1; }
echo "OK: /boot/initramfs contains tpm2-device=auto"

# TPM2 userspace presence (Step 13 prerequisite: /etc/dracut.conf.d/90-tboot-tpm2.conf
# with add_dracutmodules+=" tpm2-tss "). Without these, the initramfs can read the
# crypttab option but has no userspace to act on it: boot silently falls through
# to passphrase prompt.
TSS2_COUNT="$(lsinitrd "$INITRD" | grep -c 'libtss2' || true)"
SC_PRESENT="$(lsinitrd "$INITRD" | grep -c 'systemd-cryptsetup' || true)"
if [[ "$TSS2_COUNT" -eq 0 || "$SC_PRESENT" -eq 0 ]]; then
  echo "FAIL: TPM2 userspace missing from rebuilt initramfs"
  echo "  libtss2 entries:        $TSS2_COUNT (expected > 0)"
  echo "  systemd-cryptsetup:     $SC_PRESENT (expected > 0)"
  echo "Check /etc/dracut.conf.d/90-tboot-tpm2.conf exists with"
  echo "  add_dracutmodules+=\" tpm2-tss \""
  echo "and that tpm2-tools is installed (see 06B Step 13)."
  exit 1
fi
echo "OK: initramfs contains libtss2 ($TSS2_COUNT entries) and systemd-cryptsetup"
EOF
```

#### Stage C: rebuild UKI and verify embedded crypttab

```bash
bash <<'EOF'
set -euo pipefail
KVER="$(uname -r)"
MID="$(cat /etc/machine-id)"
UKI="/boot/efi/EFI/Linux/${MID}-${KVER}.efi"

SHA_BEFORE="$(sha256sum "$UKI" | awk '{print $1}')"
PRE_TIME="$(date '+%Y-%m-%d %H:%M:%S')"

kernel-install add "$KVER" "/lib/modules/$KVER/vmlinuz"

SHA_AFTER="$(sha256sum "$UKI" | awk '{print $1}')"
[[ "$SHA_BEFORE" != "$SHA_AFTER" ]] || { echo "FAIL: UKI unchanged"; exit 1; }

# Hook journal
ERR_COUNT="$(journalctl -t 80-tpm2-sign --since "$PRE_TIME" --no-pager | grep -c 'error:' || true)"
START_COUNT="$(journalctl -t 80-tpm2-sign --since "$PRE_TIME" --no-pager | grep -c 'info: starting for kernel' || true)"
[[ "$ERR_COUNT" == "0" && "$START_COUNT" == "1" ]] || { echo "FAIL: hook journal"; exit 1; }

# Structural
sbverify --cert /etc/uefi-keys/db.crt "$UKI"
SECTION_COUNT="$(objdump -h "$UKI" | awk '/^[[:space:]]+[0-9]+[[:space:]]+\./ {print $2}' | wc -l)"
[[ "$SECTION_COUNT" == "13" ]] || { echo "FAIL: section count"; exit 1; }

# Verify embedded crypttab (06F G.2 validation)
WORK="$(mktemp -d -p /run)"
trap 'shred -u "$WORK"/* 2>/dev/null || true; rm -rf "$WORK"' EXIT
cp "$UKI" "$WORK/uki.efi"
objcopy --dump-section ".initrd=$WORK/initrd" "$WORK/uki.efi" "$WORK/throwaway.efi"

CRYPTTAB_IN_UKI="$(lsinitrd "$WORK/initrd" -f /etc/crypttab 2>/dev/null || true)"
printf '%s\n' "$CRYPTTAB_IN_UKI" | grep -q 'tpm2-device=auto' \
  || { echo "FAIL: UKI embedded initrd missing tpm2-device=auto"; exit 1; }

echo "OK: UKI sha=$SHA_AFTER, embedded crypttab contains tpm2-device=auto"
EOF
```

### Expected output

| Check | Pass criterion |
|---|---|
| Stage A: sed diff | exactly one line changed, options field gains `,tpm2-device=auto` |
| Stage B: initramfs sha256 | changed |
| Stage B: `lsinitrd` crypttab | contains `tpm2-device=auto` |
| Stage B: TPM2 userspace in initramfs | `libtss2` entries > 0, `systemd-cryptsetup` present |
| Stage C: UKI sha256 | changed |
| Stage C: hook journal | clean, 1 start, 0 errors |
| Stage C: section count | 13 |
| Stage C: embedded UKI crypttab | contains `tpm2-device=auto` |

### Stop condition

- Stage A sed pattern doesn't match: crypttab line shape differs from baseline; halt and adapt pattern manually.
- Stage B initramfs missing option after `dracut --force`: dracut module configuration issue; investigate before continuing.
- Stage C embedded crypttab missing option after kernel-install add: G.2 not fully resolved by the dracut refresh; do not reboot.

### Snapshot point

None yet.

---

## Step 25: Regenerate expected PCR 11 prediction

**Goal:** Predict the post-`ready` runtime PCR 11 value for the newly built UKI and store it as the new architectural invariant.
**Prerequisites:** Step 24.
**Where to run:** Fedora VM as `root`.
**Last validated:** 2026-05-21. Produced `A03EB49C9335D721758AE8AA9C34DA87664C5CAF9FAD7C314C68C038B16435DD` for the post-Module-3 UKI.

> [!note] Two prediction frames
> The value computed here is the **post-`ready` runtime PCR 11**, used by health-check scripts after boot. It is NOT the value the TPM evaluates at unlock time; that's the **enter-initrd** phase value, signed by the hook and embedded in the UKI's `.pcrsig`. Both are derived from the same UKI inputs but represent different phases of the boot sequence.

### Commands

```bash
bash <<'EOF'
set -euo pipefail
KVER="$(uname -r)"
MID="$(cat /etc/machine-id)"
UKI="/boot/efi/EFI/Linux/${MID}-${KVER}.efi"
MEASURE="/usr/lib/systemd/systemd-measure"
STATE_FILE="/root/tboot-lab/state/expected-pcr11-after-hook-uki.txt"

WORK="$(mktemp -d -p /run tboot-pcr11-XXXXXX)"
trap 'shred -u "$WORK"/* 2>/dev/null || true; rm -rf "$WORK"' EXIT

cp "$UKI" "$WORK/uki.efi"

for sec in linux initrd cmdline osrel uname sbat pcrpkey; do
  objcopy --dump-section ".${sec}=$WORK/${sec}" "$WORK/uki.efi" "$WORK/throwaway-${sec}.efi" 2>/dev/null
done

# 06F D.2 parser idiom: combine stdout+stderr, take last 11:sha256= line (post-ready)
PRED_PCR11="$("$MEASURE" calculate \
  --linux="$WORK/linux" --initrd="$WORK/initrd" --cmdline="$WORK/cmdline" \
  --osrel="$WORK/osrel" --uname="$WORK/uname" --sbat="$WORK/sbat" \
  --pcrpkey="$WORK/pcrpkey" --bank=sha256 2>&1 \
  | grep -E '^11:sha256=' | tail -1 | cut -d= -f2 | tr 'a-f' 'A-F')"

[[ -n "$PRED_PCR11" ]] || { echo "FAIL: empty prediction"; exit 1; }

OLD_PCR11="$(cat "$STATE_FILE" 2>/dev/null || echo NONE)"
echo "OLD expected PCR 11: $OLD_PCR11"
echo "NEW expected PCR 11: $PRED_PCR11"

install -d -m 0700 /root/tboot-lab/state
printf '%s\n' "$PRED_PCR11" > "${STATE_FILE}.new"
mv -f "${STATE_FILE}.new" "$STATE_FILE"

cat "$STATE_FILE"
EOF
```

### Expected output

- `PRED_PCR11` is a 64-character uppercased hex string.
- Written atomically to `/root/tboot-lab/state/expected-pcr11-after-hook-uki.txt`.

### Stop condition

- Empty `PRED_PCR11`: `systemd-measure` parse failed; check the `06F` D.2 parser idiom is in use (combined stdout+stderr, `tail -1`).

### Snapshot point

None. Snapshot is taken in Step 26.

---

## Step 26: Pre-reboot snapshot and reboot validation

**Goal:** Snapshot the system at the cliff edge, then reboot and confirm TPM unlock works end-to-end without a passphrase prompt.
**Prerequisites:** Step 25.
**Where to run:** Fedora VM and Proxmox host.
**Last validated:** 2026-05-21. TPM unlock succeeded on first reboot; runtime PCR 11 byte-matched stored prediction `A03EB49C…35DD`; both LUKS keyslots intact post-reboot; `bootctl status` confirmed Secure Boot enabled (user) and Measured UKI yes.

### Commands

```bash
# 1. Capture invariants for the snapshot description (run on VM)
KVER="$(uname -r)"
MID="$(cat /etc/machine-id)"
UKI="/boot/efi/EFI/Linux/${MID}-${KVER}.efi"
UKI_SHA="$(sha256sum "$UKI" | awk '{print $1}')"
EXPECTED_PCR11="$(cat /root/tboot-lab/state/expected-pcr11-after-hook-uki.txt)"
PCR7_NOW="$(cat /sys/class/tpm/tpm0/pcr-sha256/7 | tr 'a-f' 'A-F')"

echo "Snapshot description fields:"
echo "  UKI: $UKI"
echo "  UKI sha256: $UKI_SHA"
echo "  Expected PCR 11: $EXPECTED_PCR11"
echo "  PCR 7: $PCR7_NOW"
```

```bash
# 2. From Proxmox host, take the snapshot (substitute the values above):
qm snapshot 500 module3-tpm2-enrolled-pre-reboot \
  --description "Module 3 enrollment complete: LUKS2 TPM2 keyslot 1 enrolled (split policy: PCR 7 static = <PCR7>, PCR 11 signed via policy pubkey). Keyslot 0 passphrase fallback verified. /etc/crypttab updated with tpm2-device=auto. /boot/initramfs refreshed via dracut --force. UKI rebuilt: <UKI_SHA>. Expected post-ready PCR 11: <EXPECTED_PCR11>. Rollback target if TPM unlock fails after exhausting passphrase fallback and post-boot --wipe-slot=tpm2 recovery."
```

```bash
# 3. Reboot (on VM)
sync
reboot
```

### Expected behaviour (post-reboot)

| Frame | Observation |
|---|---|
| Boot loader | systemd-boot loads MID-prefixed UKI without prompt |
| Initrd unlock | brief delay (1–3 s) while TPM computes policy session, no passphrase prompt |
| Login | normal |
| Post-login: `bootctl status` | Current Entry + Default Entry = MID-prefixed UKI, Secure Boot enabled (user), Measured UKI: yes |
| Post-login: PCR 11 | matches stored expected PCR 11 (`scripts/show_trusted_boot_result.sh`) |
| Post-login: `systemd-cryptenroll /dev/sda3` | both keyslots present |

### Post-reboot verification

After login, run the following block as root. It automates every check in the Expected-behaviour table and produces an explicit `Step 26 verdict:` line. Exit code is non-zero on any failure.

> [!note] Heuristic limit on the journal check
> The final journal grep searches for TPM-related cryptsetup lines as evidence that TPM unlock took. This is a heuristic: a system that booted via passphrase fallback can still emit some TPM-related lines from a failed unlock attempt. Always corroborate with direct observation: did a LUKS passphrase prompt appear during boot? If yes, TPM unlock did not complete regardless of what the journal shows.

```bash
bash <<'EOF'
set -euo pipefail
DEVICE="/dev/sda3"
STATE="/root/tboot-lab/state/expected-pcr11-after-hook-uki.txt"
EXPECTED_PCR11="$(cat "$STATE" | tr 'a-f' 'A-F' | tr -d '\n')"
RUNTIME_PCR11="$(cat /sys/class/tpm/tpm0/pcr-sha256/11 | tr 'a-f' 'A-F')"
echo "=== Step 26: post-reboot TPM unlock validation ==="
echo
echo "--- bootctl ---"
bootctl status 2>/dev/null \
  | grep -E '^\s*(Current Entry:|Default Entry:|Secure Boot:|Measured UKI:|TPM2 Support:)'
echo
echo "--- PCR 11 match ---"
echo "expected: $EXPECTED_PCR11"
echo "runtime:  $RUNTIME_PCR11"
if [[ "$RUNTIME_PCR11" != "$EXPECTED_PCR11" ]]; then
  echo "FAIL: runtime PCR 11 does not match stored Step 25 prediction"
  exit 1
fi
echo "OK: runtime PCR 11 matches expected state file"
echo
echo "--- LUKS enrollment state ---"
systemd-cryptenroll "$DEVICE"
if ! systemd-cryptenroll "$DEVICE" | grep -qE '^[[:space:]]+0[[:space:]]+password$'; then
  echo "FAIL: password recovery slot missing"
  exit 1
fi
if ! systemd-cryptenroll "$DEVICE" | grep -qE '^[[:space:]]+1[[:space:]]+tpm2$'; then
  echo "FAIL: TPM2 slot missing"
  exit 1
fi
echo
echo "--- TPM2 token metadata ---"
cryptsetup luksDump "$DEVICE" | awk '/^Tokens:/,/^Digests:/'
echo
echo "--- systemd-cryptsetup journal evidence ---"
journalctl -b 0 -u 'systemd-cryptsetup@*.service' --no-pager || true
echo
echo "--- TPM-related cryptsetup lines ---"
TPM_LINES="$(journalctl -b 0 -u 'systemd-cryptsetup@*.service' --no-pager \
  | grep -Ei 'tpm2|token|systemd-tpm2|unseal|pcr|activated|set up' || true)"
printf '%s\n' "$TPM_LINES"
if [[ -z "$TPM_LINES" ]]; then
  echo "FAIL: no TPM/token-related evidence in systemd-cryptsetup journal"
  echo "If you saw a LUKS passphrase prompt during boot, TPM unlock did not complete."
  exit 1
fi
echo
echo "Step 26 verdict: PCR11 matches, keyslots intact, TPM-related cryptsetup evidence present."
echo "Operator observation still matters: confirm whether a LUKS passphrase prompt appeared during boot."
EOF
```

### Stop condition

- **Passphrase prompt appears at boot:** TPM unlock failed. Type the LUKS passphrase to boot. Once logged in, run `journalctl -u systemd-cryptsetup@*.service` to see why TPM unlock didn't take. Common causes: PCR 11 drift (UKI on disk ≠ what booted), PCR 7 drift (Secure Boot policy changed), TPM policy session computation error.
- **Boot fails entirely (no passphrase fallback, drops to emergency shell):** systemd-cryptsetup encountered an unrecoverable error. From the emergency shell, manually unlock: `cryptsetup open /dev/sda3 luks-...`, then `mount /dev/mapper/luks-... /sysroot`, then `exit` to continue boot.
- **System doesn't boot at all (firmware can't load UKI):** rollback to `module3-tpm2-enrolled-pre-reboot` from Proxmox.

### Recovery: remove the TPM keyslot

If TPM unlock proves unfixable from inside the booted system:

```bash
systemd-cryptenroll /dev/sda3 --wipe-slot=tpm2
# Returns disk to keyslot 0 (passphrase) only: same as pre-Step-22 baseline
```

Then revisit the policy chain and re-enroll.

### Snapshot point

After verified TPM unlock and post-boot health checks pass:

```bash
qm snapshot 500 module3-complete \
  --description "Module 3 complete. LUKS2 TPM2 keyslot 1 actively unlocks the root volume at boot via TPM policy (PCR 7 static + PCR 11 signed). Passphrase keyslot 0 retained as recovery fallback. Runtime PCR 11 matches stored expected value at /root/tboot-lab/state/expected-pcr11-after-hook-uki.txt. Trusted boot chain end-to-end operational. Next: Module 4 (governance, recovery, ESP cleanup, lifecycle)."
```

✅ Module 3 complete.

---

## Step 27: Install `libdnf5-plugin-actions` and confirm plugin loads

**Goal:** Add the libdnf5 actions plugin to the system as the DNF trigger layer for the dracut-only update path, and confirm it is loaded by libdnf5 at transaction time. The plugin is installed but no action file is written yet, so the trigger fires nothing.
**Prerequisites:** Module 3 complete (`module3-complete` snapshot reachable). Hook sha matches `5857e51d5551e05a7ca384f71529ec2fef07f5a6dbe40df70c6bbe49d720947e`. Booted UKI sha matches the value recorded in `00_Current_Project_State.md`. `dnf upgrade systemd*` discipline gate in force.
**Where to run:** Proxmox host for the snapshot; Fedora VM as `root` for the install.
**Last validated:** 2026-05-21. `libdnf5-plugin-actions-5.2.18.0-3.fc43.x86_64` installed cleanly (1 package, 0 dependencies, no `systemd*` packages in the transaction). Plugin load confirmed in `/var/log/dnf5.log`: `INFO Loaded libdnf plugin "actions" ... version="1.4.0"`.

> [!important] dnf5 has no runtime verbosity flag
> `--debuglevel`, `-v`, and `--verbose` all return `Unknown argument` on dnf5. Plugin-load evidence comes from `/var/log/dnf5.log` after the fact. See `06F` finding H.1.

### Commands

```bash
# 1. Pre-change snapshot (run on Proxmox host pve-host)
qm snapshot 500 module5-b2-pre-actions-install \
  --description "Pre-install libdnf5-plugin-actions; baseline for B.2.1 trigger validation"
```

```bash
# 2. Install (run on VM, interactive: do NOT use -y; review the summary before y/N)
dnf install libdnf5-plugin-actions
# Required to all hold before answering 'y':
#   - Installing: lists exactly libdnf5-plugin-actions, nothing else
#   - No Installing dependencies: section, OR only non-{systemd,kernel,dracut,cryptsetup}* deps
#   - No Upgrading: / Reinstalling: / Removing: / Downgrading: sections
```

```bash
# 3. Verify install integrity
rpm -V libdnf5-plugin-actions                                    # expect empty output
rpm -ql libdnf5-plugin-actions                                   # expect actions.so + actions.conf + actions.d/ + man page
rpm -q --qf '%{name}-%{evr}.%{arch}\n' libdnf5-plugin-actions    # record NVR
```

```bash
# 4. Confirm plugin loads via /var/log/dnf5.log delta.
# Use a harmless package that is NOT currently installed: dnf install --assumeno
# on an already-installed package gets weird resolver semantics. The reference run
# used 'figlet' (its candidate-suitability is established in Step 28); on any other
# rebuild pick a candidate the same way Step 28's candidate-selection probe does,
# or substitute any small, harmless, not-yet-installed package known to be in repos.
DNF_LOG=/var/log/dnf5.log
PRE_LINES=$(wc -l <"$DNF_LOG")
dnf install --assumeno figlet >/dev/null 2>&1 || true   # --assumeno exits non-zero on abort; || true is intentional
POST_LINES=$(wc -l <"$DNF_LOG")
ADDED=$((POST_LINES - PRE_LINES))
echo "dnf5.log delta: $ADDED lines"
tail -n "$ADDED" "$DNF_LOG" | grep -E 'libdnf5/plugins/actions\.so|Loaded libdnf plugin "actions"'
```

### Expected output

`actions.conf` content (read-only inspection):

```
[main]
name = actions
enabled = 1
```

`actions.d/` directory exists and is empty. `rpm -ql` includes `/usr/lib64/libdnf5/plugins/actions.so`, `/etc/dnf/libdnf5-plugins/actions.conf`, `/etc/dnf/libdnf5-plugins/actions.d`, and `/usr/share/man/man8/libdnf5-actions.8.gz`. Plugin-load delta query returns two lines:

```
... DEBUG Loading plugin library file="/usr/lib64/libdnf5/plugins/actions.so"
... INFO Loaded libdnf plugin "actions" ("/usr/lib64/libdnf5/plugins/actions.so"), version="1.4.0"
```

### Stop condition

- Any `systemd*` package in the proposed `dnf install` transaction → answer `N`; discipline gate violation.
- `rpm -V` reports drift, or `rpm -ql` is missing `actions.so` / `actions.conf` / `actions.d` → rollback to `module5-b2-pre-actions-install`.
- Plugin-load delta query returns zero hits → check `actions.conf` for `enabled = 1` and re-run the `--assumeno` probe. If still empty, rollback.

### Snapshot point

None at this step. The next snapshot is in Step 28 after the trigger fires and the six side-effect invariants are validated.

---

## Step 28: Author probe action, run trigger test, clean up

**Goal:** Write a log-only `post_transaction` action, fire it on a real install transaction (`dnf install -y figlet`), and confirm it logs exactly once while the trusted boot chain remains byte-identical across six side-effect invariants. Then remove the test residue.
**Prerequisites:** Step 27.
**Where to run:** Fedora VM as `root`; Proxmox host for the closing snapshot.
**Last validated:** 2026-05-21. `dnf install -y figlet` fired the action exactly once at 23:05:04 under journal tag `tboot-trigger-probe`. UKI sha `75b9cf90…d94a` unchanged, hook sha `a45544…02f` unchanged, action file sha `04cddf3f…95ce` unchanged, initramfs mtime unchanged, `kernel-install` and `80-tpm2-sign` journal tags empty in test window. Two snapshots taken: `module5-b2-1-validated` (evidence, pre-cleanup) and `module5-b2-1-cleaned` (continuation, post-cleanup; figlet removed and probe action renamed to `.disabled-after-b21-validation`).

> [!important] Trigger package: `dnf install` of a not-currently-installed package, not `dnf reinstall`
> `dnf reinstall` requires the installed NEVRA to be present in an enabled repo, which is brittle on long-lived Fedora 43 systems. Use a fresh `dnf install` of a known-not-installed candidate. The validated candidate for the reference rebuild is `figlet`; alternatives that passed the same gates are `cowsay`, `sl`, `pv`. See `06F` finding H.2.

> [!warning] Action rule must never invoke signing-pipeline tools
> The action's `command` field is `/usr/bin/logger`: nothing else. The validated `80-tpm2-sign.install` remains the only UKI signing authority in this project. If a future action file calls `dracut`, `kernel-install`, `ukify`, `sbsign`, or `systemd-measure` directly, B.2's role separation has been broken: that work belongs in the helper script (B.2.2), invoked through `kernel-install add`, not in the action rule.

### Commands

```bash
# 1. Candidate-selection probe (read-only). Confirms figlet is not installed,
# available in repos, owns no sensitive paths, has no dangerous dependencies.
rpm -q figlet || echo "figlet not installed (good)"
dnf repoquery --quiet figlet
dnf repoquery -l figlet | sort -u > /tmp/figlet-files.txt
grep -E '^(/boot|/etc/kernel|/etc/dracut|/usr/lib/dracut|/usr/lib/systemd|/usr/lib/modules|/usr/lib/firmware|/usr/lib/dnf|/etc/dnf)' /tmp/figlet-files.txt \
  || echo "OK: figlet owns no sensitive paths"
dnf install --assumeno figlet     # confirm 1 package, 0 deps, no sensitive packages
```

```bash
# 2. Write the log-only probe action (atomic write via install)
install -m 0644 -o root -g root /dev/stdin \
  /etc/dnf/libdnf5-plugins/actions.d/00-tboot-trigger-probe.actions <<'EOF'
# tboot-trigger-probe: B.2.1 trigger validation, log-only
# Scope:    Log-only via /usr/bin/logger to journal tag "tboot-trigger-probe".
# Boundary: Must never invoke dracut, kernel-install, ukify, sbsign, or
#           systemd-measure. The validated 80-tpm2-sign.install (sha256
#           5857e51d5551e05a7ca384f71529ec2fef07f5a6dbe40df70c6bbe49d720947e)
#           remains the only UKI signing authority in this project.
# Format:   callback_name:package_filter:direction:options:command
#           See man 8 libdnf5-actions.

post_transaction:figlet:in:enabled=1 raise_error=0:/usr/bin/logger -t tboot-trigger-probe post_transaction\ fired\ for\ figlet
EOF

# Verify the write
sha256sum /etc/dnf/libdnf5-plugins/actions.d/00-tboot-trigger-probe.actions
# Expected for this exact content: 04cddf3f90807d53fb5505fcb75abf2cb192f00afb536b71232abf664a2495ce
```

```bash
# 3. Trigger test + six-invariant assertion
bash <<'EOF'
set -euo pipefail
EXPECTED_UKI_SHA="$(sha256sum "/boot/efi/EFI/Linux/$(cat /etc/machine-id)-$(uname -r).efi" | awk '{print $1}')"
EXPECTED_HOOK_SHA="$(sha256sum /etc/kernel/install.d/80-tpm2-sign.install | awk '{print $1}')"
ACTION_FILE="/etc/dnf/libdnf5-plugins/actions.d/00-tboot-trigger-probe.actions"
EXPECTED_ACTION_SHA="$(sha256sum "$ACTION_FILE" | awk '{print $1}')"
INITRAMFS="/boot/initramfs-$(uname -r).img"
INITRAMFS_MTIME="$(stat -c '%Y' "$INITRAMFS")"

rpm -q figlet >/dev/null 2>&1 && { echo "FAIL: figlet already installed"; exit 1; }
PRE_TEST_TIME="$(date -Iseconds)"

dnf install -y figlet

HITS=$(journalctl --since "$PRE_TEST_TIME" -t tboot-trigger-probe --no-pager \
        | grep -c 'post_transaction fired for figlet' || true)
[ "$HITS" -eq 1 ] || { echo "FAIL: action hits=$HITS, expected 1"; exit 20; }

[ "$(sha256sum "/boot/efi/EFI/Linux/$(cat /etc/machine-id)-$(uname -r).efi" | awk '{print $1}')" = "$EXPECTED_UKI_SHA"    ] || { echo "FAIL: UKI moved";       exit 21; }
[ "$(sha256sum /etc/kernel/install.d/80-tpm2-sign.install                    | awk '{print $1}')" = "$EXPECTED_HOOK_SHA"   ] || { echo "FAIL: hook moved";      exit 22; }
[ "$(sha256sum "$ACTION_FILE"                                                | awk '{print $1}')" = "$EXPECTED_ACTION_SHA" ] || { echo "FAIL: action moved";    exit 23; }
[ "$(stat -c '%Y' "$INITRAMFS")" = "$INITRAMFS_MTIME"                                            ] || { echo "FAIL: initramfs moved"; exit 24; }

KI_HITS=$(journalctl --since "$PRE_TEST_TIME" -t kernel-install --no-pager 2>/dev/null | grep -cv '^-- ' || true)
[ "$KI_HITS" -eq 0 ] || { echo "FAIL: kernel-install fired"; exit 25; }
HOOK_HITS=$(journalctl --since "$PRE_TEST_TIME" -t 80-tpm2-sign --no-pager 2>/dev/null | grep -cv '^-- ' || true)
[ "$HOOK_HITS" -eq 0 ] || { echo "FAIL: signing hook fired"; exit 26; }

echo "=== Step 28: B.2.1 trigger validation PASSED ==="
EOF
```

```bash
# 4. Evidence snapshot: captures the validated state BEFORE cleanup mutates it.
# Run on Proxmox host pve-host. See Snapshot point section below for the full command;
# the snapshot must be taken at this point in the sequence, not after cleanup.
echo "Take snapshot module5-b2-1-validated NOW from the Proxmox host before continuing."
echo "Command: qm snapshot 500 module5-b2-1-validated --description \"...\""
echo "Press enter once the snapshot is confirmed in qm listsnapshot 500."
read -r _
```

```bash
# 5. Cleanup: remove figlet and disable the probe action (rename out of .actions suffix)
dnf remove -y figlet
mv /etc/dnf/libdnf5-plugins/actions.d/00-tboot-trigger-probe.actions \
   /etc/dnf/libdnf5-plugins/actions.d/00-tboot-trigger-probe.actions.disabled-after-b21-validation
find /etc/dnf/libdnf5-plugins/actions.d/ -maxdepth 1 -name '*.actions' -type f \
  | grep -q . && echo "WARN: other .actions files present" \
  || echo "OK: no active .actions files. Plugin will not fire until B.2.2 lands."
```

```bash
# 6. Clean continuation snapshot: captures the post-cleanup state.
# Run on Proxmox host pve-host. See Snapshot point section below for the full command;
# this is the rollback anchor for B.2.2.
echo "Take snapshot module5-b2-1-cleaned NOW from the Proxmox host."
echo "Command: qm snapshot 500 module5-b2-1-cleaned --description \"...\""
```

### Expected output

Probe write: file mode `-rw-r--r--.`, owner `root root`, 981 bytes, sha256 `04cddf3f90807d53fb5505fcb75abf2cb192f00afb536b71232abf664a2495ce`.

Trigger test final line: `=== Step 28: B.2.1 trigger validation PASSED ===`. Journal contains exactly one line under tag `tboot-trigger-probe` of the form `tboot-trigger-probe[<pid>]: post_transaction fired for figlet`. All six invariants hold byte-for-byte.

Cleanup: `figlet` no longer in `rpm -q figlet`; `actions.d/` contains only the renamed `.disabled-after-b21-validation` file (preserved as forensic evidence, not active because the plugin only reads files with `.actions` suffix).

### Stop condition

- Exit 20 (action hits ≠ 1): rule did not fire, or fired more than once. Check the action file syntax via `cat`; confirm the `dnf5.log` delta from Step 27 still shows the plugin loading. If still wrong, `rm -f` the action file, rollback to `module5-b2-pre-actions-install`, and re-author.
- Exits 21–26 (any invariant moved): severe. The log-only rule mutated something outside its scope. Rollback to `module5-b2-pre-actions-install` immediately. Do not attempt to debug forward.
- Probe sha256 ≠ `04cddf3f…95ce`: heredoc was mangled by terminal. `rm -f` the file and re-run the `install -m 0644` block; do not edit in place.

### Snapshot point

Two snapshots, taken in order. The first captures the evidence (before cleanup runs) so the validated state is preserved exactly as it was when the six invariants held. The second is the clean continuation anchor for B.2.2.

```bash
# 1. Evidence snapshot: take BEFORE running the cleanup commands above.
# Captures: figlet installed, probe action still named *.actions, journal contains
# the tboot-trigger-probe entry, six invariants confirmed.
qm snapshot 500 module5-b2-1-validated \
  --description "B.2.1 trigger validation evidence: libdnf5-plugin-actions installed; action fired exactly once for figlet; UKI/hook/action/initramfs unchanged; kernel-install and 80-tpm2-sign silent during test window. Pre-cleanup state."
```

```bash
# 2. Clean continuation snapshot: take AFTER running the cleanup commands above
# (figlet removed, probe action renamed to .disabled-after-b21-validation).
# This is the correct rollback anchor for B.2.2.
qm snapshot 500 module5-b2-1-cleaned \
  --description "B.2.1 cleaned continuation state: figlet removed; probe action renamed to .disabled-after-b21-validation (preserved on disk, not active because plugin only reads *.actions). No active .actions files. Correct rollback anchor for B.2.2."
```

✅ Block B.2.1 complete. Trigger layer is validated as orthogonal to the signing pipeline. `module5-b2-1-cleaned` is the rollback anchor for B.2.2; `module5-b2-1-validated` is preserved as evidence and should not be used as a forward continuation point because it carries figlet and the active probe rule.

---

## Step 29: Install B.2.2 helper scripts; self-test and dry-run validation

**Goal:** Place `tboot-dnf-helper` and `tboot-predict-pcr11` at `/usr/local/sbin/`, create state directory `/var/lib/tboot-dnf-helper`, verify shas, run `--self-test` and `--dry-run`. No DNF `.actions` rule wired; helper is operator-invokable but not yet triggered by transactions.

**Prerequisites:** Step 28 closed (`module5-b2-1-cleaned` is the rollback target). This step is self-contained: both `tboot-dnf-helper` and `tboot-predict-pcr11` are embedded as heredocs below and SHA256-verified immediately after staging. The companion files distributed alongside this runbook are extracted/auditable copies; they are not required as inputs to the procedure.

**Where to run:** Proxmox host for the snapshot; Fedora VM as `root` for the install.

**Historical live validation:** 2026-05-22. The predecessor artifacts were installed and self-tested. The current repository revision is `tboot-dnf-helper` v1.0.0 (sha `937afc7a…5e44`) and `tboot-predict-pcr11` v1.0.0 (sha `b729ec27…3de`). These revised artifacts have passed static checks but still require a new live transaction and reboot regression run.

> [!important] Trust-boundary preservation
> The helper never invokes signing tooling directly. It runs `dracut --force` per kernel, then `kernel-install add` per kernel (which fires the validated `80-tpm2-sign.install` hook), then `tboot-predict-pcr11 --store` once against the booted/default UKI. The hook (sha `5857e51d…47e`) at `/etc/kernel/install.d/` remains the only UKI signing authority. The helper is **not** globally fail-atomic across all enumerated kernels: a failure on kernel N+1 does not undo the ESP UKI that was already written for kernels 1..N. What is preserved is the trust boundary itself: every per-kernel UKI on the ESP is the output of a `kernel-install add → 80-tpm2-sign` transaction; nothing else writes signed UKIs. If the helper fails mid-run, rollback uses Step 30's `module5-b2-2-pre-real-run` snapshot (taken before any helper invocation).

> [!note] State directory is installation responsibility, not helper responsibility
> `/var/lib/tboot-dnf-helper` must exist as `root:root 0700` before the helper runs. The helper refuses to write markers if the directory is missing, a symlink, or has the wrong owner or mode. This is deliberate: the helper does not create state directories silently, because a misconfigured one is an audit signal that must not be papered over.

### Commands

```bash
# 1. Pre-install snapshot (run on Proxmox host pve-host)
qm snapshot 500 module5-b2-2-pre-install \
  --description "Pre-B.2.2 helper install. Baseline module5-b2-1-cleaned. No tboot-dnf-helper or tboot-predict-pcr11 anywhere. No /var/lib/tboot-dnf-helper. Rollback target if Step 29 install drifts."
```

```bash
# 2. State directory (run on VM as root)
install -d -m 0700 -o root -g root /var/lib/tboot-dnf-helper
```

```bash
# 3a. Stage tboot-dnf-helper to /tmp via heredoc (full source inline below).
cat > /tmp/tboot-dnf-helper.staged <<'TBOOT_DNF_HELPER_EOF'
#!/bin/bash
#
# /usr/local/sbin/tboot-dnf-helper
#
# B.2.2 helper for the dracut-only update path of the trusted boot chain.
# Invoked from libdnf5-plugin-actions post_transaction rules (after B.2.3
# lands), or directly by an operator for manual rebuilds.
#
# Responsibilities:
#   - Refresh /boot/initramfs-${KVER}.img via `dracut --force` for every
#     kernel under /lib/modules/*/vmlinuz (per 06F finding G.2).
#   - Invoke `kernel-install add ${KVER} /lib/modules/${KVER}/vmlinuz` for
#     each kernel. This is the entry point that fires the validated
#     /etc/kernel/install.d/80-tpm2-sign.install hook: the only UKI signing
#     authority in this project.
#   - Refresh /root/tboot-lab/state/expected-pcr11-after-hook-uki.txt by
#     shelling out to the sibling /usr/local/sbin/tboot-predict-pcr11
#     against the currently-booted kernel's MID-prefixed UKI on the ESP.
#   - Log everything to journald under tag `tboot-dnf-helper`.
#   - Write atomic state markers under /var/lib/tboot-dnf-helper/, but
#     only when that directory exists and passes safety checks.
#
# Trust-boundary preservation. This helper is NOT a signing authority. It
# must never (enforced at review time, not in code):
#   - call ukify directly
#   - call `systemd-measure sign`
#   - call sbsign, or invoke sbverify as a signing step
#   - modify /etc/kernel/uki.conf, /etc/kernel/install.conf, /etc/kernel/cmdline
#   - modify /etc/crypttab
#   - call systemd-cryptenroll; wipe or alter any LUKS keyslot
#   - write PK/KEK/db variables
#   - enable systemd-boot-update.service
#   - create, edit, rename, or remove DNF libdnf5-plugin-actions files
#     (production action wiring is B.2.3's scope, not this script's)
#
# Filesystem mutation policy:
#   - Preflight is READ-ONLY.
#   - The helper NEVER creates STATE_DIR or PCR_STATE_DIR. Both are the
#     install step's responsibility (gate 4). If they are missing or
#     unsafe at run time, marker writes are skipped with a warning to
#     journald; the helper does not attempt to repair them.
#   - The helper's only write paths are: the ESP UKI (indirectly, via the
#     existing kernel-install drop-in chain), /boot/initramfs-*.img (via
#     dracut), the PCR 11 state file (via the predict sibling), markers
#     under STATE_DIR (when safe), and a scratch dir under /run (tmpfs).
#
# References:
#   05_Update_Workflows_and_Key_Storage.md §3.3 (target architecture)
#   06F finding G.2 (dracut-then-kernel-install propagation under UKI layout)
#   06B Step 28      (B.2.1 trigger-layer validation methodology)
#

set -euo pipefail

# ============================================================================
# Constants
# ============================================================================

readonly HELPER_VERSION="1.0.0"
readonly TAG="tboot-dnf-helper"

# Lock and state
readonly LOCK_FILE="/run/${TAG}.lock"
readonly LOCK_TIMEOUT=120
readonly STATE_DIR="/var/lib/${TAG}"
readonly TXN_MARKER="${STATE_DIR}/last-transaction-id"
readonly SUCCESS_MARKER="${STATE_DIR}/last-success"
readonly FAILURE_MARKER="${STATE_DIR}/last-failure"
readonly FAILURE_LOG="${STATE_DIR}/last-failure.log"
readonly STATE_DIR_MODE="700"
readonly STATE_DIR_OWNER="root:root"

# Signing-authority pin (matches 00_Current_Project_State.md invariant table).
# Drift here means the only signing authority has been edited; refuse to run.
readonly SIGNING_HOOK="/etc/kernel/install.d/80-tpm2-sign.install"
readonly SIGNING_HOOK_SHA="5857e51d5551e05a7ca384f71529ec2fef07f5a6dbe40df70c6bbe49d720947e"

# PCR 11 prediction store (canonical project path; not under STATE_DIR).
readonly PCR_STATE_DIR="/root/tboot-lab/state"
readonly PCR_STATE_FILE="${PCR_STATE_DIR}/expected-pcr11-after-hook-uki.txt"
readonly PCR_STATE_DIR_MODE="700"
readonly PCR_STATE_DIR_OWNER="root:root"

# Sibling helpers / required binaries
readonly PREDICT_HELPER="/usr/local/sbin/tboot-predict-pcr11"
readonly KERNEL_INSTALL_BIN="/usr/bin/kernel-install"
readonly DRACUT_BIN="/usr/bin/dracut"
readonly BOOTCTL_BIN="/usr/bin/bootctl"

# Dedup identity sources
readonly BOOT_ID_FILE="/proc/sys/kernel/random/boot_id"

# Hard timeouts (sec)
readonly DRACUT_TIMEOUT=600
readonly KERNEL_INSTALL_TIMEOUT=600
readonly PREDICT_TIMEOUT=120

# ============================================================================
# Mutable state (set during execution)
# ============================================================================

MODE="production"     # production | dry-run | self-test
TRIGGER_PKG="(none)"  # ${pkg.name} from action rule, or "(none)" if manual
SCRATCH=""            # tmpdir on /run (tmpfs) for per-step log capture

# ============================================================================
# Logging
# ============================================================================

log()  { logger -t "$TAG" -- "$*"; printf '[%s] %s\n' "$TAG" "$*" >&2; }
info() { log "info: $*"; }
warn() { log "warn: $*"; }
err()  { log "error: $*"; }

# ============================================================================
# Trap: cleanup only. Failure markers are written by explicit error paths.
# ============================================================================

# shellcheck disable=SC2317,SC2329  # invoked via trap below
_cleanup() {
    local rc=$?
    # Paranoid: never let cleanup recurse or fail loudly
    set +e
    trap - EXIT HUP INT TERM
    if [[ -n "$SCRATCH" && -d "$SCRATCH" ]]; then
        rm -rf "$SCRATCH" 2>/dev/null || true
    fi
    exit "$rc"
}
trap _cleanup EXIT HUP INT TERM

# ============================================================================
# Directory-safety helpers
#
# _check_dir_safe: read-only check; pushes ok/FAIL into a report array.
#                  Used from preflight against STATE_DIR and PCR_STATE_DIR.
#
# _state_dir_safe: same checks, condensed to a single yes/no return.
#                  Used at marker-write time to gate every write.
#
# Neither function mutates the filesystem.
# ============================================================================

_check_dir_safe() {
    local dir="$1" expect_mode="$2" expect_owner="$3"
    local -n _report_ref="$4"
    if [[ -L "$dir" ]]; then
        _report_ref+=("FAIL: ${dir} is a symlink")
        return 1
    fi
    if [[ ! -d "$dir" ]]; then
        _report_ref+=("FAIL: ${dir} does not exist (install step must create it as ${expect_owner} ${expect_mode})")
        return 1
    fi
    local owner mode
    owner="$(stat -c '%U:%G' "$dir" 2>/dev/null || true)"
    mode="$(stat -c '%a' "$dir" 2>/dev/null || true)"
    local bad=0
    if [[ "$owner" != "$expect_owner" ]]; then
        _report_ref+=("FAIL: ${dir} owner=${owner} (expected ${expect_owner})")
        bad=1
    fi
    if [[ "$mode" != "$expect_mode" ]]; then
        _report_ref+=("FAIL: ${dir} mode=${mode} (expected ${expect_mode})")
        bad=1
    fi
    if [[ ! -w "$dir" ]]; then
        _report_ref+=("FAIL: ${dir} not writable by current uid")
        bad=1
    fi
    if (( bad == 0 )); then
        _report_ref+=("ok: ${dir} safe (not symlink, ${expect_owner}, ${expect_mode}, writable)")
        return 0
    fi
    return 1
}

# Yes/no gate for marker writes. NEVER creates STATE_DIR.
_state_dir_safe() {
    [[ ! -L "$STATE_DIR" ]] || return 1
    [[ -d "$STATE_DIR" ]] || return 1
    [[ "$(stat -c '%U:%G' "$STATE_DIR" 2>/dev/null)" == "$STATE_DIR_OWNER" ]] || return 1
    [[ "$(stat -c '%a'   "$STATE_DIR" 2>/dev/null)" == "$STATE_DIR_MODE"  ]] || return 1
    [[ -w "$STATE_DIR" ]] || return 1
    return 0
}

# ============================================================================
# State helpers
#
# Marker writers are best-effort: they check _state_dir_safe at entry, log
# a warning and return 0 if the directory is unsafe or absent, and only
# write when the safety profile holds. The helper NEVER creates STATE_DIR.
# ============================================================================

# Usage: _write_marker_atomic <target_path>  (stdin = marker body)
# Caller must have already confirmed STATE_DIR is safe.
_write_marker_atomic() {
    local target="$1"
    local tmp
    tmp="$(mktemp -p "$STATE_DIR" .marker.XXXXXX)"
    cat > "$tmp"
    chmod 0600 "$tmp"
    mv -f "$tmp" "$target"
}

# Usage: _write_failure_marker <code> <stage> <kver> <detail> [<src_log>]
# Always returns 0: inability to persist is logged to journald and the
# caller's error-path `exit 1` runs regardless.
_write_failure_marker() {
    local code="$1" stage="$2" kver="$3" detail="$4" src_log="${5:-}"

    if ! _state_dir_safe; then
        warn "STATE_DIR ${STATE_DIR} unsafe or absent; failure metadata for stage=${stage} rc=${code} not persisted (journald only)"
        return 0
    fi

    local log_target=""
    if [[ -n "$src_log" && -f "$src_log" ]]; then
        local tmp
        if tmp="$(mktemp -p "$STATE_DIR" .last-failure-log.XXXXXX 2>/dev/null)"; then
            if cat "$src_log" > "$tmp" 2>/dev/null \
               && chmod 0600 "$tmp" 2>/dev/null \
               && mv -f "$tmp" "$FAILURE_LOG" 2>/dev/null; then
                log_target="$FAILURE_LOG"
            else
                rm -f "$tmp" 2>/dev/null || true
                err "could not stage failure log under ${STATE_DIR}"
            fi
        else
            err "could not mktemp under ${STATE_DIR}; failure log not preserved"
        fi
    fi

    _write_marker_atomic "$FAILURE_MARKER" <<EOF
epoch=$(date +%s)
date=$(date -Iseconds)
mode=${MODE}
trigger=${TRIGGER_PKG}
stage=${stage}
kver=${kver}
rc=${code}
detail=${detail}
log=${log_target}
helper_version=${HELPER_VERSION}
EOF
    return 0
}

_write_success_marker() {
    local kvers="$1" final_pcr11="$2"
    if ! _state_dir_safe; then
        warn "STATE_DIR ${STATE_DIR} unsafe or absent; success marker not persisted (journald only)"
        return 0
    fi
    _write_marker_atomic "$SUCCESS_MARKER" <<EOF
epoch=$(date +%s)
date=$(date -Iseconds)
mode=${MODE}
trigger=${TRIGGER_PKG}
kernels=${kvers}
pcr11=${final_pcr11}
helper_version=${HELPER_VERSION}
EOF
    return 0
}

_write_txn_marker() {
    local boot_id="$1" ppid="$2" ppid_start="$3"
    if ! _state_dir_safe; then
        warn "STATE_DIR ${STATE_DIR} unsafe or absent; txn marker not persisted; dedup will not protect a same-transaction re-invocation"
        return 0
    fi
    _write_marker_atomic "$TXN_MARKER" <<EOF
boot_id=${boot_id}
ppid=${ppid}
ppid_start=${ppid_start}
epoch=$(date +%s)
EOF
    return 0
}

# ============================================================================
# Transaction-ID derivation
#
# Identity = (boot_id, ppid, ppid_start_jiffies).
#
#   boot_id          /proc/sys/kernel/random/boot_id (UUID, changes per boot)
#   ppid             the actions plugin / shell invoking the helper
#   ppid_start_jiff  field 22 of /proc/<ppid>/stat (starttime in kernel jiffies)
#
# This triple is robust against PID reuse within a boot AND across boots
# (boot_id catches the latter; starttime catches the former).
#
# Field 22 of /proc/<pid>/stat is the 20th field after the closing ") "
# that terminates the comm field. The "${var##*) }" expansion correctly
# handles comm fields that contain ")" or whitespace.
# ============================================================================

_get_txn_id() {
    local ppid="$1"
    local boot_id="" ppid_start=""
    if [[ -r "$BOOT_ID_FILE" ]]; then
        boot_id="$(< "$BOOT_ID_FILE")"
    fi
    if [[ -r "/proc/${ppid}/stat" ]]; then
        local stat_line rest
        stat_line="$(< "/proc/${ppid}/stat")"
        rest="${stat_line##*) }"
        local -a fields=()
        read -ra fields <<< "$rest"
        # Field 22 of /proc/<pid>/stat = field 20 of post-comm tail = index 19.
        if [[ "${#fields[@]}" -ge 20 ]]; then
            ppid_start="${fields[19]}"
        fi
    fi
    # | is a safe separator (won't appear in UUID or numeric values).
    printf '%s|%s|%s\n' "$boot_id" "$ppid" "$ppid_start"
}

_already_handled() {
    local cur_boot="$1" cur_ppid="$2" cur_start="$3"
    [[ -f "$TXN_MARKER" ]] || return 1
    local prev_boot prev_ppid prev_start
    prev_boot="$(awk -F= '$1=="boot_id"{print $2}' "$TXN_MARKER" 2>/dev/null || true)"
    prev_ppid="$(awk -F= '$1=="ppid"{print $2}' "$TXN_MARKER" 2>/dev/null || true)"
    prev_start="$(awk -F= '$1=="ppid_start"{print $2}' "$TXN_MARKER" 2>/dev/null || true)"
    [[ -n "$prev_boot" && -n "$prev_ppid" && -n "$prev_start" ]] || return 1
    [[ "$prev_boot"  == "$cur_boot"  ]] || return 1
    [[ "$prev_ppid"  == "$cur_ppid"  ]] || return 1
    [[ "$prev_start" == "$cur_start" ]] || return 1
    return 0
}

# ============================================================================
# Preflight: READ-ONLY checks. Never mutates the filesystem.
# Used by --self-test and by every real run before lock acquisition.
# ============================================================================

_preflight() {
    local rc=0
    local report=()

    # Required binaries
    local b
    for b in "$DRACUT_BIN" "$KERNEL_INSTALL_BIN" "$BOOTCTL_BIN" "$PREDICT_HELPER"; do
        if [[ -x "$b" ]]; then
            report+=("ok: $b executable")
        else
            report+=("FAIL: $b missing or not executable")
            rc=1
        fi
    done

    # Signing hook present and pinned
    if [[ -f "$SIGNING_HOOK" ]]; then
        local hook_sha
        hook_sha="$(sha256sum "$SIGNING_HOOK" | awk '{print $1}')"
        if [[ "$hook_sha" == "$SIGNING_HOOK_SHA" ]]; then
            report+=("ok: ${SIGNING_HOOK} sha matches pinned value")
        else
            report+=("FAIL: ${SIGNING_HOOK} sha=${hook_sha} pinned=${SIGNING_HOOK_SHA}")
            rc=1
        fi
    else
        report+=("FAIL: ${SIGNING_HOOK} missing")
        rc=1
    fi

    # /etc/kernel/install.conf: BOOT_ROOT must point at the ESP (read-only)
    if [[ -f /etc/kernel/install.conf ]]; then
        if grep -qE '^BOOT_ROOT=/boot/efi[[:space:]]*$' /etc/kernel/install.conf; then
            report+=("ok: /etc/kernel/install.conf BOOT_ROOT=/boot/efi")
        else
            report+=("FAIL: /etc/kernel/install.conf BOOT_ROOT not set to /boot/efi")
            rc=1
        fi
    else
        report+=("FAIL: /etc/kernel/install.conf missing")
        rc=1
    fi

    # /etc/kernel/cmdline must be non-empty (read-only)
    if [[ -s /etc/kernel/cmdline ]]; then
        report+=("ok: /etc/kernel/cmdline non-empty")
    else
        report+=("FAIL: /etc/kernel/cmdline missing or empty")
        rc=1
    fi

    # ESP must be mounted (read-only)
    if mountpoint -q /boot/efi; then
        report+=("ok: /boot/efi mounted")
    else
        report+=("FAIL: /boot/efi not a mountpoint")
        rc=1
    fi

    # machine-id must be present
    if [[ -s /etc/machine-id ]]; then
        report+=("ok: /etc/machine-id present")
        local mid kver uki
        mid="$(cat /etc/machine-id)"
        kver="$(uname -r)"
        uki="/boot/efi/EFI/Linux/${mid}-${kver}.efi"
        if [[ -f "$uki" ]]; then
            report+=("ok: current-kernel UKI present at ${uki}")
        else
            report+=("warn: current-kernel UKI not yet present at ${uki} (acceptable pre-first-rebuild)")
        fi
    else
        report+=("FAIL: /etc/machine-id missing or empty")
        rc=1
    fi

    # State directory: must exist, not be a symlink, root:root, 0700, writable.
    # Install step creates it: never the helper.
    if ! _check_dir_safe "$STATE_DIR" "$STATE_DIR_MODE" "$STATE_DIR_OWNER" report; then
        rc=1
    fi

    # PCR 11 state directory: same safety profile as STATE_DIR
    if ! _check_dir_safe "$PCR_STATE_DIR" "$PCR_STATE_DIR_MODE" "$PCR_STATE_DIR_OWNER" report; then
        rc=1
    fi

    # Boot-id readable (advisory; affects dedup quality, not correctness)
    if [[ -r "$BOOT_ID_FILE" ]]; then
        report+=("ok: ${BOOT_ID_FILE} readable")
    else
        report+=("warn: ${BOOT_ID_FILE} not readable; dedup quality degraded")
    fi

    local line
    for line in "${report[@]}"; do
        case "$line" in
            FAIL:*) err  "${line#FAIL: }" ;;
            warn:*) warn "${line#warn: }" ;;
            ok:*)   info "${line#ok: }"   ;;
            *)      info "$line"           ;;
        esac
    done

    return "$rc"
}

# ============================================================================
# Kernel enumeration: deterministic order via sort -V
# ============================================================================

_enumerate_kernels() {
    find /lib/modules -mindepth 2 -maxdepth 2 -type f -name vmlinuz -printf '%h\n' 2>/dev/null \
        | sort -V \
        | while IFS= read -r dir; do
            basename "$dir"
        done
}

# ============================================================================
# Current-UKI resolution and default-entry sanity check
# ============================================================================

_resolve_current_uki() {
    local mid kver uki
    mid="$(cat /etc/machine-id 2>/dev/null)" || return 1
    kver="$(uname -r)"
    uki="/boot/efi/EFI/Linux/${mid}-${kver}.efi"
    [[ -f "$uki" ]] || return 1
    printf '%s\n' "$uki"
}

# Advisory: parses `bootctl status` for the Default Entry line and warns if
# its basename does not match the MID-prefixed UKI we are about to predict
# against. bootctl output format drifts across systemd versions, so this is
# warn-and-proceed, not hard-fail.
_check_default_entry_consistent() {
    local mid kver expected
    mid="$(cat /etc/machine-id 2>/dev/null)" || return 0
    kver="$(uname -r)"
    expected="${mid}-${kver}.efi"

    local status
    if ! status="$("$BOOTCTL_BIN" status 2>/dev/null)"; then
        warn "bootctl status returned non-zero; cannot verify default entry"
        return 0
    fi
    if grep -Eq "Default Entry:.*${expected}" <<<"$status"; then
        info "default entry consistent with ${expected}"
        return 0
    fi
    warn "bootctl default entry does not visibly contain ${expected}; PCR11 refresh will target the currently-booted UKI anyway"
    return 0
}

# ============================================================================
# Usage and argument parsing
# ============================================================================

usage() {
    cat >&2 <<EOF
usage: tboot-dnf-helper [--dry-run|--self-test|--version|--help] [<trigger-pkg-name>]

  --self-test  Read-only preflight checks. No lock, no enumeration, no
               mutation. Exits 0 if all checks pass, 1 otherwise.
  --dry-run    Acquire lock, enumerate kernels, log the dracut and
               kernel-install commands that would have run, and the
               prediction command that would have refreshed PCR 11. Does
               not touch initramfs, ESP, prediction file, transaction
               marker, or success marker.
  --version    Print helper version and exit.
  --help, -h   This text.
  (no flag)    Production mode. Refresh initramfs and UKIs, refresh PCR 11
               prediction, write success marker.
EOF
}

# ============================================================================
# Main
# ============================================================================

case "${1:-}" in
    --self-test) MODE="self-test"; shift ;;
    --dry-run)   MODE="dry-run";   shift ;;
    --version)   printf '%s %s\n' "$TAG" "$HELPER_VERSION"; exit 0 ;;
    --help|-h)   usage; exit 0 ;;
    --*)         err "unknown flag: $1"; usage; exit 2 ;;
esac
TRIGGER_PKG="${1:-(none)}"

if [[ "$(id -u)" -ne 0 ]]; then
    err "must run as root (euid=$(id -u))"
    exit 1
fi

info "starting version=${HELPER_VERSION} mode=${MODE} trigger=${TRIGGER_PKG} ppid=${PPID}"

# --- self-test path: no lock, no mutation ---
if [[ "$MODE" == "self-test" ]]; then
    if _preflight; then
        info "self-test: PASS"
        exit 0
    else
        err "self-test: FAIL"
        exit 1
    fi
fi

# --- preflight gates both dry-run and production ---
if ! _preflight; then
    err "preflight failed; refusing to proceed"
    _write_failure_marker 1 "preflight" "(none)" "preflight checks did not pass"
    exit 1
fi

# --- acquire lock ---
exec 9>"$LOCK_FILE"
if ! flock -w "$LOCK_TIMEOUT" 9; then
    err "could not acquire lock ${LOCK_FILE} within ${LOCK_TIMEOUT}s"
    _write_failure_marker 1 "lock" "(none)" "flock -w ${LOCK_TIMEOUT} failed"
    exit 1
fi
info "lock acquired: ${LOCK_FILE}"

# --- in-transaction dedup (production only; dry-run never writes the marker) ---
IFS='|' read -r CUR_BOOT CUR_PPID CUR_START < <(_get_txn_id "$PPID")
DEDUP_USABLE=1
if [[ -z "$CUR_BOOT" || -z "$CUR_PPID" || -z "$CUR_START" ]]; then
    DEDUP_USABLE=0
    warn "could not derive full transaction identity (boot_id=${CUR_BOOT:-?} ppid=${CUR_PPID:-?} start=${CUR_START:-?}); dedup disabled for this run"
fi
if [[ "$MODE" == "production" ]] && (( DEDUP_USABLE == 1 )) \
   && _already_handled "$CUR_BOOT" "$CUR_PPID" "$CUR_START"; then
    info "dedup: already ran for this dnf transaction (boot=${CUR_BOOT} ppid=${CUR_PPID} start=${CUR_START}); exiting 0"
    exit 0
fi

# --- scratch directory on /run (tmpfs): verified before use ---
SCRATCH="$(mktemp -d -p /run -t tboot-helper-XXXXXX)"
chmod 0700 "$SCRATCH"

SCRATCH_FSTYPE="$(stat -f -c '%T' "$SCRATCH")"
if [[ "$SCRATCH_FSTYPE" != "tmpfs" ]]; then
    err "scratch dir ${SCRATCH} is on ${SCRATCH_FSTYPE}, not tmpfs: aborting"
    _write_failure_marker 1 "scratch" "(none)" "scratch dir ${SCRATCH} on ${SCRATCH_FSTYPE}, not tmpfs"
    exit 1
fi
info "scratch on tmpfs: ${SCRATCH}"

# --- enumerate kernels ---
mapfile -t KVERS < <(_enumerate_kernels)
if [[ "${#KVERS[@]}" -eq 0 ]]; then
    err "no kernels found under /lib/modules/*/vmlinuz"
    _write_failure_marker 1 "enumerate" "(none)" "no kernels enumerated"
    exit 1
fi
info "kernels enumerated (${#KVERS[@]}): ${KVERS[*]}"

# --- dry-run path: print intentions, exit clean, no mutation ---
if [[ "$MODE" == "dry-run" ]]; then
    for kver in "${KVERS[@]}"; do
        info "[DRY-RUN] would run: timeout ${DRACUT_TIMEOUT} ${DRACUT_BIN} --force /boot/initramfs-${kver}.img ${kver}"
        info "[DRY-RUN] would run: timeout ${KERNEL_INSTALL_TIMEOUT} ${KERNEL_INSTALL_BIN} add ${kver} /lib/modules/${kver}/vmlinuz"
    done
    info "[DRY-RUN] would resolve current-kernel UKI (MID + uname -r)"
    info "[DRY-RUN] would refresh PCR 11 via: timeout ${PREDICT_TIMEOUT} ${PREDICT_HELPER} --store <current-kernel-UKI>"
    info "[DRY-RUN] complete; no state changed; no markers written"
    exit 0
fi

# --- production path: refresh initramfs and UKIs per kernel ---
for kver in "${KVERS[@]}"; do
    INITRAMFS_PATH="/boot/initramfs-${kver}.img"
    DRACUT_LOG="${SCRATCH}/dracut-${kver}.log"

    info "dracut: starting for ${kver}"
    if timeout "$DRACUT_TIMEOUT" "$DRACUT_BIN" --force "$INITRAMFS_PATH" "$kver" \
            >"$DRACUT_LOG" 2>&1; then
        info "dracut: ok for ${kver}"
    else
        rc=$?
        err "dracut failed for ${kver} (rc=${rc})"
        while IFS= read -r line; do err "  dracut[${kver}]: $line"; done <"$DRACUT_LOG"
        _write_failure_marker "$rc" "dracut" "$kver" \
            "dracut --force returned non-zero or timed out" "$DRACUT_LOG"
        exit 1
    fi

    KI_LOG="${SCRATCH}/kernel-install-${kver}.log"
    info "kernel-install add: starting for ${kver}"
    if timeout "$KERNEL_INSTALL_TIMEOUT" "$KERNEL_INSTALL_BIN" add \
            "$kver" "/lib/modules/${kver}/vmlinuz" >"$KI_LOG" 2>&1; then
        info "kernel-install add: ok for ${kver}"
    else
        rc=$?
        err "kernel-install add failed for ${kver} (rc=${rc})"
        while IFS= read -r line; do err "  kernel-install[${kver}]: $line"; done <"$KI_LOG"
        _write_failure_marker "$rc" "kernel-install" "$kver" \
            "kernel-install add returned non-zero or timed out" "$KI_LOG"
        exit 1
    fi
done

# --- PCR 11 prediction refresh ---
info "resolving current-kernel UKI for PCR 11 refresh"
CURRENT_UKI="$(_resolve_current_uki || true)"
if [[ -z "$CURRENT_UKI" || ! -f "$CURRENT_UKI" ]]; then
    err "current-kernel UKI not resolvable; cannot refresh PCR 11"
    _write_failure_marker 1 "pcr11-resolve" "$(uname -r)" \
        "cannot resolve current-kernel UKI from MID+KVER"
    exit 1
fi
info "current-kernel UKI: ${CURRENT_UKI}"

# Advisory check: warn-and-proceed
_check_default_entry_consistent || true

PREDICT_LOG="${SCRATCH}/predict.log"
info "predict: starting against ${CURRENT_UKI}"
if timeout "$PREDICT_TIMEOUT" "$PREDICT_HELPER" --store "$CURRENT_UKI" \
        >"$PREDICT_LOG" 2>&1; then
    :
else
    rc=$?
    err "predict failed against ${CURRENT_UKI} (rc=${rc})"
    while IFS= read -r line; do err "  predict: $line"; done <"$PREDICT_LOG"
    _write_failure_marker "$rc" "pcr11-predict" "$(uname -r)" \
        "tboot-predict-pcr11 --store returned non-zero or timed out" "$PREDICT_LOG"
    exit 1
fi

FINAL_PCR11="$(awk -F= '/^FINAL_PCR11=/{print $2}' "$PREDICT_LOG" | tail -1)"
if [[ -z "$FINAL_PCR11" ]]; then
    err "could not parse FINAL_PCR11 from predict output"
    while IFS= read -r line; do err "  predict: $line"; done <"$PREDICT_LOG"
    _write_failure_marker 1 "pcr11-parse" "$(uname -r)" \
        "no FINAL_PCR11=... line in predict output" "$PREDICT_LOG"
    exit 1
fi
info "predict: ok; FINAL_PCR11=${FINAL_PCR11}"
info "PCR 11 state file refreshed: ${PCR_STATE_FILE}"

# --- mark transaction handled (only when identity is fully resolved) ---
if (( DEDUP_USABLE == 1 )); then
    _write_txn_marker "$CUR_BOOT" "$CUR_PPID" "$CUR_START"
fi
_write_success_marker "${KVERS[*]}" "$FINAL_PCR11"

info "completed mode=${MODE} kernels=${#KVERS[@]} pcr11=${FINAL_PCR11}"
exit 0
TBOOT_DNF_HELPER_EOF

EXPECTED_HELPER_SHA="937afc7a0d199bf7b7d1a5f5bcccf45c37c0f4f7dd0ad924b2567c4da7125e44"
STAGED_HELPER_SHA="$(sha256sum /tmp/tboot-dnf-helper.staged | awk '{print $1}')"
[ "$STAGED_HELPER_SHA" = "$EXPECTED_HELPER_SHA" ] || { echo "FAIL: helper sha $STAGED_HELPER_SHA"; exit 30; }
```

```bash
# 3b. Stage tboot-predict-pcr11 to /tmp via heredoc (full source inline below).
cat > /tmp/tboot-predict-pcr11.staged <<'TBOOT_PREDICT_PCR11_EOF'
#!/usr/bin/env bash
#
# /usr/local/sbin/tboot-predict-pcr11
#
# Predict the final (post-`ready`) PCR 11 value the system should see at
# userland after booting a UKI signed by the validated
# /etc/kernel/install.d/80-tpm2-sign.install hook. Optionally store the
# prediction at /root/tboot-lab/state/expected-pcr11-after-hook-uki.txt for
# later runtime comparison.
#
# Role: PREDICTION ONLY.
#
# Allowed:
#   - systemd-measure calculate (read-only PCR-chain math)
#   - objdump -h / objcopy --dump-section against a tmpfs copy of the input UKI
#   - atomic write of expected PCR 11 value to the canonical state file
#
# Forbidden (audit boundary; not code-enforced):
#   - systemd-measure sign        (signing authority belongs to 80-tpm2-sign)
#   - ukify build                 (signing authority belongs to 80-tpm2-sign)
#   - sbsign / sbverify (as signing step)
#   - systemd-cryptenroll         (LUKS keyslot authority belongs to Module 3)
#   - any write outside /root/tboot-lab/state/expected-pcr11-after-hook-uki.txt
#     and a tmpfs scratch directory
#
# State-dir policy:
#   - The script NEVER creates ${STATE_DIR}. The install step (gate 4) is
#     responsible for creating it as root:root 0700. When --store is set,
#     the script refuses to write unless the directory exists, is not a
#     symlink, is owned root:root, and has mode 0700.
#
# This script is the canonical sibling of /usr/local/sbin/tboot-dnf-helper.
# It is also safe to invoke standalone for ad-hoc prediction.
#
# Scratch directory lives on /run (tmpfs) and is verified to be tmpfs before
# any UKI section is written into it. The cleanup trap is identical in
# shape to the helper's: rc preservation, recursion disable, then shred + rm.
#
# References:
#   06F finding D    (systemd-measure calculate output format)
#   06B Steps 17, 25 (canonical PCR 11 prediction flow)
#

set -euo pipefail

readonly PREDICT_VERSION="1.0.0"
readonly TAG="tboot-predict-pcr11"

readonly MEASURE="/usr/lib/systemd/systemd-measure"
readonly STATE_DIR="/root/tboot-lab/state"
readonly STATE_FILE="${STATE_DIR}/expected-pcr11-after-hook-uki.txt"
readonly STATE_DIR_MODE="700"
readonly STATE_DIR_OWNER="root:root"

# ============================================================================
# Mutable state (set during execution)
# ============================================================================

WORK=""

# ============================================================================
# Logging
# ============================================================================

log()  { logger -t "$TAG" -- "$*"; printf '[%s] %s\n' "$TAG" "$*" >&2; }
info() { log "info: $*"; }
err()  { log "error: $*"; }

# ============================================================================
# Trap: preserve rc, disable recursion, shred + rm scratch, exit rc
# ============================================================================

# shellcheck disable=SC2317,SC2329  # invoked via trap below
_cleanup() {
    local rc=$?
    set +e
    trap - EXIT HUP INT TERM
    if [[ -n "$WORK" && -d "$WORK" ]]; then
        find "$WORK" -type f -exec shred -u {} + 2>/dev/null || true
        rm -rf "$WORK" 2>/dev/null || true
    fi
    exit "$rc"
}
trap _cleanup EXIT HUP INT TERM

# ============================================================================
# Usage and argument parsing
# ============================================================================

usage() {
    cat >&2 <<EOF
usage: tboot-predict-pcr11 [--store|--version|--help] /path/to/uki.efi

  --store     Write FINAL_PCR11 atomically to ${STATE_FILE}.
              Refuses to write unless ${STATE_DIR} exists, is not a
              symlink, is owned ${STATE_DIR_OWNER}, and has mode ${STATE_DIR_MODE}.
  --version   Print version and exit
  --help, -h  This text
EOF
}

STORE=0
while [[ "${1:-}" == --* ]]; do
    case "$1" in
        --store)     STORE=1; shift ;;
        --version)   printf '%s %s\n' "$TAG" "$PREDICT_VERSION"; exit 0 ;;
        --help|-h)   usage; exit 0 ;;
        *)           err "unknown flag: $1"; usage; exit 2 ;;
    esac
done

UKI="${1:-}"
if [[ -z "$UKI" ]]; then
    usage
    exit 2
fi
if [[ ! -f "$UKI" ]]; then
    err "UKI not found: $UKI"
    exit 2
fi

if [[ "$(id -u)" -ne 0 ]]; then
    err "must run as root (writes under ${STATE_DIR} when --store is set)"
    exit 1
fi

if [[ ! -x "$MEASURE" ]]; then
    err "missing executable: ${MEASURE}"
    exit 1
fi

# ============================================================================
# Scratch directory on /run (tmpfs): verified before use
# ============================================================================

WORK="$(mktemp -d -p /run -t pcr11-predict-XXXXXX)"
chmod 0700 "$WORK"

WORK_FSTYPE="$(stat -f -c '%T' "$WORK")"
if [[ "$WORK_FSTYPE" != "tmpfs" ]]; then
    err "scratch dir ${WORK} is on ${WORK_FSTYPE}, not tmpfs: aborting"
    exit 1
fi

info "predicting PCR 11 for ${UKI}"

# ============================================================================
# Section extraction (against a copy; never run write paths on the source)
# ============================================================================

cp "$UKI" "$WORK/uki.efi"

for sec in linux initrd cmdline osrel uname sbat pcrpkey; do
    if objdump -h "$WORK/uki.efi" \
        | awk -v s=".$sec" '$2==s {found=1} END{exit !found}'; then
        objcopy --dump-section ".$sec=$WORK/$sec" "$WORK/uki.efi"
    else
        err "missing required section .$sec in $UKI"
        exit 1
    fi
done

# ============================================================================
# PCR 11 chain calculation
# ============================================================================

if "$MEASURE" calculate \
        --linux="$WORK/linux" \
        --initrd="$WORK/initrd" \
        --cmdline="$WORK/cmdline" \
        --osrel="$WORK/osrel" \
        --uname="$WORK/uname" \
        --sbat="$WORK/sbat" \
        --pcrpkey="$WORK/pcrpkey" \
        --bank=sha256 \
        >"$WORK/pcr11-calc.txt" 2>"$WORK/pcr11-calc.err"; then
    :
else
    rc=$?
    err "systemd-measure calculate failed (rc=${rc})"
    while IFS= read -r line; do err "  measure: $line"; done <"$WORK/pcr11-calc.err"
    exit "$rc"
fi

# Parse the final (post-`ready`) PCR 11 line. systemd-measure emits one
# 11:sha256=... line per phase; the last is the post-`ready` value, which
# is what userland sees and what the runtime comparison file should hold.
# See 06F finding D.
FINAL="$(grep -oE '11:sha256=[0-9a-f]{64}' "$WORK/pcr11-calc.txt" \
        | tail -1 | cut -d= -f2 | tr 'a-f' 'A-F')"

if [[ -z "$FINAL" ]]; then
    err "failed to parse final PCR 11 from systemd-measure output"
    err "raw systemd-measure stdout follows:"
    while IFS= read -r line; do err "  measure: $line"; done <"$WORK/pcr11-calc.txt"
    exit 1
fi

printf 'FINAL_PCR11=%s\n' "$FINAL"
info "predicted FINAL_PCR11=${FINAL} for ${UKI}"

# ============================================================================
# Optional atomic store
#
# Read-only validation of STATE_DIR safety before any write. The install
# step (gate 4) is responsible for creating ${STATE_DIR} as
# ${STATE_DIR_OWNER} ${STATE_DIR_MODE}. This script NEVER creates it.
# ============================================================================

if [[ "$STORE" -eq 1 ]]; then
    if [[ -L "$STATE_DIR" ]]; then
        err "state directory ${STATE_DIR} is a symlink; refusing to write"
        exit 1
    fi
    if [[ ! -d "$STATE_DIR" ]]; then
        err "state directory ${STATE_DIR} does not exist; --store cannot proceed (install step must create it as ${STATE_DIR_OWNER} ${STATE_DIR_MODE})"
        exit 1
    fi
    sd_owner="$(stat -c '%U:%G' "$STATE_DIR" 2>/dev/null || true)"
    sd_mode="$(stat -c '%a'   "$STATE_DIR" 2>/dev/null || true)"
    if [[ "$sd_owner" != "$STATE_DIR_OWNER" ]]; then
        err "state directory ${STATE_DIR} owner=${sd_owner} (expected ${STATE_DIR_OWNER}); refusing to write"
        exit 1
    fi
    if [[ "$sd_mode" != "$STATE_DIR_MODE" ]]; then
        err "state directory ${STATE_DIR} mode=${sd_mode} (expected ${STATE_DIR_MODE}); refusing to write"
        exit 1
    fi
    if [[ ! -w "$STATE_DIR" ]]; then
        err "state directory ${STATE_DIR} not writable by current uid; refusing to write"
        exit 1
    fi

    TMP="$(mktemp -p "$STATE_DIR" .expected-pcr11.XXXXXX)"
    printf '%s\n' "$FINAL" >"$TMP"
    chmod 0600 "$TMP"
    mv -f "$TMP" "$STATE_FILE"
    printf 'stored: %s\n' "$STATE_FILE"
    info "stored ${STATE_FILE}"
fi

exit 0
TBOOT_PREDICT_PCR11_EOF

EXPECTED_PREDICT_SHA="b729ec271dd76cd8b4581f16c86e50fc28b947f2a1de5a7e16c6d5828591e3de"
STAGED_PREDICT_SHA="$(sha256sum /tmp/tboot-predict-pcr11.staged | awk '{print $1}')"
[ "$STAGED_PREDICT_SHA" = "$EXPECTED_PREDICT_SHA" ] || { echo "FAIL: predict sha $STAGED_PREDICT_SHA"; exit 31; }
```

```bash
# 3c. Install atomically; shred staging files.
install -m 0755 -o root -g root /tmp/tboot-dnf-helper.staged    /usr/local/sbin/tboot-dnf-helper
install -m 0755 -o root -g root /tmp/tboot-predict-pcr11.staged /usr/local/sbin/tboot-predict-pcr11
shred -u /tmp/tboot-dnf-helper.staged /tmp/tboot-predict-pcr11.staged 2>/dev/null \
  || rm -f /tmp/tboot-dnf-helper.staged /tmp/tboot-predict-pcr11.staged
```

```bash
# 4. Verify installed shas, mode, owner
sha256sum /usr/local/sbin/tboot-dnf-helper /usr/local/sbin/tboot-predict-pcr11
stat -c 'mode=%a owner=%U:%G  type=%F  %n' /usr/local/sbin/tboot-dnf-helper /usr/local/sbin/tboot-predict-pcr11
```

```bash
# 5. Self-test (preflight only; no lock acquired, no scratch, no mutation)
/usr/local/sbin/tboot-dnf-helper --self-test
# Expect: 13 [tboot-dnf-helper] info: ok: ... lines + 'info: self-test: PASS' + exit 0
```

```bash
# 6. Post-install snapshot (run on Proxmox host pve-host)
qm snapshot 500 module5-b2-2-helper-installed \
  --description "Helper and predict installed at /usr/local/sbin/tboot-{dnf-helper,predict-pcr11} (root:root 0755). /var/lib/tboot-dnf-helper created (root:root 0700). --self-test PASS. No DNF .actions rule wired. Boot chain artefacts unchanged. Rollback anchor before B.2.2 dry-run + production gates."
```

```bash
# 7. Dry-run + 10-invariant assertion
bash <<'EOF'
set -uo pipefail
KVER="$(uname -r)"; MID="$(cat /etc/machine-id)"
UKI="/boot/efi/EFI/Linux/${MID}-${KVER}.efi"
INITRAMFS="/boot/initramfs-${KVER}.img"
HOOK="/etc/kernel/install.d/80-tpm2-sign.install"
HELPER="/usr/local/sbin/tboot-dnf-helper"
PREDICT="/usr/local/sbin/tboot-predict-pcr11"
PCR11_FILE="/root/tboot-lab/state/expected-pcr11-after-hook-uki.txt"

EXP_UKI="$(sha256sum "$UKI" | awk '{print $1}')"
EXP_HOOK="$(sha256sum "$HOOK" | awk '{print $1}')"
EXP_HELPER="$(sha256sum "$HELPER" | awk '{print $1}')"
EXP_PREDICT="$(sha256sum "$PREDICT" | awk '{print $1}')"
EXP_INIT="$(stat -c '%Y' "$INITRAMFS")"
EXP_PCR11="$(cat "$PCR11_FILE")"

PRE="$(date -Iseconds)"
/usr/local/sbin/tboot-dnf-helper --dry-run || { echo "FAIL: dry-run rc=$?"; exit 40; }

[ "$(sha256sum "$UKI"     | awk '{print $1}')" = "$EXP_UKI"     ] || { echo "FAIL: UKI moved";       exit 41; }
[ "$(sha256sum "$HOOK"    | awk '{print $1}')" = "$EXP_HOOK"    ] || { echo "FAIL: hook moved";      exit 42; }
[ "$(sha256sum "$HELPER"  | awk '{print $1}')" = "$EXP_HELPER"  ] || { echo "FAIL: helper moved";    exit 43; }
[ "$(sha256sum "$PREDICT" | awk '{print $1}')" = "$EXP_PREDICT" ] || { echo "FAIL: predict moved";   exit 44; }
[ "$(stat -c '%Y' "$INITRAMFS")" = "$EXP_INIT"  ] || { echo "FAIL: initramfs mtime moved"; exit 45; }
[ "$(cat "$PCR11_FILE")"         = "$EXP_PCR11" ] || { echo "FAIL: PCR 11 file changed";   exit 46; }

KI=$(journalctl --since "$PRE" -t kernel-install --no-pager | grep -cv '^-- ' || true)
SH=$(journalctl --since "$PRE" -t 80-tpm2-sign   --no-pager | grep -cv '^-- ' || true)
[ "$KI" -eq 0 ] || { echo "FAIL: kernel-install fired"; exit 47; }
[ "$SH" -eq 0 ] || { echo "FAIL: signing hook fired";   exit 48; }
[ ! -e /var/lib/tboot-dnf-helper/last-success        ] || { echo "FAIL: success marker"; exit 49; }
[ ! -e /var/lib/tboot-dnf-helper/last-transaction-id ] || { echo "FAIL: txn marker";     exit 50; }
[ ! -e /var/lib/tboot-dnf-helper/last-failure        ] || { echo "FAIL: failure marker"; exit 51; }
[ ! -e /var/lib/tboot-dnf-helper/last-failure.log    ] || { echo "FAIL: failure log";    exit 52; }

echo "=== Step 29 dry-run validation PASSED ==="
EOF
```

### Expected output

Step 3: post-install verification reports helper sha `937afc7a…5e44`, predict sha `b729ec27…3de`, both files `mode=755 owner=root:root type=regular file`.

Step 5: 13 preflight `ok:` lines (4 binaries, hook sha matches pinned value, `BOOT_ROOT=/boot/efi`, `/etc/kernel/cmdline` non-empty, ESP mounted, machine-id present, current-kernel UKI present, both state dirs safe, `boot_id` readable), `info: self-test: PASS`, exit 0.

Step 7: helper journal shows preflight + `lock acquired` + `scratch on tmpfs` + `kernels enumerated (N): …` + 2N `[DRY-RUN] would run:` lines (dracut + kernel-install per kernel) + `[DRY-RUN] complete; no state changed; no markers written`. Final line: `=== Step 29 dry-run validation PASSED ===`.

### Stop condition

- Exit 30/31 (sha mismatch on staged): source file corruption. Re-copy from canonical project location and re-verify.
- Exit 40 (`--dry-run` rc != 0): preflight failed. Read `[tboot-dnf-helper] error:` line. Most common: state directory missing or wrong ACL (re-run step 2). Helper journal-only output; no markers written.
- Exits 41-52 (any invariant moved or unexpected marker present): severe; dry-run mode mutated state, indicating a bug in the helper's mode dispatch. Rollback to `module5-b2-2-pre-install`. Do not proceed.

### Snapshot point

Two snapshots, taken in order. `module5-b2-2-pre-install` before the install. `module5-b2-2-helper-installed` after the install + self-test. Dry-run does not warrant its own snapshot; it is read-only.

---

## Step 30: B.2.2 production helper invocation; reboot validation

**Goal:** First state-changing helper run. Verify the dracut + kernel-install + signing-hook chain produces correctly signed, `.pcrsig`-attested UKIs for every installed kernel. Verify expected-PCR-11 state file refreshed for the booted/default kernel. Reboot under TPM enforcement; confirm runtime PCR 11 matches stored prediction and LUKS unlocks without passphrase.

**Prerequisites:** Step 29.

**Where to run:** Proxmox host for snapshots; Fedora VM as `root` for the invocation, assertion, and validation. Proxmox console must be reachable in case TPM unlock fails and a passphrase prompt appears.

**Last validated:** 2026-05-22. Two kernels processed (6.17.1-300.fc43.x86_64 secondary, 6.19.14-200.fc43.x86_64 booted/default) in 83 seconds. Helper exit 0; full lifecycle journal; both UKIs sbverify-OK against db.crt with `.pcrpkey` + `.pcrsig`. PCR 11 file rewritten with same value (`A03EB49C…35DD`) because measured-content inputs were bit-identical; see `06F` E.3. Post-reboot: boot_id changed; runtime PCR 11 byte-matched stored prediction; LUKS unlocked via TPM2 without passphrase prompt (operator-confirmed); bootctl reports `Measured UKI: yes`. Two snapshots taken: `module5-b2-2-pre-real-run` (required rollback target), `module5-b2-2-real-run-validated` (B.2.2 closure anchor). The optional forensic snapshot `module5-b2-2-post-helper-pre-reboot` (step 3 below) was **not** taken in this run; the pre-real-run snapshot already serves as the rollback target.

> [!warning] Cliff edge
> The helper invocation mutates ESP UKIs and initramfs for every installed kernel. The subsequent reboot crosses a second cliff edge: TPM unlock under enforcement. If TPM unlock fails, the Proxmox console will display the LUKS passphrase prompt. Type the keyslot 0 passphrase to recover, then rollback to `module5-b2-2-pre-real-run`.

> [!note] Secondary kernels are processed by design
> The helper enumerates all `/lib/modules/*/vmlinuz`. Secondary installed kernels (installed but not currently booted) receive new MID-prefixed UKIs as a side effect; these are signed and `.pcrsig`-attested but not reboot-validated by this step. They are forensic artefacts pending Module 4 governance review. Default boot entry is unchanged.

### Commands

```bash
# 1. Pre-real-run snapshot (run on Proxmox host pve-host)
qm snapshot 500 module5-b2-2-pre-real-run \
  --description "Pre-first-real-helper-invocation. Helper installed, self-test PASS, dry-run validated. Rollback target if helper invocation or reboot fails."
```

```bash
# 2. Production-mode helper invocation + post-invocation assertion (VM as root).
# Helper non-zero -> wrapping shell non-zero -> halt before reboot.
bash <<'EOF'
set -uo pipefail
PRE_TEST_TIME="$(date -Iseconds)"
PRE_TEST_EPOCH="$(date -d "$PRE_TEST_TIME" +%s)"
printf '%s\n' "$PRE_TEST_TIME" > /run/tboot-gate6-pretime

KVER="$(uname -r)"; MID="$(cat /etc/machine-id)"
HOOK="/etc/kernel/install.d/80-tpm2-sign.install"
HELPER="/usr/local/sbin/tboot-dnf-helper"
PREDICT="/usr/local/sbin/tboot-predict-pcr11"
PCR11_FILE="/root/tboot-lab/state/expected-pcr11-after-hook-uki.txt"
UKI_BOOTED="/boot/efi/EFI/Linux/${MID}-${KVER}.efi"

EXP_HOOK="$(sha256sum "$HOOK"    | awk '{print $1}')"
EXP_HELPER="$(sha256sum "$HELPER"  | awk '{print $1}')"
EXP_PREDICT="$(sha256sum "$PREDICT" | awk '{print $1}')"
PRE_UKI_BOOTED="$(sha256sum "$UKI_BOOTED" | awk '{print $1}')"

declare -A PRE_INIT
for V in /lib/modules/*/vmlinuz; do
  K="$(basename "$(dirname "$V")")"
  PRE_INIT[$K]="$(stat -c '%Y' "/boot/initramfs-${K}.img" 2>/dev/null || echo 0)"
done

/usr/local/sbin/tboot-dnf-helper
HELPER_RC=$?
if [ "$HELPER_RC" -ne 0 ]; then
  echo "FAIL: helper rc=$HELPER_RC; do not reboot; triage /var/lib/tboot-dnf-helper/last-failure"
  exit "$HELPER_RC"
fi

# Invariants that must NOT change
[ "$(sha256sum "$HOOK"    | awk '{print $1}')" = "$EXP_HOOK"    ] || { echo "FAIL: hook drifted";    exit 61; }
[ "$(sha256sum "$HELPER"  | awk '{print $1}')" = "$EXP_HELPER"  ] || { echo "FAIL: helper drifted";  exit 62; }
[ "$(sha256sum "$PREDICT" | awk '{print $1}')" = "$EXP_PREDICT" ] || { echo "FAIL: predict drifted"; exit 63; }
[ "$(find /etc/dnf/libdnf5-plugins/actions.d/ -maxdepth 1 -name '*.actions' -type f | wc -l)" -eq 0 ] \
  || { echo "FAIL: active .actions present (B.2.3 deferred)"; exit 64; }
bootctl status 2>/dev/null | grep -Eq "Default Entry:.*${MID}-${KVER}\.efi" \
  || { echo "FAIL: default boot entry changed"; exit 65; }

# Expected state changes: per-kernel initramfs mtime + UKI present + sbverify + TPM2 userspace
for V in /lib/modules/*/vmlinuz; do
  K="$(basename "$(dirname "$V")")"
  NEW_INIT="$(stat -c '%Y' "/boot/initramfs-${K}.img")"
  [ "$NEW_INIT" -gt "${PRE_INIT[$K]}" ] || { echo "FAIL: ${K} initramfs mtime did not advance"; exit 66; }
  U="/boot/efi/EFI/Linux/${MID}-${K}.efi"
  [ -f "$U" ] || { echo "FAIL: ${K} UKI absent"; exit 67; }
  objdump -h "$U" | awk '$2==".pcrpkey"{p=1} $2==".pcrsig"{s=1} END{exit !(p && s)}' \
    || { echo "FAIL: ${K} UKI missing .pcrpkey or .pcrsig"; exit 68; }
  sbverify --cert /etc/uefi-keys/db.crt "$U" >/dev/null 2>&1 \
    || { echo "FAIL: ${K} UKI sbverify failed"; exit 69; }
  T="$(lsinitrd "/boot/initramfs-${K}.img" 2>/dev/null | grep -c 'libtss2'          || true)"
  S="$(lsinitrd "/boot/initramfs-${K}.img" 2>/dev/null | grep -c 'systemd-cryptsetup' || true)"
  [ "$T" -gt 0 ] && [ "$S" -gt 0 ] || { echo "FAIL: ${K} initramfs missing TPM2 userspace"; exit 70; }
done

# Booted-kernel UKI sha must differ (was rebuilt)
[ "$(sha256sum "$UKI_BOOTED" | awk '{print $1}')" != "$PRE_UKI_BOOTED" ] \
  || { echo "FAIL: booted-kernel UKI sha did not change"; exit 71; }

# PCR 11 file: mtime must advance (proves predict --store ran). Value may or may
# not change; see 06F E.3.
[ "$(stat -c '%Y' "$PCR11_FILE")" -gt "$PRE_TEST_EPOCH" ] \
  || { echo "FAIL: PCR 11 file mtime did not advance"; exit 72; }

# Markers
[ -f /var/lib/tboot-dnf-helper/last-success        ] || { echo "FAIL: success marker missing"; exit 73; }
[ -f /var/lib/tboot-dnf-helper/last-transaction-id ] || { echo "FAIL: txn marker missing";     exit 74; }
[ ! -e /var/lib/tboot-dnf-helper/last-failure      ] || { echo "FAIL: failure marker present"; exit 75; }
[ ! -e /var/lib/tboot-dnf-helper/last-failure.log  ] || { echo "FAIL: failure log present";    exit 76; }

# Hook fired once per installed kernel
N_KERNELS=$(ls /lib/modules/*/vmlinuz | wc -l)
HOOK_HITS=$(journalctl -t 80-tpm2-sign --since "$PRE_TEST_TIME" --no-pager 2>/dev/null \
            | grep -cE 'kernel [0-9].*ukify-native signed UKI ready' || true)
[ "$HOOK_HITS" -eq "$N_KERNELS" ] \
  || { echo "FAIL: 80-tpm2-sign fired $HOOK_HITS times, expected $N_KERNELS"; exit 77; }

echo "=== Step 30 post-invocation PASSED (helper rc=0, $N_KERNELS kernels processed) ==="
EOF
```

```bash
# 3. Optional forensic snapshot (Proxmox host). Not a rollback target. Take it
# only if you want a separate post-helper, pre-reboot anchor for forensic
# comparison. The required rollback target remains module5-b2-2-pre-real-run
# (step 1). Skipping step 3 is the default; the reference 2026-05-22 run did
# not take it.
#
# qm snapshot 500 module5-b2-2-post-helper-pre-reboot \
#   --description "Optional post-helper, pre-reboot forensic anchor. Helper completed; all kernels regenerated and signed. Not a rollback target; module5-b2-2-pre-real-run remains the rollback target on hard failure."
```

```bash
# 4. Reboot (VM as root). SSH disconnects.
# Capture the expected post-reboot kernel persistently. /run is tmpfs and
# is wiped at reboot, so the marker lives under the helper state directory.
echo "$(uname -r)" > /var/lib/tboot-dnf-helper/expected-boot-kver
chmod 0600 /var/lib/tboot-dnf-helper/expected-boot-kver
echo "Recovery: keyslot 0 passphrase at Proxmox console if TPM unlock fails."
echo "Rollback target: module5-b2-2-pre-real-run."
sync; sleep 3; systemctl reboot
```

```bash
# 5. After SSH reconnect: post-reboot validation
bash <<'EOF'
set -uo pipefail
KVER="$(uname -r)"; MID="$(cat /etc/machine-id)"
PCR11_FILE="/root/tboot-lab/state/expected-pcr11-after-hook-uki.txt"
PRE_BOOT_ID="$(grep '^boot_id=' /var/lib/tboot-dnf-helper/last-transaction-id | cut -d= -f2)"
NOW_BOOT_ID="$(cat /proc/sys/kernel/random/boot_id)"
EXPECTED_KVER="$(cat /var/lib/tboot-dnf-helper/expected-boot-kver 2>/dev/null)"

[ -n "$EXPECTED_KVER" ] || { echo "FAIL: expected-boot-kver marker missing or empty"; exit 79; }
[ "$NOW_BOOT_ID" != "$PRE_BOOT_ID" ] || { echo "FAIL: boot_id unchanged"; exit 80; }
[ "$KVER" = "$EXPECTED_KVER" ] || { echo "FAIL: booted kernel=$KVER, expected=$EXPECTED_KVER"; exit 81; }

EXPECTED_ENTRY="${MID}-${EXPECTED_KVER}.efi"
[ "$(bootctl status | awk '/Current Entry:/{print $3}')" = "$EXPECTED_ENTRY" ] || { echo "FAIL: Current Entry"; exit 82; }
[ "$(bootctl status | awk '/Default Entry:/{print $3}')" = "$EXPECTED_ENTRY" ] || { echo "FAIL: Default Entry"; exit 83; }
bootctl status 2>/dev/null | grep -q 'Measured UKI: yes' || echo "note: Measured UKI not reported; inspect manually"

RUNTIME_PCR11="$(tr -d ' \n' < /sys/class/tpm/tpm0/pcr-sha256/11 | tr 'a-f' 'A-F')"
STORED_PCR11="$(tr -d '\n' < "$PCR11_FILE")"
[ "$RUNTIME_PCR11" = "$STORED_PCR11" ] || { echo "FAIL: runtime PCR 11 != stored prediction"; exit 84; }

LUKS_NAME="$(awk '$1 ~ /^luks-/ {print $1; exit}' /etc/crypttab)"
[ -b "/dev/mapper/${LUKS_NAME}" ] || { echo "FAIL: LUKS not open"; exit 85; }

# Passphrase keyword is a hard failure. Absence of TPM keyword alone is not;
# the post-switch-root cryptsetup unit may not log TPM evidence even on TPM
# unlock. Operator must confirm no passphrase prompt at Proxmox console.
CRYPT_LOG="$(journalctl --boot -u 'systemd-cryptsetup@*.service' --no-pager 2>/dev/null || true)"
PASS_HITS="$(printf '%s\n' "$CRYPT_LOG" | grep -ciE 'Password|passphrase|Please enter|Enter passphrase' || true)"
[ "$PASS_HITS" -eq 0 ] || { echo "FAIL: passphrase evidence in cryptsetup journal"; exit 86; }

[ ! -e /var/lib/tboot-dnf-helper/last-failure ] || { echo "FAIL: failure marker appeared"; exit 87; }
[ ! -e /var/lib/tboot-dnf-helper/last-failure.log ] || { echo "FAIL: failure log appeared"; exit 88; }
[ -f /var/lib/tboot-dnf-helper/last-success ] || { echo "FAIL: success marker missing"; exit 89; }
RECORDED_PCR11="$(grep '^pcr11=' /var/lib/tboot-dnf-helper/last-success | cut -d= -f2)"
[ "$RECORDED_PCR11" = "$STORED_PCR11" ] || { echo "FAIL: last-success pcr11=$RECORDED_PCR11 != stored=$STORED_PCR11"; exit 90; }

echo "=== Step 30 post-reboot validation PASSED ==="
echo "Operator confirmation required: no LUKS passphrase prompt appeared at boot."
EOF
```

```bash
# 6. Closure snapshot (Proxmox host); ONLY if Step 5 passed all automated checks
# AND operator confirmed no passphrase prompt at boot.
qm snapshot 500 module5-b2-2-real-run-validated \
  --description "Gate B.2.2 closed: helper end-to-end validated. Runtime PCR 11 byte-matches stored prediction. LUKS unlocked via TPM2 without passphrase (operator-confirmed). Default/Current Entry consistent with MID-prefixed booted UKI. Secondary kernel UKIs present on ESP but not boot-validated (forensic, Module 4 cleanup target)."
```

### Expected output

Step 2: helper journal shows preflight + `lock acquired` + `scratch on tmpfs` + `kernels enumerated (N): …` + per-kernel `dracut: starting`/`ok` and `kernel-install add: starting`/`ok` (4N lines) + `resolving current-kernel UKI for PCR 11 refresh` + `predict: ok; FINAL_PCR11=…` + `PCR 11 state file refreshed` + `completed mode=production kernels=N pcr11=…`. Final line: `=== Step 30 post-invocation PASSED (helper rc=0, N kernels processed) ===`.

Step 5: all checks `ok:` followed by `=== Step 30 post-reboot validation PASSED ===` and the operator-confirmation reminder.

Step 6: `qm listsnapshot 500 | tail -3` shows `module5-b2-2-real-run-validated → current`.

### Stop condition

- Step 2 helper rc != 0: read `/var/lib/tboot-dnf-helper/last-failure` and `last-failure.log`. **Do not reboot.** Rollback to `module5-b2-2-pre-real-run`.
- Step 2 exits 61-77 (any invariant FAIL): helper completed but produced inconsistent state. **Do not reboot.** Rollback to `module5-b2-2-pre-real-run`.
- Step 5 exits 79-90 (post-reboot check FAIL): boot completed but a runtime invariant failed. Read the `FAIL:` line. Common causes: missing `expected-boot-kver` marker (exit 79: step 4 reboot block skipped or interrupted); PCR 11 mismatch (measured-content chain drifted between predict and runtime; rare); passphrase prompt appeared (TPM unlock failed; operator typed passphrase to recover); `last-success` marker value diverged from `STORED_PCR11`. Rollback target: `module5-b2-2-pre-real-run`.
- LUKS passphrase prompt at Proxmox console during boot: TPM unlock failed. Type keyslot 0 passphrase to recover. **Step 30 is not passed.** Do not take `module5-b2-2-real-run-validated`. Rollback after triage.

### Snapshot points

Two required snapshots: `module5-b2-2-pre-real-run` (taken before step 2; rollback target on hard failure) and `module5-b2-2-real-run-validated` (taken after step 5 PASS and operator passphrase confirmation; B.2.2 closure anchor). One optional snapshot: `module5-b2-2-post-helper-pre-reboot` (step 3 above): skip it unless a separate forensic anchor is needed. The reference 2026-05-22 run did not take the optional snapshot.

✅ Block B.2.2 complete. Helper is production-validated for the dracut-only update path. The next block, B.2.3, places a root-owned **decider** (`tboot-dnf-posttrans`) in front of the helper: an always-run `post_transaction` `.actions` rule invokes the decider, which computes a conservative boot-input manifest, compares against a primed baseline, and invokes the helper only on detected drift. B.2.3 is split into seven gates: Gate 1 (design lock) and Gate 2 (decider staged + external read-back validation) are non-mutating and run before any install; Gate 3 installs the decider and primes the baseline; Gates 4–7 install the rule, run negative validation, and close B.2.3. Gate 1 and Gate 2 are covered in Step 31 below. Gates 3–7 and B.2.4 will be appended once executed.

---

## Step 31: Block B.2.3 Gate 1 + Gate 2: design lock and staged-decider external read-back validation

**Goal:** Close the two non-mutating B.2.3 gates. Gate 1 freezes the `.actions` rule shape, the 14-category boot-input manifest, the decider-vs-helper trust boundary, and the lock model in a design-decisions notes file. Gate 2 authors the decider source (`tboot-dnf-posttrans`), stages it at `/tmp/tboot-dnf-posttrans.staged`, and externally validates it through seven read-only checks: pre-stage invariants, syntax parse, ShellCheck, trust-boundary token scan (Deviation D regex), helper-lock-path scan, `if !`-rc-loss regression scan, and staging-only invariants. No install, no execution, no state-dir creation, no `.actions` file, no helper invocation, no Secure Boot / LUKS / ESP / `/etc/kernel/*` mutation.

**Prerequisites:** Step 30 closed (`module5-b2-2-real-run-validated` is the rollback target).

**Where to run:** Fedora VM as `root` (in `tmux`). No Proxmox host work: Gate 2 does not take a snapshot.

**Last validated:** 2026-05-24: staged decider sha256 `1e75110935a72fc9433c9b4bf5688d26bc6d3bd1e02f4388b5e1f32ff1e5cc05` (0644 root:root, 35,164 bytes, 1001 lines); `bash -n` PASS; ShellCheck clean at `severity=warning`; trust-boundary + helper-lock + `if !`-rc-loss scans PASS; staging-only invariants PASS. See Appendix C and `06F` H.4.

> [!important] Gate 2 is staging/read-back only
> No mutation to the canonical decider path, no decider execution, no state-dir creation, no `.actions` file, no helper invocation, no Secure Boot / LUKS / ESP / `/etc/kernel/*` mutation. The only commands run against the staged file are read-only checks such as `sha256sum`, `stat`, `wc -l`, `bash -n`, `awk` read-back scans (the Step 6 trust-boundary, helper-lock-path, and `if !` regression scanners), and optionally `shellcheck`. Step 7 explicitly verifies that none of the deferred-mutation surfaces have moved.

> [!important] Decider lock is `/run/tboot-dnf-posttrans.lock`: NOT the helper lock
> The decider owns its own lock and must never pre-acquire the helper's lock. Discussion of the helper's lock lives only in the top-of-source comment block of the staged decider source. Step 6 verifies externally that no runtime code names `/run/tboot-dnf-helper.lock`.

> [!warning] Do not patch the staged source to silence audit-harness false positives
> If the external Step-6 token scan fires a violation that the decider's in-script `_self_test_trust_boundary` self-scan does not, the conclusion is that the **harness regex is wrong**, not the staged source. Editing the staged file invalidates Step 3's recorded sha256 and bakes a workaround for a harness bug into the production decider source. Fix the harness to match the Deviation D regex used by the in-script self-scan. See `06F` H.4.

### Gate 1 closure (design lock: recorded in the notes file, not re-run here)

Gate 1 froze the design in `/root/tboot-lab/notes/b2-3-design-decisions.md` (sha256 `4301ea0f9556739886d0ab249fd4d6c89d44553ee1bb523508ada1579be270c8`). To rebuild from scratch, recreate that notes file so its sha matches; the design rationale (why 14 manifest categories, the baseline-update policy, the performance budget) is in `05_Update_Workflows_and_Key_Storage.md` §3.3. The execution-critical frozen facts the later gates assert against:

- **Rule shape (man-page validated against `libdnf5-plugin-actions` 1.4.0):**
  ```
  post_transaction:::enabled=host-only raise_error=1:/usr/local/sbin/tboot-dnf-posttrans
  ```
  Empty `package_filter` makes the rule always-run; package discrimination is the decider's job.
- **14 manifest categories:** dracut-config, dracut-modules, kernel-modules, firmware, udev-rules, modprobe-load, systemd-early-boot, cryptsetup-tooling, tpm2-tss, storage-tooling, fstab-crypttab, kernel-install, public-trust-config, fedora-kernel-config.
- **Locks:** decider `/run/tboot-dnf-posttrans.lock` (decider-owned); helper `/run/tboot-dnf-helper.lock` (helper-owned: decider must never pre-acquire it).
- **State dirs:** `/var/lib/tboot-dnf-posttrans/` (0700; files 0600); reboot-safety sentinel `/var/lib/tboot-dnf-helper/UNSAFE-TO-REBOOT` (0600).
- **Decider trust boundary (forbidden tokens):** `ukify sbsign sbverify systemd-measure systemd-cryptenroll efi-updatevar cryptsetup`.
- **Baseline-update policy:** only on explicit `--prime` or on helper success after detected drift. No silent auto-prime, no repair on unchanged manifest.

Gate 1 was non-mutating. No snapshot taken. `module5-b2-2-real-run-validated` remains the rollback anchor.

### Gate 2 procedure (seven verification steps)

#### Gate 2 Step 1: Pre-stage verification

```bash
# 1. Pre-stage verification.
bash <<'EOF'
set -uo pipefail
EXP_NOTES_SHA="4301ea0f9556739886d0ab249fd4d6c89d44553ee1bb523508ada1579be270c8"
NOTES="/root/tboot-lab/notes/b2-3-design-decisions.md"
[ -f "$NOTES" ] || { echo "FAIL: Gate 1 notes file missing"; exit 11; }
CUR="$(sha256sum "$NOTES" | awk '{print $1}')"
[ "$CUR" = "$EXP_NOTES_SHA" ] || { echo "FAIL: notes file sha drifted: $CUR (expected $EXP_NOTES_SHA)"; exit 12; }

HELPER_SHA="$(sha256sum /usr/local/sbin/tboot-dnf-helper | awk '{print $1}')"
[ "$HELPER_SHA" = "937afc7a0d199bf7b7d1a5f5bcccf45c37c0f4f7dd0ad924b2567c4da7125e44" ] \
  || { echo "FAIL: helper sha drifted"; exit 13; }
HOOK_SHA="$(sha256sum /etc/kernel/install.d/80-tpm2-sign.install | awk '{print $1}')"
[ "$HOOK_SHA" = "5857e51d5551e05a7ca384f71529ec2fef07f5a6dbe40df70c6bbe49d720947e" ] \
  || { echo "FAIL: hook sha drifted"; exit 14; }

[ ! -e /tmp/tboot-dnf-posttrans.staged ] \
  || { echo "FAIL: /tmp/tboot-dnf-posttrans.staged pre-exists; rm -f and re-run"; exit 15; }

echo "=== pre-stage invariants hold ==="
echo "  notes sha:  $CUR"
echo "  helper sha: $HELPER_SHA"
echo "  hook sha:   $HOOK_SHA"
EOF
```

#### Gate 2 Step 2: Stage the v3 decider source via single-quoted heredoc

The single-quoted heredoc tag (`'TBOOT_DNF_POSTTRANS_EOF'`) prevents expansion of `$variables` and `$(commands)` so the body is written byte-for-byte. The full canonical v3 source is below.

```bash
# 2. Stage the v3 source.
cat > /tmp/tboot-dnf-posttrans.staged <<'TBOOT_DNF_POSTTRANS_EOF'
#!/bin/bash
#
# /usr/local/sbin/tboot-dnf-posttrans
#
# B.2.3 decider for the dracut-only update path of the trusted boot chain.
# Invoked exactly once per DNF5 transaction by the libdnf5-plugin-actions
# post_transaction rule (50-tboot-posttrans.actions, installed in Gate 5):
#
#   post_transaction:::enabled=host-only raise_error=1:/usr/local/sbin/tboot-dnf-posttrans
#
# Per-transaction flow:
#   1. preflight (root, binaries, helper presence, state-dir safety, trust-boundary self-scan)
#   2. acquire /run/tboot-dnf-posttrans.lock with flock -w 120
#   3. compute the boot-input manifest (14 categories per Gate 1 design)
#   4. compare against /var/lib/tboot-dnf-posttrans/boot-input-manifest.baseline
#   5a. unchanged → log, exit 0, NO state writes beyond last-computed-manifest + last-decision
#   5b. drift OR no baseline → write UNSAFE-TO-REBOOT sentinel, invoke
#       /usr/local/sbin/tboot-dnf-helper
#       - helper success: atomic baseline update, clear sentinel, exit 0
#       - helper failure: sentinel REMAINS, last-error written, exit non-zero
#   6. decider lock timeout → write UNSAFE-TO-REBOOT, exit non-zero
#   7. manifest compute error → write UNSAFE-TO-REBOOT, exit non-zero
#
# Lock model (design reference; runtime code names only the decider's lock):
#   - Decider lock: /run/tboot-dnf-posttrans.lock  (THIS script)
#   - Helper lock:  the helper owns its own internal lock; the decider must
#                   never pre-acquire it. The helper's lock path is documented
#                   in /root/tboot-lab/notes/b2-3-design-decisions.md and in
#                   the helper's own source. It is intentionally not named in
#                   runtime code so the trust-boundary read-back assertion
#                   can verify the decider does not reference it outside
#                   this comment.
#
# Trust-boundary policy (audit boundary; partially code-enforced via self-scan):
# The decider is the DECIDER, not the EXECUTOR. It must never:
#   - call ukify
#   - call systemd-measure (any subcommand)
#   - call sbsign or sbverify
#   - call systemd-cryptenroll
#   - call cryptsetup (binary path hashed for manifest only; nolint:trust-boundary)
#   - call efi-updatevar
#   - mutate /etc/uefi-keys/, LUKS keyslots, /etc/crypttab, ESP UKIs,
#     /etc/kernel/uki.conf, /etc/kernel/cmdline, /etc/kernel/install.conf,
#     /etc/kernel/install.d/
#   - create, edit, rename, or remove DNF libdnf5-plugin-actions files
#
# Exit codes:
#    0  success
#    1  generic / unhandled
#    2  not running as root
#    3  required binary missing
#    4  state directory unsafe
#    5  helper missing or not executable
#    7  scratch directory not on tmpfs
#   10  invalid mode / unknown argument
#   11  --prime: baseline exists and --confirm-prime not given
#   20  decider lock acquisition timeout
#   30  manifest computation error
#   31  manifest computation bounded-command timeout
#   40  baseline read error
#   41  baseline write error
#   50  helper invocation failed
#   60  self-test trust-boundary scan: forbidden token in non-comment line
#
# Modes (preflight enforces helper presence in normal/dry-run/prime; the
# dry-run requirement is INTENTIONAL: dry-run validates production readiness
# end-to-end, including the helper's installation profile):
#   (none) | --self-test | --debug-print | --dry-run | --prime [--confirm-prime]
#   --version | --help
#
# References:
#   06B Gate 1; Steps 27-30 (B.2.1, B.2.2); 05_Update_Workflows §3.3;
#   fedora-43-dnf5-libdnf5-plugin-support.md;
#   /root/tboot-lab/notes/b2-3-design-decisions.md
#

set -euo pipefail

# ============================================================================
# Constants
# ============================================================================

readonly DECIDER_VERSION="1.0.0"
readonly TAG="tboot-dnf-posttrans"

readonly LOCK_FILE="/run/tboot-dnf-posttrans.lock"
readonly LOCK_TIMEOUT=120

readonly STATE_DIR="/var/lib/tboot-dnf-posttrans"
readonly STATE_DIR_MODE="700"
readonly STATE_DIR_OWNER="root:root"
readonly BASELINE_FILE="${STATE_DIR}/boot-input-manifest.baseline"
readonly LAST_MANIFEST_FILE="${STATE_DIR}/last-computed-manifest"
readonly LAST_DECISION_FILE="${STATE_DIR}/last-decision"
readonly LAST_ERROR_FILE="${STATE_DIR}/last-error"

readonly HELPER_STATE_DIR="/var/lib/tboot-dnf-helper"
readonly SENTINEL_FILE="${HELPER_STATE_DIR}/UNSAFE-TO-REBOOT"
readonly SENTINEL_MODE="600"

readonly HELPER_BIN="/usr/local/sbin/tboot-dnf-helper"

readonly MANIFEST_COMPUTE_TIMEOUT=60
readonly MANIFEST_SOFT_BUDGET_SECONDS=5

readonly SCRATCH_PARENT="/run"

COMPUTE_START_EPOCH=0
COMPUTE_END_EPOCH=0
MANIFEST_COMPUTE_RC=0
MODE="normal"
CONFIRM_PRIME=0
SCRATCH=""

# ============================================================================
# Logging
# ============================================================================

_log() {
    local level="$1"; shift
    local msg="$*"
    logger -t "$TAG" -p "user.${level}" -- "${level}: ${msg}" 2>/dev/null || true
    echo "[${TAG}] ${level}: ${msg}" >&2
}
info() { _log info    "$@"; }
warn() { _log warning "$@"; }
err()  { _log err     "$@"; }

# ============================================================================
# Usage / version
# ============================================================================

_usage() {
    cat <<USAGE
Usage: ${TAG} [MODE]

Modes:
  (none)         normal post_transaction mode (production)
  --self-test    preflight + trust-boundary self-scan; no compute, no state writes
  --debug-print  compute manifest, dump to stdout, no state writes
  --dry-run      compute, compare to baseline, print decision; no state writes,
                 no helper invocation (preflight still asserts helper presence;
                 dry-run intentionally validates production readiness)
  --prime        Gate 3 only: write current manifest as baseline. Refuses if
                 baseline already present unless --confirm-prime is also given.
  --version      print version and exit
  --help         print this help and exit

See top-of-source for exit code table and trust-boundary policy.
USAGE
}

_version() { echo "${TAG} ${DECIDER_VERSION}"; }

# ============================================================================
# Argument parsing
# ============================================================================

_parse_args() {
    while (( $# > 0 )); do
        case "$1" in
            --self-test)     MODE="self-test";    shift ;;
            --debug-print)   MODE="debug-print";  shift ;;
            --dry-run)       MODE="dry-run";      shift ;;
            --prime)         MODE="prime";        shift ;;
            --confirm-prime) CONFIRM_PRIME=1;     shift ;;
            --version)       _version; exit 0 ;;
            --help|-h)       _usage;   exit 0 ;;
            *)               err "unknown argument: $1"; _usage >&2; exit 10 ;;
        esac
    done
}

# ============================================================================
# Trust-boundary self-scan
#
# Token-boundary regex uses an explicit non-identifier class on both sides
# (with begin/end-of-line as boundaries) to avoid POSIX-awk variation in
# \< / \> word-boundary semantics that could treat `-` as a word separator.
# ============================================================================

_self_test_trust_boundary() {
    local src="${BASH_SOURCE[0]:-$0}"
    [[ -f "$src" ]] || { err "self-test: cannot locate own source"; return 1; }

    local -a forbidden=(ukify sbsign sbverify systemd-measure systemd-cryptenroll efi-updatevar cryptsetup)  # nolint:trust-boundary

    local violations=0 tok
    for tok in "${forbidden[@]}"; do
        if awk -v t="$tok" '
            BEGIN { pat = "(^|[^A-Za-z0-9_-])" t "($|[^A-Za-z0-9_-])" }
            /^[[:space:]]*#/ { next }
            /# nolint:trust-boundary/ { next }
            $0 ~ pat { print NR": "$0; found=1 }
            END { exit !found }
        ' "$src" >/dev/null 2>&1; then
            err "trust-boundary: forbidden token '${tok}' appears in non-comment, non-nolint line of ${src}"
            violations=$((violations+1))
        fi
    done

    if awk '
        /^[[:space:]]*#/ { next }
        /# nolint:trust-boundary/ { next }
        /\/run\/tboot-dnf-helper\.lock/ { print NR": "$0; found=1 }
        END { exit !found }
    ' "$src" >/dev/null 2>&1; then
        err "trust-boundary: decider references the helper lock path in non-comment line"
        violations=$((violations+1))
    fi

    return $violations
}

# ============================================================================
# Preflight + state-dir safety
# ============================================================================

_preflight() {
    if (( EUID != 0 )); then
        err "must run as root (EUID=${EUID})"
        return 2
    fi
    info "ok: running as root"

    local -a req=(sha256sum find flock install awk sort timeout logger stat readlink mktemp mv rm cat tee cp shred)
    local b missing=0
    for b in "${req[@]}"; do
        command -v "$b" >/dev/null 2>&1 || { err "required binary missing: ${b}"; missing=$((missing+1)); }
    done
    (( missing == 0 )) || return 3
    info "ok: required binaries present"

    case "$MODE" in
        normal|dry-run|prime)
            # INTENTIONAL: dry-run also requires helper presence so dry-run
            # validates production readiness end-to-end.
            if [[ ! -f "$HELPER_BIN" ]]; then
                err "helper missing at ${HELPER_BIN}"
                return 5
            fi
            if [[ ! -x "$HELPER_BIN" ]]; then
                err "helper not executable: ${HELPER_BIN}"
                return 5
            fi
            info "ok: helper present and executable at ${HELPER_BIN}"
            ;;
        self-test|debug-print)
            [[ -f "$HELPER_BIN" ]] || warn "helper missing at ${HELPER_BIN} (not fatal in ${MODE})"
            ;;
    esac

    # Trust-boundary self-scan. Its rc is informational; the case below maps
    # any non-zero from this function to a fixed 60.
    if _self_test_trust_boundary; then
        info "ok: trust-boundary self-test PASS"
    else
        err "trust-boundary self-test FAIL"
        return 60
    fi

    case "$MODE" in
        normal|prime)
            if _check_state_dir_safe; then
                :
            else
                return 4
            fi
            ;;
    esac

    return 0
}

_check_state_dir_safe() {
    local dir="$STATE_DIR"
    if [[ -L "$dir" ]]; then
        err "state-dir ${dir} is a symlink"; return 1
    fi
    if [[ ! -d "$dir" ]]; then
        err "state-dir ${dir} does not exist (install step must create it as ${STATE_DIR_OWNER} ${STATE_DIR_MODE})"; return 1
    fi
    local owner mode
    owner="$(stat -c '%U:%G' "$dir" 2>/dev/null || true)"
    mode="$(stat -c '%a'     "$dir" 2>/dev/null || true)"
    [[ "$owner" == "$STATE_DIR_OWNER" ]] || { err "state-dir ${dir} owner=${owner} (expected ${STATE_DIR_OWNER})"; return 1; }
    [[ "$mode"  == "$STATE_DIR_MODE"  ]] || { err "state-dir ${dir} mode=${mode} (expected ${STATE_DIR_MODE})";   return 1; }
    [[ -w "$dir" ]] || { err "state-dir ${dir} not writable"; return 1; }
    info "ok: state-dir ${dir} safe (${STATE_DIR_OWNER}, ${STATE_DIR_MODE}, writable, not symlink)"
    return 0
}

# ============================================================================
# Scratch (tmpfs)
# ============================================================================

_setup_scratch() {
    SCRATCH="$(mktemp -d -p "$SCRATCH_PARENT" "${TAG}.XXXXXX")"
    local fstype
    fstype="$(stat -f -c '%T' "$SCRATCH_PARENT" 2>/dev/null || true)"
    if [[ "$fstype" != "tmpfs" ]]; then
        err "scratch parent ${SCRATCH_PARENT} not tmpfs (fstype=${fstype})"
        rmdir "$SCRATCH" 2>/dev/null || true
        return 7
    fi
    chmod 0700 "$SCRATCH"
    info "ok: scratch ${SCRATCH} on tmpfs"
    return 0
}

_cleanup_scratch() {
    local rc=$?
    trap - EXIT
    if [[ -n "${SCRATCH:-}" && -d "$SCRATCH" ]]; then
        find "$SCRATCH" -type f -exec shred -u {} + 2>/dev/null || true
        rm -rf "$SCRATCH" 2>/dev/null || true
    fi
    return $rc
}

# ============================================================================
# Manifest emitters
# ============================================================================

_emit_file_content() {
    local cat="$1" path="$2"
    if [[ -L "$path" ]]; then
        local tgt h
        tgt="$(readlink "$path" 2>/dev/null || echo '<readlink-failed>')"
        h="$(printf 'symlink|target=%s' "$tgt" | sha256sum | awk '{print $1}')"
        printf '%s\t%s\t%s\t%s\n' "$cat" "$path" "content" "$h"
        return 0
    fi
    if [[ ! -f "$path" ]]; then
        printf '%s\t%s\t%s\t%s\n' "$cat" "$path" "content" "MISSING-FILE"
        return 0
    fi
    local h rc
    if h="$(timeout "$MANIFEST_COMPUTE_TIMEOUT" sha256sum "$path" 2>/dev/null | awk 'NR==1{print $1}')"; then
        if [[ -z "$h" ]]; then
            MANIFEST_COMPUTE_RC=30
            printf '%s\t%s\t%s\t%s\n' "$cat" "$path" "content" "COMPUTE-ERROR-empty-hash"
        else
            printf '%s\t%s\t%s\t%s\n' "$cat" "$path" "content" "$h"
        fi
    else
        rc=$?
        MANIFEST_COMPUTE_RC=$rc
        if (( rc == 124 )); then
            printf '%s\t%s\t%s\t%s\n' "$cat" "$path" "content" "COMPUTE-TIMEOUT"
        else
            printf '%s\t%s\t%s\t%s\n' "$cat" "$path" "content" "COMPUTE-ERROR-rc-$rc"
        fi
    fi
}

_emit_dir_content() {
    local cat="$1" path="$2"
    if [[ ! -e "$path" ]]; then
        printf '%s\t%s\t%s\t%s\n' "$cat" "$path" "content" "MISSING-DIR"
        return 0
    fi
    if [[ ! -d "$path" ]]; then
        _emit_file_content "$cat" "$path"
        return 0
    fi

    local tmp_list rc
    tmp_list="$(mktemp -p "${SCRATCH:-/tmp}" .dirlist.XXXXXX)"
    if timeout "$MANIFEST_COMPUTE_TIMEOUT" find "$path" \( -type f -o -type l \) -print0 2>/dev/null \
         | LC_ALL=C sort -z > "$tmp_list"; then
        rc=0
    else
        rc=$?
    fi

    if (( rc != 0 )); then
        rm -f "$tmp_list" 2>/dev/null || true
        MANIFEST_COMPUTE_RC=$rc
        if (( rc == 124 )); then
            printf '%s\t%s\t%s\t%s\n' "$cat" "$path" "content" "COMPUTE-TIMEOUT-dir-walk"
        else
            printf '%s\t%s\t%s\t%s\n' "$cat" "$path" "content" "COMPUTE-ERROR-dir-walk-rc-$rc"
        fi
        return 0
    fi

    if [[ ! -s "$tmp_list" ]]; then
        rm -f "$tmp_list" 2>/dev/null || true
        if [[ -z "$(ls -A "$path" 2>/dev/null)" ]]; then
            printf '%s\t%s\t%s\t%s\n' "$cat" "$path" "content" "EMPTY-DIR"
        else
            printf '%s\t%s\t%s\t%s\n' "$cat" "$path" "content" "EMPTY-OF-FILES-AND-LINKS"
        fi
        return 0
    fi

    local f
    while IFS= read -r -d '' f; do
        [[ -n "$f" ]] || continue
        _emit_file_content "$cat" "$f"
    done < "$tmp_list"
    rm -f "$tmp_list" 2>/dev/null || true
}

_emit_dir_meta() {
    local cat="$1" path="$2"
    if [[ ! -e "$path" ]]; then
        printf '%s\t%s\t%s\t%s\n' "$cat" "$path" "meta" "MISSING-DIR"
        return 0
    fi
    local h rc
    if h="$(timeout "$MANIFEST_COMPUTE_TIMEOUT" find "$path" -printf '%p %y %s %T@ %m %U %G %l\n' 2>/dev/null \
              | LC_ALL=C sort | sha256sum | awk 'NR==1{print $1}')"; then
        if [[ -z "$h" ]]; then
            MANIFEST_COMPUTE_RC=30
            printf '%s\t%s\t%s\t%s\n' "$cat" "$path" "meta" "COMPUTE-ERROR-empty-hash"
        else
            printf '%s\t%s\t%s\t%s\n' "$cat" "$path" "meta" "$h"
        fi
    else
        rc=$?
        MANIFEST_COMPUTE_RC=$rc
        if (( rc == 124 )); then
            printf '%s\t%s\t%s\t%s\n' "$cat" "$path" "meta" "COMPUTE-TIMEOUT"
        else
            printf '%s\t%s\t%s\t%s\n' "$cat" "$path" "meta" "COMPUTE-ERROR-rc-$rc"
        fi
    fi
}

_emit_dir_meta_filtered() {
    local cat="$1" path="$2"
    shift 2
    if [[ ! -e "$path" ]]; then
        printf '%s\t%s\t%s\t%s\n' "$cat" "$path" "meta-filtered" "MISSING-DIR"
        return 0
    fi
    local h rc
    if h="$(timeout "$MANIFEST_COMPUTE_TIMEOUT" find "$path" "$@" -printf '%p %y %s %T@ %m %U %G %l\n' 2>/dev/null \
              | LC_ALL=C sort | sha256sum | awk 'NR==1{print $1}')"; then
        if [[ -z "$h" ]]; then
            MANIFEST_COMPUTE_RC=30
            printf '%s\t%s\t%s\t%s\n' "$cat" "$path" "meta-filtered" "COMPUTE-ERROR-empty-hash"
        else
            printf '%s\t%s\t%s\t%s\n' "$cat" "$path" "meta-filtered" "$h"
        fi
    else
        rc=$?
        MANIFEST_COMPUTE_RC=$rc
        if (( rc == 124 )); then
            printf '%s\t%s\t%s\t%s\n' "$cat" "$path" "meta-filtered" "COMPUTE-TIMEOUT"
        else
            printf '%s\t%s\t%s\t%s\n' "$cat" "$path" "meta-filtered" "COMPUTE-ERROR-rc-$rc"
        fi
    fi
}

_emit_glob_content() {
    local cat="$1" pattern="$2"
    local -a matches=()
    # shellcheck disable=SC2206
    matches=( $pattern )
    if (( ${#matches[@]} == 1 )) && [[ "${matches[0]}" == "$pattern" ]] && [[ ! -e "${matches[0]}" ]]; then
        printf '%s\t%s\t%s\t%s\n' "$cat" "$pattern" "content" "MISSING-GLOB"
        return 0
    fi
    local g
    for g in "${matches[@]}"; do
        [[ -e "$g" || -L "$g" ]] && _emit_file_content "$cat" "$g"
    done
}

# ============================================================================
# Manifest categories (14)
# ============================================================================

_cat01_dracut_config() {
    local c="01-dracut-config"
    _emit_file_content "$c" /etc/dracut.conf
    _emit_dir_content  "$c" /etc/dracut.conf.d
    _emit_dir_content  "$c" /usr/lib/dracut/dracut.conf.d
}

_cat02_dracut_modules() {
    local c="02-dracut-modules"
    _emit_dir_meta "$c" /usr/lib/dracut/modules.d
}

_cat03_kernel_modules() {
    local c="03-kernel-modules"
    if [[ ! -d /lib/modules ]]; then
        printf '%s\t%s\t%s\t%s\n' "$c" "/lib/modules" "meta" "MISSING-DIR"
        return 0
    fi
    local kdir found=0
    for kdir in /lib/modules/*/; do
        [[ -d "$kdir" ]] || continue
        _emit_dir_meta "$c" "${kdir%/}"
        found=1
    done
    (( found == 0 )) && printf '%s\t%s\t%s\t%s\n' "$c" "/lib/modules/*" "meta" "EMPTY-DIR"
}

_cat04_firmware() {
    local c="04-firmware"
    _emit_dir_meta "$c" /usr/lib/firmware
}

_cat05_udev_rules() {
    local c="05-udev-rules"
    _emit_dir_content "$c" /etc/udev/rules.d
    _emit_dir_content "$c" /usr/lib/udev/rules.d
}

_cat06_modprobe_load() {
    local c="06-modprobe-load"
    _emit_dir_content "$c" /etc/modprobe.d
    _emit_dir_content "$c" /usr/lib/modprobe.d
    _emit_dir_content "$c" /etc/modules-load.d
    _emit_dir_content "$c" /usr/lib/modules-load.d
}

_cat07_systemd_early_boot() {
    local c="07-systemd-early-boot"
    _emit_dir_meta_filtered "$c" /usr/lib/systemd        -maxdepth 1 -name 'systemd-*'
    _emit_dir_meta_filtered "$c" /usr/lib/systemd/system -maxdepth 1 -name 'initrd*'
    _emit_dir_meta          "$c" /usr/lib/systemd/system-generators
}

_cat08_cryptsetup_tooling() {
    local c="08-cryptsetup-tooling"
    _emit_file_content "$c" /usr/sbin/cryptsetup  # nolint:trust-boundary
    _emit_glob_content "$c" '/usr/lib64/libcryptsetup.so*'
}

_cat09_tpm2_tss() {
    local c="09-tpm2-tss"
    _emit_dir_meta_filtered "$c" /usr/sbin  -maxdepth 1 -name 'tpm2_*'
    _emit_dir_meta_filtered "$c" /usr/lib64 -maxdepth 1 -name 'libtss2-*'
}

_cat10_storage() {
    local c="10-storage-tooling"
    _emit_file_content "$c" /usr/sbin/lvm
    _emit_file_content "$c" /usr/sbin/dmsetup
    _emit_file_content "$c" /usr/sbin/mdadm
    _emit_file_content "$c" /etc/lvm/lvm.conf
    _emit_dir_content  "$c" /etc/lvm/profile
    _emit_file_content "$c" /etc/multipath.conf
    _emit_dir_content  "$c" /etc/multipath
    _emit_dir_content  "$c" /etc/iscsi
    _emit_file_content "$c" /etc/mdadm.conf
}

_cat11_fstab_crypttab() {
    local c="11-fstab-crypttab"
    _emit_file_content "$c" /etc/fstab
    _emit_file_content "$c" /etc/crypttab
}

_cat12_kernel_install() {
    local c="12-kernel-install"
    _emit_file_content "$c" /etc/kernel/install.conf
    _emit_file_content "$c" /etc/kernel/cmdline
    _emit_file_content "$c" /etc/kernel/uki.conf
    _emit_dir_content  "$c" /etc/kernel/install.d
    _emit_dir_meta     "$c" /usr/lib/kernel/install.d
}

_cat13_public_trust() {
    local c="13-public-trust-config"
    _emit_file_content "$c" /etc/uefi-keys/db.crt
    _emit_file_content "$c" /etc/systemd/tpm2-pcr-public-key.pem
}

_cat14_fedora_kernel() {
    local c="14-fedora-kernel-config"
    _emit_file_content "$c" /etc/sysconfig/kernel
}

# ============================================================================
# Manifest computation
# ============================================================================

_compute_manifest() {
    local out="$1"
    MANIFEST_COMPUTE_RC=0
    COMPUTE_START_EPOCH=$(date +%s)

    local raw="${SCRATCH}/manifest.raw"
    : >"$raw"

    _cat01_dracut_config       >>"$raw"
    _cat02_dracut_modules      >>"$raw"
    _cat03_kernel_modules      >>"$raw"
    _cat04_firmware            >>"$raw"
    _cat05_udev_rules          >>"$raw"
    _cat06_modprobe_load       >>"$raw"
    _cat07_systemd_early_boot  >>"$raw"
    _cat08_cryptsetup_tooling  >>"$raw"
    _cat09_tpm2_tss            >>"$raw"
    _cat10_storage             >>"$raw"
    _cat11_fstab_crypttab      >>"$raw"
    _cat12_kernel_install      >>"$raw"
    _cat13_public_trust        >>"$raw"
    _cat14_fedora_kernel       >>"$raw"

    COMPUTE_END_EPOCH=$(date +%s)
    local duration=$((COMPUTE_END_EPOCH - COMPUTE_START_EPOCH))

    LC_ALL=C sort "$raw" > "$out"

    info "manifest computation duration: ${duration}s ($(wc -l <"$out") records)"
    if (( duration > MANIFEST_SOFT_BUDGET_SECONDS )); then
        warn "manifest computation took ${duration}s (>${MANIFEST_SOFT_BUDGET_SECONDS}s soft budget); continuing"
    fi

    if (( MANIFEST_COMPUTE_RC != 0 )); then
        err "manifest computation completed with errors (last rc=${MANIFEST_COMPUTE_RC})"
        (( MANIFEST_COMPUTE_RC == 124 )) && return 31
        return 30
    fi
    return 0
}

# ============================================================================
# Sentinel + state markers
# ============================================================================

_write_sentinel() {
    local reason="$1"
    if [[ ! -d "$HELPER_STATE_DIR" ]]; then
        err "cannot write sentinel: ${HELPER_STATE_DIR} missing"
        return 1
    fi
    local tmp
    tmp="$(mktemp -p "$HELPER_STATE_DIR" .UNSAFE-TO-REBOOT.XXXXXX 2>/dev/null)" || {
        err "cannot mktemp under ${HELPER_STATE_DIR}"; return 1; }
    {
        echo "epoch=$(date +%s)"
        echo "date=$(date -Iseconds)"
        echo "set_by=${TAG}"
        echo "decider_version=${DECIDER_VERSION}"
        echo "reason=${reason}"
    } > "$tmp"
    chmod "$SENTINEL_MODE" "$tmp"
    mv -f "$tmp" "$SENTINEL_FILE"
    info "sentinel written: ${SENTINEL_FILE} (reason=${reason})"
    return 0
}

_clear_sentinel() {
    if [[ -e "$SENTINEL_FILE" ]]; then
        rm -f "$SENTINEL_FILE" 2>/dev/null || { err "could not remove sentinel: ${SENTINEL_FILE}"; return 1; }
        info "sentinel cleared: ${SENTINEL_FILE}"
    fi
    return 0
}

_write_state_atomic() {
    local target="$1"
    [[ -d "$STATE_DIR" ]] || { err "state-dir ${STATE_DIR} missing; cannot write ${target}"; return 1; }
    local tmp
    tmp="$(mktemp -p "$STATE_DIR" .state.XXXXXX)"
    cat > "$tmp"
    chmod 0600 "$tmp"
    mv -f "$tmp" "$target"
}

_write_last_decision() {
    local decision="$1" detail="${2:-}"
    {
        echo "epoch=$(date +%s)"
        echo "date=$(date -Iseconds)"
        echo "decision=${decision}"
        echo "detail=${detail}"
        echo "compute_duration_s=$((COMPUTE_END_EPOCH - COMPUTE_START_EPOCH))"
        echo "decider_version=${DECIDER_VERSION}"
    } | _write_state_atomic "$LAST_DECISION_FILE"
}

_write_last_error() {
    local stage="$1" rc="$2" detail="$3"
    {
        echo "epoch=$(date +%s)"
        echo "date=$(date -Iseconds)"
        echo "stage=${stage}"
        echo "rc=${rc}"
        echo "detail=${detail}"
        echo "decider_version=${DECIDER_VERSION}"
    } | _write_state_atomic "$LAST_ERROR_FILE" 2>/dev/null || true
}

_persist_last_manifest() {
    local src="$1"
    [[ -d "$STATE_DIR" ]] || { err "state-dir ${STATE_DIR} missing"; return 1; }
    local tmp
    tmp="$(mktemp -p "$STATE_DIR" .last-manifest.XXXXXX)"
    cp -f "$src" "$tmp"
    chmod 0600 "$tmp"
    mv -f "$tmp" "$LAST_MANIFEST_FILE"
}

# ============================================================================
# Baseline comparison
# ============================================================================

_compare_to_baseline() {
    local current="$1"
    if [[ ! -f "$BASELINE_FILE" ]]; then
        echo "no-baseline"
        return 0
    fi
    if [[ ! -r "$BASELINE_FILE" ]]; then
        err "baseline ${BASELINE_FILE} exists but is unreadable"
        echo "baseline-error"
        return 40
    fi
    local cur_sha base_sha
    cur_sha="$(sha256sum "$current"        | awk '{print $1}')"
    base_sha="$(sha256sum "$BASELINE_FILE" | awk '{print $1}')"
    if [[ "$cur_sha" == "$base_sha" ]]; then
        echo "unchanged"
    else
        echo "drift"
    fi
    return 0
}

# ============================================================================
# Helper invocation (no pre-acquisition of the helper's own lock)
# ============================================================================

_invoke_helper() {
    info "invoking helper: ${HELPER_BIN} (helper owns its own internal lock)"
    local rc
    "$HELPER_BIN"
    rc=$?
    if (( rc == 0 )); then
        info "helper exited 0 (success)"
        return 0
    fi
    err "helper exited ${rc} (failure)"
    return $rc
}

# ============================================================================
# Mode: self-test
# ============================================================================

_mode_self_test() {
    info "mode=self-test; preflight + trust-boundary scan only; no compute, no state writes"
    info "exit reason: self-test PASS"
    return 0
}

# ============================================================================
# Mode: debug-print
# ============================================================================

_mode_debug_print() {
    info "mode=debug-print; computing manifest to stdout; no state writes"
    local cur="${SCRATCH}/manifest.current"
    local rc
    if _compute_manifest "$cur"; then
        :
    else
        rc=$?
        err "manifest compute failed in debug-print (rc=${rc})"
        return $rc
    fi
    cat "$cur"
    info "exit reason: debug-print complete"
    return 0
}

# ============================================================================
# Mode: dry-run
# ============================================================================

_mode_dry_run() {
    info "mode=dry-run; computing manifest; comparing to baseline; no state writes; no helper invocation"
    local cur="${SCRATCH}/manifest.current"
    local rc
    if _compute_manifest "$cur"; then
        :
    else
        rc=$?
        err "manifest compute failed in dry-run (rc=${rc})"
        return $rc
    fi

    local cmp
    cmp="$(_compare_to_baseline "$cur" || true)"
    info "baseline compare result: ${cmp}"

    case "$cmp" in
        unchanged)
            info "helper invocation decision (dry-run): would EXIT 0 (manifest unchanged); helper NOT invoked"
            ;;
        drift)
            info "helper invocation decision (dry-run): would WRITE SENTINEL and INVOKE HELPER; baseline would update on helper success"
            ;;
        no-baseline)
            info "helper invocation decision (dry-run): no baseline present; would WRITE SENTINEL and INVOKE HELPER (treat as drift)"
            ;;
        baseline-error|*)
            err "helper invocation decision (dry-run): UNKNOWN comparison result '${cmp}'"
            return 40
            ;;
    esac
    info "exit reason: dry-run complete (no state writes performed)"
    return 0
}

# ============================================================================
# Mode: prime (Gate 3 only)
# ============================================================================

_mode_prime() {
    info "mode=prime (Gate 3 baseline-prime)"
    if [[ -f "$BASELINE_FILE" ]] && (( CONFIRM_PRIME == 0 )); then
        err "baseline already exists at ${BASELINE_FILE}; refusing to overwrite without --confirm-prime"
        return 11
    fi
    local cur="${SCRATCH}/manifest.current"
    local rc
    if _compute_manifest "$cur"; then
        :
    else
        rc=$?
        err "manifest compute failed in prime mode (rc=${rc})"
        return $rc
    fi

    local btmp
    btmp="$(mktemp -p "$STATE_DIR" .baseline.XXXXXX)" || {
        err "could not mktemp baseline staging under ${STATE_DIR}"
        return 41
    }
    cp -f "$cur" "$btmp"
    chmod 0600 "$btmp"
    if ! mv -f "$btmp" "$BASELINE_FILE"; then
        err "baseline write failed: ${BASELINE_FILE}"
        rm -f "$btmp" 2>/dev/null || true
        return 41
    fi
    _persist_last_manifest "$cur"
    _write_last_decision "primed" "Gate 3 baseline prime"
    info "exit reason: baseline primed at ${BASELINE_FILE}"
    return 0
}

# ============================================================================
# Mode: normal (production post_transaction)
# ============================================================================

_mode_normal() {
    info "mode=normal; acquiring decider lock ${LOCK_FILE} (flock -w ${LOCK_TIMEOUT})"

    local sub_rc=0
    (
        if ! flock -w "$LOCK_TIMEOUT" 9; then
            exit 20
        fi
        info "decider lock acquired; recomputing manifest"

        local cur="${SCRATCH}/manifest.current"
        local compute_rc
        if _compute_manifest "$cur"; then
            :
        else
            compute_rc=$?
            err "manifest compute failed under decider lock (rc=${compute_rc})"
            _write_sentinel "manifest-compute-error-rc-${compute_rc}" || true
            _write_last_error "manifest-compute" "${compute_rc}" "see journal"
            exit "$compute_rc"
        fi

        local cmp
        cmp="$(_compare_to_baseline "$cur" || true)"
        info "baseline compare result: ${cmp}"

        case "$cmp" in
            unchanged)
                _persist_last_manifest "$cur"
                _write_last_decision "unchanged" "manifest matches baseline; helper not invoked; baseline NOT updated (per design)"
                info "helper invocation decision: SKIP (manifest unchanged)"
                info "exit reason: manifest unchanged; baseline NOT updated; helper not invoked"
                exit 0
                ;;
            drift|no-baseline)
                local reason="drift-detected"
                [[ "$cmp" == "no-baseline" ]] && reason="no-baseline-present"
                if ! _write_sentinel "$reason"; then
                    err "sentinel write failed; refusing to invoke helper"
                    _write_last_decision "sentinel-write-failed" "${reason}; helper not invoked"
                    _write_last_error "sentinel-write" "42" "helper not invoked"
                    exit 42
                fi
                _persist_last_manifest "$cur"
                _write_last_decision "helper-pending" "${reason}; invoking helper"
                info "helper invocation decision: INVOKE (${reason})"

                if _invoke_helper; then
                    local btmp
                    btmp="$(mktemp -p "$STATE_DIR" .baseline.XXXXXX)" || {
                        err "could not mktemp baseline staging after helper success"
                        _write_last_decision "helper-success-baseline-write-failed" "mktemp failed"
                        exit 41
                    }
                    cp -f "$cur" "$btmp"
                    chmod 0600 "$btmp"
                    if ! mv -f "$btmp" "$BASELINE_FILE"; then
                        err "baseline atomic replace failed after helper success"
                        rm -f "$btmp" 2>/dev/null || true
                        _write_last_decision "helper-success-baseline-write-failed" "mv -f failed"
                        exit 41
                    fi
                    _clear_sentinel || warn "sentinel clear failed after helper success; will retry next run"
                    _write_last_decision "helper-success" "${reason}; baseline updated; sentinel cleared"
                    info "exit reason: helper succeeded; baseline updated; sentinel cleared"
                    exit 0
                else
                    local helper_rc=$?
                    err "helper invocation failed (rc=${helper_rc}); sentinel REMAINS; baseline NOT updated"
                    _write_last_decision "helper-failed" "rc=${helper_rc}; sentinel remains; baseline NOT updated"
                    _write_last_error "helper-invocation" "${helper_rc}" "see /var/lib/tboot-dnf-helper/last-failure*"
                    exit 50
                fi
                ;;
            baseline-error)
                err "baseline read error"
                _write_sentinel "baseline-read-error" || true
                _write_last_error "baseline-read" "40" "baseline exists but unreadable"
                exit 40
                ;;
            *)
                err "unknown comparison result: ${cmp}"
                _write_sentinel "unknown-compare-result-${cmp}" || true
                _write_last_error "compare" "1" "unknown result ${cmp}"
                exit 1
                ;;
        esac
    ) 9>"$LOCK_FILE"
    sub_rc=$?

    if (( sub_rc == 20 )); then
        err "decider lock acquisition timeout after ${LOCK_TIMEOUT}s on ${LOCK_FILE}"
        _write_sentinel "decider-lock-timeout" || true
        _write_last_error "lock-acquire" "20" "flock -w ${LOCK_TIMEOUT} timed out"
        info "exit reason: decider lock timeout"
        return 20
    fi
    return $sub_rc
}

# ============================================================================
# main
# ============================================================================

main() {
    _parse_args "$@"

    local pre_rc
    if _preflight; then
        :
    else
        pre_rc=$?
        err "preflight failed (rc=${pre_rc})"
        exit "$pre_rc"
    fi

    if [[ "$MODE" == "self-test" ]]; then
        _mode_self_test
        exit $?
    fi

    local scratch_rc
    if _setup_scratch; then
        :
    else
        scratch_rc=$?
        err "scratch setup failed (rc=${scratch_rc})"
        exit "$scratch_rc"
    fi
    trap _cleanup_scratch EXIT

    local mode_rc=0
    case "$MODE" in
        debug-print) _mode_debug_print; mode_rc=$? ;;
        dry-run)     _mode_dry_run;     mode_rc=$? ;;
        prime)       _mode_prime;       mode_rc=$? ;;
        normal)      _mode_normal;      mode_rc=$? ;;
        *)           err "unhandled mode: ${MODE}"; mode_rc=10 ;;
    esac
    exit "$mode_rc"
}

main "$@"
TBOOT_DNF_POSTTRANS_EOF
```

#### Gate 2 Step 3: Verify staged file metadata

```bash
# 3. Verify staged file.
ls -la /tmp/tboot-dnf-posttrans.staged
stat -c 'mode=%a owner=%U:%G size=%s type=%F  %n' /tmp/tboot-dnf-posttrans.staged
wc -l /tmp/tboot-dnf-posttrans.staged
sha256sum /tmp/tboot-dnf-posttrans.staged
```

Expected (reference run 2026-05-24): mode `644`, owner `root:root`, size `35164` bytes, lines `1001`, sha256 `1e75110935a72fc9433c9b4bf5688d26bc6d3bd1e02f4388b5e1f32ff1e5cc05`. Record the sha256 in the Gate 2 closure notes; this is the canonical Gate 2 artifact identity.

#### Gate 2 Step 4: Parse-only bash syntax check

```bash
# 4. Bash syntax check.
bash -n /tmp/tboot-dnf-posttrans.staged && echo "ok: bash -n PASS" \
  || { echo "FAIL: bash -n"; exit 16; }
```

Expected: `ok: bash -n PASS`.

#### Gate 2 Step 5: Optional ShellCheck

```bash
# 5. (Optional) ShellCheck.
if command -v shellcheck >/dev/null 2>&1; then
    SC_LOG=/tmp/tboot-dnf-posttrans.shellcheck.log
    shellcheck --shell=bash --severity=warning /tmp/tboot-dnf-posttrans.staged > "$SC_LOG" 2>&1 \
      && echo "ok: shellcheck clean at severity=warning" \
      || { echo "info: shellcheck findings (see $SC_LOG):"; head -50 "$SC_LOG"; }
else
    echo "info: shellcheck not installed; skipping"
fi
```

Expected: `ok: shellcheck clean at severity=warning` on the reference run.

#### Gate 2 Step 6: Trust-boundary read-back + v3 `if !` regression scan (Deviation D regex; corrected harness)

This is the harness step that was corrected in the 2026-05-24 run. The initial attempt used GNU awk `\<TOKEN\>` word-boundary form and produced a false positive on the manifest category label `08-cryptsetup-tooling` (the substring `cryptsetup` matched between hyphens because GNU awk treats `-` as a word boundary). The corrected harness uses the same regex as the in-script `_self_test_trust_boundary` (Deviation D form), which is the authoritative scanner. See `06F` H.4 for the full rationale. **Do not patch the staged source for this class of false positive.**

```bash
# 6. Read-back assertion: trust-boundary forbidden tokens DO NOT appear in
#    non-comment, non-nolint lines of the staged source, AND the v3 regression
#    'if ! func; then ... $? ...' pattern is absent. Token-boundary regex
#    matches the in-script self-scan (Deviation D form).
bash <<'EOF'
set -uo pipefail
SRC=/tmp/tboot-dnf-posttrans.staged

# --- gate-safety guard: refuse to read-back a missing/empty/unreadable source ---
# Without this, each `awk "$SRC"` below would fail non-zero and the surrounding
# `if awk ...; then` would treat "no violations found" as PASS: masking a
# missing staged file as a clean read-back.
[[ -f "$SRC" && -s "$SRC" && -r "$SRC" ]] || {
    echo "FAIL: staged source missing, empty, or unreadable: $SRC"
    exit 16
}

TOKENS=(ukify sbsign sbverify systemd-measure systemd-cryptenroll efi-updatevar cryptsetup)
violations=0

# --- trust-boundary token scan (Deviation D regex, matches the in-script self-scan) ---
for t in "${TOKENS[@]}"; do
    if awk -v t="$t" '
        BEGIN { pat = "(^|[^A-Za-z0-9_-])" t "($|[^A-Za-z0-9_-])" }
        /^[[:space:]]*#/ { next }
        /# nolint:trust-boundary/ { next }
        $0 ~ pat { print NR": "$0; found=1 }
        END { exit !found }
    ' "$SRC"; then
        echo "FAIL: forbidden token '${t}' in non-comment, non-nolint line"
        violations=$((violations+1))
    fi
done

# --- helper lock path must not be referenced outside comments ---
if awk '
    /^[[:space:]]*#/ { next }
    /# nolint:trust-boundary/ { next }
    /\/run\/tboot-dnf-helper\.lock/ { print NR": "$0; found=1 }
    END { exit !found }
' "$SRC"; then
    echo "FAIL: decider source references /run/tboot-dnf-helper.lock outside comments"
    violations=$((violations+1))
fi

# --- v3 regression: 'if ! func; then rc=$?' would lose the rc ---
# Portable awk end-of-block: ^[[:space:]]*(elif|else|fi)([[:space:];]|$)
# (the original \b form is not portable across POSIX awk implementations).
if awk '
    /^[[:space:]]*#/ { next }

    # Catch dangerous same-line form:
    /^[[:space:]]*if[[:space:]]+![[:space:]]+/ && /\$\?/ {
        printf "line %d: %s\n", NR, $0
        found=1
        next
    }

    # Track multiline if ! blocks.
    /^[[:space:]]*if[[:space:]]+![[:space:]]+/ {
        inblk = 1
        blkstart = NR
        next
    }

    inblk {
        if ($0 ~ /^[[:space:]]*(elif|else|fi)([[:space:];]|$)/) {
            inblk = 0
            next
        }
        if ($0 ~ /\$\?/) {
            printf "line %d (inside if !-block starting at line %d): %s\n", NR, blkstart, $0
            found=1
        }
    }

    END { exit !found }
' "$SRC"; then
    echo "FAIL: v3 regression: if ! block captures \$? and would lose the failing rc"
    violations=$((violations+1))
fi

if (( violations == 0 )); then
    echo "=== trust-boundary + v3 regression read-back: PASS ==="
    echo "  staged source present, non-empty, readable: $SRC"
    echo "  forbidden tokens absent from non-comment, non-nolint lines: ${TOKENS[*]}"
    echo "  /run/tboot-dnf-helper.lock absent from non-comment, non-nolint lines"
    echo "  no 'if ! ...; then ... \$? ...' patterns present"
else
    echo "FAIL: ${violations} read-back violation(s)"
    exit 17
fi
EOF
```

Expected:
```
=== trust-boundary + v3 regression read-back: PASS ===
  staged source present, non-empty, readable: /tmp/tboot-dnf-posttrans.staged
  forbidden tokens absent from non-comment, non-nolint lines: ukify sbsign sbverify systemd-measure systemd-cryptenroll efi-updatevar cryptsetup
  /run/tboot-dnf-helper.lock absent from non-comment, non-nolint lines
  no 'if ! ...; then ... $? ...' patterns present
```

> [!warning] If Step 6 fires `FAIL: forbidden token 'cryptsetup' in non-comment, non-nolint line` at the `08-cryptsetup-tooling` category label
> The Deviation D regex above is the correct fix; an earlier draft of this harness used the laxer `$0 ~ ("\\<" t "\\>")` form, which produces this false positive. **Do not patch the staged source** (do not add nolint, do not rename the category). Use the Deviation D regex shown above and re-run Step 6 only. See `06F` H.4.

#### Gate 2 Step 7: Staging-only invariants

```bash
# 7. Staging-only invariants.
bash <<'EOF'
set -uo pipefail
EXP_HELPER_SHA="937afc7a0d199bf7b7d1a5f5bcccf45c37c0f4f7dd0ad924b2567c4da7125e44"
EXP_HOOK_SHA="5857e51d5551e05a7ca384f71529ec2fef07f5a6dbe40df70c6bbe49d720947e"

[ ! -e /usr/local/sbin/tboot-dnf-posttrans ] \
  || { echo "FAIL: decider installed (should be staging-only in Gate 2)"; exit 21; }
[ ! -e /var/lib/tboot-dnf-posttrans ] \
  || { echo "FAIL: decider state-dir created (Gate 3's responsibility)"; exit 22; }
[ ! -e /var/lib/tboot-dnf-helper/UNSAFE-TO-REBOOT ] \
  || { echo "FAIL: sentinel created"; exit 23; }

ACTIVE=$(find /etc/dnf/libdnf5-plugins/actions.d/ -maxdepth 1 -name '*.actions' -type f 2>/dev/null | wc -l)
[ "$ACTIVE" -eq 0 ] || { echo "FAIL: .actions file appeared ($ACTIVE)"; exit 24; }

[ "$(sha256sum /usr/local/sbin/tboot-dnf-helper | awk '{print $1}')" = "$EXP_HELPER_SHA" ] \
  || { echo "FAIL: helper drift"; exit 25; }
[ "$(sha256sum /etc/kernel/install.d/80-tpm2-sign.install | awk '{print $1}')" = "$EXP_HOOK_SHA" ] \
  || { echo "FAIL: hook drift"; exit 26; }

PLUGIN_ENABLED="$(awk -F'=' '/^[[:space:]]*enabled[[:space:]]*=/{gsub(/[[:space:]]/,"",$2); print $2; exit}' /etc/dnf/libdnf5-plugins/actions.conf 2>/dev/null || true)"
[ "$PLUGIN_ENABLED" = "1" ] || { echo "FAIL: plugin-level enabled changed: $PLUGIN_ENABLED"; exit 27; }

[ -f /tmp/tboot-dnf-posttrans.staged ] || { echo "FAIL: staged file missing"; exit 28; }

echo "=== Gate 2 staging-only invariants hold ==="
echo "  installed decider:    absent (correct)"
echo "  decider state-dir:    absent (correct)"
echo "  UNSAFE sentinel:      absent (correct)"
echo "  active .actions:      0 (correct)"
echo "  helper sha:           $EXP_HELPER_SHA (verified)"
echo "  hook sha:             $EXP_HOOK_SHA (verified)"
echo "  plugin-level enabled: 1 (unchanged)"
echo "  staged file:          present at /tmp/tboot-dnf-posttrans.staged"
EOF
```

Expected: `=== Gate 2 staging-only invariants hold ===` followed by eight `correct/verified/unchanged/present` lines.

### Expected output (Gate 2 overall)

- Step 1: `=== pre-stage invariants hold ===` with notes/helper/hook sha; exit 0.
- Step 2: heredoc completes silently. No stdout.
- Step 3: regular file `root:root` 0644, ~35 KB, ~1001 lines; sha256 recorded.
- Step 4: `ok: bash -n PASS`.
- Step 5: one of `ok: shellcheck clean at severity=warning`, `info: shellcheck findings (see …)`, or `info: shellcheck not installed; skipping`.
- Step 6: `=== trust-boundary + v3 regression read-back: PASS ===` with the four labelled assertions.
- Step 7: `=== Gate 2 staging-only invariants hold ===` with the eight labelled assertions.

### Stop condition

- Step 1 exits 11–15: Gate 1 invariants invalidated, B.2.2 closure invalidated, or stale staged file from prior attempt. Halt; investigate before re-staging.
- Step 4 exits 16: the heredoc produced syntactically invalid bash. Inspect the line reported by `bash -n` and re-stage.
- Step 6 exits 16: the staged source is missing/empty/unreadable. Re-run Step 2.
- Step 6 exits 17 on `cryptsetup` at `08-cryptsetup-tooling`: the harness uses an outdated regex form. Apply the Deviation D regex shown above (in-script-self-scan-equivalent) and re-run Step 6 only. **Do not patch the staged source.** See `06F` H.4.
- Step 6 exits 17 on a genuine token violation (a forbidden token in non-comment, non-nolint runtime code): the heredoc body has a real trust-boundary breach. Fix the source and re-stage; do not proceed.
- Step 6 exits 17 on the helper-lock-path leak or on a surviving `if ! func; then ... $?` pattern: same: fix the heredoc body, re-stage, re-run.
- Step 7 exits 21–28: a Gate-2 mutation invariant moved. Severe. Stop, write the failure into the Gate-2 attempt log, and consider whether rollback to `module5-b2-2-real-run-validated` is required before any further work.

### Snapshot point

None. Gate 2 is non-mutating. `module5-b2-2-real-run-validated` remains the rollback anchor for B.2.3. The next snapshot is `module5-b2-3-decider-installed-primed`, taken at Gate 3 close.

### Gate 2 closure (recorded in attempt log, not in this runbook)

The Gate 2 closure block, with run timestamp, recovered Step 3 metadata, Step 4 PASS, Step 5 ShellCheck status, the Step 6 first-attempt failure + harness correction note + second-attempt PASS, and the Step 7 PASS, lives at `/root/tboot-lab/notes/b2-3-gate-2-attempts.md` in the Obsidian vault: not in this runbook. The runbook captures the procedure; the attempt log captures the run.

✅ Block B.2.3 Gate 1 and Gate 2 complete. The decider source is authored, externally read-back-validated, and staged at `/tmp/tboot-dnf-posttrans.staged` (sha256 `1e75110935a72fc9433c9b4bf5688d26bc6d3bd1e02f4388b5e1f32ff1e5cc05`) ready for Gate 3 install. The canonical path `/usr/local/sbin/tboot-dnf-posttrans` is intentionally absent; the state directory `/var/lib/tboot-dnf-posttrans/` is intentionally absent; no `.actions` rule is installed; the system has accumulated no mutation from Gate 2.

---

## Step 32: Block B.2.3 Gates 3.0–3.8: decider install, state-dir create, baseline prime, closure snapshot

**Goal.** Install the Gate-2-validated decider at `/usr/local/sbin/tboot-dnf-posttrans` (sub-gate 3.2), create the decider state directory `/var/lib/tboot-dnf-posttrans/` (sub-gate 3.4), prime the baseline manifest via `tboot-dnf-posttrans --prime` (sub-gate 3.6), prove zero drift between the primed baseline and a single `--debug-print` invocation (sub-gate 3.7), and take the Gate 3 closure snapshot `module5-b2-3-decider-installed-primed` on the Proxmox host (sub-gate 3.8). All other sub-gates (3.0 pre-flight, 3.1 staged-artifact re-verification, 3.3 installed-decider identity, 3.5 `--self-test`) are read-only verifications around these mutations.

The production `.actions` rule (`50-tboot-posttrans.actions`) and the negative-validation transaction are out of scope for Step 32: those are Gates 4–7. Block B.2.4 (live dracut-sensitive transaction validation) is also out of scope.

**Rollback anchor.** `module5-b2-2-real-run-validated` (B.2.2 closure).

**New closure snapshot.** `module5-b2-3-decider-installed-primed` (taken at end of sub-gate 3.8).

**Operational paths used in Step 32:**

| Path | Purpose |
|---|---|
| `/usr/local/sbin/tboot-dnf-posttrans` | Canonical (user-facing) decider path |
| `/usr/local/bin/tboot-dnf-posttrans` | Resolved real path on Fedora 43 (UsrMerge; see `06F` I.3) |
| `/var/lib/tboot-dnf-posttrans/` | Decider state directory (0700 root:root) |
| `/var/lib/tboot-dnf-posttrans/boot-input-manifest.baseline` | Primed baseline manifest |
| `/var/lib/tboot-dnf-posttrans/last-computed-manifest` | Last manifest written by decider |
| `/var/lib/tboot-dnf-posttrans/last-decision` | Structured key=value decision record |
| `/var/lib/tboot-dnf-posttrans/last-error` | Absent on success; written by `_write_last_error` |
| `/run/tboot-dnf-posttrans.*` | Per-invocation scratch dir (tmpfs) |
| `/run/tboot-dnf-posttrans.lock` | Decider lock (acquired only by `_mode_normal`) |
| `/etc/dnf/libdnf5-plugins/actions.d/` | DNF actions plugin drop-in dir (still empty in Step 32) |
| `/var/lib/tboot-dnf-helper/UNSAFE-TO-REBOOT` | Reboot-safety sentinel (still absent in Step 32) |
| `/usr/local/sbin/tboot-dnf-helper` | Helper (frozen since B.2.2) |
| `/usr/local/sbin/tboot-predict-pcr11` | Predict sibling (frozen since B.2.2) |
| `/etc/kernel/install.d/80-tpm2-sign.install` | Signing hook (frozen since Module 5 / B.1) |

> [!important] Fedora 43 UsrMerge layout
> On Fedora 43, `/usr/local/sbin` is a relative symlink to `bin` (`lrwxrwxrwx /usr/local/sbin -> bin`). Install-target safety checks that reject any symlink in the path will falsely block the install. The sub-gate 3.2 block below resolves `/usr/local/sbin` via `readlink -f`, asserts the resolved literal equals `/usr/local/bin`, and performs the atomic publish (`mktemp` + `install` + `mv -T`) inside the resolved real directory. The canonical user-facing path `/usr/local/sbin/tboot-dnf-posttrans` continues to work via the directory symlink. See `06F` I.3.

### Sub-gate 3.0: Pre-flight + rollback verification (read-only)

Step 1 runs on the Proxmox host (`pve-host`). All other steps run inside `tmux` on VM 500 as root. All operations are read-only.

```bash
# 1. [Proxmox host pve-host] Rollback anchor reachable (tree-prefix-tolerant column-1 match per 06F I.1).
qm listsnapshot 500 \
  | awk '
      {
        line=$0
        sub(/^[[:space:]]*/, "", line)
        sub(/^`->[[:space:]]*/, "", line)
        sub(/^->[[:space:]]*/, "", line)
        split(line, f, /[[:space:]]+/)
        if (f[1]=="module5-b2-2-real-run-validated") found=1
      }
      END{exit !found}
    ' \
  || { echo "FAIL: rollback anchor module5-b2-2-real-run-validated missing"; exit 1; }
echo "ok: rollback anchor module5-b2-2-real-run-validated present"
```

```bash
# 2a. [VM 500] Currently running the B.2.2-validated kernel.
[[ "$(uname -r)" == "6.19.14-200.fc43.x86_64" ]] \
  || { echo "FAIL: uname=$(uname -r); not the B.2.2 kernel"; exit 1; }
echo "ok: kernel $(uname -r)"

# 2b. Booted UKI sha matches B.2.2 closure value.
sha256sum /boot/efi/EFI/Linux/0123456789abcdef0123456789abcdef-6.19.14-200.fc43.x86_64.efi \
  | awk '{print $1}' \
  | grep -Ex 'b6002e666afdf71e5a083311295ac6a5f3ef3b443ceb4ec8c16c9ddb41077943' \
  || { echo "FAIL: booted UKI sha drift"; exit 1; }
echo "ok: booted UKI sha b6002e666afdf71e5a083311295ac6a5f3ef3b443ceb4ec8c16c9ddb41077943"
```

```bash
# 2c-d. systemd-boot default + current entry via bootctl status parsing (per 06F I.2:
#       bootctl get-default returns empty on this Fedora 43 VM despite bootctl status
#       reporting the correct Default Entry: parse bootctl status instead).
HOOK_ID="0123456789abcdef0123456789abcdef-6.19.14-200.fc43.x86_64.efi"
bootctl_status="$(bootctl status 2>/dev/null)"
default_entry="$(awk -F': ' '/^[[:space:]]*Default Entry:/ {print $2; exit}' <<< "$bootctl_status" | xargs)"
current_entry="$(awk -F': ' '/^[[:space:]]*Current Entry:/ {print $2; exit}' <<< "$bootctl_status" | xargs)"
[[ "$default_entry" == "$HOOK_ID" ]] \
  || { echo "FAIL: systemd-boot default is '${default_entry}', expected '${HOOK_ID}'"; printf '%s\n' "$bootctl_status"; exit 1; }
[[ "$current_entry" == "$HOOK_ID" ]] \
  || { echo "FAIL: current boot entry is '${current_entry}', expected '${HOOK_ID}'"; printf '%s\n' "$bootctl_status"; exit 1; }
echo "ok: systemd-boot default entry is $default_entry"
echo "ok: current boot entry is $current_entry"
```

```bash
# 3. Runtime PCR 11 byte-matches stored prediction, both validated as 64-uppercase-hex.
runtime_pcr11="$(tr -d '[:space:]' < /sys/class/tpm/tpm0/pcr-sha256/11 | tr 'a-f' 'A-F')"
expected_pcr11="$(tr -d '[:space:]' < /root/tboot-lab/state/expected-pcr11-after-hook-uki.txt | tr 'a-f' 'A-F')"
[[ "$runtime_pcr11" =~ ^[0-9A-F]{64}$ ]] || { echo "FAIL: runtime PCR 11 format"; exit 1; }
[[ "$expected_pcr11" =~ ^[0-9A-F]{64}$ ]] || { echo "FAIL: expected PCR 11 format"; exit 1; }
[[ "$runtime_pcr11" == "$expected_pcr11" ]] \
  || { echo "FAIL: PCR 11 mismatch (runtime=${runtime_pcr11}, expected=${expected_pcr11})"; exit 1; }
echo "ok: PCR 11 ${runtime_pcr11}"
```

```bash
# 4. /run is tmpfs (decider scratch substrate).
[[ "$(stat -f -c '%T' /run)" == "tmpfs" ]] || { echo "FAIL: /run is not tmpfs"; exit 1; }
echo "ok: /run is tmpfs"

# 5-7. Frozen artifact hashes from B.2.2 closure (hook + helper + predict + booted UKI).
sha256sum /etc/kernel/install.d/80-tpm2-sign.install | awk '{print $1}' \
  | grep -Ex '5857e51d5551e05a7ca384f71529ec2fef07f5a6dbe40df70c6bbe49d720947e' \
  || { echo "FAIL: hook sha drift"; exit 1; }
sha256sum /usr/local/sbin/tboot-dnf-helper | awk '{print $1}' \
  | grep -Ex '937afc7a0d199bf7b7d1a5f5bcccf45c37c0f4f7dd0ad924b2567c4da7125e44' \
  || { echo "FAIL: helper sha drift"; exit 1; }
sha256sum /usr/local/sbin/tboot-predict-pcr11 | awk '{print $1}' \
  | grep -Ex 'b729ec271dd76cd8b4581f16c86e50fc28b947f2a1de5a7e16c6d5828591e3de' \
  || { echo "FAIL: predict sha drift"; exit 1; }
echo "ok: 4 frozen-artifact shas stable"
```

```bash
# 8. Gate 3 negative pre-conditions: decider absent, state-dir absent, sentinel absent.
#    Use compound -e + ! -L tests to reject dangling symlinks at the install target.
[[ ! -e /usr/local/sbin/tboot-dnf-posttrans && ! -L /usr/local/sbin/tboot-dnf-posttrans ]] \
  || { echo "FAIL: decider path already exists or is a symlink"; exit 1; }
[[ ! -e /var/lib/tboot-dnf-posttrans && ! -L /var/lib/tboot-dnf-posttrans ]] \
  || { echo "FAIL: decider state-dir path already exists or is a symlink"; exit 1; }
[[ ! -e /var/lib/tboot-dnf-helper/UNSAFE-TO-REBOOT && ! -L /var/lib/tboot-dnf-helper/UNSAFE-TO-REBOOT ]] \
  || { echo "FAIL: UNSAFE-TO-REBOOT sentinel present"; exit 1; }
echo "ok: 3 Gate 3 negative pre-conditions held"
```

```bash
# 9. actions.d/ a real directory, zero active .actions files.
[[ -d /etc/dnf/libdnf5-plugins/actions.d && ! -L /etc/dnf/libdnf5-plugins/actions.d ]] \
  || { echo "FAIL: actions.d missing or symlink"; exit 1; }
n_actions="$(find /etc/dnf/libdnf5-plugins/actions.d/ -maxdepth 1 -name '*.actions' -type f | wc -l)"
[ "$n_actions" -eq 0 ] || { echo "FAIL: active .actions present (count=${n_actions})"; exit 1; }
echo "ok: actions.d clean (0 active .actions)"

# 10. Helper state-dir intact (0700 root:root, not symlink).
[[ -d /var/lib/tboot-dnf-helper && ! -L /var/lib/tboot-dnf-helper ]] \
  || { echo "FAIL: helper state-dir missing or symlink"; exit 1; }
stat -c 'mode=%a owner=%U:%G  %n' /var/lib/tboot-dnf-helper \
  | grep -Ex 'mode=700 owner=root:root  /var/lib/tboot-dnf-helper' \
  || { echo "FAIL: helper state-dir wrong perms"; exit 1; }

# 11. libdnf5-plugin-actions NVR + plugin-level enabled flag.
rpm -q libdnf5-plugin-actions | grep -Ex 'libdnf5-plugin-actions-5\.2\.18\.0-3\.fc43\.x86_64' \
  || { echo "FAIL: libdnf5-plugin-actions NVR drift"; exit 1; }
grep -E '^[[:space:]]*enabled[[:space:]]*=[[:space:]]*1[[:space:]]*$' \
     /etc/dnf/libdnf5-plugins/actions.conf \
  || { echo "FAIL: plugin-level enabled flag not =1"; exit 1; }

# 12. Staged decider artifact still present at /tmp (sub-gate 3.1 will sha-verify it).
[[ -f /tmp/tboot-dnf-posttrans.staged && ! -L /tmp/tboot-dnf-posttrans.staged ]] \
  || { echo "FAIL: /tmp/tboot-dnf-posttrans.staged missing or symlink: re-run Step 31 (Gate 2)"; exit 1; }
echo "ok: staged decider present at /tmp/tboot-dnf-posttrans.staged"
echo "=== sub-gate 3.0 PASS ==="
```

### Sub-gate 3.1: Re-verify staged decider identity (read-only)

```bash
bash <<'EOF'
set -u

STAGED="/tmp/tboot-dnf-posttrans.staged"
EXP_SHA="1e75110935a72fc9433c9b4bf5688d26bc6d3bd1e02f4388b5e1f32ff1e5cc05"
EXP_SIZE=35164
EXP_MODE=644
EXP_OWNER="root:root"

[[ -f "$STAGED" && ! -L "$STAGED" ]] \
  || { echo "FAIL: ${STAGED} is missing or is a symlink"; exit 1; }

stat_line="$(stat -c 'type=%F mode=%a owner=%U:%G size=%s' "$STAGED" 2>/dev/null || true)"
expect="type=regular file mode=${EXP_MODE} owner=${EXP_OWNER} size=${EXP_SIZE}"
[[ "$stat_line" == "$expect" ]] \
  || { echo "FAIL: staged file stat mismatch"; echo "  got:    $stat_line"; echo "  expect: $expect"; exit 1; }

sha256sum "$STAGED" | awk '{print $1}' | grep -Ex "$EXP_SHA" \
  || { echo "FAIL: staged sha256 drift"; sha256sum "$STAGED"; exit 1; }

bash -n "$STAGED" || { echo "FAIL: bash -n parse error"; exit 1; }

echo "=== sub-gate 3.1 PASS ==="
echo "  path:    ${STAGED}"
echo "  sha256:  ${EXP_SHA}"
echo "  size:    ${EXP_SIZE} bytes"
echo "  mode:    ${EXP_MODE}"
echo "  owner:   ${EXP_OWNER}"
echo "  bash -n: PASS"
EOF
```

### Sub-gate 3.2: Install decider at canonical path (UsrMerge-aware atomic publish)

> [!important] UsrMerge handling
> `/usr/local/sbin` is a symlink to `bin` on Fedora 43 (per `06F` I.3). The block below resolves the symlink via `readlink -f`, asserts the resolved literal equals `/usr/local/bin`, creates the temp file inside the resolved real directory, copies the staged content with `install -m 0755`, and atomically publishes via `mv -T`. A path-prefix-guarded `trap` cleans up the temp on any failure before publish; after a successful publish the trap is disarmed (`TMP=""`).

```bash
bash <<'EOF'
set -u

SRC="/tmp/tboot-dnf-posttrans.staged"
DST="/usr/local/sbin/tboot-dnf-posttrans"           # canonical (user-facing) path
DST_DIR="/usr/local/sbin"                            # canonical directory (a symlink on Fedora)
DST_REALDIR="$(readlink -f "$DST_DIR" 2>/dev/null || true)"
DST_REAL="${DST_REALDIR}/tboot-dnf-posttrans"       # resolved real path
EXP_REALDIR="/usr/local/bin"                         # Fedora 43 UsrMerge tail
EXP_SHA="1e75110935a72fc9433c9b4bf5688d26bc6d3bd1e02f4388b5e1f32ff1e5cc05"
EXP_SIZE=35164
EXP_MODE=755
EXP_OWNER="root:root"
TMP_PREFIX="${DST_REALDIR}/.tboot-dnf-posttrans."

TMP=""
cleanup_tmp() {
    if [[ -n "$TMP" && -e "$TMP" && ! -L "$TMP" ]]; then
        case "$TMP" in
            "${TMP_PREFIX}"*) rm -f -- "$TMP"; echo "cleanup: removed temp ${TMP}" ;;
            *) echo "cleanup: refusing to remove ${TMP} (does not match temp prefix ${TMP_PREFIX})" ;;
        esac
    fi
}
trap cleanup_tmp EXIT

# Pre-checks.
[[ -f "$SRC" && ! -L "$SRC" ]] || { echo "FAIL: ${SRC} missing or is a symlink"; exit 1; }
sha256sum "$SRC" | awk '{print $1}' | grep -Ex "$EXP_SHA" \
  || { echo "FAIL: staged source sha drift"; sha256sum "$SRC"; exit 1; }
[[ -n "$DST_REALDIR" ]] || { echo "FAIL: readlink -f ${DST_DIR} returned empty"; exit 1; }
[[ "$DST_REALDIR" == "$EXP_REALDIR" ]] \
  || { echo "FAIL: ${DST_DIR} resolves to ${DST_REALDIR} (expected ${EXP_REALDIR})"; exit 1; }
echo "ok: ${DST_DIR} -> ${DST_REALDIR} (UsrMerge layout)"
[[ -d "$DST_REALDIR" && ! -L "$DST_REALDIR" ]] \
  || { echo "FAIL: resolved ${DST_REALDIR} missing or is a symlink"; exit 1; }
dst_dir_stat="$(stat -c 'mode=%a owner=%U:%G type=%F' "$DST_REALDIR" 2>/dev/null || true)"
[[ "$dst_dir_stat" == "mode=755 owner=root:root type=directory" ]] \
  || { echo "FAIL: ${DST_REALDIR} unsafe metadata (got: ${dst_dir_stat})"; exit 1; }
[[ ! -e "$DST" && ! -L "$DST" ]] || { echo "FAIL: ${DST} already exists"; exit 1; }
[[ ! -e "$DST_REAL" && ! -L "$DST_REAL" ]] || { echo "FAIL: ${DST_REAL} already exists"; exit 1; }
echo "ok: pre-checks PASS"

# Stage temp inside DST_REALDIR (same filesystem as DST_REAL, so mv -T is atomic).
TMP="$(mktemp "${TMP_PREFIX}XXXXXX")" || { TMP=""; echo "FAIL: mktemp failed"; exit 1; }
case "$TMP" in "${TMP_PREFIX}"*) : ;; *) echo "FAIL: mktemp prefix mismatch ${TMP}"; exit 1 ;; esac
echo "ok: temp staged at ${TMP}"

install -m "$EXP_MODE" -o root -g root "$SRC" "$TMP" \
  || { echo "FAIL: install(1) failed"; exit 1; }
echo "ok: install(1) wrote ${TMP}"

# Pre-publish verification on TMP.
[[ -f "$TMP" && ! -L "$TMP" ]] || { echo "FAIL: TMP invalid after install"; exit 1; }
stat_line="$(stat -c 'type=%F mode=%a owner=%U:%G size=%s' "$TMP" 2>/dev/null || true)"
expect="type=regular file mode=${EXP_MODE} owner=${EXP_OWNER} size=${EXP_SIZE}"
[[ "$stat_line" == "$expect" ]] || { echo "FAIL: TMP stat mismatch"; exit 1; }
sha256sum "$TMP" | awk '{print $1}' | grep -Ex "$EXP_SHA" || { echo "FAIL: TMP sha drift"; exit 1; }
[[ -x "$TMP" ]] || { echo "FAIL: TMP not executable"; exit 1; }
bash -n "$TMP" || { echo "FAIL: TMP bash -n"; exit 1; }
echo "ok: TMP verified"

# Atomic publish: mv -T TMP -> DST_REAL (same filesystem, atomic rename).
mv -T "$TMP" "$DST_REAL" || { echo "FAIL: mv -T failed"; exit 1; }
TMP=""  # disarm trap; publish succeeded
echo "ok: atomically published ${DST_REAL}"

# Post-publish verification through both paths.
[[ -f "$DST_REAL" && ! -L "$DST_REAL" ]] || { echo "FAIL: DST_REAL invalid"; exit 1; }
[[ -f "$DST" && ! -L "$DST" ]] || { echo "FAIL: canonical DST invalid"; exit 1; }
resolved_dst="$(readlink -f "$DST" 2>/dev/null || true)"
[[ "$resolved_dst" == "$DST_REAL" ]] \
  || { echo "FAIL: canonical resolves to '${resolved_dst}', expected '${DST_REAL}'"; exit 1; }
echo "ok: canonical ${DST} resolves to ${DST_REAL}"

stat_line="$(stat -c 'type=%F mode=%a owner=%U:%G size=%s' "$DST_REAL" 2>/dev/null || true)"
[[ "$stat_line" == "$expect" ]] || { echo "FAIL: DST_REAL stat mismatch"; exit 1; }
sha256sum "$DST_REAL" | awk '{print $1}' | grep -Ex "$EXP_SHA" || { echo "FAIL: DST_REAL sha drift"; exit 1; }
[[ -x "$DST_REAL" ]] || { echo "FAIL: DST_REAL not executable"; exit 1; }
bash -n "$DST_REAL" || { echo "FAIL: DST_REAL bash -n"; exit 1; }
echo "ok: DST_REAL verified"

# SRC integrity post-publish (install copies; does not move).
[[ -f "$SRC" && ! -L "$SRC" ]] || { echo "FAIL: SRC disappeared"; exit 1; }
sha256sum "$SRC" | awk '{print $1}' | grep -Ex "$EXP_SHA" || { echo "FAIL: SRC sha changed"; exit 1; }
echo "ok: SRC unchanged by install"

# Negative invariants (7).
[[ ! -e /var/lib/tboot-dnf-posttrans && ! -L /var/lib/tboot-dnf-posttrans ]] \
  || { echo "FAIL: state-dir appeared (sub-gate 3.4 scope)"; exit 1; }
[[ ! -e /var/lib/tboot-dnf-helper/UNSAFE-TO-REBOOT && ! -L /var/lib/tboot-dnf-helper/UNSAFE-TO-REBOOT ]] \
  || { echo "FAIL: sentinel appeared"; exit 1; }
n_actions="$(find /etc/dnf/libdnf5-plugins/actions.d/ -maxdepth 1 -name '*.actions' -type f | wc -l)"
[ "$n_actions" -eq 0 ] || { echo "FAIL: .actions appeared"; exit 1; }
for tuple in \
  '/etc/kernel/install.d/80-tpm2-sign.install:5857e51d5551e05a7ca384f71529ec2fef07f5a6dbe40df70c6bbe49d720947e' \
  '/usr/local/sbin/tboot-dnf-helper:937afc7a0d199bf7b7d1a5f5bcccf45c37c0f4f7dd0ad924b2567c4da7125e44' \
  '/usr/local/sbin/tboot-predict-pcr11:b729ec271dd76cd8b4581f16c86e50fc28b947f2a1de5a7e16c6d5828591e3de' \
  "/boot/efi/EFI/Linux/0123456789abcdef0123456789abcdef-6.19.14-200.fc43.x86_64.efi:b6002e666afdf71e5a083311295ac6a5f3ef3b443ceb4ec8c16c9ddb41077943"; do
    path="${tuple%%:*}"; sha="${tuple##*:}"
    sha256sum "$path" | awk '{print $1}' | grep -Ex "$sha" \
      || { echo "FAIL: ${path} sha drifted"; exit 1; }
done
pcr_now="$(tr -d '[:space:]' < /root/tboot-lab/state/expected-pcr11-after-hook-uki.txt | tr 'a-f' 'A-F')"
[[ "$pcr_now" == "A03EB49C9335D721758AE8AA9C34DA87664C5CAF9FAD7C314C68C038B16435DD" ]] \
  || { echo "FAIL: PCR 11 prediction file changed"; exit 1; }
echo "ok: 7 negative invariants held"
echo "=== sub-gate 3.2 PASS ==="
EOF
```

### Sub-gate 3.3: Verify installed decider identity (read-only)

```bash
bash <<'EOF'
set -u

DST="/usr/local/sbin/tboot-dnf-posttrans"
DST_REAL="/usr/local/bin/tboot-dnf-posttrans"
EXP_SHA="1e75110935a72fc9433c9b4bf5688d26bc6d3bd1e02f4388b5e1f32ff1e5cc05"
EXP_VERSION="tboot-dnf-posttrans 1.0.0"

[[ -f "$DST" && ! -L "$DST" ]] || { echo "FAIL: ${DST} invalid"; exit 1; }
[[ -f "$DST_REAL" && ! -L "$DST_REAL" ]] || { echo "FAIL: ${DST_REAL} invalid"; exit 1; }
resolved="$(readlink -f "$DST" 2>/dev/null || true)"
[[ "$resolved" == "$DST_REAL" ]] || { echo "FAIL: canonical resolves to '${resolved}'"; exit 1; }
echo "ok: ${DST} -> ${DST_REAL}"

# Inode identity through both paths (confirms one file, two name paths).
ino_dst="$(stat -c '%i' "$DST")"; ino_real="$(stat -c '%i' "$DST_REAL")"
dev_dst="$(stat -c '%d' "$DST")"; dev_real="$(stat -c '%d' "$DST_REAL")"
[[ "$ino_dst" == "$ino_real" && "$dev_dst" == "$dev_real" ]] \
  || { echo "FAIL: paths do not share inode (${dev_dst}/${ino_dst} vs ${dev_real}/${ino_real})"; exit 1; }
echo "ok: both paths share inode ${dev_real}:${ino_real}"

stat_line="$(stat -c 'type=%F mode=%a owner=%U:%G size=%s' "$DST_REAL")"
[[ "$stat_line" == "type=regular file mode=755 owner=root:root size=35164" ]] \
  || { echo "FAIL: installed stat mismatch (got: ${stat_line})"; exit 1; }
sha256sum "$DST_REAL" | awk '{print $1}' | grep -Ex "$EXP_SHA" || { echo "FAIL: sha drift"; exit 1; }
[[ -x "$DST_REAL" ]] || { echo "FAIL: not executable"; exit 1; }
bash -n "$DST_REAL" || { echo "FAIL: bash -n"; exit 1; }

# --version / --help bypass _preflight.
version_output="$("$DST" --version 2>&1)"
[[ $? -eq 0 ]] || { echo "FAIL: --version rc"; exit 1; }
[[ "$version_output" == "$EXP_VERSION" ]] \
  || { echo "FAIL: --version: '${version_output}'"; exit 1; }
echo "ok: --version: ${version_output}"

help_output="$("$DST" --help 2>&1)"
[[ $? -eq 0 ]] || { echo "FAIL: --help rc"; exit 1; }
for mode_flag in --self-test --debug-print --dry-run --prime --version --help; do
    grep -F -- "$mode_flag" <<< "$help_output" >/dev/null \
      || { echo "FAIL: --help missing flag ${mode_flag}"; exit 1; }
done
echo "ok: --help lists all 6 mode flags"

# Negative invariants: state-dir still absent, no sentinel, no .actions.
[[ ! -e /var/lib/tboot-dnf-posttrans && ! -L /var/lib/tboot-dnf-posttrans ]] \
  || { echo "FAIL: state-dir appeared"; exit 1; }
[[ ! -e /var/lib/tboot-dnf-helper/UNSAFE-TO-REBOOT ]] || { echo "FAIL: sentinel appeared"; exit 1; }
n_actions="$(find /etc/dnf/libdnf5-plugins/actions.d/ -maxdepth 1 -name '*.actions' -type f | wc -l)"
[ "$n_actions" -eq 0 ] || { echo "FAIL: .actions appeared"; exit 1; }
echo "ok: sub-gate 3.3 negative invariants held"
echo "=== sub-gate 3.3 PASS ==="
EOF
```

### Sub-gate 3.4: Create decider state directory

```bash
bash <<'EOF'
set -u

STATE_DIR="/var/lib/tboot-dnf-posttrans"
STATE_DIR_PARENT="/var/lib"

[[ -d "$STATE_DIR_PARENT" && ! -L "$STATE_DIR_PARENT" ]] \
  || { echo "FAIL: ${STATE_DIR_PARENT} invalid"; exit 1; }
parent_stat="$(stat -c 'mode=%a owner=%U:%G type=%F' "$STATE_DIR_PARENT")"
[[ "$parent_stat" == "mode=755 owner=root:root type=directory" ]] \
  || { echo "FAIL: ${STATE_DIR_PARENT} unsafe metadata"; exit 1; }
[[ ! -e "$STATE_DIR" && ! -L "$STATE_DIR" ]] \
  || { echo "FAIL: ${STATE_DIR} already exists"; exit 1; }

install -d -m 700 -o root -g root "$STATE_DIR" \
  || { echo "FAIL: install -d failed"; exit 1; }
echo "ok: created ${STATE_DIR}"

[[ -d "$STATE_DIR" && ! -L "$STATE_DIR" ]] || { echo "FAIL: not a real directory"; exit 1; }
dir_stat="$(stat -c 'mode=%a owner=%U:%G type=%F' "$STATE_DIR")"
[[ "$dir_stat" == "mode=700 owner=root:root type=directory" ]] \
  || { echo "FAIL: ${STATE_DIR} metadata mismatch (got: ${dir_stat})"; exit 1; }
[[ -w "$STATE_DIR" ]] || { echo "FAIL: not writable"; exit 1; }
n_entries="$(find "$STATE_DIR" -mindepth 1 -maxdepth 1 -printf '.' | wc -c)"
[[ "$n_entries" -eq 0 ]] || { echo "FAIL: not empty (${n_entries})"; exit 1; }
[[ ! -e "${STATE_DIR}/boot-input-manifest.baseline" ]] \
  || { echo "FAIL: baseline appeared before sub-gate 3.6"; exit 1; }
dev_dir="$(stat -c '%d' "$STATE_DIR")"; dev_parent="$(stat -c '%d' "$STATE_DIR_PARENT")"
[[ "$dev_dir" == "$dev_parent" ]] \
  || { echo "FAIL: ${STATE_DIR} on different fs than parent"; exit 1; }

echo "=== sub-gate 3.4 PASS ==="
echo "  state-dir: ${STATE_DIR}"
echo "  mode:      0700 root:root"
echo "  empty:     yes"
echo "  fs dev:    ${dev_dir}"
EOF
```

### Sub-gate 3.5: Run `--self-test` (first preflighted execution)

> [!warning] `--self-test` does not emit `ok: helper present and executable`
> Per `06F` I.5: the decider's `_preflight` switches on mode; for `self-test`/`debug-print` the helper-presence branch is `warn`-only on missing helper, and emits no positive `ok:` line on helper present. Do NOT assert the positive helper line in self-test output. The helper sha is verified separately via the frozen-artifact invariant block below.

```bash
bash <<'EOF'
set -u

DST="/usr/local/sbin/tboot-dnf-posttrans"
STATE_DIR="/var/lib/tboot-dnf-posttrans"
JOURNAL_TAG="tboot-dnf-posttrans"

SINCE="$(date '+%Y-%m-%d %H:%M:%S')"
scratch_before="$(find /run -maxdepth 1 -name 'tboot-dnf-posttrans.*' -type d 2>/dev/null | sort)"
sleep 1

selftest_output="$("$DST" --self-test 2>&1)"
selftest_rc=$?

[[ "$selftest_rc" -eq 0 ]] || { echo "FAIL: --self-test rc=${selftest_rc}"; exit 1; }
echo "ok: --self-test rc=0"

# Positive contract lines that --self-test MUST emit (per 06F I.5: helper line excluded).
for needle in \
  "ok: running as root" \
  "ok: required binaries present" \
  "ok: trust-boundary self-test PASS" \
  "mode=self-test; preflight + trust-boundary scan only; no compute, no state writes" \
  "exit reason: self-test PASS"; do
    grep -Fq "$needle" <<< "$selftest_output" \
      || { echo "FAIL: missing line: ${needle}"; exit 1; }
done

if grep -Eq '\] err:' <<< "$selftest_output"; then
    echo "FAIL: err: lines in --self-test output"; exit 1
fi
if grep -Fq "trust-boundary: forbidden token" <<< "$selftest_output" \
   || grep -Fq "trust-boundary: decider references the helper lock path" <<< "$selftest_output"; then
    echo "FAIL: trust-boundary violation"; exit 1
fi

# Journal cross-check (use -q per 06F I.4 to suppress "-- No entries --" advisory).
journal_output="$(journalctl -q -t "$JOURNAL_TAG" --since "$SINCE" --no-pager 2>/dev/null || true)"
grep -Fq "trust-boundary self-test PASS" <<< "$journal_output" \
  || { echo "FAIL: trust-boundary PASS not in journald"; exit 1; }
grep -Eq 'err:' <<< "$journal_output" && { echo "FAIL: err: in journal"; exit 1; }

# State-dir, scratch, lock unchanged.
[[ -d "$STATE_DIR" && ! -L "$STATE_DIR" ]] || { echo "FAIL: state-dir invalid"; exit 1; }
dir_stat="$(stat -c 'mode=%a owner=%U:%G type=%F' "$STATE_DIR")"
[[ "$dir_stat" == "mode=700 owner=root:root type=directory" ]] \
  || { echo "FAIL: state-dir drift"; exit 1; }
n="$(find "$STATE_DIR" -mindepth 1 -maxdepth 1 -printf '.' | wc -c)"
[[ "$n" -eq 0 ]] || { echo "FAIL: state-dir not empty"; exit 1; }
scratch_after="$(find /run -maxdepth 1 -name 'tboot-dnf-posttrans.*' -type d 2>/dev/null | sort)"
[[ "$scratch_after" == "$scratch_before" ]] || { echo "FAIL: scratch leak"; exit 1; }
[[ ! -e /run/tboot-dnf-posttrans.lock ]] || { echo "FAIL: decider lock present"; exit 1; }

# Frozen artifact invariants (helper sha verified here, not via --self-test output).
for tuple in \
  '/usr/local/bin/tboot-dnf-posttrans:1e75110935a72fc9433c9b4bf5688d26bc6d3bd1e02f4388b5e1f32ff1e5cc05' \
  '/etc/kernel/install.d/80-tpm2-sign.install:5857e51d5551e05a7ca384f71529ec2fef07f5a6dbe40df70c6bbe49d720947e' \
  '/usr/local/sbin/tboot-dnf-helper:937afc7a0d199bf7b7d1a5f5bcccf45c37c0f4f7dd0ad924b2567c4da7125e44' \
  '/usr/local/sbin/tboot-predict-pcr11:b729ec271dd76cd8b4581f16c86e50fc28b947f2a1de5a7e16c6d5828591e3de' \
  "/boot/efi/EFI/Linux/0123456789abcdef0123456789abcdef-6.19.14-200.fc43.x86_64.efi:b6002e666afdf71e5a083311295ac6a5f3ef3b443ceb4ec8c16c9ddb41077943"; do
    path="${tuple%%:*}"; sha="${tuple##*:}"
    sha256sum "$path" | awk '{print $1}' | grep -Ex "$sha" \
      || { echo "FAIL: ${path} sha drift"; exit 1; }
done
echo "=== sub-gate 3.5 PASS ==="
EOF
```

### Sub-gate 3.6: Baseline prime via `tboot-dnf-posttrans --prime`

> [!important] `--prime` exactly once; no re-prime refusal probe in this gate
> Per `06F` I.6: the decider intentionally logs `err: baseline already exists at ${BASELINE_FILE}; refusing to overwrite without --confirm-prime` and returns rc=11 on a second `--prime` without `--confirm-prime`. That refusal-guard probe deliberately injects an `err:` line into the `tboot-dnf-posttrans` journal slice, which would contaminate Gate 3.6's "journal err-free on success" invariant. Run `--prime` exactly ONCE in this gate. The refusal-guard probe belongs in a separate optional error-path validation, not in the clean Gate 3.6.

```bash
bash <<'EOF'
set -u

DST="/usr/local/sbin/tboot-dnf-posttrans"
STATE_DIR="/var/lib/tboot-dnf-posttrans"
BASELINE_FILE="${STATE_DIR}/boot-input-manifest.baseline"
LAST_MANIFEST_FILE="${STATE_DIR}/last-computed-manifest"
LAST_DECISION_FILE="${STATE_DIR}/last-decision"
LAST_ERROR_FILE="${STATE_DIR}/last-error"
HELPER_TAG="tboot-dnf-helper"
JOURNAL_TAG="tboot-dnf-posttrans"

SINCE="$(date '+%Y-%m-%d %H:%M:%S')"
scratch_before="$(find /run -maxdepth 1 -name 'tboot-dnf-posttrans.*' -type d 2>/dev/null | sort)"
sleep 1

# Pre-prime invariants.
[[ -d "$STATE_DIR" && ! -L "$STATE_DIR" ]] || { echo "FAIL: state-dir invalid"; exit 1; }
n_entries="$(find "$STATE_DIR" -mindepth 1 -maxdepth 1 -printf '.' | wc -c)"
[[ "$n_entries" -eq 0 ]] || { echo "FAIL: state-dir not empty"; exit 1; }
[[ ! -e "$BASELINE_FILE" && ! -L "$BASELINE_FILE" ]] \
  || { echo "FAIL: baseline already present"; exit 1; }

# Run --prime exactly once.
prime_output="$("$DST" --prime 2>&1)"
prime_rc=$?
[[ "$prime_rc" -eq 0 ]] || { echo "FAIL: --prime rc=${prime_rc}"; printf '%s\n' "$prime_output"; exit 1; }

# Positive contract lines (prime mode DOES emit the helper line: see 06F I.5).
for needle in \
  "ok: running as root" \
  "ok: required binaries present" \
  "ok: helper present and executable" \
  "ok: trust-boundary self-test PASS" \
  "ok: state-dir ${STATE_DIR} safe" \
  "ok: scratch" \
  "mode=prime (Gate 3 baseline-prime)" \
  "exit reason: baseline primed at ${BASELINE_FILE}"; do
    grep -Fq "$needle" <<< "$prime_output" \
      || { echo "FAIL: missing line: ${needle}"; exit 1; }
done

if grep -Eq '\] err:' <<< "$prime_output"; then
    echo "FAIL: --prime emitted err:"; exit 1
fi
if grep -Eqi 'invoke /usr/local/sbin/tboot-dnf-helper' <<< "$prime_output"; then
    echo "FAIL: --prime invoked helper (out of scope for prime mode)"; exit 1
fi

# Journal cross-check (-q per 06F I.4).
journal_output="$(journalctl -q -t "$JOURNAL_TAG" --since "$SINCE" --no-pager 2>/dev/null || true)"
grep -Fq "exit reason: baseline primed at ${BASELINE_FILE}" <<< "$journal_output" \
  || { echo "FAIL: baseline-primed line missing from journal"; exit 1; }
grep -Eq 'err:' <<< "$journal_output" && { echo "FAIL: err: in decider journal"; exit 1; }

# Helper journal silent (-q per 06F I.4).
helper_journal="$(journalctl -q -t "$HELPER_TAG" --since "$SINCE" --no-pager 2>/dev/null || true)"
[[ -z "$helper_journal" ]] \
  || { echo "FAIL: helper journal fired during --prime"; printf '%s\n' "$helper_journal"; exit 1; }

# Post-prime state-dir contents.
[[ -f "$BASELINE_FILE" && ! -L "$BASELINE_FILE" ]] || { echo "FAIL: baseline missing"; exit 1; }
bf_stat="$(stat -c 'mode=%a owner=%U:%G type=%F size=%s' "$BASELINE_FILE")"
[[ "$bf_stat" =~ ^mode=600\ owner=root:root\ type=regular\ file\ size=([1-9][0-9]*)$ ]] \
  || { echo "FAIL: baseline metadata mismatch (got: ${bf_stat})"; exit 1; }
bf_size="${BASH_REMATCH[1]}"
bf_sha="$(sha256sum "$BASELINE_FILE" | awk '{print $1}')"
echo "ok: baseline ${bf_stat}"
echo "ok: baseline sha ${bf_sha}"

[[ -f "$LAST_MANIFEST_FILE" && ! -L "$LAST_MANIFEST_FILE" ]] \
  || { echo "FAIL: last-computed-manifest missing"; exit 1; }
lmf_stat="$(stat -c 'mode=%a owner=%U:%G type=%F size=%s' "$LAST_MANIFEST_FILE")"
[[ "$lmf_stat" == "mode=600 owner=root:root type=regular file size=${bf_size}" ]] \
  || { echo "FAIL: last-computed-manifest stat / size mismatch"; exit 1; }
lmf_sha="$(sha256sum "$LAST_MANIFEST_FILE" | awk '{print $1}')"
[[ "$lmf_sha" == "$bf_sha" ]] \
  || { echo "FAIL: last-computed-manifest sha != baseline sha"; exit 1; }

[[ -f "$LAST_DECISION_FILE" && ! -L "$LAST_DECISION_FILE" ]] \
  || { echo "FAIL: last-decision missing"; exit 1; }
ld_stat="$(stat -c 'mode=%a owner=%U:%G type=%F' "$LAST_DECISION_FILE")"
[[ "$ld_stat" == "mode=600 owner=root:root type=regular file" ]] \
  || { echo "FAIL: last-decision stat mismatch"; exit 1; }
# Schema: 6 key=value lines.
for key in epoch date decision detail compute_duration_s decider_version; do
    grep -Eq "^${key}=" "$LAST_DECISION_FILE" \
      || { echo "FAIL: last-decision missing key '${key}'"; exit 1; }
done
grep -Fxq "decision=primed" "$LAST_DECISION_FILE" \
  || { echo "FAIL: last-decision: decision != 'primed'"; exit 1; }
grep -Fxq "detail=Gate 3 baseline prime" "$LAST_DECISION_FILE" \
  || { echo "FAIL: last-decision: detail != 'Gate 3 baseline prime'"; exit 1; }

[[ ! -e "$LAST_ERROR_FILE" && ! -L "$LAST_ERROR_FILE" ]] \
  || { echo "FAIL: last-error present after clean prime"; exit 1; }

n_stray="$(find "$STATE_DIR" -mindepth 1 -maxdepth 1 -name '.*' -type f | wc -l)"
[[ "$n_stray" -eq 0 ]] || { echo "FAIL: stray mktemp leftovers"; exit 1; }
n_entries="$(find "$STATE_DIR" -mindepth 1 -maxdepth 1 -printf '.' | wc -c)"
[[ "$n_entries" -eq 3 ]] || { echo "FAIL: state-dir entry count ${n_entries}, expected 3"; exit 1; }

# Scratch + lock cleanup.
scratch_after="$(find /run -maxdepth 1 -name 'tboot-dnf-posttrans.*' -type d 2>/dev/null | sort)"
[[ "$scratch_after" == "$scratch_before" ]] || { echo "FAIL: scratch leak"; exit 1; }
[[ ! -e /run/tboot-dnf-posttrans.lock ]] || { echo "FAIL: decider lock present"; exit 1; }

# Frozen invariants + no sentinel + no .actions.
[[ ! -e /var/lib/tboot-dnf-helper/UNSAFE-TO-REBOOT ]] || { echo "FAIL: sentinel appeared"; exit 1; }
n_actions="$(find /etc/dnf/libdnf5-plugins/actions.d/ -maxdepth 1 -name '*.actions' -type f | wc -l)"
[ "$n_actions" -eq 0 ] || { echo "FAIL: .actions appeared"; exit 1; }

echo "=== sub-gate 3.6 PASS: baseline primed ==="
echo "  baseline path:     ${BASELINE_FILE}"
echo "  baseline sha256:   ${bf_sha}"
echo "  baseline size:     ${bf_size} bytes"
echo "  state-dir entries: 3"
echo "  helper invoked:    NO"
echo "  sentinel set:      NO"
EOF
```

**Sub-gate 3.6 result for the canonical rebuild (2026-05-24):** baseline sha `75dc4f5651fc8996d39b545ac0bf7244a2a5888403b0ba0cd448a671daf80160`, size 19752 bytes, 157 records across the 14 manifest categories, compute duration 1 s. These values are rebuild-specific; the architectural invariants (path, mode 0600, owner, the 14-category structure, the same-sha relation between baseline and `last-computed-manifest`) are stable and recorded in the architectural-invariants table at the end of this runbook. The raw baseline content is NOT included in this runbook by design; it is regenerable via `--prime` on a clean state.

### Sub-gate 3.7: Post-prime cross-validation (single `--debug-print`)

> [!important] Single `--debug-print` invocation; subshell + tempfile rc capture
> Per `06F` I.7: do NOT run `--debug-print` multiple times. Do NOT capture exit codes via `output=$(CMD 2>&1 || true)`: `|| true` masks failures. The correct pattern is a subshell with separate stdout/stderr files and an `echo $? > rc_file` line inside the same subshell, parsed afterwards with `dbg_rc="$(cat "$dbg_rc_file")"`.

```bash
bash <<'EOF'
set -u

DST="/usr/local/sbin/tboot-dnf-posttrans"
STATE_DIR="/var/lib/tboot-dnf-posttrans"
BASELINE_FILE="${STATE_DIR}/boot-input-manifest.baseline"
LAST_MANIFEST_FILE="${STATE_DIR}/last-computed-manifest"
LAST_DECISION_FILE="${STATE_DIR}/last-decision"
LAST_ERROR_FILE="${STATE_DIR}/last-error"
HELPER_TAG="tboot-dnf-helper"
JOURNAL_TAG="tboot-dnf-posttrans"

# Rebuild-specific expected values: substitute with the values printed by your sub-gate 3.6 run.
EXP_BASELINE_SHA="75dc4f5651fc8996d39b545ac0bf7244a2a5888403b0ba0cd448a671daf80160"
EXP_BASELINE_SIZE=19752
EXP_DECIDER_VERSION="1.0.0"

WORK=""
TMP_PREFIX="/run/gate37."
cleanup_work() {
    if [[ -n "$WORK" && -d "$WORK" && ! -L "$WORK" ]]; then
        case "$WORK" in
            "${TMP_PREFIX}"*) rm -rf -- "$WORK" ;;
            *) echo "cleanup: refusing to remove ${WORK}" ;;
        esac
    fi
}
trap cleanup_work EXIT

WORK="$(mktemp -d "${TMP_PREFIX}XXXXXX")" || { WORK=""; echo "FAIL: mktemp -d in /run failed"; exit 1; }
case "$WORK" in "${TMP_PREFIX}"*) : ;; *) echo "FAIL: WORK prefix unexpected"; exit 1 ;; esac
chmod 0700 "$WORK"

SINCE="$(date '+%Y-%m-%d %H:%M:%S')"
scratch_before="$(find /run -maxdepth 1 -name 'tboot-dnf-posttrans.*' -type d 2>/dev/null | sort)"
sleep 1

# 1. Baseline identity stable since sub-gate 3.6 close.
bf_stat="$(stat -c 'mode=%a owner=%U:%G type=%F size=%s' "$BASELINE_FILE")"
[[ "$bf_stat" == "mode=600 owner=root:root type=regular file size=${EXP_BASELINE_SIZE}" ]] \
  || { echo "FAIL: baseline stat drift (got: ${bf_stat})"; exit 1; }
bf_sha="$(sha256sum "$BASELINE_FILE" | awk '{print $1}')"
[[ "$bf_sha" == "$EXP_BASELINE_SHA" ]] || { echo "FAIL: baseline sha drift (${bf_sha})"; exit 1; }
echo "ok: baseline identity stable (sha=${bf_sha}, size=${EXP_BASELINE_SIZE})"

# Pre-debug-print snapshot of state-dir for byte-stable comparison.
ld_content_pre="$(cat "$LAST_DECISION_FILE")"
lmf_sha_pre="$(sha256sum "$LAST_MANIFEST_FILE" | awk '{print $1}')"

# 2. last-decision schema (6 keys; key=value form).
for key in epoch date decision detail compute_duration_s decider_version; do
    grep -Eq "^${key}=" <<< "$ld_content_pre" \
      || { echo "FAIL: last-decision missing key '${key}'"; exit 1; }
done
grep -Eq '^decision=primed$' <<< "$ld_content_pre" || { echo "FAIL: decision != primed"; exit 1; }
grep -Eq '^detail=Gate 3 baseline prime$' <<< "$ld_content_pre" || { echo "FAIL: detail"; exit 1; }
grep -Eq "^decider_version=${EXP_DECIDER_VERSION}$" <<< "$ld_content_pre" \
  || { echo "FAIL: decider_version"; exit 1; }
grep -Eq '^epoch=[1-9][0-9]+$' <<< "$ld_content_pre" || { echo "FAIL: epoch"; exit 1; }
grep -Eq '^date=20[0-9]{2}-[0-9]{2}-[0-9]{2}T[0-9]{2}:[0-9]{2}:[0-9]{2}([+-][0-9]{2}:?[0-9]{2}|Z)$' \
  <<< "$ld_content_pre" || { echo "FAIL: date not ISO 8601"; exit 1; }
grep -Eq '^compute_duration_s=[0-9]+$' <<< "$ld_content_pre" || { echo "FAIL: compute_duration_s"; exit 1; }
echo "ok: last-decision schema validated (6 keys)"

# --version cross-check against last-decision decider_version.
installed_version="$("$DST" --version)"
[[ $? -eq 0 ]] || { echo "FAIL: --version rc"; exit 1; }
[[ "$installed_version" == "tboot-dnf-posttrans ${EXP_DECIDER_VERSION}" ]] \
  || { echo "FAIL: installed --version mismatch (got: '${installed_version}')"; exit 1; }
echo "ok: --version matches last-decision decider_version"

# 3. Single --debug-print with correct rc capture (no || true masking: see 06F I.7).
dbg_stdout_file="${WORK}/debug-stdout"
dbg_stderr_file="${WORK}/debug-stderr"
dbg_rc_file="${WORK}/debug-rc"
( "$DST" --debug-print >"$dbg_stdout_file" 2>"$dbg_stderr_file" ; echo $? >"$dbg_rc_file" )
dbg_rc="$(cat "$dbg_rc_file")"
[[ "$dbg_rc" =~ ^[0-9]+$ ]] || { echo "FAIL: could not parse rc ('${dbg_rc}')"; exit 1; }
[[ "$dbg_rc" -eq 0 ]] \
  || { echo "FAIL: --debug-print rc=${dbg_rc}"; echo "--- stderr ---"; cat "$dbg_stderr_file"; exit 1; }
echo "ok: --debug-print exit rc=0"

grep -Fq "ok: trust-boundary self-test PASS" "$dbg_stderr_file" \
  || { echo "FAIL: --debug-print did not run trust-boundary self-scan"; exit 1; }
grep -Fq "ok: scratch" "$dbg_stderr_file" \
  || { echo "FAIL: --debug-print did not setup scratch"; exit 1; }
if grep -Eq '\] err:' "$dbg_stderr_file"; then
    echo "FAIL: --debug-print emitted err: lines"; grep -E '\] err:' "$dbg_stderr_file"; exit 1
fi
echo "ok: --debug-print stderr clean"

# 4. Zero drift: --debug-print manifest sha matches baseline byte-for-byte (newline-normalized).
baseline_norm_sha="$(awk '{print}' "$BASELINE_FILE" | sha256sum | awk '{print $1}')"
dbg_norm_sha="$(awk '{print}' "$dbg_stdout_file" | sha256sum | awk '{print $1}')"
[[ "$baseline_norm_sha" == "$dbg_norm_sha" ]] \
  || { echo "FAIL: drift (baseline=${baseline_norm_sha}, debug-print=${dbg_norm_sha})"; \
       diff <(awk '{print}' "$BASELINE_FILE") <(awk '{print}' "$dbg_stdout_file") | head -40; \
       exit 1; }
echo "ok: --debug-print matches baseline (zero drift, sha=${dbg_norm_sha})"

# 5. State-dir not mutated by --debug-print.
bf_sha_post="$(sha256sum "$BASELINE_FILE" | awk '{print $1}')"
[[ "$bf_sha_post" == "$EXP_BASELINE_SHA" ]] || { echo "FAIL: baseline sha mutated"; exit 1; }
lmf_sha_post="$(sha256sum "$LAST_MANIFEST_FILE" | awk '{print $1}')"
[[ "$lmf_sha_post" == "$lmf_sha_pre" ]] || { echo "FAIL: last-computed-manifest sha mutated"; exit 1; }
ld_content_post="$(cat "$LAST_DECISION_FILE")"
[[ "$ld_content_post" == "$ld_content_pre" ]] || { echo "FAIL: last-decision content mutated"; exit 1; }
[[ ! -e "$LAST_ERROR_FILE" && ! -L "$LAST_ERROR_FILE" ]] \
  || { echo "FAIL: last-error appeared during --debug-print"; exit 1; }
n_entries="$(find "$STATE_DIR" -mindepth 1 -maxdepth 1 -printf '.' | wc -c)"
[[ "$n_entries" -eq 3 ]] || { echo "FAIL: state-dir entry count ${n_entries}"; exit 1; }
echo "ok: state-dir unmutated by --debug-print"

# 6. Scratch + lock cleanup.
scratch_after="$(find /run -maxdepth 1 -name 'tboot-dnf-posttrans.*' -type d 2>/dev/null | sort)"
[[ "$scratch_after" == "$scratch_before" ]] || { echo "FAIL: scratch leak"; exit 1; }
[[ ! -e /run/tboot-dnf-posttrans.lock && ! -L /run/tboot-dnf-posttrans.lock ]] \
  || { echo "FAIL: decider lock present"; exit 1; }
echo "ok: scratch + lock clean (--debug-print does not acquire lock)"

# 7. Helper not invoked (narrow scope: journal silent, sentinel absent, perms intact, sha unchanged).
helper_journal="$(journalctl -q -t "$HELPER_TAG" --since "$SINCE" --no-pager 2>/dev/null || true)"
[[ -z "$helper_journal" ]] || { echo "FAIL: helper journal fired during sub-gate 3.7"; printf '%s\n' "$helper_journal"; exit 1; }
[[ ! -e /var/lib/tboot-dnf-helper/UNSAFE-TO-REBOOT ]] || { echo "FAIL: sentinel appeared"; exit 1; }
helper_state_stat="$(stat -c 'mode=%a owner=%U:%G type=%F' /var/lib/tboot-dnf-helper)"
[[ "$helper_state_stat" == "mode=700 owner=root:root type=directory" ]] \
  || { echo "FAIL: helper state-dir drift"; exit 1; }
sha256sum /usr/local/sbin/tboot-dnf-helper | awk '{print $1}' \
  | grep -Ex '937afc7a0d199bf7b7d1a5f5bcccf45c37c0f4f7dd0ad924b2567c4da7125e44' \
  || { echo "FAIL: helper sha drift"; exit 1; }
echo "ok: helper not invoked (journal silent, sentinel absent, perms intact, sha unchanged)"

# 8. No .actions; no err: in decider journal slice.
n_actions="$(find /etc/dnf/libdnf5-plugins/actions.d/ -maxdepth 1 -name '*.actions' -type f | wc -l)"
[ "$n_actions" -eq 0 ] || { echo "FAIL: .actions appeared"; exit 1; }
decider_journal="$(journalctl -q -t "$JOURNAL_TAG" --since "$SINCE" --no-pager 2>/dev/null || true)"
if grep -Eq 'err:' <<< "$decider_journal"; then
    echo "FAIL: err: in decider journal slice"; grep -E 'err:' <<< "$decider_journal"; exit 1
fi
echo "ok: 0 .actions; no err: in decider journal slice"

# 9. Frozen artifacts.
for tuple in \
  '/usr/local/bin/tboot-dnf-posttrans:1e75110935a72fc9433c9b4bf5688d26bc6d3bd1e02f4388b5e1f32ff1e5cc05' \
  '/etc/kernel/install.d/80-tpm2-sign.install:5857e51d5551e05a7ca384f71529ec2fef07f5a6dbe40df70c6bbe49d720947e' \
  '/usr/local/sbin/tboot-predict-pcr11:b729ec271dd76cd8b4581f16c86e50fc28b947f2a1de5a7e16c6d5828591e3de' \
  "/boot/efi/EFI/Linux/0123456789abcdef0123456789abcdef-6.19.14-200.fc43.x86_64.efi:b6002e666afdf71e5a083311295ac6a5f3ef3b443ceb4ec8c16c9ddb41077943"; do
    path="${tuple%%:*}"; sha="${tuple##*:}"
    sha256sum "$path" | awk '{print $1}' | grep -Ex "$sha" || { echo "FAIL: ${path} sha drift"; exit 1; }
done
pcr_now="$(tr -d '[:space:]' < /root/tboot-lab/state/expected-pcr11-after-hook-uki.txt | tr 'a-f' 'A-F')"
[[ "$pcr_now" == "A03EB49C9335D721758AE8AA9C34DA87664C5CAF9FAD7C314C68C038B16435DD" ]] \
  || { echo "FAIL: PCR 11 prediction file changed"; exit 1; }
echo "ok: 6 frozen-artifact invariants stable"

echo "=== sub-gate 3.7 PASS: post-prime cross-validation ==="
echo "  baseline sha (stable):     ${bf_sha}"
echo "  --debug-print rc:          0"
echo "  --debug-print vs baseline: zero drift (sha=${dbg_norm_sha})"
echo "  state-dir mutated:         NO (3 files byte-stable)"
echo "  helper invoked:            NO"
EOF
```

### Sub-gate 3.8: Closure snapshot

#### Sub-gate 3.8a: VM-side final sanity check (read-only)

The 13 read-only checks below confirm the VM is still in the Gate 3.7 closure state before the snapshot is taken on the Proxmox host. Any FAIL halts Gate 3; do not proceed to sub-gate 3.8b.

```bash
bash <<'EOF'
set -u

EXP_BASELINE_SHA="75dc4f5651fc8996d39b545ac0bf7244a2a5888403b0ba0cd448a671daf80160"
EXP_DECIDER_SHA="1e75110935a72fc9433c9b4bf5688d26bc6d3bd1e02f4388b5e1f32ff1e5cc05"
EXP_HELPER_SHA="937afc7a0d199bf7b7d1a5f5bcccf45c37c0f4f7dd0ad924b2567c4da7125e44"
EXP_HOOK_SHA="5857e51d5551e05a7ca384f71529ec2fef07f5a6dbe40df70c6bbe49d720947e"
EXP_PREDICT_SHA="b729ec271dd76cd8b4581f16c86e50fc28b947f2a1de5a7e16c6d5828591e3de"
EXP_UKI_SHA="b6002e666afdf71e5a083311295ac6a5f3ef3b443ceb4ec8c16c9ddb41077943"
EXP_PCR11="A03EB49C9335D721758AE8AA9C34DA87664C5CAF9FAD7C314C68C038B16435DD"
UKI_PATH="/boot/efi/EFI/Linux/0123456789abcdef0123456789abcdef-6.19.14-200.fc43.x86_64.efi"
HELPER_TAG="tboot-dnf-helper"

# 1. baseline SHA stable
sha256sum /var/lib/tboot-dnf-posttrans/boot-input-manifest.baseline | awk '{print $1}' \
  | grep -Ex "$EXP_BASELINE_SHA" \
  || { echo "FAIL: baseline SHA drift"; exit 1; }

# 2. decider SHA stable
sha256sum /usr/local/bin/tboot-dnf-posttrans | awk '{print $1}' \
  | grep -Ex "$EXP_DECIDER_SHA" \
  || { echo "FAIL: decider SHA drift"; exit 1; }

# 3. state-dir count = 3
n="$(find /var/lib/tboot-dnf-posttrans -mindepth 1 -maxdepth 1 -printf '.' | wc -c)"
[[ "$n" -eq 3 ]] || { echo "FAIL: state-dir count ${n} (expected 3)"; exit 1; }

# 4. UNSAFE-TO-REBOOT sentinel absent
[[ ! -e /var/lib/tboot-dnf-helper/UNSAFE-TO-REBOOT && ! -L /var/lib/tboot-dnf-helper/UNSAFE-TO-REBOOT ]] \
  || { echo "FAIL: UNSAFE-TO-REBOOT sentinel present"; exit 1; }

# 5. zero active .actions
n_act="$(find /etc/dnf/libdnf5-plugins/actions.d/ -maxdepth 1 -name '*.actions' -type f | wc -l)"
[ "$n_act" -eq 0 ] || { echo "FAIL: active .actions present (count=${n_act})"; exit 1; }

# 6. helper SHA unchanged
sha256sum /usr/local/sbin/tboot-dnf-helper | awk '{print $1}' \
  | grep -Ex "$EXP_HELPER_SHA" \
  || { echo "FAIL: helper SHA drift"; exit 1; }

# 7. hook SHA unchanged
sha256sum /etc/kernel/install.d/80-tpm2-sign.install | awk '{print $1}' \
  | grep -Ex "$EXP_HOOK_SHA" \
  || { echo "FAIL: hook SHA drift"; exit 1; }

# 8. predict SHA unchanged
sha256sum /usr/local/sbin/tboot-predict-pcr11 | awk '{print $1}' \
  | grep -Ex "$EXP_PREDICT_SHA" \
  || { echo "FAIL: predict SHA drift"; exit 1; }

# 9. booted UKI SHA unchanged
sha256sum "$UKI_PATH" | awk '{print $1}' \
  | grep -Ex "$EXP_UKI_SHA" \
  || { echo "FAIL: booted UKI SHA drift"; exit 1; }

# 10. PCR11 prediction file stable
pcr_file="$(tr -d '[:space:]' < /root/tboot-lab/state/expected-pcr11-after-hook-uki.txt | tr 'a-f' 'A-F')"
[[ "$pcr_file" == "$EXP_PCR11" ]] \
  || { echo "FAIL: PCR11 prediction file drift (now=${pcr_file})"; exit 1; }

# 11. runtime PCR11 matches prediction
runtime="$(tr -d '[:space:]' < /sys/class/tpm/tpm0/pcr-sha256/11 | tr 'a-f' 'A-F')"
[[ "$runtime" == "$EXP_PCR11" ]] \
  || { echo "FAIL: runtime PCR11 != prediction (runtime=${runtime})"; exit 1; }

# 12. helper journal silent in last 5 minutes (journalctl -q per 06F I.4)
SINCE="$(date -d '5 minutes ago' '+%Y-%m-%d %H:%M:%S')"
helper_journal="$(journalctl -q -t "$HELPER_TAG" --since "$SINCE" --no-pager 2>/dev/null || true)"
[[ -z "$helper_journal" ]] \
  || { echo "FAIL: helper journal not silent in last 5 min"; printf '%s\n' "$helper_journal"; exit 1; }

# 13. decider lock absent
[[ ! -e /run/tboot-dnf-posttrans.lock && ! -L /run/tboot-dnf-posttrans.lock ]] \
  || { echo "FAIL: decider lock present"; exit 1; }

echo "=== sub-gate 3.8a PASS: VM 500 in Gate 3.7 closure state, safe to snapshot ==="
echo "  baseline SHA:                ${EXP_BASELINE_SHA}"
echo "  decider SHA:                 ${EXP_DECIDER_SHA}"
echo "  state-dir count:             3"
echo "  UNSAFE-TO-REBOOT:            absent"
echo "  active .actions:             0"
echo "  helper SHA:                  ${EXP_HELPER_SHA}"
echo "  hook SHA:                    ${EXP_HOOK_SHA}"
echo "  predict SHA:                 ${EXP_PREDICT_SHA}"
echo "  booted UKI SHA:              ${EXP_UKI_SHA}"
echo "  PCR11 prediction file:       stable"
echo "  runtime PCR11 == prediction: yes"
echo "  helper journal (last 5 min): silent"
echo "  decider lock:                absent"
EOF
```

#### Sub-gate 3.8b: Proxmox-side closure snapshot (simplified)

Run on the Proxmox host `pve-host`. We do not over-engineer Proxmox thin-pool parsing here: verify thin-pool headroom separately with `lvs` if your lab needs it. The `qm listsnapshot` calls before and after use the simple `tail -12` display rather than tree-prefix-aware parsing.

```bash
# [Proxmox host pve-host]
VMID=500
SNAP_NAME="module5-b2-3-decider-installed-primed"

echo "=== Snapshot lineage before ==="
qm listsnapshot "$VMID" | tail -12

qm snapshot "$VMID" "$SNAP_NAME" \
  --description "B.2.3 Gate 3 closed. Decider installed and validated. State-dir exists. Baseline primed sha=75dc4f5651fc8996d39b545ac0bf7244a2a5888403b0ba0cd448a671daf80160 size=19752. debug-print matches baseline zero drift. No active .actions. Helper not invoked. UNSAFE sentinel absent. Gates 4-7 pending. B.2.4 pending."

echo "=== Snapshot lineage after ==="
qm listsnapshot "$VMID" | tail -12

qm status "$VMID"
```

Inspect the `qm listsnapshot` "after" output to confirm `module5-b2-3-decider-installed-primed` appears in the lineage. Direct-parent relationships are visible in the rendered tree but are not asserted by this command pattern; `module5-b2-2-real-run-validated` remains the deeper rollback anchor regardless of intermediate snapshots. The `qm status` output should show the VM still `running` after the snapshot.

✅ **Sub-gate 3.8 PASS: B.2.3 Gate 3 closed.**

### Expected output (Step 32 overall)

- Sub-gate 3.0: `=== sub-gate 3.0 PASS ===` after 12 ok lines (rollback anchor, kernel, UKI sha, bootctl default + current, PCR 11, /run tmpfs, frozen shas, 3 negative pre-conditions, actions.d clean, helper state-dir, libdnf5 NVR, staged file).
- Sub-gate 3.1: `=== sub-gate 3.1 PASS ===` with path, sha, size, mode, owner, bash -n PASS.
- Sub-gate 3.2: `=== sub-gate 3.2 PASS ===` after the UsrMerge resolve, pre-checks, install, atomic publish, both-path verification, SRC unchanged, 7 negative invariants.
- Sub-gate 3.3: `=== sub-gate 3.3 PASS ===` with inode identity, --version, --help, and negative invariants.
- Sub-gate 3.4: `=== sub-gate 3.4 PASS ===` with state-dir path, mode, empty, fs dev.
- Sub-gate 3.5: `=== sub-gate 3.5 PASS ===` with --self-test rc=0, the five expected lines, no err:, journald cross-check, scratch+lock+state-dir unchanged, frozen artifacts stable.
- Sub-gate 3.6: `=== sub-gate 3.6 PASS: baseline primed ===` with the rebuild-specific baseline sha, size, state-dir entry count 3, helper invoked NO, sentinel set NO.
- Sub-gate 3.7: `=== sub-gate 3.7 PASS: post-prime cross-validation ===` with baseline sha stable, --debug-print rc=0, zero-drift sha match, state-dir not mutated, helper not invoked.
- Sub-gate 3.8: `=== sub-gate 3.8 PASS: B.2.3 Gate 3 closed ===` with snapshot name, parent anchor, VM status running.

### Stop condition

- Sub-gate 3.0 FAIL (any of 12 ok checks): VM is not in B.2.2 closure state. Halt; do not proceed. Investigate which invariant moved.
- Sub-gate 3.1 FAIL (staged file missing or sha drift): re-run Step 31 (Gate 2) to re-stage the decider. Do not bypass.
- Sub-gate 3.2 FAIL (UsrMerge resolution unexpected, install failed, sha mismatch post-publish): the trap cleans the temp; investigate the assertion that fired. If `/usr/local/sbin` does not resolve to `/usr/local/bin`, halt: the lab layout has drifted from Fedora 43 defaults.
- Sub-gate 3.3 FAIL (inode mismatch, --version mismatch, --help missing flag): the install completed but the installed artefact does not match the expected version. Halt; investigate.
- Sub-gate 3.4 FAIL (state-dir not created with correct mode, or different fs from parent): halt; do not proceed to --prime.
- Sub-gate 3.5 FAIL (--self-test rc!=0, expected lines missing, scratch/lock leak): the decider has a runtime preflight issue. Halt; do not run --prime.
- Sub-gate 3.6 FAIL (--prime rc!=0, baseline missing, state-dir count != 3, err: in journal, helper journal fired, sentinel appeared): the baseline prime did not complete cleanly. Halt; investigate. If state was partially written, rollback to `module5-b2-2-real-run-validated` and restart from sub-gate 3.0.
- Sub-gate 3.7 FAIL (drift between baseline and --debug-print, state-dir mutated, helper invoked, lock leak): the decider is non-deterministic on steady state OR a runtime mutation surface has moved that should not have. Severe; rollback to `module5-b2-2-real-run-validated`.
- Sub-gate 3.8 FAIL (VM-side sanity check fails, `qm snapshot` fails, snapshot not visible, VM not running post-snapshot): the system has drifted between Gate 3.7 and Gate 3.8, or the Proxmox snapshot operation failed. Halt; do not declare Gate 3 closed. Investigate before any further work.

### Snapshot points within Step 32

`module5-b2-3-decider-installed-primed` is the ONLY snapshot taken in Step 32, at the end of sub-gate 3.8. Intermediate snapshots are NOT taken between sub-gates 3.2, 3.4, 3.6 (despite each being a mutation point) because the Gate 3 workflow is fast enough to re-run cleanly from the deeper rollback anchor `module5-b2-2-real-run-validated` (or from the Gate 1 pre-design-verification anchor `module5-b2-3-pre-design-verification` if present) if any sub-gate fails, and intermediate snapshots would multiply the thin-pool storage cost without operational benefit.

### Step 32 closure (recorded in attempt log, not in this runbook)

The Step 32 closure block, with run timestamps and sub-gate-by-sub-gate output, lives in the Obsidian vault under the B.2.3 attempt-log subtree: not in this runbook. The runbook captures the procedure; the attempt log captures the run.

✅ **B.2.3 Gate 3 closed.** Decider installed at `/usr/local/sbin/tboot-dnf-posttrans`, state-dir created, baseline primed, `--debug-print` proven deterministic vs baseline (zero drift), closure snapshot `module5-b2-3-decider-installed-primed` taken. Continued in Step 33 (Gates 4–7).


## Step 33: Block B.2.3 Gates 4.0–4.3 + 5 + 6A + 6B + 7: production `.actions` rule install, no-drift validation, B.2.3 closure

**Goal:** Wire the decider installed in Step 32 to real DNF transactions by installing the production `.actions` rule (Gate 1 design-locked shape), then validate the no-drift path end-to-end through two real transactions (one install, one remove); take the B.2.3 closure snapshot. The drift path (decider detects drift → invokes helper) is intentionally NOT exercised here: that is B.2.4. After this step every DNF transaction invokes the decider; blind `dnf update`/`dnf upgrade` stays blocked until B.2.4, and `dnf upgrade systemd*` until B.4/B.5.
**Prerequisites:**

- Step 32 closed; closure snapshot `module5-b2-3-decider-installed-primed` taken and visible via `qm listsnapshot 500` on the Proxmox host.
- Decider installed at `/usr/local/sbin/tboot-dnf-posttrans` (resolved real path `/usr/local/bin/tboot-dnf-posttrans` via Fedora 43 UsrMerge symlink), sha `1e75110935a72fc9433c9b4bf5688d26bc6d3bd1e02f4388b5e1f32ff1e5cc05`, mode 0755 root:root, size 35164 bytes, version `1.0.0`.
- State-dir `/var/lib/tboot-dnf-posttrans/` exists with 3 entries; baseline sha `75dc4f5651fc8996d39b545ac0bf7244a2a5888403b0ba0cd448a671daf80160`, size 19752 bytes.
- `last-decision` reflects Gate 3 close: `decision=primed`, `epoch=1779645173`, `detail=Gate 3 baseline prime`, `decider_version=1.0.0`.
- B.2.1 probe rule preserved as disabled artefact: `/etc/dnf/libdnf5-plugins/actions.d/00-tboot-trigger-probe.actions.disabled-after-b21-validation` (regular file, root:root, not active because `libdnf5-plugin-actions` reads only files with the `.actions` suffix).
- Zero active `.actions` files in `/etc/dnf/libdnf5-plugins/actions.d/`.
- Booted UKI sha `b6002e666afdf71e5a083311295ac6a5f3ef3b443ceb4ec8c16c9ddb41077943` at `/boot/efi/EFI/Linux/${MID}-6.19.14-200.fc43.x86_64.efi`; runtime PCR 11 byte-matches stored prediction `A03EB49C9335D721758AE8AA9C34DA87664C5CAF9FAD7C314C68C038B16435DD`.

**Where to run:** All sub-gates run as `root` on VM 500 (`tboot-lab`) inside `tmux`, except Gate 4.0's snapshot-lineage visual confirmation and Gate 7 Phase 2's `qm snapshot` + `qm listsnapshot`, which run on Proxmox host `pve-host`. VM-side commands are copy-paste `bash <<'EOF' ... EOF` blocks.

**Last validated:** 2026-05-25: all seven sub-gates closed in sequence; closure snapshot `module5-b2-3-validated`. See Appendix C and `06F` J.1–J.3.

> [!note] Scripts here are the post-correction canonical form
> The inlined scripts incorporate the J.1/J.2/J.3 corrections from the live run, so a clean rebuild should not hit those harness bugs. The live-run history and the why are in `06C` and `06F` J.1–J.3.

---

### Gate 4.0: read-only preflight + rollback anchor visual confirmation

**Goal.** Verify the VM is in the Step 32 closure state expected, and that the rollback anchor `module5-b2-3-decider-installed-primed` is reachable on the Proxmox host. No mutation.

**Mutation scope.** None. Read-only.

**Commands (VM-side, 18 invariants):**

```bash
bash <<'EOF'
set -u

EXP_DECIDER_SHA="1e75110935a72fc9433c9b4bf5688d26bc6d3bd1e02f4388b5e1f32ff1e5cc05"
EXP_BASELINE_SHA="75dc4f5651fc8996d39b545ac0bf7244a2a5888403b0ba0cd448a671daf80160"
EXP_HOOK_SHA="5857e51d5551e05a7ca384f71529ec2fef07f5a6dbe40df70c6bbe49d720947e"
EXP_HELPER_SHA="937afc7a0d199bf7b7d1a5f5bcccf45c37c0f4f7dd0ad924b2567c4da7125e44"
EXP_PREDICT_SHA="b729ec271dd76cd8b4581f16c86e50fc28b947f2a1de5a7e16c6d5828591e3de"
EXP_UKI_SHA="b6002e666afdf71e5a083311295ac6a5f3ef3b443ceb4ec8c16c9ddb41077943"
EXP_PCR11="A03EB49C9335D721758AE8AA9C34DA87664C5CAF9FAD7C314C68C038B16435DD"
EXP_DECIDER_VERSION="1.0.0"
EXP_GATE3_DECISION_EPOCH="1779645173"   # last-decision epoch at Gate 3 close

DECIDER="/usr/local/sbin/tboot-dnf-posttrans"
STATE_DIR="/var/lib/tboot-dnf-posttrans"
BASELINE="${STATE_DIR}/boot-input-manifest.baseline"
LAST_DECISION="${STATE_DIR}/last-decision"
LAST_MANIFEST="${STATE_DIR}/last-computed-manifest"
LAST_ERROR="${STATE_DIR}/last-error"
SENTINEL="/var/lib/tboot-dnf-helper/UNSAFE-TO-REBOOT"
DECIDER_LOCK="/run/tboot-dnf-posttrans.lock"
HELPER_LOCK="/run/tboot-dnf-helper.lock"
ACTIONS_DIR="/etc/dnf/libdnf5-plugins/actions.d"
DISABLED_PROBE="${ACTIONS_DIR}/00-tboot-trigger-probe.actions.disabled-after-b21-validation"
UKI_PATH="/boot/efi/EFI/Linux/0123456789abcdef0123456789abcdef-6.19.14-200.fc43.x86_64.efi"
PCR11_FILE="/root/tboot-lab/state/expected-pcr11-after-hook-uki.txt"

# 401. running as root
[ "$(id -u)" -eq 0 ] || { echo "FAIL 401: must run as root"; exit 401; }

# 402. decider installed
sha256sum "$DECIDER" | awk '{print $1}' | grep -Exq "$EXP_DECIDER_SHA" \
  || { echo "FAIL 402: decider sha drift"; exit 402; }

# 403. decider --version
"$DECIDER" --version | grep -Fq "$EXP_DECIDER_VERSION" \
  || { echo "FAIL 403: decider --version mismatch"; exit 403; }

# 404. state-dir mode + owner
SD_META="$(stat -c '%a %U %G' "$STATE_DIR")"
[ "$SD_META" = "700 root root" ] \
  || { echo "FAIL 404: state-dir metadata drift (got: $SD_META)"; exit 404; }

# 405. state-dir entry count = 3
SD_N="$(find "$STATE_DIR" -mindepth 1 -maxdepth 1 -printf '.' | wc -c)"
[ "$SD_N" -eq 3 ] \
  || { echo "FAIL 405: state-dir entry count = $SD_N (expected 3)"; exit 405; }

# 406. baseline sha
sha256sum "$BASELINE" | awk '{print $1}' | grep -Exq "$EXP_BASELINE_SHA" \
  || { echo "FAIL 406: baseline sha drift"; exit 406; }

# 407. last-computed-manifest sha = baseline sha
sha256sum "$LAST_MANIFEST" | awk '{print $1}' | grep -Exq "$EXP_BASELINE_SHA" \
  || { echo "FAIL 407: last-computed-manifest sha drift from baseline"; exit 407; }

# 408. last-decision shows Gate 3 close state (decision=primed, exact epoch)
grep -Fxq 'decision=primed' "$LAST_DECISION" \
  || { echo "FAIL 408a: last-decision is not decision=primed"; cat "$LAST_DECISION"; exit 408; }
CUR_DECISION_EPOCH="$(awk -F= '/^epoch=/{print $2; exit}' "$LAST_DECISION")"
[ "$CUR_DECISION_EPOCH" = "$EXP_GATE3_DECISION_EPOCH" ] \
  || { echo "FAIL 408b: last-decision epoch ($CUR_DECISION_EPOCH) != Gate 3 close epoch ($EXP_GATE3_DECISION_EPOCH)"; exit 408; }

# 409. last-error absent
[ ! -e "$LAST_ERROR" ] && [ ! -L "$LAST_ERROR" ] \
  || { echo "FAIL 409: last-error present"; exit 409; }

# 410. sentinel absent
[ ! -e "$SENTINEL" ] && [ ! -L "$SENTINEL" ] \
  || { echo "FAIL 410: UNSAFE-TO-REBOOT sentinel present"; exit 410; }

# 411. decider lock not actively held (J.3 pattern)
if [ -e "$DECIDER_LOCK" ] || [ -L "$DECIDER_LOCK" ]; then
  [ -f "$DECIDER_LOCK" ] && [ ! -L "$DECIDER_LOCK" ] \
    || { echo "FAIL 411: decider lock path is not a regular non-symlink file"; exit 411; }
  ( exec 9<>"$DECIDER_LOCK"; flock -n 9 ) \
    || { echo "FAIL 411: decider lock is actively held"; exit 411; }
fi

# 412. helper lock not actively held
if [ -e "$HELPER_LOCK" ] || [ -L "$HELPER_LOCK" ]; then
  [ -f "$HELPER_LOCK" ] && [ ! -L "$HELPER_LOCK" ] \
    || { echo "FAIL 412: helper lock path is not a regular non-symlink file"; exit 412; }
  ( exec 9<>"$HELPER_LOCK"; flock -n 9 ) \
    || { echo "FAIL 412: helper lock is actively held"; exit 412; }
fi

# 413. zero active *.actions entries (anything ending in .actions, recursively the directory listing)
ACTIVE_N="$(find "$ACTIONS_DIR" -maxdepth 1 -name '*.actions' -printf '.' | wc -c)"
[ "$ACTIVE_N" -eq 0 ] \
  || { echo "FAIL 413: $ACTIVE_N active *.actions entries before Gate 5 (expected 0)"; ls -la "$ACTIONS_DIR"; exit 413; }

# 414. disabled B.2.1 probe present as a regular non-symlink file
[ -f "$DISABLED_PROBE" ] && [ ! -L "$DISABLED_PROBE" ] \
  || { echo "FAIL 414: disabled B.2.1 probe missing at $DISABLED_PROBE"; exit 414; }

# 415. frozen trust-chain shas
sha256sum /etc/kernel/install.d/80-tpm2-sign.install | awk '{print $1}' | grep -Exq "$EXP_HOOK_SHA"    || { echo "FAIL 415a: hook sha drift";    exit 415; }
sha256sum /usr/local/sbin/tboot-dnf-helper           | awk '{print $1}' | grep -Exq "$EXP_HELPER_SHA"  || { echo "FAIL 415b: helper sha drift";  exit 415; }
sha256sum /usr/local/sbin/tboot-predict-pcr11        | awk '{print $1}' | grep -Exq "$EXP_PREDICT_SHA" || { echo "FAIL 415c: predict sha drift"; exit 415; }

# 416. booted UKI sha + runtime PCR 11
sha256sum "$UKI_PATH" | awk '{print $1}' | grep -Exq "$EXP_UKI_SHA" \
  || { echo "FAIL 416a: booted UKI sha drift"; exit 416; }
RT_PCR11="$(awk '{gsub(/^0x/,""); print toupper($0)}' /sys/class/tpm/tpm0/pcr-sha256/11)"
ST_PCR11="$(tr -d '[:space:]' < "$PCR11_FILE" | tr 'a-f' 'A-F')"
{ [ "$RT_PCR11" = "$ST_PCR11" ] && [ "$ST_PCR11" = "$EXP_PCR11" ]; } \
  || { echo "FAIL 416b: PCR11 mismatch (rt=$RT_PCR11 stored=$ST_PCR11)"; exit 416; }

# 417. dnf5-plugin-actions present (Gate 5 needs it to do anything)
rpm -q libdnf5-plugin-actions >/dev/null 2>&1 \
  || { echo "FAIL 417: libdnf5-plugin-actions not installed"; exit 417; }

# 418. helper journal silent in last 5 min
HELPER_HITS="$(journalctl -q -t tboot-dnf-helper --since '-5 min' --no-pager | wc -l)"
[ "$HELPER_HITS" -eq 0 ] \
  || { echo "FAIL 418: tboot-dnf-helper journal not silent ($HELPER_HITS lines)"; exit 418; }

echo "=== Gate 4.0 VM-side preflight: 18/18 PASS ==="
EOF
```

**Commands (Proxmox host, snapshot lineage visual confirmation):**

```
qm listsnapshot 500 | tail -15
```

Visually confirm `module5-b2-3-decider-installed-primed` is present in the snapshot tree with `current` below it. The runbook's canonical pattern (also used in Step 32 Gate 3.0) is simple visual inspection, not awk-parsed equality checks against tree-glyph-prefixed names; see `06F` I.1 for the rationale.

**Expected output.**

```
=== Gate 4.0 VM-side preflight: 18/18 PASS ===
```

Followed on the host by a snapshot tree showing `module5-b2-3-decider-installed-primed` with `current` underneath.

**Closure criteria.** All 18 VM-side invariants PASS; the rollback anchor is visually confirmed on the host. Proceed to Gate 4.1.

**Stop condition.** Any FAIL 4xx halts Step 33. Investigate which invariant moved between Step 32 close and Step 33 start.

---

### Gate 4.1: author and stage the production `.actions` rule

**Goal.** Write the production rule file to `/tmp/50-tboot-posttrans.actions.staged` with the Gate 1 design-locked shape. Verify the staged file's identity (path, size, mode, owner, sha). No install yet.

**Mutation scope.** One new file at `/tmp/50-tboot-posttrans.actions.staged`. Volatile (`/tmp` is tmpfs).

**Rollback / safety.** No system mutation. If the rule body is wrong, simply delete `/tmp/50-tboot-posttrans.actions.staged` and re-run Gate 4.1.

**Commands:**

```bash
bash <<'EOF'
set -u

STAGED="/tmp/50-tboot-posttrans.actions.staged"
EXP_PRODRULE_SHA="dfbffe56fa95f4e0231569bb3a1aabc8952b9f99c9d290e4f8fb1005ef90d798"
EXP_PRODRULE_SIZE=87

# 421. running as root
[ "$(id -u)" -eq 0 ] || { echo "FAIL 421: must run as root"; exit 421; }

# 422. no stale staged file: hard absence + symlink-rejection check before any write.
#      The previous design allowed overwrite on an existing path. That is not symlink-safe
#      (a planted symlink at the staged path would redirect the heredoc write to an
#      attacker-controlled target). The trust boundary requires the staged path to be
#      provably absent before we open it for write.
[ ! -e "$STAGED" ] && [ ! -L "$STAGED" ] \
  || { echo "FAIL 422: staged path already exists or is a symlink"; ls -la "$STAGED" 2>&1; exit 422; }

# 423. write the staged rule via single-quoted heredoc to suppress any shell expansion.
#      The single-quoted tag 'TBOOT_RULE_EOF' is critical: it prevents the shell
#      from interpreting any character inside the heredoc body. The rule must be
#      byte-exact, including the trailing newline. With check 422 in place above,
#      `cat > "$STAGED"` is now creating a brand-new file with no race against a
#      pre-planted symlink target.
cat > "$STAGED" <<'TBOOT_RULE_EOF'
post_transaction:::enabled=host-only raise_error=1:/usr/local/sbin/tboot-dnf-posttrans
TBOOT_RULE_EOF

chmod 0644 "$STAGED"
chown root:root "$STAGED"

# 424. staged file exists as a regular non-symlink file
[ -f "$STAGED" ] && [ ! -L "$STAGED" ] \
  || { echo "FAIL 424: staged file missing or wrong type"; exit 424; }

# 425. size = 87 bytes
ACT_SIZE="$(stat -c '%s' "$STAGED")"
[ "$ACT_SIZE" -eq "$EXP_PRODRULE_SIZE" ] \
  || { echo "FAIL 425: staged size = $ACT_SIZE (expected $EXP_PRODRULE_SIZE)"; xxd "$STAGED"; exit 425; }

# 426. mode = 0644
ACT_MODE="$(stat -c '%a' "$STAGED")"
[ "$ACT_MODE" = "644" ] \
  || { echo "FAIL 426: staged mode = $ACT_MODE (expected 644)"; exit 426; }

# 427. owner = root:root
ACT_OWN="$(stat -c '%U:%G' "$STAGED")"
[ "$ACT_OWN" = "root:root" ] \
  || { echo "FAIL 427: staged owner = $ACT_OWN (expected root:root)"; exit 427; }

# 428. sha256 = design-locked value
ACT_SHA="$(sha256sum "$STAGED" | awk '{print $1}')"
[ "$ACT_SHA" = "$EXP_PRODRULE_SHA" ] \
  || { echo "FAIL 428: staged sha = $ACT_SHA (expected $EXP_PRODRULE_SHA)"; exit 428; }

# 429. exactly 1 line + trailing newline
ACT_LINES="$(wc -l < "$STAGED")"
[ "$ACT_LINES" -eq 1 ] \
  || { echo "FAIL 429: staged file line count = $ACT_LINES (expected 1)"; exit 429; }

echo "=== Gate 4.1 stage: 9/9 PASS ==="
echo "  path:  $STAGED"
echo "  sha:   $ACT_SHA"
echo "  size:  $ACT_SIZE bytes"
echo "  mode:  $ACT_MODE root:root"
echo "  lines: $ACT_LINES"
EOF
```

**Expected output.**

```
=== Gate 4.1 stage: 9/9 PASS ===
  path:  /tmp/50-tboot-posttrans.actions.staged
  sha:   dfbffe56fa95f4e0231569bb3a1aabc8952b9f99c9d290e4f8fb1005ef90d798
  size:  87 bytes
  mode:  644 root:root
  lines: 1
```

**Closure criteria.** Staged file sha byte-matches the Gate 1 design-locked value. Proceed to Gate 4.2.

**Stop condition.** Any FAIL 42x: most likely sha drift caused by smart-quote substitution or a shell that interprets the heredoc differently. Verify the shell is `bash`, the heredoc tag is single-quoted (`'TBOOT_RULE_EOF'`), and the body is the exact ASCII string. Re-run Gate 4.1.

---

### Gate 4.2: staged-rule external read-back validation against Gate 1 design lock

**Goal.** Parse the staged rule into its 5 fields (`callback_name`, `package_filter`, `direction`, `options`, `command`) and assert each against the Gate 1 design lock. Apply the Deviation D regex scan for forbidden tokens (defence in depth: the rule should contain none).

**Mutation scope.** None. Read-only.

**Commands:**

```bash
bash <<'EOF'
set -u

STAGED="/tmp/50-tboot-posttrans.actions.staged"

EXP_CALLBACK="post_transaction"
EXP_PKG_FILTER=""
EXP_DIRECTION=""
EXP_OPTIONS="enabled=host-only raise_error=1"
EXP_COMMAND="/usr/local/sbin/tboot-dnf-posttrans"

# 441. staged file present (idempotent re-check after Gate 4.1)
[ -f "$STAGED" ] && [ ! -L "$STAGED" ] \
  || { echo "FAIL 441: staged file missing"; exit 441; }

# 442. read the single rule line into 5 fields, split by ':' but careful:
#      the design-locked rule has empty package_filter and empty direction, so
#      the colon-delimited structure is: callback:pkg:direction:options:command
#      Use awk with FS=":" and capture exactly 5 fields. Anything else is a shape violation.
ACT_RULE="$(grep -v '^[[:space:]]*#' "$STAGED" | grep -v '^[[:space:]]*$' | head -1)"
ACT_COLS="$(awk -F':' '{print NF}' <<<"$ACT_RULE")"
[ "$ACT_COLS" -eq 5 ] \
  || { echo "FAIL 442: rule field count = $ACT_COLS (expected 5)"; echo "rule: $ACT_RULE"; exit 442; }

# 443. callback_name
ACT_CALLBACK="$(awk -F':' '{print $1}' <<<"$ACT_RULE")"
[ "$ACT_CALLBACK" = "$EXP_CALLBACK" ] \
  || { echo "FAIL 443: callback_name = '$ACT_CALLBACK' (expected '$EXP_CALLBACK')"; exit 443; }

# 444. package_filter (must be empty)
ACT_PKG="$(awk -F':' '{print $2}' <<<"$ACT_RULE")"
[ "$ACT_PKG" = "$EXP_PKG_FILTER" ] \
  || { echo "FAIL 444: package_filter = '$ACT_PKG' (expected empty)"; exit 444; }

# 445. direction (must be empty: fires on both inbound and outbound transactions)
ACT_DIR="$(awk -F':' '{print $3}' <<<"$ACT_RULE")"
[ "$ACT_DIR" = "$EXP_DIRECTION" ] \
  || { echo "FAIL 445: direction = '$ACT_DIR' (expected empty)"; exit 445; }

# 446. options (host-only + raise_error=1)
ACT_OPTS="$(awk -F':' '{print $4}' <<<"$ACT_RULE")"
[ "$ACT_OPTS" = "$EXP_OPTIONS" ] \
  || { echo "FAIL 446: options = '$ACT_OPTS' (expected '$EXP_OPTIONS')"; exit 446; }

# 447. command (decider canonical path)
ACT_CMD="$(awk -F':' '{print $5}' <<<"$ACT_RULE")"
[ "$ACT_CMD" = "$EXP_COMMAND" ] \
  || { echo "FAIL 447: command = '$ACT_CMD' (expected '$EXP_COMMAND')"; exit 447; }

# 448. Deviation D regex scan: forbidden tokens absent (defence in depth)
#      The rule must invoke the decider, not any signing tool directly.
FORBIDDEN="ukify sbsign sbverify systemd-measure systemd-cryptenroll efi-updatevar cryptsetup"
for tok in $FORBIDDEN; do
  if grep -E "(^|[^A-Za-z0-9_-])${tok}($|[^A-Za-z0-9_-])" "$STAGED" >/dev/null; then
    echo "FAIL 448: forbidden token '$tok' present in staged rule"; cat "$STAGED"; exit 448
  fi
done

# 449. no extra non-comment, non-empty lines
EXTRA_N="$(grep -v '^[[:space:]]*#' "$STAGED" | grep -v '^[[:space:]]*$' | wc -l)"
[ "$EXTRA_N" -eq 1 ] \
  || { echo "FAIL 449: extra rule lines beyond the design-locked one ($EXTRA_N total)"; exit 449; }

echo "=== Gate 4.2 read-back: 9/9 PASS ==="
echo "  callback_name  = $ACT_CALLBACK"
echo "  package_filter = (empty)"
echo "  direction      = (empty)"
echo "  options        = $ACT_OPTS"
echo "  command        = $ACT_CMD"
echo "  forbidden tokens absent (Deviation D regex)"
EOF
```

**Expected output.**

```
=== Gate 4.2 read-back: 9/9 PASS ===
  callback_name  = post_transaction
  package_filter = (empty)
  direction      = (empty)
  options        = enabled=host-only raise_error=1
  command        = /usr/local/sbin/tboot-dnf-posttrans
  forbidden tokens absent (Deviation D regex)
```

**Closure criteria.** All 5 fields match Gate 1 design lock; forbidden tokens absent. Proceed to Gate 4.3.

**Stop condition.** Any FAIL 44x. Most likely cause: rule body corruption between Gate 4.1 and Gate 4.2 (rare). Re-run Gate 4.1 to re-stage.

---

### Gate 4.3: pre-install negative preconditions

**Goal.** Confirm `/etc/dnf/libdnf5-plugins/actions.d/` is empty of any active rule, that the disabled B.2.1 probe is preserved as a non-active artefact, and that `actions.d/` and its parent `plugins/` are on the same filesystem (required for the Gate 5 atomic `mv -T`).

**Mutation scope.** None. Read-only.

**Commands:**

```bash
bash <<'EOF'
set -u

ACTIONS_DIR="/etc/dnf/libdnf5-plugins/actions.d"
PLUGINS_DIR="/etc/dnf/libdnf5-plugins"
DISABLED_PROBE="${ACTIONS_DIR}/00-tboot-trigger-probe.actions.disabled-after-b21-validation"
PRODRULE="${ACTIONS_DIR}/50-tboot-posttrans.actions"

# 451. running as root
[ "$(id -u)" -eq 0 ] || { echo "FAIL 451: must run as root"; exit 451; }

# 452. actions.d/ exists, is a regular directory, owned root:root, mode 0755 or 0750
[ -d "$ACTIONS_DIR" ] && [ ! -L "$ACTIONS_DIR" ] \
  || { echo "FAIL 452a: actions.d/ missing or is a symlink"; exit 452; }
AD_OWN="$(stat -c '%U:%G' "$ACTIONS_DIR")"
[ "$AD_OWN" = "root:root" ] \
  || { echo "FAIL 452b: actions.d/ owner = $AD_OWN (expected root:root)"; exit 452; }

# 453. actions.d/ not group/world-writable (per-finding precaution: no shared write surface)
AD_MODE="$(stat -c '%a' "$ACTIONS_DIR")"
[[ "$AD_MODE" =~ ^[0-7][0-5][0-5]$ ]] \
  || { echo "FAIL 453: actions.d/ mode $AD_MODE permits group or world write"; exit 453; }

# 454. zero active *.actions files
ACTIVE_N="$(find "$ACTIONS_DIR" -maxdepth 1 -name '*.actions' -printf '.' | wc -c)"
[ "$ACTIVE_N" -eq 0 ] \
  || { echo "FAIL 454: $ACTIVE_N active *.actions files present"; ls -la "$ACTIONS_DIR"; exit 454; }

# 455. PRODRULE specifically absent (Gate 5 publishes it)
[ ! -e "$PRODRULE" ] && [ ! -L "$PRODRULE" ] \
  || { echo "FAIL 455: production rule already present pre-publish"; exit 455; }

# 456. disabled probe preserved
[ -f "$DISABLED_PROBE" ] && [ ! -L "$DISABLED_PROBE" ] \
  || { echo "FAIL 456: disabled B.2.1 probe missing"; exit 456; }

# 457. same-filesystem assertion: actions.d/ device == plugins/ device
DEV_ACTIONS="$(stat -c '%d' "$ACTIONS_DIR")"
DEV_PLUGINS="$(stat -c '%d' "$PLUGINS_DIR")"
[ "$DEV_ACTIONS" = "$DEV_PLUGINS" ] \
  || { echo "FAIL 457: actions.d/ (dev $DEV_ACTIONS) and plugins/ (dev $DEV_PLUGINS) on different filesystems; mv -T cannot be atomic"; exit 457; }

# 458. libdnf5-plugin-actions config has the plugin enabled
ACTCONF="/etc/dnf/libdnf5-plugins/actions.conf"
[ -f "$ACTCONF" ] || { echo "FAIL 458: actions.conf missing"; exit 458; }
grep -Eq '^[[:space:]]*enabled[[:space:]]*=[[:space:]]*1' "$ACTCONF" \
  || { echo "FAIL 458: actions.conf does not have enabled=1"; cat "$ACTCONF"; exit 458; }

echo "=== Gate 4.3 pre-install preconditions: 8/8 PASS ==="
echo "  actions.d/: mode $AD_MODE root:root, 0 active *.actions"
echo "  disabled probe preserved: $DISABLED_PROBE"
echo "  actions.d/ and plugins/ on same filesystem (dev $DEV_ACTIONS)"
echo "  plugin enabled=1 in $ACTCONF"
EOF
```

**Expected output.**

```
=== Gate 4.3 pre-install preconditions: 8/8 PASS ===
  actions.d/: mode 755 root:root, 0 active *.actions
  disabled probe preserved: /etc/dnf/libdnf5-plugins/actions.d/00-tboot-trigger-probe.actions.disabled-after-b21-validation
  actions.d/ and plugins/ on same filesystem (dev XXXXX)
  plugin enabled=1 in /etc/dnf/libdnf5-plugins/actions.conf
```

**Closure criteria.** All 8 preconditions PASS. Proceed to Gate 5.

**Stop condition.** Any FAIL 45x. The same-filesystem assertion (457) is the most important: if it fails, the atomic publish in Gate 5 will not be atomic, and Step 33 must halt until the filesystem layout is corrected.

---

### Gate 5: atomic publish of the production `.actions` rule

**Goal.** Publish the staged rule from `/tmp/50-tboot-posttrans.actions.staged` to `/etc/dnf/libdnf5-plugins/actions.d/50-tboot-posttrans.actions` atomically: `mktemp` inside `actions.d/`, `install -m 0644 -o root -g root` the staged content into the temp, then `mv -T` the temp to the production path. From this point onward, every DNF transaction invokes the decider.

**Mutation scope.** One new file at `/etc/dnf/libdnf5-plugins/actions.d/50-tboot-posttrans.actions`. The trigger surface goes live.

**Rollback / safety.** A `trap` removes the per-invocation temp file on any failure. The trap's `rm -f` is guarded by a path-prefix check so it can only remove files whose path begins with `${DST_DIR}/.tboot-50-tboot-posttrans.actions.`. The production path is never touched by the trap. If Gate 5 fails after `install -m 0644` but before `mv -T`, the temp is removed and the production path remains absent (system goes back to "0 active `.actions`" state). The rollback anchor for any Gate 5 failure is `module5-b2-3-decider-installed-primed`.

**Commands:**

```bash
bash <<'EOF'
set -u

STAGED="/tmp/50-tboot-posttrans.actions.staged"
DST_DIR="/etc/dnf/libdnf5-plugins/actions.d"
DST="${DST_DIR}/50-tboot-posttrans.actions"
EXP_PRODRULE_SHA="dfbffe56fa95f4e0231569bb3a1aabc8952b9f99c9d290e4f8fb1005ef90d798"
EXP_PRODRULE_SIZE=87

# 501. running as root
[ "$(id -u)" -eq 0 ] || { echo "FAIL 501: must run as root"; exit 501; }

# 502. staged file present + sha matches design lock
[ -f "$STAGED" ] && [ ! -L "$STAGED" ] \
  || { echo "FAIL 502a: staged file missing"; exit 502; }
STAGE_SHA="$(sha256sum "$STAGED" | awk '{print $1}')"
[ "$STAGE_SHA" = "$EXP_PRODRULE_SHA" ] \
  || { echo "FAIL 502b: staged sha drift ($STAGE_SHA)"; exit 502; }

# 503. dst absent pre-publish
[ ! -e "$DST" ] && [ ! -L "$DST" ] \
  || { echo "FAIL 503: dst already present pre-publish"; exit 503; }

# 503b. defence in depth: re-verify zero active *.actions before the mutation window.
#       Gate 4.0 already checked this, but `actions.d/` is a directory that any other
#       process could have written into between Gate 4.0 and Gate 5. The atomic publish
#       must not be the first thing that breaks the "exactly one production rule"
#       invariant if a stray *.actions file appeared in the interim.
N_ACT_PRE="$(find "$DST_DIR" -maxdepth 1 -name '*.actions' -printf '.' 2>/dev/null | wc -c)"
[ "$N_ACT_PRE" -eq 0 ] \
  || { echo "FAIL 503b: active *.actions entries before publish = $N_ACT_PRE (expected 0)"; \
       find "$DST_DIR" -maxdepth 1 -name '*.actions' -ls; exit 503; }

# Capture pre-publish reference (PRE_PUBLISH_TIME for the helper-silence assertion below)
PRE_PUBLISH_TIME="$(date -Iseconds)"

# 504. mktemp inside DST_DIR (same filesystem as DST; trap removes on failure)
TMP="$(mktemp "${DST_DIR}/.tboot-50-tboot-posttrans.actions.XXXXXX")"
trap 'case "$TMP" in "${DST_DIR}/.tboot-50-tboot-posttrans.actions."*) rm -f -- "$TMP" ;; esac' EXIT
[ -f "$TMP" ] && [ ! -L "$TMP" ] \
  || { echo "FAIL 504: mktemp did not produce a regular file"; exit 504; }

# 505. install staged content into TMP with mode 0644 root:root
install -m 0644 -o root -g root "$STAGED" "$TMP" \
  || { echo "FAIL 505: install rc=$?"; exit 505; }

# 506. verify TMP sha matches design lock before publish
TMP_SHA="$(sha256sum "$TMP" | awk '{print $1}')"
[ "$TMP_SHA" = "$EXP_PRODRULE_SHA" ] \
  || { echo "FAIL 506: TMP sha drift ($TMP_SHA)"; exit 506; }

# 507. atomic rename: mv -T overwrites DST atomically (or fails atomically)
mv -T -- "$TMP" "$DST" \
  || { echo "FAIL 507: mv -T rc=$?"; exit 507; }

# trap can be disarmed now: TMP was consumed by mv -T
trap - EXIT

# 508. post-publish: dst sha + size + mode + owner
[ -f "$DST" ] && [ ! -L "$DST" ] \
  || { echo "FAIL 508a: dst missing post-publish"; exit 508; }
DST_SHA="$(sha256sum "$DST" | awk '{print $1}')"
[ "$DST_SHA" = "$EXP_PRODRULE_SHA" ] \
  || { echo "FAIL 508b: dst sha drift ($DST_SHA)"; exit 508; }
DST_SIZE="$(stat -c '%s' "$DST")"
[ "$DST_SIZE" -eq "$EXP_PRODRULE_SIZE" ] \
  || { echo "FAIL 508c: dst size = $DST_SIZE (expected $EXP_PRODRULE_SIZE)"; exit 508; }
DST_META="$(stat -c '%a %U %G' "$DST")"
[ "$DST_META" = "644 root root" ] \
  || { echo "FAIL 508d: dst metadata drift ($DST_META)"; exit 508; }

# 509. exactly 1 *.actions entry, and it is the production rule
mapfile -t ACTIONS_POST < <(find "$DST_DIR" -maxdepth 1 -name '*.actions' -printf '%p\n' 2>/dev/null | sort)
{ [ "${#ACTIONS_POST[@]}" -eq 1 ] && [ "${ACTIONS_POST[0]}" = "$DST" ]; } \
  || { echo "FAIL 509: unexpected *.actions entries: ${ACTIONS_POST[*]}"; exit 509; }

# 510. helper journal silent since PRE_PUBLISH_TIME (the install must not have invoked the helper)
HELPER_HITS="$(journalctl -q -t tboot-dnf-helper --since "$PRE_PUBLISH_TIME" --no-pager | wc -l)"
[ "$HELPER_HITS" -eq 0 ] \
  || { echo "FAIL 510: helper journal not silent ($HELPER_HITS lines since $PRE_PUBLISH_TIME)"; exit 510; }

echo "=== Gate 5 atomic publish: 11/11 PASS ==="
echo "  pre-publish time: $PRE_PUBLISH_TIME"
echo "  dst:   $DST"
echo "  sha:   $DST_SHA"
echo "  size:  $DST_SIZE bytes"
echo "  meta:  $DST_META"
echo "  active *.actions count: 1 (production rule only)"
echo "  helper journal silent since publish"
echo ""
echo "*** From this point onward, every DNF transaction invokes the decider. ***"
echo "*** Blind dnf update remains blocked (B.2.4 not started). ***"
EOF
```

**Expected output.**

```
=== Gate 5 atomic publish: 11/11 PASS ===
  pre-publish time: 2026-05-XXTXX:XX:XX+02:00
  dst:   /etc/dnf/libdnf5-plugins/actions.d/50-tboot-posttrans.actions
  sha:   dfbffe56fa95f4e0231569bb3a1aabc8952b9f99c9d290e4f8fb1005ef90d798
  size:  87 bytes
  meta:  644 root root
  active *.actions count: 1 (production rule only)
  helper journal silent since publish
```

**Closure criteria.** Production rule published, sha matches design lock, exactly 1 active `*.actions` entry, helper not invoked by the install operation itself. Proceed to Gate 6A.

**Stop condition.** Any FAIL 5xx. If the trap fires on FAIL 506 or 507, the dst is still absent: system is at "0 active `.actions`" state (same as pre-Gate-5). Retry by re-running Gate 5. If FAIL 508+ (post-rename), the dst exists but is wrong; do not run any DNF command until the production rule is repaired or removed.

**Snapshot point.** None. Gate 6A and Gate 6B run against the rollback anchor `module5-b2-3-decider-installed-primed`; the first new snapshot is taken at Gate 7 close.

---

### Gate 6A: `dnf install -y figlet` no-drift validation

**Goal.** Run one real inbound DNF transaction that adds a package with no boot-relevant content. The decider must fire exactly once, compute the manifest, compare against baseline, find zero drift, exit 0 with `decision=unchanged`, and write a fresh `last-decision`. The helper must not be invoked; the sentinel must not be written; the baseline must not be updated; the 6 boot-artefact shas plus runtime PCR 11 must remain byte-stable.

**Mutation scope.** One RPM transaction: `figlet 2.2.5-38.fc43.x86_64` installed. Plus the decider's state-dir writes: `last-computed-manifest` (sha unchanged = baseline) and `last-decision` (new epoch/date/compute_duration_s). Decider lock acquired and released (file persists; `flock` released: see `06F` J.3). Decider journal entries under `tboot-dnf-posttrans`. DNF log entries.

Not mutated: baseline (no helper success on no-drift); `last-error` (no error); sentinel; helper lock; helper state-dir; `/etc/kernel/*`; production rule; booted UKI; initramfs; PCR 11; Secure Boot; LUKS.

**Rollback / safety.** Rollback anchor is `module5-b2-3-decider-installed-primed`. If Gate 6A fails after the transaction commits, do not run `dnf remove figlet` to clean up; capture state and rollback. Cleanup is Gate 6B's job under symmetric controlled invariant capture.

**Commands:**

```bash
bash <<'EOF'
set -u

# Frozen constants
EXP_DECIDER_SHA="1e75110935a72fc9433c9b4bf5688d26bc6d3bd1e02f4388b5e1f32ff1e5cc05"
EXP_BASELINE_SHA="75dc4f5651fc8996d39b545ac0bf7244a2a5888403b0ba0cd448a671daf80160"
EXP_PRODRULE_SHA="dfbffe56fa95f4e0231569bb3a1aabc8952b9f99c9d290e4f8fb1005ef90d798"
EXP_HOOK_SHA="5857e51d5551e05a7ca384f71529ec2fef07f5a6dbe40df70c6bbe49d720947e"
EXP_HELPER_SHA="937afc7a0d199bf7b7d1a5f5bcccf45c37c0f4f7dd0ad924b2567c4da7125e44"
EXP_PREDICT_SHA="b729ec271dd76cd8b4581f16c86e50fc28b947f2a1de5a7e16c6d5828591e3de"
EXP_UKI_SHA="b6002e666afdf71e5a083311295ac6a5f3ef3b443ceb4ec8c16c9ddb41077943"
EXP_PCR11="A03EB49C9335D721758AE8AA9C34DA87664C5CAF9FAD7C314C68C038B16435DD"
EXP_DECIDER_VERSION="1.0.0"

UKI_PATH="/boot/efi/EFI/Linux/0123456789abcdef0123456789abcdef-6.19.14-200.fc43.x86_64.efi"
INITRAMFS_PATH="/boot/initramfs-6.19.14-200.fc43.x86_64.img"
PCR11_FILE="/root/tboot-lab/state/expected-pcr11-after-hook-uki.txt"
PRODRULE="/etc/dnf/libdnf5-plugins/actions.d/50-tboot-posttrans.actions"
ACTIONS_DIR="/etc/dnf/libdnf5-plugins/actions.d"
STATE_DIR="/var/lib/tboot-dnf-posttrans"
BASELINE="${STATE_DIR}/boot-input-manifest.baseline"
LAST_DECISION="${STATE_DIR}/last-decision"
LAST_MANIFEST="${STATE_DIR}/last-computed-manifest"
LAST_ERROR="${STATE_DIR}/last-error"
SENTINEL="/var/lib/tboot-dnf-helper/UNSAFE-TO-REBOOT"
DECIDER_LOCK="/run/tboot-dnf-posttrans.lock"
HELPER_LOCK="/run/tboot-dnf-helper.lock"

# ===== Phase 0: precondition =====

[ "$(id -u)" -eq 0 ] || { echo "FAIL 601: must run as root"; exit 601; }
rpm -q figlet >/dev/null 2>&1 \
  && { echo "FAIL 602: figlet already installed; Gate 6A precondition violated"; exit 602; } \
  || true

# ===== Phase 1: pre-transaction reference capture =====

sha256sum "$PRODRULE" | awk '{print $1}' | grep -Exq "$EXP_PRODRULE_SHA" \
  || { echo "FAIL 611: production rule sha drift"; exit 611; }
[ -f "$PRODRULE" ] && [ ! -L "$PRODRULE" ] \
  || { echo "FAIL 611b: production rule not a regular non-symlink file"; exit 611; }
PR_META="$(stat -c '%a %U %G %s' "$PRODRULE")"
[ "$PR_META" = "644 root root 87" ] \
  || { echo "FAIL 611c: production rule metadata drift ($PR_META)"; exit 611; }

mapfile -t ACTIONS_PRE < <(find "$ACTIONS_DIR" -maxdepth 1 -name '*.actions' -printf '%p\n' | sort)
{ [ "${#ACTIONS_PRE[@]}" -eq 1 ] && [ "${ACTIONS_PRE[0]}" = "$PRODRULE" ]; } \
  || { echo "FAIL 612: unexpected *.actions entries: ${ACTIONS_PRE[*]}"; exit 612; }

sha256sum /usr/local/bin/tboot-dnf-posttrans | awk '{print $1}' | grep -Exq "$EXP_DECIDER_SHA" \
  || { echo "FAIL 613: decider sha drift"; exit 613; }
sha256sum "$BASELINE" | awk '{print $1}' | grep -Exq "$EXP_BASELINE_SHA" \
  || { echo "FAIL 614: baseline sha drift"; exit 614; }
sha256sum "$LAST_MANIFEST" | awk '{print $1}' | grep -Exq "$EXP_BASELINE_SHA" \
  || { echo "FAIL 615: last-computed-manifest sha drift from baseline"; exit 615; }
[ ! -e "$LAST_ERROR" ] || { echo "FAIL 616: last-error present"; exit 616; }
SD_N="$(find "$STATE_DIR" -mindepth 1 -maxdepth 1 -printf '.' | wc -c)"
[ "$SD_N" -eq 3 ] || { echo "FAIL 617: state-dir entry count = $SD_N (expected 3)"; exit 617; }
[ ! -e "$SENTINEL" ] || { echo "FAIL 618: sentinel present pre-transaction"; exit 618; }

# J.3 lock checks pre-transaction
if [ -e "$DECIDER_LOCK" ] || [ -L "$DECIDER_LOCK" ]; then
  [ -f "$DECIDER_LOCK" ] && [ ! -L "$DECIDER_LOCK" ] \
    || { echo "FAIL 619a: decider lock not regular non-symlink"; exit 619; }
  ( exec 9<>"$DECIDER_LOCK"; flock -n 9 ) \
    || { echo "FAIL 619a: decider lock actively held pre-tx"; exit 619; }
fi
if [ -e "$HELPER_LOCK" ] || [ -L "$HELPER_LOCK" ]; then
  [ -f "$HELPER_LOCK" ] && [ ! -L "$HELPER_LOCK" ] \
    || { echo "FAIL 619b: helper lock not regular non-symlink"; exit 619; }
  ( exec 9<>"$HELPER_LOCK"; flock -n 9 ) \
    || { echo "FAIL 619b: helper lock actively held pre-tx"; exit 619; }
fi

sha256sum /etc/kernel/install.d/80-tpm2-sign.install | awk '{print $1}' | grep -Exq "$EXP_HOOK_SHA"    || { echo "FAIL 620a: hook sha drift";    exit 620; }
sha256sum /usr/local/sbin/tboot-dnf-helper           | awk '{print $1}' | grep -Exq "$EXP_HELPER_SHA"  || { echo "FAIL 620b: helper sha drift";  exit 620; }
sha256sum /usr/local/sbin/tboot-predict-pcr11        | awk '{print $1}' | grep -Exq "$EXP_PREDICT_SHA" || { echo "FAIL 620c: predict sha drift"; exit 620; }

sha256sum "$UKI_PATH" | awk '{print $1}' | grep -Exq "$EXP_UKI_SHA" \
  || { echo "FAIL 621a: booted UKI sha drift"; exit 621; }
RT_PCR11="$(awk '{gsub(/^0x/,""); print toupper($0)}' /sys/class/tpm/tpm0/pcr-sha256/11)"
[ "$RT_PCR11" = "$EXP_PCR11" ] || { echo "FAIL 621b: runtime PCR 11 mismatch ($RT_PCR11)"; exit 621; }

PRE_INITRAMFS_MTIME="$(stat -c '%Y' "$INITRAMFS_PATH")"
PRE_UKI_MTIME="$(stat -c '%Y' "$UKI_PATH")"
PRE_PRODRULE_INO="$(stat -c '%i' "$PRODRULE")"

echo "=== Gate 6A Phase 1: pre-transaction reference capture 12/12 PASS ==="

PRE_TIME_6A="$(date -Iseconds)"
PRE_TIME_6A_EPOCH="$(date -d "$PRE_TIME_6A" +%s)"

# ===== Phase 2: single mutation: dnf install -y figlet =====

DNF_RC=0
dnf install -y figlet || DNF_RC=$?

# ===== Phase 3: post-transaction invariant assertion =====

[ "$DNF_RC" -eq 0 ] \
  || { echo "FAIL 631: dnf install -y figlet rc=$DNF_RC"; exit 631; }
rpm -q figlet >/dev/null 2>&1 \
  || { echo "FAIL 632: figlet not installed after dnf install"; exit 632; }

# Authoritative evidence: journald, NOT /var/log/dnf5.log (J.1)
# Comparison: ISO-8601 via journalctl --since (J.2)
EXIT_HITS="$(journalctl -q -t tboot-dnf-posttrans --since "$PRE_TIME_6A" --no-pager \
              | grep -c 'exit reason: manifest unchanged; baseline NOT updated; helper not invoked')"
[ "$EXIT_HITS" -eq 1 ] \
  || { echo "FAIL 633: decider exit-reason lines = $EXIT_HITS (expected 1)"; \
       journalctl -q -t tboot-dnf-posttrans --since "$PRE_TIME_6A" --no-pager; exit 633; }

journalctl -q -t tboot-dnf-posttrans --since "$PRE_TIME_6A" --no-pager \
  | grep -Fq 'helper invocation decision: SKIP (manifest unchanged)' \
  || { echo "FAIL 634: decider SKIP-decision line absent"; exit 634; }

ERR_HITS="$(journalctl -q -t tboot-dnf-posttrans --since "$PRE_TIME_6A" --no-pager | grep -cE '^.* err: ')"
[ "$ERR_HITS" -eq 0 ] || { echo "FAIL 635: decider emitted $ERR_HITS err: lines"; exit 635; }

grep -Fxq 'decision=unchanged' "$LAST_DECISION" \
  || { echo "FAIL 636: last-decision is not decision=unchanged"; cat "$LAST_DECISION"; exit 636; }
grep -E '^detail=.*helper not invoked'   "$LAST_DECISION" >/dev/null || { echo "FAIL 637: last-decision detail missing 'helper not invoked'"; exit 637; }
grep -E '^detail=.*baseline NOT updated' "$LAST_DECISION" >/dev/null || { echo "FAIL 638: last-decision detail missing 'baseline NOT updated'"; exit 638; }

NEW_DECISION_EPOCH="$(awk -F= '/^epoch=/{print $2; exit}' "$LAST_DECISION")"
{ [ -n "$NEW_DECISION_EPOCH" ] && [ "$NEW_DECISION_EPOCH" -ge "$PRE_TIME_6A_EPOCH" ]; } \
  || { echo "FAIL 639: last-decision epoch ($NEW_DECISION_EPOCH) earlier than PRE_TIME_6A epoch ($PRE_TIME_6A_EPOCH)"; cat "$LAST_DECISION"; exit 639; }

grep -Fxq "decider_version=${EXP_DECIDER_VERSION}" "$LAST_DECISION" \
  || { echo "FAIL 640: last-decision decider_version != $EXP_DECIDER_VERSION"; exit 640; }

sha256sum "$BASELINE"      | awk '{print $1}' | grep -Exq "$EXP_BASELINE_SHA" || { echo "FAIL 641: baseline sha drifted"; exit 641; }
sha256sum "$LAST_MANIFEST" | awk '{print $1}' | grep -Exq "$EXP_BASELINE_SHA" || { echo "FAIL 642: last-computed-manifest sha drifted from baseline"; exit 642; }
[ ! -e "$LAST_ERROR" ] || { echo "FAIL 643: last-error appeared"; exit 643; }
SD_N_POST="$(find "$STATE_DIR" -mindepth 1 -maxdepth 1 -printf '.' | wc -c)"
[ "$SD_N_POST" -eq 3 ] || { echo "FAIL 644: state-dir entry count = $SD_N_POST"; exit 644; }

# Post-transaction sentinel + lock checks (J.3 pattern)
[ ! -e "$SENTINEL" ] || { echo "FAIL 645a: sentinel appeared post-tx"; exit 645; }
if [ -e "$DECIDER_LOCK" ] || [ -L "$DECIDER_LOCK" ]; then
  [ -f "$DECIDER_LOCK" ] && [ ! -L "$DECIDER_LOCK" ] \
    || { echo "FAIL 645b: decider lock not regular non-symlink"; exit 645; }
  ( exec 9<>"$DECIDER_LOCK"; flock -n 9 ) \
    || { echo "FAIL 645b: decider lock actively held post-tx"; exit 645; }
fi
if [ -e "$HELPER_LOCK" ] || [ -L "$HELPER_LOCK" ]; then
  [ -f "$HELPER_LOCK" ] && [ ! -L "$HELPER_LOCK" ] \
    || { echo "FAIL 645c: helper lock not regular non-symlink"; exit 645; }
  ( exec 9<>"$HELPER_LOCK"; flock -n 9 ) \
    || { echo "FAIL 645c: helper lock actively held post-tx"; exit 645; }
fi

# Helper journal silent
HELPER_HITS="$(journalctl -q -t tboot-dnf-helper --since "$PRE_TIME_6A" --no-pager | wc -l)"
[ "$HELPER_HITS" -eq 0 ] || { echo "FAIL 646: helper journal not silent ($HELPER_HITS lines)"; exit 646; }

# 6 boot-artefact content invariants + mtimes forensic
sha256sum /etc/kernel/install.d/80-tpm2-sign.install | awk '{print $1}' | grep -Exq "$EXP_HOOK_SHA"    || { echo "FAIL 647a: hook sha drift";    exit 647; }
sha256sum /usr/local/sbin/tboot-dnf-helper           | awk '{print $1}' | grep -Exq "$EXP_HELPER_SHA"  || { echo "FAIL 647b: helper sha drift";  exit 647; }
sha256sum /usr/local/sbin/tboot-predict-pcr11        | awk '{print $1}' | grep -Exq "$EXP_PREDICT_SHA" || { echo "FAIL 647c: predict sha drift"; exit 647; }
sha256sum /usr/local/bin/tboot-dnf-posttrans         | awk '{print $1}' | grep -Exq "$EXP_DECIDER_SHA" || { echo "FAIL 647d: decider sha drift"; exit 647; }
sha256sum "$UKI_PATH"                                | awk '{print $1}' | grep -Exq "$EXP_UKI_SHA"     || { echo "FAIL 647e: booted UKI sha drift"; exit 647; }
RT_PCR11_POST="$(awk '{gsub(/^0x/,""); print toupper($0)}' /sys/class/tpm/tpm0/pcr-sha256/11)"
[ "$RT_PCR11_POST" = "$EXP_PCR11" ] || { echo "FAIL 647f: runtime PCR11 drift (got $RT_PCR11_POST)"; exit 647; }
POST_INITRAMFS_MTIME="$(stat -c '%Y' "$INITRAMFS_PATH")"
POST_UKI_MTIME="$(stat -c '%Y' "$UKI_PATH")"
[ "$POST_INITRAMFS_MTIME" -eq "$PRE_INITRAMFS_MTIME" ] || { echo "FAIL 647g: initramfs mtime advanced"; exit 647; }
[ "$POST_UKI_MTIME"       -eq "$PRE_UKI_MTIME" ]       || { echo "FAIL 647h: ESP UKI mtime advanced"; exit 647; }

# Production rule unchanged
mapfile -t ACTIONS_POST < <(find "$ACTIONS_DIR" -maxdepth 1 -name '*.actions' -printf '%p\n' | sort)
{ [ "${#ACTIONS_POST[@]}" -eq 1 ] && [ "${ACTIONS_POST[0]}" = "$PRODRULE" ]; } \
  || { echo "FAIL 648: unexpected *.actions entries post-tx"; exit 648; }
sha256sum "$PRODRULE" | awk '{print $1}' | grep -Exq "$EXP_PRODRULE_SHA" \
  || { echo "FAIL 649a: production rule sha drift post-tx"; exit 649; }
POST_PRODRULE_INO="$(stat -c '%i' "$PRODRULE")"
[ "$POST_PRODRULE_INO" = "$PRE_PRODRULE_INO" ] \
  || { echo "FAIL 649b: production rule inode changed ($PRE_PRODRULE_INO -> $POST_PRODRULE_INO)"; exit 649; }

echo "=== Gate 6A Phase 3 post-transaction: 19/19 PASS ==="
echo ""
echo "=== Gate 6A OVERALL: PASS ==="
echo "  transaction       : dnf install -y figlet (rc=0)"
echo "  PRE_TIME_6A       : $PRE_TIME_6A (epoch $PRE_TIME_6A_EPOCH)"
echo "  decider firings   : 1 ('exit reason: manifest unchanged...')"
echo "  decision          : unchanged"
echo "  last-decision     : epoch=$NEW_DECISION_EPOCH"
echo "  baseline          : sha unchanged ($EXP_BASELINE_SHA)"
echo "  helper            : journal silent, lock not actively held, sentinel never written"
echo "  boot artefacts    : 6/6 sha-byte-stable + initramfs/UKI mtime unchanged"
echo ""
echo "Gate 6A closed. figlet is INSTALLED. Proceed to Gate 6B (remove)."
EOF
```

**Expected output.**

```
=== Gate 6A Phase 1: pre-transaction reference capture 12/12 PASS ===
[... dnf install -y figlet output ...]
[... decider info-line stream: preflight, lock acquire, recompute, baseline compare result: unchanged, SKIP, exit reason: manifest unchanged; baseline NOT updated; helper not invoked ...]
=== Gate 6A Phase 3 post-transaction: 19/19 PASS ===

=== Gate 6A OVERALL: PASS ===
  transaction       : dnf install -y figlet (rc=0)
  ...
```

**Closure criteria.** dnf rc=0 + figlet installed + decider fired exactly once + decision=unchanged + helper not invoked + baseline not updated + boot artefacts byte-stable. Proceed to Gate 6B (cleanup with symmetric controlled-invariant capture).

**Stop condition.** Any FAIL 6xx halts Step 33; capture state and rollback to `module5-b2-3-decider-installed-primed`. Do not run `dnf remove figlet` to "clean up"; Gate 6B is the controlled cleanup.

**Live-run note (2026-05-25).** The original Gate 6A verifier ran with a `/var/log/dnf5.log` timestamp-bound grep (check 183 in the original numbering) and a "decider lock path absent" invariant (check 197). Both FAILed despite the transaction itself succeeding (figlet installed, decider clearly fired per its journald stream). Diagnostic showed:

- `/var/log/dnf5.log` contained zero `Loaded libdnf plugin` entries: recorded as `06F` J.1.
- `date -Iseconds` and `dnf5.log` use incompatible TZ-offset formats (`+02:00` vs `+0200`): recorded as `06F` J.2.
- `/run/tboot-dnf-posttrans.lock` persisted as a 0-byte root:root 0644 file after the decider exited; `fuser -v` confirmed no current holder; `flock -n` confirmed not actively held: recorded as `06F` J.3.

The script above is the post-correction canonical form, free of all three pitfalls. The original FAIL trace and the two-step correction sequence are preserved in the Obsidian B.2.3 attempt-log subtree.

---

### Gate 6B: `dnf remove -y figlet` no-drift validation (symmetric)

**Goal.** Symmetric to Gate 6A. One real outbound DNF transaction removes the package installed in Gate 6A. The decider must again fire exactly once on the same no-drift path. This validates that the production rule's empty `direction:` field fires on outbound transactions as the man page documents.

**Mutation scope.** One RPM transaction: `figlet 2.2.5-38.fc43.x86_64` removed. Plus the decider's state-dir writes: `last-computed-manifest` (still = baseline) and `last-decision` (new epoch advanced past Gate 6A's epoch).

Not mutated: baseline, last-error, sentinel, helper lock, helper state-dir, `/etc/kernel/*`, production rule, booted UKI, initramfs, PCR 11, Secure Boot, LUKS.

**Rollback / safety.** Rollback anchor: `module5-b2-3-decider-installed-primed`. If Gate 6B fails, both 6A and 6B effects are wiped on rollback (neither was snapshotted).

**Commands.** Same shape as Gate 6A with two semantic differences: precondition asserts figlet IS installed (Gate 6A close state); the new `last-decision` epoch must strictly exceed the captured Gate 6A epoch (not just exceed PRE_TIME_6B's epoch, which is the weaker check inherited from Gate 6A's pattern).

```bash
bash <<'EOF'
set -u

EXP_DECIDER_SHA="1e75110935a72fc9433c9b4bf5688d26bc6d3bd1e02f4388b5e1f32ff1e5cc05"
EXP_BASELINE_SHA="75dc4f5651fc8996d39b545ac0bf7244a2a5888403b0ba0cd448a671daf80160"
EXP_PRODRULE_SHA="dfbffe56fa95f4e0231569bb3a1aabc8952b9f99c9d290e4f8fb1005ef90d798"
EXP_HOOK_SHA="5857e51d5551e05a7ca384f71529ec2fef07f5a6dbe40df70c6bbe49d720947e"
EXP_HELPER_SHA="937afc7a0d199bf7b7d1a5f5bcccf45c37c0f4f7dd0ad924b2567c4da7125e44"
EXP_PREDICT_SHA="b729ec271dd76cd8b4581f16c86e50fc28b947f2a1de5a7e16c6d5828591e3de"
EXP_UKI_SHA="b6002e666afdf71e5a083311295ac6a5f3ef3b443ceb4ec8c16c9ddb41077943"
EXP_PCR11="A03EB49C9335D721758AE8AA9C34DA87664C5CAF9FAD7C314C68C038B16435DD"
EXP_DECIDER_VERSION="1.0.0"

# Read Gate 6A's last-decision epoch from disk (captured at Gate 6A close).
# In a fresh rebuild, this will be the epoch from the immediately preceding Gate 6A run.
EXP_6A_DECISION_EPOCH="$(awk -F= '/^epoch=/{print $2; exit}' /var/lib/tboot-dnf-posttrans/last-decision)"

UKI_PATH="/boot/efi/EFI/Linux/0123456789abcdef0123456789abcdef-6.19.14-200.fc43.x86_64.efi"
INITRAMFS_PATH="/boot/initramfs-6.19.14-200.fc43.x86_64.img"
PCR11_FILE="/root/tboot-lab/state/expected-pcr11-after-hook-uki.txt"
PRODRULE="/etc/dnf/libdnf5-plugins/actions.d/50-tboot-posttrans.actions"
ACTIONS_DIR="/etc/dnf/libdnf5-plugins/actions.d"
STATE_DIR="/var/lib/tboot-dnf-posttrans"
BASELINE="${STATE_DIR}/boot-input-manifest.baseline"
LAST_DECISION="${STATE_DIR}/last-decision"
LAST_MANIFEST="${STATE_DIR}/last-computed-manifest"
LAST_ERROR="${STATE_DIR}/last-error"
SENTINEL="/var/lib/tboot-dnf-helper/UNSAFE-TO-REBOOT"
DECIDER_LOCK="/run/tboot-dnf-posttrans.lock"
HELPER_LOCK="/run/tboot-dnf-helper.lock"

# Phase 0: precondition (Gate 6A close state)
[ "$(id -u)" -eq 0 ] || { echo "FAIL 651: must run as root"; exit 651; }
rpm -q figlet >/dev/null 2>&1 \
  || { echo "FAIL 652: figlet not installed (Gate 6A must close before Gate 6B)"; exit 652; }
echo "=== Gate 6B Phase 0: precondition 2/2 PASS ==="

# Phase 1: pre-remove reference capture (Gate 6A close state verification)
sha256sum "$PRODRULE" | awk '{print $1}' | grep -Exq "$EXP_PRODRULE_SHA" || { echo "FAIL 661: production rule sha drift"; exit 661; }
[ -f "$PRODRULE" ] && [ ! -L "$PRODRULE" ] || { echo "FAIL 661b: production rule not regular non-symlink"; exit 661; }
PR_META="$(stat -c '%a %U %G %s' "$PRODRULE")"
[ "$PR_META" = "644 root root 87" ] || { echo "FAIL 661c: production rule metadata drift ($PR_META)"; exit 661; }

mapfile -t ACTIONS_PRE < <(find "$ACTIONS_DIR" -maxdepth 1 -name '*.actions' -printf '%p\n' | sort)
{ [ "${#ACTIONS_PRE[@]}" -eq 1 ] && [ "${ACTIONS_PRE[0]}" = "$PRODRULE" ]; } || { echo "FAIL 662: unexpected *.actions entries"; exit 662; }

sha256sum /usr/local/bin/tboot-dnf-posttrans | awk '{print $1}' | grep -Exq "$EXP_DECIDER_SHA" || { echo "FAIL 663: decider sha drift"; exit 663; }
sha256sum "$BASELINE"       | awk '{print $1}' | grep -Exq "$EXP_BASELINE_SHA" || { echo "FAIL 664: baseline sha drift"; exit 664; }
sha256sum "$LAST_MANIFEST"  | awk '{print $1}' | grep -Exq "$EXP_BASELINE_SHA" || { echo "FAIL 665: last-computed-manifest sha drift from baseline"; exit 665; }

grep -Fxq 'decision=unchanged' "$LAST_DECISION" \
  || { echo "FAIL 666a: last-decision not decision=unchanged (Gate 6A close state)"; cat "$LAST_DECISION"; exit 666; }

[ ! -e "$LAST_ERROR" ] || { echo "FAIL 667: last-error present"; exit 667; }
SD_N="$(find "$STATE_DIR" -mindepth 1 -maxdepth 1 -printf '.' | wc -c)"
[ "$SD_N" -eq 3 ] || { echo "FAIL 668: state-dir entry count = $SD_N"; exit 668; }
[ ! -e "$SENTINEL" ] || { echo "FAIL 669a: sentinel present"; exit 669; }

if [ -e "$DECIDER_LOCK" ] || [ -L "$DECIDER_LOCK" ]; then
  [ -f "$DECIDER_LOCK" ] && [ ! -L "$DECIDER_LOCK" ] || { echo "FAIL 669b: decider lock not regular non-symlink"; exit 669; }
  ( exec 9<>"$DECIDER_LOCK"; flock -n 9 ) || { echo "FAIL 669b: decider lock actively held pre-tx"; exit 669; }
fi
if [ -e "$HELPER_LOCK" ] || [ -L "$HELPER_LOCK" ]; then
  [ -f "$HELPER_LOCK" ] && [ ! -L "$HELPER_LOCK" ] || { echo "FAIL 669c: helper lock not regular non-symlink"; exit 669; }
  ( exec 9<>"$HELPER_LOCK"; flock -n 9 ) || { echo "FAIL 669c: helper lock actively held pre-tx"; exit 669; }
fi

sha256sum /etc/kernel/install.d/80-tpm2-sign.install | awk '{print $1}' | grep -Exq "$EXP_HOOK_SHA"    || { echo "FAIL 670a: hook sha drift";    exit 670; }
sha256sum /usr/local/sbin/tboot-dnf-helper           | awk '{print $1}' | grep -Exq "$EXP_HELPER_SHA"  || { echo "FAIL 670b: helper sha drift";  exit 670; }
sha256sum /usr/local/sbin/tboot-predict-pcr11        | awk '{print $1}' | grep -Exq "$EXP_PREDICT_SHA" || { echo "FAIL 670c: predict sha drift"; exit 670; }
sha256sum "$UKI_PATH"                                | awk '{print $1}' | grep -Exq "$EXP_UKI_SHA"     || { echo "FAIL 671a: booted UKI sha drift"; exit 671; }
RT_PCR11="$(awk '{gsub(/^0x/,""); print toupper($0)}' /sys/class/tpm/tpm0/pcr-sha256/11)"
ST_PCR11="$(tr -d '[:space:]' < "$PCR11_FILE" | tr 'a-f' 'A-F')"
{ [ "$RT_PCR11" = "$ST_PCR11" ] && [ "$ST_PCR11" = "$EXP_PCR11" ]; } || { echo "FAIL 671b: PCR11 mismatch"; exit 671; }

PRE_INITRAMFS_MTIME="$(stat -c '%Y' "$INITRAMFS_PATH")"
PRE_UKI_MTIME="$(stat -c '%Y' "$UKI_PATH")"
PRE_PRODRULE_INO="$(stat -c '%i' "$PRODRULE")"

echo "=== Gate 6B Phase 1: pre-remove reference capture 13/13 PASS ==="

PRE_TIME_6B="$(date -Iseconds)"
PRE_TIME_6B_EPOCH="$(date -d "$PRE_TIME_6B" +%s)"

# Phase 2: single mutation: dnf remove -y figlet
DNF_RC=0
dnf remove -y figlet || DNF_RC=$?

# Phase 3: post-remove invariant assertion
[ "$DNF_RC" -eq 0 ] || { echo "FAIL 681: dnf remove rc=$DNF_RC"; exit 681; }
rpm -q figlet >/dev/null 2>&1 && { echo "FAIL 682: figlet still installed"; exit 682; } || true

EXIT_HITS="$(journalctl -q -t tboot-dnf-posttrans --since "$PRE_TIME_6B" --no-pager \
              | grep -c 'exit reason: manifest unchanged; baseline NOT updated; helper not invoked')"
[ "$EXIT_HITS" -eq 1 ] || { echo "FAIL 683: decider exit-reason lines = $EXIT_HITS"; exit 683; }
journalctl -q -t tboot-dnf-posttrans --since "$PRE_TIME_6B" --no-pager \
  | grep -Fq 'helper invocation decision: SKIP (manifest unchanged)' \
  || { echo "FAIL 684: SKIP decision absent"; exit 684; }
ERR_HITS="$(journalctl -q -t tboot-dnf-posttrans --since "$PRE_TIME_6B" --no-pager | grep -cE '^.* err: ')"
[ "$ERR_HITS" -eq 0 ] || { echo "FAIL 685: decider $ERR_HITS err: lines"; exit 685; }

grep -Fxq 'decision=unchanged' "$LAST_DECISION" || { echo "FAIL 686: decision != unchanged"; exit 686; }
grep -E '^detail=.*helper not invoked'   "$LAST_DECISION" >/dev/null || { echo "FAIL 687: 'helper not invoked' missing"; exit 687; }
grep -E '^detail=.*baseline NOT updated' "$LAST_DECISION" >/dev/null || { echo "FAIL 688: 'baseline NOT updated' missing"; exit 688; }

NEW_DECISION_EPOCH="$(awk -F= '/^epoch=/{print $2; exit}' "$LAST_DECISION")"
# Stronger than Gate 6A's check: epoch must STRICTLY exceed Gate 6A's epoch
{ [ -n "$NEW_DECISION_EPOCH" ] \
    && [ "$NEW_DECISION_EPOCH" -ge "$PRE_TIME_6B_EPOCH" ] \
    && [ "$NEW_DECISION_EPOCH" -gt "$EXP_6A_DECISION_EPOCH" ]; } \
  || { echo "FAIL 689: last-decision epoch did not advance correctly (6A=$EXP_6A_DECISION_EPOCH pre-6B=$PRE_TIME_6B_EPOCH new=$NEW_DECISION_EPOCH)"; cat "$LAST_DECISION"; exit 689; }

grep -Fxq "decider_version=${EXP_DECIDER_VERSION}" "$LAST_DECISION" || { echo "FAIL 690: decider_version mismatch"; exit 690; }

sha256sum "$BASELINE"      | awk '{print $1}' | grep -Exq "$EXP_BASELINE_SHA" || { echo "FAIL 691: baseline drift"; exit 691; }
sha256sum "$LAST_MANIFEST" | awk '{print $1}' | grep -Exq "$EXP_BASELINE_SHA" || { echo "FAIL 692: last-computed drift"; exit 692; }
[ ! -e "$LAST_ERROR" ] || { echo "FAIL 693: last-error appeared"; exit 693; }
SD_N_POST="$(find "$STATE_DIR" -mindepth 1 -maxdepth 1 -printf '.' | wc -c)"
[ "$SD_N_POST" -eq 3 ] || { echo "FAIL 694: state-dir count drift"; exit 694; }
[ ! -e "$SENTINEL" ] || { echo "FAIL 695a: sentinel appeared"; exit 695; }

if [ -e "$DECIDER_LOCK" ] || [ -L "$DECIDER_LOCK" ]; then
  [ -f "$DECIDER_LOCK" ] && [ ! -L "$DECIDER_LOCK" ] || { echo "FAIL 695b: decider lock not regular non-symlink"; exit 695; }
  ( exec 9<>"$DECIDER_LOCK"; flock -n 9 ) || { echo "FAIL 695b: decider lock actively held post-tx"; exit 695; }
fi
if [ -e "$HELPER_LOCK" ] || [ -L "$HELPER_LOCK" ]; then
  [ -f "$HELPER_LOCK" ] && [ ! -L "$HELPER_LOCK" ] || { echo "FAIL 695c: helper lock not regular non-symlink"; exit 695; }
  ( exec 9<>"$HELPER_LOCK"; flock -n 9 ) || { echo "FAIL 695c: helper lock actively held post-tx"; exit 695; }
fi

HELPER_HITS="$(journalctl -q -t tboot-dnf-helper --since "$PRE_TIME_6B" --no-pager | wc -l)"
[ "$HELPER_HITS" -eq 0 ] || { echo "FAIL 696: helper journal not silent"; exit 696; }

sha256sum /etc/kernel/install.d/80-tpm2-sign.install | awk '{print $1}' | grep -Exq "$EXP_HOOK_SHA"    || { echo "FAIL 697a: hook sha drift";    exit 697; }
sha256sum /usr/local/sbin/tboot-dnf-helper           | awk '{print $1}' | grep -Exq "$EXP_HELPER_SHA"  || { echo "FAIL 697b: helper sha drift";  exit 697; }
sha256sum /usr/local/sbin/tboot-predict-pcr11        | awk '{print $1}' | grep -Exq "$EXP_PREDICT_SHA" || { echo "FAIL 697c: predict sha drift"; exit 697; }
sha256sum /usr/local/bin/tboot-dnf-posttrans         | awk '{print $1}' | grep -Exq "$EXP_DECIDER_SHA" || { echo "FAIL 697d: decider sha drift"; exit 697; }
sha256sum "$UKI_PATH"                                | awk '{print $1}' | grep -Exq "$EXP_UKI_SHA"     || { echo "FAIL 697e: booted UKI sha drift"; exit 697; }
RT_PCR11_POST="$(awk '{gsub(/^0x/,""); print toupper($0)}' /sys/class/tpm/tpm0/pcr-sha256/11)"
[ "$RT_PCR11_POST" = "$EXP_PCR11" ] || { echo "FAIL 697f: runtime PCR11 drift"; exit 697; }
POST_INITRAMFS_MTIME="$(stat -c '%Y' "$INITRAMFS_PATH")"
POST_UKI_MTIME="$(stat -c '%Y' "$UKI_PATH")"
[ "$POST_INITRAMFS_MTIME" -eq "$PRE_INITRAMFS_MTIME" ] || { echo "FAIL 697g: initramfs mtime advanced"; exit 697; }
[ "$POST_UKI_MTIME"       -eq "$PRE_UKI_MTIME" ]       || { echo "FAIL 697h: ESP UKI mtime advanced"; exit 697; }

mapfile -t ACTIONS_POST < <(find "$ACTIONS_DIR" -maxdepth 1 -name '*.actions' -printf '%p\n' | sort)
{ [ "${#ACTIONS_POST[@]}" -eq 1 ] && [ "${ACTIONS_POST[0]}" = "$PRODRULE" ]; } || { echo "FAIL 698: unexpected *.actions entries"; exit 698; }
sha256sum "$PRODRULE" | awk '{print $1}' | grep -Exq "$EXP_PRODRULE_SHA" || { echo "FAIL 699a: production rule sha drift"; exit 699; }
POST_PRODRULE_INO="$(stat -c '%i' "$PRODRULE")"
[ "$POST_PRODRULE_INO" = "$PRE_PRODRULE_INO" ] || { echo "FAIL 699b: production rule inode changed"; exit 699; }

echo "=== Gate 6B Phase 3 post-transaction: 19/19 PASS ==="
echo ""
echo "=== Gate 6B OVERALL: PASS ==="
echo "  transaction       : dnf remove -y figlet (rc=0)"
echo "  PRE_TIME_6B       : $PRE_TIME_6B (epoch $PRE_TIME_6B_EPOCH)"
echo "  decider firings   : 1 ('exit reason: manifest unchanged...')"
echo "  decision          : unchanged"
echo "  last-decision     : epoch advanced 6A:$EXP_6A_DECISION_EPOCH -> 6B:$NEW_DECISION_EPOCH"
echo "  baseline          : sha unchanged ($EXP_BASELINE_SHA)"
echo "  helper            : journal silent, lock not actively held, sentinel never written"
echo "  boot artefacts    : 6/6 sha-byte-stable + initramfs/UKI mtime unchanged"
echo ""
echo "Both halves of Gate 6 (install + remove) validated the no-drift decider path."
echo "Ready for Gate 7 (B.2.3 closure snapshot module5-b2-3-validated)."
EOF
```

**Expected output.**

```
=== Gate 6B Phase 0: precondition 2/2 PASS ===
=== Gate 6B Phase 1: pre-remove reference capture 13/13 PASS ===
[... dnf remove -y figlet output ...]
[... decider info-line stream (same shape as Gate 6A) ...]
=== Gate 6B Phase 3 post-transaction: 19/19 PASS ===

=== Gate 6B OVERALL: PASS ===
  ...
  last-decision     : epoch advanced 6A:<6A epoch> -> 6B:<6B epoch>
  ...
```

**Closure criteria.** dnf rc=0 + figlet removed + decider fired exactly once + decision=unchanged + helper not invoked + baseline not updated + boot artefacts byte-stable + new last-decision epoch strictly greater than Gate 6A's. Proceed to Gate 7.

**Stop condition.** Any FAIL 6xx. Capture state and rollback to `module5-b2-3-decider-installed-primed`; both Gate 6A and Gate 6B effects are wiped on rollback (intentional: neither was snapshotted).

---

### Gate 7: B.2.3 closure snapshot

**Goal.** Take the closure snapshot `module5-b2-3-validated` on the Proxmox host. Three phases: VM-side final sanity check (read-only), Proxmox-host snapshot (mutation on the host, not the VM), VM-side post-snapshot health (read-only).

**Mutation scope.** One Proxmox snapshot on VM 500: `module5-b2-3-validated`, parent `module5-b2-3-decider-installed-primed`. No VM-side mutation.

**Rollback / safety.** If Phase 1 FAILs, do not take the snapshot. If Phase 2 FAILs (snapshot operation failed), VM state is intact; retry after host triage. If Phase 3 FAILs, the snapshot exists but the VM may have been disturbed by the snapshot operation; capture forensics and decide whether to delete the new snapshot (`qm delsnapshot 500 module5-b2-3-validated`) or keep it as forensic evidence. Deeper rollback target stays `module5-b2-3-decider-installed-primed`.

**Phase 1 (VM-side final sanity check, 14 invariants):**

```bash
bash <<'EOF'
set -u

EXP_DECIDER_SHA="1e75110935a72fc9433c9b4bf5688d26bc6d3bd1e02f4388b5e1f32ff1e5cc05"
EXP_BASELINE_SHA="75dc4f5651fc8996d39b545ac0bf7244a2a5888403b0ba0cd448a671daf80160"
EXP_PRODRULE_SHA="dfbffe56fa95f4e0231569bb3a1aabc8952b9f99c9d290e4f8fb1005ef90d798"
EXP_HOOK_SHA="5857e51d5551e05a7ca384f71529ec2fef07f5a6dbe40df70c6bbe49d720947e"
EXP_HELPER_SHA="937afc7a0d199bf7b7d1a5f5bcccf45c37c0f4f7dd0ad924b2567c4da7125e44"
EXP_PREDICT_SHA="b729ec271dd76cd8b4581f16c86e50fc28b947f2a1de5a7e16c6d5828591e3de"
EXP_UKI_SHA="b6002e666afdf71e5a083311295ac6a5f3ef3b443ceb4ec8c16c9ddb41077943"
EXP_PCR11="A03EB49C9335D721758AE8AA9C34DA87664C5CAF9FAD7C314C68C038B16435DD"

# Read Gate 6B's epoch from disk (the most recent last-decision write)
EXP_6B_DECISION_EPOCH="$(awk -F= '/^epoch=/{print $2; exit}' /var/lib/tboot-dnf-posttrans/last-decision)"

UKI_PATH="/boot/efi/EFI/Linux/0123456789abcdef0123456789abcdef-6.19.14-200.fc43.x86_64.efi"
PCR11_FILE="/root/tboot-lab/state/expected-pcr11-after-hook-uki.txt"
PRODRULE="/etc/dnf/libdnf5-plugins/actions.d/50-tboot-posttrans.actions"
ACTIONS_DIR="/etc/dnf/libdnf5-plugins/actions.d"
DISABLED_PROBE="${ACTIONS_DIR}/00-tboot-trigger-probe.actions.disabled-after-b21-validation"
STATE_DIR="/var/lib/tboot-dnf-posttrans"
BASELINE="${STATE_DIR}/boot-input-manifest.baseline"
LAST_DECISION="${STATE_DIR}/last-decision"
LAST_MANIFEST="${STATE_DIR}/last-computed-manifest"
LAST_ERROR="${STATE_DIR}/last-error"
SENTINEL="/var/lib/tboot-dnf-helper/UNSAFE-TO-REBOOT"
DECIDER_LOCK="/run/tboot-dnf-posttrans.lock"
HELPER_LOCK="/run/tboot-dnf-helper.lock"

[ "$(id -u)" -eq 0 ] || { echo "FAIL 711: must run as root"; exit 711; }
rpm -q figlet >/dev/null 2>&1 && { echo "FAIL 712: figlet still installed (Gate 6B should have removed it)"; exit 712; } || true

sha256sum "$PRODRULE" | awk '{print $1}' | grep -Exq "$EXP_PRODRULE_SHA" || { echo "FAIL 713a: production rule sha drift"; exit 713; }
PR_META="$(stat -c '%a %U %G %s' "$PRODRULE")"
[ "$PR_META" = "644 root root 87" ] || { echo "FAIL 713b: production rule metadata drift"; exit 713; }

mapfile -t ACTIONS_NOW < <(find "$ACTIONS_DIR" -maxdepth 1 -name '*.actions' -printf '%p\n' | sort)
{ [ "${#ACTIONS_NOW[@]}" -eq 1 ] && [ "${ACTIONS_NOW[0]}" = "$PRODRULE" ]; } || { echo "FAIL 714a: unexpected *.actions"; exit 714; }
[ -f "$DISABLED_PROBE" ] && [ ! -L "$DISABLED_PROBE" ] || { echo "FAIL 714b: disabled probe missing"; exit 714; }

sha256sum /usr/local/bin/tboot-dnf-posttrans | awk '{print $1}' | grep -Exq "$EXP_DECIDER_SHA" || { echo "FAIL 715a: decider sha drift"; exit 715; }
SD_META="$(stat -c '%a %U %G' "$STATE_DIR")"
[ "$SD_META" = "700 root root" ] || { echo "FAIL 715b: state-dir metadata drift"; exit 715; }
SD_N="$(find "$STATE_DIR" -mindepth 1 -maxdepth 1 -printf '.' | wc -c)"
[ "$SD_N" -eq 3 ] || { echo "FAIL 715c: state-dir count drift"; exit 715; }
sha256sum "$BASELINE"      | awk '{print $1}' | grep -Exq "$EXP_BASELINE_SHA" || { echo "FAIL 715d: baseline drift"; exit 715; }
sha256sum "$LAST_MANIFEST" | awk '{print $1}' | grep -Exq "$EXP_BASELINE_SHA" || { echo "FAIL 715e: last-computed drift"; exit 715; }

grep -Fxq 'decision=unchanged' "$LAST_DECISION" || { echo "FAIL 716a: last-decision not decision=unchanged"; exit 716; }
CUR_DECISION_EPOCH="$(awk -F= '/^epoch=/{print $2; exit}' "$LAST_DECISION")"
[ "$CUR_DECISION_EPOCH" = "$EXP_6B_DECISION_EPOCH" ] || { echo "FAIL 716b: last-decision epoch drift since Gate 6B"; exit 716; }

[ ! -e "$LAST_ERROR" ] || { echo "FAIL 717a: last-error present"; exit 717; }
[ ! -e "$SENTINEL" ]   || { echo "FAIL 717b: sentinel present"; exit 717; }
if [ -e "$DECIDER_LOCK" ] || [ -L "$DECIDER_LOCK" ]; then
  [ -f "$DECIDER_LOCK" ] && [ ! -L "$DECIDER_LOCK" ] || { echo "FAIL 717c: decider lock not regular non-symlink"; exit 717; }
  ( exec 9<>"$DECIDER_LOCK"; flock -n 9 ) || { echo "FAIL 717c: decider lock actively held"; exit 717; }
fi
if [ -e "$HELPER_LOCK" ] || [ -L "$HELPER_LOCK" ]; then
  [ -f "$HELPER_LOCK" ] && [ ! -L "$HELPER_LOCK" ] || { echo "FAIL 717d: helper lock not regular non-symlink"; exit 717; }
  ( exec 9<>"$HELPER_LOCK"; flock -n 9 ) || { echo "FAIL 717d: helper lock actively held"; exit 717; }
fi

sha256sum /etc/kernel/install.d/80-tpm2-sign.install | awk '{print $1}' | grep -Exq "$EXP_HOOK_SHA"    || { echo "FAIL 718a: hook sha drift";    exit 718; }
sha256sum /usr/local/sbin/tboot-dnf-helper           | awk '{print $1}' | grep -Exq "$EXP_HELPER_SHA"  || { echo "FAIL 718b: helper sha drift";  exit 718; }
sha256sum /usr/local/sbin/tboot-predict-pcr11        | awk '{print $1}' | grep -Exq "$EXP_PREDICT_SHA" || { echo "FAIL 718c: predict sha drift"; exit 718; }

sha256sum "$UKI_PATH" | awk '{print $1}' | grep -Exq "$EXP_UKI_SHA" || { echo "FAIL 719a: booted UKI sha drift"; exit 719; }
RT_PCR11="$(awk '{gsub(/^0x/,""); print toupper($0)}' /sys/class/tpm/tpm0/pcr-sha256/11)"
ST_PCR11="$(tr -d '[:space:]' < "$PCR11_FILE" | tr 'a-f' 'A-F')"
{ [ "$RT_PCR11" = "$ST_PCR11" ] && [ "$ST_PCR11" = "$EXP_PCR11" ]; } || { echo "FAIL 719b: PCR11 mismatch"; exit 719; }

HELPER_HITS="$(journalctl -q -t tboot-dnf-helper --since '-5 min' --no-pager | wc -l)"
[ "$HELPER_HITS" -eq 0 ] || { echo "FAIL 720: helper journal not silent in last 5 min"; exit 720; }

echo "=== Gate 7 Phase 1 VM-side final sanity check: 10/10 PASS ==="
echo "  READY for Phase 2 (Proxmox snapshot on host pve-host)."
EOF
```

**Phase 2 (Proxmox host pve-host):**

```
qm snapshot 500 module5-b2-3-validated \
  --description "B.2.3 closure: production .actions rule 50-tboot-posttrans.actions installed at /etc/dnf/libdnf5-plugins/actions.d/ (sha dfbffe56fa95f4e0231569bb3a1aabc8952b9f99c9d290e4f8fb1005ef90d798, mode 0644 root:root, 87 bytes); rule shape post_transaction:::enabled=host-only raise_error=1:/usr/local/sbin/tboot-dnf-posttrans (Gate 1 design lock). Decider at /usr/local/sbin/tboot-dnf-posttrans (sha 1e75110935a72fc9433c9b4bf5688d26bc6d3bd1e02f4388b5e1f32ff1e5cc05) + state-dir /var/lib/tboot-dnf-posttrans (root:root 0700) with primed baseline (sha 75dc4f5651fc8996d39b545ac0bf7244a2a5888403b0ba0cd448a671daf80160, 19752 bytes). No-drift path validated end-to-end through two real DNF transactions: dnf install -y figlet (Gate 6A) and dnf remove -y figlet (Gate 6B). Both produced decision=unchanged with detail 'manifest matches baseline; helper not invoked; baseline NOT updated (per design)'. Helper never invoked; sentinel never written; baseline never updated; 6 trust-chain shas + runtime PCR 11 byte-stable; initramfs/ESP UKI mtimes unchanged from B.2.2 closure. Gates 4-7 closed. B.2.4 (live drift-detection through a real dracut-sensitive transaction) still pending. Blind dnf update still blocked. dnf upgrade systemd* still separately blocked (scope of B.4/B.5)."

qm listsnapshot 500
```

Visually confirm `module5-b2-3-validated` appears as a new node parented to `module5-b2-3-decider-installed-primed`, with `current` below it.

**Phase 3 (VM-side post-snapshot health, 3 invariants):**

```bash
bash <<'EOF'
set -u

EXP_PCR11="A03EB49C9335D721758AE8AA9C34DA87664C5CAF9FAD7C314C68C038B16435DD"
EXP_BASELINE_SHA="75dc4f5651fc8996d39b545ac0bf7244a2a5888403b0ba0cd448a671daf80160"
EXP_PRODRULE_SHA="dfbffe56fa95f4e0231569bb3a1aabc8952b9f99c9d290e4f8fb1005ef90d798"
PCR11_FILE="/root/tboot-lab/state/expected-pcr11-after-hook-uki.txt"
BASELINE="/var/lib/tboot-dnf-posttrans/boot-input-manifest.baseline"
PRODRULE="/etc/dnf/libdnf5-plugins/actions.d/50-tboot-posttrans.actions"

RT_PCR11="$(awk '{gsub(/^0x/,""); print toupper($0)}' /sys/class/tpm/tpm0/pcr-sha256/11)"
ST_PCR11="$(tr -d '[:space:]' < "$PCR11_FILE" | tr 'a-f' 'A-F')"
{ [ "$RT_PCR11" = "$ST_PCR11" ] && [ "$ST_PCR11" = "$EXP_PCR11" ]; } \
  || { echo "FAIL 731: PCR11 mismatch post-snapshot"; exit 731; }

sha256sum "$BASELINE" | awk '{print $1}' | grep -Exq "$EXP_BASELINE_SHA" \
  || { echo "FAIL 732: baseline sha drift post-snapshot"; exit 732; }

sha256sum "$PRODRULE" | awk '{print $1}' | grep -Exq "$EXP_PRODRULE_SHA" \
  || { echo "FAIL 733: production rule sha drift post-snapshot"; exit 733; }

echo "=== Gate 7 Phase 3 post-snapshot VM health: 3/3 PASS ==="
echo ""
echo "=== Gate 7 OVERALL: PASS ==="
echo "  Phase 1: 10/10 VM-side final sanity check"
echo "  Phase 2: snapshot module5-b2-3-validated taken on host pve-host (visual confirmation)"
echo "  Phase 3: 3/3 VM survived the snapshot operation"
echo ""
echo "*** B.2.3 IS COMPLETE. ***"
echo "Gates 4-7 closed. Decider + production rule + no-drift path all validated."
echo ""
echo "Still pending:"
echo "  - B.2.4: live drift-detection through a real dracut-sensitive transaction"
echo "  - B.4 / B.5: systemd-boot signing automation + validation"
echo "  - Module 4: governance / recovery / lifecycle"
echo ""
echo "Blind dnf update STILL blocked (lifted only after B.2.4 closure)."
echo "dnf upgrade systemd* STILL separately blocked (lifted only after B.5 closure)."
EOF
```

**Expected output across all three phases.**

```
=== Gate 7 Phase 1 VM-side final sanity check: 10/10 PASS ===
  READY for Phase 2 (Proxmox snapshot on host pve-host).

[on host pve-host:]
qm snapshot 500 module5-b2-3-validated ...
qm listsnapshot 500
  ... `-> module5-b2-3-decider-installed-primed
       `-> module5-b2-3-validated
            `-> current

[back on VM:]
=== Gate 7 Phase 3 post-snapshot VM health: 3/3 PASS ===

=== Gate 7 OVERALL: PASS ===
  Phase 1: 10/10 VM-side final sanity check
  Phase 2: snapshot module5-b2-3-validated taken on host pve-host (visual confirmation)
  Phase 3: 3/3 VM survived the snapshot operation

*** B.2.3 IS COMPLETE. ***
```

**Closure criteria.** All three phases PASS. The snapshot exists in `qm listsnapshot 500` output. B.2.3 is complete.

**Stop condition.**

- Phase 1 FAILs 7xx: do not take the snapshot. Investigate which invariant moved.
- Phase 2 FAIL on `qm snapshot`: triage on host (thin-pool full, name collision, parent missing). VM state is intact; only the snapshot artefact failed.
- Phase 2 visual confirmation fails: do not declare Gate 7 closed even if `qm snapshot` reported success. Triage on host.
- Phase 3 FAILs 73x: severe. Snapshot operation disturbed the VM. Capture forensics on host and VM; decide whether the new snapshot is trustworthy or should be deleted.

**Snapshot point.** `module5-b2-3-validated`: the B.2.3 closure anchor; preferred rollback target for B.2.4. Parent: `module5-b2-3-decider-installed-primed`.

---

### Step 33 closure

✅ **B.2.3 closed end-to-end.** Production `.actions` rule live; decider invoked on every DNF transaction; no-drift path validated in both directions (install + remove); closure snapshot `module5-b2-3-validated` taken. **At the time of Step 33 closure, B.2.4 was pending and blind `dnf update` / `dnf upgrade` remained blocked pending B.2.4 closure.** B.2.4 was subsequently closed end-to-end via **Steps 34–37 below** (Gates 1–3 in Step 34: positive drift transaction `dnf install -y dracut-network`; Gates 4–5 in Step 35: pre-reboot cross-validation + pre-reboot snapshot `module5-b2-4-pre-reboot`; Gates 6A–6B in Step 36: reboot with no LUKS passphrase prompt + post-reboot validation 28/28 PASS, runtime PCR 11 = `28E66CE2…C90F` byte-matched stored prediction; Gate 7 in Step 37: closure snapshot `module5-b2-4-validated` + read-only ESP audit + staged DNF discipline lift). **`dnf upgrade systemd*` remains separately blocked** (scope of B.4 / B.5, not B.2.3 / B.2.4).

From this point onward, every DNF transaction invokes the decider. The decider's no-drift path is validated. The drift path (detect drift → write sentinel → invoke helper → update baseline on helper success → clear sentinel) remains B.2.4's job.

---


# Forward work

Genuinely-pending blocks are tracked in **Appendix D** (end of document); completed blocks are in the validation ledger (**Appendix C**) and `00_Current_Project_State.md`.

---

# Reference tables

The architectural-invariants table and the trust-boundary (Deviation D) regex are in **Appendix A** (end of document). Rebuild-specific values (PCR 7, PCR 11, pkfp, pol, machine-id, ESP UKI sha) live in `00_Current_Project_State.md`.

---

## Step 34: Block B.2.4 Gates 1–3: live drift-detection through a real dracut-sensitive transaction

**Goal:** validate the positive drift branch end-to-end on a live system. One scoped `dnf install` of a known-not-installed, dracut-sensitive, trust-chain-safe candidate (`dracut-network`) must (a) fire the production `.actions` rule, (b) drive the decider into the drift branch, (c) sentinel-then-invoke-then-clear the helper, (d) cause `dracut` + `kernel-install add` + the `80-tpm2-sign.install` hook to run for every enumerated kernel, (e) atomically replace the baseline manifest with the post-helper computed manifest, (f) advance the stored PCR 11 prediction file (mtime always; value if dracut autodetect pulls the candidate's module into the host-only initramfs). Reboot validation is the scope of a later step (B.2.4 Gates 4–7), not Step 34.
**Prerequisites:** Step 33 (B.2.3 Gates 4.0–7) closed; rollback anchor `module5-b2-3-validated` on Proxmox host pve-host.
**Where to run:** Fedora VM 500 as root inside `tmux` for Gates 1 and 3; Proxmox host pve-host as root for the Gate 2 snapshot command itself. No reboot in this step.

> [!important] Candidate-selection note
> The validated candidate for the reference rebuild is `dracut-network`. Alternatives that pass Gate 1 selection criteria in dry-run analysis are `dracut-live`, `dracut-squash`, `dracut-tools-extra`. If running on a system where `dracut-network` is already installed, Gate 1 must fall back to one of these. The rest of the gate scripts are candidate-agnostic so long as the `CANDIDATE` variable at the top of each script is updated consistently.

> [!warning] Harness terminology pitfall (`06F` K.1)
> The decider logs its production path as `mode=normal`; the helper logs the same path as `mode=production`. Any harness assertion on helper-start lines must use `mode=production`. The corrected replay harness in this step already does. Full explanation: `06C` / `06F` K.1.

> [!info] Helper scratch path
> The helper creates its per-invocation scratch directory under `/run/tboot-helper-XXXXXX` (e.g. observed at Gate 3: `/run/tboot-helper-Uvj19o`). The decider's scratch directory pattern is `/run/tboot-dnf-posttrans.XXXXXX` (different prefix, different separator).

### Commands: Gate 1: candidate selection + read-only preflight

No persistent trust-chain mutation. Expected and accepted: temporary files under `/tmp`; journald entries under tags `tboot-dnf-posttrans` and `tboot-dnf-helper`; DNF metadata *read* from existing local cache (no `--refresh`); `dnf install --assumeno` resolves a transaction plan and aborts at the prompt (no commit, no `post_transaction` rule fires); helper lock acquisition + release; helper scratch dir under `/run/tboot-helper-XXXXXX`. Asserted byte-stable: baseline, last-computed-manifest, last-decision, last-error absence, decider state-dir entry count, helper state-dir digest, sentinel absence, booted UKI / initramfs / PCR 11 prediction file mtimes, production rule. Rollback not required (read-only).

```bash
bash <<'EOF'
set -u

# ============================================================================
# B.2.4 Gate 1: candidate selection + read-only preflight
# ============================================================================

EXP_DECIDER_SHA="1e75110935a72fc9433c9b4bf5688d26bc6d3bd1e02f4388b5e1f32ff1e5cc05"
EXP_BASELINE_SHA="75dc4f5651fc8996d39b545ac0bf7244a2a5888403b0ba0cd448a671daf80160"
EXP_PRODRULE_SHA="dfbffe56fa95f4e0231569bb3a1aabc8952b9f99c9d290e4f8fb1005ef90d798"
EXP_HOOK_SHA="5857e51d5551e05a7ca384f71529ec2fef07f5a6dbe40df70c6bbe49d720947e"
EXP_HELPER_SHA="937afc7a0d199bf7b7d1a5f5bcccf45c37c0f4f7dd0ad924b2567c4da7125e44"
EXP_PREDICT_SHA="b729ec271dd76cd8b4581f16c86e50fc28b947f2a1de5a7e16c6d5828591e3de"
EXP_UKI_SHA="b6002e666afdf71e5a083311295ac6a5f3ef3b443ceb4ec8c16c9ddb41077943"
EXP_PCR11="A03EB49C9335D721758AE8AA9C34DA87664C5CAF9FAD7C314C68C038B16435DD"
EXP_DECIDER_VERSION="1.0.0"
EXP_HELPER_VERSION="1.0.0"
EXP_MID="0123456789abcdef0123456789abcdef"
EXP_KVER_BOOTED="6.19.14-200.fc43.x86_64"
EXP_KVER_SECONDARY="6.17.1-300.fc43.x86_64"

UKI_PATH="/boot/efi/EFI/Linux/${EXP_MID}-${EXP_KVER_BOOTED}.efi"
INITRAMFS_PATH="/boot/initramfs-${EXP_KVER_BOOTED}.img"
PCR11_FILE="/root/tboot-lab/state/expected-pcr11-after-hook-uki.txt"
PRODRULE="/etc/dnf/libdnf5-plugins/actions.d/50-tboot-posttrans.actions"
ACTIONS_DIR="/etc/dnf/libdnf5-plugins/actions.d"

DECIDER="/usr/local/sbin/tboot-dnf-posttrans"
HELPER="/usr/local/sbin/tboot-dnf-helper"
PREDICT="/usr/local/sbin/tboot-predict-pcr11"
HOOK="/etc/kernel/install.d/80-tpm2-sign.install"

DECIDER_STATE="/var/lib/tboot-dnf-posttrans"
BASELINE="${DECIDER_STATE}/boot-input-manifest.baseline"
LAST_DECISION="${DECIDER_STATE}/last-decision"
LAST_MANIFEST="${DECIDER_STATE}/last-computed-manifest"
LAST_ERROR="${DECIDER_STATE}/last-error"

HELPER_STATE="/var/lib/tboot-dnf-helper"
SENTINEL="${HELPER_STATE}/UNSAFE-TO-REBOOT"
FAILURE_MARKER="${HELPER_STATE}/last-failure"
FAILURE_LOG="${HELPER_STATE}/last-failure.log"

DECIDER_LOCK="/run/tboot-dnf-posttrans.lock"
HELPER_LOCK="/run/tboot-dnf-helper.lock"

CANDIDATE="dracut-network"

SENSITIVE_RE='^[[:space:]]+(systemd|kernel|cryptsetup|libcryptsetup|luks|tpm2|shim|grub|grub2)([[:space:]-]|$)'

TMP_FILES=()
new_tmp() { local t; t="$(mktemp)"; TMP_FILES+=("$t"); printf '%s\n' "$t"; }
_cleanup() { local f; for f in "${TMP_FILES[@]:-}"; do rm -f "$f" 2>/dev/null || true; done; }
trap _cleanup EXIT HUP INT TERM

PASS=0
fail() { echo "FAIL $1: $2"; exit "$1"; }

hsd_digest() {
    if [ ! -d "$HELPER_STATE" ]; then echo "no-state-dir"; return 0; fi
    find "$HELPER_STATE" -maxdepth 1 -type f -printf '%f\n' 2>/dev/null \
        | sort \
        | while read -r f; do sha256sum "$HELPER_STATE/$f" 2>/dev/null; done \
        | sha256sum | awk '{print $1}'
}

assert_lock_free() {
    local lock="$1" code="$2"
    if [ -e "$lock" ] || [ -L "$lock" ]; then
        [ -f "$lock" ] && [ ! -L "$lock" ] || fail "$code" "$lock not regular non-symlink"
        ( exec 9<>"$lock"; flock -n 9 ) || fail "$code" "$lock actively held"
    fi
}

# Phase 1: B.2.3 closure invariants (21 assertions)
[ "$(id -u)" -eq 0 ] || fail 101 "must run as root"; PASS=$((PASS+1))
sha256sum "$DECIDER"  | awk '{print $1}' | grep -Exq "$EXP_DECIDER_SHA"  || fail 110 "decider sha drift";    PASS=$((PASS+1))
sha256sum "$HELPER"   | awk '{print $1}' | grep -Exq "$EXP_HELPER_SHA"   || fail 111 "helper sha drift";     PASS=$((PASS+1))
sha256sum "$PREDICT"  | awk '{print $1}' | grep -Exq "$EXP_PREDICT_SHA"  || fail 112 "predict sha drift";    PASS=$((PASS+1))
sha256sum "$HOOK"     | awk '{print $1}' | grep -Exq "$EXP_HOOK_SHA"     || fail 113 "hook sha drift";       PASS=$((PASS+1))
sha256sum "$PRODRULE" | awk '{print $1}' | grep -Exq "$EXP_PRODRULE_SHA" || fail 114 "production rule drift"; PASS=$((PASS+1))
sha256sum "$BASELINE" | awk '{print $1}' | grep -Exq "$EXP_BASELINE_SHA" || fail 115 "baseline sha drift";   PASS=$((PASS+1))
sha256sum "$UKI_PATH" | awk '{print $1}' | grep -Exq "$EXP_UKI_SHA"      || fail 116 "booted UKI sha drift"; PASS=$((PASS+1))

RT_PCR11="$(awk '{gsub(/^0x/,""); print toupper($0)}' /sys/class/tpm/tpm0/pcr-sha256/11)"
[ "$RT_PCR11" = "$EXP_PCR11" ] || fail 120 "runtime PCR 11 mismatch"; PASS=$((PASS+1))

STORED_PCR11="$(tr -d '[:space:]' <"$PCR11_FILE" | awk '{print toupper($0)}')"
[ "$STORED_PCR11" = "$EXP_PCR11" ] || fail 121 "stored PCR 11 file drift"; PASS=$((PASS+1))

SD_N="$(find "$DECIDER_STATE" -mindepth 1 -maxdepth 1 -printf '.' | wc -c)"
[ "$SD_N" -eq 3 ] || fail 130 "decider state-dir entry count = $SD_N"; PASS=$((PASS+1))
sha256sum "$LAST_MANIFEST" | awk '{print $1}' | grep -Exq "$EXP_BASELINE_SHA" || fail 131 "last-manifest sha != baseline sha"; PASS=$((PASS+1))
[ ! -e "$LAST_ERROR" ] || fail 132 "decider last-error present"; PASS=$((PASS+1))

LD_DECISION="$(awk -F= '$1=="decision"{print $2}' "$LAST_DECISION")"
LD_VERSION="$(awk -F= '$1=="decider_version"{print $2}' "$LAST_DECISION")"
LD_EPOCH="$(awk -F= '$1=="epoch"{print $2}' "$LAST_DECISION")"
[ "$LD_DECISION" = "unchanged" ] || fail 140 "last-decision.decision != unchanged"; PASS=$((PASS+1))
[ "$LD_VERSION"  = "$EXP_DECIDER_VERSION" ] || fail 141 "decider_version drift"; PASS=$((PASS+1))
[[ "$LD_EPOCH" =~ ^[0-9]+$ ]] || fail 142 "last-decision.epoch not numeric"; PASS=$((PASS+1))

[ ! -e "$SENTINEL" ] || fail 150 "UNSAFE-TO-REBOOT sentinel present"; PASS=$((PASS+1))

mapfile -t ACTIONS_ACTIVE < <(find "$ACTIONS_DIR" -maxdepth 1 -type f -name '*.actions' -printf '%p\n' | sort)
{ [ "${#ACTIONS_ACTIVE[@]}" -eq 1 ] && [ "${ACTIONS_ACTIVE[0]}" = "$PRODRULE" ]; } || fail 160 "unexpected *.actions entries"
PASS=$((PASS+1))

assert_lock_free "$DECIDER_LOCK" 170
assert_lock_free "$HELPER_LOCK"  171
PASS=$((PASS+1))

[ "$(uname -r)" = "$EXP_KVER_BOOTED" ] || fail 180 "uname -r drift"; PASS=$((PASS+1))
[ "$(cat /etc/machine-id)" = "$EXP_MID" ] || fail 181 "machine-id drift"; PASS=$((PASS+1))

echo "=== Phase 1: B.2.3 closure invariants: ${PASS}/21 PASS ==="

# Phase 2: Candidate suitability by file ownership (6 assertions)
rpm -q "$CANDIDATE" >/dev/null 2>&1 && fail 201 "$CANDIDATE already installed; fall back to dracut-live | dracut-squash | dracut-tools-extra" || true
PASS=$((PASS+1))

dnf repoquery --quiet "$CANDIDATE" 2>/dev/null | grep -q . \
    || fail 202 "$CANDIDATE not found in enabled repos"
PASS=$((PASS+1))

FILES_LIST="$(new_tmp)"
dnf repoquery -l "$CANDIDATE" 2>/dev/null | sort -u >"$FILES_LIST"
[ -s "$FILES_LIST" ] || fail 203 "$CANDIDATE file list empty"; PASS=$((PASS+1))

DRACUT_MOD_HITS="$(grep -Ec '^/usr/lib/dracut/modules\.d/' "$FILES_LIST")"
[ "$DRACUT_MOD_HITS" -ge 1 ] || fail 204 "$CANDIDATE files do not intersect /usr/lib/dracut/modules.d/"; PASS=$((PASS+1))

grep -Eq '^(/etc/kernel/install\.d/|/etc/uefi-keys/|/etc/systemd/tpm2-pcr-)' "$FILES_LIST" && fail 205 "$CANDIDATE owns signing-pipeline paths" || true
PASS=$((PASS+1))

grep -Eq '^/(boot|lib/modules)/' "$FILES_LIST" && fail 206 "$CANDIDATE owns /boot or /lib/modules" || true
PASS=$((PASS+1))

echo "--- Candidate $CANDIDATE: manifest-category intersection preview ---"
echo "cat 02-dracut-modules     hits: ${DRACUT_MOD_HITS}"
echo "cat 01-dracut-config      hits: $(grep -Ec '^/etc/dracut\.conf|^/etc/dracut\.conf\.d/|^/usr/lib/dracut/dracut\.conf\.d/' "$FILES_LIST")"
echo "cat 04-firmware           hits: $(grep -Ec '^/usr/lib/firmware/' "$FILES_LIST")"
echo "cat 05-udev-rules         hits: $(grep -Ec '^/etc/udev/rules\.d/|^/usr/lib/udev/rules\.d/' "$FILES_LIST")"
echo "cat 06-modprobe-load      hits: $(grep -Ec '^/etc/modprobe\.d/|^/usr/lib/modprobe\.d/|^/etc/modules-load\.d/|^/usr/lib/modules-load\.d/' "$FILES_LIST")"
echo "cat 07-systemd-early-boot hits: $(grep -Ec '^/usr/lib/systemd/' "$FILES_LIST")"
echo "cat 08-cryptsetup-tooling hits: $(grep -Ec '^/usr/sbin/cryptsetup|^/usr/lib64/libcryptsetup' "$FILES_LIST")"
echo "cat 09-tpm2-tss           hits: $(grep -Ec '^/usr/sbin/tpm2_|^/usr/lib64/libtss2-' "$FILES_LIST")"
echo "cat 12-kernel-install     hits: $(grep -Ec '^/etc/kernel/install\.|^/usr/lib/kernel/install\.d/' "$FILES_LIST")"

echo "=== Phase 2: candidate file-ownership suitability: ${PASS}/27 cumulative PASS ==="

# Phase 3: Transaction preview via dnf install --assumeno (7 assertions; see 06F K.2)
ASSUMENO_OUT="$(new_tmp)"; ASSUMENO_RC=0
PRE_ASSUMENO_TS="$(date -Iseconds)"
sleep 1
dnf install --assumeno "$CANDIDATE" >"$ASSUMENO_OUT" 2>&1 || ASSUMENO_RC=$?
sleep 1
POST_ASSUMENO_TS="$(date -Iseconds)"

echo "--- dnf install --assumeno $CANDIDATE  (rc=$ASSUMENO_RC) ---"
cat "$ASSUMENO_OUT"

case "$ASSUMENO_RC" in
    0|1) : ;;
    *)   fail 300 "dnf --assumeno unexpected rc=$ASSUMENO_RC" ;;
esac
PASS=$((PASS+1))

grep -Eiq 'Operation aborted|Is this ok.*\[?y/N\]?|Nothing to do' "$ASSUMENO_OUT" \
    || fail 301 "dnf --assumeno output did not match a documented preview-refusal shape"
PASS=$((PASS+1))

[ -s "$ASSUMENO_OUT" ] || fail 302 "--assumeno produced empty output"; PASS=$((PASS+1))
grep -Eq "^[[:space:]]+${CANDIDATE}[[:space:]]" "$ASSUMENO_OUT" || fail 303 "$CANDIDATE not listed as installing"; PASS=$((PASS+1))

if grep -E "$SENSITIVE_RE" "$ASSUMENO_OUT" >/dev/null 2>&1; then
    grep -E "$SENSITIVE_RE" "$ASSUMENO_OUT"
    fail 304 "transaction would pull trust-chain sensitive package(s)"
fi
PASS=$((PASS+1))

UNEXPECTED_DRACUT="$(new_tmp)"
grep -E '^[[:space:]]+dracut-' "$ASSUMENO_OUT" \
    | grep -Ev "^[[:space:]]+${CANDIDATE}[[:space:]]" >"$UNEXPECTED_DRACUT" || true
if [ -s "$UNEXPECTED_DRACUT" ]; then
    cat "$UNEXPECTED_DRACUT"
    fail 305 "transaction would pull unexpected dracut-* package(s)"
fi
PASS=$((PASS+1))

JOURNAL_OUT="$(new_tmp)"
journalctl -q -t tboot-dnf-posttrans -o cat --since "$PRE_ASSUMENO_TS" --until "$POST_ASSUMENO_TS" >"$JOURNAL_OUT" 2>&1 || true
if [ -s "$JOURNAL_OUT" ]; then
    cat "$JOURNAL_OUT"
    fail 306 "decider fired during --assumeno window"
fi
PASS=$((PASS+1))

echo "=== Phase 3: transaction preview via --assumeno: ${PASS}/34 cumulative PASS ==="

# Phase 4A: Decider --dry-run with full before/after (12 assertions)
DEC_VER="$("$DECIDER" --version 2>&1 | head -1)"
echo "$DEC_VER" | grep -Fq "$EXP_DECIDER_VERSION" || fail 401 "decider --version unexpected"; PASS=$((PASS+1))

LAST_DECISION_SHA_BEFORE="$(sha256sum "$LAST_DECISION" | awk '{print $1}')"
LAST_DECISION_MTIME_BEFORE="$(stat -c '%Y' "$LAST_DECISION")"

DRY_OUT="$(new_tmp)"; DRY_RC=0
"$DECIDER" --dry-run >"$DRY_OUT" 2>&1 || DRY_RC=$?
cat "$DRY_OUT"
[ "$DRY_RC" -eq 0 ] || fail 410 "decider --dry-run rc=$DRY_RC"; PASS=$((PASS+1))
grep -Eq 'baseline compare result: unchanged' "$DRY_OUT" || fail 411 "decider --dry-run did not report 'unchanged'"; PASS=$((PASS+1))
grep -Eq 'would EXIT 0 \(manifest unchanged\)' "$DRY_OUT" || fail 412 "decider --dry-run missing expected decision phrase"; PASS=$((PASS+1))

SD_N_POST="$(find "$DECIDER_STATE" -mindepth 1 -maxdepth 1 -printf '.' | wc -c)"
[ "$SD_N_POST" -eq 3 ] || fail 413 "decider state-dir count changed by --dry-run"; PASS=$((PASS+1))
sha256sum "$BASELINE"      | awk '{print $1}' | grep -Exq "$EXP_BASELINE_SHA" || fail 414 "baseline mutated by --dry-run"; PASS=$((PASS+1))
sha256sum "$LAST_MANIFEST" | awk '{print $1}' | grep -Exq "$EXP_BASELINE_SHA" || fail 415 "last-manifest mutated by --dry-run"; PASS=$((PASS+1))

LAST_DECISION_SHA_AFTER="$(sha256sum "$LAST_DECISION" | awk '{print $1}')"
LAST_DECISION_MTIME_AFTER="$(stat -c '%Y' "$LAST_DECISION")"
[ "$LAST_DECISION_SHA_AFTER"   = "$LAST_DECISION_SHA_BEFORE" ]   || fail 416 "last-decision content changed by --dry-run"; PASS=$((PASS+1))
[ "$LAST_DECISION_MTIME_AFTER" = "$LAST_DECISION_MTIME_BEFORE" ] || fail 417 "last-decision mtime changed by --dry-run"; PASS=$((PASS+1))

[ ! -e "$LAST_ERROR" ] || fail 418 "last-error created by --dry-run"; PASS=$((PASS+1))
[ ! -e "$SENTINEL" ]   || fail 419 "sentinel created by --dry-run"; PASS=$((PASS+1))
assert_lock_free "$DECIDER_LOCK" 420; PASS=$((PASS+1))

# Phase 4B: Helper --self-test + --dry-run with full before/after (12 assertions)
HSD_BEFORE="$(hsd_digest)"
PCR11_MTIME_BEFORE="$(stat -c '%Y' "$PCR11_FILE")"
UKI_MTIME_BEFORE="$(stat -c '%Y' "$UKI_PATH")"
INITRAMFS_MTIME_BEFORE="$(stat -c '%Y' "$INITRAMFS_PATH")"
FAILURE_MARKER_BEFORE_EXISTS=0; [ -e "$FAILURE_MARKER" ] && FAILURE_MARKER_BEFORE_EXISTS=1
FAILURE_LOG_BEFORE_EXISTS=0;    [ -e "$FAILURE_LOG" ]    && FAILURE_LOG_BEFORE_EXISTS=1

HELP_VER="$("$HELPER" --version 2>&1 | head -1)"
echo "$HELP_VER" | grep -Fq "$EXP_HELPER_VERSION" || fail 430 "helper --version unexpected"; PASS=$((PASS+1))

ST_OUT="$(new_tmp)"; ST_RC=0
"$HELPER" --self-test >"$ST_OUT" 2>&1 || ST_RC=$?
cat "$ST_OUT"
[ "$ST_RC" -eq 0 ] || fail 440 "helper --self-test rc=$ST_RC"; PASS=$((PASS+1))

DR_OUT="$(new_tmp)"; DR_RC=0
"$HELPER" --dry-run >"$DR_OUT" 2>&1 || DR_RC=$?
head -80 "$DR_OUT"
[ "$DR_RC" -eq 0 ] || fail 450 "helper --dry-run rc=$DR_RC"; PASS=$((PASS+1))
grep -Eq "kernels enumerated \(2\):" "$DR_OUT" || fail 451 "helper --dry-run did not enumerate 2 kernels"; PASS=$((PASS+1))
grep -Fq "$EXP_KVER_BOOTED"    "$DR_OUT" || fail 452 "helper --dry-run missing booted kernel";    PASS=$((PASS+1))
grep -Fq "$EXP_KVER_SECONDARY" "$DR_OUT" || fail 453 "helper --dry-run missing secondary kernel"; PASS=$((PASS+1))

HSD_AFTER="$(hsd_digest)"
[ "$HSD_AFTER" = "$HSD_BEFORE" ] || fail 460 "helper state-dir digest changed across --dry-run"; PASS=$((PASS+1))

PCR11_MTIME_AFTER="$(stat -c '%Y' "$PCR11_FILE")"
[ "$PCR11_MTIME_AFTER" = "$PCR11_MTIME_BEFORE" ] || fail 461 "PCR 11 file mtime advanced across --dry-run"; PASS=$((PASS+1))

UKI_MTIME_AFTER="$(stat -c '%Y' "$UKI_PATH")"
[ "$UKI_MTIME_AFTER" = "$UKI_MTIME_BEFORE" ] || fail 462 "UKI mtime advanced across --dry-run"; PASS=$((PASS+1))

INITRAMFS_MTIME_AFTER="$(stat -c '%Y' "$INITRAMFS_PATH")"
[ "$INITRAMFS_MTIME_AFTER" = "$INITRAMFS_MTIME_BEFORE" ] || fail 463 "initramfs mtime advanced across --dry-run"; PASS=$((PASS+1))

FAILURE_MARKER_AFTER_EXISTS=0; [ -e "$FAILURE_MARKER" ] && FAILURE_MARKER_AFTER_EXISTS=1
FAILURE_LOG_AFTER_EXISTS=0;    [ -e "$FAILURE_LOG" ]    && FAILURE_LOG_AFTER_EXISTS=1
{ [ "$FAILURE_MARKER_AFTER_EXISTS" = "$FAILURE_MARKER_BEFORE_EXISTS" ] \
    && [ "$FAILURE_LOG_AFTER_EXISTS" = "$FAILURE_LOG_BEFORE_EXISTS" ]; } \
    || fail 464 "helper failure marker/log created or removed across --dry-run"
PASS=$((PASS+1))

assert_lock_free "$HELPER_LOCK" 465; PASS=$((PASS+1))

echo "=== Phase 4: live read-only probes: ${PASS}/58 cumulative PASS ==="
echo "=== B.2.4 Gate 1: ${PASS} assertions PASS ==="
EOF
```

Expected output (reference run, 2026-05-25): 58/58 PASS. Manifest-category intersection preview shows `dracut-network` with 72 hits in cat 02-dracut-modules and 0 in every other category. `dnf install --assumeno dracut-network` returns rc=1 with `Operation aborted by the user.`, plan lists 1 package + 0 deps. Decider `--dry-run` reports `baseline compare result: unchanged` and `would EXIT 0`. Helper `--self-test` emits all 13 preflight ok lines. Helper `--dry-run` enumerates 2 kernels and logs the `[DRY-RUN] would run:` sequence with scratch under `/run/tboot-helper-XXXXXX`. All before/after byte-stability assertions hold.

### Commands: Gate 2: pre-drift-transaction snapshot (snapshot gate; no validation harness)

VM 500: no mutation (storage-level snapshot without `--vmstate`; VM continues running). Host pve-host: one new snapshot entry; LVM thin-pool data delta advances.

Snapshot gates produce a single artefact (the snapshot itself) whose existence and lineage placement are the evidence. The questions a harness would answer, is the VM healthy, are the trust-chain shas frozen, is the state at expected closure values, are answered by the *adjacent* gates: Gate 1 immediately before, and Gate 3 immediately after. This matches how prior snapshot gates in this runbook are handled (B.2.3 Gate 7, B.2.1 Step 28 closing snapshot, Module 3 Step 26 pre-reboot snapshot). The evidence for Gate 2 is therefore (i) the snapshot command succeeds, (ii) the snapshot appears in `qm listsnapshot 500` parented correctly with `current` below, and (iii) VM 500 remains `status: running` per `qm status 500`.

If the host snapshot command fails, no new anchor was created; `module5-b2-3-validated` remains the rollback target. Do not proceed to Gate 3.

```bash
# Proxmox host pve-host as root

# Identity check
qm status 500

# Lineage check: parent must already exist
qm listsnapshot 500

# Create snapshot with descriptive metadata
DESC="B.2.4 Gate 2: pre-drift-transaction anchor. Frozen at B.2.3 closure values: decider sha 1e751109...e5cc05, helper sha 937afc7a...7125e44, predict sha b729ec27...91e3de, hook sha 5857e51d...20947e, prodrule sha dfbffe56...90d798, baseline sha 75dc4f56...0160, booted UKI sha b6002e66...7943, runtime PCR 11 A03EB49C...B16435DD. Gate 1 closed 58/58 PASS. Candidate for Gate 3: dracut-network (1 pkg, 0 deps, 72 files in /usr/lib/dracut/modules.d/). Parent: module5-b2-3-validated."

qm snapshot 500 module5-b2-4-pre-drift-transaction --description "$DESC"

# Verify visibility and lineage
qm listsnapshot 500
qm status 500
```

Expected output (reference run, 2026-05-25). `qm status 500` reports `status: running` both before and after. `qm snapshot 500 module5-b2-4-pre-drift-transaction …` returns rc=0. `qm listsnapshot 500` post-creation lists `module5-b2-4-pre-drift-transaction` parented to `module5-b2-3-validated`, with `current` immediately below it. VM continues running uninterrupted. No PASS counter is reported because no assertion harness is run.

### Commands: Gate 3: controlled live mutation `dnf install -y dracut-network`

One DNF transaction commits: install `dracut-network-107-8.fc43.x86_64`. Helper executes `dracut --force` and `kernel-install add` for both enumerated kernels; the 80-tpm2-sign hook re-signs both UKIs; predictor refreshes `/root/tboot-lab/state/expected-pcr11-after-hook-uki.txt`; baseline manifest is atomically replaced; sentinel is written then cleared; `last-decision` is rewritten with `decision=helper-success`. Helper scratch dir created under `/run/tboot-helper-XXXXXX` and removed by the helper's trap. Runtime PCR 11 unchanged (no reboot: correct). Trust-chain script/rule shas remain byte-stable.

Rollback anchor: `module5-b2-4-pre-drift-transaction`. On any Stop-Condition failure, `qm rollback 500 module5-b2-4-pre-drift-transaction` on host pve-host returns the VM to pre-Gate-3 state cleanly.

**Reference-run outcome (2026-05-25).** The Gate 3 reference run used the rev-3 harness, which contained a regex bug in the helper-start assertion at position 411: the harness checked for `mode=normal` in the helper journal, but the helper logs its production code path as `mode=production` (the decider logs `mode=normal`; the two components use different terminology for the same end-to-end execution: see `06F` K.1). The harness halted at FAIL 411. The mutation itself completed correctly. The captured journal output from the reference run shows the full positive drift path: decider fired exactly once with `mode=normal`, drift detected, sentinel written `(reason=drift-detected)`, helper invoked, both kernels processed by `dracut` + `kernel-install add`, 80-tpm2-sign signed both UKIs (`ukify-native signed UKI ready (1 signature)` per kernel), helper exited 0, sentinel cleared, baseline atomically replaced, `last-decision=helper-success`, stored PCR 11 prediction advanced to `28E66CE2…C90F`.

Gate 3 was closed by running the post-hoc continuation verifier from `06F` K.1 against the now-mutated live state. The continuation verifier passed **43/43 assertions**, covering: package installed; decider evidence (journal non-empty, `mode=normal` start count = 1, drift reported, INVOKE logged, `helper exited 0`, sentinel cleared, no error-like lines); helper evidence (journal non-empty, `mode=production` start count = 1, both kernels enumerated, `dracut: ok` + `kernel-install add: ok` for each, predict OK, completion line present, no error-like lines); 80-tpm2-sign evidence (journal non-empty, exactly one `starting for kernel <kver>` per kernel, `ukify-native signed UKI ready` per kernel, no error-like lines); state transitions (`last-decision=helper-success`, baseline `75dc4f56…0160` → `6b032bd8…2a47`, last-manifest equals new baseline, sentinel absent, last-error absent, success marker fresh, failure marker/log absent); boot-artefact transitions (booted UKI sha `b6002e66…7943` → `0ac32c8b…6949`, both UKIs sbverify-OK, stored PCR 11 value `28E66CE2…C90F`, runtime PCR 11 unchanged at `A03EB49C…35DD`); trust-chain (decider/helper/predict/hook/production-rule shas byte-stable; exactly 1 active `*.actions`; locks not actively held).

**Replay form: corrected 61-assertion harness for future Gate 3 reruns.** This harness is the canonical replay form, with the K.1 `mode=production` fix applied at line 411. It was not executed against the 2026-05-25 reference-run state. Intended use: future Gate 3 reruns from a clean `qm rollback 500 module5-b2-4-pre-drift-transaction`. Running it now against the already-mutated state would fail at every assertion comparing to `EXP_BASELINE_SHA` (the pre-Gate-3 baseline), because the baseline has already advanced.

```bash
bash <<'EOF'
set -u

# ============================================================================
# B.2.4 Gate 3: controlled live mutation: dnf install -y dracut-network
# REPLAY FORM with K.1 fix applied (helper mode=production assertion).
# Intended for use after qm rollback 500 module5-b2-4-pre-drift-transaction.
# ============================================================================

EXP_DECIDER_SHA="1e75110935a72fc9433c9b4bf5688d26bc6d3bd1e02f4388b5e1f32ff1e5cc05"
EXP_BASELINE_SHA="75dc4f5651fc8996d39b545ac0bf7244a2a5888403b0ba0cd448a671daf80160"
EXP_PRODRULE_SHA="dfbffe56fa95f4e0231569bb3a1aabc8952b9f99c9d290e4f8fb1005ef90d798"
EXP_HOOK_SHA="5857e51d5551e05a7ca384f71529ec2fef07f5a6dbe40df70c6bbe49d720947e"
EXP_HELPER_SHA="937afc7a0d199bf7b7d1a5f5bcccf45c37c0f4f7dd0ad924b2567c4da7125e44"
EXP_PREDICT_SHA="b729ec271dd76cd8b4581f16c86e50fc28b947f2a1de5a7e16c6d5828591e3de"

PRE_UKI_SHA="b6002e666afdf71e5a083311295ac6a5f3ef3b443ceb4ec8c16c9ddb41077943"
PRE_PCR11="A03EB49C9335D721758AE8AA9C34DA87664C5CAF9FAD7C314C68C038B16435DD"

EXP_MID="0123456789abcdef0123456789abcdef"
EXP_KVER_BOOTED="6.19.14-200.fc43.x86_64"
EXP_KVER_SECONDARY="6.17.1-300.fc43.x86_64"

UKI_PATH="/boot/efi/EFI/Linux/${EXP_MID}-${EXP_KVER_BOOTED}.efi"
UKI_PATH_SECONDARY="/boot/efi/EFI/Linux/${EXP_MID}-${EXP_KVER_SECONDARY}.efi"
INITRAMFS_PATH="/boot/initramfs-${EXP_KVER_BOOTED}.img"
PCR11_FILE="/root/tboot-lab/state/expected-pcr11-after-hook-uki.txt"

PRODRULE="/etc/dnf/libdnf5-plugins/actions.d/50-tboot-posttrans.actions"
ACTIONS_DIR="/etc/dnf/libdnf5-plugins/actions.d"

DECIDER="/usr/local/sbin/tboot-dnf-posttrans"
HELPER="/usr/local/sbin/tboot-dnf-helper"
PREDICT="/usr/local/sbin/tboot-predict-pcr11"
HOOK="/etc/kernel/install.d/80-tpm2-sign.install"

DECIDER_STATE="/var/lib/tboot-dnf-posttrans"
BASELINE="${DECIDER_STATE}/boot-input-manifest.baseline"
LAST_DECISION="${DECIDER_STATE}/last-decision"
LAST_MANIFEST="${DECIDER_STATE}/last-computed-manifest"
LAST_ERROR="${DECIDER_STATE}/last-error"

HELPER_STATE="/var/lib/tboot-dnf-helper"
SENTINEL="${HELPER_STATE}/UNSAFE-TO-REBOOT"
SUCCESS_MARKER="${HELPER_STATE}/last-success"
FAILURE_MARKER="${HELPER_STATE}/last-failure"
FAILURE_LOG="${HELPER_STATE}/last-failure.log"

DECIDER_LOCK="/run/tboot-dnf-posttrans.lock"
HELPER_LOCK="/run/tboot-dnf-helper.lock"

CANDIDATE="dracut-network"

TMP_FILES=()
new_tmp() { local t; t="$(mktemp)"; TMP_FILES+=("$t"); printf '%s\n' "$t"; }
_cleanup() { local f; for f in "${TMP_FILES[@]:-}"; do rm -f "$f" 2>/dev/null || true; done; }
trap _cleanup EXIT HUP INT TERM

PASS=0
fail() { echo "FAIL $1: $2"; exit "$1"; }

assert_lock_free() {
    local lock="$1" code="$2"
    if [ -e "$lock" ] || [ -L "$lock" ]; then
        [ -f "$lock" ] && [ ! -L "$lock" ] || fail "$code" "$lock not regular non-symlink"
        ( exec 9<>"$lock"; flock -n 9 ) || fail "$code" "$lock actively held"
    fi
}

hsd_digest() {
    if [ ! -d "$HELPER_STATE" ]; then echo "no-state-dir"; return 0; fi
    find "$HELPER_STATE" -maxdepth 1 -type f -printf '%f\n' 2>/dev/null \
        | sort \
        | while read -r f; do sha256sum "$HELPER_STATE/$f" 2>/dev/null; done \
        | sha256sum | awk '{print $1}'
}

journal_has_error() {
    grep -Eiq '(^|[[:space:]])(err|error|fail|failed|failure)([[:space:]:]|$)' "$1"
}
show_error_lines() {
    grep -Ein '(^|[[:space:]])(err|error|fail|failed|failure)([[:space:]:]|$)' "$1" || true
}

# --- Phase 1: pre-mutation reference capture (17 assertions) ---
[ "$(id -u)" -eq 0 ] || fail 101 "must run as root"; PASS=$((PASS+1))
sha256sum "$DECIDER"  | awk '{print $1}' | grep -Exq "$EXP_DECIDER_SHA"  || fail 110 "decider sha drift";    PASS=$((PASS+1))
sha256sum "$HELPER"   | awk '{print $1}' | grep -Exq "$EXP_HELPER_SHA"   || fail 111 "helper sha drift";     PASS=$((PASS+1))
sha256sum "$PREDICT"  | awk '{print $1}' | grep -Exq "$EXP_PREDICT_SHA"  || fail 112 "predict sha drift";    PASS=$((PASS+1))
sha256sum "$HOOK"     | awk '{print $1}' | grep -Exq "$EXP_HOOK_SHA"     || fail 113 "hook sha drift";       PASS=$((PASS+1))
sha256sum "$PRODRULE" | awk '{print $1}' | grep -Exq "$EXP_PRODRULE_SHA" || fail 114 "production rule drift"; PASS=$((PASS+1))
sha256sum "$BASELINE" | awk '{print $1}' | grep -Exq "$EXP_BASELINE_SHA" || fail 115 "baseline sha drift";   PASS=$((PASS+1))
sha256sum "$UKI_PATH" | awk '{print $1}' | grep -Exq "$PRE_UKI_SHA"      || fail 116 "booted UKI sha drift"; PASS=$((PASS+1))
sha256sum "$LAST_MANIFEST" | awk '{print $1}' | grep -Exq "$EXP_BASELINE_SHA" || fail 117 "last-manifest sha drift"; PASS=$((PASS+1))

PRE_LD_SHA="$(sha256sum "$LAST_DECISION" | awk '{print $1}')"
PRE_LD_MTIME="$(stat -c '%Y' "$LAST_DECISION")"
PRE_LD_DECISION="$(awk -F= '$1=="decision"{print $2}' "$LAST_DECISION")"
PRE_LD_EPOCH="$(awk -F= '$1=="epoch"{print $2}' "$LAST_DECISION")"
[ "$PRE_LD_DECISION" = "unchanged" ] || fail 120 "pre-Gate-3 last-decision != unchanged"; PASS=$((PASS+1))
[[ "$PRE_LD_EPOCH" =~ ^[0-9]+$ ]]   || fail 121 "last-decision.epoch not numeric"; PASS=$((PASS+1))

[ ! -e "$LAST_ERROR" ] || fail 130 "last-error present pre-Gate-3"; PASS=$((PASS+1))
[ ! -e "$SENTINEL" ]   || fail 131 "sentinel present pre-Gate-3";   PASS=$((PASS+1))

mapfile -t ACTIONS_PRE < <(find "$ACTIONS_DIR" -maxdepth 1 -type f -name '*.actions' -printf '%p\n' | sort)
{ [ "${#ACTIONS_PRE[@]}" -eq 1 ] && [ "${ACTIONS_PRE[0]}" = "$PRODRULE" ]; } || fail 140 "unexpected *.actions entries pre-Gate-3"
PASS=$((PASS+1))

assert_lock_free "$DECIDER_LOCK" 150
assert_lock_free "$HELPER_LOCK"  151
PASS=$((PASS+1))

rpm -q "$CANDIDATE" >/dev/null 2>&1 && fail 160 "$CANDIDATE already installed" || true; PASS=$((PASS+1))

PRE_PCR11_VALUE="$(tr -d '[:space:]' <"$PCR11_FILE" | awk '{print toupper($0)}')"
[ "$PRE_PCR11_VALUE" = "$PRE_PCR11" ] || fail 170 "stored PCR 11 drift pre-Gate-3"; PASS=$((PASS+1))

PRE_UKI_MTIME="$(stat -c '%Y' "$UKI_PATH")"
PRE_INITRAMFS_MTIME="$(stat -c '%Y' "$INITRAMFS_PATH")"
PRE_PCR11_MTIME="$(stat -c '%Y' "$PCR11_FILE")"
PRE_HELPER_DIGEST="$(hsd_digest)"

echo "=== Phase 1: pre-mutation: ${PASS}/17 PASS ==="

# --- Phase 2: mutation (1 assertion) ---
PRE_TS="$(date -Iseconds)"; PRE_EPOCH="$(date +%s)"
sleep 1
DNF_OUT="$(new_tmp)"; DNF_RC=0
dnf install -y "$CANDIDATE" >"$DNF_OUT" 2>&1 || DNF_RC=$?
POST_EPOCH="$(date +%s)"; POST_TS="$(date -Iseconds)"
echo "--- dnf output (rc=$DNF_RC, duration $((POST_EPOCH-PRE_EPOCH))s) ---"
cat "$DNF_OUT"
[ "$DNF_RC" -eq 0 ] || fail 200 "dnf install rc=$DNF_RC"; PASS=$((PASS+1))
echo "=== Phase 2: DNF: ${PASS}/18 cumulative PASS ==="

# --- Phase 3: candidate installed (1 assertion) ---
rpm -q "$CANDIDATE" >/dev/null 2>&1 || fail 301 "$CANDIDATE not installed after rc=0"
INSTALLED_NEVRA="$(rpm -q "$CANDIDATE")"; echo "installed: $INSTALLED_NEVRA"
PASS=$((PASS+1))

# --- Phase 4: journal evidence (14 assertions; K.1 fix at 411) ---
DEC_JOURNAL="$(new_tmp)"; HELP_JOURNAL="$(new_tmp)"; HOOK_JOURNAL="$(new_tmp)"
journalctl -t tboot-dnf-posttrans --since "$PRE_TS" --no-pager -o cat >"$DEC_JOURNAL"  2>&1 || true
journalctl -t tboot-dnf-helper    --since "$PRE_TS" --no-pager -o cat >"$HELP_JOURNAL" 2>&1 || true
journalctl -t 80-tpm2-sign        --since "$PRE_TS" --no-pager -o cat >"$HOOK_JOURNAL" 2>&1 || true

[ -s "$DEC_JOURNAL" ] || fail 401 "decider journal empty"; PASS=$((PASS+1))
DEC_START_COUNT="$(grep -Ec 'mode=normal; acquiring decider lock' "$DEC_JOURNAL" || true)"
[ "$DEC_START_COUNT" -eq 1 ] || fail 402 "decider mode=normal count = $DEC_START_COUNT"; PASS=$((PASS+1))
grep -Eq 'baseline compare result: drift'    "$DEC_JOURNAL" || fail 403 "decider did not report drift"; PASS=$((PASS+1))
grep -Eq 'baseline compare result: unchanged' "$DEC_JOURNAL" && fail 404 "decider logged 'unchanged'" || true; PASS=$((PASS+1))
grep -Eq 'helper invocation decision: INVOKE' "$DEC_JOURNAL" || fail 405 "decider did not log INVOKE"; PASS=$((PASS+1))
grep -Eq 'helper exited 0'                     "$DEC_JOURNAL" || fail 406 "decider did not record helper exit 0"; PASS=$((PASS+1))
if journal_has_error "$DEC_JOURNAL"; then show_error_lines "$DEC_JOURNAL"; fail 407 "decider has errors"; fi; PASS=$((PASS+1))

[ -s "$HELP_JOURNAL" ] || fail 410 "helper journal empty"; PASS=$((PASS+1))

# 06F K.1 fix: helper logs production code path as mode=production, NOT mode=normal.
HELP_START_COUNT="$(grep -Ec '(^|[[:space:]])starting version=.*mode=production' "$HELP_JOURNAL" || true)"
[ "$HELP_START_COUNT" -eq 1 ] || fail 411 "helper production-mode start lines = $HELP_START_COUNT"
PASS=$((PASS+1))

grep -Fq "kernels enumerated (2):" "$HELP_JOURNAL" || fail 412 "helper did not enumerate 2 kernels"; PASS=$((PASS+1))
grep -Fq "$EXP_KVER_BOOTED"    "$HELP_JOURNAL" || fail 413 "helper missing booted kernel"; PASS=$((PASS+1))
grep -Fq "$EXP_KVER_SECONDARY" "$HELP_JOURNAL" || fail 414 "helper missing secondary kernel"; PASS=$((PASS+1))
if journal_has_error "$HELP_JOURNAL"; then show_error_lines "$HELP_JOURNAL"; fail 415 "helper has errors"; fi; PASS=$((PASS+1))

[ -s "$HOOK_JOURNAL" ] || fail 420 "80-tpm2-sign journal empty"; PASS=$((PASS+1))
HOOK_BOOTED_COUNT="$(grep -Fc    "starting for kernel ${EXP_KVER_BOOTED}"    "$HOOK_JOURNAL" || true)"
[ "$HOOK_BOOTED_COUNT" -eq 1 ] || fail 421 "hook booted count = $HOOK_BOOTED_COUNT"; PASS=$((PASS+1))
HOOK_SECONDARY_COUNT="$(grep -Fc "starting for kernel ${EXP_KVER_SECONDARY}" "$HOOK_JOURNAL" || true)"
[ "$HOOK_SECONDARY_COUNT" -eq 1 ] || fail 422 "hook secondary count = $HOOK_SECONDARY_COUNT"; PASS=$((PASS+1))
if journal_has_error "$HOOK_JOURNAL"; then show_error_lines "$HOOK_JOURNAL"; fail 423 "hook has errors"; fi; PASS=$((PASS+1))

echo "=== Phase 4: journals: ${PASS}/36 cumulative PASS ==="

# --- Phase 5: state transitions (10 assertions) ---
POST_LD_SHA="$(sha256sum "$LAST_DECISION" | awk '{print $1}')"
POST_LD_MTIME="$(stat -c '%Y' "$LAST_DECISION")"
POST_LD_DECISION="$(awk -F= '$1=="decision"{print $2}' "$LAST_DECISION")"
POST_LD_EPOCH="$(awk -F= '$1=="epoch"{print $2}' "$LAST_DECISION")"
POST_LD_DETAIL="$(awk -F= '$1=="detail"{ $1=""; sub(/^=/,""); print; }' "$LAST_DECISION")"
[ "$POST_LD_SHA"   != "$PRE_LD_SHA" ]    || fail 501 "last-decision sha unchanged"; PASS=$((PASS+1))
[ "$POST_LD_MTIME" -gt "$PRE_LD_MTIME" ] || fail 502 "last-decision mtime did not advance"; PASS=$((PASS+1))
[ "$POST_LD_DECISION" = "helper-success" ] || fail 503 "last-decision = '$POST_LD_DECISION'"; PASS=$((PASS+1))
[[ "$POST_LD_EPOCH" =~ ^[0-9]+$ ]] || fail 504 "epoch not numeric"
[ "$POST_LD_EPOCH" -gt "$PRE_LD_EPOCH" ] || fail 505 "epoch did not advance"; PASS=$((PASS+1))
POST_BASELINE_SHA="$(sha256sum "$BASELINE" | awk '{print $1}')"
[ "$POST_BASELINE_SHA" != "$EXP_BASELINE_SHA" ] || fail 506 "baseline did not advance"; PASS=$((PASS+1))
POST_LM_SHA="$(sha256sum "$LAST_MANIFEST" | awk '{print $1}')"
[ "$POST_LM_SHA" = "$POST_BASELINE_SHA" ] || fail 507 "last-manifest != new baseline"; PASS=$((PASS+1))
[ ! -e "$SENTINEL" ]  || fail 508 "sentinel remains"; PASS=$((PASS+1))
[ ! -e "$LAST_ERROR" ] || fail 509 "last-error present"; PASS=$((PASS+1))
[ -f "$SUCCESS_MARKER" ] || fail 510 "success marker absent"
SM_MTIME="$(stat -c '%Y' "$SUCCESS_MARKER")"
[ "$SM_MTIME" -ge "$PRE_EPOCH" ] || fail 511 "success marker predates PRE_EPOCH"; PASS=$((PASS+1))
{ [ ! -e "$FAILURE_MARKER" ] && [ ! -e "$FAILURE_LOG" ]; } || fail 512 "failure marker/log present"; PASS=$((PASS+1))

echo "=== Phase 5: state transitions: ${PASS}/46 cumulative PASS ==="

# --- Phase 6: UKI / initramfs / PCR 11 (7 assertions; PCR value info-only) ---
POST_UKI_SHA="$(sha256sum "$UKI_PATH" | awk '{print $1}')"
POST_UKI_MTIME="$(stat -c '%Y' "$UKI_PATH")"
UKI_SHA_CHANGED=0; UKI_MTIME_ADVANCED=0
[ "$POST_UKI_SHA" != "$PRE_UKI_SHA" ] && UKI_SHA_CHANGED=1
[ "$POST_UKI_MTIME" -gt "$PRE_UKI_MTIME" ] && UKI_MTIME_ADVANCED=1
{ [ "$UKI_SHA_CHANGED" = 1 ] || [ "$UKI_MTIME_ADVANCED" = 1 ]; } || fail 601 "UKI sha AND mtime both unchanged"; PASS=$((PASS+1))
{ [ -f "$UKI_PATH" ] && [ ! -L "$UKI_PATH" ]; } || fail 602 "booted UKI not regular non-symlink"; PASS=$((PASS+1))
sbverify --cert /etc/uefi-keys/db.crt "$UKI_PATH" >/dev/null 2>&1 || fail 603 "booted UKI sbverify failed"; PASS=$((PASS+1))
POST_INITRAMFS_MTIME="$(stat -c '%Y' "$INITRAMFS_PATH")"
[ "$POST_INITRAMFS_MTIME" -gt "$PRE_INITRAMFS_MTIME" ] || fail 604 "initramfs mtime did not advance"; PASS=$((PASS+1))
[ -f "$UKI_PATH_SECONDARY" ] && [ ! -L "$UKI_PATH_SECONDARY" ] || fail 605 "secondary UKI missing"
sbverify --cert /etc/uefi-keys/db.crt "$UKI_PATH_SECONDARY" >/dev/null 2>&1 || fail 606 "secondary UKI sbverify failed"; PASS=$((PASS+1))
POST_PCR11_MTIME="$(stat -c '%Y' "$PCR11_FILE")"
[ "$POST_PCR11_MTIME" -gt "$PRE_PCR11_MTIME" ] || fail 607 "PCR 11 file mtime did not advance"; PASS=$((PASS+1))
RT_PCR11="$(awk '{gsub(/^0x/,""); print toupper($0)}' /sys/class/tpm/tpm0/pcr-sha256/11)"
[ "$RT_PCR11" = "$PRE_PCR11" ] || fail 608 "runtime PCR 11 changed without reboot"; PASS=$((PASS+1))
POST_PCR11_VALUE="$(tr -d '[:space:]' <"$PCR11_FILE" | awk '{print toupper($0)}')"
echo "PCR 11 value PRE/POST: $PRE_PCR11_VALUE / $POST_PCR11_VALUE   (informational per 06F K.3)"
echo "=== Phase 6: UKI/initramfs/PCR 11: ${PASS}/53 cumulative PASS ==="

# --- Phase 7: trust-chain + locks (8 assertions) ---
sha256sum "$DECIDER"  | awk '{print $1}' | grep -Exq "$EXP_DECIDER_SHA"  || fail 701 "decider sha drifted"; PASS=$((PASS+1))
sha256sum "$HELPER"   | awk '{print $1}' | grep -Exq "$EXP_HELPER_SHA"   || fail 702 "helper sha drifted";  PASS=$((PASS+1))
sha256sum "$PREDICT"  | awk '{print $1}' | grep -Exq "$EXP_PREDICT_SHA"  || fail 703 "predict sha drifted"; PASS=$((PASS+1))
sha256sum "$HOOK"     | awk '{print $1}' | grep -Exq "$EXP_HOOK_SHA"     || fail 704 "hook sha drifted";    PASS=$((PASS+1))
sha256sum "$PRODRULE" | awk '{print $1}' | grep -Exq "$EXP_PRODRULE_SHA" || fail 705 "production rule sha drifted"; PASS=$((PASS+1))
mapfile -t ACTIONS_POST < <(find "$ACTIONS_DIR" -maxdepth 1 -type f -name '*.actions' -printf '%p\n' | sort)
{ [ "${#ACTIONS_POST[@]}" -eq 1 ] && [ "${ACTIONS_POST[0]}" = "$PRODRULE" ]; } || fail 710 "unexpected *.actions"
PASS=$((PASS+1))
assert_lock_free "$DECIDER_LOCK" 720; PASS=$((PASS+1))
assert_lock_free "$HELPER_LOCK"  721; PASS=$((PASS+1))
echo "=== Phase 7: trust-chain + locks: ${PASS}/61 cumulative PASS ==="

POST_HELPER_DIGEST="$(hsd_digest)"
echo "=== B.2.4 Gate 3 (replay form): ${PASS} assertions PASS ==="
echo "Gate 3 complete; stop for review. Do not reboot yet."
EOF
```

Expected output of the replay form (only valid against a freshly-rolled-back clean state): 61/61 PASS, with the K.1 helper-start regex fix at assertion 411 matching `mode=production`. State transitions as documented: baseline `75dc4f56…0160` → `6b032bd8…2a47`, booted UKI `b6002e66…7943` → `0ac32c8b…6949`, stored PCR 11 `A03EB49C…35DD` → `28E66CE2…C90F`, runtime PCR 11 unchanged.

### Stop condition

Halt on any FAIL emitted by the Gate 1 or Gate 3 harnesses. Gate 2 fails if `qm snapshot` returns non-zero or if post-creation `qm listsnapshot 500` does not show `module5-b2-4-pre-drift-transaction` parented to `module5-b2-3-validated` with `current` below. For a Gate 3 FAIL 411 specifically, see the Recovery procedures table and `06F` K.1: read the helper journal manually before any rollback; the mutation likely succeeded and the post-hoc continuation verifier from `06F` K.1 is the canonical recovery path.

### Closure

VM 500 is left in the Gate 3 post-mutation state. `dracut-network` remains installed. Baseline manifest reflects the post-drift content. PCR 11 prediction file holds the new value `28E66CE2…C90F`. Sentinel absent. Trust-chain script and rule shas byte-stable. Booted UKI on disk has been re-signed forward; runtime PCR 11 at Step 34 close still reflects the pre-Gate-3 boot (correct; no reboot in Step 34). Continue with Step 35 (B.2.4 Gates 4–5: pre-reboot cross-validation + pre-reboot snapshot), then Step 36 (Gates 6A–6B: reboot + post-reboot validation), then Step 37 (Gate 7: closure snapshot + ESP audit + staged discipline-gate lift).

✅ **Block B.2.4 Gates 1–3 closed.** Gate 1 via 58/58 harness pass; Gate 2 via snapshot creation + lineage verification (no harness); Gate 3 via post-hoc continuation verifier at 43/43 PASS after a harness regex bug (`06F` K.1) interrupted the reference-run harness. **B.2.4 Gates 4–7: closed via Steps 35–37 below.** Blind `dnf update` / `dnf upgrade` lifts after Step 37 closure under staged discipline (see Step 37). `dnf upgrade systemd*` remains separately blocked (B.4 / B.5 scope).

---

## Step 35: Block B.2.4 Gates 4–5: pre-reboot read-only cross-validation + pre-reboot snapshot

**Goal:** prove on-disk consistency of the rebuilt UKI without mutating any monitored state, then take the pre-reboot snapshot. Gate 4 must independently reproduce the stored PCR 11 prediction from the on-disk rebuilt UKI byte-for-byte. Gate 5 is snapshot-only, no validation harness.
**Prerequisites:** Step 34 (B.2.4 Gates 1–3) closed; VM in the post-Gate-3 mutation state with `dracut-network-107-8.fc43.x86_64` installed, baseline sha `6b032bd8…2a47`, stored PCR 11 prediction `28E66CE2…C90F`, runtime PCR 11 still pre-reboot `A03EB49C…35DD`, current UKI sha `0ac32c8b…6949`.
**Where to run:** Gate 4 inside VM 500 as root (tmux). Gate 5 on Proxmox host pve-host as root.

### Commands

**Gate 4: pre-reboot read-only cross-validation harness (36/36 PASS target).** Read-only with respect to all monitored state: trust-chain artifacts, decider state-dir, helper state-dir, ESP UKIs, `/etc/dnf/libdnf5-plugins/actions.d/`, `/root/tboot-lab/state/expected-pcr11-after-hook-uki.txt`, runtime PCR 11. The `tboot-predict-pcr11` invocation scratches to `/run` tmpfs and shreds on exit (no `--store` flag → predictor's `STORE=0` default; STATE_FILE never opened for write). `assert_lock_free` opens fd 9 briefly and never unlinks. No DNF, no kernel-install, no dracut, no `bootctl set-default`, no `systemd-cryptenroll`, no `ukify`, no `objcopy --add-section`, no reboot, no `qm snapshot`.

```bash
# Run on VM 500 as root (tmux)
bash <<'EOF'
set -u

# ============================================================================
# B.2.4 Gate 4: pre-reboot read-only cross-validation
# Proves that the rebuilt on-disk UKI is internally consistent and that the
# stored PCR 11 prediction is reproducible from it, without mutating any
# monitored state.
# ============================================================================

EXP_DECIDER_SHA="1e75110935a72fc9433c9b4bf5688d26bc6d3bd1e02f4388b5e1f32ff1e5cc05"
EXP_HELPER_SHA="937afc7a0d199bf7b7d1a5f5bcccf45c37c0f4f7dd0ad924b2567c4da7125e44"
EXP_PREDICT_SHA="b729ec271dd76cd8b4581f16c86e50fc28b947f2a1de5a7e16c6d5828591e3de"
EXP_HOOK_SHA="5857e51d5551e05a7ca384f71529ec2fef07f5a6dbe40df70c6bbe49d720947e"
EXP_PRODRULE_SHA="dfbffe56fa95f4e0231569bb3a1aabc8952b9f99c9d290e4f8fb1005ef90d798"

EXP_BASELINE_SHA="6b032bd81a7f3a57b8350c76cf690deb3a942704a76455b881f7a5d98df82a47"
EXP_UKI_SHA="0ac32c8b98428f9a44339a5991eb86e2704986e84ce6e09ae9721bba41726949"
EXP_STORED_PCR11="28E66CE224C9BB0711059470D77103A5AB5A5C1674C03D1BA1563C2FEE52C90F"
EXP_RUNTIME_PCR11="A03EB49C9335D721758AE8AA9C34DA87664C5CAF9FAD7C314C68C038B16435DD"

EXP_MID="0123456789abcdef0123456789abcdef"
EXP_KVER_BOOTED="6.19.14-200.fc43.x86_64"
EXP_KVER_SECONDARY="6.17.1-300.fc43.x86_64"
EXP_DEFAULT_ENTRY="${EXP_MID}-${EXP_KVER_BOOTED}.efi"
VMA_THRESHOLD="0x180000000"

CANDIDATE="dracut-network"

DECIDER="/usr/local/sbin/tboot-dnf-posttrans"
HELPER="/usr/local/sbin/tboot-dnf-helper"
PREDICT="/usr/local/sbin/tboot-predict-pcr11"
HOOK="/etc/kernel/install.d/80-tpm2-sign.install"
PRODRULE="/etc/dnf/libdnf5-plugins/actions.d/50-tboot-posttrans.actions"
ACTIONS_DIR="/etc/dnf/libdnf5-plugins/actions.d"

DECIDER_STATE="/var/lib/tboot-dnf-posttrans"
BASELINE="${DECIDER_STATE}/boot-input-manifest.baseline"
LAST_DECISION="${DECIDER_STATE}/last-decision"
LAST_MANIFEST="${DECIDER_STATE}/last-computed-manifest"
LAST_ERROR="${DECIDER_STATE}/last-error"

HELPER_STATE="/var/lib/tboot-dnf-helper"
SENTINEL="${HELPER_STATE}/UNSAFE-TO-REBOOT"
FAILURE_MARKER="${HELPER_STATE}/last-failure"
FAILURE_LOG="${HELPER_STATE}/last-failure.log"

DECIDER_LOCK="/run/tboot-dnf-posttrans.lock"
HELPER_LOCK="/run/tboot-dnf-helper.lock"

UKI_PATH="/boot/efi/EFI/Linux/${EXP_MID}-${EXP_KVER_BOOTED}.efi"
UKI_PATH_SECONDARY="/boot/efi/EFI/Linux/${EXP_MID}-${EXP_KVER_SECONDARY}.efi"
PCR11_FILE="/root/tboot-lab/state/expected-pcr11-after-hook-uki.txt"
DB_CRT="/etc/uefi-keys/db.crt"

TMP_FILES=()
new_tmp()  { local t; t="$(mktemp)"; TMP_FILES+=("$t"); printf '%s\n' "$t"; }
_cleanup() { local f; for f in "${TMP_FILES[@]:-}"; do rm -f "$f" 2>/dev/null || true; done; }
trap _cleanup EXIT HUP INT TERM

PASS=0
fail() { echo "FAIL $1: $2"; exit "$1"; }

# 06F J.3 lock probe: non-blocking flock on fd 9.
assert_lock_free() {
    local lock="$1" code="$2"
    if [ -e "$lock" ] || [ -L "$lock" ]; then
        { [ -f "$lock" ] && [ ! -L "$lock" ]; } || fail "$code" "$lock not regular non-symlink"
        ( exec 9<>"$lock"; flock -n 9 ) || fail "$code" "$lock actively held"
    fi
}

# 06F K.1 broad-error pattern compatible with `journalctl -o cat`.
journal_has_error() {
    grep -Eiq '(^|[[:space:]])(err|error|fail|failed|failure)([[:space:]:]|$)' "$1"
}
show_error_lines() {
    grep -Ein '(^|[[:space:]])(err|error|fail|failed|failure)([[:space:]:]|$)' "$1" || true
}

uki_has_pcr_sections() {
    objdump -h "$1" 2>/dev/null \
        | awk '$2==".pcrpkey"{p=1} $2==".pcrsig"{s=1} END{exit !(p && s)}'
}

uki_vma_sane() {
    objdump -h "$1" 2>/dev/null \
        | awk -v threshold="$2" '
            /^[[:space:]]+[0-9]+[[:space:]]+\./ {
                vma = strtonum("0x" $4)
                if (vma >= strtonum(threshold)) {
                    printf "anomalous VMA: %s at 0x%s\n", $2, $4 > "/dev/stderr"
                    bad = 1
                }
            }
            END { exit !!bad }
        '
}

# Phase 1 (1)
[ "$(id -u)" -eq 0 ] || fail 101 "must run as root"; PASS=$((PASS+1))

# Phase 2 (5): trust-chain byte-stability
sha256sum "$DECIDER"  | awk '{print $1}' | grep -Exq "$EXP_DECIDER_SHA"  || fail 110 "decider sha drift";          PASS=$((PASS+1))
sha256sum "$HELPER"   | awk '{print $1}' | grep -Exq "$EXP_HELPER_SHA"   || fail 111 "helper sha drift";           PASS=$((PASS+1))
sha256sum "$PREDICT"  | awk '{print $1}' | grep -Exq "$EXP_PREDICT_SHA"  || fail 112 "predict sha drift";          PASS=$((PASS+1))
sha256sum "$HOOK"     | awk '{print $1}' | grep -Exq "$EXP_HOOK_SHA"     || fail 113 "hook sha drift";             PASS=$((PASS+1))
sha256sum "$PRODRULE" | awk '{print $1}' | grep -Exq "$EXP_PRODRULE_SHA" || fail 114 "production rule sha drift";  PASS=$((PASS+1))

# Phase 3 (1): exactly one active *.actions == PRODRULE
mapfile -t ACTIONS_ACTIVE < <(find "$ACTIONS_DIR" -maxdepth 1 -type f -name '*.actions' -printf '%p\n' | sort)
{ [ "${#ACTIONS_ACTIVE[@]}" -eq 1 ] && [ "${ACTIONS_ACTIVE[0]}" = "$PRODRULE" ]; } \
    || fail 120 "unexpected *.actions entries (count=${#ACTIONS_ACTIVE[@]}; first='${ACTIONS_ACTIVE[0]:-}')"
PASS=$((PASS+1))

# Phase 4 (4): decider state invariants
sha256sum "$BASELINE"      | awk '{print $1}' | grep -Exq "$EXP_BASELINE_SHA" || fail 130 "baseline sha != post-Gate-3 expected"; PASS=$((PASS+1))
sha256sum "$LAST_MANIFEST" | awk '{print $1}' | grep -Exq "$EXP_BASELINE_SHA" || fail 131 "last-manifest sha != current baseline"; PASS=$((PASS+1))
LD_DECISION="$(awk -F= '$1=="decision"{print $2}' "$LAST_DECISION")"
[ "$LD_DECISION" = "helper-success" ] || fail 132 "last-decision.decision='$LD_DECISION' (expected helper-success)"; PASS=$((PASS+1))
[ ! -e "$LAST_ERROR" ] || fail 133 "decider last-error present"; PASS=$((PASS+1))

# Phase 5 (3): helper state invariants
[ ! -e "$SENTINEL" ]       || fail 140 "UNSAFE-TO-REBOOT sentinel present"; PASS=$((PASS+1))
[ ! -e "$FAILURE_MARKER" ] || fail 141 "helper failure marker present";     PASS=$((PASS+1))
[ ! -e "$FAILURE_LOG" ]    || fail 142 "helper failure log present";        PASS=$((PASS+1))

# Phase 6 (1): candidate still installed
rpm -q "$CANDIDATE" >/dev/null 2>&1 || fail 150 "$CANDIDATE no longer installed"
INSTALLED_NEVRA="$(rpm -q "$CANDIDATE")"
PASS=$((PASS+1))

# Phase 7 (2): locks not actively held (06F J.3)
assert_lock_free "$DECIDER_LOCK" 160; PASS=$((PASS+1))
assert_lock_free "$HELPER_LOCK"  161; PASS=$((PASS+1))

# Phase 8 (3): current UKI identity
{ [ -f "$UKI_PATH" ] && [ ! -L "$UKI_PATH" ]; } || fail 200 "current UKI not regular non-symlink: $UKI_PATH"; PASS=$((PASS+1))
sha256sum "$UKI_PATH" | awk '{print $1}' | grep -Exq "$EXP_UKI_SHA" \
    || fail 201 "current UKI sha != post-Gate-3 expected ($EXP_UKI_SHA)"; PASS=$((PASS+1))
sbverify --cert "$DB_CRT" "$UKI_PATH" >/dev/null 2>&1 \
    || fail 202 "current UKI sbverify failed against $DB_CRT"; PASS=$((PASS+1))

# Phase 9 (2): current UKI structural sanity
uki_has_pcr_sections "$UKI_PATH" || fail 210 "current UKI missing .pcrpkey or .pcrsig"; PASS=$((PASS+1))
uki_vma_sane "$UKI_PATH" "$VMA_THRESHOLD" \
    || fail 211 "current UKI section(s) at VMA >= $VMA_THRESHOLD (06F B.1 layout-corruption signal)"; PASS=$((PASS+1))

# Phase 10 (4): secondary UKI structural sanity (no sha pin per Gate 4 spec)
{ [ -f "$UKI_PATH_SECONDARY" ] && [ ! -L "$UKI_PATH_SECONDARY" ]; } \
    || fail 220 "secondary UKI not regular non-symlink: $UKI_PATH_SECONDARY"; PASS=$((PASS+1))
sbverify --cert "$DB_CRT" "$UKI_PATH_SECONDARY" >/dev/null 2>&1 \
    || fail 221 "secondary UKI sbverify failed"; PASS=$((PASS+1))
uki_has_pcr_sections "$UKI_PATH_SECONDARY" \
    || fail 222 "secondary UKI missing .pcrpkey or .pcrsig"; PASS=$((PASS+1))
uki_vma_sane "$UKI_PATH_SECONDARY" "$VMA_THRESHOLD" \
    || fail 223 "secondary UKI section(s) at VMA >= $VMA_THRESHOLD"; PASS=$((PASS+1))

# Phase 11 (2): bootctl Default + Current Entry (06F I.2 parse bootctl status)
BOOTCTL_OUT="$(new_tmp)"
bootctl status >"$BOOTCTL_OUT" 2>/dev/null || fail 300 "bootctl status returned non-zero"
DEFAULT_ENTRY="$(awk -F': ' '/^[[:space:]]*Default Entry:/ {print $2; exit}' "$BOOTCTL_OUT" | xargs)"
CURRENT_ENTRY="$(awk -F': ' '/^[[:space:]]*Current Entry:/ {print $2; exit}' "$BOOTCTL_OUT" | xargs)"
[ "$DEFAULT_ENTRY" = "$EXP_DEFAULT_ENTRY" ] \
    || fail 301 "Default Entry='$DEFAULT_ENTRY' (expected '$EXP_DEFAULT_ENTRY')"; PASS=$((PASS+1))
[ "$CURRENT_ENTRY" = "$EXP_DEFAULT_ENTRY" ] \
    || fail 302 "Current Entry='$CURRENT_ENTRY' (expected '$EXP_DEFAULT_ENTRY')"; PASS=$((PASS+1))

# Phase 12 (5): PCR 11 cross-validation
STORED_PCR11="$(tr -d '[:space:]' <"$PCR11_FILE" | awk '{print toupper($0)}')"
[ "$STORED_PCR11" = "$EXP_STORED_PCR11" ] \
    || fail 400 "stored PCR 11='$STORED_PCR11' (expected '$EXP_STORED_PCR11')"; PASS=$((PASS+1))

# Independent recomputation via tboot-predict-pcr11 (no --store flag; predictor
# default STORE=0; STATE_FILE never opened for write; scratch on /run tmpfs
# shredded on predictor's own EXIT trap).
PRED_OUT="$(new_tmp)"; PRED_ERR="$(new_tmp)"
PRED_RC=0
"$PREDICT" "$UKI_PATH" >"$PRED_OUT" 2>"$PRED_ERR" || PRED_RC=$?
if [ "$PRED_RC" -ne 0 ]; then
    echo "--- tboot-predict-pcr11 stderr (rc=$PRED_RC) ---"
    cat "$PRED_ERR"
    fail 401 "tboot-predict-pcr11 against $UKI_PATH failed (rc=$PRED_RC)"
fi
PASS=$((PASS+1))

INDEPENDENT_PCR11="$(awk -F= '/^FINAL_PCR11=/{print $2; exit}' "$PRED_OUT" \
                     | tr -d '[:space:]' | tr 'a-f' 'A-F')"
if [ -z "$INDEPENDENT_PCR11" ]; then
    echo "--- tboot-predict-pcr11 stdout ---"
    cat "$PRED_OUT"
    fail 402 "FINAL_PCR11= line missing from predictor stdout"
fi
PASS=$((PASS+1))

# Independent prediction must byte-match stored value
[ "$INDEPENDENT_PCR11" = "$STORED_PCR11" ] \
    || fail 403 "independent prediction '$INDEPENDENT_PCR11' != stored '$STORED_PCR11'"; PASS=$((PASS+1))

# Runtime PCR 11 must still equal pre-reboot value (no reboot yet; mismatch
# against the new STORED prediction is expected here)
RT_PCR11="$(awk '{gsub(/^0x/,""); print toupper($0)}' /sys/class/tpm/tpm0/pcr-sha256/11)"
[ "$RT_PCR11" = "$EXP_RUNTIME_PCR11" ] \
    || fail 404 "runtime PCR 11='$RT_PCR11' (expected pre-reboot '$EXP_RUNTIME_PCR11')"; PASS=$((PASS+1))

# Phase 13 (3): journal cleanliness since Gate 3 close (-o cat broad pattern)
LD_MTIME="$(stat -c %Y "$LAST_DECISION")"
SCAN_SINCE="@${LD_MTIME}"
DEC_JOURNAL="$(new_tmp)"; HELP_JOURNAL="$(new_tmp)"; HOOK_JOURNAL="$(new_tmp)"
journalctl -t tboot-dnf-posttrans --since "$SCAN_SINCE" --no-pager -o cat >"$DEC_JOURNAL"  2>&1 || true
journalctl -t tboot-dnf-helper    --since "$SCAN_SINCE" --no-pager -o cat >"$HELP_JOURNAL" 2>&1 || true
journalctl -t 80-tpm2-sign        --since "$SCAN_SINCE" --no-pager -o cat >"$HOOK_JOURNAL" 2>&1 || true

if journal_has_error "$DEC_JOURNAL";  then show_error_lines "$DEC_JOURNAL";  fail 500 "decider journal has error-like lines since Gate 3 close"; fi; PASS=$((PASS+1))
if journal_has_error "$HELP_JOURNAL"; then show_error_lines "$HELP_JOURNAL"; fail 501 "helper journal has error-like lines since Gate 3 close";  fi; PASS=$((PASS+1))
if journal_has_error "$HOOK_JOURNAL"; then show_error_lines "$HOOK_JOURNAL"; fail 502 "80-tpm2-sign journal has error-like lines since Gate 3 close"; fi; PASS=$((PASS+1))

echo "=== B.2.4 Gate 4: ${PASS}/36 PASS ==="
EOF
```

**Gate 5: pre-reboot snapshot.** Snapshot-only gate, no harness; convention identical to B.2.4 Gate 2 and B.2.3 Gate 7.

```bash
# Run on Proxmox host pve-host as root
qm snapshot 500 module5-b2-4-pre-reboot \
  --description "B.2.4 Gate 4 passed: pre-reboot read-only cross-validation 36/36 PASS. Current baseline sha 6b032bd81a7f3a57b8350c76cf690deb3a942704a76455b881f7a5d98df82a47. Stored PCR11 prediction 28E66CE224C9BB0711059470D77103A5AB5A5C1674C03D1BA1563C2FEE52C90F independently reproduced from on-disk rebuilt UKI. Runtime PCR11 still pre-reboot A03EB49C9335D721758AE8AA9C34DA87664C5CAF9FAD7C314C68C038B16435DD. Current UKI sha 0ac32c8b98428f9a44339a5991eb86e2704986e84ce6e09ae9721bba41726949. Bootctl Current and Default Entry both MID-prefixed 6.19.14 UKI. Sentinel absent; last-decision helper-success; trust-chain shas stable. Checkpoint before Gate 6 reboot validation."

qm listsnapshot 500
qm status 500
```

### Expected output

- Gate 4: `=== B.2.4 Gate 4: 36/36 PASS ===`. Independent prediction equals stored prediction byte-for-byte. Runtime PCR 11 still pre-reboot value.
- Gate 5: `qm snapshot` exits 0. `qm listsnapshot 500` shows `module5-b2-4-pre-reboot` with `current` directly below; parented to `module5-b2-4-pre-drift-transaction`. `qm status 500` returns `running`.

### Stop condition

- Gate 4 FAIL 200–202 / 210–211 / 220–223: current or secondary UKI integrity broken: do not snapshot, do not reboot, capture forensics, rollback to `module5-b2-4-pre-drift-transaction`.
- Gate 4 FAIL 400–404: PCR 11 cross-validation broken (stored ≠ independent, or runtime changed without reboot, or predictor errored). Severe: investigate before any reboot.
- Gate 4 FAIL 130–142 / 110–114 / 120: trust-chain or state drift between Step 34 closure and Gate 4: investigate; do not proceed.
- Gate 5 fails if `qm snapshot` returns non-zero or post-creation `qm listsnapshot 500` does not show `module5-b2-4-pre-reboot` parented correctly. Do not declare Gate 5 closed even if `qm snapshot` reported success without lineage verification.

### Snapshot point

`module5-b2-4-pre-reboot`: pre-reboot cliff-edge anchor; rollback target for any Gate 6 reboot failure (passphrase prompt, firmware Secure-Boot rejection, kernel panic). Parent: `module5-b2-4-pre-drift-transaction`.

---

## Step 36: Block B.2.4 Gates 6A–6B: reboot validation (operator-attested clean boot + post-reboot read-only harness)

**Goal:** prove the trusted-boot chain end-to-end across an actual reboot transition: firmware loads the rebuilt UKI; systemd-cryptsetup performs TPM2 unlock of LUKS keyslot 1 without an operator passphrase prompt; userspace comes up clean; runtime PCR 11 byte-matches the new stored prediction `28E66CE2…C90F`. Gate 6A is operator-attested (manual Proxmox console observation is mandatory and cannot be inferred from journal evidence). Gate 6B is the read-only post-reboot validation harness.
**Prerequisites:** Step 35 (Gates 4–5) closed; snapshot `module5-b2-4-pre-reboot` taken with `current` directly below; VM running.
**Where to run:** Gate 6A: pre-reboot probe + reboot inside VM 500 as root (tmux); manual observation on Proxmox web UI console for VM 500 on host pve-host; post-reboot liveness inside new SSH session. Gate 6B: inside the post-reboot SSH/tmux session as root.

### Commands

**Gate 6A Step 1: pre-reboot cliff-edge probe (17/17 PASS target).** Confirms nothing has drifted between Gate 4 / Gate 5 close and the imminent reboot. If any assertion fails, halt: do NOT reboot.

```bash
# Run on VM 500 as root (tmux)
bash <<'EOF'
set -u

EXP_DECIDER_SHA="1e75110935a72fc9433c9b4bf5688d26bc6d3bd1e02f4388b5e1f32ff1e5cc05"
EXP_HELPER_SHA="937afc7a0d199bf7b7d1a5f5bcccf45c37c0f4f7dd0ad924b2567c4da7125e44"
EXP_PREDICT_SHA="b729ec271dd76cd8b4581f16c86e50fc28b947f2a1de5a7e16c6d5828591e3de"
EXP_HOOK_SHA="5857e51d5551e05a7ca384f71529ec2fef07f5a6dbe40df70c6bbe49d720947e"
EXP_PRODRULE_SHA="dfbffe56fa95f4e0231569bb3a1aabc8952b9f99c9d290e4f8fb1005ef90d798"
EXP_BASELINE_SHA="6b032bd81a7f3a57b8350c76cf690deb3a942704a76455b881f7a5d98df82a47"
EXP_UKI_SHA="0ac32c8b98428f9a44339a5991eb86e2704986e84ce6e09ae9721bba41726949"
EXP_STORED_PCR11="28E66CE224C9BB0711059470D77103A5AB5A5C1674C03D1BA1563C2FEE52C90F"
EXP_RUNTIME_PCR11_PRE="A03EB49C9335D721758AE8AA9C34DA87664C5CAF9FAD7C314C68C038B16435DD"

EXP_MID="0123456789abcdef0123456789abcdef"
EXP_KVER_BOOTED="6.19.14-200.fc43.x86_64"
UKI_PATH="/boot/efi/EFI/Linux/${EXP_MID}-${EXP_KVER_BOOTED}.efi"
PCR11_FILE="/root/tboot-lab/state/expected-pcr11-after-hook-uki.txt"

DECIDER="/usr/local/sbin/tboot-dnf-posttrans"
HELPER="/usr/local/sbin/tboot-dnf-helper"
PREDICT="/usr/local/sbin/tboot-predict-pcr11"
HOOK="/etc/kernel/install.d/80-tpm2-sign.install"
PRODRULE="/etc/dnf/libdnf5-plugins/actions.d/50-tboot-posttrans.actions"
BASELINE="/var/lib/tboot-dnf-posttrans/boot-input-manifest.baseline"
LAST_DECISION="/var/lib/tboot-dnf-posttrans/last-decision"
LAST_ERROR="/var/lib/tboot-dnf-posttrans/last-error"
SENTINEL="/var/lib/tboot-dnf-helper/UNSAFE-TO-REBOOT"
DECIDER_LOCK="/run/tboot-dnf-posttrans.lock"
HELPER_LOCK="/run/tboot-dnf-helper.lock"
CANDIDATE="dracut-network"

PASS=0
fail() { echo "FAIL $1: $2: DO NOT REBOOT"; exit "$1"; }

assert_lock_free() {
    local lock="$1" code="$2"
    if [ -e "$lock" ] || [ -L "$lock" ]; then
        { [ -f "$lock" ] && [ ! -L "$lock" ]; } || fail "$code" "$lock not regular non-symlink"
        ( exec 9<>"$lock"; flock -n 9 ) || fail "$code" "$lock actively held"
    fi
}

[ "$(id -u)" -eq 0 ] || fail 101 "must run as root"; PASS=$((PASS+1))
[ ! -e "$SENTINEL" ] || fail 102 "UNSAFE-TO-REBOOT sentinel present"; PASS=$((PASS+1))

LD_DECISION="$(awk -F= '$1=="decision"{print $2}' "$LAST_DECISION")"
[ "$LD_DECISION" = "helper-success" ] || fail 103 "last-decision.decision='$LD_DECISION' (expected helper-success)"; PASS=$((PASS+1))
[ ! -e "$LAST_ERROR" ] || fail 104 "decider last-error present"; PASS=$((PASS+1))

STORED_PCR11="$(tr -d '[:space:]' <"$PCR11_FILE" | awk '{print toupper($0)}')"
[ "$STORED_PCR11" = "$EXP_STORED_PCR11" ] || fail 105 "stored PCR 11 drift since Gate 4"; PASS=$((PASS+1))

RT_PCR11="$(awk '{gsub(/^0x/,""); print toupper($0)}' /sys/class/tpm/tpm0/pcr-sha256/11)"
[ "$RT_PCR11" = "$EXP_RUNTIME_PCR11_PRE" ] \
    || fail 106 "runtime PCR 11='$RT_PCR11' (expected pre-reboot '$EXP_RUNTIME_PCR11_PRE'); halt and triage"; PASS=$((PASS+1))

sha256sum "$UKI_PATH" | awk '{print $1}' | grep -Exq "$EXP_UKI_SHA" || fail 107 "current UKI sha drift"; PASS=$((PASS+1))

# bootctl Default + Current still point to the MID-prefixed rebuilt UKI
EXP_ENTRY="${EXP_MID}-${EXP_KVER_BOOTED}.efi"
BOOTCTL_OUT="$(mktemp)"
bootctl status >"$BOOTCTL_OUT" 2>/dev/null || fail 108 "bootctl status failed"
DEFAULT_ENTRY="$(awk -F': ' '/^[[:space:]]*Default Entry:/ {print $2; exit}' "$BOOTCTL_OUT" | xargs)"
CURRENT_ENTRY="$(awk -F': ' '/^[[:space:]]*Current Entry:/ {print $2; exit}' "$BOOTCTL_OUT" | xargs)"
rm -f "$BOOTCTL_OUT"
[ "$DEFAULT_ENTRY" = "$EXP_ENTRY" ] || fail 108 "Default Entry='$DEFAULT_ENTRY' expected '$EXP_ENTRY'"
[ "$CURRENT_ENTRY" = "$EXP_ENTRY" ] || fail 108 "Current Entry='$CURRENT_ENTRY' expected '$EXP_ENTRY'"
PASS=$((PASS+1))

rpm -q "$CANDIDATE" >/dev/null 2>&1 || fail 109 "$CANDIDATE no longer installed"; PASS=$((PASS+1))

assert_lock_free "$DECIDER_LOCK" 110; PASS=$((PASS+1))
assert_lock_free "$HELPER_LOCK"  111; PASS=$((PASS+1))

sha256sum "$BASELINE" | awk '{print $1}' | grep -Exq "$EXP_BASELINE_SHA" || fail 112 "baseline sha drift"; PASS=$((PASS+1))

sha256sum "$DECIDER"  | awk '{print $1}' | grep -Exq "$EXP_DECIDER_SHA"  || fail 113 "decider sha drift";          PASS=$((PASS+1))
sha256sum "$HELPER"   | awk '{print $1}' | grep -Exq "$EXP_HELPER_SHA"   || fail 114 "helper sha drift";           PASS=$((PASS+1))
sha256sum "$PREDICT"  | awk '{print $1}' | grep -Exq "$EXP_PREDICT_SHA"  || fail 115 "predict sha drift";          PASS=$((PASS+1))
sha256sum "$HOOK"     | awk '{print $1}' | grep -Exq "$EXP_HOOK_SHA"     || fail 116 "hook sha drift";             PASS=$((PASS+1))
sha256sum "$PRODRULE" | awk '{print $1}' | grep -Exq "$EXP_PRODRULE_SHA" || fail 117 "production rule sha drift";  PASS=$((PASS+1))

echo "=== B.2.4 Gate 6A pre-reboot probe: ${PASS}/17 PASS ==="
echo "Cliff-edge OK. Probe complete. STOP: open Proxmox console BEFORE reboot."
EOF
```

**Gate 6A Step 2: open Proxmox console BEFORE issuing reboot.** Web UI: navigate to VM 500 (`tboot-lab`) → **Console** on host `pve-host`. noVNC viewer must be rendered and actively watched before Step 3.

**Gate 6A Step 3: issue the reboot.** From inside VM 500, same tmux session:

```bash
# Run on VM 500 as root, AFTER the Proxmox console is open and being watched
systemctl reboot
```

SSH drops. tmux state inside the VM terminates with the reboot.

**Gate 6A Step 4: manual Proxmox console observation (mandatory).** Watch:

| Boot phase | Pass | Hard fail |
|---|---|---|
| Firmware POST / OVMF banner | No "Image authentication failed" / "Security Violation" | Firmware rejects rebuilt UKI → roll back to `module5-b2-4-pre-reboot` |
| systemd-boot menu | Default = MID-prefixed 6.19.14 UKI; auto-selects on timeout | Different entry selected (forensic UKI hijack) → triage |
| Kernel boot | Standard kernel messages | Kernel panic → capture screenshot, roll back |
| systemd-cryptsetup phase | **No passphrase prompt.** Brief pause (< 2 s typical) then proceed | **LUKS passphrase prompt appearing = Gate 6 hard fail. Do NOT type the passphrase as part of the validation path.** See Recovery procedures. |
| Login prompt | `tboot login:` appears | System hung: capture screenshot, triage |

**Gate 6A Step 5: SSH reconnect liveness (read-only).**

```bash
# Run on VM 500 as root (new SSH session, new tmux)
uname -r                                            # expected: 6.19.14-200.fc43.x86_64
cat /proc/sys/kernel/random/boot_id                 # expected: differs from pre-reboot boot_id
systemctl is-system-running                         # expected: running (degraded acceptable for Gate 6A; Gate 6B will diagnose)
uptime -p                                           # expected: small (< 2 min)
```

**Gate 6A closure deliverables:** pre-reboot probe output (17/17 PASS) + operator console-observation log + Step 5 liveness output + explicit operator attestation "no passphrase prompt observed".

**Gate 6B: post-reboot read-only validation harness (28/28 PASS target).** Strict allowlist: `bootctl status`, `cat /sys/class/tpm/tpm0/pcr-sha256/11`, `cat /root/tboot-lab/state/expected-pcr11-after-hook-uki.txt`, `sha256sum`, `sbverify`, `objdump -h`, `rpm -q dracut-network`, `journalctl`. Strict banlist: no `dnf`, no `dracut`, no `kernel-install`, no `ukify`, no `bootctl set-*`, no `systemd-cryptenroll`, no `cryptsetup`, no `efi-updatevar`, no cleanup, no snapshot.

```bash
# Run on VM 500 as root (post-reboot SSH/tmux session)
bash <<'EOF'
set -u

EXP_DECIDER_SHA="1e75110935a72fc9433c9b4bf5688d26bc6d3bd1e02f4388b5e1f32ff1e5cc05"
EXP_HELPER_SHA="937afc7a0d199bf7b7d1a5f5bcccf45c37c0f4f7dd0ad924b2567c4da7125e44"
EXP_PREDICT_SHA="b729ec271dd76cd8b4581f16c86e50fc28b947f2a1de5a7e16c6d5828591e3de"
EXP_HOOK_SHA="5857e51d5551e05a7ca384f71529ec2fef07f5a6dbe40df70c6bbe49d720947e"
EXP_PRODRULE_SHA="dfbffe56fa95f4e0231569bb3a1aabc8952b9f99c9d290e4f8fb1005ef90d798"
EXP_BASELINE_SHA="6b032bd81a7f3a57b8350c76cf690deb3a942704a76455b881f7a5d98df82a47"
EXP_UKI_SHA="0ac32c8b98428f9a44339a5991eb86e2704986e84ce6e09ae9721bba41726949"
EXP_STORED_PCR11="28E66CE224C9BB0711059470D77103A5AB5A5C1674C03D1BA1563C2FEE52C90F"
EXP_OLD_PCR11="A03EB49C9335D721758AE8AA9C34DA87664C5CAF9FAD7C314C68C038B16435DD"

EXP_MID="0123456789abcdef0123456789abcdef"
EXP_KVER_BOOTED="6.19.14-200.fc43.x86_64"
EXP_KVER_SECONDARY="6.17.1-300.fc43.x86_64"
EXP_DEFAULT_ENTRY="${EXP_MID}-${EXP_KVER_BOOTED}.efi"
VMA_THRESHOLD="0x180000000"
CANDIDATE="dracut-network"

DECIDER="/usr/local/sbin/tboot-dnf-posttrans"
HELPER="/usr/local/sbin/tboot-dnf-helper"
PREDICT="/usr/local/sbin/tboot-predict-pcr11"
HOOK="/etc/kernel/install.d/80-tpm2-sign.install"
PRODRULE="/etc/dnf/libdnf5-plugins/actions.d/50-tboot-posttrans.actions"

DECIDER_STATE="/var/lib/tboot-dnf-posttrans"
BASELINE="${DECIDER_STATE}/boot-input-manifest.baseline"
LAST_DECISION="${DECIDER_STATE}/last-decision"
LAST_MANIFEST="${DECIDER_STATE}/last-computed-manifest"
LAST_ERROR="${DECIDER_STATE}/last-error"
HELPER_STATE="/var/lib/tboot-dnf-helper"
SENTINEL="${HELPER_STATE}/UNSAFE-TO-REBOOT"
FAILURE_MARKER="${HELPER_STATE}/last-failure"
FAILURE_LOG="${HELPER_STATE}/last-failure.log"

UKI_PATH="/boot/efi/EFI/Linux/${EXP_MID}-${EXP_KVER_BOOTED}.efi"
UKI_PATH_SECONDARY="/boot/efi/EFI/Linux/${EXP_MID}-${EXP_KVER_SECONDARY}.efi"
PCR11_FILE="/root/tboot-lab/state/expected-pcr11-after-hook-uki.txt"
DB_CRT="/etc/uefi-keys/db.crt"
TPM_PCR11_SYSFS="/sys/class/tpm/tpm0/pcr-sha256/11"

TMP_FILES=()
new_tmp()  { local t; t="$(mktemp)"; TMP_FILES+=("$t"); printf '%s\n' "$t"; }
_cleanup() { local f; for f in "${TMP_FILES[@]:-}"; do rm -f "$f" 2>/dev/null || true; done; }
trap _cleanup EXIT HUP INT TERM

PASS=0
fail() { echo "FAIL $1: $2"; exit "$1"; }

journal_has_error() {
    grep -Eiq '(^|[[:space:]])(err|error|fail|failed|failure)([[:space:]:]|$)' "$1"
}
show_error_lines() {
    grep -Ein '(^|[[:space:]])(err|error|fail|failed|failure)([[:space:]:]|$)' "$1" || true
}
uki_has_pcr_sections() {
    objdump -h "$1" 2>/dev/null | awk '$2==".pcrpkey"{p=1} $2==".pcrsig"{s=1} END{exit !(p && s)}'
}
uki_vma_sane() {
    objdump -h "$1" 2>/dev/null | awk -v threshold="$2" '
        /^[[:space:]]+[0-9]+[[:space:]]+\./ {
            vma = strtonum("0x" $4)
            if (vma >= strtonum(threshold)) { printf "anomalous VMA: %s at 0x%s\n", $2, $4 > "/dev/stderr"; bad = 1 }
        }
        END { exit !!bad }'
}

# Phase 1 (1)
[ "$(id -u)" -eq 0 ] || fail 101 "must run as root"; PASS=$((PASS+1))

# Phase 2 (3): CORE PCR 11 pass condition: heart of Gate 6B
STORED_PCR11="$(tr -d '[:space:]' <"$PCR11_FILE" | awk '{print toupper($0)}')"
[ "$STORED_PCR11" = "$EXP_STORED_PCR11" ] || fail 200 "stored PCR 11='$STORED_PCR11' (expected '$EXP_STORED_PCR11')"; PASS=$((PASS+1))
RT_PCR11="$(awk '{gsub(/^0x/,""); print toupper($0)}' "$TPM_PCR11_SYSFS")"
[ "$RT_PCR11" = "$EXP_STORED_PCR11" ] || fail 201 "runtime PCR 11='$RT_PCR11' (expected new prediction '$EXP_STORED_PCR11'); chain did not advance"; PASS=$((PASS+1))
[ "$RT_PCR11" != "$EXP_OLD_PCR11" ] || fail 202 "runtime PCR 11 still pre-Gate-3 value '$EXP_OLD_PCR11'"; PASS=$((PASS+1))

# Phase 3 (3): bootctl-attested boot path
BOOTCTL_OUT="$(new_tmp)"
bootctl status >"$BOOTCTL_OUT" 2>/dev/null || fail 300 "bootctl status returned non-zero"
SECURE_BOOT_LINE="$(awk -F': ' '/^[[:space:]]*Secure Boot:/ {print $2; exit}' "$BOOTCTL_OUT" | xargs)"
echo "${SECURE_BOOT_LINE}" | grep -Eq '^enabled([[:space:]]|$)' \
    || fail 301 "Secure Boot='${SECURE_BOOT_LINE}' (expected 'enabled' prefix)"; PASS=$((PASS+1))
MEASURED_UKI_LINE="$(awk -F': ' '/^[[:space:]]*Measured UKI:/ {print $2; exit}' "$BOOTCTL_OUT" | xargs)"
[ "$MEASURED_UKI_LINE" = "yes" ] || fail 302 "Measured UKI='$MEASURED_UKI_LINE' (expected 'yes')"; PASS=$((PASS+1))
CURRENT_ENTRY="$(awk -F': ' '/^[[:space:]]*Current Entry:/ {print $2; exit}' "$BOOTCTL_OUT" | xargs)"
[ "$CURRENT_ENTRY" = "$EXP_DEFAULT_ENTRY" ] || fail 303 "Current Entry='$CURRENT_ENTRY' (expected '$EXP_DEFAULT_ENTRY')"; PASS=$((PASS+1))
DEFAULT_ENTRY="$(awk -F': ' '/^[[:space:]]*Default Entry:/ {print $2; exit}' "$BOOTCTL_OUT" | xargs)"

# Phase 4 (5): running kernel + current UKI integrity
KVER_NOW="$(uname -r)"
[ "$KVER_NOW" = "$EXP_KVER_BOOTED" ] || fail 400 "running kernel='$KVER_NOW' (expected '$EXP_KVER_BOOTED')"; PASS=$((PASS+1))
sha256sum "$UKI_PATH" | awk '{print $1}' | grep -Exq "$EXP_UKI_SHA" || fail 410 "current UKI sha drift"; PASS=$((PASS+1))
sbverify --cert "$DB_CRT" "$UKI_PATH" >/dev/null 2>&1 || fail 411 "current UKI sbverify failed"; PASS=$((PASS+1))
uki_has_pcr_sections "$UKI_PATH" || fail 412 "current UKI missing .pcrpkey or .pcrsig"; PASS=$((PASS+1))
uki_vma_sane "$UKI_PATH" "$VMA_THRESHOLD" || fail 413 "current UKI section VMA anomaly"; PASS=$((PASS+1))

# Phase 5 (4): decider state across reboot
sha256sum "$BASELINE"      | awk '{print $1}' | grep -Exq "$EXP_BASELINE_SHA" || fail 500 "baseline sha drift"; PASS=$((PASS+1))
sha256sum "$LAST_MANIFEST" | awk '{print $1}' | grep -Exq "$EXP_BASELINE_SHA" || fail 501 "last-manifest != baseline"; PASS=$((PASS+1))
LD_DECISION="$(awk -F= '$1=="decision"{print $2}' "$LAST_DECISION")"
[ "$LD_DECISION" = "helper-success" ] || fail 502 "last-decision.decision='$LD_DECISION'"; PASS=$((PASS+1))
[ ! -e "$LAST_ERROR" ] || fail 503 "decider last-error present"; PASS=$((PASS+1))

# Phase 6 (3): helper state across reboot
[ ! -e "$SENTINEL" ]       || fail 600 "UNSAFE-TO-REBOOT sentinel present after reboot"; PASS=$((PASS+1))
[ ! -e "$FAILURE_MARKER" ] || fail 601 "helper failure marker present after reboot";     PASS=$((PASS+1))
[ ! -e "$FAILURE_LOG" ]    || fail 602 "helper failure log present after reboot";        PASS=$((PASS+1))

# Phase 7 (1): candidate still installed
rpm -q "$CANDIDATE" >/dev/null 2>&1 || fail 700 "$CANDIDATE no longer installed"
INSTALLED_NEVRA="$(rpm -q "$CANDIDATE")"; PASS=$((PASS+1))

# Phase 8 (5): trust-chain byte-stability across reboot
sha256sum "$DECIDER"  | awk '{print $1}' | grep -Exq "$EXP_DECIDER_SHA"  || fail 800 "decider sha drift";          PASS=$((PASS+1))
sha256sum "$HELPER"   | awk '{print $1}' | grep -Exq "$EXP_HELPER_SHA"   || fail 801 "helper sha drift";           PASS=$((PASS+1))
sha256sum "$PREDICT"  | awk '{print $1}' | grep -Exq "$EXP_PREDICT_SHA"  || fail 802 "predict sha drift";          PASS=$((PASS+1))
sha256sum "$HOOK"     | awk '{print $1}' | grep -Exq "$EXP_HOOK_SHA"     || fail 803 "hook sha drift";             PASS=$((PASS+1))
sha256sum "$PRODRULE" | awk '{print $1}' | grep -Exq "$EXP_PRODRULE_SHA" || fail 804 "production rule sha drift";  PASS=$((PASS+1))

# Phase 9 (3): journal cleanliness since this boot (journalctl -b 0)
DEC_JOURNAL="$(new_tmp)"; HELP_JOURNAL="$(new_tmp)"; HOOK_JOURNAL="$(new_tmp)"
journalctl -t tboot-dnf-posttrans -b 0 --no-pager -o cat >"$DEC_JOURNAL"  2>&1 || true
journalctl -t tboot-dnf-helper    -b 0 --no-pager -o cat >"$HELP_JOURNAL" 2>&1 || true
journalctl -t 80-tpm2-sign        -b 0 --no-pager -o cat >"$HOOK_JOURNAL" 2>&1 || true
if journal_has_error "$DEC_JOURNAL";  then show_error_lines "$DEC_JOURNAL";  fail 900 "decider journal has errors since this boot"; fi; PASS=$((PASS+1))
if journal_has_error "$HELP_JOURNAL"; then show_error_lines "$HELP_JOURNAL"; fail 901 "helper journal has errors since this boot";  fi; PASS=$((PASS+1))
if journal_has_error "$HOOK_JOURNAL"; then show_error_lines "$HOOK_JOURNAL"; fail 902 "80-tpm2-sign journal has errors since this boot"; fi; PASS=$((PASS+1))

echo "=== B.2.4 Gate 6B: ${PASS}/28 PASS ==="
EOF
```

### Expected output

- Gate 6A pre-reboot probe: `=== B.2.4 Gate 6A pre-reboot probe: 17/17 PASS ===`.
- Gate 6A console observation (manual): firmware POST clean; systemd-boot menu auto-selects MID-prefixed 6.19.14 entry; **no LUKS passphrase prompt**; login prompt reached.
- Gate 6A liveness: `uname -r` = `6.19.14-200.fc43.x86_64`; `boot_id` differs from pre-reboot value; `systemctl is-system-running` = `running`; small uptime.
- Gate 6B: `=== B.2.4 Gate 6B: 28/28 PASS ===`. **Runtime PCR 11 = stored prediction `28E66CE2…C90F` byte-for-byte.** Secure Boot enabled (user). Measured UKI yes. Current Entry = MID-prefixed rebuilt UKI.

### Stop condition

- Gate 6A pre-reboot probe FAIL: halt; do not reboot; triage the specific assertion.
- Gate 6A LUKS passphrase prompt at boot: hard fail. **Do not type the passphrase as part of validation**: typing it proves recovery works, not TPM auto-unlock. Roll back to `module5-b2-4-pre-reboot` or investigate in place; add a `06F` finding for the failure mode.
- Gate 6A firmware Secure Boot rejection: roll back to `module5-b2-4-pre-reboot`. Capture forensics on the rebuilt UKI before rollback.
- Gate 6B FAIL 200–202: PCR 11 byte-match broken: the core property B.2.4 was built to prove. Severe. Capture full TPM PCR state via `tpm2_pcrread sha256:0,1,2,3,4,5,6,7,8,9,10,11,12,13,14,15` (read-only) before rollback. Investigate firmware behaviour, hook output, predictor algorithm.
- Gate 6B FAIL 301–303 / 400 / 410–413: boot-path or UKI-identity drift across reboot: severe; rollback.
- Gate 6B FAIL 500–503 / 600–602: state mutation across the reboot transition that should not have happened. Investigate, then rollback.
- Gate 6B FAIL 900–902: post-reboot errors in decider/helper/hook journals; capture journal contents; do not snapshot or close Gate 7.

### Snapshot point

None at this step. Gate 7 closure snapshot is the next step.

---

## Step 37: Block B.2.4 Gate 7: closure snapshot + read-only ESP forensic audit + staged DNF discipline lift

**Goal:** take the closure snapshot, perform the read-only `/EFI/Linux/` forensic audit, document the audit results, and apply the staged discipline-gate lift. No cleanup at this step: cleanup of forensic UKIs is the scope of Block B.2.5.
**Prerequisites:** Step 36 (Gates 6A–6B) closed.
**Where to run:** Step 1 (snapshot) on Proxmox host pve-host as root. Step 2 (audit) inside VM 500 as root.

### Commands

**Step 1: closure snapshot.**

```bash
# Run on Proxmox host pve-host as root
qm snapshot 500 module5-b2-4-validated \
  --description "B.2.4 validated: real dracut-sensitive DNF transaction dracut-network installed; decider detected boot-input drift; helper-success path rebuilt initramfs/UKIs; baseline advanced to 6b032bd81a7f3a57b8350c76cf690deb3a942704a76455b881f7a5d98df82a47; stored PCR11 advanced to 28E66CE224C9BB0711059470D77103A5AB5A5C1674C03D1BA1563C2FEE52C90F; Gate 4 independently reproduced stored PCR11 from on-disk UKI; Gate 6A rebooted with no LUKS passphrase prompt; Gate 6B 28/28 PASS with runtime PCR11 matching stored prediction; Secure Boot enabled, Measured UKI yes, current UKI sha 0ac32c8b98428f9a44339a5991eb86e2704986e84ce6e09ae9721bba41726949; trust-chain shas stable."

qm listsnapshot 500
qm status 500
```

**Step 2: read-only ESP `/EFI/Linux/` forensic audit.** Inventory every `.efi` / `.EFI` file: `ls -la`, sha256sum, sbverify against `db.crt`, `objdump -h` for section names, presence/absence of `.pcrpkey` and `.pcrsig`, VMA sanity vs `0x180000000`. No rename, no delete, no suffix application. Use `-iname` (case-insensitive): ESP is VFAT and case-insensitive at the filesystem level.

```bash
# Run on VM 500 as root
bash <<'EOF'
set -u

EFI_DIR="/boot/efi/EFI/Linux"
DB_CRT="/etc/uefi-keys/db.crt"
VMA_THRESHOLD="0x180000000"
EXP_MID="0123456789abcdef0123456789abcdef"
EXP_KVER_BOOTED="6.19.14-200.fc43.x86_64"
EXP_KVER_SECONDARY="6.17.1-300.fc43.x86_64"
EXP_DEFAULT_ENTRY="${EXP_MID}-${EXP_KVER_BOOTED}.efi"
EXP_SECONDARY_ENTRY="${EXP_MID}-${EXP_KVER_SECONDARY}.efi"
EXP_CURRENT_UKI_SHA="0ac32c8b98428f9a44339a5991eb86e2704986e84ce6e09ae9721bba41726949"

[ "$(id -u)" -eq 0 ] || { echo "must run as root"; exit 1; }
[ -d "$EFI_DIR" ] || { echo "no $EFI_DIR"; exit 1; }

uki_has_pcr_sections() {
    objdump -h "$1" 2>/dev/null \
        | awk '$2==".pcrpkey"{p=1} $2==".pcrsig"{s=1} END { printf "%s/%s", (p?"present":"absent"), (s?"present":"absent") }'
    echo
}
uki_vma_sane() {
    objdump -h "$1" 2>/dev/null | awk -v threshold="$VMA_THRESHOLD" '
        /^[[:space:]]+[0-9]+[[:space:]]+\./ {
            vma = strtonum("0x" $4)
            if (vma >= strtonum(threshold)) { bad[++n] = sprintf("%s@0x%s", $2, $4) }
        }
        END {
            if (n == 0) { print "sane" }
            else { printf "anomaly:"; for (i = 1; i <= n; i++) printf " %s", bad[i]; print "" }
        }'
}
uki_section_names_csv() {
    objdump -h "$1" 2>/dev/null | awk '/^[[:space:]]+[0-9]+[[:space:]]+\./ {print $2}' | paste -sd, -
}

echo "=== B.2.4 Gate 7 Step 2: ESP /EFI/Linux/ forensic audit ==="
echo "audit_dir: $EFI_DIR; verify_cert: $DB_CRT; vma_threshold: $VMA_THRESHOLD (06F B.1)"
echo ""
echo "--- A. Full ls -la ---"
ls -la "$EFI_DIR"
echo ""

mapfile -t EFI_FILES < <(find "$EFI_DIR" -maxdepth 1 -type f -iname '*.efi' -printf '%f\n' | sort)
TOTAL="${#EFI_FILES[@]}"
echo "--- B. Found $TOTAL .efi/.EFI files ---"
for f in "${EFI_FILES[@]}"; do echo "  $f"; done
echo ""

EXPECTED_CSV="${EXP_DEFAULT_ENTRY},${EXP_SECONDARY_ENTRY}"
UNEXPECTED=()
for f in "${EFI_FILES[@]}"; do
    case ",$EXPECTED_CSV," in *",$f,"*) ;; *) UNEXPECTED+=("$f") ;; esac
done
echo "--- C. Classification ---"
for f in "${EFI_FILES[@]}"; do
    case ",$EXPECTED_CSV," in *",$f,"*) echo "  EXPECTED   : $f" ;; esac
done
if [ "${#UNEXPECTED[@]}" -eq 0 ]; then
    echo "  unexpected : (none)"
else
    for f in "${UNEXPECTED[@]}"; do echo "  UNEXPECTED : $f"; done
fi
echo ""

echo "--- D. Per-file forensic record ---"
for f in "${EFI_FILES[@]}"; do
    path="$EFI_DIR/$f"
    expected_flag="UNEXPECTED"
    case ",$EXPECTED_CSV," in *",$f,"*) expected_flag="EXPECTED" ;; esac
    echo ""
    echo "[file] $f"
    echo "  status        : $expected_flag"
    echo "  ls -la        :"
    ls -la "$path" | sed 's/^/    /'
    echo "  size_bytes    : $(stat -c '%s' "$path")"
    echo "  mtime_iso     : $(stat -c '%y' "$path")"
    sha="$(sha256sum "$path" | awk '{print $1}')"
    echo "  sha256        : $sha"
    if [ "$expected_flag" = "EXPECTED" ] && [ "$f" = "$EXP_DEFAULT_ENTRY" ]; then
        if [ "$sha" = "$EXP_CURRENT_UKI_SHA" ]; then
            echo "  sha_match     : yes (matches Gate-6B expected)"
        else
            echo "  sha_match     : NO (drift; expected $EXP_CURRENT_UKI_SHA)"
            exit 2
        fi
    fi
    if sbverify --cert "$DB_CRT" "$path" >/dev/null 2>&1; then
        echo "  sbverify      : OK (signed by db.crt)"
    else
        echo "  sbverify      : FAIL"
    fi
    sig_count="$(pesign --show-signature --in="$path" 2>/dev/null | grep -c 'common name' || true)"
    echo "  signer_count  : $sig_count"
    echo "  pcrpkey/sig   : $(uki_has_pcr_sections "$path")"
    echo "  vma_sanity    : $(uki_vma_sane "$path")"
    echo "  sections      : $(uki_section_names_csv "$path")"
done
echo ""
echo "=== Audit summary ==="
echo "total_efi_files  : $TOTAL"
echo "expected_count   : $((TOTAL - ${#UNEXPECTED[@]}))"
echo "unexpected_count : ${#UNEXPECTED[@]}"
echo "This audit is READ-ONLY. No rename, no delete, no suffix change."
EOF
```

### Expected output

- Step 1: `qm snapshot` exit 0; `module5-b2-4-validated` present in `qm listsnapshot 500` with `current` directly below; parented to `module5-b2-4-pre-reboot`; VM `running`.
- Step 2 (2026-05-25 reference run): 5 total `.efi` files. 2 expected (`cf44500de…-6.19.14-200.fc43.x86_64.efi` sha `0ac32c8b…6949` + `cf44500de…-6.17.1-300.fc43.x86_64.efi` sha `b8b158012835db23e5ba435ea15c5acf1d4e9628344fcf98cee50de4147f8994`). 3 unexpected forensic UKIs, all db.crt-signed: `6.19.14-200.fc43.x86_64.efi` (bare-kver, sha `bcfa64037dc57df5485495af4d74c9bc522d7c6db4dc96a444f05f1e4844e097`, no `.pcrpkey`/`.pcrsig`), `test-tpm2initrd-pcrsig-…` (sha `b94b273e5d2612c3cdba8132bc639c3456ac508146b0f405a53829e89985c4ca`), `test-ukify-native-pcrsig-…` (sha `56e0aa680028a50e2258ae0088c1e24f3bb76c51a6be0704e1d4013ce42bd277`). The `.loaderror`-suffixed broken-objcopy UKI is correctly excluded by the `-iname '*.efi'` filter (B.1 forensic-preservation convention working as designed).

### Stop condition

- Step 1 fails if `qm snapshot` returns non-zero or `qm listsnapshot 500` does not show `module5-b2-4-validated` parented to `module5-b2-4-pre-reboot` with `current` directly below.
- Step 2 `exit 2` if the booted-kernel MID-prefixed UKI sha drifts from `0ac32c8b…6949` (mutation between Gate 6B and Gate 7 that should not have happened).
- Step 2 finding any `EXPECTED` file failing `sbverify` against `db.crt`, missing `.pcrpkey`/`.pcrsig`, or with anomalous VMA: severe: rollback to `module5-b2-4-pre-reboot` before further work; the validated MID-prefixed UKIs must remain firmware-loadable and structurally sound.

### Staged DNF discipline lift (applied at Step 37 close)

After Gate 7 closes (snapshot + audit + this runbook step + companion `06F` L.1 + companion `05_Update_Workflows_and_Key_Storage.md` §3.4):

**Allowed:** `dnf install`/`remove`/`reinstall`/`upgrade` for non-systemd, non-kernel single packages (including dracut-sensitive ones) under normal operator review. The decider's `post_transaction` `.actions` rule fires on every transaction.

**Still gated at the time of this step:** `dnf upgrade systemd*` (B.4/B.5
scope); `dnf reinstall systemd-boot*` (B.4 scope); `dnf upgrade kernel*`
(kernel-package class not yet live-validated); blind `dnf update`/`dnf upgrade`
without an explicit package argument. B.4/B.5 later validated the systemd-boot
path, and the 19 June 2026 operational run subsequently validated the full
update and kernel-upgrade path.

### Snapshot point

`module5-b2-4-validated`: **B.2.4 closure anchor; preferred rollback target for B.2.5 (ESP forensic cleanup), B.4 (systemd-boot signing automation), and all subsequent operational DNF transactions under the staged discipline lift.** Parent: `module5-b2-4-pre-reboot`. The deeper anchor `module5-b2-3-validated` remains valid as a pre-B.2.4 fallback.

### Step 37 closure

✅ **Block B.2.4 closed end-to-end.** Gates 1–3 in Step 34 (positive drift path pre-reboot, 58/58 + snapshot + 43/43 PASS); Gate 4 in Step 35 (pre-reboot read-only cross-validation, 36/36 PASS); Gate 5 in Step 35 (`module5-b2-4-pre-reboot` snapshot); Gates 6A–6B in Step 36 (operator-attested clean reboot + post-reboot validation, 17/17 + 28/28 PASS); Gate 7 in Step 37 (closure snapshot `module5-b2-4-validated` + read-only ESP audit + staged DNF discipline lift). **Block B.2 full production flow: complete with staged discipline.** `dnf upgrade systemd*` remains separately blocked pending B.4/B.5. B.2.5 (ESP forensic UKI cleanup) is the next scheduled block: see `06F` L.1 and the futures table.

---

# Recovery procedures

The full per-step recovery table is in **Appendix B** (end of document); it now includes the B.4 (Step 38) and B.5 (Step 39) rows.

## Step 38: Block B.4 install/publish gate: install the systemd-boot signing chain, prime a non-converged baseline, mask the native updater (chain UNARMED)

**Goal:** install the independent B.4 systemd-boot signing chain (helper + decider + state-dir), prime an intentionally non-converged baseline, and mask `systemd-boot-update.service`: leaving the chain installed but **unarmed**. The `60-tboot-sbloader.actions` rule is deliberately NOT published here; publishing it and the first controlled `dnf reinstall systemd-boot-unsigned` are Step 39 / Block B.5. This split (install + prime + mask in B.4; arm + fire in B.5) avoids a latent armed window over the non-converged baseline.
**Prerequisites:** Block B.2 complete (`module5-b2-4-validated`). B.4 Gate 1 closed (rev-2 read-only preflight A–M + F3 supplemental unit-body probe N.1–N.6; mask-list locked = `systemd-boot-update.service` only; see `06F` N.5). B.4 Gate 2 probes closed (`install`-vs-`update` → `install`, `06F` N.1; direction → `in`, `06F` N.3/N.4). The three B.4 artifacts authored, staged in `/tmp`, and read-back-validated (`bash -n`, ShellCheck `-S warning` clean, forbidden-token scan, field/path/mode parses): helper `tboot-sbloader-helper` sha `b9df2534b1539355a3c040ee296611f5da629e76467fccfcd44f6ed56325690f` (`1.0.0`, 421 lines), decider `tboot-sbloader-posttrans` sha `b8d9a04c30cd39ee3600fb3ff8f581d7cf284f7f4aa639f03cf0fd733e705f0c` (`1.0.0`, 315 lines), rule `60-tboot-sbloader.actions` sha `2e83fcfcd237359d02a831cb381d40ca221d319108d66c264b847956c0418274` (115 bytes; `post_transaction:systemd-boot-unsigned:in:enabled=host-only raise_error=1:/usr/local/sbin/tboot-sbloader-posttrans`). The decider `--help` must avoid the literal token `bootctl` in its heredoc (use "never invokes the loader installer") so the non-`#` forbidden-token scan stays clean (`06F` N.2).
**Where to run:** Step 0 + Step 8 snapshots on Proxmox host pve-host; Steps 1–7 on VM 500 as root.

> [!important] Run every mutation block inside a child shell
> Each install step runs inside `bash <<'B4_STEPn' ... B4_STEPn` with `set -euo pipefail` and a `fail(){ echo "ABORT: $*" >&2; exit 1; }` so a failed precondition or gate invariant hard-aborts. Do not paste `set -u` at the interactive top level (`06F` N.2). Each step ends with `STEP n PASS` only if every assertion held.

### Step 38 artifact sources (inline, byte-exact: required for reproducibility)

These are the complete, byte-exact sources of the three B.4 artifacts. Stage them in `/tmp` first (single-quoted heredocs → no shell expansion), then the gate steps below install/prime them. After staging, the sha-verify block must match the locked shas exactly before proceeding. Run inside VM 500.

**Helper**: `tboot-sbloader-helper` (sha `b9df2534b1539355a3c040ee296611f5da629e76467fccfcd44f6ed56325690f`, 421 lines):

```bash
cat > /tmp/tboot-sbloader-helper.staged <<'TBOOT_SBLOADER_HELPER_EOF'
#!/usr/bin/env bash
# tboot-sbloader-helper: B.4 systemd-boot signing + propagation worker
# Version: 1.0.0
# Role: sign the source systemd-boot loader with the local db key, PROVE in a
#       throwaway loopback ESP that 'bootctl --no-variables install' selects the
#       signed binary for BOTH propagation targets, then propagate to the real
#       ESP. Fail-closed before any real-ESP write. UKI/PCR signing is NOT this
#       script's job.
#
# State layout: B.4 single locked dir /var/lib/tboot-sbloader (Option A):
#   helper-last-error, helper-last-success            (this script)
#   sbloader-manifest.baseline, last-computed-manifest,
#   last-decision, last-error                         (decider: tboot-sbloader-posttrans)
#   UNSAFE-TO-REBOOT                                   (shared sentinel)

set -euo pipefail
umask 077

readonly VERSION="1.0.0"
readonly TAG="tboot-sbloader-helper"

readonly SOURCE_EFI="/usr/lib/systemd/boot/efi/systemd-bootx64.efi"
readonly SIGNED="/usr/lib/systemd/boot/efi/systemd-bootx64.efi.signed"
readonly ESP_SYSTEMD="/boot/efi/EFI/systemd/systemd-bootx64.efi"
readonly ESP_BOOT="/boot/efi/EFI/BOOT/BOOTX64.EFI"
readonly DB_KEY="/etc/uefi-keys/db.key"
readonly DB_CRT="/etc/uefi-keys/db.crt"
readonly STATE_DIR="/var/lib/tboot-sbloader"
readonly SENTINEL="${STATE_DIR}/UNSAFE-TO-REBOOT"
readonly LAST_ERROR="${STATE_DIR}/helper-last-error"
readonly LAST_SUCCESS="${STATE_DIR}/helper-last-success"
readonly LOCK="/run/tboot-sbloader-helper.lock"

# MODE is set by main() before any stage runs: normal|self-test|dry-run
MODE="normal"

SCRATCH_MOUNTS=()
SCRATCH_FILES=()
SCRATCH_DIRS=()

log() {
  logger -t "$TAG" -- "$*" 2>/dev/null || true
  printf '%s %s\n' "$TAG" "$*" >&2
}

_sha() {
  sha256sum "$1" 2>/dev/null | awk '{print $1}' || true
}

# Hard signer-count gate: exactly one Authenticode signer.
_signer_count() {
  pesign --show-signature --in="$1" 2>/dev/null | grep -c 'common name' || true
}

_ensure_state_dir() {
  if [ ! -e "$STATE_DIR" ]; then
    install -d -m 0700 -o root -g root "$STATE_DIR" || return 1
  fi
  [ ! -L "$STATE_DIR" ] || return 1
  [ -d "$STATE_DIR" ] || return 1
  [ "$(stat -c '%U:%G' "$STATE_DIR" 2>/dev/null)" = "root:root" ] || return 1
  [ "$(stat -c '%a' "$STATE_DIR" 2>/dev/null)" = "700" ] || return 1
  [ -w "$STATE_DIR" ] || return 1
}

_write_last_error() {
  # normal mode only
  [ "$MODE" = "normal" ] || return 0

  local stage="$1"
  local detail="$2"

  _ensure_state_dir 2>/dev/null || return 0

  {
    printf 'epoch=%s\n' "$(date +%s)"
    printf 'date=%s\n'  "$(date -Iseconds)"
    printf 'stage=%s\n' "$stage"
    printf 'detail=%s\n' "$detail"
    printf 'helper_version=%s\n' "$VERSION"
  } > "${LAST_ERROR}.tmp" 2>/dev/null && mv -f "${LAST_ERROR}.tmp" "$LAST_ERROR" 2>/dev/null || true

  chmod 0600 "$LAST_ERROR" 2>/dev/null || true
}

_ensure_sentinel() {
  # normal mode only: never mark the boot path unsafe from self-test/dry-run
  [ "$MODE" = "normal" ] || return 0

  local reason="$1"

  _ensure_state_dir 2>/dev/null || return 0
  [ -e "$SENTINEL" ] && return 0

  printf 'reason=%s\ndate=%s\n' "$reason" "$(date -Iseconds)" > "$SENTINEL" 2>/dev/null || true
  chmod 0600 "$SENTINEL" 2>/dev/null || true
}

die_stage() {
  local stage="$1"
  local detail="$2"

  _write_last_error "$stage" "$detail"
  _ensure_sentinel "${stage}: ${detail}"
  log "FAIL stage=${stage} mode=${MODE}: ${detail}"
  exit 1
}

_cleanup() {
  local m
  local d
  local f

  for m in "${SCRATCH_MOUNTS[@]}"; do
    [ -n "$m" ] && mountpoint -q "$m" 2>/dev/null && umount "$m" 2>/dev/null || true
  done

  for d in "${SCRATCH_DIRS[@]}"; do
    [ -n "$d" ] && rmdir "$d" 2>/dev/null || true
  done

  for f in "${SCRATCH_FILES[@]}"; do
    [ -n "$f" ] && rm -f "$f" 2>/dev/null || true
  done
}

trap _cleanup EXIT

require_root() {
  [ "$(id -u)" -eq 0 ] || {
    log "must run as root"
    exit 1
  }
}

acquire_lock() {
  [ -L "$LOCK" ] && die_stage "lock" "lock path is a symlink: $LOCK"

  exec 9>"$LOCK" || die_stage "lock" "cannot open lock: $LOCK"
  flock -n 9 || die_stage "lock" "another instance holds $LOCK"
}

preflight() {
  require_root

  local b
  for b in sbsign sbverify pesign bootctl mkfs.vfat mount umount losetup sha256sum logger install awk flock stat; do
    command -v "$b" >/dev/null 2>&1 || die_stage "preflight" "missing binary: $b"
  done

  [ -r "$SOURCE_EFI" ] || die_stage "preflight" "source loader unreadable: $SOURCE_EFI"
  [ -r "$DB_KEY" ] || die_stage "preflight" "db key unreadable: $DB_KEY"
  [ -r "$DB_CRT" ] || die_stage "preflight" "db cert unreadable: $DB_CRT"

  # State-dir creation is a NORMAL-mode action only.
  if [ "$MODE" = "normal" ]; then
    _ensure_state_dir || die_stage "preflight" "cannot create state dir: $STATE_DIR"
  fi
}

sign_source() {
  local dir
  local tmp

  dir="$(dirname "$SIGNED")"
  tmp="$(mktemp "${dir}/.systemd-bootx64.efi.signed.XXXXXX")" || die_stage "sign" "mktemp failed in $dir"
  SCRATCH_FILES+=("$tmp")

  sbsign \
    --key "$DB_KEY" \
    --cert "$DB_CRT" \
    --output "$tmp" \
    "$SOURCE_EFI" >/dev/null 2>&1 || die_stage "sign" "sbsign failed on $SOURCE_EFI"

  chmod 0644 "$tmp" 2>/dev/null || true
  mv -f "$tmp" "$SIGNED" || die_stage "sign" "atomic mv to $SIGNED failed"

  log "signed: $SOURCE_EFI -> $SIGNED ($(_sha "$SIGNED"))"
}

verify_signed() {
  sbverify --cert "$DB_CRT" "$SIGNED" >/dev/null 2>&1 \
    || die_stage "verify-signed" "sbverify rejected $SIGNED against db.crt"

  [ -s "$SIGNED" ] || die_stage "verify-signed" "$SIGNED is empty"

  local sc
  sc="$(_signer_count "$SIGNED")"

  [ "${sc:-0}" -eq 1 ] || die_stage "verify-signed" "$SIGNED signer_count=${sc:-0} (want exactly 1)"

  log "verify-signed OK: $SIGNED sbverify-clean, signer_count=1"
}

loopback_checkpoint() {
  local signed_sha
  local src_sha
  local mnt
  local img
  local got_sys
  local got_boot
  local rc

  signed_sha="$(_sha "$SIGNED")"
  src_sha="$(_sha "$SOURCE_EFI")"

  mnt="$(mktemp -d /var/tmp/tboot-sbl-ckpt.XXXXXX)" || die_stage "checkpoint" "mktemp -d failed"
  SCRATCH_DIRS+=("$mnt")

  img="${mnt}.img"
  SCRATCH_FILES+=("$img")

  dd if=/dev/zero of="$img" bs=1M count=64 status=none 2>/dev/null \
    || die_stage "checkpoint" "scratch image alloc failed"

  mkfs.vfat "$img" >/dev/null 2>&1 \
    || die_stage "checkpoint" "mkfs.vfat failed"

  mount -o loop "$img" "$mnt" \
    || die_stage "checkpoint" "loop mount failed"

  SCRATCH_MOUNTS+=("$mnt")

  mountpoint -q "$mnt" \
    || die_stage "checkpoint" "scratch ESP not a mountpoint"

  rc=0
  SYSTEMD_RELAX_ESP_CHECKS=1 bootctl --no-variables --esp-path="$mnt" install >/dev/null 2>&1 || rc=$?

  got_sys="$(_sha "${mnt}/EFI/systemd/systemd-bootx64.efi")"
  got_boot="$(_sha "${mnt}/EFI/BOOT/BOOTX64.EFI")"

  umount "$mnt" 2>/dev/null || true
  rmdir "$mnt" 2>/dev/null || true
  rm -f "$img" 2>/dev/null || true

  [ "$rc" -eq 0 ] || die_stage "checkpoint" "sandbox bootctl install rc=$rc (real ESP NOT touched)"

  # BOTH targets must equal .efi.signed and differ from the unsigned source.
  [ "$got_sys" = "$signed_sha" ] \
    || die_stage "checkpoint" "sandbox /EFI/systemd != .efi.signed (got=$got_sys signed=$signed_sha): real ESP NOT touched"

  [ "$got_boot" = "$signed_sha" ] \
    || die_stage "checkpoint" "sandbox /EFI/BOOT/BOOTX64.EFI != .efi.signed (got=$got_boot signed=$signed_sha): real ESP NOT touched"

  [ "$got_sys" != "$src_sha" ] \
    || die_stage "checkpoint" "sandbox /EFI/systemd == unsigned source: real ESP NOT touched"

  [ "$got_boot" != "$src_sha" ] \
    || die_stage "checkpoint" "sandbox /EFI/BOOT == unsigned source: real ESP NOT touched"

  log "checkpoint PASS: bootctl install selected .efi.signed for BOTH targets in sandbox ESP ($signed_sha)"
}

real_install() {
  bootctl --no-variables install >/dev/null 2>&1 \
    || die_stage "install" "real bootctl --no-variables install failed"

  log "propagated: bootctl --no-variables install -> ESP"
}

verify_real_esp() {
  local signed_sha
  local a
  local b
  local sa
  local sb

  signed_sha="$(_sha "$SIGNED")"
  a="$(_sha "$ESP_SYSTEMD")"
  b="$(_sha "$ESP_BOOT")"

  [ "$a" = "$signed_sha" ] \
    || die_stage "verify-esp" "$ESP_SYSTEMD sha=$a != signed $signed_sha"

  [ "$b" = "$signed_sha" ] \
    || die_stage "verify-esp" "$ESP_BOOT sha=$b != signed $signed_sha"

  sbverify --cert "$DB_CRT" "$ESP_SYSTEMD" >/dev/null 2>&1 \
    || die_stage "verify-esp" "$ESP_SYSTEMD sbverify failed"

  sbverify --cert "$DB_CRT" "$ESP_BOOT" >/dev/null 2>&1 \
    || die_stage "verify-esp" "$ESP_BOOT sbverify failed"

  sa="$(_signer_count "$ESP_SYSTEMD")"
  [ "${sa:-0}" -eq 1 ] \
    || die_stage "verify-esp" "$ESP_SYSTEMD signer_count=${sa:-0} (want 1)"

  sb="$(_signer_count "$ESP_BOOT")"
  [ "${sb:-0}" -eq 1 ] \
    || die_stage "verify-esp" "$ESP_BOOT signer_count=${sb:-0} (want 1)"

  log "verify-esp OK: both ESP loaders == .efi.signed, sbverify-clean, signer_count=1"
}

write_success() {
  _ensure_state_dir 2>/dev/null || true

  {
    printf 'epoch=%s\n' "$(date +%s)"
    printf 'date=%s\n'  "$(date -Iseconds)"
    printf 'signed_sha=%s\n' "$(_sha "$SIGNED")"
    printf 'helper_version=%s\n' "$VERSION"
  } > "${LAST_SUCCESS}.tmp" 2>/dev/null && mv -f "${LAST_SUCCESS}.tmp" "$LAST_SUCCESS" 2>/dev/null || true

  chmod 0600 "$LAST_SUCCESS" 2>/dev/null || true
}

mode_self_test() {
  preflight

  log "self-test PASS: root, binaries, keys, source. No signing, no install, no state-dir, no sentinel."
  printf 'self-test OK\n'
}

mode_dry_run() {
  preflight

  local dir
  local tmp
  local src_sha
  local dry_sha
  local sc

  dir="$(mktemp -d /var/tmp/tboot-sbl-dry.XXXXXX)" || die_stage "dry-run" "mktemp -d failed"
  SCRATCH_DIRS+=("$dir")

  tmp="${dir}/systemd-bootx64.efi.signed"
  SCRATCH_FILES+=("$tmp")

  sbsign \
    --key "$DB_KEY" \
    --cert "$DB_CRT" \
    --output "$tmp" \
    "$SOURCE_EFI" >/dev/null 2>&1 || die_stage "dry-run" "sbsign failed (scratch)"

  sbverify --cert "$DB_CRT" "$tmp" >/dev/null 2>&1 \
    || die_stage "dry-run" "scratch sbverify failed"

  sc="$(_signer_count "$tmp")"
  [ "${sc:-0}" -eq 1 ] || die_stage "dry-run" "scratch signer_count=${sc:-0} (want 1)"

  src_sha="$(_sha "$SOURCE_EFI")"
  dry_sha="$(_sha "$tmp")"

  rm -f "$tmp" 2>/dev/null || true
  rmdir "$dir" 2>/dev/null || true

  log "dry-run OK: scratch sign+verify+signer_count passed (src=$src_sha signed=$dry_sha). No BOOTLIBDIR/ESP/state-dir/sentinel mutation. Loopback checkpoint + real install run only in normal mode / B.5."
  printf 'dry-run OK (no BOOTLIBDIR/ESP/state mutation)\n'
}

mode_normal() {
  acquire_lock
  preflight
  sign_source
  verify_signed
  loopback_checkpoint
  real_install
  verify_real_esp
  write_success

  log "SUCCESS: source signed, checkpoint passed (both targets), ESP propagated and verified."
}

main() {
  case "${1:-}" in
    --version)
      printf '%s %s\n' "$TAG" "$VERSION"
      exit 0
      ;;

    --help|-h)
      cat <<USAGE
$TAG $VERSION

Usage: $TAG [--self-test|--dry-run|--version|--help]

  (no args)
      normal: sign source loader, prove .efi.signed selection for BOTH
      ESP targets in a loopback ESP, then 'bootctl --no-variables install'
      to the real ESP, with full fail-closed verification
      (sha match + sbverify + signer_count==1). May create state-dir,
      helper-last-error, helper-last-success, and UNSAFE-TO-REBOOT on failure.

  --self-test
      preflight only. No mutation, no state-dir, no sentinel.

  --dry-run
      preflight + sign to scratch + sbverify + signer_count. No
      BOOTLIBDIR/ESP/state-dir/sentinel mutation.
USAGE
      exit 0
      ;;

    --self-test)
      MODE="self-test"
      mode_self_test
      exit 0
      ;;

    --dry-run)
      MODE="dry-run"
      mode_dry_run
      exit 0
      ;;

    "")
      MODE="normal"
      mode_normal
      exit 0
      ;;

    *)
      log "unknown argument: $1"
      exit 2
      ;;
  esac
}

main "$@"
TBOOT_SBLOADER_HELPER_EOF
```

**Decider**: `tboot-sbloader-posttrans` (sha `b8d9a04c30cd39ee3600fb3ff8f581d7cf284f7f4aa639f03cf0fd733e705f0c`, 315 lines):

```bash
cat > /tmp/tboot-sbloader-posttrans.staged <<'TBOOT_SBLOADER_POSTTRANS_EOF'
#!/usr/bin/env bash
# tboot-sbloader-posttrans: B.4 systemd-boot signing DECIDER (read-and-decide)
# Version: 1.0.0
# Role: compute the 5-entry boot-loader manifest, compare to the stored baseline,
#       and decide. On confirmed drift, invoke the helper (which signs and
#       propagates to the ESP). This script NEVER signs, NEVER runs the loader
#       installer, and NEVER writes the ESP: it only reads, hashes, compares,
#       records its own state files, manages the shared sentinel, and calls the
#       helper.
# Future canonical path: /usr/local/sbin/tboot-sbloader-posttrans (UsrMerge-aware).
#
# State layout: B.4 single locked dir /var/lib/tboot-sbloader (Option A):
#   sbloader-manifest.baseline, last-computed-manifest,
#   last-decision, last-error                 (this decider)
#   helper-last-error, helper-last-success    (helper: tboot-sbloader-helper)
#   UNSAFE-TO-REBOOT                           (shared sentinel)

set -euo pipefail
umask 077

readonly VERSION="1.0.0"
readonly TAG="tboot-sbloader-posttrans"

# Inputs (read-only here)
readonly SOURCE_EFI="/usr/lib/systemd/boot/efi/systemd-bootx64.efi"
readonly SIGNED="/usr/lib/systemd/boot/efi/systemd-bootx64.efi.signed"
readonly ESP_SYSTEMD="/boot/efi/EFI/systemd/systemd-bootx64.efi"
readonly ESP_BOOT="/boot/efi/EFI/BOOT/BOOTX64.EFI"
readonly PKG="systemd-boot-unsigned"

# Helper: sole privileged worker; the ONLY external command the decider invokes.
readonly HELPER="/usr/local/sbin/tboot-sbloader-helper"

# Locked Option A single state dir
readonly STATE_DIR="/var/lib/tboot-sbloader"
readonly BASELINE="${STATE_DIR}/sbloader-manifest.baseline"
readonly LAST_COMPUTED="${STATE_DIR}/last-computed-manifest"
readonly LAST_DECISION="${STATE_DIR}/last-decision"
readonly LAST_ERROR="${STATE_DIR}/last-error"
readonly SENTINEL="${STATE_DIR}/UNSAFE-TO-REBOOT"

readonly LOCK="/run/tboot-sbloader-posttrans.lock"

# MODE set by main() before any mode runs: normal|self-test|debug-print|prime
MODE="normal"

# Evaluation result globals
DECISION=""
REASON=""

log() {
  logger -t "$TAG" -- "$*" 2>/dev/null || true
  printf '%s %s\n' "$TAG" "$*" >&2
}

_cleanup() {
  rm -f "${BASELINE}.tmp" "${LAST_COMPUTED}.tmp" "${LAST_DECISION}.tmp" \
        "${LAST_ERROR}.tmp" "${SENTINEL}.tmp" 2>/dev/null || true
}
trap _cleanup EXIT

# set -e safe: never aborts the script; empty output on missing file.
_sha() {
  sha256sum "$1" 2>/dev/null | awk '{print $1}' || true
}

_sha_or() {
  # $1=path  $2=fallback-token
  local s
  s="$(_sha "$1")"
  if [ -n "$s" ]; then printf '%s' "$s"; else printf '%s' "$2"; fi
}

get_field() {
  # $1=manifest-string  $2=key
  printf '%s\n' "$1" | awk -F= -v k="$2" '$1==k {print $2; exit}'
}

require_root() {
  [ "$(id -u)" -eq 0 ] || { log "must run as root"; exit 1; }
}

_check_binaries() {
  local b
  for b in sha256sum rpm awk logger install flock date id cat head mv stat; do
    command -v "$b" >/dev/null 2>&1 || die "preflight" "missing binary: $b"
  done
}

_ensure_state_dir() {
  if [ ! -e "$STATE_DIR" ]; then
    install -d -m 0700 -o root -g root "$STATE_DIR" || return 1
  fi
  [ ! -L "$STATE_DIR" ] || return 1
  [ -d "$STATE_DIR" ] || return 1
  [ "$(stat -c '%U:%G' "$STATE_DIR" 2>/dev/null)" = "root:root" ] || return 1
  [ "$(stat -c '%a' "$STATE_DIR" 2>/dev/null)" = "700" ] || return 1
  [ -w "$STATE_DIR" ] || return 1
}

die() {
  # config/precondition error: NO sentinel (boot path untouched). Records
  # last-error only in state-writing modes (normal|prime).
  local stage="$1" detail="$2"
  log "FAIL stage=${stage} mode=${MODE}: ${detail}"
  case "$MODE" in
    normal|prime)
      _ensure_state_dir 2>/dev/null || true
      {
        printf 'epoch=%s\n' "$(date +%s)"
        printf 'date=%s\n'  "$(date -Iseconds)"
        printf 'stage=%s\n' "$stage"
        printf 'detail=%s\n' "$detail"
        printf 'decider_version=%s\n' "$VERSION"
      } > "${LAST_ERROR}.tmp" 2>/dev/null && mv -f "${LAST_ERROR}.tmp" "$LAST_ERROR" 2>/dev/null || true
      chmod 0600 "$LAST_ERROR" 2>/dev/null || true
      ;;
  esac
  exit 1
}

acquire_lock() {
  [ -L "$LOCK" ] && die "lock" "lock path is a symlink: $LOCK"
  exec 9>"$LOCK" || die "lock" "cannot open lock: $LOCK"
  flock -n 9 || die "lock" "another instance holds $LOCK"
}

compute_manifest() {
  # Emits the frozen 5-entry manifest in fixed order.
  local nevra se ss es eb
  nevra="$(rpm -q "$PKG" 2>/dev/null | head -1 || true)"
  case "$nevra" in
    *"is not installed"*|"") nevra="NOTINSTALLED" ;;
  esac
  se="$(_sha_or "$SOURCE_EFI" MISSING)"
  ss="$(_sha_or "$SIGNED" ABSENT)"
  es="$(_sha_or "$ESP_SYSTEMD" MISSING)"
  eb="$(_sha_or "$ESP_BOOT" MISSING)"
  printf 'nevra=%s\n'        "$nevra"
  printf 'src_efi=%s\n'      "$se"
  printf 'src_signed=%s\n'   "$ss"
  printf 'esp_systemd=%s\n'  "$es"
  printf 'esp_boot=%s\n'     "$eb"
}

_converged_shape() {
  # rc 0 iff: src_signed present AND both ESP copies == src_signed
  local m="$1" ss es eb
  ss="$(get_field "$m" src_signed)"
  es="$(get_field "$m" esp_systemd)"
  eb="$(get_field "$m" esp_boot)"
  [ "$ss" != "ABSENT" ] && [ "$ss" != "MISSING" ] && [ "$es" = "$ss" ] && [ "$eb" = "$ss" ]
}

evaluate() {
  # $1=current manifest ; sets DECISION + REASON.
  # D-1: decision=unchanged REQUIRES baseline equality AND converged-shape.
  local m="$1" base
  base="$(cat "$BASELINE" 2>/dev/null || true)"
  local equal=0 shape=0
  if [ -n "$base" ] && [ "$m" = "$base" ]; then equal=1; fi
  if _converged_shape "$m"; then shape=1; fi
  if [ "$equal" = 1 ] && [ "$shape" = 1 ]; then
    DECISION="unchanged"
    REASON="manifest==baseline AND converged-shape"
  else
    DECISION="drift"
    REASON="equal=${equal} shape=${shape}"
  fi
}

write_last_decision() {
  # $1=decision $2=detail
  _ensure_state_dir 2>/dev/null || return 0
  {
    printf 'epoch=%s\n' "$(date +%s)"
    printf 'date=%s\n'  "$(date -Iseconds)"
    printf 'decision=%s\n' "$1"
    printf 'detail=%s\n' "$2"
    printf 'decider_version=%s\n' "$VERSION"
  } > "${LAST_DECISION}.tmp" 2>/dev/null && mv -f "${LAST_DECISION}.tmp" "$LAST_DECISION" 2>/dev/null || true
  chmod 0600 "$LAST_DECISION" 2>/dev/null || true
}

_write_manifest_file() {
  # $1=dest  stdin=manifest ; atomic, same-dir
  cat > "${1}.tmp" && mv -f "${1}.tmp" "$1"
  chmod 0600 "$1" 2>/dev/null || true
}

mode_self_test() {
  require_root
  _check_binaries
  if [ -x "$HELPER" ]; then
    log "self-test: helper present and executable ($HELPER)"
  else
    log "self-test WARN: helper absent/not-executable (expected pre-install): $HELPER"
  fi
  log "self-test PASS: root + binaries OK. No compute, no state writes, no helper invocation."
  printf 'self-test OK\n'
}

mode_debug_print() {
  require_root
  local m
  m="$(compute_manifest)"
  printf -- '--- computed manifest ---\n%s\n' "$m"
  if [ -f "$BASELINE" ]; then
    printf -- '--- baseline present: %s ---\n' "$BASELINE"
  else
    printf -- '--- baseline ABSENT (normal mode would FAIL CLOSED; run --prime) ---\n'
  fi
  evaluate "$m"
  printf -- '--- would-be decision: %s (%s) ---\n' "$DECISION" "$REASON"
  printf 'debug-print OK (no state writes, no helper, no lock)\n'
}

mode_prime() {
  require_root
  _check_binaries
  _ensure_state_dir || die "prime" "cannot create $STATE_DIR"
  local m
  m="$(compute_manifest)"
  printf '%s\n' "$m" | _write_manifest_file "$BASELINE" || die "prime" "baseline write failed"
  printf '%s\n' "$m" | _write_manifest_file "$LAST_COMPUTED" || true
  write_last_decision "primed" "baseline established"
  log "prime OK: baseline written ($(_sha "$BASELINE"))"
  printf 'prime OK\n'
}

mode_normal() {
  acquire_lock
  require_root
  _check_binaries
  _ensure_state_dir || die "normal" "cannot create $STATE_DIR"

  [ -f "$BASELINE" ] || die "baseline" "baseline absent; refuse to auto-init in normal mode: run: $TAG --prime"

  local cur
  cur="$(compute_manifest)"
  printf '%s\n' "$cur" | _write_manifest_file "$LAST_COMPUTED" || true

  evaluate "$cur"
  if [ "$DECISION" = "unchanged" ]; then
    write_last_decision "unchanged" "$REASON"
    log "decision=unchanged: $REASON"
    exit 0
  fi

  # --- drift path: mark unsafe BEFORE invoking helper ---
  local sentinel_tmp="${SENTINEL}.tmp.$$"
  if ! printf 'reason=%s\ndate=%s\n' "drift before sbloader propagation: $REASON" "$(date -Iseconds)" \
      > "$sentinel_tmp" 2>/dev/null \
      || ! chmod 0600 "$sentinel_tmp" 2>/dev/null \
      || ! mv -f "$sentinel_tmp" "$SENTINEL" 2>/dev/null; then
    rm -f "$sentinel_tmp" 2>/dev/null || true
    die "sentinel" "cannot mark system unsafe; helper not invoked"
  fi
  write_last_decision "drift-detected" "$REASON; invoking helper"
  log "drift: $REASON: invoking helper"

  [ -x "$HELPER" ] || die "helper" "helper not executable: $HELPER (sentinel retained)"

  local hrc=0
  "$HELPER" </dev/null || hrc=$?
  if [ "$hrc" -ne 0 ]; then
    write_last_decision "helper-failure" "helper exit=$hrc; sentinel retained"
    die "helper" "helper exited $hrc"
  fi

  # helper succeeded -> confirm converged-shape, then advance baseline + clear sentinel
  local post
  post="$(compute_manifest)"
  if _converged_shape "$post"; then
    printf '%s\n' "$post" | _write_manifest_file "$BASELINE" || die "baseline" "baseline update failed"
    printf '%s\n' "$post" | _write_manifest_file "$LAST_COMPUTED" || true
    rm -f "$SENTINEL" 2>/dev/null || true
    write_last_decision "helper-success" "baseline advanced; converged; sentinel cleared"
    log "helper-success: baseline advanced, converged, sentinel cleared"
    exit 0
  fi

  write_last_decision "helper-success-not-converged" "helper exit 0 but manifest not converged; sentinel retained"
  die "post-helper" "helper succeeded but system not converged (sentinel retained)"
}

main() {
  case "${1:-}" in
    --version) printf '%s %s\n' "$TAG" "$VERSION"; exit 0 ;;
    --help|-h)
      cat <<USAGE
$TAG $VERSION
Usage: $TAG [--self-test|--debug-print|--prime|--version|--help]
  (no args)      normal: compute manifest, compare to baseline, decide.
                 decision=unchanged ONLY IF manifest==baseline AND src_signed
                 present AND both ESP copies == src_signed. Otherwise drift ->
                 write UNSAFE-TO-REBOOT -> invoke helper -> on success advance
                 baseline and clear sentinel. Baseline absent => FAIL CLOSED
                 (run --prime in the correct gate). Never signs; never invokes
                 the loader installer; never writes the ESP.
  --self-test    root + binaries (+ helper presence, warn-only). No compute,
                 no state writes, no helper invocation.
  --debug-print  compute + show manifest and would-be decision. No state writes,
                 no helper, no lock.
  --prime        establish baseline from the current manifest.
USAGE
      exit 0 ;;
    --self-test)   MODE="self-test";   mode_self_test;   exit 0 ;;
    --debug-print) MODE="debug-print"; mode_debug_print; exit 0 ;;
    --prime)       MODE="prime";       mode_prime;       exit 0 ;;
    "")            MODE="normal";      mode_normal;      exit 0 ;;
    *) log "unknown argument: $1"; exit 2 ;;
  esac
}
main "$@"
TBOOT_SBLOADER_POSTTRANS_EOF
```

**Rule**: `60-tboot-sbloader.actions` (sha `2e83fcfcd237359d02a831cb381d40ca221d319108d66c264b847956c0418274`, 115 bytes, 1 line; published in B.5, not B.4):

```bash
printf '%s\n' 'post_transaction:systemd-boot-unsigned:in:enabled=host-only raise_error=1:/usr/local/sbin/tboot-sbloader-posttrans' > /tmp/60-tboot-sbloader.actions.staged
```

**Verify staged shas before proceeding (must match exactly):**

```bash
for f in helper posttrans; do sha256sum /tmp/tboot-sbloader-$f.staged; done
sha256sum /tmp/60-tboot-sbloader.actions.staged
# expect:
#   b9df2534b1539355a3c040ee296611f5da629e76467fccfcd44f6ed56325690f  /tmp/tboot-sbloader-helper.staged
#   b8d9a04c30cd39ee3600fb3ff8f581d7cf284f7f4aa639f03cf0fd733e705f0c  /tmp/tboot-sbloader-posttrans.staged
#   2e83fcfcd237359d02a831cb381d40ca221d319108d66c264b847956c0418274  /tmp/60-tboot-sbloader.actions.staged
```

### Commands

```bash
# --- Step 0 (host pve-host): pre-install snapshot (after Data% < 75% + qm status running) ---
lvs -o vg_name,lv_name,lv_size,data_percent,metadata_percent vm-storage
qm status 500
qm snapshot 500 module5-b4-pre-install-publish \
  --description "B.4 install/publish pre-install anchor. Staged artifacts only in /tmp; nothing in /usr/local/sbin or /var/lib/tboot-sbloader; .efi.signed absent; ESP 73b0c6c4...9f41; 60- rule UNpublished."
qm listsnapshot 500

# --- Step 1 (VM 500): install helper (atomic, UsrMerge-aware) ---
bash <<'B4_STEP1'
set -euo pipefail
SRC=/tmp/tboot-sbloader-helper.staged; DST=/usr/local/sbin/tboot-sbloader-helper
EXPECT=b9df2534b1539355a3c040ee296611f5da629e76467fccfcd44f6ed56325690f
fail(){ echo "ABORT: $*" >&2; exit 1; }
[ "$(sha256sum "$SRC"|awk '{print $1}')" = "$EXPECT" ] || fail "staged sha"
[ ! -e "$DST" ] && [ ! -e /usr/local/bin/tboot-sbloader-helper ] || fail "already present"
REALDIR="$(readlink -f "$(dirname "$DST")")"; [ -d "$REALDIR" ] || fail "realdir"
TMP="$(mktemp "${REALDIR}/.tboot-sbloader-helper.XXXXXX")"; trap 'rm -f "$TMP" 2>/dev/null||true' EXIT
install -m 0755 -o root -g root "$SRC" "$TMP"; mv -T "$TMP" "${REALDIR}/tboot-sbloader-helper"; trap - EXIT
[ "$(sha256sum "$DST"|awk '{print $1}')" = "$EXPECT" ] || fail "installed sha"
[ "$(stat -c '%a %U:%G' "$DST")" = "755 root:root" ] || fail "mode/owner"
"$DST" --version
echo "STEP 1 PASS"
B4_STEP1

# --- Step 2 (VM 500): install decider (atomic) ---
bash <<'B4_STEP2'
set -euo pipefail
SRC=/tmp/tboot-sbloader-posttrans.staged; DST=/usr/local/sbin/tboot-sbloader-posttrans
EXPECT=b8d9a04c30cd39ee3600fb3ff8f581d7cf284f7f4aa639f03cf0fd733e705f0c
fail(){ echo "ABORT: $*" >&2; exit 1; }
[ "$(sha256sum "$SRC"|awk '{print $1}')" = "$EXPECT" ] || fail "staged sha"
[ "$(sha256sum /usr/local/sbin/tboot-sbloader-helper|awk '{print $1}')" = b9df2534b1539355a3c040ee296611f5da629e76467fccfcd44f6ed56325690f ] || fail "helper drift"
[ ! -e "$DST" ] && [ ! -e /usr/local/bin/tboot-sbloader-posttrans ] || fail "already present"
REALDIR="$(readlink -f "$(dirname "$DST")")"
TMP="$(mktemp "${REALDIR}/.tboot-sbloader-posttrans.XXXXXX")"; trap 'rm -f "$TMP" 2>/dev/null||true' EXIT
install -m 0755 -o root -g root "$SRC" "$TMP"; mv -T "$TMP" "${REALDIR}/tboot-sbloader-posttrans"; trap - EXIT
[ "$(sha256sum "$DST"|awk '{print $1}')" = "$EXPECT" ] || fail "installed sha"
[ "$(stat -c '%a %U:%G' "$DST")" = "755 root:root" ] || fail "mode/owner"
"$DST" --version
echo "STEP 2 PASS"
B4_STEP2

# --- Step 3 (VM 500): create state-dir 0700 ---
install -d -m 0700 -o root -g root /var/lib/tboot-sbloader
stat -c 'mode=%a owner=%U:%G' /var/lib/tboot-sbloader   # expect mode=700 owner=root:root
find /var/lib/tboot-sbloader -mindepth 1 | wc -l         # expect 0

# --- Step 4 (VM 500): non-mutating self-tests + debug-print (state-dir must stay empty) ---
/usr/local/sbin/tboot-sbloader-helper --self-test
/usr/local/sbin/tboot-sbloader-posttrans --self-test
/usr/local/sbin/tboot-sbloader-posttrans --debug-print   # baseline ABSENT -> "drift (equal=0 shape=0)"
find /var/lib/tboot-sbloader -mindepth 1 | wc -l         # still 0 (no writes)

# --- Step 5 (VM 500): prime NON-CONVERGED baseline + D-1 live proof ---
/usr/local/sbin/tboot-sbloader-posttrans --prime          # baseline sha 2547bd82...cc5e (src_signed=ABSENT)
/usr/local/sbin/tboot-sbloader-posttrans --debug-print | grep -q 'drift (equal=1 shape=0)' \
  && echo "D-1 PROVEN: baseline-equal + non-converged => drift (never unchanged)" \
  || { echo "D-1 VIOLATED"; exit 1; }

# --- Step 6 (VM 500): mask the native updater (ONLY this unit, per 06F N.5) ---
systemctl mask systemd-boot-update.service
systemctl is-enabled systemd-boot-update.service          # expect: masked
readlink -f /etc/systemd/system/systemd-boot-update.service  # expect: /dev/null

# --- Step 7 (VM 500): read-only post-gate verification sweep ---
# Reconstructed from the Step 38 invariant list (see Expected output) and the
# canonical Step 39 Gate 0 read-only check idioms (same helper functions, same
# pinned values). READ-ONLY: no writes, no mask changes, no ESP/state mutation
# (`--debug-print` is the proven read-only keystone). Expect final line: STEP 7 PASS.
bash <<'B4_STEP7'
set -uo pipefail
fail=0
ok(){ printf 'PASS  %s\n' "$1"; }
no(){ printf 'FAIL  %s\n' "$1"; fail=$((fail+1)); }
chk(){ if [ "$2" = "$3" ]; then ok "$1 ($2)"; else no "$1 got=$2 want=$3"; fi; }
shachk(){ if [ ! -e "$1" ]; then no "MISSING (sha): $1"; return; fi
  g="$(sha256sum "$1" | awk '{print $1}')"
  if [ "$g" = "$2" ]; then ok "sha $1"; else no "sha $1 got=$g want=$2"; fi; }
absent(){ if [ -e "$1" ]; then no "PRESENT (must be absent): $1"; else ok "absent: $1"; fi; }
permchk(){ if [ ! -e "$1" ]; then no "MISSING (perms): $1"; return; fi
  g="$(stat -c '%a %U:%G' "$1")"; if [ "$g" = "$2" ]; then ok "perms $1 ($g)"; else no "perms $1 got=$g want=$2"; fi; }
masked(){ s="$(systemctl is-enabled "$1" 2>/dev/null || true)"; if [ "$s" = "masked" ]; then ok "masked: $1"; else no "NOT masked: $1 (is-enabled=$s)"; fi; }
notmasked(){ s="$(systemctl is-enabled "$1" 2>/dev/null || true)"; if [ "$s" = "masked" ]; then no "MASKED (must not be): $1"; else ok "not-masked: $1 (is-enabled=$s)"; fi; }

echo "===== STEP 7: B.4 post-gate read-only sweep ====="
echo "host=$(hostname)  kver=$(uname -r)  utc=$(date -u +%FT%TZ)"

echo "---- artifacts: B.4 chain (sha + perms) ----"
shachk  /usr/local/sbin/tboot-sbloader-helper    b9df2534b1539355a3c040ee296611f5da629e76467fccfcd44f6ed56325690f
shachk  /usr/local/sbin/tboot-sbloader-posttrans b8d9a04c30cd39ee3600fb3ff8f581d7cf284f7f4aa639f03cf0fd733e705f0c
permchk /usr/local/sbin/tboot-sbloader-helper    "755 root:root"
permchk /usr/local/sbin/tboot-sbloader-posttrans "755 root:root"

echo "---- artifacts: B.2 chain (must be untouched) ----"
shachk  /usr/local/sbin/tboot-dnf-posttrans      1e75110935a72fc9433c9b4bf5688d26bc6d3bd1e02f4388b5e1f32ff1e5cc05
shachk  /usr/local/sbin/tboot-dnf-helper         937afc7a0d199bf7b7d1a5f5bcccf45c37c0f4f7dd0ad924b2567c4da7125e44

echo "---- state-dir: exactly 3 files, 0700 dir / 0600 files ----"
permchk /var/lib/tboot-sbloader                            "700 root:root"
n=$(find /var/lib/tboot-sbloader -mindepth 1 | wc -l); chk "state-dir entry count" "$n" "3"
permchk /var/lib/tboot-sbloader/sbloader-manifest.baseline "600 root:root"
permchk /var/lib/tboot-sbloader/last-computed-manifest     "600 root:root"
permchk /var/lib/tboot-sbloader/last-decision              "600 root:root"

echo "---- baseline non-converged + last-computed == baseline + last-decision primed ----"
shachk /var/lib/tboot-sbloader/sbloader-manifest.baseline 2547bd821f978b2a85a919a11d08b32c5062925a9da7fdcf8e1d8a6e7ab4cc5e
if grep -q 'src_signed=ABSENT' /var/lib/tboot-sbloader/sbloader-manifest.baseline 2>/dev/null; then ok "baseline src_signed=ABSENT (non-converged)"; else no "baseline src_signed not ABSENT"; fi
if cmp -s /var/lib/tboot-sbloader/last-computed-manifest /var/lib/tboot-sbloader/sbloader-manifest.baseline; then ok "last-computed-manifest == baseline"; else no "last-computed-manifest != baseline"; fi
if grep -q 'primed' /var/lib/tboot-sbloader/last-decision 2>/dev/null; then ok "last-decision shows primed"; else no "last-decision not primed"; fi

echo "---- must-be-absent: unarmed / no helper run / no errors ----"
absent /usr/lib/systemd/boot/efi/systemd-bootx64.efi.signed
absent /var/lib/tboot-sbloader/UNSAFE-TO-REBOOT
absent /var/lib/tboot-sbloader/last-error
absent /var/lib/tboot-sbloader/helper-last-success
absent /var/lib/tboot-sbloader/helper-last-error
n60=$(find /etc/dnf/libdnf5-plugins/actions.d/ -maxdepth 1 -name '60-*' 2>/dev/null | wc -l)
if [ "$n60" -eq 0 ]; then ok "actions.d has no 60-* (60- rule UNpublished)"; else no "60-* PRESENT (already armed): $n60"; fi

echo "---- source .efi + ESP loaders (unsigned, pre-B.5) ----"
shachk /usr/lib/systemd/boot/efi/systemd-bootx64.efi 3d7e17b12a37b288b794765055429e7e6bff39af59e516f478657d763685fb4b
shachk /boot/efi/EFI/systemd/systemd-bootx64.efi     73b0c6c4da8386a812749de088a442269a2f6869974d7f03e87643d52a449f41
shachk /boot/efi/EFI/BOOT/BOOTX64.EFI                73b0c6c4da8386a812749de088a442269a2f6869974d7f03e87643d52a449f41

echo "---- mask state: ONLY update.service masked ----"
masked    systemd-boot-update.service
ml="$(readlink -f /etc/systemd/system/systemd-boot-update.service 2>/dev/null || true)"
chk "update.service mask target" "$ml" "/dev/null"
notmasked systemd-boot-clear-sysfail.service
notmasked systemd-bootctl@.service
notmasked systemd-bootctl.socket
notmasked systemd-boot-random-seed.service

echo "---- D-1: primed but non-converged => drift (never unchanged) ----"
dp="$(/usr/local/sbin/tboot-sbloader-posttrans --debug-print 2>&1 || true)"
if printf '%s' "$dp" | grep -q 'drift (equal=1 shape=0)'; then ok "decider --debug-print: drift (equal=1 shape=0)"; else no "decider --debug-print not 'drift (equal=1 shape=0)'"; fi

echo "---- runtime PCR 11 == stored expected ----"
pcr11="$(cat /sys/class/tpm/tpm0/pcr-sha256/11 2>/dev/null | tr 'a-f' 'A-F' | tr -d '[:space:]')"
chk "runtime PCR 11" "$pcr11" "28E66CE224C9BB0711059470D77103A5AB5A5C1674C03D1BA1563C2FEE52C90F"

echo "===== SUMMARY ====="
if [ "$fail" -eq 0 ]; then echo "STEP 7 PASS"; else echo "STEP 7: ${fail} CHECK(S) FAILED"; fi
B4_STEP7

# --- Step 8 (host pve-host): closure snapshot ---
qm snapshot 500 module5-b4-installed \
  --description "B.4 install/publish CLOSED. Chain installed + primed + masked, UNARMED. helper b9df2534..690f; decider b8d9a04c..5f0c; baseline 2547bd82..cc5e (PRE-CONVERGENCE src_signed=ABSENT); systemd-boot-update.service masked (-> /dev/null); .efi.signed ABSENT; 60- UNPUBLISHED (B.5). ESP 73b0c6c4..9f41; B.2 chain + 50- intact; runtime PCR11 = 28E66CE2..C90F. Rollback target for B.5."
qm listsnapshot 500
```

### Expected output

Step 1/2: `--version` prints `...-r4` (helper) and `1.0.0` (decider); both resolve to `/usr/local/bin` via UsrMerge (`06F` I.3), shared inode source/canonical. Step 3: `mode=700 owner=root:root`, 0 entries. Step 4: helper self-test PASS (preflight only, no state/sentinel), decider self-test PASS (helper-present non-warn), `--debug-print` shows `src_signed=ABSENT` + `would-be decision: drift (equal=0 shape=0)`; state-dir still 0 entries. Step 5: baseline `2547bd82...cc5e`, `last-decision=primed`, no sentinel; **post-prime `--debug-print` = `drift (equal=1 shape=0)`** (D-1 proof; `06F` N.6). Step 6: `is-enabled`=masked, symlink `/dev/null`; the other four `bootctl`-referencing units NOT masked. Step 8: `module5-b4-installed` present, `current` directly below, VM `status: running`.

### Stop condition

- **Any staged-sha mismatch (Step 1/2):** the artifact in `/tmp` is not the read-back-validated one. Re-stage from the canonical source; do not install.
- **Step 5 post-prime `--debug-print` reports `unchanged`:** D-1 is broken (baseline-equality alone yielded a no-op). Halt; do not mask, do not snapshot; investigate `evaluate()`/`_converged_shape()`. Rollback to `module5-b4-pre-install-publish`.
- **Step 6 masks more than `systemd-boot-update.service`, or any of the four F3-cleared units shows `masked`:** over-masking (would break e.g. the loader random seed, `06F` N.5). Unmask the extras; re-verify.
- **Step 7 finds `.efi.signed` present, the `60-` rule published, a sentinel, or any ESP/source/B.2/PCR-11 drift:** the chain armed early or something mutated outside the gate. Halt; rollback to `module5-b4-pre-install-publish`.
- **Step 0/8 `qm snapshot` fails:** Proxmox-host issue (thin-pool Data% ≥ 75%, name collision, parent missing). Triage on host pve-host. Without the Step 8 closure snapshot, do not begin B.5.

### Snapshot point

- Step 0: `module5-b4-pre-install-publish` (pre-install anchor; rollback target for Steps 1–8).
- Step 8: `module5-b4-installed` (gate-close anchor; rollback target for B.5). Chain installed + primed (non-converged) + masked, **UNARMED**.

> [!note] Step 39 / Block B.5: executed and validated 2026-05-28; full procedure is documented as Step 39 below
> Publishing `/etc/dnf/libdnf5-plugins/actions.d/60-tboot-sbloader.actions` (atomic `mktemp`+`install`+`mv -T` from the staged `2e83fcfc...8274`) and the first controlled `dnf reinstall systemd-boot-unsigned` are Block B.5. On first fire the decider detects drift (primed baseline is non-converged), writes the sentinel, invokes the helper; the helper signs `systemd-bootx64.efi` → `.efi.signed`, verifies (sbverify + signer_count==1), runs the loopback both-target checkpoint (proves `bootctl --no-variables install` picks `.efi.signed`, fail-closed before touching `/boot/efi`; `06F` N.1), propagates to `/EFI/systemd/` + `/EFI/BOOT/BOOTX64.EFI`; the decider confirms converged-shape, advances the baseline, clears the sentinel. Then reboot validation: runtime PCR 11 byte-match, no LUKS passphrase. Closes the discipline gate on `dnf upgrade systemd*`.

## Step 39: Block B.5: arm, first-fire, converge, reboot-validate (closes the discipline gate on `dnf upgrade systemd*`)

**Goal:** Take the B.4 systemd-boot signing chain from installed + primed + masked + **UNARMED** to **armed + converged + reboot-validated**. Publish the single arming action `60-tboot-sbloader.actions`, fire one controlled `dnf reinstall systemd-boot-unsigned`, prove both signing chains converged, and confirm across a real reboot that LUKS still unlocks via TPM2 with no passphrase. Closes the `dnf upgrade systemd*` discipline gate. No new privileged source is added: the reproducible artifacts are the seven gate scripts below.
**Prerequisites:** Step 38 closed; rollback anchor `module5-b4-installed`. Entry state: helper `b9df2534…690f`, decider `b8d9a04c…5f0c`, baseline `2547bd82…cc5e` (`src_signed=ABSENT`, non-converged), `60-` rule staged at `/tmp/60-tboot-sbloader.actions.staged` (`2e83fcfc…8274`, 115 bytes) but UNPUBLISHED, `.efi.signed` absent, `systemd-boot-update.service` masked, B.2 chain intact, runtime PCR 11 = `28E66CE2…C90F`.
**Where to run:** Fedora VM 500 as root for Gates 0–4 and 6; Proxmox host pve-host for the Gate 5 and Gate 7 snapshots; Gate 6a is the reboot.

> [!important] Two-chain co-fire on first fire: expected, not a failure
> The B.2 rule (`50-`, empty `package_filter`) fires on every transaction; the B.4 rule (`60-`) is narrow (`systemd-boot-unsigned`). The first-fire transaction legitimately triggers **both**: B.2 rebuilds + re-signs both UKIs first (PCR 11 stays `28E66CE2…C90F`, baseline advances), then B.4 signs `systemd-bootx64.efi` → `.efi.signed` and converges. Consequence the gates handle: the B.2 baseline and both ESP UKIs advance during Gate 3, so their pre-B.5 pins are superseded: Gate 4 re-pins them. The booted UKI's file-sha changes while predicted PCR 11 stays byte-identical (PCR 11 measures section content, not the whole-file signature/timestamp), which is exactly why Gate 6 unlocks LUKS with no passphrase. Do not read the co-fire as a failure and do not roll back on it. Full assumption: `06C`; findings `06F` O.1–O.3, E.3, K.3.

### Step 39 commands

Seven gates, strictly one at a time. Read-only verification gates (0, 2, 4, 6b) tally all checks then exit non-zero on any failure; state-changing gates (1, 3) run `set -euo pipefail` **inside** the `bash <<'EOF'` child only (never at the interactive top level: `06F` N.2) and fail closed. Gate 5 is a host-side snapshot. Gate 6a is the reboot. Decider `--debug-print` is read-only (proven at B.4) and is used as the authoritative ESP-hash source so the harness never hardcodes ESP loader paths.

```bash
# --- GATE 0 (VM 500): read-only pre-arm verification (no publish, no DNF) ---
bash <<'EOF'
fail=0
ok(){ printf 'PASS  %s\n' "$1"; }
no(){ printf 'FAIL  %s\n' "$1"; fail=$((fail+1)); }
info(){ printf 'INFO  %s\n' "$1"; }
chk(){ if [ "$2" = "$3" ]; then ok "$1 ($2)"; else no "$1 got=$2 want=$3"; fi; }
shachk(){ if [ ! -e "$1" ]; then no "MISSING (sha): $1"; return; fi
  g="$(sha256sum "$1" | awk '{print $1}')"
  if [ "$g" = "$2" ]; then ok "sha $1"; else no "sha $1 got=$g want=$2"; fi; }
absent(){ if [ -e "$1" ]; then no "PRESENT (must be absent): $1"; else ok "absent: $1"; fi; }
present(){ if [ -e "$1" ]; then ok "present: $1"; else no "MISSING: $1"; fi; }
permchk(){ if [ ! -e "$1" ]; then no "MISSING (perms): $1"; return; fi
  g="$(stat -c '%a %U:%G' "$1")"; if [ "$g" = "$2" ]; then ok "perms $1 ($g)"; else no "perms $1 got=$g want=$2"; fi; }
masked(){ s="$(systemctl is-enabled "$1" 2>/dev/null || true)"; if [ "$s" = "masked" ]; then ok "masked: $1"; else no "NOT masked: $1 (is-enabled=$s)"; fi; }
notmasked(){ s="$(systemctl is-enabled "$1" 2>/dev/null || true)"; if [ "$s" = "masked" ]; then no "MASKED (must not be): $1"; else ok "not-masked: $1 (is-enabled=$s)"; fi; }

echo "===== GATE 0: B.5 read-only pre-arm verification ====="
echo "host=$(hostname)  kver=$(uname -r)  utc=$(date -u +%FT%TZ)"
echo

echo "---- 1. artifact SHA256 (B.4) ----"
shachk /usr/local/sbin/tboot-sbloader-helper    b9df2534b1539355a3c040ee296611f5da629e76467fccfcd44f6ed56325690f
shachk /usr/local/sbin/tboot-sbloader-posttrans b8d9a04c30cd39ee3600fb3ff8f581d7cf284f7f4aa639f03cf0fd733e705f0c
echo
echo "---- 1b. artifact SHA256 (B.2 + UKI hook, must be untouched) ----"
shachk /usr/local/sbin/tboot-dnf-posttrans      1e75110935a72fc9433c9b4bf5688d26bc6d3bd1e02f4388b5e1f32ff1e5cc05
shachk /usr/local/sbin/tboot-dnf-helper         937afc7a0d199bf7b7d1a5f5bcccf45c37c0f4f7dd0ad924b2567c4da7125e44
shachk /etc/kernel/install.d/80-tpm2-sign.install 5857e51d5551e05a7ca384f71529ec2fef07f5a6dbe40df70c6bbe49d720947e
echo

echo "---- 2. baseline manifest (SHA + content) ----"
shachk /var/lib/tboot-sbloader/sbloader-manifest.baseline 2547bd821f978b2a85a919a11d08b32c5062925a9da7fdcf8e1d8a6e7ab4cc5e
echo "      baseline content:"
sed 's/^/        /' /var/lib/tboot-sbloader/sbloader-manifest.baseline 2>/dev/null || no "cannot read baseline"
echo

echo "---- 3. source .efi unchanged (vs baseline src_efi) ----"
shachk /usr/lib/systemd/boot/efi/systemd-bootx64.efi 3d7e17b12a37b288b794765055429e7e6bff39af59e516f478657d763685fb4b
echo

echo "---- 4. must-be-absent objects ----"
absent /usr/lib/systemd/boot/efi/systemd-bootx64.efi.signed
absent /var/lib/tboot-sbloader/UNSAFE-TO-REBOOT
absent /var/lib/tboot-sbloader/helper-last-success
absent /var/lib/tboot-sbloader/helper-last-error
n60=$(find /etc/dnf/libdnf5-plugins/actions.d/ -maxdepth 1 -name '60-*' 2>/dev/null | wc -l)
if [ "$n60" -eq 0 ]; then ok "actions.d has no 60-* (unarmed)"; else no "actions.d 60-* PRESENT (already armed): $n60"; fi
echo

echo "---- 5. must-be-present state objects ----"
present /var/lib/tboot-sbloader/sbloader-manifest.baseline
present /var/lib/tboot-sbloader/last-computed-manifest
present /var/lib/tboot-sbloader/last-decision
for p in '50-' '00-'; do
  n=$(find /etc/dnf/libdnf5-plugins/actions.d/ -maxdepth 1 -name "${p}*" 2>/dev/null | wc -l)
  if [ "$n" -ge 1 ]; then ok "actions.d has ${p}* ($n)"; else no "actions.d missing ${p}*"; fi
done
echo

echo "---- 6. perms ----"
permchk /var/lib/tboot-sbloader                            "700 root:root"
permchk /var/lib/tboot-sbloader/sbloader-manifest.baseline "600 root:root"
permchk /var/lib/tboot-sbloader/last-computed-manifest     "600 root:root"
permchk /var/lib/tboot-sbloader/last-decision              "600 root:root"
permchk /usr/local/sbin/tboot-sbloader-helper              "755 root:root"
permchk /usr/local/sbin/tboot-sbloader-posttrans           "755 root:root"
echo

echo "---- 7. actions.d listing (INFO) ----"
ls -la /etc/dnf/libdnf5-plugins/actions.d/ 2>/dev/null | sed 's/^/      /'
echo

echo "---- 8. mask state (only update.service masked) ----"
masked systemd-boot-update.service
ml="$(readlink -f /etc/systemd/system/systemd-boot-update.service 2>/dev/null || true)"
chk "update.service mask target" "$ml" "/dev/null"
notmasked systemd-boot-clear-sysfail.service
notmasked systemd-bootctl@.service
notmasked systemd-bootctl.socket
notmasked systemd-boot-random-seed.service
echo

echo "---- 9. staged 60- rule (INFO; Gate 1 regenerates if absent) ----"
if [ -e /tmp/60-tboot-sbloader.actions.staged ]; then
  shachk /tmp/60-tboot-sbloader.actions.staged 2e83fcfcd237359d02a831cb381d40ca221d319108d66c264b847956c0418274
  sz=$(stat -c '%s' /tmp/60-tboot-sbloader.actions.staged); chk "staged size" "$sz" "115"
else
  info "staged rule absent: Gate 1 will regenerate from canonical one-liner and verify SHA"
fi
echo

echo "---- 10. state file contents (INFO) ----"
echo "      last-decision:";        sed 's/^/        /' /var/lib/tboot-sbloader/last-decision 2>/dev/null
echo "      last-computed-manifest:"; sed 's/^/        /' /var/lib/tboot-sbloader/last-computed-manifest 2>/dev/null
echo

echo "---- 11. PCR11 runtime vs stored expected ----"
pcr11="$(tpm2_pcrread sha256:11 2>/dev/null | awk '/0x/{v=$NF} END{print v}' | sed 's/^0x//' | tr 'a-f' 'A-F')"
chk "PCR11 runtime" "$pcr11" "28E66CE224C9BB0711059470D77103A5AB5A5C1674C03D1BA1563C2FEE52C90F"
echo

echo "---- 12. decider dry-run (read-only keystone; proves ESP equality + shape) ----"
dp="$(/usr/local/sbin/tboot-sbloader-posttrans --debug-print 2>&1 || true)"
printf '      %s\n' "$dp"
if printf '%s' "$dp" | grep -q 'equal=1' && printf '%s' "$dp" | grep -q 'shape=0'; then
  ok "decider drift equal=1 shape=0 (primed, non-converged)"
else
  no "decider drift NOT equal=1/shape=0 (see output above; check --debug-print flag)"
fi
echo

echo "===== SUMMARY ====="
if [ "$fail" -eq 0 ]; then echo "GATE 0 RESULT: ALL CHECKS PASSED"; else echo "GATE 0 RESULT: ${fail} CHECK(S) FAILED"; fi
EOF
```

```bash
# --- GATE 1 (VM 500): publish 60-tboot-sbloader.actions (the ARMING action) ---
# State-changing. set -euo pipefail INSIDE the child only. Atomic mktemp -> install -m 0644 -> mv -T.
bash <<'EOF'
set -euo pipefail

dest_dir="$(readlink -f /etc/dnf/libdnf5-plugins/actions.d)"
final="$dest_dir/60-tboot-sbloader.actions"
staged="/tmp/60-tboot-sbloader.actions.staged"
STAGED_SHA="2e83fcfcd237359d02a831cb381d40ca221d319108d66c264b847956c0418274"
ESP_HASH="73b0c6c4da8386a812749de088a442269a2f6869974d7f03e87643d52a449f41"
SRC_EFI_SHA="3d7e17b12a37b288b794765055429e7e6bff39af59e516f478657d763685fb4b"
PCR_FILE="/root/tboot-lab/state/expected-pcr11-after-hook-uki.txt"
PCR_SYS="/sys/class/tpm/tpm0/pcr-sha256/11"

die(){ printf 'FAIL: %s\n' "$1" >&2; exit 1; }
ok(){ printf 'PASS  %s\n' "$1"; }
shaeq(){ [ -e "$1" ] || die "missing for sha: $1"
  local g; g="$(sha256sum "$1" | awk '{print $1}')"
  [ "$g" = "$2" ] || die "sha mismatch $1 got=$g want=$2"; ok "sha $1"; }
must_absent(){ [ ! -e "$1" ] || die "must be absent: $1"; ok "absent: $1"; }
must_present(){ [ -e "$1" ] || die "must be present: $1"; ok "present: $1"; }
norm(){ tr -d '[:space:]' < "$1" | sed 's/^0[xX]//' | tr 'a-f' 'A-F'; }
decider_field(){ printf '%s\n' "$1" | sed -n "s/^$2=//p"; }

echo "===== GATE 1: publish 60-tboot-sbloader.actions (ARMING) ====="
echo "utc=$(date -u +%FT%TZ)  host=$(hostname)"
echo
echo "---- pre-publish guard ----"

# artifact integrity (B.4)
shaeq /usr/local/sbin/tboot-sbloader-helper    b9df2534b1539355a3c040ee296611f5da629e76467fccfcd44f6ed56325690f
shaeq /usr/local/sbin/tboot-sbloader-posttrans b8d9a04c30cd39ee3600fb3ff8f581d7cf284f7f4aa639f03cf0fd733e705f0c
# artifact integrity (B.2 + UKI hook untouched)
shaeq /usr/local/sbin/tboot-dnf-posttrans      1e75110935a72fc9433c9b4bf5688d26bc6d3bd1e02f4388b5e1f32ff1e5cc05
shaeq /usr/local/sbin/tboot-dnf-helper         937afc7a0d199bf7b7d1a5f5bcccf45c37c0f4f7dd0ad924b2567c4da7125e44
shaeq /etc/kernel/install.d/80-tpm2-sign.install 5857e51d5551e05a7ca384f71529ec2fef07f5a6dbe40df70c6bbe49d720947e

# baseline integrity + non-converged shape
shaeq /var/lib/tboot-sbloader/sbloader-manifest.baseline 2547bd821f978b2a85a919a11d08b32c5062925a9da7fdcf8e1d8a6e7ab4cc5e
grep -qx 'src_signed=ABSENT' /var/lib/tboot-sbloader/sbloader-manifest.baseline \
  || die "baseline missing src_signed=ABSENT"; ok "baseline src_signed=ABSENT"

# must-be-absent
must_absent /usr/lib/systemd/boot/efi/systemd-bootx64.efi.signed
must_absent /var/lib/tboot-sbloader/UNSAFE-TO-REBOOT
must_absent /var/lib/tboot-sbloader/last-error
must_absent /var/lib/tboot-sbloader/helper-last-error
must_absent /var/lib/tboot-sbloader/helper-last-success
must_absent "$final"

# staged source
must_present "$staged"
shaeq "$staged" "$STAGED_SHA"

# updater mask
[ "$(systemctl is-enabled systemd-boot-update.service 2>/dev/null || true)" = masked ] \
  || die "systemd-boot-update.service not masked"; ok "masked: systemd-boot-update.service"
mt="$(readlink -f /etc/systemd/system/systemd-boot-update.service 2>/dev/null || true)"
[ "$mt" = /dev/null ] || die "update.service mask target=$mt want=/dev/null"; ok "update.service -> /dev/null"

# source .efi unchanged
shaeq /usr/lib/systemd/boot/efi/systemd-bootx64.efi "$SRC_EFI_SHA"

# ESP loader hashes (read-only via decider --debug-print)
dp="$(/usr/local/sbin/tboot-sbloader-posttrans --debug-print 2>&1 || true)"
esp_sys="$(decider_field "$dp" esp_systemd)"
esp_bt="$(decider_field "$dp" esp_boot)"
ss_dp="$(decider_field "$dp" src_signed)"
[ "$esp_sys" = "$ESP_HASH" ] || die "esp_systemd=$esp_sys want=$ESP_HASH"; ok "esp_systemd == $ESP_HASH"
[ "$esp_bt"  = "$ESP_HASH" ] || die "esp_boot=$esp_bt want=$ESP_HASH";     ok "esp_boot    == $ESP_HASH"
[ "$ss_dp"   = "ABSENT" ]    || die "computed src_signed=$ss_dp want=ABSENT"; ok "computed src_signed=ABSENT"

# B.2 production rule integrity
shaeq /etc/dnf/libdnf5-plugins/actions.d/50-tboot-posttrans.actions dfbffe56fa95f4e0231569bb3a1aabc8952b9f99c9d290e4f8fb1005ef90d798

# PCR11 sysfs == stored expected
[ -e "$PCR_SYS" ]  || die "missing $PCR_SYS"
[ -e "$PCR_FILE" ] || die "missing $PCR_FILE"
p_now="$(norm "$PCR_SYS")"; p_exp="$(norm "$PCR_FILE")"
[ -n "$p_now" ] || die "empty runtime PCR11"
[ "$p_now" = "$p_exp" ] || die "PCR11 mismatch now=$p_now exp=$p_exp"; ok "PCR11 sysfs==expected ($p_now)"

echo "guard: ALL PASS"
echo
echo "---- atomic publish ----"
tmp="$(mktemp "$dest_dir/.60-tboot-sbloader.actions.XXXXXX")"
trap 'rm -f "$tmp"' EXIT
install -m 0644 -o root -g root "$staged" "$tmp"
mv -T "$tmp" "$final"
trap - EXIT
ok "published: $final"
echo
echo "---- post-publish verification ----"

# published bytes + perms (fatal)
shaeq "$final" "$STAGED_SHA"
pp="$(stat -c '%a %U:%G' "$final")"
[ "$pp" = "644 root:root" ] || die "published perms=$pp want=644 root:root"; ok "published perms ($pp)"

# field count + parse (exactly 5 colon-fields)
rule_line="$(sed -n '/^[[:space:]]*#/d;/^[[:space:]]*$/d;{p;q}' "$final")"
nf="$(awk -F: '{print NF}' <<<"$rule_line")"
[ "$nf" -eq 5 ] || die "field count=$nf want=5"; ok "field count == 5"
IFS=: read -r c1 c2 c3 c4 c5 <<<"$rule_line"
[ "$c1" = "post_transaction" ]                          || die "callback=$c1";        ok "callback        = post_transaction"
[ "$c2" = "systemd-boot-unsigned" ]                     || die "filter=$c2";          ok "package_filter  = systemd-boot-unsigned"
[ "$c3" = "in" ]                                        || die "direction=$c3";       ok "direction       = in"
[ "$c4" = "enabled=host-only raise_error=1" ]           || die "options=$c4";         ok "options         = enabled=host-only raise_error=1"
[ "$c5" = "/usr/local/sbin/tboot-sbloader-posttrans" ]  || die "command=$c5";         ok "command         = /usr/local/sbin/tboot-sbloader-posttrans"

# active .actions set
mapfile -t act < <(find "$dest_dir" -maxdepth 1 -type f -name '*.actions' -printf '%f\n' | sort)
printf '      active .actions: %s\n' "${act[*]}"
case " ${act[*]} " in *" 50-tboot-posttrans.actions "*) ok "active set includes 50-tboot-posttrans.actions";; *) die "50- missing from active set";; esac
case " ${act[*]} " in *" 60-tboot-sbloader.actions "*) ok "active set includes 60-tboot-sbloader.actions";; *) die "60- missing from active set";; esac
[ ! -e "$dest_dir/00-tboot-trigger-probe.actions" ] || die "active probe rule 00-tboot-trigger-probe.actions present"
ok "no active 00- probe rule (exact name)"
if [ -e "$dest_dir/00-tboot-trigger-probe.actions.disabled-after-b21-validation" ]; then
  ok "00- probe present but inert (.disabled-after-b21-validation, not a *.actions file)"
fi

# nothing beyond publication moved
shaeq /usr/local/sbin/tboot-sbloader-helper    b9df2534b1539355a3c040ee296611f5da629e76467fccfcd44f6ed56325690f
shaeq /usr/local/sbin/tboot-sbloader-posttrans b8d9a04c30cd39ee3600fb3ff8f581d7cf284f7f4aa639f03cf0fd733e705f0c
shaeq /var/lib/tboot-sbloader/sbloader-manifest.baseline 2547bd821f978b2a85a919a11d08b32c5062925a9da7fdcf8e1d8a6e7ab4cc5e
shaeq /usr/lib/systemd/boot/efi/systemd-bootx64.efi "$SRC_EFI_SHA"
dp2="$(/usr/local/sbin/tboot-sbloader-posttrans --debug-print 2>&1 || true)"
[ "$(decider_field "$dp2" esp_systemd)" = "$ESP_HASH" ] || die "esp_systemd changed post-publish"; ok "esp_systemd unchanged"
[ "$(decider_field "$dp2" esp_boot)"    = "$ESP_HASH" ] || die "esp_boot changed post-publish";    ok "esp_boot unchanged"
[ "$(norm "$PCR_SYS")" = "$p_exp" ] || die "PCR11 changed post-publish"; ok "PCR11 unchanged"
must_absent /usr/lib/systemd/boot/efi/systemd-bootx64.efi.signed
must_absent /var/lib/tboot-sbloader/UNSAFE-TO-REBOOT
must_absent /var/lib/tboot-sbloader/last-error
must_absent /var/lib/tboot-sbloader/helper-last-error
must_absent /var/lib/tboot-sbloader/helper-last-success

echo
echo "===== GATE 1 DONE: ARMED (50- + 60- active; no transaction run) ====="
EOF
```

```bash
# --- GATE 2 (VM 500): verify ARMED / no mutation beyond publication (read-only) ---
bash <<'EOF'
fail=0
ok(){ printf 'PASS  %s\n' "$1"; }
no(){ printf 'FAIL  %s\n' "$1"; fail=$((fail+1)); }
info(){ printf 'INFO  %s\n' "$1"; }
shachk(){ if [ ! -e "$1" ]; then no "MISSING (sha): $1"; return; fi
  g="$(sha256sum "$1" | awk '{print $1}')"
  [ "$g" = "$2" ] && ok "sha $1" || no "sha $1 got=$g want=$2"; }
absent(){ [ ! -e "$1" ] && ok "absent: $1" || no "PRESENT (must be absent): $1"; }
present(){ [ -e "$1" ] && ok "present: $1" || no "MISSING: $1"; }
permchk(){ if [ ! -e "$1" ]; then no "MISSING (perms): $1"; return; fi
  g="$(stat -c '%a %U:%G' "$1")"; [ "$g" = "$2" ] && ok "perms $1 ($g)" || no "perms $1 got=$g want=$2"; }
norm(){ tr -d '[:space:]' < "$1" | sed 's/^0[xX]//' | tr 'a-f' 'A-F'; }
dfield(){ printf '%s\n' "$1" | sed -n "s/^$2=//p"; }

dest_dir="/etc/dnf/libdnf5-plugins/actions.d"
final="$dest_dir/60-tboot-sbloader.actions"
ESP_HASH="73b0c6c4da8386a812749de088a442269a2f6869974d7f03e87643d52a449f41"
PCR_FILE="/root/tboot-lab/state/expected-pcr11-after-hook-uki.txt"
PCR_SYS="/sys/class/tpm/tpm0/pcr-sha256/11"

echo "===== GATE 2: verify ARMED / no mutation beyond publication (read-only) ====="
echo "utc=$(date -u +%FT%TZ)  host=$(hostname)"
echo
echo "---- 1. published 60- rule integrity ----"
present "$final"
shachk  "$final" 2e83fcfcd237359d02a831cb381d40ca221d319108d66c264b847956c0418274
permchk "$final" "644 root:root"
echo
echo "---- 2. active rule-set composition ----"
mapfile -t act < <(find "$dest_dir" -maxdepth 1 -type f -name '*.actions' -printf '%f\n' | sort)
printf '      active .actions: %s\n' "${act[*]}"
case " ${act[*]} " in *" 50-tboot-posttrans.actions "*) ok "50- active";; *) no "50- missing";; esac
case " ${act[*]} " in *" 60-tboot-sbloader.actions "*) ok "60- active";; *) no "60- missing";; esac
[ ! -e "$dest_dir/00-tboot-trigger-probe.actions" ] && ok "no active 00- probe (exact name)" || no "active 00- probe present"
[ -e "$dest_dir/00-tboot-trigger-probe.actions.disabled-after-b21-validation" ] && ok "00- probe inert (disabled suffix)" || info "00- disabled probe file not found"
[ "${#act[@]}" -eq 2 ] && ok "exactly 2 active .actions files" || no "active .actions count=${#act[@]} want=2"
echo
echo "---- 3. published rule re-parse (5 fields) ----"
rule_line="$(sed -n '/^[[:space:]]*#/d;/^[[:space:]]*$/d;{p;q}' "$final")"
nf="$(awk -F: '{print NF}' <<<"$rule_line")"
[ "$nf" -eq 5 ] && ok "field count == 5" || no "field count=$nf want=5"
IFS=: read -r c1 c2 c3 c4 c5 <<<"$rule_line"
[ "$c1" = "post_transaction" ]                         && ok "callback=post_transaction"               || no "callback=$c1"
[ "$c2" = "systemd-boot-unsigned" ]                    && ok "filter=systemd-boot-unsigned"            || no "filter=$c2"
[ "$c3" = "in" ]                                       && ok "direction=in"                            || no "direction=$c3"
[ "$c4" = "enabled=host-only raise_error=1" ]          && ok "options=enabled=host-only raise_error=1" || no "options=$c4"
[ "$c5" = "/usr/local/sbin/tboot-sbloader-posttrans" ] && ok "command=tboot-sbloader-posttrans"        || no "command=$c5"
echo
echo "---- 4. artifact integrity (B.4 + B.2 + UKI hook) ----"
shachk /usr/local/sbin/tboot-sbloader-helper    b9df2534b1539355a3c040ee296611f5da629e76467fccfcd44f6ed56325690f
shachk /usr/local/sbin/tboot-sbloader-posttrans b8d9a04c30cd39ee3600fb3ff8f581d7cf284f7f4aa639f03cf0fd733e705f0c
shachk /usr/local/sbin/tboot-dnf-posttrans      1e75110935a72fc9433c9b4bf5688d26bc6d3bd1e02f4388b5e1f32ff1e5cc05
shachk /usr/local/sbin/tboot-dnf-helper         937afc7a0d199bf7b7d1a5f5bcccf45c37c0f4f7dd0ad924b2567c4da7125e44
shachk /etc/kernel/install.d/80-tpm2-sign.install 5857e51d5551e05a7ca384f71529ec2fef07f5a6dbe40df70c6bbe49d720947e
shachk /etc/dnf/libdnf5-plugins/actions.d/50-tboot-posttrans.actions dfbffe56fa95f4e0231569bb3a1aabc8952b9f99c9d290e4f8fb1005ef90d798
echo
echo "---- 5. baseline still non-converged ----"
shachk /var/lib/tboot-sbloader/sbloader-manifest.baseline 2547bd821f978b2a85a919a11d08b32c5062925a9da7fdcf8e1d8a6e7ab4cc5e
grep -qx 'src_signed=ABSENT' /var/lib/tboot-sbloader/sbloader-manifest.baseline && ok "baseline src_signed=ABSENT" || no "baseline not ABSENT"
echo
echo "---- 6. source .efi + ESP loader hashes unchanged ----"
shachk /usr/lib/systemd/boot/efi/systemd-bootx64.efi 3d7e17b12a37b288b794765055429e7e6bff39af59e516f478657d763685fb4b
dp="$(/usr/local/sbin/tboot-sbloader-posttrans --debug-print 2>&1 || true)"
[ "$(dfield "$dp" esp_systemd)" = "$ESP_HASH" ] && ok "esp_systemd == ESP_HASH" || no "esp_systemd=$(dfield "$dp" esp_systemd)"
[ "$(dfield "$dp" esp_boot)"    = "$ESP_HASH" ] && ok "esp_boot    == ESP_HASH" || no "esp_boot=$(dfield "$dp" esp_boot)"
[ "$(dfield "$dp" src_signed)"  = "ABSENT" ]    && ok "computed src_signed=ABSENT" || no "computed src_signed=$(dfield "$dp" src_signed)"
echo
echo "---- 7. decider would-be decision still drift (equal=1 shape=0) ----"
if printf '%s' "$dp" | grep -q 'equal=1' && printf '%s' "$dp" | grep -q 'shape=0'; then
  ok "drift equal=1 shape=0 (armed, still non-converged)"
else
  no "decision not equal=1/shape=0"
fi
printf '%s\n' "$dp" | sed -n 's/^/      /; /would-be decision/p'
echo
echo "---- 8. must-be-absent (no first-fire side effects) ----"
absent /usr/lib/systemd/boot/efi/systemd-bootx64.efi.signed
absent /var/lib/tboot-sbloader/UNSAFE-TO-REBOOT
absent /var/lib/tboot-sbloader/last-error
absent /var/lib/tboot-sbloader/helper-last-error
absent /var/lib/tboot-sbloader/helper-last-success
echo
echo "---- 9. updater mask intact ----"
[ "$(systemctl is-enabled systemd-boot-update.service 2>/dev/null || true)" = masked ] && ok "update.service masked" || no "update.service not masked"
[ "$(readlink -f /etc/systemd/system/systemd-boot-update.service 2>/dev/null || true)" = /dev/null ] && ok "update.service -> /dev/null" || no "mask target wrong"
echo
echo "---- 10. PCR11 unchanged ----"
[ -e "$PCR_SYS" ] && [ -e "$PCR_FILE" ] && { [ "$(norm "$PCR_SYS")" = "$(norm "$PCR_FILE")" ] && ok "PCR11 sysfs==expected ($(norm "$PCR_SYS"))" || no "PCR11 mismatch"; } || no "PCR11 source file(s) missing"
echo
echo "---- 11. libdnf5 actions plugin enablement (INFO-tolerant) ----"
pconf=""
for c in /etc/dnf/libdnf5-plugins/actions.conf /usr/etc/dnf/libdnf5-plugins/actions.conf; do
  [ -e "$c" ] && pconf="$c" && break
done
if [ -n "$pconf" ]; then
  info "actions plugin conf: $pconf"
  sed 's/^/        /' "$pconf"
  if grep -Eqi '^[[:space:]]*enabled[[:space:]]*=[[:space:]]*(0|false|no)\b' "$pconf"; then
    no "actions plugin appears DISABLED in $pconf: first-fire would not trigger"
  else
    ok "actions plugin not explicitly disabled"
  fi
else
  info "no explicit actions.conf found; confirm the Fedora 43 plugin default before Gate 3"
fi
echo
echo "===== SUMMARY ====="
if [ "$fail" -eq 0 ]; then
  echo "GATE 2 RESULT: ALL CHECKS PASSED (ARMED, no mutation beyond publication)"
else
  echo "GATE 2 RESULT: ${fail} CHECK(S) FAILED"
  exit 1
fi
EOF
```

Gate 3 is preceded by the pre-first-fire snapshot on host pve-host (`module5-b5-pre-firstfire`, after confirming thin-pool Data% < 75% and `qm status 500` running).

```bash
# --- GATE 3 (VM 500): controlled first-fire: dnf reinstall systemd-boot-unsigned ---
# State-changing. set -uo pipefail (NOT -e): must keep collecting logs/state even if DNF fails.
bash <<'EOF'
set -uo pipefail   # NOT -e: must keep collecting logs/state even if DNF fails

ts="$(date -u +%Y%m%dT%H%M%SZ)"
artdir="/root/tboot-lab/artifacts"
log="$artdir/b5-gate3-firstfire-$ts.log"
jlog_dec="$artdir/b5-gate3-journal-sbloader-posttrans-$ts.log"
jlog_hlp="$artdir/b5-gate3-journal-sbloader-helper-$ts.log"
jlog_b2="$artdir/b5-gate3-journal-dnf-posttrans-$ts.log"
install -d -m 0755 "$artdir"

ESP_HASH="73b0c6c4da8386a812749de088a442269a2f6869974d7f03e87643d52a449f41"
SRC_EFI_SHA="3d7e17b12a37b288b794765055429e7e6bff39af59e516f478657d763685fb4b"
PCR_FILE="/root/tboot-lab/state/expected-pcr11-after-hook-uki.txt"
PCR_SYS="/sys/class/tpm/tpm0/pcr-sha256/11"
norm(){ tr -d '[:space:]' < "$1" | sed 's/^0[xX]//' | tr 'a-f' 'A-F'; }
dfield(){ printf '%s\n' "$1" | sed -n "s/^$2=//p"; }
shaof(){ sha256sum "$1" | awk '{print $1}'; }
report(){ echo "evidence: $1"; echo "  sha256: $(shaof "$1")"; echo "  size:   $(stat -c '%s' "$1") bytes"; }

echo "===== GATE 3: controlled first-fire (dnf reinstall systemd-boot-unsigned) ====="
echo "utc=$ts  host=$(hostname)  log=$log"
echo

# ---- pre-fire guard (fatal): re-pin armed invariants immediately before fire ----
g=0
gd(){ printf 'GUARD-FAIL: %s\n' "$1" >&2; g=$((g+1)); }
shaeq(){ [ -e "$1" ] && [ "$(shaof "$1")" = "$2" ] || gd "$3"; }

# B.4 chain
shaeq /usr/local/sbin/tboot-sbloader-posttrans b8d9a04c30cd39ee3600fb3ff8f581d7cf284f7f4aa639f03cf0fd733e705f0c "decider SHA drift"
shaeq /usr/local/sbin/tboot-sbloader-helper    b9df2534b1539355a3c040ee296611f5da629e76467fccfcd44f6ed56325690f "helper SHA drift"
shaeq /var/lib/tboot-sbloader/sbloader-manifest.baseline 2547bd821f978b2a85a919a11d08b32c5062925a9da7fdcf8e1d8a6e7ab4cc5e "baseline SHA drift"
shaeq /etc/dnf/libdnf5-plugins/actions.d/60-tboot-sbloader.actions 2e83fcfcd237359d02a831cb381d40ca221d319108d66c264b847956c0418274 "60-rule SHA drift"
# B.2 chain + UKI hook (must be untouched)
shaeq /etc/dnf/libdnf5-plugins/actions.d/50-tboot-posttrans.actions dfbffe56fa95f4e0231569bb3a1aabc8952b9f99c9d290e4f8fb1005ef90d798 "50-rule SHA drift"
shaeq /usr/local/sbin/tboot-dnf-posttrans 1e75110935a72fc9433c9b4bf5688d26bc6d3bd1e02f4388b5e1f32ff1e5cc05 "B.2 decider SHA drift"
shaeq /usr/local/sbin/tboot-dnf-helper    937afc7a0d199bf7b7d1a5f5bcccf45c37c0f4f7dd0ad924b2567c4da7125e44 "B.2 helper SHA drift"
shaeq /etc/kernel/install.d/80-tpm2-sign.install 5857e51d5551e05a7ca384f71529ec2fef07f5a6dbe40df70c6bbe49d720947e "80-tpm2-sign hook SHA drift"
# source .efi unchanged
shaeq /usr/lib/systemd/boot/efi/systemd-bootx64.efi "$SRC_EFI_SHA" "source .efi SHA drift"
# armed + clean + non-converged
[ -e /etc/dnf/libdnf5-plugins/actions.d/60-tboot-sbloader.actions ] || gd "60- rule not published (not armed)"
grep -Eq '^[[:space:]]*enabled[[:space:]]*=[[:space:]]*1\b' /etc/dnf/libdnf5-plugins/actions.conf 2>/dev/null || gd "actions.conf not enabled=1"
[ ! -e /usr/lib/systemd/boot/efi/systemd-bootx64.efi.signed ] || gd ".efi.signed already present (not a clean first-fire)"
[ ! -e /var/lib/tboot-sbloader/UNSAFE-TO-REBOOT ] || gd "sentinel already present"
[ ! -e /var/lib/tboot-sbloader/last-error ] || gd "stale last-error present"
[ ! -e /var/lib/tboot-sbloader/helper-last-error ] || gd "stale helper-last-error present"
[ ! -e /var/lib/tboot-sbloader/helper-last-success ] || gd "helper-last-success present before first-fire"
grep -qx 'src_signed=ABSENT' /var/lib/tboot-sbloader/sbloader-manifest.baseline || gd "baseline not non-converged"
# updater mask
[ "$(systemctl is-enabled systemd-boot-update.service 2>/dev/null || true)" = masked ] || gd "systemd-boot-update.service not masked"
[ "$(readlink -f /etc/systemd/system/systemd-boot-update.service 2>/dev/null || true)" = /dev/null ] || gd "update.service mask target != /dev/null"
# ESP hashes + computed src_signed (read-only via decider --debug-print)
dp="$(/usr/local/sbin/tboot-sbloader-posttrans --debug-print 2>&1 || true)"
[ "$(dfield "$dp" esp_systemd)" = "$ESP_HASH" ] || gd "esp_systemd != expected"
[ "$(dfield "$dp" esp_boot)"    = "$ESP_HASH" ] || gd "esp_boot != expected"
[ "$(dfield "$dp" src_signed)"  = "ABSENT" ]    || gd "computed src_signed != ABSENT"
# PCR11
{ [ -e "$PCR_SYS" ] && [ -e "$PCR_FILE" ] && [ "$(norm "$PCR_SYS")" = "$(norm "$PCR_FILE")" ]; } || gd "runtime PCR11 != stored expected"

if [ "$g" -ne 0 ]; then echo "ABORT: $g guard failure(s); transaction NOT run." >&2; exit 1; fi
echo "pre-fire guard: PASS (armed, clean, non-converged, all invariants pinned)"
echo

# ---- the one state-changing action: controlled, logged first-fire ----
PRE_TIME="$(date -Iseconds)"
echo "PRE_TIME=$PRE_TIME"
echo "---- running: dnf -y reinstall systemd-boot-unsigned (tee -> log) ----"
dnf -y reinstall systemd-boot-unsigned 2>&1 | tee "$log"
rc=${PIPESTATUS[0]}
echo
echo "transaction exit code (dnf): rc=$rc"
echo

# ---- bounded journal capture (separate named logs) ----
echo "---- capturing bounded journals since $PRE_TIME ----"
journalctl -t tboot-sbloader-posttrans --since "$PRE_TIME" --no-pager > "$jlog_dec" 2>&1 || true
journalctl -t tboot-sbloader-helper    --since "$PRE_TIME" --no-pager > "$jlog_hlp" 2>&1 || true
journalctl -t tboot-dnf-posttrans      --since "$PRE_TIME" --no-pager > "$jlog_b2"  2>&1 || true
sync
echo

# ---- evidence integrity (every log: sha + size) ----
echo "---- evidence integrity ----"
report "$log"
report "$jlog_dec"
report "$jlog_hlp"
report "$jlog_b2"
echo

# ---- MINIMAL immediate sanity only (full proof deferred to Gate 4) ----
echo "---- minimal sanity (Gate 4 carries the convergence proof) ----"
unsafe=0
[ "$rc" -eq 0 ] && echo "SANITY  dnf rc=0" || { echo "SANITY  dnf rc=$rc (NON-ZERO)"; unsafe=1; }

if [ -e /usr/lib/systemd/boot/efi/systemd-bootx64.efi.signed ]; then
  echo "SANITY  .efi.signed exists"
else
  echo "SANITY  .efi.signed ABSENT (unexpected after first-fire; inspect in Gate 4)"
fi

if [ -r /var/lib/tboot-sbloader/last-decision ]; then
  echo "SANITY  last-decision readable:"
  sed 's/^/          /' /var/lib/tboot-sbloader/last-decision
else
  echo "SANITY  last-decision NOT readable"
fi

if [ -e /var/lib/tboot-sbloader/UNSAFE-TO-REBOOT ]; then
  echo "SANITY  sentinel PRESENT (UNSAFE-TO-REBOOT)"; unsafe=1
else
  echo "SANITY  sentinel absent"
fi

if [ -e /var/lib/tboot-sbloader/last-error ]; then
  echo "SANITY  last-error PRESENT:"; sed 's/^/          /' /var/lib/tboot-sbloader/last-error; unsafe=1
else
  echo "SANITY  last-error absent"
fi

if [ -e /var/lib/tboot-sbloader/helper-last-error ]; then
  echo "SANITY  helper-last-error PRESENT:"; sed 's/^/          /' /var/lib/tboot-sbloader/helper-last-error; unsafe=1
else
  echo "SANITY  helper-last-error absent"
fi

if [ -e /var/lib/tboot-sbloader/helper-last-success ]; then
  echo "SANITY  helper-last-success present:"
  sed 's/^/          /' /var/lib/tboot-sbloader/helper-last-success 2>/dev/null || echo "          (present, not readable as text)"
else
  echo "SANITY  helper-last-success absent (Gate 4 will assess whether expected)"
fi
echo

# ---- fatal verdict (after all evidence collected) ----
echo "===== GATE 3 VERDICT ====="
if [ "$unsafe" -eq 0 ]; then
  echo "GATE 3 RESULT: transaction completed; continue to Gate 4 validation"
  exit 0
else
  echo "GATE 3 RESULT: FAILED/UNSAFE: do not reboot"
  exit 1
fi
EOF
```

```bash
# --- GATE 4 (VM 500): convergence proof: B.4 systemd-boot + B.2/UKI re-convergence (read-only) ---
# A3 uses the dual-form decider-decision matcher (numeric `equal=N shape=N` on drift, worded on converge).
bash <<'EOF'
fail=0
ok(){ printf 'PASS  %s\n' "$1"; }
no(){ printf 'FAIL  %s\n' "$1"; fail=$((fail+1)); }
info(){ printf 'INFO  %s\n' "$1"; }
shaof(){ sha256sum "$1" 2>/dev/null | awk '{print $1}'; }
norm(){ tr -d '[:space:]' < "$1" | sed 's/^0[xX]//' | tr 'a-f' 'A-F'; }
dfield(){ printf '%s\n' "$1" | sed -n "s/^$2=//p"; }
assert_sha(){ if [ ! -e "$1" ]; then no "MISSING: $1"; return; fi
  g="$(shaof "$1")"; [ "$g" = "$2" ] && ok "sha $1" || no "sha $1 got=$g want=$2"; }
absent(){ [ ! -e "$1" ] && ok "absent: $1" || no "PRESENT (must be absent): $1"; }
present(){ [ -e "$1" ] && ok "present: $1" || no "MISSING: $1"; }
nonempty(){ [ -s "$1" ] && ok "present+nonempty: $1" || no "MISSING or EMPTY: $1"; }

DBCRT="/etc/uefi-keys/db.crt"
sbv_db(){ [ -e "$DBCRT" ] || { no "db.crt missing at $DBCRT (cannot verify $1)"; return; }
  if sbverify --cert "$DBCRT" "$1" >/dev/null 2>&1; then ok "sbverify db.crt OK: $1"; else no "sbverify db.crt FAILED: $1"; fi; }
# pesign-based signer count (project-validated idiom; show_trusted_boot_result.sh:23)
signer1(){ n="$(pesign --show-signature --in="$1" 2>/dev/null | grep -c 'common name')"
  [ "$n" = "1" ] && ok "signer_count==1 (pesign): $1" || no "signer_count=$n (want 1, pesign): $1"; }
# section presence via objdump -h (read-only header read)
has_section(){ if objdump -h "$2" 2>/dev/null | awk '{print $2}' | grep -Fxq "$1"; then ok "section $1 present: $2"; else no "section $1 MISSING: $2"; fi; }
# section-VMA sanity: validated 0x180000000 threshold (06F B.1)
vma_sane(){ bad="$(objdump -h "$1" 2>/dev/null | awk '
    /^[[:space:]]*[0-9]+[[:space:]]+\./ { v=strtonum("0x" $4); if (v>=strtonum("0x180000000")) print $2"="$4 }')"
  [ -z "$bad" ] && ok "section-VMA sanity (<0x180000000): $1" || no "ANOMALOUS section VMA(s) in $1: $bad"; }

SIGNED="/usr/lib/systemd/boot/efi/systemd-bootx64.efi.signed"
SIGNED_SHA="4859f48ac4d0776751e21cefc39cd42017ff496153dc8ed3577e2b0fdb114986"
SRC_EFI="/usr/lib/systemd/boot/efi/systemd-bootx64.efi"
SRC_EFI_SHA="3d7e17b12a37b288b794765055429e7e6bff39af59e516f478657d763685fb4b"
ESP_SYS="/boot/efi/EFI/systemd/systemd-bootx64.efi"
ESP_BOOT="/boot/efi/EFI/BOOT/BOOTX64.EFI"
PCR_FILE="/root/tboot-lab/state/expected-pcr11-after-hook-uki.txt"
PCR_SYS="/sys/class/tpm/tpm0/pcr-sha256/11"
PCR_EXP="28E66CE224C9BB0711059470D77103A5AB5A5C1674C03D1BA1563C2FEE52C90F"
B4_DIR="/var/lib/tboot-sbloader"
B2_DEC="/var/lib/tboot-dnf-posttrans"
B2_HLP="/var/lib/tboot-dnf-helper"
MID="$(cat /etc/machine-id 2>/dev/null)"; KVER="$(uname -r)"
EXP_UKI="/boot/efi/EFI/Linux/${MID}-${KVER}.efi"
EXP_TOKEN="${MID}-${KVER}"

echo "===== GATE 4: convergence proof: B.4 systemd-boot + B.2/UKI re-convergence (read-only) ====="
echo "utc=$(date -u +%FT%TZ)  host=$(hostname)  MID=${MID}  KVER=${KVER}"
echo

echo "######## PART A: B.4 systemd-boot convergence (ASSERT) ########"
echo
echo "---- A1. .efi.signed integrity ----"
present "$SIGNED"; assert_sha "$SIGNED" "$SIGNED_SHA"; signer1 "$SIGNED"; sbv_db "$SIGNED"
echo
echo "---- A2. source unsigned .efi unchanged (same-version reinstall) ----"
assert_sha "$SRC_EFI" "$SRC_EFI_SHA"
echo
echo "---- A3. ESP convergence via decider --debug-print (authoritative; dual-form matcher) ----"
dp="$(/usr/local/sbin/tboot-sbloader-posttrans --debug-print 2>&1 || true)"
[ "$(dfield "$dp" src_signed)" = "$SIGNED_SHA" ] && ok "decider src_signed == signed (no longer ABSENT)" || no "decider src_signed=$(dfield "$dp" src_signed)"
[ "$(dfield "$dp" esp_systemd)" = "$SIGNED_SHA" ] && ok "decider esp_systemd == src_signed" || no "decider esp_systemd=$(dfield "$dp" esp_systemd)"
[ "$(dfield "$dp" esp_boot)"    = "$SIGNED_SHA" ] && ok "decider esp_boot == src_signed"   || no "decider esp_boot=$(dfield "$dp" esp_boot)"
[ "$(dfield "$dp" src_efi)"     = "$SRC_EFI_SHA" ] && ok "decider src_efi unchanged"        || no "decider src_efi=$(dfield "$dp" src_efi)"
dl="$(printf '%s\n' "$dp" | sed -n 's/.*would-be decision:[[:space:]]*//p' | sed 's/[[:space:]]*---[[:space:]]*$//')"
echo "      decision-line: ${dl:-<none>}"
case "$dl" in
  unchanged*converged-shape*) ok "converged decision: unchanged (manifest==baseline AND converged-shape)";;
  drift*)                     no "STILL DRIFT: $dl";;
  *)                          no "unexpected decision form: ${dl:-<none>}";;
esac
echo
echo "---- A4. ESP loader files: hash==signed, signer==1, sbverify, path/decider cross-check ----"
for f in "$ESP_SYS" "$ESP_BOOT"; do
  if [ ! -e "$f" ]; then no "ESP loader MISSING: $f"; continue; fi
  [ "$(shaof "$f")" = "$SIGNED_SHA" ] && ok "ESP == .efi.signed: $f" || no "ESP != .efi.signed ($(shaof "$f")): $f"
  signer1 "$f"; sbv_db "$f"
done
[ "$(shaof "$ESP_SYS")"  = "$(dfield "$dp" esp_systemd)" ] && ok "ESP_SYS path agrees with decider"  || no "ESP_SYS/decider hash mismatch"
[ "$(shaof "$ESP_BOOT")" = "$(dfield "$dp" esp_boot)"    ] && ok "ESP_BOOT path agrees with decider" || no "ESP_BOOT/decider hash mismatch"
echo
echo "---- A5. B.4 baseline advanced to converged shape ----"
BL="$B4_DIR/sbloader-manifest.baseline"; nonempty "$BL"
grep -q '^src_signed=ABSENT$' "$BL" && no "B.4 baseline STILL src_signed=ABSENT (did not advance)" || ok "B.4 baseline src_signed != ABSENT"
b_ss="$(sed -n 's/^src_signed=//p' "$BL")"; b_es="$(sed -n 's/^esp_systemd=//p' "$BL")"; b_eb="$(sed -n 's/^esp_boot=//p' "$BL")"
[ "$b_ss" = "$SIGNED_SHA" ] && ok "B.4 baseline src_signed == signed" || no "B.4 baseline src_signed=$b_ss"
[ "$b_es" = "$b_ss" ] && ok "B.4 baseline esp_systemd == src_signed" || no "B.4 baseline esp_systemd=$b_es"
[ "$b_eb" = "$b_ss" ] && ok "B.4 baseline esp_boot == src_signed"   || no "B.4 baseline esp_boot=$b_eb"
echo "      B.4 baseline:"; sed 's/^/        /' "$BL"
echo
echo "---- A6. B.4 decision/markers/sentinel ----"
present "$B4_DIR/last-decision"
grep -q '^decision=helper-success$' "$B4_DIR/last-decision" && ok "B.4 last-decision=helper-success" || no "B.4 last-decision != helper-success"
present "$B4_DIR/helper-last-success"
[ "$(sed -n 's/^signed_sha=//p' "$B4_DIR/helper-last-success")" = "$SIGNED_SHA" ] && ok "helper-last-success signed_sha == signed" || no "helper-last-success signed_sha mismatch"
absent "$B4_DIR/UNSAFE-TO-REBOOT"; absent "$B4_DIR/last-error"; absent "$B4_DIR/helper-last-error"
echo

echo "######## PART B: B.2 / UKI re-convergence (ASSERT) ########"
echo
echo "---- B1. B.2 decider state files ----"
present "$B2_DEC/last-decision"
grep -q '^decision=helper-success$' "$B2_DEC/last-decision" && ok "B.2 last-decision=helper-success" || no "B.2 last-decision != helper-success"
echo "      B.2 last-decision:"; sed 's/^/        /' "$B2_DEC/last-decision" 2>/dev/null
nonempty "$B2_DEC/boot-input-manifest.baseline"
present  "$B2_DEC/last-computed-manifest"
echo
echo "---- B2. B.2 converged invariant: last-computed-manifest SHA == baseline SHA ----"
lcm="$(shaof "$B2_DEC/last-computed-manifest")"; blb="$(shaof "$B2_DEC/boot-input-manifest.baseline")"
[ -n "$lcm" ] && [ "$lcm" = "$blb" ] && ok "B.2 last-computed == baseline ($lcm)" || no "B.2 last-computed=$lcm baseline=$blb (mismatch)"
echo
echo "---- B3. B.2 errors + sentinel + helper failure marker absent ----"
absent "$B2_DEC/last-error"
absent "$B2_HLP/UNSAFE-TO-REBOOT"
absent "$B2_HLP/last-failure"
absent "$B2_HLP/last-failure.log"
echo
echo "---- B4. B.2 + UKI-hook chain SHAs unchanged ----"
assert_sha /usr/local/sbin/tboot-dnf-posttrans 1e75110935a72fc9433c9b4bf5688d26bc6d3bd1e02f4388b5e1f32ff1e5cc05
assert_sha /usr/local/sbin/tboot-dnf-helper    937afc7a0d199bf7b7d1a5f5bcccf45c37c0f4f7dd0ad924b2567c4da7125e44
assert_sha /etc/kernel/install.d/80-tpm2-sign.install 5857e51d5551e05a7ca384f71529ec2fef07f5a6dbe40df70c6bbe49d720947e
assert_sha /etc/dnf/libdnf5-plugins/actions.d/50-tboot-posttrans.actions dfbffe56fa95f4e0231569bb3a1aabc8952b9f99c9d290e4f8fb1005ef90d798
assert_sha /etc/dnf/libdnf5-plugins/actions.d/60-tboot-sbloader.actions 2e83fcfcd237359d02a831cb381d40ca221d319108d66c264b847956c0418274
echo
echo "---- B5. PCR11 expected file + runtime unchanged (pre-reboot) ----"
present "$PCR_FILE"
[ "$(norm "$PCR_FILE")" = "$PCR_EXP" ] && ok "expected-pcr11 file == 28E66CE2…C90F" || no "expected-pcr11 file=$(norm "$PCR_FILE")"
[ -e "$PCR_SYS" ] && { [ "$(norm "$PCR_SYS")" = "$PCR_EXP" ] && ok "runtime PCR11 (sysfs) == 28E66CE2…C90F (unchanged)" || no "runtime PCR11=$(norm "$PCR_SYS")"; } || no "missing $PCR_SYS"
echo
echo "---- B6. bootctl current/default entry -> expected MID-prefixed UKI ----"
bs="$(bootctl status 2>/dev/null)"
cur="$(printf '%s\n' "$bs" | grep -iE 'Current Entry:' || true)"
def="$(printf '%s\n' "$bs" | grep -iE 'Default Entry:' || true)"
[ -n "$cur" ] && printf '%s' "$cur" | grep -Fq "$EXP_TOKEN" && ok "Current Entry -> ${EXP_TOKEN}" || no "Current Entry not ${EXP_TOKEN}: ${cur:-<absent>}"
[ -n "$def" ] && printf '%s' "$def" | grep -Fq "$EXP_TOKEN" && ok "Default Entry -> ${EXP_TOKEN}" || no "Default Entry not ${EXP_TOKEN}: ${def:-<absent>}"
echo
echo "---- B7. current/default UKI structural integrity ----"
present "$EXP_UKI"
sbv_db "$EXP_UKI"
signer1 "$EXP_UKI"
has_section .pcrpkey "$EXP_UKI"
has_section .pcrsig  "$EXP_UKI"
vma_sane "$EXP_UKI"
echo

echo "######## PART C: RE-PIN (advanced during Gate 3; record for ratification) ########"
echo
echo "---- C1. NEW B.2 baseline + last-computed (post-Gate-3) ----"
info "RE-PIN  B.2 baseline  sha=$(shaof "$B2_DEC/boot-input-manifest.baseline")  size=$(stat -c '%s' "$B2_DEC/boot-input-manifest.baseline" 2>/dev/null)B"
info "RE-PIN  B.2 last-computed sha=$(shaof "$B2_DEC/last-computed-manifest")"
echo
echo "---- C2. NEW B.2 helper persistent markers ----"
if [ -d "$B2_HLP" ]; then
  find "$B2_HLP" -maxdepth 1 -type f -printf '%f\n' 2>/dev/null | sort | while read -r f; do
    info "RE-PIN  $B2_HLP/$f  sha=$(shaof "$B2_HLP/$f")  size=$(stat -c '%s' "$B2_HLP/$f")B"; done
else info "B.2 helper state dir absent: $B2_HLP"; fi
echo
echo "---- C3. NEW ESP UKI set (rebuilt by B.2 helper) ----"
if [ -d /boot/efi/EFI/Linux ]; then
  find /boot/efi/EFI/Linux -maxdepth 1 -type f -name '*.efi' -printf '%f\n' 2>/dev/null | sort | while read -r f; do
    info "RE-PIN  /boot/efi/EFI/Linux/$f  sha=$(shaof "/boot/efi/EFI/Linux/$f")  size=$(stat -c '%s' "/boot/efi/EFI/Linux/$f")B"; done
else no "ESP Linux dir missing"; fi
echo
echo "---- C4. NEW signed systemd-boot loaders (record) ----"
info "RE-PIN  .efi.signed = $SIGNED_SHA"
info "RE-PIN  ESP systemd  = $(shaof "$ESP_SYS")"
info "RE-PIN  ESP fallback = $(shaof "$ESP_BOOT")"
echo

echo "===== SUMMARY ====="
if [ "$fail" -eq 0 ]; then
  echo "GATE 4 RESULT: ALL ASSERTIONS PASSED. B.4 converged (signed+propagated) AND B.2/UKI re-converged (loadable, forward-sealable, PCR11 stable). Record the RE-PIN values in the validation ledger."
else
  echo "GATE 4 RESULT: ${fail} CHECK(S) FAILED"
  exit 1
fi
EOF
```

```bash
# --- GATE 5 (host pve-host): pre-reboot snapshot (rollback anchor after Gate 4) ---
qm snapshot 500 module5-b5-pre-reboot --description "B.5 Gate 4 passed; pre-first-reboot anchor"
```

```bash
# --- GATE 6a (VM 500): reboot. Watch the console: no LUKS passphrase prompt expected. ---
systemctl reboot
```

```bash
# --- GATE 6b (VM 500, after reboot): post-reboot validation (read-only) ---
bash <<'EOF'
fail=0
ok(){ printf 'PASS  %s\n' "$1"; }
no(){ printf 'FAIL  %s\n' "$1"; fail=$((fail+1)); }
info(){ printf 'INFO  %s\n' "$1"; }
shaof(){ sha256sum "$1" 2>/dev/null | awk '{print $1}'; }
norm(){ tr -d '[:space:]' < "$1" | sed 's/^0[xX]//' | tr 'a-f' 'A-F'; }
assert_sha(){ if [ ! -e "$1" ]; then no "MISSING: $1"; return; fi
  g="$(shaof "$1")"; [ "$g" = "$2" ] && ok "sha $1" || no "sha $1 got=$g want=$2"; }
absent(){ [ ! -e "$1" ] && ok "absent: $1" || no "PRESENT (must be absent): $1"; }
DBCRT="/etc/uefi-keys/db.crt"
sbv_db(){ sbverify --cert "$DBCRT" "$1" >/dev/null 2>&1 && ok "sbverify db.crt OK: $1" || no "sbverify db.crt FAILED: $1"; }
signer1(){ n="$(pesign --show-signature --in="$1" 2>/dev/null | grep -c 'common name')"; [ "$n" = "1" ] && ok "signer_count==1: $1" || no "signer_count=$n: $1"; }
vma_sane(){ bad="$(objdump -h "$1" 2>/dev/null | awk '/^[[:space:]]*[0-9]+[[:space:]]+\./ {v=strtonum("0x" $4); if (v>=strtonum("0x180000000")) print $2"="$4}')"
  [ -z "$bad" ] && ok "section-VMA sanity: $1" || no "ANOMALOUS VMA $1: $bad"; }

MID="$(cat /etc/machine-id)"; KVER="$(uname -r)"
UKI="/boot/efi/EFI/Linux/${MID}-${KVER}.efi"
UKI_SHA="1187c78ce2435e0a5c27ed8f35e0fd653da3580ac7a2597b03a30bbfd5deb843"
SIGNED_SHA="4859f48ac4d0776751e21cefc39cd42017ff496153dc8ed3577e2b0fdb114986"
ESP_SYS="/boot/efi/EFI/systemd/systemd-bootx64.efi"
ESP_BOOT="/boot/efi/EFI/BOOT/BOOTX64.EFI"
PCR_FILE="/root/tboot-lab/state/expected-pcr11-after-hook-uki.txt"
PCR_EXP="28E66CE224C9BB0711059470D77103A5AB5A5C1674C03D1BA1563C2FEE52C90F"
LUKS_FALLBACK_UUID="11111111-2222-3333-4444-555555555555"
ACT="/etc/dnf/libdnf5-plugins/actions.d"

echo "===== GATE 6b: post-reboot validation (read-only) ====="
echo "utc=$(date -u +%FT%TZ)  host=$(hostname)  KVER=$KVER"
echo "boot time: $(uptime -s 2>/dev/null)  ($(uptime -p 2>/dev/null))"
echo "(boot succeeded: this shell is running the booted system)"
echo

echo "---- 1. runtime PCR11 == ratified prediction ----"
[ "$(norm /sys/class/tpm/tpm0/pcr-sha256/11)" = "$PCR_EXP" ] && ok "runtime PCR11 == 28E66CE2…C90F" || no "runtime PCR11=$(norm /sys/class/tpm/tpm0/pcr-sha256/11)"
[ "$(norm "$PCR_FILE")" = "$PCR_EXP" ] && ok "expected-pcr11 file == 28E66CE2…C90F" || no "expected-pcr11 file drift"
info "runtime PCR7 = $(tr 'a-f' 'A-F' < /sys/class/tpm/tpm0/pcr-sha256/7)"
echo

echo "---- 2. TPM2 LUKS unlock proof (structural; no passphrase) ----"
cline="$(awk '!/^[[:space:]]*#/ && NF>=2 {print; exit}' /etc/crypttab 2>/dev/null)"
echo "      active crypttab line: ${cline:-<none>}"
cdev="$(printf '%s\n' "$cline" | awk '{print $2}')"
info "crypttab field 2 (device): ${cdev:-<none>}"
rdev=""
case "$cdev" in
  UUID=*) rdev="/dev/disk/by-uuid/${cdev#UUID=}" ;;
  /dev/*) rdev="$cdev" ;;
  *)      info "field 2 not UUID=/ /dev/* form; will use fallback UUID" ;;
esac
if [ -z "$rdev" ] || ! cryptsetup isLuks "$rdev" 2>/dev/null; then
  info "crypttab-derived device unusable; falling back to known LUKS UUID $LUKS_FALLBACK_UUID"
  rdev="/dev/disk/by-uuid/$LUKS_FALLBACK_UUID"
fi
real="$(readlink -f "$rdev" 2>/dev/null || echo "$rdev")"
info "resolved LUKS device: $rdev -> $real"
if cryptsetup isLuks "$rdev" 2>/dev/null; then
  ok "resolved device is LUKS: $rdev"
  dump="$(cryptsetup luksDump "$rdev" 2>/dev/null)"
  printf '%s\n' "$dump" | grep -qE '^[[:space:]]*0:[[:space:]]*luks2' && ok "keyslot 0 present (passphrase fallback intact)" || no "keyslot 0 not found"
  printf '%s\n' "$dump" | grep -qiE 'systemd-tpm2' && ok "systemd-tpm2 token present" || no "systemd-tpm2 token missing"
else
  no "could not confirm a LUKS device (crypttab + fallback both failed)"
fi
printf '%s\n' "$cline" | grep -q 'tpm2-device=auto' && ok "crypttab uses tpm2-device=auto" || no "crypttab missing tpm2-device=auto"
if journalctl -b 0 --no-pager 2>/dev/null | grep -qiE 'Please enter passphrase|Passphrase for'; then
  no "journal shows a passphrase prompt this boot"
else
  ok "no passphrase prompt in this boot's journal"
fi
if journalctl -b 0 --no-pager 2>/dev/null | grep -qiE 'tpm2|Unlocked|via TPM2'; then
  info "corroborating: journal references TPM2/unlock this boot"
else
  info "corroborating: no TPM2 journal string matched (version-dependent; not a failure)"
fi
echo

echo "---- 3. Secure Boot + Measured UKI + boot entry ----"
bs="$(bootctl status 2>/dev/null)"
printf '%s\n' "$bs" | grep -iE 'Secure Boot:'  | grep -qiE 'enabled' && ok "Secure Boot enabled" || no "Secure Boot not enabled"
printf '%s\n' "$bs" | grep -iE 'Measured UKI:' | grep -qiE 'yes'     && ok "Measured UKI: yes"  || info "Measured UKI line not 'yes' (see below)"
printf '%s\n' "$bs" | grep -iE 'Current Entry:' | grep -Fq "${MID}-${KVER}" && ok "Current Entry == MID-prefixed booted UKI" || no "Current Entry != ${MID}-${KVER}"
printf '%s\n' "$bs" | grep -iE 'Default Entry:' | grep -Fq "${MID}-${KVER}" && ok "Default Entry == MID-prefixed booted UKI" || no "Default Entry != ${MID}-${KVER}"
printf '%s\n' "$bs" | grep -iE 'Secure Boot:|Measured UKI:|Current Entry:|Default Entry:' | sed 's/^/      /'
echo

echo "---- 4. booted UKI integrity (ratified pin) ----"
[ -e "$UKI" ] && ok "booted UKI present" || no "booted UKI missing: $UKI"
assert_sha "$UKI" "$UKI_SHA"; sbv_db "$UKI"; signer1 "$UKI"; vma_sane "$UKI"
echo

echo "---- 5. systemd-boot loaders survived reboot == 4859f48a ----"
assert_sha "$ESP_SYS"  "$SIGNED_SHA"; sbv_db "$ESP_SYS";  signer1 "$ESP_SYS"
assert_sha "$ESP_BOOT" "$SIGNED_SHA"; sbv_db "$ESP_BOOT"; signer1 "$ESP_BOOT"
echo

echo "---- 6. both chains converged + sentinels clear across reboot ----"
b4="/var/lib/tboot-sbloader/sbloader-manifest.baseline"
[ "$(sed -n 's/^src_signed=//p' "$b4")" = "$SIGNED_SHA" ] && ok "B.4 baseline still converged" || no "B.4 baseline drift"
absent /var/lib/tboot-sbloader/UNSAFE-TO-REBOOT
absent /var/lib/tboot-sbloader/last-error
absent /var/lib/tboot-sbloader/helper-last-error
[ "$(shaof /var/lib/tboot-dnf-posttrans/boot-input-manifest.baseline)" = "47903d22f4dc76a34e1a459d7fba67ee1b3b4150a4058f426b974aa7c44c220d" ] && ok "B.2 baseline == ratified 47903d22…" || no "B.2 baseline drift"
absent /var/lib/tboot-dnf-posttrans/last-error
absent /var/lib/tboot-dnf-helper/UNSAFE-TO-REBOOT
absent /var/lib/tboot-dnf-helper/last-failure
echo

echo "---- 7. action rules present with expected SHA (post-reboot) ----"
assert_sha "$ACT/50-tboot-posttrans.actions" dfbffe56fa95f4e0231569bb3a1aabc8952b9f99c9d290e4f8fb1005ef90d798
assert_sha "$ACT/60-tboot-sbloader.actions"  2e83fcfcd237359d02a831cb381d40ca221d319108d66c264b847956c0418274
echo

echo "---- 8. chain SHAs stable ----"
assert_sha /usr/local/sbin/tboot-sbloader-posttrans b8d9a04c30cd39ee3600fb3ff8f581d7cf284f7f4aa639f03cf0fd733e705f0c
assert_sha /usr/local/sbin/tboot-sbloader-helper    b9df2534b1539355a3c040ee296611f5da629e76467fccfcd44f6ed56325690f
assert_sha /usr/local/sbin/tboot-dnf-posttrans      1e75110935a72fc9433c9b4bf5688d26bc6d3bd1e02f4388b5e1f32ff1e5cc05
assert_sha /usr/local/sbin/tboot-dnf-helper         937afc7a0d199bf7b7d1a5f5bcccf45c37c0f4f7dd0ad924b2567c4da7125e44
assert_sha /etc/kernel/install.d/80-tpm2-sign.install 5857e51d5551e05a7ca384f71529ec2fef07f5a6dbe40df70c6bbe49d720947e
echo

echo "===== SUMMARY ====="
[ "$fail" -eq 0 ] && echo "GATE 6b RESULT: ALL PASSED: reboot-validated; TPM2 LUKS unlock (no passphrase), Secure Boot, both chains converged" || { echo "GATE 6b RESULT: ${fail} CHECK(S) FAILED"; exit 1; }
EOF
```

Closure snapshot on host pve-host (Gate 7):

```bash
qm snapshot 500 module5-b5-validated --description "B.5 validated: B.4 systemd-boot signing armed+converged+reboot-validated; B.2/UKI re-converged; TPM2 LUKS unlock no-passphrase; both chains fail-closed stable"
```

### Expected output

| Gate | Expected result |
|---|---|
| 0 | ALL CHECKS PASSED: armed-state preconditions intact; decider reports `drift (equal=1 shape=0)` against the non-converged baseline. |
| 1 | `guard: ALL PASS`; `published: …/60-tboot-sbloader.actions`; 5-field parse correct; active set = `50-` + `60-`; `GATE 1 DONE: ARMED`. |
| 2 | ALL CHECKS PASSED; `actions.conf` shows `enabled = 1`. |
| 3 | `rc=0`. Two-chain co-fire in the tee'd log: B.2 (`tboot-dnf-posttrans`/`tboot-dnf-helper`) drift → rebuild both kernels → `FINAL_PCR11=28E66CE2…C90F` → baseline updated → sentinel cleared; then B.4 (`tboot-sbloader-posttrans drift: equal=1 shape=0`) → helper `signed … (4859f48a…)`, `verify-signed OK … signer_count=1`, `checkpoint PASS … BOTH targets`, `propagated: bootctl --no-variables install`, `verify-esp OK`, `SUCCESS`; decider `helper-success: baseline advanced, converged, sentinel cleared`. End state: `.efi.signed` exists, `last-decision=helper-success`, sentinel absent, `helper-last-success.signed_sha=4859f48a…`; verdict `transaction completed; continue to Gate 4 validation`. |
| 4 | ALL ASSERTIONS PASSED: decision-line `unchanged (manifest==baseline AND converged-shape)`; B.4 `.efi.signed` + both ESP loaders == `4859f48a…`; B.2 `last-computed-manifest` == baseline (`47903d22…`); PCR 11 stable; booted UKI `1187c78c…` VMA-sane. Part C records the RE-PIN values. |
| 5 | Snapshot `module5-b5-pre-reboot` created. |
| 6a | Clean boot, no LUKS passphrase prompt (operator-attested at the Proxmox console). |
| 6b | ALL PASSED: runtime PCR 11 = `28E66CE2…C90F`; `tpm2-device=auto` + systemd-tpm2 token + keyslot-0 fallback; Secure Boot enabled (user); Measured UKI yes; Current = Default = MID-prefixed UKI (sha `1187c78c…`); loaders `4859f48a…` survived the reboot byte-identical; both chains converged. |
| 7 | Snapshot `module5-b5-validated` created (B.5 closure anchor). |

### Stop condition

- **Gate 3 `rc != 0`, sentinel present, or `helper-last-error`/`last-error` present:** the verdict prints `FAILED/UNSAFE: do not reboot`. Do not proceed to Gate 5/6. Inspect the tee'd transaction log and the bounded per-tag journals under `/root/tboot-lab/artifacts/`, then follow the evidence to the first unsuccessful stage. Roll back to `module5-b5-pre-firstfire` if the ESP or state is left in an unrecoverable shape.
- **Gate 4 A3 decision-line still `drift`:** convergence did not complete: the B.4 helper signed/propagated but the decider did not advance the baseline to converged shape. Halt; do not reboot. (A bare `equal=1 shape=1` numeric form is NOT produced on the converged path; the worded `unchanged (… converged-shape)` form is correct: `06F` O.1.)
- **Gate 6b runtime PCR 11 != `28E66CE2…C90F`, a passphrase prompt appeared, or Secure Boot not enabled:** the boot chain did not validate. The keyslot-0 passphrase remains the fallback (fail-safe, not a brick). Roll back to `module5-b5-pre-reboot` and triage.
- **Any `qm snapshot` fails (Gate 5 / Gate 7):** Proxmox-host issue (thin-pool Data% ≥ 75%, name collision, parent missing). Triage on host pve-host.

### Snapshot point

- Pre-Gate-3: `module5-b5-pre-firstfire` (armed, pre-first-fire anchor; rollback target for the first-fire transaction).
- Gate 5: `module5-b5-pre-reboot` (converged, pre-first-reboot anchor; rollback target for reboot failures).
- Gate 7: `module5-b5-validated` (B.5 closure anchor; **new newest rollback target**). Chain armed + converged + reboot-validated; the `dnf upgrade systemd*` discipline gate is now lifted for the systemd-boot signing path.

---

# Appendix A: Architectural invariants

These values are stable across rebuilds (modulo current-rebuild specifics like machine-id):

| Invariant | Value |
|---|---|
| Validated hook source sha256 | `5857e51d5551e05a7ca384f71529ec2fef07f5a6dbe40df70c6bbe49d720947e` |
| Hook canonical path | `/etc/kernel/install.d/80-tpm2-sign.install` |
| Hook mode | `0755` |
| systemd-measure binary path | `/usr/lib/systemd/systemd-measure` |
| Forward-seal phase target | `enter-initrd` |
| PCR bank | `sha256` |
| ESP UKI path (kernel-install-produced) | `/boot/efi/EFI/Linux/${MID}-${KVER}.efi` |
| ESP UKI path (Phase 1 manual fallback) | `/boot/efi/EFI/Linux/${KVER}.efi` |
| B.2.2 helper canonical path | `/usr/local/sbin/tboot-dnf-helper` |
| B.2.2 helper source sha256 | `937afc7a0d199bf7b7d1a5f5bcccf45c37c0f4f7dd0ad924b2567c4da7125e44` |
| B.2.2 helper version | `1.0.0` |
| B.2.2 helper journal tag | `tboot-dnf-helper` |
| B.2.2 predict sibling canonical path | `/usr/local/sbin/tboot-predict-pcr11` |
| B.2.2 predict sibling source sha256 | `b729ec271dd76cd8b4581f16c86e50fc28b947f2a1de5a7e16c6d5828591e3de` |
| B.2.2 helper state directory | `/var/lib/tboot-dnf-helper` (root:root 0700) |
| B.2.2 helper lock path | `/run/tboot-dnf-helper.lock` |
| B.2.3 decider canonical path | `/usr/local/sbin/tboot-dnf-posttrans` (installed) |
| B.2.3 decider resolved real path (Fedora 43 UsrMerge) | `/usr/local/bin/tboot-dnf-posttrans` (`/usr/local/sbin` is a relative symlink to `bin`; see `06F` I.3) |
| B.2.3 decider sha256 (installed; identical to Gate 2 staged) | `1e75110935a72fc9433c9b4bf5688d26bc6d3bd1e02f4388b5e1f32ff1e5cc05` |
| B.2.3 decider mode + owner + size | mode 0755, owner root:root, size 35164 bytes |
| B.2.3 decider staged path (Gate 2 reference) | `/tmp/tboot-dnf-posttrans.staged` (volatile; sha `1e75110935a72fc9433c9b4bf5688d26bc6d3bd1e02f4388b5e1f32ff1e5cc05`, mode 0644) |
| B.2.3 decider version | `1.0.0` |
| B.2.3 decider lock path | `/run/tboot-dnf-posttrans.lock` (acquired only by `_mode_normal`; absent in `--prime`, `--debug-print`, `--self-test`) |
| B.2.3 decider state directory | `/var/lib/tboot-dnf-posttrans` (mode 0700 root:root; 3 entries at Gate 3 close) |
| B.2.3 decider baseline file | `/var/lib/tboot-dnf-posttrans/boot-input-manifest.baseline` (mode 0600 root:root; 157 records across 14 manifest categories) |
| B.2.3 decider last-computed-manifest file | `/var/lib/tboot-dnf-posttrans/last-computed-manifest` (mode 0600 root:root; same sha as baseline at Gate 3 close) |
| B.2.3 decider last-decision file | `/var/lib/tboot-dnf-posttrans/last-decision` (mode 0600; structured key=value: `epoch`, `date`, `decision`, `detail`, `compute_duration_s`, `decider_version`) |
| B.2.3 decider last-error file | `/var/lib/tboot-dnf-posttrans/last-error` (mode 0600; absent on success path) |
| B.2.3 decider scratch parent | `/run` (must be tmpfs; per-invocation scratch dir `/run/tboot-dnf-posttrans.XXXXXX`) |
| B.2.3 boot-input manifest categories (14) | `dracut-config`, `dracut-modules`, `kernel-modules`, `firmware`, `udev-rules`, `modprobe-load`, `systemd-early-boot`, `cryptsetup-tooling`, `tpm2-tss`, `storage-tooling`, `fstab-crypttab`, `kernel-install`, `public-trust-config`, `fedora-kernel-config` |
| B.2.3 reboot-safety sentinel | `/var/lib/tboot-dnf-helper/UNSAFE-TO-REBOOT` (mode 0600; absent on safe-to-reboot path) |
| B.2.3 trust-boundary self-scan regex (Deviation D) | see fenced code block immediately below this table |
| B.2.3 forbidden tokens (decider trust boundary) | `ukify sbsign sbverify systemd-measure systemd-cryptenroll efi-updatevar cryptsetup` |
| B.2.3 Gate 1 design-decisions notes | `/root/tboot-lab/notes/b2-3-design-decisions.md` (sha `4301ea0f9556739886d0ab249fd4d6c89d44553ee1bb523508ada1579be270c8`) |
| B.2.3 production `.actions` rule canonical path | `/etc/dnf/libdnf5-plugins/actions.d/50-tboot-posttrans.actions` |
| B.2.3 production `.actions` rule sha256 | `dfbffe56fa95f4e0231569bb3a1aabc8952b9f99c9d290e4f8fb1005ef90d798` |
| B.2.3 production `.actions` rule size | 87 bytes (one line + trailing newline) |
| B.2.3 production `.actions` rule mode + owner | 0644 root:root |
| B.2.3 production `.actions` rule shape (Gate 1 design lock) | `post_transaction:::enabled=host-only raise_error=1:/usr/local/sbin/tboot-dnf-posttrans` |

Current project state is **not** duplicated here. Milestone closure snapshots, current preferred rollback targets, current UKI shas, the current PCR 11 value, current DNF discipline status, and the current ESP forensic-UKI allowlist live in `00_Current_Project_State.md` (the ESP forensic allowlist is also in `06F` L.1). Other rebuild-specific values (PCR 7, PCR 11, pkfp, pol, machine-id, ESP UKI sha) are in `00_Current_Project_State.md` as well. This appendix holds only the static runbook constants above.

Deviation D (trust-boundary self-scan) regex, rendered unambiguously:

```
(^|[^A-Za-z0-9_-])TOKEN($|[^A-Za-z0-9_-])
```

Replace `TOKEN` with each entry from the forbidden-tokens list. The hyphen is inside the identifier-class set (not the boundary set); hyphenated identifiers such as `08-cryptsetup-tooling` do NOT match `cryptsetup` as a separate token. See `06F` H.4 for the rationale and the GNU-awk `\<TOKEN\>` failure mode it corrects.


# Appendix B: Recovery procedures

Per-step halt/triage table. Each row says what to do or where to roll back.

| Scenario | Recovery |
|---|---|
| Step 16 hook fails mid-DNF | DNF aborts the transaction; ESP UKI unchanged. Triage hook journal. |
| Step 17 reboot fails to load hook UKI | systemd-boot one-shot is consumed; next boot loads `bootctl get-default` (Phase 1 UKI). Triage with VM still bootable. |
| PCR 11 mismatch after reboot | Environmental drift between Step 16 build and Step 17 boot (rare). Re-run Steps 16–17. |
| Hook itself broken or canonical path corrupted | Rollback to `module5-default-entry-cleaned` from Proxmox host: `qm rollback 500 module5-default-entry-cleaned`. If that snapshot is itself suspect, fall back to `module5-kernel-signing-hook-validated`. |
| Step 28 negative-invariant FAIL (exits 21–26) | The log-only probe rule mutated something it must not have. Severe. Rollback to `module5-b2-pre-actions-install`. Investigate before any further B.2 work. |
| Step 28 action-file authoring error (sha mismatch, missing escapes) | `rm -f /etc/dnf/libdnf5-plugins/actions.d/00-tboot-trigger-probe.actions` and re-run the `install -m 0644` heredoc. If the system has otherwise drifted, rollback to `module5-b2-pre-actions-install`. |
| Step 29 sha mismatch on staged scripts | Source corruption during copy. Re-copy from canonical project location and re-verify shas. If persistent, rollback to `module5-b2-1-cleaned`. |
| Step 29 `--self-test` reports `error:` line | Read the line. Most common: state directory `/var/lib/tboot-dnf-helper` missing or wrong ACL; re-run the `install -d -m 0700 -o root -g root` command. Rollback target: `module5-b2-1-cleaned`. |
| Step 30 helper non-zero exit | Read `/var/lib/tboot-dnf-helper/last-failure` and `last-failure.log`. **Do not reboot.** Rollback to `module5-b2-2-pre-real-run`. |
| Step 30 post-helper invariant FAIL (exits 61-77) | Severe: helper completed but produced inconsistent state. **Do not reboot.** Rollback to `module5-b2-2-pre-real-run`. |
| Step 30 LUKS passphrase prompt at reboot | TPM unlock failed. Type keyslot 0 passphrase to recover. **Do not take** `module5-b2-2-real-run-validated`. Triage: PCR 11 mismatch most likely. Rollback target: `module5-b2-2-pre-real-run`. |
| Step 30 post-reboot PCR 11 mismatch (exit 84) | Measured-content chain drifted between predict and runtime. Rollback to `module5-b2-2-pre-real-run` and investigate dracut-output non-determinism in the regeneration. |
| Step 31 Step 6 fails on `cryptsetup` at `08-cryptsetup-tooling` (or similar hyphenated category label) | External audit harness uses outdated `\<TOKEN\>` GNU awk word-boundary form. Replace the harness regex with the Deviation D form `(^|[^A-Za-z0-9_-])TOKEN($|[^A-Za-z0-9_-])` (matches the in-script `_self_test_trust_boundary`) and re-run Step 6 only. **Do not patch the staged source.** See `06F` H.4. |
| Step 31 Step 7 invariant FAIL (exits 21–28) | Severe: Gate 2 was supposed to be non-mutating but a mutation surface has moved (decider installed, decider state-dir created, sentinel written, `.actions` file appeared, helper/hook drift, plugin-level `enabled` changed). Stop. Write the failure into `/root/tboot-lab/notes/b2-3-gate-2-attempts.md`. Consider whether rollback to `module5-b2-2-real-run-validated` is required before any further work. |
| Step 32 sub-gate 3.0 FAIL | VM not in B.2.2 closure state. Halt; do not proceed. Investigate which invariant moved (kernel, UKI sha, bootctl entries, PCR 11, frozen artefact shas, negative pre-conditions). Most common: `qm listsnapshot` parsing: apply the tree-prefix-tolerant awk (`06F` I.1). Second most common: `bootctl get-default` returns empty on this Fedora 43 VM: parse `bootctl status` instead (`06F` I.2). |
| Step 32 sub-gate 3.1 FAIL (staged file missing or sha drift) | `/tmp` was wiped or the staged decider was overwritten. Re-run Step 31 (Gate 2) to re-stage. Do not bypass. |
| Step 32 sub-gate 3.2 FAIL on UsrMerge resolution | `/usr/local/sbin` does not resolve to `/usr/local/bin` (Fedora 43 default). Lab layout has drifted from Fedora 43 defaults. Halt; investigate filesystem state before forcing the install. See `06F` I.3. |
| Step 32 sub-gate 3.2 FAIL post-publish (sha drift on DST_REAL) | The atomic-publish window was disturbed. The trap will have cleaned the temp. Rollback to `module5-b2-2-real-run-validated`, then re-run sub-gates 3.0+. |
| Step 32 sub-gate 3.3 FAIL (inode mismatch or --version mismatch) | The install completed but the installed artefact does not match the expected version. Halt; investigate. Most likely cause: a stale older decider remained installed at the resolved real path and the install did not overwrite it. Remove it manually and re-run sub-gate 3.2. |
| Step 32 sub-gate 3.4 FAIL (state-dir not created with correct mode) | `install -d -m 700` failed or `/var/lib` is on the wrong filesystem. Halt; do not run `--prime`. Rollback to `module5-b2-2-real-run-validated`. |
| Step 32 sub-gate 3.5 FAIL on missing `ok: helper present and executable` | Harness bug, not a decider bug. `--self-test` mode does NOT emit this line. Drop the assertion from the harness; verify helper presence via the frozen-sha invariant. See `06F` I.5. |
| Step 32 sub-gate 3.5 FAIL on `-- No entries --` in journal capture | `journalctl` without `-q` emits the advisory line, breaking emptiness tests. Switch to `journalctl -q`. See `06F` I.4. |
| Step 32 sub-gate 3.6 FAIL (--prime rc!=0, baseline missing, err: in journal, sentinel appeared, state-dir count != 3) | The baseline prime did not complete cleanly. Halt. If state was partially written, rollback to `module5-b2-2-real-run-validated` and restart from sub-gate 3.0. **Do not** include a re-prime refusal probe in this gate (`06F` I.6). |
| Step 32 sub-gate 3.7 FAIL on drift (--debug-print sha != baseline sha) | The decider is non-deterministic on steady state, OR a background process mutated a manifest input between Gate 3.6 and Gate 3.7. Severe; rollback to `module5-b2-2-real-run-validated` and investigate the manifest category that drifted (the `diff` head in the FAIL output identifies the category). |
| Step 32 sub-gate 3.7 FAIL on rc capture returning 0 from a failed command | Harness used `output=$(CMD 2>&1 || true)` pattern. The `|| true` masked the failure. Switch to the subshell + tempfile pattern: `( CMD >out 2>err ; echo $? >rc_file )`. See `06F` I.7. |
| Step 32 sub-gate 3.8 FAIL on `qm snapshot` | Proxmox-host issue (thin-pool full, name collision, parent missing). Triage on the host. The decider state on the VM is intact; the only missing artefact is the closure snapshot itself, which can be retried after triage. Do not declare Gate 3 closed until the snapshot exists. |
| Step 32 sub-gate 3.8 FAIL on VM status != running post-snapshot | The snapshot operation stopped the VM unexpectedly. Capture `qm status 500`, `journalctl -u pve*`. Restart the VM (`qm start 500`), verify boot-chain health (PCR 11 match, UKI boot OK), then re-evaluate snapshot trustworthiness. |
| Step 33 Gate 4.1 staged-rule sha mismatch | Heredoc smart-quote substitution or trailing whitespace. Re-run Gate 4.1 from a clean shell; verify shell is not autocompleting/expanding the heredoc body. Expected sha is `dfbffe56fa95f4e0231569bb3a1aabc8952b9f99c9d290e4f8fb1005ef90d798`; expected size 87 bytes; expected mode/owner 0644 root:root. |
| Step 33 Gate 4.2 deviation-D regex flags a forbidden token in the staged rule | Severe: the staged rule should be a single line containing only the design-locked shape. If a token from the forbidden list appears, the heredoc was corrupted. Restart from Gate 4.1 and re-stage. |
| Step 33 Gate 4.3 same-filesystem assertion FAIL (actions.d not on same fs as plugins/) | Lab filesystem layout has drifted from Fedora 43 defaults. Halt; investigate. The atomic-publish `mv -T` requires `actions.d/` and the parent `plugins/` to be on the same filesystem. |
| Step 33 Gate 5 atomic publish trap fires | The TMP file under `actions.d/` was not consumed by the `mv -T` (the script aborted between `install -m 0644` and the rename). The trap removes it cleanly. Investigate which assertion failed; re-run Gate 5 from a clean state. The trap's path-prefix guard means it never removes anything outside `${DST_DIR}/.tboot-50-tboot-posttrans.actions.*`. |
| Step 33 Gate 6A FAIL on dnf rc != 0 | The DNF transaction itself failed (network, dependency, repo metadata). Capture the transaction output; do not advance to Phase 3 invariant assertions. The decider may have fired during a partial transaction; check `last-decision` and `journalctl -t tboot-dnf-posttrans` before any rollback decision. |
| Step 33 Gate 6A FAIL on decider err: lines in journal window | Severe: the decider hit an error path during a no-drift transaction. The transaction did commit (figlet installed), but the decider was unhappy. Capture the journal under `tboot-dnf-posttrans` since `PRE_TIME_6A`, `cat /var/lib/tboot-dnf-posttrans/last-error` (if present), then rollback to `module5-b2-3-decider-installed-primed`. |
| Step 33 Gate 6A FAIL on `Loaded libdnf plugin` not in `dnf5.log` | Harness bug, not a trigger failure. dnf5.log does not record plugin-load entries in this environment (`06F` J.1). The canonical Step 33 Gate 6A script does NOT assert this; if you encounter a derived verifier that does, remove the assertion and use `journalctl -t tboot-dnf-posttrans` evidence instead. |
| Step 33 Gate 6A FAIL on lock-file presence after clean decider exit | Harness bug, not a runtime bug. `flock(1)` releases on fd close without unlinking the lock file (`06F` J.3). The canonical invariant is "not actively held", probed via non-blocking `flock <> -n` with symlink rejection. Replace any `[ ! -e "$DECIDER_LOCK" ]` invariant with the canonical probe in `06B` Step 33 Gate 6A. |
| Step 33 Gate 6A FAIL on baseline sha drift on a no-drift transaction | Severe: the decider updated the baseline despite `decision=unchanged`. This contradicts the design (baseline updates only on helper success, never on no-drift). Capture state-dir contents, journal, `last-decision`, then rollback to `module5-b2-3-decider-installed-primed`. |
| Step 33 Gate 6A FAIL on initramfs / UKI mtime advancement on a no-drift transaction | Severe: a kernel-install scriptlet fired during figlet install. This should be impossible: figlet is not a kernel-package and triggers no kernel-install hooks. Capture `journalctl -t kernel-install --since "$PRE_TIME_6A"` and the dnf transaction output, then rollback to `module5-b2-3-decider-installed-primed`. |
| Step 33 Gate 6B FAIL with last-decision epoch not advanced past Gate 6A's epoch | Severe: the decider did not fire on the outbound (`dnf remove`) transaction, meaning empty `direction:` did not actually fire on outbound in this rule. This contradicts the libdnf5-plugin-actions 1.4.0 man page; investigate before declaring B.2.3 complete. |
| Step 33 Gate 7 Phase 2 `qm snapshot` fails | Proxmox-host issue (thin-pool full, name collision with `module5-b2-3-validated`, parent missing). Triage on the host. VM state is intact; Gates 4–6B closure is preserved on disk. After triage, retry Phase 2 only; do not re-run earlier phases. |
| Step 33 Gate 7 Phase 3 FAIL on PCR 11 drift post-snapshot | The Proxmox snapshot operation disturbed the VM's measured state. Capture `qm status 500`, `journalctl -u pve* --since '-5 min'` on the host; `uname -r`, `cat /sys/class/tpm/tpm0/pcr-sha256/11` on the VM. The snapshot may need to be treated as forensic-evidence-only and re-taken after a reboot stabilises the boot-chain measurement chain. |
| Step 34 Gate 1 FAIL on Phase 1 invariant (decider/helper/predict/hook/prodrule sha drift, baseline sha drift, booted UKI sha drift, runtime or stored PCR 11 drift) | VM not at B.2.3 closure state. Rollback to `module5-b2-3-validated` and re-run Step 34 from Gate 1. If the drift is in the trust-chain script shas, investigate before rollback: a script may have been edited out-of-band. |
| Step 34 Gate 1 FAIL on `dracut-network already installed` (assertion 201) | The reference candidate is unavailable for this rebuild. Set `CANDIDATE` to one of `dracut-live`, `dracut-squash`, `dracut-tools-extra` and re-run Gate 1 from the top. All four candidates pass the Gate 1 selection criteria. |
| Step 34 Gate 1 FAIL on `--assumeno` rc not 0 or 1 (assertion 300), missing refusal phrase (301), or decider fired in window (306) | DNF5 `--assumeno` did not produce a documented refusal shape. Capture `$ASSUMENO_OUT` and `journalctl -t tboot-dnf-posttrans --since "$PRE_ASSUMENO_TS" --until "$POST_ASSUMENO_TS"`. Most likely cause for rc!=0\|1: stale DNF metadata cache; refresh with `dnf makecache` and retry. For 306: decider unexpectedly fired during preview: investigate which DNF transaction actually committed. See `06F` K.2. |
| Step 34 Gate 1 FAIL on transaction-preview sensitive package or unexpected dracut-* (assertions 304, 305) | Candidate is not safe for this rebuild. Do not proceed to Gate 3. Pick a different candidate, re-run Gate 1. |
| Step 34 Gate 2 FAIL on `qm snapshot` | Proxmox-host issue (thin-pool full, name collision with `module5-b2-4-pre-drift-transaction`, parent missing). Triage on the host. VM state on VM 500 is intact. Do not proceed to Gate 3: without the snapshot anchor, Gate 3 is uncovered. |
| Step 34 Gate 2 FAIL on VM status != running post-snapshot | The snapshot operation stopped the VM unexpectedly. Capture `qm status 500`, `journalctl -u pve*` on the host. Restart VM (`qm start 500`), verify boot-chain health (PCR 11 match, UKI boot OK), then re-evaluate snapshot trustworthiness. Do not proceed to Gate 3 until VM is `status: running` and the snapshot is trustworthy. |
| Step 34 Gate 3 FAIL on `dnf install` rc != 0 (assertion 200) | The DNF transaction itself failed. The production rule has `raise_error=1`, so a decider non-zero exit also surfaces here. Capture `$DNF_OUT`, `journalctl -t tboot-dnf-posttrans --since "$PRE_TS"`, `cat /var/lib/tboot-dnf-posttrans/last-error` (if present), `cat /var/lib/tboot-dnf-helper/last-failure` and `last-failure.log` (if present). If the helper sentinel is present (`/var/lib/tboot-dnf-helper/UNSAFE-TO-REBOOT`), the helper failed and the boot chain is in a known-unsafe state: rollback to `module5-b2-4-pre-drift-transaction`. |
| Step 34 Gate 3 FAIL 411 "helper production-mode start lines = 0" with the rest of the journal looking healthy | The rev-3 reference run hit this exact symptom. The mutation likely succeeded; the harness regex is wrong. **Do not roll back immediately.** Read the helper journal manually: `journalctl -t tboot-dnf-helper --since "$PRE_TS" --no-pager -o cat`. If the journal shows `info: starting version=… mode=production` followed by the full kernel-enumeration + dracut + kernel-install + 80-tpm2-sign flow ending in `helper exited 0`, the mutation succeeded and Gate 3 is recoverable via the post-hoc continuation verifier in `06F` K.1. If the helper journal does NOT show this pattern, rollback to `module5-b2-4-pre-drift-transaction` and investigate. |
| Step 34 Gate 3 FAIL on decider journal evidence (401–407) | The decider did not behave as expected. Capture the full decider journal since `$PRE_TS`; capture `$LAST_DECISION`; if `$LAST_ERROR` exists, capture it. Most likely causes: decider crashed mid-run (rollback to `module5-b2-4-pre-drift-transaction`); decider logged unchanged on a drift transaction (severe: baseline-comparison logic is broken; rollback and investigate). |
| Step 34 Gate 3 FAIL on helper journal evidence other than 411 (412–415) | Helper started but did not complete cleanly. If 412 (kernels enumerated != 2), check `/lib/modules/*/vmlinuz` enumeration; rollback if drift. If 415 (helper has errors), capture `/var/lib/tboot-dnf-helper/last-failure*` and rollback. |
| Step 34 Gate 3 FAIL on 80-tpm2-sign hook count (421, 422) | Hook fired wrong number of times. If count = 0, hook is not installed at `/etc/kernel/install.d/80-tpm2-sign.install` or is non-executable: `06B` Step 14 invariants drifted; rollback. If count > 1, kernel-install was invoked more than once per kernel: investigate helper. |
| Step 34 Gate 3 FAIL on state transitions (501–512) | The mutation completed but the decider did not advance state cleanly. 503 (last-decision != helper-success) with `last-decision=helper-pending` suggests the decider was killed between sentinel-write and helper-exit. 508 (sentinel remains): helper did not complete or decider did not clear sentinel: **do not reboot, system is in known-unsafe state**, rollback to `module5-b2-4-pre-drift-transaction`. 509 (last-error present): read the file; rollback. 511 (success marker predates PRE_EPOCH): the helper success marker is stale: the helper did not actually rewrite it on this transaction; severe, rollback. |
| Step 34 Gate 3 FAIL on UKI evidence (601–606) | Either UKI did not advance (601: helper claims success but UKI did not rebuild; severe), or sbverify failed (603, 606: Secure Boot signing chain broken; severe). Rollback to `module5-b2-4-pre-drift-transaction`. |
| Step 34 Gate 3 FAIL on runtime PCR 11 changed without reboot (608) | The TPM2 was extended out-of-band during the transaction. Should not happen. Capture `journalctl --since "$PRE_TS" --until "$POST_TS"` system-wide; the offending PCR-extending operation is in there. Rollback. |
| Step 34 Gate 3 FAIL on trust-chain script sha drift (701–705) | A trust-chain script was mutated during the transaction. Severe: the decider or helper rewrote its own source, or a parallel process did. Capture all five script shas, capture the production rule sha, rollback to `module5-b2-4-pre-drift-transaction`, and investigate. |
| Step 34 Gate 3 PCR 11 value did not advance (informational, not a FAIL) | `PCR 11 value PRE/POST` shows identical values. This is outcome (b) per `06F` K.3: helper-path-only closure. Not a failure: chain machinery is proven, but the value-tracking axis was not exercised by this candidate. Gates 4–6 will still pass; Gate 7 closure-criterion documents the closure path as (b) rather than (a). Consider re-running Gate 3 with a different candidate in a future cycle to exercise the strong-closure path. |
| Step 35 Gate 4 FAIL 200–211 (current UKI integrity broken) | UKI on ESP corrupted between Step 34 closure and Gate 4. Severe. Capture sha + sbverify + objdump output, rollback to `module5-b2-4-pre-drift-transaction`, re-run B.2.4 from Gate 3. |
| Step 35 Gate 4 FAIL 400–404 (PCR 11 cross-validation broken) | Stored prediction ≠ independent prediction, or runtime PCR 11 advanced without reboot, or predictor errored. Capture `tboot-predict-pcr11` stdout/stderr, predictor sha, hook sha, current UKI sha. **Do not snapshot, do not reboot.** Rollback to `module5-b2-4-pre-drift-transaction`. |
| Step 35 Gate 4 FAIL 110–142 (trust-chain or state drift since Step 34) | Something mutated decider/helper/hook/predict/production-rule or decider/helper state between Step 34 closure and Gate 4. Identify the mutating process via `journalctl --since` for the affected `mtime`; rollback to `module5-b2-4-pre-drift-transaction` if drift is unexplained. |
| Step 35 Gate 5 snapshot fails | `qm snapshot` returned non-zero, or `qm listsnapshot 500` does not show `module5-b2-4-pre-reboot` with correct parent + `current` below. Triage on host pve-host (Proxmox storage Data% must be under 75% per `06_Lab_Setup_Runbook.md`). Do not declare Gate 5 closed; do not reboot. |
| Step 36 Gate 6A pre-reboot probe FAIL | At least one invariant drifted between Gate 5 snapshot and the imminent reboot. Halt; do not issue `systemctl reboot`. Triage the specific assertion. If unrecoverable, rollback to `module5-b2-4-pre-reboot`. |
| Step 36 Gate 6A LUKS passphrase prompt at boot | **Hard fail.** Do NOT type the passphrase as part of validation: that proves recovery works, not TPM auto-unlock. Two recovery paths: (1) recovery: type keyslot 0 passphrase to reach userspace, then investigate (read PCR 11 sysfs, journal `systemd-cryptsetup`, check `bootctl status`, verify UKI sha + sbverify); (2) rollback: power-cycle VM via Proxmox UI, then `qm rollback 500 module5-b2-4-pre-reboot` on host. Add a `06F` finding documenting the failure mode before re-attempting. |
| Step 36 Gate 6A firmware Secure Boot rejection / kernel panic | Capture Proxmox console screenshot. Power-cycle VM. Rollback to `module5-b2-4-pre-reboot`. Investigate the rebuilt UKI offline (sbverify, objdump section table, `.pcrpkey`/`.pcrsig` presence, VMA sanity per `06F` B.1). |
| Step 36 Gate 6B FAIL 200–202 (runtime PCR 11 byte-match broken) | **Most severe failure mode**: the property the entire chain was built to prove. Boot succeeded, LUKS unlocked, but runtime PCR 11 != stored prediction. Capture full TPM PCR state via `tpm2_pcrread sha256:0,1,2,3,4,5,6,7,8,9,10,11,12,13,14,15` (read-only) before any rollback. Capture full firmware event log: `cat /sys/kernel/security/tpm0/binary_bios_measurements > /root/binary_bios_measurements_b2_4_gate6b_fail.bin` (read-only). Investigate the divergence: hook output, predictor algorithm, firmware-measured content. Rollback to `module5-b2-4-pre-reboot` after forensics capture. |
| Step 36 Gate 6B FAIL 301–303 / 400 / 410–413 (boot-path or UKI-identity drift across reboot) | Severe. The firmware loaded something other than the validated UKI, or the UKI on the ESP mutated across the reboot transition. Rollback to `module5-b2-4-pre-reboot`. |
| Step 36 Gate 6B FAIL 500–602 (decider/helper state changed across reboot) | State mutation across reboot transition that should not have happened. Identify which file changed; correlate with journal across the reboot window; rollback to `module5-b2-4-pre-reboot`. |
| Step 36 Gate 6B FAIL 900–902 (post-reboot errors in decider/helper/hook journals) | The decider, helper, or 80-tpm2-sign hook is firing post-reboot when it should be quiet. Capture journal contents; do not snapshot Gate 7; investigate before further work. |
| Step 37 Gate 7 snapshot fails | `qm snapshot 500 module5-b2-4-validated` returned non-zero, or `qm listsnapshot 500` does not show the new snapshot parented to `module5-b2-4-pre-reboot`. Triage on host pve-host. **Do not apply the staged DNF discipline lift** until the closure snapshot is verified. |
| Step 37 Gate 7 audit `exit 2` (booted UKI sha drift) | The MID-prefixed booted UKI changed between Gate 6B and Gate 7. Severe. Rollback to `module5-b2-4-pre-reboot`. |
| Step 37 Gate 7 audit reveals NEW unexpected `.efi` files (outside the documented allowlist) | A new firmware-loadable artifact appeared on the ESP that is not the booted 6.19.14 UKI, the secondary 6.17.1 UKI, the bare-kver B.1 artifact, or the two `test-*-pcrsig-*` B.3 artifacts. Investigate provenance immediately; do not declare Gate 7 closed. See `06F` L.1 for the documented allowlist scheme. |
| Step 37 Gate 7 audit reveals an EXPECTED UKI failing sbverify, missing `.pcrpkey`/`.pcrsig`, or with anomalous VMA | The validated MID-prefixed UKIs must remain firmware-loadable and structurally sound. Severe. Rollback to `module5-b2-4-pre-reboot`. |
| Step 38 prime/mask fails (state-dir wrong mode, mask not applied) | Chain not installed cleanly. Do not publish the `60-` rule. Rollback to `module5-b4-pre-install-publish` (deeper: `module5-b4-pre-design-verification`). |
| Step 38 Step 0/8 `qm snapshot` fails | Proxmox-host issue (thin-pool Data% ≥ 75%, name collision, parent missing). Triage on host pve-host. Without the Step 8 closure snapshot `module5-b4-installed`, do not begin B.5. |
| Step 39 Gate 3 `rc != 0`, sentinel present, or `helper-last-error`/`last-error` present | Verdict prints `FAILED/UNSAFE: do not reboot`. Do not proceed to Gate 5/6. Inspect the tee'd transaction log and bounded per-tag journals under `/root/tboot-lab/artifacts/`. Roll back to `module5-b5-pre-firstfire` if the ESP or state is left unrecoverable. |
| Step 39 Gate 4 decision-line still `drift` | Convergence did not complete: B.4 helper signed/propagated but the decider did not advance the baseline to converged shape. Halt; do not reboot. (A bare `equal=1 shape=1` numeric form is NOT produced on the converged path; the worded `unchanged (… converged-shape)` form is correct: `06F` O.1.) |
| Step 39 Gate 6b runtime PCR 11 != `28E66CE2…C90F`, passphrase prompt appears, or Secure Boot not enabled | Boot chain did not validate. The keyslot-0 passphrase remains the fallback (fail-safe, not a brick). Roll back to `module5-b5-pre-reboot` and triage. |
| Step 39 Gate 5/7 `qm snapshot` fails | Proxmox-host issue (thin-pool Data% ≥ 75%, name collision, parent missing). Triage on host pve-host. |
| Total VM corruption | Rollback to most recent good snapshot from `qm listsnapshot 500`. The lineage in `00_Current_Project_State.md` shows valid targets. |


# Appendix C: Validation history

The authoritative validation history lives in `00_Current_Project_State.md`.

This runbook records the reproducible execution path. It intentionally does not maintain a second milestone ledger, because duplicated validation history can drift from the project-state file.


# Appendix D: Forward work

Forward work and next-session priorities are tracked in `00_Current_Project_State.md`.
