# The pressed replay key shows that its press is still being worked on

> **Issue:** [#1437](https://github.com/niklam/iracedeck/issues/1437) · **Supersedes:** _none_ · **Superseded by:** _none_
>
> Point-in-time design record. The code and `.claude/rules/` are the truth; this is not documentation.

A replay key that starts a search or a long jump looks the same while the replay is on its way as it did before the press, and since #1430 a press can also be held behind another jump, or given up, with nothing on the key to say so. This spec gives the pressed key three looks: working, waiting and not done. It reads everything from the state seam of #1430's command gate (`2026-10-12-issue-1430-replay-command-gate.md`) and sends nothing itself. The searches it was asked for are those of Jump to Fastest Lap (#1203) and Jump to Green Flag (#1354).

## The decision

1. **Only the pressed key shows it.** Not the other replay keys on the page. A press that came from a dial shows on that dial's own box (proposed; maintainer's leaning, 2026-10-13: "Maybe it would look neater if only the pressed key has the spinner").
2. **Three looks.** **Working**: a spinner, stepped, while the key's own jump or search is under way. **Waiting**: a still mark while the key's press is held behind something else. **Not done**: a sign for one second when the plugin gave up or ignored the press.
3. **The gate is the only source.** The looks are drawn from the per-key view that #1430's gate hands out. The indicator decides nothing about the replay and sends nothing.
4. **The spinner is redrawn images** through the image path every key already uses, four a second, eight frames to a turn (maintainer, 2026-10-13: "we can just redraw the images. We don't need that many per second to make it good-enough"). No GIF, no new member on any adapter.
5. **Nothing shows for 0.75 s after the press**, so a jump that lands at once never flickers. The look ends on what the gate reports, never on a clock.
6. **One timer for the plugin**, alive only while a key on screen is showing the spinner. With nothing under way nothing is redrawn and no timer exists.

## What this rests on

Durations, measured on the maintainer's sim on 2026-10-12 and 2026-10-13 (one fast machine, a recording of 204563 frames; the full record is in #1430's spec):

- A position jump and iRacing's own searches travel at about 2048 frames a sim update. Jumps of 30000, 60000, 120000 and 195000 frames took 165, 400, 896 and 1495 ms; a session search across 166094 frames took 1.37 s. A jump sent within about 415 ms of another's landing is carried out that much later.
- Extrapolated, and the case to design for: on a six-hour recording a jump from end to end takes about 10 s and a far search about 5 s.
- The fallback search of Jump to Fastest Lap and Jump to Green Flag took 4 to 7 s on that recording and 19.6 s for a car it never found; for six hours its author expects about 15 s and up to about a minute.
- On a paused replay nothing in telemetry shows a search on its way, and a search that found nothing cannot be told from one still travelling until the time its distance could take has passed. On a playing replay the gate knows within about 0.4 s.

The code as it stands on `master`:

- **Nothing redraws a replay key while a jump runs.** Replay Control's keys redraw on a telemetry change in three modes only (Play / Pause, Play Backward, Speed Display); Replay Markers' Next and Previous redraw when a jump becomes available or stops being so. Telemetry callbacks arrive only when `SessionTick` advances, so a spinner cannot ride on them.
- **Nothing reports a jump's end.** `currentReplayCursorOwner()` can be polled; `onReplayCursorTaken` fires on a take and not on a release; a plain search (Next Lap and the like) records nothing. That is what #1430's seam adds.
- **An ignored press is a log line.** `showAlert` is implemented by the Elgato adapter only, so it cannot be the sign.
- **Timed redraws exist already**: the flags overlay every 500 ms on every opted-in key (`FLAG_FLASH_INTERVAL_MS`, `base-action.ts`), Session Info's flash every 250 ms, Telemetry Control's capture title once a second, and the Replay Control dial every 50 ms through a 10-a-second throttle while a move has not read back. Each hand-rolls its timer; there is no shared helper.
- **No image is de-duplicated or rate-limited** in `setKeyImage` or in any adapter. The repo's own ceiling is ten sets a second for one key or dial (`IconUpdateThrottle`, `.claude/rules/encoders-and-touchscreen.md`).
- **Every key image is rasterized to PNG** before it leaves the plugin, on all three hosts (`pngRasterization`), by resvg off the main thread, and cached: `toDeviceImage` keeps the last 512 results keyed by size and SVG text (`rasterizer-service.ts`).

