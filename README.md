# Tardigrade

A StarCraft II Extension Mod for 1v1 (and team) melee games. It runs a short
**three-part draft** before the match — races, battlefield modifiers, unit
rosters — and then plays out as standard melee, with each side fighting under
its own drafted set of always-on modifiers and its own restricted unit roster.

Everything else is vanilla SC2: normal economy, normal tech, normal win
conditions.

---

## Game modes

Before anything else, team 1's player (or the only human, in a game against
the AI) picks a mode. Everyone else sees the choice being made.

| Mode | What changes |
|---|---|
| **Casual** | The drafts below with no time limits, then a normal game. |
| **Tournament** | The same, but every draft step (each race ban/pick, modifier ban/pick, unit pick/ban) has a **10-second** limit, shown at the top of the screen. When it runs out, a random legal option is taken for whoever was up. |
| **Testing** | No draft. Everyone keeps their lobby race's town hall and gets **one SCV, one Probe and one Drone** (so all three races can be built), **100,000 minerals and gas**, **instant** build/train/research/morph times and **every unit unlocked**. A panel at the top right switches any modifier on or off for your side at will, with no activation delay. |

The Testing panel is a dialog rather than command-card buttons: a command card
would need an ability per modifier placed on every unit, which is the kind of
data the mod deliberately dropped (see "No modifier grants a clickable ability"
below).

## The three drafts

The drafts run in sequence on map init, while the game is paused. A 3-2-1
countdown follows, then workers spawn and the game begins.

```
Mode → Race Draft → Modifier Draft → Unit Draft → 3-2-1 → Game
```

Throughout, **P1** is the player who bans first and **P2** is the player who
picks first. Referees and spectators see every draft screen read-only.

### 1. Race Draft

- **P1 bans** one of the three races.
- **P2 picks** their race from the remaining two.
- **P1** is assigned the last remaining race.

In team games (2v2, 3v3, …) each team plays a **single shared race** — the whole
team is set to the drafted race. Starting workers, town hall and supply spawn
after the countdown and auto-mine; the worker count is read from the base game
rather than hardcoded, so it tracks whatever the current patch uses.

### 2. Modifier Draft

