# CLAUDE — mod-ale

> Inhalt/Zweck dieses Repos. Mechaniken: siehe [`functions.md`](./functions.md). Datei-Tree: siehe [`data_structure.md`](./data_structure.md).

## Was ist das?

**ALE = AzerothCore Lua Engine.** Eigenständiger Fork des [Eluna](https://github.com/ElunaLuaEngine/Eluna)-Projekts, spezifisch für AzerothCore optimiert. Stellt eine Lua-/MoonScript-Skript-Runtime im `worldserver`-Prozess bereit und exposiert die AzerothCore-API (`Player`, `Creature`, `GameObject`, DB-Zugriff, Hooks etc.) gegenüber Lua-Code.

Anders als reguläre Gameplay-Module enthält mod-ale **keine Spielmechanik** — es ist die **Infrastruktur**, auf der die anderen Lua-/AIO-basierten Module dieses Projekts laufen.

## Rolle im Gesamtprojekt

mod-ale ist eine **harte Build- und Laufzeit-Abhängigkeit** für jedes Modul, das Lua-/AIO-Code mitbringt:

| Konsument | Lua-Pfad | Funktion die ALE bereitstellt |
|-----------|----------|--------------------------------|
| `share-public/AIO_Server/` | `lua_scripts/AIO/` | Server-Client-Messaging-Framework |
| `share-public/AIO_Server/Dep_Validation/` | `lua_scripts/Dep_Validation/` | Shared `_G.Validate`-Library |
| `mod-paragon` | `Paragon_System_LUA/Paragon_*.lua` | `CharDBQuery`, `Player:*`, AIO-Handler-Registry |
| `mod-paragon-itemgen` | `Paragon_System_LUA/ItemGen_*.lua` | Player-/Item-Hooks, AIO |
| `mod-loot-filter` | `Loot_Filter_LUA/LootFilter_*.lua` | DB-Zugriff, AIO |
| `mod-endless-storage` | `lua_scripts/Storage/endless_storage_*.lua` | DB-Zugriff, AIO |

Wird ALE nicht geladen oder Lua-Skripte schlagen beim Reload fehl, sind alle UI-Frames dieser Module tot, obwohl deren C++-Backends funktionieren.

## Origin & Divergenz

- **Upstream**: `azerothcore/mod-ale` (community-maintained Fork von Eluna).
- **Diese Repo-Kopie**: `shoro2/mod-ale` (master). Aktuell **kein lokaler Custom-Patch** — reine Spiegelung. Falls Custom-Änderungen aufgenommen werden, gehören sie als Custom-Commit oben in [`log.md`](./log.md) und cross-vermerkt in `share-public/claude_log.md`.
- **Inkompatibel mit Original-Eluna**: API-Differenzen sind dokumentiert in [`README.md`](./README.md) und [`docs/USAGE.md`](./docs/USAGE.md).

## Konfiguration (Custom-Daten-Index)

Konfig-Datei: [`conf/mod_ale.conf.dist`](./conf/mod_ale.conf.dist). Wichtige Optionen für unser Projekt:

| Key | Default | Bedeutung |
|-----|---------|-----------|
| `ALE.Enabled` | `true` | Engine an/aus |
| `ALE.ScriptPath` | `"lua_scripts"` | Worldserver-relative Pfad zu Skript-Quellen. Hier liegen `AIO/`, `Storage/`, `Paragon_System_LUA/` etc. |
| `ALE.AutoReload` | `false` | File-Watcher: triggert Reload bei `.lua`-Änderungen. Empfehlung: dev=true, prod=false. |
| `ALE.AutoReloadInterval` | `1` | Polling-Intervall in s |
| `ALE.BytecodeCache` | `true` | In-Memory-Bytecode-Cache; beschleunigt `.reload ALE` |
| `ALE.TraceBack` | `false` | Lua-Errors mit `debug.traceback` |
| `ALE.RequirePaths` / `ALE.RequireCPaths` | `""` | Zusätzliche `package.path` / `package.cpath` Einträge |
| `ALE.PlayerAnnounceReload` | `false` | Reload-Broadcast an Spieler (low security) |

Build-Time-Option (`CMakeLists.txt`):

| Variable | Werte | Default |
|----------|-------|---------|
| `LUA_VERSION` | `luajit` / `lua51` / `lua52` / `lua53` / `lua54` | `lua52` |
| `LUA_STATIC` | `ON` / `OFF` | `ON` |

## Slash- / Console-Commands

Reload-Command: `.reload ALE` (Worldserver Console / GM-Chat). Lädt alle Skripte unter `ALE.ScriptPath` neu. Beim Reload: existierende `AIO.AddHandlers`-Registrierungen sind durch `_G`-Cache überlebbar, neu zugewiesene Handler-Closures gewinnen — Re-Registrierungs-Falle siehe [`share-public/docs/04-aio-framework.md`](https://github.com/Shoro2/share-public/blob/main/docs/04-aio-framework.md).

## Sicherheit / SQL-Injection

ALE selbst macht keine `CharDBExecute`/`CharDBQuery`-Calls mit player-supplied Input — es **stellt die API bereit**. Verantwortung für Input-Validation liegt beim **konsumierenden Lua-Code**. Pattern:

```lua
local Validate = require("validation") -- share-public/AIO_Server/Dep_Validation
if not Validate.IntInRange(statId, 1, 17) then return end
```

Status pro Konsument: siehe `<repo>/todo.md` ("SQL-Injection-Risiko"-Items).

## DB / SQL

`sql/`-Unterverzeichnisse (`auth/`, `characters/`, `world/`) sind aktuell **leer** (nur `sql/README.md`). Die Engine selbst legt keine Tabellen an. Falls ALE in Zukunft Persistenz braucht, gehört das nach `sql/world/` (analog AzerothCore-Konvention).

## Lizenz

GPL v3. Siehe [`LICENSE`](./LICENSE).

## Cross-Refs

- [`functions.md`](./functions.md) — Hook-Klassen, Lua-API-Surface, EventMgr, FileWatcher, BytecodeCache.
- [`data_structure.md`](./data_structure.md) — Datei-/Ordner-Tree mit Lese-Hinweisen für die großen Files (`LuaFunctions.cpp` ~96 KB, `LuaEngine.cpp` ~54 KB, `Hooks.h` ~30 KB).
- [`log.md`](./log.md) — Custom-Commits seit Onboarding.
- [`todo.md`](./todo.md) — offene Aufgaben.
- [`docs/`](./docs/) — Upstream-Engine-Doku (INSTALL/USAGE/IMPL_DETAILS/MERGING/CONTRIBUTING).
- Projekt-weite Doku: [`share-public/AI_GUIDE.md`](https://github.com/Shoro2/share-public/blob/main/AI_GUIDE.md), [`docs/04-aio-framework.md`](https://github.com/Shoro2/share-public/blob/main/docs/04-aio-framework.md).
