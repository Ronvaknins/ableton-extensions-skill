# Scaffolding, building, running & packaging

## Prerequisites

- **Node.js >= 24.14.1** (the generated `package.json` pins `engines.node` to this).
- An **Ableton Live beta build** that supports Extensions, with **Developer Mode** enabled in *Preferences → Extensions*.
- The SDK distribution unpacked at `C:\DEV\extensions-sdk-1.0.0-beta.0`.

## Creating a new project

Run the bundled `create-extension` tool via `npx`, pointing at its tarball. Pass an optional target directory:

```sh
npx file:C:/DEV/extensions-sdk-1.0.0-beta.0/ableton-create-extension-1.0.0-beta.0.tgz my-extension
```

Interactive prompts (in order):
1. **Extension name** — sanitized to lowercase alphanumeric + hyphens; defaults to the directory name.
2. **Author**.
3. **Path to Ableton Live** — auto-detected on macOS/Windows; choose "Other" to type it manually. Used to locate the Extension Host module.
4. **UI support?** — if yes, scaffolds a WebView modal dialog (`ui/interface.html`, `src/html.d.ts`, Vite, the `.html` esbuild loader).

Flags: `--skip-install` (skip the automatic `npm install`).

The tool copies the SDK and CLI tarballs into the project's `vendor/` and wires `package.json` dependencies to `file:./vendor/<name>.tgz`. If VS Code is on `PATH`, it also writes `.vscode/launch.json` and `.vscode/tasks.json`.

### Generated layout

```
my-extension/
├── .env                 # EXTENSION_HOST_PATH (gitignored, per-machine)
├── .gitignore
├── README.md
├── manifest.json        # extension metadata Live reads
├── package.json
├── tsconfig.json
├── build.ts             # esbuild bundler script
├── src/
│   └── extension.ts     # entry point — exports activate()
├── ui/                  # only if UI support chosen
│   └── interface.html
├── src/html.d.ts        # only if UI — declares *.html module
├── .vscode/             # only if VS Code detected
│   ├── launch.json      # "Debug Extension" → npm start
│   └── tasks.json
└── vendor/
    ├── ableton-extensions-sdk-1.0.0-beta.0.tgz
    └── ableton-extensions-cli-1.0.0-beta.0.tgz
```

## Canonical config files

### `manifest.json`
Read by Live to identify and load the extension.

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
- `minimumApiVersion` — the lowest SDK API version the extension needs; should match the string passed to `initialize(...)`.

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
    "start": "npm run build:dev && extensions-cli run",
    "package": "npm run build && extensions-cli package"
  },
  "dependencies": {
    "@ableton-extensions/sdk": "file:./vendor/ableton-extensions-sdk-1.0.0-beta.0.tgz"
  },
  "devDependencies": {
    "@ableton-extensions/cli": "file:./vendor/ableton-extensions-cli-1.0.0-beta.0.tgz",
    "@types/node": "^24.1.0",
    "esbuild": "0.28.0",
    "tsx": "^4.19.0",
    "typescript": "^5.9.3"
  }
}
```

Add `"vite": "^6.0.0"` to devDependencies when the extension has a UI.

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

(Examples additionally enable `noUncheckedIndexedAccess`, `exactOptionalPropertyTypes` — good practice given handle arrays are indexed.)

### `build.ts`
esbuild bundles `src/extension.ts` → the manifest's `entry`. `--production` minifies and drops sourcemaps.

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

### `.env`
Per-machine, gitignored. Tells the CLI where Live's Extension Host module is.

```
EXTENSION_HOST_PATH=/path/to/ExtensionHostNodeModule.node
```

### `src/html.d.ts` (UI projects)
Lets TypeScript accept `import html from "./interface.html"`.

```typescript
declare module "*.html" {
  const content: string;
  export default content;
}
```

## npm scripts

| Script | What it does |
|---|---|
| `npm start` | dev build (sourcemaps, not minified) then `extensions-cli run` — loads into Live |
| `npm run build` | production bundle (minified, no sourcemaps) |
| `npm run build:dev` | dev bundle |
| `npm run package` | production build, then `.ablx` archive |

## The CLI — `extensions-cli` (`@ableton-extensions/cli`)

```
extensions-cli run [dir]        # build output already present; launch in Live's Extension Host
extensions-cli package [dir]    # zip the built extension into a .ablx
extensions-cli help
```

**`run` flags:**
- `--live <path>` — override `EXTENSION_HOST_PATH`.
- `--storage-directory <path>` / `--temp-directory <path>` — override the extension's storage/temp dirs.
- `--inspect` — enable the VS Code debugger (Node `--inspect`).

**`package` flags:**
- `-o, --output <path>` — output `.ablx` path (default under `dist/`).
- `-i, --include <paths...>` — bundle extra files/dirs (assets, templates) into the archive. Repeatable.

```sh
npx extensions-cli package -o dist/my-extension.ablx -i assets
```

An `.ablx` is a ZIP containing the bundled `extension.js`, `manifest.json`, and any included assets. Users install it by dropping it into Live's Extensions settings.

## Debugging & logs

`console.log/info/warn/error(...)` and uncaught-exception stack traces go to **`ExtensionHost.txt`**:

- **Windows:** `\Users\<username>\AppData\Roaming\Ableton\Live x.x.x\Preferences\ExtensionHost.txt`
- **macOS:** `/Users/<username>/Library/Preferences/Ableton/Live x.x.x/ExtensionHost.txt`

In VS Code, the generated `launch.json` runs `npm start` under the debugger ("Debug Extension", F5). For ad-hoc attach, run `extensions-cli run --inspect`.
