# SwitchScripter setup guide (for the per-script README)

This is the reference Claude uses to fill out the per-script `README.md` so the user knows exactly what to configure manually in **SwitchScripter** (the GUI declaration editor) and in **Switch Designer** (the flow canvas). Use it as a checklist, not a copy-paste.

SwitchScripter is launched by double-clicking the `<ScriptID>.sscript` placeholder in the script folder, or via "Edit in Scripter" on a Script element in Switch Designer.

## Step 1 — Open the script in SwitchScripter

The user double-clicks `<ScriptID>.sscript` inside the script folder. The Declaration pane shows the script's tree with the script-ID root node, **Flow element properties**, and **Outgoing connections properties**.

## Step 2 — Set the main script properties

Selecting the root node (the script ID) in the Declaration pane shows the main properties in the right pane. Fill these in based on what the script does:

- **Script ID** — already set by `--create`; matches the folder name.
- **Display Name** — user-facing name on the flow element. Defaults to Script ID if empty.
- **Type** — for script folders: locked to `Script`. For `.sscript` packages: `Script` (regular) or `App` (scripted plug-in / Appstore-style installable). Stick with `Script`.
- **Legacy** — must be `No` for script folders. (`Yes` is for legacy JavaScript/VBScript-only `.sscript` packages.)
- **Include Node.js script** — points to `main.ts` (TypeScript) or `main.js` (JavaScript) inside the folder. Already linked when the folder was created with `--create`.
- **Version** — integer, start at 1.
- **Icon** — optional. PNG, exactly 32×32, RGB with transparency, not interlaced. Otherwise the default Script icon is used.
- **Execution mode** — `Concurrent` (default) or `Serialized`. See "Choosing execution mode" below.
- **Execution group** — for `Serialized` only. Use a reverse-DNS-style string (e.g. `com.printops.routing`) to avoid colliding with other people's groups. For `Concurrent`, ignored.
- **Enable performance tuning** — `Yes` exposes `Idle after job (secs)` and (when concurrent) `Number of slots` to the flow designer.
- **Incoming connections** / **Require at least one** — most scripts say Yes/Yes (script processes incoming jobs).
- **Outgoing connections** — `No`, `One`, or `Unlimited`.
- **Connection type** — `Move` (single Move connection) or `Traffic light` (separate connections per Connection.Level — Success/Warning/Error). Pick based on whether the script needs to route by status.
- **Detailed info** — flags for traffic light: `Data success`/`Data warning`/`Data error` (data jobs go through these levels), `Log success`/`Log warning`/`Log error` (log dataset jobs).
- **Require at least one** outgoing — usually Yes.

### Choosing execution mode

| Use case | Mode |
|---|---|
| Pure metadata / file processing in Node, no external resource | Concurrent |
| Calls a CLI app / external service that's safe to run in parallel | Concurrent (with sane slot limits) |
| Wraps a CLI app that crashes or corrupts state when run in parallel | Serialized |
| Reads/writes a single shared file or database | Serialized |
| Uses `flowElement.getJobs` from `jobArrived` | Serialized (required) |

## Step 3 — Define Flow element properties

Select **Flow element properties** in the Declaration pane, then right-click → **Add** for each property the script reads via `getPropertyStringValue`. For each property, fill:

- **Tag** — internal name passed to `getPropertyStringValue`. Use a stable identifier (e.g. `MaxPages`, `OutputFolder`, `ApiToken`). Don't change later — scripts reference it by tag.
- **Name** — user-facing label in Switch Designer's Properties pane (e.g. `Maximum pages`, `API token`).
- **Tooltip** — short description shown on hover.
- **Inline editor** — primary editor (see Property editors table in `property-types.md`). Common choices: Single-line text, Number, No-yes list, Dropdown list (for fixed choices), Choose folder, Password.
- **Editor 1–5** — alternate editors. Useful combos: `[Inline: Single-line text] + [Editor 1: Single-line text with variables]` so the user can either type a literal or use Switch variables. Or `[Choose file] + [Select from library]` for property-set selection.
- **Default** — initial value.
- **Depends on master / Show if master equals** — for properties that should only appear when another property has a specific value. Master is the nearest preceding non-dependent property. One level only.
- **Validation** — usually `Standard`. Use `Custom` (and implement `validateProperties` in the script) if standard validation isn't enough.

For **Dropdown list**: also set **Dropdown items** (one per line).
For **Select from library / Select many from library / String list**: optional **Message** shown above the list dialog, plus the script must implement `getLibraryForProperty(s, flowElement, tag)` to populate the list.
For **OAuth 2.0 authorization**: set Application ID, Application password, Authorization URL, Token URL, Scope, Redirection port — see `property-types.md`.

## Step 4 — Define outgoing-connection properties (optional)

If the script needs custom per-connection settings (beyond the standard Connection.Level flags), select **Outgoing connections properties** in the Declaration pane and add properties the same way. Read in the script via `connection.getPropertyStringValue(tag)`.

## Step 5 — Save

`File > Save script`. SwitchScripter rewrites `<ScriptID>.xml` (and `manifest.xml`).

## Switch Designer setup (for the README)

After SwitchScripter is saved, the user goes to Switch Designer:

1. **Drag a Script element** onto the flow canvas.
2. **Properties pane → Script package property** → choose the `<ScriptID>.sscript` placeholder file inside the script folder. (For production: choose the packed `.sscript` from the production folder.)
3. The flow element now exposes:
   - The standard Script-element properties (Name, Description, Idle after job if performance tuning is on, Enable debug mode, etc.).
   - Each custom property defined in SwitchScripter, in the order they were added.
4. **Wire connections** — incoming folder, outgoing connections matching what was declared (one Move, or N Traffic-light with proper levels).
5. Configure each property with appropriate values or Switch variables.
6. Activate the flow.

## Reloading after declaration changes

If the user changes the script declaration in SwitchScripter (added/removed/renamed a property, changed connection rules) **after** the element is in a flow:

1. Deactivate the flow (or it's already inactive).
2. Right-click the Script element → **Reload script package**.
3. Re-bind any properties that may have shifted.

Code-only changes to `main.js` (i.e. transpiled `main.ts`) are reloaded automatically without flow deactivation.

## Debug mode (Switch Designer)

For iterative debugging:

1. Script element properties → **Enable debug mode** = Yes.
2. **Port** — default 9229.
3. **Debug entry points** — pick from `jobArrived`, `timerFired`, `httpRequestTriggeredAsync`, `findExternalEditorPath` (others are not debuggable).
4. Activate the flow → drop a job.
5. Switch waits for the debugger to attach.
6. In VS Code, open the script folder (its `.vscode/launch.json` is preconfigured) → press F5 → F5 again to step into `main.ts`.

**Always set Debug mode back to No** when done — otherwise the flow hangs waiting for a debugger.

## Switching to production

1. Pack: `SwitchScriptTool --pack <ScriptFolder> <ProductionFolder>`.
2. (Optional) Add `--password <pwd>` for protection. Enfocus cannot recover a lost password.
3. In Switch Designer, on the Script element, copy current property values via context menu → choose the new `.sscript` package → paste property values back.
