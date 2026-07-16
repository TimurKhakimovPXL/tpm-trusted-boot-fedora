<!--
PUBLIC RELEASE VERSION - sanitized for publication.
Machine-id and host names are format-valid EXAMPLE values, not the original lab
identifiers. PCR values, policy-key digests and file hashes are genuine
measurements from the validated build and are public values by design.
-->

---
tags:
  - trusted-boot
  - forensic
  - diagnostic
  - bug-catalog
  - deprecated-procedures
module: 6
category: forensic-catalog
status: stable
last-updated: 2026-07-16
---

# 06F: Diagnostic Findings and Deprecated Procedures

This catalog records observed failures, near misses and known risks in the
trusted boot project. It is written for two audiences:

- Operators following `06B_Golden_Path_Rebuild_Runbook.md` who need the reason
  behind a step or need to match an unexpected symptom to a known failure.
- Maintainers auditing the engineering process and checking whether a procedure
  remains safe.

Every entry has a verdict that tells you whether the procedure described is forbidden, conditionally safe, or just informational context. Entries that document forbidden procedures are marked clearly so they cannot be misread as recommendations.

This file replaces the earlier split between `06F_Diagnostic_Findings_Catalog.md` and `06X_Deprecated_Procedures.md`. The two views (forensic findings and deprecated rules) covered the same content from different angles, and keeping them in sync was friction. They are merged here.

**Companion documents:**

- `06B_Golden_Path_Rebuild_Runbook.md`: the validated rebuild path. Already incorporates every correction listed in this file.
- `00_Current_Project_State.md`: where the lab is right now.
- `06_Lab_Setup_Runbook*.md`: the original chronological evidence files. Preserved for audit until the migration to this file is verified complete; references in this document should resolve there until then.

**Per-finding format:**

- **Verdict:** `FORBIDDEN`, `CONDITIONAL`, or `CONTEXT-ONLY`.
  - `FORBIDDEN` means the procedure produces broken systems and must not be followed even with apparent justification.
  - `CONDITIONAL` means the procedure is safe in some uses and unsafe in others; the entry explains where the line is.
  - `CONTEXT-ONLY` means the entry is informational, capturing a behavior or constraint you should know about. No procedure to forbid.
- **Symptom:** what made the issue visible.
- **Root cause:** why it happens, at the mechanism level.
- **Reproduction:** the minimum command sequence that triggers it.
- **Fix or workaround:** what to do instead.
- **Cross-reference:** which evidence section, and which `06B` step (if relevant).

**Categories:**

