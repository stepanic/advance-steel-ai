# Kako automatizirati razradu nacrta u Advance Steelu 2022

Završni plan (stanje 2026-09-29), izveden iz zahtjeva klijenta (`requirements/00-requirements.md`)
i šest research dokumenata (`research/01`–`06`). Oznake pouzdanosti: **[V]** službeno/Autodesk staff,
**[C]** community, **[I]** zaključak — netestirano na stvarnoj instalaciji.

## TL;DR

1. **Gotovog rješenja nema** — nitko ne radi AI razradu AS nacrta; nijedan MCP ne radi s AS *nacrtima* [V].
2. **AS nema API za nacrte**, namjerno od početka, i neće ga dobiti (maintenance mode od 2023.) [V/C].
3. **Automatizacija je ipak izvediva u tri sloja**, bez rušenja veze nacrta s modelom:

| Sloj | Što radi | Tko/kada | Udio posla [I] |
|---|---|---|---|
| **A. Stilovi i procesi** | User drawing styles/processes koji odmah generiraju nacrt „skoro gotov": fiksna mjerila, clipping, bez automatskih kota i zavara na listovima okvira, filtri okvir/ispuna po ulozi, labele s `%MainPartQuantity`, apsolutne + lančane kote „from end" | Čovjek u Drawing Style Manageru, jednom; AI predlaže izmjene i uspoređuje s pravilima | **~60–70 %** |
| **B. Plugin + MCP (noćni rad)** | NETLOAD .NET plugin u AS procesu + MCP server; AS naredbe (`_AstM4…`) i AutoCAD API: otvori nacrte redom, update, briši višak, premjesti poglede, dodaj kote/labele, provjeri okvire, PNG, undo po koraku | Claude Code headless / Agent SDK preko noći, samo *predlaže* ili primjenjuje u kopiji | **~25–35 %** |
| **C. UI automatizacija dijaloga** | Samo za ono bez naredbe: „Advance View properties" (mjerilo + clipping za više pogleda odjednom), po potrebi Edit detail | Iz plugina (Windows UI Automation), fallback Claude Computer Use | ostatak |

4. Izbor pogleda/UCS-a, *Define assembly detailing direction* i modeliranje iz scana **ostaju ljudski**.
5. Prvi korak nije kodiranje nego **1–2 dana spikeova** na stroju s AS 2022 (popis u §5).

## 1. Korak klijenta → način automatizacije

Sažetak tablice iz `research/06` §1 (detalji i fallbackovi tamo).

| Korak (requirements §5) | Primarni put | Pouzdanost |
|---|---|---|
| List 1: samo 3D pogled | Stil: user kopija stila samo s 3D view requestom | Visoka |
| Mjerilo pogleda | Stil: *Scale in process = Fixed* / alternativna mjerila; fallback dijalog „Advance View properties" (više pogleda odjednom, UIA) | Visoka (stil) / srednja (UIA) |
| Pozicija pogleda | `_AstM4DetViewTransform` (+ `_AstM4CommDetArrangeViews`) | Visoka |
| Labele za sve elemente, „3X" | Stil: labeling strategy za sve dijelove + `%MainPartQuantity`; duplikate briše AutoCAD API | Srednja |
| Show hidden lines | Dijalog „Advance View properties"; pravilo kada — AI | Srednja |
| Brisanje automatskih kota i zavara | Stil (bez dimension requesta, weld symbols off); fallback erase — obrisane kote ostaju obrisane nakon updatea [V] | Visoka |
| Okvir bez ispuna / ispune bez okvira | Stil: filtar model objekata po ulozi (role) | Srednja — ovisi o dosljednim ulogama u modelu |
| Apsolutne + lančane kote od otvorenog kuta | Stil: „Linear, relative and absolute (from end)"; fallback `_AstM4CommDetAddDim` s točkama koje bira AI | Srednja; izbor kuta = AI |
| Elementi 1:2, clipping 40–40 | Stil: single-part stil s fiksnim 1:2 i clipping strategijom | Visoka |
| Kutne kote „u materijal" | `_AstM4CommDetAddDimAngle` s točkama koje bira AI | Niska–srednja |
| Naslov „Sklop 1" | AutoCAD API: ne-token atribut u `HYPERSTEELPAGEHEADER` (token za to ne postoji) | Srednja |
| Zeleni okviri unutar papira | Deterministička provjera (extents pogleda vs `HYPERSTEELPAGEFRAME`) + page setup prototipa na Layout/Window | Visoka |
| Force update + otvaranje nacrta redom | Plugin otvara DWG-ove sortirano + `_AstM4CommDetUpdateDetail` (zaobilazi Document Manager) | Srednja |
| Pokretanje procesa #34/#4 | `_AstM4CommDetailingProc User <n>` | Srednja |

## 2. Arhitektura

```
Claude Desktop / Claude Code (headless noću)
        │  MCP (stdio)
        ▼
MCP server (Python za PoC, C# .NET 8 za produkciju)
        │  named pipe (per-user)
        ▼
AS plugin (C#, net48, NETLOAD) — unutar Advance Steel 2022 procesa
   ├─ AutoCAD .NET API: čitanje entiteta, extents, erase, atributi, PlotEngine → PNG, undo grupe
   ├─ AS naredbe: Editor.Command("_AstM4…", entitet, točke) na glavnoj niti uz DocumentLock
   ├─ AS model API: koji dijelovi/sklopovi su na nacrtu (DerivedObjectDetailInfo)
   └─ UI Automation: samo dijalozi bez naredbe
        │
        ▼
Pravila firme (MCP resource: markdown/JSON) · audit log (JSONL) · PNG prije/poslije
```

