# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## Project

Fork of [eez-studio](https://github.com/eez-open/studio) — an Electron application for embedded UI design (LVGL) and test & measurement instrument management. This fork is focused on **LVGL + EEZ-Flow**.

## Tech Stack

- **Runtime**: Electron (main + renderer processes)
- **Language**: TypeScript (strict, `noImplicitAny`, `strictNullChecks`)
- **UI**: React
- **Build**: TypeScript compiler (`tsc`) + Gulp bundler
- **State management**: MobX (`makeAutoObservable`, no legacy decorators)
- **Database**: SQLite (via `better-sqlite3`)
- **Tests**: Jest

## Project Structure

```
packages/
├── main/              # Electron main process (IPC, menus, lifecycle)
├── home/              # Home window renderer entry point
├── project-editor/    # Core: flows, LVGL pages, actions, variables, build
│   ├── core/          # Base object model, ProjectType enum
│   ├── features/      # Feature plugins (action, variable, page, font…)
│   ├── flow/          # Flow runtime, component registry
│   ├── lvgl/          # LVGL-specific widgets, actions, styles
│   ├── project/       # Project class, traits, Wizard UI
│   └── store/         # ProjectStore, features registry
├── eez-studio-shared/ # Shared types: extensions, IPC channels
├── eez-studio-types/  # Public TypeScript types (npm-publishable)
├── eez-studio-ui/     # Shared React UI components + LESS stylesheets
├── instrument/        # SCPI instrument communication
└── …                  # Other feature packages (notebook, shortcuts, …)
```

TypeScript `baseUrl` is `./packages`, so imports are bare package names:
```typescript
import { ProjectStore } from "project-editor/store";
import { IExtensionDefinition } from "eez-studio-shared/extensions/extension";
```

## Essential Commands

```bash
npm install              # Install dependencies (also runs electron-rebuild)
npm run watch            # Compile TypeScript + Gulp in watch mode (dev)
npm run start            # Launch Electron (run after watch in a separate terminal)
npm run build            # Full production build (tsc + gulp + CSS + WASM runtimes)
npm run build-src        # TypeScript + Gulp only (faster rebuild)
npm run dist             # Package with electron-builder
npm run typecheck        # tsc --noEmit — type-check without emitting
```

> **Dev workflow**: run `npm run watch` in one terminal, then `npm run start` in another.

## Electron Architecture

- **IPC**: Main↔renderer via typed channels in `packages/eez-studio-shared/`.
  Always use `contextBridge` + preload scripts. Never `nodeIntegration: true`.
- **Serialization**: No MobX class instances over IPC — serialize to plain objects first.
- **Multi-window**: Secondary `BrowserWindow` instances each have their own preload.

## Code Conventions

- **Naming**: `PascalCase` components/classes, `camelCase` functions/vars, `UPPER_SNAKE_CASE` constants.
- **MobX**: `makeAutoObservable(this)` in constructors. No legacy `@observable` decorators.
- **Types**: Avoid `any`. Use `unknown` + type guards at system boundaries.
- **No `require()`**: ES module imports only (except in extension `renderContent` late-binding — see `packages/notebook/extension.ts` for the pattern).

## Key Architectural Patterns

### ProjectType system
- `packages/project-editor/core/object.ts` — `ProjectType` enum
- `packages/project-editor/project/project-type-traits.ts` — trait classes, one per type
- `packages/project-editor/project/project.tsx` — `PROJECT_TYPE_NAMES`, feature visibility (`isOptional`)
- `packages/project-editor/project/ui/Wizard.tsx` — new-project dialog

### Feature/plugin system (`packages/project-editor/store/features.ts`)
Each `ProjectEditorFeature` has a `create()` factory and optional `check()`, `toJsHook()`, etc. Features are enabled per project via `Settings > General > Extensions`.

### Flow component registration
`packages/project-editor/flow/components/components-registry.ts` — components are filtered by project type. In this fork, components whose `name` includes `"/"` bypass the LVGL filter (allows cross-type components).

### EEZ-Flow extension API
Extensions live in `build/<name>/` and are loaded at startup. Entry point must `export default` an `IExtensionDefinition`. Use `eezFlowExtensionInit(eezStudio)` to register action components via `eezStudio.registerActionComponent(definition)`.

## roxer-actions Extension

Custom extension at `packages/roxer-actions/`:
- `package.json` — `"eez-studio": { "node-module": true }` marks it as a native extension
- `index.ts` — extension entry point, registers action components
- `flow-components.ts` — imported by `project-editor-create.tsx` for global registration

To add a new action: copy the template pattern in `index.ts`, write a `registerXxx(eezStudio)` function, call it in `eezFlowExtensionInit`.

To activate in a project: `Settings > General > Extensions > Add > "roxer-actions"` then reload the project.

## Variable / Type System Notes

- `packages/project-editor/features/variable/value-type.tsx` — fork patches: SYSTEM_STRUCTURES visible in LVGL context, `uint8`/`int8`/etc. included in `isValueTypeOf`.

## What to Avoid

- Do not import Node.js modules (`fs`, `path`, `child_process`) in renderer packages — go through IPC.
- Do not use `require()` except for intentional late-binding in extension `renderContent`.
- Do not modify the SQLite schema without a migration.
- Do not create circular dependencies across packages (use `madge-circular` script to check).
- The `.eez-project` format is JSON — all changes must be backward-compatible.

## Troubleshooting

- **Native module errors** (`better-sqlite3`, etc.): run `npx electron-rebuild`.
- **Blank window in dev**: ensure `npm run watch` has finished its first build before launching `npm run start`.
- **Type errors after pull**: run `npm run typecheck` — a shared type likely changed.
- **Extension not loaded**: verify `build/<extension-name>/` exists after `npm run build-src`; all directories in `build/` matching the extension pattern are auto-loaded.
