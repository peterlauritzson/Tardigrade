# Tardigrade — Parked Ideas

Ideas that were discussed and liked but deliberately not scheduled. Nothing here
is implemented. Move an entry into CURRENT_STATE.md when it gets built.

## Doctrines: built

Shipped as a separate optional stage after the units (non-exclusive, up to N
picks each, not a fifth bucket); see CURRENT_STATE.md section 3b. Rapid Fire,
Long Barrels and Heavy Plating shipped with changed numbers, Heavy Rounds became
Heavy Hitters, and High-Explosive is parked below.

## Doctrine ideas, not built

Same rule as the built ones: a pro, and a con on the side's own stats.

- **High-Explosive:** attacks gain a small splash, -30% damage. Needs script splash on every hit.
- **Evasive:** 25% chance to dodge a hit, -25% health. Rejected for now: on average no trade-off, only variance.
- **Hardened:** no hit deals more than X, -30% health. Rejected: a flat cap favours big units far more than small ones.
- **Flak Rounds / Ground Pounders:** +30% vs air / -25% vs ground, and the reverse. Only meaningful on units that hit both planes; `CycleMod_UnitOriginalPlanes` could scope it.
- **Elite Training / Conscripts:** +20% stats but builds 25% slower / -20% stats but 20% cheaper. Catalog cost and time edits; interacts with Minerals Only and Factory Lines.
- **Replicate:** enemies you kill come back as your units. Ownership, supply and roster rules.
- **Split on death:** units leave smaller copies behind (Legion TD 2's Hydra). Supply and roster rules.
- **Thorns:** reflect part of melee damage (Legion Banana Bunk). Overlaps Riot Gear.
- **Momentum:** each hit gives attack speed for a few seconds, stacking. Stack timers unclear.
- **Kinetic Charge:** range grows with distance moved (Mechabellum Steel Ball). Needs movement tracking.
- **Stun chance:** hits sometimes stun (Legion Pulverizer). SC2 stun data unproven, very swingy.
- **Tech disable:** disable enemy abilities on hit (Mechabellum EM shot). No SC2 equivalent beyond silence effects.
- **Field Medicine:** +50% healing received, -15% health. Only matters with healers.

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
