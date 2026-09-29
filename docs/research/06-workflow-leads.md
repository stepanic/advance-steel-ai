# 06 — Leads from the client's live workflow (call part 002)

As of 2026-09-29. Input: `docs/requirements/00-requirements.md` (anonymised from call notes) (the client's step-by-step cleanup of a stair-railing job). This document only covers what is **new** compared with `01`–`05`.
Tags: **[V]** official Autodesk help or Autodesk staff, **[C]** community or reseller, **[I]** inferred and not tested.

Research limits: forums.autodesk.com threads older than about 2019 are now archived and redirect to the forum index, so they were only read through search snippets (marked "snippet"). Newer threads and Autodesk support articles were read in full through the browser. The Graitec *Advance Steel User's Guide 7.1*, the *AS 2015 User's Guide* and the *Drawing Style Manager guide 8.1* PDFs were downloaded and grepped.

---

## TL;DR

1. **"Advanced view properties" is the AS context-menu entry "Advance View properties".** It is a modal Advance Steel dialog, not the AutoCAD Properties palette. It shows the *View dialog* with a **Properties** tab (Scale, Rotation, **Show hidden lines**, viewport, running-dimension switch) and a **Clip** tab (minimum length to cut, how much is kept on each side, and so on) [V]. It accepts **several selected views at once**: the AS 2021 release notes fix "unnecessary updates due to changes through Advance View properties after selecting multiple detail drawing elements" [V]. So multi-view editing existed before 2021; I did not find the version that introduced it. "Advance Properties" on a label opens the *Label and Symbols* dialog, whose **Leader Line tab → Show line** is the switch the client uses [V]. **No source says any of these values are exposed as OPM/COM properties.** No command name is documented for either context-menu entry, so the next step is a spike (§3.1).
2. **The most important new finding is that one dialog pass can scale and clip every view on a sheet.** Automating "Advance View properties" once, with every view preselected, is far cheaper and more robust than calling `_AstM4DetChange` once per view (the fallback in `05` §4.3). It still needs a quick update, so the drawing must stay registered in its original folder [V, `04` Q1].
3. **Most of the client's cleanup can be removed upstream in user drawing styles and processes. All the settings below exist in the Drawing Style Manager (DSM) [V]:**
   - Fixed view scale: set "Scale in process = **Fixed**", or give the process alternative scales.
   - Clipping strategy: "Minimum length to cut" and "How much should be kept on each side". The client's "40–40" is almost certainly these two values [I].
   - Leaving out dimension requests and weld symbols per view.
   - Object-presentation rules that filter model objects by **type + model role + additional filter**. This is the route to "frame only" and "infill only" views.
   - Dimension types **"Linear, relative and absolute"** and **"… from end"**, plus running-dimension ordinate references. This is the client's absolute + chain dimensioning.
   - Label variables **`%MainPartQuantity`** / `%Quantity` for the "3X" notation.
4. **Batch Plot blanks are very likely a plot-area problem [I].** The Document Manager batch plot uses each drawing's saved page setup, inherited from the prototype [C]. If that setup plots *Extents*, a view frame outside the sheet enlarges the extents, and the sheet comes out shrunk, shifted or empty. Switching the prototype to *Layout* or *Window* fixes the cause. A frame-inside-sheet checker catches the symptom, and it is a deterministic first MCP tool.
5. **Commands and routes found:**
   - Process by number: `_AstM4CommDetailingProc User <n>` [V].
   - Process suite: `AstProcessesSuite` [V/C].
   - Update page header: `_AstM4CommDetUpdateProjectData` [V snippet].
   - "Define assembly detailing direction" is a Tools-palette button that takes the current UCS plus a pick of the assembly [V]. Its command name is undocumented.
   - The Document Manager's *Force update* and *Open drawing* have no documented command-line form. The plugin can instead open the DWGs itself in a fixed order, which also fixes the "random order" annoyance, and update each one with `_AstM4CommDetUpdateDetail`.

---

## 1. Step → best automation route

Routes: **Style** = user drawing style or process configuration (done once by a human in DSM or the Process Manager). **Prop** = the AS property dialog ("Advance View properties" / "Advance Properties"), driven by UI Automation until an API is found. **AS cmd** = scripted `_AstM4…` command. **ACAD API** = AutoCAD .NET (select, erase, attributes, extents). **UIA** = UI Automation of an AS dialog. **Human** = stays manual.

| # | The client's manual step (requirements §5) | Best route | Fallback | Confidence |
|---|---|---|---|---|
| 1.1 | Sheet 1: delete the front, side and top views, keep only 3D | **Style**: a user copy of the assembly style that contains only the 3D view request | ACAD/AS cmd: `_AstM4DetViewDelete` on the unwanted views | High (style) [V feature, I config] |
| 1.2 | Double-click the big frame → change scale, then position it on the paper | **Style**: fixed scale for that view. Position with **AS cmd** `_AstM4DetViewTransform` | **Prop/UIA**: Advance View properties → Scale | High for position, medium for scale |
| 1.3 | AS labels only the first BOM item; the client wants every part labelled | **Style**: object-presentation rules for attached parts with a labeling strategy (all parts, not only the main part) | AS cmd `_AstM4CommDetInsertAnno` per part | Medium [I] |
| 1.4 | Show hidden lines on/off per case | **Prop**: View dialog → *Show hidden lines* (multi-select). Needs a quick update | Style default. "Switch off hidden lines" in *Modify settings* at creation | Medium; which cases need it stays **Human/AI judgement** |
| 1.5 | Insert label per element; repeated elements get one label with "3X" | **Style**: label content `%MainPartQuantity`x`%PosNum` (or `%Quantity`). Deduplicate by **ACAD API** erase of extra labels with the same mark | AS cmd `_AstM4CommDetInsertAnno` + option `C` (content definition) | Medium: the tokens exist [V], exact semantics untested [I] |
| 1.6 | Overview (§2.1): Select Similar labels → Advance Properties → Leader Line → Show line; space the labels by hand | **Style / Management Tools**: labeling-strategy leader definition and Drawing-Labeling defaults | **UIA** of the Label dialog (multi-select). AS 2020+ *Arrange labels* / *Align labels to reference* for spacing | Medium |
| 2.1 | Sheet 2: delete the unneeded view; scale 1:10 | **Style**: fixed 1:10 and only the needed view requests | Prop/UIA scale; `_AstM4DetViewDelete` | High (style) |
| 2.2 | Delete all automatic dimensions and weld symbols | **Style**: a view without dimension requests, weld symbols switched off (object presentation / *Modify settings* "Switch off weld symbols") | **ACAD API** erase by class/layer. Deleted automatic dimensions stay deleted [V, `04` Q3]. Weld symbols have their own layers since AS 2016 [C snippet] | High |
| 2.3 | Frame only: delete the infill (Select Similar → Delete) | **Style**: model-object filter by role, so the frame view shows frame roles only | **ACAD API** erase by class/layer/handle list, from `DerivedObjectDetailInfo` + geometry | Medium: needs consistent roles in the model [I] |
| 2.4 | Position the frame on the paper → Update detail so the green frames move back | **AS cmd** `_AstM4DetViewTransform`, then `_AstM4CommDetUpdateDetail` | — | High [V] |
| 2.5 | Dimension the frame: absolute + chain from the open corner, top and other side, zero on the right | **Style**: dimension request type *Linear, relative and absolute (from end)* + running-dimension ordinate reference (left/right of main/total assembly) | **AS cmd** `_AstM4CommDetAddDim` / `…HorLinearDim` with explicit points; the AI picks the corner | Medium for the style; choosing the "open corner" is **AI judgement** |
| 3.1 | Sheet 3: infill only, 1:10 | **Style**: the inverse role filter of 2.3 | as 2.3 | Medium |
| 4.1 | Rename the sheet title to "Sklop 1" | **ACAD API**: write a **non-token** title-block attribute. No token holds the assembly name (§2.5) | Process *filename selector* / detail title | Medium [I] |
| 4.2 | Select all part views → Advance View properties → 1:2, clipping 40–40 | **Style**: single-part style with fixed 1:2 and a clipping strategy (40/40) | **Prop/UIA**: one multi-select dialog pass per sheet | High (style) / medium (UIA) |
| 4.3 | Arrange the views on the paper; delete unneeded dimensions | **AS cmd** `_AstM4DetViewTransform` / `_AstM4CommDetArrangeViews`; **ACAD API** erase | — | Medium (layout quality is **AI judgement**) |
| 4.4 | Angle dimensions always "into the material", outside | **AS cmd** `_AstM4CommDetAddDimAngle` with picks chosen by the AI | Style angle requests (untested) | Low–medium |
| 5.1 | Green view frames must stay inside the paper or Batch Plot prints blank | **ACAD API** check (view extents vs `HYPERSTEELPAGEFRAME`) + prototype page setup = Layout/Window | Human prints by hand | High (check) [I] |
| 5.2 | Document Manager: Update required → Force update → Open drawings (random order) | **ACAD API** open the DWGs in sorted order + `_AstM4CommDetUpdateDetail` | **UIA** of the Document Manager | Medium |
| — | Drawing process #34 / #4, Define assembly detailing direction | **AS cmd** `_AstM4CommDetailingProc User <n>` | — | Direction/UCS choice stays **Human** (the client) |

---

## 2. Details

### 2.1 "Advance View properties" / "Advance Properties"

- **What it is.** AS drawings have AS context-menu entries. Right-clicking a part in a detail offers *"Advance detail properties"*. In the same thread, the inner green frame is identified as *Advance View properties* [C snippet, forum "Green frame around details lost"]. Advance Properties is AS's own dialog, distinct from AutoCAD *Properties* / `DDCHPROP` [V, AS User's Guide 7.1 and 2015: "right click on the object and pick Properties (not Advance Properties)"].
- **What it contains.** The View dialog *Properties tab*: Scale, Rotation, **Show hidden lines**, view frame dimensions (xy-viewport, Automatic), view depth (front/rear), "Use running dimension settings", "Restore user-erased parts" [V, AS 2022 help *View Dialog: Properties Tab*]. The *Clip tab*: minimum length to cut, X/Y clipping, how much is kept on each side, clipping representation length, clipping type and line type, extension, clip oblique lines [V, *View Dialog: Clip Tab*]. The *Edit detail* dialog (`_AstM4DetChange`) shows the same view properties, one view at a time [V].
- **Multi-selection.** AS 2021 release notes: *"Fixed an issue regarding unnecessary updates due to changes through Advance View properties after selecting multiple detail drawing elements."* [V]. AS 2021 also added multi-dimension editing: *"Dimension label characteristics can be modified for multiple dimensions at once."* [V]. So multi-view editing is available on AS 2022. I did not find the release that introduced it (2019/2020 notes not readable).
- **Labels.** The Label and Symbols dialog → **Leader Line tab** has *Show line*, *Arrow style*, *Leader line attachment* (Underline/Middle) and *Horizontal landing* [V]. The AS 2022 defaults for new labels are in Management Tools → Drawing-Labeling → General → Defaults [V]. Community tip: "select a label, select Similar, and then select the Advance properties" to switch leaders on for many labels [C snippet, forum "Turn on label leaders"].
- **API exposure.** No source says scale or clipping is available as an OPM/COM dynamic property, or via `getpropertyvalue`/`setpropertyvalue`. `01` already notes that AS drawing objects look like proxies/custom objects to AutoCAD [V/C]. Scale and clipping changes run a **quick update** that needs the model: "apply / disable hidden lines, clipping etc." all need it (Autodesk staff, quoted in `04` Q1) [V]. That is strong evidence these are AS-kernel operations, not plain property writes [I]. Even if a property turned up in `dumpallproperties`, writing it would probably skip the update and desynchronise the view [I]. Treat the dialog as the supported path.
- **Command name.** Not documented. It can be found by right-clicking a view on the Windows box and reading the command echoed on the command line (or `CMDNAMES`/`LASTPROMPT` from a reactor), or from the shortcut-menu macros in the AS CUIx.

