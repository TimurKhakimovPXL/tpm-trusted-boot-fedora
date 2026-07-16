# TPM-Enforced Trusted Boot Chain on Fedora / RHEL

My thesis project for the associate degree (graduaat) in System and Network
Administration at PXL University of Applied Sciences, built with an industry
partner: a trusted boot chain on Fedora Server 43 that keeps working when you
update the system. Custom Secure Boot keys, signed Unified Kernel Images, TPM2
forward sealing and LUKS2 auto-unlock, held together by a DNF plugin chain that
re-signs the boot path whenever a package transaction touches it. The LUKS
keyslot is never re-enrolled.

RHEL is the design target; Fedora 43 is what it was validated on.

## Why

TPM-sealed disk encryption is well documented for static systems and breaks on
mutable ones. Run a kernel update and the measured boot components change, PCR
values shift, and the TPM refuses to release the LUKS key on the next boot. You
either get a passphrase prompt on a headless server, or an operator "fixes" it
by re-enrolling the keyslot and quietly weakens the setup every time.

The usual answers are freezing the kernel, moving to an immutable image-based
OS, or accepting manual re-sealing after every update. None of those fit a
normal package-managed server, so this project takes a different route.

## How it works

Five layers:

| Layer | Mechanism |
|---|---|
| 1. Secure Boot | Custom PK/KEK/db, manufacturer keys removed, no shim |
| 2. Boot artifact | Signed UKIs built with `ukify`, driven by a `kernel-install` hook |
| 3. Forward sealing | PCR 11 signed policy (PolicyAuthorize) via a TPM-sealed RSA-3072 key |
| 4. Disk encryption | LUKS2 split policy: PCR 7 (static) + PCR 11 (signed) |
| 5. Governance | `libdnf5-actions` decider/helper chain that re-signs on drift |

Layer 5 is the interesting one. A decider plugin runs inside every RPM
transaction and checks whether the transaction touched the boot chain. If it
did, a helper rebuilds the initramfs, re-signs the UKI and refreshes the signed
PCR 11 prediction. Because the LUKS policy is signed rather than pinned to a
literal PCR value, a fresh prediction is all it takes. The keyslot itself is
never touched.

The chain is fail-closed: if any step of the re-signing fails, the transaction
is blocked instead of leaving behind a system that will not unlock on reboot.

## What was validated

- A full `dnf update` including a kernel version upgrade, the hardest drift
  case. Drift detected, UKI rebuilt and re-signed, PCR 11 prediction refreshed,
  baseline advanced, all without operator input.
- Reboot after that update: no passphrase prompt, runtime PCR 11 matched the
  stored prediction exactly, the TPM released the LUKS key, and the keyslot was
  not re-enrolled.
- The passphrase fallback still works as a recovery path.

Captured console evidence is in
[docs/07_Runtime_Validation_Evidence.md](docs/07_Runtime_Validation_Evidence.md),
tamper scenarios in [docs/08_Attack_Scenarios.md](docs/08_Attack_Scenarios.md).

## Runtime demonstration

A recorded validation run shows the DNF decider/helper chain, the canonical
`80-tpm2-sign.install` hook, UKI rebuilding and signing, PCR 11 prediction
refresh, sentinel clearing, Secure Boot and measured-UKI verification, and a
reboot into the signed UKI.

[Watch the runtime trusted-boot demonstration](validation/tpm-trusted-boot-runtime-demo.mp4).

The video is an original laboratory console capture. Text documents in this
repository use sanitized machine identifiers.

## Thesis report

The English public edition of the graduation project report is available in
both publication and editable formats:

- [English thesis report (PDF)](docs/thesis/Timur_Khakimov_TPM_Trusted_Boot_Thesis_EN.pdf)
- [English thesis source (DOCX)](docs/thesis/source/Timur_Khakimov_TPM_Trusted_Boot_Thesis_EN.docx)

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
hooks/          80-tpm2-sign.install - kernel-install hook: build and sign UKI
dnf-actions/    libdnf5-actions decider/helper chain, both boot-loader paths
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
