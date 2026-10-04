# Scaffolding, building, running & packaging

## Prerequisites

- **Node.js** — the SDK/CLI packages declare `>=24.16.0`; the scaffolded `package.json` and the docs still say `>=24.14.1`. On anything older than 24.16 `npm install` prints `EBADENGINE`; update Node to silence it. (Your Node runs only the build tools — the extension runs on Live's bundled Node.)
- An **Ableton Live beta build** that supports Extensions (the Live 12.4.5 public beta for SDK 1.0.0-beta.1 — earlier Live versions don't work), with **Developer Mode** enabled in *Settings → Extensions* while developing.
- The SDK distribution unpacked somewhere on your machine — referred to as `<sdk-dir>` (the folder containing `ableton-create-extension-1.0.0-beta.1.tgz`).

## Creating a new project

Make an empty folder, `cd` into it, and run the bundled creator via `npx`, pointing at its tarball (substitute the `<version>` in your `<sdk-dir>`):

```sh
# macOS
mkdir ~/Projects/my-extension && cd ~/Projects/my-extension
npx file:/path/to/sdk-dir/ableton-create-extension-1.0.0-beta.1.tgz

# Windows
mkdir C:\Users\you\Projects\my-extension
cd C:\Users\you\Projects\my-extension
npx file:C:\path\to\sdk-dir\ableton-create-extension-1.0.0-beta.1.tgz
```

Interactive prompts (in order):
1. **Extension name** — the package name and the name shown in Live.
2. **Author** — your name or organisation.
3. **Ableton Live application** — detected installations that support Extensions are listed; pick one or enter a custom path. Used to fill `EXTENSION_HOST_PATH` in `.env`.
4. **Does your extension need a user interface?** — yes scaffolds a WebView modal dialog (`ui/interface.html`, `src/html.d.ts`, `vite.config.ts`, the `.html` esbuild loader).

Flag: `--skip-install` (skip the automatic `npm install`).

The creator copies the SDK and CLI tarballs into `vendor/` and wires the dependencies to them. If VS Code is on `PATH`, it also writes `.vscode/launch.json` ("Debug Extension" → `npm start`) and `.vscode/tasks.json`.

### Generated layout

```
my-extension/
├── .env                 # EXTENSION_HOST_PATH (gitignored, per machine)
├── .gitignore           # node_modules/, dist/, vendor/, .env, *.ablx, …
├── README.md
├── manifest.json        # extension metadata Live reads
├── package.json
├── tsconfig.json
├── build.ts             # esbuild bundler script
├── src/
│   ├── extension.ts     # entry point — exports activate()
│   └── html.d.ts        # only with UI — declares *.html modules
├── ui/                  # only with UI
│   └── interface.html   # imported from src as "../ui/interface.html"
├── vite.config.ts       # only with UI — `root: "ui"`, port 5173
├── .vscode/             # only if VS Code was detected
└── vendor/
    ├── ableton-extensions-sdk-1.0.0-beta.1.tgz
    └── ableton-extensions-cli-1.0.0-beta.1.tgz
```

Note that the generated `.gitignore` ignores `vendor/`. A fresh clone of the repo then can't `npm install` until the two tarballs are copied back in — either commit `vendor/` (remove it from `.gitignore`) or document the copy step.

Without UI, the generated `src/extension.ts` just logs `Hello from <name>! Your Live Set's tempo is: <tempo> bpm.`; with UI, it registers an `AudioClip` menu item that opens the dialog.

## Canonical config files

### `manifest.json`
Read by Live and by the CLI.

```json
{
    "name": "my-extension",
    "author": "Your Name",
    "entry": "dist/extension.js",
    "version": "1.0.0",
    "minimumApiVersion": "1.0.0"
}
```

- `entry` — the bundled output file (must match `build.ts` `outfile` / `package.json` `main`).
- `minimumApiVersion` — the lowest API version the extension needs; keep it equal to the string passed to `initialize(...)`.
- `name` + `version` also name the packaged file (`my-extension-1.0.0.ablx`).

### `package.json`
ESM project, CJS output. Note `"type": "module"`.

```json
{
  "name": "my-extension",
  "version": "1.0.0",
  "main": "dist/extension.js",
  "engines": { "node": ">=24.14.1" },
  "type": "module",
  "scripts": {
    "build": "tsc --noEmit && tsx build.ts --production",
    "build:dev": "tsc --noEmit && tsx build.ts",
    "start": "npm run build:dev && ableton-extensions-cli run",
    "package": "npm run build && ableton-extensions-cli package"
  },
  "dependencies": {
    "@ableton-extensions/sdk": "file:vendor/ableton-extensions-sdk-1.0.0-beta.1.tgz"
  },
  "devDependencies": {
    "@ableton-extensions/cli": "file:vendor/ableton-extensions-cli-1.0.0-beta.1.tgz",
    "@types/node": "^24.1.0",
    "esbuild": "0.28.1",
    "tsx": "^4.19.0",
    "typescript": "^5.9.3"
  }
}
```

- The CLI installs two equivalent binaries, `ableton-extensions-cli` (used by beta.1's templates and docs) and `extensions-cli` (older projects). Either works.
- Add `"vite": "^6.0.0"` to devDependencies when the extension has a UI.
- `npm start` runs `tsc --noEmit` first — a type error stops the launch.

### `tsconfig.json`

```json
{
  "compilerOptions": {
    "module": "nodenext",
    "target": "esnext",
    "moduleResolution": "nodenext",
    "outDir": "./dist",
    "rootDir": "./src",
    "strict": true,
    "esModuleInterop": true,
    "types": ["node"]
  },
  "include": ["src/**/*"]
}
```

(The examples additionally enable `noUncheckedIndexedAccess` and `exactOptionalPropertyTypes` — good practice given handle arrays are indexed.)

### `build.ts`
esbuild bundles `src/extension.ts` → the manifest's `entry`. `--production` minifies and drops sourcemaps. The host does **not** resolve `node_modules` at runtime, so everything must be bundled into this one file.

```typescript
import * as esbuild from "esbuild";
import * as fs from "node:fs";

const manifest = JSON.parse(fs.readFileSync("manifest.json", "utf8"));
const production = process.argv.includes("--production");

await esbuild.build({
  entryPoints: ["src/extension.ts"],
  outfile: manifest.entry,
  bundle: true,
  format: "cjs",      // Extension Host loads CommonJS
  platform: "node",
  sourcesContent: false,
  logLevel: "info",
  minify: production,
  sourcemap: !production,
  loader: { ".html": "text" }, // ONLY if importing .html (UI dialogs)
});
```

Any other bundler works too: `ableton-extensions-cli run` and `package` only care about the file named by `manifest.json`'s `entry`.

### `.env`
Per machine, gitignored. Tells the CLI where Live's Extension Host is. The CLI loads `.env` from the current directory.

```
EXTENSION_HOST_PATH=/path/to/ExtensionHostNodeModule.node
```

The module is always `ExtensionHostNodeModule.node`, in an `ExtensionHost/` folder inside the Live install, next to the `node` runtime that actually runs your extension:

- **Windows:** `<install root>\Program\ExtensionHost\ExtensionHostNodeModule.node`, e.g.
  `C:\ProgramData\Ableton\Live 12 Beta\Program\ExtensionHost\ExtensionHostNodeModule.node`
  (backslashes are fine in `.env`).
- **macOS:** inside the Live application bundle, in an `ExtensionHost` folder; pointing `--live` at the `.app` is enough.

`--live` accepts the Live `.app` (macOS), the Live `.exe` (Windows), an install root, or a direct path to `ExtensionHostNodeModule.node` (Execution guide).

### `src/html.d.ts` (UI projects)
Lets TypeScript accept `import html from "../ui/interface.html"`.

```typescript
declare module "*.html" {
  const content: string;
  export default content;
}
```

## npm scripts

| Script | What it does |
|---|---|
| `npm start` | dev build (sourcemaps, not minified) then `ableton-extensions-cli run` — loads into Live |
| `npm run build` | production bundle (minified, no sourcemaps) |
| `npm run build:dev` | dev bundle |
| `npm run package` | production build, then the `.ablx` archive |

Pass CLI flags through npm with `--`: `npm start -- --live "C:\ProgramData\Ableton\Live 12 Beta\Program\Ableton Live 12 Beta.exe"`.

## The CLI — `ableton-extensions-cli` (`@ableton-extensions/cli`)

```
ableton-extensions-cli run [dir]        # launch the already-built extension in Live's Extension Host
ableton-extensions-cli package [dir]    # zip the built extension into a .ablx (does NOT build)
ableton-extensions-cli help
```

**`run` flags** (unknown flags are rejected):
- `--live <path>` — override `EXTENSION_HOST_PATH` (see the accepted forms above).
- `--storage-directory <path>` / `--temp-directory <path>` — what the extension gets as `context.environment.storageDirectory` / `tempDirectory` (typed `string | undefined`). Pass them if your code uses the directories, and create the folders (`fs.mkdir(dir, { recursive: true })`) before writing.
- `-v`, `--verbose` — print the raw Extension Host log instead of the filtered view.

`run` prints `✓ Connected to Live. Press Ctrl+C to stop.` once connected, then streams your `console.*` output. If it can't connect, the usual causes are Developer Mode being off or a wrong `EXTENSION_HOST_PATH`.

There is no `--inspect` flag in beta.1 (the CLI rejects unknown flags). The generated VS Code `launch.json` runs `npm start` as a Node launch configuration.

**`package` flags:**
- `-o, --output <path>` — output `.ablx` path. Default: `<name>-<version>.ablx` in the extension folder (spaces in the name become `-`).
- `-i, --include <paths...>` — add files or folders (recursively) to the archive. Paths must be relative and inside the extension folder.

```sh
npx ableton-extensions-cli package -o dist/my-extension.ablx
npx ableton-extensions-cli package -i assets templates/index.html
```

An `.ablx` is a ZIP with `manifest.json`, the `entry` file at its manifest path, and the included paths at theirs. Users install it by dropping it onto the Extensions page in Live's Settings.

## Debugging & logs

- **Under `npm start`** (Developer Mode): your `console.log/info/warn/error` print in the terminal. The CLI filters out the host's own log lines; if something fails silently (an uncaught exception, a load error), rerun with `-v` to see the raw host log.
- **When Live hosts the extension** (Developer Mode off, installed `.ablx`): the same output goes to **`ExtensionHost.txt`**:
  - **Windows:** `C:\Users\<username>\AppData\Roaming\Ableton\Live x.x.x\Preferences\ExtensionHost.txt`
  - **macOS:** `/Users/<username>/Library/Preferences/Ableton/Live x.x.x/ExtensionHost.txt`

## Upgrading a project from 1.0.0-beta.0

From `<sdk-dir>/upgrade.md`:

1. Empty the project's `vendor/` folder (keep the folder).
2. Copy `ableton-extensions-sdk-1.0.0-beta.1.tgz` and `ableton-extensions-cli-1.0.0-beta.1.tgz` from `<sdk-dir>` into `vendor/`.
3. In `package.json`, change the two tarball paths from `beta.0` to `beta.1`. Keep the package names as they are — `@ableton-extensions/sdk` and `@ableton-extensions/cli`. (`upgrade.md` writes them as `@ableton/extensions-sdk` / `@ableton/extensions-cli`; those names don't exist.)
4. `npm install`, then `npm run build` to confirm.
5. Update Node to ≥ 24.16 if `npm install` warns `EBADENGINE`.

Also worth doing when upgrading: drop any `--inspect` from scripts (beta.1's CLI rejects it).

## Running the bundled examples

Each folder in `<sdk-dir>/examples/` is a complete extension whose dependencies point at `../../*.tgz`, so run it **in place** — a copied example folder can't `npm install`. The examples have no `.env`; pass the Live path:

```sh
cd <sdk-dir>/examples/context-menu
npm install
npm start -- --live "/Applications/Ableton Live 12 Beta.app"
# Windows: npm start -- --live "C:\ProgramData\Ableton\Live 12 Beta\Program\Ableton Live 12 Beta.exe"
```

For a project you can move around, use the creator instead.
