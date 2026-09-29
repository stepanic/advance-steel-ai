# Advance Steel 2022 Drawing Cleanup via Claude/MCP: What Is Actually Possible

The short answer is no: Advance Steel 2022 has no documented or community-proven way to change a view's scale without the dialog. What you can do is wrap the native, interactive Advance Steel drawing commands inside AutoCAD, drive them with explicit picks, and put most of the "AI tuning" effort into drawing styles and processes (stored in SQL LocalDB .mdf files) rather than into editing sheets after the fact. Most of what you listed as "established" is confirmed. Two points need correcting: exploding does not produce clean AutoCAD geometry, and the API gap is a deliberate design decision, not a side effect of maintenance mode.

## TL;DR

- **Your API assessment holds.** The Advance Steel .NET/COM API has no drawing-side access. A long-time API developer on the Autodesk forum says this was a deliberate decision from the start, not something lost when the product went into maintenance mode in March 2023. No source shows a way to change the scale of an existing view other than `_AstM4DetChange` (the Edit detail dialog) or double-clicking the view frame, which opens the same dialog. Any scale change also needs a "quick update" against the registered model.
- **What survives an update is partly documented.** Deleted automatic dimensions and deleted dimension points stay deleted. There is one exception: when several combined dimension requests cover the same object, a hidden one reappears. Moved views keep their position. AutoCAD dimensions never follow the view. Revision clouds are lost on Update detail. Hand-typed title-block values that map to tokens are overwritten by "Update Page Header". No source says what happens to plain AutoCAD lines, text or leaders.
- **Recommendation:** make "tune the drawing style/process per company rule" the main lever. Leave a thin layer of overnight edits: move views, add manual Advance Steel dimensions and labels through the native commands, and fill non-token title-block attributes. Before building any MCP tooling, spend the first week on the spike experiments listed below. Upgrading past 2022 buys nothing on the drawing side, and Autodesk's own MCP servers do not touch Advance Steel.

## Key Findings: Verification of Your "Established" Assumptions

| Your assumption | Verdict | Evidence |
|---|---|---|
| No drawing-side API in AS 2022+ | **Confirmed** | Christian Blei (Advisor, CBT Christian Blei Tools, Autodesk forum, March 2024): "There is nothing in the API that deals with Detail styles, except maybe the option to start processes, if att all. This is due to a decision in the beginning of the AdvS, API when it was decided not to include detailing functions."\[1\] Sebastian Eiche (Mensch und Maschine): "you have no chance to get access to the layout via the AS API."\[1\] https://forums.autodesk.com/t5/advance-steel-forum/advance-steel-api-access-layout-elements/td-p/12547358 [Community]. John Bennett, in a separate thread: "There is no API for drawing creation for advance steel… if you can understand the database relationships and required entries you can also manage via that DB entries." https://forums.autodesk.com/t5/advance-steel-forum/generate-2d-drawing-with-net-api/td-p/11763769 [Community]\[2\] |
| Maintenance mode since 2023 | **Confirmed, with a nuance** | David Smolker, Autodesk AEC blog, "Advancing Capabilities for Steel Detailing and Design in BIM" (29 March 2023): "pivoting away from developing new features in Advance Steel and placing it on maintenance mode… Autodesk is committed to maintaining Advance Steel's existing features and quality." The same post names the partners to turn to: "Autodesk recommends Advance Steel users can find enhanced value through: Graitec and its PowerPack for Advance Steel [and] Mensch und Maschine and its M+M practice package steel construction." https://www.autodesk.com/blogs/aec/2023/03/29/advancing-steel-detailing-and-design-at-autodesk/ [Verified: official]. Yearly versions still ship: Autodesk hosts "Welcome to Advance Steel 2027 Learning" with a "What's New in Advance Steel 2027" section (help.autodesk.com/view/ADSTPR/2027/ENU/), and Graitec (11 Aug 2026) says "Graitec Steel 2027 runs on Autodesk Advance Steel 2026 and 2027." |
| Detail views are `AstAtomicDetView` custom entities | **Confirmed** | The forum thread above shows a plate outline in a drawing reported as "AstAtomicDetView".\[1\] Autodesk support articles cite the error "Class 'AstAtomicDetView' parent named 'AstDbObject' not found", which ties the class to `asobjectscad.dbx`.\[3\] https://www.autodesk.com/support/technical/article/caas/sfdcarticles/sfdcarticles/Exception-in-asobjectscad-dbx-ARX-command-when-starting-Advance-Steel.html [Verified: official] |
| AS dimensions are opaque `ImpEntity` | **Not verified** | No source found. Blei says AutoCAD sees the AS objects as proxies ("for it the AdvS objects are proxies"),\[1\] which fits the "opaque" part. The class name is yours alone [Inferred]. |
| Title block = `HYPERSTEELPAGEHEADER` with token attributes | **Confirmed** | Autodesk help: re-create the block "using the same name: HYPERSTEELPAGEHEADER";\[4\] attribute tags must come "from the list of recognized tokens." https://help.autodesk.com/cloudhelp/2025/ESP/AdvSteel-Drawings/files/GUID-00036924-E652-4D4A-B121-B0CE96859CE5.htm [Verified: official].\[5\] The token list "is hard-coded and there is no possibility to add the custom tokens." https://knowledge.autodesk.com/support/advance-steel/learn-explore/caas/sfdcarticles/sfdcarticles/Advance-Steel-List-of-tokens-and-attributes-available-for-page-header-title-block.html [Verified: official]\[6\] |
| Move view = `_AstM4DetViewTransform` | **Confirmed** | "Command line: _AstM4DetViewTransform… Select the view to move. Select the reference point for the view (P1). Select the second point (P2)." Also: "The view position does not change on drawing update after the model modification."\[7\] https://knowledge.autodesk.com/support/advance-steel/learn-explore/caas/CloudHelp/cloudhelp/2021/ENU/AdvSteel-Drawings/files/GUID-27F8BF44-A363-4F51-94CB-27F3D284D8B7-htm.html [Verified: official] |
| Scale/clipping only via `_AstM4DetChange` dialog | **Confirmed (no counter-evidence found)** | See Q1 below. |
| AS does not run in AcCoreConsole | **Plausible, not verified** | No source either way. `AstAtomicDetView` depends on `asobjectscad.dbx` plus SQL LocalDB, and the AS commands are ARX/UI commands, so a headless AcCoreConsole run is very unlikely to load them [Inferred]. Test it anyway (Spike 1). |
| EMV-MCP-AS is model-only | **Partly verified** | The repo exists. Its add-in "starts the local IPC server on http://127.0.0.1:5055" and exposes tools for "model, read, diagnose, script" through a Python FastMCP server.\[8\] https://github.com/eslinmoises/EMV-MCP-AS [Community]. I did not check which AS versions it targets or whether it has drawing tools. The listed tool categories suggest model-only [Inferred]. I did not review teknovizier/tekla_mcp_server. |