One frame's cost, measured for this spec on 2026-10-13 on the maintainer's development machine, by a script that assembled Replay Control and Replay Markers keys with each candidate mark through `assembleIcon` and rendered them with `@iracedeck/rasterizer`, warm, 120 renders a case:

| Size | Median | Slowest | PNG |
| --- | --- | --- | --- |
| 144 px (Stream Deck, Mirabox, Ulanzi) | 1.1 to 1.4 ms | 2.1 ms | 4 to 7 KB |
| 288 px (above the largest Elgato size in use, 240 px) | 1.7 to 2.3 ms | 2.9 ms | 9 to 13 KB |

The keys had the default border and no glow. #642 measured 2.86 ms at 144 px for a key with a border glow and a two-line title, so a glowing key costs about twice this.

## Where the state comes from

#1430 ships the gate's read-only seam (coordinator, 2026-10-13), and the list below was agreed with its owner by message the same day. The names and the field table are #1430's and are defined in its spec; this spec says what each one draws.

- **An origin token on every send.** The Stream Deck context id of the key or dial that was pressed, passed by the surface and never read by the gate. Whatever the gate sends on a press's behalf keeps that press's token: a held search once it goes out, a takeover's jump, a sequence's playback step. `beginReplayJump` takes the token and its `end` takes a result, so the fastest-lap and green-flag searches, a record jump and a marker jump report through the same seam.
- **A view per token**, with two facts that can both hold: **under way** (this token's position jump has a target standing, its search has not landed, or its replay jump holds the cursor) and **waiting** (this token's command is in the gate's waiting slot). Each carries `since`: the time of the press, kept across a hand-over from waiting to under way.
- **A fact is dated from the first command of its token's unbroken run.** A token that keeps sending while its fact stands (a spin of marker detents, #801's turns that keep retargeting, a held search that goes out) stays in one run, and `since` is that run's first command. The 0.75 s delay depends on this rule: without it a spin would restart the delay at every detent and never show anything, and a spinner already showing would blink off at a hand-over. The rule is #1430's to state, and #801's spec leans on it too.
- **How a fact ends**, in three classes with the reason underneath: **completed**, **superseded** (the user's own newer replay command ended it) and **gave up** (off target, the wait ran out, nothing moved, the search gave up, the command was refused).
- **One notice per change** through the gate's `subscribe`, and a getter for the same view.

The indicator derives nothing from the gate's raw record. "Landed" has one definition, the gate's; a second copy in the drawing code is the fault #1430's own spec found between the jump sequence and the target.

One thing never reaches the gate: a press the action ignores itself because a fastest-lap or green-flag press is already in flight. The action reports that to the indicator directly.

## What each state draws

| The gate says, for this key's token | The key shows |
| --- | --- |
| Under way, and the key's mode is a jump or a search | Working, from 0.75 s after `since` |
| Waiting | Waiting, from 0.75 s after `since` |
| Both | Waiting |
| A fact ended as completed or superseded | Its ordinary icon, at once |
| A fact ended as gave up, for any reason | Not done for one second, at once and whatever the delay, then its ordinary icon |
| The action ignored the press | Not done for one second, unless the key is already showing working |
| Nothing | Its ordinary icon |

- **Working is drawn only for a mode whose own command is a jump or a search**: Next and Previous Session, Lap and Incident, Jump to Beginning, Jump to Live, Jump to Fastest Lap, Jump to Green Flag, and Replay Markers' Next and Previous. A Frame Forward key held down sends a run of one-frame jumps that is under way for as long as the finger is on it, and a speed key's command is never under way at all. Every mode that sends a replay command draws waiting and not done.
- **Waiting wins over working** because it is the one that tells the user something to do. Under the rule #1430 is settling (a press behind a search waits for its landing, and pressing again acts at once), a second press is the way out of a long wait, and the still mark is what says a press is waiting.
- **Stillness and motion mean different things.** Motion: the plugin is moving the replay for this key. A still mark: this press is queued behind something else.
- **One drawing whether or not telemetry shows the search.** The gate's view says whether a search is seen or only possible. The key does not claim to know more than the gate, and the two differ only in when they end.
- **A search with nothing to find on a paused replay** shows working until the time its distance could take has passed, then not done. Deep in a six-hour recording that is up to about 20 s of spinner on, say, Next Incident with no incident ahead. It is the honest drawing of what the gate believes, and a press made meanwhile is held for the same time unless it is pressed again.
- **Superseded shows nothing.** The user's own newer press is the reason, and that key now carries the look.
- **Not done is not delayed.** It is the answer to the press: no next incident, the search found no lap, the jump ended somewhere else.

## Which surface is "the pressed key"

- **A second key pressed while the first is under way.** If its command is held, it shows waiting after its own delay, and the first key goes on showing working until its jump lands; both usually end together. If it takes over (a marker key pressed during a fastest-lap search), the first key returns to its icon at once and the second starts its own delay. If it is ignored (another Jump to Fastest Lap or Jump to Green Flag key while one is in flight), it shows not done; the same key pressed again changes nothing, since it is already showing why.
- **The same mode on two keys.** Only the one that was pressed. On two decks, the same.
- **A key that leaves the screen** (a page turn, a profile switch) stops being drawn. The fact is the gate's and outlives the key: if the key comes back while it still stands, it shows the current look at once, with no new delay.
- **An input that came from a dial** has no key. The dial's box shows it (below). A push or a gesture passes the dial's token, and so does a turn that asks for a jump or a search: a detent of the Replay Markers dial, and #801's stepping. A turn that moves the replay frame by frame or changes its speed (Frame Jog, Speed Control) passes none and draws nothing: a Frame Jog spin is under way for as long as the hand turns, and the box already shows where it is going.
- **A search the key did not press for by name.** A search held behind a marker jump belongs to the key that was pressed for the search, and that key shows waiting and then working. #801's search runner belongs to the dial that was turned for it, and #801 passes that dial's token.
- **A seek nobody recorded** (iRacing's own replay bar) has no token and draws nothing.

## The looks

Three families were drawn through the real pipeline and are with the maintainer as a browser page:

- **A, in place of the artwork (recommended).** The artwork gives way to the mark and the title and border stay, so the key still says what it is. Working is eight dots in a ring with one bright dot and a fading tail; waiting is three dots in a row; not done is a red cross. It is the largest of the three, and a ring of dots reads as stepping on purpose at four frames a second, where a smooth arc looks like it stutters.
- **B, a corner badge.** The key stays as it is and a round badge in the top right corner carries a small arc, three dots or a red cross. Small on a 72-pixel key.
- **C, on the border.** The key stays as it is and a bright stretch runs round its edge; waiting is a dashed edge, not done a red one. It competes with the user's own border colour and glow.

Whichever is chosen:

- **The marks are drawn in code**, in `@iracedeck/icon-composer` beside the binding-missing glyph, from circles, lines and opacity only. They need no filter, so they survive the `pngRasterization` kill-switch, and they add no icon file, preview or `icon-defaults.json` entry.
- **They take the key's own colours**: the mark in `graphic1Color`, falling back to `textColor`, on the user's background. The red of not done is the fixed semantic red (`#e74c3c`), as on every other key.
- **A key with its title hidden** shows the mark alone under A. It is the key that was just pressed, for a few seconds.
- **`assembleIcon` takes the mark as an option**, as it takes `bindingMissing` and `dimmed`. Replay Control's two hand-assembled icons (Speed Display, Set Speed) call the same mark function.

## Frame rate and cost

**Four frames a second** (`REPLAY_BUSY_FRAME_MS`, 250), eight frames to a turn, so one turn takes two seconds. It is the cadence Session Info's flash already runs at, and under half the repo's ceiling for one key.

- **Frames are drawn when shown, and the rasterizer's cache is the frame cache.** A frame is the key's own SVG with the mark at step _n_, so the same key at the same step is the same text and a cache hit. The first turn costs eight renders, 10 to 25 ms in all by size and glow, off the main thread. Every turn after that costs none. No frame set is precomputed and no second cache is built, so a colour or title change needs no invalidation: it is simply different text.
- **Host messages**: four image sets a second for a key that is spinning, each a PNG of 4 to 7 KB at 144 px, so 20 to 40 KB a second as base64 over the local socket. Waiting and not done cost one image set each.
- **Several keys at once** is two or three by construction. The gate has one waiting slot (one still key), and one jump or search under way with at most a hand-over beside it. Three keys spinning would be twelve image sets a second for the whole plugin. For scale, the flags overlay redraws every opted-in key on every page twice a second for as long as a flag flies, and #642 budgeted 32 keys each changing ten times a second as about one core.
- **The 512-entry cache** takes eight entries for a spinning key's look. They are the newest entries while it spins and age out like any other.
- **Per host.** Stream Deck: `setImage` with a PNG at 144 to 240 px by device. Mirabox: `setImage` at 144 px. Ulanzi: one `state` frame, type 1, at 144 px. None has a documented rate limit, and four a second on one key is inside what the flags overlay and Session Info already send on all three.

## When it starts and when it ends

- **`REPLAY_BUSY_SHOW_AFTER_MS` is 750**, counted from the fact's `since`, which is the press. It is a display delay and nothing correct rests on it.
- **Measured against** the jumps above: on the maintainer's machine nothing shows for a jump shorter than about 100000 frames (some 28 minutes of recording), for a search that finds something within that distance, for a press held behind any of those, or for the 415 ms a jump is carried out late. What shows: the fallback searches from 0.75 s on, a jump across a long recording, a far search, a search with nothing to find on a paused replay, and a press held behind any of these. A slower machine shows it for shorter jumps, which is right: there the wait is real.
- **A jump that takes between 0.75 s and 1 s shows one frame.** No minimum showing time is added (see _Rejected_).
- **It ends on the gate's notice**, and at the latest one frame after the gate's view stops listing the fact (below).

## The timer, and why it stops

A new module, `shared/replay-busy-indicator.ts`, owns everything timed:

- **Surfaces register a token with a redraw callback** when the key appears and remove it when it disappears. The indicator answers one pure question, "what does this token show now" (rest, working at step _n_, waiting, not done), and calls the redraw when the answer changes. The action's own icon function stays the only place a key is drawn; it takes the answer as an input.
- **One interval for the plugin**, started when the first registered token begins to show working and cleared when none does. One step counter, so keys spinning together are in step.
- **A one-shot timer per token** for the 0.75 s, and one for the second of not done.
- **Every tick reads the gate's view again** for each spinning token. A token with nothing under way stops there. So a notice that never arrives costs one frame, not a spinner that never ends: the tick is the reader, and a notice only makes it prompt.
- **Each way it stops:** the fact ends (landed, superseded, gave up); the key disappears; the sim disconnects, which ends every fact at once; the replay is left, which ends a search, a waiting press and a replay jump with no target as soon as the debounced replay state says so; and, as the last line, a spinner that has been showing for five minutes stops with a warning (`REPLAY_BUSY_MAX_SHOW_MS`). That ceiling sends nothing and decides nothing about the replay. Reaching it means the gate reported a fact that never ended, which is a fault to find, or a search on a recording far longer than the six hours this is designed for, where the key goes back to its icon while the search runs on.
- **A position jump's target outlives the replay being left**, by at most its jump's own wait in the gate: the larger of 3 s and twice the jump's length at 2048 frames a sim update, about 20 s across a six-hour recording (added 2026-10-13, from the gate's owner). A seek and the garage read the same in telemetry, so the gate cannot end it sooner without dropping a press held behind a real ten-second jump. A key whose jump was under way when the user left the replay therefore goes on showing working for up to that long and then shows not done.
- **The timers are unreferenced**, so none holds the process open.
- **A redraw that fails** (the host went away) is caught and logged at debug, and the next tick tries again or finds the key gone.
- **While the flags overlay is flashing a key**, `setKeyImage` stores the image and does not send it, as for every other redraw. The flag wins.
- **Logging**: `Replay busy look started` and `Replay busy look ended` at debug with the token and the reason, a warning for the ceiling, nothing at info. The press and its outcome are the gate's and the action's lines.

## Dials

The Replay Control and Replay Markers dial boxes show the same three states, for a push, a gesture, a marker detent and #801's runner:

- **At the left end of the box's caption line**, at the caption's own height: three dots with the bright one stepping across for working, three even still dots for waiting, a small red cross for not done. The label and the value stay where they are. This is the box's own mark and is the same whichever of the key looks is chosen; a ring of dots does not fit a caption line.
- **The caption's text keeps its place beside the mark** when it is #801's pending count (`+3`, `-2`): the count is progress, which a mark cannot show, so the mark never replaces it (amended 2026-10-13, with #801's spec). The play state, which is all the caption says otherwise, gives way to the mark as it gives way to the count. `DialBoxArgs` gains the mark as an optional argument, drawn by both the strip and the knob renderer; a surface that passes none draws exactly as before.
- **Two dials on one run** (#801): the run keeps the token of the dial whose turn started it, so that dial shows the mark. A second dial that adds to the same count shows the count and no mark. A turn that replaces the count for another target starts a run with its own token, and the first ends superseded and shows nothing more.
- **Car mode passes no token and draws nothing**, with the car and camera keys (#1435).
- **Through the surface's existing render path**: the frame's signature gains the indicator's answer, and its 10-a-second throttle already covers four a second. On a Stream Deck + that is `setFeedback`; on a Mirabox knob the 176 × 112 screen image; Ulanzi dials are not declared.
- **A Frame Jog or Speed Control turn draws nothing**, as above.

A marker jump from the Replay Markers dial across a long recording takes the same ten seconds as from a key, which is why the dials are in this issue and not a later one.

## Why not an animated image

Recorded as the reason it was not chosen (the maintainer asked: "Maybe Stream Deck supports animated GIFs? What about other platforms?"). Read from each host's documentation and our adapters on 2026-10-13; nothing was tried on hardware.

- **Stream Deck**: Elgato's keys guide says "The `setImage` function does not support animated image formats, such as GIF." A manifest state image may be a GIF, but it is a static file in the plugin folder that cannot carry the user's colours or title.
- **Ulanzi**: the plugin SDK lists `setGifDataIcon` and `setGifPathIcon`. Our client sends type 1 (base64 data) only, so using them would be a new code path, unverified without hardware.
- **Mirabox**: the SDK's `setImage` example includes a GIF data URI and does not say whether it animates.
- **In all three cases** our keys are assembled from SVG with the user's colours, title and border and rasterized to PNG. A pre-made GIF cannot carry those, and making one at run time needs an encoder the plugin does not have. Redrawing eight cached frames does the job on every host with the path that exists.

## Rejected

- **Every replay key on the page shows it.** It is true that the others are affected: their presses would be held or would take over. But it turns one press into a page of spinners, costs an image set per key per frame, and says the same thing a dozen times. The pressed key answers "did my press take", which is the question. If the others turn out to be missed, the indicator's token model already allows it.
- **A still "busy" look with no motion.** Cheaper, and what waiting uses. For a search of ten seconds or more, a still key cannot be told from a plugin that has stopped.
- **A minimum showing time**, so a spinner once shown stays for at least half a second. It would keep the look on a key whose jump has landed. A single frame for a narrow band of jump lengths is the smaller fault.
- **Progress as a share of the way.** Telemetry reports no seek progress, and the fallback search has budgets (up to 30 probes, 10 lap steps) but no estimate. A coarse stage name (mapping the recording, searching, stepping to the line) is possible later and is not asked for.
- **Elgato's `showAlert` for not done.** One host of three.
- **Driving the frames from the telemetry callback.** A subscriber is told only when `SessionTick` advances, and a paused replay changes without it, which is exactly when a search is invisible.
- **Deriving the state in the indicator** from the cursor claim and the dial's pending target, without #1430. It would be a second judge of "landed", and plain searches record nothing today.

## Where it sits in the release

It ships in 3.6.0 (maintainer, 2026-10-13) with the features it decorates.

- **After #1430**, which is a hard dependency: the seam, the token on every send, and `beginReplayJump`'s token and result are all in #1430's diff.
- **After #1426 and #801** as well, because its render-path edits are in files both of them edit (`replay-control.ts`, the dial surface, the Replay Markers surfaces). The order is #1354, #1430, #1426, #801, #1437. If #801 slips, #1437 can go before it, and #801 then passes its runner's token as its own change.
- **Its own module and tests need only the seam**, so they can be written as soon as #1430's names are final.
- **Changelog**: one fragment of its own, `changelog.d/1437-replay-key-busy-look.md`, category Features, weight 50. It is one thing a user sees across two actions, one of which shipped before this release, so it is not an edit to the dial's or the jump's fragment.

Files touched:

- **New:** `shared/replay-busy-indicator.ts` and its test; `icon-composer/src/busy-mark.ts` and its test.
- **`icon-composer`:** `assembleIcon` gains the mark option; the package's `CLAUDE.md`.
- **`replay-control.ts`:** the icon function takes the indicator's answer; keys register and unregister; the ignored press is reported; the redraw de-duplication key gains the answer.
- **`replay-markers.ts`:** the same for Next and Previous. The busy look wins over the greyed "nothing to jump to" look, and `needsRedraw` learns of it.
- **The two dial surfaces:** the caption line and the signature. **`shared/dial-box.ts`** and its strip and knob renderers: the optional caption mark.
- **Docs:** the Replay Control and Replay Markers website pages (a short section on the three looks, and "press again to act now" if #1430 lands that rule), `.claude/rules/key-icon-types.md` beside the binding-missing glyph, `.claude/rules/encoders-and-touchscreen.md` for the dial boxes, `packages/iracing-actions/CLAUDE.md`. The `iracedeck-actions` skill lists modes and settings, none of which change. There is no manifest or Property Inspector change, so the three plugins need nothing but the build.

## What the user sees

- Press Jump to Fastest Lap when the plugin has no record of the lap: after three quarters of a second the key's artwork gives way to a ring of stepping dots, until the replay arrives. If the search finds nothing the key shows a red cross for a second.
- Press Next Marker for a marker hours away in a long recording: the same dots until the replay is there.
- Press Play while a far jump is still on its way: the Play key shows three still dots until the jump lands and the replay plays.
- Press a second Jump to Fastest Lap key while one is searching: it shows the red cross for a second.
- Press anything that lands at once: nothing changes on the key.

## Open questions

For the maintainer; each has a proposal above.

1. **Which look**: A, B or C.
2. **The frame rate**: four a second, or another of the rates on the page.
3. **Waiting as a still mark of its own**, or the same spinner as working.
4. **Not done**: the red cross for a second on every give-up and on an ignored press, or nothing.
5. **The dials in this issue**, or keys only.
6. **The delay** of 0.75 s.
7. **No setting** for any of it, including one to turn it off.
8. **The website wording**, shown in full when it is written.

## Out of scope

- **Car and camera keys** (#1435). They send no replay command and are outside the gate.
- **The hidden legacy actions** (Replay Transport, Replay Speed, Replay Navigation). They pass no token and keep their icons.
- **Presses refused for other reasons** (no replay open, no fastest lap, a speed already at its limit). The sign is built so they could use it; whether they should is its own question.
- **A stage name or a percentage** for a long search.
- **The rule for what a held press does** (wait, take over, drop). That is #1430's, and this spec draws whatever it decides.
- **Animated images** on any host, and any change to `IDeckPlatformAdapter` or an adapter.
- **A shared timer helper** for the repo's other flashing keys.
- **Ulanzi dials**, which are not declared.

## Testing

**Unit:**

- **The mark:** each of the three, for the chosen family, is built from the safe SVG feature set only (a test refuses `filter`, `mask`, `clipPath` and `style`); step _n_ and step _n_ + 8 are the same text; the colours follow the slots; `assembleIcon` with no mark gives byte-identical output to today's.
- **The indicator, on fake timers and a fake gate:** nothing is drawn and no timer exists while no fact stands; under way for less than 0.75 s draws nothing at all; at 0.75 s it draws working and steps every 250 ms; waiting draws once and no interval is created; both facts draw waiting; completed and superseded return to rest with one redraw; gave up draws not done at once, inside the delay too, and rest one second later; an ignored press draws not done, and nothing on a key already working; a hand-over from waiting to under way does not restart the delay; a key that disappears is not redrawn and the interval ends with the last one; a key that reappears with a fact standing draws at once; a tick that finds the fact gone stops without a notice; the five-minute ceiling stops and warns; a redraw that throws does not stop the others; two keys spinning share one interval and one step.
- **The per-mode rule:** a Frame Forward key with a long run of its own jumps never draws working and does draw waiting and not done.
- **The actions:** Replay Control and Replay Markers pass the answer to their icon functions, register on appear and unregister on disappear, and Replay Markers' busy look wins over "nothing to jump to".
- **The dial surfaces:** the signature changes with the answer, a turn with no token draws nothing, and pushes stay under the throttle; with a pending count the caption draws the mark and the count together, on the strip and on the knob; a box with no mark is byte-identical to today's.
- **Cost:** a test renders eight frames twice through the rasterizer service and holds that the second turn renders nothing.

**The real code against the sim** (the harness used for #1317 and #1430), with image sets recorded in place of a deck:

1. A jump of 30000 frames: no image set beyond the key's ordinary ones.
2. A jump across the recording: the first working frame between 0.75 and 1 s after the press, four a second after it, and the ordinary icon within one frame of the landing reading back.
3. Jump to Fastest Lap with no record, for a car that is found and for one that is not: working throughout, then rest or not done.
4. A Play held behind that long jump: waiting on the Play's token, then rest.
5. Next Incident with nothing ahead on a paused and on a playing replay: not done after the search's bound, and after about 0.4 s.

**Manual, in an iRacing replay**, on a Stream Deck + and on the Mirabox, since how a key looks while it is redrawn four times a second is a property of the host and the hardware:

1. Each of the five cases under _What the user sees_.
2. A search of ten seconds or more: the spinner steps evenly, the key does not flicker between frames, and the other keys on the page stay responsive.
3. Turn the page while a key spins and turn back: it is spinning; turn back after the landing: it is at rest.
4. Leave the replay for the car while a search key spins: it returns to its icon. Leave it during a long position jump: the key goes on spinning for up to the jump's wait, then shows not done.
5. A detent of the Replay Markers dial to a marker far away: the box shows the dots. A Frame Jog spin of several seconds: it does not.
6. With the user's own colours, a hidden title and a border with glow on the key: the mark still reads.
7. With a flag flying and the flags overlay on: the flag flashes, and the key is right when it clears.

The Ulanzi host cannot be tested by the maintainer (no hardware). What is unknown there, and on any host, is only how four image sets a second look on the device, and the flags overlay already sends two a second to every opted-in key on all three.
