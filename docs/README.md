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
| `05_Update_Workflows_and_Key_Storage.md` | Update workflows and key custody |
| `06B_Golden_Path_Rebuild_Runbook.md` | Full step-by-step rebuild recipe with gates and stop conditions (large file; GitHub may not render it inline — view raw or clone) |
| `06C_Golden_Path_Operator_Notes.md` | Operator mental models: what each part of 06B does and why |
| `06F_Diagnostic_Findings_Catalog.md` | Every failure, bug and finding: symptom / root cause / reproduction / fix |
| `07_Runtime_Validation_Evidence.md` | Captured console evidence of the validated state (2026-05-28 snapshot) |
| `08_Attack_Scenarios.md` | Tamper scenarios with per-scenario evidence status, incl. open residual risks |
| `09_Forensic_Artifact_objcopy_UKI.md` | Post-mortem of the firmware-rejected objcopy UKI (finding B.1) |

## A note on internal references

These documents were extracted from a larger internal lab documentation set.
References to `00_Current_Project_State.md`, `06_Lab_Setup_Runbook*.md`,
`06X_Deprecated_Procedures.md` and `proxmox_technical_docs_v13.md` point to
internal lab documents that are **not included in this repository**: they track
live lab state (current PCR values, snapshot lineage, Proxmox host
configuration) and are not needed to understand the published architecture or
findings.

## Signing authority: pipeline model vs local hook

The architecture chapters (02, 03, 04) describe the signing authority
generically as a CI/CD pipeline holding the policy private key. The validated
lab implementation uses the local-hook variant instead: the `kernel-install`
hook and the `libdnf5-actions` chain in `dnf-actions/` sign on the machine at
update time. Chapter 05 compares the two models directly (see the "Who signs"
table there); the security roles are identical, only the place where signing
happens differs.
