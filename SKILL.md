---
name: enfocus-switch-script
description: Use this skill when the user wants to write, scaffold, or modify a Node.js/TypeScript script for an Enfocus Switch flow element — phrases like "write a Switch script", "create a Switch script for X", "build a script for the print flow", "new Enfocus script", "update my Switch script", or any work inside an existing Switch script folder (folder containing main.ts/main.js + a .sscript placeholder + manifest.xml). Scope is the SCRIPT ONLY — scaffolding the folder via SwitchScriptTool, writing main.ts, transpiling, and packing for production. Do NOT use this skill for designing or wiring Switch flows in Switch Designer; flow integration is done manually by the user.
---

# enfocus-switch-script

Help the user create and develop Enfocus Switch scripts (the Node.js/TypeScript code that runs inside a Switch `Script element`). This skill knows the SwitchScriptTool CLI, the Switch scripting API, the entry-point lifecycle, and the gotchas that bite script writers.

## Scope — what this skill does and does not do

**In scope:**
- Scaffold a new script folder (`SwitchScriptTool --create`).
- Write/edit `main.ts` (preferred) or `main.js`.
- Install third-party `node_modules` for the script.
- Transpile TypeScript with `SwitchScriptTool --transpile`.
- Pack a script folder into an `.sscript` package for production.
- Generate a `README.md` inside the script folder describing what the user has to configure in SwitchScripter (script declaration, properties, connections) and what to wire up in Switch Designer (incoming/outgoing connections, user-facing properties on the flow element).

**Out of scope — leave to the user:**
- Editing the script declaration XML (must be done in the SwitchScripter GUI app — never edit `<ScriptID>.xml` or `manifest.xml` by hand).
- Designing or activating a Switch flow.
- Dropping the script onto the canvas, choosing the placeholder `.sscript`, or wiring connections in Switch Designer.

The script and the flow integration are two separate jobs. This skill produces a self-contained script folder + README; the user takes it from there.

## Local environment

`SwitchScriptTool` is installed at `/Applications/Enfocus/Enfocus Switch/SwitchScriptTool/SwitchScriptTool` and symlinked to `/usr/local/bin/SwitchScriptTool`, so the bare command `SwitchScriptTool` works from any shell. Verify with `SwitchScriptTool --version` if you need to confirm. Bundled Node.js for Switch 2023 Fall supports Node 12, 14, 16, and 18.

## Workflow for a new script

Follow these steps in order. Do each one only when it's needed for the current task.

### 1. Confirm the script ID and target folder

Ask the user (in one short turn) for:
- **Script ID** — short identifier. Allowed chars: letters, digits, hyphens, underscores. The script folder will be named exactly the Script ID and **must not be renamed afterward** (Switch keys off the folder name). Reverse-domain isn't required here, but if the user says nothing, suggest something descriptive like `JobRouter` or `OrderEnricher`.
- **Parent folder** — where the dev folder should live. Suggest a `dev/` subfolder so production `.sscript` packages can sit in a sibling `production/` folder later.
- **Language** — default to **TypeScript** (better autocomplete via `@types/switch-scripting`, catches bugs early). Only use plain JavaScript if the user explicitly asks.

If any of these are obvious from context (the user already gave you a path, etc.), don't ask — just proceed.

### 2. Scaffold the folder with SwitchScriptTool

```bash
# TypeScript (default — preferred):
SwitchScriptTool --create <ScriptID> <PathToParentFolder>

# JavaScript (only if user requested):
SwitchScriptTool --create <ScriptID> <PathToParentFolder> --JavaScript
```

This creates `<PathToParentFolder>/<ScriptID>/` with:
- `manifest.xml`, `<ScriptID>.xml`, `<ScriptID>.sscript` (placeholder — DO NOT edit by hand)
- `main.ts` (or `main.js`) with an empty `jobArrived` stub
- `.vscode/launch.json` and `.vscode/switch.code-snippets`
- `package.json`, `tsconfig.json`
- `node_modules/@types/node` and `node_modules/@types/switch-scripting`

### 3. Write `main.ts`

Read the file first, replace the body with the actual script. Key rules — **read `references/gotchas.md` before writing** if you're not already familiar:

- Import the right Node modules at the top. The Switch types (`Switch`, `FlowElement`, `Job`, `LogLevel`, `AccessLevel`, `Connection`, etc.) are globally available — don't import them.
- All Switch API calls returning `Promise` must be `await`ed. Forgetting `await` is the most common bug; an ESLint config with `@typescript-eslint/no-floating-promises` catches it.
- Type the entry-point parameters: `async function jobArrived(s: Switch, flowElement: FlowElement, job: Job)`.
- Always wrap the body of `jobArrived` (and other entry points) in a `try / catch` and call `job.fail(...)` (or `flowElement.failProcess(...)` for non-job entry points) on error — uncaught exceptions hang or stick the element.
- Read the input job path with `await job.get(AccessLevel.ReadOnly)` (or `ReadWrite` if modifying). After modifying, `sendTo*` automatically uploads the change.
- After any `sendTo*` call, **no further calls on that Job are allowed** except other `sendTo*` or `Job.fail`. If the script needs to route to multiple connections, use `job.createChild(path)` to spawn child jobs first.
- Property values are always strings — cast as needed. Use `await flowElement.getPropertyStringValue("Tag")`. The `Tag` you pass here is the `Tag` field set in SwitchScripter (not the user-facing `Name`).