- A: Boot path, Secure Boot, firmware
- B: UKI construction (PE+/Authenticode/sections)
- C: TPM2 and systemd-creds
- D: systemd-measure and ukify
- E: kernel-install and DNF
- F: Tooling and linting
- G: LUKS2 and systemd-cryptenroll
- H: DNF5 plugin layer and trigger mechanics
- I: Gate 3 environment and harness findings (decider install + baseline prime)
- J: Gate 4–7 environment and harness findings (DNF live trigger)
- M: Mentor-notes deprecations (Debian-isms that don't translate to Fedora)

A symptom-keyword index for fast lookup is at the end, followed by a quick-reference table of forbidden procedures.


---

## A: Boot path, Secure Boot, firmware

### A.1: systemd-boot does not chainload BLS Type #1 entries from `/boot/loader/entries/`

**Verdict:** FORBIDDEN (the chainload approach, not the symptom).

**Symptom:**
After enrolling custom PK/KEK/db and signing Fedora's `/boot/vmlinuz-*` with the new db key, the system boots to the systemd-boot menu, but Fedora's installed kernel entries are not visible. Selecting them produces no usable boot. systemd-boot offers only its own minimal menu without the kernel entries that Fedora's kernel-install populated under `/boot/loader/entries/*.conf`.

**Root cause:**
systemd-boot is a Type #2 BLS loader. It reads UKI (`.efi`) entries from `$BOOT_ROOT/EFI/Linux/`. It does not read Type #1 BLS entries (`.conf` files in `/boot/loader/entries/`) populated by Fedora's GRUB-driven layout. Fedora puts kernels and BLS entries on the separate `/boot` partition (typically sda2), and BLS Type #1 entries are designed for GRUB's parser, not systemd-boot. The assumption that "systemd-boot will pick up Fedora's BLS entries automatically" is incorrect.

**Reproduction:**

1. Install Fedora with default partition layout (separate `/boot`).
2. Enroll custom Secure Boot keys in OVMF.
3. Sign `/boot/vmlinuz-*` with the custom db key.
4. Install systemd-boot via `bootctl install`.
5. Reboot. Fedora kernel entries are absent from the systemd-boot menu.

**Fix or workaround:**
Build a Unified Kernel Image and place it at `$BOOT_ROOT/EFI/Linux/${KVER}.efi`. systemd-boot discovers Type #2 entries automatically. The validated approach is `ukify build` against `/lib/modules/${KVER}/vmlinuz` and `/boot/initramfs-${KVER}.img`, signed with the db key, written to ESP under `EFI/Linux/`.

**Operational consequence (former 06X D.5):**
Do not sign Fedora's vmlinuz directly and expect systemd-boot to chainload via Type #1 BLS. The transitional approach in `06_Lab_Setup_Runbook.md` §11 is preserved as a failure record. Use the UKI-first approach (`06B` Steps 7 and 8).

**Cross-reference:**
- `06B`: Steps 7 (uki.conf), 8 (build initial UKI), 9 (install systemd-boot)
- Evidence: `06_Lab_Setup_Runbook.md` §11 (Phase 1 Attempt 1, failure analysis), §12 (Phase 1 Attempt 2, UKI-first)

---

### A.2: `--cpu host` causes kernel panic in nested virtualisation

**Verdict:** CONTEXT-ONLY. Operational fix, not a deprecation.

**Symptom:**
On Contabo VPS hosts running Proxmox in nested virt, VMs created with `--cpu host` panic during boot:

```
Attempted to kill init! exitcode=0x0000000b
SRCU: Active srcu_struct ffffffffb15ed580 read locked.
Kernel panic - not syncing: Attempted to kill init!
```

The panic is reproducible across multiple VM creations and kernel versions; it is not a one-off corruption.

**Root cause:**
`--cpu host` exposes the host's full CPU feature set to the guest, including features the nested virtualisation layer doesn't faithfully implement. Specific extensions (likely AVX-512 family on Contabo's underlying hardware) crash early boot when the guest kernel attempts to use them. The host CPU advertises capabilities that the L1 hypervisor (Proxmox running nested) cannot properly virtualise to the L2 guest.

**Reproduction:**

```bash
qm create 500 --cpu host ...
qm start 500
# observe panic in console
```

**Fix or workaround:**
Use a baseline CPU profile that excludes problematic extensions:

```bash
qm set 500 --cpu x86-64-v2-AES
```

`x86-64-v2-AES` provides modern features (SSE4.2, AES-NI, POPCNT) without the AVX-512+ extensions that break nested virt. Validated as stable across all subsequent project work.

**Cross-reference:**
- `06B`: Step 2 (VM creation, `--cpu x86-64-v2-AES`)
- Evidence: `06_Lab_Setup_Runbook.md` §6.1 (Kernel Panic), §16.1

---

### A.3: Anaconda refuses partition layout without `/boot`

**Verdict:** CONTEXT-ONLY. Fedora installer constraint, accepted as given.

**Symptom:**
Attempting to install Fedora with `/boot/efi` and `/` only (no separate `/boot`), to simplify the partition layout for UKI-only boot, Anaconda halts the install with:

```
You must include at least one /boot partition.
```

The installer cannot be coerced past this gate even with manual partitioning.

**Root cause:**
Fedora's Anaconda has a hard-coded validation that the partition layout includes a separate `/boot` partition. This is a legacy of GRUB-era expectations and isn't relaxed for UKI-only setups, even though UKIs go on the ESP and a separate `/boot` is technically redundant when no BLS Type #1 entries are used.

**Reproduction:**
Run the Fedora 43 Server installer, manual partition layout: assign `/boot/efi` (vfat, 1 GiB) and `/` (ext4, remainder). Anaconda blocks with the above error.

**Fix or workaround:**
Accept the separate `/boot` partition. Default layout: `/boot/efi` (vfat, 1 GiB), `/boot` (ext4, 1 GiB), `/` (ext4 or LUKS, remainder). The `/boot` partition is unused by the trusted-boot path. Kernels and initramfs are referenced from `/lib/modules/${KVER}/vmlinuz` and `/boot/initramfs-${KVER}.img` for UKI builds; the bare-kver `/boot/vmlinuz-*` files are also present and harmless.

**Cross-reference:**
- `06B`: Step 3 (Fedora install)
- Evidence: `06_Lab_Setup_Runbook.md` §6.2

---

### A.4: `efi-readvar` puts the CN value on a separate indented line

**Verdict:** CONTEXT-ONLY.

**Symptom:**
After enrolling PK/KEK/db, an operator verifies the firmware variables with `efi-readvar -v db | grep -E 'Subject|Issuer'` and gets effectively empty output:

```
        Subject:
        Issuer:
```

The CN values (e.g. `tboot-lab Signature Database Key`) are absent. This looks like enrollment failed or the variable is empty, but the keys are actually present and the firmware is correctly populated.

**Root cause:**
`efi-readvar -v` formats output across multiple lines. The `Subject:` and `Issuer:` labels print on one line, and the CN value prints on the *next* line, indented:

```
        Subject:
            CN=tboot-lab Signature Database Key
        Issuer:
            CN=tboot-lab Signature Database Key
```

A grep that filters by label captures only the labels and drops the CN lines that follow.

**Reproduction:**

```bash
efi-readvar -v db | grep -E 'Subject|Issuer'
# Output: just the labels, values appear missing.

efi-readvar -v db
# Full output: labels and CN values both visible, CN indented under each label.
```

**Fix or workaround:**
For verifying efivar contents, prefer reading the raw efivar bytes directly. The kernel exposes them under `/sys/firmware/efi/efivars/` and `strings` will pull printable substrings out cleanly:

```bash
DB_VAR=$(ls /sys/firmware/efi/efivars/db-* | head -1)
stat -c%s "$DB_VAR"               # size includes 4-byte attr prefix + ESL
strings "$DB_VAR" | grep tboot-lab
```

The kernel's view of the bytes bypasses any tool parsing. Three matches for `tboot-lab` against the db efivar means the cert is in there.

If you do want to use `efi-readvar`, drop the grep and read the full output, or grep with sufficient context (`-A 1`) to capture the CN line:

```bash
efi-readvar -v db | grep -A 1 -E '^\s*(Subject|Issuer):'
```

**Cross-reference:**
- `06B`: Step 11 (post-enrollment verification)
- Evidence: `06_Lab_Setup_Runbook.md` §16.3

---

## B: UKI construction (PE+/Authenticode/sections)

### B.1: `objcopy --add-section` produces a firmware-rejected UKI

**Verdict:** FORBIDDEN for production UKI construction. CONDITIONAL for read-only inspection (see B.3 and the "objcopy: safe vs unsafe" subsection below).

**Symptom:**
A UKI built by manually attaching `.pcrsig` via `objcopy --add-section` passes every Linux-side validation check:

```
.pcrpkey present
.pcrsig present
sbverify OK
signer count = 1
```

But UEFI rejects it at `LoadImage()` with:

```
Error loading EFI binary \EFI\Linux\<id>.efi: Load Error
```

Boot fails. The fallback UKI loads cleanly, proving the firmware works correctly. Only the objcopy-built UKI is rejected.

**Root cause:**
`objcopy --add-section` does not perform full PE+ layout rewriting. When adding a new section to an already-signed PE file, objcopy places the section at an anomalous virtual address, typically around `0x200000000`, far from the contiguous low-GB span of the original sections. UEFI's `LoadImage()` enforces PE+ layout invariants that `sbverify` does not check, so the file passes signature validation while violating loadability invariants.

The Linux toolchain (`sbverify`, `objdump -h`, `pesign --show-signature`) inspects sections individually and verifies the Authenticode hash; none of these tools enforce VMA layout sanity.

**Reproduction:**

```bash
# Build a UKI normally, then attach .pcrsig via objcopy
ukify build --linux=... --initrd=... --output=base.efi
objcopy --add-section .pcrsig=pcrsig.json base.efi enriched.efi
sbsign --key db.key --cert db.crt enriched.efi --output final.efi

# Linux-side checks all pass:
objdump -h final.efi | grep -E '\.pcrsig|\.pcrpkey'   # both present
sbverify --cert db.crt final.efi                       # exit 0

# But check VMAs. An anomalous one indicates the bug:
objdump -h final.efi | awk '/^[[:space:]]+[0-9]+[[:space:]]+\./ {print $2 " " $4}'
# Look for a VMA at 0000000200000000 or similar high value

# Boot the result:
bootctl set-oneshot final.efi-name
reboot
# Boot fails: Error loading EFI binary \EFI\Linux\<id>.efi: Load Error
```

**Fix or workaround:**
Build the UKI with all sections in one ukify invocation, including PCR signing flags:

```bash
ukify build \
  --linux=... \
  --initrd=... \
  --pcr-private-key=... \
  --pcr-public-key=... \
  --phases=enter-initrd \
  --pcr-banks=sha256 \
  --secureboot-private-key=db.key \
  --secureboot-certificate=db.crt \
  --output=final.efi
```

ukify computes `.pcrsig`, places it at a sane VMA contiguous with other sections, and signs the result, all in one PE+ construction pass. No separate objcopy step.

For defense in depth, the validated 80-tpm2-sign hook (stage 10.4) checks every section's VMA against a `0x180000000` threshold and rejects UKIs with anomalous values:

```bash
objdump -h "$INSPECT_COPY" | awk '
  /^[[:space:]]+[0-9]+[[:space:]]+\./ {
    vma = strtonum("0x" $4)
    if (vma >= strtonum("0x180000000")) { bad = 1 }
  }
  END { exit !bad }
'
```

**Operational consequences:**

*Hook validation (former 06X D.2):* Validating a hook by checking only that all sections are present, sbverify passes, and signer count is 1, is insufficient. These structural checks pass on the broken-objcopy hook but do not detect firmware rejection. Validation must include section-VMA sanity (above) or per-section sha256 comparison against a known-good reference UKI.

*Broken hook installation (former 06X D.3):* Installing the objcopy-form hook at `/etc/kernel/install.d/80-tpm2-sign.install` is forbidden. Any subsequent `dnf upgrade kernel*` or `dnf reinstall kernel*` would invoke the broken hook and produce a firmware-rejected UKI on the ESP, bricking the kernel update path until manual recovery. The hook with sha `0660b0f50e3d0ba583e266bfdbcd323faea8c951a5e02affc4c1f96e3f572673` is the broken form. The validated v2.1.2-r1 hook is sha `5857e51d5551e05a7ca384f71529ec2fef07f5a6dbe40df70c6bbe49d720947e`.

*Broken-hook snapshot (former 06X D.4):* The Proxmox snapshot `module5-kernel-signing-hook-installed` captures the broken-hook state. It is preserved for forensic reference but **must not be used as a rollback target for active work**. Rolling forward from it leaves a non-bootable update path the next time DNF runs a kernel transaction. The current rollback target is `module5-kernel-signing-hook-validated`. See `00_Current_Project_State.md` snapshot lineage.

*Reboot to objcopy UKI (former 06X D.8):* Booting a UKI built via the broken objcopy path is reproducibly destructive. The only legitimate use is forensic reproduction of the failure mode. The forensic UKI is preserved at `/boot/efi/EFI/Linux/cf44500de…6bc70-6.19.14-200.fc43.x86_64.efi.loaderror` with `.loaderror` suffix to prevent accidental boot selection.

*Stale dry-run methodology (former 06X S.1):* The dry-run pattern in `06_Lab_Setup_Runbook_continuation.md` §18.4 was sound, but the specific dry-run validated the broken-objcopy hook, so its output values are not authoritative. The validated equivalent is in `06_Lab_Setup_Runbook_continuation_v3.md` §20.3 to §20.5, against the v2.1.2-r1 hook.

**objcopy: safe vs unsafe (former 06X C.3):**

objcopy is **safe** for read-only inspection on copied files. It is **not safe** for production UKI construction.

Safe (read-only, on a copy):

```bash
cp /boot/efi/EFI/Linux/${MID}-${KVER}.efi /tmp/inspect.efi
objcopy --dump-section .pcrsig=/tmp/pcrsig.json /tmp/inspect.efi /tmp/throwaway.efi
rm /tmp/throwaway.efi
# inspect /tmp/pcrsig.json, then shred it
```

Unsafe (production write paths):

```bash
# DO NOT FOLLOW. Produces firmware-rejected UKI per the root cause above.
objcopy --add-section .pcrsig=... uki.efi
```

The hook's stage 10 validation deliberately copies the UKI to a tmpfs scratch path before running objdump or objcopy inspection on it, to prevent any accidental write-path on the production file.

**Cross-reference:**
- `06B`: Step 14 (validated v2.1.2-r1 hook source), Step 15 (install hook), Step 18 (validated snapshot)
- Evidence: `06_Lab_Setup_Runbook_continuation.md` §16.13 (root cause), §18 (Block B.1 broken-objcopy execution log), `06_Lab_Setup_Runbook_continuation_v2_1_patches.md` §20.5b (forensic reboot record)

**Meta-finding:**
**`sbverify OK` is not proof of firmware-loadability.** Linux-side validation
tools check signature integrity but not layout invariants. Production UKI
validation must include either per-section sha256 comparison against a
known-good reference or a section-VMA sanity check against the `0x180000000`
threshold. Block B established this requirement.

---

### B.2: `ukify --join-pcrsig` silently no-ops on systemd 258.7

**Verdict:** FORBIDDEN. The flag exists but does not do what its documentation says.

**Symptom:**
Running ukify with `--join-pcrsig=$EXISTING_UKI --pcrsig=@$PCRSIG_JSON --output=$OUT` exits 0 with no stderr, but the resulting UKI does not contain the `.pcrsig` section. `objdump -h` confirms no new section. The hook silently produces a UKI that the runtime cannot forward-seal against.

**Root cause:**
In systemd 258.7-1.fc43, the `--join-pcrsig` code path is incomplete. Upstream documentation (Example 8 in the man page) shows the syntax as supported, but the implementation does not actually attach the section. ukify exits 0, no warning, no error, silent no-op.

**Reproduction:**

```bash
# Have a base UKI without .pcrsig:
ukify build --linux=... --initrd=... --output=base.efi

# Generate a .pcrsig JSON from systemd-measure:
/usr/lib/systemd/systemd-measure sign \
  --linux=... --initrd=... --osrel=... --cmdline=... --uname=... --sbat=... --pcrpkey=... \
  --private-key=$KEY --public-key=$PUB --bank=sha256 --phases=enter-initrd > pcrsig.json

# Try to join:
ukify --config=/dev/null build --join-pcrsig=base.efi --pcrsig=@pcrsig.json --output=joined.efi

# Verify:
objdump -h joined.efi | grep '.pcrsig'   # absent
echo "exit code was: $?"                 # ukify exited 0
```

**Fix or workaround:**
Do not use `--join-pcrsig`. Build the UKI fresh with all native PCR signing flags in a single `ukify build`:

```bash
ukify build \
  --linux=... --initrd=... --cmdline=@/etc/kernel/cmdline --os-release=@/etc/os-release \
  --pcr-private-key=$KEY --pcr-public-key=$PUB \
  --phases=enter-initrd --pcr-banks=sha256 \
  --secureboot-private-key=db.key --secureboot-certificate=db.crt \
  --output=$OUT
```

The validated `80-tpm2-sign` hook uses this path.

**Cross-reference:**
- `06B`: Step 14 (hook stage 7 uses native PCR signing)
- Evidence: `06_Lab_Setup_Runbook_continuation.md` §16.13, §18.4.1

---

### B.3: `objcopy --add-section` may shrink the resulting PE due to layout rewrite

**Verdict:** CONTEXT-ONLY. A symptom-level finding that complements B.1.

**Symptom:**
After running `objcopy --add-section` on a signed UKI, the output file is *smaller* than the input despite a section being added. The Authenticode signature has been silently dropped during objcopy's PE rewrite, and the signature bytes that were appended in the PE overlay area are gone.

**Root cause:**
`objcopy` rewrites the PE structure when manipulating sections. PE+ Authenticode signatures live in the *overlay area*, the bytes appended after the last formal section. The PE section table doesn't reference the overlay, so objcopy considers it junk and drops it on rewrite. This is true even for nominally read-only operations like `--dump-section` if objcopy decides to rewrite the file.

**Reproduction:**

```bash
ukify build ... --output=signed.efi
SIZE_BEFORE=$(stat -c%s signed.efi)
objcopy --add-section .extra=/dev/null signed.efi modified.efi
SIZE_AFTER=$(stat -c%s modified.efi)
echo "before=$SIZE_BEFORE after=$SIZE_AFTER"
sbverify --cert db.crt modified.efi   # signature verification fails
```

**Fix or workaround:**
For inspection, always copy to `/tmp` first and operate on the copy. Never run objcopy directly against the production UKI on the ESP. See B.1 "objcopy: safe vs unsafe" for the safe usage pattern.

**Cross-reference:**
- `06B`: Step 16 (forensic capture pattern uses tmpfs copies)
- Evidence: `06_Lab_Setup_Runbook_continuation.md` §16.13

---

### B.4: `[Signing]` section in `/etc/kernel/uki.conf` is silently ignored

**Verdict:** FORBIDDEN. A non-recognised config section that produces unsigned UKIs without warning.

**Symptom:**
A `uki.conf` written with the `[Signing]` section name produces unsigned UKIs from kernel-install:

```ini
[UKI]
Cmdline=@/etc/kernel/cmdline
OSRelease=@/etc/os-release

[Signing]
SecureBootPrivateKey=/etc/uefi-keys/db.key
SecureBootCertificate=/etc/uefi-keys/db.crt
```

The kernel-install run completes without error, but the resulting UKI is unsigned. Downstream, the 80-tpm2-sign hook's stage 3 sbverify check then fails and aborts the transaction.

**Root cause:**
`[Signing]` is **not** a recognised ukify config section. Both manual `ukify` and the `60-ukify.install` plugin's internal `apply_config()` parse the same set of sections; neither recognises `[Signing]`. The parser silently skips unknown sections. The configuration silently does nothing. Section names recognised by ukify are `[UKI]`, `[PCRSignature:NAME]`, and a few others documented in `ukify(1)`. There is no `[Signing]`.

**Reproduction:**
Write the above `uki.conf`, then run `kernel-install add $KVER /lib/modules/$KVER/vmlinuz`. Inspect the produced UKI:

```bash
sbverify --list /boot/efi/EFI/Linux/${MID}-${KVER}.efi
# image signature issuers: (none)
```

**Fix or workaround:**
Place `SecureBootPrivateKey=` and `SecureBootCertificate=` inside the `[UKI]` section, per `ukify(1)` Example 4:

```ini
[UKI]
Cmdline=@/etc/kernel/cmdline
OSRelease=@/etc/os-release
SecureBootPrivateKey=/etc/uefi-keys/db.key
SecureBootCertificate=/etc/uefi-keys/db.crt

[PCRSignature:initial]
PCRPublicKey=/etc/systemd/tpm2-pcr-public-key.pem
```

**Cross-reference:**
- `06B`: Step 7 (uki.conf with corrected `[UKI]` section)
- Evidence: `06_Lab_Setup_Runbook.md` §16.2 (original incorrect form), `06_Lab_Setup_Runbook_continuation.md` §16.2 (correction)

---

## C: TPM2 and systemd-creds

### C.1: `systemd-creds encrypt` auto-discovers public key, silently embeds forward-seal policy

**Verdict:** CONTEXT-ONLY. An undocumented systemd-creds behavior that must be worked around.

**Symptom:**
Sealing the policy private key with `systemd-creds encrypt --tpm2-pcrs=7` produces a credential that, on decrypt, requires `--tpm2-signature=` despite no `--tpm2-public-key=` argument being passed at encrypt time. This breaks the desired model where the credential is bound only to PCR 7 (Secure Boot policy, available immediately at boot) without forward-seal coupling.

**Root cause:**
`systemd-creds encrypt` checks for `/etc/systemd/tpm2-pcr-public-key.pem` and, if found, silently treats the operation as if `--tpm2-public-key=...` were specified, embedding a forward-sealed PCR policy in the credential. The auto-discovery is undocumented in the systemd-creds man page as of systemd 258.

**Reproduction:**

```bash
# With /etc/systemd/tpm2-pcr-public-key.pem present:
systemd-creds encrypt --tpm2-pcrs=7 --name=foo plain.txt encrypted.cred
# Decrypt fails:
systemd-creds decrypt --with-key=tpm2 --name=foo encrypted.cred decrypted.txt
# Error: signature missing (or similar)
```

**Fix or workaround:**
Park the public key out of `/etc/systemd/` during the encrypt step, then restore it:

```bash
mv /etc/systemd/tpm2-pcr-public-key.pem /tmp/pubkey-park.pem
systemd-creds encrypt --tpm2-pcrs=7 --name=tpm2-pcr-signing-key \
  /etc/systemd/tpm2-pcr-private-key.pem \
  /etc/systemd/tpm2-pcr-private-key.pem.enc
mv /tmp/pubkey-park.pem /etc/systemd/tpm2-pcr-public-key.pem
```

`06B` Step 12 applies this workaround.

**Cross-reference:**
- `06B`: Step 12 (Block A sealing, with the public-key parking workaround)
- Evidence: `06_Lab_Setup_Runbook_continuation.md` §16.7

---

### C.2: `systemd-creds decrypt` writes non-fatal stderr noise

**Verdict:** CONTEXT-ONLY.

**Symptom:**
A successful `systemd-creds decrypt` invocation in a hook context emits stderr lines like:

```
Failed to read system token: No such file or directory
```

The decrypt itself succeeds (exit 0, plaintext written), but the stderr noise is visible in the hook's logger output and looks like a failure.

**Root cause:**
systemd-creds tries to read system credentials from EFI variables and other
locations as part of its discovery chain. When some of those sources are
absent, which is normal for a pure TPM2 unseal in a hook context, it logs the
absence to stderr without making the operation fail. Determine success from the
exit code, not from the presence of stderr output.

**Fix or workaround:**
Trust the exit code. Capture stderr if you want to log it, but do not treat non-empty stderr as a failure signal. The validated hook does this correctly.

**Cross-reference:**
- `06B`: Step 14 (hook stage 5 unseal, treats exit code as authoritative)
- Evidence: `06_Lab_Setup_Runbook_continuation.md` §16.8

---

## D: systemd-measure and ukify

### D.1: `systemd-measure` is not on `$PATH`

**Verdict:** CONTEXT-ONLY.

**Symptom:**
A script that calls `systemd-measure calculate ...` fails with `bash: systemd-measure: command not found`, even though the systemd package is installed.

**Root cause:**
systemd-measure is a private utility, installed at `/usr/lib/systemd/systemd-measure`, not in any directory on `$PATH`. This is intentional per upstream packaging: it is not a user-facing command, and its CLI is not stable across systemd versions.

**Fix or workaround:**
Use the absolute path `/usr/lib/systemd/systemd-measure` everywhere, or define a constant once near the top of any script:

```bash
readonly MEASURE="/usr/lib/systemd/systemd-measure"
"$MEASURE" calculate --linux=... ...
```

The validated hook uses the absolute path. The runbook's prediction step (`06B` Step 17) does the same.

**Cross-reference:**
- `06B`: Step 17 (PCR 11 prediction)
- Evidence: `06_Lab_Setup_Runbook_continuation.md` §16.10, `06_Lab_Setup_Runbook_continuation_v3.md` §16.22

---

### D.2: `systemd-measure calculate` parser must handle stderr/stdout split

**Verdict:** CONTEXT-ONLY.

**Symptom:**
A naive parser of `systemd-measure calculate` output (`awk '/^11:sha256=/ {print $1}'`) returns empty. The expected `11:sha256=...` lines are nowhere visible when piping to a single grep.

**Root cause:**
`systemd-measure calculate` writes phase boundary headers to stderr and the actual `11:sha256=...` values to stdout. If the script captures only stdout (or only stderr), it sees a partial output. Additionally, the command emits one `11:sha256=...` line per phase boundary; the *last* line is the final post-`ready` value, not the first.

**Reproduction:**

```bash
"$MEASURE" calculate --linux=... ... 2>&1 \
  | grep '^11:sha256='
# Multiple lines:
# 11:sha256=...   (after enter-initrd)
# 11:sha256=...   (after leave-initrd)
# 11:sha256=...   (after sysinit)
# 11:sha256=...   (after ready, this is the one we want)
```

**Fix or workaround:**
Combine stderr and stdout, grep for the `11:sha256=` lines, take the last one:

```bash
PRED_PCR11="$("$MEASURE" calculate ... 2>&1 | grep -E '^11:sha256=' | tail -1 | cut -d= -f2)"
```

The runbook step 17 uses this idiom.

**Cross-reference:**
- `06B`: Step 17 (parser idiom in commands and note)
- Evidence: `06_Lab_Setup_Runbook_continuation_v3.md` §16.23

---

### D.3: ukify accepts `[PCRSignature:NAME]` with only `PCRPublicKey=`

**Verdict:** CONTEXT-ONLY.

**Symptom:**
A `uki.conf` with `[PCRSignature:initial]` containing only `PCRPublicKey=` (no private key, no phases) does not produce a `.pcrsig` section in the resulting UKI. ukify accepts the config and exits 0, but the UKI lacks the section.

**Root cause:**
Without a private key, ukify cannot create a signature. A
`[PCRSignature:NAME]` section containing only `PCRPublicKey=` defines the
profile used when a private key is supplied later through command-line flags.
The validated hook supplies that key with `--pcr-private-key=` and ukify then
creates the `.pcrsig` section.

**Fix or workaround:**
None needed. This is the intended design. The `uki.conf` slot in `06B` Step 7 is shaped this way deliberately so that future hook runs don't have to mutate the config.

**Cross-reference:**
- `06B`: Step 7 (uki.conf slot), Step 14 (hook provides private key at runtime)
- Evidence: `06_Lab_Setup_Runbook_continuation.md` §16.11

---

### D.4: `ukify build` does not read `/etc/kernel/uki.conf`

**Verdict:** CONTEXT-ONLY. Two separate config-file paths exist; knowing which is which prevents silently producing unsigned UKIs.

**Symptom:**
`ukify build --linux=... --initrd=... --output=...` (without explicit signing flags) produces an unsigned UKI with no `.cmdline` section, despite `/etc/kernel/uki.conf` containing `[UKI] Cmdline=...` and the matching SecureBoot key paths.

**Root cause:**
Two config-file paths exist, and they are read by different invocation paths:

| Config | Consumed by | When |
|---|---|---|
| `/etc/kernel/uki.conf` | `kernel-install`'s `60-ukify.install` plugin | Automatic, on kernel package install or `kernel-install add` |
| `/etc/systemd/ukify.conf` and `ukify.conf.d/*.conf` | `ukify` invoked directly from the command line | When you run `ukify build` manually |

Neither path consumes the other's config. A manual `ukify build` invocation does not read `/etc/kernel/uki.conf`; it reads `/etc/systemd/ukify.conf` (if present) and CLI flags only.

**Reproduction:**

```bash
# /etc/kernel/uki.conf is fully populated with [UKI] section + SecureBoot keys
ukify build --linux=/boot/vmlinuz-... --initrd=/boot/initramfs-...img --output=test.efi

sbverify --list test.efi
# image signature issuers: (none)
objdump -h test.efi | grep '.cmdline'
# (absent)
```

**Fix or workaround:**
Manual `ukify build` invocations need explicit CLI flags for everything they want included:

```bash
ukify build \
  --linux=/boot/vmlinuz-${KVER} \
  --initrd=/boot/initramfs-${KVER}.img \
  --cmdline=@/etc/kernel/cmdline \
  --os-release=@/etc/os-release \
  --secureboot-private-key=/etc/uefi-keys/db.key \
  --secureboot-certificate=/etc/uefi-keys/db.crt \
  --output=/boot/efi/EFI/Linux/${KVER}.efi
```

`06B` Step 8 passes these options explicitly. The same values also appear in
`uki.conf`, but the two copies serve different invocation paths. `uki.conf` is
used by `kernel-install` from Step 16 onward, while the Phase 1 manual build in
Step 8 requires the command-line options.

If you want a single config consumed by both paths, write the same content to both `/etc/kernel/uki.conf` and `/etc/systemd/ukify.conf`. The bundle does not do this; the manual Phase 1 path is one-shot, and duplicating the config to keep both in sync long-term is a maintenance burden.

**Cross-reference:**
- `06B`: Step 7 (uki.conf for kernel-install), Step 8 (manual ukify build with explicit flags)
- Evidence: `06_Lab_Setup_Runbook.md` §16.2

---

## E: kernel-install and DNF

### E.1: kernel-install produces MID-prefixed UKIs, not bare-kver UKIs

**Verdict:** CONDITIONAL. Both paths are valid; choose based on context.

**Symptom:**
After a `dnf reinstall kernel-core-${KVER}` run with the validated hook in place, an unexpected file appears on the ESP:

```
/boot/efi/EFI/Linux/0123456789abcdef0123456789abcdef-6.19.14-200.fc43.x86_64.efi
```

next to the existing Phase 1 manual UKI:

```
/boot/efi/EFI/Linux/6.19.14-200.fc43.x86_64.efi
```

Both are signed; both reference the same kernel; only the MID-prefixed one has `.pcrsig`.

**Root cause:**
kernel-install names produced UKIs as `${KERNEL_INSTALL_ENTRY_TOKEN}-${KVER}.efi`, where `$KERNEL_INSTALL_ENTRY_TOKEN` defaults to the contents of `/etc/machine-id`. This is the upstream-documented behavior, not a bug. The Phase 1 manual `ukify build --output=/boot/efi/EFI/Linux/${KVER}.efi` step (`06B` Step 8) writes to a different path, so the two coexist without conflict.

**Operational consequence (former 06X C.2):**
When referring to "the canonical ESP UKI path," distinguish between:

- `/boot/efi/EFI/Linux/${KVER}.efi`: the Phase 1 manual fallback UKI, no `.pcrsig`.
- `/boot/efi/EFI/Linux/${MID}-${KVER}.efi`: the kernel-install-produced, hook-signed, active forward-sealed UKI.

Both coexist on the ESP after Block B.3. Module 3 LUKS keyslot enrollment (`systemd-cryptenroll --tpm2-signature=`) must reference the MID-prefixed path.

**Fix or workaround:**
None needed; this is upstream-documented behavior. The runbook's Step 19 (`bootctl set-default ${MID}-${KVER}.efi`) explicitly converts the MID-prefixed UKI to the persistent default before Module 3.

**Cross-reference:**
- `06B`: Step 16 (live transaction, MID-prefixed UKI produced), Step 19 (set as default)
- Evidence: `06_Lab_Setup_Runbook_continuation_v3.md` §16.21

---

### E.2: "DNF holds" referenced throughout the runbook were operator discipline, not configuration

**Verdict:** CONDITIONAL CLARIFICATION. The framing is the issue, not a procedure to forbid.

**Symptom:**
Snapshot descriptions and runbook narrative reference "Holds in effect: dnf upgrade kernel*, dnf reinstall kernel*, dnf upgrade systemd*". A new operator reads this and assumes DNF will refuse those commands. It will not.

**Root cause:**
These are **operator discipline** gates ("do not run these commands"), not DNF configuration. Verified across the lab:

- No `versionlock` plugin installed.
- No `exclude=` or `excludepkgs=` in `/etc/dnf/dnf.conf` or repo files.
- No kernel/systemd entries in `/etc/dnf/protected.d/`.

Saying "the hold is in effect" is shorthand for "we have committed not to run this command yet"; it does not describe machine-enforced state.

**Fix or workaround:**
To make a hold machine-enforceable:

```bash
dnf install python3-dnf-plugin-versionlock
dnf versionlock add kernel-core kernel-modules kernel-modules-core
# Or, for a temporary block:
echo 'excludepkgs=kernel*' >> /etc/dnf/dnf.conf   # remember to revert
```

The `06B` runbook treats holds as discipline only. `00_Current_Project_State.md` makes the discipline-only nature explicit.

**Cross-reference:**
- `06B`: Step 16 (live DNF transaction lifts the kernel hold)
- `00_Current_Project_State.md`: Discipline gates table
- Evidence: `06_Lab_Setup_Runbook_continuation_v3.md` §16.20

---

### E.3: PCR 11 value invariance across UKI regeneration when measured-content inputs are unchanged

**Verdict:** CONTEXT-ONLY.

**Symptom:**
After a successful `tboot-dnf-helper` production invocation (or any other path that rebuilds the UKI via `kernel-install add`), the booted-kernel UKI sha256 changes but the `/root/tboot-lab/state/expected-pcr11-after-hook-uki.txt` file content is identical to its pre-invocation value. A naive assertion comparing pre- and post-invocation PCR 11 values reports the value as unchanged and may be misread as a failure.

**Root cause:**
`systemd-measure calculate` produces a deterministic hash over the UKI's measured sections: `.linux`, `.initrd`, `.cmdline`, `.osrel`, `.uname`, `.sbat`, `.pcrpkey`. If those section contents are bit-identical across regenerations (same vmlinuz, same `/etc/kernel/cmdline`, same `/etc/os-release`, same dracut output, same policy pubkey), the resulting PCR 11 hash is identical by construction.

Modern dracut is largely deterministic given the same inputs (same crypttab, same dracut modules, same kernel modules, same package contributions). The UKI sha256 still differs across regenerations because of two non-deterministic operations inside `ukify build`:

- RSA-PSS signing for the `.pcrsig` section uses a random salt; the signature payload differs each run for the same digest.
- sbsign Authenticode signing on the final PE+ binary uses fresh timestamps.

Neither operation contributes to the content measured into PCR 11. Only the section payloads do. So an unchanged input chain produces an unchanged PCR 11 value, even when the resulting UKI sha differs.

This matches the measured-boot design. PCR 11 represents the content measured
by the firmware and stub at runtime, not the file's identity on disk. A changed
value between rebuilds indicates a change in the measured-content chain, such
as an initramfs module, kernel command line or microcode payload.

**Reproduction:**

Run the helper twice with no system mutation between runs. The booted-kernel UKI sha will differ between the two runs; the stored PCR 11 value will be identical.

```bash
# Before
sha256sum "/boot/efi/EFI/Linux/$(cat /etc/machine-id)-$(uname -r).efi"
cat /root/tboot-lab/state/expected-pcr11-after-hook-uki.txt

/usr/local/sbin/tboot-dnf-helper

# After: UKI sha differs, PCR 11 value identical
sha256sum "/boot/efi/EFI/Linux/$(cat /etc/machine-id)-$(uname -r).efi"
cat /root/tboot-lab/state/expected-pcr11-after-hook-uki.txt
```

**Fix or workaround:**

For assertion logic in helper validation, check the PCR 11 file's *mtime advancement* to prove `predict --store` ran (the helper rewrote the file with the latest computed value). Optionally compare the *value* and emit an informational note distinguishing the two cases:

- value identical → measured-content inputs unchanged (the architecturally normal outcome for a regeneration with no upstream change)
- value advanced → measured-content inputs drifted (expected after a real dracut-sensitive update like a microcode or driver bump)

Both outcomes are valid. Only mtime stagnation is a failure signal; it means the predictor did not run.

`06B` Step 30 implements this pattern: `PCR_MTIME -gt $PRE_TEST_EPOCH` is the asserted invariant; value comparison is informational only.

**Operational consequence:**

B.2.4 (live validation through a real dracut-sensitive transaction) is the gate that will exercise the value-change branch. The B.2.2 invocation that closed Step 30 did not trigger any dracut-affecting upstream change, so the value-identical outcome is expected. Do not regard B.2.4 as "stronger validation than B.2.2"; they cover orthogonal axes. B.2.2 proves the machinery runs end-to-end; B.2.4 proves the value tracks input changes.

**Cross-reference:**
- `06B`: Step 30 (helper post-invocation assertion uses mtime advancement, not value comparison)
- Evidence: 2026-05-22 B.2.2 session
- Related determinism context: B.1 (.pcrsig non-determinism inside ukify), B.2 (ukify native PCR signing)

---

## F: Tooling and linting

### F.1: ShellCheck SC2317/SC2329 fires on functions only invoked via `trap`

**Verdict:** CONTEXT-ONLY.

**Symptom:**
ShellCheck (run with `-S info` or stricter) flags the hook's `_cleanup` function:

```
SC2317 (info): Command appears to be unreachable.
SC2329 (info): This function is never invoked.
```

The function is invoked by the `trap _cleanup EXIT HUP INT TERM` line, but ShellCheck's static analysis does not parse trap targets as call sites.

**Root cause:**
ShellCheck's reachability analysis treats trap registration as a string argument, not a function call. Functions that are only ever invoked through trap appear unreachable to the linter. This is a known limitation, not a real bug in the script.

**Fix or workaround:**
Suppress the warning at the function definition with a directive:

```bash
# shellcheck disable=SC2317,SC2329  # invoked via trap below
_cleanup() {
    ...
}
trap _cleanup EXIT HUP INT TERM
```

The validated hook v2.1.2-r1 has this directive. Removing it (e.g. by re-pasting through a markdown renderer that strips comments) will cause the lint check in `06B` Step 14 to fail.

**Cross-reference:**
- `06B`: Step 14 (hook source, SC2317/SC2329 disable on `_cleanup`)
- Evidence: `06_Lab_Setup_Runbook_continuation_v3.md` §16.24

---

### F.2: `pesign --show-signature` outputs one "common name" line per Authenticode signer

**Verdict:** CONTEXT-ONLY.

**Symptom:**
Counting Authenticode signers via `pesign --show-signature ... | grep -c 'common name'` returns 2 on a freshly-signed UKI when only one signing pass was performed.

**Root cause:**
On a doubly-signed PE (e.g. a UKI signed by Fedora, then sbsigned again with the lab db key), pesign reports both signers. The counting idiom is sensitive to this. A single `ukify build` invocation that produces a freshly-built UKI from raw inputs results in exactly one signature; `sbsign` against an already-signed binary appends a second.

**Fix or workaround:**
The validated hook builds the UKI fresh via `ukify build` from raw `vmlinuz` and `initramfs` inputs, which produces a single Authenticode signature. The hook's stage 10.3 enforces signer count == 1 as a sanity check. If it fires, something appended a second signature, which is a workflow bug worth investigating.

For Phase 1 fallback UKIs built via `ukify build` from raw inputs, this is also one signature.

**Cross-reference:**
- `06B`: Step 14 (hook stage 10.3 checks signer count)
- Evidence: `06_Lab_Setup_Runbook_continuation.md` §16.15

---

### F.3: Direct `60-ukify.install` invocation with synthesized environment

**Verdict:** CONDITIONAL. Useful for development methodology, not production.

**Status:**
Synthesizing `KERNEL_INSTALL_*` environment variables and invoking `60-ukify.install` directly is a useful one-off methodology for hook dry-runs. `06B` Step 14 uses this pattern for lint-and-dry-run before installing the hook on the canonical path. It is not a production-supported path. End-to-end validation must use a real `dnf reinstall kernel-core ...` transaction (`06B` Step 16).

**Cross-reference:**
- `06B`: Step 14 (lint and dry-run), Step 16 (live transaction)

---

### F.4: Interactive `set -u` can break shell prompts with unbound `PROMPT_START`

**Verdict:** CONTEXT-ONLY. Not a trusted-boot failure; a shell-environment quirk that bites when validation snippets are pasted directly into an interactive root shell instead of executed as scripts.

**Symptom:**
After enabling strict mode in an interactive root shell (e.g. `set -euo pipefail` then any subsequent command), bash exits with:

```
-bash: PROMPT_START: unbound variable
```

The shell may also drop to an unresponsive state where every prompt redraw fails for the same reason. The user typed nothing wrong; the failure is internal to bash's PS1 expansion machinery.

**Root cause:**
Fedora's bash configuration (specifically `/etc/profile.d/bash-color-prompt.sh` on Fedora 43) references prompt-related variables such as `PROMPT_START`, `PROMPT_END`, and friends via the `${VAR@P}` parameter expansion. These variables are referenced for prompt assembly *only when the prompt is rendered*, and they may legitimately be unset in some prompt themes. With `set -u` active in an interactive shell, every prompt redraw triggers the unbound-variable trap and either prints the error or aborts the read loop.

Bash strict mode can conflict with prompt themes that reference optional
variables. The failure is not specific to Fedora. `set -u` is unsuitable for
this interactive shell configuration.

**Reproduction:**

```bash
# In an interactive root shell on Fedora 43:
set -u
ls /tmp
# -bash: PROMPT_START: unbound variable
```

**Fix or workaround:**
Run strict-mode validation blocks in a clean non-interactive shell. Either of these works:

```bash
# Heredoc form (preferred for runbook snippets):
sudo bash <<'EOF'
set -euo pipefail
# ... your validation block ...
EOF

# Script form (preferred for repeatable workflows):
sudo bash /path/to/validation_script.sh
```

The non-interactive shell has no prompt to expand and no `bash-color-prompt.sh` to source, so `set -u` behaves as intended.

The bundle's three convenience scripts under `scripts/` are deliberately *not* run with `set -u` in their preamble for this reason: they are documented as runnable both as files and via paste-into-interactive-shell, and the latter case would break under strict mode. The validation hook itself (`hooks/80-tpm2-sign.install`) keeps strict mode because it runs as a non-interactive child of `kernel-install`, not interactively.

**Cross-reference:**
- `06B`: Step 19 (post-reboot PCR regeneration uses the heredoc form for exactly this reason)
- `scripts/README.md`: documents why operational scripts omit `set -u`

---

### F.5: Interactive prompts inside heredoc-wrapped strict-mode blocks read from the consumed heredoc, not from the terminal

**Verdict:** CONTEXT-ONLY. A scripting quirk; the fix is a single stdin redirect.

**Symptom:**
A heredoc-wrapped validation block that calls `cryptsetup open --test-passphrase` or any tool that prompts for input fails immediately with what looks like an authentication error:

```
🔐 Please enter current passphrase for disk /dev/sda3:
No key available with this passphrase.
```

No prompt was actually presented to the operator. The script behaves as if an
empty passphrase was supplied.

**Root cause:**
`bash <<'EOF' ... EOF` redirects bash's stdin to the heredoc content. The heredoc is fully consumed during script parsing. When a child process (like `cryptsetup`) then reads from stdin during execution, it gets EOF or empty input. `cryptsetup` interprets this as a (empty) passphrase attempt and reports the standard "no key matches" failure.

`set -e` does not abort the script because the failing command was the left operand of `&&`, which is exempted from strict-mode failure per the bash manual.

Unlike F.4, which concerns prompt rendering under `set -u`, this failure affects
input read by a child process.

**Reproduction:**

```bash
bash <<'EOF'
set -euo pipefail
cryptsetup open --test-passphrase /dev/sda3 && echo PASSPHRASE_OK
EOF
# Output: "No key available with this passphrase."
# (No prompt presented to the operator)
```

**Fix or workaround:**
Redirect the prompting tool's stdin from `/dev/tty` explicitly. This forces the read to come from the controlling terminal regardless of what the heredoc did to bash's stdin:

```bash
bash <<'EOF'
set -euo pipefail
cryptsetup open --test-passphrase /dev/sda3 </dev/tty && echo PASSPHRASE_OK
EOF
# Operator is prompted for passphrase as expected.
```

`systemd-cryptenroll` is more forgiving because it uses systemd's `ask-password` mechanism which opens `/dev/tty` directly, but `</dev/tty` is the safe, explicit, portable pattern for any interactive command inside a heredoc.

**Cross-reference:**
- `06B`: Step 20 (pre-flight passphrase test), Step 22 (TPM2 enrollment), Step 23 (post-enrollment passphrase test)
- Evidence: 2026-05-21 Module 3 execution session

---

## G: LUKS2 and systemd-cryptenroll

### G.1: `systemd-cryptenroll --tpm2-public-key-pcrs=` must exactly match the PCR list signed in `.pcrsig`; `--tpm2-signature` validates against current PCR state, not the signed phase

**Verdict:** FORBIDDEN combinations described below; CONDITIONAL on operator selecting the working policy shape.

**Symptom:**
Two distinct failure modes, both producing the same opaque error:

```
Failed to unseal secret using TPM2: No such device or address
```

The TPM device itself is reachable and functional; the error reflects a policy-session unseal failure that gets mapped to `ENXIO`.

**Failure mode A: PCR list mismatch:**
Calling `systemd-cryptenroll` with `--tpm2-public-key-pcrs=7+11` when the hook signs only PCR 11 fails. The policy session reconstructed by `systemd-cryptenroll` looks up signature entries by exact PCR-list match. The embedded `.pcrsig` has one entry with `pcrs=[11]`; demanding `[7, 11]` finds no matching entry.

**Failure mode B: phase-chain mismatch on dry-run validation:**
Even with the correct PCR list `--tpm2-public-key-pcrs=11`, passing `--tpm2-signature=<json>` fails. `systemd-cryptenroll --tpm2-signature` performs a dry-run validation: it constructs the policy from the signature, seals a test secret to that policy, then attempts to unseal against current TPM state. The hook signs the **enter-initrd** phase value of PCR 11. The current runtime PCR 11 is the **post-`ready`** value. The two values differ; the dry-run unseal fails.

**Root cause:**
Two distinct architectural truths:

1. `--tpm2-public-key-pcrs=` defines which PCRs must be covered by a *signed policy entry*. PCRs not in this list are not consulted via signed policy. The signature in `.pcrsig` must have an entry whose `pcrs` field exactly matches this list.

2. `--tpm2-signature` is an enrollment-time validator. It assumes the signature attests to the **current** TPM state, which is only true if the signature was generated for the post-`ready` phase. Hook-signed `.pcrsig` is generated for `--phases=enter-initrd`, by design: that's the phase at which `systemd-cryptsetup` requests unlock at boot.

The combination "signed policy for the PCRs that change + static binding for the PCRs that don't, with no enrollment-time signature validation" reflects the architecture correctly:

- PCR 7 statically bound: represents stable Secure Boot policy state. Static binding pins the TPM to the current SB policy value. Re-enrollment is only needed on SB key rotation (Module 4).
- PCR 11 signed-policy bound: changes when measured UKI content changes. Signed binding lets the policy key authorise future PCR 11 values (forward sealing).

**Reproduction:**

```bash
# Failure A (PCR list mismatch):
systemd-cryptenroll /dev/sda3 \
  --tpm2-device=auto \
  --tpm2-public-key=/etc/systemd/tpm2-pcr-public-key.pem \
  --tpm2-public-key-pcrs=7+11
# Fails: "Failed to unseal secret using TPM2: No such device or address"
# (because .pcrsig only has pcrs=[11])

# Failure B (phase-chain mismatch on dry-run):
systemd-cryptenroll /dev/sda3 \
  --tpm2-device=auto \
  --tpm2-public-key=/etc/systemd/tpm2-pcr-public-key.pem \
  --tpm2-public-key-pcrs=11 \
  --tpm2-signature=/run/tboot-pcrsig-module3/tpm2-pcr-signature.json
# Also fails with the same error string
# (because current PCR 11 is post-ready, signature attests to enter-initrd)
```

**Fix or workaround:**
Use split policy and omit `--tpm2-signature`:

```bash
systemd-cryptenroll /dev/sda3 \
  --tpm2-device=auto \
  --tpm2-pcrs=7 \
  --tpm2-public-key=/etc/systemd/tpm2-pcr-public-key.pem \
  --tpm2-public-key-pcrs=11
```

The two PCRs have different change rates and trust boundaries. PCR 7 is stable,
while PCR 11 changes when the measured boot content changes. PCR 7 therefore
uses static binding and PCR 11 uses signed authorization. Omitting
`--tpm2-signature` also omits enrollment-time validation, so the reboot test is
the end-to-end check. Keyslot 0 remains available as the passphrase recovery
path throughout.

**Operational consequences:**
The Module 3 design doc (`03_Disk_Encryption_and_Policy_Enforcement.md` §1.2) originally showed `--tpm2-public-key-pcrs=7+11 --tpm2-signature=…` as the example. That example is incompatible with the hook's `--phases=enter-initrd` signing and has been updated to the split-policy form.

`systemd-cryptenroll(8)` documents `--tpm2-signature` as optional. Omitting it is supported upstream behaviour; the keyslot policy is still correctly constructed from `--tpm2-public-key` alone.

**Cross-reference:**
- `06B`: Step 22 (enrollment with split policy)
- `03_Disk_Encryption_and_Policy_Enforcement.md` §1.2 (design doc, updated 2026-05-21)
- systemd documentation: `systemd-cryptenroll(8)`, `crypttab(5)`
- Evidence: 2026-05-21 Module 3 execution session

---

### G.2: `kernel-install add` does not refresh `/boot/initramfs-${KVER}.img` under UKI layout when the kernel is already installed

**Verdict:** CONDITIONAL. The behaviour is upstream-correct; operators must understand the propagation path when modifying initramfs-content inputs.

**Symptom:**
After editing `/etc/crypttab` (or any other initramfs-content input like `/etc/dracut.conf.d/*.conf`) and running `kernel-install add ${KVER}`, the new UKI on the ESP has a fresh sha256, the hook journal is clean, all structural checks pass, but the embedded initramfs inside the UKI still contains the **old** content.

Inspecting the embedded initramfs reveals the stale crypttab:

```bash
# After editing /etc/crypttab to add tpm2-device=auto, then running kernel-install add:
objcopy --dump-section .initrd=initrd /boot/efi/EFI/Linux/${MID}-${KVER}.efi /dev/null
lsinitrd initrd -f /etc/crypttab
# Output: stale crypttab without tpm2-device=auto, despite live /etc/crypttab having it
```

**Root cause:**
Under `KERNEL_INSTALL_LAYOUT=uki`, `kernel-install add` invokes the drop-ins in `/etc/kernel/install.d/` and `/usr/lib/kernel/install.d/` in lexical order. The `50-dracut.install` drop-in (which regenerates `/boot/initramfs-${KVER}.img`) fires on actual kernel package transactions but **does not** fire on explicit `kernel-install add` of an already-installed kernel. `60-ukify.install` generates its own ephemeral initramfs for its base UKI output, but the validated 80-tpm2-sign hook discards that output and rebuilds the UKI via:

```bash
ukify build \
  --linux=/lib/modules/${KVER}/vmlinuz \
  --initrd=/boot/initramfs-${KVER}.img    ← here
  ...
```

The hook reads `/boot/initramfs-${KVER}.img` directly. If that file wasn't refreshed before the hook ran, the UKI inherits its stale content.

Verified directly: in the failed Module 3 sequence, the on-disk initramfs timestamp was 13 days before the `/etc/crypttab` edit. `kernel-install add` made no attempt to refresh it; the hook used it verbatim.

**Reproduction:**

```bash
# 1. Edit /etc/crypttab to add tpm2-device=auto
sed -i '...' /etc/crypttab

# 2. Run kernel-install add (without refreshing /boot/initramfs first)
kernel-install add "$(uname -r)" "/lib/modules/$(uname -r)/vmlinuz"

# 3. Verify embedded crypttab: option is missing
WORK=$(mktemp -d)
cp "/boot/efi/EFI/Linux/$(cat /etc/machine-id)-$(uname -r).efi" "$WORK/uki.efi"
objcopy --dump-section .initrd="$WORK/initrd" "$WORK/uki.efi" "$WORK/throwaway.efi"
lsinitrd "$WORK/initrd" -f /etc/crypttab
# Output: no tpm2-device=auto
```

Confirmation that dracut itself respects the option (i.e. the problem is upstream of dracut, in the kernel-install flow):

```bash
dracut --force "/boot/initramfs-$(uname -r).img" "$(uname -r)"
lsinitrd "/boot/initramfs-$(uname -r).img" -f /etc/crypttab
# Output: contains tpm2-device=auto
```

**Fix or workaround:**
When initramfs-content inputs change outside a kernel package transaction (typical for Module 3 setup, dracut module configuration, or any operator-driven crypttab edit), the propagation requires three explicit steps:

```bash
# Step 1: refresh /boot/initramfs-${KVER}.img from current host state
dracut --force "/boot/initramfs-${KVER}.img" "${KVER}"

# Step 2: verify the new content is present BEFORE the UKI rebuild
lsinitrd "/boot/initramfs-${KVER}.img" -f /etc/crypttab | grep <expected-option> \
  || { echo "FAIL: dracut did not propagate the change"; exit 1; }

# Step 3: rebuild the UKI through the standard kernel-install path
kernel-install add "${KVER}" "/lib/modules/${KVER}/vmlinuz"

# Step 4: verify the embedded UKI initrd matches expectations
WORK=$(mktemp -d -p /run)
cp "/boot/efi/EFI/Linux/$(cat /etc/machine-id)-${KVER}.efi" "$WORK/uki.efi"
objcopy --dump-section .initrd="$WORK/initrd" "$WORK/uki.efi" "$WORK/throwaway.efi"
lsinitrd "$WORK/initrd" -f /etc/crypttab | grep <expected-option>
```

`06B` Step 24 codifies this three-stage pattern.

**Operational consequences:**
Any future operation that affects initramfs content must follow the dracut-refresh-then-kernel-install pattern. This includes:

- `/etc/crypttab` modifications (Module 3, Module 4 lifecycle work)
- `/etc/dracut.conf.d/*.conf` changes (e.g. adding modules, changing compression)
- `/etc/cmdline.d/*.conf` changes (some dracut modules read these)
- Adding or removing kernel modules from `/etc/modules-load.d/`

The Block B.3 validation pattern (`scripts/validate_dnf_kernel_reinstall.sh`) is unaffected because it operates on `dnf reinstall kernel-core`, which IS a kernel package transaction and DOES trigger `50-dracut.install`.

**Cross-reference:**
- `06B`: Step 24 (validated three-stage propagation pattern); Steps 29-30 automate the dracut refresh → kernel-install add pattern for all installed kernels under the production helper `tboot-dnf-helper`.
- Evidence: 2026-05-21 Module 3 execution session; initramfs timestamp drift caught by post-build `lsinitrd` verification
- systemd documentation: `kernel-install(8)`, dracut module structure

---

## H: DNF5 plugin layer and trigger mechanics

### H.1: dnf5 on Fedora 43 exposes no runtime verbosity flag; plugin debugging requires `/var/log/dnf5.log` inspection

**Verdict:** CONDITIONAL on knowing the actual observability surface; legacy dnf4 operational concepts do not apply.

**Symptom:**
Attempting to obtain inline plugin-load or transaction-resolution debug output from dnf5 via familiar dnf4 flags returns an error:

```
$ dnf --debuglevel=8 reinstall --assumeno less
Unknown argument "--debuglevel=8" for command "dnf5". Add "--help" for more information about the arguments.

$ dnf -v repoquery --installed less
Unknown argument "-v" for command "dnf5". Add "--help" for more information about the arguments.
```

Surveying the help text for verbosity-related flags returns only two options, neither suitable for live plugin debugging:

```
$ dnf --help | grep -iE 'verbose|debug|log|trace|quiet'
  -q, --quiet                            In combination with a non-interactive command, shows just the relevant content. ...
  --debugsolver                          Dump detailed solving results into files
```

`--debugsolver` is depsolver-only and writes to files, not relevant for plugin-load tracing.

**Root cause:**
dnf5 deliberately collapsed dnf4's verbosity surface. The runtime stdout/stderr stream is intentionally terse. Operational debug output is written instead to a persistent log file at `/var/log/dnf5.log`, which is rotated automatically. Plugin load events (`DEBUG Loading plugin library file=…`, `INFO Loaded libdnf plugin "..."`) appear in this log on every transaction-shaped invocation; they do not appear on stdout/stderr at all.

This is a behaviour change, not a bug: dnf5 considers the structured log file the supported observability mechanism. Project automation that depended on parsing dnf debug output via flags will not work; automation must read `/var/log/dnf5.log` delta windows instead.

**Reproduction:**

```bash
# Confirm dnf5 has no usable runtime verbosity flag
dnf --help 2>&1 | grep -iE 'verbose|debug|log|trace|quiet'
# Output: only --quiet and --debugsolver

# Confirm dnf5 maintains a persistent log
ls -la /var/log/dnf5*
# Output: /var/log/dnf5.log plus rotated /var/log/dnf5.log.{1..N}

# Demonstrate that read-only repoquery does not write to dnf5.log
PRE=$(wc -l </var/log/dnf5.log)
dnf repoquery --installed less >/dev/null 2>&1
POST=$(wc -l </var/log/dnf5.log)
echo "delta: $((POST - PRE)) lines"     # 0 lines: repoquery is too quiet

# Demonstrate that transaction-shaped --assumeno DOES write to dnf5.log.
# Use a harmless package that is NOT currently installed; --assumeno on an
# already-installed package gets weird resolver semantics. See H.2 for the
# four-gate candidate-selection probe used to pick one safely.
PRE=$(wc -l </var/log/dnf5.log)
dnf install --assumeno figlet >/dev/null 2>&1 || true   # substitute any harmless not-installed package
POST=$(wc -l </var/log/dnf5.log)
echo "delta: $((POST - PRE)) lines"     # several hundred lines including plugin loads
```

**Fix or workaround:**
For any project automation that needs to verify plugin-layer behaviour, the canonical pattern is:

```bash
DNF_LOG=/var/log/dnf5.log
PRE_LINES=$(wc -l <"$DNF_LOG")

# Trigger a transaction-shaped invocation that does not commit.
# <some-package> must be a harmless package that is NOT currently installed;
# selecting it safely is the four-gate probe in finding H.2.
dnf install --assumeno <some-package> >/dev/null 2>&1 || true

POST_LINES=$(wc -l <"$DNF_LOG")
ADDED=$((POST_LINES - PRE_LINES))

# Inspect the delta for the evidence you need
tail -n "$ADDED" "$DNF_LOG" | grep -E 'Loaded libdnf plugin|<plugin-specific-marker>'
```

`dnf install --assumeno` is the lowest-overhead transaction-shaped invocation: it loads the full plugin chain and resolves the transaction but never commits.

**Operational consequences:**
- `dnf reinstall --assumeno` exits non-zero on the abort path. Any wrapper script must use `|| true` or equivalent to keep `set -e` from aborting on the benign abort.
- The dnf5.log file rotates aggressively (every 1–2 days based on the reference rebuild). Historical evidence may require `zgrep` across `dnf5.log.{1..N}.gz` as well as the current file.
- Tooling that depends on dnf debug output (legacy CI scripts, third-party wrappers) must be re-tooled against the log-file pattern.

**Cross-reference:**
- `06B`: Step 27 (plugin-load confirmation via dnf5.log delta, documented as part of B.2.1)
- Evidence: 2026-05-21 B.2.1 session, dnf5-5.2.18.0-3.fc43, libdnf5-5.2.18.0-3.fc43
- DNF5 documentation: [DNF5 Workflow](https://dnf5.readthedocs.io/en/latest/dnf5_workflow.html), [DNF5 configuration reference](https://dnf5.readthedocs.io/en/latest/dnf5.conf.5.html)

---

### H.2: `dnf reinstall <pkg>` requires the installed NEVRA to be present in an enabled repo; version drift makes reinstall an unreliable test driver

**Verdict:** CONDITIONAL on candidate-selection method; prefer fresh `dnf install` of a known-not-installed package.

**Symptom:**
Running `dnf reinstall` against a package that is clearly installed produces a refusal:

```
$ dnf reinstall --assumeno less
Updating and loading repositories:
Repositories loaded.
Failed to resolve the transaction:
Installed packages for argument 'less' are not available in repositories in the same version,
available versions: less-679-2.fc43.x86_64, less-692-6.fc43.x86_64, cannot reinstall.
```

This happens even though `less` is installed (`rpm -q less` succeeds) and `less` is available in repos (`dnf repoquery less` returns hits). The specific installed NEVRA is no longer in any enabled repo because the package was once shipped through `updates-testing` (or a now-superseded `updates` revision) and has since been replaced upstream.

**Root cause:**
`dnf reinstall` requires the installed NEVRA: exact name-epoch-version-release-architecture match: to exist in an enabled repository. The operation downloads the matching `.rpm` and reinstalls atop the existing installation. If no enabled repo offers the exact installed NEVRA, the resolver cannot construct a valid transaction and refuses.

Long-lived Fedora 43 installations accumulate packages whose installed version no longer matches any enabled-repo offering, especially for fast-moving utilities. The failure is benign but breaks automation that assumes `reinstall` is always available.

**Reproduction:**

```bash
# Inspect installed NEVRA vs available NEVRAs
INSTALLED=$(rpm -q --qf '%{name}-%{evr}.%{arch}' less)
AVAILABLE=$(dnf repoquery --quiet --qf '%{name}-%{evr}.%{arch}' less)
echo "installed: $INSTALLED"
echo "available:"
echo "$AVAILABLE" | sed 's/^/  /'

# If $INSTALLED does not appear in $AVAILABLE, reinstall will fail
echo "$AVAILABLE" | grep -qFx "$INSTALLED" \
  && echo "reinstallable" \
  || echo "NOT reinstallable (drift)"
```

**Fix or workaround:**
For automated test drivers (such as the B.2.1 trigger validation in `06B` Step 28), do not rely on `dnf reinstall`. Instead select a package that is **not currently installed** and use `dnf install`. The transaction direction (`in`) is identical from the action-rule perspective; the trigger fires the same way.

The candidate must pass four independent gates before it is used as a trigger test driver. Skipping any of these reintroduces risk: gate 3 protects against a candidate that quietly drops files into `/boot` or under `/etc/dracut`; gate 4 protects against a candidate whose dependency closure pulls in `systemd-*` or `kernel-*` packages.

1. **Not currently installed.** `rpm -q <pkg>` returns non-zero.
2. **Available in enabled repos.** `dnf repoquery --quiet <pkg>` returns at least one NEVRA.
3. **No sensitive file paths.** `dnf repoquery -l <pkg>` lists nothing under `/boot`, `/etc/kernel`, `/etc/dracut`, `/usr/lib/dracut`, `/usr/lib/systemd`, `/usr/lib/modules`, `/usr/lib/firmware`, `/usr/lib/dnf`, or `/etc/dnf`.
4. **No sensitive transaction dependencies.** `dnf install --assumeno <pkg>` output contains no `systemd*`, `kernel*`, `dracut*`, `cryptsetup*`, `shim*`, `grub2-*`, or `systemd-boot*` packages anywhere in the `Installing:` / `Installing dependencies:` sections. Scan with a word-boundary regex (`\b(systemd|kernel|dracut|cryptsetup|luks|tpm2|shim|grub|boot)([-_].*)?\b`) to avoid the `inbound`-keyword false positive documented in finding H.3.

Candidate-selection probe (read-only; includes all four gates):

```bash
CANDIDATES=(bc figlet cowsay sl pv ripgrep fd-find tmux htop ncdu jq)
for pkg in "${CANDIDATES[@]}"; do
  # Gate 1: not installed
  rpm -q "$pkg" >/dev/null 2>&1 && continue
  # Gate 2: available in repos
  dnf repoquery --quiet "$pkg" 2>/dev/null | grep -q . || continue
  # Gate 3: no sensitive file paths
  if dnf repoquery -l "$pkg" 2>/dev/null \
       | grep -qE '^(/boot|/etc/kernel|/etc/dracut|/usr/lib/dracut|/usr/lib/systemd|/usr/lib/modules|/usr/lib/firmware|/usr/lib/dnf|/etc/dnf)'; then
    continue
  fi
  # Gate 4: no sensitive deps in proposed transaction
  if dnf install --assumeno "$pkg" 2>&1 \
       | grep -qE '\b(systemd|kernel|dracut|cryptsetup|luks|tpm2|shim|grub|boot)([-_].*)?\b'; then
    continue
  fi
  echo "candidate: $pkg"
  break
done
```

On the reference rebuild this probe selected `figlet`. Other rebuilds may select a different package depending on what is already installed and what is currently in repos.

**Operational consequences:**
The validated B.2.1 trigger package on the reference rebuild was `figlet`, not `less`, for this reason. Future B.2.x test drivers should reach for fresh-install candidates rather than reinstall.

In production B.2.3+, the action rule fires on any inbound transaction matching the package filter regardless of operator command: `dnf install nvidia-driver`, `dnf upgrade dracut`, `dnf reinstall systemd-udev`, etc. The reinstall fragility affects only the test driver, not the production trigger.

**Cross-reference:**
- `06B`: Step 28 (trigger-package selection probe and validated trigger pattern using `dnf install -y figlet`)
- Evidence: 2026-05-21 B.2.1 session, `less` reinstall failure during initial trigger-package selection
- DNF5 documentation: `dnf5 reinstall` semantics

---

### H.3: Substring `grep -Ei 'boot'` against dnf5 transaction output produces false positives on the `inbound` header keyword

**Verdict:** CONDITIONAL on use of word-boundary regex.

**Symptom:**
A sensitive-package safety gate that scans dnf transaction output for boot-related keywords fires on a transaction that contains no boot-related packages:

```bash
$ dnf install --assumeno figlet | tee /tmp/preview.txt
... transaction summary showing 1 package, 0 dependencies, figlet only ...
Total size of inbound packages is 138 KiB.
... rest of summary ...

$ grep -Ei 'systemd|kernel|dracut|cryptsetup|luks|tpm2|shim|grub|boot' /tmp/preview.txt
Total size of inbound packages is 138 KiB.
```

The grep hits because `inbound` contains the substring `boot`. The transaction is in fact entirely safe: one package, no dependencies, nothing boot-related, but the gate fires anyway and aborts the surrounding script.

**Root cause:**
dnf5 prints standardised summary lines including `Total size of inbound packages is N KiB.` on every transaction-shaped invocation. The word `inbound` contains the four-character substring `boot`. A `grep -Ei` pattern with the bare alternation `…|boot` matches the substring anywhere it appears in the line, including inside other words. Other short alternation terms in the same pattern have similar exposure (`grub` inside `grubbing`, etc.), but `boot` inside `inbound` is the one that fires reliably because the header line is always present.

This is not specific to dnf5: any tool that prints prose summaries containing `inbound`, `reboot`, `bootstrap`, etc. will trigger the same false positive when scanned with an unanchored substring pattern.

**Reproduction:**

```bash
# Generate a clean transaction preview
dnf install --assumeno figlet >/tmp/preview.txt 2>&1 || true

# Unsafe pattern: matches 'boot' as substring
grep -Ei 'systemd|kernel|dracut|cryptsetup|luks|tpm2|shim|grub|boot' /tmp/preview.txt
# Hits on "Total size of inbound packages is ..."

# Safe pattern: word boundary plus optional name-character suffix
grep -Ei '\b(systemd|kernel|dracut|cryptsetup|luks|tpm2|shim|grub|boot)([-_].*)?\b' /tmp/preview.txt
# No false positive; will still match real package names like 'systemd-libs',
# 'kernel-core', 'systemd-boot', 'boot-tools', etc.
```

**Fix or workaround:**
Always use word-boundary regex when matching package-name-shaped tokens against prose-bearing output. The validated pattern for the sensitive-package gate is:

```
\b(systemd|kernel|dracut|cryptsetup|luks|tpm2|shim|grub|boot)([-_].*)?\b
```

The trailing `([-_].*)?\b` allows matches like `systemd-libs`, `kernel-modules-core`, `boot-tools`, while requiring the term to begin at a word boundary. This rejects `inbound`, `rebooting`, `kernels` (plural prose), and similar substring traps.

A stricter alternative is to scope the grep only to lines beginning with package-shaped output (i.e. lines under `Installing:` or `Upgrading:` headers), but that adds parser complexity for marginal gain.

**Operational consequences:**
The B.2.1 validation session hit this false positive once (figlet transaction flagged on `inbound`). The runbook (`06B` Step 28) uses the word-boundary form from the start, so this finding documents the trap rather than codifying it in the runbook itself.

Any future sensitive-package gate in this project: Module 4 cleanup procedures, B.4 systemd-boot signing automation, attack-scenario test rigs: must use the word-boundary form.

**Cross-reference:**
- `06B`: Step 28 (sensitive-path grep, uses word-boundary form by design)
- Evidence: 2026-05-21 B.2.1 session, figlet trigger preview false positive
- General regex principle: GNU grep `-w` flag or `\b` anchors

---

### H.4: External audit-harness token-boundary regex must match the in-script self-scan; do not patch staged sources for harness false positives

**Verdict:** CONDITIONAL on harness/self-scan regex parity.

**Symptom:**
A read-back assertion that scans a staged decider source for forbidden trust-boundary tokens fires on a manifest category label that contains the token as a substring:

```text
532:     local c="08-cryptsetup-tooling"
FAIL: forbidden token 'cryptsetup' in non-comment, non-nolint line
FAIL: 1 read-back violation(s)
```

The line in question is a string-literal category tag for the manifest emitter, not a function call. The in-script `_self_test_trust_boundary` self-scan, which runs at production preflight, does **not** flag the same line. Two scanners that are supposed to enforce the same invariant disagree.

**Root cause:**
The two scanners use different regex forms for "token bounded by non-identifier characters":

- **In-script self-scan (correct, Deviation D form):**
  ```awk
  BEGIN { pat = "(^|[^A-Za-z0-9_-])" t "($|[^A-Za-z0-9_-])" }
  $0 ~ pat { ... }
  ```
  The `-` character is **inside** the identifier character class. `08-cryptsetup-tooling` is treated as one hyphenated identifier; `cryptsetup` inside it is **not** bounded by a non-identifier character, so the pattern does not match.

- **External audit harness (laxer, buggy form previously used in Gate 2 Step 6):**
  ```awk
  $0 ~ ("\\<" t "\\>") { ... }
  ```
  GNU awk's POSIX word-boundary anchors `\<` / `\>` use the word-character class `[A-Za-z0-9_]`. The `-` character is **outside** that class and therefore acts as a word boundary. `08-cryptsetup-tooling` matches `\<cryptsetup\>` between the two surrounding hyphens. False positive.

Deviation D was introduced specifically because POSIX awk's word-boundary anchors treat `-` inconsistently across implementations and would let `efi-updatevar-fake` slip through (or, symmetrically, would falsely flag `08-cryptsetup-tooling`). The decider's in-script self-scan inherited Deviation D from the start. The external audit harness was authored later and did not inherit it.

**Reproduction:**

```bash
cat > /tmp/h4.txt <<'EOF'
local c="08-cryptsetup-tooling"
EOF

# Laxer form (false positive)
awk -v t=cryptsetup '
    $0 ~ ("\\<" t "\\>") { print NR": "$0 }
' /tmp/h4.txt
# Output: 1:     local c="08-cryptsetup-tooling"

# Deviation D form (correct: no match)
awk -v t=cryptsetup '
    BEGIN { pat = "(^|[^A-Za-z0-9_-])" t "($|[^A-Za-z0-9_-])" }
    $0 ~ pat { print NR": "$0 }
' /tmp/h4.txt
# Output: (empty)
```

**Fix or workaround:**
Harmonise every external audit harness with the in-script Deviation D regex. The canonical form for token-with-hyphen-aware boundary matching is:

```awk
BEGIN { pat = "(^|[^A-Za-z0-9_-])" t "($|[^A-Za-z0-9_-])" }
$0 ~ pat { ... }
```

Do **not** patch the staged source to work around the harness. Specifically, do not:

- Append `# nolint:trust-boundary` to category-label lines to silence the false positive. The label is legitimate code; the nolint marker is for genuine cross-boundary references (e.g. the `cryptsetup` binary path hashed in the manifest emitter), not for string literals that happen to contain a token substring.
- Rename manifest categories to avoid the forbidden tokens. The 14-category structure is part of the Gate 1 design lock.
- Edit the staged file in place. The staged file's sha256 is the canonical Gate 2 artifact. Mutating it after Step 3 records the sha256 silently invalidates the recorded identity.

When the external harness and the in-script self-scan disagree, use the
**in-script self-scan** as the reference because it runs during production
preflight. Update the external harness to use the same expression.

Deviation D (the authoritative regex), rendered unambiguously:

```
(^|[^A-Za-z0-9_-])TOKEN($|[^A-Za-z0-9_-])
```

Replace `TOKEN` with each forbidden token (`ukify`, `sbsign`, `sbverify`, `systemd-measure`, `systemd-cryptenroll`, `efi-updatevar`, `cryptsetup`). The hyphen is inside the identifier-class set, not the boundary set.

> [!warning] Do not patch staged sources to silence audit-harness false positives
> If an external audit step fires a violation that the in-script self-scan does not, the conclusion is **the harness is wrong**, not the source. Mutating the staged source invalidates the Step-3-recorded sha256 and bakes a workaround for a harness bug into the production decider. Fix the harness instead.

**Operational consequences:**
B.2.3 Gate 2 Step 6 originally used the `\<TOKEN\>` form and was therefore subject to this false positive. The first Step 6 attempt failed at line 532 (`08-cryptsetup-tooling`). The corrected Step 6, and every subsequent external read-back harness in this project (Gate 4 staged `.actions` rule review, future Gate 6 negative-validation read-backs, B.2.4 live-transaction read-backs): must use the Deviation D form.

One proposed workaround was to patch the staged decider source with a `python3`
`replace()` call that appended `# nolint:trust-boundary` to the category-label
line. That workaround was rejected for the reasons listed above. The Gate 2
staged source sha256
`35ad8733f190483f1bd6d071d2aa7cb8b1549286f9040086182443c584473cdd`
remained unchanged between the failing and passing Step 6 runs.

**Cross-reference:**
- `06B`: Step 31 (B.2.3 Gate 2: staged decider read-back validation; corrected Step 6 uses Deviation D regex)
- `00_Current_Project_State.md`: B.2.3 trust-boundary self-scan regex (Deviation D) recorded in architectural invariants
- Evidence: 2026-05-24 B.2.3 Gate 2 session, first Step 6 attempt
- Related: B.2.1 word-boundary regex finding H.3 (same family of trap: substring matching against tokens with non-`\w` characters)

---

## I: Gate 3 environment and harness findings

Findings discovered during the 2026-05-24 B.2.3 Gate 3 (sub-gates 3.0–3.8) execution sessions. These are predominantly environmental (Fedora 43 / Proxmox / systemd / journald observed behaviour) and harness-engineering (validation-script-side) lessons. None of them are decider-source bugs; the decider behaved per its source contract throughout Gate 3.

### I.1: `qm listsnapshot` output prefixes snapshot names with tree glyphs; column-1 awk match without prefix stripping is unreliable

**Verdict:** CONTEXT-ONLY (harness-engineering trap).

**Symptom:**
A Proxmox-host harness that attempts to confirm the rollback-anchor snapshot exists with the apparently-strict pattern `qm listsnapshot 500 | awk '$1=="module5-b2-2-real-run-validated"'` returns a false negative. The snapshot is genuinely present (visible in the Proxmox web UI and in `qm listsnapshot` output), but the awk match fails.

**Root cause:**
`qm listsnapshot` renders the snapshot tree with tree-drawing prefixes such as `` `-> `` (backtick + arrow + space) and `-> ` (arrow + space) at the start of each non-root snapshot line. These prefixes are part of `$1` after awk's default whitespace splitting on tokens that include the backtick or the arrow. A strict `$1=="snapshot_name"` test sees `\`->` or similar prefixes glued to the name and never matches the bare name. Additionally on Proxmox 8.x the PVE `GuestHelpers.pm` Perl module emits `Wide character in printf at /usr/share/perl5/PVE/GuestHelpers.pm line 176.` warnings on STDOUT/STDERR because the snapshot-description rendering uses UTF-8 box-drawing characters without an explicit UTF-8 STDOUT layer. The warnings are cosmetic, not a parse failure, but they can confuse simpler grep-based parsers.

**Reproduction:**
```bash
qm listsnapshot 500 | head
# observe lines like:
#  `-> module5-b2-2-real-run-validated      ...
#  `-> module5-b2-2-helper-installed        ...
```
A naive `awk '$1=="module5-b2-2-real-run-validated"'` does not match.

**Fix or workaround:**
Strip leading whitespace and tree-prefix glyphs before matching:
```bash
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
    '
```

The Perl wide-character warnings can be ignored; they do not affect the awk consumer.

**Cross-reference:**
- `06B`: Step 32 Gate 3.0 (rollback-anchor check)
- Evidence: 2026-05-24 Gate 3.0 first attempt and `qm listsnapshot 500` output

---

### I.2: `bootctl get-default` returned empty on Fedora 43 VM despite `bootctl status` reporting the correct default; parse `bootctl status` instead

**Verdict:** CONTEXT-ONLY (environment quirk).

**Symptom:**
A Gate 3.0 sanity check that asserts the systemd-boot default entry equals the expected hook-generated UKI via:
```bash
default_entry="$(bootctl get-default 2>/dev/null | tr -d '[:space:]' || true)"
```
fails because `bootctl get-default` returns an empty string. At the same time, `bootctl status` (run with no arguments) correctly reports the expected UKI in both `Default Entry:` and `Current Entry:` fields.

**Root cause:**
On the validated Fedora 43 VM (`tboot.local.internal`, VMID 500), `bootctl get-default` returns empty even though a default entry is set and is correctly reported by `bootctl status`. The exact mechanism is not fully diagnosed; the most likely cause is the `LoaderEntryDefault` EFI variable being read differently by the `get-default` and `status` code paths in this systemd build, or the entry being set via `bootctl set-default` having been written into a location that `get-default` does not consult on this layout. The behaviour is reproducible and stable, not transient.

**Reproduction:**
```bash
bootctl get-default   # returns empty
bootctl status        # reports correct Default Entry, and Current Entry:
```

**Fix or workaround:**
Treat `bootctl get-default` as non-authoritative on this setup. Parse `bootctl status` instead, with awk anchored to the labelled rows:
```bash
HOOK_ID="0123456789abcdef0123456789abcdef-6.19.14-200.fc43.x86_64.efi"
bootctl_status="$(bootctl status 2>/dev/null)"
default_entry="$(awk -F': ' '/^[[:space:]]*Default Entry:/ {print $2; exit}' <<< "$bootctl_status" | xargs)"
current_entry="$(awk -F': ' '/^[[:space:]]*Current Entry:/ {print $2; exit}' <<< "$bootctl_status" | xargs)"
[[ "$default_entry" == "$HOOK_ID" ]] || { echo "FAIL: default '${default_entry}' != '${HOOK_ID}'"; exit 1; }
[[ "$current_entry" == "$HOOK_ID" ]] || { echo "FAIL: current '${current_entry}' != '${HOOK_ID}'"; exit 1; }
```

Both `Default Entry:` and `Current Entry:` should be asserted. Asserting `Default Entry:` alone leaves a gap where the default is correct but a one-shot or fallback selected a different boot loader entry.

**Cross-reference:**
- `06B`: Step 32 Gate 3.0 (bootctl default + current entry check)
- Evidence: 2026-05-24 Gate 3.0 first attempt vs corrected harness

---

### I.3: Fedora 43 `/usr/local/sbin -> bin` UsrMerge symlink: install-target safety checks must `readlink -f` before applying symlink-pure-rejecting tests

**Verdict:** CONDITIONAL (install-time safety pattern).

**Symptom:**
A Gate 3.2 atomic-publish harness that defensively rejects any symlink in the install path with `[[ -d "$DST_DIR" && ! -L "$DST_DIR" ]]` aborts with `FAIL: /usr/local/sbin missing or is a symlink` even though `/usr/local/sbin/tboot-dnf-helper` and `/usr/local/sbin/tboot-predict-pcr11` (installed during B.2.2) are visibly present and functional.

**Root cause:**
On Fedora 43, the UsrMerge layout makes `/usr/local/sbin` a relative symlink to `bin`:
```
lrwxrwxrwx. 1 root root 3 May  7 02:50 /usr/local/sbin -> bin
```
`/usr/local/bin` is the real directory; `/usr/local/sbin` is a system-managed legitimate symlink. RPM and `install -d` consolidate any sbin/bin distinction onto the single `/usr/local/bin` inode. B.2.2 silently relied on this resolution (the B.2.2 helper-sha verification still worked because `sha256sum` follows directory-component symlinks transparently); B.2.3 surfaced it because the install-target safety check was tightened to reject symlinks for tampering defense.

This is Fedora's documented UsrMerge layout, not a security event. The harness
assertion was too strict.

**Reproduction:**
```bash
ls -ld /usr/local /usr/local/sbin
# drwxr-xr-x. 11 root root 131 May  7 02:50 /usr/local
# lrwxrwxrwx.  1 root root   3 May  7 02:50 /usr/local/sbin -> bin
[[ -L /usr/local/sbin ]] && echo "is symlink"
# is symlink
```

**Fix or workaround:**
Resolve the canonical directory path via `readlink -f` before applying symlink-rejection tests; assert the resolved literal equals an expected value to detect future layout changes; perform the atomic publish (`mktemp` + `install -m mode -o owner -g group SRC TMP` + `mv -T TMP DST_REAL`) inside the resolved real directory so the rename is a same-filesystem atomic operation:

```bash
DST_DIR="/usr/local/sbin"
DST_REALDIR="$(readlink -f "$DST_DIR" 2>/dev/null || true)"
EXP_REALDIR="/usr/local/bin"  # Fedora 43 UsrMerge tail

[[ -n "$DST_REALDIR" ]] || { echo "FAIL: readlink -f ${DST_DIR} returned empty"; exit 1; }
[[ "$DST_REALDIR" == "$EXP_REALDIR" ]] \
  || { echo "FAIL: ${DST_DIR} resolves to ${DST_REALDIR} (expected ${EXP_REALDIR})"; exit 1; }
[[ -d "$DST_REALDIR" && ! -L "$DST_REALDIR" ]] \
  || { echo "FAIL: resolved ${DST_REALDIR} missing or is a symlink"; exit 1; }

# Verify type/mode/owner of the resolved directory:
dst_dir_stat="$(stat -c 'mode=%a owner=%U:%G type=%F' "$DST_REALDIR" 2>/dev/null || true)"
[[ "$dst_dir_stat" == "mode=755 owner=root:root type=directory" ]] \
  || { echo "FAIL: ${DST_REALDIR} unsafe metadata"; exit 1; }
```

After publish, verify through BOTH paths and assert they share the same inode:
```bash
ino_dst="$(stat -c '%i' "$DST")"; ino_real="$(stat -c '%i' "$DST_REAL")"
[[ "$ino_dst" == "$ino_real" ]] || { echo "FAIL: paths do not share inode"; exit 1; }
```

The canonical user-facing path remains `/usr/local/sbin/tboot-dnf-posttrans`. The resolved real path is `/usr/local/bin/tboot-dnf-posttrans`. Both refer to the same file via the directory symlink.

**Cross-reference:**
- `06B`: Step 32 Gate 3.2 (UsrMerge-aware atomic publish) and Gate 3.3 (inode identity through both paths)
- Evidence: 2026-05-24 Gate 3.2 first attempt failure + diagnostic re-listing

---

### I.4: `journalctl -t TAG --since X` prints `-- No entries --` on stdout when no entries match; tag-silence assertions must use `journalctl -q`

**Verdict:** CONTEXT-ONLY (harness-engineering trap).

**Symptom:**
A Gate 3.6 harness that asserts the helper journal tag (`tboot-dnf-helper`) remained silent during the `--prime` window with the pattern:
```bash
helper_journal="$(journalctl -t tboot-dnf-helper --since "$SINCE" --no-pager 2>/dev/null || true)"
[[ -z "$helper_journal" ]] || { echo "FAIL: helper journal fired"; exit 1; }
```
fires the FAIL branch even though the helper was genuinely not invoked. The captured `$helper_journal` contains the literal string `-- No entries --`.

**Root cause:**
`journalctl` emits an advisory line `-- No entries --` to stdout when the matching window has no entries. This is informational output, not an error. The `-z` test treats the non-empty string as "journal fired," producing a false positive.

**Reproduction:**
```bash
out="$(journalctl -t nonexistent-tag --since "5 minutes ago" --no-pager 2>/dev/null)"
echo "[$out]"
# [-- No entries --]
[[ -z "$out" ]] || echo "non-empty"
# non-empty
```

**Fix or workaround:**
Use `journalctl -q` to suppress the advisory line. The `-q` flag makes `journalctl` emit no output when there are no matches, restoring the `[[ -z "$out" ]]` emptiness semantics:
```bash
helper_journal="$(journalctl -q -t tboot-dnf-helper --since "$SINCE" --no-pager 2>/dev/null || true)"
[[ -z "$helper_journal" ]] || { echo "FAIL: helper journal fired"; exit 1; }
```

This applies to every silence-assertion harness in the project: Gates 3.6, 3.7, 3.8 helper-silence checks, all future Gate 6 negative-validation harnesses, and B.2.4 helper-silence-pre-trigger checks.

**Cross-reference:**
- `06B`: Step 32 Gates 3.5, 3.6, 3.7 (helper-journal-silent assertions)
- Evidence: 2026-05-24 Gate 3.6 first harness run (FAIL on `-- No entries --`) and corrected harness with `-q`

---

### I.5: `--self-test` mode does not emit `ok: helper present and executable`; the helper-presence branch in self-test is `warn`-only on missing helper

**Verdict:** CONTEXT-ONLY (validation-harness over-specification).

**Symptom:**
A Gate 3.5 harness that asserts `--self-test` output contains the line `ok: helper present and executable` (modelled on the equivalent line emitted by `--prime` and `_mode_normal` preflight) fires FAIL even though `--self-test` exited rc=0 and all other expected lines (`ok: running as root`, `ok: required binaries present`, `ok: trust-boundary self-test PASS`, `mode=self-test; preflight + trust-boundary scan only; no compute, no state writes`, `exit reason: self-test PASS`) are present.

**Root cause:**
The decider's `_preflight` function switches on `$MODE` for the helper-presence check. For `normal` and `prime` modes, missing helper is FATAL (returns rc=5 with `err: helper missing at ${HELPER_BIN}`) and present helper emits `ok: helper present and executable at ${HELPER_BIN}`. For `self-test` and `debug-print` modes, missing helper produces a `warn: helper missing at ${HELPER_BIN} (not fatal in ${MODE})` line only, and present helper emits no positive `ok:` line at all. This is per the decider source design contract:

```bash
case "$MODE" in
    normal|dry-run|prime)
        if [[ ! -f "$HELPER_BIN" ]]; then
            err "helper missing at ${HELPER_BIN}"
            return 5
        fi
        info "ok: helper present and executable at ${HELPER_BIN}"
        ;;
    self-test|debug-print)
        [[ -f "$HELPER_BIN" ]] || warn "helper missing at ${HELPER_BIN} (not fatal in ${MODE})"
        ;;
esac
```

The harness assertion was over-strict. The decider's positive contract for `--self-test` is: rc=0, the trust-boundary PASS line, the explicit "no compute, no state writes" line, and the `exit reason: self-test PASS` line. Helper presence in self-test should be cross-verified by a separate frozen-sha invariant on `/usr/local/sbin/tboot-dnf-helper`, not by asserting a positive `ok:` line that the mode doesn't emit.

**Reproduction:**
Run `--self-test` on a system where the helper is present. Observe that `ok: helper present and executable` does not appear in the output, even though the helper IS present.

**Fix or workaround:**
Drop the `ok: helper present and executable` assertion from `--self-test` validation harnesses. Validate helper presence in self-test via the frozen-artifact sha block:
```bash
sha256sum /usr/local/sbin/tboot-dnf-helper | awk '{print $1}' \
  | grep -Ex '4bef223925d3a2520e692403f0c0dd8a932770ec17136008a3d5f6f0b2620f4b' \
  || { echo "FAIL: helper sha drift"; exit 1; }
```

Assert the positive `ok: helper present and executable` line only in `--prime` and `--dry-run` harnesses (where the decider DOES emit it).

**Cross-reference:**
- `06B`: Step 32 Gate 3.5 (`--self-test` harness: corrected to drop the helper-presence positive assertion)
- Decider source `_preflight` mode-switch on helper-presence
- Evidence: 2026-05-24 Gate 3.5 first attempt (FAIL on missing helper-positive line) and corrected harness

---

### I.6: Do not bundle a re-prime refusal probe inside the clean Gate 3.6 baseline-prime gate

**Verdict:** CONDITIONAL (gate-hygiene rule).

**Symptom:**
A Gate 3.6 proposal that runs `--prime` twice in the same gate: once to successfully prime (rc=0), then again without `--confirm-prime` to verify the refusal guard (expecting rc=11 and the `err: baseline already exists` message): deliberately injects an `err:` line into the `tboot-dnf-posttrans` journal under the Gate 3.6 banner. This breaks the gate-hygiene property that "Gate 3.6's journal slice is err-free on success" and contaminates the `last-decision` record's role as the canonical Gate-3-close marker.

**Root cause:**
The decider's `_mode_prime` correctly logs `err: baseline already exists at ${BASELINE_FILE}; refusing to overwrite without --confirm-prime` and returns rc=11 when invoked a second time without `--confirm-prime`. This is the documented refusal guard, and it is correct behaviour. But folding the refusal-guard validation into the same gate as the successful prime mixes two concerns:

1. **Clean prime**: produce the baseline and the unblemished `last-decision` record; the journal slice for this gate should be err-free.
2. **Refusal-guard probe**: confirm the decider's `--confirm-prime` guard works; this is an error-path validation that legitimately produces an err-level line.

Mixing them defeats the purpose of asserting "no err: lines in the Gate 3.6 decider journal slice" as a gate invariant.

**Reproduction:**
After a successful first `--prime`:
```bash
$ tboot-dnf-posttrans --prime
[tboot-dnf-posttrans] err: baseline already exists at /var/lib/tboot-dnf-posttrans/boot-input-manifest.baseline; refusing to overwrite without --confirm-prime
[tboot-dnf-posttrans] err: preflight failed (rc=11)
$ echo $?
11
```
The refusal is expected. Test it outside the clean Gate 3.6 journal window.

**Fix or workaround:**
Gate 3.6 runs `--prime` exactly once, validates rc=0, validates no err: lines in the journal slice, validates the three state-dir files have correct identity, and stops. The refusal-guard probe belongs in a separate optional error-path validation gate (call it Gate 3.6.X or fold into Gates 4–7), NOT in the clean Gate 3.6.

The same hygiene rule applies to all gates: do not deliberately inject error events inside a gate whose contract is "the journal slice is clean."

**Cross-reference:**
- `06B`: Step 32 Gate 3.6 (single `--prime` invocation; no refusal probe in this gate)
- Evidence: 2026-05-24 Gate 3.6 first proposal (rejected due to bundled refusal probe) and corrected single-`--prime` proposal

---

### I.7: In Gate 3.7, run `--debug-print` exactly once and capture rc correctly; do not mask exit codes with `|| true` inside command substitution

**Verdict:** CONDITIONAL (harness-correctness rule).

**Symptom:**
A Gate 3.7 proposal that runs `--debug-print` three times (once for stdout, once for stderr, once for a "determinism re-check") creates extra decider journal sessions, raises the cost of the gate, and adds complexity. Worse, a proposal that captures the exit code via:
```bash
dbg_output="$("$DST" --debug-print 2>/dev/null || true)"
dbg_rc=$?
```
silently masks `--debug-print` failures because `|| true` always succeeds, making `dbg_rc` equal to 0 regardless of whether the command failed. The harness then asserts `[[ "$dbg_rc" -eq 0 ]]` and passes: even if `--debug-print` actually exited non-zero.

**Root cause:**
1. **Multiple `--debug-print` invocations**: each invocation runs preflight and the manifest compute; each produces ~7 info lines under `tboot-dnf-posttrans` in the journal. Running it three times triples the journal noise inside what should be a single-execution verification gate.
2. **`|| true` rc masking**: the rc seen by `dbg_rc=$?` is the rc of the last command in the pipeline, which after `|| true` is always 0. The actual `--debug-print` rc is lost.

**Reproduction:**
```bash
$ false_out="$(/bin/false 2>/dev/null || true)"; echo "rc=$?"
rc=0    # rc of /bin/false (1) was masked by || true
```

**Fix or workaround:**
Run `--debug-print` exactly once. Use a subshell with separate stdout / stderr / rc-capture files under a `mktemp -d` tempdir (cleaned by a path-prefix-guarded `trap`):

```bash
WORK="$(mktemp -d "/run/gate37.XXXXXX")"
trap 'case "$WORK" in /run/gate37.*) rm -rf -- "$WORK" ;; esac' EXIT

dbg_stdout_file="${WORK}/debug-stdout"
dbg_stderr_file="${WORK}/debug-stderr"
dbg_rc_file="${WORK}/debug-rc"

# Subshell preserves rc cleanly via tempfile, without || true masking:
( "$DST" --debug-print >"$dbg_stdout_file" 2>"$dbg_stderr_file" ; echo $? >"$dbg_rc_file" )

dbg_rc="$(cat "$dbg_rc_file")"
[[ "$dbg_rc" -eq 0 ]] || { echo "FAIL: --debug-print rc=${dbg_rc}"; cat "$dbg_stderr_file"; exit 1; }
```

The `( CMD >out 2>err ; echo $? >rc_file )` subshell form preserves the rc through the file. Parsing it afterwards with `dbg_rc="$(cat "$dbg_rc_file")"` recovers the actual exit code.

This pattern applies to any future harness that needs to capture stdout, stderr, and rc from a single command invocation without masking failures.

**Cross-reference:**
- `06B`: Step 32 Gate 3.7 (single `--debug-print` with subshell + tempfile rc capture)
- Evidence: 2026-05-24 Gate 3.7 first proposal (rejected: three invocations + `|| true` masking) and corrected single-invocation proposal

---

## J: Gate 4–7 environment and harness findings (DNF live trigger)

Findings discovered during the 2026-05-25 B.2.3 Gates 4–7 execution session, when the production `.actions` rule was first wired and live DNF transactions invoked the decider end-to-end. None of these are decider-source bugs; they are environmental observations (J.1, J.2) and harness-semantics corrections (J.3) that must be reflected in any future Gate 6A / 6B verifier.

### J.1: `/var/log/dnf5.log` does not record plugin-load entries in this Fedora 43 environment

**Verdict:** CONTEXT-ONLY.

**Symptom:**
A Gate 6A verifier asserting `grep -Fq 'Loaded libdnf plugin "actions"' /var/log/dnf5.log` (or any timestamp-bounded variant) FAILs. Diagnostic inspection of `/var/log/dnf5.log` shows zero `Loaded libdnf plugin` entries across the entire file history, even though the plugin is demonstrably loading (the decider's journald evidence proves the rule fired and the decider ran).

**Root cause:**
The B.2.1 evidence note recorded a `Loaded libdnf plugin "actions" ... version="1.4.0"` line in `/var/log/dnf5.log`. By the time of B.2.3 Gate 6A, plugin-load events were no longer being written to that file in this rebuild. Possible causes (not investigated, since the diagnosis sufficed to switch evidence sources): a dnf5 log-level default change between minor releases, log rotation having truncated the historical entry, or a journald-only logging path that the B.2.1-era note misattributed. The mechanism is not in scope; the fact that `dnf5.log` is not authoritative for plugin-load evidence in this environment is what matters.

**Reproduction:**

```bash
# After a real DNF transaction invokes the production rule:
grep -F 'Loaded libdnf plugin' /var/log/dnf5.log
# returns nothing.

# But the journald-side evidence is present:
journalctl -q -t tboot-dnf-posttrans --since '-5 min' --no-pager
# shows the decider's full info-line stream (preflight, lock acquire,
# manifest recompute, baseline compare result, SKIP, exit reason).
```

**Fix or workaround:**
Use journald evidence (`journalctl -t tboot-dnf-posttrans --since "$PRE_TIME"`) as authoritative. The decider emits its info-line stream under tag `tboot-dnf-posttrans`; if the decider ran, the journal has the proof. The dnf5.log dependency in any verifier is a portability hazard and must be removed.

For any Gate 6A / 6B / B.2.4 verifier, the correct evidence chain is:

1. dnf transaction rc=0 (transaction committed).
2. `journalctl -q -t tboot-dnf-posttrans --since "$PRE_TIME"` contains exactly 1 `exit reason: …` line.
3. The `last-decision` epoch advanced past `$PRE_TIME`'s epoch.

Steps 2 and 3 together prove the plugin loaded and the decider ran; step 1 proves dnf was the trigger. No `dnf5.log` assertion is needed.

**Cross-reference:**
- `06B`: Step 33 Gate 6A canonical script (uses journald, not dnf5.log).
- Evidence: 2026-05-25 Gate 6A original verifier FAIL 183 (dnf5.log grep returned empty) → diagnostic → first correction removed the timestamp bound (still failed) → second correction removed the check entirely.
- Companion: H.1 (dnf5 has no runtime verbosity flag: same family of "dnf5.log is not the evidence surface you think it is").

---

### J.2: `date -Iseconds` and `/var/log/dnf5.log` use incompatible timezone-offset formats

**Verdict:** CONTEXT-ONLY (a harness-engineering pitfall).

**Symptom:**
A Gate 6A verifier captures a pre-transaction timestamp via `PRE_TIME="$(date -Iseconds)"` (which produces `2026-05-24T23:42:55+02:00` on this system) and then tries to assert that a `dnf5.log` line with timestamp >= `PRE_TIME` exists. The lexicographic string comparison fails even when the log clearly contains a newer line.

**Root cause:**
`date -Iseconds` formats the TZ offset as `+02:00` (with the colon). `dnf5.log`'s timestamp format omits the colon: `+0200`. Lexicographic comparison of `2026-05-24T23:42:55+02:00` against `2026-05-24T23:43:01+0200` does not compare as the operator intends. Even semantic comparison would need both sides normalised to the same format. The two formats are both ISO-8601-related but not interchangeable as string keys.

**Reproduction:**

```bash
$ date -Iseconds
2026-05-24T23:42:55+02:00

$ tail -1 /var/log/dnf5.log
2026-05-24T23:43:01+0200 [...some content...]

$ [[ "2026-05-24T23:43:01+0200" > "2026-05-24T23:42:55+02:00" ]] && echo NEWER || echo OLDER
OLDER   # wrong by semantic intent: the second timestamp IS newer
```

**Fix or workaround:**
Do not lexicographically compare timestamps against `dnf5.log`. Two correct alternatives:

1. **Preferred** (and what J.1 already recommends): drop the `dnf5.log` evidence path entirely. Use `journalctl -q -t TAG --since "$PRE_TIME"`, which handles `+02:00`-style ISO-8601 input natively via systemd's time parser.
2. **If you absolutely must compare against `dnf5.log`**: convert both sides to epoch seconds via `date -d "$TIMESTAMP" +%s` and compare numerically. Strip the colon from the `+02:00` offset before passing to `date -d` (some `date` builds do not accept the colon form on input).

**Cross-reference:**
- `06B`: Step 33 Gate 6A canonical script (uses `journalctl --since` with `date -Iseconds`-format input).
- Evidence: 2026-05-25 Gate 6A original FAIL 183 was triggered by the timestamp-bound `dnf5.log` grep before J.1 made the file's contents the dominant problem; J.2 is the lesson on why even a populated `dnf5.log` would have been hard to use safely.

---

### J.3: Lock-path invariants must be "not actively held" via non-blocking `flock`, not "path absent"

**Verdict:** FORBIDDEN (the "path absent" invariant; not the symptom).

**Symptom:**
A Gate 6A verifier asserts `[ ! -e "$DECIDER_LOCK" ] && [ ! -L "$DECIDER_LOCK" ]` against `/run/tboot-dnf-posttrans.lock` after the decider has exited cleanly. The check FAILs because the lock file persists on disk: `ls -la /run/tboot-dnf-posttrans.lock` shows a regular 0-byte root:root 0644 file, owned by the dead decider process. `fuser -v /run/tboot-dnf-posttrans.lock` returns no current holder; the lock is not actively held; the decider released its `flock` cleanly. The file persists because the decider does not unlink it on exit, which is the standard `flock(1)` idiom.

**Root cause:**
The `flock(1)` shell built-in / utility acquires an advisory `LOCK_EX` (or `LOCK_SH`) on an open file descriptor. The lock is automatically released when the file descriptor is closed (which happens at process exit). The lock state lives in kernel memory keyed by the inode, not in the filesystem. The file itself is not removed by `flock`; the standard idiom is to leave it on disk between invocations so that the next holder reuses the same inode without a race. Tools that wrap `flock` (`flock -e /run/foo.lock command...`) inherit this behaviour: lock file persists, kernel state clears.

Asserting "lock path is absent" is therefore an incorrect invariant. It is testing a side-effect (unlinking the lock file) that the standard `flock` idiom never produces. The correct invariant is "the lock is not actively held", which is testable via a non-blocking `flock -n` probe against the persistent file.

**Reproduction:**

```bash
LOCK="/run/tboot-dnf-posttrans.lock"

ls -la "$LOCK"
# -rw-r--r--. 1 root root 0 May 24 23:42 /run/tboot-dnf-posttrans.lock

fuser -v "$LOCK"
# (empty: no current holder)

# The wrong invariant FAILs:
[ ! -e "$LOCK" ] && echo OK || echo FAIL
# FAIL

# The right invariant PASSes:
( exec 9<>"$LOCK"; flock -n 9 ) && echo "OK: not actively held" || echo "FAIL: held"
# OK: not actively held
```

**Fix or workaround:**
Replace the "lock path is absent" invariant with a "lock is not actively held" probe. The probe must be:

- **Non-truncating.** Open the file for read/write via `9<>` (not `9>`, which truncates) so a verifier never modifies the lock file's contents.
- **Symlink-rejecting.** Refuse to probe a symlink: symlinks at well-known lock paths are a tampering indicator. Use `[ -f "$LOCK" ] && [ ! -L "$LOCK" ]` as the gate before the `flock` probe.
- **Tolerant of an absent path.** If `$LOCK` does not exist, that is also "not actively held": pass.
- **Non-blocking.** `flock -n 9` either acquires-and-releases (when scope ends) or returns rc!=0 immediately. No timeout, no hang.

The canonical bash pattern is:

```bash
if [ -e "$LOCK" ] || [ -L "$LOCK" ]; then
  [ -f "$LOCK" ] && [ ! -L "$LOCK" ] \
    || { echo "FAIL: lock path is not a regular non-symlink file"; exit 1; }

  (
    exec 9<>"$LOCK"
    flock -n 9
  ) || { echo "FAIL: lock is actively held"; exit 1; }
fi
```

This pattern is used unchanged for both `/run/tboot-dnf-posttrans.lock` (decider) and `/run/tboot-dnf-helper.lock` (helper) in the canonical Gate 6A / 6B / 7 verifiers in `06B` Step 33.

**Cross-reference:**
- `06B`: Step 33 Gate 6A / 6B / 7 canonical scripts (every lock-state assertion uses the `flock <> -n` probe).
- Evidence: 2026-05-25 Gate 6A continuation verifier FAIL 197 (`/run/tboot-dnf-posttrans.lock` persisted as a regular root:root 0644 0-byte file after the decider exited cleanly) → diagnostic (`fuser -v` empty; `( exec 9>"$LOCK"; flock -n 9 )` succeeded with "lock file exists but is NOT held") → first correction switched to `9>`-open flock probe (truncating; rejected on second review) → second correction switched to `9<>`-open flock probe with symlink rejection (this is the canonical form).

---

## K: B.2.4: decider/helper terminology, DNF5 `--assumeno` semantics, baseline-vs-PCR-value advancement

### K.1: Decider logs production code path as `mode=normal`; helper logs it as `mode=production`: harnesses must not conflate them

**Verdict:** CONTEXT-ONLY, but harness-critical.

**Symptom:**
A B.2.4 Gate 3 validation harness fails at a single assertion (`FAIL 411: helper normal-mode start lines = 0 (expected exactly 1)`) even though the full positive drift path completed correctly end-to-end. Every other piece of evidence in the same run confirms success: the decider fired exactly once with `mode=normal`, logged `baseline compare result: drift`, logged `helper exited 0 (success)`, and logged `exit reason: helper succeeded; baseline updated; sentinel cleared`; the helper journal is non-empty and contains the full helper run trace. The transaction itself returned rc=0; `dracut-network` is installed; the baseline manifest has advanced; the PCR 11 prediction has advanced; the sentinel was written then cleared; trust-chain shas are byte-stable.

**Root cause:**
The `tboot-dnf-posttrans` decider and the `tboot-dnf-helper` helper use different vocabulary for "the real (production) execution path versus the dry / test paths":

| Component | Production code path log tag | Dry-run / test path log tags |
|---|---|---|
| Decider (`tboot-dnf-posttrans`) | `mode=normal` (`info: mode=normal; acquiring decider lock`) | `mode=dry-run`, `mode=debug-print`, `mode=self-test`, `mode=prime` |
| Helper (`tboot-dnf-helper`) | `mode=production` (`info: starting version=… mode=production`) | `mode=dry-run`, `mode=self-test` |

The two refer to the same end-to-end execution. The terminology differs because the decider was authored later (in B.2.3) and chose `normal` to distinguish from its `prime`, `dry-run`, `debug-print`, and `self-test` modes; the helper was authored earlier (in B.2.2) and uses `production` to distinguish from its `dry-run` and `self-test` modes. A harness assertion of the shape `grep -Ec '…starting version=.*mode=normal' "$HELP_JOURNAL"` matches zero lines because the helper does not log `mode=normal`. The reverse mistake: looking for `mode=production` in the decider journal: also fails.

**Reproduction:**
After any successful Gate 3 transaction, inspect both journals:

```bash
journalctl -t tboot-dnf-posttrans --since "<gate-3-PRE_TS>" --no-pager -o cat | grep -E 'mode='
journalctl -t tboot-dnf-helper    --since "<gate-3-PRE_TS>" --no-pager -o cat | grep -E 'mode='
```

The decider output includes `info: mode=normal; acquiring decider lock /run/tboot-dnf-posttrans.lock (flock -w 120)`. The helper output includes `info: starting version=1.0.0 mode=production trigger=(none) ppid=<pid>`. The two `mode=` values differ.

**What was observed on 2026-05-25 (reference run):**
The rev-3 Gate 3 harness used the wrong regex for the helper-start assertion and failed at FAIL 411. The mutation itself completed correctly: every other piece of evidence in the captured journal confirms this. The post-hoc continuation verifier below was then run against the now-mutated live state and passed 43/43, providing complete coverage of the Gate 3 evidence. **Gate 3 is closed by post-hoc verification, not by a clean harness replay against the reference-run state.** A corrected 61-assertion harness with the `mode=production` fix applied is recorded in `06B` Step 34 as the canonical replay form for future Gate 3 reruns from a clean rollback to `module5-b2-4-pre-drift-transaction`; it was not executed against the 2026-05-25 reference-run state.

**Fix or workaround:**
For harnesses (the recommended path; lightweight, no source change):

- When asserting decider production firing: `grep -Ec 'mode=normal; acquiring decider lock'` against the `tboot-dnf-posttrans` journal.
- When asserting helper production firing: `grep -Ec '(^|[[:space:]])starting version=.*mode=production'` against the `tboot-dnf-helper` journal.
- Never assume the two components use the same `mode=` vocabulary.

**Continuation verifier: canonical recovery script for FAIL 411.** When a Gate 3 harness fails at FAIL 411 (or an equivalent "helper start lines = 0" assertion) but the rest of the transaction completed correctly, do **not** roll back to `module5-b2-4-pre-drift-transaction` purely on that signal. Confirm the mutation succeeded by reading the helper journal manually, then run the continuation verifier below. This is the verifier that closed B.2.4 Gate 3 on 2026-05-25 at 43/43 PASS.

```bash
bash <<'EOF'
set -u

# B.2.4 Gate 3 continuation verifier: applies the K.1 fix and revalidates.
# Closed B.2.4 Gate 3 on 2026-05-25 at 43/43 PASS after the rev-3 reference-run
# harness failed at FAIL 411 on the mode=normal/mode=production regex bug.
# Replace PRE_TS / PRE_EPOCH with the actual values captured from the failing
# run if reproducing on a different rebuild.

PRE_TS="2026-05-25T09:53:38+02:00"
PRE_EPOCH="1779695618"

EXP_DECIDER_SHA="35ad8733f190483f1bd6d071d2aa7cb8b1549286f9040086182443c584473cdd"
EXP_PRODRULE_SHA="dfbffe56fa95f4e0231569bb3a1aabc8952b9f99c9d290e4f8fb1005ef90d798"
EXP_HOOK_SHA="5857e51d5551e05a7ca384f71529ec2fef07f5a6dbe40df70c6bbe49d720947e"
EXP_HELPER_SHA="4bef223925d3a2520e692403f0c0dd8a932770ec17136008a3d5f6f0b2620f4b"
EXP_PREDICT_SHA="2d4985fa27726efff50bd91ba33ed1dd6e516fe2dd7350a48c73369b61aff160"
EXP_OLD_BASELINE_SHA="75dc4f5651fc8996d39b545ac0bf7244a2a5888403b0ba0cd448a671daf80160"
EXP_PRE_UKI_SHA="b6002e666afdf71e5a083311295ac6a5f3ef3b443ceb4ec8c16c9ddb41077943"
EXP_PRE_PCR11="A03EB49C9335D721758AE8AA9C34DA87664C5CAF9FAD7C314C68C038B16435DD"

EXP_MID="0123456789abcdef0123456789abcdef"
EXP_KVER_BOOTED="6.19.14-200.fc43.x86_64"
EXP_KVER_SECONDARY="6.17.1-300.fc43.x86_64"

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
SUCCESS_MARKER="${HELPER_STATE}/last-success"
FAILURE_MARKER="${HELPER_STATE}/last-failure"
FAILURE_LOG="${HELPER_STATE}/last-failure.log"

UKI_PATH="/boot/efi/EFI/Linux/${EXP_MID}-${EXP_KVER_BOOTED}.efi"
UKI_PATH_SECONDARY="/boot/efi/EFI/Linux/${EXP_MID}-${EXP_KVER_SECONDARY}.efi"
INITRAMFS_PATH="/boot/initramfs-${EXP_KVER_BOOTED}.img"
PCR11_FILE="/root/tboot-lab/state/expected-pcr11-after-hook-uki.txt"

DECIDER_LOCK="/run/tboot-dnf-posttrans.lock"
HELPER_LOCK="/run/tboot-dnf-helper.lock"

PASS=0
fail() { echo "FAIL $1: $2"; exit "$1"; }

assert_lock_free() {
    local lock="$1" code="$2"
    if [ -e "$lock" ] || [ -L "$lock" ]; then
        [ -f "$lock" ] && [ ! -L "$lock" ] || fail "$code" "$lock not regular non-symlink"
        ( exec 9<>"$lock"; flock -n 9 ) || fail "$code" "$lock actively held"
    fi
}

journal_has_error() {
    grep -Eiq '(^|[[:space:]])(err|error|fail|failed|failure)([[:space:]:]|$)' "$1"
}
show_error_lines() {
    grep -Ein '(^|[[:space:]])(err|error|fail|failed|failure)([[:space:]:]|$)' "$1" || true
}

tmpd="$(mktemp -d)"
trap 'rm -rf "$tmpd"' EXIT HUP INT TERM

DEC_JOURNAL="$tmpd/decider.log"
HELP_JOURNAL="$tmpd/helper.log"
HOOK_JOURNAL="$tmpd/hook.log"

journalctl -t tboot-dnf-posttrans --since "$PRE_TS" --no-pager -o cat >"$DEC_JOURNAL"  2>&1 || true
journalctl -t tboot-dnf-helper    --since "$PRE_TS" --no-pager -o cat >"$HELP_JOURNAL" 2>&1 || true
journalctl -t 80-tpm2-sign        --since "$PRE_TS" --no-pager -o cat >"$HOOK_JOURNAL" 2>&1 || true

rpm -q "$CANDIDATE" >/dev/null 2>&1 || fail 101 "$CANDIDATE not installed"
INSTALLED_NEVRA="$(rpm -q "$CANDIDATE")"; PASS=$((PASS+1))

[ -s "$DEC_JOURNAL" ] || fail 201 "decider journal empty"; PASS=$((PASS+1))
DEC_START_COUNT="$(grep -Ec 'mode=normal; acquiring decider lock' "$DEC_JOURNAL" || true)"
[ "$DEC_START_COUNT" -eq 1 ] || fail 202 "decider mode=normal count=$DEC_START_COUNT"; PASS=$((PASS+1))
grep -Eq 'baseline compare result: drift' "$DEC_JOURNAL" || fail 203 "decider did not report drift"; PASS=$((PASS+1))
grep -Eq 'helper invocation decision: INVOKE' "$DEC_JOURNAL" || fail 204 "decider did not invoke helper"; PASS=$((PASS+1))
grep -Eq 'helper exited 0' "$DEC_JOURNAL" || fail 205 "decider did not record helper exit 0"; PASS=$((PASS+1))
grep -Eq 'sentinel cleared' "$DEC_JOURNAL" || fail 206 "decider did not clear sentinel"; PASS=$((PASS+1))
if journal_has_error "$DEC_JOURNAL"; then show_error_lines "$DEC_JOURNAL"; fail 207 "decider has error-like lines"; fi; PASS=$((PASS+1))

[ -s "$HELP_JOURNAL" ] || fail 301 "helper journal empty"; PASS=$((PASS+1))

# THE K.1 FIX: helper logs mode=production, NOT mode=normal.
HELP_PROD_COUNT="$(grep -Ec '(^|[[:space:]])starting version=.*mode=production' "$HELP_JOURNAL" || true)"
[ "$HELP_PROD_COUNT" -eq 1 ] || fail 302 "helper production-mode start count=$HELP_PROD_COUNT"; PASS=$((PASS+1))

grep -Fq "kernels enumerated (2):" "$HELP_JOURNAL" || fail 303 "helper did not enumerate 2 kernels"; PASS=$((PASS+1))
grep -Fq "dracut: ok for $EXP_KVER_BOOTED"             "$HELP_JOURNAL" || fail 304 "dracut ok missing for booted"; PASS=$((PASS+1))
grep -Fq "dracut: ok for $EXP_KVER_SECONDARY"          "$HELP_JOURNAL" || fail 305 "dracut ok missing for secondary"; PASS=$((PASS+1))
grep -Fq "kernel-install add: ok for $EXP_KVER_BOOTED" "$HELP_JOURNAL" || fail 306 "kernel-install ok missing for booted"; PASS=$((PASS+1))
grep -Fq "kernel-install add: ok for $EXP_KVER_SECONDARY" "$HELP_JOURNAL" || fail 307 "kernel-install ok missing for secondary"; PASS=$((PASS+1))
grep -Eq 'predict: ok; FINAL_PCR11=' "$HELP_JOURNAL" || fail 308 "helper predict ok missing"; PASS=$((PASS+1))
grep -Eq 'completed mode=production kernels=2 pcr11=' "$HELP_JOURNAL" || fail 309 "helper completion line missing"; PASS=$((PASS+1))
if journal_has_error "$HELP_JOURNAL"; then show_error_lines "$HELP_JOURNAL"; fail 310 "helper has error-like lines"; fi; PASS=$((PASS+1))

[ -s "$HOOK_JOURNAL" ] || fail 401 "hook journal empty"; PASS=$((PASS+1))
HOOK_BOOTED_COUNT="$(grep -Fc "starting for kernel ${EXP_KVER_BOOTED}" "$HOOK_JOURNAL" || true)"
[ "$HOOK_BOOTED_COUNT" -eq 1 ] || fail 402 "hook booted count=$HOOK_BOOTED_COUNT"; PASS=$((PASS+1))
HOOK_SECONDARY_COUNT="$(grep -Fc "starting for kernel ${EXP_KVER_SECONDARY}" "$HOOK_JOURNAL" || true)"
[ "$HOOK_SECONDARY_COUNT" -eq 1 ] || fail 403 "hook secondary count=$HOOK_SECONDARY_COUNT"; PASS=$((PASS+1))
grep -Fq "kernel ${EXP_KVER_BOOTED}: ukify-native signed UKI ready"    "$HOOK_JOURNAL" || fail 404 "hook ready line missing for booted"; PASS=$((PASS+1))
grep -Fq "kernel ${EXP_KVER_SECONDARY}: ukify-native signed UKI ready" "$HOOK_JOURNAL" || fail 405 "hook ready line missing for secondary"; PASS=$((PASS+1))
if journal_has_error "$HOOK_JOURNAL"; then show_error_lines "$HOOK_JOURNAL"; fail 406 "hook has error-like lines"; fi; PASS=$((PASS+1))

POST_LD_DECISION="$(awk -F= '$1=="decision"{print $2}' "$LAST_DECISION")"
[ "$POST_LD_DECISION" = "helper-success" ] || fail 501 "last-decision=$POST_LD_DECISION"; PASS=$((PASS+1))
POST_BASELINE_SHA="$(sha256sum "$BASELINE" | awk '{print $1}')"
[ "$POST_BASELINE_SHA" != "$EXP_OLD_BASELINE_SHA" ] || fail 502 "baseline did not change"; PASS=$((PASS+1))
POST_LM_SHA="$(sha256sum "$LAST_MANIFEST" | awk '{print $1}')"
[ "$POST_LM_SHA" = "$POST_BASELINE_SHA" ] || fail 503 "last-manifest != new baseline"; PASS=$((PASS+1))
[ ! -e "$SENTINEL" ]   || fail 504 "sentinel still present"; PASS=$((PASS+1))
[ ! -e "$LAST_ERROR" ] || fail 505 "last-error present"; PASS=$((PASS+1))
[ -f "$SUCCESS_MARKER" ] || fail 506 "success marker absent"
SM_MTIME="$(stat -c '%Y' "$SUCCESS_MARKER")"
[ "$SM_MTIME" -ge "$PRE_EPOCH" ] || fail 507 "success marker predates Gate 3"; PASS=$((PASS+1))
{ [ ! -e "$FAILURE_MARKER" ] && [ ! -e "$FAILURE_LOG" ]; } || fail 508 "failure marker/log present"; PASS=$((PASS+1))

POST_UKI_SHA="$(sha256sum "$UKI_PATH" | awk '{print $1}')"
[ "$POST_UKI_SHA" != "$EXP_PRE_UKI_SHA" ] || fail 601 "booted UKI sha did not change"; PASS=$((PASS+1))
sbverify --cert /etc/uefi-keys/db.crt "$UKI_PATH" >/dev/null 2>&1 || fail 602 "booted UKI sbverify failed"; PASS=$((PASS+1))
[ -f "$UKI_PATH_SECONDARY" ] && [ ! -L "$UKI_PATH_SECONDARY" ] || fail 603 "secondary UKI missing/not regular"
sbverify --cert /etc/uefi-keys/db.crt "$UKI_PATH_SECONDARY" >/dev/null 2>&1 || fail 604 "secondary UKI sbverify failed"; PASS=$((PASS+1))
POST_PCR11_VALUE="$(tr -d '[:space:]' <"$PCR11_FILE" | awk '{print toupper($0)}')"
[ -n "$POST_PCR11_VALUE" ] || fail 605 "stored PCR11 empty"; PASS=$((PASS+1))
RT_PCR11="$(awk '{gsub(/^0x/,""); print toupper($0)}' /sys/class/tpm/tpm0/pcr-sha256/11)"
[ "$RT_PCR11" = "$EXP_PRE_PCR11" ] || fail 606 "runtime PCR11 changed without reboot"; PASS=$((PASS+1))

sha256sum "$DECIDER"  | awk '{print $1}' | grep -Exq "$EXP_DECIDER_SHA"  || fail 701 "decider sha drift"
sha256sum "$HELPER"   | awk '{print $1}' | grep -Exq "$EXP_HELPER_SHA"   || fail 702 "helper sha drift"
sha256sum "$PREDICT"  | awk '{print $1}' | grep -Exq "$EXP_PREDICT_SHA"  || fail 703 "predict sha drift"
sha256sum "$HOOK"     | awk '{print $1}' | grep -Exq "$EXP_HOOK_SHA"     || fail 704 "hook sha drift"
sha256sum "$PRODRULE" | awk '{print $1}' | grep -Exq "$EXP_PRODRULE_SHA" || fail 705 "prod rule sha drift"
PASS=$((PASS+5))

mapfile -t ACTIONS_POST < <(find "$ACTIONS_DIR" -maxdepth 1 -type f -name '*.actions' -printf '%p\n' | sort)
{ [ "${#ACTIONS_POST[@]}" -eq 1 ] && [ "${ACTIONS_POST[0]}" = "$PRODRULE" ]; } || fail 706 "unexpected active actions"
PASS=$((PASS+1))

assert_lock_free "$DECIDER_LOCK" 707
assert_lock_free "$HELPER_LOCK"  708
PASS=$((PASS+1))

echo ""
echo "=== B.2.4 Gate 3 continuation verifier: ${PASS} assertions PASS ==="
echo "installed_candidate:          $INSTALLED_NEVRA"
echo "baseline_sha_POST:            $POST_BASELINE_SHA"
echo "last_decision_decision_POST:  $POST_LD_DECISION"
echo "uki_sha_POST:                 $POST_UKI_SHA"
echo "pcr11_value_POST:             $POST_PCR11_VALUE"
echo "runtime_pcr11_now:            $RT_PCR11"
echo "hook_count_booted_kernel:     $HOOK_BOOTED_COUNT"
echo "hook_count_secondary_kernel:  $HOOK_SECONDARY_COUNT"
EOF
```

**Reference-run result (2026-05-25): 43/43 PASS.** Installed candidate `dracut-network-107-8.fc43.x86_64`. Baseline sha post-Gate-3 `6b032bd81a7f3a57b8350c76cf690deb3a942704a76455b881f7a5d98df82a47`. `last-decision=helper-success`. Booted UKI sha post-Gate-3 `0ac32c8b98428f9a44339a5991eb86e2704986e84ce6e09ae9721bba41726949`. Stored PCR 11 value post-Gate-3 `28E66CE224C9BB0711059470D77103A5AB5A5C1674C03D1BA1563C2FEE52C90F`. Runtime PCR 11 unchanged at `A03EB49C9335D721758AE8AA9C34DA87664C5CAF9FAD7C314C68C038B16435DD` (correct: no reboot). Hook count 1 per kernel.

**Operational consequence:**
- Harness reviewers must check helper-start regexes specifically for `mode=production` and decider-start regexes specifically for `mode=normal`. Cross-mixing is the failure mode.
- A failing harness at "helper start count = 0" must be triaged by reading the helper journal manually before any rollback is considered. The mutation may have completed correctly.
- The continuation verifier above is the canonical recovery path for FAIL 411 specifically, and is what closed B.2.4 Gate 3 on 2026-05-25 at 43/43 PASS.

**Cross-reference:**
- `06B` Step 34: Gate 3 reference-run outcome section (documents the FAIL 411 + post-hoc closure path) and the replay-form 61-assertion harness with the K.1 fix.
- Evidence: 2026-05-25 reference run on VM 500.

---

### K.2: DNF5 `--assumeno` returns rc=1 on Fedora 43; refusal phrase is `Operation aborted by the user.`

**Verdict:** CONTEXT-ONLY.

**Symptom:**
A Gate 1 preflight harness that runs `dnf install --assumeno "$CANDIDATE"` to preview the resolved transaction plan needs to know which return codes are "normal refusal" versus "unexpected error". A naive `set -e` script would treat the non-zero rc as failure even when the preview succeeded. Conversely, a harness that accepts any non-zero rc as "normal refusal" without inspecting the output content could mask a real DNF error and approve an unsafe candidate.

**Root cause:**
On Fedora 43 with `dnf5-5.x` (`libdnf5-plugin-actions-5.2.18.0-3.fc43.x86_64`), `dnf install --assumeno <pkg>` resolves the transaction, prints the planned actions, prompts the user with `Is this ok [y/N]:`, treats the `--assumeno` flag as an automatic "no" reply, prints `Operation aborted by the user.`, and exits with rc=1. This rc=1 is the documented refusal exit, not a resolution failure: the transaction simply never commits.

Other DNF5 outcomes that also produce non-zero exits with normal-looking output: `Nothing to do` (rc=0 on dnf5 in observed cases; can vary by package state); resolution conflicts produce different output shapes (`Problem: …`) and different non-zero rc values that should not be confused with the assumeno refusal path.

**Reproduction:**

```bash
dnf install --assumeno dracut-network; echo "rc=$?"
# … transaction plan …
# Operation aborted by the user.
# rc=1
```

```bash
dnf install --assumeno some-nonexistent-package-name; echo "rc=$?"
# … error message …
# rc=1   (but output shape is different: no "Operation aborted" phrase)
```

**Fix or workaround:**
Harness pattern that is both safe and tight:

```bash
case "$ASSUMENO_RC" in
    0|1) : ;;
    *)   fail 300 "dnf --assumeno unexpected rc=$ASSUMENO_RC" ;;
