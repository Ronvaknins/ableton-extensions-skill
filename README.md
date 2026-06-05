# Ableton Extensions Skill

An **Agent Skill** that teaches an AI coding agent how to scaffold, write, build, and package **Ableton Live extensions** with the [Ableton Extensions SDK](https://www.ableton.com/en/live/extensions) (`@ableton-extensions/sdk`, TypeScript).

When this skill is installed, you can ask your agent things like *"create an Ableton extension that transposes the selected MIDI clip up an octave"* and it will produce a correct, buildable project using the real SDK APIs, conventions, and tooling — instead of guessing.

It follows the open **Agent Skills** convention — a `SKILL.md` file plus supporting reference docs — so any agent that understands that format can use it. Installation instructions below are for [Claude Code](https://claude.com/claude-code); adapt the path for other agents that load skills.

---

## What it does

The skill gives the agent accurate, version-pinned knowledge of the SDK so it can:

- **Scaffold** a new extension project (manifest, build config, entry point).
- **Write** extension code using the real object model — tracks, clips, MIDI notes, devices, tempo, scenes, selections.
- **Add UI** — right-click context-menu actions, modal dialogs (WebView HTML), and progress dialogs.
- **Build & package** — bundle with esbuild and produce a distributable `.ablx` archive.
- **Stay within the SDK's boundaries** — it knows extensions are an *offline editing/automation* layer, **not** a real-time MIDI/audio effect (that's a job for Live's built-in MIDI effects or Max for Live).

---

## How it works

Agent Skills use **progressive disclosure**: a small always-loaded entry file points to larger references that the agent reads only when relevant. This keeps context lean.

```
ableton-extensions-skill/
├── SKILL.md                  # entry point — always loaded; frontmatter triggers the skill
├── README.md                 # this file (human-facing; not loaded by the agent)
└── reference/                # loaded on demand, when the agent needs the detail
    ├── scaffolding.md        # create a project, manifest/package/build configs, CLI, .env, debugging, packaging
    ├── api.md                # the full object model: ExtensionContext, Song/Track/Clip/MidiClip, enums, scopes
    └── recipes.md            # copy-paste code for context menus, dialogs, note/clip edits, transactions
```

- **`SKILL.md`** has YAML frontmatter (`name`, `description`). The agent reads the `description` to decide *when* to activate the skill (e.g. a request mentions an Ableton extension, MIDI clip editing, a context-menu action). Its body covers the entry-point shape, core concepts, and the workflow.
- **`reference/*.md`** are pulled in only when needed — e.g. the agent opens `reference/recipes.md` when writing a modal dialog, or `reference/scaffolding.md` when setting up the build.

---

## What you need to download / install

| Requirement | Why | Where |
|---|---|---|
| **Ableton Extensions SDK** (`extensions-sdk-<version>.zip`) | the SDK library, CLI, scaffolder, docs, examples | [ableton.com/en/live/extensions](https://www.ableton.com/en/live/extensions) (beta program) |
| **Ableton Live beta** with Extensions support | to actually load/run extensions | from the Ableton beta / Centercode |
| **Node.js ≥ 24.14.1** | build tooling (esbuild, tsx, TypeScript) | [nodejs.org](https://nodejs.org) |
| **A code editor** (VS Code recommended) | optional F5 debug integration | — |

> **Note:** Ableton Extensions are a **beta** feature. Access to the SDK and an Extensions-capable Live build is currently via Ableton's beta program — start at the link above.

This skill is built against **SDK `1.0.0-beta.0`** (API version `1.0.0`). The reference docs point at a local unpacked SDK at `C:\DEV\extensions-sdk-1.0.0-beta.0` (containing `docs/`, `api/`, `examples/`, and the `.tgz` tool packages). If you unpack the SDK elsewhere, update those absolute paths in `SKILL.md` / `reference/scaffolding.md` to match.

---

## Installing the skill

Skills are discovered from a `skills/` directory. Copy this folder into one (rename it to match the skill name, `ableton-extensions`). The directory that contains `SKILL.md` becomes the skill.

**Claude Code — personal (available in every project):**
```sh
# Windows (PowerShell)
Copy-Item -Recurse C:\DEV\ableton-extensions-skill "$env:USERPROFILE\.claude\skills\ableton-extensions"
```

**Claude Code — project-scoped (shared with a repo via git):**
```sh
# from the project root
mkdir -p .claude/skills
cp -r /path/to/ableton-extensions-skill .claude/skills/ableton-extensions
```

After copying, the skill activates automatically when your request matches its description; in Claude Code you can also invoke it explicitly with `/ableton-extensions`.

**Other agents:** if your agent supports the Agent Skills (`SKILL.md`) format, place this folder in that agent's skills directory instead. The skill content itself is agent-agnostic — only the install location differs.

---

## Using it

Once installed, just describe what you want. Examples:

- *"Scaffold a new Ableton extension called clip-doubler."*
- *"Add a right-click action on MIDI clips that humanizes note velocities."*
- *"Make a modal dialog to pick a chord type, then turn each note in the selected clip into that chord."*
- *"Build and package my extension into an .ablx."*

The agent will follow the SDK's canonical patterns: an `activate()` entry point, `initialize(activation, "1.0.0")`, `registerCommand` + `registerContextMenuAction`, handle resolution with `getObjectFromHandle`, and `withinTransaction` for batched, single-undo edits.

### Worked example (validated)

This skill was used to build a working **One Note Chords** extension (right-click a MIDI clip → choose a chord in a dialog → every note becomes that chord). The full toolchain was verified end-to-end: `npm install` → `npm run build` (typecheck + esbuild bundle) → `npm run package` (`.ablx`).

---

## Capabilities & limitations

**Good for** (what extensions do): transforming clips and MIDI notes at edit time, batch-editing tracks/devices/scenes, importing files, rendering audio from the arrangement, and adding context-menu tools, dialogs, and progress UI.

**Not for** (hard boundary): real-time processing. Extensions have **no MIDI/audio stream callbacks** — they are not devices in the signal chain and only act when the user triggers a command. For *real-time* transformations like "play one key, hear a chord," use a **MIDI effect device** (Live's built-in **Chord** device, or a **Max for Live** device), not an extension.

---

## Contributing

Contributions are very welcome! This skill is only as good as its accuracy and coverage, so feel free to:

- **Fix or sharpen** anything in `SKILL.md` or the `reference/` docs (especially mismatches with the real SDK).
- **Add recipes** for common tasks, or **share example extensions** built with the skill.
- **Keep it current** as the Extensions SDK evolves (new API versions, renamed APIs, new capabilities).

Open an issue or a pull request — small corrections are just as appreciated as big additions. If you change anything API-related, please verify it against the SDK's type definitions or a real build before submitting.

---

## Sources & references

- **Ableton Extensions (official):** https://www.ableton.com/en/live/extensions
- **Bundled SDK docs** (in the SDK zip): `docs/` — getting-started, essentials, development, design
- **Bundled API reference:** `api/` — TypeDoc HTML for every class, interface, enum, and type
- **Bundled examples:** `examples/` — `context-menu`, `modal-dialog`, `progress-dialog`, `audio-clips`, `arrangementselection`, `warpMode`, `strip-silence`
- **Tooling packages** (in the SDK zip): `@ableton-extensions/sdk`, `@ableton-extensions/cli`, `@ableton-extensions/create-extension`

The reference files in this skill are distilled from those sources and verified against the actual SDK type definitions.
