# Recipes — idiomatic code for common tasks

All snippets assume `const context = initialize(activation, "1.0.0")` inside `activate`. Each example folder under `<sdk-dir>/examples` is the canonical source — read the closest one when in doubt.

| Task | Example folder |
|---|---|
| Minimal context menu | `context-menu` |
| Toggle a clip property | `warpMode` |
| Modal dialog (WebView UI) | `modal-dialog` |
| Progress dialog + cancel | `progress-dialog` |
| Create audio clips / import | `audio-clips` |
| Arrangement time selection (incl. take lanes) | `arrangementselection` |
| Render + analyze + edit | `strip-silence` |

## 1. Context menu → command (the base pattern)

```typescript
context.commands.registerCommand("myext.process", () => {
  console.log("You right-clicked a ClipSlot!");
});

context.ui.registerContextMenuAction(
  "ClipSlot",            // scope (see api.md)
  "Process this slot",   // label shown in the menu
  "myext.process",       // command id (namespace it — it must be unique)
);
```

Async commands: catch their errors (see the notes at the end), so wrap them once:
```typescript
const safe = (fn: (arg: unknown) => Promise<unknown>) => (arg: unknown) => {
  fn(arg).catch((e) => console.error(String(e)));
};
context.commands.registerCommand("myext.asyncThing", safe(async (arg) => { /* await … */ }));
```
The recipes below use `safe`.

Register the same command under several scopes:
```typescript
(["MidiClip", "AudioClip"] as const).forEach((scope) =>
  context.ui.registerContextMenuAction(scope, "Rename this clip", "myext.rename"),
);
```

Unregister later (the registration resolves to an unregister fn):
```typescript
const unregister = await context.ui.registerContextMenuAction("AudioClip", "X", "myext.x");
await unregister();
```

## 2. Resolve a handle, narrow the type

A scope may match objects you must reject — resolve to the base, then check.

```typescript
import { initialize, AudioClip, Clip, WarpMode, type Handle, type ActivationContext } from "@ableton-extensions/sdk";

const WARP_MODES = [
  WarpMode.Beats, WarpMode.Tones, WarpMode.Texture,
  WarpMode.Repitch, WarpMode.Complex, WarpMode.ComplexPro,
]; // WarpMode skips 5 — never cycle with (mode + 1) % n

context.commands.registerCommand("myext.nextWarpMode", (arg: unknown) => {
  const clip = context.getObjectFromHandle(arg as Handle, Clip);
  if (!(clip instanceof AudioClip)) {
    console.error("Not an AudioClip.");
    return;
  }
  const i = WARP_MODES.indexOf(clip.warpMode);
  clip.warpMode = WARP_MODES[(i + 1) % WARP_MODES.length]!; // mutate by assignment
});
```

## 3. Edit MIDI notes

`MidiClip.notes` is read/written as a whole array. Map, then assign back.

```typescript
import { initialize, MidiClip, type Handle } from "@ableton-extensions/sdk";

context.commands.registerCommand("myext.transposeUp", (arg: unknown) => {
  const clip = context.getObjectFromHandle(arg as Handle, MidiClip);
  clip.notes = clip.notes.map((n) => ({ ...n, pitch: Math.min(127, n.pitch + 12) }));
});
```

Add notes:
```typescript
clip.notes = [...clip.notes, { pitch: 60, startTime: 0, duration: 1, velocity: 100 }];
```

## 4. Arrangement time selection

Register under `"<Audio|Midi>Track.ArrangementSelection"`; the arg is an `ArrangementSelection`. `selected_lanes` may mix tracks and take lanes.

```typescript
import { initialize, MidiTrack, TakeLane, DataModelObject, type ActivationContext, type ArrangementSelection } from "@ableton-extensions/sdk";

context.ui.registerContextMenuAction("MidiTrack.ArrangementSelection", "Fill with clips", "myext.fill");

context.commands.registerCommand("myext.fill", safe(async (arg) => {
  const sel = arg as ArrangementSelection;
  const start = sel.time_selection_start;
  const length = sel.time_selection_end - start;

  const lanes = sel.selected_lanes
    .map((h) => context.getObjectFromHandle(h, DataModelObject))
    .filter((o): o is MidiTrack<"1.0.0"> | TakeLane<"1.0.0"> =>
      o instanceof MidiTrack || (o instanceof TakeLane && o.parent instanceof MidiTrack));

  const clips = await context.withinTransaction(() =>
    Promise.all(lanes.map((lane) => lane.createMidiClip(start, length))),
  );
  context.withinTransaction(() => clips.forEach((c, i) => { c.name = `Clip ${i + 1}`; }));
}));
```

## 5. Create / import audio clips

Always import outside files first and use the returned path.

```typescript
import * as ableton from "@ableton-extensions/sdk";

// Into a Session clip slot:
context.commands.registerCommand("myext.dropClip", safe(async (arg) => {
  const slot = context.getObjectFromHandle(arg as ableton.Handle, ableton.ClipSlot);
  const imported = await context.resources.importIntoProject("C:/path/to/file.wav");
  await slot.createAudioClip({
    filePath: imported,
    isWarped: true,                       // required whenever loopSettings is given
    loopSettings: {                       // all five fields are required
      looping: true, startMarker: 0, endMarker: 4, loopStart: 0, loopEnd: 2,
    },
  });
}));

// Into an arrangement time selection on an AudioTrack:
context.commands.registerCommand("myext.dropArr", safe(async (arg) => {
  const sel = arg as ableton.ArrangementSelection;
  const track = context.getObjectFromHandle(sel.selected_lanes[0]!, ableton.AudioTrack);
  const imported = await context.resources.importIntoProject("C:/path/to/file.wav");
  await track.createAudioClip({
    filePath: imported,
    startTime: sel.time_selection_start,
    duration: sel.time_selection_end - sel.time_selection_start, // capped at the file length unless looping
    isWarped: false,
  });
}));
```

