# Tardigrade — Parked Ideas

Ideas that were discussed and liked but deliberately not scheduled. Nothing here
is implemented. Move an entry into CURRENT_STATE.md when it gets built.

## Doctrine bucket (trade-off modifiers)

Inspired by Mechabellum's most common unit techs, which are trade-offs rather
than straight buffs (Launcher Overload, Armor-Piercing, High-Explosive Ammo).
Intended as a fifth bucket for the pick-one-per-bucket mode. The appeal for SC2
is that flat armor, upgrades and splash make each one good for some units and
bad for others, so picking is a real decision rather than "bigger numbers".

| Card | Effect | Notes |
|---|---|---|
| Rapid Fire | +40% attack speed, -1 range | Melee units get the speed only |
| Heavy Rounds | +60% damage per hit, -30% attack speed | DPS roughly even; flat armor makes it strong vs armored, weak vs small fast units |
| High-Explosive | Attacks gain a small splash (0.5 radius), -30% damage | Great on balled-up mass units, bad on single-target units |
| Long Barrels | +1 range, -15% attack speed | |
| Heavy Plating | +25% life, -10% move speed | Conflicts with Maneuver picks by design |

All five are per-player catalog or behavior edits, the same kind of work as
Factory Lines and Open Skies.

## Other modifier ideas, not picked

- **Pursuit** (Maneuver): +30% move speed while moving toward an enemy below 50% health.
- **Unstoppable** (Maneuver): immune to slows and roots (Concussive Shells, Fungal Growth, Time Warp, ...). Remove a known list of slow behaviors from the scan.
- **Slipstream** (Maneuver): +5% move speed per nearby unit of the same type, up to +30%.
- **Hit and Run** (Maneuver): +30% move speed for 1.5s after attacking. Dropped.
- **Ambush** (Maneuver): units idle and out of combat for 5s cloak until they attack or move.
- **Suppression** (Firepower): attacks slow the target by 20% for 1s.
- **Death Throes** (Firepower): units explode on death for 20% of their max health in a small radius.
- **Conscription** (Command): each production structure spawns a free basic unit every 60s. Needs care around supply and production-structure spam.

## Protect and Ban (dormant, code kept)

Protect and Ban is a unit-draft style where a side puts points on its units, the
opponent bans up to a points budget, and the side keeps what is left. It was
reached from the old drafting-style screen (before the race draft), which chose
it per side and set `g_cycle_p1Protect` / `g_cycle_p2Protect`. That screen is gone
and nothing sets those flags now, so the mode is unreachable. All of it is still
in the tree: `ProtectDraft.galaxy` (the points stage), the Protect branches in
`RosterDraft.galaxy` (`RosterDraft_IsProtect`, `RosterDraft_Start` ->
`ProtectDraft_Start`) and the No Bans +4 points rule. To bring it back, give a
side a way to set its Protect flag (a new setup row or a stage button) before the
unit draft starts.
