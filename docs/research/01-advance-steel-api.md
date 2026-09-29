# 01 — What the Advance Steel 2022 API allows on the drawing side

Research date: 2026-09-29. Target: Autodesk Advance Steel 2022 (runs on AutoCAD 2022, .NET Framework 4.8, x64).
Scope: 2D shop drawings only (single-part, assembly, GA). The 3D model is not modified.

**Evidence labels used throughout**

| Label | Meaning |
|---|---|
| **[V]** | Verified from official Autodesk documentation, Autodesk-authored source code, or an Autodesk employee's forum answer. |
| **[C]** | Community statement (experienced forum users, partners), not official. |
| **[I]** | Inferred from general AutoCAD/ObjectARX behaviour or by analogy. Not tested. Must be checked in a spike on a real AS 2022 install. |

---

## TL;DR

**The Advance Steel .NET/COM API has no drawing-side API.** It cannot read or edit detail views, drawing dimensions, labels or drawing styles inside a drawing DWG. This has not changed from 2022 to 2027 [V]. An Autodesk software architect confirmed in 2018 that detailing automation is limited to setting defaults and calling commands [V]. Community experts say *"There is nothing in the API that deals with Detail styles"* and that the AutoCAD API only sees AS objects as proxies [C]. Autodesk put Advance Steel into **maintenance mode in March 2023** (no new features), so no drawing API is coming [V].

What still works for the PoC:

1. **AutoCAD .NET API on the drawing DWG.** It enumerates all entities (AS views and dimensions show up as opaque custom entities), reads bounding boxes, erases entities, reads and edits title-block attributes, plots to PNG, saves copies and handles undo.
2. **AS command-line commands** (`_AstM4...`), driven through `Editor.Command` / `SendStringToExecute`. They cover move view, change detail, update detail, drawing processes and create details.
3. **AS model-side API.** `KernelServices.GetDerivedDocumentsForDwg`, `DerivedObjectDetailInfo` and `DwgManager.IsCurrentDWGDerived` link a drawing to the model parts it shows.
4. **Configuration-level automation**: Drawing Style Manager, dimension requests, Management Tools defaults, and detail-style scripts in the AstorDetails database. These control what AS generates, not what is already on the sheet.

### Feasibility per required capability

| Capability | Verdict | How (best path) | Evidence |
|---|---|---|---|
| Read sheet info (drawing no., sheet size, title data) | **Yes** | AutoCAD API: read the `HYPERSTEELPAGEHEADER` block reference attributes (tokens such as `DRAWING_NO`, `SCALE`, `POS_NUM`) and the `HYPERSTEELPAGEFRAME` block extents. `DwgManager.IsCurrentDWGDerived(out parentId)` tells you whether the DWG is an AS drawing. | [V] block names and tokens, [V] API method, [I] reading via `AttributeReference` |
| List views | **Partial** | AutoCAD API: iterate the space and filter entities whose class/DXF name is `AstAtomicDetView` (and any related Ast* view classes). You get handle, layer and `GeometricExtents`. View name, scale and direction are **not** available through a public API. | [C] class name `AstAtomicDetView`, [I] extents |
| View geometry (lines/contours) | **Partial** | `Entity.Explode(DBObjectCollection)` in memory (non-destructive) on the view entity, if AS implements explode for it. AS supports "Detail explode" into plain AutoCAD entities, so this is likely. Otherwise, run the batch "detail explode" on a **copy** of the DWG. | [V] detail explode exists, [I] in-memory explode |
| Element data (which parts/assemblies appear) | **Partial (per drawing, not per view)** | From the **model** DWG: `KernelServices.GetDerivedDocumentsForDwg(db)` → `DerivedDocument.GetDerivedObjects()` → `DerivedObjectDetailInfo.GetSinglePartsDetailed()` / `GetMainPartsDetailed()`. Needs an AS transaction (2019+). Mapping a part to a specific view needs geometry heuristics. | [V/C] forum code, confirmed working |
| Existing dimensions | **Partial** | AutoCAD API sees AS dimensions as opaque entities (`ImpEntity` in .NET, reported by a user). You get handle, extents and layer, but **not** the value or points. In-memory explode may give text and lines [I]. | [C] ImpEntity, [I] rest |
| Overlap check (views, dims, labels, title block) | **Yes** | 2D axis-aligned bounding-box intersection on `GeometricExtents` of all entities, plus the header, frame and `ADVANCESTEEL_RESERVED_AREA` block extents. | [V] block names, [I] extents accuracy |
| PNG snapshot | **Yes** | AutoCAD `PlotEngine` with `PublishToWeb PNG.pc3` (the plot engine renders AS entities as displayed), or `PNGOUT`/`JPGOUT` commands for the current viewport. | [V] PlotEngine API, [I] AS rendering in plot |
| Change view scale | **Hard / no API** | Only `_AstM4DetChange` (Edit Detail dialog: scale, clipping, title per view). It is dialog-driven with no known command-line options. Options: (a) UI automation of the dialog (fragile), (b) re-create the detail with a different scale (drawing style "Modify settings"), (c) spike: check whether the view's scale appears in the AutoCAD Properties palette (OPM/COM dynamic property). | [V] command and dialog, [I] options |
| Move view | **Yes (command-driven)** | `_AstM4DetViewTransform`: select view, base point P1, target P2. Script it with `Editor.Command("_AstM4DetViewTransform", <ent>, p1, p2)` or `SendStringToExecute` with `(handent "…")`. The position is kept on drawing update. Do **not** use a raw `TransformBy`, because dimensions and labels may not follow [I]. | [V] command and behaviour, [I] scriptability |
| Add linear/aligned dimension | **Partial** | Option A: AS "Parametric Dimensions" commands (associative, scale-aware, survive update). Command names are not in the help and must be read from the AS ribbon/CUIx macros on the machine. They are point-pick driven, so they can be scripted with explicit points and the "Preferred for Manual Dimensions" snap. Option B: plain AutoCAD `RotatedDimension`/`AlignedDimension` via the API. These are easy, but do **not** update or follow views, and the scale must be set manually per dimension (DIMLFAC = 1/view scale). | [V] behaviour of both, [I] scripting |
| Add chain (continuous) dimension | **Partial** | Same as above. AutoCAD: several `RotatedDimension`s sharing a baseline line. AS: chain dimension command (name TBD). | [I] |
| Add absolute/ordinate (running) dimension | **Partial** | AS has "running dimension" (Dimension dialog: Running Dimension tab). AutoCAD alternative: `OrdinateDimension`. | [V] AS feature exists, [I] scripting |
| Add hole dimensions | **Partial** | AS dimension requests and "Dimension Point Behavior on Hole Patterns" in drawing styles (config level). Manual: AS parametric dims snapped to hole centres. AutoCAD alternative: plain dims to hole centre points taken from exploded geometry. | [V] config exists, [I] rest |
| Move dimension | **Yes (with caveat)** | Entity grip/`MOVE`/`TransformBy`. Moving an automatic dimension breaks its link to the automatic rules, and it keeps the manual position on update. | [V] behaviour, [I] via API |
| Delete dimension / dimension point | **Yes** | `Entity.Erase()` / `ERASE`. Deleted automatic dimensions normally stay deleted on update (exception: several dimension requests for the same object). AS "Delete point" / "Insert point" commands edit individual points. | [V] behaviour, [I] Erase via API |
| Labels with leaders | **Partial** | AS labels (Insert Label, labeling strategies, arrange labels) are command-driven. Plain AutoCAD `MLeader` via the API is easy but not associative or updated. | [V] features, [I] scripting |
| Title block attributes | **Yes (caveat)** | Edit `AttributeReference`s of `HYPERSTEELPAGEHEADER` via the AutoCAD API. Caveat: token-driven attributes (for example `SCALE`, `POS_NUM`, revision fields) are likely rewritten on drawing update. Write only free/user fields, or set the values in the model or Document Manager instead. | [V] block and tokens, [I] overwrite behaviour |
| Save copy | **Yes** | `Database.SaveAs(path, DwgVersion.Current)` on a copy, or copy the file after a normal save. Caveat: the copy carries the derived-drawing link; the Document Manager may see two drawings, and renaming breaks the link (restorable with `_AstorDetRegist`). | [V] link/rename behaviour, [I] copy semantics |
| Undo | **Yes** | Wrap each MCP "operation" in `UNDO _Begin` … `UNDO _End` (via `Editor.Command`), so one `_.UNDO 1` reverts it. Pure-API edits can also run in an AutoCAD `Transaction` and be aborted. AS commands are normal AutoCAD commands and join the undo stack. | [I] (standard AutoCAD) |