### 2.2 Styles and processes that remove manual work (DSM 8.1 guide [V, old but same model]; AS 2022 help [V])

- **View scale.** Each view has a scale and a *Scale in process* mode: *Fixed* ("the process cannot override the view scale"), *Process can override*, *Depend on main view* [V]. Processes add *Alternative scales*, tried when the detail does not fit, and *Special scales* for dependent views [V]. So 1:10 for frame/infill sheets and 1:2 for single parts is a style setting.
- **Clipping (shortening).** The clipping strategy is Clipping X / Y / XY / off. The parameters are *Minimum length to cut*, *How much should be kept on each side of the clipping area*, *Clipping representation length* and type. Guide example: "minimum length to cut of 60mm with 20mm kept on each side" [V]. Graitec UK's clipping guide explains the same parameters and adds that "individual detail properties can supersede drawing style settings" [C]. The drawing style also has *Automatic clipping*, which fills the page left/right/top/bottom [V]. The clip line type comes from Management Tools, not the style [V, support article].
- **Suppressing dimensions and welds.** A view's dimension strategy is a list of dimension chains (requests), and an empty list means no automatic dimensions [V]. At creation, *Modify settings* has "Switch off" boxes for level symbols, **weld symbols**, building grids, **hidden lines**, **clipping** and hole representation. "Once modified, these settings are saved for future use." [V, AS 2022 *Create a Detail Drawing*]. A community answer confirms that turning off welds on the *Modify settings* → *New Drawing* page works [C snippet]. For processes, put it in the style [I].
- **Frame-only vs infill-only views.** Object-presentation rules assign presentation and labeling per *model object*. A model object "is defined by the Advance object type (e.g., Beam), model role (e.g., Column) and an additional filter" [V]. Two view requests with rules that show only "frame" roles or only "infill" roles give the client's sheets 2 and 3 without Select Similar → Delete. This requires consistent model roles on railing parts, which is a modelling rule for the client [I].
- **Absolute + chain dimensions.** The dimension types include *Linear, absolute*, *Linear, absolute from end*, *Linear, relative*, *Linear, relative and absolute*, *Linear, relative and absolute from end* [V]. Running-dimension settings take an ordinate reference: *Left/Right side of the main/total assembly* [V]. That covers "zero on the right" and "from the open corner" when the corner is at an assembly extreme. When it is not, the AI adds a manual chain with `_AstM4CommDetAddDim` [I].
- **Labels.** Label content is built from variables, including `%PosNum`, `%SinglePartPosNum`, `%MainPartPosNum`, **`%Quantity`** and **`%MainPartQuantity`** ("Quantity in main part") [V]. A content such as `%MainPartQuantity` + "X " + `%PosNum` gives "3X 1028" [I]. A labeling strategy has a *Max distance* option that "combines labels", and leader definitions [V]. I found no documented "label each identical part only once" switch. Plan on the AI erasing duplicates and checking whether deleted automatic labels stay deleted, as dimensions do (spike). Watch out: manual labels on "complete model" details can be empty unless the tokens are loaded (Management Tools "Tokens that will always come from model") [V, support article].
- **Angle dimensions.** Style-level angle requests exist only as arc-angle types in the list above. "Into the material, outside" is a placement rule, so treat it as AI + `_AstM4CommDetAddDimAngle` [I].

