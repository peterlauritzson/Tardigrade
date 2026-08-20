# Tardigrade

A StarCraft II Extension Mod for 1v1 (and team) melee games. It runs a short
**three-part draft** before the match — races, battlefield modifiers, unit
rosters — and then plays out as standard melee, with each side fighting under
its own drafted set of always-on modifiers and its own restricted unit roster.

Everything else is vanilla SC2: normal economy, normal tech, normal win
conditions.

---

## The three drafts

The drafts run in sequence on map init, while the game is paused. A 3-2-1
countdown follows, then workers spawn and the game begins.

```
Race Draft → Modifier Draft → Unit Draft → 3-2-1 → Game
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

There are **10 battlefield modifiers**. This is a **per-player** draft — there is
no rotation and no shared phase:

- Each player **bans 2** (10 → 6), ban order `P1, P2, P1, P2`.
- Each player then **picks 3** (6 → 0), snake order `P1, P2, P2, P1, P1, P2`.

Every modifier is taken; each pick is **always active in-game, but only for the
picking side's own units** — never the opponent's. A top-center panel lists both
sides' picks (YOURS / OPPONENT).

All modifiers stay **dormant for the first 3 minutes** and switch on at the 3:00
mark (`c_cycleActivationDelay` in `CycleMod.galaxy`), so the early game is
untouched.

The 10 modifiers:

| # | Modifier | Effect (applies to your units only) |
|---|---|---|
| 1 | Open Skies | All your weapons can target ground and air |
| 2 | Medivac Boost | Clickable ability: burst a unit's move speed (the Medivac's afterburners), 15s cooldown |
| 3 | Free Labor | Your workers no longer cost supply |
| 4 | Predator Protocol | Your attacks restore 30% of the damage dealt |
| 5 | Eyes Everywhere | The battlefield and hidden units are revealed, for you only (excludes neutrals — minerals, Xel'Naga towers, critters) |
| 6 | Entrenchment | Your stationary units gain +2 armor and +1 range after 3 seconds |
| 7 | Arcane Surge | +3 energy per second and +50 maximum energy |
| 8 | Overwatch | Your first attack after 5 seconds idle gains +3 range and +50% damage — for that one shot only |
| 9 | Battle Blink | Clickable ability: short-range teleport (8 range) any unit, 12s cooldown |
| 10 | Veteran Forces | Each kill grants a permanent +3% time-speed (haste) buff, stacking to 15 |

Medivac Boost and Battle Blink are the only two modifiers that grant a
**clickable ability** (with its own command-card button) instead of a passive
buff — every other pick is always-on for as long as the modifier is active.

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
- **Final snake picks:** the remaining 4 each, until both rosters hold 6.

Both rosters are shown live side by side (YOUR / OPPONENT) so players can
counter-draft, with a modifier reference strip along the bottom.

The roster is **hard-enforced** in-game: every combat unit you did not draft is
disabled via `TechTreeUnitAllow`. Workers, town halls, production/tech
structures and supply are never touched, so the economy and tech tree work
normally.

Draft pools live in `TardigradeRosterConfig` (`GameData.xml`) and are
independent per race — Terran 15, Protoss 16, Zerg 14 units.

Special cases:

- **Detection floor:** Observer (Protoss) and Overseer (Zerg) sit outside the
  pool entirely and are **always buildable**, so detection is never drafted away.
- **Hellbat** comes with Hellion (they transform into each other).
- **Archon** is not a pool pick; it unlocks if you drafted either templar.
- **Ravager / Lurker / Brood Lord / Baneling** get a direct larva-build path
  (`AbilData.xml`), so they stay buildable even when their morph parent was
  banned.

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