**Bottom line:** A PoC is feasible if it is built as an **in-process AutoCAD .NET plugin** inside Advance Steel 2022. The plugin uses AutoCAD API calls for read/inspect/overlap/PNG/title block/erase/save/undo, and scripts `_AstM4…` commands for view moves and AS-native edits. Plain AutoCAD dimensions and leaders cover "add annotation" cases where associativity is not essential. **View scale change and fully associative AS dimensions/labels are the weak spots** and need a hands-on spike. AcCoreConsole (headless) is **not** an option for AS objects [V].

---

## 1. Advance Steel API: assemblies, namespaces and drawing coverage

### 1.1 Assemblies and namespaces (AS 2022)

From the AS 2022 "Advance Steel .NET and COM API" page and the "Hello World" walkthrough [V]:

| DLL | Contents |
|---|---|
| `ASNetRuntime.dll` | `Autodesk.AdvanceSteel.Runtime`: `ExtensionApplicationAttribute`, `IExtensionApplication`, `CommandClassAttribute`, `CommandMethodAttribute`, `CommandFlags` |
| `ASMgd.dll` | `Autodesk.AdvanceSteel.ExtDb`, `.ServicesDatabaseUtils`, `.Application`, `.BuildingStructure`, `.CADAccess` (Transaction, TransactionManager, FilerObject, DwgManager), `.ConstructionTypes`, `.DocumentManagement` (DocumentManager, Document, DocumentAccess, OpenDatabase) |
| `ASObjectsMgd.dll` | `.Arrangement`, `.Connection`, `.ConstructionHelper` (Camera), `.Contours`, `.Exceptions`, `.InteropServices`, `.Modelling` (beams, plates, Grid), `.StructuralAnalysis`, `.Utils` |
| `ASCADLinkMgd.dll` | `Autodesk.AdvanceSteel.CADLink.Database` (ObjectId, …) |
| `ASGeometryMgd.dll` | `Autodesk.AdvanceSteel.Geometry` |
| `ASProfilesMgd.dll`, `ASModelerMgd.dll` | profiles DB, modeler |
| `Extensions\DotNetRoots.dll` | `Autodesk.AdvanceSteel.DotNetRoots` (for example `Units.UnitsSet`) |
| `ASSettingsMgd.dll`, `ASSettings.dll` | settings (referenced by Dynamo for AS) |

