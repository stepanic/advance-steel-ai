# 02 — Existing solutions: AI / MCP for cleaning up Advance Steel shop drawings

Research date: 2026-09-29. Scope: what already exists that could let Claude (via MCP) tidy up
2D shop drawings produced by **Autodesk Advance Steel 2022**. That means changing view scale, arranging
views, adding and fixing dimensions (including hole dimensions) and adding labels. Context is in
[`../requirements/00-requirements.md`](../requirements/00-requirements.md).

Legend used below:
- **[V]** verified: read on the project page, the repo README or GitHub API data (stars and last push as of 2026-09-29).
- **[S]** secondary: taken from a search snippet or a third-party article, not confirmed at the primary source.
- **[I]** inference or opinion by the author of this report.

---

## TL;DR

**No ready-made product covers our case.** Nothing we found cleans up Advance Steel drawings with AI, and
nothing gives Claude an MCP interface to Advance Steel *drawings* (as opposed to the 3D model). This holds
for Advance Steel 2022 and for newer versions. The closest things are:

1. an **open-source Tekla MCP server** with real drawing tools (views, scale, collision checks, annotation reads). It is a design template for us, not something we can use directly;
2. a **brand-new open-source Advance Steel MCP** (model-side only, AS 2025/2026, 0 stars). Its architecture can be reused, but its drawing tools cannot, because it has none;
3. **generic AutoCAD MCP servers** (COM/AutoLISP) that can add plain AutoCAD dimensions. These do not create *Advance Steel* associative dimensions;
4. **vendor AI in competing products** (Tekla AI Cloud Fabrication Drawings, Parabuild AI dimension placement). This shows the idea is commercially validated, but we cannot use any of it from Advance Steel 2022.

Autodesk's official MCP push (AutoCAD/Civil 3D, Revit 2027, Dynamo 4.2) is real, but it is in Tech Preview.
It targets 2027-era products, is mostly read-oriented and is aimed at Autodesk Assistant as the client. None of it
touches Advance Steel drawings. **[I]** The realistic path remains the one in the problem statement: our own .NET
plugin (NETLOAD, .NET Framework 4.8) inside AS 2022, plus a thin MCP server, with the Tekla MCP drawing toolset as the design reference.
Claude computer use or Windows-MCP is a useful fallback for the few actions the API cannot reach, and for visual checks.

### Ranked table (most relevant first)