## Details

### Q1. Changing the scale of an existing detail view

**Finding: no dialog-free route is documented or reported anywhere I searched.**

- The official route is `_AstM4DetChange`: "This tool allows you to make additional changes to the detail parameters, such as scale, clipping or title text… In the Edit detail window, in the left upper side, select the view."\[9\] The text is the same in the 2018–2023 help.\[9\]\[10\] https://knowledge.autodesk.com/support/advance-steel/learn-explore/caas/CloudHelp/cloudhelp/2023/ENU/AdvSteel-Drawings/files/GUID-7CE73442-F35F-4A80-AB7C-53312C6C3C43-htm.html [Verified: official]
- Double-clicking the green view frame opens the same properties dialog, not a separate Properties-palette route. John Bennett: "if you double click it should open the properties and you should be able to change the scale of the view under view properties". The original poster then reported: "I lost all information that I previously had on the detail. If I change back to the original scale it also doesn't return." The workaround offered (alex.sample) was to delete the project's `Databases` folder contents and force-update, or regenerate the drawing at the right scale.\[11\] https://forums.autodesk.com/t5/advance-steel-forum/having-difficulty-changing-scale-of-view-in-shop-drawing/td-p/10738410 [Community]
- A scale change needs the model. Dumitru Berteanu (Autodesk staff) wrote: "The change scale operation requires Advance Steel to perform a quick update, so the drawing needs to be registered to a model… If the drawing was moved away from its default folder structure in the project, then the quick update can't happen and the scale will not be changed (or any other operation that need an update, like create Cut view, apply / disable hidden lines, clipping etc.)"\[12\] https://forums.autodesk.com/t5/advance-steel-forum/cant-change-the-view-scale/td-p/7581208 [Verified: Autodesk staff]
- Scale is set upstream in the drawing style or drawing process. Sebastian Eiche: "What you are looking for is the process manager… You could use alternative scales."\[13\] https://forums.autodesk.com/t5/advance-steel-forum/changing-the-scale/td-p/11646251 [Community, snippet only]. I found no source confirming that changing the style and then running `_AstM4CommDetUpdateDetail` re-scales a view that already exists.
- For manual cut and callout views, the scale is a session-level setting reached with the `S` key. After you choose a view request, it "is saved in the local memory and reused for the manual cut view command until you exit Advance Steel, similarly to the scale setting."\[14\] https://knowledge.autodesk.com/support/advance-steel/learn-explore/caas/CloudHelp/cloudhelp/2021/ENU/AdvSteel-Drawings/files/GUID-89D0B54C-EBAE-4468-A7A4-8B7FB091A893-htm.html [Verified: official]. So a *new* view can be created at a chosen scale with a single dialog pass per session.
- **Remaining options, in order of promise [Inferred]:**
  1. **UI automation of the Edit detail dialog.** Use Win32/UIA from your in-process plugin: send `_AstM4DetChange`, pick the frame, find the scale combo in the dialog, set it, press OK. This is brittle but workable for a single-machine overnight run.
  2. **Regenerate instead of rescale.** Delete the drawing and recreate it through a process or style whose scale was picked from a rule, for example "largest standard scale that fits". This is cleaner, but manual edits are lost.
  3. **LISP `entget`/`entmod` on `AstAtomicDetView`.** Nothing reported. Custom-object data usually lives in a private DWG filer rather than in DXF groups, so expect `entget` to return only the common groups [Inferred]. Spike 3 checks this.

### Q2. Command names for drawing-side tools

The table below comes from the Advance Steel User's Guide command appendix (Graitec-era, still valid in later versions).\[15\]\[16\] Commands marked ✓ also appear in current Autodesk help or forum posts. Sources: https://www.yumpu.com/en/document/view/27952935/advance-steel-users-guide-graitec-info/177 and https://manualzz.com/doc/o/p1a2k/advance-steel-user-s-guide-appendix [Verified: official doc, older version via third-party mirror]

| Function | Global command |
|---|---|
| Change detail (scale/clipping/title) | `_AstM4DetChange` ✓ |\[17\]
| Move view | `_AstM4DetViewTransform` ✓ |\[17\]
| Delete view | `_AstM4DetViewDelete` |\[17\]
| Update detail | `_AstM4CommDetUpdateDetail` ✓ |\[17\]
| Rearrange all details / views in detail | `_AstM4CommDetArrangeDetails` / `_AstM4CommDetArrangeViews` |\[17\]
| Create view / section (manual cut) | `_AstM4DetManualCut` ✓ |\[17\]
| Detail explode settings | `_AstM4CommExplodeDetail` |\[17\]
| Check details status | `_AstDetCheck` |\[17\]
| Drawing Style Manager (old / new) | `_AstM5DetailStylesExplorer` / `_AstM5DetailStylesManagerNew` |\[17\]
| Drawing Process Manager | `_AstM5DetailProcessExplorer` |\[17\]
| Generic call of a drawing style / process | `_AstM4CommDetailing` / `_AstM4CommDetailingProc` |\[17\]
| Linear horizontal / vertical / sloped dimension | `_AstM4CommDetAddHorLinearDim` / `_AstM4CommDetAddVertLinearDim` / `_AstM4CommDetAddSlopeLinearDim` |\[17\]
| General dimension (chain) | `_AstM4CommDetAddDim` |\[17\]
| Angle / arc / radial dimension | `_AstM4CommDetAddDimAngle` / `_AstM4CommDetAddDimArc` / `_AstM4CommDetAddDimRadial` |\[17\]
| Slope symbol | `_AstM4CommDetAddDimSlope` |\[17\]
| Level (elevation) symbol | `_AstM4CommDetCrHeightKote` |\[17\]
| Add / delete dimension point | `_AstM4CommDetModDimAdd` / `_AstM4CommDetModDimRemove` |\[17\]
| Change dimension / text style | `_AstM4CommDetModDimStyle` / `_AstM4CommDetModTextStyle` |\[17\]
| Label / user label | `_AstM4CommDetInsertAnno` ✓ / `_AstM4CommDetAddAnnotation` |\[17\]
| Weld symbol | `_AstM4CommDetCrWeldSeam` |\[17\]
| Update lists (BOM) / insert BOM | `_AstM4CommDetUpdateBOM` / `_AstM4CommDetInsertBom` |\[17\]
| Toggle green frames | `_AstM4CommDetSwitchHighlightState` |\[17\]
| Revision mark / revision table | `_AstM4CommDetailRevision` / `_AstM4CommDetRevisionTable` |\[17\]