### 2.3 Processes and assembly detailing direction

- **Running processes.** `_AstM4CommDetailingProc User <n>` runs process *n* ("_AstM4CommDetailingProc User 52 calls the All Assembly PageFull A1 drawing process") [V]. The client's #34 ("Selected assemblies, one per sheet") and #4 ("single parts to fill the sheet") are user-category numbers. Close-page options are *On next object / On next main part / On page full* [V]. *Selected …* processes prompt for a selection. Whether they accept a PICKFIRST preselection or a selection passed by `Editor.Command` is undocumented [I, spike].
- **Process suite.** `AstProcessesSuite` runs several processes in sequence with saved selections ("You will not be able to Use the suite until you have made a selection for each line requiring one"). Suites are stored per machine [V/C, Graitec UK blog; AS UG 7.1 command table].
- **Define assembly detailing direction.** Advance Tool Palette → Tools → *Define assembly detailing direction*: place the UCS so that X/Y match the wanted front view, click, then select the assembly [V, Autodesk support article, Apr 2025]. Verify with main part → right-click → *Show Assembly CS* [C snippet]. The direction is reset when the assembly numbering is deleted [V snippet]. No `_Ast…` name is published. It is scriptable in principle (set UCS, then the command, then pick), but choosing the UCS is the client's judgement, so it stays **Human**.

