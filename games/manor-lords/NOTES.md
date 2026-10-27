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

## Phase 3: first command map (observed, 2026-10-26, owner play session, 28 F9 marks)
New game, building, road, field, worker assignment, game speed, militia, squad select, save, load.
Calls seen right at the tester's actions (reflected, so hookable and callable from Lua):

| Player action | Calls observed |
|---|---|
| Pick a building to place | `PawnCPP:selectBuildingMode(id, FStat)`, `PawnCPP:setPlacedBuilding(id)`, `SetBuildingCategory(n)` |
| Confirm building placement | **no reflected call**; only `Construction:setupConstructionFast()` afterwards (native path) |
| Draw and confirm a road | `PawnCPP:setRoadMode(true, FVector, false)` then `PawnCPP:commitRoad()` (no params) |
| Confirm a field/pasture | `PawnCPP:addCommitFieldRequest()` (no params) |
| Add/remove workers | `SMBuildingMaster:addActiveWorkerFamily()`, `SMBuildingMaster:changeActiveLivestockWorkers(+1/-1)` |
| Pause / speed | `PawnCPP:setGameSpeed(0 / 1 / 4 / 8)` |
| Raise militia | `PawnCPP:addMilitiaSquad(ARegion, "bowMilitia")` |
| Select squad | `PawnCPP:selectSquad(id, bool, bool)`, `deselectSquad(id)`; **move order: no reflected call** |
| Save | `PawnCPP:requestSaveFile(name, name)`, `onSavingFinished()`, `MLGameInstance:ExportSaveThumbnailRT` |
| Load | `PawnCPP:stopVAMP()`, LoadMap, `setMainUIMode` |
| Open/close building panel | `SMBuildingMaster:setTabIsOpen(bool)`, `PawnCPP:deselectBuilding(building)` |

Noise to filter next time: `setIsUsingUI` (every frame), `setDipToBlackOpacity`, `OnParticleSystemFinished`,
`clearBufferedMovement`, `SetCategoryCounterWidget`, settings calls on every pause menu open.

### Derived
- Commit calls without parameters (`commitRoad`, `addCommitFieldRequest`) read the pawn's in-progress state
  (road points, field polygon). To replay them on the host, capture that state from the pawn's properties in a
  pre-hook (offsets are in the CXX dump) and set it on the host's pawn for the remote player before calling.
- Building placement and squad move orders need native hooks (C++ UE4SS mod), found from the
  `ASMBuildingMaster`/`ASMUnit` spawn path and `UUnitMovementComponent`.

## Phase 5: command replay prototype (observed, 2026-10-26)
`MLCoopNet` Lua mod. Commands are text lines, e.g.
`ROAD|<start x,y,z>|<tangentScale>|<p1>;<p2>;...` and `SPEED|<value>`.

**How a road command is replayed** (found by a failing test first):
setting `roadPoints` and calling `commitRoad()` does nothing; the commit reads the pawn's preview road
(`APawnCPP.dummyRoad`, an `ARoad` with a `USplineComponent`). Working sequence on the game thread:
`setRoadMode(true, start, true)` -> next frame: `dummyRoad.Spline:ClearSplinePoints(true)`,
`AddSplinePoint(p, World, true)` per point -> next frame: `commitRoad()` -> `setRoadMode(false, ...)`.
Capture on the sending side reads the same spline (`GetLocationAtSplinePoint(i, World)`).

**Autoload without a human:** at the first InitGameState (main menu), set
`MLGameInstance.savefileToLoad = "saveGame_0"` and call `GameplayStatics:OpenLevel(pc, "/Game/NotStronghold/Maps/Mission01_ROBBER")`.
The save loads. Tests end with the console command `quit`.

