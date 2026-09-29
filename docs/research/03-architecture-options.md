# 03 — Architecture options: Claude ↔ Advance Steel 2022 shop drawings

Status: research note, 2026-09-29. Scope: Claude Desktop (MCP client) on a Windows PC running Autodesk Advance Steel 2022 (AutoCAD 2022 platform, .NET Framework 4.8). Goal: read and modify 2D shop drawings (view scale, view placement, dimensions, labels, title block) with undo, a "propose then apply" mode, logging, and edits that stay visible on screen so engineers can fix them by hand.

Legend used throughout:
- **[V]** verified: read in an official doc, a package manifest, or source code (link given).
- **[I]** inferred: engineering judgement or community knowledge that has not been tested on this project's machine. Each [I] item is a candidate for a spike.

---

## TL;DR

**Fastest PoC (1–3 days): option A, the client's proposal, with a narrow scope.**
- A **C# net48 plugin** loaded with `NETLOAD` inside Advance Steel. It listens on a **per-user named pipe** (or 127.0.0.1 TCP) and runs every request on the AutoCAD main thread under `LockDocument()`.
- A **separate MCP server process** speaking stdio to Claude Desktop, in **Python (`mcp` SDK)** for speed of iteration, or in **C# (.NET 8, `ModelContextProtocol`)** if the team wants a single language.
- Phase 1 is read-only: a JSON dump of the layout (viewports, dimensions, texts, title-block attributes, AS object class names) plus a **PNG of the sheet**. Two open-source projects already run this exact pattern on AutoCAD 2022 net48 (`bimwright/dwg-mcp`, `hoganmas/autocad-mcp`), so there is working code to copy.

**Production path: the same option A, hardened.**
- The MCP server becomes a C# .NET 8 stdio server (the official SDK) with a typed contract shared with the plugin.
- Writes go through a **plan → apply** protocol: `propose_*` returns a diff and a plan id with no side effects, and `apply_plan(plan_id)` executes it as **one registered AutoCAD command**. That gives one named undo step, visible in the command history, which `U` reverts.
- Every call is logged to a JSONL audit file.
- Edits to Advance Steel–specific objects (AS detail views, AS dimensions, AS labels) use the **Advance Steel .NET API where it covers them**. Where it does not, they use **AS's own commands driven from the plugin**. Where neither works, plain AutoCAD API edits are allowed only for objects AS does not regenerate: title-block attributes, free text, and "manual" dimensions.
- Batch mode reuses the same MCP server from **Claude Code headless (`claude -p`) or the Claude Agent SDK**, with a plugin tool that opens the next drawing.

**Do not build on:**
- **E (offline DXF/DWG editing).** It destroys or ignores AS custom objects and bypasses AS's model link.
- **D (computer use) as the main path.** It is too slow and too flaky for dimension-level edits. Keep it as a fallback for dialogs the API cannot reach.

**The biggest unknown is not the plumbing, it is Advance Steel.** Community sources state there is no public API to *create* AS drawings [V-ish, forum, see §A.7]. How much of an *existing* AS detail view (scale, position, AS dimensions) can be changed through the AS .NET/COM API is unverified [I]. Day 1 of the PoC must include a discovery spike on the real machine: dump the RX class names in a drawing DWG, probe properties, and try one scale change.

---

## Comparison

