# API object model (`@ableton-extensions/sdk`)

Everything imports from the package root: `import { ... } from "@ableton-extensions/sdk"`. Full TypeDoc HTML is at `C:\DEV\extensions-sdk-1.0.0-beta.0\api`. Classes are generic over the API version (`Track<"1.0.0">`); the version usually infers — write `Track`, `AudioClip`, etc.

## Initialization

```typescript
initialize(activation: ActivationContext, apiVersion: "1.0.0"): ExtensionContext
```
- `ActivationContext` — passed to `activate()`; has `hostApiVersion`. Throws if the host doesn't support the requested version.
- `EXTENSIONS_API_VERSIONS` — exported array of supported versions (`["1.0.0"]`).

## `ExtensionContext` — the root

| Member | Type / signature | Purpose |
|---|---|---|
| `application` | `Application` | `.song` → current Live Set |
| `commands` | `Commands` | register/execute commands |
| `ui` | `Ui` | context menus, modal & progress dialogs |
| `resources` | `Resources` | import files, render audio |
| `environment` | `Environment` | `storageDirectory`, `tempDirectory`, `language` (uppercase ISO‑639‑1, e.g. `"EN"`) |
| `getObjectFromHandle(handle, Class)` | `<T>(Handle, Constructor<T>) => T` | resolve a handle to a typed object |
| `withinTransaction(fn)` | `<T>(() => T) => T` | group mutations into one undo step (nested calls collapse) |

```typescript
interface Handle { id: bigint }
```
Handles invalidate on delete / move / Set change. Resolve to `DataModelObject` when the type is unknown, then branch with `instanceof`.

## `Commands`

```typescript
commands.registerCommand(id: string, cb: (...args: unknown[]) => void): void
commands.executeCommand(id: string, ...args: unknown[]): void
```
The callback's first arg depends on the menu scope it's invoked from (see **Scopes & selections** below). Cast it explicitly.

## `Ui`

```typescript
ui.registerContextMenuAction(scope, title: string, commandId: string)
   : Promise<() => Promise<void>>     // resolves to an unregister fn

ui.showModalDialog(url: string, width: number, height: number): Promise<string>
   // url is usually `data:text/html,${encodeURIComponent(html)}`
   // resolves to the string the dialog posts back via the WebView bridge

ui.withinProgressDialog(
  text: string,
  options: { progress?: number },
  body: (update: (text: string, progress?: number) => Promise<void>,
         signal: AbortSignal) => Promise<unknown>
): Promise<unknown>
   // progress 0–100, or undefined for indeterminate; call signal.throwIfAborted()
```

## `Resources`

```typescript
resources.importIntoProject(filePath: string): Promise<string>  // → project-relative path for clip creation
resources.renderPreFxAudio(track, startTime: number, endTime: number): Promise<string>  // → WAV path on disk
```
Rendering is how you "read" audio — there's no live audio access.

## `Application` / `Song`

`Song` = the current Live Set.

- Collections: `tracks[]`, `returnTracks[]`, `mainTrack`, `scenes[]`, `cuePoints[]`
- Transport/grid: `tempo` (get/set), `gridQuantization`, `gridIsTriplet`
- Scale: `rootNote`, `scaleName`, `scaleMode`, `scaleIntervals`
- Methods (async): `createAudioTrack()`, `createMidiTrack()`, `createScene(index)` (`-1` appends), `createCuePoint(time)`, `deleteTrack(t)`, `deleteScene(s)`, `deleteCuePoint(c)`, `duplicateTrack(t)`, `duplicateScene(s)`

## Tracks

`Track` (base) → `AudioTrack`, `MidiTrack`.

- Props: `name`, `arm`, `mute`, `solo`, `mutedViaSolo`, `groupTrack`, `mixer: TrackMixer`
- Collections: `devices[]`, `clipSlots[]`, `arrangementClips[]`, `takeLanes[]`
- Methods (async): `insertDevice(name, index)`, `duplicateDevice(d)`, `deleteDevice(d)`, `deleteClip(c)`, `clearClipsInRange(startTime, endTime)`, `createTakeLane()`
- `MidiTrack.createMidiClip(startTime, duration)` and `AudioTrack.createAudioClip({ filePath, startTime, duration, isWarped? })` create arrangement clips on a track/lane.

