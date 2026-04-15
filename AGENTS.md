# AGENTS.md

This file provides guidance to WARP (warp.dev) when working with code in this repository.

## Project summary
This is a Raycast extension (`mise-manager`) that wraps the `mise` CLI to manage tools, plugins, settings, and tasks from Raycast UI commands.

Important prerequisite from `README.md`: `mise` must be installed and available on PATH (or discoverable in common install locations).

## Development commands
Run from repository root:

- Install dependencies: `npm install`
- Start local extension development: `npm run dev`
- Build extension: `npm run build`
- Lint: `npm run lint`
- Auto-fix lint issues: `npm run fix-lint`
- Publish to Raycast store: `npm run publish`

Direct per-file linting (useful as a “single-target” check):

- `npx eslint src/show-installed.tsx`

TypeScript check (no dedicated script exists):

- `npx tsc --noEmit`

Tests:

- There is currently no test framework or test script configured in this repository (`package.json` has no `test` script and no test config/files were found).
- “Run a single test” is not applicable until a test runner is added.

## High-level architecture
### 1) Command surface is declared in `package.json`
Raycast commands are registered in `package.json` under `commands`. Each command name maps to a same-named entry file in `src/` (for example `show-installed` → `src/show-installed.tsx`).

### 2) `src/tools/mise.ts` is the core integration layer
Most extension logic calls into `src/tools/mise.ts`, which:

- Resolves the `mise` binary across PATH and common install locations (plus optional `MISE_BIN` override).
- Executes CLI calls via `execFile` (`runMise`) with controlled environment defaults (`MISE_YES=1`, `MISE_NO_COLOR=1`, `MISE_SKIP_VERSION_CHECK=1`) and extended PATH.
- Normalizes/parsers outputs into typed objects (`listInstalledTools`, `listOutdatedTools`, `listRegistryEntries`, `listTasks`, etc.).
- Exposes all mutation operations (install/uninstall/upgrade, plugin operations, settings update, prune/cache/reshim).

When adding new behavior, prefer extending `src/tools/mise.ts` first, then consume from UI commands.

### 3) Shared types and error handling
- `src/tools/types.ts` defines contracts used by command UIs.
- `src/tools/utils.ts` provides `MiseCommandError`, ANSI-stripped error formatting (`formatMiseError`), and registry backend URL derivation.

UI commands should consistently surface errors using `formatMiseError`.

### 4) UI layer pattern in `src/*.tsx`
Most view commands follow this pattern:

- Use `usePromise(...)` to load data from `tools/mise`.
- Render `List`/`Detail`/`Form` from `@raycast/api`.
- Provide optimistic feedback through `showToast`.
- Trigger mutations via actions, then revalidate list data.

Reusable UI pieces currently live in `src/components/`:

- `ToolSection.tsx`: installed tool/version management interactions.
- `SearchResultItem.tsx`: registry search result actions and install flows.

### 5) Two command styles
- **View commands** (`mode: "view"`): interactive lists/forms/details (e.g. `show-installed`, `search`, `settings`).
- **No-view commands** (`mode: "no-view"`): immediate side effects + toast status (e.g. `prune`, `clean-cache`, `upgrade-mise`, `reshim`, `update-plugins`).

For new no-view commands, keep behavior short-lived and user-visible via toast outcomes.

## Repository-specific implementation notes
- Keep `mise` process interactions in `runMise` to preserve environment consistency and error behavior.
- Some `mise` JSON outputs vary in shape; `listOutdatedTools` demonstrates defensive normalization. Follow that approach for new parsers.
- `tsconfig.json` is strict; maintain explicit typing for parsed CLI payloads.
- Linting relies on `@raycast/eslint-config` via `eslint.config.js`; align new code with existing Raycast extension conventions.
