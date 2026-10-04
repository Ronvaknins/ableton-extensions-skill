# API object model (`@ableton-extensions/sdk`)

Everything imports from the package root: `import { ... } from "@ableton-extensions/sdk"`. Full TypeDoc HTML is at `<sdk-dir>/api`; the authoritative signatures are in `node_modules/@ableton-extensions/sdk/dist/index.d.mts`.

Classes are generic over the API version (`Track<"1.0.0">`). In **value positions** (`instanceof Track`, `getObjectFromHandle(h, Track)`) write the bare class. In **type positions** — annotations and `is` predicates — supply the version you passed to `initialize`: `(o): o is MidiTrack<"1.0.0"> => o instanceof MidiTrack`. Omitting it is error `TS2314`.

Units: every time, position, length and duration is in **beats**.

## Initialization

```typescript
initialize(activation: ActivationContext, apiVersion: "1.0.0"): ExtensionContext
```
- `ActivationContext` — passed to `activate()`; `hostApiVersion` is the newest API version the host supports.
- `initialize` throws if the host doesn't support the requested version. Pass the **lowest** version that has what you need — the host keeps old versions working, so a lower one runs on more Live releases.
- `EXTENSIONS_API_VERSIONS` — exported array of supported versions, newest first (`["1.0.0"]`).

## `ExtensionContext` — the root

| Member | Type / signature | Purpose |
|---|---|---|
| `application` | `Application` | `.song` → current Live Set |
| `commands` | `Commands` | register/execute commands |
| `ui` | `Ui` | context menus, modal & progress dialogs |
| `resources` | `Resources` | import files, render audio |
| `environment` | `Environment` | `storageDirectory`, `tempDirectory`, `language` — all `string \| undefined` |
| `getObjectFromHandle(handle, Class)` | `<T>(Handle, Constructor<T>) => T` | resolve a handle to a typed object |
| `withinTransaction(fn)` | `<T>(() => T) => T` | group mutations into one undo step |

```typescript
interface Handle { id: bigint }
```
Only host-provided handles are valid — never construct one. Handles invalidate on delete / move / closing the Set.

`getObjectFromHandle` caches objects by handle id (the same Live object returns the same instance) and **throws** if the object was deleted, is not an instance of `Class`, or has an unrecognised type. Resolve to `DataModelObject` when the type is unknown, then branch with `instanceof`.

`withinTransaction`:
- The callback must be **synchronous** — no `await` inside. Return `Promise.all([...])` to group async operations, then `await` the result.
- Each mutation is already its own undo step; use a transaction only to merge several into one.
- Nested calls collapse into the outermost one.
- You can't create an object and modify it in the same transaction (you don't have the instance yet) — use two transactions.

## `Commands`

```typescript
commands.registerCommand(id: string, cb: (...args: unknown[]) => void): void  // id must be unique
commands.executeCommand(id: string, ...args: unknown[]): void
```
The callback's first arg depends on the menu scope it's invoked from (see **Scopes & selections**). Cast it explicitly.

## `Ui`

```typescript
ui.registerContextMenuAction(scope, title: string, commandId: string)
   : Promise<() => Promise<void>>     // resolves to an unregister fn

ui.showModalDialog(url: string, width: number, height: number): Promise<string>
   // url schemes: data:, file:, https:, http://localhost
   // usually `data:text/html,${encodeURIComponent(html)}`
   // resolves to the string the page posts with {method: "close_and_send", params: [string]}
   // rejects if url is malformed or an unexpected error occurred

ui.withinProgressDialog(
  text: string,
  options: { progress?: number },          // 0–100
  body: (update: (text: string, progress?: number) => Promise<void>,
         signal: AbortSignal) => Promise<unknown>
): Promise<unknown>                        // resolves/rejects with body's result
```
The progress dialog closes when `body` settles. **Cancel** aborts `signal`; check it (`signal.aborted` / `signal.throwIfAborted()`) and return. While it's open the user can't edit the Set.

## `Resources`

```typescript
resources.importIntoProject(filePath: string): Promise<string>
   // copies the file into the project folder; returns the copy's path — use that path from now on.
resources.renderPreFxAudio(track: AudioTrack, startTime: number, endTime: number): Promise<string>
   // pre-FX audio of an arrangement range → path of a file in the extension's temp directory.
   // format follows Live's Record File Type setting (WAV or AIFF) — check the extension.
```
Rendering is the only way to "read" audio — there's no live audio access.

