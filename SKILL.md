---
name: ableton-extensions
description: Scaffold, write, build, debug, and package Ableton Live extensions with the Ableton Extensions SDK (@ableton-extensions/sdk, TypeScript, 1.0.0-beta.1). Use when creating or editing an Ableton Live extension — adding context-menu actions, modal/progress dialogs, or code that manipulates tracks, clips, MIDI notes, devices, racks, tempo, scenes, cue points, audio rendering, or selections in a Live Set — or when upgrading an extension project to a newer SDK.
---

# Ableton Extensions SDK

Build tools that run inside Ableton Live. An extension is a Node/TypeScript module bundled to a single CommonJS file that Live's **Extension Host** loads. It registers **commands** and wires them to **context-menu actions**, then reads and mutates the Live Set through a typed object model (tracks, clips, notes, devices, tempo, scenes).

Extensions are an **offline editing / automation** layer. They are **not** real-time processors and **not** devices in the signal chain: they cannot process the live audio **or MIDI** stream. The SDK has no note-on/off, MIDI-input, or stream callbacks, and no change listeners — an extension only acts when the user triggers a command, then mutates the data model. For **real-time MIDI/audio transformation** (e.g. "play one key, hear a chord"), the right tool is a **MIDI/audio effect device** in the track chain: Live's built-in MIDI effects (the **Chord** device does exactly one-note→chord) or a **Max for Live** device — *not* an Extension. Extensions also can't draw into Live's native UI, run without Live open, or act as a control surface.

## Local SDK location

The SDK distribution lives wherever you unpacked it — referred to below as `<sdk-dir>`. It contains:
- `docs/` — pre-built HTML documentation (getting-started, essentials, development, design)
- `api/` — TypeDoc HTML API reference
- `examples/` — 7 runnable examples (read these for canonical patterns)
- `upgrade.md` — steps to move an existing project to this SDK version
- `ableton-create-extension-<version>.tgz` — the scaffolding tool
- `ableton-extensions-cli-<version>.tgz` — the build/run/package CLI
- `ableton-extensions-sdk-<version>.tgz` — the SDK library