Unwarped clips (`isWarped: false`) can't loop: their `loopSettings.looping` must be `false`, with `loopStart === startMarker` and `loopEnd === endMarker`.

## 6. Modal dialog (WebView UI)

esbuild inlines the HTML; pass it as a data URL. The dialog talks back over the platform WebView bridge.

`build.ts` must include `loader: { ".html": "text" }`. Add `src/html.d.ts` (see scaffolding.md). Projects made by the creator keep the page in `ui/interface.html`.

```typescript
import { initialize, Clip, type Handle, type ActivationContext } from "@ableton-extensions/sdk";
import modalHtml from "../ui/interface.html"; // inlined string

context.commands.registerCommand("myext.rename", safe(async (arg) => {
  const clip = context.getObjectFromHandle(arg as Handle, Clip);
  const url = `data:text/html,${encodeURIComponent(modalHtml)}`;
  const result = await context.ui.showModalDialog(url, 360, 240); // string posted by the dialog
  const { name } = JSON.parse(result) as { name: string | null };
  if (name) clip.name = name;                                      // null = the user pressed Escape
}));

context.ui.registerContextMenuAction("AudioClip", "Rename…", "myext.rename");
```

The HTML posts its result via the bridge — support both platforms:

```html
<script>
  const wk = window.webkit?.messageHandlers?.live;          // macOS
  const wv = window.chrome?.webview;                         // Windows (WebView2)
  function send(message) {
    if (wk) wk.postMessage(message);
    else if (wv) wv.postMessage(message);
  }
  function closeWithResult(result) {
    // params must be an array holding ONE string
    send({ method: "close_and_send", params: [JSON.stringify(result)] });
  }
  document.addEventListener("keydown", (e) => {
    if (e.key === "Enter") closeWithResult({ name: document.getElementById("name").value });
    if (e.key === "Escape") closeWithResult({ name: null });
  });
</script>
```

Use Ableton's dark-theme CSS variables so the dialog matches Live (from the `modal-dialog` example):

```css
:root {
  --p-live-ui-bg: hsl(0,0%,21%);
  --p-live-control-bg: hsl(0,0%,16%);
  --p-live-input-bg: hsl(0,0%,12%);
  --p-live-text-primary: hsl(0,0%,71%);
  --p-live-text-secondary: hsl(0,0%,41%);
  --p-live-control-border: hsl(0,0%,7%);
  --p-live-accent-primary: hsl(31,100%,67%);
  --p-live-control-text--enabled: hsl(0,0%,7%);
}
html { font-family: "AbletonSansSmall", sans-serif; font-size: 11.5px; font-weight: 500; }
```

## 7. Progress dialog with cancellation

```typescript
const delay = (ms: number) => new Promise<void>((r) => setTimeout(r, ms));

context.commands.registerCommand("myext.longTask", safe(() =>
  context.ui.withinProgressDialog("Working…", {}, async (update, signal) => {
    for (let i = 0; i < 100; i++) {
      await delay(50);
      await update("Processing…", i);   // i = percent (0–100)
      if (signal.aborted) {             // user clicked Cancel
        console.warn(`Cancelled at ${i}%`);
        return;
      }
    }
    await update("Cleaning up", undefined);
  }),
));
```

The dialog closes when the callback returns or throws; `withinProgressDialog` resolves to the callback's return value.

## 8. Transactions — one undo step, batched updates

Wrap mutations so the user gets a single undo and Live updates once. The callback is synchronous but may return promises to await afterward.

```typescript
const promises = context.withinTransaction(() =>
  ranges.map((r) => track.clearClipsInRange(r.start, r.end)),
);
await Promise.all(promises);
```

## 9. Render audio off a track, then act on it (advanced)

From `strip-silence`: render a selection, decode it (e.g. the `audio-decode` npm package), analyze, then edit within a transaction.

```typescript
import * as fs from "node:fs/promises";

const audioPath = await context.resources.renderPreFxAudio(
  track, sel.time_selection_start, sel.time_selection_end,
);
// The format follows Live's Settings → Record File Type (WAV or AIFF).
if (!audioPath.toLowerCase().endsWith(".wav")) throw new Error("Set Record File Type to WAV");
// const decoded = await decodeAudio(await fs.readFile(audioPath)); analyze → ranges in seconds
const beatsPerSecond = context.application.song.tempo / 60;
const ops = context.withinTransaction(() =>
  silenceRanges.map((r) => track.clearClipsInRange(
    sel.time_selection_start + r.start * beatsPerSecond,
    sel.time_selection_start + r.end * beatsPerSecond,
  )),
);
await Promise.all(ops);
```
Extra runtime deps (like `audio-decode`) go in `package.json` `dependencies`; esbuild bundles them. Check that a dependency doesn't rely on globals the extension context lacks (`TextDecoder`, `setImmediate`, `global`, … — see SKILL.md).

## Async / error-handling notes

- Most object methods return `Promise` — `await` them and wrap handlers in `try/catch`. Under `npm start`, logs show in the terminal; for an installed extension, in `ExtensionHost.txt`.
- Register everything synchronously inside `activate` — don't `await` before registering commands/menus.
- Don't cache resolved objects or handles between invocations; re-resolve from the handle each command call.
- **Catch errors in async commands.** The extension runs in Node, where an unhandled promise rejection terminates the process by default. Wrap every `async` command body in `try/catch`, or end it with `.catch((e) => console.error(String(e)))` as `strip-silence` does — including fire-and-forget calls like `void context.ui.withinProgressDialog(...)`.