- **Label.** `_AstM4CommDetInsertAnno` has a command-line option ("Type C for Change"). The change is saved per object type: "Scroll down to Plates (or beams), Scroll to SP, and change to Name… settings should be saved." (Ilko Dimitrov)\[18\] https://forums.autodesk.com/t5/advance-steel-forum/change-default-setting-for-label/td-p/9283655 [Community]. The same command is also used for manual level labels.\[19\] https://graitec.microsoftcrmportals.com/en-gb/public/7539275b-41a1-e711-8118-e0071b65be81/forum-posts?page=1 [Community/reseller]
- **Not found:** a dedicated "running/ordinate dimension" command in 2022 (running style is likely a property of the dimension request or style [Inferred]), and the "Update Page Header" command string. Look for both in `AdvanceSteel*.cuix` (Spike 2).
- **Scriptability.** No source says whether these commands accept points from `(command ...)` or `Editor.CommandAsync`. They prompt for entity picks and points on the command line (for example Move view asks for P1/P2), so feeding them `(entity . point)` pick pairs is the natural test [Inferred]. The risk is commands that check pick context, such as "pick on a detail view" or snaps on the "Preferred for Manual Dimensions" point. Autodesk recommends that snap for manual dimensions.\[20\] https://help.autodesk.com/view/ADSTPR/2022/ENU/?guid=GUID-D462912E-5259-476F-9EC8-B27A361B7A0E [Verified: official]

### Q3. What survives "Update drawing" in AS 2022

| Item | Survives update? | Source |
|---|---|---|
| Deleted automatic dimension | **Yes.** "You can safely delete automatic dimensions and no setting will affect the behavior - a deleted automatic dimension will not reappear." Exception: when a style has several dimension requests for the same object,\[21\] "deleting an automatic dimension triggers the other to reappear". The fix is to "perform the delete action multiple times and update"; "this type of dimension combination will have 2 or 3 layers." A deleted dimension also comes back if the model object is recreated.\[22\] | AS 2022 help, "Modify Automatic Dimensions on Drawings" https://knowledge.autodesk.com/support/advance-steel/learn-explore/caas/CloudHelp/cloudhelp/2022/ENU/AdvSteel-Drawings/files/GUID-40D8C07C-47A9-4876-BAB9-5DF4187521F5-htm.html [Verified: official] |
| Deleted dimension point | **Yes.** "Any point you remove from an existing automatic or manual dimension, using this command, remains deleted during update," unless that "contradicts the respective scenario requirements."\[21\] | Same page [Verified: official] |
| Moved or edited automatic dimensions | **Yes, by design.** Manual edits keep the dimension's "logic", and added points behave like manual points.\[21\] Some dimension and label moves after model updates can be controlled in Management Tools.\[23\] | Same page; https://www.autodesk.com/support/technical/article/caas/sfdcarticles/sfdcarticles/How-to-prevent-moving-the-labels-and-dimensions-on-detail-after-update-of-model-in-Advance-Steel.html [Verified: official, snippet] |
| Manual AS dimensions | **Yes, but check the setting.** The Management Tools default "Manual dimension point behavior on update" offers "Erase all invalid dimension point", a keep option, and "Erase all invalid points no longer on objects". It "will apply to any existing drawings, during update."\[24\] A community tip: set "Don't erase any manual dimension point".\[25\] | https://knowledge.autodesk.com/support/advance-steel/learn-explore/caas/CloudHelp/cloudhelp/2019/ENU/AdvSteel-Drawings/files/GUID-86265F98-594E-4586-88A1-8DD8E2B83D9E-htm.html [Verified: official]; https://forums.autodesk.com/t5/advance-steel-forum/dimensions-disappearing-after-a-drawing-update/td-p/8159145 [Community] |
| Moved views | **Yes.** "The view position does not change on drawing update after the model modification." | Move View help, above [Verified: official] |
| Changed view scale | **Stored on the view**, since scale changes are applied through a quick update. Community reports of lost annotation after a scale change are a separate risk.\[11\] | Q1 sources |
| Plain AutoCAD dimensions | **Kept but dead.** They "will never update or follow an Advance Steel detail view", and their scale "needs to be inputted manually for each dimension."\[26\] A community user adds: "Sometimes after the drawing is updated, the dimensions with either get deleted or moved."\[27\] | https://help.autodesk.com/view/ADSTPR/2022/ENU/?guid=GUID-D462912E-5259-476F-9EC8-B27A361B7A0E [Verified: official]; https://forums.autodesk.com/t5/advance-steel-forum/edit-detail-drawing-questions/td-p/9224423 [Community] |
| Plain AutoCAD lines, text, MLeaders | **Undocumented.** Probably left in place but not tied to the view [Inferred]. | Test with Spike 5 |
| Revision clouds | **Lost.** "Any revision clouds in the detail will be lost."\[28\] | https://help.autodesk.com/cloudhelp/2019/ENU/AdvSteel-Drawings/files/GUID-58840661-450B-492A-B929-ED4ACF006A78.htm [Verified: official] |
| Title-block token attributes edited by hand | **Overwritten by "Update Page Header".** "If I manually change the date in my drawing and then use 'Update Page Header,' it will undo my change." Relayed from Graitec: the date "will always be the date the drawing was created."\[29\] In AS 2023 one user saw the tool lose "all data except for DRAWING_NO and REVISION_INDEX".\[30\] | https://forums.autodesk.com/t5/advance-steel-forum/update-title-block-date/td-p/13356614; https://forums.autodesk.com/t5/advance-steel-forum/page-header-is-cleared-after-using-the-update-page-header-tool/td-p/11411610 [Community] |
| Non-token attributes / AutoCAD objects on the prototype | Non-token attributes are "driven manually by you typing it in".\[31\] On a prototype change, "it takes only frame and header blocks."\[32\] | https://forums.autodesk.com/t5/advance-steel-forum/need-help/td-p/10584685 [Community, snippet]; https://www.autodesk.com/support/technical/article/caas/sfdcarticles/sfdcarticles/Changing-the-prototype-does-not-show-the-objects-not-included-to-titleblock-of-drawing-in-Advance-Steel.html [Verified: official, snippet] |
| Cut-view annotations after connection updates | Kept since AS 2020: "updating connections in the model will no longer force the automatic cut views to reset during drawing update."\[33\] | AS 2020 What's New [Verified: official, snippet] |

