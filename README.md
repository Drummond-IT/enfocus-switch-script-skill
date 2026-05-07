# enfocus-switch-script

A [Claude Code](https://claude.com/claude-code) skill for writing, scaffolding, and modifying Node.js/TypeScript scripts for [Enfocus Switch](https://www.enfocus.com/en/switch) flow elements.

The skill teaches Claude how to use `SwitchScriptTool`, the Switch scripting API, the entry-point lifecycle, and the gotchas that bite script writers (forgotten `await`, post-`sendTo` rules, transpile-after-edit, property tag namespacing, etc.).

## What it does

When triggered, Claude will:

- Scaffold a new script folder (`SwitchScriptTool --create`)
- Write or edit `main.ts` (or `main.js`)
- Install third-party `node_modules` for the script
- Transpile TypeScript with `SwitchScriptTool --transpile`
- Pack a script folder into an `.sscript` package for production
- Generate a per-script `README.md` describing what to configure in SwitchScripter and what to wire up in Switch Designer

## What it doesn't do

- Edit `<ScriptID>.xml` or `manifest.xml` by hand (those are SwitchScripter-managed)
- Design or wire flows in Switch Designer (that's the user's manual half)
- Support Switch < 2020 Spring (no Node.js scripting on those versions)

## Installation

Clone into your Claude Code skills directory:

```bash
git clone https://github.com/johnvoncolln/enfocus-switch-script-skill.git ~/.claude/skills/enfocus-switch-script
```

Or, if you already have the folder:

```bash
cp -R enfocus-switch-script ~/.claude/skills/
```

The skill will be discovered automatically the next time you start Claude Code.

## Requirements

- macOS or Windows with [Enfocus Switch](https://www.enfocus.com/en/switch) installed
- `SwitchScriptTool` available on `PATH`. On macOS the default install puts it at `/Applications/Enfocus/Enfocus Switch/SwitchScriptTool/SwitchScriptTool` — symlink it:
  ```bash
  sudo ln -s "/Applications/Enfocus/Enfocus Switch/SwitchScriptTool/SwitchScriptTool" /usr/local/bin/SwitchScriptTool
  ```
- Switch 2023 Fall bundles Node.js with support for Node 12, 14, 16, and 18.

## How to invoke it

The skill triggers automatically on phrases like:

- "write a Switch script"
- "create a Switch script for X"
- "build a script for the print flow"
- "new Enfocus script"
- "update my Switch script"

It also activates when Claude is working inside an existing Switch script folder (a folder containing `main.ts`/`main.js` + `.sscript` placeholder + `manifest.xml`).

## Layout

```
enfocus-switch-script/
├── SKILL.md                          # The skill itself — workflow, scope, rules
├── references/
│   ├── api-reference.md              # Switch API: entry points, classes, enums
│   ├── property-types.md             # Property editor types and return values
│   ├── gotchas.md                    # Pitfalls (await, transpile, post-sendTo, etc.)
│   └── switchscripter-guide.md       # Manual SwitchScripter declaration setup
└── templates/
    └── README.template.md            # Per-script README scaffold
```

## License

MIT — see [LICENSE](LICENSE).

## Contributing

Issues and PRs welcome. If you find a Switch API quirk that isn't documented in `references/gotchas.md`, please open a PR adding it.
