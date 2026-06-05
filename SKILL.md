---
name: ableton-extensions
description: Scaffold, write, build, and package Ableton Live extensions with the Ableton Extensions SDK (@ableton-extensions/sdk, TypeScript). Use when creating or editing an Ableton Live extension — adding context-menu actions, modal/progress dialogs, or code that manipulates tracks, clips, MIDI notes, devices, tempo, audio rendering, or selections in a Live Set.
---

# Ableton Extensions SDK

Build tools that run inside Ableton Live. An extension is a Node/TypeScript module bundled to a single CommonJS file that Live's **Extension Host** loads. It registers **commands** and wires them to **context-menu actions**, then reads and mutates the Live Set through a typed object model (tracks, clips, notes, devices, tempo, scenes).

Extensions are **not** real-time audio processors. They automate edits, transform musical data, and add UI — they cannot process the audio stream live. Visualize results instead.

## Local SDK location

The SDK distribution lives at `C:\DEV\extensions-sdk-1.0.0-beta.0`:
- `docs/` — pre-built HTML documentation (getting-started, essentials, development, design)
- `api/` — TypeDoc HTML API reference
- `examples/` — 7 runnable examples (read these for canonical patterns)
- `ableton-create-extension-1.0.0-beta.0.tgz` — the scaffolding tool
- `ableton-extensions-cli-1.0.0-beta.0.tgz` — the build/run/package CLI
- `ableton-extensions-sdk-1.0.0-beta.0.tgz` — the SDK library

Version: **1.0.0-beta.0**, API version **"1.0.0"**. Requires **Node.js >= 24.14.1**.

## Reference files (load as needed)

- **[reference/scaffolding.md](reference/scaffolding.md)** — create a project, the generated file layout, `manifest.json` / `package.json` / `build.ts` / `tsconfig.json`, the `extensions-cli run|package` commands, the `.env` / `EXTENSION_HOST_PATH`, debugging, log file locations, packaging to `.ablx`.
- **[reference/api.md](reference/api.md)** — the full object model: `ExtensionContext`, handles, transactions, `Song`/`Track`/`Clip`/`MidiClip`/`AudioClip`/`Device`/`Scene`, MIDI `NoteDescription`, enums (`WarpMode`, `GridQuantization`), context-menu scopes, selection types.
- **[reference/recipes.md](reference/recipes.md)** — copy-paste code for every common task: context menus, modal dialogs (with the WebView bridge + Ableton-themed CSS), progress dialogs with cancellation, creating/editing clips and notes, audio import/render, transactions.

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
- `context.resources` — `importIntoProject(path)`, render audio from a track
- `context.environment` — `storageDirectory`, `tempDirectory`, `language`
- `context.getObjectFromHandle(handle, Class)` — turn a `Handle` into a typed object
- `context.withinTransaction(() => …)` — group mutations into one undo step

## Two concepts that trip people up

**Handles are not objects.** Live passes commands lightweight `Handle`s (`{ id: bigint }`). Resolve to a typed object with `context.getObjectFromHandle(handle, Track)`. Handles go **invalid** when the object is deleted, moved in the arrangement, or the Set changes — never cache objects or handles across operations.

**Commands receive scope-dependent args.** Object scopes (`"AudioClip"`, `"MidiTrack"`, `"ClipSlot"`, `"Scene"`, …) pass a single `Handle`. Selection scopes pass a structured object: `"AudioTrack.ArrangementSelection"` / `"MidiTrack.ArrangementSelection"` pass an `ArrangementSelection` (`time_selection_start`, `time_selection_end`, `selected_lanes: Handle[]`); `"ClipSlotSelection"` passes `{ selected_clip_slots }`. Match the scope you register to the arg type you cast.

## Standard workflow

1. **Scaffold** (new project): `npx file:C:/DEV/extensions-sdk-1.0.0-beta.0/ableton-create-extension-1.0.0-beta.0.tgz <dir>` — interactive prompts for name, author, Live path, UI support. See [reference/scaffolding.md](reference/scaffolding.md).
2. **Write** `src/extension.ts` — register commands + menu actions; see [reference/recipes.md](reference/recipes.md).
3. **Run in Live**: `npm start` (builds with esbuild, then `extensions-cli run` loads it into the Extension Host). Enable Developer Mode first in Live's *Preferences → Extensions*.
4. **Debug**: `console.log/warn/error` → `ExtensionHost.txt` (paths in [reference/scaffolding.md](reference/scaffolding.md)). In VS Code, F5 / "Debug Extension".
5. **Package**: `npm run package` → a distributable `.ablx` archive users drop into Live.

## Conventions that matter

- **TypeScript, ESM project, CJS bundle.** `package.json` is `"type": "module"`; `build.ts` bundles to `format: "cjs"`, `platform: "node"`. Keep it that way — the Extension Host loads CommonJS.
- **Namespace command IDs** (`"myext.action"`) to avoid collisions with other extensions.
- **Most mutations are async** (`Promise`). Batch related edits in `context.withinTransaction(() => promises)` then `await Promise.all(...)` so the user gets one undo step and Live updates once.
- **Narrow handle types defensively** with `instanceof` (e.g. resolve to `Clip`, then check `clip instanceof AudioClip`) — a menu scope can match objects the code must reject.
- **HTML dialogs are inlined** by esbuild (`loader: { ".html": "text" }`) and passed as a `data:text/html,` URL — there's no web server.
