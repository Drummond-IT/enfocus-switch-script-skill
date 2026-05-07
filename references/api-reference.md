# Switch Scripting API reference

Concise reference for the Node.js Switch scripting API. All methods marked `async` return `Promise<T>` — always `await`. The classes `Switch`, `FlowElement`, `Job`, `Connection`, `PdfDocument`, `PdfPage`, `ImageDocument`, `XmlDocument`, `XmpDocument`, `HttpRequest`, `HttpResponse`, and the enums (`AccessLevel`, `LogLevel`, `Connection.Level`, `DatasetModel`, `PropertyType`, `Scope`, `EnfocusSwitchPrivateDataTag`, `Priority`) are globally available — do not import them.

## Entry points

A script must define at least one of `jobArrived` or `timerFired`. All entry points are `async` in practice. The first two args are always `s: Switch` and `flowElement: FlowElement`.

| Entry point | Signature | When |
|---|---|---|
| `jobArrived` | `(s, flowElement, job: Job)` | Each new job in the input folder |
| `timerFired` | `(s, flowElement)` | At regular intervals (default 300s; change with `flowElement.setTimerInterval(seconds)` — only callable inside `timerFired`) |
| `flowStartTriggered` | `(s, flowElement)` | Once on flow activation, before any `jobArrived`/`timerFired`. Not debuggable. |
| `flowStopTriggered` | `(s, flowElement)` | On flow deactivation, after other entry points stopped. Not debuggable. |
| `getLibraryForProperty` | `(s, flowElement, tag: string) → string[]` | Populates a "Select from library" property's dialog |
| `getLibraryForConnectionProperty` | `(s, flowElement, c: Connection, tag) → string[]` | Same, but for a connection property |
| `validateProperties` | `(s, flowElement, tags: string[]) → {tag, valid}[]` | Custom property validation. Must return one entry per input tag. |
| `validateConnectionProperties` | `(s, flowElement, c, tags) → {tag, valid}[]` | Custom connection-property validation |
| `findExternalEditorPath` | `(s, flowElement, tag: string) → string` | Resolves the external editor application path for an "External editor" property |
| `httpRequestTriggeredSync` | `(request: HttpRequest, args: any[], response: HttpResponse, s: Switch)` | Webhook arrived — set the HTTP response. Concurrent. Must finish in 60s. |
| `httpRequestTriggeredAsync` | `(request, args, s, flowElement)` | Async webhook handler — fires after sync (if sync returned 2xx). Subject to element concurrency/slots. |
| `abort` | `(s, flowElement, job, abortData)` | Time-out cleanup for a long-running entry point. 60s max. Use `s.setAbortData(...)` from the main entry point. |
| `calculateScriptExpression` | `(s, flowElement, job) → string \| number \| boolean` | Script-expression entry point — read-only context only. |

## Switch class

| Method | Notes |
|---|---|
| `async getGlobalData(scope: Scope, tag: string \| string[], lock?: boolean)` | Returns value or list. With `lock=true`, holds a lock until next `setGlobalData`. Date instances come back as strings. |
| `async setGlobalData(scope, tag, value)` / `async setGlobalData(scope, [{tag, value}, ...])` | Value can be string/number/boolean/object/array/Date/null. Releases any held lock. |
| `async removeGlobalData(scope, tag \| tags)` | |
| `async httpRequestSubscribe(method: HttpRequest.Method, path: string, args: any[])` | Subscribe to webhook — call from `flowStartTriggered` or `timerFired`. |
| `async httpRequestUnsubscribe(method, path)` | |
| `setAbortData(data: any)` | Sets data passed to `abort` entry point if the current entry point is timed out. |
| `static Switch.tr(str)` | Marks a string for translation (used by `--generate-translations`). |

`Scope` values: `FlowElement` (this instance only), `FlowElements` (same script in same flow), `Element` (same script across all flows), `Flow` (all elements in same flow), `Global` (everything). **Avoid `Global` unless you really need it; namespace tag names with the company name when you do.**

## FlowElement class

| Method | Notes |
|---|---|
| `getName()` / `getFlowName()` | Names |
| `getOutConnections(): Connection[]` | All outgoing connections |
| `setTimerInterval(seconds)` | **Only callable from `timerFired`.** |
| `async getPropertyStringValue(tag): Promise<string \| string[]>` | Returns a string for single-line editors, string[] for multi-line. **Throws** if the tag is unknown OR if it's a dependent property whose master doesn't currently match — wrap in try/catch or check the master first. Properties using variables/script expressions only resolve in `jobArrived`. |
| `getPropertyType(tag): PropertyType` | Useful when a property has multiple editors, to know what kind of value came back. |
| `hasProperty(tag): boolean` | False for dependent properties whose master doesn't currently match. |
| `getPropertyDisplayName(tag): string` | Untranslated user-facing name; for log messages. |
| `failProcess(message, messageParam?)` | Fatal error — puts element in problem state. Outside `jobArrived`/`timerFired` it just logs. |
| `async log(level: LogLevel, message, params?)` | Log at `Error/Warning/Info/Debug` level. Use `%1`, `%2`, ... placeholders in `message` and pass values in `params`. |
| `async createJob(path): Promise<Job>` | New job from existing file/folder — does not inherit input job. Valid in `jobArrived`/`timerFired`/`httpRequestTriggeredAsync`. **Caller must clean up the source path after sending/failing.** |
| `getPluginResourcesPath()` | Scripted plug-in resources folder (apps only). For regular scripts in test, returns the script folder/package path. |
| `async getJobs(ids: string[]): Promise<Job[]>` | Up to 10,000 jobs. **Once per entry point execution. Not callable from concurrent `jobArrived`/`httpRequestTriggeredAsync`.** Cannot read private data/metadata of returned jobs in the same execution. |
| `subscribeToChannel(channelId, backingFolderPath)` | **Only callable from `flowStartTriggered`.** Sets up cross-element job channel. |

