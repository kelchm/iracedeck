# Replay record steps: lap and session jumps from the record, session starts and incidents recorded

> **Issue:** [#1426](https://github.com/niklam/iracedeck/issues/1426) · **Supersedes:** _none_ · **Superseded by:** _none_
>
> Point-in-time design record. The code and `.claude/rules/` are the truth; this is not documentation.

Split out of #801, whose dial modes are built on this. It builds on the per-session replay store (#1162), the lap record (#1203), the green record's recorder seam (#1354) and the shared jump sequence (#1276), and restates none of them.

## The decision

1. **Next / Previous Lap and Next / Previous Session jump from the record.** The press finds its target among the frames iRaceDeck recorded and makes one waited jump there. A step the record cannot answer sends iRacing's search, as the key does today.
2. **Session starts are recorded the moment a new session is first seen live**, in the car or out of it, into a new `sessions` section. That is normally the tick the translator already detects the change on.
3. **Incidents are recorded and not yet used.** A new `incidents` section holds each incident at the frame it happened. Next / Previous Incident keeps sending iRacing's search, and a debug line compares where that search lands with what was recorded. Switching the keys over is a later issue, once the comparison has been read (maintainer, 2026-10-12).
4. **One walk rule for every recorded list.** Markers, lap starts and session starts are all a frame-ordered list stepped from the cursor, so they share the Replay Markers rule instead of each growing its own.

## The sim facts this rests on

Measured on 2026-10-12 over the 92 snapshots in the maintainer's `local/` that carry the replay fields:

- **In the car, `ReplayFrameNumEnd − SessionTime × 60` is constant for a session**, to within one frame: five sessions with two to four snapshots each (SubSessionID 86697546 reads 27994 at 73 s and at 133 s). So a session's start frame can be derived from any in-car tick. A negative result is a session that began before the driver joined, whose start is not on the tape.
- **Out of the car in a live session, `ReplayFrameNumEnd` counts the frames from the cursor to the end of the tape**, so `ReplayFrameNum + ReplayFrameNumEnd` is the newest frame recorded. Ten snapshots: at the live edge it reads 1 (five snapshots), and scrubbed back it reads the distance to it (1201 at 21 s back, twice; 27630, 34780 and 40182 further back). In each, `(SessionTime − ReplaySessionTime) × 60 − ReplayFrameNumEnd` is between 57 and 60. This settles the reading `replay-laps.ts` calls unconfirmed, and its comment is updated.
- **`SessionTime` stays live while an in-session replay is scrubbed back**; `ReplaySessionTime` follows the cursor (the same ten snapshots: 1704 s live against 1123 s on screen). Whether `SessionNum` stays live too is not measured, since every one of them is a single-session event. The design does not depend on it (see _Recording session starts_).
- **A frame recorded live sits about 59 frames before the same moment in replay numbering**: the figure above, and 59 exactly in the one session measured both in the car and across its saved file. This is #1162's `REPLAY_FILE_FRAME_LAG`, recorded and not applied.
- **Other cars' incident counts are hidden while an official session runs** (survey of 2026-09-21): `CurDriverIncidentCount` reads −1 for every opponent until the race is over. They were visible in the one hosted capture and are always visible with AI.

Measured the same day by a probe against the maintainer's sim, run for #1317 (a paused saved replay, around frame 15500; added 2026-10-12). His machine is a fast one and slower ones may seek more slowly, so every duration here is a lower bound:

- **A short position jump after a quiet replay lands in one sim update** (10 to 20 ms, a jump 12000 frames back included). **One sent less than about 500 ms after another takes a slow path**, and so does a long one by itself (60000 frames forward, twice, about 490 ms). On that path telemetry reads `ReplayFrameNum` 0, `ReplaySessionTime` 0, `IsReplayPlaying` false, `SessionNum` and `ReplaySessionNum` −1 and `ReplayFrameNumEnd` the whole tape's length (all 57 mid-seek rows of the trace) until the cursor lands. For two jumps that was 415 to 435 ms after the first was sent, whatever the gap between them.
- **During a slow seek an absolute jump retargets it and the last target wins** (10 of 10 at gaps of 20 and 60 ms). Fifty jumps 60 ms apart landed on the last one's frame, and telemetry read mid-seek for the whole three seconds. A jump relative to the cursor resolves from the start of the tape, and a frame step ends the seek wherever it has got to.
- **Out of the car at the live edge the raw `IsReplayPlaying` reads true** (the five edge snapshots above), so a tick that reads it false there is a seek under way.

And the searches, from the same probe's runs on that saved replay of 204563 frames: 30 cases, then 102, then the playing set again with its gaps corrected, then 47 for the captures this spec had named (added 2026-10-12):

- **A search after a quiet replay lands in 13 to 200 ms, with the old frame on screen until the new one.** No frame 0 on the way and no drop of the flag, except that a Previous Lap whose landing was frame 0 read one mid-seek sample first. An incident search moves the camera to the car involved, one sample after the frame.
- **Two searches on their way at once do not add up; one at a time does** (paused). Three Next Lap 200, 60 or 20 ms apart: the first landed at once and the cursor moved once more about 430 ms after the first was sent, to two laps on once and otherwise to a frame that is no lap start at all, in both runs. But a second search sent 100 ms after the first had read back landed exactly one lap further, 10 of 10 (Next Lap and Previous Lap, five each), carried out about 400 ms after the first landed; and two or three Next Lap 300 to 500 ms apart were right every time. Telemetry shows nothing in between that says a search is waiting.
- **On a replay playing at 1x** a second lap search about 115 ms after the first landed two laps on, 12 of 12 over two runs, and so did two Next Lap 300, 350, 400 and 450 ms apart, 4 of 4. Two sent inside one sim update counted as one, 4 of 4. Three in a burst were not tried there.
- **Incident searches did not repeat.** From one frame with the camera on one car, two Next Incident ended 0, 600 or 900 frames on from case to case, the pair sent 100 ms apart alternated between 0 and 600 from run to run, and a single one landed 300 frames behind the cursor: the search lands some seconds before the incident it finds. Putting the camera on one car before every search changed nothing: from one frame, five single Next Incident landed 300 frames back, on the frame, 300 and 600 frames on, and 300 back again. Each search stepped on from the last one's landing wherever the cursor had been put since, which reads as iRacing keeping an incident pointer of its own (an inference from the pattern). Searches sent one after another always stepped the same way.
- **A search sent in the poll that first reads a landing can be lost.** After a long jump's landing: lost 3 of 3, where one sent 100 or 300 ms later landed, 6 of 6. After a first search's landing: taken 3 of 3; after a second's: lost 2 of 3. A search sent 100 ms after the reading worked every time it was tried, 16 of 16; nothing between 0 and 100 ms was measured.
- **The opening #801's runner will send works:** on a replay playing at 1x, a pause, its read-back held 60 ms, then Next Lap, landed 6 of 6.
- **A camera switch does not end a seek**, unlike every replay command tried: sent 100 ms into a position jump, long or on the slow path, the jump landed on its frame, 6 of 6. That was the camera broadcast; the Next Car key binding was not measured.
- **A search sent during a position jump ends the jump where it stands**: on the start frame, half-way, and five frames short twice, in the four cases. An absolute jump sent 20 or 200 ms after a search won, 8 of 8 over the two runs. A pause sent 170 ms into a jump, with nothing after it, left the replay about 8700 frames off, 3 of 3: every replay command but a position jump ends a seek where it stands.
- **A search with nothing to find does nothing and shows nothing** (Next Session in the last session), so only a timeout tells. Next Lap in a car's last lap goes to the tape's last frame once, and then does nothing.
- **Not measured:** anything between 0 and 100 ms after a landing is read, the Next Car key binding during a seek, and where an incident search lands after the cursor has been moved a long way with no search in between. They are step 8 under _Testing_.

## Stepping through a recorded list

A list is frame-ordered **landing frames**: the frame a jump to that entry sends. The walk is Replay Markers' (`replay-markers-ops.ts`), moved to `iracing-actions/src/shared/` so Replay Markers and Replay Control call the same functions:

- **The anchor** is the pending landing while it stands, otherwise the cursor (`resolveAnchorFrame`). A second press during a jump therefore steps from where the first is going. Two things keep a seek's own telemetry out of the anchor, since a tick read mid-seek shows frame 0 and a walk from there would send the tape's first entry (amended 2026-10-12). A landing stands for as long as the jump that recorded it holds the cursor, however long the seek takes, and only after that by the marker rule (pressed less than a second ago, the cursor not yet within 60 frames of it): the second alone is a guess about the machine. And the cursor is the controller's debounced frame (`getReplayState().frame`, #1324), which keeps the last frame a replay tick showed while the raw flag reads false, for the second its grace lasts. A press that has neither, no standing landing and a tick still reading mid-seek once the grace is over, sends nothing and says so at debug: there is no cursor to step from, and a search sent into a seek is not known to be safe.
- **Next** is the first landing more than 60 frames past the anchor. **Previous** goes back to the entry the anchor is in when the anchor is more than 120 frames past it, and to the one before otherwise. These are the marker windows, unchanged.
- **`steps`** is how many entries to move, at least 1. The keys pass 1; #801's dial passes the detents of a turn. The walk stops at the end of the list and reports how many steps it took. Zero steps is a miss.

**Lap.** The list is every recorded lap start of the car the camera is on (`CamCarIdx`, verified by `carNumberRaw` like every lap lookup), across all sessions of the file. Sessions follow each other in frames, so stepping past a session's last lap reaches the next session's first without a rule for it. A landing is the lap start minus `LAP_START_APPROACH_FRAMES` (60), the lead-in Jump to Fastest Lap already uses. The store gains one reader, `laps.listLapStarts({ carIdx, carNumberRaw })`, and no format change; starts within 60 frames of each other are one entry, so a lap that two recorded sessions both hold is one step.

**Session.** The list is the `sessions` section. A landing is the recorded start plus `SESSION_START_LANDING_FRAMES` (120): with the 59-frame lag, the raw frame lands in the last second of the session before, where the cursor would still read the old session and a second press would find the same start again.

**A miss sends iRacing's search**, the call the key makes today, and logs `record MISS (<reason>)` at debug. The reasons are the store's (`no file`, `no car`, `car mismatch`, `newer format`) plus `nothing that way`. The last one matters: the record ends where the driver left the car, and a step beyond it must still move the replay. Presses on a miss go out as they are pressed, as today, so three in quick succession can end in the wrong place (see the sim facts); sending one at a time is #801's search runner, which the keys do not use here.

## The jump

`beginReplayJump("replay-control-<mode>", telemetry)` at the press, `recordReplayLanding(frame, now)` as the seek goes out, `runReplayJump`, and `jump.end()` in a `finally`. That gives the press everything #1276 decided: the replay is paused for the seek, nothing follows it until `ReplayFrameNum` reads the frame, and playback afterwards follows the **Replay jump playback** setting. Claiming the cursor cancels a Jump to Fastest Lap walk in flight, which is what these modes do today through `cancelFastestLapWalk`. The outcome updates the action's local speed, so the Play / Pause icons follow. A jump that ends any other way than landed drops its own pending landing, since the replay is no longer headed there.

**Nothing here waits on a clock** (maintainer, 2026-10-12). Each step goes out on what telemetry reads back: the seek once the pause shows, the playback step once `ReplayFrameNum` reads the frame that was sent. A timeout is a give-up that sends nothing more and leaves the replay paused where it stopped. Seeking is the sim's own work and was measured on one fast machine, so the Replay Control page says in user terms that quick presses ride on iRacing's replay seeking and take longer on a slower machine or in a long replay, in the wording the Frame Jog section settles on (#1317).

No camera switch: the lap list is the viewed car's, and a session belongs to everyone. Outside a replay nothing is sent, as now. The four modes stay `api` in the comms catalog.

## Recording session starts

The translator already detects a session change on the tick it happens, and the Race Engineer's brief hangs on that. The record is written at the same moment, by a rule that does not need the edge:

**A session's start is `live frame − round(SessionTime × 60)`, clamped at 0, on the first tick a new `(sessionNum, sessionUniqueId)` is seen live.** On the change tick `SessionTime` is near zero and the start is the live frame. Seen later (the plugin started mid-session, the driver was scrubbed back in a replay when the session rolled over), the same sum gives the same frame, which is what the in-car measurement above shows. So there is one rule and one kind of entry, and no separate path for a change that was missed. A start that would be negative is a session that began before the tape did, and is the tape's start.

**Which ticks are live.** `replay-record-gate.ts` gains a second resolver beside `resolveReplayRecordTick`, which is left exactly as #1354 wrote it and is still the only gate the lap and green recorders use. The new one returns the same shape for two cases:

- **In the car:** `resolveReplayRecordTick`'s own answer, with the frame `ReplayFrameNumEnd`.
- **Out of the car at the live edge of a live session:** the controller's debounced replay state says a replay, the tick's own `IsReplayPlaying` reads true, the session is not replay-only (#604's `SimMode`), and `ReplayFrameNumEnd` is at most `LIVE_EDGE_MAX_FRAMES` (120). The frame is `ReplayFrameNum + ReplayFrameNumEnd`. The raw flag is the seek guard (added 2026-10-12): a tick read during a slow seek shows `ReplayFrameNum` 0 and `SessionNum` −1 with the flag false. On the measured tape `ReplayFrameNumEnd` read the tape's whole length on such a tick, which the edge limit refuses anyway, but that is one tape. The debounced state only picks the case; nothing is read from a tick the flag does not vouch for.

The live-edge condition is what makes a scrub harmless. If `SessionNum` turns out to follow the cursor in an in-session replay, scrubbing back across a session boundary would otherwise look like a new session seen from the garage. Scrubbed back, the cursor is not at the edge and the tick is not live, so nothing is read from it; the session is recorded on the first tick the driver is back at the edge or in the car, at the same frame. The module comment says there are two classes of tick and which recorder uses which: this one needs only the session's identity, its clock and a frame, and no per-car array.

**Two ticks must agree.** The recorder, `diff/replay-sessions.ts`, runs before the replay guard beside the others. It publishes a session's start once two consecutive live ticks give the same frame to within two frames, so a tick on which the session number has moved and the clock has not cannot put a wrong start in the file. Agreement does not catch a seek, whose readings hold still for a quarter of a second or more; the raw-flag condition above is what keeps those ticks out. It logs the tick's frame and `SessionTime` at debug when it publishes.

```typescript
"replay.sessionStarted": SimEvent<
  "replay.sessionStarted",
  {
    /** `WeekendInfo.SubSessionID`; 0 for an offline session. */
    subSessionId: number;
    sessionNum: number;
    sessionUniqueId: number;
    /** The live replay frame the session began at; 0 when it began before the recording. */
    frame: number;
  }
>;
```

The `sessions` section holds one entry per pair, in frame order, and the first one recorded stays. A restarted offline event puts the same `sessionNum` on the tape again under a new unique id (maintainer, 2026-10-12): that is a new entry further along the tape, and since every step is by frame, nothing here looks a session up by its number.

## Recording incidents

Both sources sit in one recorder, `diff/replay-incidents.ts`, before the replay guard on #1354's in-car gate, unchanged.

- **The driver's own car** from `PlayerCarMyIncidentCount`: a rise over the previous eligible tick publishes `replay.incident` with that tick's frame. No burst logic is copied from the Race Engineer's incident diff. The store folds a rise within 600 frames of the same car's last entry into that entry (ten seconds, #1122's sequence gap), so one crash is one entry at its first frame.
- **Other cars** from the session YAML's `DriverInfo.Drivers[].CurDriverIncidentCount`, where the sim shows it: a rise from a non-negative baseline. A −1 is no baseline, and a step from −1 to a number (the reveal after an official race) re-seeds silently, because the frame it belongs to is long past. A decrease re-seeds too (a driver swap). The YAML lags the incident by an unmeasured amount; the comparison below is what measures it.

```typescript
"replay.incident": SimEvent<
  "replay.incident",
  {
    subSessionId: number;
    sessionNum: number;
    sessionUniqueId: number;
    carIdx: number;
    carNumberRaw: number;
    /** The live replay frame on the tick the count rose, recorded raw. */
    frame: number;
    sessionTimeMs: number;
    /** How far the count rose on that tick. */
    delta: number;
    /** The driver's own counter, or another car's count in the session YAML. */
    source: "telemetry" | "sessionInfo";
  }
>;
```

**Stored at the moment, offset at the jump** (maintainer, 2026-10-12): the entry keeps the incident's own frame, and the later consumer subtracts five seconds (300 frames) as a constant that can change without touching a file. No setting.

**The comparison.** After a Next / Previous Incident press, an observer claims the replay cursor, waits for the search to land, and logs at debug the frame and camera car at the press, the frame it landed on, the camera car as the later of its two reads shows it (the search moves the camera one sample after the frame), the nearest recorded incident and the difference in frames, or that none is recorded. It sends nothing, and any other replay command cancels it. Landed means the frame has jumped, by more between two reads than playback at the speed read explains, and then two reads 50 ms apart, both out of a seek (the raw `IsReplayPlaying` true and `SessionNum` not −1), show frames that differ by no more than playback explains. That is the fastest-lap walk's `search` rule widened from a paused replay to a playing one: paused, any change is a jump; playing, the frame leaves its value by itself at once, so leaving it proves nothing (the probe's own read-back test was met by playback this way). A search that moves the cursor less than playback would is not seen, and is logged as not landed. The incident search steps a pointer of its own and not from the cursor (see the sim facts), so the comparison is between where it landed and the recorded incident nearest that frame, never the one a walk from the press's cursor would have picked; the frame at the press is logged so that the difference between the two can be read as well. The walk already refuses a sample whose `SessionNum` is −1, which is what a mid-seek tick reads beside frame 0; the flag is named with it (2026-10-12) so that frame 0 holding still for a third of a second can never pass for a landing. The Fastest Lap Search Delay is a floor under that read-back, as in the walk, and never the wait itself. A search that has not landed by the walk's timeout is logged as not landed and compared with nothing. The wait moves to `shared/replay-seek.ts`, where #801's paced fallback will use it too.

## The store

Two more sections, `sessions` and `incidents`, each with its own module of pure logic and its own `version: 1`, added to `replay-session-store.ts` the way #1354 added `greens`: parallel to the others, changing none of them. Both follow the same compatibility rules: the envelope version does not move, an older build carries the sections through untouched, a section with a higher version is carried through and neither read nor written, and unknown fields survive a write. The unreadable-file merge keeps the file's entry where both have one.

**`no file` keeps its meaning.** `findLapStart` and `findGreen` answer `no file` for a record that was not loaded and holds neither laps nor greens. A record holding only a session start still answers that to them: nothing of theirs was recorded.

Four sections each carrying a parallel set of fields and branches is where a shared shape starts to pay. It is not done here, because it would rewrite two shipped sections inside a feature change; it is a hygiene issue to file once this has merged.

## Out of scope

- **The dial** (#801): the stepping modes, the multi-detent turn and the paced search behind a miss. This issue only makes `steps` a parameter.
- **Jumping to a recorded incident**, and anything that reads the `incidents` section.
- **Other cars' incidents in official sessions.** The sim does not show them while the frame is known.
- **Recording laps or greens from outside the car.** The live-edge frame would allow it, and a spectator's lap record is empty without it, but those recorders read per-car arrays and flags whose behaviour out of the car is its own question (`replay-laps.ts`). A later issue.
- **A Next / Previous Green pair** (#1354's designed extension).
- **A setting** for any lead-in or landing offset.
- **The legacy Replay Navigation action** (#1334). It gains nothing.
- **Generalising the store's per-section code** (see _The store_).

## Testing

**Unit**, in the package that owns each piece:

- **The shared walk:** next and previous under the marker windows, `steps` above 1, stopping at either end with the count taken, zero steps as a miss, the anchor taken from a pending landing, a landing still anchoring after more than a second while its jump holds the cursor, the cursor taken from the debounced frame on a tick that reads mid-seek, and nothing sent when there is neither. Replay Markers' existing walk tests pass unchanged against the moved functions.
- **Replay Control** (deck-core mock, mocked store): each of the four modes on a hit sends one waited jump to the landing frame and no camera switch; the lap landing is the start minus 60 and the session landing the start plus 120; a second press during a jump steps from the pending landing; a jump that stops short drops its landing; every miss reason sends the sim's search once and nothing else; outside a replay nothing is sent; the playback step follows each value of the setting.
- **`replay-record-gate.ts`:** the new resolver in the car, out of the car at the live edge, out of the car scrubbed back (not live), in a replay-only session, and on a tick that reads mid-seek (frame 0, raw flag false) whatever `ReplayFrameNumEnd` says; `resolveReplayRecordTick` is untouched and its tests pass unchanged.
- **Session starts:** a session first seen on its change tick, in the car and out of it, publishes one event at the live frame; first seen ten minutes in, it publishes the same frame; a start before the tape publishes 0; a single tick whose clock disagrees with the next publishes nothing until two agree; a scrubbed-back replay across a session boundary publishes nothing, and the session is published once the cursor is back at the edge; a replay-only session publishes nothing; a pair already published is not published again; a restarted event's second instance of a session number publishes a second start, later on the tape. The recorder's position in `handleTick` is pinned as pre-guard. A fixture is cut from the 2026-09-28 and 2026-09-29 snapshots for the out-of-car frame.
- **Incidents:** the first tick seeds; a rise publishes with that tick's frame; an ineligible window re-seeds; for other cars, −1 is no baseline, −1 to a number is silent, a decrease re-seeds, and the player's car is not published twice.
- **The landing wait:** a frame that jumped and held with the flag true is a landing, paused and playing; a playing replay's own advance is not a jump; frame 0 held with the flag false is not a landing; a search that never lands times out with nothing compared.
- **The store:** record and list for both sections; a session's first start staying when a second arrives; the 600-frame fold; a newer version carried through; unknown fields kept; every other section surviving a write; the unreadable-file merge; the offline record never reaching disk; `findLapStart` and `findGreen` still answering `no file` for a record with only a session start.
- **Catalog and harness:** both events compile in `event-names.ts`, and the settings-phase test holds each to its section.

**Manual, in the sim**, with the log at debug:

1. **Lap keys, record path.** After a race driven in the car, in the in-session replay: Next Lap and Previous Lap land a second before the viewed car's line, from near and from more than 20000 frames away, once paused and once playing. Two quick presses land two laps on, and five as fast as the key allows land five laps on, each jump measured from the one before it in the log.
2. **Session keys, and the capture for `SessionNum`.** An event with practice, qualifying and a race, out of the car as each session ends. The file must hold one start per session, and Next / Previous Session must land just inside each. Then, during the race session and out of the car, scrub back into qualifying with a Telemetry Display key showing `{{telemetry.SessionNum}}` beside `{{telemetry.ReplaySessionNum}}`: record in this spec whether `SessionNum` followed the cursor. Either way, no new entry may appear in the file.
3. **Fallback.** A replay recorded without the plugin: all four keys behave as they do today, with `record MISS (no file)` in the log. In a recorded session, Next Lap past the last recorded lap logs `nothing that way` and still moves.
4. **Replay jump playback.** Each of the three values, with a lap step and a session step.
5. **Incidents.** An AI race with at least one incident of the driver's own and one of another car: the session file holds both, the driver's at the frame the count rose. Step through them with Next Incident and read the comparison lines. **These are the capture for the YAML lag and for the later decision.**
6. **Official session.** After an official race, the file holds the driver's own incidents and no other car's.
7. **Saved file.** Reopen a saved replay of a recorded session: the lap and session keys land where they did in the session, about a second earlier.
8. **What the search captures left open.** The probe's runs are recorded under the sim facts (2026-10-12), the captures this step first named among them. Still to see by hand, and to record there: Next Incident after scrubbing a long way with no search in between (does it find the incident after the cursor, or step on from its last one?), and the Next Car key pressed during a long jump (the jump must still land). Neither holds up the build.