## Clips

`Clip` (base) → `AudioClip`, `MidiClip`.

- Common: `name`, `color`, `muted`, `looping`, plus timing `startTime`, `endTime`, `duration`, `loopStart`, `loopEnd`, `startMarker`, `endMarker`
- `AudioClip`: `filePath` (get), `warping` (boolean, get/set), `warpMode: WarpMode` (get/set), `warpMarkers` (get). Note `isWarped` is **not** a clip property — it's only an argument to `createAudioClip(...)`.
- `MidiClip`: `notes: NoteDescription[]` — **get/set the whole array** to edit MIDI

```typescript
interface NoteDescription {
  pitch: number;          // 0–127
  startTime: number;      // beats
  duration: number;       // beats
  velocity?: number;      // 0–127
  releaseVelocity?: number;
  probability?: number;   // 0–100
  velocityDeviation?: number;
  muted?: boolean;
  selected?: boolean;
}
```

## Clip slots (Session view)

`ClipSlot`:
- `clip: Clip | null`
- `createMidiClip(length): Promise<MidiClip>`
- `createAudioClip({ filePath, isWarped?, loopSettings? }): Promise<AudioClip>`
- `deleteClip(): Promise<void>`

```typescript
interface ClipLoopSettings {
  looping?: boolean; startMarker?: number; endMarker?: number;
  loopStart?: number; loopEnd?: number;
}
```

## Devices & parameters

`Device` (base) → `RackDevice` → `DrumRack`; `Simpler`.

- `Device`: `name`, `parameters: DeviceParameter[]`
- `RackDevice`: `chains: Chain[]`, `selectedChain`
- `Chain` → `DrumChain`; `Chain.devices[]`, `Chain.mixer: ChainMixer`
- `DeviceParameter`: `name`, `min`, `max`, `defaultValue`, `isQuantized`, `valueItems: DeviceParameterValueItem[]`; async `getValue()` / `setValue(v)`

```typescript
interface DeviceParameterValueItem { name: string; value: number }
```

## Other objects

`Scene`, `CuePoint`, `TakeLane`, `Sample`, `TrackMixer`, `ChainMixer`. `DataModelObject` is the abstract base of all of them (`handle`, `parent`).

## Enums

```typescript
enum WarpMode { Beats=0, Tones=1, Texture=2, Repitch=3, Complex=4, ComplexPro=6 }

enum GridQuantization {
  NoGrid=0, EightBars=1, FourBars=2, TwoBars=3, Bar=4,
  Half=5, Quarter=6, Eighth=7, Sixteenth=8, ThirtySecond=9
}
```
Combine `gridQuantization` with `gridIsTriplet` for the full grid setting.

## Scopes & selections (context-menu registration)

`registerContextMenuAction(scope, title, commandId)` — the **scope** string determines both where the item appears and the **arg** the command receives.

**Object scopes** → command receives a single `Handle`:
`"AudioClip"`, `"MidiClip"`, `"AudioTrack"`, `"MidiTrack"`, `"ClipSlot"`, `"Scene"`, `"Sample"`, `"Simpler"`, `"DrumRack"`.

**Selection scopes** → command receives a structured object:
- `"AudioTrack.ArrangementSelection"` / `"MidiTrack.ArrangementSelection"` → `ArrangementSelection`:
  ```typescript
  interface ArrangementSelection {
    time_selection_start: number;   // beats
    time_selection_end: number;     // beats
    selected_lanes: Handle[];       // tracks / take lanes
  }
  ```
- `"ClipSlotSelection"` → `{ selected_clip_slots: Handle[] }`.

Resolve `selected_lanes` to objects and filter by type:
```typescript
const tracks = selection.selected_lanes
  .map((h) => context.getObjectFromHandle(h, DataModelObject))
  .filter((o): o is AudioTrack => o instanceof AudioTrack);
```
