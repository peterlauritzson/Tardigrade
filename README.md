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
the AI) sets the game up on one **setup screen**: a mode, how units are chosen,
how modifiers are chosen, and whether doctrines are in, then **Start** (or
**Custom game / rematch**, below). Everyone else sees the same screen,
disabled, updating as the choices are made. The defaults are Casual, Alternating
draft and Free draft, with doctrines off.

| Setup row | Options |
|---|---|
| **Mode** | Casual / Tournament / Testing (below) |
| **Roster** | **Full roster:** every unit is allowed and there is no unit draft (the in-game roster panel is not shown, and No Bans is unavailable). **Unit groups:** every unit is allowed, and your modifiers each power one group of your units (see Unit groups below). **Alternating draft:** the unit draft below |
| **Modifiers** | **Free draft:** the modifier draft below. **Buckets:** the bucket draft - one ban in each of four buckets, then four picks, one from each (see the Modifier Draft) |
| **Free draft size** | **Bans each** (0-4, default 2) and **Picks each** (1-5, default 3) for the free draft. Each click steps the number up and wraps around. Casual only: unavailable in Tournament (always 2 / 3), with Buckets, and in Testing |
| **Doctrines** | A tick box (default off) and **Up to N each** (1-3, default 2; each click steps up and wraps; unavailable while the box is off). With the box ticked, one more step follows the unit stage (see Doctrines below). Casual and Tournament |

Every row below Mode is greyed out in Testing, which has no draft.
Choosing Unit groups forces the Buckets draft: Buckets shows as chosen and Free
draft is unavailable.


| Mode | What changes |
|---|---|
| **Casual** | The drafts below with no time limits, then a normal game. |
| **Tournament** | The same, but every draft step (each race ban/pick, modifier ban/pick, unit pick/ban) has a **15-second** limit, shown at the top of the screen. Clicking a card only selects it (marked **[>] SELECTED**, visible to you alone); the **Confirm** button next to the clock takes it. When the clock runs out, your selection is taken — or, if you selected nothing, a random legal option. |
| **Testing** | No draft. Everyone keeps their lobby race's town hall and gets **one SCV, one Probe and one Drone** (so all three races can be built), **100,000 minerals and gas**, **200 supply**, **instant** build/train/research/morph times and larva and **every unit unlocked**. A panel at the top right switches any modifier on or off for your side at will, with no activation delay; a second panel to its left does the same for doctrines. |

The Testing panel is a dialog rather than command-card buttons: a command card
would need an ability per modifier placed on every unit, which is the kind of
data the mod deliberately dropped (see "No modifier grants a clickable ability"
below).

### Custom game / rematch

The setup screen's second button, **Custom game / rematch**, skips the draft.
On the custom game screen both players edit both sides directly: race, any
number of modifiers, units (a unit list, Full roster or Unit groups with each
side's split and which modifier powers which group) and any number of
doctrines. Past a draft's counts (5 modifiers, 3 doctrines) the in-game panel's
last row reads "+N more" with the rest on its tooltip, and the summary lists
names only. It opens
with the last game's setup when there is one, and can load a code from a
player's bank file or from a code typed into chat (SC2 does not paste into
chat). Any change clears both players' **Accept**; once both accept, the game
starts. Anything goes: the same modifier on both sides, a mirror match, any
number of units, two modifiers in one group.

Every game shows its **match code** on the summary screen, posts it to chat at
GO and saves it to each player's bank file, `Tardigrade.SC2Bank` (in the
StarCraft II `Banks` folder). In that file, `last` is the code of your last game
and `load` is where you paste a code someone sent you, before starting the game.

```
p1-z-m041119-uzlrohyluinvi+p2-t-m020713-umamdmvtkvkth
```

`+` joins the two sides (`p1` / `p2` = lobby team 1 / 2, so swapping them
trades armies), `-` separates sections, and dots and spaces are ignored:

| Section | Meaning |
|---|---|
| `t` / `p` / `z` | the race |
| `m0411` | modifiers on all your units, 2-digit numbers |
| `uzlro` | units, 2 letters each; leave it out for Full roster |
| `gs071142` | Unit groups: split (`t` tech, `r` role, `s` speed, `c` cost), then modifier + group (07 in group 1, 14 in group 2). Command modifiers always cover all units, so they go in `m` |
| `d0526` | doctrines, 2-digit numbers |

Modifier and doctrine numbers are their numbers on the modifier page and the
custom game screen. The unit tokens are listed in `scripts/MatchCode.galaxy`
(`c_mcUnitTable`): `ma` Marine, `tk` Siege Tank, `zl` Zergling and so on. A bad
code is rejected with the reason, e.g. "p1 modifiers: 27 is too large (max 23)".

## The three drafts

The drafts run in sequence on map init, while the game is paused. A summary
screen follows (both sides' race, modifiers and units) with a 30-second
countdown, then workers spawn and the game begins. Each player has a **Ready**
button there: once both are ready the countdown drops to 3 seconds. A YOLO
player also has **Reroll units**: a new random roster one unit smaller than
the current one (never below 1), which restarts the 30 seconds and un-readies
both players. Draft progress is shown
on the draft screens only; nothing is posted to chat.

```
Setup → Race Draft → Modifier Draft → Unit Draft → [Doctrines] → Summary (30s) → Game
```

Doctrines are an optional extra step (see below); it is skipped unless the
setup screen's Doctrines box is ticked.

Throughout, **P1** is the player who bans first and **P2** is the player who
picks first. Referees and spectators see every draft screen read-only.

### YOLO

Every draft stage - race, modifiers, units and doctrines - has a **YOLO** button for each
human player. It stays there until you make your own first choice (a ban or a
pick) in that stage, then disappears. Pressing it is announced to everyone, and
from then on **your turns in that stage are random**. It is per stage: going
YOLO on the race says nothing about your modifiers or units. YOLO on doctrines
picks random doctrines for you and locks in at once. If it is your turn
when you press, the random action happens at once. An AI side never goes YOLO.
What YOLO means in each stage is described in the stage and under modifier 16.

### 1. Race Draft

- **P1 bans** one of the three races.
- **P2 picks** their race from the remaining two.
- **P1** is assigned the last remaining race.

A side that pressed YOLO on this stage has its ban or pick made at random, so its race is chance.

In team games (2v2, 3v3, …) each team plays a **single shared race** — the whole
team is set to the drafted race. Starting workers, town hall and supply spawn
after the countdown and auto-mine; the worker count is read from the base game
rather than hardcoded, so it tracks whatever the current patch uses.

### 2. Modifier Draft

The board holds **22
battlefield modifiers** (YOLO isn't one of the cards). This is a **per-player**
draft — there is no rotation and no shared phase:

- Each player **bans 2** (22 → 18), ban order `P1, P2, P1, P2`.
- Each player then **picks 3** (18 → 12), snake order `P1, P2, P2, P1, P1, P2`.

In Casual, the setup screen's **Free draft size** row changes both numbers
(0-4 bans, 1-5 picks each). Bans still alternate and picks still snake
(`P1, P2, P2, P1, P1, P2, P2, P1, ...`); with 0 bans the draft starts at the
picks.

Twelve modifiers go unpicked in a normal game. Each pick is **always active in-game, but
only for the picking side's own units** — never the opponent's. A top-center
panel lists both sides' picks (YOURS / OPPONENT). It stays open until you
fold it with its toggle button.

**Buckets draft.** If the setup screen's Modifiers row is **Buckets**, the board is
four columns - **Command**, **Maneuver**, **Endurance**, **Firepower** - and the
draft runs differently: 4 bans (`P1, P2, P1, P2`), each in a bucket nobody has
banned from yet, so every bucket loses exactly one; then 8 picks (`P1, P2, P2, P1,
P1, P2, P2, P1`), each from a bucket you don't have yet (never a banned card or
one your opponent took). You end with **four modifiers, one from each bucket**.
Cards you can't use right now are greyed out. YOLO works the same, but its random
choices obey those rules. No Bans and YOLO are not on the bucket board.

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
Labor and Auto Refineries, which are about workers to begin with.

The 23 modifiers. **Every duration below is in real seconds**, the way you
experience it on Faster — the same numbers the in-game descriptions use. The
data files store game seconds, which tick 1.4× quicker.

| # | Modifier | Effect (applies to your units only) |
|---|---|---|
| 1 | Open Skies | All your weapons can target ground and air. Against the plane a unit couldn't hit before, it deals half damage (Zealot vs air, Phoenix vs ground); units that already hit both, and structures, are unchanged |
| 2 | Forced March | +40% move speed on all your units. Being in combat switches it off — taking damage, dealing it, or casting a spell (any spell from a unit with energy: Auto-Turret, heal, Force Field, cloak…; Medivac Afterburners and the Ghost's Hold Fire don't count). It comes back 10s after the unit's last hit, attack or spell. Disruptor novas, Adept shades and flying Locusts are not sped up |
| 3 | Free Labor | Your workers no longer cost supply |
| 4 | Predator Protocol | Your attacks restore 30% of the damage dealt — health first, then shields once health is full — up to 3% of the unit's max health (life + shields) per second |
| 5 | Eyes Everywhere | The battlefield and hidden units are revealed, for you only (excludes neutrals — minerals, Xel'Naga towers, critters). The modifier tooltip also shows how many workers the enemy has alive (burrowed drones and workers in refineries or transports included, MULEs not) |
| 6 | Entrenchment | Your units gain +2 armor and +1 range after holding position for 7.5s. “Holding” is lenient — drifting up to 1.5 from where you settled still counts, so being bumped by an ally or shuffling to acquire a target doesn't reset it. Once earned it lingers ~2s after you move off, but re-earning it takes the full 7.5s. Units only — structures never entrench |
| 7 | Arcane Surge | Double energy regeneration on all your units and structures |
| 8 | Overwatch | After 10s without attacking, a unit deals +50% damage for its next 1s of attacking — everything it lands inside that window, not just the first shot. Spells count as attacking, so a caster opens the window and is then "active". Banelings always get it (they only ever attack once). Units only — static defense never arms |
| 9 | Battle Blink | Automatic: a unit that drops to 30% health teleports 8 range straight back from whatever last hit it. Once per 12s per unit — no button, no hotkey |
| 10 | Veteran Forces | Each kill grants a permanent +3% to movement speed, attack speed, ability cooldowns/charges, and life/shield/energy regeneration, stacking to 15 (shown on the unit as "Veteran"). Units only — kills by structures earn nothing. Kills by Interceptors, Locusts, Broodlings and Auto-Turrets count for the unit that spawned them. Killing free or disposable units (Locusts, Broodlings, Larva, Changelings, MULEs, hallucinations) earns nothing |
| 11 | Auto Refineries | **Live from 0:00.** Your finished gas buildings mine by themselves at the three-worker rate (rich geysers double). One with none of your finished town halls within 10 mines at a third of that rate. Workers physically can't harvest them — the building stops being a resource at all, and its worker counter disappears. Your Refineries / Assimilators / Extractors (and rich versions) cost **200 minerals** |
| 12 | No Bans | Your opponent gets no bans against your pool in the unit draft; their ban turns are skipped. If you went YOLO on the unit stage: 1 extra random unit (8 instead of 7) — YOLO rolls can now land on it. Unavailable (greyed out on the board) with Full roster, which has no unit draft |
| 13 | Refund | Any unit of yours that dies pays back 17.5% of its cost (a Marine returns 8.75 minerals), whoever killed it - a detonating Baneling counts. Workers and buildings are excluded. Things that vanish without dying pay nothing: morphs (Zergling into Baneling), Archon merges, eggs and cocoons, units timing out |
| 14 | Shared Damage | Every hit on one of your units (after armor): it keeps half, rounded up, and the other half is dealt out in whole points — at least 1 each — to as many of your other units within 3 range as it stretches to. A shared piece never takes a unit below 40% of its max health (life + shields), and a unit already under 40% takes none, so it can never kill a unit that wasn't shot at; what the others can't hold goes back onto the unit that was hit. A unit that would die even on its own half takes the full hit and shares nothing (no spreading the overkill), and a unit alone takes the full hit |
| 15 | Minerals Only | **Live from 0:00.** Nothing you buy costs gas: every gas cost is added to the mineral cost at 1.5× and set to 0 (a 100/100 upgrade costs 250/0, a Stalker 125/50 costs 200/0). Units, structures, morphs and upgrades are all converted. Any gas you mine (or get from Auto Refineries or Refund) is turned into minerals at the same 1.5× (a 4-gas trip pays 6 minerals), so refineries are still worth taking |
| 16 | YOLO | **Not on the board — a button on each draft stage** (see YOLO above). On the **modifier** stage your bans are made at random and your picks rolled from the whole pool (bans and your opponent's picks don't matter; No Bans can come up and means +1 unit if you are YOLO on units); if both sides go YOLO there the board is skipped and all six are rolled at once. On the **race** stage your ban or pick is random. On the **unit** stage your roster is **7 units** at random from your race's pool (instead of 6), with nothing guaranteed, revealed at the normal draft's pace (your 6th unit turn rolls the 7th too); your opponent can't ban from your pool, and your own bans are random |
| 17 | Factory Lines | Everything you build, train, warp in or morph into is 15% faster: structures, add-ons, units, warp gate recharge, research, and morphs like Orbital, Lair or Baneling. Hatcheries also spawn larva 15% faster. Not workers |
| 18 | Adrenaline | Taking damage from an enemy gives the unit +50% move speed for 3s; every hit restarts the 3s |
| 19 | Martyrdom | When one of your units dies, 35% of its max health (life + shields) heals your other units within 3 range — split evenly among the injured ones, life first then shields. Anyone who needs less than their share is topped off and the rest goes to the others; what nobody can use is lost. Like Shared Damage, summons, hallucinations, eggs and workers neither give nor receive it |
| 20 | Quickstep | Your units (workers excepted) accelerate and brake instantly and turn twice as fast, standing turns included |
| 21 | Reinforcements | Every unit you create gets +50% move speed for 30s (shown on the unit as "Reinforcements"). Not workers, buildings, hallucinations, larva, eggs or spawned units like Interceptors and Locusts |
| 22 | Field Repair | A unit that has taken no damage for 5s regains 3% of its max health (life + shields) every second until full or hit again: life first, then shields once life is full, on top of their normal regeneration. Not buildings or workers |
| 23 | Photon Coating | The first 8s of each fight, a unit takes 15% less damage (shown on the unit as "Photon Coating"). A fight starts when the unit deals or takes damage, or begins an attack; the coating re-arms after 20s with none of that. Not buildings or workers |

**No modifier grants a clickable ability.** Every pick is either always-on or
fires itself off a condition. Battle Blink, Salvage and Medivac Boost all used
to be buttons; making them conditions removed every command-card slot and
hotkey the mod added to ~113 unit types, and with them the collision where a
Stalker ended up owning two separate Blinks.

An older set of ten flat stat modifiers (`TardigradeMod_FleetFooted`,
`Ironhide`, `Juggernaut`, …) is still defined in `BehaviorData.xml` as a legacy
pool, but nothing links to it.

### Unit groups

With the **Unit groups** roster option there is no unit draft: every unit is
allowed, the modifier draft is the Buckets draft, and after it comes one more
step. You pick one of four **splits** of your race's units into three groups -
**Tech** (where a unit is built), **Role** (how it fights), **Speed** (its base
move speed) or **Cost** (what it costs) - and then give each of your three
non-Command modifiers (your Maneuver, Endurance and Firepower picks) to one of
the three groups. Your Command modifier still applies to everything. Both
players choose at the same time and see each other's choice only on the summary
screen. A group modifier only ever affects units in its group; workers,
structures and units outside every group (Observer, Overseer, Overlord,
Changeling, cocoons and eggs, MULE) get none. Units that change form or spawn
others count as their base unit's group (a sieged Tank, Interceptors, Locusts).
The in-game modifier panel shows each modifier's group in brackets. In
Tournament the step has a 30-second clock, after which your current choice is
locked in; YOLO picks a random split and assignment and locks in at once.

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

A side that pressed **YOLO** on this stage sits the rest out: each of its pick
turns rolls a random unit on the spot (its 6th turn rolls the 7th as well), bans
against it are dropped, and its own bans are made at random for it. The schedule
is rebuilt when the button is pressed; if both sides are YOLO, both 7-unit
rosters are dealt at once. With **Full roster** this whole stage is skipped.

Both rosters are shown live (YOUR / OPPONENT) so players can counter-draft,
each with a line listing what was banned from it. A panel on the left lists
both sides' drafted modifiers with their descriptions.

The roster is **hard-enforced** in-game: every combat unit you did not draft is
disabled via `TechTreeUnitAllow`. Workers, town halls, production/tech
structures and supply are never touched, so the economy and tech tree work
normally.

Draft pools live in `TardigradeRosterConfig` (`GameData.xml`) and are
independent per race — Terran 16, Protoss 17, Zerg 14 units.

Special cases:

- **Detection floor:** Observer (Protoss) and Overseer (Zerg) sit outside the
  pool entirely and are **always buildable**, so detection is never drafted away.
- **Queen floor:** the Queen is in the Zerg pool and can be banned, but Zerg
  can always train **one Queen per completed Hatchery / Lair / Hive**, drafted
  or not, so injects and creep never go away. Without the Queen drafted, the
  normal Queen button greys out once your Queens (training, alive or burrowed)
  match your Hatcheries; its tooltip says so. Drafting the Queen removes the
  cap.
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

### Doctrines

An optional step after the units (tick **Doctrines** on the setup screen). Each
side picks up to N doctrines (1-3, default 2; fewer, even none, is fine), then
locks in. Both players choose at the same time and see each other's picks only
on the summary screen. Picks are not exclusive: both sides may take the same
doctrine, so there is nothing to ban. In Tournament the step has a 30-second
clock, after which your current picks are locked in.

A doctrine is a trade-off for your army. **Every doctrine has a pro and a con,
and the con is always about your own units (or your own units caught in a
blast), so it always bites.** (The pro may depend on who you face; the con never
does, because rosters are known when you pick.) A caster's con hits its casting
(energy, max energy, regeneration), not just its health, since a High Templar
spends its energy at once. Like modifiers, doctrines switch on at the activation time, and they
only touch their own side's units. They do not depend on Unit groups. A card is
marked N/A if none of your units are covered. "Health" means life and shields.

The board has four columns.

**All army units** (casters and support included)
- **Loose Formation** - -40% splash damage taken, -20% health
- **Spell Ward** - -40% spell damage taken, -15% health
- **Deflector Plating** - -20% ranged damage taken, -15% move speed
- **Riot Gear** - -40% melee damage taken, -10% health
- **Phalanx** - +3 armor (life and shields), -25% health
- **Heavy Plating** - +25% health, -15% move speed
- **Stripped Down** - +20% move speed, -2 armor
- **Battle Rhythm** - every 20s your army alternates Surge (+25% attack and move speed) and Lull (-20%), Surge first; you get a message at each turn
- **Last Breath** - once every 45s a hit that would kill a unit does nothing; -20% health

**Units with a weapon** (casters such as High Templar and Sentry are exempt;
weapon damage never includes spells)
- **Heavy Hitters** - +50% weapon damage per hit, attacks 35% slower
- **Hair Trigger** - attacks 60% faster, -35% weapon damage per hit
- **Armor-Piercing** - +40% weapon damage vs Armored, attacks 15% slower
- **Anti-Personnel** - +40% weapon damage vs Light, attacks 15% slower
- **Glass Cannon** - +30% weapon damage, -25% health
- **Skirmishers** - +25% move speed, -20% weapon damage
- **Last Stand** - +50% attack speed below 35% health, -15% health always
- **Pack Tactics** - +5% weapon damage per other unit of yours within 3 (up to +25%), -15% health
- **Lone Wolves** - with none of your other units within 4: +30% attack speed and +2 armor; otherwise attacks 10% slower

**On hit, ranged and melee** (on-hit effects trigger on direct weapon hits only,
not splash or spells; each costs weapon damage)
- **Exposing Fire** - hits make the target take +25% damage from everything for 3s; -20% weapon damage
- **Frost Rounds** - hits slow the target's movement 35% for 2s; -20% weapon damage
- **Ricochet** - half of each hit also strikes the closest other enemy within 2.5; -25% weapon damage
- **Culling Rounds** - a hit that leaves the target below 15% health destroys it; -15% weapon damage
- **Ignite** - hits set the target burning for 3s (about 3% of its max health per second, no regeneration); -25% weapon damage
- **Rapid Fire** (ranged) - +25% attack speed, -1 range
- **Long Barrels** (ranged) - +1 range, attacks 20% slower
- **Sniper Scopes** (ranged) - +2 range, -4 sight (you need a spotter)
- **Bloodlust** (melee) - +35% attack speed, -2 armor

**Whole army and units with energy**
- **Champion** - your most expensive unit gets +60% health and +30% damage of every kind (a new one is crowned if it dies); every other unit -10% health
- **Volatile Cores** - units explode on death for 30% of their max health to everything within 2, your own units included
- **Bounty Hunters** - each enemy army unit your units kill pays you 15% of its cost; -10% health
- **Spellweaver** - +35% spell damage, energy regenerates 30% slower
- **War Banners** - your units within 4 of a unit with energy get +15% attack speed and +1 armor; -40% max energy
- **Overcharged Cells** - double energy regeneration, -35% max energy
- **Deep Reserves** - new units start with full energy, -40% energy regeneration

Workers, structures, Overlords / Overseers / Observers, Changelings, cocoons
and eggs get no doctrine. The in-game modifier panel lists each side's doctrines
under its modifiers.

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
| `ProtectDraft.galaxy` | Protect and Ban: the points + bans screen that replaces a side's unit draft. Dormant: nothing reaches it any more (see IDEAS.md). |
| `RosterEnforce.galaxy` | Disables every non-drafted combat unit; grants coupled units. |
| `UnitGroups.galaxy` | Unit groups: the split + modifier assignment screen, and the "does this modifier reach this unit" lookup. |
| `Doctrine.galaxy` | Doctrines: the pick screen, the 34 doctrines and their unit classes, in-game application, the Testing doctrine panel. |
| `MatchCode.galaxy` | Match codes: the setup as text (parse, encode), writing a code into the draft globals, the bank file. |
| `CustomSetup.galaxy` | The custom game screen: edit both sides or load a code, both Accept. |
| `CycleMod.galaxy` | Modifier draft + per-player in-game modifiers (legacy rotating-phase system kept behind `c_cyclePhaseMode`). |
| `Debug.galaxy` | Solo-test detection, auto-run, logging. |

> Note: internal identifiers (the library `LibC9EAC993`, the `Tardigrade*`
> config IDs, and the `Tardigrade.SC2Mod` folder) still carry the original
> project name to keep the SC2 build stable. Only the player-facing name is
> "Tardigrade".

## License

Unofficial mod for StarCraft II. All assets are property of Blizzard
Entertainment. Code is provided under the MIT License.
