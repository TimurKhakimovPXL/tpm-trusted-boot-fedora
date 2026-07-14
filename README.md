# TPM-Enforced Trusted Boot Chain on Fedora / RHEL

A hardware-backed, fail-closed trusted boot chain for package-managed Linux: custom
Secure Boot keys, signed Unified Kernel Images, TPM2 forward sealing, and LUKS2
auto-unlock — all of it surviving arbitrary `dnf` transactions without manual
intervention and without re-enrolling the LUKS keyslot.

Built and validated on Fedora Server 43 (RHEL as design target) as a graduaat thesis
in System and Network Administration at PXL University of Applied Sciences, in
collaboration with an industry partner.

---

## The problem

Measured boot with TPM-sealed disk encryption is well documented for *static* systems.
It breaks the moment you run a package manager: a kernel update changes the measured
boot components, PCR values shift, the TPM refuses to release the LUKS key, and the
machine drops to a passphrase prompt on next reboot — or worse, an operator "fixes" it
by re-enrolling the keyslot, quietly weakening the security posture every time.

The usual answers are to freeze the kernel, to use an immutable image-based OS, or to
accept manual re-sealing after every update. None of those are acceptable on a mutable,
package-managed enterprise server.

## The approach

Five layers, each enforcing the next:

| Layer | Mechanism |
|---|---|
| 1. Secure Boot | Custom PK/KEK/db; manufacturer keys removed; no shim |
| 2. Boot artifact | Signed UKIs built via `ukify`, driven by a `kernel-install` hook |
| 3. Forward sealing | PCR 11 **signed policy** (PolicyAuthorize) via a TPM-sealed RSA-3072 key |
| 4. Disk encryption | LUKS2 split policy: PCR 7 (static) + PCR 11 (signed) |
| 5. Governance | `libdnf5-actions` decider/helper chain driving fail-closed re-signing |

The load-bearing idea is layer 5. A **decider** plugin runs inside every RPM transaction,
detects drift in the boot chain, and writes a sentinel; a **helper** then rebuilds the
initramfs, re-signs the UKI, and refreshes the signed PCR 11 prediction. Because the
policy is *signed* rather than pinned to a literal PCR value, a new prediction is enough
— the LUKS keyslot is never touched.

Fail-closed: if any step of the re-signing chain fails, the transaction is blocked rather
than silently producing a system that will not unlock on reboot.

## What was validated

- Full un-gated `dnf update` **including a kernel version upgrade** — the hardest drift
  class — handled end to end: drift detected, UKI rebuilt and re-signed, PCR 11
  prediction refreshed, sentinel cleared, baseline advanced.
- Reboot after that update: **no passphrase prompt**, runtime PCR 11 matched the stored
  prediction exactly, TPM2 released the LUKS key, **no keyslot re-enrollment**.
- Passphrase fallback confirmed available as recovery path.

## Original findings

The diagnostic catalog ([`docs/06F_Diagnostic_Findings_Catalog.md`](docs/06F_Diagnostic_Findings_Catalog.md))
documents every finding with symptom / root cause / reproduction / fix. Selected
highlights:

- **`objcopy --add-section` produces a firmware-rejected UKI** (B.1). The result passes
  `sbverify` but is rejected by `LoadImage` due to a corrupt section VMA. A signature
  check is not a validity check — this failure mode is invisible to the obvious test.
- **`ukify --join-pcrsig` silently no-ops** on systemd 258.7 (B.2). Exits zero, does
  nothing.
- **PCR 11 is invariant across UKI regeneration** when dracut's inputs are unchanged
  (E.3): the UKI's own hash changes (RSA-PSS salt, fresh signing timestamp) while the
  measured-content hash does not.
- **systemd's PolicyAuthorize path requires an RSA policy key**; EC P-256 fails at unseal
  with `VerifySignature` error `0x2d2`.

## Repository layout

```
docs/           Architecture (00-05), diagnostic findings catalog (06F),
                golden-path rebuild runbook (06B), operator notes (06C),
                runtime validation / attack scenarios / forensic appendices
hooks/          80-tpm2-sign.install   — kernel-install hook: build + sign UKI
dnf-actions/    libdnf5-actions decider/helper chain (both boot-loader paths)
SHA256SUMS      Integrity manifest for all shipped scripts
```

Start with [`docs/00_Introduction.md`](docs/00_Introduction.md) for the architecture, or
[`docs/06F_Diagnostic_Findings_Catalog.md`](docs/06F_Diagnostic_Findings_Catalog.md) if
you want the engineering substance first.

## Scope and honest limitations

- Validated on **Fedora Server 43 in a VM with a virtual TPM**. RHEL is a design target,
  not a deployed-and-tested platform.
- **Not validated on physical hardware**, where firmware PCR behaviour differs.
- Not validated against a customer package stream.
- This is thesis work, not a supported product. Read the runbook before running anything
  against a system you care about.

## Sanitization note

Machine-id, host names, IP addresses and filesystem/LUKS UUIDs in these documents are
format-valid **example** values, not the original lab identifiers. PCR values, policy-key
digests and file hashes are genuine measurements from the validated build and are public
values by design.

## Licence

Apache-2.0. See [LICENSE](LICENSE).
