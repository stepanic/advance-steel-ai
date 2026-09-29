# 05 — Sinteza (iteracije 1 + 2) i spike plan

Stanje: 2026-09-29. Spaja `01`–`03` (Claude Code subagenti) i `04` (Claude Desktop Research).
Oznake: **[V]** službena dokumentacija / Autodesk staff, **[C]** community, **[I]** zaključak, netestirano.

## 1. Što je potvrđeno iz dva neovisna izvora

- **AS API nema drawing-side API — i to je namjerno od početka**, ne posljedica maintenance moda (Christian Blei, CBT, 2024) [C]; AS u maintenance modu od 29.3.2023., Autodesk upućuje na Graitec i Mensch und Maschine [V].
- Pogledi su custom entiteti `AstAtomicDetView` (`asobjectscad.dbx`); za AutoCAD API su AS objekti „proxyji" [V/C].
- Zaglavlje je blok `HYPERSTEELPAGEHEADER`, tokeni su hard-kodirani [V].
- Premještanje pogleda: `_AstM4DetViewTransform` (pogled, P1, P2); pozicija preživljava update [V].
- **Mjerilo postojećeg pogleda: nema puta bez dijaloga** (`_AstM4DetChange` ili dvoklik na zeleni okvir). Promjena mjerila traži quick update pa nacrt mora biti registriran na model u originalnoj strukturi foldera [V: Autodesk staff]. Community prijavljuje gubitak anotacija nakon promjene mjerila [C].
- Nijedan Autodesk MCP (AutoCAD, Revit 2027, Fusion) ne spominje Advance Steel [V]. Upgrade s 2022 ne donosi ništa za nacrte [V/C].

## 2. Što je novo iz iteracije 2

- **Popis globalnih naredbi za nacrte** (vidi tablicu u `04`, §Q2). Ključne za nas:
  `_AstM4CommDetAddHorLinearDim` / `…VertLinearDim` / `…SlopeLinearDim`, `_AstM4CommDetAddDim` (lanac),
  `_AstM4CommDetAddDimAngle`, `_AstM4CommDetCrHeightKote` (kota visine), `_AstM4CommDetModDimAdd` / `…Remove` (točke kote),
  `_AstM4CommDetInsertAnno` (oznaka), `_AstM4CommDetArrangeViews` / `…ArrangeDetails`, `_AstM4CommDetUpdateDetail`,
  `_AstM4DetViewDelete`, `_AstM4CommDetailingProc`. Izvor je stari Graitec User's Guide → **provjeriti u CUIx-u na 2022**.
- **Matrica preživljavanja updatea** [V uglavnom]: obrisane automatske kote ostaju obrisane (osim kod kombiniranih dimension requesta: brisati više puta); obrisane točke kota ostaju obrisane; pomaknuti pogledi ostaju; AutoCAD kote ostaju „mrtve"; **revision cloudovi se gube**; tokeni u zaglavlju se prepišu na „Update Page Header". Nepoznato: obične AutoCAD linije/tekst/MLeader.
- **Management Tools postavka „Manual dimension point behavior on update"** mora se svjesno postaviti prije nego AI-dodane kote idu u proizvodnju.
- **Explode ne daje čistu AutoCAD geometriju** — eksplodirani nacrti i dalje sadrže AS objekte [V]. In-memory `Entity.Explode` ostaje otvoreno pitanje (spike).
- **Stilovi žive u SQL LocalDB `.mdf`** (`AstorDetails`, `AstorAddin`, `AstorDetailsBase`) [V]. AI ih može **čitati i uspoređivati s pravilima firme**; pisanje samo za mapirana polja, inače predlaže čovjeku u Drawing Style Manageru.
- **Tekla AI Cloud Fabrication Drawings = retrieval + kloniranje** najsličnijeg odobrenog crteža, uz čovjeka u petlji; single-part nacrti izuzeti [V]. To je najrealniji uzor za nas.

## 3. Proturječja i korekcije

| Tema | 01–03 | 04 | Zaključak |
|---|---|---|---|
| AS kote kao `ImpEntity` | community navod | nije verificirano | Spike 1 |
| AcCoreConsole | Autodesk QA: ne radi [V] | plauzibilno, bez izvora | Držimo „ne radi"; brza provjera u spikeu nije prioritet |
| Explode za čitanje geometrije | „vjerojatno radi" [I] | AS objekti ostaju unutra [V] | Ne oslanjati se; spike 3 |
| Asocijativne AS kote | „imena naredbi nepoznata" | popis naredbi postoji (stari UG) | Spike 4 odlučuje |

## 4. Revidirana strategija

