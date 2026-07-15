<!--
PUBLIC RELEASE VERSION — sanitized for publication.
Machine-id, IP addresses (RFC 5737 documentation range), hostnames, and the
LUKS UUID are format-valid EXAMPLE values, not the original lab identifiers.
All PCR values, policy-key digests, and script hashes are genuine measurements
from the validated lab build and are safe to publish (public values by design).
-->

---
tags:
  - trusted-boot
  - operator-notes
module: golden-path-operator-notes
category: orientation
status: draft
last-updated: 2026-05-29
---

# 06C: Golden Path Operator Notes

Operator orientation for the trusted boot lab rebuild: the mental models, the build sequence, and the per-component context that help a human understand *what* each part of `06B_Golden_Path_Rebuild_Runbook.md` is doing and *why* it is shaped that way — without re-deriving the theory that already lives in the architecture modules.

> [!IMPORTANT]
> **This file is not on the execution path**
> Nothing here is required to rebuild the lab. `06B` is self-contained and carries every command, expected result, stop condition, snapshot, and dangerous-mistake guard. This file is read-alongside context only. The deep architecture lives in `01`–`05`; failure and forensic detail lives in `06F`; current values and lineage live in `00_Current_Project_State.md`.

---

## 1. How to use this document set

The project is split so that each kind of question has exactly one authoritative answer. As an operator, route by intent:

| If you want to… | Read |
|---|---|
| Understand a *concept* (what a UKI is, what forward sealing means, how the disk unlocks) | `01`–`04` (architecture modules) |
| Understand the *update/automation design* and key-storage rationale | `05_Update_Workflows_and_Key_Storage.md` |
| Get *oriented* — the build sequence, how to read the gates, what each component is for operationally | **this file (`06C`)** |
| Actually *execute* a rebuild, step by step | `06B_Golden_Path_Rebuild_Runbook.md` |
| Understand what *not* to do, or why a specific bug appears | `06F_Diagnostic_Findings_Catalog.md` |
| Find *current* values — hashes, PCRs, snapshot lineage, next priorities | `00_Current_Project_State.md` |
| See Proxmox host-side detail | `proxmox_technical_docs_v13.md` |

The one rule that keeps this clean: `06C` explains and *points*; it does not copy commands from `06B`, current values from `00_state`, design rationale from `05`, or failure write-ups from `06F`. When this file says "deep dive:", that is where the authoritative explanation lives.

---

## 2. The build at a glance — phases and dependencies

The runbook builds five layers, each depending on the one before it. Knowing where you are in this chain is the single most useful orientation an operator can have, because every gate is really asking "is the layer below me still intact?"

1. **Block A — Secure Boot + UKI + the signing hook** (`06B` Steps 1–19). Establishes custom Secure Boot keys (manufacturer keys removed), a single signed Unified Kernel Image per kernel, and the `kernel-install` hook that forward-seals and signs every UKI. End state: the machine boots only UKIs signed by your `db` key, and every new kernel is automatically signed and forward-sealed.
2. **Module 3 — LUKS2 + TPM2 unlock** (`06B` Steps 20–26). Welds the encrypted root disk to the boot chain: the disk releases its key only if the machine booted the exact measured chain (PCR 7 + PCR 11), with a passphrase keyslot as the recovery anchor. Depends on Block A's measurements being stable and predictable.
3. **Block B.2 — kernel-signing automation** (`06B` Steps 27–37). Wraps the manual kernel/UKI rebuild in a fail-closed DNF post-transaction chain (decider + helper + action rule) so that an `dnf` transaction touching boot inputs automatically rebuilds, re-signs, re-seals, and re-predicts — or refuses to leave the machine bootable-but-unmeasured. Depends on the hook (Block A) being the trusted signing authority.
4. **Block B.4 / B.5 — systemd-boot signing automation** (`06B` Steps 38–39). Applies the same decider/helper pattern to the bootloader binary itself, so a `systemd-boot-unsigned` update re-signs and re-propagates the loader under your `db` key. B.4 installs + primes + masks the native updater (unarmed); B.5 arms, fires once under observation, converges, and reboot-validates.
5. **Module 4 — governance, recovery, decommissioning** (architecture in `04`; procedures pending). Key rotation, compromise response, disaster recovery, audit, separation of duties.

The exact snapshot that closes each phase — and the current rollback lineage — is tracked in `00_Current_Project_State.md` ("Snapshot lineage"). Do not memorise anchor names from anywhere else; that file is the single source.

---

## 3. Component orientation