## `Application` / `Song`

`Song` = the current Live Set (`context.application.song`).

Properties:
- `tracks: Track[]` — regular tracks only (no return tracks, no main track)
- `returnTracks: Track[]`; `mainTrack: Track`
- `scenes: Scene[]`, `cuePoints: CuePoint[]`
- `tempo: number` (get/set)
- `gridQuantization: GridQuantization`, `gridIsTriplet: boolean` (get) — combine for the full grid setting
- `rootNote: number` — 0 (C) … 11 (B); `scaleName: string`; `scaleMode: boolean` (Scale Mode on/off); `scaleIntervals: number[]` (semitones from the root)

Methods (async):
- `createAudioTrack()`, `createMidiTrack()` — inserted after the last selected track, else appended
- `createScene(index)` — 0-based position in `[0, scenes.length]`; `-1` appends
- `duplicateTrack(t)`, `duplicateScene(s)` — the copy is inserted right after the original
- `deleteTrack(t)`, `deleteScene(s)`
- `createCuePoint(time)`, `deleteCuePoint(c)`

## Tracks

`Track` (base) → `AudioTrack`, `MidiTrack`.

- Props: `name` (get/set), `mute`, `solo`, `arm` (get/set), `mutedViaSolo` (get), `groupTrack: Track | null`, `mixer: TrackMixer`
- Collections: `devices: Device[]`, `clipSlots: ClipSlot[]`, `arrangementClips: Clip[]`, `takeLanes: TakeLane[]`
- `insertDevice(deviceName, index)` — a **built-in** Live device by name (`"Reverb"`, `"Auto Filter"`, …) with its default preset, at 0-based `index`. Third-party plug-ins can't be loaded.
- `duplicateDevice(d)` — copy goes right after the original; `deleteDevice(d)`
- `deleteClip(c)` — arrangement clips only (Session clips: `ClipSlot.deleteClip()`)
- `clearClipsInRange(startTime, endTime)` — deletes clips in the range; clips crossing a boundary are truncated to it
- `createTakeLane()` — appended to `takeLanes`
- `MidiTrack.createMidiClip(startTime, duration)` → `MidiClip`
- `AudioTrack.createAudioClip(args)` → `AudioClip`:

```typescript
track.createAudioClip({
  filePath: string,                 // absolute path — use the path importIntoProject returned
  startTime: number,                // arrangement position
  duration?: number,                // default: the sample's natural length at the current tempo;
                                    // capped at that length unless looping (looping fills it)
  isWarped?: boolean,               // default: the file's .asd settings, else Live's Auto-Warp preference;
                                    // REQUIRED when loopSettings is given
  loopSettings?: ClipLoopSettings,  // if isWarped is false, loopSettings.looping must be false
})
```

`TakeLane`: `name` (get/set), `clips: Clip[]`, `createMidiClip(startTime, duration)`, `createAudioClip(args)` (same args as `AudioTrack.createAudioClip`). A take lane's `parent` is its track — check `lane.parent instanceof MidiTrack` before creating MIDI clips on it.

## Clips

`Clip` (base) → `AudioClip`, `MidiClip`.

- Common: `name` (get/set), `color: number` (get/set), `muted` (get/set), `looping` (get/set — enabling it on an unwarped audio clip turns warping on), and read-only `startTime`, `endTime`, `duration`, `startMarker`, `endMarker`, `loopStart`, `loopEnd`
- Markers and loop points are **read-only** on existing clips; set them at creation time with `ClipLoopSettings`.
- `AudioClip`: `filePath` (get), `warping` (get/set), `warpMode: WarpMode` (get/set), `warpMarkers: WarpMarker[]` (get)
- `MidiClip`: `notes: NoteDescription[]` — **get/set the whole array** to edit MIDI

### `NoteDescription`

```typescript
type NoteDescription = {
  pitch: number;              // MIDI note number   (required)
  startTime: number;          // beats              (required)
  duration: number;           // beats              (required)
  velocity?: number;
  probability?: number;
  velocityDeviation?: number;
  muted?: boolean;
  releaseVelocity?: number;
  selected?: boolean;
};
```
## Clip slots (Session view)

`ClipSlot`:
- `clip: Clip | null`
- `createMidiClip(length): Promise<MidiClip>`
- `createAudioClip({ filePath, isWarped?, loopSettings? }): Promise<AudioClip>` — same rules as `AudioTrack.createAudioClip`
- `deleteClip(): Promise<void>`

