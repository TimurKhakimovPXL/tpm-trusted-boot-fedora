<!--
PUBLIC RELEASE VERSION - sanitized for publication.
Machine-id and filesystem/LUKS UUIDs are format-valid EXAMPLE values, not the
original lab identifiers. PCR values, policy-key digests and file hashes are
genuine measurements from the validated build and are public values by design.
-->

# Bijlage 4: Validatie-evidentie tijdens runtime

> Vastgelegd op de testopstelling (VM 500, tboot-lab) na een schone reboot. De blokken zijn de
> feitelijke console-uitvoer. CLI-uitvoer staat in het Engels, de toelichting in het Nederlands.
> Lange sleutel- en blob-dumps zijn met [...] ingekort.

## 1. Huidige gevalideerde toestand

### 1.1 Boot-identiteit (bootctl status, ingekort)

```text
System:
      Firmware: UEFI 2.70 (Proxmox distribution of EDK II 1.00)
   Secure Boot: enabled (user)
  TPM2 Support: yes
  Measured UKI: yes
  Boot into FW: supported

Current Boot Loader:
       Product: systemd-boot 258.7-1.fc43
 Current Entry: 0123456789abcdef0123456789abcdef-6.19.14-200.fc43.x86_64.efi
 Default Entry: 0123456789abcdef0123456789abcdef-6.19.14-200.fc43.x86_64.efi

Current Stub:
       Product: systemd-stub 258.7-1.fc43

Default Boot Loader Entry:
         type: Boot Loader Specification Type #2 (UKI, .efi)
           id: 0123456789abcdef0123456789abcdef-6.19.14-200.fc43.x86_64.efi
       source: /boot/efi/EFI/Linux/0123456789abcdef0123456789abcdef-6.19.14-200.fc43.x86_64.efi
      options: root=UUID=aaaaaaaa-bbbb-cccc-dddd-eeeeeeeeeeee ro rd.luks.uuid=luks-11111111-2222-3333-4444-555555555555 quiet
```

Secure Boot staat in user-modus onder de eigen sleutels, de TPM meet het UKI (`Measured UKI: yes`),
en de actieve default-entry is het hook-gegenereerde, MID-geprefixte UKI als Type #2 (UKI, `.efi`).

### 1.2 Runtime-PCR's tegen de opgeslagen voorspelling

```text
runtime  PCR 7 : BF8878BD8B70654924BD39CA71D770F556F03558AC200BAD46A9CEB5A2B596BD
runtime  PCR 11: 28E66CE224C9BB0711059470D77103A5AB5A5C1674C03D1BA1563C2FEE52C90F
expected PCR 11: 28E66CE224C9BB0711059470D77103A5AB5A5C1674C03D1BA1563C2FEE52C90F
```

De runtime-PCR 11 komt exact overeen met de opgeslagen voorspelling. PCR 7 komt overeen met de
baseline van de Secure Boot-toestand.

### 1.3 LUKS-keyslots en TPM2-token (luksDump, ingekort)

```text
$ systemd-cryptenroll /dev/sda3
SLOT TYPE
   0 password
   1 tpm2

$ cryptsetup luksDump /dev/sda3
UUID:           11111111-2222-3333-4444-555555555555
Keyslots:
  0: luks2   PBKDF: argon2id        (wachtwoord, herstelanker)
  1: luks2   PBKDF: pbkdf2 / sha512 (TPM2)
Tokens:
  0: systemd-tpm2
        tpm2-hash-pcrs:   7
        tpm2-pcr-bank:    sha256
        tpm2-pubkey:      [PEM-sleutel, afgekort]
        tpm2-pubkey-pcrs: 11
        tpm2-primary-alg: ecc
        tpm2-pin:         false
        tpm2-srk:         true
        tpm2-policy-hash: [afgekort]
        tpm2-blob:        [afgekort]
        Keyslot:    1
```

Keyslot 0 is het wachtwoord-herstelanker (argon2id). Keyslot 1 is via TPM2-token 0 gekoppeld aan
de split policy: PCR 7 statisch (`tpm2-hash-pcrs: 7`) en PCR 11 via de ondertekende publieke
sleutel (`tpm2-pubkey-pcrs: 11`), in de sha256-bank, met de SRK als primaire sleutel.