Install location for AS 2022: `C:\Program Files\Autodesk\AutoCAD 2022\ADVS` (from the Dynamo-Advance-Steel `AS2022_2.10.1` branch build props; the same branch targets .NET Framework v4.8) [V]. The AS 2022 Hello World page cites registry key `HKLM\SOFTWARE\Autodesk\AutoCAD\R23.1\ACAD-3026` and ".NET Framework 4.7". Those look like stale text carried over from earlier releases. AutoCAD 2022 is R24.1, so check the registry on the target machine [I].

### 1.2 What exists that touches drawings (all model-side)

| API | What it does | Version | Evidence |
|---|---|---|---|
| `KernelServices.GetDerivedDocumentsForDwg(OpenDatabase)` → `DerivedDocument[]` → `GetDerivedObjects()` → `DerivedObject.GetInfo()` as `DerivedObjectDetailInfo` → `GetSinglePartsDetailed()`, `GetMainPartsDetailed()` | Lists the drawings derived from a model and which parts/assemblies each one details | ≥2018. Since 2019 it must run inside an AS transaction | [C] forum code, confirmed working on 2020 after adding a transaction |
| `DwgManager.IsCurrentDWGDerived(out strParentDwgId)` | True if the current DWG is a detail drawing; returns the parent model id | **new in 2022** | [V] 2022 API changes |
| `Camera` / `ModelViewObject`: `SetObjectsFilteredForDetailing`, `Get…FilteredForDetailing`, `Mark…`, `Reset…`, `DetailingFilterEnabled` | Restrict which objects a camera or model view details (inputs to drawing creation) | **new in 2022** | [V] 2022 API changes |
| `Camera.setSelectedQueries` / `getSelectedQueriesNames`; `BuildingStructure.getQuery/getAllQueries` | Search-filter queries used when detailing | 2023 | [V] 2023 API changes |
| `Grid.VerticalProjection` etc. | Grid projection in GA drawings and model views | 2023 | [V] |
| `DocumentManagement.ModelReport`, `Services.EqualParts` | Equal-parts detection feedback | 2023 | [V] |
| `KernelServices.RunBOMProcess(OpenDatabase, ObjectId[], BOMTemplateInfo[])` | Runs BOM (lists) from the API | signature changed in 2022 | [V] |
| `DatabaseUtils.setDwgDataDefault` | Change AS defaults (the same values Management Tools edits) from code | long-standing | [V] Autodesk architect on forum |

**Not present in any version (2022 → 2027):** a class for detail views, drawing dimensions, labels, drawing styles or drawing processes as objects. There is also no "create drawing" API. The Autodesk "API Changes" pages for 2024, 2025, 2026 and 2027 carry the same body text as 2023: no new API entries were published after 2023, which matches maintenance mode [V, by comparing the pages]. The 2021 change log also contains nothing drawing-related [V].

### 1.3 Official and expert statements

- **Autodesk (Mihai Dobrescu, Software Architect, 2018)** [V]: *"we are not offering very good and easy to use API today for numbering and detailing"*. The workarounds he lists are Management Tools defaults, `AstM4SetDefault`, `DatabaseUtils.setDwgDataDefault` and calling detailing commands from the API. *"selecting an already existing detail drawing as the output for a new detail or detail process is not possible right now"*. Both items went to the backlog, and nothing has shipped since.
- **Christian Blei (long-time AS API developer, 2024)** [C]: *"There is nothing in the API that deals with Detail styles, except maybe the option to start processes"*. The API was designed without detailing functions, and the AutoCAD API cannot be used because *"for it the AdvS objects are proxies"*.
- **Sebastian Eiche (Mensch und Maschine, 2024)** [C]: *"you have no chance to get access to the layout via the AS API"*.
- A user (2022) reported that a drawing dimension is seen by the AutoCAD .NET API as `Autodesk.AutoCAD.DatabaseServices.ImpEntity` [C].
- A community answer to "Generate 2D drawing with .NET API" (2023) [C]: there is no drawing-creation API. Use Drawing Style Manager / Drawing Process Manager / Drawing Process Suite, or edit their database entries, or draw your own geometry with the AutoCAD API.
- **Maintenance mode** [V]: Autodesk blog, 29 March 2023. No new features for Advance Steel; the focus moved to Revit and cloud.

### 1.4 COM API

The COM API ("SteelAutomation", `IAstUI`, `IJoint`, …) is aimed at joints and modelling [V, AS 2022 API page]. No forum source or documentation shows COM access to drawing views or dimensions. Asked directly, experts say there is none [C].

---

## 2. How AS drawings are stored and what they look like to AutoCAD

### 2.1 Separate DWGs, linked to the model

- Each detail drawing is a **separate DWG derived from the model DWG** [V]. The **Document Manager** lists drawings, BOMs and NC files, detects which need updating after model changes, controls updates and manages revisions.
- The link breaks when the model or the drawing is renamed or moved. It can be restored with `_AstorDetRegist` (Register documents to model) and removed with `_AstorDetDeRegist` [V]. AS checks that the document really derives from that model, so foreign DWGs cannot be registered [V].
- In AS 2022, a new detail can be placed on an existing registered drawing through the "Modify settings" dialog when creating a detail ("existing drawing settings") [V, 2022 "Create a Detail Drawing"]. This is interactive, not API.

### 2.2 Entities in the drawing DWG