esac

# rc alone is not enough: also require the output to look like a refusal.
grep -Eiq 'Operation aborted|Is this ok.*\[?y/N\]?|Nothing to do' "$ASSUMENO_OUT" \
    || fail 301 "dnf --assumeno output did not match a documented preview-refusal shape"
```

Additional safety: after the assumeno call, scan journald for the relevant tag(s) over the assumeno time window and assert no decider firing: `--assumeno` should never commit a transaction. If the decider journal contains entries during the window, something has gone wrong (the transaction unexpectedly committed, or some other transaction overlapped).

**Operational consequence:**
- Acceptable assumeno rc: 0 or 1.
- Required output evidence: refusal phrase present.
- Required negative evidence: decider journal silent over the assumeno window.
- This three-part check is what passed at the B.2.4 Gate 1 reference run on 2026-05-25.

**Cross-reference:**
- `06B` Step 34 Gate 1 Phase 3 commands.
- Evidence: 2026-05-25 Gate 1 reference run (rc=1, `Operation aborted by the user.`, decider journal silent).
- Related: H.2 (`dnf reinstall` NEVRA pitfall: different transaction class, same family of DNF5 observability concerns).

---

### K.3: Baseline manifest sha and PCR 11 value advance on different axes: baseline tracks file-content manifest drift, PCR 11 tracks measured-UKI-section drift; both advancing simultaneously is the strong-closure outcome

**Verdict:** CONTEXT-ONLY.

**Symptom:**
After a successful B.2.4 drift transaction (Gate 3), the operator observes that the decider's baseline manifest sha has changed (`75dc4f56…0160` → `6b032bd8…2a47`) and the stored PCR 11 prediction value has also changed (`A03EB49C…35DD` → `28E66CE2…C90F`). It is tempting to assume one implies the other, or that one is redundant with the other. Both can change. Both can fail to change. The two outcomes are orthogonal and measure different things; treating them as one signal misreads the system.

**Root cause:**
Two independent computations are happening:

1. **Baseline manifest sha.** Computed by `tboot-dnf-posttrans` from the on-disk *boot-input file-content manifest*: the 14-category enumeration of `/etc/dracut.conf`, `/usr/lib/dracut/modules.d/`, `/lib/modules/<kver>/`, `/usr/lib/firmware/`, udev rules, modprobe configs, systemd early-boot units, cryptsetup tooling, tpm2-tss, storage tooling, fstab/crypttab, kernel-install configs, public-trust certs, and Fedora kernel config. The decider's `_compare_to_baseline` returns `drift` when any file in any category changes content (or directory metadata for `_emit_dir_meta` categories). This signal determines whether the helper is invoked at all.

2. **Stored PCR 11 prediction value.** Computed by `tboot-predict-pcr11` from the freshly-rebuilt booted UKI's *measured-content sections*: `.linux`, `.initrd`, `.cmdline`, `.osrel`, `.uname`, `.sbat`, `.pcrpkey`. The predictor runs `systemd-measure calculate` on these sections to produce the PCR 11 value the firmware will produce at next boot. This signal is what `systemd-cryptsetup` will compare against runtime PCR 11 to authorise the TPM2 disk unlock.

These cover different surfaces:

| Outcome | Baseline sha | PCR 11 value | Operational meaning |
|---|---|---|---|
| (a) Both advance | changed | changed | **Strong-closure path.** Drift actually changed something measured-into-the-initramfs. dracut autodetect or kernel-install behaviour pulled the dracut-sensitive content forward into the UKI. The new prediction at runtime is the live test that the chain tracks measured-content drift end-to-end. |
| (b) Baseline advances, PCR 11 unchanged | changed | identical | **Helper-path-only closure.** Drift on disk was real and the helper ran correctly, but the measured-content inputs to the new UKI were bit-identical to the prior UKI (dracut omitted the new module from host-only mode, or the package's payload was outside the dracut module-resolution path). Chain machinery is proven; value-tracking is not exercised. See finding E.3 for the reverse scenario (UKI sha differs, PCR 11 value unchanged due to ukify non-determinism). |
| (c) Baseline unchanged | unchanged | (not refreshed) | **No-drift path.** Decider returns `unchanged`, helper not invoked, PCR 11 file not touched. This is what Gates 6A + 6B exercised at B.2.3 closure. |
| (d) Baseline unchanged, PCR 11 advances | unchanged | changed | **Anomalous.** Should not occur via the trusted-boot chain. Indicates either an out-of-band manual `predict_pcr11_from_uki.sh` run or a chain bypass. Investigate. |

**Reproduction:**

Outcome (a), B.2.4 Gate 3 on 2026-05-25:

```
PRE baseline:  75dc4f5651fc8996d39b545ac0bf7244a2a5888403b0ba0cd448a671daf80160
POST baseline: 6b032bd81a7f3a57b8350c76cf690deb3a942704a76455b881f7a5d98df82a47
PRE PCR 11:    A03EB49C9335D721758AE8AA9C34DA87664C5CAF9FAD7C314C68C038B16435DD
POST PCR 11:   28E66CE224C9BB0711059470D77103A5AB5A5C1674C03D1BA1563C2FEE52C90F
```

Outcome (b) example pattern: install a dracut module package that is intentionally not auto-loaded (`dracut-fips` on a non-FIPS host typically); baseline will change (a file appeared under `/usr/lib/dracut/modules.d/`), helper will run, predictor will run, but the UKI's `.initrd` section will be bit-identical because dracut chose not to include the module.

Outcome (c): Gates 6A and 6B of B.2.3 closure (`dnf install -y figlet` and `dnf remove -y figlet`).

**Fix or workaround:**
Not a defect. The architecture deliberately separates these signals. Operationally:

- **B.2.4 Gate 3 assertion logic:** require baseline sha advancement AND PCR 11 *file mtime* advancement; do NOT require PCR 11 *value* advancement. Value advancement is recorded as informational and elevates the closure path from (b) to (a). The 2026-05-25 reference run produced outcome (a).
- **B.2.4 Gate 7 discipline-gate-lift decision:** the strong-closure path (outcome a) is sufficient to lift the blind `dnf update` discipline gate at Gate 7 close, **subject to Gate 6 reboot validation** (runtime PCR 11 must byte-match the new stored prediction). The strong-closure path is *available* at Gate 7 because of the Gate 3 value advancement; it is not yet *confirmed* until Gate 6 runs.
- **For Module 4 monitoring and attack scenarios:** drift in either signal independently is a useful detection surface.

**Operational consequence:** Document the closure outcome at Gate 7 close: "B.2.4 closed via outcome (a): strong-closure path; PCR 11 value advanced from `A03EB49C…35DD` to `28E66CE2…C90F` at Gate 3 and was validated against runtime PCR 11 at Gate 6 reboot."

**Cross-reference:**
- `06B` Step 34 Gate 3; future runbook step covering Gates 4–7 (pending) for the discipline-gate-lift decision.
- Related: E.3 (PCR 11 value invariance when measured-content inputs are unchanged).
- Architecture: `05_Update_Workflows_and_Key_Storage.md` §3.3.
- Evidence: 2026-05-25 Gate 3 reference run.

---

## L: ESP forensic UKI audit discipline

### L.1: ESP `/EFI/Linux/` accumulates firmware-loadable forensic UKIs from earlier development; audit before high-risk reboot gates; cleanup is a separate gate

**Verdict:** CONDITIONAL: read-only audit-discipline + cleanup procedure, not a runtime bug.

**Symptom.** A `find /boot/efi/EFI/Linux -maxdepth 1 -type f -iname '*.efi'` inventory on a system that has been through Module 5 development (B.1 / B.3 hook iteration) returns more `.efi` files than the kernel-install + helper chain would produce on its own. On the reference VM 500 system at B.2.4 Gate 7 (2026-05-25): 5 total `.efi` files where only 2 are expected (the booted-kernel MID-prefixed UKI and the secondary-kernel MID-prefixed UKI). The 3 unexpected files are not `.loaderror`-suffixed (B.1's broken-objcopy forensic-preservation convention) and are therefore *enumerated by systemd-boot* and *loadable by firmware*.

**Why this is stronger than "menu clutter".** Every unexpected `.efi` file on the reference run was independently verified as:

- **`sbverify`-OK against `/etc/uefi-keys/db.crt`**: signed by the project's own custom Secure Boot key chain.
- **`signer_count = 1`**: no double-signing artefact, no rejection by the validated `80-tpm2-sign.install` hook's one-signer rule.
- **VMA-sane** (no section at VMA ≥ `0x180000000`): no `06F` B.1-style layout corruption that would cause firmware `LoadImage()` rejection.

Together these mean **firmware would accept any of these UKIs at boot if it selected one**. The firmware would not reject the UKI at the Secure Boot gate. The selection mechanism (systemd-boot's "highest version string" fallback when `LoaderEntryDefault` is unset, or a one-shot operator selection via Proxmox console, or NVRAM reset clearing `LoaderEntryDefault`) is what determines which `.efi` runs, and the answer is no longer guaranteed to be one of the two MID-prefixed expected UKIs.

**Two failure-mode classes for the unexpected UKIs**, based on the audit's observed `.pcrpkey`/`.pcrsig` presence:

| Forensic UKI class | Has `.pcrpkey`/`.pcrsig`? | Expected boot behaviour if selected |
|---|---|---|
| **Bare-kver / pre-hook B.1 development artifact** (e.g. `6.19.14-200.fc43.x86_64.efi`) | **No**: both sections absent | Outside the validated forward-sealed path. Firmware accepts (signed by `db.crt`); kernel boots; systemd-stub measures into PCR 11 normally (the extend operation does not require `.pcrsig` to be present); but the runtime PCR 11 will be a different value than any policy-signed prediction. **Expected to fail closed at LUKS unlock**: the policy chain has no signed authorisation for the measured PCR state. Operator sees a passphrase prompt. Fail-safe boundary holds, but the operator does not know whether to roll back or investigate (same symptom as a true Gate 6 failure). |
| **Test forensic UKI with embedded `.pcrsig`** (e.g. `test-tpm2initrd-pcrsig-*.efi`, `test-ukify-native-pcrsig-*.efi`) | **Yes**: both sections present | Same firmware-acceptance, same kernel boot, same measurement. **MAY unlock LUKS** if the embedded `.pcrpkey` matches the live policy public key the runtime policy chain expects. **This was not proven by the 2026-05-25 audit**: verification requires extracting `.pcrpkey` from each forensic UKI via `objcopy --dump-section .pcrpkey=/tmp/pcrpkey-$f.pem` (on a read-only copy in tmpfs per `06F` B.1) and comparing against `/etc/systemd/tpm2-pcr-public-key.pem`. If they match: silent boot substitution: the operator may not notice from LUKS behaviour alone that they booted a forensic artifact rather than the validated production UKI. If they don't match: same passphrase-prompt symptom as the bare-kver case. The verified facts are sbverify + signer count + VMA + section presence + db.crt signature. Whether they would actually unlock remains an open question. |

**This was NOT a B.2.4 Gate 6 blocker.** `bootctl status` at Gate 4, Gate 6A pre-reboot probe, and Gate 6B post-reboot validation all reported Default Entry and Current Entry pointing to the MID-prefixed rebuilt 6.19.14 UKI. `LoaderEntryDefault` correctly persisted across the reboot. The forensic UKIs were menu-visible but not selected. The audit-discipline scope is "boot-surface hygiene" + "preventive inventory contract" + "cleanup procedure", not a remediation of an active Gate 6 failure.

**Reference-run findings (2026-05-25 B.2.4 Gate 7 audit on VM 500).**

| File (under `/boot/efi/EFI/Linux/`) | Size (bytes) | mtime | sha256 | sbverify | signer_count | `.pcrpkey`/`.pcrsig` | VMA | Classification |
|---|---:|---|---|---|---:|---|---|---|
| `cf44500de…-6.19.14-200.fc43.x86_64.efi` | 56868424 | 2026-05-25 09:55:00 | `0ac32c8b…6949` | OK | 1 | present/present | sane | **expected (booted)** |
| `cf44500de…-6.17.1-300.fc43.x86_64.efi` | 56262216 | 2026-05-25 09:54:20 | `b8b15801…f994` | OK | 1 | present/present | sane | **expected (secondary)** |
| `6.19.14-200.fc43.x86_64.efi` (bare-kver) | 53059144 | 2026-05-07 14:59:48 | `bcfa6403…e097` | OK | 1 | **absent/absent** | sane | unexpected (B.1 development artifact) |
| `test-tpm2initrd-pcrsig-6.19.14-200.fc43.x86_64.efi` | 56429128 | 2026-05-08 03:10:44 | `b94b273e…c4ca` | OK | 1 | present/present | sane | unexpected (B.3 development artifact) |
| `test-ukify-native-pcrsig-6.19.14-200.fc43.x86_64.efi` | 53061192 | 2026-05-08 02:57:26 | `56e0aa68…d277` | OK | 1 | present/present | sane | unexpected (B.3 development artifact) |
| `cf44500de…-6.19.14-200.fc43.x86_64.efi.loaderror` | 53057096 | 2026-05-08 02:50 | (recorded in `06F` B.1) | n/a (not audited: excluded by `-iname '*.efi'` filter) | n/a | n/a | n/a | **correctly excluded** by `.loaderror` suffix (06F B.1 forensic-preservation convention working as designed) |

The B.1 `.loaderror`-suffixed file is the validated cleanup pattern: preserved on disk for forensic reference, made non-bootable by suffix renaming so systemd-boot does not enumerate it. The three unexpected files in the table above have not had that treatment applied. The audit recipe is read-only (no rename, no delete): cleanup is a separate gate.

**Read-only audit recipe.**

```bash
# Run on VM as root. READ-ONLY: no rename, no delete, no suffix change.
# Verified at B.2.4 Gate 7 (06B Step 37 Step 2).

