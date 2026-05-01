# functions — mod-ale

> Mechaniken, Hook-Klassen, Lua-API-Surface und Konfig-Wirkung. Inhalt/Zweck: [`CLAUDE.md`](./CLAUDE.md). Datei-Tree: [`data_structure.md`](./data_structure.md).

## Architektur in 30 Sekunden

```
worldserver Prozess
└── ScriptMgr (AzerothCore)
    └── AddSC_ALE()                            ← src/ALE_loader.cpp
        ├── WorldScript "ALE_World"            ← src/ALE_SC.cpp
        ├── PlayerScript "ALE_Player"          ← src/ALE_SC.cpp
        ├── CreatureScript "ALE_Creature"      ← src/ALE_SC.cpp
        ├── GameObjectScript / ItemScript / ...
        └── Lua State (per worldserver-Instanz)
            ├── BindingMap: Hook-Slot → Lua-Function-Ref
            ├── EventMgr   : timed callbacks (RegisterEvent)
            ├── BytecodeCache (optional)
            └── lädt rekursiv alle .lua/.moon unter ALE.ScriptPath
```

Jeder C++-`*Script`-Hook von ALE ruft den passenden Eintrag in der `BindingMap` auf und reicht die WoW-Objekte als Lua-Userdata weiter.

## Hook-Klassen (Lua-API)

Vollständige Liste: [`src/LuaEngine/Hooks.h`](./src/LuaEngine/Hooks.h) (Enum-Sektionen).

| Klasse | Beispiel-Events | Lua-Registrierung |
|--------|------------------|-------------------|
| Server | `WORLD_EVENT_ON_OPEN_STATE_CHANGE`, `WORLD_EVENT_ON_CONFIG_LOAD` | `RegisterServerEvent(eventId, function)` |
| Player | `PLAYER_EVENT_ON_LOGIN/LOGOUT/MAP_CHANGE/KILL_CREATURE/CHAT/COMMAND` | `RegisterPlayerEvent(eventId, function)` |
| Creature | `CREATURE_EVENT_ON_ENTER_COMBAT`, `CREATURE_EVENT_ON_DIED` | `RegisterCreatureEvent(entry, eventId, function)` |
| GameObject | `GAMEOBJECT_EVENT_ON_USE` | `RegisterGameObjectEvent(entry, eventId, function)` |
| Item | `ITEM_EVENT_ON_USE`, `ITEM_EVENT_ON_QUEST_ACCEPT` | `RegisterItemEvent(entry, eventId, function)` |
| Spell | `SPELL_EVENT_ON_CAST` | `RegisterSpellEvent(spellId, eventId, function)` |
| Group | `GROUP_EVENT_ON_MEMBER_ADD/REMOVE` | `RegisterGroupEvent(eventId, function)` |
| Guild | `GUILD_EVENT_ON_LOGIN`, `GUILD_EVENT_ON_INVITE` | `RegisterGuildEvent(eventId, function)` |
| Map | `MAP_EVENT_ON_CREATE`, `INSTANCE_EVENT_ON_PLAYER_ENTER` | `RegisterMapEvent / RegisterInstanceEvent` |
| BattleGround | `BG_EVENT_ON_START/END` | `RegisterBGEvent(eventId, function)` |

Alle Registrierungen geben einen Numeric-Handle zurück, mit dem per `RemoveEventByHandle` wieder de-registriert werden kann.

## Globale Lua-Funktionen (Auswahl)

Quelle: [`src/LuaEngine/LuaFunctions.cpp`](./src/LuaEngine/LuaFunctions.cpp) (~96 KB — gezielt grep-en).

### Datenbank

```lua
local q = WorldDBQuery("SELECT entry FROM creature_template WHERE entry = 12345")
WorldDBExecute("UPDATE creature_template SET ... WHERE entry = 12345")

local q = CharDBQuery("SELECT name FROM characters WHERE guid = " .. guid)
CharDBExecute("UPDATE characters SET ... WHERE guid = " .. guid)

local q = AuthDBQuery("SELECT id FROM account WHERE username = '" .. nameSan .. "'")
AuthDBExecute(...)
```

`*Query` ist **synchron** (blockiert bis Result da). `*Execute` ist **asynchron** (Fire-and-forget). Beide nehmen einen rohen SQL-String — Eluna/ALE bietet **keine Prepared Statements** für Lua. SQL-Injection-Mitigation ist Sache des konsumierenden Codes (siehe `share-public/AIO_Server/Dep_Validation/validation.lua`).

> **Projekt-Hinweis**: Wegen fehlender PreparedStatements ist Konvention im Projekt:
> 1. **Numerische Inputs** vor Concat per `Validate.IntInRange` prüfen.
> 2. **Strings** per Whitelist (Tabellen-Lookup) nicht per `string.format("'%s'", ...)`.
> 3. Race-Conditions bei mehreren async-`*Execute`-Calls auf dieselbe Row vermeiden, indem alle Spalten in **einem** SQL-Statement gesetzt werden (siehe `mod-paragon` M2-Fix vom 2026-05-01: `Paragon.UpdateAllocationAndUnspent` verwendet bewusst `CharDBQuery` statt `CharDBExecute` für synchrone, atomare Updates).

### Player-API (häufig genutzt)

```lua
player:GetGUIDLow()        -- characterID
player:GetAccountId()
player:GetSession():GetAccountId()
player:GetName()
player:GetLevel() / GetClass() / GetRace() / GetGender()
player:AddItem(entry, count)
player:RemoveItem(entry, count)
player:HasAura(spellId) / AddAura(spellId, target) / RemoveAura(spellId)
player:SendBroadcastMessage(text)
player:SendNotification(text)
player:GetGroup() / GetGuild()
```