**Takeaway [Inferred]:** anything the AI adds should be native Advance Steel content: manual AS dimensions, labels, dimension-point edits and view moves. The title block should hold only project-driven tokens plus separate non-token attributes that the AI owns. Treat plain AutoCAD annotation as throwaway until Spike 5 shows otherwise.

A separate warning: several update-integrity problems come from the project's `Databases` folder, including missing `.DB` files on Vault check-in and stale objects. Autodesk has a support article on disappearing views and dimensions.\[34\]\[35\] https://www.autodesk.com/support/technical/article/caas/sfdcarticles/sfdcarticles/Disappearing-views-and-dimension-when-updating-drawings-in-Advance-Steel-projects-on-Vault.html [Verified: official]. Your overnight runner must never leave that folder half-written.

### Q4. Drawing-style configuration as the main lever

- **Where styles live.** `AstorDetails.mdf` "contains all the drawing styles and processes available with the Advance Steel installation" (the Advance category). User styles are in `AstorAddin` under the user/country folder. `AstorDetailsBase` holds "generic detailing settings, used by the Drawing Style Manager."\[36\]\[37\]\[38\] Sources: https://www.slideshare.net/slideshow/advance-steel-implementationguide/192868922 and https://microsolresources.com/wp-content/uploads/2016/07/Advance-Steel-2017-Implementation-Guide.pdf [Verified: official doc via reseller/mirror]; https://forums.autodesk.com/t5/advance-steel-forum/how-to-restore-drawing-styles/td-p/8239641 [Community]
- **Format and tooling.** These are SQL Server MDF databases on LocalDB 2014 or later: "For more complex editing operations, it is recommended you use SQL Server Management Studio."\[39\] https://knowledge.autodesk.com/support/advance-steel/getting-started/caas/CloudHelp/cloudhelp/2021/ENU/AdvSteel-GetStarted/files/GUID-F0DE8220-20D8-479C-BDF3-D0E525F79C70-htm.html [Verified: official]. The Drawing Style Manager, Process Manager, Insert label settings and Detail explode settings all write to `AstorAddin`.\[40\] https://knowledge.autodesk.com/support/advance-steel/getting-started/caas/CloudHelp/cloudhelp/2022/ENU/AdvSteel-GetStarted/files/GUID-7B6D216E-C6B5-4F6F-A2E0-DDFA5E72C014-htm.html [Verified: official]. Delete the `*.LDF` files when swapping databases (Bennett, above).\[36\]
- **What styles control.** View requests (object presentation, "labeling rules, line type, and dimensions"),\[14\] "Necessary views" logic,\[41\] and the "Used for Cuts and Callouts" flag.\[42\] Sub-styles are shared, so editing one changes every style that uses it: "modifying a 'sub-style' modifies all the…"\[43\]\[44\] https://www.graitec.info/Common/Downloads/Documentation/AS-DSM%20guide-6.1-EN-051028.pdf [Verified: official, old]; https://www.autodesk.com/support/technical/article/caas/sfdcarticles/sfdcarticles/Not-all-views-from-drawing-style-definition-are-drawn-as-standard-in-Advance-Steel-detail.html [Verified: official]. Viewport size and depth for single-part and assembly views "are calculated automatically" and are greyed out.\[45\] https://www.autodesk.com/support/technical/article/caas/sfdcarticles/sfdcarticles/The-Viewport-settings-are-grayed-out-and-cannot-be-modified-in-Advance-Steel.html [Verified: official]
- **Limits.** Styles cannot dimension non-orthogonal plate edges, even with the "Sp Plate (sloped dimension)" strategy (george1985, thread above) [Community].\[1\] Title-block tokens are hard-coded.
- **Is SQL editing a practical route for an AI? [Inferred]** Yes for *reading*: an AI can diff your styles against company rules. It is only cautiously yes for *writing*. The schema is undocumented, the style tables are relational (style → view requests → dimension/label requests → sub-styles), and a wrong foreign key fails silently. Autodesk warns that the Drawing Style Manager "will erroneously show the addition having taken place" when a write actually failed (SQL Server Support page above).\[40\] The safe pattern: the AI proposes a change, a human applies it in the Drawing Style Manager, and the result is exported to MDB and version-controlled. Direct SQL writes are only for bulk parameter changes you have mapped in the Spike 7 schema dump.

### Q5. Third-party add-ons

- **GRAITEC PowerPack for Advance Steel (now part of GRAITEC STEEL 2027).** On the drawing side it offers "Explode details: explodes a drawing and to transforms the Advance Steel dimensions into AutoCAD dimensions" plus batch explode drawings.\[46\] https://www.graitec.com/Help/PowerPack_for_Advance_Steel/En/Tools_panel.htm [Verified: vendor doc]. Also drawing and BOM templates and a camera browser. Its 2027 "GRAITEC Assistant" covers "natural-language Q&A… AI-driven model filtering… and an intelligent BOM generator".\[47\] Drawing cleanup is not mentioned. https://www.graitec.com/Help/PowerPack_for_Advance_Steel/En/What_is_New.htm [Verified: vendor doc]. Graitec North America's blog "What's New In Graitec STEEL – For Advance Steel" (11 Aug 2026) says "Existing PowerPack users move across on a single installer, and the features you rely on come with you" and that "Graitec STEEL has been updated to stay fully compatible with .Net 10". It runs only on AS 2026/2027, so the current release does not support your AS 2022 install.
- **CBT Christian Blei Tools** (christianblei.de) is a long-standing AS API developer and a likely consultancy for custom work [Community].\[1\]
- I found no product that automates dimension or label cleanup on Advance Steel sheets. I did not verify Kiwa, CADLearning or "AS Utilities".