EFI_DIR="/boot/efi/EFI/Linux"
DB_CRT="/etc/uefi-keys/db.crt"
VMA_THRESHOLD="0x180000000"
EXP_MID="0123456789abcdef0123456789abcdef"
EXP_KVER_BOOTED="6.19.14-200.fc43.x86_64"
EXP_KVER_SECONDARY="6.17.1-300.fc43.x86_64"
EXP_CURRENT_UKI_SHA="0ac32c8b98428f9a44339a5991eb86e2704986e84ce6e09ae9721bba41726949"

# Use -iname (case-insensitive): ESP is VFAT and case-insensitive at the filesystem level.
# Excludes .loaderror-suffixed forensic artifacts (B.1 convention).
mapfile -t EFI_FILES < <(find "$EFI_DIR" -maxdepth 1 -type f -iname '*.efi' -printf '%f\n' | sort)

# For each file: ls -la, stat, sha256sum, sbverify against db.crt, signer count via
# pesign --show-signature, .pcrpkey/.pcrsig presence via objdump -h, VMA sanity.
# Hard-fail only if the booted-kernel MID-prefixed UKI sha drifts from EXP_CURRENT_UKI_SHA.
# All other classifications (EXPECTED / UNEXPECTED) are informational at audit time.
```

The full canonical script is in `06B` Step 37 Step 2.

**Cleanup recipe (deferred to a separate gate: Block B.2.5).** Do not execute as part of the audit. The cleanup gate's full scope:

1. **Pre-cleanup snapshot** `module5-b2-5-pre-cleanup` on Proxmox host pve-host (parent `module5-b2-4-validated`).
2. **Read-only re-audit** of `/boot/efi/EFI/Linux/` to re-confirm the three known forensic UKIs are still the only unexpected files; halt if a new unexpected `.efi` has appeared since Gate 7.
3. **Per-file forensic record**: capture filename, sha256, size, mtime, sbverify, `.pcrpkey`/`.pcrsig` presence, VMA sanity (this is what the audit already captures; re-run to lock in the pre-cleanup state).
4. **Atomic rename** each unexpected firmware-visible artifact to a non-enumerated forensic suffix. **No deletion.** The rename preserves forensic value (the bits remain on disk; sha256 and contents are unchanged) while removing systemd-boot enumeration risk (`*.forensic-b1` and `*.forensic-b3` do not match `*.efi` for systemd-boot's BLS Type #2 auto-enumeration). Use `mv -T` for atomic rename within the same filesystem (ESP is single-volume VFAT).
5. **Reboot is NOT required** unless the cleanup touches the current/default entries, which it must not (the cleanup targets only the *unexpected* artifacts; the two MID-prefixed expected UKIs are untouched).
6. **Verify `bootctl status`** still reports Default Entry and Current Entry pointing to the MID-prefixed rebuilt UKI after the rename. The `LoaderEntryDefault` NVRAM variable references the file by basename; the rename changes the basename of artifacts that were NOT the default, so `LoaderEntryDefault` is not affected.
7. **Post-cleanup snapshot** `module5-b2-5-validated` on Proxmox host pve-host (parent `module5-b2-5-pre-cleanup`).

**Preferred suffix strategy** (preserves provenance in the suffix):

| Original path | Suffixed path |
|---|---|
| `/boot/efi/EFI/Linux/6.19.14-200.fc43.x86_64.efi` | `/boot/efi/EFI/Linux/6.19.14-200.fc43.x86_64.efi.forensic-b1` |
| `/boot/efi/EFI/Linux/test-tpm2initrd-pcrsig-6.19.14-200.fc43.x86_64.efi` | `/boot/efi/EFI/Linux/test-tpm2initrd-pcrsig-6.19.14-200.fc43.x86_64.efi.forensic-b3` |
| `/boot/efi/EFI/Linux/test-ukify-native-pcrsig-6.19.14-200.fc43.x86_64.efi` | `/boot/efi/EFI/Linux/test-ukify-native-pcrsig-6.19.14-200.fc43.x86_64.efi.forensic-b3` |

The `.forensic-b1` and `.forensic-b3` suffixes follow the precedent set by B.1's `.loaderror` (informational, parsable, distinguishable, ignored by systemd-boot). `-b1` and `-b3` record which Module 5 block produced the artifact (B.1 pre-hook bare-kver builds; B.3 hook live-validation `test-*` variants), useful for forensic reconstruction without needing to read mtime.

**Preventive ESP-inventory invariant (staged for future Gate-6A-class pre-reboot probes).** Until B.2.5 closes, the pre-reboot probe contract for any subsequent reboot gate (B.4 Gate 6, B.5 Gate 6, kernel-upgrade validation Gate 6, future Module 4 gates) must:

1. Enumerate `/boot/efi/EFI/Linux/*.efi` case-insensitively (matches the audit recipe).
2. Require the two expected MID-prefixed UKIs to be present.
3. **Allow the three known forensic artifacts** (`6.19.14-200.fc43.x86_64.efi`, `test-tpm2initrd-pcrsig-…`, `test-ukify-native-pcrsig-…`) but not require them.
4. **Hard-fail** on any *additional* unexpected `.efi` file outside this documented allowlist.

After B.2.5 closes, the allowlist drops to zero and the rule becomes the simpler hard form: exactly the MID-prefixed expected UKIs, zero unexpected `.efi`. Sketch:

```bash
# Pre-cleanup form (staged invariant, in force until B.2.5 closes)
EXP_MID="0123456789abcdef0123456789abcdef"
EXP_KVERS="6.19.14-200.fc43.x86_64 6.17.1-300.fc43.x86_64"
KNOWN_FORENSIC="6.19.14-200.fc43.x86_64.efi test-tpm2initrd-pcrsig-6.19.14-200.fc43.x86_64.efi test-ukify-native-pcrsig-6.19.14-200.fc43.x86_64.efi"

ALLOWLIST=""
for k in $EXP_KVERS; do ALLOWLIST="$ALLOWLIST ${EXP_MID}-${k}.efi"; done
ALLOWLIST="$ALLOWLIST $KNOWN_FORENSIC"

UNEXPECTED=""
while IFS= read -r f; do
    found=0
    for allowed in $ALLOWLIST; do
        [ "$f" = "$allowed" ] && { found=1; break; }
    done
    [ "$found" -eq 0 ] && UNEXPECTED="$UNEXPECTED $f"
done < <(find /boot/efi/EFI/Linux -maxdepth 1 -type f -iname '*.efi' -printf '%f\n')

[ -z "$UNEXPECTED" ] || fail XXX "unexpected ESP UKIs outside allowlist:$UNEXPECTED"

# Post-cleanup form (drops the KNOWN_FORENSIC inclusion; B.2.5 close converts to this)
# UNEXPECTED contains anything NOT matching ${EXP_MID}-*.efi
```

**Operational consequence of NOT auditing.** Without the audit-discipline contract, a future operator (or attacker with NVRAM write access) could clear `LoaderEntryDefault` and rely on systemd-boot's fallback to select a forensic UKI. Two outcomes follow from the section table above: the bare-kver UKI would fail closed at LUKS unlock (operator sees passphrase prompt: fail-safe but visually identical to a real Gate 6 failure); either `test-*-pcrsig-*` UKI might or might not unlock (boot-time silent substitution remains an open question pending the `.pcrpkey` extraction comparison described above). Both outcomes are within the trust chain's fail-safe envelope (no compromise of LUKS keys, no compromise of policy keys, no bypass of Secure Boot), but the bare-kver outcome creates operator-attention noise and the `test-*` outcome may create silent operational ambiguity.

**Cross-references.**

- `06F` B.1: `.loaderror` forensic-preservation convention. Validates the suffix-rename cleanup pattern; `06B` B.2.5 follows the same precedent at semantic scale (`.forensic-b1` / `.forensic-b3` instead of `.loaderror`).
- `06F` I.2: `bootctl status` parsing for Default Entry / Current Entry. The pre-reboot probe's bootctl assertion remains the operational guard regardless of ESP inventory state; even if a forensic UKI were selected by systemd-boot fallback, `bootctl get-default` would (incorrectly, per I.2) appear silent, but `bootctl status` parsing would report the unexpected Default Entry. The L.1 ESP-inventory check is therefore complementary to the I.2 bootctl-status parsing guard, not redundant.
- `06B` Step 37 Step 2: the canonical read-only audit recipe.
- `06B` Step 37 closure: the staged DNF discipline lift; ESP cleanup is explicitly scheduled as B.2.5, not folded into Module 4.
- `05_Update_Workflows_and_Key_Storage.md` §3.4: the operational-policy framing that includes the pre-reboot ESP-inventory invariant cross-reference.
- `00_Current_Project_State.md` Next session priorities: B.2.5 listed as the next near-term gate.

**Evidence:** 2026-05-25 B.2.4 Gate 7 read-only audit run on VM 500 (per-file forensic record above; full reference-run output captured in the Obsidian B.2.4 closure attempt-log subtree).

---

### L.2: `/boot/efi/EFI/BOOT/BOOTX64.EFI` is RPM-path-owned by `shim-x64` even though its content is our sbsigned systemd-boot

**Verdict:** CONTEXT-ONLY (latent RPM-path-ownership conflict; legacy-cleanup scope; **explicitly NOT in B.4 scope**).

**Symptom:**
`rpm -qf /boot/efi/EFI/BOOT/BOOTX64.EFI` returns `shim-x64-15.8-3.x86_64` (rc=0), despite the file's content being `sbsign`-signed systemd-boot (db.crt-signed; signer_count=1; CN `tboot-lab Signature Database Key`; byte-identical to `/EFI/systemd/systemd-bootx64.efi` at sha256 `73b0c6c4da8386a812749de088a442269a2f6869974d7f03e87643d52a449f41`).

**Root cause:**
Fedora's default install placed shim at `/EFI/BOOT/BOOTX64.EFI` and recorded the path in `shim-x64`'s RPM filelist (`BASENAMES`/`DIRNAMES` tags). The `06B` Step 9 transition overwrote the *content* via `bootctl install` + `sbsign` + `mv -f`, but did **not** unbind the RPM path-ownership. RPM still considers `shim-x64` the owner of that path; `rpm -V shim-x64` will eventually flag the file as `S.5....T.` (size, MD5 digest, mtime drift) on its next verify run.

The latent operational consequence: `dnf reinstall shim-x64` or `dnf upgrade shim-x64` would restore Microsoft-CA-signed shim bytes at `/EFI/BOOT/BOOTX64.EFI`, breaking the fallback-loadability invariant under our enforced PK/KEK/db. Under our keys, the result is **fail-closed** (firmware refuses to load Microsoft-CA-signed shim): operationally a surprise, not a security compromise.

**Reproduction (read-only):**
```bash
rpm -qf /boot/efi/EFI/BOOT/BOOTX64.EFI
# → shim-x64-15.8-3.x86_64  (rc=0)
sha256sum /boot/efi/EFI/BOOT/BOOTX64.EFI
# → 73b0c6c4…9f41  (= /EFI/systemd/systemd-bootx64.efi, our sbsigned content)
sbverify --cert /etc/uefi-keys/db.crt /boot/efi/EFI/BOOT/BOOTX64.EFI
# → Signature verification OK  (our db.crt, NOT Microsoft's UEFI CA)
```

**Fix or workaround:**
Treat as legacy-residue-cleanup-scope, not B.4 scope. B.4 does not manage `shim-x64`; B.4's manifest does not include `shim-x64` NEVRA; B.4's trigger filter (`package_filter=systemd-boot-unsigned`) does not match shim transactions. Resolution belongs to Module 4A. Current preferred direction (not yet binding):

1. Preserve `/EFI/BOOT/BOOTX64.EFI` as signed systemd-boot to retain UEFI removable-media fallback compatibility.
2. Accept and explicitly document the RPM-ownership conflict in `00_Current_Project_State.md`.
3. Hold the `dnf upgrade shim*` / `dnf reinstall shim*` discipline gate until Module 4A explicitly closes it (or until the conflict is resolved by path remap, alternate filename, or shim-package mask).

**Operational consequence right now:** the `dnf upgrade shim*` / `dnf reinstall shim*` discipline-gate row is added to `00_Current_Project_State.md`. Operators must reject `-y` on any DNF resolution containing `shim*` packages until Module 4A closes.

**Cross-reference:**
- `00_Current_Project_State.md`: discipline-gates table (`dnf upgrade shim*` row).
- `04A_Legacy_Residue_Cleanup_Gate.md`: full cleanup-gate scope.
- `06B` Step 9: original shim → systemd-boot transition where path-ownership was inherited.
- L.4: `/EFI/BOOT/` directory-level invariant for the cleanup gate.

**Evidence:** 2026-05-25 B.4 Gate 1 (rev-2) Block F output on VM 500.

---

### L.3: `/EFI/systemd/` directory invariant for B.4

**Verdict:** CONTEXT-ONLY (B.4-scope target invariant; enforced by B.4 helper post-Gate-2).

**What it specifies:**
Under the B.4 chain, the `/boot/efi/EFI/systemd/` directory must contain exactly one file: `systemd-bootx64.efi`. The file must be db.crt-signed with `pesign --show-signature` reporting `signer_count=1` and CN `tboot-lab Signature Database Key`. The file's sha256 must match the content of `/usr/lib/systemd/boot/efi/systemd-bootx64.efi.signed` (modulo any per-write metadata that `bootctl install/update` may introduce: empirically validated at B.4 Gate 2). Zero unexpected `.efi` files in this directory.

**Hard form from day one** (no allowlist). At B.4 Gate 1 (2026-05-25 rev-2 Block F) the directory was already clean: exactly `systemd-bootx64.efi`, no other entries.

**Why it matters:**
The `/EFI/systemd/` path is the primary bootloader location enumerated by `bootctl status` and referenced by the active `Linux Boot Manager` (0x0002) NVRAM entry. Any unexpected file here is either a B.4 helper bug or external tampering. Both warrant investigation, not allowlisting.

**Reproduction (read-only):**
```bash
ls -la /boot/efi/EFI/systemd/
find /boot/efi/EFI/systemd -maxdepth 1 -type f -iname '*.efi' -printf '%f\n'
```

**Expected output (post-B.4-Gate-2):**
```
systemd-bootx64.efi
```

**Cross-reference:**
- B.4 Gate 2 design freeze (when drafted).
- L.4: `/EFI/BOOT/` directory-level invariant for the cleanup gate.

**Evidence:** 2026-05-25 B.4 Gate 1 (rev-2) Block F output on VM 500: directory already in the target state pre-B.4.

---

### L.4: `/EFI/BOOT/` directory invariant: cleanup-scope target with pre-cleanup allowlist for `fbx64.efi`

**Verdict:** CONTEXT-ONLY (legacy-cleanup-scope target invariant; **explicitly NOT B.4 scope**).

**What it specifies:**
After Module 4A (Legacy Residue Cleanup Gate) closes, the `/boot/efi/EFI/BOOT/` directory must contain exactly one `.efi` file: `BOOTX64.EFI`, byte-matching `/EFI/systemd/systemd-bootx64.efi`, db.crt-signed with `signer_count=1`. Zero unexpected `.efi` files.

**Pre-cleanup state (current, 2026-05-25):** `fbx64.efi` (Microsoft-CA-signed shim fallback bridge, mtime 2024-03-19, size 87816 bytes) is **allowlisted** as known legacy residue inherited from the original Fedora install. The pre-Module-4A-close form of the invariant is:

```
exactly BOOTX64.EFI  +  optionally fbx64.efi  +  zero other .efi files
```

After Module 4A closes (rename `fbx64.efi` to `.forensic-shim-fbx64` per the B.1 `.loaderror` precedent), the allowlist drops to zero.

**Why `fbx64.efi` is not B.4 scope:**
- Not in our trust chain (custom PK/KEK/db → signed systemd-boot → signed UKI → TPM/LUKS unlock).
- Microsoft-CA-signed → firmware rejects under enforced PK/KEK/db → fail-closed.
- B.4 signs systemd-boot only. Widening B.4 to manage shim's fallback bridge violates scope (the same mission-creep failure mode that led to rev-1 Option A rejection).

**Cleanup precedent:** rename-with-suffix mirrors B.1 (`.loaderror`) and B.2.5 (`.forensic-b1` / `.forensic-b3`). No deletion. Preserves forensic value while removing systemd-boot enumeration risk and firmware fallback risk.

**Reproduction (read-only):**
```bash
ls -la /boot/efi/EFI/BOOT/
find /boot/efi/EFI/BOOT -maxdepth 1 -type f -iname '*.efi' -printf '%f  size=%s  mtime=%TY-%Tm-%TdT%TH:%TM:%TSZ\n'
```

**Pre-cleanup expected output (current):**
```
BOOTX64.EFI   size=135752  mtime=2026-05-07T15:03:32Z
fbx64.efi     size=87816   mtime=2024-03-19T01:00:00Z
```

**Post-cleanup expected output (after Module 4A):**
```
BOOTX64.EFI                 size=135752  mtime=2026-05-07T15:03:32Z
fbx64.efi.forensic-shim-fbx64  size=87816   mtime=2024-03-19T01:00:00Z   (renamed; not .efi-enumerated by systemd-boot or firmware)
```

**Cross-reference:**
- `04A_Legacy_Residue_Cleanup_Gate.md`: full cleanup-gate procedural sketch.
- L.2: `/EFI/BOOT/BOOTX64.EFI` RPM-path-ownership context.
- L.5: NVRAM `Boot0008 Fedora` + `/EFI/fedora/` residue companion finding.
- B.1: `.loaderror` suffix precedent.

**Evidence:** 2026-05-25 B.4 Gate 1 (rev-2) Block F output on VM 500 (pre-cleanup state with `fbx64.efi` allowlisted).

---

### L.5: NVRAM `Boot0008 Fedora` and `/boot/efi/EFI/fedora/` are dormant Fedora-default-install residue

**Verdict:** CONTEXT-ONLY (legacy-cleanup-scope; **explicitly NOT B.4 scope**).

**Symptom:**
`bootctl status` reports two active boot loaders in EFI Variables:
```
Title: Linux Boot Manager  ID: 0x0002  Status: active, boot-order
       File: /boot/efi//EFI/systemd/systemd-bootx64.efi
Title: Fedora              ID: 0x0008  Status: active, boot-order
       File: /boot/efi//EFI/fedora/shimx64.efi
```
`/EFI/fedora/` directory was not enumerated in B.4 Gate 1 (out of scope: Gate 1 inventoried `/EFI/systemd/` and `/EFI/BOOT/`); expected contents are Microsoft-CA-signed `shimx64.efi` plus supporting files (GRUB config, BLS Type #1 entries).

**Root cause:**
The original Fedora install placed shim at `/EFI/fedora/shimx64.efi` and registered the matching `Boot####` NVRAM entry. The `06B` Step 9 transition created the new `Linux Boot Manager` entry (0x0002) and promoted it to BootOrder priority, but did not remove the dormant `Fedora` entry. Under our enforced PK/KEK/db, the target binary `/EFI/fedora/shimx64.efi` is non-loadable (Microsoft-CA-signed) → if firmware ever falls through to `Boot0008`, it fails closed.

The entry is **dormant**, not active: BootOrder priority puts `Linux Boot Manager` (0x0002) first, and firmware only progresses to subsequent entries on prior-entry failure. Under our keys, if `Linux Boot Manager` fails, so does `Fedora` (shim is non-loadable). The security envelope holds.

**Why it is not B.4 scope:**
- Not in our trust chain.
- Firmware-non-loadable → fail-closed → not a security compromise.
- NVRAM mutation (`efibootmgr -B -b 0008`) is an EFI variable mutation, not a bootloader-signing operation. B.4 owns only signing automation.

**Reproduction (read-only):**
```bash
efibootmgr -v
ls -la /boot/efi/EFI/fedora/ 2>/dev/null
find /boot/efi/EFI/fedora -maxdepth 1 -type f -iname '*.efi' \
  -printf '%p  size=%s  mtime=%TY-%Tm-%TdT%TH:%TM:%TSZ\n' 2>/dev/null
```

**Fix or workaround:**
Treat as legacy-residue-cleanup-scope. Module 4A's cleanup procedure (when drafted) includes:
1. `efibootmgr -B -b 0008` to remove the dormant `Fedora` NVRAM entry after verifying `Linux Boot Manager` (0x0002) is BootOrder-priority and reachable.
2. Optionally rename `/EFI/fedora/shimx64.efi` to `.forensic-fedora` (mirrors B.1 / L.4 / B.2.5 rename-with-suffix precedent) and `/EFI/fedora/grub.cfg` / BLS files to similarly non-enumerated suffixes; or leave them in-place since they are firmware-non-loadable already.

**Cross-reference:**
- `04A_Legacy_Residue_Cleanup_Gate.md`: full cleanup-gate procedural sketch.
- L.2: companion finding for `/EFI/BOOT/BOOTX64.EFI` RPM-path-ownership conflict.
- L.4: companion finding for `fbx64.efi` cleanup precedent.

**Evidence:** 2026-05-25 B.4 Gate 1 (rev-2) Block I `bootctl status` output on VM 500.

---

## M: Mentor-notes deprecations + packaging clarifications

The original mentor notes (`Mentor_notes_and_starting_setup.md` in the Obsidian tree) cover Phase 1 and 2 against a Debian-based reference implementation. They are useful for high-level approach (Secure Boot ownership, UKI architecture, forward-sealing concept) but contain specific procedures that do not translate to Fedora 43.

### M.1: Debian-specific package names and procedures

**Verdict:** FORBIDDEN to follow verbatim on Fedora.

**What's deprecated:** `apt`, `update-initramfs`, Debian package names like `systemd-boot-efi`.

**Why:** The lab is Fedora 43; Debian commands do not translate directly.

**Fedora equivalents:**

- `apt install` becomes `dnf install`.
- `update-initramfs -u` becomes `dracut --force` (or `kernel-install add` for full UKI rebuild).
- `systemd-boot-efi` package becomes `systemd-udev` plus `systemd-boot-unsigned` on Fedora.

The `06B` runbook uses Fedora commands throughout.

---

### M.2: Double signing with sbsign

**Verdict:** FORBIDDEN.

**What's deprecated:** running `sbsign` against an already-sbsigned artifact, producing a UKI with multiple Authenticode signatures.

**Why:** The validated hook's stage 10.3 enforces signer count == 1. Multiple signatures don't break boot per se, but they indicate something has appended a redundant signature, which is a signal of a workflow bug. The validated hook builds the UKI fresh via `ukify build` (which calls sbsign once internally), so there is no separate sbsign step needed.

**Correct approach:** trust ukify to handle Authenticode signing inside its build pipeline. Do not run a separate sbsign pass after `ukify build`.

---

### M.3: Cmdline UUID duplication

**Verdict:** FORBIDDEN.

**What's deprecated:** `/etc/kernel/cmdline` containing duplicated fields, e.g. `root=UUID=... rd.luks.uuid=... root=UUID=...`.

**Why:** The kernel command-line parser may take the last value, but the behavior is implementation-defined. Use unique fields.

**Correct form:**

```
root=UUID=<filesystem-uuid> ro rd.luks.uuid=luks-<luks-uuid> quiet
```

`06B` Step 7 generates this from `findmnt` and `blkid` output.

---

### M.4: `bootctl` manpage owned by `systemd-udev` on Fedora 43 (not `systemd`)

**Verdict:** CONTEXT-ONLY (packaging clarification; documentation only).

**Symptom:**
Operator looks for the `bootctl(1)` manpage source via `rpm -ql systemd | grep bootctl` and finds nothing. The manpage is present on the system (visible via `zgrep` on `/usr/share/man/man1/bootctl.1.gz`) but the main `systemd` package does not own it.

**Root cause:**
On Fedora 43, the systemd source RPM splits into multiple binary subpackages: `systemd` (PID 1, journald, core), `systemd-libs` (libsystemd, libudev), `systemd-udev` (udev daemon **plus the `bootctl` binary and its manpage**), `systemd-boot-unsigned` (the EFI binary and stubs), `systemd-ukify`, `systemd-pam`, `systemd-resolved`, `systemd-oomd-defaults`, `systemd-sysusers`, `systemd-shared`. Discovered at B.4 Gate 1 (rev-2 Block L manpage lookup): `rpm -ql systemd-udev` returns `/usr/share/man/man1/bootctl.1.gz`.

**Reproduction (read-only):**
```bash
for p in $(rpm -qa | grep -E '^systemd'); do
  rpm -ql "$p" 2>/dev/null | grep -E '/man[0-9]+/bootctl\.[0-9](\.gz)?$' \
    | while read -r m; do echo "${p}:${m}"; done
done | sort -u
# → systemd-udev-258.7-1.fc43.x86_64:/usr/share/man/man1/bootctl.1.gz
```

**Fix or workaround:**
Documentation correction only. When referencing "the bootctl manpage owner," say `systemd-udev` on Fedora 43, not `systemd`. Anyone reproducing the project on a different distribution should verify the bootctl-owning package locally: Debian/Ubuntu split this differently (typically `systemd-boot-tools`).

The `bootctl` binary itself may live in a different subpackage from its manpage depending on Fedora packaging revisions; verify both with `rpm -qf $(command -v bootctl)` and `rpm -qf /usr/share/man/man1/bootctl.1.gz` if it matters.

**Cross-reference:**
- B.4 Gate 1 rev-2 Block L (bootctl manpage discovery).
- M.1: companion Debian/Fedora package-name translation finding.

**Evidence:** 2026-05-25 B.4 Gate 1 (rev-2) Block L output on VM 500.

---

## N: Block B.4 (systemd-boot signing) findings

> [!note] Section-letter note
> During the 2026-05-28 working session these findings were referred to informally as `K.4`–`K.7` and `M.2`. They were assigned to a new section **N** on catalog entry: section K is already B.2.4 (Gates 1–3) findings, and M.2/M.4 are already taken (Debian double-signing / `bootctl`-manpage-owner). The F3 probe blocks `N.1`–`N.6` from the working session are unrelated command labels (a different namespace); these are the persisted finding IDs.

### N.1: `bootctl --no-variables update` skips a re-signed same-version binary; only `install` force-copies: use `install` for loader propagation

**Verdict:** CONTEXT-CRITICAL (design lock; wrong choice silently leaves the re-signed loader unpropagated).

**Symptom:**
After source-side signing of `systemd-bootx64.efi` into `systemd-bootx64.efi.signed`, the operator must propagate the signed binary onto the ESP at `/EFI/systemd/systemd-bootx64.efi` and `/EFI/BOOT/BOOTX64.EFI`. `bootctl --no-variables update` appears to be the natural verb but is a no-op here.

**Root cause:**
`bootctl update` compares the embedded `LoaderInfo` version string of the source against the installed copy and **skips when the version matches**: even when the byte content differs (a re-sign changes bytes but not `LoaderInfo`, and `sbsign` does not alter the version string). Re-signing the same systemd-boot release therefore produces a version-identical, byte-different binary that `update` refuses to copy (rc=1, message `"same boot loader version in place already"`). `bootctl install` force-copies unconditionally.

**Reproduction (read-only / loopback; does NOT touch `/boot/efi`):**
```bash
# loopback vfat ESP (relax-checks scratch dir is rejected; bootctl needs a real mountpoint)
truncate -s 64M /tmp/esp.img && mkfs.vfat /tmp/esp.img >/dev/null
mp=$(mktemp -d); mount -o loop /tmp/esp.img "$mp"
mkdir -p "$mp/EFI/systemd" "$mp/EFI/BOOT" "$mp/loader/entries"
# place a byte-different same-version binary, then:
bootctl --no-variables --esp-path="$mp" update   # rc=1: "same boot loader version in place already" (SKIP)
bootctl --no-variables --esp-path="$mp" install   # rc=0: force-copies to /EFI/systemd + /EFI/BOOT/BOOTX64.EFI
umount "$mp"; rmdir "$mp"; rm -f /tmp/esp.img
```

**Fix or workaround:**
The B.4 helper's propagation primitive is `bootctl --no-variables install`. The one open caveat: that `install` picks `*.efi.signed` over the unsigned source when both exist in `BOOTLIBDIR`, is closed at helper runtime by a mandatory per-run loopback checkpoint (sign → `.efi.signed` → checkpoint `install` to a throwaway loopback ESP → assert both targets == `src_signed` and ≠ `src_efi` → only then `install` to the real `/boot/efi`). The checkpoint is fail-closed: on mismatch the helper writes the sentinel + `helper-last-error` and exits non-zero without touching `/boot/efi`. Live exercise completed at B.5 Gate 3 (2026-05-28): the checkpoint ran for real and PASSED for both targets before any real-ESP write, the loader was signed to `.efi.signed` `4859f48a…4986` and propagated, and the signed loaders survived the reboot byte-identical (section O.3; `06B` Step 39).

**Cross-reference:**
- `06B` Step 38 (B.4 install/publish gate). B.5 (publish + first fire).
- N.2 (the loopback ESP needs a real mountpoint, not a relax-checks scratch dir).

**Evidence:** 2026-05-28 B.4 Gate 2 install-vs-update probe (P.2-loop) on VM 500.

---

### N.2: Read-back/probe harness hygiene: interactive `set -u`, history expansion, bad substitution, forbidden tokens in help prose, and multi-line presence checks

**Verdict:** CONTEXT-ONLY (harness-engineering discipline; none are script-source bugs).

**Symptom:**
Several distinct read-back-harness failures, all where the *harness or probe text* collided with shell or scan semantics rather than any defect in the artifact under test:
1. `set -u` at the **interactive** shell top level corrupts a custom `PS1`/`PROMPT_START` and breaks the prompt.
2. `!` inside an `echo` header triggers bash history expansion.
3. `${...}` literals in non-quoted echo text trigger bad-substitution / unbound-variable errors.
4. The literal token `bootctl` in a decider `--help` heredoc body tripped the forbidden-token scan: the scan only excludes lines beginning with `#`, so heredoc help prose is treated as runtime text.
5. A single-line `grep` presence check missed a token that the in-script self-scan (multi-line-aware) caught.

**Root cause:**
The forbidden-token scan is deliberately dumb-but-trustworthy: it greps non-`#` lines for a fixed token list. It does not parse heredocs, quoting, or comments-after-code. Probe wrappers run in the operator's interactive shell unless explicitly sandboxed.

**Fix or workaround:**
- Run `set -euo pipefail` blocks inside a child shell: `bash <<'EOF' ... EOF`, never at the interactive top level.
- `set +H` before any block whose echo headers contain `!`.
- Avoid `${` and forbidden-token literals (`bootctl`, `sbsign`, …) anywhere on a non-`#` line, **including heredoc help text**: describe capabilities in prose that avoids the literal tool name (e.g. the decider `--help` says "never invokes the loader installer" rather than naming the loader installer). This is the standing rule: forbidden-token names must not appear on any non-`#` line of a script, help text included.
- Presence checks must be multi-line-aware and exclude `#`-prefixed lines the same way the in-script self-scan does.

**Cross-reference:**
- Same class as the B.2.x harness corrections (`06F` H.4, I.4, J.1–J.3). This is the section-N continuation for the B.4 chain.
- `06B` Step 38; the decider help-text patch was applied before staging (sha `3ccef59c…f8e7`).

**Evidence:** 2026-05-28 B.4 Gate 2 authoring + read-back sessions on VM 500.

---

### N.3: `libdnf5-plugins` actions rule layout and direction semantics: `reinstall` matches both `in` and `out`; per-package command dedup is guaranteed

**Verdict:** CONTEXT-CRITICAL (rule-shape design lock).

**Symptom:**
The B.4 `.actions` rule must fire exactly once on `dnf reinstall systemd-boot-unsigned` and feed the decider. The `direction` field's behaviour for `reinstall` is not obvious from the package name alone.

**Root cause:**
Per `libdnf5-actions(8)`, an actions rule is `callback_name:package_filter:direction:options:command`. `direction=in` enumerates the inbound action set `{downgrade, install, reinstall, upgrade}`; `direction=out` enumerates `{remove, ...}`. A `reinstall` is internally a remove-then-install, so it matches **both** `in` and `out` filters. The plugin guarantees per-package command de-duplication, so a single rule with `direction=in` fires the command exactly once per matching package even though reinstall touches the package twice.

**Fix or workaround:**
The B.4 rule uses `direction=in` with `package_filter=systemd-boot-unsigned`:
```
post_transaction:systemd-boot-unsigned:in:enabled=host-only raise_error=1:/usr/local/sbin/tboot-sbloader-posttrans
```
`in` is correct for the reinstall driver (and also covers a future genuine `upgrade`/`install`). `raise_error=1` surfaces a decider non-zero exit as a transaction error. `enabled=host-only` scopes the rule to host transactions.

**Cross-reference:**
- N.4 (the `--cacheonly --assumeno` probe that confirmed reinstall resolution).
- `06F` K.2 (DNF5 `--assumeno` rc=1 refusal semantics: companion).
- `06B` Step 38; B.5 (rule publish + first fire).

**Evidence:** 2026-05-28 B.4 Gate 2 `libdnf5-actions(8)` manpage review (Q.1) on VM 500.

---

### N.4: `dnf reinstall --cacheonly --assumeno systemd-boot-unsigned` resolves as `Reinstalling: 1 package` then aborts rc=1: safe transaction-preview for the B.4 driver

**Verdict:** CONTEXT-ONLY (transaction-preview confirmation).

**Symptom:**
Need to confirm: without mutating the system or publishing the rule: that `dnf reinstall systemd-boot-unsigned` will resolve to a single-package reinstall (the intended B.4 first-fire driver) and that the decider stays silent across a preview.

**Root cause / behaviour:**
`dnf reinstall --cacheonly --assumeno systemd-boot-unsigned` resolves `Reinstalling: 1 package` and aborts at the consent prompt with rc=1 and the documented refusal phrase (`06F` K.2). `--cacheonly` avoids network metadata refresh; `--assumeno` declines at the prompt. No transaction runs; the decider's `post_transaction` callback is not invoked (the actions rule is not even published at this stage, and an aborted transaction has no post_transaction phase anyway).

**Fix or workaround:**
Treat this as the read-only preview for the B.5 first-fire driver. NEVRA `systemd-boot-unsigned-258.7-1.fc43.x86_64` confirmed reinstall-eligible (present in the `updates` repo). The actual fire happens in B.5 after the `60-` rule is published.

**Cross-reference:**
- N.3 (direction semantics). `06F` K.2 (assumeno rc=1). `06B` Step 38; B.5.

**Evidence:** 2026-05-28 B.4 Gate 2 direction probe (Q.2) on VM 500.

---

### N.5: `systemd-boot-random-seed.service` writes only `/boot/efi/loader/random-seed`: out of the B.4 boundary; do NOT mask it

**Verdict:** CONTEXT-CRITICAL (mask-list scoping; over-masking would break the loader random seed).

**Symptom:**
The B.4 mask decision (which units to mask so the native systemd updater cannot auto-replace the signed loader) must not be over-broad. Four units reference `bootctl`; only one writes loader binaries.

**Root cause:**
F3 supplemental read-only probe captured the bodies of all four `bootctl`-referencing units. `systemd-boot-clear-sysfail.service` (`bootctl --graceful set-sysfail ""`, condition-skipped every boot, never executed), `systemd-bootctl@.service` (bare `bootctl` = Varlink `io.systemd.BootControl` server, never instantiated), and `systemd-bootctl.socket` (`ListenStream=/run/systemd/io.systemd.BootControl`) do not install/update the loader or write `/EFI/systemd/`, `/EFI/BOOT/`, or `/usr/lib/systemd/boot/efi/`. `systemd-boot-random-seed.service` runs `bootctl --graceful random-seed`, which writes **only** `/boot/efi/loader/random-seed`: the loader's entropy pool, not a boot binary. Masking it would break the random-seed mechanism without any boot-chain integrity benefit.

**Fix or workaround:**
B.4 mask-list = `systemd-boot-update.service` **only** (the sole unit that runs `bootctl ... install/update` to replace the loader binary). The other four units are explicitly left unmasked. The B.4 install/publish gate verifies post-mask that exactly those four remain unmasked (`is-enabled` ≠ masked).

**Cross-reference:**
- `06F` L.3 (`/EFI/systemd/` invariant), L.4 (`/EFI/BOOT/` invariant). `06B` Step 38.

**Evidence:** 2026-05-28 B.4 Gate 1 F3 supplemental probe on VM 500.

---

### N.6: B.4 decider no-op (D-1) requires baseline-equality AND converged-shape: baseline-equality alone is insufficient

**Verdict:** CONTEXT-CRITICAL (decider correctness lock).

**Symptom:**
A decider that emits `decision=unchanged` whenever the current manifest equals the stored baseline would wrongly treat a stale-but-equal state as safe: e.g. after a source-package update that leaves a stale `.efi.signed`, or before the first convergence when `src_signed=ABSENT`.

**Root cause / design lock (D-1):**
The B.4 decider emits `decision=unchanged` **only if all four hold**: (1) manifest == baseline, (2) `src_signed` ≠ `ABSENT`, (3) `esp_systemd` == `src_signed`, (4) `esp_boot` == `src_signed`. Baseline-equality (condition 1) alone is insufficient; the converged-shape (conditions 2–4) must also hold. Baseline absent in normal mode is a hard FAIL CLOSED (no auto-prime, no helper invocation; the decider tells the operator to run `--prime` in the correct gate). After helper success the decider recomputes, requires converged-shape, advances the baseline, and clears the sentinel.

**Proof (live, 2026-05-28):**
The primed B.4 baseline is intentionally **non-converged**
(`src_signed=ABSENT`). It is a pre-convergence reference, not an unchanged safe
state. After priming, the current manifest equals the baseline (`equal=1`) but
the shape remains non-converged (`shape=0`). `--debug-print` therefore reports
`would-be decision: drift (equal=1 shape=0)`, never `unchanged`. This confirms
that condition 1 alone does not yield a no-op. If the post-prime result ever
reads `unchanged`, D-1 is broken.

**Fix or workaround:**
The decider source enforces D-1 in `evaluate()` / `_converged_shape()`. The install/publish gate asserts the live proof (`drift (equal=1 shape=0)` post-prime) as a hard check. The first genuine convergence event is the B.5 first-fire transaction.

**Cross-reference:**
- N.1 (install propagation primitive). `06B` Step 38 (Step 5 prime + D-1 proof). B.5 (first convergence).

**Evidence:** 2026-05-28 B.4 install/publish gate Step 5 on VM 500 (baseline `2547bd82…cc5e`, post-prime `--debug-print` = `drift (equal=1 shape=0)`).

---

## O: Block B.5 (arm / first-fire / converge / reboot) findings

### O.1: The B.4 decider emits two different decision-string forms: numeric `(equal=N shape=N)` on the drift branch, worded `unchanged (manifest==baseline AND converged-shape)` on the converged branch; matchers must accept both

**Verdict:** CONTEXT-CRITICAL (harness-correctness lock).

**Symptom:**
A Gate 4 convergence check that asserted the literal substring `shape=1` in the decider's `--debug-print` would-be-decision line FAILED after a genuinely-converged first-fire, even though every independent convergence signal (decider `src_signed == esp_systemd == esp_boot`, advanced baseline, `last-decision=helper-success`) passed. The system was fully converged; the matcher was wrong.

**Root cause:**
`tboot-sbloader-posttrans --debug-print` renders its would-be decision in two different shapes depending on the branch taken. On the **drift** branch it prints the numeric diagnostic form, e.g. `--- would-be decision: drift (equal=1 shape=0) ---`. On the **converged / no-op (D-1)** branch it prints the worded form `--- would-be decision: unchanged (manifest==baseline AND converged-shape) ---`. There is no `equal=1 shape=1` numeric form on the converged path; expecting one assumes a symmetry the decider does not implement.

**Fix or workaround:**
Parse the decision token and branch on its text, accepting both forms:

```bash
dl="$(printf '%s\n' "$dp" | sed -n 's/.*would-be decision:[[:space:]]*//p' | sed 's/[[:space:]]*---[[:space:]]*$//')"
case "$dl" in
  unchanged*converged-shape*) ok "converged decision";;
  drift*)                     no "STILL DRIFT: $dl";;
  *)                          no "unexpected decision form: ${dl:-<none>}";;
