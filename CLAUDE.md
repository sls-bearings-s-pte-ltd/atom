# CLAUDE.md

Guidance for AI assistants (and human contributors) working in this repository.

## What this is

**Atom** is a hackable desktop text editor built on [Electron](https://github.com/electron/electron). This repo (`atom/atom`) is **Atom Core**: the main process, the renderer/workspace, the editor model, and a set of bundled first-party packages. Most user-facing features are actually implemented as packages (bundled here or fetched from atom.io); the core provides the runtime, APIs, and window/workspace scaffolding those packages plug into.

- **Language:** Mixed JavaScript (Standard Style) and CoffeeScript. New code should be JavaScript — CoffeeScript is legacy and being migrated away from.
- **Runtime:** Electron `2.0.18` (see `electronVersion` in `package.json`). Node built-ins, Electron modules, and DOM are all available depending on which process the code runs in.
- **Version:** `package.json` → `1.37.0-dev`. This is a historical/archived snapshot of the Atom codebase; upstream Atom was sunset by GitHub, so treat external links (atom.io, flight-manual) as potentially dead.
- **Two processes:** the **main process** (`src/main-process/`, Node/Electron) and the **renderer process** (everything else in `src/`, runs in the browser window). Keep the distinction in mind — IPC bridges them.

## Repository layout

```
src/                    Core source (main + renderer). Mix of .js and .coffee.
  main-process/         Electron main process: app lifecycle, windows, menus, auto-update, protocol handling.
  *.js / *.coffee       Renderer/core: editor model, workspace, panes, config, grammars, packages, git, etc.
exports/                Public require targets (require('atom') resolves here → exports/atom.js).
packages/               Bundled first-party packages (themes, git-diff, about, grammar-selector, ...).
                        Some are file: symlinked here; many others are pulled from atom.io (see package.json).
spec/                   Jasmine specs for core. *-spec.js / *-spec.coffee mirror files in src/.
  main-process/         Specs that run in the main process.
  integration/          Integration specs.
  fixtures/             Test fixtures.
script/                 Build/test/lint tooling (Node). Entry points: bootstrap, build, test, lint, clean, cibuild.
  lib/                  Individual build steps (transpile, snapshot, package, code-sign, installers, ...).
  vsts/                 Azure DevOps pipeline definitions.
static/                 LESS stylesheets, index.html, UI assets, babelrc for the app.
keymaps/                Core keybindings per platform (base/darwin/linux/win32 .cson).
menus/                  Core application menus per platform (.cson).
resources/              App icons and per-platform packaging resources (mac/win/linux).
apm/                    Pins the atom-package-manager (apm) version used for builds.
benchmarks/             Performance benchmarks.
dot-atom/               Default ~/.atom skeleton (init.coffee, keymap.cson, snippets, styles).
docs/                   Contributor docs, RFCs, build instructions, API doc generation notes.
vendor/                 Vendored third-party code.
```

## Development workflow

All tooling lives under `script/` and is invoked as `script/<name>` (there is a `.cmd` twin for Windows). These are Node scripts, **not** npm scripts — `npm test` just shells out to `node script/test`.

| Command | What it does |
| --- | --- |
| `script/bootstrap` | Install & link all dependencies (core, apm, script, bundled packages). Run this first. |
| `script/build` | Transpile (CoffeeScript/Babel/CSON/PEG.js), bundle, generate the startup snapshot, and produce an app under `out/`. |
| `script/test` | Run the test suite against the built app in `out/`. Requires a prior `script/build`. |
| `script/lint` | Run all linters: Standard (JS), coffeelint (CoffeeScript), stylelint (LESS). |
| `script/clean` | Remove build output, caches, and installed dependencies. |
| `script/cibuild` | The full CI pipeline (bootstrap → lint → build → test). |

Typical local loop: `script/bootstrap` once, then `script/build && script/test` after changes. Lint early with `script/lint`.

### Running tests selectively

`script/test` accepts flags (see `--help`):
- `--core-main` — core main-process specs only
- `--core-renderer` — core renderer specs only
- `--core-benchmark` — run benchmarks
- `--package` — run bundled package specs
- `--skip-main` — skip main-process specs

Tests run **inside a built Atom instance** (Electron), driven by Jasmine. `script/test` locates the executable in `out/` (`.app` on macOS, `atom-*/atom` on Linux, `atom.exe` on Windows), so you must build before testing.

### Requirements

- **Python 2.7** (`.python-version` → `2.7.13`) — required by node-gyp for native module builds.
- **Node** — CI uses `8.9.3` with npm `6.2.0` (`.travis.yml`). This is an old toolchain; expect to match it rather than use modern Node.
- Native build deps per platform (see `README.md` and `docs/build-instructions/`). A `Dockerfile` exists for building `.rpm` artifacts on Fedora.

## Coding conventions

Full details are in `CONTRIBUTING.md` (Styleguides section). Highlights:

### JavaScript
- Must pass **[JavaScript Standard Style](https://standardjs.com/)** (no semicolons, single quotes, 2-space indent). `script/lint` enforces it; `package.json` configures the `standard` env with `atomtest`, `browser`, `jasmine`, `node` and the `atom` / `snapshotResult` globals.
- Prefer object spread (`{...obj}`) over `Object.assign()`.
- Inline `export` with the declaration (`export default class Foo {}`).
- **Require order:** Node built-ins → Atom/Electron modules → local relative requires.
- **Class member order:** static methods/properties first, then instance methods/properties.

### CoffeeScript (legacy — prefer JS for new code)
- Enforced by `coffeelint.json` (arrow spacing, no debugger, English operators like `is`/`and`, spacing after commas, no standalone `@`, etc.).
- No spaces inside hash braces: `{a: 1}` not `{ a: 1 }`.
- Capitalize initialisms except the first word: `getURI`, `uriToOpen`.
- Use `this` instead of standalone `@`.

### LESS / styles
- Enforced by `stylelint.config.js` (extends `stylelint-config-standard`, with a set of rules relaxed for Atom/octicons). `static/atom.less` is excluded from linting.

### Specs
- Jasmine specs in `spec/` (and each package's `spec/`), named `*-spec.js` / `*-spec.coffee`, mirroring the source file.
- `describe` reads as a noun/situation; `it` reads as a statement about state.

### Git commit messages
- Present tense, imperative mood ("Add feature", not "Added feature").
- First line ≤ 72 chars; reference issues/PRs after the first line.
- Add `[ci skip]` to the title for docs-only changes.
- Emoji prefixes are conventional (see `CONTRIBUTING.md`): 🐛 bug, 🎨 formatting, 🏇 perf, ✅ tests, etc. — optional; don't add if unsure.

## Working with packages

- Bundled packages live under `packages/` and follow the standard Atom package layout: `package.json`, `lib/`, `spec/`, `styles/`, plus optional `keymaps/`, `menus/`, `grammars/`.
- `package.json` at the repo root declares two package lists: `dependencies` (how each package is fetched — either `file:packages/...` for in-tree packages or an `atom.io` tarball URL) and `packageDependencies` (the versions bundled into a release). Keep these consistent when changing a bundled package version.
- The public API surface consumed by packages is re-exported through `exports/atom.js` (this is what `require('atom')` resolves to). Core managers like `Workspace`, `TextEditor`, `Config`, `CommandRegistry`, `GrammarRegistry`, `PackageManager`, etc. live in `src/`.

## Key architectural touchpoints

- **Entry point:** `package.json` `main` → `src/main-process/main.js` (Electron main process bootstrap).
- **Main process** (`src/main-process/`): `atom-application.js` (app singleton), `atom-window.js` (windows), `application-menu.js`, `auto-update-manager.js`, `parse-command-line.js`.
- **Window bootstrap:** `src/initialize-application-window.coffee`, `src/initialize-test-window.coffee`, `src/initialize-benchmark-window.js`.
- **Editor core:** `src/text-editor.js`, `src/text-editor-component.js`, `src/text-editor-element.js`, `src/selection.js`, `src/cursor.js`, `src/decoration*.js`.
- **Workspace/UI model:** `src/workspace.js`, `src/pane.js`, `src/pane-container.js`, `src/dock.js`, `src/panel*.js`.
- **Grammars/syntax:** `src/grammar-registry.js`, `src/text-mate-language-mode.js`, `src/tree-sitter-language-mode.js` (both TextMate and Tree-sitter grammars are supported).
- **Config:** `src/config.js`, `src/config-schema.js`, `src/config-file.js`.
- **Startup snapshot:** the build uses `electron-link` + `electron-mksnapshot` (`script/lib/generate-startup-snapshot.js`) to snapshot core modules; `snapshotResult` is a runtime global — be aware some modules are snapshot-sensitive.

## CI

- **Travis** (`.travis.yml`): Linux; runs `script/lint && script/build ... && script/test`, produces `.deb`/`.rpm`/`.tar.gz` artifacts.
- **AppVeyor** (`appveyor.yml`): Windows.
- **Azure DevOps / VSTS** (`script/vsts/`): production/nightly/release pipelines.

## Gotchas for AI assistants

- **Two languages, two processes.** Before editing, check whether a file is `.js` or `.coffee` and whether it runs in the main or renderer process — match the existing style of the file you're in.
- **Don't reformat wholesale.** Match the surrounding file. Standard Style for JS, coffeelint rules for CoffeeScript. Run `script/lint` before committing.
- **Build before test.** `script/test` runs against the built app in `out/`, not the raw source.
- **Old toolchain.** Node 8, npm 6, Python 2.7, Electron 2. Don't assume modern JS runtime features or tooling are available.
- **Bundled-package version bumps** touch both `dependencies` and `packageDependencies` in the root `package.json`.
- **This is an archived codebase.** External services and links may be dead; prefer reading the code over trusting doc URLs.
