# Property editors and types

A flow-element property is configured in SwitchScripter and read by the script via `flowElement.getPropertyStringValue("Tag")`. Same for connection properties via `connection.getPropertyStringValue("Tag")`. Values are always strings (or `string[]` for list editors); the script casts as needed.

Each property has one **Inline editor** (rendered directly in the Properties pane) and up to five additional **Editor** dialogs (1–5). At least one of these must be set (otherwise SwitchScripter falls back to single-line text). Specifying the same editor twice is meaningless except for the External editor (multiple allowed).

## Inline editors

| Inline editor | Returns | Notes |
|---|---|---|
| Single-line text | `string` | Plain text |
| Password | `string` | Same as single-line, masked input |
| Number | `string` | Decimal integer only — no floating point in SwitchScripter; cast with `parseInt` |
| Hours and minutes | `string` "hh:mm" | 4-digit + colon |
| No-yes list | `"No" \| "Yes"` | Compare exact strings |
| Dropdown list | `string` | One of the predefined dropdown items |

## Other editors (dialog-based)

| Editor | Returns |
|---|---|
| Literal | Fixed string from a predefined list (`"Default"`, `""` for None, `"Automatic"`, `"No Files"`, `"All Files"`, `"All Other Files"`, `"No Folders"`, `"All Folders"`) |
| Choose file | Absolute file path |
| Choose folder | Absolute folder path |
| Regular expression | Regex source string |
| File type | Filename pattern(s) for a chosen standard file type |
| Select from library | One library item (resolved by `getLibraryForProperty`) |
| Multi-line text | One string with embedded newlines |
| Script expression | Result of expression at runtime, coerced to string |
| Single-line text with variables | One string with `[Variables]` resolved |
| Multi-line text with variables | Same, multi-line |
| Condition with variables | `"true"` or `"false"` |
| File patterns | `string[]` of patterns |
| Folder patterns | `string[]` of patterns |
| File types | `string[]` of patterns |
| String list | `string[]` |
| Select many from library | `string[]` |
| External editor | File path to a third-party-edited property set |
| OAuth 2.0 authorization | OAuth access token, or empty string if not authorized yet |

## PropertyType enum (returned by `getPropertyType`)

When a property allows multiple editors, ask `getPropertyType("Tag")` to know what came back.

| `PropertyType.*` | Editor that produces it |
|---|---|
| `Literal` | Literal |
| `Number` | Inline: Number |
| `Date` | Inline: Date / Date and time |
| `HoursAndMinutes` | Inline: Hours and minutes |
| `Boolean` | Inline: No/Yes list, or Condition with variables |
| `String` | Single-line text, Password, Multi-line text, text-with-variables, Script expression, Dropdown list, Select from library/Select many from library, String list, Multi-line text with variables |
| `FilePath` | Choose file, External editor |
| `FolderPath` | Choose folder |
| `FileType` | File type / File types / File patterns |
| `FolderPattern` | Folder patterns |
| `Regex` | Regular expression |
| `OAuthToken` | OAuth 2.0 authorization |

## Validation

Each property has a Validation setting:

- **None** — no validation.
- **Standard** — Switch's built-in validation based on the property's editors (typed: file path must exist, regex must compile, integer must be digits, etc.). Properties with variables/script expressions are validated at runtime, not design time.
- **Custom** — calls `validateProperties` (or `validateConnectionProperties`) entry point with the tags. Must return `[{tag, valid: boolean}]` for every input tag — missing entries are treated as invalid.
- **Standard and custom** — both run; custom only runs if standard passed.

## Dependent properties

A property can have "Depends on master = Yes" plus "Show if master equals = X" — it's only visible when the nearest preceding non-dependent property has value `X`. Only one level of dependency is allowed.

**Gotcha:** `getPropertyStringValue`, `getPropertyType`, and `hasProperty` all behave differently when the property is hidden (master condition not met):
- `getPropertyStringValue` and `getPropertyType` **throw**.
- `hasProperty` returns **false**.

Wrap reads in try/catch, or check the master value first:
```ts
if (await flowElement.getPropertyStringValue("Mode") === "Advanced") {
  const advancedOpt = await flowElement.getPropertyStringValue("AdvancedOption");
}
```

## Where properties can be read

| Entry point | `getPropertyStringValue` for ... |
|---|---|
| `jobArrived` | Any property (including those with variables / script expressions / multi-line text with variables) |
| All other entry points (`timerFired`, `flowStartTriggered`, `flowStopTriggered`, etc.) | Static properties only — properties with variables or script expressions throw |

Property values in Node.js scripting are evaluated at script start, not at every `getPropertyStringValue` call. This differs from legacy scripting and means a variable that pulls from a database doesn't re-evaluate within one execution.

## OAuth 2.0 properties

Recommended setup in SwitchScripter:
- Inline editor: None
- Editor 1: OAuth 2.0 authorization
- Editors 2–5: None

Set Application ID, Application password (or `None` if not required), Authorization URL, Token URL, Scope (or `None`), and Redirection port (specific port, comma-separated list, or `Automatic`). Switch refreshes tokens automatically; `getPropertyStringValue` returns a valid access token.

## External editor properties

For property sets edited by an external GUI app:
- Application: absolute or relative path. If empty, `findExternalEditorPath` is invoked.
- Arguments for new / Arguments for edit: command-line arg strings — `%1` is the property-set file path, `%2` is the locale (e.g. `enUS`).
- File format: `Switch` (XML format Switch parses for sub-files to bundle in flow exports) or `Custom`.
- Editor name, Value overlay (label shown after editing).

External app must exit 0 on success, non-zero on cancel/error.
