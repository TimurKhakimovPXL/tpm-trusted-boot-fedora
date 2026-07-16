<!--
PUBLIC RELEASE VERSION - sanitized for publication.
Machine-id and filesystem/LUKS UUIDs are format-valid EXAMPLE values, not the
original lab identifiers. PCR values, policy-key digests and file hashes are
genuine measurements from the validated build and are public values by design.
-->

# Appendix 08: Attack Scenarios

> Reconstructed from the project logs (21 and 28 May 2026). The chain was tested
> against live enforcement, drift detection and reboot validation. Each scenario
> lists its evidence status: **captured** (real output available), **mechanism**
> (guaranteed by the chain logic and exercised during the gates; no separate
> tamper output captured), **residual risk** (identified weakness, not a defeated
> scenario) or **future** (designed, not yet executed).

## A. Enforcement layer (boot and disk unlock)

**A1. Tampering with the PCR 7 input (Secure Boot state).**
Attack: the Secure Boot state is changed (in the test, by disabling Secure Boot
in the UEFI firmware). Defense: the PCR 7 value diverges from the sealed
prediction, the TPM does not release the secret, and the disk falls back to the
passphrase prompt. Evidence status: **captured**. With Secure Boot disabled,
PCR 7 became `DEBA227BF2493EC6DC5311D8838D42A4AEE572643AB8B808B0B1F06B1FF589CF`,
against the baseline
`BF8878BD8B70654924BD39CA71D770F556F03558AC200BAD46A9CEB5A2B596BD`. The TPM
refused the unseal and the boot prompted for the passphrase:

```text
systemd-cryptsetup[450]: TPM policy does not match current system state.
Either system has been tempered with or policy out-of-date: Operation not permitted
Please enter passphrase for disk QEMU_HARDDISK (luks-11111111-...-555555555555)::
```

The boot did reach `tpm2.target` and found `/dev/tpm0` (SRK fingerprint
`ac77a6a7e3c971160c78dbe65144c1df0977e5d85ab8469fac242710c0aa2d9d`): the TPM was
present and working, but withheld the key because the measured state was wrong.
After entering the passphrase (keyslot 0) the system started normally. After
re-enabling Secure Boot, PCR 7 returned to the baseline and the TPM unlocked
again without a passphrase prompt.

**A2. Firmware rejection of a malformed but signed UKI.**
Test: `objcopy` added a section using a layout that produced invalid PE/COFF
section addresses. The resulting file was then signed with `sbsign`. Linux-side
signature checks passed, but the UEFI firmware rejected it at `LoadImage()`.
Evidence status: **captured**. See `09_Forensic_Artifact_objcopy_UKI.md`.

This test demonstrates that a valid Authenticode signature does not guarantee
that firmware can load a structurally malformed image. It is not evidence of a
post-signing tamper test; changing the signed bytes after `sbsign` would instead
invalidate the Authenticode signature.

```text
Error loading EFI binary \EFI\Linux\<id>.efi: Load Error
```

**A3. Replacement of the public policy key.**
Attack: the attacker replaces the public key enrolled in the LUKS2 keyslot.
Defense: the signature does not match the enrolled key, the TPM releases
nothing, and the unseal fails. Evidence status: **mechanism**. The standalone
run with captured output was not recovered.

## B. Automation layer (fail-closed chains)

**B1. Corruption of a staged helper.**
Attack: a helper is modified before installation. Defense: the installation
gate compares the staged sha256 with the reviewed value and refuses to deploy a
mismatch. Evidence status: **mechanism** (exercised during installation).

This check does not provide continuous runtime integrity. A helper modified
after installation is executed by path unless another host-integrity control
detects it. No such monitoring service is shipped in this repository.

**B2. Mutation of the signing hook.**
Attack: the `kernel-install` hook is modified. Defense: the helper's preflight
refuses to run. Evidence status: **mechanism**.

**B3. Tampering with the production `.actions` line.**
Attack: the line that invokes the decider is modified or removed. Result: the
decider may no longer run. The repository provides install-time hashes and
audit commands, but it does not ship continuous monitoring for the installed
action file. Evidence status: **residual risk**.

**B4. Out-of-band baseline modification.**
Attack: a privileged process writes a new baseline outside the decider flow and
causes later drift to appear authorised. The normal reboot validation proves
that a legitimate helper-driven update converged, but it does not prevent or
detect a privileged baseline rewrite. The B.2 state directory rejects symlinks
and requires root ownership with mode 0700. The B.4 chain now applies the same
checks. These controls reduce accidental and unprivileged modification; they do
not protect against a compromised root account. Evidence status: **residual
risk**.

## C. Residual risk

**C1. Substitution of a forensic UKI on the ESP (finding L.1).**
The fallback selection becomes attacker-controlled when `LoaderEntryDefault` is
empty and loadable forensic UKIs sit on the ESP alongside the production entry.
This is an identified residual risk, not a defeated scenario. Mitigation: ESP
audit discipline (rename forensic UKIs or remove them from the boot menu).
Evidence status: **residual risk**.

## D. Future scenario

**D1. External command-line injection in a deployment that permits it.**
The exact boot-menu attack in which an attacker appends `init=/bin/sh` is not
reachable in the validated configuration. systemd-boot editing is disabled,
Secure Boot is enabled, and the UKI contains an embedded `.cmdline`; under
those conditions systemd-stub does not accept an externally supplied
replacement command line.

Deployments that allow external command-line arguments, credentials,
extensions or add-ons may still need PCR 12. Binding those deployments to PCR
12 is possible future work. It was not needed or tested in this configuration.
Evidence status: **future / deployment-dependent**.

## What is missing

- **A3 (pubkey replacement):** the mechanism is demonstrated, but a standalone
  tamper run with captured console output has not been executed. A1 (PCR 7
  tamper) has since been captured (see above).
- **B1, B2:** the install-time and preflight mechanisms were exercised during
  the gates; standalone tamper output was not captured as a separate block.
- **B3, B4:** continuous action-file monitoring and protection against a
  compromised root account are outside the shipped implementation.
- **D1 (external command-line inputs):** not applicable to the validated
  configuration; future testing applies only to deployments that deliberately
  permit external PCR 12 inputs.
- **C1:** open residual risk; the mitigation is documented but not captured as
  a test.