### Test 1: selftest, one copy (2026-10-26 05:39)
```
PASS main pawn found MyPawnCPP_BP3_C ...
PASS apply SPEED SPEED|2.000 -> speed 2.0
PASS apply ROAD call ROAD|58563.60,31716.53,-1574.79|1.0000|... -> road with 3 points (async commit)
PASS road actor created 1 new road(s) Road_2147463383 (spline points 3, committed)
```
Can fail: the previous build (roadPoints + commitRoad, no spline) gave `FAIL road actor created Road actors 70 -> 70`.

### Test 2: duo, two copies on one PC (2026-10-26 05:46)
Second instance = hard links to the test copy, private `ue4ss/`, own `-userdir`, role file `coop_role.txt`
next to the exe. Transport: a shared text file (`to_host.txt` / `to_client.txt` with ACKs), host polls every 250 ms.
The client has hooks on `PawnCPP:commitRoad` and `PawnCPP:setGameSpeed` (real capture path); a client bot
drives the game, the host applies and checks its own world.
```
host:   PASS host apply #3 ROAD|64563.60,31716.53,-1574.79|1.0000|... -> road with 3 points (async commit)
host:   PASS host world has client road #3 1 new road(s) (spline points 3, committed)
host:   PASS host received all commands 4 commands          ALL OK
client: PASS client got acks 4 ack(s) for 4 command(s)       ALL OK
```
### What this does NOT prove yet (honest scope)
- Transport is a file on one PC, not a network (Steam relay is next).
- The host applies commands to its own lord, not to a separate lord for the friend.
- Nothing flows back: the client's world is not synced from the host and will drift.
- Road checked by "a new ARoad actor exists", not by position, cost or pathfinding; no screenshot.
- Unwanted capture: the game calls `setGameSpeed` itself while loading a save, and the client sent that as a
  player command (#1). Capture must be limited to calls from player input/UI.

## Phase 4: decision and lord probe (2026-10-26)
**Owner's decision:** each player is their own lord with their own region on one map (not a shared town).

### Test 3: lordprobe, one copy (2026-10-26 05:55, observed)
Save at the start of a game: 3 lords (main player + 2 `APawnCPP_AI`), 8 regions. `ARegion.ownerPawn` holds the owner.
```
PASS AI lord with a region PawnCPP_AI_2147467453 / hofstetten
PASS apply ROAD to AI lord ROAD|80937.48,-82747.33,2870.00|1.0000|... -> road with 3 points (async commit)
PASS AI lord road created 1 new road(s) (spline points 3, committed)
PASS spawnNewAILord creates a lord pcall=true new=PawnCPP_AI_2147463466
PASS free region found Goldhof
PASS finalizeClaim gives the region to the friend pcall=true owner=PawnCPP_AI_2147463466
PASS settleRegion founds a town for the friend pcall=true settled=true owner=PawnCPP_AI_2147463466
after: lord 3 PawnCPP_AI main=false ai=true regions=[Goldhof(settled)]
```
Calls: `RTSMultiEngineCPP:spawnNewAILord("Friend", false, nil, false)`, then `friend:finalizeClaim(region, false)`,
then `friend:settleRegion(region, FTransform{region actor location}, 1, ESettlementType.Town, nil)`.

So the same command layer works for a lord that is not the main player, and a new lord can get its own town.

### Not checked yet
- Whether the spawned lord's AI acts on its own (`APawnCPP_AI` has AI timers; `startAI(speed, canBuild)`).
  A remote human lord must have its AI off. Unknown.
- Whether the road built for the AI lord belongs to its region (`ARoad.regionA/B` not read).
- Whether a spawned lord survives save and load.
- No mutation run for test 3; no screenshots.

### Test 4: friend lord, AI off, save/load, main-player swap (2026-10-26, observed)
Phase A (from saveGame_0, 06:11): spawn "Friend", claim + settle Goldhof, measure AI, pause AI, save as saveGame_7.
```
PASS friend town founded Goldhof lordsName=Friend
PASS baseline: AI counters move while AI is on friend 8/2 -> 28/7, control 33/8 -> 53/13
PASS friend AI stopped (counters frozen) friend 28/7 -> 28/7 (AICommandHandle=paused active=false ...)
PASS control AI still runs control 53/13 -> 73/18
PASS game saved with the friend lord saveGame_7                                   ALL OK
```
Can fail: the run before used `K2_ClearAndInvalidateTimerHandle`, which UE4SS rejects (the handle is a
by-reference param and needs an out table): `FAIL friend AI stopped friend 28/6 -> 48/11`.
Working call: `KismetSystemLibrary:K2_PauseTimerHandle(lord, lord.AICommandHandle)` for
`AICommandHandle`, `AIGeneralStrategyHandle`, `AILetterwritingHandle` (all three report active=false).
AI progress is visible in `APawnCPP_AI.AI_general_cycles` / `AI_tactics_cycles`.

Phase B (load saveGame_7, 06:15):
```
lord 3 PawnCPP_AI main=false ai=true regions=[Goldhof(settled)]
PASS friend lord exists after load ... lordsName=Friend
PASS friend still owns a settled town after load Goldhof
PASS after load: friend AI runs again (expected, info) friend 25/6 -> 45/11
PASS friend AI stopped (counters frozen) friend 45/11 -> 45/11
PASS control AI still runs control 45/11 -> 65/16
PASS controller now drives the friend lord flags ok, playerRef ok, possess ok | pc.Pawn=PawnCPP_AI_2147467440
PASS engine playerRef is the friend
PASS friend (as main) builds a road 1 new (spline points 2, committed)
```
Main-player swap: `old.isMainPlayer=false; friend.isMainPlayer=true; friend.isAI=false;
RTSMultiEngineCPP.playerRef=friend; PlayerController:Possess(friend)`.

Facts learned: a spawned lord is kept in the save (name and town); its AI restarts on every load, so the
mod must pause it after each load. Identify lords after load by their town (`ARegion` names are stable),
object names change.

Not verified: the screen. `HighResShot 1` produced no file (game quit 3 s later). Whether the HUD, resources,
building menu and camera now show the friend's town is unknown until a visual check.

### Test 5: screen check after the runtime swap (2026-10-26 06:48-07:05, observed)
Same phase B with `noquit=1`, camera moved to the friend town (`friend:SetPosition(region location)`, ok).
Screenshot of the running game after the swap: the top bar still says **Eichenhau** and the portrait and coat
of arms are the original lord's, while the camera is at Goldhof. So the runtime swap is partial: game logic
(`playerRef`, `isMainPlayer`, possessed pawn, commands) uses the friend, but the HUD stays bound to the lord the
save was loaded with.

**CORRECTION (2026-10-27): the "autosave rewrote saveGame_7" conclusion was wrong.** `saveGame_7` in the
test folder was last written at 06:11; the autosave went to a separate `autosave.sav`. The 07:02 run that showed
the player owning Goldhof was an extra instance started by a computer-use `open_application` call, which runs the
game **without `-userdir`**, so it used the owner's real save folder. That folder has no `saveGame_7`, so the game
most likely started a fresh game in which the player's random start region was Goldhof. Lesson: never use
`open_application` for the game (it starts a new process without the test arguments); start copies only through
the scripts. The owner's real save folder was checked afterwards (read-only): saves unchanged; only
`rtsSettings.sav` (game settings) was rewritten by such an extra instance.

The next run on that save crashed the game: the test treated the (now non-AI) player pawn as the friend and
paused AI timer handles that pawn does not have. Test bug, not a game finding. Guard needed: only touch AI
handles on `APawnCPP_AI`.

Unverified: whether a save made this way shows Goldhof in the HUD when loaded (needs a screenshot of a fresh
load), and what happens to the original player lord's town in such a save (lost owner here).

### Test 6: full co-op loop, two copies (2026-10-27 09:29, observed)
`mode=coop`. Host loads saveGame_0, spawns "Friend", claims and settles a free region, pauses its AI, then for
the length of one save makes the friend the player (`old.isMainPlayer=false`, `friend.isMainPlayer=true`,
`friend.isAI=false`, `RTSMultiEngineCPP.playerRef=friend`), calls `requestSaveFile("saveGame_8")`, restores
everything, copies the save files into the client's `userdir2`, and then applies client commands to the
friend lord. The client waits at the main menu for the save, loads saveGame_8, and a bot draws a road.
```
host:   PASS host: friend town founded Goldhof shown as "Am Aiberbach"
host:   PASS host: host lord is the player again
host:   PASS host: host still owns its town Eichenhau
host:   PASS host: client save copied to the client 5 file(s)
client: lord 3 MyPawnCPP_BP3_C main=true ai=false regions=[Eichenhau(settled)]
client: lord 4 MyPawnCPP_BP3_C main=true ai=false regions=[Goldhof(settled)]
client: PASS client: I am the friend lord (own the friend town) own=Goldhof shown as "Am Aiberbach" lordsName=Friend
client: FAIL client: current region is my town currentRegion=selbitz shown as "Boswies"
client: PASS client: another lord still owns a settled town Eichenhau("Eichenhau")=Fritz, Goldhof("Am Aiberbach")=Friend(me)
host:   PASS host apply #1 ROAD|75327.53,-15211.79,2870.00|... -> road with 3 points (async commit)
host:   PASS host world has client road #1 1 new road(s) (spline points 3, committed)
client: PASS client: host acknowledged my commands 1 ack(s) for 1 command(s)
```
Screenshot of the client copy: top bar shows **"Am Aiberbach"** (the friend's town), empty portrait (the friend
lord was spawned without one), resources of the new town, the new-settlement tutorial. The client plays as the
friend. The `currentRegion` check is a bad oracle (it follows the camera), not a game failure.

Findings:
- `ARegion.regionName` is the town's display name, chosen at random when a town is founded; the FName
  (Goldhof) is the stable id.
- In the client save both lords are of the player pawn class and both report `isMainPlayer=true`; the one the
  `PlayerController` possesses after load is the friend. Use `PlayerController.Pawn` to find the local lord.
- Crash found and fixed: also setting `old.isAI = true` during the client save crashed the host
  (`EXCEPTION_ACCESS_VIOLATION reading address 0x15f8`; 0x15f8 lies inside `APawnCPP_AI`'s own fields,
  so the save treated the player pawn as an AI lord). Never set `isAI` on a non-AI pawn class.
- Tooling note: a file commit to the PC can carry a stale copy of a just-edited file; verify with SHA-256
  after every commit before launching.

Not yet: the host lord on the client is a player-class pawn with no input (does it do anything over time?),
time sync, the client's world drifting from the host's, and getting host changes back to the client.

### Derived plan (not tested)
- Host: one `APawnCPP` per remote player (spawned once, kept in the save), AI off; remote commands applied to it.
- Client: the friend needs to see the shared world with their lord as the controlled one. Without determinism,
  a practical start is: client sends commands; host is the only simulation; client view is resynced from host
  snapshots (the game's own save file sent over the network and loaded with the friend's lord made the main
  player), with command replay in between. Later: per-object state sync for buildings, roads, units.

## Derived (not yet tested)
- Since each lord is an `APawnCPP` with an owner id used by spawn functions, a second human could be
  modelled as another lord pawn on the host whose commands come from the network instead of AI.
- Several placement paths (building placement, field/road drawing) look native and not reflected:
  expect native hooks (C++ UE4SS mod / MinHook) for command capture.

## Unknowns
- Which APawnCPP functions are the real command entry points (vs UI refresh). Next: hook candidates
  with RegisterHook and log calls while playing a scripted session.
- Determinism: unknown; assume none (host-authoritative state sync).
