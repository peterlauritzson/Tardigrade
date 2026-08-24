# Tardigrade — Early Defense & Anti-Snowball Design

Design record for the layer that sits **outside** the modifier draft: tools that
keep early aggression from being decisive and keep a won fight from compounding
into a won game.

**Status: design only.** Nothing here is implemented yet. See
"Implementation notes" for what the code already gives us for free and what has
to be built.

---

## Which doc to trust

| Doc | Scope | Status |
|---|---|---|
| `CURRENT_STATE.md` | Implementation reference — wiring, constants, data, gotchas. | **Authoritative** for what exists. |
| `DESIGN_DEFENSE.md` (this file) | Design intent for the defensive layer. Not yet built. | Current. **Nothing here is in the code.** |
| `README.md` | Player-facing overview + project layout. | Current. |

---

## Premise

The 10 drafted modifiers are **working as intended, including the overpowered
ones**. Location-agnostic Entrenchment, uncatchable Overwatch + Medivac Boost
kiting, Veteran Forces stacking off kills — none of these are bugs to tune down.
They are the toys, and they should feel like toys.

The problem they create is not that any single modifier is too strong. It is
that a player can be **structurally unable to respond**: rushed before they have
any army at all, or buried by a snowball they had no lever against. The fix is
an additive layer that raises the floor, never one that lowers the ceiling.

Three goals, in priority order:

1. **Early aggression shouldn't be decisive.** Being rushed at 2:00 with nothing
   on the field is not a loss you can play your way out of.
2. **Anti-snowball.** Winning the first engagement shouldn't compound into
   winning every subsequent one.
3. **More active play with fewer units.** Reward skirmishing over deathballing.
   Lowest priority; treated as a bonus when a mechanic happens to serve it.

---

## The design contract

Every idea below was checked against these. They were derived the hard way — by
proposing things that violated them and watching them break. **Check new ideas
against this list before proposing them.**

| # | Rule | Why |
|---|---|---|
| 1 | **Never nerf a drafted modifier.** | They're WAI. The layer is additive only. |
| 2 | **No relative-state comparisons** ("if behind in army value…"). | Feels like a handout, and needs bookkeeping that doesn't match the mod's flat trigger→effect shape. |
| 3 | **No anchor that can be forward-deployed.** | Creep, Pylons, Sensor Towers, and Town Halls are all cheap enough to proxy — a "near your X" buff becomes "how strong is my cheese." Zerg's drone-morph Hatchery makes even Town Halls proxyable. |
| 4 | **Nothing tied to starting position.** | Punishes expanding and dies the moment you leave home. |
| 5 | **No assumptions about what got drafted.** | Zerg may not draft Zerglings; Terran may go mech and never build a Barracks. Any rule naming a specific unit or production building will be a silent no-op for some roster. |
| 6 | **Nothing outside the draft may gain combat power.** | Buff workers and people stop drafting melee units and fight with workers instead. Any free combat-capable asset competes with the roster draft and devalues it. |
| 7 | **No side effects that amplify aggression.** | A low-HP speed boost "helps you survive" — and also lets every harasser escape every punish. Check what a mechanic does for the *attacker* before shipping it. |
| 8 | **No one-off conditions** ("the first town hall you lose grants…"). | Unpredictable, ungeneralizable, and impossible to reason about mid-game. |

### The anti-snowball trick

Rule 2 appears to make anti-snowball impossible — how do you help the losing
player without measuring who's losing? The answer, and the shape every
anti-snowball entry should follow:

> **A rule that is uniform in statement but pays out in proportion to losses is
> automatically self-targeting.**

Both players have a 50% death refund. Only the one bleeding units collects on
it. No comparison is ever made. If a proposed anti-snowball mechanic doesn't
have this shape, it isn't anti-snowball.

A corollary that turned out to matter: **anything free and cooldown-gated is
inherently anti-snowball**, because it's the one component of a player's
strength that does not scale with income. A player on one base and a player on
four receive it at identical rates.

### Making a tool defensive without a condition