## Job class

### Identity & content
| Method | Notes |
|---|---|
| `getName(includeExtension = true)` | Job filename without the `_<id>_` prefix |
| `getId()` | The unique job ID (the prefix without underscores) |
| `isFile()` / `isFolder()` | |
| `async get(accessLevel: AccessLevel)` | Path on disk. `ReadOnly` throws if you modified content; `ReadWrite` uploads modifications on `sendTo*`. |

### Logging & failure
| Method | Notes |
|---|---|
| `async log(level, message, params?)` | Per-job log entry. Use `'%1'` placeholders. |
| `fail(message, params?)` | Move job to Problem jobs and log. After `fail`, can't do further routing for this job. Created child jobs are also not moved to Problem jobs. |
| `async processLater(seconds)` | Re-run `jobArrived` for this job later (min 10s when called from `jobArrived`). |

### Routing — once routed, no further calls allowed except other `sendTo*` or `fail`
| Method | Notes |
|---|---|
| `async sendToNull()` | Drop without output |
| `async sendToSingle(newName?)` | Send to the single Move connection (rename optional) |
| `async sendTo(connection, newName?)` | Send to a specific Move connection |
| `async sendToData(level: Connection.Level, newName?)` | Send via traffic-light data connection |
| `async sendToLog(level, model: DatasetModel, newName?)` | Send via traffic-light log connection. Model = `XML/JDF/XMP/JSON/Opaque`. |
| `async sendToChannel(channelId, newName?)` | Cross-element via channel (paired with `subscribeToChannel`) |

### Children & datasets
| Method | Notes |
|---|---|
| `async createChild(path): Promise<Job>` | Like `createJob` but inherits parent's history, private data, metadata. Caller cleans up source path. |
| `async listDatasets()` | `[{name, model, extension}]` |
| `async getDataset(name, accessLevel): Promise<string>` | Path to the dataset file |
| `async createDataset(name, filePath, model: DatasetModel)` | Caller cleans up source path. |
| `async removeDataset(name)` | Throws if not found |

### Private data
| Method | Notes |
|---|---|
| `async getPrivateData(tag \| tags?)` | One value, list of pairs, or all if no arg |
| `async setPrivateData(tag, value)` / `async setPrivateData([{tag, value}, ...])` | Namespace your tags with company prefix; never use `EnfocusSwitch.*` |
| `async removePrivateData(tag \| tags)` | |

Built-in tags (see `EnfocusSwitchPrivateDataTag`): `hierarchy`, `emailAddresses`, `emailBody`, `userName`, `userFullName`, `userEmail`, `origin` (read-only), `initiated`, `submittedTo`, `state`.

### Priority
| Method | Notes |
|---|---|
| `getPriority(): number` / `setPriority(n)` | `Priority.Low/BelowNormal/Normal/AboveNormal/High` are the constants (-100000 … 100000). Any number works. |

## Connection class

| Method | Notes |
|---|---|
| `getName()` / `getId()` | |
| `async getPropertyStringValue(tag)` / `getPropertyType(tag)` / `hasProperty(tag)` / `getPropertyDisplayName(tag)` | Same semantics as on FlowElement, but for connection properties. |

## PdfDocument & PdfPage

`PdfDocument.open(path)` → instance with `close()`, `getNumberOfPages()`, `getPDFVersion()`, `getPDFXVersion()`, `getSecurityMethod()`, `getPage(n=1): PdfPage`, `getXMP(): XmpDocument`. Each method also exists as a static taking `(path, ...)` for one-shot calls. Always wrap in try/catch and `close()` in a finally.

`PdfPage` methods: `getRotation()`, `getScaling()`, `getPageLabel()`, `getHeight(effective?)`, `getWidth(effective?)`, plus `get<Box>Height/Width(effective?)` for `MediaBox/CropBox/BleedBox/TrimBox/ArtBox`. With `effective=true`, page rotation and scaling are applied.

## ImageDocument

`ImageDocument.open(path)` → instance with `close()`, `getWidth()`, `getHeight()`, `getColorMode()`, `getColorSpace()`, `getICCProfile()` (Photoshop), `getSamplesPerPixel()`. Same methods exist as statics. Supports JPEG, TIFF, PNG.

