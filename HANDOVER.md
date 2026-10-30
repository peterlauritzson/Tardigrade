# Handover: Custom game and match codes (2026-10-30)

**Committed. Not playtested.** The first version compiled, and the screen opened in an editor test run. The lobby-race fix and the no-caps change came after that and are not compiled.

A rematch in one click, or any setup by hand. Every game now has a readable **match code**, e.g. `p1-z-m041119-uzlrohyluinvi+p2-t-m020713-umamdmvtkvkth`. The setup screen's new **Custom game / rematch** button opens a screen where both players edit both sides or load a code, then both Accept. Format and rules: CURRENT_STATE section 0b.

- **Why a bank file:** SC2 won't paste into chat or dialogs. So at GO the code goes into every human's `Tardigrade.SC2Bank` as `last`. A code from someone else goes into `load` by hand, before the game. Typing a code in chat also works.
- **Code:**
  - New `scripts/MatchCode.galaxy`: the state, parse with per-section errors, encode, `MatchCode_FromLive` / `MatchCode_Apply`, and the bank.
  - New `scripts/CustomSetup.galaxy`: the screen.
  - GameMode: `g_game_custom` and the button. The dialog is now 600x760.
  - CycleMod: `CycleMod_PicksEach` uses the larger side's count in a custom game.
  - UnitGroups: in `UnitGroups_Covers`, group 0 now means all units, but only for a modifier the side holds. An unheld one still covers nothing, which some CycleMod callers rely on. A drafted game never has a held modifier without a group.
- **Review:** a read-only review found no compile errors. Fixed from it:
  - The Covers regression above.
  - Bank signing is now explicitly off.
  - An empty `u` parses, so every code round-trips.
  - Custom game no longer overwrites the Mode row.
  - Limits are counted once per redraw.
  - Open doubt: the `bank[16]` array. The GUI makes bank arrays, so it should compile.
  - TardigradeLogic: includes, the custom branch and callbacks, `MatchCode_LoadBanks` in init, the code on the summary, the chat line and bank save at GO, the pre-readied summary.