Every attempt to express "helps defenders, not attackers" as a *condition* got
exploited (rules 3, 4, 7). The way through is to build the asymmetry into the
**object** instead:

| Property | Why it's defensive with no rule attached |
|---|---|
| **Temporary** | Defense is reactive, immediate and local — you need force *now*, for a minute, where you're being hit. Offense is accumulative: you gather, walk, commit. A 60-second unit is ideal for the first and useless for the second. |
| **Slow** | Can't chase, can't keep pace with a push. Fine standing in your base. |
| **Weak vs structures** | A rush's actual goal is killing production and workers. Something that shreds units but barely scratches buildings can hold a base but never take one. |
| **Telegraphed arrival** | A summon that lands 4–5s after the cast can't be a surprise alpha strike, but defends a base you can see being attacked perfectly well — because defense comes with warning by definition and offense depends on denying it. |

---

## Accepted

### 1. Tactical bar (primary deliverable)

Co-op-style global abilities, available to both players, live from 0:00.

**Charge economy** — one shared pool:

| Setting | Value | Rationale |
|---|---|---|
| Max charges | 2 | Enough to react twice; not enough to bank an army. |
| Accrual | 1 per ~75s | Clock-driven only. **Never** scales with income (see anti-snowball corollary). |
| Starting charges | 1 | Live at 0:00. |
| Resource cost | None | Costed would scale with economy and reintroduce a snowball vector. |

Shared (not per-ability) charges are deliberate: independent cooldowns mean
everyone pops everything in every fight and there is no decision. A shared pool
makes "turret now, or bank for mercs?" a real choice and caps total output with
one number.

**Starting with one charge banked replaces the "start with some drafted units"
idea**, and is strictly better: you spend it only if actually threatened, on the
tool that matches the threat you can see, rather than one guessed at during the
draft. In the majority of games where nobody rushes, it carries forward instead
of idling in your base.

It likewise **replaces the "spawn with a free static defense structure" idea** —
the turret below does the same job but gets placed where the attack actually
lands.

#### Ability: Summon Mercs — Infested Terrans

Chosen because several required properties are **native to the vanilla unit**,
so they don't have to be authored:

- Already has a **timed lifespan** — the temporary property the whole design
  rests on, for free.
- Already **spawns from an egg** — the arrival telegraph, for free.
- **Shoots air** — quietly patches "they drafted air, my roster has no AA yet,"
  which is one of the worst blowouts the roster draft can produce.
