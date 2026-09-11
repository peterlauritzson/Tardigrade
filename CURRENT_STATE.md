# Tardigrade — Current State

StarCraft II Extension Mod for 1v1 (and team) melee. It runs a short **draft**
before the match (races → modifiers → units), then plays as standard melee with
each side operating under its own drafted set of always-on battlefield
modifiers. Everything else is vanilla SC2 (economy, tech, win conditions).

Internal identifiers still carry the original project name (library
`LibC9EAC993`, `Tardigrade*` config IDs, `Tardigrade.SC2Mod` folder) to keep the
SC2 build stable. Only the player-facing name is "Tardigrade".

Last major change: the "cycle" system was converted from a rotating
Day/Dusk/Night phase system into a **per-player modifier draft** (see below).

---

## Which doc to trust

| Doc | Scope | Status |
|---|---|---|
| `CURRENT_STATE.md` (this file) | Implementation reference — wiring, constants, data, gotchas. | **Authoritative.** Verified against the source. |
| `README.md` | Player-facing overview + project layout. | Current (rewritten to match this file). |
| `DESIGN_DEFENSE.md` | Design intent for the early-defense / anti-snowball layer (tactical bar, refund, alerts, high ground). | Design only — **nothing in it is implemented**. Includes the rejected-ideas list and the rules new proposals must satisfy. |
| `references/readme.md` | How to search the extracted Blizzard data. | Current. Note: all of `mods/` exists locally, but only `mods/voidmulti.sc2mod/` is tracked in git (see `.gitignore`). |
| Script header comments | — | **Partly stale**, flagged inline below. Constants and code win. |

---

## Draft chain

Runs on map init while the game is paused. Wired in `TardigradeLogic.galaxy`
via a callback chain:

```
Race Draft → Modifier Draft → Unit (Roster) Draft → 3-2-1 countdown → Game
```

- `Tardigrade_Init` → `RaceDraft_Start`
- `Tardigrade_OnDraftFinished` → `CycleMod_StartDraft`
- `Tardigrade_OnCycleFinished` → `RosterDraft_Start`
- `Tardigrade_OnRosterFinished` → `Tardigrade_StartGame`

`Tardigrade_StartGame` applies roster enforcement, runs the countdown, starts
the modifier scan loop, and unpauses (`GameSetMissionTimePaused(false)` +
`Tardigrade_DraftLock_Stop()`).

### 1. Race Draft (`RaceDraft.galaxy`)
- P1 (the "banner") bans one race, P2 picks from the remaining two, P1 gets the
  last race. In team games each team plays one shared race.
