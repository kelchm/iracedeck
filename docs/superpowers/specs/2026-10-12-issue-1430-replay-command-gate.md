# One gate for every replay command: only a jump goes into a jump under way

> **Issue:** [#1430](https://github.com/niklam/iracedeck/issues/1430) · **Supersedes:** _none_ · **Superseded by:** _none_
>
> Point-in-time design record. The code and `.claude/rules/` are the truth; this is not documentation.

A replay command sent while a position jump is still under way ends the jump somewhere else, and today every key and dial sends at once. This spec puts one gate in front of every replay command the plugin sends. It builds on the jump sequence of #1276 (`2026-09-28-issue-1276-replay-jump-playback.md`) and on the pending target Frame Jog introduced (#1317, in `2026-10-04-issue-1316-replay-control-dial.md`), and it is the gate that #1426 (`2026-10-12-issue-1426-replay-record-steps.md`) and #801 (`2026-10-12-issue-801-replay-control-dial-stepping.md`) both call for: they build none of their own.

## The decision

1. **One module sends every replay command.** `shared/replay-command-gate.ts` is the only caller of `getCommands().replay` in the plugin, and a test refuses any other. A surface says what it wants sent; the gate decides when it goes out, and whether.
2. **One record of where the replay is headed.** Every position jump the plugin sends is recorded as the pending target, whichever surface sent it. It is #1317's rule moved to `shared/`, and it replaces the two records there are today: the dial's pending target and the replay cursor's pending landing.
3. **While something is under way, only an absolute position jump goes out.** A frame step becomes such a jump, a pause sends nothing, and a playing speed or a search waits for the landing to read back (the table below).
4. **A command that waits is sent once or dropped, never sent late.** It goes out when the landing has read back. It is dropped, with a debug line and nothing sent, when the jump ends anywhere but on its target or when three seconds have passed since the press. Holding the press until the jump lands was chosen over dropping it (maintainer, 2026-10-12).
5. **Every wait ends on what telemetry reads back.** No measured duration is a constant that correctness rests on (maintainer, 2026-10-12): the durations below are one fast machine's, and each clock here is a give-up that sends nothing.

## The sim facts this rests on

All measured on 2026-10-12 by a scripted probe and by the real dial code against the maintainer's sim (a paused saved replay of 204563 frames, one fast machine). The full record is in #1426's spec and in the header of `replay-position-state.ts` (#1317); these are the ones the gate is built on.

- **Any replay command but a position jump, sent into a position jump under way, ends the jump where it stands.** A frame step (1684 to 8768 frames off its target), a lap, session or incident search (on the start frame, half-way, and five frames short), a play command (about 9000 frames on), and a pause sent alone 170 ms into a jump (about 8700 frames on, 3 of 3), even on a replay that is already paused.
- **An absolute position jump sent meanwhile retargets the seek, and the newest target wins** (10 of 10; fifty jumps 60 ms apart landed on the last one's frame). A jump relative to the cursor is resolved from the start of the tape.
- **A seek shows in telemetry only from about 20 ms after the send**, as `IsReplayPlaying` false and `ReplayFrameNum` 0 until the cursor lands. `ReplayPlaySpeed` reads 0 throughout (223 of 223 mid-seek samples, all on a paused replay). The first sample that shows the target frame is back in the replay.
- **A spin that keeps retargeting reads "in a replay" on an older target for a tick or two about every 400 ms**, while a newer jump has been sent and does not show yet. So a reading that is out of a seek does not mean nothing is under way.
- **A jump sent within about 415 ms of another's landing is carried out about 415 ms after that landing**, at any depth of the tape; a lone jump of tens of thousands of frames forward takes about half a second.
- **A search after quiet shows nothing on its way**: the old frame until the new one, no frame 0 and no drop of the flag. Two searches on their way at once do not add up; one sent after the last has read back does.
- **A search sent in the poll that first reads a landing can be lost**: 3 of 3 after a long jump, where one sent 100 ms or more later landed 6 of 6. Nothing between was measured.
- **A speed sent on the first read of a landing is acted on**: the dial's held Play, 6 of 6, and #1276's playback step.
- **A camera switch does not end a seek** (6 of 6, the broadcast).
- **In the maintainer's hardware round** (Stream Deck+, the same day) a Replay Control key in Play / Pause mode pressed about 50 ms after the last jump of a Frame Jog spin threw the replay twice: headed for frame 15811 it read 24530, and headed for 24642 it read 33291, playing. The dial's own Press in the same situation held the Play and landed exactly.

## What counts as under way

The gate answers one question for every sender: is the plugin moving the replay right now. Five things say yes.

- **A pending target stands.** Recorded when a position jump goes out, and ended when its frame reads back from a replay in a way that can only mean the move is done (`conclusive`, as #1317 defined it: a frame the replay could not have been showing anyway, or a reading that was not the replay's seen since the send). The first out-of-seek reading does not end it, for the reason in the sim facts.
- **A target that never reads back is given up**: for as long as the jump that sent it holds the replay cursor it stands (the bound is the #1276 sequence's own wait, whose clock stops while a seek shows), and otherwise `REPLAY_TARGET_GIVE_UP_MS` (3 s) after the send, as on the dial today. Not a clock that stops while a seek shows: a replay left for the garage reads as a seek, and a target kept through it would send the next turn from a frame that is minutes old.
- **A replay jump that knows its frame is under way from the press, not from its seek** (added 2026-10-12, found reading #1433 against #1276). The #1276 sequence pauses a moving replay and waits for the pause to show before it seeks, a poll or more. In that window nothing has gone out that a target or telemetry could show, and the replay reads paused on a valid frame: a Frame Jog detent there steps one frame and takes the cursor, the sequence is cancelled before its seek, and the marker or fastest-lap jump is lost. So the sequence records the frame it is going to as the target when it starts, before its pause. The fastest-lap walk's probes have no frame to record; a command during them ends the walk, as now.
- **A search the gate sent has not landed.** Telemetry cannot show one, so the gate records it, until its landing reads back or its landing wait runs out (the caller's bound; for a key, the wait the fastest-lap walk already uses).
- **The latest reading shows a seek** (`isSeeking`: `IsReplayPlaying` false and nobody driving) while the debounced replay state still says a replay is on screen. This is the witness for a seek nobody recorded. Past the debounce's grace a seek and the garage read the same, and nothing is held on it.

A jump to the frame a replay reading already shows, sent with no target standing, is sent and records no target: it can never read back conclusively, and everything behind it would wait three seconds and be dropped. A marker key on the marker the replay stands on and a second Jump to Fastest Lap are the cases.

The target is cleared only by its read-back, its give-up, or a newer jump replacing it. Today any other action's command clears it and is then sent into the jump; that pair of steps is the bug.

## What each command does

| Command | Nothing under way | A target stands, or a seek shows | A search has not landed |
| --- | --- | --- | --- |
| Absolute position jump | sent, target recorded | sent, replaces the target | sent |
| Frame step | sent; on a paused replay the target is recorded one frame on | becomes an absolute jump one frame from the target | waits |
| Pause | sent | **not sent** | sent |
| Play and every other speed | sent | waits | sent |
| Search (lap, session, incident, start, end, Jump to Live) | sent, recorded as under way | waits, and the landing must hold | waits |
| Anything else (`searchSessionTime`, `eraseTape`, a relative jump) | sent | waits, and the landing must hold | waits |

- **A frame step into a jump is Frame Jog's own rule**, now for every sender: `planFrameMove` over the shared target, so a Frame Forward key, its long-press repeat and a dial detent are one code path with one end-of-tape margin. When only the witness says a seek is under way there is no frame to measure from, and the step is dropped. The key's repeat waits 250 ms after each step, so a held key on a quiet replay keeps sending real steps.
- **A pause sends nothing because the replay is already paused.** Every jump the plugin makes pauses a moving replay before it seeks (the #1276 sequence, Frame Jog). The press still takes the replay cursor, which ends a jump sequence's playback step and empties the waiting slot, so the replay rests paused on the target. The same holds while a replay jump (`beginReplayJump`: a sequence, the fastest-lap walk, #801's search runner) holds the cursor having paused the replay: the pause ends it and sends nothing.
- **A speed waits for the target's read-back and goes out on that reading.** Measured safe on the first read. It does not wait behind a search: nothing measured says a speed harms one, and an unmoved search looks under way until its wait runs out, which would make Play after Next Session in the last session wait a second or more.
- **A search waits for the read-back and then for the reading to hold**: three reads 50 ms apart, the read-back form of the 100 ms that worked every time where the first read lost the search. This is #801's rule, and its only home is here.
- **A command that waits takes the replay cursor at the press, as every command does today.** A jump sequence under way sends nothing more, so a Play pressed during a fastest-lap walk ends the walk and does not wait for it. A speed or a pause pressed during a jump is the driver's own word and replaces the sequence's playback step, which is #1276's rule unchanged.
- **A sequence still in its pause always gets a seek, and never before the pause has shown** (added 2026-10-12). A sequence whose claim is taken sends nothing more today, which is right for what follows its landing and wrong for a seek it has not sent yet: a Play pressed 30 ms after Next Marker on a playing replay would lose the jump, and the same Play 100 ms later would not. So whatever takes the cursor in that window, one seek goes out to the newest target once the pause has read back. It goes to the sequence's own frame when the taker sends no position jump: a press the gate holds, a pause it does not send, or a replay jump that waits on the target, which #801's search runner is and the gate's own jump for a waiting search is. It goes to the taker's frame when the taker is a position jump or a frame step turned into one, which moves the target and sends no second pause. The taker owns only what follows the landing. The wait for the pause stays because "a pause followed at once by a jump lands on the jump" was measured only on a replay that was already paused; from a playing one it is the fourth capture below. A sequence that stops before any seek can go out, its pause refused or never showing, clears the target it recorded.
- **A search that waits behind a jump sequence carries that jump's playback on.** Otherwise the same key pressed slowly would leave the replay playing and pressed quickly would leave it paused, and #1426's commonest miss is exactly that: two record jumps toward the end of what was recorded, then a press the record cannot answer. So the waiting search inherits the speed the sequence captured, as a jump that takes over from a jump does today, and when the search has landed the gate applies the **Replay jump playback** step. Search first and playback after: a search on the held reading of a paused landing is the measured case. If the search is dropped as stale, the step goes out when the jump lands on its target; a jump that ends off its target sends nothing more, as now. A search pressed on a quiet replay is unchanged: no pause and no playback step. Behind a dial's frame move there is no captured speed and nothing to carry.
- **A claim holder's own commands go through the gate too.** The jump sequence's pause is not sent while a target stands (today's takeover rule, now also true when the jump under way is a dial's), its seek is a jump, and its playback step goes out after its own read-back wait.

## The waiting slot

One command waits at a time, for the whole plugin, and the newest wins: a second Play replaces the first, a search replaces a waiting speed. A pause empties it. So does a new jump, which is a newer word about where the replay should be: a waiting search was meant to start from the old target, and a frame move is a request to stand still. That holds for every replay jump (`beginReplayJump`), whatever it sends first, #801's search runner included, whose first send is a pause or a search and not a position jump. A jump that starts while a speed waits takes that speed as the one it captured, so the **Replay jump playback** setting decides what becomes of it, and the speed is not sent on the read-back the jump is itself waiting for.

The slot is checked on a 20 ms timer (`REPLAY_JUMP_POLL_MS`) that runs only while something waits. It decides one thing: send or drop. A dropped command writes a debug line saying what was dropped and why, and the key's icon follows telemetry as before.

**Three seconds** (`REPLAY_TARGET_GIVE_UP_MS`, one number for both) is the longest a key press waits. A press that fires many seconds late is worse than one that is dropped.

## What it gives #1426 and #801

Two waits, exported from the gate, each taking its time bound from the caller and resolving settled, gave up, or superseded by another replay command:

- **Wait until settled, holding N reads.** Nothing under way, shown on N reads 50 ms apart.
- **Wait for a search to land.** The frame moved by more than playback explains, then two out-of-seek reads 50 ms apart that agree within what playback explains. A timeout means not landed; an unmoved search and a lost one look the same, and the caller decides whether to try again.

Policy stays with the caller. #801's runner keeps its signed count, its clamp of ten, its one retry and its playback step, built on the two waits, and is not cut off by the keys' three seconds. #1426's Lap and Session keys step from the shared target while it stands and from the debounced frame otherwise, and its miss path sends a search through the gate like any other key. Both specs are amended to say so.

## The seam, and what it costs

About 73 send sites in seven files change from `getCommands().replay.x()` to the gate. The files touched:

- **New:** `shared/replay-command-gate.ts` and its test; the single-sender test.
- **Moved to `shared/`:** `replay-control/replay-position-state.ts` and its test, and `planFrameMove` out of `replay-control-frame-jog.ts`. `keepReplayTarget` goes: the dial's commands no longer need to protect the target from the dial's own cursor takes.
- **`shared/replay-cursor.ts`:** the pending landing (`recordReplayLanding`, `pendingReplayLanding`, `clearReplayLanding`) is replaced by the shared target; a cursor take no longer clears it. The claim and the pending jump speed are unchanged.
- **`shared/replay-jump-playback.ts`:** sends through the gate; `tookOver` reads "something is under way" from it.
- **`replay-control.ts`:** 42 sites in `executeMode`, the fastest-lap walk and the key repeat.
- **`replay-control-dial-surface.ts`:** `heldSpeed`, `isPausedForMove` and the follow timer's send move into the gate; the surface keeps its redraw.
- **`replay-markers-ops.ts`:** its jump records the shared target and its anchor reads it. The marker surfaces' one-second, 60-frame rule for when a landing still anchors is replaced by the target's read-back.
- **The hidden legacy actions** Replay Transport (8 sites), Replay Speed (3) and Replay Navigation (11): the same swap. They are still runnable from older profiles, so they are not left out.
- **Tests** for each of the above, which mock `getCommands` today and will mock or drive the gate.
- **Docs:** `.claude/rules/encoders-and-touchscreen.md`, `packages/iracing-actions/CLAUDE.md`, the Replay Control and Replay Markers website pages, and a Bug Fixes changelog fragment (a key thrown by a Replay Markers jump shipped before the dial did).

#1429 (Replay Markers treats a long seek as the replay being left) wants the SDK's replay state told about the plugin's own seeks. The gate is the one place that knows every one of them, so that becomes one call from here; it is not made in this issue.

## What the user sees

- A key or dial input made while the replay is still jumping acts when the jump lands, about 0.4 s later on the maintainer's machine, instead of throwing the replay thousands of frames.
- A Frame Forward or Frame Backward press during a jump moves one frame from where the jump is going.
- A Pause during a jump leaves the replay paused on the frame the jump was going to.
- A Next Lap, Session or Incident press during a marker or fastest-lap jump moves on from where that jump lands, and the replay then plays or stays paused as the **Replay jump playback** setting says, the same as when the two are pressed slowly.
- If the jump takes longer than three seconds, or does not land on its frame, the waiting press does nothing.

The website says it in the wording Frame Jog's section settled on: quick inputs ride on iRacing's own replay seeking, and take longer on a slower machine or in a long replay.

## What would change this

Four captures were not taken: the sim was down when this was written, and the spec goes first with the sim on a later day (maintainer, 2026-10-12). Each is modelled as decided above, and none is waited for.

- **Play, pause and a frame step sent 50 to 150 ms after a lap search following quiet.** Modelled: a speed and a pause do not harm a search on its way; a step does. If a speed or a pause ends a search part-way, the "A search has not landed" column changes to "waits" for speeds and "not sent" for a pause. If a step is harmless, its cell becomes "sent".
- **Go to end sent into a jump under way.** Modelled as a search: it waits. If it turns out to retarget the seek like a position jump, Jump to Live may go out at once.
- **A pause and then an absolute jump at once, on a replay playing at 1x and at 4x.** Modelled as unsafe: no seek goes out until the plugin's own pause has read back. If the jump lands on its frame every time, a takeover and a detent in that window may seek at once.
- **The Next Car key binding sent into a jump.** Modelled on the camera broadcast, which is safe: the tap goes out at once and never passes through the gate. If the key ends the seek, the car-cycle tap waits like a search.

Also unmeasured, and covered by the read-backs and not by any number here: a slower machine, a jump on a replay that is playing, and anything between 0 and 100 ms after a landing is read.

## Out of scope

- **Commands from iRacing's own replay bar or keyboard.** The plugin cannot see them; the target's give-up and the witness are what limit the damage.
- **Making key searches add up.** With one waiting slot three quick Next Lap presses move two laps, never part-way. A count is #801's runner.
- **The record steps and the search runner themselves** (#1426, #801).
- **Telling the SDK's replay state about the plugin's seeks** (#1429).
- **The step-per-sim-update limit**: two frame steps inside one sim update still count as one.
- **A setting** for the three seconds or the hold.
- **Ulanzi dials**, which are not declared.

## Testing

**Unit:**

- **The gate:** each cell of the table, by command kind and by what is under way; a waiting speed goes out on the first reading that ends the target and a waiting search only on the third held read; newest wins in the slot; a pause empties it and sends nothing; a jump empties it, and a jump sequence that starts while a speed waits captures that speed; a drop at three seconds and on a landing off target, each with nothing sent; nothing held outside a replay or past the debounce's grace.
- **The target:** the moved `replay-position-state` tests pass unchanged; a cursor take no longer clears it; it stands while its jump holds the cursor and for three seconds otherwise; a jump to the frame on screen records none.
- **Each sender:** Replay Control's Play / Pause, a speed mode, a frame step, the repeat and a search, each pressed with a marker jump, a fastest-lap record jump and a dial jump under way; the dial's Speed Control detent, Press and Jump to Live with a marker jump under way; a Frame Jog detent on a replay-true reading with another surface's jump just sent; the three legacy actions. In every case: nothing but a position jump is sent while the target stands.
- **The jump sequence:** its pause is not sent when a dial jump is under way; a speed pressed during it goes out once, at the landing; a search pressed during it goes out after the landing has held and is followed by the playback step with the speed the sequence captured, under each value of the setting; a search dropped as stale still gets the playback step on the landing, and none when the jump ends off its target; a replay jump that starts while a speed waits captures it and the speed is not sent, for a position jump and for one whose first send is a pause; with the sequence's pause sent and not yet shown: a Play, a pause, a search and a replay jump that sends no position jump each leave one seek to go out to the sequence's frame once the pause shows, and then do what the table says; a second jump and a Frame Jog detent each move the target, send no second pause and no seek until the pause shows, and then one seek to the newest target; a pause that never shows clears the target and sends no seek.
- **The single-sender test** fails on a planted `getCommands().replay` call outside the gate.
- **The two waits:** settled, gave up and superseded for each; the bound is the caller's.

**The real code against the sim** (the harness used for #1317, with the observer in a child process):

1. **Before the fix, as the control:** a Frame Jog spin, then 50 ms after its last jump the Replay Control key's Play, its pause, a frame step and a lap search, each as the key sends it. Each should throw the replay, the Play by about 8700 frames as in the maintainer's round. A control that does not fail proves nothing about step 2.
2. **After the fix, the same four:** each lands on the spin's target, then plays, rests, stands one frame on, or stands one lap on.
3. **The other way round:** a marker jump across the tape, then a Speed Control detent, a Press and a Frame Jog detent 20, 50 and 200 ms later.
4. **The whole logged minute of the maintainer's Frame Jog round** again at three depths of the tape, which must still act on every detent.
5. **The four captures** under _What would change this_.

**Manual, in an iRacing replay**, on a Stream Deck+ and on the Mirabox knob:

1. Spin Frame Jog and press a Replay Control key in Play / Pause mode straight after: the replay lands where the spin was going and then plays.
2. The same with Frame Forward, with a held Frame Forward, with Next Lap and with Next Incident.
3. Press Next Marker and then Play / Pause on another key before the picture has moved; then Pause instead. Do both on a playing replay too, as fast as the two keys can be pressed, and with a Frame Jog detent in place of the key: the replay must always end at the marker, or one frame from it for the detent.
4. With the replay playing and **Replay jump playback** on Remember state, press Next Marker and then Next Incident at once: the replay must be playing after it lands, as it is when the two are pressed slowly. Again with the setting on Stopped.
5. Press Jump to Fastest Lap and turn a Speed Control dial during the walk: the walk stops and the speed is obeyed.
6. Press a marker key for the marker the replay stands on, then Play at once: it plays without a wait.
7. An old profile's Replay Transport and Replay Navigation keys, pressed during a marker jump.
8. From the car: every key and dial does what it did before.