| # | Name | Type | URL | Relevance to our case | Maturity |
|---|------|------|-----|-----------------------|----------|
| 1 | **teknovizier/tekla_mcp_server** | Open-source MCP (Tekla Structures) | https://github.com/teknovizier/tekla_mcp_server | **High as a design blueprint.** Has a `Drawings` provider with `get_drawing_views`, `set_views_attributes` (scale), `move_view`, `align_section_views`, `get_view_annotations`, `check_drawing_collisions`, `check_for_unattached_dimensions`, `check_for_unmarked_objects`, `print_drawings`. This is almost exactly our tool list, but for Tekla. **[V]** | 49★, GPL-3.0, pushed 2026-08-01, tested on Tekla 2022 and 2026 **[V]** |
| 2 | **eslinmoises/EMV-MCP-AS** | Open-source MCP (Advance Steel) | https://github.com/eslinmoises/EMV-MCP-AS | **Medium.** It is the only Advance Steel MCP found. Architecture: Python FastMCP → localhost HTTP :5055 → in-process .NET plugin, with a Roslyn C# scripting sandbox and viewport screenshots. That is essentially our proposed architecture. It is **model-side only**: drawing support is limited to `get_drawing_status`. It targets **AS 2025/2026 (.NET 8)**, so it will not load in AS 2022 (.NET Framework 4.8). **[V]** | 0★, MIT, created 2026-09-18, about 30 commits, no releases **[V]** |
| 3 | **Advance Steel native: Drawing Styles + Drawing Processes + Document Manager** | Built-in (rules-based automation) | https://help.autodesk.com/view/ADSTPR/2023/ENU/?guid=GUID-F76C2725-A748-4425-97E6-3CB68A77AE66 | **High, and not AI.** Tuning drawing styles (dimension requests, hole symbols and labels, view arrangement strategies, sub-styles) cuts clean-up at the source. Drawing processes can be triggered from code (`SendCommand "_.ASTM4COMMDETAILINGPROC ..."`). **[V]/[S]** **[I]** The AI should work on top of good styles, not replace them. | Mature, shipped in AS 2022 |
| 4 | **christiannp/autocad-mcp** | Open-source MCP (full AutoCAD, COM + AutoLISP) | https://github.com/christiannp/autocad-mcp | **Medium.** 121 tools including `draw_dimension`, `dimension_chain`, `dimension_edit`, layouts/viewports and `accoreconsole` batch processing. Tested only on AutoCAD 2025. Creates *AutoCAD* dimensions, not AS associative ones. **[V]** | 0★, no license file, pushed 2026-09-25, 7 commits **[V]** |
| 5 | **puran-water/autocad-mcp** | Open-source MCP (AutoCAD LT, File-IPC + AutoLISP) | https://github.com/puran-water/autocad-mcp | **Medium-low.** It is the most popular AutoCAD MCP, with a clean LISP dispatcher pattern that has no .NET dependency and linear/aligned/angular/radius dimension tools. Built for LT 2024+. **[I]** The pattern would also work in full AutoCAD / AS 2022. | 540★, MIT, pushed 2026-02-20 **[V]** |
| 6 | **Tekla Structures 2025/2026: AI Cloud Fabrication Drawings** | Commercial vendor AI | https://support.tekla.com/doc/tekla-structures/2026/rel_dra_new_in_ai_cloud_fabrication_drawings | **Validation of the idea.** AI picks the closest finished drawing from the company's past projects and *clones* it, including placement of dimensions, annotations and views, with a human-in-the-loop step. Tekla only. **[V]** | Preview since 2025, improved in 2026. The Diamond licence requirement is **[S]** |
| 7 | **Parabuild** (AutoCAD/BricsCAD plugin) | Commercial steel detailing | https://www.parabuild.com/steel-detailing-results/automatic-shop-drawings/ | **Validation / competitor.** Claims an "AI engine" that finds the best location for automatic dimensions and tags, with collision avoidance, and outputs native DWG. It would replace Advance Steel rather than plug into it. **[V]** (marketing claim) | Commercial, mature |
| 8 | **Autodesk AutoCAD & Civil 3D MCP Server** | Official Autodesk MCP (local) | https://help.autodesk.com/view/ADSKMCP/ENU/ | **Low for us today.** Can query, analyze, navigate and modify objects in the open drawing and check drawings against templates. Tech Preview, 2027-era, documented client is Autodesk Assistant. It does not mention Advance Steel. **[V]/[S]** | Tech Preview |
| 9 | **CursorTouch/Windows-MCP** (and pywinauto MCPs, Claude computer use) | Generic Windows UI automation | https://github.com/CursorTouch/Windows-MCP | **Fallback and verification layer.** Can click dialogs the API cannot reach and take screenshots for visual QA. Too brittle to be the main mechanism for placing dimensions. **[I]** | 7.5k★, MIT, very active **[V]** |
| 10 | **GRAITEC STEEL 2027 (ex-PowerPack) + GRAITEC Assistant** | Commercial Advance Steel add-on with AI | https://graitec.com/us/blog/whats-new-in-graitec-steel-for-advance-steel/ | **Low.** The AI Assistant does Ask, Find (model filtering) and Create (beams, plates, joints). No drawing clean-up. Runs on AS **2026/2027 only**. **[V]** | Commercial, 2027 release |
| 11 | **mcp-servers-for-revit** / Revit Public MCP (Autodesk) | MCP (Revit) | https://github.com/mcp-servers-for-revit/mcp-servers-for-revit | **Low as a proxy** (see §4). The community server has `create_dimensions`, but for Revit views and sheets, not Advance Steel drawings. **[V]** | Community: 346★ / 466★ (older `revit-mcp`), MIT. Autodesk: Revit 2027 Tech Preview |
| 12 | **DraftAid** | Startup: AI auto-dimensioned 2D drawings | https://draftaid.io/ | **Low.** AI generates dimensioned views of structural steel and aluminium extrusions from Inventor or STEP. It does not integrate with Advance Steel. **[S]** (Engineering.com, 2024) | Small startup, YC-backed, paying customers **[S]** |

---

## 1. MCP servers for Autodesk (and Tekla) products

### 1.1 AutoCAD

**Official (Autodesk)**
- **Autodesk MCP Servers hub**: https://help.autodesk.com/view/ADSKMCP/ENU/ **[V]** lists these servers:
  - *AutoCAD and Civil 3D*: local, **Tech Preview**. It lets you "query, analyze, navigate, and modify objects in your open drawing" and "check drawings against templates". There are no dimension-specific tools in the description.
  - *Fusion* (local, GA), *Fusion Data* (remote, GA), *Product Help* (remote, GA, 110+ products), *Revit 2027 Read-Tools* (local, Tech Preview).
  - **Advance Steel is not listed.**