```typescript
interface ClipLoopSettings {      // ALL fields required, all in beats
  looping: boolean;
  startMarker: number; endMarker: number;
  loopStart: number;   loopEnd: number;
}
```
Enforced: `startMarker ≤ endMarker`; the loop is at least 0.25 beats; when `looping` is `false`, `loopStart === startMarker` and `loopEnd === endMarker`; when `isWarped` is `false`, positions are ≥ 0 and `looping` is `false`.

## Devices & parameters

`Device` (base) → `RackDevice` → `DrumRack`; `Device` → `Simpler`.

- `Device`: `name` (get), `parameters: DeviceParameter[]`
- `RackDevice`: `chains: Chain[]`, `insertChain(index)` (0-based in `[0, chains.length]`)
- `DrumRack`: `chains: DrumChain[]`
- `Chain`: `devices: Device[]`, `mixer: ChainMixer`, `insertDevice(name, index)`, `deleteDevice(d)`, `duplicateDevice(d)` — same rules as on `Track`
- `DrumChain` (extends `Chain`): `receivingNote: number` (get/set) — the MIDI note that triggers the pad
- `Simpler`: `sample: Sample | null`, `replaceSample(filePath): Promise<Sample>` (absolute path; import outside files first)
- `Sample`: `filePath` (get)
- `TrackMixer` / `ChainMixer`: `volume: DeviceParameter`, `panning: DeviceParameter`, `sends: DeviceParameter[]`
- `DeviceParameter`: `name`, `min`, `max`, `defaultValue`, `isQuantized`, `valueItems: DeviceParameterValueItem[]` (get); async `getValue(): Promise<number>` / `setValue(v): Promise<void>`

```typescript
interface DeviceParameterValueItem { name: string; shortName: string }
// the labels of a quantized parameter's steps, in order
```
Read `min`/`max` for a parameter's range instead of assuming 0–1.

## Other objects

- `Scene`: `name` (get/set), `tempo`, `signatureNumerator`, `signatureDenominator` (get)
- `CuePoint`: `name` (get/set), `time` (get)
- `DataModelObject` — abstract base of all model classes: `handle`, `parent` (canonical parent or `null`); every class also has a static `className` (`"AudioClip"`, `"DrumRackDevice"`, …) the SDK uses to resolve handles

## Enums

```typescript
enum WarpMode { Beats=0, Tones=1, Texture=2, Repitch=3, Complex=4, ComplexPro=6 }  // 5 is unused

enum GridQuantization {
  NoGrid=0, EightBars=1, FourBars=2, TwoBars=3, Bar=4,
  Half=5, Quarter=6, Eighth=7, Sixteenth=8, ThirtySecond=9
}
```
`WarpMode` is not contiguous — cycle through an explicit array, never `(mode + 1) % n`. Combine `gridQuantization` with `gridIsTriplet` for the full grid setting.

```typescript
interface WarpMarker { sampleTime: number; beatTime: number }
```

## Scopes & selections (context-menu registration)

The scope parameter's type is `ContextMenuScope<"1.0.0">`.

`registerContextMenuAction(scope, title, commandId)` — the **scope** string determines both where the item appears and the **arg** the command receives.

**Object scopes** → command receives a single `Handle`:
`"AudioClip"`, `"MidiClip"`, `"AudioTrack"`, `"MidiTrack"`, `"ClipSlot"`, `"Scene"`, `"Sample"`, `"Simpler"`, `"DrumRack"`.

**Selection scopes** → command receives a structured object:
- `"AudioTrack.ArrangementSelection"` / `"MidiTrack.ArrangementSelection"` — right-click on a time selection in Arrangement View → `ArrangementSelection`:
  ```typescript
  interface ArrangementSelection {
    time_selection_start: number;   // beats
    time_selection_end: number;     // beats
    selected_lanes: Handle[];       // tracks and/or take lanes, possibly mixed types
  }
  ```
- `"ClipSlotSelection"` — right-click on a selection of Session clip slots → `ClipSlotSelection { selected_clip_slots: Handle[] }`.

Resolve `selected_lanes` to objects and filter by type:
```typescript
const tracks = selection.selected_lanes
  .map((h) => context.getObjectFromHandle(h, DataModelObject))
  .filter((o): o is AudioTrack<"1.0.0"> => o instanceof AudioTrack);
```