| Option | Effort to PoC | Reliability | Keeps AS associativity? | Needs AS API? | Demo in 1 day? |
|---|---|---|---|---|---|
| **A. In-process .NET plugin + out-of-process MCP server** | Medium (2–5 days for read + PNG + 2–3 edits) | High: deterministic, transactional, one undo group | **Yes**, if edits go through AS objects, AS commands, or non-AS entities only | For AS-specific edits yes; for read/PNG/title block no | **Yes** for read + PNG (copy dwg-mcp patterns) |
| A′. MCP server *in-process* inside AutoCAD | Medium-high | Medium: dependency conflicts in net48, no stdio | Same as A | Same as A | Unlikely |
| **B. Out-of-process COM (pywin32 / C# COM interop)** | Low-medium | Medium-low: slow cross-process calls, `RPC_E_CALL_REJECTED` when AS is busy, limited AS surface | Mostly yes (runs inside AS), but little AS-specific access | AS COM is mostly for joints/rules; drawing objects unclear | **Yes** for read-only demos |
| **C. LISP / script / `SendStringToExecute`** | Low | Medium-low: string commands are brittle (prompts, locale, dialogs); hard to return structured results | Yes, if it calls AS commands | No, uses AS commands | **Yes** (e.g. `puran-water/autocad-mcp`, `EmptyKot/acad-cmd`) |
| **D. Vision / computer use / UI Automation** | Very low to start, very high to make reliable | Low: pixel/UI-tree dependent, seconds per action | Yes, it is literally the user's UI | No | Yes, as a stunt; not trustworthy |
| **E. Offline DXF/DWG (ezdxf, netDxf, ODA)** | Low for reading, high for writing | Reading OK; writing **breaks** AS objects | **No**: AS entities become proxies or are dropped | No (that is the problem) | Read-only yes, edit no |

---

## A. In-process .NET plugin + out-of-process MCP server (recommended)

### A.1 Topology

```
Claude Desktop ──stdio/MCP──► mcp-server.exe (Python or .NET 8)
                                   │  NDJSON over named pipe \\.\pipe\as-ai-<user>  (or 127.0.0.1:<port>)
                                   ▼
             acad.exe (Advance Steel 2022) ── AsAi.Plugin.dll (net48, NETLOADed)
                     main-thread queue → LockDocument → Transaction → AS/ACAD API
```

Why separate the two processes:
1. Claude Desktop starts local MCP servers as **stdio child processes**. It cannot attach to stdio of an already-running `acad.exe` [V: Claude Desktop local-server model, https://modelcontextprotocol.io/docs/2026-07-28/develop/connect-local-servers].
2. The MCP SDKs pull in modern dependencies that are awkward inside a net48 AutoCAD process (§A.3).
3. If AS crashes or restarts, the MCP server survives and reports "plugin not connected" instead of Claude losing the tool list.

### A.2 MCP SDK choice

| | C# `ModelContextProtocol` | Python `mcp` |
|---|---|---|
| Version | 2.2.0 (2026-08-13) [V: https://www.nuget.org/packages/ModelContextProtocol.Core] | 2.2.0 (2026-09-07), Python ≥ 3.10, v2 targets the 2026-07-28 spec [V: https://pypi.org/project/mcp/] |
| Target frameworks | `net10.0;net9.0;net8.0;netstandard2.0` for both `ModelContextProtocol.Core` and `ModelContextProtocol` [V: csproj, https://github.com/modelcontextprotocol/csharp-sdk/blob/main/src/ModelContextProtocol.Core/ModelContextProtocol.Core.csproj, https://github.com/modelcontextprotocol/csharp-sdk/blob/main/src/ModelContextProtocol/ModelContextProtocol.csproj]. `ModelContextProtocol.AspNetCore` (HTTP server) needs ASP.NET Core, so it does not run on net48 [I, strong] | n/a |
| Pros | One language with the plugin; shared DTO assembly (netstandard2.0) between server and plugin; typed | Fastest to iterate; image handling (Pillow) trivial; many AutoCAD MCP examples in Python |
| Cons | Needs .NET 8 runtime on the PC (or a self-contained publish) | Two languages; Python runtime distribution (use `uv`) |

Recommendation: **Python for the 1-day PoC, C# .NET 8 for production** (or stay in Python if the team prefers; the plugin contract is language-neutral NDJSON).

### A.3 Could the MCP server live in-process (net48)?

- Technically the SDK has a `netstandard2.0` target, so it *can* be referenced from net48 [V]. Its netstandard2.0 dependencies include `System.Text.Json ≥ 10.0.10`, `System.IO.Pipelines ≥ 10.0.10`, `Microsoft.Extensions.AI.Abstractions ≥ 10.8.3` and `System.Net.ServerSentEvents` [V: NuGet page]. Inside `acad.exe`, which loads its own and other add-ins' versions of `System.*`/`Microsoft.*` assemblies, net48 has **no AssemblyLoadContext isolation**. This is the classic binding-redirect / "first loaded wins" conflict [I, strong].
- Transport problem: in-process there is no stdio. You would need Streamable HTTP inside AutoCAD, built by hand on `HttpListener` (the AspNetCore package is not an option on net48 [I]).
- `HttpListener` on anything other than `localhost` needs a URL ACL reservation (`netsh http add urlacl url=http://+:8080/ user=...`) or admin rights [V: https://learn.microsoft.com/windows-server/networking/technologies/netsh/netsh-http#add-urlacl, https://learn.microsoft.com/aspnet/web-api/overview/older-versions/self-host-a-web-api#optional-add-an-http-url-namespace-reservation]. Binding exactly `http://localhost:<port>/` generally works without admin on modern Windows [I, community knowledge; verify].
- Claude Desktop's config file launches local **stdio** servers. Pointing it at a local HTTP endpoint needs a stdio↔HTTP shim such as `mcp-remote` [I]. Claude Code can add local HTTP servers directly (`--transport http`) [I, per Claude Code MCP docs].
- **Verdict:** possible, but it adds risk for no user-visible benefit. Keep the in-process part to a tiny, dependency-light JSON-over-pipe server using `Newtonsoft.Json`, which AutoCAD already ships and which dwg-mcp uses [V: dwg-mcp `plugin-acad24.csproj` references Newtonsoft.Json 13.0.3].

### A.4 IPC choices (net48 plugin side)

| Transport | Notes |
|---|---|
| **Named pipe** (`System.IO.Pipes.NamedPipeServerStream`) | Available in net48. Local only, no firewall prompt, per-user ACL possible via `PipeSecurity`. Used by `power-cad-mcp` (per-user pipe + token), `765T-Forge` (named pipe + `FORGE_AUTOCAD_TOKEN`), `cad-mcp` (JSON-RPC framing), and `dwg-mcp` for AutoCAD 2025+ [V: READMEs linked in §A.8]. |
| **TCP on 127.0.0.1** (`TcpListener`) | Simplest. `dwg-mcp` uses TCP NDJSON for AutoCAD 2022–2024 (ephemeral port + per-session token) [V: `src/shared/Transport/TcpTransportServer.cs` binds `IPAddress.Loopback, 0`]. `hoganmas/autocad-mcp` uses a fixed loopback port [V: `Plugin/AutoCADMCPBridge.cs`]. Loopback binding does not need a URL ACL. It may still trigger a firewall prompt on some setups [I]. |
| `HttpListener` | Works, but has URL ACL friction (above) and no benefit over raw pipe/TCP for a private channel. |
| COM (`GetInterfaceObject` of a registered COM server) | Pattern from Kean Walmsley: an external COM app calls a .NET module in-process [V: https://keanw.com/2009/05/interfacing-an-external-com-application-with-a-net-module-in-process-to-autocad.html]. Needs COM registration; more ceremony than a pipe. |

**Recommendation:** named pipe `\\.\pipe\as-ai-<username>`, NDJSON framing `{id, method, params, token}` → `{id, ok, result|error, log_ref}`, plus a random token written to `%LOCALAPPDATA%\as-ai\session.json` on `MCPSTART`. Use TCP loopback as a fallback if pipe ACLs cause trouble.

### A.5 Getting onto the AutoCAD main thread

The AutoCAD .NET API is single-threaded. The pipe listener runs on a worker thread and must marshal each job onto the UI thread.

| Mechanism | Context it gives you | Notes |
|---|---|---|
| **`Application.Idle` + queue** | Application context; you must `LockDocument()` yourself | The pattern in both reference repos. dwg-mcp: `DocumentInvoker` + `MainThreadExecutor` (ConcurrentQueue, `Application.Idle += OnIdle`, dequeues one job per Idle, `using (doc.LockDocument())`, 15 s timeout that cancels only *unstarted* jobs so a timed-out mutation never runs later) [V: `src/shared/Infrastructure/DocumentInvoker.cs`, `MainThreadExecutor.cs`]. hoganmas: `Application.Idle += ProcessCommands` [V]. Caveat: Idle fires when the message queue drains, so a job queued while AS sits idle may wait for the next UI message (mouse move) [I; mitigate by posting a no-op message to the main window, or with a WinForms `Control.BeginInvoke` on a control created on the main thread]. |
| **Captured `SynchronizationContext` / hidden `Control.BeginInvoke`** | Application context | Capture `SynchronizationContext.Current` (WindowsFormsSynchronizationContext) when `MCPSTART` runs, then `Post` jobs. Wakes immediately [I]. Still needs `LockDocument()`. |
| **`DocumentCollection.ExecuteInCommandContextAsync`** (AutoCAD 2016+) | **Command context**: `Editor.Command` is allowed, undo/grip behaviour matches a real command | Documented use: firing commands from external/OS events (FileSystemWatcher sample) and from palettes [V: https://keanw.com/2015/03/autocad-2016-calling-commands-from-external-events-using-net.html, https://keanw.com/2015/03/autocad-2016-calling-commands-from-autocad-events-using-net.html]. Kean warns it is "highly volatile": any exception thrown in the delegate can crash AutoCAD, so wrap everything in try/catch [V: same blog series]. Must be called from the main thread, so combine it with one of the above. |
| **`SendStringToExecute("_ASAI_APPLY <planId>\n", …)`** | Real command, runs when AutoCAD is ready | The simplest way to get **one named undo group** and a visible command-history entry. Asynchronous: the result must come back through the pipe once the command finishes. hoganmas uses it for a render step [V: `ViewCommands.cs`]. |

`CommandFlags.Session` is needed for commands that must work across documents or open/switch drawings (e.g. `MCPSTART`, and batch "open next drawing"). dwg-mcp registers all its control commands with it [V: `plugin-acad24/App.cs`]. Document-modifying commands should be normal document commands (`CommandFlags.Modal`), because they run in that document's context and get automatic locking and undo grouping [I, standard AutoCAD .NET practice].

**Recommended design:**
- **Reads** run as Idle/SyncContext job → `LockDocument(DocumentLockMode.ProtectedAutoWrite or Read)` → `OpenMode.ForRead` transaction → JSON.
- **Writes** go through `apply_plan`. The plugin stores the plan, then either calls `ExecuteInCommandContextAsync` or posts `SendStringToExecute("_ASAI_APPLY " + id + " ")`. The `ASAI_APPLY` command (Modal) wraps the entire plan in one `Transaction` (plus AS `DocumentAccess` where needed), commits, regenerates, and reports back.
- The result is **one undo step named ASAI_APPLY**. The user sees it in the command line and can press `U`/Ctrl+Z. The edited objects stay ordinary drawing objects the engineer can grip-edit.
- If the user is mid-command (`CMDACTIVE` ≠ 0), reject writes with a clear error instead of queueing. dwg-mcp's capture rejects "an active command or a document/camera change during capture" [V: README].

### A.6 Transactions, locking, undo, "propose then apply", logging

- **Transactions:** `db.TransactionManager.StartTransaction()` … `tx.Commit()`. dwg-mcp states "each operation wraps in a transaction, providing single undo" [V: README; e.g. `src/shared/Rewriting/RewriteExecutor.cs`].
- **AS objects:** AS docs show combining the AutoCAD `Transaction` with the AS `DocumentAccess` (`using (DocumentAccess da = new DocumentAccess(null, false))`), and aliasing the clashing `ObjectId` types (`Autodesk.AdvanceSteel.CADLink.Database.ObjectId`). They also note that AS objects are "composed of two AutoCAD entities: the 'internal' entity and the 'representation' entity" [V: https://help.autodesk.com/cloudhelp/2022/ESP/AdvSteel-Developers/files/GUID-0AFEFD2B-EED4-4F6A-8F97-BA0814E99643.htm].
- **Undo:** use one command per plan, as above. As an alternative inside application context, bracket with `UNDO BEGIN/END` via `Editor.Command` in command context [I].
- **Propose then apply:** `propose_*` tools compute and validate against the live DB with no writes and return:
  - a human-readable diff (handle, property, old → new)
  - a plan id plus a DB fingerprint (e.g. `DBMOD` + handle seed + file name)
  - optionally a "ghost" preview: transient graphics via `TransientManager`, which are drawn but not in the DWG, so there is no undo pollution [I]

  `apply_plan` refuses the plan if the fingerprint changed. There is precedent: `beiming183-cloud/AutoCAD-MCP` requires `session_id`, `document_id` and `expected_revision` for mutations and returns `E_DOCUMENT_REVISION_MISMATCH` on stale state [V: README]. `765T-Forge` has dry-run plus backup-before-write [V: README]. dwg-mcp uses `dry_run=true` then `confirm=true` [V: README].
- **Logging:** a JSONL audit per day with request, plan, before/after values, user, drawing path, duration and result. 765T-Forge does "JSONL audit … with secrets redacted" [V]. Also echo a one-line summary to the AutoCAD command line (`ed.WriteMessage`) so the engineer sees what the AI did.

### A.7 The Advance Steel layer (critical path, largely unverified)

- AS runs on AutoCAD, and the AS API is used together with the AutoCAD .NET API in one plugin (reference `AcCoreMgd`, `AcDbMgd`, `AcMgd` plus the AS assemblies) [V: AS 2022 developer walkthrough above; AS .NET and COM API guide https://help.autodesk.com/view/ADSTPR/2022/ITA/?guid=GUID-AF2072A9-3F8F-4DAC-B595-470A9E32BEE0].
- **Drawing creation is not in the API**, per a 2023 AS forum thread: "There is no API for drawing creation for Advance Steel … can only be adjusted within the drawing style manager, drawing process manager …" [forum statement, not official; seen in the search-result excerpt only, because the page returned 403 to the fetcher: https://forums.autodesk.com/t5/advance-steel-forum/generate-2d-drawing-with-net-api/td-p/11763769]. A 2018 thread claims someone automated shop-drawing creation from C# [unverified: https://forums.autodesk.com/t5/advance-steel-forum/advance-steel-api-options-for-numbering-and-detail-drawing/td-p/8216125]. The AS 2023 API change list mentions `Camera` query methods for "detail object operations" but nothing on drawing views or dimensions [V: https://help.autodesk.com/cloudhelp/2023/ENU/AdvSteel-Developers/files/GUID-309B2ACB-5CD4-4B91-98B7-D00F2026BBD7.htm].
- **Drawing DWGs contain AS custom objects.** Without AS or the Object Enabler they show as proxies or "lines or points only" [V: https://www.autodesk.com/support/technical/article/caas/sfdcarticles/sfdcarticles/proxy-warning-when-opening-advance-steel-drawings-in-autocad.html, https://www.cadforum.cz/en/how-to-display-a-dwg-from-advance-steel-including-as-objects-ast-tip12325].
- **Drawing updates and manual edits:** manual dimensions can disappear on a Document Manager update unless "Erase incorrect manual dimensions on update" is set to "Don't erase any manual dimension point". Autodesk recommends AS dimension tools over AutoCAD linear dimensions because AS dimensions "show the correct value regardless of drawing scale and scale along with detail views" [V: https://forums.autodesk.com/t5/advance-steel-forum/dimensions-disappearing-after-a-drawing-update/td-p/8159145, https://help.autodesk.com/view/ADSTPR/2022/ENU/?guid=GUID-D462912E-5259-476F-9EC8-B27A361B7A0E]. Implication: AI edits made with plain AutoCAD entities may be **overwritten or orphaned by the next "update drawing"** [I, strong].
- **View scale:** changing a detail's scale is a user operation on the AS detail's properties. Available scales come from the AstorBase `Scale` table [V: https://www.autodesk.com/support/technical/article/caas/sfdcarticles/sfdcarticles/How-to-add-a-custom-scale-to-be-available-for-Advance-Steel-details.html]. Whether this is exposed on an API object is **unknown** [I].
- **Day-1 spike (must do):**
  1. In a drawing DWG, list every entity with `GetRXClass().Name` / `DxfName`. Expect AS class names starting with "Ast…" [V that AS classes use the "AST" prefix: https://forums.autodesk.com/t5/visual-lisp-autolisp-and-general/advance-steel-lisp-help/td-p/9909928].
  2. `vlax-dump-object` on an AS detail view, AS dimension and AS label, to see which COM properties are exposed.
  3. Reflect over the AS assemblies in `C:\Program Files\Autodesk\AutoCAD 2022\ADVS\` for drawing-related types.
  4. Try to change one detail view's scale via property, then via AS command.
  5. Check that the change survives the AS "update drawing".

### A.8 Existing open-source AutoCAD MCP servers using this pattern

| Repo | Server | Plugin | IPC | Main-thread / undo | Notes |
|---|---|---|---|---|---|
| **bimwright/dwg-mcp** (Apache-2.0) https://github.com/bimwright/dwg-mcp | .NET 8 C#, stdio | per-version shells: **net48 for AutoCAD 2022–2024**, net8.0-windows 2025–26, net10 2027 | TCP NDJSON (2022–24), named pipe (2025+), per-session token | `Application.Idle` queue + `LockDocument()`, a transaction per op = single undo [V: source] | 41 default / 65 total tools; `dwg_capture_view_image` via `Document.CapturePreviewImage` returns an inline PNG; `dwg_send_code` Roslyn C# snippets (30 s timeout); dry_run/confirm on destructive ops. **Closest match to our stack; best code to study.** |
| **hoganmas/autocad-mcp** (MIT) https://github.com/hoganmas/autocad-mcp | Python (`uv run server.py`) | **net47**, `NETLOAD` + `STARTMCP` command | TCP loopback, fixed port | `Application.Idle += ProcessCommands`; `doc.LockDocument()` in command templates [V: source] | Almost exactly the client's proposal; small and readable. |
| **meococ/765T-Forge** https://github.com/meococ/765T-Forge | C# stdio, ModelContextProtocol 1.3.0 | net462 (2017–2024), net8 (2025–26) | Named pipe + token | Main-thread marshalling, 120 s timeouts | Dry-run + backup, JSONL audit, sheet-set publish, title-block campaigns. Good reference for logging and propose/apply. |
| **shtirlitsDva/cad-mcp** https://github.com/shtirlitsDva/cad-mcp | .NET 8 bridge exe | AutoCAD/Civil 3D **2025+** only | Named pipe, JSON-RPC framing, dependency-free kernel | Runs C# (Roslyn) "on the host's own API thread" | License TBD; mechanism not documented; not directly usable on 2022. |
| **khs0927/power-cad-mcp** https://github.com/khs0927/power-cad-mcp/pull/1 | C# + Python COM/DXF server | AutoCAD 2027 | Per-user named pipe, NDJSON + token | "verified-change" | 2027 only. |
| **beiming183-cloud/AutoCAD-MCP** https://github.com/beiming183-cloud/AutoCAD-MCP | Python | signed .NET 8 worker (2025/26) + COM/LISP fallback | Current-user named pipe + token | `DocumentLock` + DB transaction; revision check; COM path uses undo marks and admits it "cannot guarantee atomic rollback" | Good design notes on native vs. COM guarantees; PNG via native plotting. |

No public MCP server targeting **Advance Steel** specifically was found (searches for "Advance Steel MCP" only return an unrelated browser product called "Steel") [V: search 2026-09-29].

---

## B. Out-of-process COM / ActiveX automation

- Connect from Python (`pywin32`: `win32com.client.GetActiveObject("AutoCAD.Application.24.1")`, since AutoCAD 2022 = R24.1 [I, strong]) or from C# COM interop. `EmptyKot/acad-cmd` is a Python/FastMCP server that does exactly this: `GetActiveObject` then `Dispatch`, commands via `SendCommand`, output captured through `LOGFILEMODE`/`LOGFILENAME`, and retry on COM "busy" [V: https://github.com/EmptyKot/acad-cmd].
- **What you get:** the AutoCAD ActiveX model (documents, layouts, `PaperSpace`, `AcadDimension*`, `AcadMText`, `AcadBlockReference.GetAttributes()`, `AcadPViewport.CustomScale`, plotting). That covers title-block attributes and plain AutoCAD entities.
- **Advance Steel via COM:** AS ships COM libraries (`AstSTEELAUTOMATIONLib`, `DSCGeomCom`, `DSCProfilesAccesCom`, …). Their documented purpose is **joints/rules and model objects**, and rules "must be implemented in an in-process server" [V: AS COM API Developer Training Guide mirrors, e.g. https://vdocuments.mx/advance-steel-com-api-developer-training-guide.html]. AS objects created with `vlax-create-object "AstSTEELAUTOMATIONLib.AstCreator"` suggest some model-level COM access. **No evidence of COM access to drawing views or AS dimensions** [I]. AS entities seen through generic AutoCAD COM will likely expose only the base `AcadEntity` interface [I].
- **Downsides:**
  - Every property read is a cross-process RPC. Walking thousands of entities is slow [I, strong].
  - The call is rejected (`RPC_E_CALL_REJECTED`) whenever AS is busy or a modal dialog is open [I, strong; acad-cmd implements retry].
  - Undo grouping needs `StartUndoMark`/`EndUndoMark` [I]. beiming183's README states the COM path can only offer "compensation limits instead of promising rollback" [V].
- **Use:** good for a throwaway read-only demo without compiling anything. Not the production path.

## C. Script / LISP route

- Claude generates AutoLISP or command strings. They run via an `eval_lisp` tool (the plugin calls `SendStringToExecute("(load \"tmp.lsp\") ", …)`) or via COM `SendCommand`. Precedents:
  - `puran-water/autocad-mcp`: `execute_lisp` runs arbitrary AutoLISP via a temp file, File IPC backend with focus-free Win32 messaging, undo/redo, 8 consolidated tools, AutoCAD LT 2024+ [V: https://github.com/puran-water/autocad-mcp].
  - `acad-cmd`: `run_lisp`, `load_lisp_file` [V].
- **Pros:**
  - Can drive **AS commands**, which AS keeps associative, so this is the easiest way to invoke AS functionality the .NET API lacks.
  - Zero compile for experimentation.
  - Great as a *developer* escape hatch in the plugin (like dwg-mcp's `dwg_send_code`).
- **Cons:**
  - Command prompts vary by locale (use `_` prefixes and `.` for built-ins).
  - Dialogs (`CMDDIA`, `FILEDIA`) block.
  - Results come back as text (log file / `LASTPROMPT`), not structured data.
  - `SendStringToExecute` is async, so completion has to be detected.
  - Arbitrary code from an LLM inside the engineer's CAD session is a safety risk. Gate it behind an explicit `MCPENABLECODE`-style switch, as dwg-mcp does [V: `App.cs` `MCPENABLECODE`/`MCPDISABLECODE`].
- **Recommendation:** use it inside option A as a controlled fallback (`run_as_command(name, args)` with an allow-list), not as the primary data path.

## D. Vision / computer use / UI Automation

- **Claude computer use:** a beta in Claude Desktop (Cowork / Claude Code), Pro and Max only, macOS and Windows. It works from screenshots, is "slower than connectors", "complex tasks sometimes fail", and the desktop must stay awake [V: https://support.claude.com/en/articles/14128542-let-claude-use-your-computer-in-cowork, https://claude.com/blog/dispatch-and-computer-use]. Claude itself prefers connectors, then browser, then screen [V: same].
- **Windows-MCP (CursorTouch):** MIT, `uvx windows-mcp`. It uses the Windows UI-Automation tree plus mouse/keyboard, about 19 tools (Click, Type, Snapshot, Screenshot, PowerShell, …), and no CV model [V: https://github.com/CursorTouch/Windows-MCP]. `desktop-touch-mcp` is a similar Rust UIA engine [V: https://github.com/Harusame64/desktop-touch-mcp].
- **Fit for Advance Steel:**
  - AutoCAD's drawing canvas is a single opaque window to UIA, so dimensions and views are *not* in the accessibility tree [I, strong]. Only ribbons and dialogs are.
  - Every edit becomes screenshot → reason → click at pixel coordinates. That costs roughly 5–20 s and one or more image inputs per step [I], with drift from zoom, pan and snap.
  - No structured read-back and no transactional undo, apart from the user's own Ctrl+Z.
- **Where it is useful:** reaching AS dialogs with no API (Drawing Style Manager, detail-properties dialog, Document Manager "Update"), and demoing to management. Keep it as a tool of last resort, and prefer plugin-invoked AS commands.

## E. File-based DXF/DWG manipulation outside the app

- **Tools:**
  - `ezdxf` (Python, DXF only; DWG via the ODA File Converter add-on [V: https://ezdxf.readthedocs.io/en/stable/addons/odafc.html])
  - `netDxf` (C#, DXF)
  - the ODA File Converter (free, CLI, DWG↔DXF, batch/recursive/audit [V: https://www.opendesign.com/guestfiles/oda_file_converter])
  - ODA Drawings SDK / Teigha .NET (commercial membership)
- **Why it breaks AS:**
  - AS drawing objects are AS custom objects. To ezdxf they are unknown entities kept as opaque `DXFTagStorage`, or `ACAD_PROXY_ENTITY`, which "cannot be copied or transformed". Exploding them to proxy graphics loses "the meaning and data" [V: https://ezdxf.readthedocs.io/en/stable/dxfentities/acad_proxy_entity.html].
  - Plain AutoCAD without the enabler sees proxies [V: §A.7].
  - DWG→DXF→DWG round-tripping of custom objects through third-party libraries is lossy [I, strong]. ODA's SDK preserves unknown objects as proxies but cannot *edit* AS semantics without AS's object enabler [I].
  - It also bypasses AS's Document Manager link to the model. The next "update drawing" or AS consistency check may regenerate or flag the drawing [I].
- **Fine for:** read-only analytics (text/attribute extraction, title-block audit) on *copies*, and CI-style checks without an AS licence. Not for edits.

## F. How Claude "sees" the sheet

1. **`Document.CapturePreviewImage(width, height)`** returns a `Bitmap` of the current view, available since AutoCAD 2010 [V: https://adndevblog.typepad.com/autocad/2013/07/capturepreviewimage-api-to-create-the-image-from-a-autocad-document.html, https://keanw.com/2016/02/capturing-an-image-of-the-current-autocad-document-using-net.html, API ref https://help.autodesk.com/view/OARX/2023/ENU/?guid=OARX-ManagedRefGuide-Autodesk_AutoCAD_ApplicationServices_DocumentExtension_CapturePreviewImage_this_Document_uint_modoptIsLong_uint_modoptIsLong]. dwg-mcp returns it as an MCP image block with document/viewport/camera metadata, capped at 8 MiB [V]. It is fast, but the background colour equals the screen background [V: Kean] and it shows only what is on screen.
2. **Plot the layout to PNG** with `PlotEngine` + `PlotInfo` using the `PublishToWeb PNG.pc3` device and a custom pixel media size, with `BACKGROUNDPLOT=0` for synchronous output [V: .NET plot docs https://help.autodesk.com/cloudhelp/2016/ENU/AutoCAD-NET/files/GUID-CE91C625-A278-4445-8219-3A2D880E6C22.htm, PlotEngine https://help.autodesk.com/cloudhelp/2022/ENU/OARX-ManagedRefGuide/files/OARX-ManagedRefGuide-Autodesk_AutoCAD_PlottingServices_PlotEngine.html, custom PNG sizes https://forums.autodesk.com/t5/net/how-to-create-a-user-plot-size-of-publishtoweb-png-pc3-amp/td-p/9109860]. Result: a deterministic, full-sheet, white-background image that matches the printed sheet. **Best for Phase 1.** Also plot **cropped windows** (a view's extents in paper space) at high DPI so Claude can read small dimension text.
3. `PNGOUT` / `JPGOUT` commands capture the current viewport interactively. They are less controllable [I].
4. Plot to PDF (`DWG To PDF.pc3`) and rasterize outside. Useful for the archive/QA trail [I].

**Structured JSON alongside the image (essential; images alone are not enough to edit safely):**
- per-layout sheet info (paper size, plot device)
- viewports / AS detail views (handle, class, center, size, scale, and model-space view center if a viewport)
- dimensions (handle, class, measurement, text override, style, defining points, layer)
- texts / MText / AS labels (handle, content, position, height, rotation)
- title-block block references with attribute tag → value
- per-entity bounding boxes in paper units

Give **paper→pixel transform** metadata with each PNG so Claude can map "the dimension at pixel (x,y)" to a handle.

---

## Overnight batch mode (no Claude Desktop)

- The same MCP server works with **Claude Code headless** (`claude -p "<task>" --mcp-config as-ai.json --allowedTools "mcp__asai__*"`) or the **Claude Agent SDK** (Python/TS `query()` with `mcpServers` + `allowedTools`). Prefer `allowedTools` wildcards over `bypassPermissions` [V: https://code.claude.com/docs/en/agent-sdk/mcp].
  - Headless runs use only MCP servers that do not need interactive auth [V: same].
  - Tool calls have timeouts (`MCP_TIMEOUT`, `MCP_TOOL_TIMEOUT`), so long plots must stay under them or be made async [V: same].
  - Text-only tool results over 25k tokens get spilled to a file [V: same].
- **The Windows machine must have an interactive AS session running with the plugin loaded.** Advance Steel in `accoreconsole` is doubtful [I, likely unsupported for AS objects]. A logged-in, unlocked session is also safer for plotting [I].
- Plugin tools needed for batch:
  - `list_project_drawings(project_path)`: from the AS Document Manager data if reachable, otherwise a folder scan
  - `open_drawing(path, read_only)`: a `CommandFlags.Session` command / `DocumentCollection.Open` from application context
  - `save_drawing` / `close_drawing`
- **Orchestration:** a Python driver loops over the drawings, calls `query()` once per drawing with a fixed checklist prompt, and writes a per-drawing report plus proposed plans. It does **not** apply edits overnight: the engineer reviews and approves in the morning with `apply_plan`. Log token usage per drawing.

---

## Phase 1 PoC: minimal MCP tool list (read drawing + PNG)

Signatures are in TypeScript-ish notation. Every result includes `{doc_fingerprint, elapsed_ms, log_id}`.

```ts
// session / health
status(): { plugin_connected: bool, acad_version: string, as_version: string,
            active_document?: { path: string, name: string, is_as_drawing: bool, read_only: bool },
            command_active: bool }

list_open_drawings(): { documents: { id: string, path: string, active: bool }[] }

// structure
list_layouts(doc_id?: string): { layouts: { name: string, tab_order: int, paper: { w_mm: number, h_mm: number },
                                            plot_device: string, viewport_count: int }[] }

get_sheet(layout: string, include?: ("views"|"dimensions"|"texts"|"titleblock"|"other")[],
          max_entities?: int = 2000):
  { layout: string, paper_bbox_mm: Box,
    views:       { handle, class_name, kind: "acad_viewport"|"as_detail"|"unknown", bbox_mm: Box,
                   scale?: string, custom_scale?: number, label?: string }[],
    dimensions:  { handle, class_name, is_as_object: bool, value?: number, text?: string,
                   text_override?: string, style?: string, bbox_mm: Box, layer: string, owner_view?: string }[],
    texts:       { handle, class_name, content: string, height_mm: number, rotation_deg: number,
                   bbox_mm: Box, layer: string }[],
    titleblock:  { handle, block_name: string, attributes: Record<string,string> }[],
    other_counts: Record<string /*class_name*/, int>,
    truncated: bool }

get_entity(handle: string): { handle, class_name, dxf_name, layer, bbox_mm: Box,
                              properties: Record<string, any>,        // reflected ACAD props
                              as_properties?: Record<string, any> }   // via AS API if available

find_entities(layout: string, filter: { class_name?: string, layer?: string,
              text_contains?: string, within_bbox_mm?: Box }): { handles: string[] }

// images
render_sheet_png(layout: string, dpi?: int = 150, window_mm?: Box, max_px?: int = 2400):
  MCP image content + { path: string, px: {w,h}, paper_to_px: Affine2D, sha256: string }

capture_screen_view(width?: int = 1600, height?: int = 1000):
  MCP image content + { view_center, view_height, is_paper_space: bool }

// discovery spike helper (Phase 1 only, remove later)
dump_class_census(doc_id?: string): { by_space: Record<string, Record<string /*class*/, int>> }
```

Phase 2 (write, propose/apply). Listed so the Phase 1 contracts anticipate it:

```ts
propose_set_view_scale(view_handle: string, scale: string): Plan
propose_move_view(view_handle: string, dx_mm: number, dy_mm: number): Plan
propose_set_titleblock_attributes(values: Record<string,string>): Plan
propose_edit_text(handle: string, new_content: string): Plan
propose_dimension_text_override(handle: string, override: string | null): Plan
apply_plan(plan_id: string): { applied: bool, undo_label: "ASAI_APPLY", changed: Change[] }
discard_plan(plan_id: string): {}
get_audit_log(since?: string, limit?: int): AuditEntry[]
// Plan = { plan_id, summary, changes: { handle, prop, old, new }[], warnings: string[],
//          preview_png?: image, expires_at }
```

---

## Open questions / spikes (in priority order)

1. **AS drawing object model** on the real machine: class census, reflect AS assemblies, `vlax-dump-object` on an AS detail, AS dimension and AS label. Can scale and position be changed by API? By AS command? [I, blocks Phase 2]
2. Do AI-made edits survive **AS "update drawing"** from the Document Manager? Test both "manual dimension" settings.
3. Does `NETLOAD` of an unsigned DLL work under the customer's `SECURELOAD` settings? Otherwise use a trusted-path or Autoloader bundle (`ApplicationPlugins\*.bundle`). beiming183 notes signing requirements under `SECURELOAD` [V: README].
4. Named pipe vs TCP on the customer's endpoint-security stack.
5. Idle wake-up latency on AS 2022; decide between Idle and a captured SyncContext.
6. Image token budget: sheet PNG at 1600–2400 px plus crops; measure how well Claude reads dimension text at each DPI.

## Sources

- MCP C# SDK: https://github.com/modelcontextprotocol/csharp-sdk · csproj TFMs https://github.com/modelcontextprotocol/csharp-sdk/blob/main/src/ModelContextProtocol.Core/ModelContextProtocol.Core.csproj · NuGet https://www.nuget.org/packages/ModelContextProtocol.Core
- MCP Python SDK: https://pypi.org/project/mcp/
- Local MCP servers: https://modelcontextprotocol.io/docs/2026-07-28/develop/connect-local-servers
- Claude Agent SDK MCP: https://code.claude.com/docs/en/agent-sdk/mcp
- Claude computer use: https://support.claude.com/en/articles/14128542-let-claude-use-your-computer-in-cowork · https://claude.com/blog/dispatch-and-computer-use
- AutoCAD MCP projects: https://github.com/bimwright/dwg-mcp · https://github.com/hoganmas/autocad-mcp · https://github.com/meococ/765T-Forge · https://github.com/shtirlitsDva/cad-mcp · https://github.com/khs0927/power-cad-mcp/pull/1 · https://github.com/beiming183-cloud/AutoCAD-MCP · https://github.com/EmptyKot/acad-cmd · https://github.com/puran-water/autocad-mcp
- UI automation: https://github.com/CursorTouch/Windows-MCP · https://github.com/Harusame64/desktop-touch-mcp
- AutoCAD .NET: ExecuteInCommandContextAsync https://keanw.com/2015/03/autocad-2016-calling-commands-from-external-events-using-net.html · https://keanw.com/2015/03/autocad-2016-calling-commands-from-autocad-events-using-net.html · COM↔.NET https://keanw.com/2009/05/interfacing-an-external-com-application-with-a-net-module-in-process-to-autocad.html · CapturePreviewImage https://keanw.com/2016/02/capturing-an-image-of-the-current-autocad-document-using-net.html , https://adndevblog.typepad.com/autocad/2013/07/capturepreviewimage-api-to-create-the-image-from-a-autocad-document.html · Plotting https://help.autodesk.com/cloudhelp/2016/ENU/AutoCAD-NET/files/GUID-CE91C625-A278-4445-8219-3A2D880E6C22.htm , https://keanw.com/2007/09/driving-a-basic.html · AutoCAD 2022 targets .NET 4.8 https://help.autodesk.com/view/OARX/2022/ENU/?guid=GUID-E4C5C858-787F-4D35-B7FC-1F2CBA3D4966
- HttpListener URL ACL: https://learn.microsoft.com/windows-server/networking/technologies/netsh/netsh-http#add-urlacl · https://learn.microsoft.com/aspnet/web-api/overview/older-versions/self-host-a-web-api
- Advance Steel: API + AutoCAD walkthrough https://help.autodesk.com/cloudhelp/2022/ESP/AdvSteel-Developers/files/GUID-0AFEFD2B-EED4-4F6A-8F97-BA0814E99643.htm · .NET/COM API https://help.autodesk.com/view/ADSTPR/2022/ITA/?guid=GUID-AF2072A9-3F8F-4DAC-B595-470A9E32BEE0 · 2023 API changes https://help.autodesk.com/cloudhelp/2023/ENU/AdvSteel-Developers/files/GUID-309B2ACB-5CD4-4B91-98B7-D00F2026BBD7.htm · no drawing-creation API (forum) https://forums.autodesk.com/t5/advance-steel-forum/generate-2d-drawing-with-net-api/td-p/11763769 · manual dimensions https://help.autodesk.com/view/ADSTPR/2022/ENU/?guid=GUID-D462912E-5259-476F-9EC8-B27A361B7A0E , https://forums.autodesk.com/t5/advance-steel-forum/dimensions-disappearing-after-a-drawing-update/td-p/8159145 · custom scales https://www.autodesk.com/support/technical/article/caas/sfdcarticles/sfdcarticles/How-to-add-a-custom-scale-to-be-available-for-Advance-Steel-details.html · proxies/enabler https://www.autodesk.com/support/technical/article/caas/sfdcarticles/sfdcarticles/proxy-warning-when-opening-advance-steel-drawings-in-autocad.html , https://www.cadforum.cz/en/how-to-display-a-dwg-from-advance-steel-including-as-objects-ast-tip12325 · COM guide mirror https://vdocuments.mx/advance-steel-com-api-developer-training-guide.html
- DXF/DWG: https://ezdxf.readthedocs.io/en/stable/dxfentities/acad_proxy_entity.html · https://ezdxf.readthedocs.io/en/stable/addons/odafc.html · https://www.opendesign.com/guestfiles/oda_file_converter