esac
```

The canonical Gate 4 in `06B` Step 39 carries this dual-form matcher; the pre-arm Gate 0 / Gate 2 checks (which run only against the non-converged primed/armed state) correctly use the numeric `equal=1 shape=0` form because they never observe the converged branch.

**Cross-reference:**
- N.6 (D-1 no-op semantics: the converged branch this finding observes). `06B` Step 39 (Gate 4 A3).

**Evidence:** 2026-05-28 B.5 Gate 4 on VM 500: initial A3 `shape=1` matcher FAIL against converged state; corrected dual-form matcher PASS (`unchanged (manifest==baseline AND converged-shape)`).

---

### O.2: A UKI rebuilt by the co-firing B.2 helper advances its whole-file sha256 while the predicted/runtime PCR 11 stays byte-identical: the file hash and the measurement track different things

**Verdict:** CONTEXT-CRITICAL (do not treat a UKI sha change as a PCR 11 change).

**Symptom:**
During the B.5 first-fire, the B.2 helper rebuilt the booted-kernel UKI; its file sha256 advanced (`0ac32c8b…6949` → `1187c78c…b843`) yet the stored PCR 11 prediction and the runtime PCR 11 both remained `28E66CE2…C90F`. A naive check equating "UKI sha changed" with "boot measurement changed" would wrongly predict a passphrase prompt on reboot.

**Root cause:**
PCR 11 is computed by `systemd-measure` over the UKI's *measured-content sections* (`.linux`, `.initrd`, `.cmdline`, `.osrel`, `.uname`, `.sbat`, `.pcrpkey`). When those section payloads are bit-identical across a rebuild (same vmlinuz, same cmdline, same os-release, same dracut output, same policy pubkey), PCR 11 is identical by construction. The whole-file sha256 still differs because the PE+ header carries a fresh `TimeDateStamp`, the Authenticode signature is re-applied (different signing-time), and RSA-PSS `.pcrsig` signing uses a random salt: none of which are measured into PCR 11. This is the same axis-separation documented for the B.2.4 dracut-network case.

**Fix or workaround:**
Validate PCR 11 against the stored prediction and the runtime sysfs value; validate UKI *identity* against its own RE-PINned sha; never assert that the two move together. The no-passphrase reboot is the operational proof that the measured content was stable.

**Cross-reference:**
- E.3 (PCR 11 invariance across UKI regeneration). K.3 (baseline-sha vs PCR-11 advance on different axes). O.3 (the co-fire that triggered the rebuild). `06B` Step 39 (Gate 4 / Gate 6b).

**Evidence:** 2026-05-28 B.5 Gate 4 + Gate 6b on VM 500: booted UKI `1187c78c…b843`, runtime+stored PCR 11 `28E66CE2…C90F`, no-passphrase TPM2 unlock across reboot.

---

### O.3: `dnf reinstall systemd-boot-unsigned` triggers BOTH the broad B.2 (`50-`) chain and the narrow B.4 (`60-`) chain: the two-chain co-fire is expected; B.2 fires first and rebuilds UKIs, then B.4 converges systemd-boot signing

**Verdict:** CONTEXT-CRITICAL (B.5 model correction; not a fault).

**Symptom:**
The original B.5 plan narrated only the B.4 decider/helper sequence for the first-fire. In practice the `dnf reinstall systemd-boot-unsigned` transaction fired the B.2 chain as well, rebuilding initramfs and re-signing both kernels' UKIs and refreshing the PCR 11 prediction file: additional mutation beyond the systemd-boot loader.

**Root cause:**
`50-tboot-posttrans.actions` (B.2) has an **empty** `package_filter`, so it matches every transaction; `60-tboot-sbloader.actions` (B.4) is filtered to `systemd-boot-unsigned`. Because `systemd-boot-unsigned` is boot-input-relevant, the B.2 decider correctly detects drift and runs its helper (dracut + kernel-install + `80-tpm2-sign` per kernel) before the B.4 chain runs. Both deciders are independent, both fail-closed, both converge. The PCR 11 prediction landed unchanged (`28E66CE2…C90F`) because the measured-content inputs were stable (O.2).

**Fix or workaround:**
Accept the co-fire as designed behaviour and validate **both** convergence stories at the post-fire gate: B.4 (signed `.efi.signed`, both ESP loaders == signed, baseline converged) and B.2 (`last-computed-manifest` sha == baseline sha, `last-decision=helper-success`, sentinels clear). RE-PIN the values that advanced during the transaction: they supersede the pre-B.5 pins:

| Artifact | Pre-B.5 (superseded) | RE-PINned at B.5 (authoritative) |
|---|---|---|
| B.2 boot-input baseline | `6b032bd8…2a47` | `47903d22f4dc76a34e1a459d7fba67ee1b3b4150a4058f426b974aa7c44c220d` |
| booted 6.19.14 UKI | `0ac32c8b…6949` | `1187c78ce2435e0a5c27ed8f35e0fd653da3580ac7a2597b03a30bbfd5deb843` |
| secondary 6.17.1 UKI | `b8b15801…8994` | `ac9c50903a370b9f9b79789181ef14f685a620294c56a7d26aacc5dfda78b5e7` |
| systemd-boot signed loader (`.efi.signed` + both ESP) | (absent pre-B.5) | `4859f48ac4d0776751e21cefc39cd42017ff496153dc8ed3577e2b0fdb114986` |
| PCR 11 prediction | `28E66CE2…C90F` | `28E66CE2…C90F` (unchanged: O.2) |

The B.2.5 ESP forensic-UKI residue (L.1: bare-kver `6.19.14…efi`, `test-tpm2initrd-pcrsig-…`, `test-ukify-native-pcrsig-…`) is unaffected by B.5 and remains deferred to Module 4 / B.2.5; B.5 Gate 6b deliberately validated the booted/Current/Default entry rather than asserting ESP `.efi` file count.

**Cross-reference:**
- N.3 (`50-` empty-filter matches all; `60-` filtered to `systemd-boot-unsigned`). O.1 (decider decision forms). O.2 (UKI-sha vs PCR-11 axes). L.1 (ESP residue, out of scope). `06B` Step 39 (Gate 3 / Gate 4).

**Evidence:** The 2026-05-28 B.5 Gate 3 transaction log on VM 500 shows
`tboot-dnf-posttrans` and `tboot-dnf-helper` (B.2) firing first, followed by
`tboot-sbloader-posttrans` and `tboot-sbloader-helper` (B.4). Both chains
completed successfully, confirming that the co-fire is expected behavior.

---

## Forbidden procedures: quick reference

This table is the fast-lookup version of the FORBIDDEN verdicts above. If you find yourself about to do any of these, stop and read the linked finding.

| If you're tempted to... | Don't, because... | Do this instead | Finding |
|---|---|---|---|
| Use `objcopy --add-section` to add `.pcrsig` to a UKI | Produces firmware-rejected PE+ | `ukify build --pcr-private-key= --pcr-public-key= --phases=enter-initrd ...` | B.1 |
| Validate a hook by checking only sections + sbverify | Misses VMA layout corruption | Add per-section sha256 vs reference, plus VMA sanity check | B.1 (operational consequences) |
| Roll back to `module5-kernel-signing-hook-installed` | Broken hook on canonical path | Roll back to `module5-kernel-signing-hook-validated` | B.1 (operational consequences) |
| Boot the objcopy-built forensic UKI | Will fail with `LoadImage()` Load Error | Use it for forensic reproduction only, with `.loaderror` suffix | B.1 (operational consequences) |
| Use `ukify --join-pcrsig` | Silent no-op in systemd 258.7 | Build UKI fresh with native PCR signing flags | B.2 |
| Sign Fedora kernels and chainload via BLS Type #1 | systemd-boot doesn't read `/boot/loader/entries/` | UKI-first: build UKI, place at `/boot/efi/EFI/Linux/${KVER}.efi` | A.1 |
| Put `[Signing]` section in `uki.conf` | Not a real ukify section, silently ignored | Put SecureBoot* keys in `[UKI]` section | B.4 |
| Run `apt`, `update-initramfs` from mentor notes | Debian-specific, doesn't apply to Fedora | Use `dnf` and `dracut --force` (or `kernel-install add`) | M.1 |
| Re-sbsign an already-signed UKI | Produces 2 Authenticode signers, fails the hook's signer-count check | Trust `ukify build` to sign once | M.2 |
| Duplicate `root=UUID=` fields in cmdline | Behavior is parser-implementation-defined | Single `root=` and single `rd.luks.uuid=` per cmdline | M.3 |
| Assert the literal `shape=1` in the B.4 decider's converged `--debug-print` decision line | The decider prints `drift (equal=N shape=N)` only on the drift branch; the converged branch prints the worded `unchanged (manifest==baseline AND converged-shape)`: there is no `shape=1` numeric form | Parse the decision token and branch on text (`case` accepting `unchanged*converged-shape*` and `drift*`) | O.1 |
| Assert `lock path is absent` after a clean decider/helper exit | `flock(1)` releases on fd close but does not unlink; the file is expected to persist across invocations. Path absence is therefore an invalid invariant. | Use the `flock <> -n` probe pattern with symlink rejection: `if [ -e "$LOCK" ] \|\| [ -L "$LOCK" ]; then [ -f "$LOCK" ] && [ ! -L "$LOCK" ] \|\| fail; ( exec 9<>"$LOCK"; flock -n 9 ) \|\| fail; fi` | J.3 |
| Assert `Loaded libdnf plugin "actions"` in `/var/log/dnf5.log` as Gate 6A evidence | The file does not log plugin-load entries in this Fedora 43 environment | Use `journalctl -q -t tboot-dnf-posttrans --since "$PRE_TIME"` as authoritative evidence; assert `exit reason:` line count + `last-decision` epoch advancement | J.1 |
| Lexicographically compare `date -Iseconds` output against `/var/log/dnf5.log` timestamps | TZ-offset formats are incompatible (`+02:00` vs `+0200`) and dnf5.log is not the authoritative source anyway | Use `journalctl --since` (handles `+02:00`-format input natively via systemd's time parser) | J.2 |
| Clear `LoaderEntryDefault` (NVRAM reset, manual `bootctl set-default ""`, or implicit via firmware reset) on a system with unexpected `.efi` files in `/boot/efi/EFI/Linux/` | systemd-boot's "highest version string" fallback may select a db.crt-signed forensic UKI that is firmware-loadable; the bare-kver variant fails closed at LUKS (passphrase prompt, fail-safe but visually identical to a real Gate 6 failure); the `test-*-pcrsig-*` variants may unlock if their embedded policy pubkey matches the live key (silent substitution; not proven by audit) | First execute the B.2.5 ESP forensic UKI cleanup gate (rename to `.forensic-b1` / `.forensic-b3` suffixes; no deletion). Until B.2.5 closes, do not clear `LoaderEntryDefault`. | L.1 |
| Delete forensic UKIs on first cleanup pass | Loses forensic value (sha + size + mtime + section structure no longer recoverable); over-conservative pattern would prefer rename-with-suffix first, evaluate retention later | Rename to non-enumerated forensic suffix (`.forensic-b1` for pre-hook B.1 artifacts; `.forensic-b3` for B.3 development variants); deletion is a separate later decision after retention period elapses | L.1 |
| Skip the read-only ESP audit before any high-risk reboot gate (B.4 / B.5 / kernel-upgrade / Module 4) | A new firmware-loadable artifact may have appeared on the ESP since the previous validated state; without the audit there is no contract that catches this drift before reboot | Pre-reboot probe must enumerate `/boot/efi/EFI/Linux/*.efi` case-insensitively, require the two expected MID-prefixed UKIs, allow the three known forensic artifacts only until B.2.5 closes, fail on any additional unexpected `.efi` | L.1 |

## Conditional and clarification entries: quick reference

| Topic | Clarification | Finding |
|---|---|---|
| "DNF holds in effect" in snapshot descriptions | Operator discipline only, not machine-enforced | E.2 |
| "The canonical ESP UKI path" | Differs between Phase 1 manual UKI and kernel-install-produced UKI | E.1 |
| `objcopy` use on UKIs | Safe for read-only inspection on a copy; unsafe for production write paths | B.1 (objcopy: safe vs unsafe) |
| Empty `[PCRSignature:initial]` block | Not a bug; intended config slot for hook activation | D.3 |
| Interactive prompt inside heredoc | Stdin is the consumed heredoc; redirect `</dev/tty` for prompts | F.5 |
| `systemd-cryptenroll --tpm2-signature=` | Validates against current PCR state, not the signed phase; omit if hook signs `enter-initrd` | G.1 |
| `systemd-cryptenroll --tpm2-public-key-pcrs=` | Must exactly match PCR list in `.pcrsig`; split policy via `--tpm2-pcrs=` for static PCRs | G.1 |
| `kernel-install add` initramfs propagation | Does not refresh `/boot/initramfs` under UKI layout; run `dracut --force` first | G.2 |
| dnf5 runtime verbosity | No `--debuglevel`, no `-v`, no `--verbose`; only `--quiet` and `--debugsolver`. Read `/var/log/dnf5.log` deltas instead | H.1 |
| `dnf reinstall` as test driver | Requires installed NEVRA in repos; brittle on long-lived systems. Use fresh `dnf install` of a not-installed candidate | H.2 |
| `grep -Ei` against dnf5 transaction output | `boot` substring matches `inbound` header. Use `\b(term)([-_].*)?\b` word-boundary form | H.3 |
| External audit-harness token-boundary regex | Must match the in-script self-scan (Deviation D: see H.4 body for the regex in a fenced code block). Do not patch staged sources to silence harness false positives | H.4 |
| PCR 11 value identical after helper regeneration | Expected when measured-content inputs are bit-identical; assert on mtime advancement instead of value change | E.3 |
| `qm listsnapshot` parsing | Strip tree-glyph prefixes (`` `-> ``, `-> `) before column-1 awk match; ignore cosmetic `Wide character in printf` warnings from PVE Perl module | I.1 |
| `bootctl get-default` empty on Fedora 43 VM | Treat as non-authoritative; parse `bootctl status` for `Default Entry:` and `Current Entry:` instead | I.2 |
| Fedora 43 `/usr/local/sbin -> bin` UsrMerge | Resolve via `readlink -f`, assert resolved literal == `/usr/local/bin`, atomic-publish inside resolved real directory (`mktemp` + `install` + `mv -T`); canonical user-facing path remains `/usr/local/sbin/tboot-dnf-posttrans` | I.3 |
| `journalctl` `-- No entries --` advisory breaks emptiness tests | Use `journalctl -q` for tag-silence assertions | I.4 |
| `--self-test` does not emit `ok: helper present and executable` | Mode-dependent: helper-presence branch is warn-only in self-test. Verify helper presence via frozen-sha invariant, not via positive ok: line | I.5 |
| Re-prime refusal probe in clean Gate 3.6 | Don't. Bundling deliberately injects err: into the gate's journal slice; refusal-guard probe belongs in a separate error-path gate | I.6 |
| `--debug-print` rc capture via `output=$(CMD 2>&1 || true)` | The `\|\| true` form masks failures. Use the subshell + tempfile pattern: `( CMD >out 2>err ; echo $? >rc_file )`; run `--debug-print` exactly once in Gate 3.7. See full pattern in `06B` Step 32 sub-gate 3.7 | I.7 |
| `/var/log/dnf5.log` plugin-load evidence in Fedora 43 | Not authoritative in this environment (no `Loaded libdnf plugin` entries written across the file's history). Use journald (`journalctl -q -t tboot-dnf-posttrans --since "$PRE_TIME"`) as the evidence surface instead. The decider's info-line stream under that tag proves the plugin loaded and the rule fired | J.1 |
| Timestamp comparison against `dnf5.log` | `date -Iseconds` emits `+02:00`; `dnf5.log` writes `+0200`. Lexicographic and direct string comparisons are unreliable. Prefer `journalctl --since` (systemd time parser handles both) or convert both sides to epoch with `date -d ... +%s` | J.2 |
| Lock-path persistence after a clean decider/helper exit | Expected and correct. `flock(1)` releases on fd close without unlinking; the file persists as the next holder's reuse target. The invariant to test is "not actively held" (non-blocking `flock <> -n` probe with symlink rejection), not "path absent" | J.3 |
| Decider `mode=normal` vs helper `mode=production` in journal output | Same end-to-end execution; different terminology. Decider was authored later and uses `mode=normal`; helper was authored earlier and uses `mode=production`. Harness helper-start assertions must use `mode=production`; harness decider-start assertions must use `mode=normal`. Cross-mixing causes false-negative harness FAILs at the helper-start assertion (`FAIL 411`-style) even when the mutation completed correctly. | K.1 |
| B.2.4 Gate 3 reference-run FAIL 411 with rest of journal healthy | Mutation likely succeeded; the harness regex is wrong. Do not roll back immediately. Read the helper journal manually; if `info: starting version=… mode=production` is followed by the full kernel-enumeration + dracut + kernel-install + 80-tpm2-sign chain ending in `helper exited 0`, run the K.1 continuation verifier as the canonical recovery path. The 2026-05-25 reference run was closed via this verifier at 43/43 PASS. | K.1 |
| DNF5 `--assumeno` rc=1 on Fedora 43 | Documented refusal exit, not a resolution failure. Required combined check: rc ∈ {0,1} AND output matches `Operation aborted by the user.` / `Is this ok [y/N]:` / `Nothing to do` AND decider journal silent over the assumeno window. | K.2 |
| Baseline manifest sha advanced; PCR 11 value also advanced | Strong-closure path (outcome a per K.3). dracut autodetect pulled the new module's measured content into the UKI's `.initrd` section. Gate 7 closure-criterion documents the path as (a); the strong-closure axis was confirmed by Gate 6B (runtime PCR 11 byte-matched stored prediction `28E66CE2…C90F` across real reboot, 28/28 PASS). | K.3 |
| Baseline manifest sha advanced but PCR 11 value unchanged | Helper-path-only closure (outcome b per K.3). Chain machinery proven; value-tracking axis not exercised by this candidate. Gate 7 closure-criterion documents the path as (b). | K.3 |
| Baseline unchanged but PCR 11 value advanced | Anomalous (outcome d per K.3). Should not occur via the trusted-boot chain: indicates out-of-band predictor invocation or chain bypass. Investigate. | K.3 |

---

## Symptom-keyword index

For fast lookup. If you observe the symptom on the left, look at the finding on the right.

| If you see... | Look at |
|---|---|
| `Error loading EFI binary ... Load Error` after sbverify OK | B.1 |
| `sbverify OK` but UKI rejected by firmware at boot | B.1 |
| ukify exits 0 but `.pcrsig` section absent | B.2 |
| objcopy output file smaller than input | B.3 |
| Unsigned UKI from kernel-install plus ukify | B.4 |
| `systemd-creds decrypt` requires `--tpm2-signature=` unexpectedly | C.1 |
| Stderr noise from systemd-creds despite exit 0 | C.2 |
| `bash: systemd-measure: command not found` | D.1 |
| `systemd-measure calculate` awk parser returns empty | D.2 |
| ukify-driven UKI has no `.pcrsig` despite `[PCRSignature:initial]` config | D.3 (this is intentional pre-hook) |
| `${MID}-${KVER}.efi` appears next to `${KVER}.efi` on ESP | E.1 |
| `dnf reinstall kernel-core` succeeds despite documented "hold" | E.2 |
| ShellCheck SC2317 or SC2329 on `_cleanup` | F.1 |
| pesign reports >1 signer | F.2 |
| `efi-readvar` shows empty Subject/Issuer values | A.4 |
| Manual `ukify build` produces unsigned UKI despite `uki.conf` | D.4 |
| `-bash: PROMPT_START: unbound variable` after `set -u` | F.4 |
| Kernel panic `Attempted to kill init!` in nested virt | A.2 |
| systemd-boot menu missing Fedora kernel entries | A.1 |
| Anaconda blocks install with "must include /boot" | A.3 |
| Heredoc `cryptsetup` reports passphrase failure without prompting | F.5 |
| `Failed to unseal secret using TPM2: No such device or address` | G.1 |
| UKI rebuilt successfully but `/etc/crypttab` change not in embedded initrd | G.2 |
| `Unknown argument "--debuglevel"` / `Unknown argument "-v"` from dnf5 | H.1 |
| dnf debug output is silent on stdout/stderr despite a flag being set | H.1 |
| `Installed packages for argument 'X' are not available in repositories in the same version, ... cannot reinstall` | H.2 |
| Sensitive-package grep gate fires on a clean dnf transaction (matches `inbound`) | H.3 |
| UKI sha changes but `expected-pcr11-after-hook-uki.txt` value identical after helper run | E.3 |
| Pre/post PCR 11 value comparison reports unchanged after `tboot-dnf-helper` invocation | E.3 |
| External read-back scanner flags a manifest category label (e.g. `08-cryptsetup-tooling`) but the in-script self-scan does not | H.4 |
| Two `awk`-based scanners disagree on whether a token appears in a non-comment line | H.4 |
| Someone proposes patching a staged decider source to silence a trust-boundary false positive | H.4 |
| `qm listsnapshot` column-1 awk match fails despite the snapshot being visible | I.1 |
| `Wide character in printf at /usr/share/perl5/PVE/GuestHelpers.pm` warnings | I.1 (cosmetic; ignore) |
| `bootctl get-default` returns empty but `bootctl status` shows correct Default Entry | I.2 |
| `FAIL: /usr/local/sbin missing or is a symlink` on Fedora 43 | I.3 (UsrMerge: `/usr/local/sbin -> bin`; resolve via `readlink -f`) |
| Helper / decider lives at `/usr/local/bin/...` after install to `/usr/local/sbin/...` | I.3 (UsrMerge layout) |
| Harness asserts `[[ -z "$journal_output" ]]` but captures `-- No entries --` and FAILs | I.4 |
| `tboot-dnf-posttrans --self-test` exits 0 but harness FAILs on missing `ok: helper present and executable` | I.5 |
| Gate 3.6 journal slice contains err: lines after a deliberately-failed second `--prime` | I.6 |
| `dbg_rc=0` but `--debug-print` clearly produced no output / stderr indicates failure | I.7 (`\|\| true` masking inside command substitution) |
| `grep -Fq 'Loaded libdnf plugin "actions"' /var/log/dnf5.log` returns rc=1 after a transaction that demonstrably fired the production rule | J.1 (`dnf5.log` does not record plugin-load entries in this Fedora 43 environment; switch to journald) |
| Lexicographic timestamp comparison against `dnf5.log` rejects newer entries as older | J.2 (TZ-offset format mismatch: `+02:00` vs `+0200`) |
| `[ ! -e "/run/tboot-dnf-posttrans.lock" ]` fails after a clean Gate 6A / 6B run; `fuser -v` shows no holder | J.3 (`flock` does not unlink; the right invariant is "not actively held") |
| `[ ! -e "/run/tboot-dnf-helper.lock" ]` fails (file present, no holder) | J.3 (same: both decider and helper locks share this idiom) |
| Gate 3 harness `FAIL 411: helper normal-mode start lines = 0` but decider/helper/hook journals show full successful end-to-end flow | K.1 (regex bug: harness used `mode=normal` for helper; helper logs `mode=production`. Mutation succeeded; recover via K.1 continuation verifier.) |
| Need to confirm post-hoc that a B.2.4 Gate 3 mutation succeeded after a harness regex bug | K.1 (canonical continuation verifier in this finding's body; 43/43 PASS on the 2026-05-25 reference run) |
| Decider journal shows `mode=normal` but helper journal in same transaction window shows `mode=production` | K.1 (expected; same code path, different terminology; not a bug) |
| `dnf install --assumeno <pkg>` returns rc=1 with `Operation aborted by the user.` on Fedora 43 | K.2 (documented refusal exit; rc ∈ {0,1} is acceptable when combined with refusal-phrase + decider-silent checks) |
| Harness needs to validate a DNF5 transaction preview without committing | K.2 (three-part check: rc, refusal phrase, decider journal silent over window) |
| After Gate 3: baseline manifest sha changed AND stored PCR 11 value changed | K.3 outcome (a): strong-closure path |
| After Gate 3: baseline manifest sha changed but stored PCR 11 value unchanged | K.3 outcome (b): helper-path-only closure |
| After Gate 3: stored PCR 11 value changed but baseline manifest sha unchanged | K.3 outcome (d): anomalous; investigate (out-of-band predictor invocation or chain bypass) |
| Gate 7 discipline-gate-lift decision: when does the strong-closure path apply | K.3 (outcome (a) sufficient subject to Gate 6 reboot validation; outcome (b) closes machinery only) |
| `find /boot/efi/EFI/Linux -maxdepth 1 -type f -iname '*.efi'` returns more files than the kernel-install + helper chain would produce | L.1 (forensic UKIs accumulate from B.1 / B.3 development; audit before any high-risk reboot gate) |
| An unexpected `.efi` file on the ESP is `sbverify`-OK against `db.crt` with `signer_count = 1` and VMA-sane | L.1 (firmware-loadable forensic UKI; not "menu clutter": would be accepted by firmware if selected) |
| Forensic UKI on ESP has section table missing `.pcrpkey` and `.pcrsig` | L.1 (pre-hook B.1 development artifact: would boot but fail closed at LUKS unlock if selected; passphrase prompt is the visible symptom) |
| Forensic UKI on ESP has `.pcrpkey`/`.pcrsig` present but is not the MID-prefixed kernel-install-produced UKI | L.1 (B.3 development artifact: MAY unlock LUKS if its embedded `.pcrpkey` matches the live policy public key; NOT PROVEN by the audit; silent boot substitution possible if `LoaderEntryDefault` is cleared) |
| Need to design a pre-reboot probe contract that catches new firmware-visible artifacts on the ESP before B.2.5 closes | L.1 (staged allowlist: require two MID-prefixed UKIs, allow the three known forensic artifacts, hard-fail on any additional unexpected `.efi`) |
| `.loaderror`-suffixed UKI is visible in `ls /boot/efi/EFI/Linux/` but does NOT appear in `find -iname '*.efi'` output | L.1 / B.1 (validated forensic-preservation convention working as designed; suffix excludes it from systemd-boot enumeration) |
| `LoaderEntryDefault` cleared, NVRAM reset, or firmware factory reset on a system with unexpected `.efi` files in `/EFI/Linux/` | L.1 (systemd-boot "highest version string" fallback may select a firmware-loadable forensic UKI; execute B.2.5 cleanup FIRST) |
| Cleanup proposal asks to delete the three known forensic UKIs | L.1 (rename to `.forensic-b1` / `.forensic-b3` first, preserve forensic value; deletion is a separate later decision after retention period) |
| systemd-boot menu surfaces multiple boot entries on a system documented to have only the two MID-prefixed UKIs | L.1 (audit `/EFI/Linux/` read-only via 06B Step 37 Step 2 recipe; classify and act accordingly) |
| Future Gate 6A / Gate 6B / Module 4 monitoring needs to detect a new firmware-visible artifact appearing on the ESP | L.1 (preventive ESP-inventory check in the pre-reboot probe contract; staged allowlist until B.2.5, then hard form) |
| `rpm -qf /boot/efi/EFI/BOOT/BOOTX64.EFI` returns `shim-x64-...` despite the file's content being our sbsigned systemd-boot | L.2 (RPM-path-ownership conflict inherited from Fedora-default install; legacy-cleanup-scope; **not B.4 scope**) |
| `rpm -V shim-x64` would flag `BOOTX64.EFI` as `S.5....T.` (size, digest, mtime drift) | L.2 (latent operational symptom of the path-ownership conflict) |
| `dnf reinstall shim-x64` or `dnf upgrade shim*` resolution lists a shim package | L.2 (operator must abort: shim transaction restores Microsoft-CA-signed bytes at `/EFI/BOOT/BOOTX64.EFI`, fail-closed under enforced PK/KEK/db; discipline gate retained until Module 4A) |
| Need to specify a directory-level invariant for `/EFI/systemd/` under the B.4 chain | L.3 (exactly `systemd-bootx64.efi`, db.crt-signed, signer_count=1; zero unexpected `.efi`; hard form from day one) |
| Need to specify a directory-level invariant for `/EFI/BOOT/` covering legacy `fbx64.efi` | L.4 (pre-cleanup allowlist: BOOTX64.EFI + optionally fbx64.efi + zero other; post-cleanup drop allowlist) |
| `find /boot/efi/EFI/BOOT -maxdepth 1 -type f -iname '*.efi'` returns `fbx64.efi` alongside `BOOTX64.EFI` | L.4 (pre-cleanup state; `fbx64.efi` is Microsoft-CA-signed shim fallback bridge; fail-closed under our PK/KEK/db; cleanup-scope rename-with-suffix per the B.1 `.loaderror` / B.2.5 `.forensic-*` precedent) |
| `bootctl status` enumerates a second active Boot Loader from EFI Variables (e.g. `Boot0008 Fedora` → `/EFI/fedora/shimx64.efi`) | L.5 (dormant Fedora-default-install residue; `Linux Boot Manager` 0x0002 takes BootOrder priority; firmware fall-through to Fedora entry fails closed; Module 4A removes via `efibootmgr -B -b 0008`) |
| Operator looks for the `bootctl` manpage in the `systemd` package's filelist and finds nothing | M.4 (on Fedora 43 the `bootctl(1)` manpage is owned by `systemd-udev`, not the main `systemd` package; verify locally via `rpm -ql systemd-udev | grep bootctl`) |

---

## Update history
 2026-07-16: Published source revision 1.0.0. Both deciders now stop before invoking a helper when
the unsafe-reboot sentinel cannot be written. The systemd-boot decider and helper also reject a
symlinked state directory and require `root:root` ownership with mode 0700. Documentation was
aligned with the embedded UKI policy model, controlled key rotation, TPM replacement and the June
operational validation. The catalog prose was also tightened without changing its findings,
commands or evidence. The current artifact hashes are recorded in `SHA256SUMS`; older hashes
below identify the historical May validation run. The revised scripts passed repository integrity,
syntax and embedded-copy checks. A new live transaction and reboot regression run remains
outstanding.

2026-06-19: The validated host completed an unrestricted `dnf update`, including a kernel version
upgrade. The DNF actions chain detected drift, rebuilt and signed the UKIs, refreshed the PCR 11
prediction and advanced its baseline. A subsequent reboot completed without a recovery passphrase,
and runtime PCR 11 matched the stored prediction. This operational result supersedes the earlier
discipline-gate statements that kept blind updates and kernel upgrades blocked. The raw terminal
transcript was not retained, so the May gate records below remain the detailed captured evidence.

2026-05-28 (latest, post-B.5-close): Added section O (Block B.5 arm/first-fire/converge/reboot
findings) with three findings O.1–O.3, discovered during the 2026-05-28 `06B` Step 39 (B.5)
execution session. **O.1** (the B.4 decider emits a numeric `(equal=N shape=N)` decision string on
the drift branch but a worded `unchanged (manifest==baseline AND converged-shape)` string on the
converged branch: a Gate 4 matcher asserting `shape=1` FAILS against genuinely-converged state; the
canonical Gate 4 carries a dual-form matcher). **O.2** (a UKI rebuilt by the co-firing B.2 helper
advances its whole-file sha256 while predicted+runtime PCR 11 stay byte-identical, because PCR 11
measures section content not the PE+ timestamp/Authenticode/PSS-salt: do not treat a UKI sha change
as a measurement change; cross-refs E.3/K.3). **O.3** (`dnf reinstall systemd-boot-unsigned`
triggers BOTH chains: the broad B.2 `50-` rule fires first and rebuilds UKIs, then the narrow B.4
`60-` rule converges systemd-boot signing; this two-chain co-fire is the B.5 model correction,
expected and benign; includes the RE-PIN table superseding the pre-B.5 pins). None of the three are
source bugs; O.1 is a harness-matcher correction (now baked into the canonical Gate 4), O.2/O.3 are
behavioural model corrections. **B.5 closed end-to-end** via `06B` Step 39 Gates 0–7: arm
(publish `60-tboot-sbloader.actions`), first-fire (`dnf reinstall systemd-boot-unsigned`, rc=0,
two-chain co-fire), convergence proof (both chains), reboot validation (**runtime PCR 11 =
`28E66CE2…C90F` byte-match, no LUKS passphrase, Secure Boot enabled, both chains converged**),
closure snapshot `module5-b5-validated`. **Discipline-gate lift:** `dnf upgrade systemd*` and `dnf
reinstall systemd-boot*` are now allowed under standard operator review (B.4 install + B.5
arm/converge/reboot both validated); still gated are `dnf upgrade kernel*` (kernel-upgrade class)
and blind `dnf update`/`dnf upgrade` without explicit package review. RE-PIN authoritative values
adopted: B.2 baseline `47903d22…c220d` (was `6b032bd8…2a47`), booted UKI `1187c78c…b843` (was
`0ac32c8b…6949`), secondary UKI `ac9c5090…b5e7`, systemd-boot signed loaders `4859f48a…4986`,
PCR 11 unchanged `28E66CE2…C90F`. The ESP forensic-UKI residue (L.1) is unchanged by B.5 and
remains deferred to Module 4 / B.2.5. Frozen UKI-signing authority unchanged: hook
`a455444a…3b8c502f`. Cross-references added to `06B` Step 39, `00_Current_Project_State.md`, and
`05_Update_Workflows_and_Key_Storage.md` §3.5.

 2026-05-25 (latest, post-B.4-Gate-1-evidence): Added L.2/L.3/L.4/L.5 and M.4 findings, discovered
during the 2026-05-25 B.4 Gate 1 (rev-2) read-only preflight on VM 500. **L.2**
(`/boot/efi/EFI/BOOT/BOOTX64.EFI` RPM-path-owned by `shim-x64-15.8-3.x86_64` despite content being
our sbsigned systemd-boot: context-only; latent operational consequence is that `dnf reinstall
shim-x64` or `dnf upgrade shim*` would restore Microsoft-CA-signed bytes at that path and break
fallback-loadability under our enforced PK/KEK/db; fail-closed at firmware load, not a security
compromise; new `dnf upgrade shim*` / `dnf reinstall shim*` discipline-gate row added to
`00_Current_Project_State.md`; **explicitly NOT B.4 scope**: handed to Module 4A in
`04A_Legacy_Residue_Cleanup_Gate.md`). **L.3** (`/EFI/systemd/` directory invariant for B.4: hard
form from day one: exactly `systemd-bootx64.efi`, db.crt-signed, signer_count=1, zero unexpected
`.efi`; B.4-scope target; already met at Gate 1). **L.4** (`/EFI/BOOT/` directory invariant for the
legacy-cleanup gate: pre-cleanup allowlist includes `fbx64.efi` (Microsoft-CA-signed shim fallback
bridge, mtime 2024-03-19, ~87.8 KB; fail-closed under our PK/KEK/db; rename-with-suffix
`.forensic-shim-fbx64` per B.1 `.loaderror` precedent at Module 4A), post-cleanup drops allowlist;
**explicitly NOT B.4 scope**). **L.5** (NVRAM `Boot0008 Fedora` → `\EFI\fedora\shimx64.efi`
dormant; `Linux Boot Manager` 0x0002 takes BootOrder priority; firmware fall-through fails closed
under enforced PK/KEK/db; cleanup via `efibootmgr -B -b 0008` + optional rename of
`/EFI/fedora/shimx64.efi`; **explicitly NOT B.4 scope**). **M.4** (packaging clarification: on
Fedora 43 the `bootctl(1)` manpage at `/usr/share/man/man1/bootctl.1.gz` is owned by
`systemd-udev`, not the main `systemd` package; documentation correction only). B.4 Gate 1 closure
verdict: **PASS with findings (no stop condition triggered)**; the rev-3 source-side-signing
architecture (independent B.4 chain; signing target
`/usr/lib/systemd/boot/efi/systemd-bootx64.efi.signed`; ESP propagation via `bootctl --no-variables
[install|update]` with Gate 2 picking the exact command; narrow
`package_filter=systemd-boot-unsigned`; 5-entry manifest) stands unchanged. F1/F2/F4 reclassified
as legacy-cleanup-scope (L.2/L.4/L.5 above) and explicitly removed from B.4 trigger surface and
manifest: no `shim-x64` NEVRA in B.4 manifest, no filterless `.actions` rule. F3
(`systemd-boot-clear-sysfail.service` enabled + `systemd-bootctl@.service` /
`systemd-bootctl.socket` / `systemd-boot-random-seed.service` flagged by cross-directory `bootctl`
grep) remains the sole Gate 1 closure blocker: supplemental read-only probe (Blocks N.1–N.6)
pending to capture each unit's `ExecStart` and determine whether any join
`systemd-boot-update.service` on the Gate 2 mask-list. **Most-likely outcome**: none of the four
units write ESP bootloader binaries → Gate 2 mask-list remains `systemd-boot-update.service`
only. Cross-references added to `00_Current_Project_State.md` (discipline-gates table, Where to
look for what, Next session priorities #3 + #5, Last update), `04A_Legacy_Residue_Cleanup_Gate.md`
(new file), and the symptom-keyword index here (9 new rows). Frozen trust-chain shas unchanged:
hook `a455444a…3b8c502f`, helper `4bef2239…20f4b`, predict `2d4985fa…aff160`, decider
`35ad8733…73cdd`, production rule `dfbffe56…90d798`. Source-side `.efi.signed` path still
absent (correct pre-B.4 state). Booted ESP UKI sha `0ac32c8b…6949` unchanged. Runtime PCR 11 =
stored prediction `28E66CE2…C90F`.

2026-05-25 (earlier, post-B.2.4-Gate-7): Added section L (ESP forensic UKI audit discipline) with
one finding L.1, discovered during the 2026-05-25 `06B` Step 37 (B.2.4 Gate 7) read-only ESP audit.
L.1: a `find /boot/efi/EFI/Linux -maxdepth 1 -type f -iname '*.efi'` inventory on the reference VM
500 system returned 5 total `.efi` files where only 2 are expected (booted-kernel +
secondary-kernel MID-prefixed UKIs from `kernel-install` + the helper chain). The 3 unexpected
files are pre-existing development-phase forensic artifacts from B.1 (pre-hook bare-kver ukify
build, no `.pcrpkey`/`.pcrsig`) and B.3 (hook live-validation `test-tpm2initrd-pcrsig-*.efi` and
`test-ukify-native-pcrsig-*.efi`, both with `.pcrpkey`/`.pcrsig` present). **All 3 unexpected files
are `sbverify`-OK against `db.crt`, `signer_count = 1`, and VMA-sane**: they would be accepted by
firmware if selected. They are **firmware-loadable forensic UKIs**, even though
they may appear as menu clutter.
For the bare-kver UKI: signed but lacks `.pcrpkey`/`.pcrsig`, so it is outside the validated
forward-sealed path; would be expected to fail closed at LUKS unlock if selected (operator-visible
passphrase prompt: fail-safe but symptomatically identical to a real Gate 6 failure). For the two
`test-*` UKIs: they MAY unlock LUKS if their embedded `.pcrpkey` matches the live policy public
key, but **this was not proven by the audit** (verification requires extracting
`.pcrpkey` from each forensic UKI via read-only `objcopy --dump-section` against a tmpfs copy and
comparing to `/etc/systemd/tpm2-pcr-public-key.pem`). The `.loaderror`-suffixed broken-objcopy UKI
from B.1 is correctly excluded from systemd-boot enumeration by its suffix (validates the
cleanup-via-suffix-rename pattern). **The reference-run audit recipe uses `-iname '*.efi'`**
(case-insensitive) per ESP VFAT case-insensitive semantics; this also excludes
`.loaderror`-suffixed files automatically. Per-file forensic record: `6.19.14-200.fc43.x86_64.efi`
(bare-kver; 53059144 bytes; mtime 2026-05-07 14:59:48; sha `bcfa6403…e097`; sbverify OK; no
`.pcrpkey`/`.pcrsig`); `test-tpm2initrd-pcrsig-6.19.14-200.fc43.x86_64.efi` (56429128 bytes; mtime
2026-05-08 03:10:44; sha `b94b273e…c4ca`; sbverify OK; `.pcrpkey`/`.pcrsig` present);
`test-ukify-native-pcrsig-6.19.14-200.fc43.x86_64.efi` (53061192 bytes; mtime 2026-05-08 02:57:26;
sha `56e0aa68…d277`; sbverify OK; `.pcrpkey`/`.pcrsig` present). **The audit was NOT a Gate 6
blocker**: `bootctl status` at Gate 4, 6A pre-reboot probe, and 6B post-reboot validation all
confirmed Default Entry and Current Entry pointed to the MID-prefixed rebuilt 6.19.14 UKI; the
forensic UKIs were menu-visible but not selected, and `LoaderEntryDefault` correctly persisted
across the reboot. The finding's scope is "audit discipline + cleanup procedure" + "preventive
pre-reboot ESP-inventory contract", not "active runtime bug". **Cleanup recipe deferred to a
separate gate B.2.5** (ESP forensic UKI cleanup): take pre-cleanup snapshot
`module5-b2-5-pre-cleanup` (parent `module5-b2-4-validated`), read-only re-audit, atomic rename
each unexpected firmware-visible artifact to a non-enumerated forensic suffix (`.forensic-b1` for
the bare-kver pre-hook B.1 artifact, `.forensic-b3` for the two B.3 development variants), no
deletion (preserves forensic value while removing systemd-boot enumeration risk), verify `bootctl
status` still points to MID-prefixed UKI, take post-cleanup snapshot `module5-b2-5-validated`
(parent `module5-b2-5-pre-cleanup`); reboot not required unless cleanup touches current/default
entries (which it must not). **Preventive ESP-inventory invariant added to the pre-reboot probe
contract for future high-risk reboot gates (B.4 / B.5 / kernel-upgrade / Module 4)**: staged form
until B.2.5 closes: enumerate `/boot/efi/EFI/Linux/*.efi` case-insensitively, require the two
MID-prefixed expected UKIs, allow the three known forensic artifacts only, hard-fail on any
additional unexpected `.efi`; after B.2.5 closes, the allowlist drops and the rule becomes the
simpler hard form (zero unexpected `.efi` files outside the MID-prefixed allowlist).
Quick-reference (forbidden-procedures + symptom-keyword index) extended with 3+10 new rows.
Cross-references added to `06B` Step 37 (Gate 7 closure snapshot + ESP audit) and to
`05_Update_Workflows_and_Key_Storage.md` §3.4 (operational-policy framing including the pre-reboot
ESP-inventory invariant). **B.2.4 closed end-to-end via `06B` Steps 34–37**: Gates 1–3
(positive drift path pre-reboot 58/58 + snapshot + 43/43 PASS), Gate 4 (pre-reboot read-only
cross-validation 36/36 PASS), Gate 5 (pre-reboot snapshot `module5-b2-4-pre-reboot`), Gates 6A–6B
(operator-attested clean reboot with no LUKS passphrase prompt + post-reboot read-only validation
17/17 + 28/28 PASS with **runtime PCR 11 byte-matching stored prediction `28E66CE2…C90F`**), Gate
7 (closure snapshot `module5-b2-4-validated` + read-only ESP audit + staged DNF discipline lift).
**Block B.2 full production flow: complete with staged DNF discipline.** **Staged DNF discipline
lift applied at Gate 7**: non-systemd / non-kernel single-package DNF transactions
(install/remove/reinstall/upgrade) including dracut-sensitive packages allowed under normal
operator review; `dnf upgrade systemd*` (B.4 / B.5 scope), `dnf reinstall systemd-boot*` (B.4
scope), `dnf upgrade kernel*` (kernel-package class not yet validated), and blind `dnf update`/`dnf
upgrade` without explicit package argument remain explicitly gated. Block B.2.5 (ESP forensic UKI
cleanup) is the next scheduled block. Frozen trust-chain shas unchanged across B.2.4 closure:
decider (`35ad8733…73cdd`), helper (`4bef2239…20f4b`), predict (`2d4985fa…aff160`), hook
(`a455444a…3b8c502f`), production rule (`dfbffe56…90d798`).

2026-05-28: Added section N (Block B.4 systemd-boot-signing findings) with six findings N.1–N.6,
discovered during the 2026-05-28 `06B` Step 38 (B.4 install/publish gate) execution session. N.1:
`bootctl --no-variables update` skips a re-signed same-version binary (version-identical,
byte-different; rc=1 `"same boot loader version in place already"`), so the loader propagation
primitive must be `bootctl --no-variables install` (force-copy to `/EFI/systemd/` +
`/EFI/BOOT/BOOTX64.EFI`); the one open caveat (does `install` pick `.efi.signed` over the unsigned
source) is closed at helper runtime by a mandatory fail-closed loopback both-target checkpoint,
live-exercised in B.5. N.2: read-back/probe harness hygiene: never use `set -u` at the interactive
shell top level (use `bash <<'EOF'`), `set +H` before `!`-containing headers, avoid `${` and
forbidden-token literals (`bootctl`, etc.) on any non-`#` line **including heredoc help prose**
(the dumb-but-trustworthy forbidden-token scan only excludes `#`-prefixed lines), and make presence
checks multi-line-aware; continuation of the H.4/I.4/J.1–J.3 harness-engineering class. N.3:
`libdnf5-actions(8)` rule layout is `callback:filter:direction:options:command`; `direction=in`
enumerates {downgrade,install,reinstall,upgrade}, `reinstall` matches both `in` and `out` (internal
remove+install), per-package command dedup is guaranteed: the B.4 rule uses `direction=in` with
`package_filter=systemd-boot-unsigned`. N.4: `dnf reinstall --cacheonly --assumeno
systemd-boot-unsigned` resolves `Reinstalling: 1 package` then aborts rc=1 (K.2 refusal semantics):
a safe transaction-preview confirming the B.5 first-fire driver; NEVRA
`systemd-boot-unsigned-258.7-1.fc43.x86_64` is reinstall-eligible in `updates`. N.5: of the four
`bootctl`-referencing units, only `systemd-boot-update.service` installs/updates the loader binary;
`systemd-boot-random-seed.service` writes only `/boot/efi/loader/random-seed` (out of the B.4
boundary, do NOT mask), and `clear-sysfail`/`bootctl@`/`bootctl.socket` never write loader
binaries: B.4 mask-list = `systemd-boot-update.service` only. N.6: the B.4 decider no-op (D-1)
requires baseline-equality AND converged-shape (`src_signed`≠ABSENT and both ESP copies ==
`src_signed`); baseline-equality alone is insufficient: proven live by the intentionally
non-converged primed baseline producing `drift (equal=1 shape=0)` post-prime (never `unchanged`);
baseline-absent normal mode is FAIL CLOSED. All six findings are environmental, design-lock, or
harness-engineering observations; none are helper-source or decider-source bugs. The B.4 helper
(`1a411e85…f18f`), decider (`3ccef59c…f8e7`), and rule (`2e83fcfc…8274`) artifacts are
unchanged. These findings were referred to informally as `K.4`–`K.7`/`M.2` during the working
session; assigned to section N on catalog entry to avoid collision with the existing K (B.2.4) and
M sections. Quick-reference and symptom-keyword index extended. Cross-references added to `06B`
Step 38. **B.4 install/publish gate closed; chain installed + primed (non-converged) + masked,
UNARMED (`60-` rule unpublished, `.efi.signed` absent, helper never fired); publish + first-fire
deferred to B.5.** The N findings and D-1 design-lock note were pending review at the time of this
entry and were subsequently confirmed during B.5 validation.

2026-05-25 (post-B.2.4-Gate-3): Added section K (B.2.4 Gates 1–3 findings) with three findings
K.1–K.3, discovered during the 2026-05-25 `06B` Step 34 execution session. K.1: the decider logs
its production code path as `mode=normal` and the helper logs its production code path as
`mode=production`; the two refer to the same end-to-end execution but use different vocabulary
(decider was authored later in B.2.3 with `prime`/`dry-run`/`debug-print`/`self-test` siblings;
helper was authored earlier in B.2.2 with `dry-run`/`self-test` siblings). The rev-3 Gate 3
reference-run harness asserted helper start with `mode=normal` and FAILed at assertion 411 even
though the mutation completed correctly. Body includes the full canonical post-hoc continuation
verifier (43/43 assertions) that closed Gate 3 on 2026-05-25 against the now-mutated live state;
this verifier is the canonical recovery path for FAIL 411 specifically. K.2: DNF5 `dnf install
--assumeno <pkg>` on Fedora 43 (`libdnf5-plugin-actions-5.2.18.0-3.fc43.x86_64`) returns rc=1 with
the documented refusal phrase `Operation aborted by the user.`; the rc=1 is the documented refusal
exit, not a resolution failure. Three-part harness check is required (rc ∈ {0,1}, refusal phrase
present in output, decider journal silent over the assumeno window) to safely accept this as a
transaction-preview signal. K.3: the decider's baseline manifest sha and the stored PCR 11
prediction value advance on different axes: baseline tracks the on-disk 14-category boot-input
file-content manifest, PCR 11 value tracks the freshly-rebuilt booted UKI's measured-content
sections (`.linux`, `.initrd`, `.cmdline`, `.osrel`, `.uname`, `.sbat`, `.pcrpkey`) as computed by
`systemd-measure calculate`. The (a)/(b)/(c)/(d) outcome table documents all four observable
combinations: (a) both advance = strong-closure path; (b) baseline advances, PCR 11 unchanged =
helper-path-only closure; (c) baseline unchanged = no-drift path; (d) baseline unchanged but PCR 11
advances = anomalous. The 2026-05-25 Gate 3 reference run produced outcome (a): dracut autodetect
pulled the new `dracut-network` module's measured content into the host-only initramfs, so the new
UKI's `.initrd` section actually differs and the predictor's output advanced `A03EB49C…35DD` →
`28E66CE2…C90F`. The strong-closure path was subsequently confirmed by Gate 6B post-reboot
validation on 2026-05-25: runtime PCR 11 from `/sys/class/tpm/tpm0/pcr-sha256/11` byte-matched the
stored prediction `28E66CE2…C90F` after a real `systemctl reboot` (28/28 PASS); see the top entry
above for full Gate 6A–6B–7 closure detail. All three findings are environmental or
harness-engineering observations; none are decider-source, helper-source, or predictor-source bugs.
The frozen shas of decider (`35ad8733…73cdd`), helper (`4bef2239…20f4b`), predict
(`2d4985fa…aff160`), hook (`a455444a…3b8c502f`), and production rule (`dfbffe56…90d798`) are
unchanged. Quick-reference (conditional/clarification) and symptom-keyword index extended with 7+10
new rows. Cross-references added to `06B` Step 34 (Gates 1–3). **B.2.4 Gates 1–3 closed
end-to-end via `06B` Step 34** (Gate 1 58/58 PASS, Gate 2 snapshot-only, Gate 3 closed via the K.1
continuation verifier at 43/43 PASS); B.2.4 Gates 4–7 closed via `06B` Steps 35–37 (see top
entry above).

2026-05-25 (earlier): Added section J (Gate 4–7 environment and harness findings) with three
findings J.1–J.3, discovered during the 2026-05-25 B.2.3 Gates 4–7 execution session
(production `.actions` rule first live, two real DNF transactions invoked the decider end-to-end).
J.1: `/var/log/dnf5.log` does not record `Loaded libdnf plugin` entries in this Fedora 43
environment; journald evidence under tag `tboot-dnf-posttrans` is authoritative; the Gate 6A
canonical verifier was corrected to remove all `dnf5.log` assertions. J.2: `date -Iseconds` emits
TZ offset as `+02:00` while `dnf5.log` writes `+0200`; lexicographic timestamp comparison against
`dnf5.log` is fragile; `journalctl --since` (systemd's time parser handles both forms natively) is
the correct replacement. J.3: lock-path invariants must be "not actively held" via non-blocking
`flock <> -n` (with symlink rejection and non-truncating read/write fd open), not "path absent":
`flock(1)` releases on fd close without unlinking, and the persistent lock file is the standard
idiom across `/run/tboot-dnf-posttrans.lock` and `/run/tboot-dnf-helper.lock`. All three findings
are environmental or harness-engineering observations; none are decider-source bugs. The decider's
frozen sha (`35ad8733…73cdd`) is unchanged. Quick-reference and symptom-keyword index extended
with 4+5 new rows. Cross-references added to `06B` Step 33 (Gates 4–7). **B.2.3 closed
end-to-end** with closure snapshot `module5-b2-3-validated`; B.2.4 (live drift-detection through a
real dracut-sensitive transaction) pending; blind `dnf update` still blocked; `dnf upgrade
systemd*` still separately blocked.

2026-05-24 (latest): Added section I (Gate 3 environment and harness findings) with seven findings
I.1–I.7, discovered during the 2026-05-24 B.2.3 Gate 3 (sub-gates 3.0–3.8) execution sessions.
I.1: `qm listsnapshot` output prefixes snapshot names with tree glyphs; strip prefixes before
column-1 awk match; cosmetic PVE `Wide character in printf` warnings are not parse failures. I.2:
`bootctl get-default` returns empty on this Fedora 43 VM despite `bootctl status` reporting the
correct Default Entry; parse `bootctl status` instead. I.3: Fedora 43 UsrMerge layout makes
`/usr/local/sbin` a symlink to `bin`; install-target safety checks must `readlink -f` and assert
resolved literal == `/usr/local/bin`, then atomic-publish (`mktemp` + `install` + `mv -T`) inside
the resolved real directory. I.4: `journalctl -t TAG --since X` prints `-- No entries --` on stdout
when empty; use `journalctl -q` for tag-silence assertions. I.5: `--self-test` mode does NOT emit
`ok: helper present and executable`; the helper-presence branch in self-test is warn-only; verify
helper presence via frozen-sha invariant instead. I.6: do not bundle a re-prime refusal probe
inside the clean Gate 3.6 baseline-prime gate; the deliberate err: line breaks the gate-hygiene
property. I.7: in Gate 3.7, run `--debug-print` exactly once and capture rc via a subshell +
tempfile pattern (`( CMD >out 2>err ; echo $? >rc_file )`); do not mask exit codes with `|| true`
inside command substitution. All seven findings are environmental or harness-engineering
observations; none are decider-source bugs. Quick-reference and symptom-keyword index extended with
7+10 new rows. Cross-references added to `06B` Step 32 (Gates 3.0–3.8).

2026-05-24: Added finding H.4 (external audit-harness token-boundary regex must match the in-script
self-scan; do not patch staged sources for harness false positives). Discovered during `06B` Step
31 (B.2.3 Gate 2: staged decider read-back validation): the initial external Step 6 trust-boundary
scanner fired on line 532 of the staged decider (`local c="08-cryptsetup-tooling"`) because it used
the laxer `\<TOKEN\>` GNU awk word-boundary form, which treats `-` as a word boundary. The
in-script `_self_test_trust_boundary` already used the Deviation D regex
`(^|[^A-Za-z0-9_-])TOKEN($|[^A-Za-z0-9_-])` (hyphen inside the identifier class) and did not flag
the line. Resolution: harmonize the external harness with Deviation D; do not mutate the staged
decider source. Staged sha256 `35ad8733f190483f1bd6d071d2aa7cb8b1549286f9040086182443c584473cdd`
unchanged between failing and passing runs. Quick-reference and symptom-keyword index extended.
Cross-reference added to `06B` Step 31.

2026-05-22 (latest): Added finding E.3 (PCR 11 value invariance across UKI regeneration when
measured-content inputs are unchanged). Discovered during `06B` Step 30 (Block B.2.2) execution:
the booted-kernel UKI was rebuilt with a new sha (`75b9cf90…d94a` → `b6002e66…7943`) but the
stored PCR 11 value was unchanged (`A03EB49C…35DD`), because dracut produced bit-identical
initramfs content from identical inputs. Quick-reference table and symptom-keyword index extended.
Cross-reference added to `06B` Step 30.

2026-05-21 (latest): Added section H (DNF5 plugin layer and trigger mechanics) with H.1 (dnf5 has
no runtime verbosity flag), H.2 (`dnf reinstall` requires installed NEVRA in repos), H.3 (substring
grep of `boot` matches `inbound` header). Findings discovered during `06B` Steps 27–28 (Block
B.2.1) execution. Quick-reference tables and symptom-keyword index extended. Cross-references added
to `06B` Steps 27 and 28.

2026-05-21: Added F.5 (heredoc stdin consumption breaks interactive prompts), G.1
(systemd-cryptenroll PCR list shape and signature-validation phase mismatch), G.2 (kernel-install
add does not refresh /boot/initramfs under UKI layout). New category G ("LUKS2 and
systemd-cryptenroll") created. Quick-reference tables and symptom-keyword index extended. Findings
cross-referenced from `06B` Steps 20–26 (Module 3).

2026-05-08: produced as the merged successor to the earlier split between
`06F_Diagnostic_Findings_Catalog.md` and `06X_Deprecated_Procedures.md`. All 16 findings preserved.
Forbidden-procedure rules from `06X` absorbed as `Verdict:` fields and operational consequences.
Companion to `06B_Golden_Path_Rebuild_Runbook.md` and `00_Current_Project_State.md`.
