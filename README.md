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
| 1 | Fleet-Footed | +50% move speed |
| 2 | Adrenal Frenzy | +30% attack speed |
| 3 | Extended Optics | +2 weapon range |
| 4 | Ironhide | +3 armor |
| 5 | Juggernaut | +50 max life |
| 6 | Rapid Regeneration | +5 life / second |
| 7 | Overcharged Munitions | +20% weapon damage |
| 8 | Blitz Doctrine | +25% move & +15% attack speed |
| 9 | Siege Protocol | +1 range & +15% damage |
| 10 | Bulwark | +2 armor & +30 max life |

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