| Thing on the sheet | Representation | Evidence |
|---|---|---|
| Detail view | AS custom entity, class `AstAtomicDetView` (parent `AstDbObject`) | [C] forum screenshot / class name |
| AS dimensions, labels, level symbols, weld symbols | AS custom entities. AutoCAD .NET wraps them as `ImpEntity` (no managed wrapper class) | [C] |
| Page frame | Block `HYPERSTEELPAGEFRAME` | [V] "Define the Prototype Frame" |
| Title block / page header | Block `HYPERSTEELPAGEHEADER`, with **attribute definitions whose tags are AS tokens** (`PROJECT`, `DRAWING_NO`, `SCALE`, `POS_NUM`, `REVISION_INDEX`, `USERATTR1..10`, `APPROVAL_STATUS`, …) | [V] "Edit the Prototype Page Header", "Title Block Token / Attribute List" |
| Reserved area for view placement | Block `ADVANCESTEEL_RESERVED_AREA` (rectangle) | [V] |
| BOM on drawing, revision table | AS objects (`_AstM4CommDetInsertBom`, `_AstM4CommDetRevisionTable`) | [V] commands |
| Prototypes | DWGs in `C:\ProgramData\Autodesk\Advance Steel 2022\<LANG>\Shared\Support\Prototypes` or per project in `<model>\Prototypes`; names `ASDETPROTO*` / `mm_ASDETPROTO*` | [V] |

Whether views live in **model space or a paper-space layout** of the drawing DWG is not stated in the pages reviewed. **Check this in the spike** (inspect `BlockTableRecord`s and `Layout`s of a generated drawing) [open question].

### 2.3 Can plain AutoCAD API read/modify them?

- **Inside Advance Steel** (object enablers loaded): the AutoCAD API can enumerate these entities, get `ObjectId`, `Handle`, `Layer`, `GetRXClass().Name` / `DxfName`, `GeometricExtents`, and call `Erase`, `TransformBy`, `Explode` [I]. It **cannot** read semantic properties (dimension value, referenced model object, view scale, view name) because no managed wrapper exists [C]. Some properties may be exposed through the Properties palette (OPM) and therefore via COM `AcadEntity` dynamic properties. Check this in the spike [I].
- **Outside Advance Steel** (plain AutoCAD, TrueView, AcCoreConsole): AS objects are **proxies** and cannot be edited [C, V for AcCoreConsole below].
- **Detail explode** [V] converts AS details into plain AutoCAD entities according to the explode profile (`_AstM4CommExplodeDetail` settings; batch explode from the Document Manager). It is destructive: the result is no longer associative. Useful on a *copy* for geometry extraction.

### 2.4 AutoCAD dimensions vs AS dimensions on update

From AS 2022 "Manual Dimensions" and "Modify Automatic Dimensions on Drawings" [V]:

- AS (parametric/manual) dimensions show the correct value regardless of drawing scale, move and scale with the detail views, respect clipping, and use "smart dimension points" linked to 3D model points so they update when geometry changes.
- **AutoCAD dimensions "will never update or follow an Advance Steel detail view"**, and their scale must be entered manually per dimension.
- Automatic dimensions:
  - **Deleted** ones stay deleted on update, except when the drawing style has several dimension requests for the same object. They also come back if the model object is deleted and recreated.
  - **Manually repositioned** ones lose their link to the automatic rules and keep the manual position on update.
  - Dimensions edited with **Insert/Delete point** become hybrids.
- Manual dimensions follow the **"Manual dimension point behavior on update"** default (Management Tools → Drawing-Dimensioning). The default setting erases invalid points; the alternatives are not erasing, or erasing only points no longer on objects.
- **Update detail** (`_AstM4CommDetUpdateDetail`) redraws a modified detail. **Revision clouds in the detail are lost** on update [V]. What happens to plain AutoCAD entities placed on the drawing during update is not documented. Assumption: they persist but go stale [I]. Test this in the spike.

---

## 3. Built-in automation alternatives

| Mechanism | What it controls | Scriptable? | Evidence |
|---|---|---|---|
| **Drawing Style Manager** (`_AstM5DetailStylesManagerNew`) | Views per style, view direction/content, model boxes, clipping, **dimension requests** (dimension chains per object type, direction, point selection, arrangement flags G/L, T/B, A/S), labeling strategies, view arrangement strategies (distance between views, manual/automatic placement), dimension styles | Config only (stored in the AstorDetails database) | [V] |
| **Detail style map scripts** | Script filters (for example `DetObjQualifierFilter`) that decide which objects get which style | Config (DB) | [C] forum |
| **Drawing processes** (`_AstM5DetailProcessExplorer`; run with `_AstM4CommDetailingProc User <n>`, for example `User 52` = "All Assembly PageFull A1") | Batch creation of drawings for selections with sorting, prototypes and multi-sheet rules | **Yes**, from the command line, so from `SendStringToExecute` | [V] |
| **Create detail from a style**: `_AstM4CommDetailing User <a> <b>` | Single detail with a given style | Yes (command). Prompts depend on the style | [C] Graitec DSM guide / search snippet |
| **Management Tools** defaults (AstorSettings DB) | Everything default-driven (numbering, drawing, dimensioning, presentation) | Via `AstM4SetDefault` command or `DatabaseUtils.setDwgDataDefault` API | [V] |
| **Layout detail prototype** ("Define drawing layout") | Centre, column/row arrangement of views on a sheet | Config | [V] |
| **Document Manager** | Update, revision, approval status, explode, delete | Mostly UI. Update of single details via `_AstM4CommDetUpdateDetail` (select the gray rectangle) | [V] |
| **Dynamo for Advance Steel** | Modelling only: beams, plates, bolts, features, grids, **cameras** (create by CS, set clipping/extents) | No drawing/detail nodes in the source (DynamoDS/Dynamo-Advance-Steel, branches AS2017 → AS2027) | [V] source code |
| **Graitec PowerPack** (commercial) | Extra drawing and checking tools | Commercial | [C] |

