<!--
PUBLIC RELEASE VERSION - sanitized for publication.
Machine-id and filesystem/LUKS UUIDs are format-valid EXAMPLE values, not the
original lab identifiers. PCR values, policy-key digests and file hashes are
genuine measurements from the validated build and are public values by design.
-->

# Bijlage 7: Forensisch artefact: door objcopy beschadigde UKI

> Gereconstrueerd uit de projectlogboeken van 8 mei 2026. Sluit aan bij bevinding B.1
> in het bevindingenbestand (bijlage 5). De CLI-uitvoer staat in het Engels; de toelichting in het Nederlands.

## Beschrijving van het artefact

Tijdens de uitwerking van de ondertekenketen werd een UKI gebouwd waarbij de sectie
`.pcrsig` achteraf werd toegevoegd met `objcopy --add-section`, in plaats van met een
enkele `ukify`-aanroep. Het resultaat (`bad-objcopy-pcrsig-loaderror.efi`) wordt bewaard
als forensisch bewijs van een belangrijke faalmodus. Het artefact staat op de testopstelling
onder `/root/tboot-lab/failed-ukis/`. De oorspronkelijke locatie van het bestand was
`/boot/efi/EFI/Linux/0123456789abcdef0123456789abcdef-6.19.14-200.fc43.x86_64.efi`.

## Symptoom

Het bestand doorstaat elke controle aan de Linux-zijde, maar wordt door de UEFI-firmware
geweigerd bij `LoadImage()`. De terugvalkopie van het oude UKI startte wel correct, wat
bewees dat de firmware zelf in orde was en enkel het via objcopy gebouwde UKI verwierp.

Controles aan de Linux-zijde (alle geslaagd):

```text
.pcrpkey present
.pcrsig present
sbverify OK
signer count = 1
```

Weigering door de firmware bij het opstarten:

```text
Error loading EFI binary \EFI\Linux\<id>.efi: Load Error
```

## Forensische inspectie (read-only)

Sectietabel van het beschadigde UKI (`objdump -h`):

```text
Idx Name       Size      VMA               LMA               File off  Algn
  0 .pcrsig    000002b7  0000000200000000  0000000200000000  ...
  1 .text      00000bff  000000014df92000  000000014df92000  ...
  2 .rodata    00000020  000000014df93000  ...
  ...
```

Het virtuele adres van `.pcrsig` (`0x200000000`, ongeveer 8 GB) ligt volledig buiten de
aaneengesloten reeks lage adressen van `.text` (`0x14df92000`) en de overige secties. Voor
een PE+-binary bepalen `ImageBase` en de aaneengesloten sectiereeks wat `LoadImage()`
verwacht te reserveren en te mappen. Een sectie op dit afwijkende adres maakt het bestand
onlaadbaar voor de firmware.

Handtekeningcontrole op hetzelfde bestand:

```text
Signature verification OK
  signer: tboot-lab Signature Database Key
```

De Authenticode-handtekening is geldig omdat `sbsign` de hash berekende over de bytes die
`objcopy` had weggeschreven. De structurele correctheid van de PE+-lay-out maakt geen deel
uit van het hashdomein van Authenticode, waardoor het bestand de handtekeningcontrole
doorstaat en toch onlaadbaar is.

## Besluit

`objcopy --add-section .pcrsig=...` levert een UKI op dat de validatie aan de Linux-zijde
(`sbverify`, `objdump -h`, `pesign --show-signature`) doorstaat, maar door UEFI `LoadImage()`
wordt geweigerd wegens een afwijkend sectie-VMA. De juiste werkwijze is het UKI in één
`ukify`-aanroep bouwen, met de PCR-onderteken-vlaggen erbij, zodat `.pcrsig` op een correct
adres terechtkomt en het geheel in één PE+-constructie wordt ondertekend. Deze bevinding is
de reden waarom de productieketen uitsluitend de native `ukify`-weg gebruikt en geen
losse objcopy-stap.