- **First editor run (2026-10-30):** opening the screen with no last game threw "Could not get 'player' from parameter in 'PlayerRace' (value: -1)". An editor test can leave a lobby team empty, and Reset read that team's first player. Fixed: `MatchCode_TeamLobbyRace` falls back to the team's decider.
- **No caps (2026-10-30):** the custom game screen and codes take any number of modifiers and doctrines. Display past a draft's counts: the panel keeps 5 rows a side and the last reads "+N more" (tooltip lists the rest); the doctrine row reads "N picked"; the summary lists names only and is capped at 1150 tall. The stats tooltip now reads each team's held modifiers rather than the panel rows (`g_cycle_panelModMine/Opp` removed).
- **Docs:** CURRENT_STATE (0b, chain, setup, Covers, scripts, Gotchas: never renumber modifiers/doctrines; every pool unit needs a token), README (Custom game / rematch), public page version 23 (new sections, "#NN" chips on every modifier and doctrine).
- **Online needs one editor step:** in the trigger editor, add "Bank - Preload Bank" actions to a map-initialization trigger: name `Tardigrade`, one action per player slot (they can't loop). The editor then writes `BankList.xml`. Editor test runs work without it.
- **Check first:** the Known TODO row "Custom game / match codes". The fastest test is: play one drafted game against the AI, check the bank file and the chat line, then restart and use Custom game / rematch.

---

# Handover: Doctrines, second set (2026-10-28)

**Not committed. Not compiled. Not playtested.** Wave 1 below was playtested and works as intended.

This set adds 14 doctrines (21-34), inspired by Legion TD 2 spells and auras and by Mechabellum techs. The board is now 4 x 9.

- **New doctrines:**
  - On hit: Exposing Fire, Frost Rounds, Ricochet, Culling Rounds, Ignite
  - Positional: Pack Tactics, Lone Wolves
  - Timed: Battle Rhythm
  - Cheat death: Last Breath (data-only Fatal damage response)
  - Whole army: Champion, Volatile Cores, Bounty Hunters
  - Energy units: Spellweaver, War Banners
- **Overcharged Cells changed:** now -35% max energy instead of -25% health. Your point: burst casters never traded their health.
- **New hooks** (Doctrine.galaxy):
  - A weapon-hit trigger registered for Melee and Ranged damage only, so it sees direct hits only.
  - A death trigger.
  - `Doctrine_ScanTick`, called once per scan pass.
- **New effects** in EffectData.xml: the bonus effects, which every attacker-side response excludes.
- **CycleMod:** the damage handler now returns early on self-inflicted damage (Ignite's burn, Volatile's blast).
- **Review:** a read-only review found no compile errors. Its three real findings are fixed: the self-damage early return, army-only neighbor counts, and a nested null check.
- **Check first:**
  - Last Breath: a nearly dead Marine survives one lethal shot, then shows "recharging" for 45s. If it never shows that, the unit is immortal. Switch both child effects of `TardigradeDoc_LastBreathUse` to `WhichUnit Value="Target"`.
  - Ignite and Volatile self-damage lands and kills.
  - The 4 x 9 board fits.
  - Full list: CURRENT_STATE Known TODO, items 13-23.

---

# Handover: Doctrines (2026-10-28)

**Not committed. Not compiled. Not playtested.**

A new setup row, DOCTRINES (tick box + "up to N each", 1-3, default 2, off by default). Ticked, a stage after
the unit stage lets both players pick trade-off doctrines at the same time, blind and non-exclusive. 20
doctrines, each a pro plus a con. The con is always a stat on your own units, so knowing the opponent's
roster can't make it free.

- **Code:** new `scripts/Doctrine.galaxy`; edits in GameMode (setup row, `c_stageDoctrine` = 3), CycleMod
  (YOLO arrays [4], scan hook, panel doctrine rows), TardigradeLogic (`Tardigrade_AfterUnits`, summary,
  debug auto-run), TestingMode (second "Test Doctrines" panel). Data: 21 `TardigradeDoc_*` buffs in
  BehaviorData.xml, names/tooltips in GameStrings.txt.
- **Docs:** CURRENT_STATE.md section 3b plus a Known TODO row with the 12-item unverified checklist; README.md;
  IDEAS.md (built pointer + parked doctrine ideas). Public page updated (version 20).
- **Review:** a read-only review found no compile errors or logic bugs.
- **Check first in the editor:** compile, then `VitalMaxFractionArray`. It is written as an added fraction
  (0.25 = +25%) on the SC2Mapster wiki's word. A Heavy Plating Marine must show 56 max life. If it shows 11,
  the field is a multiplier and every health value in the doctrine buffs becomes 1 + value.
- **Fastest test:** Testing mode, then the "Test Doctrines" panel (left of the modifier panel).

---

# Handover: Forced March breaks on spell casts (2026-10-21)

## Where things stand

**Not committed. Not compiled. Not playtested.** Everything below is uncommitted work in progress on `copilot/create-tardigrade-repo`.

## Why

Ravens kept Forced March after dropping an Auto-Turret. Until now the modifier only broke on three events: damage taken, damage dealt (credited to the spawner too), and a weapon attack starting. A spell with no damage, like a turret drop, Interference Matrix, heal, Force Field or Abduct, broke nothing.

You asked: spell casting should always break it, at least for spellcasters.

## What changed

**`Tardigrade.SC2Mod/scripts/CycleMod.galaxy`**
- New `CycleMod_OnUnitCastSpell`, right after `CycleMod_OnUnitStartedAttack`. It breaks the march when a unit **with energy** (`c_unitPropEnergyMax > 0`) uses one of these:
  - `CAbilEffectInstant` / `CAbilEffectTarget`, at stage **Cast** (3)
  - `CAbilBehavior`, at stage **Toggled** (1): cloak, Pulsar Beam
  - `CAbilBuild`, at stage **Construct** (2): Queen creep tumor. (The Oracle's Stasis Ward is a `CAbilEffectTarget` in voidmulti, so the Cast check covers it.)
- Stage numbers overlap between ability classes, so the handler uses the class to decide which stage counts. The class ids (11/14/15, plus the existing `c_cycleAbilClassBuild`=12) come from core `TriggerLibs/GameData/Abil.galaxy`.
- `CycleMod_IsMarchSafeAbil` exempts `MedivacSpeedBoost` (Afterburners), `GhostHoldFire` and `GhostWeaponsFree`.
- Units without energy are deliberately left out (Stim, Blink, Bile, Tactical Jump).
- New global `g_cycle_castTrig`. It is registered in init next to the damage/attack triggers, as one any-ability (`null`) trigger with three `TriggerAddEventUnitAbility` calls, one per stage.
- Each break is logged: `Forced March: <unit> broke on cast <abil>`.
- `CycleMod_Desc(2)` now reads: "+40% move speed; damage dealt/taken or a spell stops it for 10s".

**Docs**
- `enUS.SC2Data/LocalizedData/GameStrings.txt`: the tooltip now says "or casting a spell".
- `README.md`, `CURRENT_STATE.md` (modifier 2 row) and `DESIGN_DEFENSE.md` (Snare entry) are updated to match.

**Public page** (https://claude.ai/artifact/AJVDVgbsTBoLFeHGNXo6ea)
- Updated and live (version 19). The Forced March card now lists spell casts and the exemptions.

## Main risk: the event may not fire

Blizzard's own code only uses the any-ability event with the generic stage `c_unitAbilStageExecute` (`references/mods/voidmulti.sc2mod/.../VoidMultiLib.galaxy:243`). It is **unverified** that `null` ability plus class-specific stage numbers (1/2/3) fires at all.

- **If no log lines show up:** do **not** switch to `c_unitAbilStageExecute`. That breaks the march when the spell is ordered, and you ruled that out (2026-10-21): the break must land when the spell goes off. Instead, register the trigger once per spell ability, naming the ability (`AbilityCommand(abil, 0)`) at its class's stage. That is the editor's usual "Unit uses ability, stage Cast" setup. Build the ability list from the catalog at game start, using the same class filter. Blizzard's reference scripts have no example of a class-specific stage in either form, so this is also unproven until it's played.
- ~~Toggled might also fire on toggling *off*.~~ Checked 2026-10-21: `Abil.galaxy` has a separate `c_abilBehaviorStageUntoggled` = 0, so switching off is its own stage and doesn't break the march.

## Desk check (2026-10-21, before compile)

- All natives match `natives.galaxy`: `TriggerAddEventUnitAbility(trigger, unitref, abilcmd, int, bool)`, `EventUnitAbility`, `EventUnitAbilityStage`, `AbilityCommandGetAbility`, `CatalogEntryClass`.
- The class ids 11/12/14/15 and stage numbers 1/2/3 match `Abil.galaxy`.
- No reserved-word locals, and no `const bool` inside a compound condition (see Gotchas in CURRENT_STATE.md).
- Ability classes in voidmulti: every spell on the checklist below is `CAbilEffectTarget`/`CAbilEffectInstant`, except Ghost/Banshee cloak and Pulsar Beam (`CAbilBehavior`) and the Queen's creep tumor (`QueenBuild`, `CAbilBuild`). `MedivacSpeedBoost`, `GhostHoldFire` and `GhostWeaponsFree` are `CAbilEffectInstant`, so the exemption list is needed.

## Playtest checklist

Use Testing mode and switch on Forced March.

**Should break** (buff icon gone, log line appears):
- Raven: Auto-Turret, Interference Matrix, Anti-Armor Missile
- Medivac heal
- Ghost cloak / Snipe / EMP
- Sentry Force Field / Guardian Shield
- Queen creep tumor (build stage) / Transfuse
- Oracle Stasis Ward / Revelation
- Viper Abduct

**Should NOT break:**
- Medivac Afterburners, Ghost Hold Fire
- Stalker Blink, Stim

**Timing:** a Raven ordered to drop a turret far away should keep its speed until the turret lands.

**Design watch:** Medivacs auto-heal, so Medivacs escorting injured bio will lose their march. That follows the rule, but check that it feels OK.

## Not done

- **Spawned units' swings:** a spawned unit starting an attack still doesn't break its spawner's march. Only its landed damage does. With the turret drop now breaking the Raven, this matters much less.
- **Tools:** near the end of the session, the Bash tool (exit 107) and Grep/ripgrep started failing for no clear reason, so the last edits were made with Read/Edit only. Part of the reason: `references/` is gitignored, so Grep (ripgrep) skips it and returns "No matches". Search it with PowerShell `Select-String` instead.