It opens with the **YOLO?** screen (see modifier 16): each player secretly
chooses to draft normally or go fully random. Then the board: **16
battlefield modifiers** (YOLO isn't one of the cards). This is a **per-player**
draft — there is no rotation and no shared phase:

- Each player **bans 2** (16 → 12), ban order `P1, P2, P1, P2`.
- Each player then **picks 3** (12 → 6), snake order `P1, P2, P2, P1, P1, P2`.

Six modifiers go unpicked in a normal game. Each pick is **always active in-game, but
only for the picking side's own units** — never the opponent's. A top-center
panel lists both sides' picks (YOURS / OPPONENT).

All modifiers stay **dormant at first** and switch on at **about 2:09 on the game
clock** (`c_cycleActivationDelay` = 180 game seconds in `CycleMod.galaxy`; the clock
runs on Faster, 1.4× quicker), so the early game is untouched — except that
**everyone plays with Eyes Everywhere** (full map vision and detection) until that
moment. The opening vision is tied to the same constant, so changing the
activation time moves it too; afterwards only a side that drafted Eyes
Everywhere keeps it. **Auto Refineries** and **Minerals Only** are live **from
the start** — they shape the opening build, so they have to be there for it.
**No Bans** and **YOLO** are also exempt from the delay, since they act on the
unit draft that immediately follows.

Modifiers **never affect workers** (SCV, Probe, Drone, MULE), except Free
Labor, Auto Refineries and Long Reach, which are about workers to begin with.

The 17 modifiers. **Every duration below is in real seconds**, the way you
experience it on Faster — the same numbers the in-game descriptions use. The
data files store game seconds, which tick 1.4× quicker.

| # | Modifier | Effect (applies to your units only) |
|---|---|---|
| 1 | Open Skies | All your weapons can target ground and air. Against the plane a unit couldn't hit before, it deals half damage (Zealot vs air, Phoenix vs ground); units that already hit both are unchanged |
| 2 | Forced March | +40% move speed on all your units. Being in combat switches it off — taking damage or dealing it, either one. It comes back 10s after the last time the unit was hit or attacked |
| 3 | Free Labor | Your workers no longer cost supply |
| 4 | Predator Protocol | Your attacks restore 30% of the damage dealt — health first, then shields once health is full |
| 5 | Eyes Everywhere | The battlefield and hidden units are revealed, for you only (excludes neutrals — minerals, Xel'Naga towers, critters) |
| 6 | Entrenchment | Your units gain +2 armor and +1 range after holding position for 7.5s. “Holding” is lenient — drifting up to 1.5 from where you settled still counts, so being bumped by an ally or shuffling to acquire a target doesn't reset it. Once earned it lingers ~2s after you move off, but re-earning it takes the full 7.5s |
| 7 | Arcane Surge | Double energy regeneration on all your units and structures |
| 8 | Overwatch | After 10s without attacking, a unit deals +50% damage for its next 1s of attacking — everything it lands inside that window, not just the first shot. Spells count as attacking, so a caster opens the window and is then "active". Banelings always get it (they only ever attack once) |
| 9 | Battle Blink | Automatic: a unit that drops to 30% health teleports 8 range straight back from whatever last hit it. Once per 12s per unit — no button, no hotkey |
| 10 | Veteran Forces | Each kill grants a permanent +3% to movement speed, attack speed, ability cooldowns/charges, and life/shield/energy regeneration, stacking to 15 (shown on the unit as "Veteran"). Kills by Interceptors, Locusts, Broodlings and Auto-Turrets count for the unit that spawned them |
| 11 | Auto Refineries | **Live from 0:00.** Your finished gas buildings mine by themselves at the three-worker rate (rich geysers double). Workers physically can't harvest them — the building stops being a resource at all, and its worker counter disappears. Your Refineries / Assimilators / Extractors (and rich versions) cost **200 minerals** |
| 12 | No Bans | Your opponent gets no bans against your pool in the unit draft; their ban turns are skipped |
| 13 | Refund | Any unit or building of yours that dies pays back 25% of its cost (a Marine returns 12.5 minerals), whoever killed it - a detonating Baneling counts. Workers are excluded, like every other modifier. Things that vanish without dying pay nothing: morphs (Zergling into Baneling), Archon merges, cancelled buildings, eggs and cocoons, units timing out |
| 14 | Shared Damage | Every hit on one of your units (after armor): it keeps half, rounded up, and the other half is dealt out in whole points — at least 1 each — to as many of your other units within 3 range as it stretches to. Alone, or with nothing left over, it takes the full hit |
| 15 | Minerals Only | **Live from 0:00.** Nothing you buy costs gas: every gas cost is added to the mineral cost at 1.5× and set to 0 (a 100/100 upgrade costs 250/0, a Stalker 125/50 costs 200/0). Units, structures, morphs and upgrades are all converted. Any gas you mine (or get from Auto Refineries or Refund) is turned into minerals 1:1 |
| 16 | YOLO | **Not on the board — a choice made on its own "YOLO?" screen before the modifier bans.** Each player picks *Draft normally* or *Go YOLO*; the choice is hidden until both have locked in, and both may go YOLO. Going YOLO makes your whole draft chance: **3 random modifiers** rolled from the whole pool (bans and your opponent's picks don't matter; No Bans excluded), your modifier bans made at random, your pick turns skipped, and no unit draft — your roster is dealt: **7 units** at random from your race's pool (instead of 6), with nothing guaranteed. Your opponent can't ban from your unit pool, and your unit bans are random too. If both go YOLO the modifier board is skipped entirely. In Tournament mode the screen has the usual 10s; undecided = draft normally |
| 17 | Long Reach | Your workers build from range 30 and harvest from range 10 |

**No modifier grants a clickable ability.** Every pick is either always-on or
fires itself off a condition. Battle Blink, Salvage and Medivac Boost all used
to be buttons; making them conditions removed every command-card slot and
hotkey the mod added to ~113 unit types, and with them the collision where a
Stalker ended up owning two separate Blinks.

An older set of ten flat stat modifiers (`TardigradeMod_FleetFooted`,
`Ironhide`, `Juggernaut`, …) is still defined in `BehaviorData.xml` as a legacy
pool, but nothing links to it.

### 3. Unit Draft

Instead of the full tech tree, each side fields a custom roster of **6 unit
types**. There are no protected "core" units — every combat unit in the race's
arsenal is bannable and pickable.

- **Opening picks:** 2 each, made *before* the bans, so players can secure key
  units (`P1, P2, P2, P1`).
- **Cross-bans:** 2 each, banning from the *opponent's* pool (`P1, P2, P1, P2`).
  If a side drafted **No Bans**, the opponent's ban turns are dropped and the
  protected side bans twice in a row.
- **Final snake picks:** the remaining 4 each, until both rosters hold 6.

A side that drafted **YOLO** sits this out: its 7-unit roster is dealt before
the first pick, its pick turns are dropped, bans against it are dropped, and
its own bans are made at random for it.

Both rosters are shown live side by side (YOUR / OPPONENT) so players can
counter-draft, with a modifier reference strip along the bottom.

The roster is **hard-enforced** in-game: every combat unit you did not draft is
disabled via `TechTreeUnitAllow`. Workers, town halls, production/tech
structures and supply are never touched, so the economy and tech tree work
normally.

Draft pools live in `TardigradeRosterConfig` (`GameData.xml`) and are
independent per race — Terran 16, Protoss 17, Zerg 14 units.

Special cases:

- **Detection floor:** Observer (Protoss) and Overseer (Zerg) sit outside the
  pool entirely and are **always buildable**, so detection is never drafted away.
- **Derived units are their own picks.** Hellbat, Archon, Baneling, Ravager,
  Lurker and Brood Lord are only available if drafted — you can't train them
  or transform/merge into them otherwise (Hellion ↔ Hellbat and the templar
  merge are blocked when the result isn't in your roster). Drafting one alone
  is enough to build it:
  - **Hellbat** trains from the Factory (as in LotV).
  - **Archon** trains from the Gateway / warps in from a Warp Gate (100/300,
    Templar Archives), if you drafted neither templar; with a templar in your
    roster you merge as usual.
  - **Ravager / Lurker / Brood Lord / Baneling** train straight from larva, if
    you didn't draft their parent; with the parent drafted you morph as usual.

---

## How to play

Tardigrade is an Extension Mod, so it runs on any standard Melee map.

1. Open StarCraft II.
2. Go to **Custom → Melee**.
3. Select a map.
4. Click **Create with Mod**.
5. Search for **Tardigrade** and launch the lobby.

When testing solo, debug mode auto-enables and the race-draft screen gains
`[DEBUG: Random All]` (plus per-race) buttons that skip every draft and
randomize races, rosters and modifiers.

---

## Project layout

- **`Tardigrade.SC2Mod/`** — the mod source.
  - **`Base.SC2Data/GameData/`** — XML data (modifier behaviors, draft config,
    larva-build fallbacks).
  - **`scripts/`** — Galaxy scripts (see below).
- **`references/`** — extracted Blizzard game data, for lookup only. Mostly
  gitignored; only `references/mods/voidmulti.sc2mod/` is tracked. See
  [references/readme.md](references/readme.md).
- **[CURRENT_STATE.md](CURRENT_STATE.md)** — the detailed implementation
  reference: draft chain wiring, how each modifier reaches units, data files,
  compiler gotchas, and open tuning items. **Start there when changing code.**

### Scripts

| File | Role |
|---|---|
| `TardigradeLogic.galaxy` | Entry point, draft-chain orchestration, game start. |
| `GameMode.galaxy` | Game mode select (Casual / Tournament / Testing) and the Tournament draft timer. |
| `TestingMode.galaxy` | Testing mode: start setup, instant builds, the modifier on/off panel. |
| `DraftStyle.galaxy` | Shared card styling for every draft screen. |
| `RaceDraft.galaxy` | Race ban/pick; team, viewer and spectator globals. |
| `RosterDraft.galaxy` | Unit draft UI (opening picks → bans → final picks) + in-game roster HUD. |
| `RosterEnforce.galaxy` | Disables every non-drafted combat unit; grants coupled units. |
| `CycleMod.galaxy` | Modifier draft + per-player in-game modifiers (legacy rotating-phase system kept behind `c_cyclePhaseMode`). |
| `Debug.galaxy` | Solo-test detection, auto-run, logging. |

> Note: internal identifiers (the library `LibC9EAC993`, the `Tardigrade*`
> config IDs, and the `Tardigrade.SC2Mod` folder) still carry the original
> project name to keep the SC2 build stable. Only the player-facing name is
> "Tardigrade".

## License

Unofficial mod for StarCraft II. All assets are property of Blizzard
Entertainment. Code is provided under the MIT License.
