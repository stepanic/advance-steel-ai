# advance-steel-ai

Research i PoC plan: **Claude (MCP) automatizira sređivanje 2D radioničkih nacrta u Autodesk Advance Steelu 2022** —
mjerilo i raspored pogleda, brisanje suvišnih kota i oznaka, kote, labele, provjere prije ispisa —
tako da nacrti ostanu nativni AS nacrti povezani s 3D modelom.

**Počni ovdje → [`docs/AUTOMATION-PLAN.md`](docs/AUTOMATION-PLAN.md)**

## Sadržaj

| Path | Sadržaj |
|---|---|
| `docs/AUTOMATION-PLAN.md` | **Završni plan:** tri sloja automatizacije, korak → način, arhitektura, MCP alati po fazama, spike redoslijed, rizici |
| `docs/requirements/00-requirements.md` | Anonimizirani zahtjevi i stvarni workflow klijenta, list po list |
| `docs/research/01-advance-steel-api.md` | Što AS 2022 API i AutoCAD .NET API dopuštaju na strani nacrta — izvedivost po alatu |
| `docs/research/02-existing-solutions.md` | Postojeća rješenja: MCP serveri (Tekla, AS, AutoCAD, Revit), komercijalni alati, Computer Use |
| `docs/research/03-architecture-options.md` | Usporedba arhitektura, preporuka, Phase 1 MCP alati, noćni batch |
| `docs/research/04-claude-desktop-research.md` | Claude Desktop Research report: AS naredbe, što preživljava update, stilovi u SQL LocalDB |
| `docs/research/05-synthesis-and-spike-plan.md` | Sinteza 01–04, spike eksperimenti 1–11 |
| `docs/research/06-workflow-leads.md` | Tragovi iz demo workflowa: „Advance View properties", stilovi/procesi, Batch Plot, spike 12–20 |
| `docs/prompts/claude-desktop-research.md` | Prompt korišten za Claude Desktop Research |

## Glavni zaključci

1. **Gotovog rješenja nema**, a **AS nema API za nacrte** (namjerno, i ostaje tako — maintenance mode od 2023.).
2. **Izvedivo je u tri sloja:** (A) user drawing stilovi/procesi rade većinu posla odmah pri generiranju;
   (B) NETLOAD plugin + MCP server preko AS naredbi (`_AstM4…`) i AutoCAD API-ja radi ostatak preko noći;
   (C) UI automatizacija samo za dijaloge bez naredbe (npr. mjerilo + clipping za više pogleda odjednom).
3. **Ne raditi:** explode / DXF / offline DWG (ruši vezu s modelom), Revit kao proxy (prenosi model, ne nacrte),
   Computer Use kao glavni mehanizam (samo fallback i vizualna provjera).
4. **Prije kodiranja:** 1–2 dana spikeova na stroju s AS 2022 — posebno koliko posla nestaje samo podešavanjem stilova.

## Metoda

Knowledge base je nastao iterativno: Claude Code subagenti (web research) → Claude Desktop Research →
sinteza → ciljani research na temelju demonstracije stvarnog workflowa. Sve tvrdnje su označene
kao službene [V], community [C] ili zaključak [I], s izvorima u svakom dokumentu.
Sirove bilješke sa sastanaka su privatne i nisu u repozitoriju (`private/` je u `.gitignore`).
