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
- Pool of **10 modifiers**. Each player **bans 2** (10 → 6), then **picks 3**
  (6 → 0). All post-ban modifiers get taken.
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
  Pick order (steps 1–12, first 4 are the pre-ban openers):
  `P1 P2 P2 P1 | P2 P1 P1 P2 P2 P1 P1 P2`.
- Draft pools are data-driven (`TardigradeRosterConfig` in `GameData.xml`) and
  independent per race: **Terran 15, Protoss 16, Zerg 14**.
  The `RosterDraft.galaxy` header comment ("4 drafted, 2 core auto-assigned",
  snake order `P2 P1 P1 P2 …`, "12-unit pool") is **stale on all three counts**
  — trust the constants and `RosterDraft_Start`.
- Dual live roster panel (YOUR + OPPONENT) so players can counter-draft, plus a
  modifier reference strip at the bottom.
- Detection floor: Observer (Protoss) and Overseer (Zerg) are **always
  buildable** (removed from the pool, never disabled) so detection is guaranteed.
- Coupled/derived units handled in `RosterEnforce`: e.g. Hellbat granted with
  Hellion; Archon granted if HighTemplar or DarkTemplar drafted; Ravager /
  Lurker / Brood Lord / Baneling reachable via larva-build fallbacks.

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
- Open Skies is a catalog change (not per-unit), so it is applied once at 3:00
  via `CycleMod_ActivateDelayedMods()` (fired from the scan loop, guarded by
  `g_cycle_delayedApplied`), which also posts a chat announcement.

### How effects reach units
- `CycleMod_ScanTrigger` — 0.5s loop over all map units. In draft mode it applies
  each side's picked behaviors to that side's own units and clears the rest.
- `CycleMod_UnitModActive` maps a unit's owner → side (via `g_draft_p1_is_team1`
  + team groups) → whether that side drafted the behavior.
- Ownership helpers: `CycleMod_OwnerIsP1/P2`, `CycleMod_P1Team/P2Team`,
  `CycleMod_P1HasModIdx/P2HasModIdx`.
- Event handlers: `CycleMod_OnUnitDamaged` (Predator), `CycleMod_OnUnitDied`
  (Mutual Destruction, Veteran Forces). Conditional per-unit state handled in
  `CycleMod_UpdateConditionalBehaviors` (Entrenchment, Overwatch, Adrenal).
- In-game reference panel `CycleMod_InitDraftPanel`: top-center, per-viewer
  **YOURS / OPPONENT** (spectators see PLAYER 1 / PLAYER 2), lists both sides'
  3 modifiers; title notes "(active at 3:00)".

### The 10 modifiers
| # | Name | Effect | Implementation |
|---|---|---|---|
| 1 | Open Skies | All your weapons can hit ground and air | Per-player weapon `TargetFilters` via `CatalogFieldValueSet(..., player, ...)`; strips `Ground`/`Air` from required+excluded |
| 2 | Medivac Boost | Click a unit to burst its move speed (afterburners) | **Clickable ability** `TardigradeAbil_MedivacBoost` → applies buff `TardigradeMod_MedivacBoost` (`MoveSpeedMultiplier=1.7`, 15s cooldown, no cost). Granted hidden to every combat unit + worker; shown/enabled only while the side's pick is active (`CycleMod_UpdateAbilityMods`) |
| 3 | Free Labor | Your workers no longer cost supply | `UnitSetState(u, c_unitStateUsingSupply, false)` on `CycleMod_IsWorker` units (SCV/Probe/Drone); buff `TardigradeMod_WorkersNoSupply` is applied alongside as a visual marker only, it does nothing mechanically |
| 4 | Predator Protocol | Your attacks heal 30% of damage dealt | `CycleMod_OnUnitDamaged` heals the source |
| 5 | Eyes Everywhere | Reveals the map + hidden units, **for you only** — excludes neutrals (minerals, Xel'Naga towers, critters) | Buff with `Detect=500 Radar=500 DetectFilters="-;Neutral" RadarFilters="-;Neutral"` on your units |
| 6 | Entrenchment | Your stationary units get +2 armor / +1 range after 3s | Marker → `TardigradeMod_Entrenched` helper when still |
| 7 | Arcane Surge | +2 energy/s | Buff modifying Energy vitals (`VitalRegenArray`, no `VitalMaxArray`) |
| 8 | Overwatch | First attack after 5s idle: +3 range, +50% damage, consumed on that one shot | Marker → `TardigradeMod_OverwatchReady` helper, added only on the idle→ready edge (guarded by `UnitBehaviorCount`), removed the instant the shot lands in `CycleMod_OnUnitDamaged` |
| 9 | Battle Blink | Click a unit to short-range teleport it (8 range) | **Clickable ability** `TardigradeAbil_Blink` (`CEffectTeleport`, cloned from the Stalker's Blink minus its tech requirement, own cooldown). Same grant/show mechanism as Medivac Boost |
| 10 | Veteran Forces | Each kill = permanent **+3% time-speed (haste)**, stacking to 15 | `TardigradeMod_VeteranStack`, `TimeScale=1.03`, `MaxStackCount=15`, added on kill |

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
  combat unit + worker via `AbilArray` in `UnitData.xml` (52 units) so
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
- The enforce lists are **not** the draft pools. They additionally contain the
  derived units (`HellionTank`, `Archon`) that are granted rather than drafted.
  `Observer` and `Overseer` appear in **neither** list — that absence is exactly
  what keeps detection always buildable.
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
| `BehaviorData.xml` | `TardigradeMod_*` modifier buffs + helper behaviors (`Entrenched`, `OverwatchReady`, `VeteranStack`, etc.). |
| `EffectData.xml` | `TardigradeAbil_Blink` (`CEffectTeleport`) + `TardigradeAbil_MedivacBoost` (`CEffectApplyBehavior`) — the effects behind the two ability-based modifiers. |
| `AbilData.xml` | `LarvaTrain` fallback InfoArray entries, plus `TardigradeAbil_Blink` / `TardigradeAbil_MedivacBoost` ability definitions. |
| `UnitData.xml` | `Larva` command-card layout for the larva-build fallbacks, plus `AbilArray` grants of both modifier abilities (hidden by default) to ~50 combat units + workers across all three races. |
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
| Modifier balance | `MoveSpeedMultiplier=1.7`, `TimeScale=1.03×15`, ability cooldowns (12s Blink / 15s Medivac Boost, both uncosted), etc. are first-pass. |
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