### Q6. Reading detail-view geometry

- **Native explode keeps AS objects.** "Exploding Advance Steel details does not remove Advance Steel objects from them. The exploded details end up having Advance Steel objects inside them."\[48\] https://knowledge.autodesk.com/support/autocad/troubleshooting/caas/sfdcarticles/sfdcarticles/Detail-drawings-exploded-from-current-Advance-Steel-version-cannot-be-opened-in-older-AutoCAD-versions.html [Verified: official]. This partly refutes any plan built on explode alone.
- AS 2022 added batch explode in Document Manager "to explode Advance Steel drawings to pure AutoCAD drawings."\[49\] https://www.designconsulting.com.au/?p=81726 [Community/reseller]. Once exploded, "the drawing has no link to the model" (thread above) [Community].\[27\]
- **Recommended read path [Inferred]:** in memory, open a copy of the sheet with `Database.ReadDwgFile`, or `Entity.Explode` the `AstAtomicDetView` into a `DBObjectCollection` without appending it. Collect lines, arcs, circles (holes) and exploded dimension values, then map them back to view frames by their extents. Whether `Entity.Explode` works on these custom entities in-process is the most important test (Spike 4). No source gives a way to map model handles to specific views (`DerivedObjectDetailInfo`) on the drawing side.

### Q7. Upgrading, and Autodesk's AI/MCP offering

- **AS 2022 drawing additions:** leader lines attached at mid-text with an optional landing line, batch explode, "Enhanced dimension holes configurations for cleaner drawings (assemblies with attached plates or gussets)", and more assembly styles such as trusses.\[49\] https://www.designconsulting.com.au/?p=81726 [Community/reseller]. You already have all of these.
- **2023–2027:** after March 2023 there are no new features by policy.\[50\] Users described the 2024 release notes as "wow so much new functions... (a bit salty i know)".\[51\] https://forums.autodesk.com/t5/advance-steel-forum/advance-steel-2024-release-note/td-p/11876857 [Community]. The official What's New pages exist at https://help.autodesk.com/view/ADSTPR/2025/ENU/?guid=GUID-8B621888-602B-425A-8166-9F104EA91112, but I did not review them in detail. **Nothing found justifies upgrading for drawing automation.** One caveat: from AutoCAD 2025 onward the platform is .NET 8, which affects how your plugin is built, not what it can do [Inferred].
- **Autodesk MCP servers:** the Autodesk MCP hub lists a Product Help MCP (remote), Fusion MCP (local) and Fusion Data MCP (remote). A search snippet also mentions a local server "that runs inside AutoCAD or Civil 3D… to query, analyze, navigate, and modify objects".\[52\] https://help.autodesk.com/view/ADSKMCP/ENU/ [Verified: official]. Revit 2027 ships Autodesk Assistant with MCP, plus separately installed Read, "Trusted-Write" and "Experimental" MCP servers.\[53\]\[54\] https://www.autodesk.com/blogs/aec/2026/06/17/revit-public-mcp-server/ [Verified: official]; https://forums.autodesk.com/t5/revit-architecture-forum/mcp-server-for-revit-tech-preview-launch/td-p/14262009 [Verified: Autodesk staff]. **None of them mention Advance Steel.** An AutoCAD MCP server would see AS objects as proxies or custom entities, exactly as your plugin does [Inferred].

### Q8. Prior art for AI shop-drawing cleanup

- **Tekla AI Cloud Fabrication Drawings** (preview since 2025, improved in 2026). It creates drawings "based on finalized fabrication drawings from previous projects… AI is used when classifying drawings into libraries… and when looking for the best matching drawing." It then either **clones** the best-matching drawing template or creates the drawing from that drawing's settings. It offers "top three AI-based matches" with a human-in-the-loop step, and has improved "placement of annotations, dimensions, and views, avoiding issues such as stiffeners being dimensioned as connection plates." Limits: "Single-part drawings cannot be created using the AI", and uploaded drawings are not updated automatically if the model changes.\[55\]\[56\]\[57\] https://support.tekla.com/doc/tekla-structures/2026/dra_create_fabrication_drawings_cloud_ai; https://support.tekla.com/doc/tekla-structures/2026/rel_dra_new_in_ai_cloud_fabrication_drawings; https://support.tekla.com/doc/tekla-structures/2026/dra_ai_cloud_fabrication_drawings_faq [Verified: official vendor doc]. **The underlying method is retrieval of similar past assemblies followed by template cloning. It is not generative placement.**
- **DraftAid** generates 2D drawings from Inventor or STEP models of extrusions, sheet metal and machined parts. Roopinder Tara (engineering.com, "Hey, AI. Make my drawings.", 2 Apr 2024) wrote: "I estimate the dimensioned drawing to be 95% complete." Co-founder Abdullah Elqabbany gave a lower figure: DraftAid "will produce a drawing that is '80 to 90% complete'." Assembly and weldment support is a beta from June 2026. https://www.engineering.com/hey-ai-make-my-drawings/; https://www.thefabricator.com/thefabricator/article/cadcamsoftware/ai-powered-tool-automates-2d-drawing-creation-from-3d-cad-models [Community/press]. Its algorithm is not disclosed.
- **SDS2, Parabuild and academic work** on dimension placement, label placement and layout optimisation were not researched here (search budget). The classic approaches are rule-based dimension schemes, label placement framed as combinatorial optimisation (greedy, simulated annealing, ILP), and bin-packing for view layout [Inferred, background knowledge, unverified].
- **What fits your case [Inferred]:** Tekla's result backs up your plan. Retrieve the most similar *approved* past drawing for the same assembly type and "transfer the edits" (scale, view positions, deleted-dimension set, added-dimension set), expressed as native AS commands. Your engineers' morning fixes become before/after pairs for the next retrieval.

## Recommendations

