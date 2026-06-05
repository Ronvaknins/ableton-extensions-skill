# Recipes — idiomatic code for common tasks

All snippets assume `const context = initialize(activation, "1.0.0")` inside `activate`. Each example folder under `C:\DEV\extensions-sdk-1.0.0-beta.0\examples` is the canonical source — read the closest one when in doubt.

| Task | Example folder |
|---|---|
| Minimal context menu | `context-menu` |
| Toggle a clip property | `warpMode` |
| Modal dialog (WebView UI) | `modal-dialog` |
| Progress dialog + cancel | `progress-dialog` |
| Create audio clips / import | `audio-clips` |
| Arrangement time selection | `arrangementselection` |
| Render + analyze + edit | `strip-silence` |

## 1. Context menu → command (the base pattern)

```typescript
context.commands.registerCommand("myext.process", () => {
  console.log("You right-clicked a ClipSlot!");
});

context.ui.registerContextMenuAction(
  "ClipSlot",            // scope (see api.md)
  "Process this slot",   // user-facing label
  "myext.process",       // command id (namespace it!)
);
```

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

context.commands.registerCommand("myext.bumpWarp", (arg: unknown) => {
  const clip = context.getObjectFromHandle(arg as Handle, Clip);
  if (!(clip instanceof AudioClip)) {
    console.error("Not an AudioClip.");
    return;
  }
  clip.warpMode = ((clip.warpMode + 1) % 3) as WarpMode; // mutate by assignment
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
Add notes: `clip.notes = [...clip.notes, { pitch: 60, startTime: 0, duration: 1, velocity: 100 }];`

## 4. Arrangement time selection

Register under `"<Audio|Midi>Track.ArrangementSelection"`; the arg is an `ArrangementSelection`.

```typescript
import { initialize, MidiTrack, DataModelObject, type ActivationContext, type ArrangementSelection } from "@ableton-extensions/sdk";

context.ui.registerContextMenuAction("MidiTrack.ArrangementSelection", "Fill with clips", "myext.fill");

context.commands.registerCommand("myext.fill", async (arg: unknown) => {
  const sel = arg as ArrangementSelection;
  const start = sel.time_selection_start;
  const length = sel.time_selection_end - start;

  const lanes = sel.selected_lanes
    .map((h) => context.getObjectFromHandle(h, DataModelObject))
    .filter((o): o is MidiTrack => o instanceof MidiTrack);

  const clips = await Promise.all(lanes.map((lane) => lane.createMidiClip(start, length)));
  clips.forEach((c, i) => { c.name = `Clip ${i + 1}`; });
});
```

## 5. Create / import audio clips

```typescript
import * as ableton from "@ableton-extensions/sdk";

// Into a Session clip slot:
context.commands.registerCommand("myext.dropClip", async (arg: unknown) => {
  const slot = context.getObjectFromHandle(arg as ableton.Handle, ableton.ClipSlot);
  const imported = await context.resources.importIntoProject("C:/path/to/file.wav");
  await slot.createAudioClip({
    filePath: imported,
    isWarped: false,
    loopSettings: { looping: false, startMarker: 1, endMarker: 5, loopStart: 1, loopEnd: 5 },
  });
});

// Into an arrangement time selection on an AudioTrack:
context.commands.registerCommand("myext.dropArr", async (arg: unknown) => {
  const sel = arg as ableton.ArrangementSelection;
  const track = context.getObjectFromHandle(sel.selected_lanes[0]!, ableton.AudioTrack);
  const imported = await context.resources.importIntoProject("C:/path/to/file.wav");
  await track.createAudioClip({
    filePath: imported,
    startTime: sel.time_selection_start,
    duration: sel.time_selection_end - sel.time_selection_start,
    isWarped: false,
  });
});
```

## 6. Modal dialog (WebView UI)

esbuild inlines the HTML; pass it as a data URL. The dialog talks back over the platform WebView bridge.

`build.ts` must include `loader: { ".html": "text" }`. Add `src/html.d.ts` (see scaffolding.md).

```typescript
import { initialize, Clip, type Handle, type ActivationContext } from "@ableton-extensions/sdk";
import modalHtml from "../ui/interface.html"; // inlined string

context.commands.registerCommand("myext.rename", async (arg: unknown) => {
  const clip = context.getObjectFromHandle(arg as Handle, Clip);
  const url = `data:text/html,${encodeURIComponent(modalHtml)}`;
  const result = await context.ui.showModalDialog(url, 360, 240); // string posted by the dialog
  clip.name = JSON.parse(result).name;
});

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
```

## 7. Progress dialog with cancellation

```typescript
const delay = (ms: number) => new Promise<void>((r) => setTimeout(r, ms));

context.commands.registerCommand("myext.longTask", () => {
  void context.ui.withinProgressDialog("Working…", {}, async (update, signal) => {
    for (let i = 0; i < 100; i++) {
      await delay(50);
      await update("Processing…", i);   // i = percent (0–100)
      try { signal.throwIfAborted(); }  // user clicked Cancel
      catch { console.warn(`Cancelled at ${i}%`); return; }
    }
    await update("Cleaning up", undefined); // undefined = indeterminate
  });
});
```

## 8. Transactions — one undo step, batched updates

Wrap mutations so the user gets a single undo and Live updates once. The callback is synchronous but may return promises to await afterward.

```typescript
const promises = context.withinTransaction(() =>
  ranges.map((r) => track.clearClipsInRange(r.start, r.end)),
);
await Promise.all(promises);
```

## 9. Render audio off a track, then act on it (advanced)

From `strip-silence`: render a selection to WAV, decode it (e.g. the `audio-decode` npm package), analyze, then edit within a transaction.

```typescript
const wavPath = await context.resources.renderPreFxAudio(
  track, sel.time_selection_start, sel.time_selection_end,
);
// const decoded = await decodeAudio(await fs.readFile(wavPath)); analyze, then:
const ops = context.withinTransaction(() =>
  silenceRanges.map((r) => track.clearClipsInRange(r.start, r.end)),
);
await Promise.all(ops);
```
Extra runtime deps (like `audio-decode`) go in `package.json` `dependencies`; esbuild bundles them.

## Async / error-handling notes

- Most object methods return `Promise` — `await` them and wrap handlers in `try/catch`; uncaught errors land in `ExtensionHost.txt`.
- Register everything synchronously inside `activate` — don't `await` before registering commands/menus.
- Don't cache resolved objects or handles between invocations; re-resolve from the handle each command call.
