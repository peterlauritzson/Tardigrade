# UX pass — handover (2026-10-06)

Goal: the game "feels clunky to play". This session audited the draft screens
and the in-game experience, then started a batch of low-risk fixes.
**Nothing here has been compiled or played.** All changes are uncommitted in
the working tree (`git diff`).

## Diagnosis (why it feels clunky)

In-game:
1. **Modifiers fire invisibly.** No text tags, sounds or actors anywhere
   (`ActorData.xml` is empty). Every modifier buff was `Hidden`. So Battle
   Blink, Refund, Overwatch and Veteran feel random, or as if nothing happened.
2. **Dead first ~2 minutes.** Picks stay dormant until 2:09, with full map
   vision for everyone. The switch-over was announced only by one chat line.
3. **The modifier panel covered the top-centre** (470×232, expanded) for the
   whole game.
4. **Roster limits are silent.** Undrafted units are just greyed out by
   `TechTreeUnitAllow`. Nothing says "not drafted", and the roster HUD starts
   collapsed.
5. **Possible late-game hitching.** `CycleMod_ScanTrigger` walks every unit
   on the map every 0.5s and does ~6 modifiers' worth of work per unit. A
   global UnitDamaged trigger runs on every hit.

Draft:
1. **~31 strictly sequential turns** (2 race, 10 modifier, 16 unit, plus the
   rest), and the waiting player sits idle on a disabled board.
2. **Text-only cards.** Modifier cards squeeze a 50–70 character description
   into 280px. Unit cards show a name plus a two-word role, with no cost,
   tooltip or icon.
3. **No audio**, and no "your turn" cue.
4. Every screen is a different dialog size. The style screen opens with a
   ~100-word wall of text. The summary waits 30s unless both players find
   READY.

## Done this session (uncommitted, unverified)

In-game, by me (`CycleMod.galaxy`, `BehaviorData.xml`, `GameStrings.txt`):
- **Floating text:** `CycleMod_FloatText(point, msg, hex, players)`, placed
  just above `CycleMod_RefundCanVanish`. It is fogged, rises and fades in
  1.6s. Wired in for:
  - Battle Blink: "Blink!" where the unit left from; everyone sees it.
  - Refund: "+12" or "+m / +g" at the dead unit; owner only.
  - Overwatch: "Overwatch!" when the window opens; everyone sees it.
  - Veteran: "Veteran N" on a kill; owner only.
- **Activation:**
  - A heads-up about 10 real seconds before activation
    (`c_cycleActivationWarnLead` = 14 game seconds, `CycleMod_WarnActivation`):
    a subtitle plus a beep.
  - At activation (`CycleMod_ActivateDelayedMods`): a "MODIFIERS ACTIVE"
    subtitle, a sound, and every folded panel re-opens for 10s.
- **Modifier panel auto-fold:** it starts open and folds itself after about
  20 real seconds.
  - New globals: `g_cycle_panelFoldAt[16]` and `g_cycle_activationWarned`.
  - New functions: `CycleMod_PanelPeekAll` and `CycleMod_PanelFoldTick`,
    with prototypes above `CycleMod_ActivateDelayedMods`. The scan loop calls
    the fold tick.
  - Clicking the toggle cancels the timer for that viewer.
- **Visible buffs:** Forced March, Entrenched, Overwatch Ready, Adrenaline
  Rush and Battle Blink cooldown now show an icon in the unit panel. Hidden
  was replaced with an `InfoIcon` using stock melee icons. Names and tooltips
  were added at the end of `GameStrings.txt`.

Draft, by a helper agent (finished; not yet reviewed line by line):
- **Sound helpers**, appended to the end of `DraftStyle.galaxy`:
  - `DraftStyle_TurnCue`
  - `DraftStyle_TurnCuePlayer`, which only plays for human players
  - `DraftStyle_ClickCue`
  - `DraftStyle_SetTip` and `DraftStyle_UnitTip`
- **Turn beep** for the player who is up, played once per step from:
  - `GameMode_Select`
  - `CycleMod_YoloChoiceStart`
  - `RaceDraft_UpdateUI`
  - `CycleMod_UpdateDraftUI`
  - `RosterDraft_BeginStep`
  - `ProtectDraft_BeginStep`
- **Click sound** on accepted clicks in every draft screen, and on
  Tournament's `DraftSelect_Pick`.
- **Hover tooltips:**
  - Modifier cards show the name plus `CycleMod_Desc`.
  - Race and mode/style cards show their detail line.
  - Unit cards show the name, role and "Cost: min / gas". They are set on
    every update, because the buttons are reused across pools.
  - Not on the picks-panel labels or the Done/-1 buttons.
- **Summary screen** (`TardigradeLogic.galaxy`):
  - Beeps on each of the last 3 seconds. `lastCue` re-arms them after a
    reroll.
  - The `UI_Bnet_MatchCountdownGo` sound at GO!.
  - `Tardigrade_SummaryReadyHint`: "Click READY to start now" on the status
    line until that player readies.
- All three sound ids exist in the liberty reference `SoundData.xml`.

## Next steps

1. Run `git diff` (9 files, about 230 lines added) and review the helper's
   draft edits.
2. **Compile in the SC2 Editor.** Risk points:
   - The prototype declarations.
   - `SoundLink(id, -1)` with `c_maxPlayers` as the owner.
   - Text markup inside `TextTagCreate`.
   - `g_cycle_panelDlg[key]` being 0 for unbuilt keys. `PanelPeekAll` skips
     both 0 and `c_invalidDialogId`.
3. Playtest in Testing mode and toggle each modifier: blink text, refund text,
   buff icons, panel fold, and the activation alert (Casual mode, at 2:09).
4. Update `CURRENT_STATE.md` / `README.md` to match, then commit.

## Proposed but not started (needs your call)

- **Shorter, partly simultaneous draft:** blind simultaneous modifier picks
  per round, and/or units 1-1 instead of opening + snake. This is the biggest
  lever on how clunky the draft feels.
- **Live modifiers earlier,** or drop the 2:09 dormancy and the opening
  vision.
- **Message when building an undrafted unit**, and start the roster HUD open
  with the same auto-fold.
- **Performance:** run the scan only over units that changed (register on
  create, cache per unit), and spread it across ticks.
- Battle Blink visual/sound actor (needs the editor).
- Unify the draft dialog sizes, trim the style-screen text, and shorten the
  summary to 15–20s.