- Worker count mirrors base melee (counted per race before removal, not
  hardcoded). **Only the town hall** is spawned immediately, in
  `SetPlayerRace` (Race Draft finish); workers + the extra unit (e.g.
  Overlord) are deferred to `Tardigrade_SpawnDeferredWorkers()`, called from
  `Tardigrade_StartGame` right after the 3-2-1 countdown finishes, right
  before the draft lock releases and the game actually unpauses. Getting here
  took three iterations:
  - **v1 — spawn everything immediately at Race Draft finish.**
    `GameSetMissionTimePaused` only pauses the mission timer, not player
    input, so a real Town Hall sitting idle through the Modifier + Roster
    Draft was clickable/orderable — Zerg in particular could train from
    Larva with zero workers.
  - **v2 — defer the *entire* starting pack (town hall included) to game
    start.** "Fixed" the exploit but left every player with **zero
    units/structures** for the several real-time minutes the remaining
    drafts take. That trips the native "no units/structures = defeated"
    watchdog for every player on both teams at once, permanently disarming
    elimination tracking for the match — games stopped ending, always
    running out the clock into a tie.
  - **v3 (rejected) — keep v1's immediate full spawn, bracket the draft with
    `UnitPauseAll(true)`/`(false)`** on the theory that it blocks all unit
    orders map-wide. **Confirmed wrong in a live playtest** — `UnitPauseAll`
    did not stop players from selecting and commanding their Town
    Hall/workers at all; orders went through exactly as before.
  - **v4 (rejected) — keep v1's immediate full spawn, make every unit
    unselectable instead** (`Tardigrade_DraftLock_Start()`/`_Stop()`,
    `UnitSetState(u, c_unitStateSelectable, false)`). This *did* block
    commands (confirmed) — no selection means no command card, no hotkey
    target, nothing to click or drag-box — but relocated rather than solved
    the underlying complaint: real workers sat there mining, visibly, for
    the whole draft, before the game had "started."
  - **v5 (current) — split the spawn.** Town hall only, immediately (satisfies
    elimination tracking, and still needs the unselectable lock so it can't
    be trained from directly); workers + extra unit deferred to
    `Tardigrade_SpawnDeferredWorkers()` after the countdown, so nothing mines
    or exists to command until the game visibly begins.
  - `Tardigrade_DraftLock_Start()`/`_Stop()` (`RaceDraft.galaxy`, just above
    `RaceDraft_Start`) still guard the town hall for the whole draft.
    `SetPlayerRace` locks it unselectable immediately at spawn; a repeating
    0.5s scan (`Tardigrade_DraftLockScan`, `Wait(0.5, c_timeReal)` — **not**
    `c_timeGame`, which doesn't advance while mission time is paused, see
    `Tardigrade_Countdown`) mirrors `CycleMod_ScanTrigger`'s pattern and
    catches anything the immediate lock misses, chiefly Zerg's Hatchery
    auto-spawning Larva up to its cap over the following ~30-45s — each new
    Larva must be locked too, or a player could morph units directly off it
    with zero workers.
  - **Nexus energy:** the Nexus exists from Race Draft finish and doesn't
    come with the melee opening energy, so `Tardigrade_SpawnDeferredWorkers()`
    sets every Nexus at the player's start location to exactly
    `c_tardigradeNexusStartEnergy` = 50 as the game begins (set, not add).
  - **Not yet verified in a live playtest**: that workers spawning right
    after the countdown (while `Tardigrade_DraftLock_Stop()` hasn't run yet)
    doesn't leave a one-frame window where they're briefly selectable before
    the lock's release sweep runs — should be a non-issue since both happen
    in the same synchronous call with no `Wait` between them, but confirm
    in-editor.
- Defines shared viewer/audience helpers used everywhere:
  `Tardigrade_Viewers()` (all active players incl. spectators),
  `Tardigrade_Audience()` (referees/spectators only), `Tardigrade_HasAudience()`,
  and the team globals `g_draft_team1/2`, `g_draft_p1/p2`, `g_draft_p1_is_team1`,
  `g_draft_p1Race/p2Race`, plus `RaceName()`.

### 2. Modifier Draft (`CycleMod.galaxy`)
The former "cycle" draft. **Per-player draft, no rotation.**
- Pool of **14 modifiers** (`c_cycleModCount`). Each player **bans 2**
  (14 → 10), then **picks 3** (10 → 4) — **four go unpicked** every game.
  The draft dialog lays them out as two columns of `c_cycleModRows` = 7
  (640×540); both the per-player and legacy draft share that layout.
- Ban order: `P1, P2, P1, P2`. Snake pick order: `P1, P2, P2, P1, P1, P2`.
- Each pick is **always active in-game, but only for the picking side's own
  units** (never the opponent's).
- Results stored in `g_cycle_p1Mods[1..3]` / `g_cycle_p2Mods[1..3]`.
- See the dedicated section below for in-game behavior.

### 3. Unit (Roster) Draft (`RosterDraft.galaxy`)
- Each player ends with **6 drafted unit types** (no protected "core" — the
  header comment in the file saying "2 core + 4 drafted" is **stale**; see
  `c_rosterRosterSize = 6` and the "no protected core" notes in code).
- Structure: opening picks → cross-bans (each bans from the opponent's pool) →
  final snake picks. Constants: `c_rosterPickTotal = 12` (6 each),
  `c_rosterPreBanPicks = 4` (2 each), `c_rosterBanTotal = 4` (2 each),
  `c_rosterPoolMax = 18` (UI grid: 2 columns of 9).
- Ban order: `P1, P2, P1, P2` (each bans from the *opponent's* pool).
  **No Bans** (modifier 12) removes the opponent's steps from that order rather
  than passing them: `RosterDraft_Start` builds it through
  `RosterDraft_ScheduleBan`, which drops any step whose target pool belongs to
  a No Bans side, so the actual count lives in `g_roster_banCount` (the UI's
  "Ban N of M" and the end-of-bans check both read it, not
  `c_rosterBanTotal`). With P1 protected the order is just `P1, P1`.
  `RosterDraft_AnnounceNoBans` posts it to chat so the unprotected side knows
  why its turns vanished. A ban count of 0 skips straight to final picks.
  Pick order (steps 1–12, first 4 are the pre-ban openers):
  `P1 P2 P2 P1 | P2 P1 P1 P2 P2 P1 P1 P2`.
- Draft pools are data-driven (`TardigradeRosterConfig` in `GameData.xml`) and
  independent per race: **Terran 16, Protoss 17, Zerg 14** (Hellbat and Archon
  are Terran Pool16 / Protoss Pool17).
  The `RosterDraft.galaxy` header comment ("4 drafted, 2 core auto-assigned",
  snake order `P2 P1 P1 P2 …`, "12-unit pool") is **stale on all three counts**
  — trust the constants and `RosterDraft_Start`.
- Dual live roster panel (YOUR + OPPONENT) so players can counter-draft, plus a
  modifier reference strip at the bottom.
- Detection floor: Observer (Protoss) and Overseer (Zerg) are **always
  buildable** (removed from the pool, never disabled) so detection is guaranteed.
- **Derived units are ordinary picks** — no more freebies (Hellbat used to come
  with Hellion, Archon with either templar). `RosterEnforce_IsAllowed` is now
  just "is it in the roster". Each can be built when drafted alone:
  Hellbat from the Factory (LotV's own `FactoryTrain` entry); Archon from
  Gateway / Warp Gate (`GatewayTrain` / `WarpGateTrain` **Train8**, 100/300
  from the unit's cost, 67s gateway / 57s warp charge = High Templar build +
  12s merge, Templar Archives, card cell Row 1 Col 2 next to HT/DT); the Zerg
  morphs from larva. **Transforms into an undrafted unit are blocked**
  explicitly by `RosterEnforce_GateTransforms` (`TechTreeAbilityAllow` on
  `MorphToHellionTank`/`MorphToHellion` cmd 0 and `ArchonWarp` cmds 0+1) on top
  of `TechTreeUnitAllow`. The Archon fallback follows the larva rule —
  `RosterEnforce_GateArchonFallback` disallows it when either templar is
  drafted, so you merge then.
- **Larva fallbacks share their parent's command-card cell, one at a time.**
  Each fallback button (`LarvaTrain` Train6/9/14/16) sits on the cell of its
  parent (Roach 0,3 / Hydralisk 1,0 / Corruptor 1,2 / Zergling 0,2 — the real
  positions in the merged LotV Larva card). An undrafted parent's button is
  removed by `TechTreeUnitAllow(false)`; a drafted parent gets the fallback
  disallowed by `RosterEnforce_GateLarvaFallbacks` (`TechTreeAbilityAllow`,
  command index = N−1 for TrainN), so the child morphs from the parent the
  normal way. **Bug fixed:** Ravager / Lurker / Brood Lord were originally
  one cell off (on Hydralisk, Mutalisk, Ultralisk), so e.g. drafting Ravager +
  Hydralisk without Roach left no way to build Ravagers — the Hydralisk button
  covered the fallback. Unverified in-editor.

### In-game roster HUD (`RosterHUD_*` in `RosterDraft.galaxy`)
- Top-left toggle button, collapsed by default, that expands to a two-column
  panel: **MINE** (green, left) + **OPPONENT** (orange, right), 6 rows each.
- **One dialog per participant player** (`g_rhud_dialog[16]` etc., indexed by
  player id), each visible only to that player — not one dialog shared by
  everyone. This is required, not a style choice: `DialogSetSize` and
  `DialogSetImageVisible` take no `playergroup` argument (they resize/reskin
  the dialog object itself, for every current viewer), so a single shared
  dialog cannot be expanded/collapsed privately — whoever clicked it toggled
  it for every viewer of that dialog. `RosterHUD_Init` loops both teams via
  `PlayerGroupPlayer`/`PlayerGroupCount` and calls `RosterHUD_InitForPlayer`
  once per player; `RosterHUD_OnToggle` reads `EventPlayer()` to know whose
  dialog to resize.
- Dialog width is constant (390) across collapsed/expanded — only height
  changes (34 → 236) — so control anchor offsets (relative to the dialog's
  current center) never visibly shift when it resizes.
- Spectators/referees get a separate, always-expanded, read-only dialog
  (`RosterHUD_InitAudience`, unaffected by this change) showing both P1 and
  P2 rosters side by side.

---

## Modifier system (in-game) — `CycleMod.galaxy`

### Mode gate
`const bool c_cyclePhaseMode` (top of `CycleMod.galaxy`):
- **`false` (current default)** — per-player draft; each side's 3 picks are
  always active for that side's own units only.
- **`true` (legacy, kept intact)** — the old rotating Day/Dusk/Night phase
  system: 3 drafted modifiers rotate, apply to every unit on the map, with a
  top-center legend HUD, per-phase lighting, and a countdown. All of this code
  is preserved behind the gate but unused.

### Activation delay
- `const fixed c_cycleActivationDelay = 180.0` — modifiers stay **dormant for
  the first 3 minutes** and switch on at 3:00.
- The gate lives in one place: `CycleMod_UnitModActive(unit, behavior)` returns
  `false` before 3:00 (draft mode). Because every buff application, conditional
  behavior, and event handler routes through this helper, they all respect the
  delay automatically.
- **Ban phases are red everywhere.** The roster draft's ban step was the only one
  that read clearly, because it colours three things at once: a red `BAN PHASE`
  heading, a red subtitle, and red buttons carrying a `BAN ·` marker. The
  modifier draft only did the subtitle — both phases were otherwise identical
  gold-titled screens with identical buttons, and players banned thinking they
  were picking. `CycleMod_DraftButtonText(idx, isBan)` now supplies the button
  styling for both the picker's view and the spectator view, and the race draft's
  ban step is coloured too. `isBan` is passed in rather than read from
  `g_cycle_draftPhase`, because legacy phase mode leaves that global at 0 and
  would otherwise render every button as a ban.
- **Spectators / referees.** `Tardigrade_RefreshViewerGroups` builds
  `g_tard_viewers` by walking slots `1..c_maxPlayers-1` and taking every occupied
  one, **not** from `PlayerGroupActive()` — referee/spectator slots were dropping
  out of that group, which silently emptied `g_tard_audience` and made every
  audience code path a no-op (blank screen through the whole draft, no rosters or
  modifiers in game). The read-only mirrors themselves already existed:
  `RaceDraft_UpdateAudienceUI`, `CycleMod_UpdateAudienceUI`,
  `RosterHUD_InitAudience`, and the audience branches in `RosterDraft_UpdateUI`.
  **Still blank for spectators after that fix — now understood better.**
  Playtest: a spectator sees "Tardigrade Loaded." (sent to `PlayerGroupAll()`)
  and **nothing else**, not even a probe line sent to every slot 0..15
  individually. `natives.galaxy` marks `c_playerTypeReferee`/`Spectator`
  **"obsolete and no longer exist in code"**: observers have no player slot, so
  no group built slot by slot can reach them, which is why every earlier
  version failed the same way. **Current approach:** `g_tard_viewers =
  PlayerGroupAll()`, `g_tard_audience = PlayerGroupCopy(PlayerGroupAll())` minus
  the two teams. `Tardigrade_HasAudience()` is now simply "groups built",
  because a slot count can't detect slot-less spectators. The in-game modifier
  panel is per team player plus **one shared spectator panel** (key 0,
  `CycleMod_PanelViewers`) addressed to the audience group. **Unverified:**
  whether the copy-minus-teams audience group still reaches observers. The
  temporary probe in `Tardigrade_RefreshViewerGroups` answers that in one run
  ("[audience]: this reached you"), plus `PlayerGroupAll`'s member count and a
  slot table. If "[all]" arrives but "[audience]" doesn't, the spectator views
  have to be sent to `PlayerGroupAll()` and overridden per player. Separately,
  `DialogSetObservedType` (observers see dialogs through an *observed* player)
  may still matter once chat reaches them.
  All of them disable their buttons for the audience, and the draft click
  handlers additionally reject any player who isn't the current picker.
- Open Skies and Arcane Surge are catalog changes (not per-unit), so they are
  applied once at 3:00 via `CycleMod_ActivateDelayedMods()` (fired from the scan
  loop, guarded by `g_cycle_delayedApplied`), which also posts a chat
  announcement.

### Workers are never affected
Modifiers never apply to workers (SCV, Probe, Drone, **and MULE** —
`CycleMod_IsWorker`). The only exceptions are the two that are *about*
workers: **Free Labor** and **Auto Refineries**, whose eviction has to act on
them (`CycleMod_ModReachesWorkers`). Enforced in one place, the top of
`CycleMod_UnitModActive`, which every per-unit path goes through: scan buffs,
Blink/Boost buttons, Entrenchment/Overwatch arming, Predator's source,
Veteran's killer. Shared Damage follows automatically, since partners must
carry the buff. **Open Skies** is the exception because it edits weapons in
the catalog rather than checking a unit, so `CycleMod_IsWorkerWeapon`
(`FusionCutter`, `ParticleBeam`, `Spines`) is in its skip list. The
Blink/Boost `AbilArray` grants on SCV/Probe/Drone were removed from
`UnitData.xml`, so their command cards are back to vanilla. Arcane Surge
needs nothing: workers have no energy.

### How effects reach units
- `CycleMod_ScanTrigger` — 0.5s loop over all map units. In draft mode it applies
  each side's picked behaviors to that side's own units and clears the rest.
- `CycleMod_UnitModActive` maps a unit's owner → side (via `g_draft_p1_is_team1`
  + team groups) → whether that side drafted the behavior.
- Ownership helpers: `CycleMod_OwnerIsP1/P2`, `CycleMod_P1Team/P2Team`,
  `CycleMod_P1HasModIdx/P2HasModIdx`.
- Event handlers: `CycleMod_OnUnitDamaged` (Predator, Overwatch consumption +
  its bonus damage, last-attacker bookkeeping for Veteran Forces),
  `CycleMod_OnUnitStartedAttack` (Overwatch idle timer), `CycleMod_OnUnitDied`
  (Mutual Destruction, Veteran Forces). Conditional per-unit state handled in
  `CycleMod_UpdateConditionalBehaviors` (Entrenchment, Overwatch, Adrenal).
- In-game reference panel `CycleMod_InitDraftPanel`: top-center, **one dialog per
  viewer** with its own `[+]/[-]` toggle button (`CycleMod_OnPanelToggle`,
  `g_cycle_panel*` arrays), same reason as `RosterHUD` — `DialogSetSize` takes no
  playergroup, so a shared dialog can't be resized for one player only. Starts
  expanded. Spectators get a panel on the same terms as players. Per-viewer
  **YOURS / OPPONENT** (spectators see PLAYER 1 / PLAYER 2), lists both sides'
  3 modifiers; title notes "(active at 3:00)".

### The 14 modifiers
| # | Name | Effect | Implementation |
|---|---|---|---|
| 1 | Open Skies | All your weapons can hit ground and air, splash included | Per-player weapon `TargetFilters` via `CatalogFieldValueSet(..., player, ...)`; strips `Ground`/`Air` from required+excluded. **Splash needs a second pass**: `TargetFilters` only decides what a unit may *target* — splash/beam damage is spread by effect `SearchFilters` in `c_gameCatalogEffect`, which stay plane-locked otherwise. `CycleMod_SplashEffect` lists the 12 traced effects (Siege Tank, Ultralisk, Colossus, Lurker, Hellion, Hellbat, Baneling, Liberator AA) and `CycleMod_SplashFiltersForPlayer` runs the same string surgery on them. Curated, not a catalog sweep — 97 effects carry plane tokens and most are campaign/co-op, destructible rubble, or deliberate (ForceField placement, Blinding Cloud). **`CycleMod_OpenSkiesSkipsWeapon` holds back the weapons of units that already cover both planes.** SC2 fires the *first* weapon in a unit's `WeaponArray` whose `TargetFilters` accept the target — there is no damage-based selection — so opening both weapons of a per-plane pair makes the index-0 weapon win against everything. A Thor answered Roaches with Javelin Missiles (index 0, range 10, 6×4 vs *Light air*) instead of Thor's Hammer; a Tempest answered ground armies with its 13-range anti-air gun. **Derived, not hand-listed** — `CycleMod_EnsureOpenSkiesSkipList` walks `c_gameCatalogUnit`, reads `WeaponArray[0..3].Link`, ORs each weapon's plane coverage (`CycleMod_WeaponPlanes`, which checks *both* halves of the filter string — a weapon is locked out of a plane either by requiring the other or by excluding that one), and skips every weapon of any unit that has >1 weapon and already covers both planes. A hand-list was tried first and was wrong: it missed the Tempest. Against the LotV data the rule yields Thor, ThorAP, Tempest, Queen, Hydralisk, Mothership, InfestorTerran, ScoutMP (16 weapons), and leaves every single-plane unit — Roach, Siege Tank, Corruptor, Viking, Liberator, static defence — open as intended. `CycleMod_IsVestigialMeleeWeapon` covers the one case the rule *can't* see: `RoachMelee`/`HydraliskMelee`/`LocustMPMelee` are hidden 0.5-range stubs at index 0 on ground-only units, which the derived rule correctly leaves open. The Thor AA splash searches came out of `CycleMod_SplashEffect` with them (14 → 12) |
| 2 | Medivac Boost | Click a unit to burst its move speed (afterburners) | **Clickable ability** `TardigradeAbil_MedivacBoost` → applies buff `TardigradeMod_MedivacBoost` (`MoveSpeedMultiplier=1.7`, 15s cooldown, no cost). Granted hidden to every combat unit (not workers); shown/enabled only while the side's pick is active (`CycleMod_UpdateAbilityMods`) |
| 3 | Free Labor | Your workers no longer cost supply | `UnitSetState(u, c_unitStateUsingSupply, false)` on `CycleMod_IsWorker` units (SCV/Probe/Drone); buff `TardigradeMod_WorkersNoSupply` is applied alongside as a visual marker only, it does nothing mechanically |
| 4 | Predator Protocol | Your attacks heal 30% of damage dealt, health first then shields | `CycleMod_OnUnitDamaged` heals the source: life is topped up to max and the remainder spills into shields (capped at max), so the mod still does something at full HP |
| 5 | Eyes Everywhere | Reveals the map + hidden units, **for you only** — excludes neutrals (minerals, Xel'Naga towers, critters) | Buff with `Detect=500 Radar=500 DetectFilters="-;Neutral" RadarFilters="-;Neutral"` on your units |
| 6 | Entrenchment | Your stationary units get +2 armor / +1 range after 6s | Marker → `TardigradeMod_Entrenched` helper when still. Threshold `c_cycleEntrenchDelay` = 6, i.e. the design doc's 3 normal-speed seconds doubled, because `GameGetMissionTime` counts game seconds and the game runs on Faster |
| 7 | Arcane Surge | **2x** energy regeneration on all your units *and structures* | Per-player catalog change: `CycleMod_ArcaneSurgeForPlayer` doubles `EnergyRegenRate` in `c_gameCatalogUnit`, applied once at 3:00 from `CycleMod_ActivateDelayedMods` alongside Open Skies. `CycleMod_EnsureEnergyCache` sweeps the Unit catalog once for player 1's base rates and keeps only types with rate > 0, so the per-player pass is a short list rather than a ~1300-entry walk per player. `TardigradeMod_ArcaneSurge` survives as a marker buff only. **Was +2 energy/s flat** — about 3.5x the 0.5625/s a standard caster regenerates, and the same absolute gift to an Orbital Command as to a Ghost. A behavior can't express "2x": `VitalRegenArray` is additive, and `VitalRegenMultiplier` is *not* indexed by vital (unlike `VitalMaxArray`/`VitalMaxFractionArray`/`VitalMaxAdditiveMultiplierArray`), so it would have doubled Protoss shield and Zerg life regen too |
| 8 | Overwatch | First attack after 10s idle: +50% damage, consumed on that one shot | Marker → `TardigradeMod_OverwatchReady` helper, added on the idle→ready edge (guarded by `UnitBehaviorCount`), consumed the instant the shot lands in `CycleMod_OnUnitDamaged`. **The +50% is dealt by the trigger, not by the buff.** It used to be a `DamageDealtFraction` on the marker, which only reaches damage the engine routes through the source's weapon-damage modifiers — splash, Baneling blasts and spell damage were left at face value, and the index list only covered `Melee`/`Ranged`/`Splash`, never the fourth kind `Spell`. `CycleMod_OnUnitDamaged` now deals `EventUnitDamageAmount() * 0.5` back through `UnitDamage(...,"TardigradeMod_OverwatchBonus",...)`, which lands on every damage event whatever produced it. That effect is `Amount=0` / `ArmorReduction=0` (the trigger already works from post-armor damage) and re-enters the handler, so the handler bails on its own effect id first — otherwise it would recurse and double-feed Predator Protocol. One attack = many damage events (splash on N targets, twin beams), so the consume swaps `OverwatchReady` for the 0.125s `TardigradeMod_OverwatchFiring` marker and keeps boosting while that is up; 0.125s is two game loops, far under any weapon period, so it can't spill into a second attack. `CycleMod_IsHostileTarget` gates the bonus to enemies — riding every damage event means it would otherwise amplify your own Widow Mine / Baneling friendly fire. Threshold `c_cycleOverwatchDelay` = 10, the doc's 5 normal-speed seconds doubled for the same game-speed reason as Entrenchment. **No range bonus** — it used to grant +3 `WeaponRange`/`WeaponScanBonus`, which was removed on the first damage event and stranded the unit holding a target it could no longer reach. Idle is tracked by `CycleMod_OnUnitStartedAttack` *and* the damage handler; the damage handler does its Overwatch bookkeeping above the structure filter, or static defence would arm and never consume |
| 9 | Battle Blink | Click a unit to short-range teleport it (8 range) | **Clickable ability** `TardigradeAbil_Blink` (`CEffectTeleport`, cloned from the Stalker's Blink minus its tech requirement, own cooldown). Same grant/show mechanism as Medivac Boost |
| 10 | Veteran Forces | Each kill = permanent **+3% time-speed (haste)**, stacking to 15 | `TardigradeMod_VeteranStack`, `TimeScale=1.03`, `MaxStackCount=15`, added on kill |
| 11 | Auto Refineries | Your gas buildings mine themselves at the 3-worker rate; workers can't go in | Script only (`TardigradeMod_AutoRefinery` is a key/marker, never applied). `CycleMod_UpdateAutoRefinery`, called per unit from the draft-mode scan. **Payout** (`CycleMod_AutoRefineryTick`): each finished Refinery/Assimilator/Extractor (and `*Rich`) stores the mission time it is paid up to in custom-value slot `c_cycleStateAutoGas` (8) and catches up in whole trips of 4 gas (8 rich) every `c_cycleAutoGasInterval` = 2.1 game seconds — ~160 gas per real minute on Faster, a saturated LotV geyser. Gas is drawn from the building's own `c_unitPropResources`, so geysers deplete on schedule and a dry one stops paying. Each trip also adds to `c_playerPropVespeneCollected`, the engine's "gas collected" total, which only real harvests update on their own; without it auto gas was missing from the income stats. **Unverified:** whether the observer Income tab's *rate* (`VespeneCollectionRate` score value, engine-computed) follows that total or counts only worker deliveries. The clock starts on first sight (after 3:00 or on completion), no back-pay. **Blocking workers** — the order-intercept alone **failed in playtest** (workers still went in and mined), so the primary block is now data: at 3:00 `CycleMod_BlockGasHarvestTeam` sets `ResourceAllowed[Vespene] = 0` on the drafting side's `SCVHarvest`/`ProbeHarvest`/`DroneHarvest` via per-player `CatalogFieldValueSet` — the same field that makes Blizzard's `MULEGather` minerals-only. It tries the named index, reads back, and falls back to `[1]` (unsure which a field path accepts); the resulting value is written to the debug log. Called from the scan loop's activation block, not `CycleMod_ActivateDelayedMods` (defined above it). **Clean-up** (`CycleMod_KeepWorkerOutOfGas`): any worker with a harvest order on an auto refinery *or* `UnitIsHarvesting(…, c_resourceTypeVespene)` is sent to the nearest mineral patch within 12 (neutral-owned `HarvestableResource`), returning carried gas first; `stop` if there is none |
| 12 | No Bans | Your opponent gets no unit-draft bans against your pool | Acts in `RosterDraft.galaxy` — see Unit Draft above. The one modifier not gated by the 3:00 delay (the roster draft runs before the game). `TardigradeMod_NoBans` is a key/marker only |
| 13 | Salvage | Any structure can be salvaged for 75% of its cost | **Clickable ability** `TardigradeAbil_Salvage` (`CAbilBehavior` toggle: On / Off), granted hidden to 64 structures in `UnitData.xml`, shown via `CycleMod_UpdateAbilityMods` like Blink/Boost. On applies `TardigradeMod_Salvaging` — a copy of Blizzard's Bunker salvage behavior: 5s, structure shut down, `ExpireEffect` `TardigradeMod_SalvageRefund` (`CEffectModifyUnit`, `ModifyFlags Salvage`, `SalvageFactor -0.75` — the engine removes the unit and refunds 75% of `CostResource`). Morph costs are cumulative in the data (Lair 475 = Hatchery 325 + 150; Zerg costs include the Drone), so one factor covers all. As on the live Bunker, **taking damage cancels it** (damage response → `TardigradeMod_SalvageInterrupt` removes the behavior; removal isn't expiry, so no refund); Terran burndown exempt. `HasNoCargo` blocks loaded CCs/Nydus. Skips Bunker + Sensor Tower (already salvage in LotV) and Creep Tumors (free). Card slots computed, not guessed — see the Salvage note below |
| 14 | Shared Damage | Each hit on your unit (after armor): it takes half, the other half is split evenly across your nearby units; alone it takes all | **Absorb and redeal.** `TardigradeMod_SharedDamage` carries `DamageResponse ModifyFraction=0 ModifyMinimumDamage=1` (Blizzard's `DamageTakenNone`), so the hit never lands; `TriggerAddEventUnitDamageAbsorbed` fires `CycleMod_OnSharedDamageAbsorbed`, which computes X (`CycleMod_SharedHitAmount`: absorbed − victim armor × the effect's `ArmorReduction`, floor 0.5 — responses run before armor, so the absorbed amount is pre-armor; shield armor if shields are up), finds partners (`CycleMod_SharePartners`: allied units within `c_cycleShareRadius` = 3 that carry the behavior, not dead/hidden/stasis/invulnerable) and deals the victim X × `c_cycleShareKeptFraction` (0.5) and each of the N partners (X − that) / N — or the victim all of X when N = 0 — through `TardigradeMod_SharedDamageHit` (Amount 0, ArmorReduction 0, Kind Spell) in the **original attacker's** name. Heal-back after the fact was rejected: the damaged event fires after damage lands, so the lethal hits the mod exists to spread would already have killed. The share effect is in the response's `ExcludeEffectArray` — the only thing stopping infinite re-splitting. Also excluded: the Overwatch bonus (already dealt per share) and ~20 **Kill-flag effects** (Baneling `Suicide`, `KillHallucination`, `MULEFate`, shade end, Bile vs Force Field, …), which would otherwise leave their target alive. `CycleMod_CanShareDamage` keeps structures, hallucinations and `CycleMod_IsShareExemptType` units (larva/eggs/cocoons, MULE, shade, interceptors, locusts, broodlings, changelings, Force Field, Parasitic Bomb dummy, Disruptor ball) from ever carrying it. If the attacker is gone, the victim authors its own shares and `CycleMod_OnUnitDamaged` ignores friendly-authored shares so they can't feed Predator/Overwatch |

Notes:
- **War Economy was removed** (it made no sense without phases) and replaced by
  Medivac Boost at slot 2. Its behaviors are gone from `BehaviorData.xml`;
  `CycleMod_HelperBehavior` indices 1–2 still return the old ids but are now dead
  (never called) — **do not renumber** helper indices 3–6.
- **Mutual Destruction → Free Labor** and **Adrenal Response → Battle Blink**
  were both replaced outright (not tuned) at slots 3 and 9. The old
  `TardigradeMutualDestructionDamage`/`Search` effects and
  `TardigradeMod_AdrenalResponse`/`AdrenalBoost` buffs are deleted, not kept
  dead — unlike War Economy, nothing else referenced them.
- **A `CBehaviorBuff Modification Food=".."` field does NOT retroactively
  change a player's supply-used total for a unit that's already alive** — the
  engine only re-derives that aggregate on spawn/death/morph, not
  continuously from each unit's current stats, so a buff granted well after
  the worker already exists (e.g. at the 3:00 activation delay) had no
  visible effect. First cut of Free Labor used exactly this (and with the
  wrong sign besides — `Food="-1"` stacks with the worker's own base `-1`,
  making it cost *more* supply, not less). Fixed by using
  `UnitSetState(u, c_unitStateUsingSupply, false)` instead — a per-unit
  boolean the engine does check live, and the correct tool for "this specific
  unit doesn't count toward supply" in general.
- **Ability-based modifiers (2, 9) skip the passive-buff scan path entirely.**
  `CycleMod_ApplyPickedBehavior` special-cases both behavior-id strings and
  returns without touching a buff; `CycleMod_UpdateAbilityMods` (called once
  per unit per scan tick, draft mode only) calls `UnitAbilityShow` +
  `UnitAbilityEnable` instead. The abilities are granted **hidden** to every
  combat unit via `AbilArray` in `UnitData.xml` (49 units; the 3 worker grants were removed) so
  there's something to show/hide — a unit whose side didn't draft either
  modifier just keeps both permanently hidden.
- **A granted ability with no `CardLayouts` entry never renders, even when
  shown.** First cut of this feature only added `AbilArray` grants with no
  card slot, on the assumption a hidden ability would auto-place into an open
  command-card cell once shown — it doesn't; `UnitAbilityShow`/`Enable` toggle
  visibility/usability of an *already-positioned* button, they don't create a
  position. Confirmed against Blizzard's own data: their KD8Charge patch to
  the Reaper (`voidmulti.sc2mod`) ships the `AbilArray` grant together with an
  explicit `CardLayouts` slot, repositioning an existing button to make room.
  First fix attempt gave both abilities an explicit slot at `Row="3"`
  (grep-confirmed no vanilla unit's `UnitData.xml` uses a row above 2 across
  the whole liberty/swarm/void/voidmulti chain) — **confirmed in a live
  playtest that Row 3 doesn't render at all**, so the command-card panel
  evidently caps at 3 visible rows (0–2) regardless of what the data allows.
  Moved to `Row="1" Column="0"`/`Column="1"` next — rendered, but playtesting
  across several units found the low columns collide with existing vanilla
  buttons often enough to matter (Ghost's Nuke Calldown/Weapons Free,
  Infestor's Fungal Growth, etc. tend to sit there). Currently
  `Row="1" Column="2"` (Blink) / `Column="3"` (Medivac Boost) — same row,
  still two separate columns so both can be shown/used on the same unit at
  once, just shifted right based on that playtest feedback. **Still not
  individually verified per unit** — if a button is missing or covers up a
  different ability on a specific unit, nudge that unit's `Row`/`Column` in
  `UnitData.xml`.
- Other War Economy leftovers still compiled in but inert: the
  `c_cycleStateEconomicEgg` custom-value slot, `CycleMod_IsEconomicEgg`, and the
  `CycleMod_OnEconomicEggStarted` trigger (still registered in
  `CycleMod_StartCycle` on Drone/Overlord train abilities). Harmless; remove only
  together. Same now applies to `c_cycleStateAdrenalReady`/`c_cycleStateWasLow`
  (custom-value slots 4/5) and `CycleMod_HelperBehavior(5)` — dead since Battle
  Blink replaced the low-life-triggered Adrenal Response mechanic.
- Modifier buffs carry a short `Duration` (Medivac Boost = 8s) but the 0.5s scan
  re-applies them, so they behave as permanent while the modifier is active and
  fall off on their own if the scan stops applying them. Medivac Boost's buff
  duration no longer matters for the *scan* (it's ability-triggered now) but
  still caps how long one cast's speed burst lasts.
- `TimeScale > 1` = faster. Stacks are multiplicative (≈ `1.03^15` = +56%).
- **Known gap:** `TardigradeAbil_Blink` has no custom actor wiring, so casting
  it teleports the unit with no blink flash/sound (the Stalker's own Blink
  visuals are keyed to effect id `Blink` specifically, not reusable by a
  same-behavior clone under a different id without duplicating those actors
  too). Functional but silent — flagged as a follow-up, not attempted blind.
- **Salvage's command-card slots were computed, not guessed** (the Blink/Boost
  history above is why). A throwaway script resolved each structure's
  `CardLayouts index 0` across core → liberty → swarm → void → the three multi
  layers and placed the button on a cell nothing uses: **Row 2 Col 2** on all
  64, except Orbital Command (Row 1 Col 3 — calldowns fill row 2), Nexus and
  Nydus Network (Row 2 Col 3). Cancel (Off) sits on Row 2 Col 4 like the
  Bunker's. Conditional buttons (research shown only with tech) count as
  occupying their cell, so the choice is conservative.
- **Modifiers 11–13 have no buff.** `CycleMod_Behavior` still returns an id for
  each (they're the keys `CycleMod_UnitModActive` matches on) and
  `BehaviorData.xml` defines them as empty markers, but
  `CycleMod_ApplyPickedBehavior` returns early for them. Legacy phase mode
  would add the markers harmlessly; the new mechanics themselves only run in
  draft mode.

---

## Roster enforcement (`RosterEnforce.galaxy`)
- At game start, for each team, every combat unit **not** in that team's roster
  is disabled via `TechTreeUnitAllow(player, unit, false)`.
- Master combat lists per race live in `RosterEnforce_InitLists`
  (Terran 16, Protoss 17, Zerg 14). Workers, town halls, tech/production
  structures, and supply are never touched.
- **Zerg's Swarm Host uses unit id `SwarmHostMP`**, not `SwarmHost` (`SwarmHost`
  is the campaign-only id) — same MP-suffix pattern as `LurkerMP`. The pool
  (`GameData.xml`), enforce list, and display-name lookups
  (`RosterDraft.galaxy`) all now use `SwarmHostMP`. Before this fix,
  `TechTreeUnitAllow(player, "SwarmHost", false)` was a silent no-op and Swarm
  Host stayed buildable regardless of the draft.
- **`TechTreeUnitAllow(..., false)` also hides upgrade buttons** — patched in
  `GameData/RequirementData.xml`. Blizzard gates a few upgrade buttons on
  `CRequirementAllowUnit`, the node driven by exactly that flag, reasoning that
  an upgrade is clutter when you can't build the unit it was made for. In ladder
  nothing is ever disallowed so the gates never fire; here they fire constantly.
  The gate sits on the requirement's **`Show`** node and the buttons are
  `State="Restricted"`, so they don't render at all — the command card just looks
  empty. There are exactly five such nodes in the whole dependency chain:
  | Gate | Hides | Verdict |
  |---|---|---|
  | `AllowUnitPhoenix` | **all 6 Cybernetics Core air upgrades** (Air Weapons/Armor 1–3) | **bug — patched.** Drafting Tempest/Void Ray/Carrier/Oracle without Phoenix left the air army with no upgrades at all. The Phoenix is just Blizzard's stand-in for "can this player build anything that flies" |
  | `AllowUnitMarine` | Stimpack | **bug — patched.** Marauders stim too, so a Marauder-without-Marine roster lost it |
  | `AllowUnitColossus` | Extended Thermal Lance | correct — only the Colossus uses it |
  | `AllowUnitObserver` | Gravitic Booster | never fires; Observer is in neither the pool nor the enforce list |
  | `AllowUnitImmortal` | `LearnIncreasedRange` | dead — no live `CAbilResearch` button references it |
  The patch repoints each broken `Show` node at the same requirement **minus**
  its `AllowUnit` operand, keeping the `EqCountUpgrade…QueuedOrBetter0` half that
  makes levels 1/2/3 share one command-card cell. `Use` nodes are untouched, so
  Fleet Beacon / previous-level / Tech Lab prerequisites still apply as in ladder.
- The enforce lists match the draft pools (Hellbat and Archon are pool picks
  now). `Observer` and `Overseer` appear in **neither** — that absence is
  exactly what keeps detection always buildable.
- Header comment "2 core + 4 drafted" is **stale** — rosters are 6 fully-drafted
  units with no protected core.
- In a solo/PvAI game where one human "decides" for both sides, only the human
  team's roster is enforced.

---

## Debug (`Debug.galaxy`)
- Single-player launch auto-enables debug mode.
- Race draft shows a `[DEBUG: Random All]` (and per-race) button that skips all
  drafts and randomizes races, rosters, and modifiers.
- `Tardigrade_DebugAutoRun` assigns **6 distinct random modifiers**, 3 per player
  (`g_cycle_p1Mods` / `g_cycle_p2Mods`).
- `Debug_Log(...)` writes to the debug log; viewer-group counts are logged on
  refresh.

---

## Data files
| File | Contents |
|---|---|
| `BehaviorData.xml` | `TardigradeMod_*` modifier buffs + helper behaviors (`Entrenched`, `OverwatchReady`, `OverwatchFiring`, `VeteranStack`, `Salvaging`, etc.), the absorbing `TardigradeMod_SharedDamage`, and empty markers for modifiers 11–13. |
| `EffectData.xml` | `TardigradeAbil_Blink` (`CEffectTeleport`) + `TardigradeAbil_MedivacBoost` (`CEffectApplyBehavior`) — the effects behind the ability-based modifiers — `TardigradeMod_OverwatchBonus` (`CEffectDamage`), the payload `CycleMod_OnUnitDamaged` fires for Overwatch, `TardigradeMod_SharedDamageHit` (one Shared Damage share), and Salvage's `TardigradeMod_SalvageRefund` / `TardigradeMod_SalvageInterrupt`. |
| `AbilData.xml` | `LarvaTrain` fallback InfoArray entries, plus `TardigradeAbil_Blink` / `TardigradeAbil_MedivacBoost` / `TardigradeAbil_Salvage` ability definitions. |
| `UnitData.xml` | `Larva` command-card layout for the larva-build fallbacks, `AbilArray` grants of Blink + Medivac Boost (hidden by default) to ~50 combat units (no workers) across all three races, and of Salvage to 64 structures. |
| `GameData.xml` | `TardigradeRaceStartConfig`, `TardigradeRosterConfig`, `TardigradeCycleConfig` (legacy phase timing/lighting, only used in phase mode), and a `CGame id="Dflt"` override raising `StalemateTestTime`/`StalemateWarningTime` from the 180s default (see Gotchas). |
| `ActorData / ValidatorData .xml` | Minimal / unused by current features. |

---

## Scripts
| File | Role |
|---|---|
| `TardigradeLogic.galaxy` | Entry point + draft-chain orchestration + game start. |
| `RaceDraft.galaxy` | Race ban/pick; team/viewer globals + helpers. |
| `RosterDraft.galaxy` | Unit draft UI (opening picks → bans → final picks) + in-game roster HUD. |
| `RosterEnforce.galaxy` | Disables non-drafted units; grants coupled units. |
| `RequirementData.xml` | Un-gates the upgrade buttons Blizzard hid behind `CRequirementAllowUnit` (Protoss air weapons/armor, Stimpack) — see Roster enforcement. |
| `CycleMod.galaxy` | Modifier draft + per-player in-game modifiers (+ gated legacy phase system). |
| `Debug.galaxy` | Solo-test detection, auto-run, logging. |

---

## Gotchas / conventions
- **Native stalemate detection fires during the draft, not just in-game.**
  `CGame`'s default `StalemateTestTime`/`StalemateWarningTime` are 180s
  (`core.sc2mod` `GameData.xml`), and that clock is not gated by
  `GameSetMissionTimePaused` the way mission-time waits are. Since the full
  draft chain (Race + Modifier + Roster, all real human decision time, zero
  units on the map until game start) regularly runs past 180s, the native
  "this game will be a stalemate" warning could fire mid-draft. Our
  `GameData.xml` `CGame id="Dflt"` override raises both to 5400s (90 min) so
  it can never fire during setup; a genuinely stalled real game past that
  point just won't get the prompt, which is an acceptable tradeoff.
- **Galaxy compiler dislikes `const bool` inside compound `&&`/`||` conditions**
  ("Expected a boolean expression"). Standalone `if (c_cyclePhaseMode)` is fine;
  compounds are not. Nest under a standalone `if` or use a local bool. The VS
  Code linter does **not** catch this — only the SC2 editor compiler does.
- **Target/search filters:** requiring both `Ground` and `Air` matches nothing
  (no unit is both planes). To hit both, the required list must contain neither.
- **Reserved type names** can't be used as variable identifiers (color, order,
  point, text, unit, timer, etc.).
- Verify all changes by **recompiling in the SC2 editor**; the linter passes on
  errors it can't see.

---

## Known TODO / tuning
| Item | Notes |
|---|---|
| Modifier balance | `MoveSpeedMultiplier=1.7`, `TimeScale=1.03×15`, ability cooldowns (12s Blink / 15s Medivac Boost, both uncosted), etc. are first-pass. Same for the new four: Shared Damage radius 3 with no cap on Y, Auto Refineries' 2.1s trip interval, Salvage's 5s timer. |
| **Modifiers 11–14, unverified in-editor** | Nothing below has been compiled or played. In order of risk: **(1) Shared Damage** — test first: hit one of the drafting side's units after 3:00. If it takes **no damage at all**, `TriggerAddEventUnitDamageAbsorbed` isn't firing for a `ModifyFraction=0` response (the handler already falls back to attempted − landed if the absorbed amount reads 0, but can't help if the event never fires). Also check the split is post-armor as intended — the handler assumes the absorbed amount is pre-armor. And confirm a Baneling on that side still dies when it detonates and an Adept shade still expires (Kill-flag exclusions). **(2) Salvage** — confirm the button renders in the computed cell, the 5s timer shows, the refund is 75% (Lair should return 356/75), and damage cancels it. **(3) Auto Refineries** — confirm gas lands at ~160/real min, that `c_unitPropResources` on the refinery is the geyser's remaining gas (if it reads 0 the building will never pay), and that evicted workers go to minerals. **(4) No Bans** — pure draft logic, lowest risk. |
| Activation delay | 3:00 is a starting value (`c_cycleActivationDelay`). |
| Battle Blink has no visual/audio feedback | `TardigradeAbil_Blink` teleports silently — the Stalker's Blink flash/sound actors are keyed to effect id `Blink`, not reusable under our separate id without duplicating those actor entries too. Not attempted blind (unverifiable without the SC2 Editor); functional but silent. |
| Command-card placement, not checked per-unit | Both ability buttons sit at `Row="1"` (Blink Column 2, Medivac Boost Column 3) on all 52 granted units — Row 3 didn't render at all, and Row 1 Column 0/1 collided with existing buttons on several units in playtest (Ghost, Infestor, …), so it moved to Column 2/3. Still not checked unit-by-unit; if a button is missing or covering an existing ability on a specific unit, nudge that unit's `Row`/`Column`. |
| In-game modifier panel | Static top-center, 214px tall — reposition/shrink if intrusive. |
| Stale comments | `RosterDraft.galaxy` + `RosterEnforce.galaxy` headers still say "2 core + 4 drafted" (behavior is 6 drafted, no core); `RosterDraft.galaxy` also lists the wrong snake order and a "12-unit pool". `TardigradeLogic.galaxy` still calls the modifier step "the cycle" / "day/dusk/night". |
| Dead War Economy code | `c_cycleStateEconomicEgg`, `CycleMod_IsEconomicEgg`, `CycleMod_OnEconomicEggStarted` and helper indices 1–2 are inert but still compiled/registered. |
| Dead Adrenal Response code | `c_cycleStateAdrenalReady`/`c_cycleStateWasLow` (custom-value slots 4/5) and `CycleMod_HelperBehavior(5)` are now the same kind of harmless-but-inert leftover, since Battle Blink replaced that mechanic. |
| Draft-time opponent roster panel, unverified in-editor | Don't confuse with the in-game HUD above (already fixed). `RosterDraft_UpdateRosterPanel`'s dual panel *during the draft itself* (YOUR + OPPONENT, live, shown while picking) already existed in code before this session and looked complete on read-through — if it's not showing up in an actual playtest, that's a rendering/timing bug to hunt for in the editor, not a missing feature to build from scratch. |
| Everything in this batch needs an SC2 Editor recompile to verify | Per the compiler gotcha below, several of these fixes (ability grants, filters, stalemate override, per-player dialogs) touch areas the VS Code linter cannot validate. |
| Split immediate/deferred starting-unit spawn, unverified in-editor | Five iterations to get right — see the History note under Race Draft above. Current (v5): only the town hall spawns immediately at Race Draft finish (kept unselectable via `Tardigrade_DraftLock_Start()`/`_Stop()`, confirmed in playtest to actually block commands); workers + extra unit are deferred to `Tardigrade_SpawnDeferredWorkers()`, called after the countdown so nothing mines or is commandable while the draft UI is up. Needs a live playtest to confirm: newly spawned Larva get caught by the 0.5s scan before a player can act on them, and the workers don't have a selectable window between spawning and `Tardigrade_DraftLock_Stop()`'s release sweep. |
