# TPM-Enforced Trusted Boot Chain on Fedora / RHEL

This thesis project, completed for the associate degree (graduaat) in System
and Network Administration at PXL University of Applied Sciences in
collaboration with Piros, demonstrates how an encrypted Fedora Server 43
system can install legitimate operating-system updates and retain TPM-backed
automatic LUKS2 unlock at the next boot. Signed Unified Kernel Images (UKIs)
are the central boot artifact in the design. Each UKI combines the kernel,
initramfs, embedded command line, operating-system metadata and TPM2 PCR policy
material in one measured and Secure Boot-signed file. A DNF5 actions chain
detects boot-input drift, rebuilds and signs the UKIs, and refreshes the
expected PCR 11 state. Unauthorized boot changes are rejected, while approved
updates advance the trusted state without re-enrolling the LUKS keyslot.

## Research question and conclusion

**Research question:** Can a mutable, package-managed Fedora server retain
TPM-backed automatic LUKS2 unlock across kernel, initramfs and bootloader
updates without re-enrolling the LUKS keyslot?

**Conclusion:** Yes. The project demonstrates that this is feasible by
combining static PCR 7 binding, signed PCR 11 authorization, boot-input drift
detection and transaction-triggered convergence of the UKI and bootloader
artifacts. Legitimate updates can advance the trusted state without changing
the enrolled LUKS policy key.

Fedora Server 43 was the validated platform. RHEL was treated as the
architectural design target but was not independently tested.

## Why

Static PCR-bound TPM auto-unlock becomes brittle on mutable systems because
legitimate kernel, initramfs and bootloader updates change the measured state.
The PCR values shift, and the TPM refuses to release the LUKS key on the next
boot. You either get a passphrase prompt on a headless server, or an operator
"fixes" it by re-enrolling the keyslot and quietly weakens the setup every
time.

The usual answers are freezing the kernel, moving to an immutable image-based
OS, or accepting manual re-sealing after every update. None of those fit a
normal package-managed server, so this project takes a different route.

## Original contribution

Secure Boot, UKIs, TPM2 PolicyAuthorize, `systemd-measure` and
`systemd-cryptenroll` are existing platform mechanisms. The contribution of
this project is their integration into an update-resilient workflow for a
mutable Fedora system:

- A deterministic boot-input manifest covering fourteen categories of input.
- A DNF5 decider/helper architecture that separates decisions from privileged
  changes.
- A canonical UKI rebuild and signing hook.
- Independent PCR 11 prediction and verification.
- A separate systemd-boot signing and propagation chain with a loopback-ESP
  safety checkpoint.
- Baseline advancement only after the boot artifacts have converged.
- An unsafe-reboot state that remains set after a failed rebuild.
- Validation across a full system update and reboot without LUKS keyslot
  re-enrollment.

## How it works

Five layers:

| Layer | Mechanism |
|---|---|
| 1. Secure Boot | Custom PK/KEK/db, manufacturer keys removed, no shim |
| 2. Boot artifact | Signed UKIs built with `ukify`, driven by a `kernel-install` hook |
| 3. Forward sealing | PCR 11 signed policy (PolicyAuthorize) via a TPM-sealed RSA-3072 key |
| 4. Disk encryption | LUKS2 split policy: PCR 7 (static) + PCR 11 (signed) |
| 5. Update governance and lifecycle | DNF5 convergence automation, key lifecycle, recovery and audit controls |

A `libdnf5-actions` rule runs the decider after each completed host transaction.
The decider compares the current boot-input manifest with its stored baseline.
If they differ, the helper rebuilds the initramfs and UKIs, runs the signing
hook and refreshes the expected PCR 11 value. The LUKS policy accepts PCR states
signed by the policy key, so this process does not touch the LUKS keyslot.

The action runs in `post_transaction`, after RPM has committed the package
changes. A signing failure cannot roll back those changes. DNF returns an
error, the baseline stays unchanged, and the `UNSAFE-TO-REBOOT` file remains
until the boot artifacts have been repaired. This file warns the operator; it
does not prevent the machine from rebooting.

## Methodology

The system was implemented and tested on Fedora Server 43 in a Proxmox virtual
machine using UEFI Secure Boot and a virtual TPM 2.0. Development followed an
incremental, gate-based process:

1. Establish custom Secure Boot and signed UKI boot.
2. Enroll the LUKS2 split policy.
3. Validate TPM unlock and PCR prediction.
4. Introduce DNF-driven boot-input drift detection.
5. Automate UKI and systemd-boot convergence.
6. Test update, reboot, tamper and recovery paths.

Success required a completed update, a clean reboot without a passphrase,
runtime PCR 11 matching the stored prediction, no LUKS keyslot re-enrollment
and preservation of the recovery credential.

## What was validated

- A full `dnf update` including a kernel version upgrade, the hardest drift
  case. Drift detected, UKI rebuilt and re-signed, PCR 11 prediction refreshed,
  baseline advanced, all without operator input.
- Reboot after that update: no passphrase prompt, runtime PCR 11 matched the
  stored prediction exactly, the TPM released the LUKS key, and the keyslot was
  not re-enrolled.
- The passphrase fallback still works as a recovery path.

Those results apply to the May and June 2026 lab revision. The source published
on 16 July 2026 adds stricter sentinel and state-directory failure handling and
has passed repository integrity, shell syntax and embedded-copy checks. A fresh
live transaction and reboot regression run for that source revision is still
pending.

