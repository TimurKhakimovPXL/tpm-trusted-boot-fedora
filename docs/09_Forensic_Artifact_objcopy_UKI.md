<!--
PUBLIC RELEASE VERSION - sanitized for publication.
Machine-id and filesystem/LUKS UUIDs are format-valid EXAMPLE values, not the
original lab identifiers. PCR values, policy-key digests and file hashes are
genuine measurements from the validated build and are public values by design.
-->

# Appendix 09: Forensic Artifact: objcopy-Corrupted UKI

> Reconstructed from the project logs of 8 May 2026. Companion to finding B.1 in
> `06F_Diagnostic_Findings_Catalog.md`.

## Description of the artifact

While developing the signing chain, a UKI was built in which the `.pcrsig`
section was added afterwards with `objcopy --add-section`, instead of in a
single `ukify` invocation. The result (`bad-objcopy-pcrsig-loaderror.efi`) is
preserved as forensic evidence of an important failure mode. The artifact lives
on the test rig under `/root/tboot-lab/failed-ukis/`. Its original location was
`/boot/efi/EFI/Linux/0123456789abcdef0123456789abcdef-6.19.14-200.fc43.x86_64.efi`.

## Symptom

The file passes every Linux-side check, but is rejected by the UEFI firmware at
`LoadImage()`. The fallback copy of the previous UKI booted correctly, proving
the firmware itself was fine and only the objcopy-built UKI was rejected.

Linux-side checks (all passing):

```text
.pcrpkey present
.pcrsig present
sbverify OK
signer count = 1
```

Firmware rejection at boot:

```text
Error loading EFI binary \EFI\Linux\<id>.efi: Load Error
```

## Forensic inspection (read-only)

Section table of the corrupted UKI (`objdump -h`):

```text
Idx Name       Size      VMA               LMA               File off  Algn
  0 .pcrsig    000002b7  0000000200000000  0000000200000000  ...
  1 .text      00000bff  000000014df92000  000000014df92000  ...
  2 .rodata    00000020  000000014df93000  ...
  ...
```

The virtual address of `.pcrsig` (`0x200000000`, roughly 8 GB) lies entirely
outside the contiguous run of low addresses occupied by `.text` (`0x14df92000`)
and the remaining sections. For a PE+ binary, `ImageBase` and the contiguous
section run determine what `LoadImage()` expects to reserve and map. A section
at this outlying address makes the file unloadable for the firmware.

Signature check on the same file:

```text
Signature verification OK
  signer: tboot-lab Signature Database Key
```

The Authenticode signature is valid because `sbsign` computed the hash over the
bytes that `objcopy` had written out. Structural correctness of the PE+ layout
is not part of the Authenticode hash domain, so the file passes signature
verification and is still unloadable.

## Conclusion

`objcopy --add-section .pcrsig=...` yields a UKI that passes Linux-side
validation (`sbverify`, `objdump -h`, `pesign --show-signature`) but is rejected
by UEFI `LoadImage()` due to an outlying section VMA. The correct approach is to
build the UKI in a single `ukify` invocation with the PCR-signing flags
included, so `.pcrsig` lands at a correct address and the whole image is signed
as one PE+ construction. This finding is the reason the production chain uses
only the native `ukify` path and no separate objcopy step.