For the API surface (entry points, methods on `Switch`/`FlowElement`/`Job`/`Connection`/`PdfDocument`/`XmpDocument`/`XmlDocument`/`ImageDocument`/`HttpRequest`/`HttpResponse`, plus enums), see `references/api-reference.md`.

For property editor types and what gets returned, see `references/property-types.md`.

### 4. Install third-party node modules (if needed)

```bash
cd <ScriptFolder>
npm install <package> --save
```

Modules in `node_modules/` are bundled into the script package. To slim the package before production, run `npm prune --production` to drop devDependencies.

### 5. Transpile (TypeScript only)

```bash
SwitchScriptTool --transpile <ScriptFolder>
```

This produces `main.js` and `main.js.map` next to `main.ts`. **You must re-transpile after every edit to `main.ts`** before Switch will pick up changes — Switch executes `main.js`, not `main.ts`. The Visual Studio Code "Compile TypeScript" extensions are not equivalent; always use `SwitchScriptTool --transpile` so the same options are applied as in production packing.

If the user is iterating, offer to run transpile automatically after each `main.ts` edit.

### 6. Generate the README

Once the script is ready (or as soon as it's clear what properties and connections it needs), write a `README.md` at the root of the script folder using the template at `templates/README.template.md`. The README is the user's single reference for what to configure in SwitchScripter and what to wire up in Switch Designer. Fill in:

- Brief description of what the script does.
- **Script declaration / SwitchScripter setup**: Display name, Execution mode (concurrent vs serialized — see `references/gotchas.md` for which to pick), Incoming connections, Outgoing connections (count + type: Move or Traffic light, with success/warning/error levels filled in if traffic light), and each Flow-element property the script reads via `getPropertyStringValue` — Tag, Name, Inline editor type, Default value, validation.
- **Switch Designer / flow setup**: any flow-element user properties the user will see (these match the SwitchScripter setup), suggested incoming and outgoing connection wiring, any third-party app the script shells out to.
- **Iteration notes**: that they need to re-transpile after every `main.ts` edit, that the script auto-reloads in an active flow (no flow restart needed), and that adding/removing/renaming properties in SwitchScripter requires "Reload script package" on the element.

### 7. Pack for production (when the user is ready to deploy)

```bash
# unprotected:
SwitchScriptTool --pack <ScriptFolder> <DestinationFolder>

# password-protected (Enfocus cannot recover the password — make sure the user remembers it):
SwitchScriptTool --pack <ScriptFolder> <DestinationFolder> --password <Password>

# list contents to verify:
SwitchScriptTool --list <DestinationFolder>/<ScriptID>.sscript
```

Don't pack until the user explicitly asks. Mention that `.sscript` is the file they reference from the Script element in Switch Designer for production use; script folders are dev-only and aren't included in flow exports.

## Workflow for an existing script

Detect this by the presence of a `<ScriptID>.sscript` placeholder + `manifest.xml` in the working directory. In that case:
- Skip scaffolding.
- Read `main.ts` (or `main.js`) and any existing `README.md` to understand current behavior.
- Make the requested changes.
- Re-transpile if `main.ts` was touched.
- Update the README if properties/connections changed.

If the user starts from a `.sscript` package (no folder), unpack it first:
```bash
SwitchScriptTool --unpack <PathToSscript> <DestinationFolder> [--password <Password>]
```

## When to pull in references

Read these on demand — don't load all of them up front:
- `references/api-reference.md` — entry-point signatures and class methods. Pull when writing or modifying script logic.
- `references/property-types.md` — property editor types, validation behavior, and what string each editor returns. Pull when defining or reading flow-element properties.
- `references/gotchas.md` — pitfalls (forgotten `await`, transpile-after-edit, post-`sendTo` rules, debug-mode cleanup, tag namespacing, dependent-property exceptions, abort/timeouts). **Skim this before writing the first non-trivial script.**
- `references/switchscripter-guide.md` — step-by-step for the user's manual SwitchScripter declaration setup. Pull when writing the per-script README.
- `templates/README.template.md` — the README scaffold. Read when generating step 6.

## Things to push back on

- User asks you to edit `<ScriptID>.xml` or `manifest.xml` directly: refuse and explain those are SwitchScripter-managed; their declaration changes need to happen in the GUI.
- User asks you to "set up the flow" or "wire it into Switch": that's the manual half — produce/update the README instead.
- User wants password-protected dev iteration: explain that script folders don't support passwords (only `.sscript` packages do), and that debugging a password-protected package isn't possible.
- User is on Switch < 2020 Spring: Node.js scripting isn't supported; they'd need legacy JavaScript/VBScript, which is out of scope.
