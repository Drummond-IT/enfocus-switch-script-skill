# Switch scripting gotchas

Read this before writing the first non-trivial script. These are bugs that don't show up in linting and only surface at runtime in a flow.

## TypeScript transpile is mandatory after every edit

Switch executes `main.js`, not `main.ts`. After every change to `main.ts`:

```bash
SwitchScriptTool --transpile <ScriptFolder>
```

Symptom of forgetting: Switch logs `The file 'main.ts' is newer than 'main.js'` and runs the stale code. Never use a VS Code "Compile TypeScript" extension as a substitute — `SwitchScriptTool --transpile` ensures the same options as the production pack.

## Forgotten `await` is the most common bug

Almost every Switch API method returns a `Promise`. Forgetting `await` produces silent ordering bugs (e.g., `sendToSingle()` queued before `getPropertyStringValue` resolved). Use ESLint with `@typescript-eslint/no-floating-promises` and `require-await` enabled. Also, `async function jobArrived(...)` should declare the return as `Promise<void>` to make missing awaits visible.

## After `sendTo*`, no more job calls

Once you've routed a job (`sendToSingle/sendTo/sendToData/sendToLog/sendToNull/sendToChannel`), only other `sendTo*` or `Job.fail` are valid on that job. To route the same content to multiple destinations, **create a child job first**:

```ts
const child = await job.createChild(somePath);
await child.sendToData(Connection.Level.Success);
await job.sendToSingle(); // original — still valid
```

## You're responsible for cleaning up temp files

`flowElement.createJob(path)`, `job.createChild(path)`, and `job.createDataset(name, path, model)` consume a file/folder you created. Switch does **not** delete it after `sendTo*` or `fail`. Always remove temp paths in a `finally`:

```ts
try { /* ... */ } finally {
  await fs.remove(tmpPath).catch(() => {});
}
```

## Dependent properties throw

`getPropertyStringValue`/`getPropertyType` throw if the property's master condition isn't currently met. Either gate on the master value, or wrap in try/catch. `hasProperty` returns `false` in that case (handy as a guard).

## Properties with variables only resolve in `jobArrived`

Single-line text with variables, multi-line text with variables, condition with variables, and script expression properties are only available in `jobArrived`. Trying to read them in `timerFired`, `flowStartTriggered`, `flowStopTriggered`, etc., throws. `httpRequestTriggeredAsync` is fine for variables only when the entry point has a job context.

## Property values are strings — always

`getPropertyStringValue` returns `string | string[]`. Cast with `parseInt`/`parseFloat`/`Boolean` etc. yourself. `Number` editor in SwitchScripter is integer-only; there's no float editor.

## Tag namespacing

For `setPrivateData` and `Switch.setGlobalData(Scope.Global, ...)`, prefix tags with your company name to avoid collision. **Never use `EnfocusSwitch.*`** — that namespace is reserved for built-ins.

## Execution mode + Execution group

- **Concurrent**: multiple instances run in parallel, including parallel `jobArrived` for the same flow element. Synchronize shared resources yourself.
- **Serialized**: instances in the same execution group are never run in parallel. Use when the script controls a single external app or shared resource.

If unspecified, the execution group defaults to the script ID — fine for single-script serialization. For wide deployment, set a unique reverse-DNS group name (e.g. `com.printops.routing.serialize`) to avoid silent collisions with another developer's serialized script.

## `getJobs` has tight rules

- Max 10,000 jobs per call.
- **Once per entry point execution.**
- **Not allowed inside a concurrent `jobArrived` or `httpRequestTriggeredAsync`** — switch the script to serialized, or use `timerFired`.
- Cannot read private data or metadata of returned jobs in the same execution (writing is OK).

Typical pattern: collect job IDs in `jobArrived` (push to global data), then process in batches in `timerFired`.

## Debugging: turn it OFF when done

`Enable debug mode` on a Script element forces serialized execution, makes the element wait for a debugger to attach on the configured port (default 9229), and **silently hangs the flow** if you forget to disable it. Always set Debug mode = No before leaving a session. Only `jobArrived`, `timerFired`, `httpRequestTriggeredAsync`, and `findExternalEditorPath` can be debugged.

## Script auto-reloads, declaration changes don't

Code changes to `main.js` are picked up automatically — no need to deactivate/reactivate the flow. But changes to the script declaration (`<ScriptID>.xml`, modified via SwitchScripter) require deactivating the flow and right-click → "Reload script package" on the element before they're visible.

## Don't edit the XML by hand

`<ScriptID>.xml` and `manifest.xml` inside a script folder are SwitchScripter-managed. Editing them by hand can corrupt the script. If the user wants to add/rename/remove a property or change connection rules, they must do it in SwitchScripter.

## Don't rename the script folder

The script folder name **must** equal the Script ID. If it doesn't match, Switch can't load the script. To rename, change the Script ID in SwitchScripter and rename the folder to match.

## Script folders aren't included in flow exports

Switch's flow export bundles `.sscript` packages but not raw script folders. For deployment to another machine or for backups, pack to `.sscript` first:

```bash
SwitchScriptTool --pack <ScriptFolder> <DestFolder>
```

Script folders are dev-only.

## `node_modules` is bundled — slim before packing

Everything in `node_modules/` ends up in the `.sscript`. Before packing for production:

```bash
cd <ScriptFolder>
npm prune --production
```

Drops devDependencies. Use `--verbose` on `--pack` to see exactly what's included.

## Sync `fs` calls block `abort`

If an entry point hangs in `fs.readFileSync`/`fs.readdirSync`/etc. and Switch tries to time it out, the `abort` entry point can't run — Switch kills the executor instead. Stick to `fs.promises` / `fs-extra` async APIs.

## Password-protected packages can't be debugged

Debugging requires unobfuscated source. A password-protected `.sscript` is obfuscated — you can't attach a debugger. Develop and debug in script-folder form; only password-protect for distribution. Enfocus cannot recover a forgotten password.

## `setTimerInterval` — only inside `timerFired`

Calling `flowElement.setTimerInterval` from any other entry point throws. The default interval is 300s (5 min); to change it, set inside `timerFired` itself (typically the first thing).

## Webhook entry point timing

- `httpRequestTriggeredSync` runs in concurrent mode regardless of element setting; must finish in 60s; aborted otherwise (HTTP 524).
- Body limit 1 MB → HTTP 413 if exceeded.
- Per-element queue limit 10,000 → HTTP 429 if exceeded.
- `httpRequestTriggeredAsync` follows element concurrency/slot rules, fires after sync (if sync returned 2xx), and is the place to create jobs from webhook data.

## Script expressions are read-only

The `calculateScriptExpression` entry point and `Job.get(AccessLevel.ReadOnly)` are usable from script expressions, but no mutation is allowed. Same for third-party node modules: not supported in script expressions.

## Switch version compatibility

Scripts created in version N work in version N or higher, but not lower. To support multiple Switch versions, develop with the SwitchScriptTool / SwitchScripter of the **oldest** version you want to support.

This machine: Switch 2023 Fall (Node.js 12, 14, 16, 18 supported).

## macOS Intel vs ARM

For scripts that ship native binaries: from Switch 2021 Fall onward, Switch picks Intel Node.js on Intel and ARM Node.js on ARM (Node 18). Ship binaries for both, or rely on Rosetta (slower; deprecated path).
