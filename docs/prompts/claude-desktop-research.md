# Claude Desktop Research prompt (iteracija 2)

Upute: Claude Desktop → novi Project → novi chat → uključi **Research** → isključi sve toolove/konektore →
zalijepi prompt ispod → Research. Downloadaj markdown u `docs/research/04-claude-desktop-research.md`
i reci Claude Codeu da ga integrira u KB.

---

```
I am building a proof of concept in which Claude (via an MCP server) tidies up 2D steel shop drawings in Autodesk Advance Steel 2022 (Windows, AutoCAD 2022 platform, .NET Framework 4.8). The 3D model is never touched. A small steel-fabrication company (4 engineers, 3D laser scanning → Advance Steel model → numbering → drawings) spends 1–2x the modelling time "cleaning up" auto-generated single-part, assembly and GA drawings: changing view scales, rearranging views on the sheet, adding/deleting/fixing dimensions (linear, chain, running/ordinate, hole dimensions with Ø and hole count), adding leader labels, filling the title block. The goal is an overnight run that proposes the cleanup and engineers approve/fix it by hand in the morning — so the result must remain native, editable Advance Steel drawing content.

What I have already established (please verify or refute, do not just repeat):
- The Advance Steel .NET/COM API has no drawing-side API (no access to detail views, drawing dimensions, labels, drawing styles) in 2022 or later; Advance Steel went into maintenance mode in 2023.
- Detail views appear to AutoCAD as custom entities (AstAtomicDetView); AS dimensions appear as opaque entities (ImpEntity). The title block is the HYPERSTEELPAGEHEADER block with token attributes.
- Moving a view works via the command _AstM4DetViewTransform; editing a view (scale, clipping) seems only possible via the dialog command _AstM4DetChange.
- Advance Steel does not run in AcCoreConsole.
- The only Advance Steel MCP server I found (eslinmoises/EMV-MCP-AS) is model-only and targets AS 2025/2026. The closest drawing-tool design is teknovizier/tekla_mcp_server (Tekla).
- Planned architecture: in-process AutoCAD .NET plugin (NETLOAD) + separate MCP server over a named pipe, plan→apply with one undo step per operation, PNG of the sheet via PlotEngine for visual verification.

Research questions, in priority order:
1. Changing the scale of an existing Advance Steel detail view programmatically: any command-line variant, system variable, COM/OPM dynamic property, LISP (entget/entmod on AstAtomicDetView), ObjectARX/AS "AstorDetails" database trick, or "change detail style then update" workflow that achieves it without the dialog. Any forum post, blog or add-on that does it.
2. The exact command names (global, underscore-prefixed) for Advance Steel 2022 drawing-side tools: parametric/associative dimensions (linear, chain, running, elevation, angle), "add/delete dimension point", "insert label", "arrange labels", "update drawing", "drawing processes", "create detail", "explode detail". Are they scriptable with explicit points/entities via (command ...) or Editor.Command, or do they require interactive selection? Look at CUIx macro strings, AS help, Autodesk forums, AUGI, Graitec/Kiwa/other reseller blogs, YouTube tutorials.
3. What survives "Update drawing" in AS 2022: manual AS dimensions, moved automatic dimensions, deleted automatic dimensions, plain AutoCAD dimensions/leaders/text placed inside or outside a detail view, edited title-block attributes, moved views, changed view scale.
4. How far drawing-style configuration can go to eliminate cleanup: dimension requests, hole dimensioning and hole labels (Ø, count), label strategies, view arrangement/layout rules, automatic scale selection, sub-styles, Drawing Style Manager and Management Tools tables (AstorDetails / AstorDetailsBase). Is editing these tables programmatically (SQL/MDB/SQL Server CE) a practical route for an AI to "tune" styles per company rules?
5. Any third-party add-ons, scripts or consultancies for Advance Steel drawing automation (e.g. Graitec PowerPack/GRAITEC STEEL, Kiwa, CADLearning, "AS Utilities", custom LISP libraries), including paid ones, and what they automate on the drawing side.
6. Whether reading the actual geometry of a detail view is possible: in-memory Entity.Explode on AstAtomicDetView, "detail explode" behaviour, getting dimension values/points from AS dimension entities, mapping model parts (DerivedObjectDetailInfo) to specific views.
7. Anything that changed between AS 2022 and 2023–2027 on the drawing side that would justify upgrading, and whether Autodesk Assistant / official Autodesk MCP servers (AutoCAD, Revit 2027) touch Advance Steel drawings at all.
8. Prior art for AI-driven shop-drawing cleanup in any steel detailing tool (Tekla AI Cloud Fabrication Drawings, Parabuild, SDS2, DraftAid, academic papers on automatic dimension placement / label placement / drawing layout optimisation) and which algorithms they use (template cloning from past drawings, rule engines, optimisation, learning from before/after pairs).

Output: a long, well-structured markdown report in English. For every claim give the source URL and mark it as [Verified: official doc / Autodesk staff], [Community], or [Inferred]. Include exact command strings, API member names and code snippets where found. End with (a) a table "capability → best available route in AS 2022 → confidence", (b) a list of concrete spike experiments to run on a real AS 2022 install, and (c) a list of open questions you could not resolve.
```