- **Plan → apply:** `propose_*` vraća diff bez promjena; `apply_plan(id)` izvršava kao **jednu** registriranu naredbu → jedan korak undo, vidljivo u povijesti naredbi.
- **Noćni rad:** AS mora biti pokrenut u prijavljenoj Windows sesiji (AcCoreConsole ne radi s AS objektima [V]). Claude Code headless (`claude -p`) ili Agent SDK iterira nacrte; rezultat su predložene izmjene + PNG-ovi, ujutro inženjer potvrđuje.
- **Uzori koda:** `bimwright/dwg-mcp`, `hoganmas/autocad-mcp` (AutoCAD 2022 net48 plugin + MCP), `teknovizier/tekla_mcp_server` (skup alata za nacrte). Jedini AS MCP (`eslinmoises/EMV-MCP-AS`) je samo za model i AS 2025/26.

## 3. MCP alati po fazama

**Faza 1 — čitanje (bez rizika):** `list_open_drawings`, `get_sheet_info` (format, zaglavlje, tip), `list_views` (handle, extents, klasa), `list_annotations`, `check_frames_inside_sheet`, `check_overlaps`, `render_sheet_png` (+ uvećani isječci), `get_rules`.

**Faza 2 — sigurne izmjene:** `open_drawings_sorted`, `update_detail`, `erase_entities` (kote/zavari/ispune po filtru), `move_view`, `set_header_attribute` (ne-token), `undo_last`, `save_copy`.

**Faza 3 — anotacije:** `add_linear_dim` / `add_chain_dim` / `add_angle_dim` (AS naredbe s točkama), `insert_label`, `set_view_properties` (mjerilo, clipping, hidden lines — UIA dijaloga).

**Faza 4 — cijeli list po pravilima:** `propose_sheet_cleanup(sheet, rules)` → `apply_plan`.

## 4. Kako AI odlučuje

- **Pravila** (MCP resource): mjerila po tipu lista, što se briše, kako se kotira okvir, kutne kote u materijal, „3X", okviri unutar papira.
- **Učenje iz primjera** (pristup koji koristi Tekla AI Cloud Fabrication Drawings [V]): parovi nacrta *prije/poslije* ručnog sređivanja → za novi sklop nađi najsličniji odobreni nacrt i prenesi razliku (mjerilo, pozicije, obrisane/dodane kote) kao niz AS naredbi. Jutarnje ispravke inženjera postaju novi parovi.
- **Vizualna provjera:** PNG lista nakon svakog koraka; Claude provjerava preklapanja i čitljivost.

## 5. Spike plan (prije kodiranja)

Svih 20 eksperimenata je u `research/05` §5 (1–11) i `research/06` §3 (12–20). Kritični redoslijed:

1. **Dump entiteta** u 3 nacrta (klase pogleda, kota, labela) — spike 1.
2. **Stilovi (spike 15):** kloniraj user stilove po pravilima i prebroji koliko ručnih koraka ostaje — potvrđuje ili ruši hipotezu sloja A.
3. **Skriptabilnost AS naredbi** s točkama (spike 4, 5, 20) i **„Advance View properties" iz koda** (spike 12–14).
4. **Matrica preživljavanja updatea** (spike 6) i **undo** (spike 10).
5. **Batch Plot uzrok** + `check_frames_inside_sheet` (spike 18) i **PNG** (spike 9).

**Kriterij za fazu 1:** spikeovi 1, 5, 9, 10 prolaze. **Za fazu 3:** spike 4 prolazi barem za linearnu kotu i brisanje točke kote.

## 6. Što treba od klijenta

- 2–3 stvarna projekta (kopije, registrirane na model) + nacrti **prije i poslije** ručnog sređivanja za pozicijski, sklopni i montažni list.
- Pravila kotiranja zapisana (ili izdiktirana) — polazište je requirements §5.
- Odgovori na pitanja iz `research/05` §6 i `research/06` §4 (clipping 40–40, uloge okvir/ispuna u modelu, page setup prototipa, kada hidden lines).
- Pristup Windows stroju s AS 2022 (remote ili zajednički radni blok od 3–4 h).

## 7. Rizici

| Rizik | Ublažavanje |
|---|---|
| AS naredbe ne primaju točke iz koda | UIA/Computer Use za te korake; više posla prebaciti u stilove |
| Promjena mjerila briše anotacije [C] | Mjerilo riješiti u stilu prije generiranja; mijenjati prije anotiranja |
| Uloge u modelu nisu dosljedne | Filtar po profilu/imenu; ili pravilo modeliranja za tim |
| Update prepiše AI izmjene | Matrica preživljavanja (spike 6); samo nativni AS sadržaj; postaviti *Manual dimension point behavior on update* |
| Nestabilan `Databases` folder projekta | Noćni rad samo na kopiji projekta; nikad prekinuti usred updatea |
| AS 2022 zastarijeva | Upgrade ne donosi ništa za nacrte [V]; plugin za 2025+ treba .NET 8 build |
