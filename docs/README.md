# Documentation map

Suggested reading order for a first pass: `00` → `06C` → `06F`, then `06B` if
you intend to reproduce the build.

| File | Contents |
|---|---|
| `00_Introduction.md` | Problem statement and five-layer architecture overview |
| `01_Unified_Kernel_Image.md` | UKI construction and signing (layer 2) |
| `02_Forward_Sealing.md` | PCR 11 signed-policy forward sealing (layer 3) |
| `03_Disk_Encryption_and_Policy_Enforcement.md` | LUKS2 split policy, PCR 7 + PCR 11 (layer 4) |
| `04_Governance_Recovery_Lifecycle.md` | Drift detection, fail-closed governance, recovery (layer 5) |
| `05_Update_Workflows_and_Key_Storage.md` | Validated DNF5 update workflow, systemd-boot signing and key custody |
| `06B_Golden_Path_Rebuild_Runbook.md` | Reference step-by-step rebuild with gates and stop conditions; site-specific lab state is required (large file; GitHub may not render it inline) |
| `06C_Golden_Path_Operator_Notes.md` | Operator guidance: what each part of 06B does and why |
| `06F_Diagnostic_Findings_Catalog.md` | Every failure, bug and finding: symptom / root cause / reproduction / fix |
| `07_Runtime_Validation_Evidence.md` | Captured console evidence of the validated state (2026-05-28 snapshot) |
| `08_Attack_Scenarios.md` | Tamper scenarios with per-scenario evidence status, incl. open residual risks |
| `09_Forensic_Artifact_objcopy_UKI.md` | Post-mortem of the firmware-rejected objcopy UKI (finding B.1) |

## A note on internal references

These documents were extracted from a larger internal lab documentation set.
References to `00_Current_Project_State.md`, `06_Lab_Setup_Runbook*.md`,
`06X_Deprecated_Procedures.md` and `proxmox_technical_docs_v13.md` point to
internal lab documents that are **not included in this repository**. They track
live lab state, including current PCR values, snapshot lineage and Proxmox host
configuration. The published material is sufficient to review the architecture
and validated artifacts, but gates that depend on those live values cannot be
replayed verbatim without supplying equivalent site-specific state.

## Signing authority: pipeline model vs local hook

The architecture chapters (02, 03, 04) describe the signing authority
generically as a CI/CD pipeline holding the policy private key. The validated
lab implementation uses the local-hook variant instead: the `kernel-install`
hook and the `libdnf5-actions` chain in `dnf-actions/` sign on the machine at
update time. These models do not have the same compromise boundary. In the
validated local design, an approved running host can access the plaintext
Secure Boot signing key and ask the TPM to decrypt the PCR-policy signing key.
A remote signing service can enforce an independent approval boundary, but only
if it validates requests rather than signing arbitrary endpoint-supplied data.
Chapter 05 compares the operational choices and states which alternatives are
implemented here.