### Timed Events (`EventMgr`)

```lua
local handle = player:RegisterEvent(function(eventId, delay, repeats, p)
    p:SendBroadcastMessage("Tick")
end, 1000, 5)   -- 1000 ms delay, 5 repeats (0 = infinite)

player:RemoveEventById(handle)
player:RemoveEvents()
```

Auch global: `CreateLuaEvent(callback, delay, repeats)` (kein Owner).

### AIO (Server-Client-Messaging)

AIO selbst liegt in `share-public/AIO_Server/` (Lua-Side) und wird von ALE als Skript geladen — ALE selbst hat **keine** AIO-API in C++. Die `AIO.AddHandlers` / `AIO.Handle` / `AIO.AddAddon` / `AIO.AddAddonCode`-Funktionen sind reines Lua. Doku: [`share-public/docs/04-aio-framework.md`](https://github.com/Shoro2/share-public/blob/main/docs/04-aio-framework.md).

## Reload-Verhalten

- Trigger: `.reload ALE` (Console / GM-Chat).
- Datei-Watcher (optional, `ALE.AutoReload = true`): pollt das Script-Verzeichnis im `ALE.AutoReloadInterval`-Takt.
- Beim Reload werden **alle** `.lua`/`.moon` unter `ALE.ScriptPath` neu geladen. Globale Tabellen (`_G.Foo = Foo or {}`) überleben, frische `local`-Closures gewinnen.
- AIO-Handler werden ebenfalls neu registriert — die zugehörige Re-Registrierungs-Falle (alte Closure noch im Memory, falls sie vor Reload Referenzen gesammelt hat) ist projekt-weit dokumentiert in `share-public/docs/04-aio-framework.md`.

## BytecodeCache

`ALE.BytecodeCache = true` (default). Skripte werden nach erster Kompilierung als Lua-Bytecode in einer In-Memory-Tabelle gehalten. `.reload ALE` profitiert davon, weil unmodifizierte Files nicht neu geparst werden müssen. Implementierung: `lmarshal.cpp` (Lua-State-Serialisierung).

Cache-Invalidation: file mtime-Check. Komplette Cache-Leerung nur bei Server-Restart.

## Logging

ALE definiert eigene Logger:
- `ALELog` → Datei `ALE.log` (LogLevel ≥ Debug)
- `ALEConsole` → Worldserver-Konsole (LogLevel ≥ Info)

Aus Lua: `print(...)` schreibt in beide. C++-Side: `LOG_INFO("ALE", ...)` etc.

## Konfig-Wirkung (zur Laufzeit)

| Key | Wirkung-Punkt im Code |
|-----|------------------------|
| `ALE.Enabled` | `LuaEngine::Initialize()` skippt früh wenn false |
| `ALE.ScriptPath` | `LuaEngine::LoadScripts()` rekursiver Walk-Root |
| `ALE.AutoReload` | `ALEFileWatcher` Konstruktor / Start |
| `ALE.AutoReloadInterval` | `ALEFileWatcher::CheckInterval` |
| `ALE.BytecodeCache` | Lookup in `lmarshal`-Cache vor `luaL_loadbuffer` |
| `ALE.TraceBack` | Custom-`__G.debug.traceback` als Error-Handler |
| `ALE.RequirePaths` / `ALE.RequireCPaths` | `package.path` / `package.cpath` Anhänge |
| `ALE.PlayerAnnounceReload` | Broadcast-Loop in `LuaEngine::ReloadALE()` |

## Bekannte Einschränkungen

- **Keine Prepared Statements für Lua**: alle DB-Calls sind String-basiert. Konsequente Input-Validation in jedem Konsumenten erforderlich.
- **`*Execute` ist async**: zwei `CharDBExecute`-Calls in Folge garantieren keine Reihenfolge-Konsistenz für eine nachfolgende C++-`*Query`-Lese-Operation. Workaround: alle zu setzenden Spalten in **ein** SQL bündeln, oder `CharDBQuery("UPDATE ...")` (blockierend) verwenden.
- **`uint8`-Aurastacks**: Engine reicht Stack-Werte nur als 8-bit weiter. Stacks > 255 sind via Spell-Pairing (siehe `mod-paragon` Big/Small-Auren) zu emulieren.
- **AIO-Re-Registrierungs-Falle**: nach `.reload ALE` halten Frames, die vor dem Reload erstellt wurden, möglicherweise Referenzen auf alte Closures. Doku in `share-public/docs/04-aio-framework.md`.
- **Inkompatibel mit Original-Eluna**: API hat eigene Methoden / signaturen; Eluna-Skripte müssen evtl. angepasst werden (siehe `docs/USAGE.md`).

## Cross-Refs

- Engine-Detail-Dokumentation (Upstream): [`docs/IMPL_DETAILS.md`](./docs/IMPL_DETAILS.md), [`docs/USAGE.md`](./docs/USAGE.md).
- Projekt-AIO-Doku: [`share-public/docs/04-aio-framework.md`](https://github.com/Shoro2/share-public/blob/main/docs/04-aio-framework.md).
- Validation-Lib (Konsument-Side): [`share-public/AIO_Server/Dep_Validation/validation.lua`](https://github.com/Shoro2/share-public).
- M2-Race-Fix als Beispiel-Pattern: `mod-paragon/Paragon_System_LUA/Paragon_Data.lua` Funktion `UpdateAllocationAndUnspent`.
