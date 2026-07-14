<!--
PUBLIC RELEASE VERSION - sanitized for publication.
Machine-id and filesystem/LUKS UUIDs are format-valid EXAMPLE values, not the
original lab identifiers. PCR values, policy-key digests and file hashes are
genuine measurements from the validated build and are public values by design.
-->

# Bijlage 6: Aanvalsscenario's

> Gereconstrueerd uit de projectlogboeken (21 mei en 28 mei 2026). De keten is getoetst aan
> draaiende enforcement, driftdetectie en rebootvalidatie. Per scenario staat de bewijsstatus
> vermeld: **vastgelegd** (reële uitvoer beschikbaar), **mechanisme** (door de ketenlogica
> gegarandeerd en tijdens de gates uitgeoefend, geen aparte tamper-uitvoer vastgelegd),
> **restrisico** (geïdentificeerde zwakte, geen afgeweerd scenario) of **toekomstig**
> (ontworpen, nog niet uitgevoerd). CLI-uitvoer staat in het Engels.

## A. Enforcement-laag (boot en schijfontgrendeling)

**A1. Manipulatie van PCR 7-invoer (Secure Boot-toestand).**
Aanval: de Secure Boot-toestand wordt gewijzigd (in de test door Secure Boot in de
UEFI-firmware uit te schakelen). Verdediging: de waarde van PCR 7 wijkt af van de verzegelde
voorspelling, de TPM geeft het geheim niet vrij en de schijf valt terug op de wachtwoordprompt.
Bewijsstatus: **vastgelegd**. Met Secure Boot uitgeschakeld werd PCR 7
`DEBA227BF2493EC6DC5311D8838D42A4AEE572643AB8B808B0B1F06B1FF589CF`, tegenover de baseline
`BF8878BD8B70654924BD39CA71D770F556F03558AC200BAD46A9CEB5A2B596BD`. De TPM weigerde de unseal en
de boot vroeg om het wachtwoord:

```text
systemd-cryptsetup[450]: TPM policy does not match current system state.
Either system has been tempered with or policy out-of-date: Operation not permitted
Please enter passphrase for disk QEMU_HARDDISK (luks-11111111-...-555555555555)::
```

De boot bereikte wel `tpm2.target` en vond `/dev/tpm0` (SRK-fingerprint
`ac77a6a7e3c971160c78dbe65144c1df0977e5d85ab8469fac242710c0aa2d9d`): de TPM was aanwezig en
werkte, maar gaf de sleutel niet vrij omdat de gemeten toestand niet klopte. Na invoer van het
wachtwoord (keyslot 0) startte het systeem normaal. Na het opnieuw inschakelen van Secure Boot
keerde PCR 7 terug naar de baseline en ontgrendelde de TPM weer zonder wachtwoordprompt.

**A2. Manipulatie van het UKI tussen ondertekenen en opstarten.**
Aanval: het ondertekende UKI wordt na het ondertekenen gewijzigd. Verdediging: de UEFI-firmware
weigert het bestand bij `LoadImage()`. Bewijsstatus: **vastgelegd**. Zie bijlage 7: een via
`objcopy` beschadigd UKI doorstond de Linux-zijdige controles maar werd door de firmware
geweigerd met:

```text
Error loading EFI binary \EFI\Linux\<id>.efi: Load Error
```

**A3. Vervanging van de publieke beleidssleutel (policy key).**
Aanval: de aanvaller vervangt de publieke sleutel die in de LUKS2-keyslot is ingeschreven.
Verdediging: de handtekening matcht de ingeschreven sleutel niet, de TPM laat niets vrij en de
unseal mislukt. Bewijsstatus: **mechanisme**. De afzonderlijke run met vastgelegde uitvoer
is niet teruggevonden.

## B. Automatiseringslaag (fail-closed ketens)

**B1. Corruptie van de helper-sha.**
Aanval: het helperscript wordt gewijzigd. Verdediging: de install-gate vergelijkt de sha en
weigert te deployen. Bewijsstatus: **mechanisme** (B.4-installatielogica, uitgeoefend tijdens
de gates).

**B2. Mutatie van de signing-hook.**
Aanval: de `kernel-install`-hook wordt gewijzigd. Verdediging: de preflight van de helper
weigert te draaien. Bewijsstatus: **mechanisme**.

**B3. Manipulatie van de productie-`.actions`-regel.**
Aanval: de regel die de decider aanroept wordt gewijzigd of verwijderd. Verdediging: de decider
wordt niet meer aangeroepen; de mtime- en sha-drift is detecteerbaar door de monitoring uit
Module 4. Bewijsstatus: **mechanisme**.

**B4. Omzeilen van de driftdetectie (out-of-band baseline-write).**
Aanval: de aanvaller schrijft de baseline buiten de keten om, om driftdetectie te omzeilen.
Verdediging: de keten produceert en valideert de nieuwe runtime-PCR 11 byte-voor-byte na een
echte reboot onder de helper-gestuurde update-workflow. Bewijsstatus: **vastgelegd**.
Citeerbaar bewijs uit B.2.4: Gate 3 (referentierun), Gate 4 (onafhankelijke voorspelling,
byte-match), Gate 6A (door de operator bevestigde schone reboot) en Gate 6B (runtime-PCR 11
byte-match). Aanvullende taxonomie: de (a)/(b)/(c)/(d)-uitkomsten uit bevinding K.3 en het
FAIL 411 → continuation-verifier-sluitpad uit K.1 (bijlage 5).

## C. Restrisico

**C1. Substitutie van een forensisch UKI op de ESP (bevinding L.1).**
De fallback-selectie wordt aanvaller-gestuurd wanneer `LoaderEntryDefault` leeg is en er naast
de productie-entry laadbare forensische UKI's op de ESP staan. Dit is een geïdentificeerd
restrisico, geen afgeweerd scenario. Mitigatie: ESP-auditdiscipline (forensische UKI's
hernoemen of verwijderen uit het boot-menu). Bewijsstatus: **restrisico**.

## D. Toekomstig scenario

**D1. Cmdline-injectie aan de bootloaderprompt.**
Aanval: de aanvaller voegt `init=/bin/sh` toe aan de cmdline. Verdediging (huidige policy,
PCR 7+11): omzeild: de TPM ontgrendelt onder de gewijzigde cmdline. Verdediging (voorgesteld,
PCR 4+7+11+12): geblokkeerd: PCR 12 wijkt af en de policy-handtekening faalt. Bewijsstatus:
**toekomstig**: vereist uitbreiding van de policy naar PCR 4+7+11+12; in deze iteratie niet
uitgevoerd.

## Wat ontbreekt

- **A3 (pubkey-vervanging):** het mechanisme is aangetoond, maar een afzonderlijke tamper-run
  met vastgelegde console-uitvoer is nog niet uitgevoerd. A1 (PCR 7-tamper) is intussen wel
  vastgelegd (zie hierboven).
- **B1, B2, B3:** de fail-closed garanties volgen uit de ketenlogica en zijn tijdens de gates
  uitgeoefend; aparte, op zichzelf staande tamper-uitvoer per scenario is niet als los blok
  vastgelegd.
- **D1 (cmdline-injectie):** ontworpen maar niet uitgevoerd; vereist eerst de
  policy-uitbreiding naar PCR 4+7+11+12.
- **C1:** open restrisico; de mitigatie is gedocumenteerd maar niet als test vastgelegd.