- **AutoCAD 2027 Autodesk Assistant** (Tech Preview): chat and voice prompts, checking the drawing against a standards file (layers, linetypes, text and **dimension styles**), natural-language selection and drawing queries. It is built on MCP internally.
  https://help.autodesk.com/cloudhelp/2027/ENU/AutoCAD-WhatsNew/files/GUID-99E2D8F7-D4D9-4B3A-9E57-7B73D6BB12AD.htm ,
  https://www.autodesk.com/blogs/autocad/autocad-2027-1/ **[S]**. The CAD-standards article
  (https://www.easternengineeringgroup.com/autodesk-assistant-autocad-2027-mcp-estandares-cad-2/) describes it as
  *identifying* inconsistencies, not fixing them, and warns that results may be inaccurate. **[S]**
- Vagon's guide says Autodesk documents **Autodesk Assistant as the supported client**, that write capabilities are mainly on the Civil 3D side, and that you should not assume Claude Code can connect to it: https://vagon.io/blog/how-to-use-autocad-with-claude-code **[S]**
- **[I]** None of this applies to AS 2022, which is built on AutoCAD 2022. Even when these tools reach Advance Steel, they will not understand AS drawing objects (AS dimensions and labels are custom entities).

**Community (GitHub, stats from GitHub API on 2026-09-29) [V]**

| Repo | ★ | Last push | License | Mechanism | Dimensions? |
|---|---|---|---|---|---|
| [puran-water/autocad-mcp](https://github.com/puran-water/autocad-mcp) | 540 | 2026-02-20 | MIT | File-IPC JSON ↔ `mcp_dispatch.lsp` (AutoLISP), plus a headless ezdxf backend. Targets AutoCAD **LT** 2024+ | Yes: linear, aligned, angular, radius, leader |
| [AnCode666/multiCAD-mcp](https://github.com/AnCode666/multiCAD-mcp) | 122 | 2026-06-27 | Apache-2.0 | COM. AutoCAD 2018+, ZWCAD, GstarCAD, BricsCAD | No dimension tools documented |
| [christiannp/autocad-mcp](https://github.com/christiannp/autocad-mcp) | 0 | 2026-09-25 | none | COM + AutoLISP, `accoreconsole` batch, escape hatches `cad_command`/`cad_lisp` | Yes: `draw_dimension`, `dimension_chain`, `dimension_edit`, layouts and viewports |
| [codesknight/AutoCAD-MCP](https://github.com/codesknight/AutoCAD-MCP) | 2 | 2026-07-14 | MIT | — | Entity, layer and file I/O |
| [bendourohamza-hub/CAD-MCP](https://github.com/bendourohamza-hub/CAD-MCP) | 1 | 2026-09-28 | MIT | 35 tools, command passthrough | — |
| [chichicaste/mcp-autocad-server](https://github.com/chichicaste/mcp-autocad-server) | 0 | 2025-10-24 | none | Learning stub, tools not implemented | No |

Commercial: **HMK CAD Pilot** (https://hmktools.com/autocad-mcp) **[V]** is a .NET add-in plus MCP for AutoCAD and Civil 3D
2023–2027, with dimension, layout and PDF-plot tools. It supports Claude Desktop, Claude Code, Codex and Cursor, and has a 14-day trial. It does not mention Advance Steel.
Overview list: Snyk "9 MCP servers for CAD": https://snyk.io/articles/9-mcp-servers-for-computer-aided-drafting-cad-with-ai/

**[I] Relevance.** Advance Steel 2022 *is* full AutoCAD 2022 underneath, so COM, AutoLISP and .NET all work inside it.
Any of these servers could therefore create **plain AutoCAD dimensions** on an AS drawing sheet. The Advance Steel
documentation, however, recommends AS dimension tools over AutoCAD linear dimensions, because AS dimensions keep the correct value at
any view scale and move and scale with the detail view
(https://help.autodesk.com/view/ADSTPR/2022/ENU/?guid=GUID-D462912E-5259-476F-9EC8-B27A361B7A0E) **[V]**. That lines up with
the client's requirement that everything stays "clickable" and native. Generic AutoCAD MCPs are useful for reading geometry,
drawing leaders and text, and for scaffolding, but not for the core dimensioning work.

### 1.2 Revit

- **Autodesk Revit Public MCP Server**, Tech Preview in Revit 2027. It can find elements, check parameters, count, make bulk parameter edits and capture view snapshots. Clients are accessed through Autodesk Assistant.
  https://help.autodesk.com/cloudhelp/2027/ENU/Revit-WhatsNew/files/GUID-97697CBF-0E11-484E-96E5-4277E3E8D61F.htm **[V]**,
  blog https://www.autodesk.com/blogs/aec/2026/06/17/revit-public-mcp-server/ (403 when fetched, title only),
  usage article https://www.autodesk.com/support/technical/article/caas/sfdcarticles/sfdcarticles/Usage-of-Revit-2027-MCP-Server.html
- **mcp-servers-for-revit** (community; TypeScript MCP server, C# Revit add-in and a command set):
  https://github.com/mcp-servers-for-revit/mcp-servers-for-revit: 346★, MIT, last push 2026-04-05 **[V]**. It supersedes
  https://github.com/mcp-servers-for-revit/revit-mcp (466★). Tools include `create_dimensions`, `get_current_view_info` and tagging. **[V]**
  A popular fork with 80+ tools for Revit 2023–2026: https://github.com/LuDattilo/revit-mcp-server (66★, MIT). **[V]**
- APS sample, Revit via the Design Automation API: https://github.com/autodesk-platform-services/aps-sample-mcp-server-revit-automation/ **[S]**

### 1.3 Advance Steel

- **EMV-MCP-AS**: https://github.com/eslinmoises/EMV-MCP-AS **[V]**. This is the only Advance Steel MCP found. Python FastMCP connects over
  localhost REST (:5055) to a .NET 8 ApplicationPlugin that runs in-process with a main-thread dispatcher. It uses `ASObjectsMgd`/`ASGeometryMgd`
  and has a Roslyn C# scripting sandbox. Tools cover model reads and audits, generative modelling (beams, plates, bolts,
  joints), numbering, DSTV/NC export, BOM, `get_drawing_status` and viewport capture. There are **no drawing-editing tools**.
  It supports **AS 2025 and 2026** only. It was created 2026-09-18 and has 0 stars. Its design spec lives in the repo under
  `docs/superpowers/specs/2026-09-18-advance-steel-mcp-design.md`.
  **[I]** Worth reading for IPC, threading and scripting-sandbox patterns. Porting it to AS 2022 means moving from .NET 8 to .NET Framework 4.8 and handling the
  older, single-assembly API (the 2025 API split `ASMgd` into `ASMgd` and `ASObjectsMgd`:
  https://help.autodesk.com/view/ADSTPR/2025/ENU/?guid=GUID-309B2ACB-5CD4-4B91-98B7-D00F2026BBD7 **[S]**).
- No official Autodesk MCP for Advance Steel. **[V]** (It is absent from the ADSKMCP hub.) One search snippet claims AS 2027 ships
  "the latest Autodesk Assistant updates". **[S], unconfirmed**; the AS 2027 What's New page could not be read.
- Other Advance Steel automation on GitHub: [DynamoDS/Dynamo-Advance-Steel](https://github.com/DynamoDS/Dynamo-Advance-Steel)
  (18★, official Dynamo nodes, active 2026-06). The nodes cover modelling, selection and properties, not drawing editing. **[V]/[S]**

### 1.4 Tekla Structures (reference design)

- **teknovizier/tekla_mcp_server**: https://github.com/teknovizier/tekla_mcp_server **[V]**. FastMCP 3.0 with modular providers.
  The `docs/reference.md` Drawings section contains, verbatim tool names:
  `get_drawings`, `open_drawing`, `close_drawing`, `get_drawing_views` (type, **scale**, position, size, sheet number),
  `get_view_objects`, `get_view_annotations` (marks, **dimensions**, text), `set_views_attributes` (**scale**),
  `move_view`, `align_section_views`, `delete_views`, `update_drawings`, `check_drawing_collisions` (exports to DXF and
  detects mark and view overlaps and content outside the sheet, then draws coloured revision clouds), `check_for_unattached_dimensions`,
  `check_for_unmarked_objects`, `print_drawings` (PDF), `export_drawings` (DWG/DXF).
  **[I]** This maps almost one-to-one onto the client's proposed read/modify tool list. Two ideas are worth copying: checking collisions via a DXF export and
  **marking problems with revision clouds for morning review**. The one thing it lacks is *creating* dimensions.
  49★, GPL-3.0 (licence caveat if we copy code, fine if we copy ideas), last push 2026-08-01.
- [YuriyKirillov/TeklaMCPServer](https://github.com/YuriyKirillov/TeklaMCPServer): C#, about 80 tools, mostly modelling. 12★. **[V]**
- Tekla's own **Tekla Model Assistant** (Trimble Labs preview) handles natural-language modelling and drawing operations, for example "Add part marks
  to all columns in this drawing": https://support.tekla.com/doc/tekla-structures/2026/mod_tekla_model_assistant **[S]**

### 1.5 Autodesk Platform Services, Dynamo

- APS MCP (official sample, Node.js; ACC/Forma projects, issues, documents): https://github.com/autodesk-platform-services/aps-mcp-server-nodejs ;
  experimental https://github.com/petrbroz/aps-mcp-server ; AEC Data Model .NET MCP:
  https://github.com/autodesk-platform-services/aps-aecdm-mcp-dotnet **[S]**. Cloud data only. **Not relevant** to desktop AS drawings.
- **Dynamo MCP**, built into Dynamo 4.2 (Revit 2027.3). Autodesk Assistant can build, run and tidy up Dynamo graphs:
  https://dynamobim.org/dynamo-4-2-in-revit-2027-3-autodesk-assistant-dynamo-mcp/ **[S]**. **[I]** Not usable with AS 2022.
  Dynamo for Advance Steel has no drawing-editing nodes that we know of.

---

## 2. Commercial and research tools for steel shop drawing automation and clean-up

### 2.1 Autodesk Advance Steel itself

- **Drawing Style Manager (DSM)**: https://help.autodesk.com/view/ADSTPR/2023/ENU/?guid=GUID-F76C2725-A748-4425-97E6-3CB68A77AE66 **[V]**.
  It configures view definitions, automatic dimension requests (linear, chain, running, radial, angular, relative or absolute), hole
  representation as symbols, labels, and reusable sub-styles.
  - Dimension arrangement ("free of intersections with the dimensioned local geometry or total geometry"):
    https://knowledge.autodesk.com/support/advance-steel/learn-explore/caas/CloudHelp/cloudhelp/2020/ENU/AdvSteel-Drawings/files/GUID-97C690AD-E9F3-4294-AD4A-6EE10F0FC01E-htm.html **[S]**
  - View arrangement strategies (distance between views): https://knowledge.autodesk.com/support/advance-steel/learn-explore/caas/CloudHelp/cloudhelp/2017/ENU/AdvSteel-Drawings/files/GUID-97FB601B-437E-473C-8836-1AEE7040C29F-htm.html **[S]**
  - Setting up automatic dimensions: https://www.autodesk.com/support/technical/article/caas/sfdcarticles/sfdcarticles/Setting-up-Automatic-Dimensions-in-Advance-Steel-drawings.html
  - Assembly style dimensioning attached parts: https://www.autodesk.com/support/technical/article/caas/sfdcarticles/sfdcarticles/Advance-Steel-How-to-customize-an-assembly-drawing-style-to-dimension-the-attached-parts.html
  - Legacy GRAITEC DSM guide (PDF, still the most detailed): http://www.graitec.info/Common/Downloads/Documentation/AS-DSM_guide-8.1-EN.pdf
  - AU class, "Automatic Generation of Drawings Using the Drawing Processes Feature": https://www.autodesk.com/autodesk-university/class/Advance-Steel-Automatic-Generation-Drawings-Using-Drawing-Processes-Feature-2014
- **API**: Advance Steel .NET and COM API (2022): https://help.autodesk.com/view/ADSTPR/2022/ITA/?guid=GUID-AF2072A9-3F8F-4DAC-B595-470A9E32BEE0 .
  A forum thread describes running detail processes headless from C# via `SendCommand("_.ASTM4COMMDETAILINGPROC ...")`:
  https://forums.autodesk.com/t5/advance-steel-forum/advance-steel-api-options-for-numbering-and-detail-drawing/td-p/8216125 **[S]** (403 on fetch).
  **[I]** How much of the *drawing-side* object model (views, AS dimensions, labels) is exposed in the 2022 API is the key open
  question. It is not answered by existing products and belongs in a separate API-feasibility study.
- **Advance Steel 2026/2027 new features**: we found no evidence of AI-based drawing clean-up. Autodesk's product page
  still describes rules-based automatic dimensions and labels: https://www.autodesk.com/products/advance-steel/product-details **[S]**

### 2.2 GRAITEC (PowerPack → GRAITEC STEEL)

- PowerPack for Advance Steel has become **GRAITEC STEEL 2027**, with a single installer for existing PowerPack users. It runs on **AS 2026/2027**
  and Inventor 2026/2027. **GRAITEC Assistant** (named-user licence, Advance Steel only) offers Ask (documentation Q&A),
  Find (NL/voice model filtering) and Create (beams, plates, joints from prompts), plus an AI BOM generator.
  https://graitec.com/us/blog/whats-new-in-graitec-steel-for-advance-steel/ ,
  https://www.graitec.com/Help/PowerPack_for_Advance_Steel/En/What_is_New.htm **[V]**
- **No drawing clean-up or dimensioning AI** is mentioned. **[V]** Not available for AS 2022. **[I]**
  Historical PowerPack releases include some drawing utilities (not checked in detail).

### 2.3 Trimble Tekla

- **AI Cloud Fabrication Drawings**: preview in Tekla 2025, improved in 2026. AI matches the current assembly against *finalized drawings
  from previous projects* in a cloud collection and suggests the top 3 templates. The chosen drawing is cloned, with "more reliable placement of annotations,
  dimensions, and views", and a human-in-the-loop step follows.
  https://support.tekla.com/doc/tekla-structures/2026/rel_dra_new_in_ai_cloud_fabrication_drawings ,
  https://support.tekla.com/doc/tekla-structures/2026/dra_create_fabrication_drawings_cloud_ai **[V]**;
  launch PR: https://news.trimble.com/Trimble-Unveils-2026-Tekla-Software-Accelerating-BIM-Engineering-and-Construction-Productivity-Through-Streamlined-Workflows-and-AI ;
  https://www.engineering.com/tekla-2026-update-adds-ai-tools-and-workflow-features/ **[S]** (the Diamond licence requirement is from a snippet).
  **[I]** This is a strong product signal. The winning approach is "learn from our own past drawings plus clone similar ones", not
  "LLM places every dimension from scratch". the client's before/after examples are the equivalent training set.

### 2.4 Others

- **SDS2** (Allplan/Nemetschek): long-standing rules-based auto-detailing driven by Fabricator Settings. No AI drawing features found.
  https://techsupport.sds2.com/main/Project%20Settings%20Fab/Detailing/dtl_crit.htm , https://blog.sds2.com/2025-1-release **[S]**
- **Parabuild** (AutoCAD/BricsCAD plugin): "AI engine" for dimension and tag placement with collision avoidance.
  https://www.parabuild.com/steel-detailing-results/automatic-shop-drawings/ **[V]** (vendor claim)
- **DraftAid** (YC, San Jose): AI auto-dimensioned 2D drawings of structural steel and extrusions from Inventor or STEP.
  https://draftaid.io/ , https://www.engineering.com/hey-ai-make-my-drawings/ (Apr 2024) **[S]**
- **Research**: most academic work is about *reading* drawings (VLM parsing, GD&T extraction), e.g.
  https://arxiv.org/pdf/2411.03707 , https://arxiv.org/pdf/2506.17374 , https://arxiv.org/pdf/2510.21862 . Work on *placing*
  dimensions is older and mechanical-part oriented, e.g. "Automated Dimensioning … Based on Curve Chain":
  https://www.researchgate.net/publication/343161123 ; DL-based 2D generation from BIM:
  https://www.sciencedirect.com/science/article/abs/pii/S0957417425006402 **[S]**. **[I]** Nothing directly reusable, but the VLM
  papers support using sheet screenshots for QA.

---

## 3. Generic approaches to driving Windows CAD with LLMs

| Approach | Examples / URLs | Assessment for AS 2022 **[I]** |
|---|---|---|
| **Claude computer use** (vision + mouse/keyboard) | Available in Claude Cowork and Claude Code Desktop on **Windows** (beta, Pro/Max): https://support.claude.com/en/articles/14128542-let-claude-use-your-computer-in-cowork , https://x.com/claudeai/status/2039836891508261106 **[S]** | It matches the client's "click on screen" preference and needs no API. Placing *associative* AS dimensions by clicking snap points is slow and error-prone, and an overnight unattended run with many dialogs is fragile. Best used for **visual QA** and for triggering UI-only commands. |
| **Windows UI Automation MCPs** | [CursorTouch/Windows-MCP](https://github.com/CursorTouch/Windows-MCP) (7.5k★, MIT) **[V]**; [sandraschi/pywinauto-mcp](https://github.com/sandraschi/pywinauto-mcp); [PingPong2534/winauto-mcp](https://github.com/PingPong2534/winauto-mcp); Microsoft [AutoGenesis pywinauto-mcp-server](https://github.com/microsoft/AutoGenesis/blob/main/pywinauto-mcp-server/WINDOWS-README.md) **[S]** | More deterministic than pure vision for AutoCAD dialogs, but the drawing canvas itself is opaque to UIA. Same verdict as above: an auxiliary tool. |
| **AutoCAD COM from Python** | [pyautocad](https://pypi.org/project/pyautocad/) (`AddDimAligned`, `AddDimRotated`, `AddDimOrdinate` …): https://www.supplychaindataanalytics.com/dimaligned-object-in-autocad-using-python/ ; [manufino/AutoCAD](https://github.com/manufino/AutoCAD) **[S]** | Works against AS 2022, which exposes the AutoCAD COM object, but produces plain AutoCAD dimensions. Quick prototyping only. |
| **AutoLISP generated by the LLM** | ChatGPT-to-LISP tutorials: https://resources.imaginit.com/support-blog/using-chatgpt-to-write-autocad-lisp-routines-from-idea-to-execution , https://techdrawai.com/blog/chatgpt-prompts-for-autocad ; Autodesk forum scepticism ("full of errors"): https://forums.autodesk.com/t5/visual-lisp-autolisp-and-general/can-we-use-machine-learning-and-artificial-intelligence-in/td-p/11989246 **[S]** | The puran-water and christiannp servers show a proven pattern: fixed dispatcher tools plus an `execute_lisp` escape hatch. AS commands can be scripted via `(command "_AstM4...")`. It works for AS commands that run without interactive picks. |
| **In-process .NET plugin + MCP** | EMV-MCP-AS (AS), HMK Pilot (AutoCAD), mcp-servers-for-revit (Revit), Tekla MCPs | **The recommended main path.** Every serious CAD MCP we found uses this architecture. For AS 2022 it has to be built for .NET Framework 4.8. |

Guides: "How to connect Claude to AutoCAD (MCP, 2026)": https://techdrawai.com/blog/connect-claude-to-autocad ;
Vagon Fusion MCP vs computer use: https://vagon.io/blog/how-to-use-autodesk-fusion-with-claude-mcp-computer-use **[S]**

---

## 4. Revit ↔ Advance Steel interop: can Revit MCP act as a proxy?

**Facts [V]/[S]:**
- Revit and Advance Steel exchange models via **SMLX** using the *Advance Steel Extension for Revit* (import, export and synchronize). Steel Connections for Revit
  transfer as macros. https://help.autodesk.com/cloudhelp/2023/ENU/Revit-StructEng/files/GUID-237FBEA0-B287-46C6-80B6-116FABD20A68.htm ,
  https://help.autodesk.com/cloudhelp/2020/ENU/Revit-AddIns/files/GUID-CE9F630B-C271-43CB-B321-D35C069BA27E.htm ,
  https://resources.imaginit.com/building-solutions-blog/integrating-revit-structure-model-with-advance-steel **[S]**
- Shop drawings, CNC and fabrication detailing are produced in **Advance Steel**. Revit is the design/LOD-300 side
  (AU handout: https://static.au-uw2-prd.autodesk.com/Class_Handout_BLD123767_Advance_Steel_and_RevitA_Seamless_Workflow_David_Cohn.pdf). **[S]**
- IFC is another exchange path. It is lossy for fabrication data. **[I]**

**Assessment [I]:** SMLX and IFC carry the **3D model**, not the 2D drawings. A Revit MCP can dimension *Revit* views and sheets,
but those are not the Advance Steel shop drawings the client wants to keep "clickable". Revit also lacks AS's fabrication
numbering, drawing styles and associative steel dimensioning. So Revit as a proxy would mean **moving drawing production
to Revit**, which is a process change and not an automation. Its only real value is as a *learning reference*: how mature Revit MCPs structure their view,
dimension and tag tools. **Low priority, not recommended** for the PoC.

---

## 5. Implications for the PoC [I]

1. **Build, don't buy.** No off-the-shelf tool does AI clean-up of AS 2022 drawings.
2. **Architecture confirmed by prior art:** an in-process .NET plugin (NETLOAD, .NET Framework 4.8, main-thread and document lock) plus a localhost
   IPC channel plus a thin MCP server. EMV-MCP-AS (AS 2025+) and HMK Pilot use the same shape.
3. **Tool design:** copy the Tekla MCP drawing toolset shape (views with scale and position, annotations read-back, collision
   check, unattached-dimension check, revision clouds for morning review, PDF print). Add what it lacks: *create and delete AS
   dimensions and hole labels*.
4. **Fix at the source first:** have one engineer tune AS 2022 Drawing Styles and Processes. Every rule moved into DSM is one fewer thing the AI has to do.
5. **Learn from Tekla's approach:** "match to a past finished drawing and imitate" (few-shot from the client's before/after set) is more
   robust than free-form dimensioning.
6. **Computer use and Windows-MCP** serve as a fallback for UI-only commands and as the visual QA loop (sheet PNG → Claude vision),
   not as the primary editing mechanism.
7. **Open question that decides feasibility:** whether the AS 2022 .NET/COM API exposes drawing views and AS dimension creation.
   If it doesn't, the fallbacks are scripting AS drawing commands via `SendCommand`/AutoLISP with computed pick points, or plain
   AutoCAD dimensions, which lose associativity.

---

## Sources

MCP: Autodesk
- https://help.autodesk.com/view/ADSKMCP/ENU/
- https://help.autodesk.com/cloudhelp/2027/ENU/AutoCAD-WhatsNew/files/GUID-99E2D8F7-D4D9-4B3A-9E57-7B73D6BB12AD.htm
- https://www.autodesk.com/blogs/autocad/autocad-2027-1/
- https://www.easternengineeringgroup.com/autodesk-assistant-autocad-2027-mcp-estandares-cad-2/
- https://help.autodesk.com/cloudhelp/2027/ENU/Revit-WhatsNew/files/GUID-97697CBF-0E11-484E-96E5-4277E3E8D61F.htm
- https://www.autodesk.com/blogs/aec/2026/06/17/revit-public-mcp-server/
- https://www.autodesk.com/support/technical/article/caas/sfdcarticles/sfdcarticles/Usage-of-Revit-2027-MCP-Server.html
- https://goto.archi/blog/post/whats-new-in-revit-2027
- https://dynamobim.org/dynamo-4-2-in-revit-2027-3-autodesk-assistant-dynamo-mcp/
- https://github.com/autodesk-platform-services/aps-mcp-server-nodejs
- https://github.com/autodesk-platform-services/aps-sample-mcp-server-revit-automation/
- https://github.com/autodesk-platform-services/aps-aecdm-mcp-dotnet
- https://github.com/petrbroz/aps-mcp-server

MCP: community
- https://github.com/eslinmoises/EMV-MCP-AS
- https://github.com/teknovizier/tekla_mcp_server
- https://github.com/YuriyKirillov/TeklaMCPServer
- https://github.com/puran-water/autocad-mcp
- https://github.com/christiannp/autocad-mcp
- https://github.com/AnCode666/multiCAD-mcp
- https://github.com/codesknight/AutoCAD-MCP
- https://github.com/bendourohamza-hub/CAD-MCP
- https://github.com/chichicaste/mcp-autocad-server
- https://github.com/mcp-servers-for-revit/mcp-servers-for-revit
- https://github.com/mcp-servers-for-revit/revit-mcp
- https://github.com/LuDattilo/revit-mcp-server
- https://hmktools.com/autocad-mcp
- https://snyk.io/articles/9-mcp-servers-for-computer-aided-drafting-cad-with-ai/
- https://vagon.io/blog/how-to-use-autocad-with-claude-code
- https://techdrawai.com/blog/connect-claude-to-autocad

Steel detailing and drawings
- https://help.autodesk.com/view/ADSTPR/2023/ENU/?guid=GUID-F76C2725-A748-4425-97E6-3CB68A77AE66
- https://help.autodesk.com/view/ADSTPR/2022/ENU/?guid=GUID-D462912E-5259-476F-9EC8-B27A361B7A0E
- https://help.autodesk.com/cloudhelp/2023/ENU/AdvSteel-Drawings/files/GUID-7CBFF3A7-51D8-4CCB-9DB2-C0ACE2678562.htm
- https://www.autodesk.com/support/technical/article/caas/sfdcarticles/sfdcarticles/Setting-up-Automatic-Dimensions-in-Advance-Steel-drawings.html
- https://www.autodesk.com/support/technical/article/caas/sfdcarticles/sfdcarticles/Advance-Steel-How-to-customize-an-assembly-drawing-style-to-dimension-the-attached-parts.html
- http://www.graitec.info/Common/Downloads/Documentation/AS-DSM_guide-8.1-EN.pdf
- https://www.autodesk.com/autodesk-university/class/Advance-Steel-Automatic-Generation-Drawings-Using-Drawing-Processes-Feature-2014
- https://help.autodesk.com/view/ADSTPR/2022/ITA/?guid=GUID-AF2072A9-3F8F-4DAC-B595-470A9E32BEE0
- https://help.autodesk.com/view/ADSTPR/2025/ENU/?guid=GUID-309B2ACB-5CD4-4B91-98B7-D00F2026BBD7
- https://forums.autodesk.com/t5/advance-steel-forum/advance-steel-api-options-for-numbering-and-detail-drawing/td-p/8216125
- https://www.autodesk.com/products/advance-steel/product-details
- https://github.com/DynamoDS/Dynamo-Advance-Steel
- https://graitec.com/us/blog/whats-new-in-graitec-steel-for-advance-steel/
- https://www.graitec.com/Help/PowerPack_for_Advance_Steel/En/What_is_New.htm
- https://support.tekla.com/doc/tekla-structures/2026/rel_dra_new_in_ai_cloud_fabrication_drawings
- https://support.tekla.com/doc/tekla-structures/2026/dra_create_fabrication_drawings_cloud_ai
- https://support.tekla.com/doc/tekla-structures/2026/mod_tekla_model_assistant
- https://news.trimble.com/Trimble-Unveils-2026-Tekla-Software-Accelerating-BIM-Engineering-and-Construction-Productivity-Through-Streamlined-Workflows-and-AI
- https://www.engineering.com/tekla-2026-update-adds-ai-tools-and-workflow-features/
- https://www.parabuild.com/steel-detailing-results/automatic-shop-drawings/
- https://draftaid.io/
- https://www.engineering.com/hey-ai-make-my-drawings/
- https://techsupport.sds2.com/main/Project%20Settings%20Fab/Detailing/dtl_crit.htm
- https://blog.sds2.com/2025-1-release

Generic automation
- https://support.claude.com/en/articles/14128542-let-claude-use-your-computer-in-cowork
- https://x.com/claudeai/status/2039836891508261106
- https://github.com/CursorTouch/Windows-MCP
- https://github.com/sandraschi/pywinauto-mcp
- https://github.com/PingPong2534/winauto-mcp
- https://github.com/microsoft/AutoGenesis/blob/main/pywinauto-mcp-server/WINDOWS-README.md
- https://pypi.org/project/pyautocad/
- https://www.supplychaindataanalytics.com/dimaligned-object-in-autocad-using-python/
- https://resources.imaginit.com/support-blog/using-chatgpt-to-write-autocad-lisp-routines-from-idea-to-execution
- https://techdrawai.com/blog/chatgpt-prompts-for-autocad
- https://forums.autodesk.com/t5/visual-lisp-autolisp-and-general/can-we-use-machine-learning-and-artificial-intelligence-in/td-p/11989246

Revit ↔ Advance Steel
- https://help.autodesk.com/cloudhelp/2023/ENU/Revit-StructEng/files/GUID-237FBEA0-B287-46C6-80B6-116FABD20A68.htm
- https://help.autodesk.com/cloudhelp/2020/ENU/Revit-AddIns/files/GUID-CE9F630B-C271-43CB-B321-D35C069BA27E.htm
- https://resources.imaginit.com/building-solutions-blog/integrating-revit-structure-model-with-advance-steel
- https://static.au-uw2-prd.autodesk.com/Class_Handout_BLD123767_Advance_Steel_and_RevitA_Seamless_Workflow_David_Cohn.pdf

Research
- https://arxiv.org/pdf/2411.03707
- https://arxiv.org/pdf/2506.17374
- https://arxiv.org/pdf/2510.21862
- https://www.researchgate.net/publication/343161123
- https://www.sciencedirect.com/science/article/abs/pii/S0957417425006402