Captured console evidence is in
[docs/07_Runtime_Validation_Evidence.md](docs/07_Runtime_Validation_Evidence.md),
tamper scenarios in [docs/08_Attack_Scenarios.md](docs/08_Attack_Scenarios.md).

## Threat model

The design addresses offline disk access, unauthorized pre-boot modification,
Secure Boot policy changes and unapproved changes to the kernel, initramfs or
embedded command line.

It does not protect a system after an attacker has obtained root access in the
running operating system. In the validated local-signing design, root access
can expose or invoke the signing authority, modify baselines and alter the
installed automation. Physical hardware behavior, customer package streams
and RHEL were outside the validated scope.

## Runtime demonstration

A recorded validation run shows the DNF decider/helper chain, the canonical
`80-tpm2-sign.install` hook, UKI rebuilding and signing, PCR 11 prediction
refresh, sentinel clearing, Secure Boot and measured-UKI verification, and a
reboot into the signed UKI.

[Watch the runtime trusted-boot demonstration](validation/tpm-trusted-boot-runtime-demo.mp4).

The video is an original laboratory console capture of the controlled package
reinstall and reboot path. The later unrestricted full `dnf update` result is
summarized above; its raw terminal transcript is not included. Text documents
in this repository use sanitized machine identifiers. See the
[runtime evidence appendix](docs/07_Runtime_Validation_Evidence.md#4-published-runtime-demonstration)
for the recording's evidence scope.

## Thesis report

The English public edition of the graduation project report is available in
both publication and editable formats:

- [English thesis report (PDF)](docs/thesis/Timur_Khakimov_TPM_Trusted_Boot_Thesis_EN.pdf)
- [English thesis source (DOCX)](docs/thesis/source/Timur_Khakimov_TPM_Trusted_Boot_Thesis_EN.docx)

### Where the thesis appendices live

The thesis lists seven appendices, submitted separately with the graded report. Their published counterparts in this repository:

| Thesis appendix | In this repo |
|---|---|
| 1: Build-time architecture diagram | `docs/thesis/figures/figure1_build_time_flow.png` |
| 2: Boot-time architecture diagram | `docs/thesis/figures/figure2_boot_time_flow.png` |
| 3a: kernel-install hook | `hooks/80-tpm2-sign.install` |
| 3b: first fail-closed chain (UKI) | `dnf-actions/tboot-dnf-posttrans`, `tboot-dnf-helper`, `tboot-predict-pcr11`, `50-tboot-posttrans.actions` |
| 3c: second fail-closed chain (systemd-boot) | `dnf-actions/tboot-sbloader-posttrans`, `tboot-sbloader-helper`, `60-tboot-sbloader.actions` |
| 4: Runtime validation CLI output | `docs/07_Runtime_Validation_Evidence.md` |
| 5: Diagnostic Findings Catalog | `docs/06F_Diagnostic_Findings_Catalog.md` |
| 6: Attack-scenario results | `docs/08_Attack_Scenarios.md` |
| 7: Forensic artifact `bad-objcopy-pcrsig-loaderror.efi` | Write-up in `docs/09_Forensic_Artifact_objcopy_UKI.md`; the firmware-rejected binary itself is not published |

## Findings along the way

Everything that broke is written up in
[docs/06F_Diagnostic_Findings_Catalog.md](docs/06F_Diagnostic_Findings_Catalog.md)
with symptom, root cause, reproduction and fix per entry. Some highlights:

- `objcopy --add-section` produces UKIs that pass `sbverify` but get rejected
  by UEFI `LoadImage()` because of a corrupt section VMA. A valid Authenticode
  signature says nothing about whether the firmware will load the file. Full
  post-mortem in
  [docs/09_Forensic_Artifact_objcopy_UKI.md](docs/09_Forensic_Artifact_objcopy_UKI.md).
- `ukify --join-pcrsig` silently does nothing on systemd 258.7. It exits zero.
- PCR 11 stays constant across UKI regeneration as long as dracut's inputs are
  unchanged. The file hash changes on every build (RSA-PSS salt, signing
  timestamp); the measured content does not.
- systemd's PolicyAuthorize path needs an RSA policy key. EC P-256 fails at
  unseal with VerifySignature error 0x2d2.

## Repository layout

```
docs/           Architecture (00-05), findings catalog (06F), rebuild
                runbook (06B), operator notes (06C), evidence appendices (07-09)
docs/thesis/    English thesis PDF, editable source and architecture figures
hooks/          80-tpm2-sign.install - kernel-install hook: build and sign UKI
dnf-actions/    libdnf5-actions decider/helper chain, both boot-loader paths
validation/     Published runtime demonstration
SHA256SUMS      Integrity manifest for the shipped scripts
```

Start with [docs/README.md](docs/README.md) for the reading order.

## Limitations

- Validated on Fedora Server 43 in a VM with a virtual TPM. RHEL is a design
  target, not a tested platform.
- Not validated on physical hardware, where firmware PCR behaviour differs.
- Not validated against a customer package stream.
- This is thesis work, not a supported product. Read the runbook before
  pointing any of it at a machine you care about.

## Sanitization note

Machine-id, host names, IP addresses and filesystem/LUKS UUIDs in these
documents are format-valid example values, not the original lab identifiers.
PCR values, policy-key digests and file hashes are genuine measurements from
the validated build and are public values by design.

## License

Apache-2.0, see [LICENSE](LICENSE).