### 2.4 Batch Plot and views outside the sheet

- Document Manager → *Add to batch plot* → choose the plotter → plot [V]. The page setup (plotter, plot style, **area**) comes from the drawing, which inherits it from the prototype. Community advice: "prepare drawing template and plot it first using desired pdf printer … During Batch plot it automatically selects the printer, plot style and extents set for template" [C, forum 7277871]. The *Advanced Batch Plot* app documentation recommends Layout or Window plot types, because Limits/Display/Extents "may result in junk results" [C, third-party].
- **Mechanism [I].** With plot area = Extents, any AS view whose green frame (`AstAtomicDetView` extents) or annotation sits outside `HYPERSTEELPAGEFRAME` grows the drawing extents. The output then scales down or shifts, and a thin PDF printer or a centred plot can look blank. That matches the client's symptom.
- **Programmatic check [I].** In the drawing, collect the extents of every view entity and AS annotation, and compare them with the frame block reference's extents (or the prototype paper size). Read `Layout.PlotType` / `PlotSettings` to flag *Extents*. Fixing the prototype's page setup to Layout or Window is a one-time human change. The checker still matters because frames outside the sheet are also a layout defect.

### 2.5 Document Manager and title

- **Document Manager.** Buttons: *Force update* ("updates … without recording changes"; no revision entry), *Update revision*, *Add to batch plot*, *Add to explode*, *Change prototype*, *Open drawing* ("the Document Manager closes and your selected detail drawings open") [V]. No command-line equivalents are documented. The DM became modeless in AS 2023 [C, video title]. Status lives in per-drawing **.UPD** files next to the DWGs, and corrupt UPD files stop the DM from opening [V, support article]. Since AS 2022, missing drawings go to *Unknown* instead of being deregistered [V].
  Route for the plugin [I]: skip the DM. Enumerate the details (API `KernelServices.GetDerivedDocumentsForDwg`, see `01`), open them with `Application.DocumentManager.Open` in drawing-number order, and run `_AstM4CommDetUpdateDetail` in each. That removes the client's manual sorting. Whether this is exactly equivalent to *Force update* (revision bookkeeping, UPD) needs a spike.
