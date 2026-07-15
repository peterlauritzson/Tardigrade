# Tardigrade — Current State

Clean slate, forked from the Tardigrade prototype and stripped down to three
features. Standard SC2 melee otherwise.

---

## What's built

### Draft chain (runs on map init, game paused)
`Race → Unit → Cycle → Game`, wired in `TardigradeLogic.galaxy`.

1. **Race Draft** (`RaceDraft.galaxy`) — P1 bans, P2 picks, P1 gets remainder.
   Each team is set to one shared race; every race gets a normal melee opening
   (town hall + 12 workers + supply where applicable).
2. **Unit Draft** (`RosterDraft.galaxy`) — cross-ban phase then a 1-2-2-1 snake
   pick. Each side ends with 6 units (2 core + 4 drafted). Results in
   `g_roster_p1Units[]` / `g_roster_p2Units[]`.
3. **Cycle Draft** (`CycleMod.galaxy`) — P1 picks a modifier, P2 picks a second,
   the third is random. Chosen modifiers stored in `g_cycle_chosen[1..3]`.

### Roster enforcement (`RosterEnforce.galaxy`)
At game start, for every player on each team, every combat unit **not** in that
team's roster is disabled via `TechTreeUnitAllow(player, unit, false)`. Master
combat lists are defined per race. Workers/structures/supply are untouched.

### Cycle modifiers (`CycleMod.galaxy` + `BehaviorData.xml`)
- 10 global buff behaviors `TardigradeMod_*` (verified `Modification` fields).
- Three drafted modifiers rotate as Day/Dusk/Night phases.
- A scan loop (every 2s) applies the active phase's buff to **all units** and
  removes the other two, so the swap is clean and new units are covered.
- Phase loop updates lighting + the bottom-left HUD (name, effect, countdown,
  next-phase icon).
- Config in `GameData.xml > TardigradeCycleConfig` (phase duration 45s default).

### Debug (`Debug.galaxy`)
- Single-player launch auto-enables debug mode.
- Race draft shows a `[DEBUG: Random All]` / per-race auto buttons that skip all
  drafts and randomize races, rosters, and the cycle.

---

## Data files
| File | Contents |
|---|---|
| `BehaviorData.xml` | The 10 `TardigradeMod_*` cycle buffs. |
| `GameData.xml` | `TardigradeRaceStartConfig`, `TardigradeRosterConfig`, `TardigradeCycleConfig`. |
| `EffectData / UnitData / AbilData / ActorData / ValidatorData .xml` | Empty (all prototype content removed). |

---

## Known TODO / tuning
| Item | Notes |
|---|---|
| Modifier balance | Values (+50% speed, etc.) are first-pass; tune for play. |
| Phase duration | 45s is a starting value; raise for longer phases. |
| Cycle buff scope | Applied to *all* units incl. neutral/structures (harmless but broad). Narrow the scan filter if undesired. |
| PvAI enforcement | When one human "deciders" for both sides (solo/PvAI), only the human team's roster is enforced. |
| Lighting links | `MarSaraDayTest` / `BelShirSunset` / `MarSaraNightTest` are placeholders; swap for preferred looks. |
