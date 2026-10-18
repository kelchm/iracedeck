# Replay Markers: a per-key delete window, a greyed Delete key, and Add / Delete

> **Issue:** [#1454](https://github.com/niklam/iracedeck/issues/1454) · **Supersedes:** _none_ · **Superseded by:** _none_
>
> Point-in-time design record. The code and `.claude/rules/` are the truth; this is not documentation.

Delete Marker reaches the nearest marker within a fixed 10 s (`MARKER_DELETE_WINDOW_FRAMES`, 600, in `@iracedeck/replay-store`), measured to the marker's frame or to its `pressFrame`, the frame where Add was pressed for it (`pickMarkerToDelete`, `replay-markers-ops.ts`; the #1162 spec). The key looks the same whether a press would delete something or nothing. This spec makes the window a per-key setting, greys the Delete key when nothing is in reach, and adds a fifth mode that is Add or Delete depending on the same window. The three maintainer decisions it rests on: the grey is always on, the window is per key with a default of 10 s, and the new mode switches its icon and function live.

## The decision

1. **A Delete Window setting, 1–60 whole seconds, default 10**, on the keypad (`deleteWindow`) and on the dial (`dial.deleteWindow`). It replaces the constant in the one place the window binds, the action's pick. It applies to both distances, as today.
2. **The Delete key greys out whenever a press would delete nothing**, through the same path that greys Next and Previous.
3. **A new mode, Add / Delete Marker (`add-delete`)**, that shows and does Delete while a marker is within its window and Add otherwise, acting on the look it shows.
4. **The dial's Delete gestures and preview read `dial.deleteWindow`.** The dial gains no Add / Delete gesture.

## The window

- **Range 1–60 whole seconds.** Whole seconds match Seconds Back (0–60) and the PI's number field; frames or tenths would be precision nobody can aim a press at. The upper bound matches Seconds Back's: a window wider than a minute stops being a guard against a mistaken press, which is what it is for, and the `pressFrame` distance already reaches a marker set up to 60 s back. The lower bound is 1, not 0: a 0 window would reach only the exact frame, which in a playing replay or from the car is never hit, so the key would be permanently grey — a disabled key, which is what removing it from the deck is for.
- **Both distances, as today.** The `pressFrame` distance is what lets Delete from the car reach a marker set with a long Seconds Back, the car sitting at the live edge and the marker well behind it (#1162, amended after its review). Measuring the per-key window to the marker frame only would make a 15 s Add key's marker undeletable from the car with any window under 15 s. One window, both distances, nearest wins, ties to the earlier marker: unchanged.
- **Parsing.** One field fragment, shared by `deleteWindow` and `dial.deleteWindow` like `secondsBackField`: blank (or whitespace or null) reads as unset, so the default applies; a number is rounded and clamped into 1–60, so a typed 0 becomes 1; anything unreadable reads as the default; and `.catch` returns the default, so one bad value never fails the schema and resets the key's mode.
- **The default lives once.** `DELETE_WINDOW_DEFAULT` is derived from `MARKER_DELETE_WINDOW_FRAMES / 60`, so the 10 s exists in one place. The store's constant stays: its `deleteNearest` is only ever called at the chosen marker's own frame (`deleteMarkerAt`), where the window never binds.
- **No silent default in the ops.** `pickMarkerToDelete`, `previewDeleteMarker` and `deleteMarkerAt` take the window (seconds, converted to frames inside) as a required argument, so a caller cannot keep 10 s by omission.

## The greyed Delete key

- **The predicate** is the press's own, read fresh: a replay context (connected, a store, a frame), and `previewDeleteMarker(context, window)` not null. It is the same function the press calls, so the look cannot promise a delete the press will not make, the rule that already binds Next / Previous to `resolveJumpTarget`.
- **From the car and in a replay alike.** Delete works in both (only the jumps need the replay), so unlike Next / Previous it is not greyed merely for being in the car. From the car it lights for the window's length after an Add (the `pressFrame` distance from the advancing live edge) and then greys. In a playing replay it lights the window's length before a marker and greys the window's length after it, or after the frame Add was pressed at, whichever is later. Without iRacing, or with no replay position, it greys, exactly when Next / Previous do.
- **The cost is the existing path, not a new one.** Every keypad context already subscribes to SDK ticks and runs `refreshAvailability`, which today returns at once for Add and Delete. Delete joins Next / Previous in it: `isJumpMode` becomes a set of modes with a resting look that depends on state, `shownAvailable` records what the key shows, and a redraw happens only on a flip, through the shared `IconUpdateThrottle` (at most 10 Hz). The per-tick work is one context read and a scan of the session's markers, the same order as the Next / Previous walk.
- **The look** is `unavailableLook` (`shared/unavailable-look.ts`), overrides kept and faded, the same as Next / Previous and Replay Control's Jump to Race Start.
- **A press on a grey key** reads fresh and does what it does today: nothing, with no flash. The grey is the display of the press's predicate, never a gate in front of it; a key one tick stale in either direction does what the fresh read says.
- **After the Deleted flash** the resting look is recomputed, so the key goes grey unless another marker is in reach.

## Add / Delete Marker

- **Name and value.** "Add / Delete Marker" in the Mode dropdown, after Delete Marker; stored as `add-delete`, hyphenated like the repo's other multi-word mode values.
- **The two looks** are the Add key's and the Delete key's: the same artwork (`add.svg`, `delete.svg`) and the same default titles (`MARKER` / `ADD`, `MARKER` / `DELETE`), with the key's own colour and title overrides on both. The key always says what a press will do, in the words the single-purpose keys use; a third artwork for "it depends" would hide exactly that. No new icon is needed. A user's own title text applies to both looks, which is their choice; the artwork still differs.
- **Which look** is the greyed Delete key's predicate: Delete when `previewDeleteMarker(context, deleteWindow)` names a marker, Add otherwise, re-evaluated per tick on the same path, redrawn only on a change.
- **Never grey.** When connected, one of its two operations is always possible. Without iRacing it shows the Add look, plain, as the Add Marker key does; greying the Add key is not in this issue.
- **A press does what the key shows.** The action records the look it last put on the key (`add` or `delete`) and the press runs that operation. If the key shows Delete and the fresh pick finds nothing (the marker left the window inside the throttle's 100 ms), the press does nothing and the next tick redraws Add; it never falls through to Add. If it shows Add, it adds, even if a marker came into reach in that same instant. Rejected: acting on a fresh read, because the look can lag the replay by a throttle window and a key that showed Add but deleted is the precise mistake the request exists to prevent.
- **During the confirmation flash** the recorded look is still the one shown before the press, so a second press inside the one-second flash repeats the first press's operation. After an Add that second Add lands within 1 s of the first marker and is refused as a duplicate, so a double press never adds and then deletes.
- **Confirmation.** The existing MARKER ADDED / MARKER DELETED flashes, unchanged. A press that changes nothing shows nothing, as on the single keys.
- **Seconds Back** is the key's own `secondsBack` (default 5, 0–60), the field the Add key already uses; one key has one Add, so it needs no second field.
- **Seconds Back and the window: the key flips to Delete right after an Add, and that is wanted.** An Add stores `pressFrame` at the press, so the new marker is at distance 0 and the key shows Delete — an undo — for the window's length after the press, whatever Seconds Back is. Undoing a mistaken Add is the commonest delete (the first case the #1162 issue named), and the requester's three-key replay profile has no other key for it. The cost is that a second marker within the window needs a press after the window has passed, or a separate Add key; the per-key window is the knob for that (a driver who marks two incidents seconds apart can set 2–3 s). Rejected: excluding the marker this key just added from its window, which would make the key's look depend on press history the user cannot see, and would remove the undo.
- **An Add refused as a duplicate.** The key shows Add when no marker is within the window of the current frame. With Seconds Back larger than the window, the new marker's frame can land within 1 s of a marker the window does not reach; that press is refused and shows nothing, exactly as the Add key's duplicate press does today. Rejected: widening the Delete look to "the marker an Add would duplicate", which would make the window setting not mean what it says. The corner needs Seconds Back > Delete Window and is documented on the website.
- **PI.** Seconds Back shows for Add and Add / Delete; Delete Window shows for Delete and Add / Delete, through the template's existing `updateVisibility`.

## The dial

The dial gets `dial.deleteWindow`, and every Delete it runs — Press, Long Press, Tap Display, Long Touch — and the long-press `DELETE k / N` preview read it. Consistency is the reason: the dial already keeps its own Seconds Back, the window is the same guard, and a user who tightens it on the keys to avoid a mistaken delete wants the same on the dial, whose Long Press is Delete by default. The cost is one schema field and one PI row; the preview already calls `previewDeleteMarker`. The row sits outside the `dialExtendedGestures` block, since a Mirabox knob's Press can be set to Delete Marker. Rejected: leaving the dial at 10 s, which would make the same word mean two windows on one deck.

The dial gains no Add / Delete gesture: its press and long press already put both operations on one control, which is what the combined key exists to do for keys, and the dash box has no slot to say which a press would run. The box shows nothing new for the window; the hold preview already shows nothing when no marker is in reach.

## Persisted settings and compatibility

Action settings are a persisted contract.

- **An existing key** carries no `deleteWindow`; it parses to 10, so every Delete key and every dial behaves exactly as before. The only change an existing user sees is the grey on a Delete key with nothing in reach, which is the decision.
- **A downgrade** to a build before this change reads `deleteWindow` as an unknown key and ignores it. An `add-delete` key fails that build's mode enum, so its whole parse falls back to full defaults: an Add key with the default look, until the newer build returns. That older build cannot be changed; it is accepted and noted here.
- **The next mode after this one** is spared that: the mode enum gains `.catch("add")`, so a mode this build does not know resets only the mode, keeping the key's overrides and Seconds Back. `add` is the fallback because it is what the whole-parse fallback already produces.
- **No migration**, and no manifest, UUID or registration change: the mode and both fields live in the action's settings and the shared PI template.

## Interactions with open issues

- **#1437** (busy look on replay keys) edits `restingSvg` and `needsRedraw` in `replay-markers.ts` and decides that only a mode whose own command is a jump or a search draws the busy look. Add and Delete send no replay command, so neither Delete nor Add / Delete draws it. Whichever of the two lands second rebases onto the other; the precedence is the flash, then #1437's busy look (Next / Previous only), then the resting look this spec defines.
- **#1429** (a long seek read as leaving the replay): once the replay grace lapses mid-seek the context reads the live edge, so the Delete and Add / Delete keys can show the car's state until the seek lands, as Next / Previous already do. Fixed by #1429 for every consumer, not here.
- **#1427, #1428** concern jump speed and cancellation, which Add and Delete never touch.

## Alternatives rejected

- **The grey as an opt-in** (the request's shape). Next / Previous grey without an option; a Delete key that looks pressable when a press does nothing is the inconsistency.
- **A separate Add / Delete action.** Same store, same two operations, same PI; a fifth mode is the smaller and clearer change.
- **A global delete window** in the settings window. The window is a property of how a key is used — a tight one on a race deck, a wider one on a replay deck — and Seconds Back set that precedent per key.
- **Moving the window into the store** (`deleteNearest(frame, windowFrames)`). The store's call never binds the window; the decision is the action's pick, and the store stays sim- and UI-neutral.

## Out of scope

- Greying the Add Marker key (or the Add look of Add / Delete) when Add would be refused or iRacing is not running.
- An Add / Delete gesture on the dial, or any new dash-box text for the window.
- A confirmation step for Delete, or an undo of a Delete.
- Changing the 1 s duplicate rule, Seconds Back's range, or Next / Previous's windows.
- The mid-seek state of #1429 and the busy look of #1437.

## Testing

Unit (Vitest, colocated):

- Settings: `deleteWindow` and `dial.deleteWindow` missing (10), blank (10), `"0"` (1), `61` (60), `7.4` (7), non-numeric (10); a bad window leaves `mode` and `secondsBack` intact; `add-delete` parses; an unknown mode parses to `add` with overrides kept.
- `pickMarkerToDelete` with a window argument: the edge (exactly window frames in, one frame out) for the marker-frame and the `pressFrame` distance, nearest wins, tie to the earlier marker; a marker without `pressFrame`.
- Delete key: greyed with no markers, lit within the window, grey again past it, from the car (live frame advancing past `pressFrame` + window) and in a replay; grey when not connected; a flip renders once, an unchanged tick renders nothing; a marker added by another key lights it on the next tick; a grey press flashes nothing; after a Deleted flash the key rests grey.
- Add / Delete: Add look with nothing in reach, Delete look within the window, the switch both ways on ticks; a press acts on the shown look (shown Delete with the marker gone does nothing and never adds; shown Add adds); a press during the Added flash repeats Add and is refused as a duplicate; after the Added flash the key rests on Delete and returns to Add once the window passes; the Seconds Back > window duplicate corner shows Add and flashes nothing.
- Dial: each Delete slot and the `DELETE k / N` preview honour `dial.deleteWindow`; an existing dial without the field behaves at 10 s.
- The PI template still passes `dial-pi-knob-gating.test.ts`.

Manual, on a Stream Deck with a Stream Deck+ dial, in a live session and its replay, and once in a saved replay:

1. A Delete key at the default: grey from the car; press an Add key, the Delete key lights for about 10 s, then greys. Press it while lit: MARKER DELETED, then grey.
2. In the replay, play through a marker: the Delete key lights about 10 s before it and greys about 10 s after. Set Delete Window to 3 and repeat.
3. An Add / Delete key in the replay: away from markers it reads MARKER ADD; press, MARKER ADDED, then it reads MARKER DELETE; press again within the window, MARKER DELETED, then MARKER ADD. Double-press quickly from Add: one marker, not added then deleted.
4. Quit iRacing: the Delete key greys with Next / Previous; Add / Delete shows the plain Add look.
5. Dial with Long Press = Delete and Delete Window 3: the hold preview appears only within 3 s of a marker.
6. On Mirabox, the knob PI shows Delete Window with Press set to Delete Marker; on Ulanzi, the keypad modes behave as on Stream Deck.