### 1.4 Bewijs dat de TPM (niet het wachtwoord) ontgrendelde

```text
May 28 14:39:26 tboot ... Starting systemd-cryptsetup@luks-...-4feedb562f86.service - Cryptography Setup ...
May 28 14:39:27 tboot ... Finished systemd-cryptsetup@luks-...-4feedb562f86.service - Cryptography Setup ...
May 28 14:39:26 tboot ... systemd 258.7-1.fc43 running in system mode (... +TPM2 ... +LIBCRYPTSETUP +LIBCRYPTSETUP_PLUGINS ...)
```

De cryptsetup-service startte en voltooide vroeg in de boot, zonder wachtwoordprompt; de operator
hoefde niets in te typen. Deze systemd-versie schrijft geen expliciete "unlocked with TPM2"-regel.
Het bewijs is daarom drieledig: de service voltooide schoon, systemd is met TPM2-ondersteuning
gebouwd (`+TPM2`), en de runtime-PCR 11 kwam overeen met de verzegelde voorspelling (§1.2). Samen
tonen ze aan dat de TPM de sleutel vrijgaf.

### 1.5 Integriteit van het geboot UKI

```text
1187c78ce2435e0a5c27ed8f35e0fd653da3580ac7a2597b03a30bbfd5deb843  /boot/efi/EFI/Linux/0123456789abcdef0123456789abcdef-6.19.14-200.fc43.x86_64.efi
```

## 2. Evolutie van PCR 11 over de mijlpalen

De waarde van PCR 11 veranderde bij elke ingrijpende wijziging van het opstartpad. Telkens
voorspelde en verzegelde de keten de nieuwe waarde opnieuw, en telkens kwam de runtime-waarde na
de reboot exact overeen met de voorspelling. Dat is het kernbewijs dat de forward-sealing en de
automatisering werken: een wijziging breekt de ontgrendeling niet, omdat de nieuwe toestand vooraf
ondertekend wordt.

```text
Fase 1 (Secure Boot + eerste UKI):                            42F096D23D0227A780C801C3670A7BF7E57C1583D9360E3D79B5D2AF544FE3CC
Module 3 (LUKS TPM2-inschrijving):                            A03EB49C9335D721758AE8AA9C34DA87664C5CAF9FAD7C314C68C038B16435DD
Huidige toestand (na systemd 258.7 + kernel/dracut-updates):  28E66CE224C9BB0711059470D77103A5AB5A5C1674C03D1BA1563C2FEE52C90F
PCR 7 (baseline, ongewijzigd):                                BF8878BD8B70654924BD39CA71D770F556F03558AC200BAD46A9CEB5A2B596BD
```

Bevinding E.3 (bijlage 5): de sha256 van het UKI verandert ook wanneer de gemeten inhoud gelijk
blijft, door de willekeurige RSA-PSS-salt in `.pcrsig` en een nieuwe `sbsign`-tijdstempel. PCR 11
verandert enkel wanneer de gemeten inhoud zelf wijzigt, zoals bij de overgang naar systemd 258.7.

## 3. Block B.2.2 en B.5: automatiseringsketens onder een echte transactie

Block B.2.2 oefende de eerste keten uit in een echte transactie: de helper verwerkte twee kernels
(`6.17.1-300.fc43.x86_64` secundair en `6.19.14-200.fc43.x86_64` geboot) in 83 seconden en
herbouwde, hertekende en herverzegelde de UKI's via de gevalideerde hook. Na de reboot kwam de
runtime-PCR 11 overeen met de voorspelling en ontgrendelde LUKS via de TPM zonder wachtwoordprompt.

Block B.5 sloot de tweede keten (systemd-boot-loader) met dezelfde gefaseerde gate-methode, met
als sluitstuk een echte reboot waarbij de runtime-PCR 11 byte-voor-byte overeenkwam met de
voorspelling. Anker: `module5-b5-validated`. Citeerbaar bewijs uit B.2.4: Gate 3 (referentierun),
Gate 4 (onafhankelijke voorspelling, byte-match), Gate 6A (door de operator bevestigde schone
reboot) en Gate 6B (runtime-PCR 11 byte-match).