Short, operator-facing context for each major piece the runbook builds or invokes. Each ends with a pointer to the authoritative deep explanation. None of this is needed to *run* a step — it is here so the step makes sense while you run it.

### 3.1 The Secure Boot key hierarchy (PK / KEK / db)

Three key tiers control what firmware will load: the Platform Key (PK) authorises changes to the KEK; the Key Exchange Key (KEK) authorises changes to `db`/`dbx`; the signature database (`db`) is the set of keys whose signatures the firmware will accept on a boot binary. This lab removes the manufacturer-supplied keys and enrolls its own, so the only thing the machine will boot is a binary signed by *your* `db` key. Enrolling happens in "Setup Mode" (firmware accepts new keys without authentication); once PK is set, the machine is in "User Mode" (enforcing). The operationally important consequence: after enrollment, anything not signed by your `db` — including a stock distro shim or an unsigned bootloader — is firmware-rejected, fail-closed. Deep dive: `01` §4.3 (Secure Boot vs TPM) and `04` §4.1 (trust authority model). The fwupd trade-off this creates is in `05`.

### 3.2 The Unified Kernel Image (UKI)

A UKI is a single PE/COFF (EFI) binary that bundles the kernel, initramfs, kernel command line, and metadata as named sections (`.linux`, `.initrd`, `.cmdline`, `.osrel`, `.uname`, `.pcrpkey`, `.pcrsig`, …). Because it is one file, one Authenticode signature over the whole image covers *all* of those inputs at once — you cannot swap the initramfs or edit the cmdline without breaking the signature. That single-signature property is what makes the measured boot tractable. Deep dive: `01` §2.

### 3.3 The signing hook — `80-tpm2-sign.install`

This is the piece most worth understanding, because nearly every later gate exists to protect it. It is a **`kernel-install` plugin**: a script at `/etc/kernel/install.d/80-tpm2-sign.install` that `kernel-install` runs automatically, in lexicographic order with the other `install.d` drop-ins, every time a kernel is added. Its job is to take the UKI that the stock `ukify` plugin just staged and **rebuild it as a forward-sealed, signed UKI** — joining the predicted PCR-policy signature into the image and signing the result with the `db` key — then place it on the ESP under its machine-id-prefixed name (`${MID}-${KVER}.efi`).

The operator-relevant facts:

- It is the **UKI signing authority**. The B.2 automation chain does not sign UKIs itself; it *invokes the conditions that cause this hook to run*. If the hook is wrong, every downstream measurement is wrong.
- It reads its UKI layout from `/etc/kernel/uki.conf` (the path `kernel-install`'s ukify plugin honours) — *not* `/etc/systemd/ukify.conf`, which only applies when `ukify` is called directly. This distinction is a known mentor-note error (`06F`).
- It depends on `BOOT_ROOT=/boot/efi` being persisted in `/etc/kernel/install.conf`; the default `BOOT_ROOT=/boot` puts UKIs in the wrong place.
- It forward-seals with `ukify --join-pcrsig`, **never** `objcopy --add-section` — the latter produces a binary that passes `sbverify` but is firmware-rejected at `LoadImage()` because of corrupt section VMAs (`06F` B.1).

Deep dive: the design rationale is `05` §3.2 (note: `05` shows a generic `99-`-prefixed example; the canonical artifact here is `80-tpm2-sign.install`), and the forward-sealing theory is `02`. The byte-exact hook source is inlined in `06B` Step 14 (sha-pinned).

### 3.4 Forward sealing and the two PCRs

Platform Configuration Registers (PCRs) are append-only TPM hashes that record what was measured during boot. This project binds the disk to two of them, for two different reasons:

- **PCR 7 — Secure Boot state (static binding).** Records the Secure Boot policy/keys. It is bound *statically*: the enrolled value is the current one, and it only changes if you change the Secure Boot configuration.
- **PCR 11 — UKI / kernel-phase measurements (signed-policy binding).** Records the measured sections of the booted UKI. This changes on every kernel/initramfs/cmdline change, so it cannot be bound to a fixed value or every update would lock you out.

"Forward sealing" is the trick that makes PCR 11 updatable: instead of sealing the disk key to a *value*, you **predict** what PCR 11 *will be* for the new UKI (with `systemd-measure`, which lives at `/usr/lib/systemd/systemd-measure` and is not on `PATH`), sign that prediction with the policy key, and embed the signed prediction in the UKI's `.pcrsig` section. At boot, `systemd-stub` presents that signed policy to the TPM, which releases the key if the runtime PCR 11 matches. So a new, correctly-built UKI unlocks the disk *before it has ever booted*, with no re-enrollment. Note the consequence operators trip over: a rebuilt UKI's *file* hash changes (new signature/timestamp) while its *predicted PCR 11* stays identical, because PCR 11 measures section content, not the whole-file signature (`06F` E.3 / O.2). Deep dive: `01` §4–§5, `02`.

### 3.5 The policy key, TPM sealing, and key release

Forward sealing needs a signing key (the "policy keypair"). Its public half is enrolled into the TPM policy at disk-enrollment time (the "welding" in `03` §1); its private half signs each PCR-11 prediction. That private key is itself **sealed into the TPM** so it exists in plaintext only transiently at signing time, then is shredded (`05` §2.2). Compromise of the policy private key is a governance event with a defined response (`04` §4.3). Deep dive: `01` §5 (policy-based key release), `05` §2.

### 3.6 LUKS2 unlock with the split policy

The encrypted root uses a LUKS2 header with two independent things that are easy to conflate: **keyslots** (each holds an encrypted copy of the master key, unlocked by a passphrase or a TPM token) and the **TPM policy** (the PCR conditions under which the TPM will release its share). This lab enrolls a TPM2 token bound to the *split* policy — PCR 7 static **and** PCR 11 signed — via `systemd-cryptenroll`, and keeps **keyslot 0 as a passphrase recovery anchor**. The mental model: the disk unlocks automatically only if the machine booted the exact Secure-Boot state *and* a correctly forward-sealed UKI; otherwise it falls back to the passphrase. A passphrase prompt at boot after Module 3 is therefore a *hard signal* that the measured chain did not match — never something to paper over. Deep dive: `03`.

### 3.7 The B.2 kernel-signing automation chain (decider + helper + rule)

Manual rebuilds do not survive routine `dnf` use, so B.2 automates them with a fail-closed pattern. A libdnf5 **action rule** (`50-tboot-posttrans.actions`, empty `package_filter` so it fires on every transaction) calls a **decider** (`tboot-dnf-posttrans`) after each transaction. The decider is the unprivileged brain: it computes a manifest of boot-relevant inputs, compares it to a stored baseline, and only if it detects drift does it invoke the privileged **helper** (`tboot-dnf-helper`), which does the actual `dracut --force` + `kernel-install add` (triggering the §3.3 hook) + PCR-11 re-prediction, then advances the baseline. A **sentinel** file (`UNSAFE-TO-REBOOT`) is written before mutation and cleared only on success, so an interrupted run fails closed. The decider/helper split (decide vs act) and the lock model exist so the decision path can run safely and the privileged path runs exactly once. Deep dive: `05` §3.3 (including why the rule is always-run with an empty filter, the manifest metadata format, and the baseline-update policy). The decider/helper/rule sources are inlined sha-pinned in `06B` Steps 29/31/33.

### 3.8 The B.4 / B.5 systemd-boot signing chain

The bootloader binary (`systemd-bootx64.efi`) also has to be signed by your `db` key, and a `systemd-boot-unsigned` update would otherwise drop an unsigned loader. B.4/B.5 reuse the exact decider/helper pattern from §3.7 with a second rule (`60-tboot-sbloader.actions`, `package_filter=systemd-boot-unsigned`) and a second decider/helper pair (`tboot-sbloader-posttrans` / `tboot-sbloader-helper`). Two ideas are specific to this chain and worth holding in mind:

- **The loopback both-target checkpoint.** Before touching the real ESP, the helper proves on a throwaway loopback vfat that `bootctl --no-variables install` actually picks the signed loader — a fail-closed dry run of the propagation primitive.
- **Convergence.** The baseline is "converged" only once the signed loader is on both ESP targets; a primed-but-not-yet-converged baseline deliberately reports drift, never "unchanged" (the D-1 no-op rule: baseline-equality alone is not enough; converged-shape is also required — `06F` N.6).

The install-vs-arm split and the two-chain co-fire that this produces are explained operationally in §4.6 below. Deep dive: `05` §3.5.

### 3.9 Governance, recovery, decommissioning (Module 4)

The lifecycle layer — key rotation (zero-downtime via dual keyslots), policy-key and CI/CD compromise response, TPM-failure / hardware-migration disaster recovery, separation of duties, audit logging, and decommissioning — is fully designed in `04`. Operationally it matters now mainly as the destination for the recovery procedures referenced from the gates (passphrase fallback, `--wipe-slot=tpm2` re-enrollment, decider rollback). Deep dive: `04`.

---

## 4. How to operate the runbook

The orientation above is about *what* the system is. This section is about *how* to move through `06B` without surprises.

### 4.1 The universal gate pattern

Almost every block in `06B` follows the same shape, and recognising it makes the long gate steps far easier to read:

1. **Read-only verification** — confirm the layer below is intact and the preconditions hold. No state changes.
2. **Snapshot** — take a Proxmox snapshot so the next action is reversible.
3. **Mutation** — the one state-changing action (enroll, sign, install, publish, reboot).
4. **Read-only re-verification** — confirm the mutation produced exactly the expected new state.
5. **Closure snapshot** — anchor the validated new state as the rollback target for the next block.

When a step looks like a wall of `bash`, it is almost always one of these five phases. The mutating phase is the cliff edge; everything around it is there to make that edge safe. Gates are run **one at a time**, output verified before proceeding — never batched.

### 4.2 Snapshot discipline

Snapshots are the safety net for the whole build. Take one before any expensive-to-redo or cliff-edge operation; name it for the gate it closes; and check the Proxmox thin-pool `Data%` is under ~75% before snapshotting (a full thin pool fails the snapshot and can wedge the VM). The authoritative lineage and the current preferred rollback targets are in `00_Current_Project_State.md`, not here — this note only explains *why* the discipline exists.

### 4.3 The Step 14 dry-run — what it can and cannot prove

ShellCheck and `bash -n` are syntactic checks: they prove the hook *source* parses, not that it *produces a valid UKI*. Three properties are runtime-only:

- whether `ukify build` actually emits a `.pcrsig` section,
- whether the resulting UKI passes section-VMA sanity (the firmware-rejection mode in `06F` B.1), and
- whether the TPM2 unseal in the boot phase actually releases the policy key.

The Step 14 dry-run exercises all three against a tmpfs synthetic staging area, driven exactly the way `kernel-install` would drive the hook, but without touching the ESP. It is the last checkpoint before the Step 15 cliff edge (installing the hook on the canonical path): a structurally valid forward-sealed UKI here means the hook can be installed with confidence; a failure surfaces in a throwaway directory instead of on the boot path.

#### Optional: section-by-section reference comparison

When a known-good MID-prefixed UKI from an earlier rebuild is already on the ESP (for example, when re-validating after editing the hook), the dry-run output can be compared section-by-section against it. This is the strongest validation short of a live transaction: byte-identical measured sections mean runtime PCR 11 will reach the same value the reference reached. All sections should sha256-match the reference; only the PE+ COFF `TimeDateStamp` and the Authenticode certificate table legitimately differ between runs (deterministic content, non-deterministic signing timestamps). The comparison loop (moved out of `06B` Step 14, which now points here) is reproduced below.

```bash
REFERENCE="/boot/efi/EFI/Linux/${MID}-${KVER}.efi"
for sec in text rodata data sbat sdmagic reloc osrel cmdline uname pcrpkey linux initrd pcrsig; do
  D=$(mktemp -d)
  cp "$DRY_UKI" "$D/dry.efi"; cp "$REFERENCE" "$D/ref.efi"
  objcopy --dump-section ".${sec}=$D/dry.${sec}" "$D/dry.efi" "$D/_d" 2>/dev/null
  objcopy --dump-section ".${sec}=$D/ref.${sec}" "$D/ref.efi" "$D/_r" 2>/dev/null
  if [ -s "$D/dry.${sec}" ] && [ -s "$D/ref.${sec}" ]; then
    A=$(sha256sum "$D/dry.${sec}" | awk '{print $1}')
    B=$(sha256sum "$D/ref.${sec}" | awk '{print $1}')
    [ "$A" = "$B" ] && echo "  .${sec} match" || echo "  .${sec} MISMATCH"
  fi
  rm -rf "$D"
done
```

`$DRY_UKI` is the dry-run UKI built in `06B` Step 14; `$REFERENCE` is the known-good MID-prefixed UKI already on the ESP.

### 4.4 Candidate selection for drift / no-drift validation (B.2.1, B.2.4)

Several gates need a "harmless package" to drive a real DNF transaction — either to confirm the plugin fires (B.2.1) or to drive the decider's drift branch (B.2.4). The selection criteria, applied read-only before any mutation, are:

- not currently installed (an already-installed package gives `dnf install --assumeno` odd resolver semantics — `06F` H.2);
- available in the enabled Fedora 43 repos;
- owns no files on any signing-pipeline, `/boot`, `/lib/modules`, or kernel-package path;
- pulls no trust-chain-sensitive dependencies.

For a pure trigger test (B.2.1) any such package works; the reference run used `figlet`. For a drift test (B.2.4) the package must additionally be **dracut-sensitive** (own files under `/usr/lib/dracut/modules.d/`) so dracut autodetect legitimately changes the host-only initramfs — the reference run used `dracut-network`, with `dracut-live` / `dracut-squash` / `dracut-tools-extra` as documented fallbacks. `06B` keeps the read-only selection probe inline (it is operational); this note records the methodology so a future operator can pick a fresh candidate correctly.

### 4.5 Canonical procedure vs live-run history

Some `06B` steps were validated on a live run that hit harness bugs (not source bugs) which were then diagnosed and corrected. The scripts inlined in `06B` are the **post-correction canonical form** — an operator following the runbook should not re-encounter those bugs. The historical detail (original FAIL numbers, the diagnostic path, the two-step corrections) is preserved in the Obsidian attempt-log subtree and in `06F`, deliberately *not* in the runbook, so the checklist stays clean.

Representative cases:

- **B.2.3 Step 33 (`06F` J.1–J.3):** the Gate 6A verifier was first written with a `/var/log/dnf5.log` timestamp-bound grep and a "lock path is absent" invariant. Both were wrong (journald is authoritative on this Fedora 43 host; `flock(1)` leaves the lock file in place on release). The canonical Gate 6A uses journald with `journalctl --since`, and a non-blocking `flock -n` probe with symlink rejection.
- **B.2.4 Step 34 (`06F` K.1):** the decider logs its production code path as `mode=normal`; the helper logs the *same* path as `mode=production`. A reference harness asserted `mode=normal` on the helper start line and failed at FAIL 411 even though the mutation succeeded. The canonical harness asserts `mode=production`; Gate 3 closure was confirmed by the post-hoc continuation verifier (43/43).

The lesson generalises: when an external audit harness fires on something the in-source self-check does not, fix the harness, not the validated source (see also `06F` H.4 — the Deviation-D token-boundary regex).

### 4.6 The two-chain co-fire (B.5 first fire), and why B.4 is split from B.5

This is the model behind the dangerous-mistake guard in `06B` Step 39. Read it before interpreting the Gate 3 transaction log.

Once B.5 arms the chain, two libdnf5-actions rules are active: `50-tboot-posttrans.actions` (B.2, empty `package_filter`, fires on *every* transaction) and `60-tboot-sbloader.actions` (B.4, `package_filter=systemd-boot-unsigned`). Because `systemd-boot-unsigned` is itself a boot-input-relevant package, the controlled first-fire transaction (`dnf reinstall systemd-boot-unsigned`) legitimately triggers **both** rules, in lexicographic priority order:

1. **B.2 (`50-`) fires first.** Its decider detects boot-input drift, writes its sentinel, and invokes `tboot-dnf-helper`, which rebuilds initramfs and re-signs the UKIs for both enumerated kernels via the §3.3 hook, refreshes the PCR 11 prediction, advances its baseline, and clears its sentinel. PCR 11 stays unchanged because the measured-content sections are bit-identical (`06F` E.3 / O.2).
2. **B.4 (`60-`) fires second.** Its decider sees drift against the non-converged primed baseline, invokes `tboot-sbloader-helper`, which signs `systemd-bootx64.efi` → `.efi.signed`, passes the loopback both-target checkpoint, propagates via `bootctl --no-variables install`, then advances the B.4 baseline to converged shape and clears its sentinel.

Two consequences the gates are built to handle: (a) the B.2 baseline and both ESP UKIs advance during Gate 3, so their pre-B.5 pinned hashes are superseded and Gate 4 re-pins them (`06F` O.3); (b) the rebuilt booted UKI's *file* hash changes while the *predicted* PCR 11 stays byte-identical (§3.4). The co-fire was not narrated in the original B.5 plan; it is **expected and benign** (`06F` O.3) — not a failure and not a reason to roll back.

**Why install + prime + mask (B.4) is split from arm + fire (B.5).** B.4 installs the chain, primes a deliberately **non-converged** baseline (`src_signed=ABSENT`), and masks the native `systemd-boot-update.service` — but leaves the `60-` rule unpublished. Publishing and firing it are B.5. The split exists to avoid a latent armed window over a non-converged baseline: if the rule were published at B.4 time, any unrelated `systemd-boot-unsigned` transaction before the operator was ready could fire the helper against a half-primed state. Arming and firing in one controlled, observed step (B.5) removes that window. The D-1 no-op rule (§3.8) is what makes the primed-but-non-converged state fail closed — see `06F` N.6.
