# Zahtjevi — AI sređivanje radioničkih nacrta u Advance Steelu 2022

Anonimizirana verzija zahtjeva klijenta (manja firma za čelične konstrukcije i 3D skeniranje, 4 inženjera).
Sažeta iz pisanog opisa i dva radna sastanka (29. 9. 2026), uključujući demo stvarnog workflowa na ekranu.
Sirove bilješke i transkripti nisu dio repozitorija.

## 1. Problem

- Alat: **Autodesk Advance Steel 2022** (Windows, AutoCAD 2022 platforma, .NET Framework 4.8).
- Tok rada: 3D laserski scan → ručno modeliranje u AS-u → Numbering → generiranje radioničkih nacrta
  (pozicijski, sklopni, montažni/GA) → **ručno sređivanje nacrta** → Batch Plot → radionica.
- Sređivanje (razrada) traje **1–2× vremena modeliranja**. Monotono je, uvijek isto i pravilima opisivo.
- Cilj: AI (Claude) sređuje nacrte **preko noći**, inženjeri ujutro pregledaju, poprave i šalju u proizvodnju.

## 2. Nepregovorni uvjeti

1. **Nacrti ostaju nativni AS nacrti povezani s modelom.** Explode, DXF export ili offline editiranje DWG-a
   ne dolaze u obzir: nakon promjene modela nacrt se mora moći ažurirati.
2. **Rezultat mora biti ručno popravljiv** u AS sučelju („klikabilno") — AI će griješiti.
3. **3D model se ne dira**; samo 2D nacrti.
4. **Sporost je prihvatljiva** — i višesatni rad je u redu ako ide preko noći.
5. Svaka izmjena ima **undo**; poželjan način „predloži pa potvrdi"; sve se logira.

## 3. Što ostaje ljudski posao

- Modeliranje iz scana (ručni `3DPOLY` po betonu; izvedbeni projekti se rijetko poklapaju sa stvarnim stanjem).
- Izbor pogleda / UCS-a koji „stane na papir" i **Define assembly detailing direction**.
- Pokretanje stilova/procesa i generiranje nacrta (~20 min po projektu).

## 4. Workflow generiranja (kontekst)

1. **Pregledni 3D nacrt:** Drawing Style „3D model with labels" → Document Manager → Details → Open drawing.
   Mjerilo (dvoklik u zeleni okvir), centriranje; labele sklopova: *Select Similar* → *Advanced Properties → Leader Line → Show Line*, ručni razmak labela.
2. **Po sklopu:** izolacija sklopa u Model Browseru → user Drawing Process „Selected assemblies, **one per sheet**" (3D list sklopa) →
   postavi pogled → *Define assembly detailing direction* → proces „single parts to **fill the sheet**".
3. Document Manager → *Update required* → **Force update** → otvara 5–6 nacrta (nasumičnim redom; ručno se slažu po broju).

## 5. Sređivanje — što AI treba raditi

### List 1 — 3D pogled sklopa
- Obrisati nepotrebne poglede (nacrt, bokocrt, tlocrt); ostaje 3D pogled.
- Postaviti **mjerilo** i pozicionirati pogled na papir.
- AS stavlja leader samo za prvi element iz sastavnice → **labela za svaki element**; za ponovljene elemente jedna labela s brojem, npr. **„3X"**.
- Po potrebi uključiti/isključiti **Show hidden lines**.

### List 2 — okvir (npr. ograda)
- Mjerilo **1:10**; obrisati suvišan pogled.
- **Obrisati sve automatske kote** (uglavnom krive) i **oznake zavara** (radionici ne trebaju).
- Ostaviti samo okvir (obrisati ispune — *Select Similar → Delete*).
- Pozicionirati pogled, zatim **Update detail** (vraća zelene okvire na mjesto).
- **Kotiranje:** i **apsolutne (running) i relativne (lančane)** kote, obje radi provjere; kotira se **od otvorenog kuta**
  (da se metar može zakačiti), s obje strane; nula na desnoj strani.

### List 3 — ispune
- Kao list 2 (1:10), ali ostaju samo ispune.

### Listovi elemenata („fill the sheet")
- Naslov lista mijenja se u oznaku sklopa (npr. „Sklop 1").
- Svi pogledi elemenata → *Advanced view properties* → mjerilo **1:2**, **clipping 40–40** (skraćeni prikaz s prekidnim linijama).
- Rasporediti poglede; obrisati nepotrebne kote.
- **Kutne kote uvijek u materijal**, vanjska strana (npr. 60°, 30°).

### Globalno
- **Zeleni okviri pogleda ne smiju biti izvan papira** — inače Batch Plot ispiše prazno/krivo.
- Ponoviti za sve sklopove i elemente u projektu.

## 6. Predložene MCP mogućnosti (iz klijentovog opisa)

**Čitanje:** format lista i zaglavlje, tip nacrta · popis pogleda (tip, mjerilo, položaj, prikazani element/sklop) ·
geometrija pogleda (rubovi, rupe, grupe vijaka, konture limova) · podaci o elementu (profil, materijal, duljina, pozicija) ·
postojeće kote i labele · provjera preklapanja · **PNG lista/pogleda** za vizualnu provjeru.

**Izmjene:** mjerilo i premještanje pogleda · kote (linearne, lančane, apsolutne, kote rupa) · premještanje/brisanje kota ·
labele s pokaznom linijom · atributi zaglavlja · spremanje kopije · undo.

**Pravila kotiranja firme** dostupna Claudeu kao MCP resource.

Predložene faze: 1) čitanje + PNG · 2) kote · 3) mjerilo i raspored pogleda · 4) cijelo sređivanje po pravilima.

## 7. Otvorena pitanja klijenta

1. Koliko AS 2022 API dopušta rad s pogledima i kotama na nacrtima?
2. AS asocijativne kote (ažuriraju se s modelom) ili obične AutoCAD kote (lakše, ali se ne ažuriraju)?

Odgovori: `docs/research/01`–`06` i završni plan u `docs/AUTOMATION-PLAN.md`.