This skill is written against SDK **1.0.0-beta.1** (API version **"1.0.0"**) and the Live **12.4.5 public beta** (the only Live the SDK supports, per Ableton's SDK page). Newer builds may ship a different `<version>` — check the actual `.tgz` filenames in `<sdk-dir>` and substitute it wherever it appears below. When this skill and the SDK disagree, the SDK's type definitions (`node_modules/@ableton-extensions/sdk/dist/index.d.mts`) win.

**Node.js:** the SDK/CLI packages declare `>=24.16.0` (the docs and the scaffolded `package.json` still say `>=24.14.1`, so older installs print an `EBADENGINE` warning). Your local Node only runs the build tooling — the extension itself runs on the Node runtime **bundled with Live**.

## Reference files (load as needed)

- **[reference/scaffolding.md](reference/scaffolding.md)** — create a project, the generated layout, `manifest.json` / `package.json` / `build.ts` / `tsconfig.json`, the `ableton-extensions-cli run|package` commands, `.env` / `EXTENSION_HOST_PATH`, terminal output and log files, packaging to `.ablx`, upgrading from beta.0, running the bundled examples.
- **[reference/api.md](reference/api.md)** — the full object model: `ExtensionContext`, handles, transactions, `Song`/`Track`/`TakeLane`/`Clip`/`MidiClip`/`AudioClip`/`ClipSlot`, devices/racks/chains/parameters/mixers, `Scene`, `CuePoint`, MIDI `NoteDescription`, enums, context-menu scopes, selection types.
- **[reference/recipes.md](reference/recipes.md)** — copy-paste code for every common task: context menus, modal dialogs (with the WebView bridge + Ableton-themed CSS), progress dialogs with cancellation, creating/editing clips and notes, devices and racks, audio import/render, files and network, transactions.

The `examples/` folder is the source of truth for idiomatic code — prefer reading the closest example to the task over guessing.

## The entry point — always this shape

`src/extension.ts` exports `activate`. Call `initialize` once, then register commands and menu actions synchronously:

```typescript
import { initialize, type ActivationContext } from "@ableton-extensions/sdk";

export function activate(activation: ActivationContext) {
  const context = initialize(activation, "1.0.0"); // ExtensionContext

  // 1. Register a command (the logic)
  context.commands.registerCommand("myext.doThing", (arg: unknown) => {
    // arg is a Handle, ArrangementSelection, or ClipSlotSelection
    // depending on the scope the menu action was registered for
  });

  // 2. Surface it as a right-click menu item (the trigger)
  context.ui.registerContextMenuAction("AudioClip", "Do Thing", "myext.doThing");
}
```

`context` (the `ExtensionContext`) is the root of everything:
- `context.application.song` — the Live Set (`tempo`, `tracks`, `scenes`, …)
- `context.commands` — `registerCommand(id, cb)` / `executeCommand(id, …)`
- `context.ui` — `registerContextMenuAction`, `showModalDialog`, `withinProgressDialog`
- `context.resources` — `importIntoProject(path)`, `renderPreFxAudio(track, start, end)`
- `context.environment` — `storageDirectory`, `tempDirectory`, `language` (each `string | undefined`)
- `context.getObjectFromHandle(handle, Class)` — turn a `Handle` into a typed object
- `context.withinTransaction(() => …)` — group mutations into one undo step

There is no `deactivate` hook — `activate` is the only entry point.

## Three things that trip people up

**Handles are not objects.** Live passes commands lightweight `Handle`s (`{ id: bigint }`). Resolve to a typed object with `context.getObjectFromHandle(handle, Track)`. Handles go **invalid** when the object is deleted, moved in the Set, or the Set is closed/replaced — never cache objects or handles across operations; re-resolve on each command call.

**Commands receive scope-dependent args.** Object scopes (`"AudioClip"`, `"MidiTrack"`, `"ClipSlot"`, `"Scene"`, …) pass a single `Handle`. Selection scopes pass a structured object: `"AudioTrack.ArrangementSelection"` / `"MidiTrack.ArrangementSelection"` pass an `ArrangementSelection` (`time_selection_start`, `time_selection_end`, `selected_lanes: Handle[]`); `"ClipSlotSelection"` passes `{ selected_clip_slots }`. Match the scope you register to the arg type you cast.

**Globals are a chosen set, not all of Node's.** Since beta.1 the release notes list `performance`, `AbortSignal`, `Headers`, `Request`, `Response` and `FormData` as available in an extension's context. A global Node normally has may still be missing there, and TypeScript will not warn you (`@types/node` declares them all). If one throws `ReferenceError`, import it from Node's module instead (`URL` from `node:url`, `TextEncoder` from `node:util`, `setImmediate` from `node:timers`, …). The same goes for npm dependencies you bundle.

**Bundle everything.** The Extension Host does not resolve `node_modules` at runtime (Bundling guide), and a packaged `.ablx` holds only `manifest.json`, the entry file and what you add with `-i`.

## Standard workflow

1. **Scaffold** (new project): `npx file:<sdk-dir>/ableton-create-extension-<version>.tgz` inside an empty folder — interactive prompts for name, author, Live installation, UI support. See [reference/scaffolding.md](reference/scaffolding.md).
2. **Write** `src/extension.ts` — register commands + menu actions; see [reference/recipes.md](reference/recipes.md).
3. **Run in Live**: enable **Developer Mode** in Live's *Settings → Extensions* (this stops Live's own Extension Host), then `npm start` (esbuild dev build, then `ableton-extensions-cli run` launches the Extension Host and connects to Live). Stop with Ctrl+C; rerun `npm start` to load rebuilt code — no Live restart needed.
4. **Debug**: under `npm start`, your `console.log/info/warn/error` print straight to the terminal; add `-v` / `--verbose` to `ableton-extensions-cli run` for the raw host log. When Live hosts the extension (Developer Mode off) the same output goes to `ExtensionHost.txt` (paths in [reference/scaffolding.md](reference/scaffolding.md)).
5. **Package**: `npm run package` → `<name>-<version>.ablx` in the project root; users drop it onto the Extensions page in Live's Settings.

## Conventions that matter

- **TypeScript, ESM project, CJS bundle.** `package.json` is `"type": "module"`; `build.ts` bundles to `format: "cjs"`, `platform: "node"`. Keep it that way — the Extension Host loads the CommonJS bundle and calls its exported `activate`.
- **Namespace command IDs** (`"myext.action"`) — each command needs a unique ID.
- **Most mutations are async** (`Promise`). Batch related edits in `context.withinTransaction(() => promises)` then `await Promise.all(...)` so the user gets one undo step and Live updates once. The callback must be synchronous.
- **Property setters are synchronous and each is its own undo step** (`clip.name = …`, `track.mute = …`, `midiClip.notes = …`) unless wrapped in a transaction.
- **Catch errors in every `async` command** — `try/catch` or a trailing `.catch(e => console.error(String(e)))`, as the `strip-silence` example does. The extension runs in Node, where an unhandled promise rejection terminates the process by default.
- **Narrow handle types defensively** with `instanceof` (resolve to `Clip`, then check `clip instanceof AudioClip`) — `getObjectFromHandle` throws on a type mismatch.
- **Times are in beats**, not seconds (`seconds = beats * 60 / song.tempo`).
- **Filesystem: only `context.environment.storageDirectory` (persistent) and `tempDirectory` (scratch).** This includes child processes and native addons. Bring outside files in with `context.resources.importIntoProject(path)` and use the returned path. A stricter OS sandbox is planned; code that reads Desktop/Documents/Downloads will break.
- **HTML dialogs are inlined** by esbuild (`loader: { ".html": "text" }`) and passed as a `data:text/html,` URL — no web server needed.
