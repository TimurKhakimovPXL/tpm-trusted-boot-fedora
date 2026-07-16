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

**A2. Tampering with the UKI between signing and boot.**
Attack: the signed UKI is modified after signing. Defense: the UEFI firmware
rejects the file at `LoadImage()`. Evidence status: **captured**. See
`09_Forensic_Artifact_objcopy_UKI.md`: a UKI corrupted via `objcopy` passed the
Linux-side checks but was rejected by the firmware with:

```text
Error loading EFI binary \EFI\Linux\<id>.efi: Load Error
```

**A3. Replacement of the public policy key.**
Attack: the attacker replaces the public key enrolled in the LUKS2 keyslot.
Defense: the signature does not match the enrolled key, the TPM releases
nothing, and the unseal fails. Evidence status: **mechanism**. The standalone
run with captured output was not recovered.

## B. Automation layer (fail-closed chains)

**B1. Corruption of the helper sha.**
Attack: the helper script is modified. Defense: the install gate compares the
sha and refuses to deploy. Evidence status: **mechanism** (B.4 install logic,
exercised during the gates).

**B2. Mutation of the signing hook.**
Attack: the `kernel-install` hook is modified. Defense: the helper's preflight
refuses to run. Evidence status: **mechanism**.

**B3. Tampering with the production `.actions` line.**
Attack: the line that invokes the decider is modified or removed. Defense: the
decider is no longer invoked; the resulting mtime and sha drift is detectable by
the Module 4 monitoring. Evidence status: **mechanism**.

**B4. Bypassing drift detection (out-of-band baseline write).**
Attack: the attacker writes the baseline outside the chain to bypass drift
detection. Defense: the chain produces and validates the new runtime PCR 11
byte-for-byte after a real reboot under the helper-driven update workflow.
Evidence status: **captured**. Citable evidence from B.2.4: Gate 3 (reference
run), Gate 4 (independent prediction, byte match), Gate 6A (operator-confirmed
clean reboot) and Gate 6B (runtime PCR 11 byte match). Additional taxonomy: the
(a)/(b)/(c)/(d) outcomes from finding K.3 and the FAIL 411 →
continuation-verifier closure path from K.1 (`06F_Diagnostic_Findings_Catalog.md`).

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
- **B1, B2, B3:** the fail-closed guarantees follow from the chain logic and
  were exercised during the gates; standalone per-scenario tamper output was
  not captured as a separate block.
- **D1 (external command-line inputs):** not applicable to the validated
  configuration; future testing applies only to deployments that deliberately
  permit external PCR 12 inputs.
- **C1:** open residual risk; the mitigation is documented but not captured as
  a test.
