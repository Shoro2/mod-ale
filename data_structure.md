# data_structure — mod-ale

> Datei-/Ordner-Tree des Repos mit Größen-Hinweisen. Großdateien sind explizit markiert, weil sie KI-Lese-Limits sprengen können.

## Top-Level

```
mod-ale/
├── CMakeLists.txt              # Build-Entry: LUA_VERSION + Lua-Static-Flag, delegiert an src/lualib/.
├── README.md (~6 KB)           # Upstream-README: Installation, Lua-Versionen, Compat-Hinweise.
├── README_CN.md / README_ES.md # Übersetzungen (CN/ES).
├── COMMUNITY_UPDATES.md (~9 KB)# Upstream-Changelog (Community-Style).
├── LICENSE                      # GPL v3.
├── icon.png                     # Modul-Icon.
├── _config.yml                  # Jekyll-Setting (GH-Pages-Default).
├── .editorconfig / .gitattributes / .gitignore / .git_commit_template.txt
├── include.sh                   # AzerothCore-Modul-Hook (leer — keine extra SQL-Pfade nötig).
├── conf/                        # Worldserver-Konfig
├── docs/                        # Upstream-Engine-Doku (5 Files)
├── sql/                         # Leere SQL-Slots (engine ohne eigene Tabellen)
└── src/                         # C++-Quellcode
```

## `conf/`

```
conf/
└── mod_ale.conf.dist (~7 KB)   # ALE.Enabled, ScriptPath, AutoReload, BytecodeCache, RequirePaths,
                                 # Logging-Appender (ALELog/ALEConsole/Logger.ALE).
```

## `docs/` (Upstream-Engine-Doku)

```
docs/
├── INSTALL.md (~8 KB)          # Build-Schritte, Lua-Versionen, AzerothCore-Integration.
├── USAGE.md (~9 KB)            # Erste Skripte, Beispiel-Hooks, AIO-Hinweise.
├── IMPL_DETAILS.md (~12 KB)    # Engine-Internals, BindingMap, MoonScript, BytecodeCache.
├── CONTRIBUTING.md (~11 KB)    # PR-Guidelines, Code-Style, CI.
└── MERGING.md (~9 KB)          # Custom-Builds gegen Upstream halten.
```

## `sql/`

```
sql/
├── README.md (kurz)            # Hinweis auf Schema-Pfade.
├── auth/                        # leer
├── characters/                  # leer
└── world/                       # leer
```

ALE legt keine eigenen DB-Tabellen an — Persistenz für Skripte ist Sache des konsumierenden Moduls.

## `src/` (C++-Quellcode)

```
src/
├── ALE_loader.cpp (834 B)      # Modul-Loader: ruft AddSC_ALE().
├── ALE_SC.cpp (~40 KB)         # ScriptedAI-Wrappers: WorldScript/PlayerScript/CreatureScript/etc.,
                                 # die auf den Lua-State delegieren.  Groß — chunked lesen falls nötig.
├── LuaEngine/                   # Eigentliche Engine
└── lualib/                      # Vendored Lua/LuaJIT-Sources (Build-Static-Linking)
```

### `src/LuaEngine/` (Engine-Kern)

> ⚠ Lese-Limit-Warnung: `LuaFunctions.cpp` und `LuaEngine.cpp/h` und `Hooks.h` sind groß. Vor `Read` immer mit `grep` / `search_code` einen Symbol-Anker setzen.