**Useful `_AstM4…` commands confirmed in AS 2022 help** [V]:

| Command | Purpose |
|---|---|
| `_AstM4DetViewTransform` | Move a view (P1 → P2); the position is kept on update |
| `_AstM4DetChange` | Edit Detail dialog: scale, clipping, title per view / whole detail |
| `_AstM4DetManualCut` | Create a cut view from a detail |
| `_AstM4CommDetUpdateDetail` | Update a modified detail |
| `_AstM4CommDetailingProc User <n>` | Run drawing process n |
| `_AstM4CommDetUpdateBOM`, `_AstM4CommDetInsertBom`, `_AstM4CommBillOfMaterial` | BOM on drawing / lists |
| `_AstM4CommDetRevisionTable`, `_AstM4CommDetailRevision` | Revision table / revision |
| `_AstM4CommExplodeDetail` | Detail explode settings |
| `_AstM10EditPrototypes` | Edit prototypes |
| `_AstorDetRegist`, `_AstorDetDeRegist` | Register / cancel documents to/from model |
| `_AstM5DetailStylesManagerNew`, `_AstM5DetailProcessExplorer` | Drawing Style Manager / Drawing Process Manager |

The **parametric dimension, label, insert/delete dimension point and arrange-labels commands** are documented only as ribbon buttons (Labels & Dimensions tab → Parametric Dimensions / Parametric Views panels). Their `_AstM4…` command names are not in the help pages. **Action:** in the spike, read them from the Advance Steel CUIx/ribbon macros (`CUI` → Labels & Dimensions) or from the command history after clicking each button.

---

## 4. Plugin architecture

### 4.1 Loading and command registration

Two options, both valid inside AS 2022:

1. **AS-native addon** [V]. Reference `ASNetRuntime.dll` + AS DLLs (Copy Local = False, x64). Add `[assembly: ExtensionApplication(typeof(Plugin))]` and `[assembly: CommandClass(typeof(Commands))]`, plus methods with `[CommandMethodAttribute("GROUP","Name","LocalName", CommandFlags.Modal|CommandFlags.UsePickSet|CommandFlags.Redraw)]`. Register by placing an XML (`<AddonsData><Addons><Addon><Name/><FullPath/></Addon></Addons></AddonsData>`) in an **`Addons` folder directly under the AS install folder**. Debug by attaching to `acad.exe`. Examples: AS 2022 "Walkthrough: Hello World", `flecheria/ADVSTemplate`.
2. **AutoCAD-native plugin** [I, standard AutoCAD]. Reference `AcCoreMgd`, `AcDbMgd`, `AcMgd` (AutoCAD 2022) + AS DLLs. Use `Autodesk.AutoCAD.Runtime.IExtensionApplication` / `[CommandMethod]`, loaded with `NETLOAD`, an autoloader `.bundle` (`PackageContents.xml`) or the registry. The official walkthrough "Working with AutoCAD and AS API" shows AutoCAD and AS APIs mixed in one plugin, using aliases for clashing names (`ObjectId`, `Transaction`, `Document`) [V].

**Recommendation:** use option 2 for the MCP bridge. Most drawing work is AutoCAD API, and the autoloader bundle is the easiest way to deploy. Call AS API parts (derived documents, `DwgManager`) only when needed.

### 4.2 AS transactions and document locking

Since AS 2019, AS API access needs an explicit transaction [C/V]. Two patterns work:

```csharp
// Pattern used by Dynamo for AS (OneTransactionPerContext.cs) and ADVSTemplate
DocumentManager.LockCurrentDocument();
var t = Autodesk.AdvanceSteel.CADAccess.TransactionManager.StartTransaction();
try { /* AS API */ t.Commit(); } finally { DocumentManager.UnlockCurrentDocument(); }

// or (AS walkthrough)
using (var da = new DocumentAccess(null, false)) { /* AS API */ da.Commit(); }
```

For AutoCAD API edits, use `using (doc.LockDocument())` plus an AutoCAD `Transaction` as usual [I].

### 4.3 In-process server and main-thread marshalling

AutoCAD APIs must be called on the main (UI) thread with the document locked. Suggested design [I, standard AutoCAD practice]:

- The MCP server (Node/Python, stdio for Claude Desktop) talks to the plugin over a **named pipe** (`\\.\pipe\as-mcp`) or `HttpListener` on `127.0.0.1`. The listener runs on a background thread inside `acad.exe`.
- Marshalling requests to the main thread:
  - **Commands** (anything calling `_AstM4…`, `UNDO`, `PLOT`): `await Application.DocumentManager.ExecuteInCommandContextAsync(async _ => await ed.CommandAsync(...), null)`. This runs in document command context and allows `Editor.Command`/`CommandAsync` from a non-command context. It was introduced in AutoCAD 2016 (`BeginExecuteInCommandContext` in 2015). See Kean Walmsley, "AutoCAD 2016: Calling commands from external events using .NET".
  - **Pure database reads/writes**: queue work items and drain them in the `Application.Idle` event (documented "fired when the application goes idle"), or post to a `SynchronizationContext` captured in `Initialize()`. Lock the document in each item.
  - **Fallback**: `Document.SendStringToExecute("…\n", activate:true, wrapUpInactiveDoc:false, echoCommand:false)`. It is fire-and-forget, so completion must be detected separately (for example the `CommandEnded` event).
