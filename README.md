# Tardigrade

A lightweight StarCraft II Extension Mod for 1v1 (and team) melee games. It adds
a short, three-part **draft** before the match, then plays out as standard melee
under a rotating set of global battlefield modifiers.

Everything else is vanilla SC2: normal economy, normal tech, normal win
conditions. The mod only touches three things.

---

## The three phases

### 1. Race Draft
Players draft their races instead of choosing them:

- **Player 1 bans** one of the three races.
- **Player 2 picks** their race from the remaining two.
- **Player 1** is assigned the last remaining race.

In team games (2v2, 3v3, …) each team plays a **single shared race** — the whole
team is set to the drafted race.

### 2. Unit Draft
Instead of the full tech tree, each side fields a small custom roster of **6 unit
types** (2 fixed "core" units + 4 drafted):

- **Ban phase:** players alternate banning units from their opponent's pool.
- **Snake pick:** players alternate drafting units from their own pool until each
  side has 4 picks (plus the 2 core units).

The roster is **hard-enforced** in-game: you can only build the units you
drafted. Workers, town halls, production/tech structures, and supply are always
available, so your economy and tech tree work normally.

Core units per race:

| Race | Core 1 | Core 2 |
|---|---|---|
| Terran | Marine | Medivac |
| Protoss | Stalker | Observer |
| Zerg | Zergling | Queen |

(The draftable pools and core units are data-driven — see
`TardigradeRosterConfig` in `GameData.xml`.)

### 3. Cycle Modifier Draft
There are **10 global battlefield modifiers**. Three are drafted:

- **Player 1 picks one**, **Player 2 picks a second**, and the **third is random**.

The three chosen modifiers then rotate as the **Day → Dusk → Night** cycle. The
active phase's modifier applies to **every unit on the map, on both teams**. A
bottom-left HUD shows the current modifier and a countdown to the next phase.

The 10 modifiers:

| # | Modifier | Effect |
|---|---|---|
| 1 | Open Skies | All weapons can target ground and air |
| 2 | War Economy | Workers gather faster; combat-unit production is 95% slower, while Drone and Overlord eggs retain normal speed |
| 3 | Mutual Destruction | Slain combat units explode, damaging nearby units |
| 4 | Predator Protocol | Damage dealt restores 30% of the attacker's life |
| 5 | Eyes Everywhere | The battlefield and hidden units are revealed |
| 6 | Entrenchment | Stationary units gain +2 armor and +1 range after 3 seconds |
| 7 | Arcane Surge | +3 energy per second and +50 maximum energy |
| 8 | Overwatch | The first attack after 5 seconds gains +3 range and +50% damage |
| 9 | Adrenal Response | Dropping below 35% life grants a 6-second combat boost; 45-second cooldown |
| 10 | Veteran Forces | Kills grant permanent +3% damage and attack speed, up to 10 stacks |

The original ten stat modifiers remain defined in `BehaviorData.xml` as a
legacy pool, but the draft links only to the modifiers above.

Phase length is data-driven (`TardigradeCycleConfig > PhaseDuration` in
`GameData.xml`, default 45s).

---

## How to play
Tardigrade is an Extension Mod, so it runs on any standard Melee map.

1. Open StarCraft II.
2. Go to **Custom → Melee**.
3. Select a map.
4. Click **Create with Mod**.
5. Search for **Tardigrade** and launch the lobby.

On game start the three drafts run in sequence (the game is paused during
drafting). When testing solo, a debug panel appears with buttons to skip the
drafts and randomize everything.

---

## Project layout
- **`Tardigrade.SC2Mod/`** — the mod source.
  - **`Base.SC2Data/GameData/`** — XML data (cycle-modifier behaviors + config).
  - **`scripts/`** — Galaxy scripts (see below).
- **`references/`** — extracted Blizzard game data, for lookup only (gitignored).

### Scripts
| File | Role |
|---|---|
| `TardigradeLogic.galaxy` | Entry point + draft-chain orchestration. |
| `RaceDraft.galaxy` | Phase 1 — race ban/pick, sets each team's race. |
| `RosterDraft.galaxy` | Phase 2 — unit ban + snake pick UI. |
| `RosterEnforce.galaxy` | Disables every non-drafted unit per team at game start. |
| `CycleMod.galaxy` | Phase 3 — modifier draft + the in-game Day/Dusk/Night cycle. |
| `Debug.galaxy` | Solo-test detection and logging helpers. |

> Note: internal identifiers (the library `LibC9EAC993`, the `Tardigrade*` config
> IDs, and the `Tardigrade.SC2Mod` folder) still carry the original project name
> to keep the SC2 build stable. Only the player-facing name is "Tardigrade".

## License
Unofficial mod for StarCraft II. All assets are property of Blizzard
Entertainment. Code is provided under the MIT License.
