# Replay Control dial: Lap, Session, Incident and Car stepping

> **Issue:** [#801](https://github.com/niklam/iracedeck/issues/801) · **Supersedes:** _none_ · **Superseded by:** _none_
>
> Point-in-time design record. The code and `.claude/rules/` are the truth; this is not documentation.

Four more modes on the dial that #1316 and #1317 build (`2026-10-04-issue-1316-replay-control-dial.md`). The lap and session steps are #1426's (`2026-10-12-issue-1426-replay-record-steps.md`); this spec says what a turn of the dial does with them, and nothing about how the record is made.

## The decision

The dial's Mode select gains **Lap**, **Session**, **Incident** and **Car**. Everything the surface owns is unchanged and shared: the press and gesture slots, the remembered-speed Play / Pause, the in-replay guard, the throttle and the dash box.

| Mode | A detent | How |
| --- | --- | --- |
| Lap | the viewed car's next / previous lap | #1426's record step, else iRacing's search |
| Session | the next / previous session | #1426's record step, else iRacing's search |
| Incident | the next / previous incident | iRacing's search |
| Car | the next / previous car on track | iRacing's Next Car / Previous Car bindings |

Clockwise is next in all four. Lap, Session and Incident are ordered by time in the recording and Car is iRacing's own Next Car, the car ahead, so none of them is a number-primary rotation and none flips (`encoders-and-touchscreen.md` rule 4).

## Lap and Session: one jump per turn

A turn of `ticks` detents calls #1426's walk with `steps = |ticks|` and makes **one** waited jump to the entry it reaches, through the shared jump sequence (#1276). So a fast spin of five detents is one jump five laps on, and there is no per-event cap: the list's end is the clamp (rule 3).

- **A turn during a jump steps from where that jump is going.** The anchor is the shared pending landing in `replay-cursor.ts`, as on the Replay Markers dial. The new jump takes the cursor from the old one and inherits the speed it captured, which #1276 already handles. This is the shape the sim was measured to take (#1426's sim facts, 2026-10-12): an absolute jump sent during a seek retargets it and the last target wins, so a fast spin needs no queue. The target is never computed from telemetry read mid-seek, which shows frame 0, and the landing stands for as long as its jump holds the cursor (#1426).
- **The record ran out part-way:** the jump goes as far as the record reaches and the rest of the turn is dropped. The next detent finds nothing that way and goes to the search below.
- **The record has nothing that way** (zero steps): the whole turn goes to the paced search.
- **While the search runner is working for this dial's target**, a turn goes to its count without asking the record: the cursor is between searches, and there is no landing to step from (added 2026-10-12).

## The paced search

iRacing's lap, session and incident searches cannot be chained: one sent before the last has landed leaves the cursor part-way (the fastest-lap walk waits for exactly this reason). A dial sends detents far faster than a search lands, so the dial paces its own sends.

**Measured, and the queue stays** (2026-10-12, #1426's sim facts). On a paused replay, three Next Lap 200 ms apart or closer ended in the wrong place: the first landed at once and the other two became one more move about 430 ms later. But one search sent 100 ms after the last had read back landed exactly one step on, 10 of 10, carried out by the sim about 400 ms after the first landed. So the rule is **one search on its way at a time**, and that is a read-back and not a clock. A search sent during a position jump ended the jump where it stood, and so did a pause, so nothing but a position jump is ever sent into a seek.

One runner per process, in `replay-control/replay-search-queue.ts`, because iRacing has one cursor:

1. **A turn adds its signed detents to a pending count** for its target (lap, session or incident). A turn the other way subtracts, and can carry the count through zero. A turn on a dial set to another target replaces the count.
2. **The runner is a replay jump** (`beginReplayJump`), and it holds the cursor. In this order: if it took the cursor from a record jump still on its way, it waits for that jump's frame to read back; then it pauses the replay if it still reads moving, and waits for the pause to read back. The pause never goes into a seek (a pause 170 ms into a jump left the replay 8700 frames off, 3 of 3), and a jump of #1276's has already paused the replay it is crossing. While the count is not zero it sends one search toward zero, waits for it to land (#1426's landing wait), and takes one off the count. Every one of these waits is a read-back, and no step goes out on a timer.
3. **The first search after an inherited landing is tried twice.** A Next Lap sent in the poll that read a long jump's landing was lost, 2 of 2, and nothing in telemetry says when the sim is ready again. So if that first search does not move the cursor, it is sent once more when its landing wait has run out; a second one that does not move ends the run like any other.
4. **It ends** when the count reaches zero, when a search does not move the cursor (the end of the recording, which the sim marks in no other way than by the landing wait running out: the rest of the count is dropped), when a search, the pause or an inherited landing never reads back (a warning, the rest dropped), or when any other replay command takes the cursor. Ending at zero applies the **Replay jump playback** setting through `applyReplayJumpPlayback`; every other ending sends nothing more.
5. **The count is clamped to ±`SEARCH_QUEUE_MAX` (10)** (maintainer, 2026-10-12). The sim carries out a search that follows another about 400 ms after the first landed, so a step is a little under half a second on the maintainer's machine and more on a slower one or in a long replay. Ten is then four seconds or more of the replay moving after the hand has left the dial; a longer spin is more likely a slip than a request. The record path has no such limit because it is one jump whatever the count.

The pause in step 2 is what lets a landing be read at all: in a playing replay the frame never holds still, and a frame that playback moved reads like a search that landed. It is also why the measurements above apply, since every one of the runner's searches goes into a paused replay. It also means the paced search honours the playback setting where the keypad's single search does not; that difference is the keypad's (#1276 leaves its searches as they are) and is not changed here.

## Incident

Always the paced search. #1426 records incidents and compares them with where iRacing's search lands; the mode switches to the record when that issue's follow-up does, for the keys and the dial together.

**The search itself is not steady** (measured 2026-10-12, #1426's sim facts). From one frame with the camera on one car, two Next Incident moved the cursor 0 frames in one run and 600 in the next, and a single one landed behind the cursor, since the search lands some seconds before the incident it finds. The pacing is the same as for laps and adds no error of its own, but a detent is only as good as the key is today: it can land where the last one did, and a clockwise turn can step back. An unmoved search ends the run, as at the end of the recording. This mode is not built before the capture under _Testing_ has been read, with the camera pinned; what it shows may move the mode to #1426's follow-up altogether.

## Car

One tap of iRacing's Next Car / Previous Car binding per rotate event, through the action's binding dispatch (`CAR_CYCLE_BINDING_KEYS`, V / Shift+V by default, SimHub roles included). One tap per event and not per detent: a keyboard tap holds the key for about 100 ms, and a continued spin arrives as further events. This is what Camera Controls' Track Order dial does, and the two dials share the bindings.

A camera switch in the middle of a fastest-lap walk would point its search at another car, so the tap cancels a walk first, as the keypad's car modes do. A jump on its way is neither cancelled nor interrupted (added 2026-10-12): every command measured but a position jump ends a seek where it stands, and whether a camera key does is not measured, so the tap is held until that jump has ended and dropped if the jump stops short. #1430 is the general hazard of one surface's command going into another's jump, and this follows what it settles. It is the dial's only key-binding mode: the PI shows the binding status line for it, and the box draws the missing-binding warning.

## The display

| | Lap | Session | Incident | Car |
| --- | --- | --- | --- | --- |
| Label | `LAP` | `SESSION` | `INCIDENT` | `CAR` |
| Value | the viewed car's lap at the cursor, `L12` | the cursor's session, `RACE` | replay time, `m:ss` | the viewed car, `#12` |
| Caption | play state | play state | play state | play state |
| Side marks | lit where a turn that way has somewhere to go | the same | both lit | both lit |

- **The lap comes from the record, not from `CarIdxLap`.** The maintainer's first choice was the viewed car's lap counter. In an in-session replay the per-car arrays read the live field: four snapshots (2026-09-28 and 2026-09-29) scrubbed back to moments when the camera car was racing all read `CarIdxLap` −1. The record's entry at the cursor is the lap actually on screen, and the maintainer agreed to read it from there (2026-10-12). With no entry for that car at the cursor the value is the replay time, as in Incident.
- **The session name** is `SessionInfo.Sessions[ReplaySessionNum].SessionType`, upper-cased and fitted by the box.
- **While a turn is being worked through** the caption shows what is left, `+3` or `-2`: the paced search's pending count, or nothing for a record jump, which is one step however far it goes.
- **Side marks for Lap and Session** are dim only when the record has entries and none lies that way. With no record both are lit, since the search may still move.
- **A tick read mid-seek is not drawn** (added 2026-10-12). While a seek runs, telemetry shows frame 0 and replay time 0, which would flash the tape's first lap and `0:00` on every turn. During a record jump, Lap and Session show the entry at the pending landing: the lap whose start is the last at or before it, and the session whose recorded start is. On any other tick whose raw `IsReplayPlaying` is false inside a replay, the value stays as it last was.
- The views read the store in memory and wait for nothing: `view` runs on every tick.

## The seam

All additive to what #1316 ships; Speed Control and Frame Jog do not change.

- **`REPLAY_DIAL_MODES`** gains `lap`, `session`, `incident` and `car`, one module each and one row each in `DIAL_MODES`. A stored value an older build does not know reads as Speed Control there (`.catch("speed")`); nothing is migrated.
- **`rotate` may return a promise.** The surface tracks it: it catches a rejection, logs it, and re-renders when it settles. A turn that arrives meanwhile is not dropped, because the modes above take it into the pending landing or the pending count.
- **`ReplayDialModeIo` gains** a waited jump built on #1276's jump sequence (`send` stays the cancelling one-shot every other mode relies on), the record step and the search queue, a binding tap, and `isBindingMissing`. Frame Jog's fix for a fast spin (#1317) adds an absolute jump from the pending landing to `io` first; these modes use what it adds and bring only what is missing.
- **The in-replay guard holds through the dial's own seek** (added 2026-10-12). A spin that keeps retargeting keeps the raw flag false until the last jump lands: fifty jumps 60 ms apart read mid-seek for three seconds in the probe. In a live session that outlasts the one-second grace of the debounced replay state (#1324) and would drop detents as turned from the car; in a saved replay the state's frame reads 0 from then on. While a jump or the search runner holds the cursor, the surface counts the replay as on screen. If #1317's fix has already made it so, nothing is added here; #1429 is the same gap on the Replay Markers dial.
- **`ReplayDialModeView` gains** an optional `bindingMissing`.
- **The comms catalog's `replay-control-dial` entry** gains the four modes: `api` for three, the car-cycle key pair for `car`.
- **Trigger descriptions** come from each mode's `rotateDescription`.

## What rides on iRacing's seeking

Every mode but Car ends in the sim moving its own cursor, and the durations this was designed against were measured on one fast machine (maintainer, 2026-10-12). So each step is gated on what telemetry reads back and never on a delay, every measured duration is taken as a lower bound, and each timeout is a give-up that leaves the replay paused where it is. The website section says it in user terms, in the wording Frame Jog's section settles on (#1317): fast turns ride on iRacing's replay seeking, are less exact than slow ones, and vary with the machine and the length of the replay.

## Out of scope

- **Making the record**, and the keypad's use of it (#1426).
- **Incident steps from the record** (#1426's follow-up).
- **A car-number or race-position mode.** Camera Controls' dial has both.
- **Next / Previous Green** as a stepping target (#1354's designed extension).
- **Changing what the keypad's searches do to playback.**
- **A setting** for the search limit or the pacing; the Fastest Lap Search Delay setting already tunes the landing wait.
- **Ulanzi.** No Ulanzi dial is declared until its dials are verified (rule 7).

## Testing

**Unit:**

- **Lap and Session:** one detent and five detents each send one waited jump to the entry the walk reaches; a turn during a jump steps from the pending landing and sends one more jump, on a tick that reads mid-seek and more than a second after the first; a record that runs out part-way jumps as far as it reaches; zero steps hands the whole turn to the queue; nothing is sent outside a replay.
- **The queue:** a count of three sends three searches, each only after the last has landed and never two on their way, then applies the playback setting once; a turn the other way subtracts and can reverse; another target replaces the count; an unmoved search drops the rest; a search that never lands warns and drops the rest; any other replay command ends it with nothing more sent; the clamp at ±10; the pause first on a moving replay, read back before the first search, and none on a paused one; a runner that took the cursor from a record jump waits for that jump's frame before its first search, and gives up with nothing sent if it never reads; the pause goes out after an inherited landing and never before it; the first search after an inherited landing is sent again once when it does not move the cursor, and no other search is; a Lap turn while the runner works goes to the count and not to the record; two dials turning at once share the one runner.
- **Incident:** every turn goes to the queue.
- **Car:** one tap per event whatever `ticks` is, the right binding each way, a walk cancelled first, a tap during a jump held until the jump has ended and dropped when it stops short, and the missing-binding flag in the view and the PI.
- **Display:** the lap from the record and the replay time without it; the session name; the pending count in the caption; the side marks' three states; a mid-seek tick leaving the value alone, and the pending landing's lap and session shown during a record jump; the dedupe signature (a paused dial pushes once).
- **The seam:** a rejected `rotate` promise is logged and the dial re-renders; a detent arriving after the replay-state grace has run out, while a jump holds the cursor, is still taken; Speed Control and Frame Jog tests pass unchanged; an unknown stored mode parses as Speed Control.
- The comms catalog entry and its freshness test, and the late-input case (a turn after `willDisappear` sends nothing and starts no runner).

**Manual, in an iRacing replay**, on a Stream Deck+ and on the Mirabox knob:

1. **Lap, recorded session:** single detents both ways, then a fast spin; the log's `ticks` and the laps moved must match. Spin again while the first jump is still landing.
2. **Lap, replay recorded without the plugin:** single detents pace through iRacing's search, and a spin of four lands four laps on with the caption counting down. Turn back mid-way and watch it reverse.
3. **Session** in a multi-session event: the same two cases.
4. **Incident:** step through several, and spin past the last one; the dial must stop cleanly at the end of the recording.
5. **Car:** each way, with the binding set, cleared (the warning shows) and bound to a SimHub role.
6. **Playback:** each value of Replay jump playback, with a record jump and with a paced search, once paused and once playing.
7. **Another key mid-search:** press Play / Pause during a paced search; the search stops and the press is obeyed.
8. **From the car:** every mode does nothing and the box is dimmed.
9. **A long spin in a long replay:** in Lap mode, keep the dial turning for three seconds or more in a replay of an hour or longer. The laps moved must match the log's detents, no turn may be refused as outside a replay, and the box must not flash the first lap or `0:00` on the way.

**Captures to read before the parts they decide are built** (#1426's manual step 8 lists them with the rest):

- **The runner's opening on a playing replay:** pause, read back, then three Next Lap each sent when the last has landed. Every search measured so far went into a replay that had been paused for a while, or into one still playing. If the first search after a fresh pause is lost or merged, the runner needs a read-back for "the pause has settled" that this spec does not have yet.
- **Incident searches with the camera pinned:** the same Next / Previous Incident runs from one frame, repeated, with the camera car read on every sample. It says whether the landing follows the camera, and whether the Incident mode is worth building on the search at all.
- **A search after a jump's landing:** Next Lap 0, 100 and 300 ms after a long jump's landing has read back. It says whether the second try in step 3 is the right answer, or whether a read-back exists that makes it unnecessary.
- **A camera key during a jump:** Next Car 100 ms into a position jump. If the jump still lands, the Car tap need not be held.
