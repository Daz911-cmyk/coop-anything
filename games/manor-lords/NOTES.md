# Manor Lords co-op: working notes

Path 2 (build from zero). Started 2026-10-26.

## Game identity (observed)
- Store: Steam (app 1363080). In-game version **0.8.104** (main branch).
  `ManorLords/Content/Movies/Version.txt` says 0.8.097 and is stale: do not use it to identify the build.
- `ManorLords-Win64-Shipping.exe`: 160,791,040 bytes, SHA-256 `813c4909dbe8bef3481469137c66f35cc23dec11c145b6963c4739b41539e621`
- Engine: Unreal Engine 5.5.4 (exe version resource; UE4SS also reports 5.5, Shipping, Stats off). Windows x64.
- No packer/DRM sections, no anti-cheat. Ships Steamworks v157, EOS SDK, GOG Galaxy and Stove SDK plugins.

## Prior work (research, 2026-10-26)
- No official multiplayer. Developer, Steam forum 2020: "Focusing on single first. But it would be cool to add later on."
  https://steamcommunity.com/app/1363080/discussions/1/4355617769807995432
- No co-op mod found on Nexus Mods or GitHub; not in this directory; no issue or PR here.
- Single-player modding uses UE4SS (ML-UE4SS fork by Psiae on Nexus, and upstream RE-UE4SS),
  e.g. https://github.com/BANNGRU/manor-lords-cheats (Lua), https://github.com/Maxhem2/manor-lords-fixes (C++ + MinHook).

## Workspace
- Isolated copy of the install; original untouched. robocopy, then exe SHA-256 re-checked on the copy.
- `steam_appid.txt` (1363080) next to the copied exe so it starts outside the Steam library with Steam running.
- UE4SS: upstream experimental build `UE4SS_v3.0.1-1183-gcdbc0b66.zip`
  (SHA-256 `e4a47e23e2eeb8379c4fc388d1b1686f01c6c5930d84a2ee4a6a9574b274e681`, release asset dated 2026-10-25).
  Stable v3.0.1 predates UE 5.5; the experimental notes list 5.5 support. Works on 0.8.104 (observed).
- Note: the test copy shares saves with the real game unless started with `-userdir` (see below).
- Mods kept outside the game folder via `+ModsFolderPaths = <test dir>/mods` in UE4SS-settings.ini
  (additive; default mods still load). Handy when tools limit folder depth.

## Phase 3: first dump (observed, 2026-10-26)
A one-shot Lua mod (`MLCoopProbe`) calls DumpAllObjects, GenerateSDK, GenerateLuaTypes and DumpUSMAP
inside ExecuteInGameThread, 15 s after the first InitGameState (main menu). All four succeeded in ~3 s.
Outputs: UE4SS_ObjectDump.txt (37.8 MB), CXXHeaderDump/ (938 headers), .usmap (2.4 MB).

Game module `ManorLords.hpp`: 151 classes, 232 structs. Key classes:
- `ARTSMultiEngineCPP` (AActor): central simulation singleton. Most actors hold `masterPtr` to it.
  Has `setGameSpeed`, `setDay`, `spawnArmy(pos, unitTypes, ownerPawn, companyID, arrivesInDays)`,
  `spawnCompleteUnit(..., ownerPawn, ...)`, `spawnNewAILord(name, isFemale, portrait, canBuild)`, `addPawnRelations`.
- `APawnCPP` (APawn): the player/lord. 417 reflected functions (UI, camera, roads, claims, squads,
  `settleRegion`, `disbandSquad`, `setPlaceRallyingPoint`). Has `isMainPlayer`, `isAI` flags.
  `APawnCPP_AI : APawnCPP` = rival AI lords. **Rival lords are the same pawn class as the player.**
- `ASMBuildingMaster` (buildings, 242 fn), `ASMUnit` (villagers/soldiers, 117 fn), `ARegion` (152 fn),
  `ARoad`, `AConstruction`, `ABattlefield`, `AWeatherMaster`, `UProductionSubsystem` (UWorldSubsystem).
- Saves: `UMLSaveGame`, `UMLSaveGameDescr`, `UMLMenuSaveGame` (USaveGame).
- `ANotStrongholdCPPGameModeBase` is empty: the game uses no UE replication; all state is local.

## Phase 3: call logging (observed, 2026-10-26)
- `MLCallLog` Lua mod: RegisterHook on 569 reflected functions of the game classes above (UI-refresh style
  names like update*/get*/is* skipped), logs the first 15 calls of each with parameters, then every 200th.
  562 hooks installed; 7 failed, all of them delegate signatures (not UFunctions), as expected.
  F9 in game writes a MARK line to tag tester actions.
- Hooks only fire for calls that go through the reflection system (Blueprint/UI to C++), not for
  C++-to-C++ calls. Good for finding UI command entry points; native-only paths need native hooks.
- Test copy launched with `-userdir=<test dir>/userdir`: UE 5.5 shipping honours it, Saved/ (saves, config,
  logs) goes there, so tests never touch the player's real saves in %LOCALAPPDATA%\ManorLords.

## Derived (not yet tested)
- Since each lord is an `APawnCPP` with an owner id used by spawn functions, a second human could be
  modelled as another lord pawn on the host whose commands come from the network instead of AI.
- Several placement paths (building placement, field/road drawing) look native and not reflected:
  expect native hooks (C++ UE4SS mod / MinHook) for command capture.

## Unknowns
- Which APawnCPP functions are the real command entry points (vs UI refresh). Next: hook candidates
  with RegisterHook and log calls while playing a scripted session.
- Determinism: unknown; assume none (host-authoritative state sync).