1. **Architecture.** Keep the in-process NETLOAD plugin plus the named-pipe MCP server. Run every operation as `Editor.Command` or `CommandAsync` on the document context, inside its own undo group, and follow each one with a PlotEngine PNG check. Add a UI-automation module only for `_AstM4DetChange`.
2. **Split the work.** About 70% should be style and process tuning done by humans with AI proposals. That includes scale rules via alternative scales in processes, hole-dimension requests and label strategies. The remaining ~30% is overnight sheet edits: view moves, dimension point add and remove, manual dimensions, labels, and non-token attributes [Inferred ratio].
3. **Set `Manual dimension point behavior on update`** deliberately before any AI-added dimensions reach production.
4. **Keep title-block fields separate.** Tokens are filled by AS. AI-filled fields should be non-token attributes, and nobody should run "Update Page Header" on them without testing.
5. **Do not upgrade** for drawing reasons.

## Capability → Best Route in AS 2022 → Confidence

| Capability | Best available route in AS 2022 | Confidence |
|---|---|---|
| Move view | `_AstM4DetViewTransform` with frame pick + P1/P2 | High (documented; scripting untested) |
| Change view scale | UI-automate `_AstM4DetChange`, or regenerate from a rule-picked style/process | Low–Medium |
| Change clipping | `_AstM4DetChange` dialog (UIA) | Low |
| Rearrange views | `_AstM4CommDetArrangeViews` / `_AstM4CommDetArrangeDetails`, then targeted moves | Medium |
| Delete automatic dimension | AutoCAD `ERASE` of the AS dimension entity; repeat for combined requests | High (persistence documented) |
| Add/delete dimension point | `_AstM4CommDetModDimAdd` / `_AstM4CommDetModDimRemove` | Medium (pick-driven) |
| New linear/chain/angle dimensions | `_AstM4CommDetAddHorLinearDim`/`...Vert...`/`...Slope...`/`_AstM4CommDetAddDim`/`_AstM4CommDetAddDimAngle` with scripted points | Medium–Low until Spike 2 |
| Elevation/level | `_AstM4CommDetCrHeightKote` | Medium–Low |
| Running/ordinate dimension | Style dimension request (no manual command found) | Low |
| Hole dimensions with Ø and count | Drawing-style hole dimension/label requests | Medium |
| Leader labels | `_AstM4CommDetInsertAnno` (C option for content) | Medium |
| Title block | AutoCAD `AttributeReference` edits on `HYPERSTEELPAGEHEADER` (non-token attributes) | High for non-token, Low for token attributes |
| Read view geometry | In-memory `Entity.Explode` on a DB copy, or batch-explode a copy | Medium–Low |
| Tune styles per company rules | Drawing Style Manager (human) + SSMS read-only diffs; SQL writes only for mapped fields | Medium |
| Headless batch | Not possible (AcCoreConsole); full AS session with an unattended scheduler | Medium |
| Visual verification | PlotEngine PNG | High |

## Spike Experiments to Run on AS 2022

1. **AcCoreConsole:** start `accoreconsole.exe` with `/product` and AS profile switches on a sheet, run `(entget (car (entsel)))`-equivalent via script, and check whether `AstAtomicDetView` loads or appears as a proxy.
2. **CUIx and command scriptability:** extract macro strings from `AdvanceSteel*.cuix` to find "Update Page Header" and any running-dimension command. For each dimension or label command, try `(command "_AstM4CommDetAddHorLinearDim" pt1 pt2 pt3)` and the `Editor.Command` equivalent with `(list ent pt)` picks. Log which prompts accept piped input.
3. **LISP/DXF introspection:** run `entget` (with `'("*")` xdata) on `AstAtomicDetView` and on an AS dimension. Use `dumpallproperties` / `getpropertyvalue` and `vlax-dump-object` to look for exposed scale or position properties.
4. **In-memory explode:** in C#, `Entity.Explode` an `AstAtomicDetView` and an AS dimension. Record the types returned and whether dimension text values and hole circles are recoverable.
5. **Update survival matrix:** place a plain LINE, MTEXT, MLEADER, AutoCAD DIM (inside and outside the frame), a manual AS dimension, a moved and a deleted automatic dimension, a moved view, a rescaled view and edited token and non-token attributes. Then change the model, run `_AstM4CommDetUpdateDetail` and a Document Manager update, and diff.
6. **Dialog automation:** drive `_AstM4DetChange` with UIAutomation. Find the scale control, set it, press OK, and measure how reliable this is across 50 sheets.
7. **Schema dump:** attach `AstorAddin.mdf` / `AstorDetails.mdf` in SSMS on a copy, script the schema, and trace which rows change when you edit one hole-dimension request in the Drawing Style Manager.
8. **Style-change re-scale:** change a view's style or process scale and check whether `_AstM4CommDetUpdateDetail` re-scales an existing drawing.

## Open Questions

- Is there any non-dialog way to set the scale of an existing view: a hidden `-`-prefixed command, a system variable, or a COM property? None was found.
- Is `ImpEntity` really the AS dimension class, and does it expose measured values without exploding?
- Do plain AutoCAD lines, text and MLeaders survive Update detail and Document Manager updates?
- Does a normal drawing update, as opposed to "Update Page Header", overwrite hand-edited token attributes?
- What is the exact command string for "Update Page Header", and is there a manual running/ordinate dimension command in 2022?
- Can drawing-side views be mapped to model handles (`DerivedObjectDetailInfo`) programmatically?
- Which AS versions does EMV-MCP-AS support, and does it do anything on the drawing side? teknovizier/tekla_mcp_server was not reviewed.
- What algorithms do SDS2, Parabuild and the academic literature on dimension, label and layout placement actually use? Not researched here.

## Sources

