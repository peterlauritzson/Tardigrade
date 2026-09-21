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
- `const fixed c_cycleActivationDelay = 180.0` — modifiers stay **dormant** until
  180 **game** seconds, which the in-game clock (Faster, 1.4×) shows as **2:09**.
  The modifier panel's "(active at …)" text is computed from the constant by
  `CycleMod_ActivationClockText`, so it can't drift.
  **Opening vision:** until that same moment, `CycleMod_UpdateOpeningVision`
  (called per unit from the draft-mode scan) gives every participant's units
  — workers excepted — the `TardigradeMod_EyesEverywhere` buff (map reveal +
  detection). From activation on it strips the buff from any unit whose side
  didn't draft Eyes Everywhere (explicitly, since the buff's 8s duration would
  otherwise linger), leaving the drafting side's copy to
  `CycleMod_ApplyPickedBehavior`. It reads `c_cycleActivationDelay` directly,
  so moving activation moves it. Announced at game start (with the clock
  time), at activation, and on the modifier panel. Every other duration in the
  mod (cooldowns, Entrenchment/Overwatch waits, the Battle Blink cooldown) is likewise game seconds.
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
  have to be sent to `PlayerGroupAll()` and overridden per player.
  **Playtest round 2:** the referee now receives chat (they saw the probe's slot
  table, which has no referee row, only slot 15 = type 4 Hostile, confirming
  observers have no slot), but still **no dialogs**. So the remaining problem
  is dialog-specific: every `DialogCreate` is now followed by
  `Tardigrade_ObservableDialog` (`RaceDraft.galaxy`), which sets
  `DialogSetObservedType(..., c_triggerObservedTypeObservedOrSelectedPlayerId)`
  so an observer sees each dialog through the player they watch or select.
  Expected: picking a player in the observer vision dropdown shows that
  player's view of the draft. Unknown: whether the default "Everyone" view
  shows anything. If not, the fallback is a chat-only spectator feed of the
  draft.
  **Playtest round 3 (screenshot), hard facts:** probe `[all]` arrived with
  `PlayerGroupAll count 16, audience count 14`; `[audience]` did **not**;
  the slot table was slot 0 type 3 (Neutral), 1–2 type 1 (User), 15 type 4
  (Hostile) — no observer row. "Draft Complete!" (chat to viewers =
  `PlayerGroupAll()`) arrived. Dialogs still did not render. Conclusions:
  (1) **only the unmodified built-in `PlayerGroupAll()` reaches observers.**
  A hand-built 0..15 group (round 1) and a copy minus the teams (round 3)
  both fail, despite identical or near-identical membership, so there is **no
  way to address spectators separately**. `g_tard_audience` can't reach
  them; anything for spectators has to go to everyone. (2) Dialogs don't
  follow that rule: shown to `PlayerGroupAll()` with
  `ObservedOrSelectedPlayerId`, still invisible in the default view.
  **Round 4 — RESOLVED for dialogs:** with `Tardigrade_ObservableDialog` in
  place, switching the observer vision dropdown to Player 1 or 2 shows that
  player's UI. "Everyone" still shows none, and that is an engine limit, not a
  bug. The model is therefore: **a spectator sees exactly the watched player's
  screen.** Consequences applied: (1) the *waiting* player in all three
  drafts now sees the active player's options, **disabled**, not a blank
  screen (`RaceDraft_ShowRaceButtons`, the waiter loop in
  `CycleMod_UpdateDraftUI`, and the waiter mirror in `RosterDraft_UpdateUI`,
  where during bans it is the waiter's own pool). Enabled is set explicitly
  both ways, or buttons disabled on a waiting turn would stay disabled on the
  player's own turn. (2) `RaceDraft_Start` posts a one-line chat hint telling
  spectators to pick a player, to everyone, since nothing narrower reaches
  them. The audience-group views (`RaceDraft_UpdateAudienceUI`, the audience
  branches, `RosterHUD_InitAudience`, the key-0 modifier panel) can't reach
  observers and are effectively dead. They are harmless and left in place.
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

### Per-unit state: no unit cap
A spawn's veteran root is a **buff on the unit** (see modifier 10). The one
remaining per-unit reference, the last attacker, lives in the **global
data table keyed by `UnitGetTag`** — `CycleMod_UnitKey` / `_GetUnitRef` /
`_SetUnitRef`, cleared by `CycleMod_ForgetUnit` on death so keys don't
accumulate. They were two `unit[2048]` arrays indexed by a counter handed out in
`CycleMod_RegisterUnit`, which **capped the whole match at 2048 units ever**:
past that, registration silently stopped and Veteran Forces kill credit went
with it — and Interceptors/Locusts churn through units endlessly.
`c_cycleStateRegistered` is now just a "seen before" flag, and `c_cycleWeaponLimit`
bounds only the weapon-catalog cache (Galaxy needs constant array sizes; the
catalog is far smaller than 2048).

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
| 1 | Open Skies | All your weapons can hit ground and air, splash included — at **half damage against the plane the unit couldn't originally hit** | **Cross-plane penalty:** `CycleMod_ApplyOpenSkiesPenalty` gives each unit of the Open Skies side `TardigradeMod_OpenSkiesVsAir` (weapons originally ground-only) or `…VsGround` (air-only), by `CycleMod_UnitOriginalPlanes` — the OR of its `WeaponArray` weapons' *shipped* coverage, memoised per type in the global data table (`TgOSU_<type>`). Shipped filters come from `CycleMod_EnsureWeaponCache`, which saves each weapon it will patch under `TgOSW_<weapon>` before any patch; unpatched weapons are read live. The behaviors carry `DamageResponse Location="Attacker" ModifyFraction=0.5` with `TargetFilters` naming the victim's plane, so the cut happens before the hit lands (exact on killing blows, and Predator/Shared Damage see the reduced hit). Spell kind and the Overwatch bonus are exempt. Units covering both planes, or unarmed, get no penalty. **Unverified:** that an attacker-side response's `TargetFilters` reads the victim (inferred from Blizzard's `FlyerShield`). **Opening the weapons:** per-player weapon `TargetFilters` via `CatalogFieldValueSet(..., player, ...)`; strips `Ground`/`Air` from required+excluded. **Splash needs a second pass**: `TargetFilters` only decides what a unit may *target* — splash/beam damage is spread by effect `SearchFilters` in `c_gameCatalogEffect`, which stay plane-locked otherwise. `CycleMod_SplashEffect` lists the 12 traced effects (Siege Tank, Ultralisk, Colossus, Lurker, Hellion, Hellbat, Baneling, Liberator AA) and `CycleMod_SplashFiltersForPlayer` runs the same string surgery on them. Curated, not a catalog sweep — 97 effects carry plane tokens and most are campaign/co-op, destructible rubble, or deliberate (ForceField placement, Blinding Cloud). **Third lock: plane-gated validators.** A damage effect can carry a `ValidatorArray` pointing at a `CValidatorUnitFilters` whose `Filters` require a plane — the effect runs, asks, and silently does nothing when the answer is no. This bit the **Brood Lord**, which is uniquely exposed because `BroodlingStrike` is `<Effect value=""/>`, a pure targeting shell: every point of its damage goes through the single validated effect `BroodlingEscortDamageUnit` (20), gated by `BroodlingEscortFilters` = `Ground,Visible;…`. Opening the weapon alone let a Brood Lord attack a Battlecruiser and deal **literally zero** — it acquired, launched, impacted, and the validator threw the damage away. `CycleMod_GateValidator` / `CycleMod_EnsureGateCache` / `CycleMod_GateFiltersForPlayer` now run the same string surgery on `c_gameCatalogValidator`, cached and restored exactly like the splash list. Curated for the same reason: 14 `CValidatorUnitFilters` in the data carry plane tokens and every other one is campaign or a deliberate plane use (void’s `AirUnitFilter` / `GroundUnitFilter`). The broodlings `BroodlingEscortImpactA` spawns are **not** gated and still land on the ground under an air target, where they expire — accepted: against air the Brood Lord is the 20 and nothing else. **`CycleMod_OpenSkiesSkipsWeapon` holds back the weapons of units that already cover both planes.** SC2 fires the *first* weapon in a unit's `WeaponArray` whose `TargetFilters` accept the target — there is no damage-based selection — so opening both weapons of a per-plane pair makes the index-0 weapon win against everything. A Thor answered Roaches with Javelin Missiles (index 0, range 10, 6×4 vs *Light air*) instead of Thor's Hammer; a Tempest answered ground armies with its 13-range anti-air gun. **Derived, not hand-listed** — `CycleMod_EnsureOpenSkiesSkipList` walks `c_gameCatalogUnit`, reads `WeaponArray[0..3].Link`, ORs each weapon's plane coverage (`CycleMod_WeaponPlanes`, which checks *both* halves of the filter string — a weapon is locked out of a plane either by requiring the other or by excluding that one), and skips every weapon of any unit that has >1 weapon and already covers both planes. A hand-list was tried first and was wrong: it missed the Tempest. Against the LotV data the rule yields Thor, ThorAP, Tempest, Queen, Hydralisk, Mothership, InfestorTerran, ScoutMP (16 weapons), and leaves every single-plane unit — Roach, Siege Tank, Corruptor, Viking, Liberator, static defence — open as intended. `CycleMod_IsVestigialMeleeWeapon` covers the one case the rule *can't* see: `RoachMelee`/`HydraliskMelee`/`LocustMPMelee` are hidden 0.5-range stubs at index 0 on ground-only units, which the derived rule correctly leaves open. The Thor AA splash searches came out of `CycleMod_SplashEffect` with them (14 → 12) |
| 2 | Forced March | +40% move speed, off while in contact, back 14s after the last contact | **Plain scan-applied buff, no ability.** `TardigradeMod_ForcedMarch` (`MoveSpeedMultiplier=1.4`) is handed out by `CycleMod_ApplyPickedBehavior` like any passive, but withheld while `TardigradeMod_MarchBroken` is on the unit. `CycleMod_BreakMarch` is the single place that adds the marker and strips the speed buff in the same breath, so the slowdown lands on the event rather than up to half a scan tick later. **Contact is both directions.** Damage taken: `CycleMod_OnUnitDamaged` breaks the victim's march on *any* damage, friendly fire included — unlike Battle Blink, which stays hostile-only, since a unit standing in its own Psi Storm is not marching anywhere either. Damage dealt: `CycleMod_OnUnitStartedAttack` is the primary attacker break (it fires on the swing, so a shot that misses or is fully absorbed still counts), with `CycleMod_OnUnitDamaged` also breaking on the source for damage that isn't a weapon attack (splash, Banelings, spells) and on the `CycleMod_VeteranRoot` spawner so a Carrier's Interceptor hits break the Carrier's march. The marker's `Duration` = 14 game seconds (10 real on Faster) **is** the recovery timer: re-adding it refreshes it, so it expires 14s after the LAST contact, and no per-unit timestamp is kept. **Was Medivac Boost**, a clickable `CAbilEffectInstant` granted to 49 units with a card slot each, applying +70% for 8s on a 15s cooldown. That version was half of the "I can never catch them" problem this doc's Snare entry exists to answer: it could be fired mid-fight, so a kiting long-range unit boosted while shooting. Breaking on damage taken alone didn't fully close that — a unit that out-ranges its target or shoots something that can't shoot back (Colossus into workers, Tempest into a Carrier) never took a hit and kited at full speed. Breaking on the swing too leaves no combat use at all, so it is purely a travel and disengage tool. It is still one-sided (only the drafting side has it), so it does make disengaging better for them; that is a difference of degree from the button, not a fix, and the Snare entry still stands |
| 3 | Free Labor | Your workers no longer cost supply | `UnitSetState(u, c_unitStateUsingSupply, false)` on `CycleMod_IsWorker` units (SCV/Probe/Drone); buff `TardigradeMod_WorkersNoSupply` is applied alongside as a visual marker only, it does nothing mechanically |
| 4 | Predator Protocol | Your attacks heal 30% of damage dealt, health first then shields | `CycleMod_OnUnitDamaged` heals the source: life is topped up to max and the remainder spills into shields (capped at max), so the mod still does something at full HP |
| 5 | Eyes Everywhere | Reveals the map + hidden units, **for you only** — excludes neutrals (minerals, Xel'Naga towers, critters) | Buff with `Detect=500 Radar=500 DetectFilters="-;Neutral" RadarFilters="-;Neutral"` on your units |
| 6 | Entrenchment | +2 armor / +1 range after holding position 10.5s (7.5 real), with a 1.5 drift tolerance | Marker → `TardigradeMod_Entrenched` helper when still. Threshold `c_cycleEntrenchDelay` = 10.5 game seconds = 7.5 real on Faster, because `GameGetMissionTime` counts game seconds and the game runs on Faster. Raised from 6 so the bonus is a commitment to a position rather than something a unit collects in passing, but kept under Overwatch’s 14 because not moving is a harder thing to ask than not attacking. It is a `fixed`, not an `int` — 10.5 has no integer form, and the comparison dropped its `FixedToInt` so the half second is real; the 0.5s scan tick makes that granularity meaningful. **`c_cycleStateLastX`/`LastY` hold an ANCHOR, not last tick's position:** the clock keeps running while the unit stays within `c_cycleEntrenchRadius` = 1.5 of where it settled, and only a drift past that re-anchors and resets. The tolerance used to be 0.01 map units — pixel-perfect — and any drift reset the whole clock, so collision push from an ally walking past, separation shuffle, or a nudge while acquiring a target cost everything and started over; chasing that made the modifier miserable to hold. A **fixed** anchor rather than a rolling “how far since N seconds ago” window on purpose: a rolling window never trips for anything slower than its own allowance, so an Overlord (0.9/s, 1.8 over a 2s window) would stay entrenched while flying across the map, where an anchor bounds total displacement and has nothing to creep past. `TardigradeMod_Entrenched`'s `Duration` = 3 is the **grace period on losing it**, not how long it lasts — the 0.5s scan re-adds it continuously while the unit holds, so it only counts down once the unit has left its anchor. Raised from 2 so a brief reposition doesn't strip the bonus instantly; re-earning it is still the full 10.5s |
| 7 | Arcane Surge | **2x** energy regeneration on all your units *and structures* | Per-player catalog change: `CycleMod_ArcaneSurgeForPlayer` doubles `EnergyRegenRate` in `c_gameCatalogUnit`, applied once at 3:00 from `CycleMod_ActivateDelayedMods` alongside Open Skies. `CycleMod_EnsureEnergyCache` sweeps the Unit catalog once for player 1's base rates and keeps only types with rate > 0, so the per-player pass is a short list rather than a ~1300-entry walk per player. `TardigradeMod_ArcaneSurge` survives as a marker buff only. **Was +2 energy/s flat** — about 3.5x the 0.5625/s a standard caster regenerates, and the same absolute gift to an Orbital Command as to a Ghost. A behavior can't express "2x": `VitalRegenArray` is additive, and `VitalRegenMultiplier` is *not* indexed by vital (unlike `VitalMaxArray`/`VitalMaxFractionArray`/`VitalMaxAdditiveMultiplierArray`), so it would have doubled Protoss shield and Zerg life regen too |
| 8 | Overwatch | After 14s idle (10 real): +50% damage for a 1.4s window (1 real) of attacking | Marker → `TardigradeMod_OverwatchReady` helper, added on the idle→ready edge (guarded by `UnitBehaviorCount`), consumed the instant the unit breaks idle in `CycleMod_OnUnitDamaged`, which opens the bonus window in its place. **The state belongs to the spawner, not the unit that dealt the damage.** `CycleMod_VeteranRoot` (the same child→spawner link Veteran Forces uses, falling back to the child when the spawner is dead) redirects the armed/firing lookup, because a Carrier's damage all comes from Interceptors, a Swarm Host's from Locusts, a Brood Lord's from Broodlings and a Raven's from its Auto-Turret — each created moments before it fires, so none could ever have idled long enough to arm. Drafting Overwatch on a Carrier bought literally nothing. The spawner's `c_cycleStateLastAttack` is stamped whenever a child lands a hit, or a Carrier (which never deals damage itself) would read as permanently idle and re-arm between every volley. **A unit that dies dealing its shot can't be boosted here at all** — it is dead by the time the damage event reaches triggers, so its markers are gone and it cannot author the bonus `UnitDamage` either. A Baneling's blast is weapon `VolatileBurst` → effect set `[BanelingDontExplode, SuicideTargetFriendlySwitch, VolatileBurstU2, VolatileBurstU, Suicide]`, so a Baneling always did its plain 35. The Baneling's +50% is therefore **baked into its damage numbers** instead (`CycleMod_ApplyOverwatchDamageTeam`, below); the Disruptor's nova ball and the Reaper's KD8 Charge are in the same boat and are deliberately left unboosted rather than growing a second mechanism. **Casters need no special handling:** the handler stamps `c_cycleStateLastAttack` for *any* damage a unit deals, spell included, so a Storm's first tick boosts and then marks the High Templar non-idle for the rest of it. A Yamato or Snipe from a unit that has not attacked in 14s does get the +50%, which is accepted. **The +50% is dealt by the trigger, not by the buff.** It used to be a `DamageDealtFraction` on the marker, which only reaches damage the engine routes through the source's weapon-damage modifiers — splash, Baneling blasts and spell damage were left at face value, and the index list only covered `Melee`/`Ranged`/`Splash`, never the fourth kind `Spell`. `CycleMod_OnUnitDamaged` now deals `EventUnitDamageAmount() * 0.5` back through `UnitDamage(...,"TardigradeMod_OverwatchBonus",...)`, which lands on every damage event whatever produced it. That effect is `Amount=0` / `ArmorReduction=0` (the trigger already works from post-armor damage) and re-enters the handler, so the handler bails on its own effect id first — otherwise it would recurse and double-feed Predator Protocol. One attack = many damage events (splash on N targets, twin beams), so the consume swaps `OverwatchReady` for `TardigradeMod_OverwatchFiring` and keeps boosting while that is up — and **that marker’s `Duration` IS the bonus window**, now 1.4 game seconds (1 real). It was 0.125 (two game loops), just wide enough to hold one volley together, which made the modifier read as a misfire on anything whose damage does not arrive in one instant: slow projectiles, multi-hit weapons, a volley landing a few loops late. A one-second window covers all of that and is something a player can aim — open on a full army and every unit that fires inside that second connects. Still far under the 14s idle needed to re-arm, so it cannot chain. `CycleMod_IsHostileTarget` gates the bonus to enemies — riding every damage event means it would otherwise amplify your own Widow Mine / Baneling friendly fire. Threshold `c_cycleOverwatchDelay` = 14 game seconds = 10 real on Faster, same units as Entrenchment. **Baneling flat bonus:** `CycleMod_ApplyOverwatchDamageTeam` runs from `CycleMod_ActivateDelayedMods` at 3:00 and multiplies `Amount` and `AttributeBonus[Light]` by `c_cycleOverwatchFlatMult` = 1.5 on `VolatileBurstU`, `VolatileBurstU2` and the two `VolatileBurstDirectFallbackEnemyNeutral*` payloads, per-player via `CatalogFieldValueSet`. Both fields, because 16 + 19 vs Light is where the 35 against a Marine comes from — scaling `Amount` alone would give 43, not 52.5. The two `VolatileBurstFriendly*` payloads are left alone, or the modifier would make your own Banelings hurt your own army 50% more. Values are read back and multiplied rather than hardcoded, so a balance patch carries through; the one gap is that Zerg melee upgrades are applied by the engine on top of `Amount` rather than into it, so a +3 Baneling does 52.5 + 3 rather than (35 + 3) × 1.5. `CycleMod_OverwatchIsFlatBoosted` keeps the trigger bonus off Banelings so the two can never stack. **No range bonus** — it used to grant +3 `WeaponRange`/`WeaponScanBonus`, which was removed when the window closed and stranded the unit holding a target it could no longer reach. Idle is tracked by `CycleMod_OnUnitStartedAttack` *and* the damage handler; the damage handler does its Overwatch bookkeeping above the structure filter, or static defence would arm and never consume |
| 9 | Battle Blink | A unit at 30% health blinks 8 range back from whatever last hit it, once per 17s | **No ability, no button.** `CycleMod_TryBattleBlink`, called from `CycleMod_OnUnitDamaged` for the victim of any hostile hit. Fires when life + shields drops to `c_cycleBlinkThreshold` (0.30) of their combined maximum — shields count, so a Protoss unit blinks on its real health bar rather than the instant its shields break. Destination is `PointWithOffsetPolar` at `c_cycleBlinkRange` (8) along `AngleBetweenPoints(attacker, victim)`, i.e. straight back along the line of fire; the `TardigradeAbil_Blink` `CEffectTeleport` still does the move (`UnitCreateEffectPoint`) and still does its own placement validation, so an escape into a cliff lands short rather than failing. Cooldown is the `TardigradeMod_BlinkCooldown` behavior's 17s `Duration` rather than a custom value — the engine expiring the buff **is** the cooldown, so there is no per-unit timer to keep and no custom-value slot to spend. Driven from the damage event rather than the 0.5s scan because a unit that crosses 30% is usually dead well inside half a second. **Was a clickable ability**, granted to 49 combat units with an explicit Row 1 Col 2 card slot each; that collided with abilities units already owned (a Stalker had two Blinks, and the drafted one may not have been usable at all) |
| 10 | Veteran Forces | Each kill = permanent **+3% to movement, attack, cooldowns/charges, regen**, stacking to 15 | `TardigradeMod_VeteranStack`: `MoveSpeedMultiplier` + `AttackSpeedMultiplier` + `VitalRegenMultiplier` 1.03, and `RateMultiplierArray` Cooldown/Charge/Morph/Progress/Queueable/Spawn 1.03 — the same stat set Blizzard's current Chrono Boost (voidmulti) uses instead of `TimeScale`. `MaxStackCount=15`, added on kill. **Unverified:** `VitalRegenMultiplier` as a plain attribute (the editor lists the field; the Arcane Surge investigation says it isn't per-vital) — check a veteran caster's energy regen. **Was `TimeScale=1.03`** until playtest showed the same Battlecruiser with its movement boosted throughout but its attack boosted only intermittently. The BC's attack ability (`CAbilAttack BattlecruiserAttack`, with `Min/MaxAttackSpeedMultiplier`) is built around the attack-speed stat, and no BC behavior/validator touches time (all checked), so the bonus now sits on the stats every attack path honours. Trade-off: cooldowns/energy regen no longer sped up. (The older note below, "TimeScale does speed attacks", holds for ordinary weapons; it was not the whole story for the BC.) **Visible** in the buff bar (frenzy icon, stack count; name/tooltip in `GameStrings.txt`): stacks are per unit and uneven, and while hidden it was impossible to tell whether two compared units had equal counts (prompted by "Battlecruiser attack speed sometimes changes"). `TimeScale` does speed attacks — Blizzard's Chrono Boost pairs `TimeScale 1.15` with `AttackSpeedMultiplier 0.85` just to cancel it — and the BC's live weapon (`BattlecruiserWeaponSwitch`, 0.225s period) is ordinary; the unit stat tooltip does not reflect `TimeScale`. **Spawned units credit their spawner** (`CycleMod_IsSpawnedChildType`: Interceptor, Locust, Broodling, Auto-Turret, Infested Terran, Disruptor nova): `CycleMod_OnUnitSpawned` (`TriggerAddEventUnitCreated`) applies `TardigradeMod_VeteranLink` to the spawn **with the spawner's root as the caster** — the buff *is* the record, read back by `CycleMod_VeteranRoot` via `UnitBehaviorEffectUnit(.., c_effectLocationCasterUnit, ..)`, so it needs no side table and is cleaned up with the unit. `CycleMod_VeteranRoot` redirects the kill in `CycleMod_OnUnitDied`, and `CycleMod_SyncVeteranStacks` (at spawn and from the scan) mirrors the root's stacks onto everything it spawns — so a Carrier's whole flight carries the Carrier's veterancy, and stacks no longer die with the disposable unit. The damage handler now records spawned **structures** (Auto-Turret) as last attacker too, which it previously skipped. `CycleMod_VeteranRoot` also falls back to `UnitGetMagazine` (the engine's own Interceptor→Carrier link) when no spawn was recorded. Which unit the created-event calls "the unit" vs "the created unit" isn't documented, so the handler picks the child **by type** and works either way. Ordinary production is untouched (a Barracks never owns its Marines' kills). Caster damage (Psi Storm) already credits the caster, since the damage event names it as the source. **Unverified.** |
| 11 | Auto Refineries | Your gas buildings mine themselves at the 3-worker rate; workers can't go in | Script only (`TardigradeMod_AutoRefinery` is a key/marker, never applied). `CycleMod_UpdateAutoRefinery`, called per unit from the draft-mode scan. **Payout** (`CycleMod_AutoRefineryTick`): each finished Refinery/Assimilator/Extractor (and `*Rich`) stores the mission time it is paid up to in custom-value slot `c_cycleStateAutoGas` (8) and catches up in whole trips of 4 gas (8 rich) every `c_cycleAutoGasInterval` = 2.1 game seconds — ~160 gas per real minute on Faster, a saturated LotV geyser. Gas is drawn from the building's own `c_unitPropResources`, so geysers deplete on schedule and a dry one stops paying. Each trip also adds to `c_playerPropVespeneCollected`, the engine's "gas collected" total, which only real harvests update on their own; without it auto gas was missing from the income stats. **Unverified:** whether the observer Income tab's *rate* (`VespeneCollectionRate` score value, engine-computed) follows that total or counts only worker deliveries. The clock starts on first sight (after 3:00 or on completion), no back-pay. **Round 4, the physical block (current):** on first sight after activation `CycleMod_AutoRefineryTick` snapshots the building's remaining gas into custom value `c_cycleStateAutoGasLeft` (9), then `CycleMod_MakeUnharvestable` **removes its resource behavior** (`Harvestable[Rich]VespeneGeyserGas[Protoss|Zerg]`, one per type). With no resource behavior the building isn't a resource: no worker can gather from it (as a MULE can't gather gas) and the "x/3" counter, which that behavior draws, disappears. Payouts come from the snapshot; the behavior is re-removed every tick in case anything restores it; failure to remove is debug-logged. On death, `CycleMod_RestoreGeyserGas` sets the reappearing geyser's resources to the snapshot. **Unverified:** that the engine allows removing a resource behavior, and how workers already inside at that moment are ejected. The order redirect and `ResourceAllowed` writes below remain as inert fallbacks. **Previously — the intended block was data, not orders.** `ResourceAllowed[Vespene] = 0` on the side's `SCVHarvest`/`ProbeHarvest`/`DroneHarvest` (`CycleMod_BlockGasHarvest`, per-player `CatalogFieldValueSet` at 3:00) is the field that means exactly "this worker may not take gas" — it is how `MULEGather` is minerals-only. **Bug found in the first attempt:** it wrote the `[Vespene]` name form and only tried the `[1]` index form if the value read back wrong, which can't happen — a per-player override reads back whatever was written, valid path or not — so the fallback never ran. Both forms are now written unconditionally and both are logged. Alternatives considered and rejected: no unit state/modify flag means "unharvestable" (checked the whole flag vocabulary), and `CBehaviorResource.RequiredAlliance` only ever appears as `Control` in Blizzard's data, so there's no known value that refuses the owner. The script redirect is now a **fallback behind `c_cycleAutoGasRedirect`** (set false to rely on data alone). **Blocking workers, round 3:** playtest round 2 showed workers still going in for one trip (the scan only caught them on the way out), so the catalog block below evidently isn't taking effect. The entry block is now `CycleMod_OnWorkerGatherOrder`: `TriggerAddEventUnitOrder` on `SCVHarvest`/`ProbeHarvest`/`DroneHarvest` command 0 (Gather) fires the moment a worker *receives* a gather order. If the target is an auto refinery, the worker is re-sent to minerals via `CycleMod_SendWorkerToMinerals` after one game loop (0.0625s), so the re-order isn't overwritten by the order being applied. Unverified. **Earlier rounds:** the order-intercept alone **failed in playtest** (workers still went in and mined), so the primary block is now data: at 3:00 `CycleMod_BlockGasHarvestTeam` sets `ResourceAllowed[Vespene] = 0` on the drafting side's `SCVHarvest`/`ProbeHarvest`/`DroneHarvest` via per-player `CatalogFieldValueSet` — the same field that makes Blizzard's `MULEGather` minerals-only. It tries the named index, reads back, and falls back to `[1]` (unsure which a field path accepts); the resulting value is written to the debug log. Called from the scan loop's activation block, not `CycleMod_ActivateDelayedMods` (defined above it). **Clean-up** (`CycleMod_KeepWorkerOutOfGas`): any worker with a harvest order on an auto refinery *or* `UnitIsHarvesting(…, c_resourceTypeVespene)` is sent to the nearest mineral patch within 12 (neutral-owned `HarvestableResource`), returning carried gas first; `stop` if there is none |
| 12 | No Bans | Your opponent gets no unit-draft bans against your pool | Acts in `RosterDraft.galaxy` — see Unit Draft above. The one modifier not gated by the 3:00 delay (the roster draft runs before the game). `TardigradeMod_NoBans` is a key/marker only |
| 13 | Refund | Anything of yours an enemy kills pays back 25% of its cost | **No ability, no button.** `CycleMod_PayRefund`, called from `CycleMod_OnUnitDied` before `CycleMod_ForgetUnit` drops the recorded attacker it depends on. Reads `CostResource[Minerals]` / `[Vespene]` off `c_gameCatalogUnit` for the dead unit's type and pays `c_cycleRefundFraction` (0.25) of each back with `PlayerModifyPropertyFixed`. Paid as **fixed, not rounded** — a Marine is 50/4 = 12.5, and flooring every payout would quietly lose an eighth of the modifier over an army's worth of deaths. Morph costs are cumulative in the data (Lair 475 = Hatchery 325 + 150; Zerg costs include the Drone), so the catalog number is already the right base. **Gated on an enemy killer**, which is load-bearing: the unit-died event fires for far more than combat deaths — a Zergling morphing into a Baneling is killed by `MorphZerglingToBaneling`'s `KillOnFinish`, a MULE times out, a shade expires, a Larva is consumed — and paying out on those would refund units the player never lost, making morphs an outright minerals printer. A null or friendly killer pays nothing. **Hallucinations are excluded** for the mirror-image reason: they carry the real unit's type, so `CostResource` would read the real price and a Sentry could print minerals by feeding copies to the enemy. Workers pay nothing, like every other modifier bar Free Labor and Auto Refineries — the blanket rule is worth more than the edge case, and it keeps Refund out of worker trades and harassment. **Was Salvage**, a `CAbilBehavior` toggle granted to 64 structures with computed card slots, a 5s channel and a 75% refund; all of that data is gone |
| 14 | Shared Damage | Each hit on your unit (after armor): it takes half, the other half is split evenly across your nearby units; alone it takes all | **Absorb and redeal.** `TardigradeMod_SharedDamage` carries `DamageResponse ModifyFraction=0 ModifyMinimumDamage=1` (Blizzard's `DamageTakenNone`), so the hit never lands; `TriggerAddEventUnitDamageAbsorbed` fires `CycleMod_OnSharedDamageAbsorbed`, which computes X (`CycleMod_SharedHitAmount`: absorbed − victim armor × the effect's `ArmorReduction`, rounded half-up to a whole number, floor `c_cycleShareMinTotal` = 1 — responses run before armor, so the absorbed amount is pre-armor; shield armor if shields are up), finds partners (`CycleMod_SharePartners`: allied units within `c_cycleShareRadius` = 3 that carry the behavior, not dead/hidden/stasis/invulnerable) and splits X **in whole points**: the victim takes `ceil(X / 2)` and the remaining `floor(X / 2)` is handed out in chunks of at least `c_cycleShareMinShare` = 1, which caps the number of recipients at `rest / minShare` — so a 4-damage bite reaches exactly two partners for 1 each however many are standing there, instead of giving ten units 0.2 apiece. An uneven division gives one extra point to the first `rest mod taking` of them, so the pieces sum to exactly X. With no partners in range, or nothing left to hand out, the victim takes all of X. Dealt through `TardigradeMod_SharedDamageHit` (Amount 0, ArmorReduction 0, Kind Spell) in the **original attacker's** name. Heal-back after the fact was rejected: the damaged event fires after damage lands, so the lethal hits the mod exists to spread would already have killed. The share effect is in the response's `ExcludeEffectArray` — the only thing stopping infinite re-splitting. Also excluded: the Overwatch bonus (already dealt per share) and ~20 **Kill-flag effects** (Baneling `Suicide`, `KillHallucination`, `MULEFate`, shade end, Bile vs Force Field, …), which would otherwise leave their target alive. `CycleMod_CanShareDamage` keeps structures, hallucinations and `CycleMod_IsShareExemptType` units (larva/eggs/cocoons, MULE, shade, interceptors, locusts, broodlings, changelings, Force Field, Parasitic Bomb dummy, Disruptor ball) from ever carrying it. If the attacker is gone, the victim authors its own shares and `CycleMod_OnUnitDamaged` ignores friendly-authored shares so they can't feed Predator/Overwatch |

Notes:
- **War Economy was removed** (it made no sense without phases) and replaced by
  what is now Forced March at slot 2. Its behaviors are gone from `BehaviorData.xml`;
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
  `Row="1" Column="2"` (Blink) / `Column="3"` (Medivac Boost) — same row, all
  since deleted —
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
- Modifier buffs carry a short `Duration` (8s) but the 0.5s scan re-applies
  them, so they behave as permanent while the modifier is active and fall off on
  their own if the scan stops applying them. Two behaviors invert that and use
  `Duration` as a real timer, because nothing re-adds them: `TardigradeMod_MarchBroken`
  (14s, Forced March's recovery window) and `TardigradeMod_BlinkCooldown` (17s).
  Both are re-added on the triggering event, which refreshes the duration — that
  is how "14s since the LAST hit" works without a per-unit timestamp.
- Veteran stacks: every multiplier (move, attack, rates, regen) is 1.03 per stack; 15 stacks ≈ +45–56% depending on whether the engine combines stacks additively or multiplicatively.
- **Known gap:** `TardigradeAbil_Blink` has no custom actor wiring, so casting
  it teleports the unit with no blink flash/sound (the Stalker's own Blink
  visuals are keyed to effect id `Blink` specifically, not reusable by a
  same-behavior clone under a different id without duplicating those actors
  too). Functional but silent — flagged as a follow-up, not attempted blind.
- **Salvage's command-card slots were computed, not guessed** — kept as history; the modifier is Refund now and none of these grants remain (the Blink/Boost
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
| `EffectData.xml` | `TardigradeAbil_Blink` (`CEffectTeleport`) — created directly by `CycleMod_TryBattleBlink`, no ability wraps it — `TardigradeMod_OverwatchBonus` (`CEffectDamage`), the payload `CycleMod_OnUnitDamaged` fires for Overwatch, and `TardigradeMod_SharedDamageHit` (one Shared Damage share). |
| `AbilData.xml` | `LarvaTrain` / `GatewayTrain` / `WarpGateTrain` fallback InfoArray entries only. **No modifier abilities left** — Blink, Salvage and Medivac Boost were all removed in favour of conditions. |
| `UnitData.xml` | `Larva` command-card layout for the larva-build fallbacks, and nothing else. The ~150 lines of `AbilArray` grants and `CardLayouts` cells (Blink + Medivac Boost on 49 combat units, Salvage on 64 structures) are all gone — no modifier puts a button on any command card. |
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
| Modifier balance | Forced March `MoveSpeedMultiplier=1.4` / 14s recovery, Battle Blink 30% trigger / 17s cooldown, Refund 25%, Veteran `1.03 all speeds/rates/regen ×15`, Shared Damage radius 3 with no cap on Y, Auto Refineries' 2.1s trip interval, Entrenchment's 1.5 anchor radius — all first-pass. Three values moved together in this pass and none has been played: Forced March 1.3 → **1.4**, Entrenchment's hold 6 → **10.5**, Overwatch's idle 10 → **14** with a new **1.4s bonus window** where it used to be one shot. Both waits are far longer than before, so each reward should be rarer and worth more; Overwatch's window is what pays for its wait, and Forced March's 1.4 for the fact that dealing damage now breaks it. Entrenchment sits below Overwatch because holding position costs more than holding fire. Watch the pair together: static defence clears both bars for free (a cannon never moves, so it is permanently entrenched, and it idles far past 14s between harass waves), and the longer the waits get the more these read as turtle buffs rather than army buffs. If that is what playtesting shows, `CycleMod_IsCombatUnit` is already a Structure-attribute test and the worker exclusion in `CycleMod_UnitModActive` is the pattern to copy. |
| **Modifiers 11–14, unverified in-editor** | Nothing below has been compiled or played. In order of risk: **(1) Shared Damage** — test first: hit one of the drafting side's units after 3:00. If it takes **no damage at all**, `TriggerAddEventUnitDamageAbsorbed` isn't firing for a `ModifyFraction=0` response (the handler already falls back to attempted − landed if the absorbed amount reads 0, but can't help if the event never fires). Also check the split is post-armor as intended — the handler assumes the absorbed amount is pre-armor. And confirm a Baneling on that side still dies when it detonates and an Adept shade still expires (Kill-flag exclusions). **(2) Salvage** — confirm the button renders in the computed cell, the 5s timer shows, the refund is 75% (Lair should return 356/75), and damage cancels it. **(3) Auto Refineries** — confirm gas lands at ~160/real min, that `c_unitPropResources` on the refinery is the geyser's remaining gas (if it reads 0 the building will never pay), and that evicted workers go to minerals. **(4) No Bans** — pure draft logic, lowest risk. |
| Activation delay | 3:00 is a starting value (`c_cycleActivationDelay`). |
| Battle Blink has no visual/audio feedback | `TardigradeAbil_Blink` teleports silently — the Stalker's Blink flash/sound actors are keyed to effect id `Blink`, not reusable under our separate id without duplicating those actor entries too. Not attempted blind (unverifiable without the SC2 Editor); functional but silent. |
| ~~Command-card placement~~ | **Resolved by deletion.** No modifier grants a button any more, so there are no cells to place, no Row 3 rendering problem, and no collisions with vanilla buttons. Kept here as history: the grants went through Row 3 (didn't render), Row 1 Col 0/1 (collided on Ghost, Infestor, …) and finally Row 1 Col 2/3 before the whole approach was dropped. |
| In-game modifier panel | Static top-center, 214px tall — reposition/shrink if intrusive. |
| Stale comments | `RosterDraft.galaxy` + `RosterEnforce.galaxy` headers still say "2 core + 4 drafted" (behavior is 6 drafted, no core); `RosterDraft.galaxy` also lists the wrong snake order and a "12-unit pool". `TardigradeLogic.galaxy` still calls the modifier step "the cycle" / "day/dusk/night". |
| Dead War Economy code | `c_cycleStateEconomicEgg`, `CycleMod_IsEconomicEgg`, `CycleMod_OnEconomicEggStarted` and helper indices 1–2 are inert but still compiled/registered. |
| Dead Adrenal Response code | `c_cycleStateAdrenalReady`/`c_cycleStateWasLow` (custom-value slots 4/5) and `CycleMod_HelperBehavior(5)` are now the same kind of harmless-but-inert leftover, since Battle Blink replaced that mechanic. |
| Draft-time opponent roster panel, unverified in-editor | Don't confuse with the in-game HUD above (already fixed). `RosterDraft_UpdateRosterPanel`'s dual panel *during the draft itself* (YOUR + OPPONENT, live, shown while picking) already existed in code before this session and looked complete on read-through — if it's not showing up in an actual playtest, that's a rendering/timing bug to hunt for in the editor, not a missing feature to build from scratch. |
| Everything in this batch needs an SC2 Editor recompile to verify | Per the compiler gotcha below, several of these fixes (ability grants, filters, stalemate override, per-player dialogs) touch areas the VS Code linter cannot validate. |
| Split immediate/deferred starting-unit spawn, unverified in-editor | Five iterations to get right — see the History note under Race Draft above. Current (v5): only the town hall spawns immediately at Race Draft finish (kept unselectable via `Tardigrade_DraftLock_Start()`/`_Stop()`, confirmed in playtest to actually block commands); workers + extra unit are deferred to `Tardigrade_SpawnDeferredWorkers()`, called after the countdown so nothing mines or is commandable while the draft UI is up. Needs a live playtest to confirm: newly spawned Larva get caught by the 0.5s scan before a player can act on them, and the workers don't have a selectable window between spawning and `Tardigrade_DraftLock_Stop()`'s release sweep. |
