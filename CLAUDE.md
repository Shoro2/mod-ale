# mod-ale (Forgotten Land fork)

Forgotten Land's fork of `azerothcore/mod-ale`: ALE, the AzerothCore Lua Engine (the Eluna successor).
It runs every server-side Lua and AIO script of Forgotten Land (FL): the deployed `lua_scripts` tree
and the Lua halves of the FL modules. It is infrastructure, not a gameplay module. Upstream's own
documentation is [README.md](README.md) and [docs/](docs/); this file only records what is FL-specific.
The fork carries no full per-repo doc set on purpose, so that it stays close to upstream.

## FL role

| What | Where / value |
|---|---|
| Lua tree it loads | workbench `C:\wowstuff\dcore\lua_scripts` (git `Shoro2/fl-lua-scripts`), host `/wowserver/acore-server/bin/lua_scripts`; one subdirectory per module |
| Consumers | AIO (`lua_scripts/AIO_Server`) and every AIO/Lua UI on top of it: Paragon, Paragon ItemGen, Endless Storage, Loot Filter, Dungeon Challenge, Underground, Procedural Dungeon, Chronicle, Custom Spells, FLContinentNav, rifts, flypaths. Ownership per folder: `fl-lua-scripts`' CLAUDE.md |
| Deployed config | `C:\wowstuff\dcore\configs\modules\mod_ale.conf`: `ALE.ScriptPath = "lua_scripts"`, `ALE.AutoReload = false`, `ALE.BytecodeCache = true`, `ALE.TraceBack = false` |
| Build | part of the module set: workbench `azerothcore-wotlk/modules/mod-ale`, host `/wowserver` module tree |

## Status and progress

- **Fork state (2026-10-09):** `master` = upstream up to July 2026 plus one FL commit, `5a9ac80`
  (2026-07-09: `OnPlayerResurrect` takes `bool& applySickness`, to compile against the core), and the
  merge `f8923aa`. Upstream has since made the same signature change, so the FL delta disappears at
  the next sync.
- Upstream is ahead (10 commits on 2026-10-09), among them `bd74eae` (#401, 2026-09-09: use-after-free
  and incomplete escaping in the `FormatQuery` DB helpers), `5472013` (#408, `IsBot` via
  `WorldSession::IsHeadless()`) and `cead0cb` (#393, quest objective progress in `Player:CompleteQuest`).
- Host: `f8923aa` (vault `forgotten-land/15-host-migration-log.md` §3, measured 2026-08-09). The whole FL
  Lua stack runs on it in game every day.

## Next steps

1. (medium) Merge upstream `master` (incl. the #401 DB-helper fix), build, boot with zero ALE load
   errors, run a short bot smoke, then a MIG entry for the host.
2. (medium) Before that merge, check the FL Lua that passes extra arguments to `WorldDBQuery`,
   `CharDBQuery` or the `*DBExecute` calls (the format form): after #401 the engine escapes string
   arguments itself, so a script that escapes them first would escape them twice.
3. (low) CoA's `c9553af6` (Witch Hunter Noctis Blade) needs a `UnitScript` signature change that
   breaks mod-ale and Forgotten Talents, so CoA round 3 did not take it (MIG-045). If a later CoA round
   takes it, this fork's hook override must change in the same window.

## Working here

- Keep FL changes minimal and prefer upstream fixes; branch `claude/<topic>-<sessionId>`, merge into
  `master` and push (project rule: no pull requests). Remote `upstream` = `azerothcore/mod-ale`.
- A changed engine needs a worldserver rebuild; host changes go through a MIG entry in share-public
  `docs/World of Warcraft/forgotten-land/15-host-migration-log.md`.
- AIO trap: handlers re-registered by a script reload keep old closures until `/aio reset` or a
  relog (share-public `docs/World of Warcraft/04-aio-framework.md`).
- Lua SQL has no prepared statements; validation is the consuming script's job.