1. **Arhitektura ostaje:** NETLOAD plugin (net48) + MCP server preko named pipea; svaka operacija = `Editor.Command` u vlastitoj undo grupi + PNG provjera.
2. **Podjela rada [I]:** ~70 % dobitka iz podešavanja stilova/procesa (mjerila kroz alternativne scale u procesima, hole dimension requests, label strategije) — AI predlaže, čovjek primjenjuje; ~30 % noćne izmjene na listu: premještanje pogleda, dodavanje/brisanje točaka kota, ručne AS kote, oznake, ne-token atributi.
3. **Mjerilo:** (a) pravilo u procesu/stilu da se nacrt odmah generira u dobrom mjerilu; (b) fallback: UI Automation dijaloga `_AstM4DetChange` iz plugina; (c) regeneracija nacrta.
4. **Samo nativni AS sadržaj** za sve što AI dodaje; AutoCAD anotacije tretirati kao privremene dok spike 6 ne pokaže suprotno.
5. **Učenje iz primjera (Tekla pristup):** Klijentovi „prije/poslije" parovi → za novi sklop naći najsličniji odobreni nacrt i prenijeti razliku (mjerilo, pozicije pogleda, obrisane/dodane kote) kao niz AS naredbi. Jutarnje korekcije inženjera postaju novi parovi.

## 5. Spike plan (Windows + AS 2022, 1–2 dana)

Preduvjet: kopija stvarnog projekta s 3 nacrta (pozicija, sklop, montažni), registrirana na model, u originalnoj strukturi foldera. Sve na kopiji.

| # | Eksperiment | Odgovara na |
|---|---|---|
| 1 | NETLOAD probe DLL: dump svih entiteta (RXClass, DxfName, handle, layer, extents) u sva 3 nacrta | Stvarna imena klasa pogleda/kota/oznaka |
| 2 | LISP `entget` (+xdata), `dumpallproperties`, `vlax-dump-object` na pogledu i AS koti | Postoji li skriveno svojstvo mjerila/vrijednosti kote |
| 3 | In-memory `Entity.Explode` pogleda i kote | Može li se čitati geometrija (rupe, rubovi, vrijednosti kota) |
| 4 | Iz CUIx-a izvući makroe; skriptati `_AstM4CommDetAddHorLinearDim`, `_AstM4CommDetAddDim`, `_AstM4CommDetModDimRemove`, `_AstM4CommDetInsertAnno` s eksplicitnim točkama/pickovima | Mogu li se asocijativne AS kote/oznake dodavati programski |
| 5 | `_AstM4DetViewTransform` preko `Editor.CommandAsync(viewId, p1, p2)` | Premještanje pogleda iz koda |
| 6 | Matrica preživljavanja: LINE, MTEXT, MLEADER, AutoCAD DIM, ručna AS kota, pomaknuta/obrisana automatska kota, pomaknut pogled, token/ne-token atribut → promjena modela → update | Što je sigurno za AI |
| 7 | UI Automation `_AstM4DetChange`: nađi kontrolu mjerila, postavi, OK; ponovi na 20+ pogleda | Pouzdanost mjerila kroz dijalog |
| 8 | Promijeni mjerilo u stilu/procesu → `_AstM4CommDetUpdateDetail` na postojećem nacrtu | Reskalira li update postojeći pogled |
| 9 | PlotEngine → PNG (`PublishToWeb PNG.pc3`) + uvećani isječci | Vizualna provjera za Claudea |
| 10 | `UNDO _Begin/_End` oko koraka 4–7; jedan `UNDO 1` vraća svaki | Undo po operaciji |
| 11 | Kopija `AstorAddin.mdf` u SSMS: dump sheme, promijeni jedan hole dimension request u DSM-u, vidi koji redovi se mijenjaju | Može li AI čitati/diffati stilove |

**Kriterij uspjeha za nastavak na fazu 1 (MCP):** spike 1, 5, 9, 10 prolaze. **Kriterij za fazu 2 (kote):** spike 4 prolazi za barem linearnu kotu i brisanje točke kote.

## 6. Otvorena pitanja za klijenta

- Koliko nacrta tjedno / po projektu i koji tip (pozicija, sklop, montažni) troši najviše vremena?
- Koji udio vremena odlazi na mjerilo, na raspored pogleda, a koji na kote?
- Jesu li stilovi nacrta firme prilagođeni ili se koriste default Autodesk stilovi? Tko ih održava?
- Koristi li se Vault / mrežni projekt folder (rizik s `Databases` folderom kod updatea)?
- Može li se dobiti remote pristup (RDP/AnyDesk) Windows stroju s AS 2022 za spike, ili spike radi netko od njih uz Claude Code na tom stroju?