`ColorMode`: `Bitmap | Gray | IndexedColor | RGB | CMYK | Multichannel | Duotone | LabColor | Unknown`.
`ColorSpace`: `SRGB ('sRGB') | Uncalibrated`.

## XmlDocument

`XmlDocument.open(path)` or `XmlDocument.parse(xmlString)`. Then `evaluate(xpath, prefixMap?)` — XPath 1.0. `getDefaultNSMap()` returns a map including `default_switch_ns` / `dn` for the document's default namespace. **XPath has no concept of a default namespace — always specify a prefix in expressions.**

## XmpDocument

`XmpDocument.open(path)` (XMP-only files) or via `pdfDocument.getXMP()`. `evaluate(xmpLocationPath, additionalPrefixMap?)` — XMP location-path subset of XPath. Returns boolean/number/string/`undefined`. Standard XMP namespace prefixes (`dc`, `pdf`, `xmp`, `xmpMM`, `tiff`, `exif`, `photoshop`, `Iptc4xmpCore`, etc.) are pre-registered. For language alternatives use `[@xml:lang="x-default"]` in the path.

## HttpRequest / HttpResponse

`HttpRequest`: `method`, `path`, `query: {...}`, `headers: {...}`, `remoteAddress`, `body?: ArrayBuffer`, plus `getBodyAsString()`. `HttpResponse`: `setStatusCode(n)`, `setHeader(name, value)`, `setBody(data)`. Webhook URL = `<protocol>://<switch-host>:<port>/scripting<path>` where `/scripting` is hard-coded. Body limit 1 MB. Queue limit 10,000 per element.

## Enums quick-reference

| Enum | Values |
|---|---|
| `AccessLevel` | `ReadOnly`, `ReadWrite` |
| `LogLevel` | `Error`, `Warning`, `Info`, `Debug` (Debug only shown if "Log debug messages" is on) |
| `Connection.Level` | `Error`, `Warning`, `Success` |
| `DatasetModel` | `Opaque`, `XML`, `XMP`, `JDF`, `JSON` |
| `PropertyType` | `Literal`, `Number`, `Date`, `HoursAndMinutes`, `Boolean`, `String`, `FilePath`, `FolderPath`, `FileType`, `FolderPattern`, `Regex`, `OAuthToken` |
| `Scope` | `FlowElement`, `FlowElements`, `Element`, `Flow`, `Global` |
| `HttpRequest.Method` | `POST`, `PUT`, `DELETE` |
| `Priority` | `Low=-100000`, `BelowNormal=-10000`, `Normal=0`, `AboveNormal=10000`, `High=100000` |

All enums are also accessible as `EnfocusSwitch.<EnumName>` if needed outside `main.ts`/`main.js`.

## Common patterns

### Read input job → produce one output

```ts
async function jobArrived(s: Switch, flowElement: FlowElement, job: Job): Promise<void> {
  try {
    const inputPath = await job.get(AccessLevel.ReadOnly);
    // ... read or process inputPath ...
    await job.sendToSingle();
  } catch (e: any) {
    job.fail("Failed to process %1: %2", [job.getName(), e.message]);
  }
}
```

### Read property, then route on a traffic light

```ts
async function jobArrived(s: Switch, flowElement: FlowElement, job: Job): Promise<void> {
  try {
    const threshold = parseInt(await flowElement.getPropertyStringValue("Threshold") as string, 10);
    const numPages = PdfDocument.getNumberOfPages(await job.get(AccessLevel.ReadOnly));
    if (numPages > threshold) {
      await job.sendToData(Connection.Level.Warning);
    } else {
      await job.sendToData(Connection.Level.Success);
    }
  } catch (e: any) {
    job.fail("Failed: %1", [e.message]);
  }
}
```

### Modify the job (read/write)

```ts
import * as fs from "fs-extra";
async function jobArrived(s: Switch, flowElement: FlowElement, job: Job): Promise<void> {
  try {
    const path = await job.get(AccessLevel.ReadWrite);
    await fs.writeFile(path, await fs.readFile(path, "utf8") + "\n// touched");
    await job.sendToSingle(); // upload happens here
  } catch (e: any) {
    job.fail("Failed: %1", [e.message]);
  }
}
```

### Spawn a child job from a generated artifact

```ts
import * as path from "path";
import * as os from "os";
import * as fs from "fs-extra";

async function jobArrived(s: Switch, flowElement: FlowElement, job: Job): Promise<void> {
  const tmp = path.join(os.tmpdir(), `${job.getId()}-report.xml`);
  try {
    await fs.writeFile(tmp, "<report/>");
    const child = await job.createChild(tmp);
    await child.sendToLog(Connection.Level.Success, DatasetModel.XML);
    await job.sendToData(Connection.Level.Success);
  } catch (e: any) {
    job.fail("Failed: %1", [e.message]);
  } finally {
    await fs.remove(tmp).catch(() => {});  // caller responsibility
  }
}
```

### Dependent property — reading safely

```ts
let mode: string;
try {
  mode = await flowElement.getPropertyStringValue("AdvancedMode") as string;
} catch {
  mode = ""; // master condition isn't met → property invisible → throws
}
```