| Datei | Größe | Zweck |
|-------|------:|-------|
| `LuaEngine.cpp` | ~54 KB | **(GROSS)** Haupt-Implementation: Lua-State-Mgmt, Script-Loading, Reload, Hook-Dispatch. |
| `LuaEngine.h` | ~30 KB | **(GROSS)** Public Engine-API + Klassendefinition. |
| `LuaFunctions.cpp` | ~96 KB | **(SEHR GROSS — chunked lesen!)** Registriert sämtliche Lua-Methoden für Player/Creature/GameObject/Item/Spell/Map/Group/Guild etc. |
| `Hooks.h` | ~30 KB | **(GROSS)** Enum aller verfügbaren Hook-Events (PlayerEvents, CreatureEvents, ServerEvents …). |
| `HookHelpers.h` | ~4 KB | Helper-Templates für den Hook-Dispatch. |
| `BindingMap.h` | ~9 KB | Generische Binding-Map (Hook-Slot → Lua-Function-Ref). |
| `ALETemplate.h` | ~12 KB | Template-Wrapper für C++-Object → Lua-Userdata. |
| `ALEEventMgr.cpp/.h` | ~8 KB | Event-Scheduler für `RegisterEvent`/timed callbacks. |
| `ALEFileWatcher.cpp/.h` | ~7 KB | Auto-Reload File-Watcher (Polling, `ALE.AutoReload`). |
| `ALECreatureAI.h` / `ALEInstanceAI.cpp/.h` | ~9 KB | Lua-getriebene CreatureAI / InstanceAI. |
| `ALECompat.cpp/.h` | ~4 KB | Lua-Versions-Kompat-Layer (5.1/5.2/5.3/5.4/JIT). |
| `ALEConfig.cpp/.h` | ~3 KB | Wrapper um `sConfigMgr` für die `ALE.*`-Keys. |
| `ALEDBCRegistry.cpp/.h` | ~1 KB | Registry für Lua-zugängliche DBC-Stores. |
| `ALEIncludes.h` | ~2 KB | Sammel-Include. |
| `ALEUtility.cpp/.h` | ~10 KB | String/Table-Helpers. |
| `HttpManager.cpp/.h` | ~8 KB | Optionaler HTTP-Client für Lua. |
| `lmarshal.cpp/.h` | ~17 KB | Lua-State-Serialisierung (für BytecodeCache). |
| `docs/` | – | Upstream Engine-Internal-Doku |
| `extensions/` | – | Optionale Lua-Side-Extensions |
| `hooks/` | – | Per-Subsystem Hook-Implementierungen (auto-included via `LuaEngine.cpp`) |
| `libs/` | – | Embedded Lua-Libraries |
| `methods/` | – | Per-Type Methoden-Tabellen (PlayerMethods, CreatureMethods …) |

### `src/lualib/`

```
lualib/
├── lua/                         # Vendored Lua 5.1–5.4 Sources
└── luajit/                      # Vendored LuaJIT
```

Wird bei Build je nach `LUA_VERSION` per `add_subdirectory` eingebunden.

## Empfohlene Lese-Reihenfolge für KI

1. `README.md` (Quick-Pitch + Lua-Versionen).
2. `INDEX.md` → `CLAUDE.md` (Rolle im Projekt) → `functions.md` (Mechaniken).
3. Bei API-Lookup: `grep` in `src/LuaEngine/LuaFunctions.cpp` nach Method-Name (z.B. `"GetGUIDLow"`), nicht am Stück lesen.
4. Bei Hook-Lookup: `src/LuaEngine/Hooks.h` Header-grep nach Event-Suffix (`"OnPlayerLogin"`).
5. Bei Reload-/Filewatcher-Verhalten: `ALEFileWatcher.cpp` / `LuaEngine.cpp::ReloadALE`.

## Bekannte Lese-Risiken

| Datei | Risiko | Workaround |
|-------|--------|------------|
| `src/LuaEngine/LuaFunctions.cpp` (~96 KB) | sprengt 25 K-Token-Limit | `grep -n` für Funktionsname → Section-Header lesen → `Read offset/limit` |
| `src/LuaEngine/LuaEngine.cpp` (~54 KB) | grenzwertig | Symbolisch suchen statt am Stück lesen |
| `src/LuaEngine/Hooks.h` (~30 KB) | knapp am Limit | meist als Stück lesbar, sonst chunken |
| `src/ALE_SC.cpp` (~40 KB) | grenzwertig | meist lesbar |