- **"Sklop 1".** The title-block tokens are PROJECT…DRAWING_NO, POS_NUM ("Piece mark of object"), SINGLEPARTPOSNUM, PHASE, QUANTITY, NAME, USERATTR1-10, FILENAME and others [V]. None of them means "assembly of these single parts" on a multi-part fill-the-sheet sheet. Options [I]: (a) a **non-token** attribute in the prototype header that the AI writes. Non-token attributes are "driven manually" and are not overwritten by `_AstM4CommDetUpdateProjectData` (Update page header) [C/V, `04` Q3]. (b) Put the assembly mark into the detail/view title in the style (`%MainPartPosNum` for single parts). Ask the client whether "Sklop 1" is the header field or the view title.

### 2.6 Show hidden lines, Insert label, label dimensions

- **Show hidden lines** is a per-view property (View dialog Properties tab) [V]. Switching it needs a quick update [V staff]. There is also a style default and a switch in *Modify settings* at creation [V].
- **Insert label**: `_AstM4CommDetInsertAnno`. Before picking, option **`C`** opens "Edit label content definition", saved per object type [C, `04` Q2; Graitec forum]. Quantity comes from the label variables above, not from a command option [V/I]. AS 2020 added *Arrange labels* (copies a reference label's leader length to other labels) and *Align labels to reference* [V]. Both help with the "spread labels" step. Command names are unknown (CUIx).
- **Label dimensions.** Since AS 2021, dimension label characteristics can be edited for several dimensions at once [V].

---

## 3. New spike items (extend `05` §5)

| # | Experiment | Answers |
|---|---|---|
| 12 | Right-click a view → *Advance View properties*; right-click a label → *Advance Properties*. Record the command echoed and the CUIx shortcut-menu macro. Try the command with a preselected set from `Editor.Command` / `SetImpliedSelection` | Command names; whether one dialog pass with N views works from code |
| 13 | UIA on the View dialog (multi-select): set Scale 1:2 and Clip 40/40 on all views of a fill-the-sheet sheet; time it; check annotations after the quick update | Replaces spike 7 (`_AstM4DetChange` per view) as the scale route |
| 14 | `dumpallproperties` / `vlax-dump-object` / .NET `TypeDescriptor.GetProperties` on a view, label and AS dimension (extends spike 2) | Rules OPM/COM in or out for scale, clip and leaders |
| 15 | With the client, clone their user styles: "3D only", "frame 1:10 (role filter), no dims/welds", "infill 1:10", "single parts 1:2 fixed + clipping 40/40". Re-run processes #34/#4 on one railing and count the remaining manual steps | How much work style configuration removes (the ~70 % hypothesis in `05`) |
| 16 | Label strategy with `%MainPartQuantity`X`%PosNum`, all attached parts labelled, leader on. Erase duplicate labels, update, and check that they stay deleted | Sheet 1 labels without manual Insert label |
| 17 | Dimension request "Linear, relative and absolute (from end)" + running-dimension ordinate reference on the frame view | Absolute + chain dimensions from style |
| 18 | Read the prototype page setup (`PlotType`). Move a view outside the frame, batch plot, reproduce the blank; switch to Layout/Window; write a `check_frames_inside_sheet` tool | Batch Plot root cause + first MCP check |
| 19 | Open the details via API in sorted order + `_AstM4CommDetUpdateDetail`; compare the DM state and UPD files with DM *Force update* | Document Manager bypass |
| 20 | `_AstM4CommDetailingProc User 34` with a PICKFIRST preselection of an assembly | Unattended process runs |

---

## 4. Questions for the client

- Is "40–40" *minimum length to cut* / *kept on each side*, or something else in the Clip tab?
- Are railing frame and infill parts distinguishable by **model role** (or name/section) in your models? This decides whether sheets 2/3 can come straight from the style.
- Is "Sklop 1" the header field or the view/detail title? Can the prototype carry an extra non-token attribute for it?
- What plot area does your prototype's page setup use (Extents / Layout / Window)?
- Which cases need hidden lines on sheet 1? (A rule the AI can apply.)

---

## Sources

**Autodesk help / release notes [V]**
- Change a Detail (AS 2022): https://help.autodesk.com/cloudhelp/2022/ENU/AdvSteel-Drawings/files/GUID-7CE73442-F35F-4A80-AB7C-53312C6C3C43.htm
- The Edit Detail Dialog: https://help.autodesk.com/cloudhelp/2022/ENU/AdvSteel-Drawings/files/GUID-82AD8671-61E4-4195-AF15-A486F72D62DC.htm
- View Dialog: Properties Tab: https://help.autodesk.com/cloudhelp/2022/ENU/AdvSteel-Drawings/files/GUID-49947E96-B5BD-409B-9105-6D9FE5853B8C.htm
- View Dialog: Clip Tab: https://help.autodesk.com/cloudhelp/2022/ENU/AdvSteel-Drawings/files/GUID-3DCCDC25-1A63-44A9-A6B6-7774ED4F3306.htm
- View Properties (DSM): https://help.autodesk.com/cloudhelp/2022/ENU/AdvSteel-Drawings/files/GUID-4E91200D-9A70-4D56-B376-7066F26062DA.htm
- Label and Symbols Dialog: Leader Line Tab: https://help.autodesk.com/cloudhelp/2022/ENU/AdvSteel-Drawings/files/GUID-6623DACD-4412-41EB-9A2F-4D9C65FF7820.htm
- Labeling Strategy Properties: https://help.autodesk.com/cloudhelp/2022/ENU/AdvSteel-Drawings/files/GUID-1D7AB09B-ADE0-4A40-81EE-E178C1ACB7CA.htm
- Align and Arrange Labels: https://help.autodesk.com/cloudhelp/2022/ENU/AdvSteel-Drawings/files/GUID-101AE16E-AAA3-42A7-8C95-452A9DA33033.htm
- Create a Detail Drawing (Modify settings, Switch off boxes): https://help.autodesk.com/cloudhelp/2022/ENU/AdvSteel-Drawings/files/GUID-30062C5A-58CF-4915-9984-44CB2AD7AB5D.htm
- Use a Drawing Process: https://help.autodesk.com/cloudhelp/2022/ENU/AdvSteel-Drawings/files/GUID-D81EFD24-E966-44F6-908A-F3A41BF0EBE6.htm
- Use the Document Manager: https://help.autodesk.com/cloudhelp/2022/ENU/AdvSteel-Drawings/files/GUID-962AAAB9-3AB1-4C8B-A462-F57B82B13509.htm
- The Document Manager Window: https://help.autodesk.com/cloudhelp/2022/ENU/AdvSteel-Drawings/files/GUID-E5DF2EB6-803F-4B86-BE82-D7CCA739C3EE.htm
- Document Management Changes to Drawing Registration (2022): https://help.autodesk.com/cloudhelp/2022/ENU/AdvSteel-WhatsNew/files/GUID-140FD1BE-5F9E-4783-99C9-1F8AD6F6D582.htm
- Title Block Token/Attribute List: https://help.autodesk.com/cloudhelp/2022/ENU/AdvSteel-Drawings/files/GUID-8831F91F-4AFE-489C-8DCD-CDED8DD64DD1.htm
- New in Advance Steel 2021 (Advance View properties + multiple selection fix; multi-dimension label edit): https://help.autodesk.com/cloudhelp/2021/ENU/AdvSteel-WhatsNew/files/GUID-D64995B8-B423-4CB7-8B9D-9C46F3C88081.htm
- Update the Page Header (`_AstM4CommDetUpdateProjectData`, snippet): https://help.autodesk.com/cloudhelp/2019/ENU/AdvSteel-Drawings/files/GUID-57E3D83D-A4A0-44D2-A120-59735C54E6B3.htm
- Support: How to define a custom view direction as default for assembly (Apr 2025): https://www.autodesk.com/support/technical/article/caas/sfdcarticles/sfdcarticles/Advance-Steel-How-to-define-a-rotated-assembly-front-view-direction-for-detailing.html
- Support: Inconsistent detailing view direction for assemblies with special parts: https://www.autodesk.com/support/technical/article/caas/sfdcarticles/sfdcarticles/Inconsistent-detailing-view-direction-for-assemblies-with-special-parts-inside-in-Advance-Steel.html
- Support: How to renumber assemblies while keeping the defined detailing direction (snippet): https://www.autodesk.com/support/technical/article/caas/sfdcarticles/sfdcarticles/How-to-renumber-assemblies-while-keeping-the-defined-detailing-direction-in-Advance-Steel.html
- Support: Cannot open Document Manager (UPD files): https://www.autodesk.com/support/technical/article/caas/sfdcarticles/sfdcarticles/Cannot-open-Document-Manager-for-a-project-in-Advance-Steel.html
- Support: Manual labels empty in overview drawings (tokens loaded): https://www.autodesk.com/support/technical/article/caas/sfdcarticles/sfdcarticles/Cannot-add-manual-labels-for-parts-in-Advance-Steel-overview-drawings-have-to-use-Label-Content-Manager-every-time.html
- Support: Clipping line type controlled by Management Tools (snippet): https://www.autodesk.com/support/technical/article/caas/sfdcarticles/sfdcarticles/Advance-Steel-Clipping-linetype-for-detail-view-cannot-be-adjusted-in-the-drawing-style.html
- AS 2015 User's Guide (Advance Properties vs Properties; drawing style tabs): https://download.autodesk.com/us/Advance_Steel/2015/AS-User-guide-2015-EN-Metric-140408.pdf

**Graitec (official at the time, older versions) [V-old]**
- Advance Steel User's Guide 7.1 (command appendix: `AstProcessesSuite`, `AstM4DetChange`, `AstM4CommDetInsertAnno`, …): https://download.graitec.com/public/Common/Downloads/Documentation/AS-User%20Guide-7.1-EN-Metric-070313-A4.pdf
- Drawing Style Manager guide 8.1 (scale in process, clipping, dimension types, running dimensions, model objects, label variables, processes/alternative scales): http://www.graitec.info/Common/Downloads/Documentation/AS-DSM_guide-8.1-EN.pdf

**Community / reseller [C]**
- Graitec UK, A Guide to Advance Steel Clipping: https://graitec.com/uk/blog/a-guide-to-advance-steel-clipping/
- Graitec UK, Drawing Process Suite: https://graitec.com/uk/blog/advance-steel-drawing-process-suite/
- Forum: How do I set my Batch Plot default in the Document Manager: https://forums.autodesk.com/t5/advance-steel-forum/how-do-i-set-my-batch-plot-default-in-the-document-manager/td-p/7277871
- Forum (snippet, archived): Turn on label leaders: https://forums.autodesk.com/t5/advance-steel-forum/turn-on-label-leaders/td-p/7080141
- Forum (snippet, archived): Green frame around details lost ("Advance detail properties" / inner frame = Advance View properties): https://forums.autodesk.com/t5/advance-steel-forum/green-frame-around-details-lost/td-p/6511557
- Forum (snippet, archived): Define Assembly Detailing Direction: https://forums.autodesk.com/t5/advance-steel-forum/define-assembly-detailing-direction/td-p/5817379
- Forum (snippet): Omit weld symbol for standard welds: https://forums.autodesk.com/t5/advance-steel-forum/omit-weld-symbol-for-standard-welds/td-p/5864130
- Advanced Batch Plot app (plot type recommendation): https://apps.autodesk.com/ADSTPR/en/Detail/HelpDoc?appId=3786088757909240118&appLang=en&os=Win64
- Design Consulting, What's New in AS 2022 (leader lines, batch explode): https://www.designconsulting.com.au/2021/04/07/whats-new-in-advance-steel-2022/
- YouTube, Advance Steel 2023: Modeless document manager: https://www.youtube.com/watch?v=CLDpRQWMZwg