- **Selecting AS entities for commands**: pass the ObjectId/`ObjectId[]` directly to `Editor.Command(...)`, or use `(handent "<handle>")` in a string command [I].
- **Modal dialogs** (`_AstM4DetChange`, drawing style dialogs) block the command. Avoid them, or drive them with UI Automation as a last resort.

### 4.4 Undo

- Group each MCP tool call: `ed.Command("_.UNDO", "_Begin")` … `ed.Command("_.UNDO", "_End")`. The MCP `undo` tool then calls `_.UNDO 1` [I].
- COM equivalents: `AcadDocument.StartUndoMark()` / `EndUndoMark()` [I].
- For pure-API operations, an uncommitted AutoCAD `Transaction` rolls back. After commit, the change is on the undo stack like any command [I].
- AS commands are AutoCAD commands and join the undo stack [I].

### 4.5 PNG export

- **Plot to file**: use `PlotFactory.CreatePublishEngine()` / `PlotEngine` with `PlotSettingsValidator` set to device `"PublishToWeb PNG.pc3"`, a custom pixel media size, `PlotType.Extents` or `Window`, and `PlotToFile = true` [V API, I config]. `-PLOT` can also be scripted from the command line. Plot uses a separate engine, so check `BACKGROUNDPLOT = 0` [I].
- **Quick view capture**: `PNGOUT` / `JPGOUT` commands capture the current viewport at screen resolution (zoom to a window first). `Document.CapturePreviewImage(w,h)` is another option [I].
- AS entities render in plots as displayed when AS is running [I]. Plain AutoCAD would show proxy graphics.
- For "view crops", zoom or plot a window equal to the view's `GeometricExtents` plus a margin.

### 4.6 AcCoreConsole (headless)

**Not viable for AS objects.** Bogdan Orzea (Autodesk QA) wrote that editing AS objects in AcCoreConsole will not succeed because Advance Steel does not work there [V, Autodesk staff]. A 2025 community post says the console can start with `/isolate`, but that `/product ADVS` fails and AS objects stay proxies without undocumented extra steps [C, unverified, unsupported]. So the PoC must run inside the full AS GUI session.

### 4.7 Save copy

- `db.SaveAs(path, DwgVersion.Current)` for a copy (the open document keeps its name only if you save a separate `Database` clone). Alternatively, save normally and then `File.Copy` [I].
- Renaming or moving a drawing breaks its Document Manager link, which can be restored with `_AstorDetRegist` [V]. Save PoC outputs as copies in a sandbox folder and leave the registered originals untouched unless the user asks otherwise.

---

## 5. Open-source code on GitHub

