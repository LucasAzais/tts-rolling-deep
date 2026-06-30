## Quick context

This repository is a Tabletop Simulator (TTS) mod implemented as Lua scripts. Files ending with `.ttslua` are modules loaded with `require()` and wired up from `Global.ttslua` which is the TTS entrypoint.

Key entry points:
- `Global.ttslua` — onLoad/onSave, initializes the system (EventManager, Ui, Options, GlobalUi, Chapter).
- `EventManager.ttslua` — central event routing. It injects small relay functions into object scripts and manages global and object-scoped handlers.
- `Ui.ttslua` and `GlobalUi.ttslua` — UI builder helpers and the main setup/menu implementation.
- `Constants.ttslua`, `Utils.ttslua`, `Log.ttslua` — shared primitives, tags and helpers used across modules.

## Big picture / architecture

- Modular Lua files are treated as classes/modules and returned as tables with functions (e.g. `return { init = ... }`).
- Global lifecycle: `Global.onLoad` sets up logging, EventManager, Options, UI and Chapter state. Persisted state is encoded with `JSON.encode` in `onSave`.
- Event routing: `EventManager` supports two modes:
  - Global events: `EventManager.addHandler(event, handler)` patches `_G[event]` to chain handlers.
  - Object events: `EventManager.addHandler(event, handler, object)` injects a per-object handler into the object's Lua script. This modifies `object.setLuaScript()` and tags the object with `ScriptedObject`.
  Be careful when changing event names or the relay code — EventManager edits object scripts at runtime.

## Project-specific conventions

- Naming: Classes are PascalCase, functions camelCase, globals/constants UPPER_SNAKE or underscore_separated. See top of `Global.ttslua`.
- UI: Use `Ui.getRoot()` or `Ui.createRootOnObject()` and the builder methods (`panel`, `button`, `text`, `toggleButton`, etc.). UI elements register callbacks with `onClick`, `onValueChanged`, and follow the pattern in `GlobalUi.initSetupUi`.
- Tags/GUIDs: many modules find game objects by tags. See `Constants.ttslua` for the canonical tags the code expects (e.g. `ScoringBook`, `ChapterBook`, `ScoreMarker`, `TargetMarker`). Always prefer tagging objects in the TTS scene rather than hard-coding GUIDs.
- Logging: Use `Log.ForModule('ModuleName')` to create module-scoped logs and respect log levels defined in `Log.LEVEL_BY_MODULE`.

## Important integration points / gotchas for code changes

- EventManager updates object scripts when handlers change. When adding or removing object handlers, expect `object.getLuaScript()` to be rewritten. Avoid making incompatible changes to injected event signatures.
- UI uses `Global.UI.getXmlTable()` and `object.UI.getXmlTable()`. The `Ui` wrapper builds XML-like tables and calls `.refresh()`/`.globalRefresh()` to push changes; see `Ui.globalRefresh` usage in `Global.ttslua`.
- Persisted state: `Options.save()` and `EventManager.save()` shape the JSON returned by `onSave()`. Keep their output stable if you change saved fields.
- Many modules rely on `getObjectsWithTag(...)` and `getObjectFromGUID()` — tests or debug runs in TTS require the scene to have the expected tags/GUIDs.

## Examples (copy-paste friendly)

- Add an object-scoped handler:

```lua
local EventManager = require('EventManager')
EventManager.addHandler('onDrop', function(obj, params)
  -- obj is the object that triggered the event
  -- params are event-specific values forwarded by EventManager
  print('dropped by', obj.getGUID())
end, myObject)
```

- Create a simple setup button in the UI (see `GlobalUi.initSetupUi`): the callback signature is `function()` (no implicit args) and can call `Setup.setup()`.

## Developer workflow notes

- There is no repository-level build script checked in, but the project references Benjamin-Dobell/luabundle for organizing TTS projects (see `Global.ttslua` comment). If you use luabundle or a local bundler, follow that tool's README to package scripts for TTS.
- Runtime testing is done inside Tabletop Simulator: place tagged objects (see `Constants.ttslua`), open the mod, and use the setup UI (Global menu) to run scripted setup flows.

## Files to inspect for common tasks

- Add UI: `Ui.ttslua`, `GlobalUi.ttslua`
- Add events: `EventManager.ttslua`, `Global.ttslua` (relay)
- Shared helpers: `Utils.ttslua`, `Constants.ttslua`, `Log.ttslua`
- Game state & flow: `Chapter.ttslua`, `Setup.ttslua`, `Options.ttslua`

## If something is missing

- Some requires reference `ScoreTrack` (e.g. `Setup.ttslua`). If you can't find a module, check the build/bundle output or other folders (in-game object scripts sometimes live under `In-Game Scripts/`).

---

If any section is unclear or you'd like extra examples (event handler patterns, UI element recipes, or a short checklist for safely changing object-injected scripts), tell me which area and I'll expand or revise this file.