1. [Advance Steel API - access Layout elements](https://forums.autodesk.com/t5/advance-steel-forum/advance-steel-api-access-layout-elements/td-p/12547358)
2. [Generate 2D drawing with .NET API - Autodesk Community](https://forums.autodesk.com/t5/advance-steel-forum/generate-2d-drawing-with-net-api/td-p/11763769)
3. ['Exception in asobjectscad.dbx ARX command' when starting Advance Steel or an AutoCAD product](https://www.autodesk.com/support/technical/article/caas/sfdcarticles/sfdcarticles/Exception-in-asobjectscad-dbx-ARX-command-when-starting-Advance-Steel.html)
4. [Drawing Prototypes | Advance Steel 2019 | Autodesk Knowledge Network](https://knowledge.autodesk.com/support/advance-steel/learn-explore/caas/CloudHelp/cloudhelp/2019/ENU/AdvSteel-Drawings/files/GUID-00036924-E652-4D4A-B121-B0CE96859CE5-htm.html)
5. [Drawing Prototypes](https://help.autodesk.com/cloudhelp/2025/ESP/AdvSteel-Drawings/files/GUID-00036924-E652-4D4A-B121-B0CE96859CE5.htm)
6. [How to add the custom attributes like length or weight to prototype page header in Advance Steel](https://knowledge.autodesk.com/support/advance-steel/learn-explore/caas/sfdcarticles/sfdcarticles/Advance-Steel-List-of-tokens-and-attributes-available-for-page-header-title-block.html)
7. [Move a View | Advance Steel 2021 | Autodesk Knowledge Network](https://knowledge.autodesk.com/support/advance-steel/learn-explore/caas/CloudHelp/cloudhelp/2021/ENU/AdvSteel-Drawings/files/GUID-27F8BF44-A363-4F51-94CB-27F3D284D8B7-htm.html)
8. [GitHub - eslinmoises/EMV-MCP-AS · GitHub](https://github.com/eslinmoises/EMV-MCP-AS)
9. [Change a Detail | Advance Steel 2023 | Autodesk Knowledge Network](https://knowledge.autodesk.com/support/advance-steel/learn-explore/caas/CloudHelp/cloudhelp/2023/ENU/AdvSteel-Drawings/files/GUID-7CE73442-F35F-4A80-AB7C-53312C6C3C43-htm.html)
10. [Change a Detail | Advance Steel 2018 | Autodesk Knowledge Network](https://knowledge.autodesk.com/support/advance-steel/learn-explore/caas/CloudHelp/cloudhelp/2018/ENU/AdvSteel-Drawings/files/GUID-7CE73442-F35F-4A80-AB7C-53312C6C3C43-htm.html)
11. [Having Difficulty Changing Scale of View in Shop Drawing](https://forums.autodesk.com/t5/advance-steel-forum/having-difficulty-changing-scale-of-view-in-shop-drawing/td-p/10738410)
12. [Cant change the view scale.](https://forums.autodesk.com/t5/advance-steel-forum/cant-change-the-view-scale/td-p/7581208)
13. [Solved: Changing the scale - Autodesk Community](https://forums.autodesk.com/t5/advance-steel-forum/changing-the-scale/td-p/11646251)
14. [Create a Cut View from the Detail | Advance Steel 2021 | Autodesk Knowledge Network](https://knowledge.autodesk.com/support/advance-steel/learn-explore/caas/CloudHelp/cloudhelp/2021/ENU/AdvSteel-Drawings/files/GUID-89D0B54C-EBAE-4468-A7A4-8B7FB091A893-htm.html)
15. [ADVANCE STEEL USER’S GU](https://www.yumpu.com/en/document/view/27952935/advance-steel-users-guide-graitec-info/177)
16. [Advance Steel User's Guide | Manualzz](https://manualzz.com/doc/o/p1a2k/advance-steel-user-s-guide-appendix)
17. [Advance Steel User's Guide Commands | PDF | Rectangle | Beam (Structure)](https://www.scribd.com/document/660512239/As-User-Guide-2015-Commands-1)
18. [Solved: Change default setting for label - Autodesk Community](https://forums.autodesk.com/t5/advance-steel-forum/change-default-setting-for-label/td-p/9283655)
19. [Public Profile · Graitec Advantage](https://graitec.microsoftcrmportals.com/en-gb/public/7539275b-41a1-e711-8118-e0071b65be81/forum-posts?page=1)
20. [Manual Dimensions | Advance Steel 2017 | Autodesk Knowledge Network](https://knowledge.autodesk.com/support/advance-steel/learn-explore/caas/CloudHelp/cloudhelp/2017/ENU/AdvSteel-UsrGuide/files/GUID-D462912E-5259-476F-9EC8-B27A361B7A0E-htm.html)
21. [Modify Automatic Dimensions on Drawings | Advance Steel 2022 | Autodesk Knowledge Network](https://knowledge.autodesk.com/support/advance-steel/learn-explore/caas/CloudHelp/cloudhelp/2022/ENU/AdvSteel-Drawings/files/GUID-40D8C07C-47A9-4876-BAB9-5DF4187521F5-htm.html)
22. [Modify Automatic Dimensions on Drawings](https://help.autodesk.com/cloudhelp/2017/ENU/AdvSteel-Drawings/files/GUID-40D8C07C-47A9-4876-BAB9-5DF4187521F5.htm)
23. [How to prevent moving the labels and dimensions on detail after model update in Advance Steel](https://www.autodesk.com/support/technical/article/caas/sfdcarticles/sfdcarticles/How-to-prevent-moving-the-labels-and-dimensions-on-detail-after-update-of-model-in-Advance-Steel.html)
24. [The Manual dimension point behavior on update default | Advance Steel 2019 | Autodesk Knowledge Network](https://knowledge.autodesk.com/support/advance-steel/learn-explore/caas/CloudHelp/cloudhelp/2019/ENU/AdvSteel-Drawings/files/GUID-86265F98-594E-4586-88A1-8DD8E2B83D9E-htm.html)
25. [Solved: Dimensions disappearing after a drawing update - Autodesk Community](https://forums.autodesk.com/t5/advance-steel-forum/dimensions-disappearing-after-a-drawing-update/td-p/8159145)
26. [Help | Manual Dimensions | Autodesk](https://help.autodesk.com/view/ADSTPR/2022/ENU/?guid=GUID-D462912E-5259-476F-9EC8-B27A361B7A0E)
27. [Edit detail drawing questions](https://forums.autodesk.com/t5/advance-steel-forum/edit-detail-drawing-questions/td-p/9224423)
28. [Update a Detail](https://help.autodesk.com/cloudhelp/2019/ENU/AdvSteel-Drawings/files/GUID-58840661-450B-492A-B929-ED4ACF006A78.htm)
29. [Update Title Block Date](https://forums.autodesk.com/t5/advance-steel-forum/update-title-block-date/td-p/13356614)
30. [Page header is cleared after using the Update Page Header tool](https://forums.autodesk.com/t5/advance-steel-forum/page-header-is-cleared-after-using-the-update-page-header-tool/td-p/11411610)
31. [Solved: Need help - Autodesk Community](https://forums.autodesk.com/t5/advance-steel-forum/need-help/td-p/10584685)
32. [Changing the prototype doesn't show the objects not included to titleblock of drawing in Advance Steel](https://www.autodesk.com/support/technical/article/caas/sfdcarticles/sfdcarticles/Changing-the-prototype-does-not-show-the-objects-not-included-to-titleblock-of-drawing-in-Advance-Steel.html)
33. [New in Advance Steel 2020 | Advance Steel 2020 | Autodesk Knowledge Network](https://knowledge.autodesk.com/support/advance-steel/learn-explore/caas/CloudHelp/cloudhelp/2020/ENU/AdvSteel-WhatsNew/files/GUID-D64995B8-B423-4CB7-8B9D-9C46F3C88081-htm.html)
34. [Detail drawing still displays old version of some objects after update in Advance Steel](https://www.autodesk.com/support/technical/article/caas/sfdcarticles/sfdcarticles/Detail-drawing-still-displays-old-version-of-some-objects-after-update-in-Advance-Steel.html)
35. [Disappearing views and dimension when updating drawings in Advance Steel projects on Vault](https://www.autodesk.com/support/technical/article/caas/sfdcarticles/sfdcarticles/Disappearing-views-and-dimension-when-updating-drawings-in-Advance-Steel-projects-on-Vault.html)
36. [How to restore drawing styles - Autodesk Community](https://forums.autodesk.com/t5/advance-steel-forum/how-to-restore-drawing-styles/td-p/8239641)
37. [Advance steel implementation\_guide | PDF](https://www.slideshare.net/slideshow/advance-steel-implementationguide/192868922)
38. [Advance Steel Implementation Guide](https://microsolresources.com/wp-content/uploads/2016/07/Advance-Steel-2017-Implementation-Guide.pdf)
39. [SQL Server Database Format for Advance Steel | Advance Steel 2021 | Autodesk Knowledge Network](https://knowledge.autodesk.com/support/advance-steel/getting-started/caas/CloudHelp/cloudhelp/2021/ENU/AdvSteel-GetStarted/files/GUID-F0DE8220-20D8-479C-BDF3-D0E525F79C70-htm.html)
40. [SQL Server Support for Advance Steel | Advance Steel 2022 | Autodesk Knowledge Network](https://knowledge.autodesk.com/support/advance-steel/getting-started/caas/CloudHelp/cloudhelp/2022/ENU/AdvSteel-GetStarted/files/GUID-7B6D216E-C6B5-4F6F-A2E0-DDFA5E72C014-htm.html)
41. [Not all views from drawing style definition are drawn as standard in Advance Steel detail](https://www.autodesk.com/support/technical/article/caas/sfdcarticles/sfdcarticles/Not-all-views-from-drawing-style-definition-are-drawn-as-standard-in-Advance-Steel-detail.html)
42. [Callout Views | Advance Steel 2019 | Autodesk Knowledge Network](https://knowledge.autodesk.com/support/advance-steel/learn-explore/caas/CloudHelp/cloudhelp/2019/ENU/AdvSteel-Drawings/files/GUID-3D942FF1-2C80-4EF8-B2C3-9E0F17274BDB-htm.html)
43. [Advance DRAWING STYLE MANAGER](http://www.graitec.info/Common/Downloads/Documentation/AS-DSM%20guide-6.1-EN-051028.pdf)
44. [Advance Steel Drawing Style Manager’s guide](http://www.graitec.info/Common/Downloads/Documentation/AS-DSM_guide-8.1-EN.pdf)
45. [The Viewport settings are grayed out and cannot be modified in Advance Steel](https://www.autodesk.com/support/technical/article/caas/sfdcarticles/sfdcarticles/The-Viewport-settings-are-grayed-out-and-cannot-be-modified-in-Advance-Steel.html)
46. [Tools category](https://www.graitec.com/Help/PowerPack_for_Advance_Steel/En/Tools_panel.htm)
47. [What's New](https://www.graitec.com/Help/PowerPack_for_Advance_Steel/En/What_is_New.htm)
48. [Detail drawings exploded from current Advance Steel version cannot be opened in older AutoCAD versions](https://knowledge.autodesk.com/support/autocad/troubleshooting/caas/sfdcarticles/sfdcarticles/Detail-drawings-exploded-from-current-Advance-Steel-version-cannot-be-opened-in-older-AutoCAD-versions.html)
49. [What's New In Advance Steel 2022 | Design Consulting](https://www.designconsulting.com.au/?p=81726)
50. [Advancing Capabilities for Steel Detailing and Design in BIM](https://www.autodesk.com/blogs/aec/2023/03/29/advancing-steel-detailing-and-design-at-autodesk/)
51. [Advance steel 2024 - release note - Autodesk Community](https://forums.autodesk.com/t5/advance-steel-forum/advance-steel-2024-release-note/td-p/11876857)
52. [Autodesk MCP Server Help | Autodesk](https://help.autodesk.com/view/ADSKMCP/ENU/)
53. [Introducing the Revit Public MCP Server: A Trusted Foundation for AI-Powered Workflows - AEC Tech Drop](https://www.autodesk.com/blogs/aec/2026/06/17/revit-public-mcp-server/)
54. [MCP Server for Revit - Tech Preview Launch! - Autodesk Community](https://forums.autodesk.com/t5/revit-architecture-forum/mcp-server-for-revit-tech-preview-launch/td-p/14262009)
55. [AI Cloud Fabrication drawings FAQ | Trimble User Assistance](https://support.tekla.com/doc/tekla-structures/2026/dra_ai_cloud_fabrication_drawings_faq)
56. [Create fabrication drawings using AI and cloud collections (Preview) | Trimble User Assistance](https://support.tekla.com/doc/tekla-structures/2026/dra_create_fabrication_drawings_cloud_ai)
57. [New in AI Cloud Fabrication drawings | Trimble User Assistance](https://support.tekla.com/doc/tekla-structures/2026/rel_dra_new_in_ai_cloud_fabrication_drawings)
