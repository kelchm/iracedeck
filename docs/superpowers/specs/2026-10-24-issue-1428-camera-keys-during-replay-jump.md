# Car and camera keys during a replay jump: the key waits for the landing, and a walk it stops puts the replay back

> **Issue:** [#1428](https://github.com/niklam/iracedeck/issues/1428), [#1435](https://github.com/niklam/iracedeck/issues/1435) · **Supersedes:** _none_ · **Superseded by:** _none_
>
> Point-in-time design record. The code and `.claude/rules/` are the truth; this is not documentation.

## Context

The two issues are one moment seen from two sides: a camera key pressed while the plugin is moving the replay. #1428 is what the key does to the jump (it cancelled it, and the replay stayed paused); #1435 is what the jump does to the key (iRacing drops a camera change that arrives during a position seek, 9 of 9 for the Next Car key and 3 of 3 for `CamSwitchNum`, #1430's captures). One spec covers both (Niklas).

Both issues were filed before #1467 (`2026-10-20-issue-1467-replay-engine-package.md`), and read on `master` at `f452ec5e8` most of what they describe is already gone. #1467's Decision 3 moved #801's camera waiter into the engine as `requestCameraSwitch` (`packages/replay-engine/src/camera-switch-waiter.ts`), and every camera switch of Replay Control's key goes through it: Jump to My Car, Next Car, Previous Car, Next Car Number and Previous Car Number (`replay-control.ts`, `requestKeyCameraSwitch`). On a quiet replay it sends at once; behind a position jump it waits, with no claim, on `waitForReplaySettled({ jumpsOnly: true, reads: 3, timeoutMs: REPLAY_JUMP_MAX_WAIT_MS })`; and it cancels only the two lap walks (`cancelLapWalks`, over `cancelReplayCursorOwnerIf`), never a marker jump, a record jump or the paced search. So #1428's steps (Next Marker, then Next Car) now end with the replay landed, playing at its captured speed, and the camera moved; and #1435's steps (a far marker, then Next Car) end with the camera moved. `replay-control.test.ts` pins the wait and the non-cancel ("a camera key (#1467): it waits behind the jump, and cancels only a lap walk"), though only behind a dial Frame Jog jump, which has no playback step to restore.

What is left, and what this spec settles:

1. **A lap walk a camera key stops leaves the replay paused** on whatever probe frame the walk had reached, its captured speed cleared (`cancelReplayCursorOwner` clears the pending jump speed). That is #1428 for Jump to Fastest Lap and Jump to Race Start by search, which #1467 did not touch.
2. **Camera Controls and Race Admin send at once**, so a camera change they make during a position seek is dropped (#1435). They never took the cursor (Niklas, 2026-10-10), so they never had #1428's problem.
3. **The tests #1428 asks for**: each mode during the pause wait and the landing wait, playing and paused, against a real jump sequence.
4. **One website sentence** that still says any Replay Control key cancels a jump.

## Decision: the key waits for the jump (shape a)

Of #1428's two shapes, **(a)**: a camera key does not touch a jump; it waits for the jump to land and sends then. It is what `master` does for Replay Control's keys since #1467, and this spec makes it the rule for every camera key in the plugin.

**Why not (b)**, the jump cancelled as before but restoring its captured speed when the canceller sets none. A camera change sent into a seek is dropped (#1430's fourth capture), so under (b) the key would cancel the jump, cut it short where iRacing had got to (#1275), restore the speed there, and then still not move the camera. It fixes #1428's symptom and keeps #1435 in full. It also needs the cursor API to tell a canceller that sets a speed from one that sets none, a distinction (a) never has to make for a jump.

**Checked against the code that (a) closes #1428 for every jump sequence.** The waiter's only cursor call is `cancelReplayCursorOwnerIf(LAP_WALK_OWNERS, by)`, which leaves any other owner standing and, unlike `cancelReplayCursorOwner`, neither clears the pending jump speed nor tells `onReplayCursorTaken`. A marker jump, a record jump (Jump to Fastest Lap's, Jump to Race Start's, Next / Previous Lap and Session from the record) and the dial's jumps therefore keep their claim, land, run their own step after the landing, and end in `applyReplayJumpPlayback` with the speed captured at the press. The pause-wait window is covered too: `beginReplayJump` records the jump's frame as the pending target before its pause (`openReplayJump` with `frame`), so `isUnderWay()` reads true from the press, and a key pressed before the seek has gone out waits as one pressed during it.

**Where (a) does not reach**, each handled below: a lap walk, which a camera key must stop rather than wait for (*A lap walk a camera key stops*); a jump that has not landed within `REPLAY_JUMP_MAX_WAIT_MS`, where the switch is dropped (*The wait*); and the gap between a jump's landing and its own camera switch (*Ordering with a jump's own camera switch*).

## The rule

What a camera key does, by what the replay is doing when it is pressed. "Sends" is the key's own send: a key binding tap or a camera broadcast, never a replay command.

| The replay | The key | The replay afterwards |
| --- | --- | --- |
| Quiet (nothing under way), playing or paused | sends at once | unchanged |
| A position jump under way: a jump sequence in its pause wait, its seek, or its landing; a dial Frame Jog jump; the gate's takeover jump | waits, with no claim, until no jump has been under way for three reads 50 ms apart, then sends | the jump lands on its frame and does what **Replay jump playback** says: a replay that was playing plays on at its speed under `remember`, a paused one stays paused |
| A jump sequence landed but not yet ended (its own camera switch and its playback step still to run) | waits for it to end, then sends (new, below) | as the row above |
| A search on its way (a key's Next / Previous Lap, Session, Incident; the dial's paced search) | sends at once; the search is not touched | the search lands and its playback step runs (8 of 8 landed, 5 of 5 camera changes shown, #1430) |
| A lap walk (Jump to Fastest Lap or Jump to Race Start by search) | interrupts the walk, waits for it to put the replay back, then sends (new, below) | on the frame and at the speed the walk's press found it, the camera back on the car it was on if the press had moved it, and then the key's switch |
| Outside a replay (live, in the car or spectating) | sends at once | unchanged; nothing in the waiter can read under way there, since no target stands and `isSeeking` counts only in a replay |

A key computes its target when its switch goes out, from the camera then, not at the press (as `requestKeyCameraSwitch` already does for the car-number modes): Next Car Number behind a jump steps from the car the camera is on after the landing.

## A lap walk a camera key stops

A walk rides the camera's car: iRacing's lap searches follow the camera, so a camera change during a walk points its remaining searches at another car (#1430 traced the outcome: the walk does not converge, records nothing, and rests where it stopped). So a camera key must stop a walk, as #1467 Decision 3 and #1448's rule 5 both say; waiting for it is not an option, since a walk is seconds long (up to a minute on a long tape before it gives up, per the Replay Control page) and the key was pressed because the driver wants another car now. What is open is what the stopped walk leaves.

**Decision (Niklas, 2026-10-24): the walk puts the replay back, as a walk that gives up does, the camera included, and the key's switch follows.** `putReplayBack` (`lap-walk.ts`) already does exactly this for a walk that found nothing: the cursor back to the frame the press found it on, then the speed captured at the press, and the press's own camera move undone (Jump to Fastest Lap's switch onto an always-my-car target, Jump to Race Start's borrow of the pole car). A camera key mid-walk means the driver no longer wants the lap, so the press is undone the same way and the key acts from where the driver was. The Replay Control page already promises the give-up half of this ("it puts the replay back as it was").

Rejected:

- **Leave it paused where the walk stopped** (today). That is #1428 itself, for the walks.
- **Restore the captured speed where the walk stopped** (shape b, literally). The walk stopped on a probe frame somewhere in the recording, and #1276's rule is that playing from a spot the driver did not ask for is worse than stopping there.
- **Let the key wait for the walk to finish on the old car.** It contradicts #1467 Decision 3 and #1448, and makes the driver watch a search they have already abandoned.

**What the cursor API needs.** Today a camera key ends the walk's claim (`cancelLapWalks` → `cancelReplayCursorOwner`), and a cancelled claim sends nothing more, so the walk cannot put anything back. The walk needs a second way to be stopped that ends its search but not its claim: an **interrupt**.

- `interruptLapWalks(by)` (in `lap-walk-owners.ts`, beside `cancelLapWalks`) marks the standing walk's claim interrupted, naming `by`, and only when its owner is one of `LAP_WALK_OWNERS`. It does not take the cursor: the claim stays current, the pending jump speed and the dial's pending speed are left alone, and `onReplayCursorTaken` is not told. The claim gains a read-only `interruptedBy: string | null` beside `cancelledBy`.
- The walk's `checkpoint` (and every other place it reads `cancelledBy`) throws a `LapWalkInterrupted` on an interrupt, distinct from `FastestLapPressCancelled`. `runFastestLapWalk` and Jump to Race Start's walk catch it and take their give-up path: `putReplayBack`, then the camera back if the press moved it, each after `waitToSwitchCamera` as today. The press then ends as `interrupted` in the log (`walk interrupted by <mode>; replay put back`).
- Everything else about the claim is unchanged. A replay command pressed while the walk is putting the replay back cancels the claim as it would cancel the walk, and the put-back sends nothing more: that press is newer than the camera key. Leaving the replay still **cancels** a walk (`replay-left.ts` → `cancelLapWalks(REPLAY_LEFT_CANCEL)`), with no put-back, since there is no replay to put back into.
- **A walk interrupted before it has read where the replay was** (in its opening pause, `origin.frame` still null) has moved the replay by its pause alone, so it sends only the captured speed, when that is not zero. `putReplayBack` today does nothing in that case, and a walk that gives up there is paused; the same addition fixes that too.
- The waiter counts **an interrupted walk still winding down** (`isLapPressInFlight()` with its claim interrupted) as under way, so the key's switch goes out after the put-back seek has landed and after the walk's own camera move back, never into the put-back's seek. Without this the switch could go out in the gap between the interrupt and the put-back's seek (the walk unwinds at its next await, up to one **Fastest Lap Search Delay** later) and then be overridden by the walk's camera move back.

The answer to #1428's task "what the cursor API needs to tell a speed command from one that sets none" is therefore: **nothing**, for a jump, because under (a) a camera key never cancels one; and for a walk it is not a speed distinction either but a distinction between a press that takes the cursor (cancel, sends nothing more) and a camera switch that takes none (interrupt, puts back).

## The wait

**The bound is `REPLAY_JUMP_MAX_WAIT_MS` (120 s) from the request, as built**, not #1430's per-jump wait. #1430 holds a replay key behind a jump for that jump's own wait (the larger of 3 s and twice its length at iRacing's seek rate) and drops it when the jump ends off its target, because a replay key's command means something only from the frame the jump was going to (a frame step from there, a pause there). A camera switch means the same wherever the replay stopped, so it goes out when the jump ends either way: landed, or given up by the gate (`replayTargetGiveUpMs` clears the target, which reads as quiet). The ceiling only catches a seek that is still showing after any jump's longest wait.

**What it rests on is read-back, never a timer.** "Under way" is the gate's own judgement: a pending target that has not read back (`readReplayPosition(telemetry).seeking`), a takeover or an owed seek, or a seek showing in a replay (`isSeeking`, `IsReplayPlaying` false with nobody on track). Three quiet reads 50 ms apart end the wait; a jump that starts meanwhile resets the count, so a key pressed during a marker jump that a second marker press takes over waits for the second. The 120 s ceiling only gives up.

**It is dropped**, logged at debug with the asker's name, when the ceiling is reached, when the wait throws, or when `stillValid()` is false as the switch would go out: its key or dial has left the screen, or the replay has been left. Unchanged from #1467.

**One switch waits for the whole plugin, newest wins, across every kind.** A further camera press during the wait, from any key or dial of any action, replaces the waiting one, so the camera moves once, by the last key pressed. That holds across kinds too: Next Car and then Cycle Camera during one jump end with only the camera group changed. Two switches sent back to back after a landing would each have to see the other's result in `CamCarIdx` before working out its own target, which is unmeasured, and "the last camera key you pressed during a jump is the one that acts" is the rule the Replay Control page already states.

**Ordering with a jump's own camera switch.** Jump to Fastest Lap's record jump switches the camera onto the lap's car in its step after the landing (`afterLanding` in `jumpToRecordedLapStart`), and Jump to Race Start's jump does the same for its borrowed car. Since #1467 a key does not cancel those jumps, so both switches go out: the jump's, then the key's. Today that order holds by timing only: the jump's step runs on the read-back its own 20 ms poll sees, the waiter's switch three 50 ms reads after the gate's. The waiter should not rest on that, so **a jump sequence counts as under way from its press until `jump.end()`**, through the gate's open jump handle (`openReplayJump` / `closeReplayJump`) for a jump that has a frame. A paced search's run, which opens no frame, is not held, so a key during the dial's search still sends at once. The key's switch is then the newer one, as #1430 put it ("the driver's camera choice is the newer one"), and Next Car steps from the car the jump left on screen.

## Which keys

Every camera switch an action's key or dial makes goes through `requestCameraSwitch`. Two columns decide its options: whether the switch changes **the camera's car** (and so must interrupt a walk), and how it is sent. "Measured" is #1430's captures (the Next Car binding and `CamSwitchNum` into a jump); everything else is inferred from sharing one of those two paths, or a sibling broadcast.

| Surface | Mode | Sent as | Changes the car | Measured | On `master` |
| --- | --- | --- | --- | --- | --- |
| Replay Control key | Next Car, Previous Car | Next / Previous Car binding (V / Shift+V) | yes | yes (the key) | through the waiter (#1467) |
| Replay Control key | Next Car Number, Previous Car Number, Jump to My Car | `camera.switchNum` | yes | yes (the broadcast) | through the waiter (#1467) |
| Replay Control dial | Track Order turn; Jump to My Car gesture | binding; `switchNum` | yes | yes | through the waiter (#801, #1467) |
| Replay Control dial | Car #, Position | `switchNum` | yes | inferred | #1448 |
| Camera Controls key | Cycle by Track Order | Next / Previous Car binding | yes | yes (same binding) | sends at once |
| Camera Controls key | Cycle by Car #, Switch by Car Number, Focus Your Car | `switchNum` | yes | yes (same broadcast) | sends at once |
| Camera Controls key | Switch by Position; Focus on Leader, Incident, Most Exciting | `switchPos` (the focus modes as iRacing's sentinels) | yes | inferred | sends at once |
| Camera Controls key | Cycle Camera, Cycle Driving Cam, Change Camera | `switchNum` to the same car, another group | no | inferred | sends at once |
| Camera Controls key | Cycle Sub-Camera | Next / Previous Sub Camera binding | no | inferred | sends at once |
| Camera Controls dial | every rotation mode and focus gesture | the keypad's dispatch (`executeCycle`, `focusCarNumber`, `focusMyCar`, `focusOn`) | as the keypad mode | as the keypad mode | sends at once |
| Race Admin | Car Selector key in focus intent (#790) | `switchNum` | yes | yes (same broadcast) | sends at once |

- **Camera Controls and Race Admin join the waiter**, keypad and Camera Controls' dial both (Niklas, 2026-10-24). Their switch waits behind a position jump like Replay Control's. A car-changing mode interrupts a lap walk; an angle-only mode (Cycle Camera, Cycle Driving Cam, Change Camera, Cycle Sub-Camera) does not, since the walk follows the car and not the angle, so `CameraSwitchOptions` gains `interruptsWalks` (default true). This keeps the letter of Niklas's 2026-10-10 decision that these actions take no cursor: an interrupt takes none.
- **The wrap goes at Camera Controls' shared dispatch**, the methods both its keypad and its dial call, so the dial gets the wait with no change to `camera-dial-surface.ts`. #1449 deliberately leaves "the wait for a jump under way before a camera switch" on Camera Controls' dial to this issue, and #1448 leaves "Camera Controls' own sends in a replay" here too, so without it nobody owns that dial.
- **Set Camera State is left out.** It sends `CamSetState`, flags on the camera (UI hidden, and so on), not a camera target, and nothing measured or traced says a seek drops it.
- **Focus Select Car** itself opens the Car Selector profile and sends nothing to the camera; the car key on that grid is the Race Admin row.
- `stillValid()` for the new callers: the key's or dial's context still on screen and the replay still on screen (`getReplayState().inReplay`), as Replay Control's. A switch that waited was asked for in a replay; one the replay was left under is dropped rather than sent into whatever came next.
- Race Admin's `showAlert` on a failed `switchNum` moves into the send callback, so it fires when the switch goes out, not at the press.

**No mechanical guard is added** for "every camera send goes through the waiter". The send is a callback the caller hands in, so a lexical scan of `camera.switchNum` / `tapBinding` call sites cannot tell a wrapped one from a bare one without a second convention to scan for. The `iracing-actions` `CLAUDE.md` line and the package rule ("One camera policy") carry it.

## Agreement with #1448

#1448's rule 5 (`2026-10-18-issue-1448-replay-dial-camera-car-modes.md`), for the Replay Control dial's Car # and Position detents: "a detent waits until no jump is under way: `waitForReplaySettled({ jumpsOnly: true })` with no claim, bounded by `REPLAY_JUMP_MAX_WAIT_MS`. A search on its way does not hold it [...] The detent cancels a cursor owner only when it is one of the two camera-riding walks [...] A newer detent replaces one still waiting, and the target is worked out when the switch goes out, from the camera car then."

The keypad rule here is the same rule, and it is the same code: there is one helper, `requestCameraSwitch`, which #801's Track Order mode began, #1467 moved into the engine, Replay Control's keys already call, and #1448's detents and this spec's Camera Controls and Race Admin sends will call. Point by point: the jumps-only wait with no claim, the 120 s ceiling, a search not holding it, newest wins, the target from the camera when the switch goes out, and walks the only owner touched. Two refinements made here apply to #1448's detents automatically, because they live in the helper: a walk is **interrupted** rather than cancelled, so "cancels" in #1448's rule now reads as this spec's interrupt (the replay is put back before the switch), and a jump sequence counts as under way until it has ended, not only until it has landed. #1448's rule 5 gets a one-sentence amendment on `master` saying so, in the same commit as this spec (#1448 has not shipped, so its spec is still open to edits).

## Out of scope

- **The camera switches a replay jump makes for itself** (the fastest-lap walk's switch onto its car, Jump to Race Start's borrow and its return, the record jump's switch after landing): #1430 settled them, and they do not go through the waiter. This spec only orders the key's switch after them.
- **Any change to the gate**: its table, slot, timings, or what counts as a jump for its own replay commands. The jump-sequence-open read is a new export the waiter reads, not a change in what the gate sends.
- **Shape (b)** for a jump: a camera key never cancels one.
- **A camera key's effect on the paced search** (the dial's Lap, Session, Incident steps): it sends at once and the run goes on, as built. Whether the search, which follows the camera's car, then steps the new car's laps is #801's behaviour and unchanged.
- **Set Camera State** (above), and Camera Controls' Focus Select Car profile switch.
- **More than one waiting switch** (one per kind, or a queue in press order): the newest wins.
- **Measuring** the unmeasured rows of *Which keys*; the manual test below covers the ones a user reaches.
- **#1432**, the walk's lap-start check, and every other change to how a walk searches.
- **Ulanzi dials**, which are not declared.

## Testing

**Unit, Replay Control keys against a real jump sequence** (`replay-control.test.ts`, the engine configured over the deck mocks with `fakeReplay`), for each of Jump to My Car, Next Car, Previous Car, Next Car Number, Previous Car Number, under a Replay Markers Next jump and under Jump to Fastest Lap's record jump:

- pressed during the jump's **pause wait** (the pause sent, not yet read back) and during its **landing wait** (the seek showing), each with the replay **playing** at 1x, at 4x and in slow motion, and **paused**: the jump lands on its frame; the replay ends at the captured speed under `remember` (and at 1x / paused under `play` / `stopped`); nothing camera-shaped is sent before the landing; the key's switch is sent exactly once after the jump has ended; `fakeReplay`'s `thrown` stays 0;
- under the record jump, the jump's own `switchNum` to the lap's car is sent before the key's, and the car-number key's target is computed from the camera after the jump's switch;
- a key pressed between the landing read-back and `jump.end()` waits for the end (the new under-way read), with a controlled clock that makes the waiter's reads fall inside that window.

**Unit, the walk interrupt** (`fastest-lap-press.test.ts`, `race-start-press.test.ts`, `lap-walk.test.ts`, `camera-switch-waiter.test.ts`):

- a camera key during each phase of the walk (its opening pause, a bisection probe's seek, a lap-step search, the nudge, a crossing probe and the final approach jump of #1432): the walk sends no further probe; the replay is put back on the press's frame at its captured speed (playing and paused); the press's camera move is undone when it made one, and not otherwise; the key's switch goes out after all of that, once; nothing is recorded into the lap record;
- interrupted in its opening pause (no origin frame read): only the captured speed is sent, and nothing when it is zero;
- a replay key pressed during the put-back cancels the claim and the put-back sends nothing more; the replay key's command stands;
- leaving the replay during a walk still cancels it with no put-back;
- `interruptLapWalks` leaves the pending jump speed and the dial's pending speed alone and does not notify `onReplayCursorTaken`; it touches no owner outside `LAP_WALK_OWNERS`;
- an angle-only switch (`interruptsWalks: false`) during a walk does not interrupt it.

**Unit, Camera Controls and Race Admin** (`camera-controls.test.ts`, `camera-dial-surface.test.ts`, `race-admin.test.ts`): each mode in *Which keys* sends at once live and on a quiet replay (same call, same arguments as today); behind a marker jump it sends once after the landing and the jump lands; behind a search on its way it sends at once; a car-changing mode interrupts a walk and an angle mode does not; a key that left the screen during the wait sends nothing; Set Camera State sends at once behind a jump; Race Admin alerts when the waited `switchNum` fails.

**The full gate**: `install`, `build`, `typecheck`, `format`, `lint`, `test`.

**Manual, in iRacing (Niklas)**, a saved replay and an in-session replay, the replay playing at 1x unless a step says otherwise:

1. Press Next Marker for a far marker and at once Replay Control's Next Car: the replay lands on the marker, plays on, and the camera moves to the next car. Repeat paused: it stays paused on the marker, camera moved.
2. The same with Next Car Number, and with Camera Controls' Cycle by Track Order and Cycle by Car #: the camera moves after the landing.
3. Jump to Fastest Lap with a recorded lap and **Always my car**, then Next Car straight away: the replay lands at the lap start on your car, then steps to the car ahead of you, and plays at the speed it had.
4. Jump to Fastest Lap on a replay with no record (the search), then Next Car while it searches: the replay goes back to where you pressed, plays at the speed it had, and the camera moves to the next car from the car you were watching.
5. Jump to Race Start by search, then Camera Controls' Focus on Leader while it searches: back where you pressed, the camera on the leader.
6. A Frame Jog spin on the dial, then Camera Controls' Cycle Camera: the camera group changes once the replay has landed.
7. In the car or spectating live: every Camera Controls key acts at once, as before.

## Affected artifacts

- `@iracedeck/replay-engine`: `camera-switch-waiter.ts` (`interruptsWalks`, the interrupted walk and the open jump sequence as under way), `lap-walk-owners.ts` (`interruptLapWalks`), `replay-cursor.ts` (`interruptedBy` on the claim), `lap-walk.ts` (`LapWalkInterrupted`, the checkpoint, `putReplayBack`'s no-origin case), `fastest-lap-press.ts` and `race-start-press.ts` (the interrupt's give-up path), `lap-press.ts` (the press's log line), `replay-command-gate.ts` (an export for an open jump sequence with a frame), their tests; `index.ts` for the new names; `packages/replay-engine/CLAUDE.md` (the cursor, the walk owners, "One camera policy" widened to every action's camera keys).
- `@iracedeck/iracing-actions`: `camera-controls/camera-controls.ts` (its shared dispatch through `requestCameraSwitch`), `race-admin/race-admin.ts` (`executeFocusSelect`), `replay-control/replay-control.test.ts` (the matrix above), the Camera Controls and Race Admin tests; `packages/iracing-actions/CLAUDE.md` (the sentence that every camera switch of Replay Control's key and dial goes through `requestCameraSwitch` becomes every action's).
- Website:
  - `docs/getting-started/settings.md`, the Replay jump playback paragraph: "Pressing another Replay Control key during a jump cancels it, and that key's command stands" becomes true only of a non-camera key; a camera key waits for the jump to land and does not stop it.
  - `docs/actions/view-camera/replay-control.md`, Jump to Fastest Lap ("a camera key stops only the search and otherwise waits for the jump to land") and Jump to Race Start ("a camera key only while it searches"; "a search stopped this way leaves the camera on the pole car, unless a camera key stopped it"): a search a camera key stops now puts the replay back as it was before the key acts.
  - `docs/actions/view-camera/camera-focus.md` (the Camera Controls page): one sentence that in a replay a camera key pressed while a jump is landing acts once it has landed, and a car key stops a Jump to Fastest Lap or Jump to Race Start search.
  - `docs/actions/view-camera/replay-markers.md`, the Next Marker paragraph ("A Replay Control key press or dial input made while the jump is still landing acts once it has landed"): add Camera Controls' keys.
  - `docs/actions/communication/race-admin.md` is unchanged: it does not describe the Car Selector's focus press (the Camera Controls page's Focus Select Car section does, and the sentence above covers it).
- Changelog: a `Bug Fixes` fragment `changelog.d/1435-camera-keys-during-replay-jump.md` for Camera Controls' and Race Admin's switches during a jump and for the walk put back, both of which shipped in a released version. Replay Control's keys are already `1467-replay-controls-consistent.md`'s, still unreleased, and that fragment stays as it is.
- `docs/superpowers/specs/2026-10-18-issue-1448-replay-dial-camera-car-modes.md`: the one-sentence amendment to rule 5 under *Agreement with #1448*, committed with this spec.