- **Thematically neutral** (feral infestation, not any player's race), which
  answers why a Protoss player summons the same thing a Terran player does.

| Setting | Value |
|---|---|
| Count | 3, **fixed permanently** |
| Arrival | Egg, ~4s |
| Lifespan | Vanilla Infested Terran timed life |

**Count is never a balance lever.** A blob of weak bodies is degenerate at 0:00
(surrounds, worker snipes) and worthless by 10:00. If the ability needs to stay
relevant later, scale its *stats* on the game clock — never its size. Note that
fading in relevance as the game goes long is correct behaviour for an early-game
safety net, not a defect to fix.

#### Ability: Summon Tower — nydus-flavoured

Immobile is the purest form of the "defensive by construction" principle: it
can't chase, can't join a push, can't follow a retreating army, and a timed life
means it can't be accumulated into a wall.

| Setting | Value |
|---|---|
| Lifespan | 60–90s |
| Targets | **Ground and air** |
| Damage vs structures | Near zero |
| Repairable | No |

Air capability is not optional, for the same reason as the mercs.

> **Unverified:** the exact unit id for the nydus-flavoured tower has **not**
> been confirmed against the extracted Blizzard data in `references/`. Confirm
> before implementing — see the `SwarmHostMP` vs `SwarmHost` bug in
> `CURRENT_STATE.md`, where a wrong id made `TechTreeUnitAllow` a silent no-op.

#### Cast restriction: closer to you than to them

Both summons may only be cast where the target point is **closer to the nearest
friendly start location than to the nearest enemy start location.**

```
myDist    = min over p in own team    of distance(castPoint, g_start_loc[p])
enemyDist = min over p in enemy team  of distance(castPoint, g_start_loc[p])
allowed   = myDist < enemyDist
```

Team games (XvX) are handled by the nearest-start minimum on both sides.

Why this beats a fixed radius around the enemy start:

- **No magic number to tune.**
- **Adapts automatically** to map size and to close-spawn vs cross-spawn
  positions on 4-player maps.
- **Cannot be manipulated** by either player — both anchors are fixed at match
  start.
- Gives you your whole half of the map, which is where you'd plausibly be
  defending, while making offensive use simply unavailable rather than merely
  unwise.

**Trap to avoid:** do *not* anchor this to enemy **structures** ("can't cast
near an enemy building"). That lets an opponent plant a cheap proxy near your
base to switch off your defensive bar entirely — handing the aggressor the exact
tool this is meant to deny them.

#### Decided: summons receive drafted modifiers

Summoned units **do** get their owner's drafted modifier buffs. This requires no
work — `CycleMod_ScanTrigger` walks all map units and applies each side's picks
to that side's own units, so summons are covered automatically.

#### Later: Snare

Brief root or slow in an area. Deferred until mercs and tower have had a
playtest. Worth building eventually because it is the only entry that answers
the "I can never catch them" problem (Overwatch + Medivac Boost on a long-range
unit), which is where this whole design thread started.

### 2. Standalone keepers

Independent of the bar, in rough priority order:

| Idea | Mechanic | Why it survives the contract |
|---|---|---|
| **Death refund** | X% of cost returned when your units/structures die | The archetype self-targeting anti-snowball rule. |
| **Build & tech alerts** | Notified when the opponent starts their first military production, and when a structure goes up far from their own bases | Zero combat power, so it can't feed a rush or a snowball. The historical counter to a proxy was always *finding it in time*. Consistent with the mod already showing both rosters and both modifier sets. |
| **High-ground bonus range** | Units on high ground get +range | Terrain is the only anchor that can't be built, bought or proxied. Mains are elevated, so it favours defenders by map topology. Range specifically, because range decides whether a defender can hit an army massing at the bottom of their ramp before it commits. |

---

## Rejected

Recorded so they don't get re-proposed. Several of these are *good ideas that
fail for non-obvious reasons.*

| Idea | Fails on | Detail |
|---|---|---|
| Buff workers (tougher/stronger) | Rule 6 | People stop drafting melee units and fight with workers, which are unbannable and undraftable. |
| Low-HP speed boost | Rule 7 | Buffs every harasser's escape exactly when they need it. |
| Buffs near creep / powerfield / Sensor Tower | Rule 3 | All cheaply forward-deployable. Fixing it needs build restrictions (Pylons included), which is more machinery than the buff is worth. |
| Town Hall aura / overcharge on all town halls | Rule 3 | Proxy Hatcheries. Only survives if granted *at spawn* rather than evaluated during play. |
| Buff static defense (Bunker/Cannon/Spine) | Rule 7 | Those are the rush tools themselves. |
| Speed up first military production building | Rule 5 | Mech Terran never builds a Barracks; Zerg without Zerglings gets nothing from a faster Pool. |
| Universal out-of-combat regen **on units** | Rules 2, 7 | Fails the self-targeting test: after a fight the winner has ten survivors and the loser two, both heal, ratio unchanged — and the winner re-pushes at full health. Amplifies harass. *Structures only* would be acceptable, since buildings can't chase. |
| Repair field on the tactical bar | — | Rejected by design call. |
| Free starting combat units (not from draft) | Rule 6 / general | A free combat unit at 0:00 is what *enables* a rush. |
| Starting block drawn from the drafted roster | — | Passes the contract by construction, but whatever selection rule is used becomes a drafting incentive: a supply budget rewards cheap units (16 Zerglings at 0:00 ends games), a fixed count rewards expensive ones, one-of-each front-loads tier 3 before any tech exists. Also creates unanswerable air/cloak openings. Superseded by the banked opening charge. |
| Capturable merc camps on the map | Rule 2 (effectively) | Contested neutral objectives go to whoever is already stronger — a snowball engine. Also needs map edits, which ladder maps don't allow. |
| Recall / teleport-home calldown | Rule 7 | Makes harassment consequence-free. |
| Direct-damage calldowns (nuke / airstrike / orbital strike) | Rule 7 | A large slice of co-op's top bar, and exactly the wrong shape: free damage helps a rusher kill production. |
| Lower max supply + richer patches | — | Genuinely strong anti-snowball (economic advantage saturates, and it serves goal 3), but rewrites what macro play *is*. Parked as too invasive, not as wrong. |
| Enemy-base build exclusion radius | — | Would work, but deletes cannon/proxy rushing as a strategy outright. A design statement, not a tuning knob. |
| Co-op-style commander levelling / mastery scaling | Rule 2 | Any "gets stronger based on what you've done" is a snowball vector. If the bar escalates at all it must be on the game clock only, like the existing 3:00 gate. |

---

## Implementation notes

Grounded in what the code already does. **None of this is built yet.**

**Already available, no work needed:**

- `g_start_loc[16]` (`RaceDraft.galaxy`) is populated per player in
  `SetPlayerRace` at Race Draft finish — well before game start. The cast
  restriction needs no new bookkeeping.
- `g_draft_team1` / `g_draft_team2` playergroups give the team membership the
  XvX nearest-start minimum needs.
- `CycleMod_ScanTrigger` already applies each side's modifiers to all of that
  side's units, so summons inherit drafted buffs for free.

**Has to be built:**

- Two abilities (`AbilData.xml` / `EffectData.xml`) plus the summoned units.
- Charge accrual + per-player charge state.
- The bar UI. **Per-player dialogs are mandatory, not stylistic** —
  `DialogSetSize` and `DialogSetImageVisible` take no `playergroup` argument, so
  a single shared dialog cannot be updated privately. Copy the
  `RosterHUD_InitForPlayer` pattern in `RosterDraft.galaxy`.
- The cast validator. A real `CValidator` greys the button out (better UX); a
  galaxy-side check on the ability-used event is simpler and matches the
  trigger-heavy style of the codebase but lets the cast start before failing.
  `ValidatorData.xml` is currently minimal/unused.

**Watch out for:**

- Screen space: the roster HUD (top-left) and the modifier panel (top-center,
  214px, already flagged as possibly intrusive in `CURRENT_STATE.md`) are
  already competing. Folding the bar into the existing modifier panel is worth
  considering over adding a third element.
- A top bar **avoids** the command-card problem entirely — no `CardLayouts`
  slot, no Row 3 failing to render, no collisions with vanilla buttons on 52
  units. This is a real advantage over the per-unit ability approach used by
  Medivac Boost and Battle Blink.
- The bar is deliberately live from 0:00 while modifiers stay dormant until
  3:00. This gives the early game its own identity (vanilla + a small tactical
  toolkit) and keeps the two systems from interacting in the window where either
  could break the other.
- Ship mercs and tower **first**, playtest, then consider the snare. This is a
  new subsystem with its own balance surface sitting on top of ten deliberately
  overpowered modifiers.
- Per `CURRENT_STATE.md`: **verify everything by recompiling in the SC2 editor.**
  The VS Code linter passes on errors only the editor's compiler catches.

---

## Open questions

- Exact unit id for the nydus-flavoured tower (verify in `references/`).
- Whether both players should see each other's charge counts. Consistent with
  the mod already showing both rosters and both modifier sets, and it adds real
  strategy — knowing an opponent is sitting on two banked charges says their
  base is defensible right now, while catching them at zero is a genuine window.
- Merc stat profile: whether the vanilla Infested Terran is usable as-is or
  needs a Tardigrade variant.
- Whether the three standalone keepers ship alongside the bar or after it. They
  compose *multiplicatively* in the defender's favour — bar plus terrain plus
  early warning could make sub-five-minute aggression not merely non-decisive
  but non-viable, which overshoots the premise.