| Repo | Relevance | Drawing API use? |
|---|---|---|
| [DynamoDS/Dynamo-Advance-Steel](https://github.com/DynamoDS/Dynamo-Advance-Steel) (Autodesk, branches `AS2017_1.2.0` … `AS2022_2.10.1` … `AS2027_4.0.0`) | Best reference for AS API usage, transactions (`OneTransactionPerContext.cs`), references (`ASMgd`, `ASObjectsMgd`, `ASCADLinkMgd`, `ASGeometryMgd`, `DotNetRoots`, `ASSettingsMgd`), install paths, Camera nodes | **No**. Modelling and cameras only |
| [flecheria/ADVSTemplate](https://github.com/flecheria/ADVSTemplate) | Minimal AS addon template (`IExtensionApplication`, `CommandMethodAttribute`, lock + transaction) | No |
| [tawadev/AdvanceSteel](https://github.com/tawadev/AdvanceSteel), [isatsteel/AdvanceSteel](https://github.com/isatsteel/AdvanceSteel) (same code) | AS addon mixing AutoCAD `DocumentLock` + AS transactions | No |
| [Xillis/MBS-AdvanceSteel-Automation-](https://github.com/Xillis/MBS-AdvanceSteel-Automation-) | Scripts AS commands (`AstM4CommMarkSelAdd`, `AstM10ViewSelObjects`, …) | No (model-side commands) |
| [jeremytammik/RevitSdkSamples – SampleCommandsSteelElements](https://github.com/jeremytammik/RevitSdkSamples) | Uses `Autodesk.AdvanceSteel.*` inside Revit (steel connections) | No |
| [TheRedJMMonkey/export_advance_steel_to_iges](https://github.com/TheRedJMMonkey/export_advance_steel_to_iges) | AutoLISP in AS | No |

GitHub code search found **no** public code for `KernelServices.GetDerivedDocumentsForDwg`, `AstAtomicDetView` or `_AstM4CommDetailingProc`. No open-source project manipulates AS drawings programmatically. The PoC would be the first public one.

---

## 6. Recommended spike (first 1–2 days on the Windows box)

1. NETLOAD a probe DLL in AS 2022. Open a generated single-part, assembly and GA drawing, then dump every entity: RXClass name, DxfName, handle, layer, space/layout, extents and managed type. Record the real class names (view, dimension, label, BOM, revision cloud).
2. Try `Entity.Explode` in memory on a view and on an AS dimension. Record what comes back (lines, texts, values?).
3. Dump COM `AcadEntity` properties for a view and a dimension (dynamic OPM properties). Check whether scale, view name or dimension value are exposed.
4. Script `_AstM4DetViewTransform` via `ExecuteInCommandContextAsync` + `Editor.CommandAsync(viewId, p1, p2)`. Then run update detail and confirm the position persists.
5. Find the command names of the parametric dimension and label buttons (CUIx). Test scripting each one with explicit points.
6. Add an AutoCAD `RotatedDimension` and an `MLeader`, run update detail, and check whether they survive.
7. Edit a free `HYPERSTEELPAGEHEADER` attribute and a token attribute, update, and see which ones get overwritten.
8. Plot the drawing extents to PNG with `PublishToWeb PNG.pc3`.
9. Wrap steps 4–7 in `UNDO _Begin/_End` and verify that a single `UNDO 1` reverts each step.

---

## Sources

**Autodesk documentation (Advance Steel)**
- AS 2022 .NET and COM API (DLL/namespace split): https://help.autodesk.com/cloudhelp/2022/ENU/AdvSteel-Developers/files/GUID-AF2072A9-3F8F-4DAC-B595-470A9E32BEE0.htm
- AS 2022 .NET and COM API Changes: https://help.autodesk.com/cloudhelp/2022/ENU/AdvSteel-Developers/files/GUID-309B2ACB-5CD4-4B91-98B7-D00F2026BBD7.htm
- AS 2023 API Changes: https://help.autodesk.com/cloudhelp/2023/ENU/AdvSteel-Developers/files/GUID-309B2ACB-5CD4-4B91-98B7-D00F2026BBD7.htm
- AS 2024 / 2025 / 2026 / 2027 API Changes (same body as 2023): https://help.autodesk.com/cloudhelp/2025/ENU/AdvSteel-Developers/files/GUID-309B2ACB-5CD4-4B91-98B7-D00F2026BBD7.htm (replace the year)
- AS 2021 API Changes: https://help.autodesk.com/cloudhelp/2021/ENU/AdvSteel-Developers/files/GUID-309B2ACB-5CD4-4B91-98B7-D00F2026BBD7.htm
- AS API Guide / Getting Started: https://help.autodesk.com/cloudhelp/2022/ENU/AdvSteel-Developers/files/GUID-E1F6E27F-4942-4683-9368-76AD9F1E6016.htm , https://help.autodesk.com/cloudhelp/2022/ENU/AdvSteel-Developers/files/GUID-0BA67D2B-4289-43FC-92B3-6CCFE4E22A8F.htm
- Walkthrough: Hello World (references, attributes, Addons XML): https://help.autodesk.com/cloudhelp/2022/ENU/AdvSteel-Developers/files/GUID-A4DA627E-6680-4388-9C04-79F5F3D9D075.htm
- Walkthrough: Working with AutoCAD and AS API: https://help.autodesk.com/cloudhelp/2022/ENU/AdvSteel-Developers/files/GUID-0AFEFD2B-EED4-4F6A-8F97-BA0814E99643.htm
- Document Manager: https://help.autodesk.com/cloudhelp/2022/ENU/AdvSteel-Drawings/files/GUID-FC80ADEF-5B22-4A7E-9BF3-DE933AD5DA15.htm
- Register / Cancel documents: https://help.autodesk.com/cloudhelp/2022/ENU/AdvSteel-Drawings/files/GUID-33DF948B-81BE-4FE5-A049-755777725C0C.htm , https://help.autodesk.com/cloudhelp/2022/ENU/AdvSteel-Drawings/files/GUID-06335491-DA46-4C26-8B9F-480ED885C789.htm
- Move a View: https://help.autodesk.com/cloudhelp/2022/ENU/AdvSteel-Drawings/files/GUID-27F8BF44-A363-4F51-94CB-27F3D284D8B7.htm
- Change a Detail: https://help.autodesk.com/cloudhelp/2022/ENU/AdvSteel-Drawings/files/GUID-7CE73442-F35F-4A80-AB7C-53312C6C3C43.htm
- Update a Detail: https://help.autodesk.com/cloudhelp/2022/ENU/AdvSteel-Drawings/files/GUID-58840661-450B-492A-B929-ED4ACF006A78.htm
- Manual Dimensions: https://help.autodesk.com/cloudhelp/2022/ENU/AdvSteel-Drawings/files/GUID-D462912E-5259-476F-9EC8-B27A361B7A0E.htm
- Modify Automatic Dimensions on Drawings: https://help.autodesk.com/cloudhelp/2022/ENU/AdvSteel-Drawings/files/GUID-40D8C07C-47A9-4876-BAB9-5DF4187521F5.htm
- Manual dimension point behavior on update default: https://help.autodesk.com/cloudhelp/2022/ENU/AdvSteel-Drawings/files/GUID-86265F98-594E-4586-88A1-8DD8E2B83D9E.htm
- Create a Detail Drawing: https://help.autodesk.com/cloudhelp/2022/ENU/AdvSteel-Drawings/files/GUID-30062C5A-58CF-4915-9984-44CB2AD7AB5D.htm
- Use a Drawing Process: https://help.autodesk.com/cloudhelp/2022/ENU/AdvSteel-Drawings/files/GUID-D81EFD24-E966-44F6-908A-F3A41BF0EBE6.htm
- Drawing Prototypes: https://help.autodesk.com/cloudhelp/2022/ENU/AdvSteel-Drawings/files/GUID-00036924-E652-4D4A-B121-B0CE96859CE5.htm
- Edit the Prototype Page Header: https://help.autodesk.com/cloudhelp/2022/ENU/AdvSteel-Drawings/files/GUID-49CAAEE7-2062-4966-90B3-866381702BBA.htm
- Define the Prototype Frame: https://help.autodesk.com/cloudhelp/2022/ENU/AdvSteel-Drawings/files/GUID-612DFA29-CCCF-4C46-9C2F-D1F68593352B.htm
- Add a Reserved Area: https://help.autodesk.com/cloudhelp/2022/ENU/AdvSteel-Drawings/files/GUID-479B20CE-F93A-4E32-8534-C128733E8484.htm
- Title Block Token / Attribute List: https://help.autodesk.com/cloudhelp/2022/ENU/AdvSteel-Drawings/files/GUID-8831F91F-4AFE-489C-8DCD-CDED8DD64DD1.htm
- Detail Explode Settings: https://help.autodesk.com/cloudhelp/2022/ENU/AdvSteel-Drawings/files/GUID-E24FB8CF-AC99-4895-943F-154532EABF72.htm
- Dimension Request / Dimension Arrangement: https://help.autodesk.com/cloudhelp/2022/ENU/AdvSteel-Drawings/files/GUID-09FC505F-B067-4EE9-9DA2-A881DDF08699.htm , https://help.autodesk.com/cloudhelp/2022/ENU/AdvSteel-Drawings/files/GUID-97C690AD-E9F3-4294-AD4A-6EE10F0FC01E.htm
- View Arrangement Strategies / Detail Arrangement Settings: https://help.autodesk.com/cloudhelp/2022/ENU/AdvSteel-Drawings/files/GUID-97FB601B-437E-473C-8836-1AEE7040C29F.htm , https://help.autodesk.com/cloudhelp/2022/ENU/AdvSteel-Drawings/files/GUID-35FA7479-4832-47BE-9772-25A446713271.htm
- Maintenance mode announcement (Autodesk blog, 2023-03-29): https://www.autodesk.com/blogs/aec/2023/03/29/advancing-steel-detailing-and-design-at-autodesk/ and forum thread https://forums.autodesk.com/t5/advance-steel-forum/advance-steel-officially-in-maintenance-mode/td-p/11859471

**Autodesk forums (Advance Steel Forum)**
- API options for numbering and detail drawing process (Autodesk architect answer): https://forums.autodesk.com/t5/advance-steel-forum/advance-steel-api-options-for-numbering-and-detail-drawing/td-p/8216125
- Advance Steel API – access Layout elements (AstAtomicDetView, "no API for detail styles", proxies): https://forums.autodesk.com/t5/advance-steel-forum/advance-steel-api-access-layout-elements/td-p/12547358
- Generate 2D drawing with .NET API: https://forums.autodesk.com/t5/advance-steel-forum/generate-2d-drawing-with-net-api/td-p/11763769
- Advance Steel 2020 API – Derived documents (DerivedDocument code): https://forums.autodesk.com/t5/advance-steel-forum/advance-steel-2020-api-derived-documents/td-p/9197656
- Drawing dimensions in single part drawing via .NET API (ImpEntity): https://forums.autodesk.com/t5/advance-steel-forum/drawing-dimensions-in-single-part-drawing-via-net-api/td-p/11294963
- Advance Steel crashes AcCoreConsole (Autodesk QA: AS does not work in AcCoreConsole): https://forums.autodesk.com/t5/advance-steel-forum/advance-steel-crashes-accoreconsole/td-p/5910250
- Plate edge dimensioning (limits of automatic dims): https://forums.autodesk.com/t5/advance-steel-forum/plate-edge-dimensioning/td-p/12495426
- Detail Style Map Scripts: https://forums.autodesk.com/t5/advance-steel-forum/detail-style-map-scripts/td-p/13384623
- Update Detail Command: https://forums.autodesk.com/t5/advance-steel-forum/update-detail-command/td-p/12065584
- Graitec Advance Steel DSM guide (`_astm4CommDetailing User` syntax): http://www.graitec.info/Common/Downloads/Documentation/AS-DSM_guide-8.1-EN.pdf

**AutoCAD API**
- DocumentCollection class (ExecuteInCommandContextAsync): https://help.autodesk.com/cloudhelp/2022/ENU/OARX-ManagedRefGuide/files/OARX-ManagedRefGuide-Autodesk_AutoCAD_ApplicationServices_DocumentCollection.html
- Application.Idle: https://help.autodesk.com/cloudhelp/2022/ENU/OARX-ManagedRefGuide/files/OARX-ManagedRefGuide-Autodesk_AutoCAD_ApplicationServices_Core_Application_Idle.html
- PlotEngine: https://help.autodesk.com/cloudhelp/2022/ENU/OARX-ManagedRefGuide/files/OARX-ManagedRefGuide-Autodesk_AutoCAD_PlottingServices_PlotEngine.html
- Plot your drawing (.NET dev guide): https://help.autodesk.com/cloudhelp/2016/ENU/AutoCAD-NET/files/GUID-CE91C625-A278-4445-8219-3A2D880E6C22.htm
- Kean Walmsley – Calling commands from external events (ExecuteInCommandContextAsync): https://keanw.com/2015/03/autocad-2016-calling-commands-from-external-events-using-net.html
- Kean Walmsley – AutoCAD 2015: calling commands: https://keanw.com/2014/03/autocad-2015-calling-commands.html
- Kean Walmsley – Driving a basic AutoCAD plot using .NET: https://keanw.com/2007/09/driving-a-basic.html
- AutoCAD Core Console: https://help.autodesk.com/view/OARX/2025/ENU/?guid=AUTOCAD_CORE_CONSOLE

**GitHub**
- https://github.com/DynamoDS/Dynamo-Advance-Steel (see `src/DynamoAdvanceSteel/OneTransactionPerContext.cs`, `src/Config/user_local.props` on branch `AS2022_2.10.1`, `src/AdvanceSteelNodes/Miscellaneous/Camera.cs`)
- https://github.com/flecheria/ADVSTemplate
- https://github.com/tawadev/AdvanceSteel
- https://github.com/Xillis/MBS-AdvanceSteel-Automation-
