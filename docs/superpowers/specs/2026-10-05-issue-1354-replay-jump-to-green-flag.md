# Jump to Green Flag from the session record

> **Issue:** [#1354](https://github.com/niklam/iracedeck/issues/1354) · **Supersedes:** _none_ · **Superseded by:** _none_
>
> Point-in-time design record. The code and `.claude/rules/` are the truth; this is not documentation.

This builds on the per-session replay store (#1162) and the lap record (#1203, landing fix #1275). The sim facts settled there are not restated here: live frames are `ReplayFrameNumEnd`, a saved `.rpy` keeps the live `SubSessionID` and lands live-recorded frames about a second early, offline sessions are held in memory only, and nothing may follow a seek until `ReplayFrameNum` reads the target.

## The decision

1. **The race goes green at one frame, and iRaceDeck records it live.** A new translator diff publishes `replay.greenFlag` when a race session leaves a real pre-green state for `Racing`, and on every caution restart. The per-session replay store (`@iracedeck/replay-store`, its own package since #1367) keeps those moments in a new `greens` section of the per-session file.
2. **Jump to Green Flag is a lookup first.** The press resolves the target race session, finds its recorded start, and makes one waited jump to five seconds before the green, then plays. It switches no camera: the green belongs to the whole field, and the user has already chosen which car to watch.
3. **The fallback is the pole car's lap 1.** With no green record, the press jumps to five seconds before the pole car starts lap 1. That frame comes from the lap record when there is one, and from the #1203 walk otherwise. The result is close to the green, and before it except in the edge cases listed below.
4. **Restarts are recorded but not yet jumped to.** The one key ships for the race start, which is what the request says. Restarts go into the file from day one, because a replay's restarts cannot be recorded after the fact. A Next Green / Previous Green pair is the designed extension, for a later issue if it is ever asked for (see _Restarts_ below).

## What "the green" is

### Race start: `SessionState` entering `Racing` from a real pre-green state

The translator already treats `SessionState` as the authority on whether a race has started. `isPreGreen` in `iracing-sdk` is `GetInCar | Warmup | ParadeLaps` (plus `Invalid`). The fresh-connect briefing treats `Racing` in a race session as "too late" for the same reason. The pace-laps diff records that on a 1-lap rolling formation "the green flies at the line / `SessionState` flips to `Racing`". So the race start is **the first eligible tick of a race session on which `SessionState` reads `Racing`, when the previous eligible tick read `GetInCar`, `Warmup` or `ParadeLaps`.**

- **`Invalid` is not a pre-green state for this edge**, although `isPreGreen` counts it. An `Invalid → Racing` step is what a reconnect or a session-YAML reload mid-race looks like. Recording it would put a mid-race frame in the file as the start. So the diff uses its own three-state set and does not call `isPreGreen`.
- **One start per session instance; the latest one wins.** A re-grid re-enters `ParadeLaps`, and the rolling-start diff already treats that as a fresh edge. Its green is the real start, so a second start entry for the same `(sessionNum, sessionUniqueId)` replaces the first.
- **Race sessions only**: the session YAML's `SessionType` for the tick's `SessionNum` is exactly `Race`, read through the translator's `resolveSessionType`. Practice and qualifying also show `Racing` and a green bit, but a green there is not a start anyone reviews. A `Warmup` session goes green too and is not a race, which is why this is not the Race Engineer's "anything that is not practice or qualifying". A session whose type is not known yet (the YAML not parsed) records nothing. Heat sessions are assumed to report `Race` like any other (maintainer, 2026-10-12); the action's target rule reads the same field, so a session a press targets is one the record can hold.

**Rejected: a flag-bit edge (`StartGo` for standing starts, `Green` for rolling ones).** The flag diff's comments show how much the bits need to be told apart. `StartGo` also rises on every caution restart, `Green` rises under a yellow-checkered finish, and a green can lead `StartGo` by a tick. The start-light and flag diffs handle all of that with the caution phase, which is post-guard state. `SessionState` has a single meaning here and needs none of it. Timing is not the issue either way: a few ticks between the state flip and the bit are small against a five-second lead-in. **The capture that confirms the model:** on one standing and one rolling start, the diff's debug line on the edge tick logs `SessionFlags`. The offset to the `StartGo` (standing) or `Green` (rolling) rising edge must be a few ticks at most. If `Racing` trails the green by more than a second on either start type, the start edge moves to the first of the state flip and the start bit within the race session, and this spec is amended.

**Measured, 2026-10-12, on the 2026-09 watch captures:** the offset is zero ticks on both start types. The standing start (road course, 2026-09-18) goes `Warmup → Racing` with `StartGo` and `Green` rising in the same record of the watch, session tick 3603 at 56.98 s, and never passes through `ParadeLaps`. The rolling start (oval, 2026-09-17) goes `ParadeLaps → Racing` with both bits rising on the same record, at 201.18 s. Both are fixtures replayed through the diff in `replay-greens.captures.test.ts`. The captures carry no replay frame and no session YAML, so the frame the sim reports on that tick and where a jump to it lands are still the manual test's to measure.

### Restart: the green that ends a full-course caution

A restart is **a `Green` rising edge on an eligible tick of a race session in `Racing`, with neither `Caution` nor `CautionWaving` set on that tick, after an eligible tick in the same session on which one of them was set.** This is the definition `caution.restarted` documents, and a green rising with `Caution` still set (a yellow-checkered tick) is not one. The predicate is shared with `diff/caution.ts`, not copied. The "a caution is out" latch is the recorder's own, because the caution phase it would otherwise read is wiped on every replay tick. A latch cleared by an ineligible window simply misses that restart: an absent entry, never a wrong one.

**As built, the latch lives only while a caution bit is set:** it says whether `Caution` or `CautionWaving` was set on the previous eligible tick of this session, so a restart is the green rising on the very tick the caution bits drop. That is what `caution.restarted` announces, and it is what all three captured restarts show (two on the oval of 2026-09-17, one on the road course of 2026-09-18; the test asserts the count, so it cannot pass on a capture without restarts). The orderings nobody has measured, the bits dropping a tick before or after the green rises, record nothing. A latch left standing past the caution would catch them, at the cost of having to be cleared by something other than the flags, and is not built.

**A tick is one edge at most, and the start wins.** A green rising on the tick the state enters `Racing` is the start's own green, whatever the flags read on the formation lap.

A restart after a red flag is recorded only if iRacing raises a caution bit between the red and the green. That is not designed for; see _Out of scope_.

## Detection, in the translator

`packages/sim-events-iracing/src/diff/replay-greens.ts`, with its state in `TranslatorState`. It runs **before** the replay guard, next to `diffReplayLaps`, for the reason `replay-laps.ts` gives: the guard's early return would stop it on exactly the ticks where it has to notice it cannot see. Its state is left in `wipeStateForReplay`'s wiped set, because a re-seed is what returning from a replay needs.

- **The same gate as the lap record.** A tick is eligible when it is not in a replay, `SessionNum >= 0` and a frame is readable. "Not in a replay" is the translator's `inReplay`, the controller's debounced replay state (#1324): false only once `IsReplayPlaying` has read false for the exit grace, and never in a saved replay. Every ineligible tick marks the recorder unseeded and drops the caution latch, and the first eligible tick after that re-seeds without emitting. A driver who sat in the garage through the green (where iRacing reports `IsReplayPlaying: true`) therefore gets no start entry and the press falls back. The eligibility check is extracted from `replay-laps.ts` into one function both diffs call (`resolveReplayRecordTick` in `diff/replay-record-gate.ts`), so the two records can never disagree about which ticks are live.
- **`frame` is `resolveReplayFrame(telemetry)` on the edge tick**: `ReplayFrameNumEnd` under that gate, recorded raw like a lap start (#1162: no lag correction).
- **`sessionTimeMs` is `SessionTime` on the same tick**, in whole milliseconds. Nothing reads it in this design. It is stored because it is the one frame-free coordinate of the moment: a reader of the file can check a frame against it, and a later `ReplaySearchSessionTime` path would need no format change. It costs eight bytes per green.
- **A session change** (`SessionNum` or `SessionUniqueID` moves) re-seeds, and the caution latch is read afresh from the new session's flags.
- **#1324 exposure: closed by the gate.** `IsReplayPlaying` reads false for about 300 ms after every replay seek, and when this spec was written the gate read that raw flag, so it opened briefly during replay review. #1324 has since given the controller a debounced replay state and moved the lap record's gate onto it, and this diff shares that gate: a seek's blip tick is not eligible, so it can neither seed nor forge an edge. Whether session-level telemetry (`SessionState`, `SessionFlags`) follows the replay cursor during an in-session replay is still not measured (the per-car arrays read the live field, 2026-09-29, #1281), but nothing here depends on the answer any more.

### The event

```typescript
"replay.greenFlag": SimEvent<
  "replay.greenFlag",
  {
    /** `WeekendInfo.SubSessionID`; 0 for an offline session. */
    subSessionId: number;
    sessionNum: number;
    sessionUniqueId: number;
    /** The race start, or the green that ends a full-course caution. */
    kind: "start" | "restart";
    /** The live replay frame on the green tick, recorded raw. */
    frame: number;
    /** `SessionTime` on the same tick, whole milliseconds. */
    sessionTimeMs: number;
  }
>;
```

The green flag is a racing idea, not an iRacing one, so the event fits the sim-agnostic catalog the same way the lap events do. Whether it should be one event with a `kind` or two events: one, because the store treats both the same way and a consumer that only wants starts filters on one field. The scenario harness gets an `event-names.ts` entry. No callout uses it, so there is no shortcut button.

**Rejected: subscribe the store to the existing `caution.restarted` / `startLight.start-go.raised` / `flag.green.raised`.** Those events are shaped for the Race Engineer: suppressed while another family owns the moment, standing-only or in-car-only, and post-guard. Recording from them would tie the replay record to audio decisions. For example, `flag.green.raised` is deliberately silent on a rolling start's green when `StartGo` is up. They also carry no frame.

## The `greens` section

```json
"greens": {
  "version": 1,
  "sessions": [
    {
      "sessionNum": 2,
      "sessionUniqueId": 4,
      "greens": [
        { "kind": "start", "frame": 30215, "sessionTimeMs": 512433 },
        { "kind": "restart", "frame": 71904, "sessionTimeMs": 1199870 }
      ]
    }
  ]
}
```

Entries are kept in frame order, with at most one `start` per session (latest wins, as above). Restarts are de-duplicated by frame within the store's existing 60-frame marker window, so a tick-level double emit cannot create two entries.

**Lookup** (`greens.findGreen({ subSessionId, sessionNum, sessionUniqueId?, kind: "start" })`) follows the laps rule. It matches the `(sessionNum, sessionUniqueId)` pair first. When no `sessionUniqueId` is given or none matches, it uses the unique session with that `sessionNum` that has a start, and when several do, the one with the highest frame (the latest instance in the recording). A miss carries the reason the action logs: `no file`, `no session`, `no green`, `newer format`.

**The record is fed from the bus.** `plugin-runtime`'s settings phase, which already subscribes the store to `replay.lapStarted` and `replay.lapTimed`, subscribes it to `replay.greenFlag` and calls `greens.record`. Neither the translator nor the store knows the other.

**The write path is the store's**: the debounce, the atomic replace, the flush on session change and exit. It also follows the unreadable-file merge. When a file that could not be read becomes readable, the in-memory greens merge into it: the file's start entry stays where both have one, and restarts are a union under the de-dupe rule.

### Compatibility of the persisted file

The replay file is persisted user data, so this is a contract change, and it is additive only.

- **The envelope does not move.** `REPLAY_FILE_VERSION` stays as it is, and `greens` is a new section with its own `version: 1`, like `laps`.
- **Older builds (3.4, 3.5) carry `greens` through untouched.** The store preserves every section it does not read on every write. A file opened alternately by 3.5 and 3.6 loses nothing.
- **This build reads older files.** A file with no `greens` section is a miss with reason `no green`, and the press falls back to the lap record, which those files have.
- **Newer builds' sections are carried through, not written.** A `greens` section whose `version` is above this build's is treated like a newer `laps` section: carried through verbatim, `record` returns false with one warning, and `findGreen` misses with `newer format`. An entry with an unknown `kind` and any unknown field on an entry or a session are kept on write (index signatures) and ignored by the lookup.

## Which session the press targets

1. **The cursor's session, if it is a race.** That is `SessionNum` and `SessionUniqueID` from the replay's telemetry, classified by the `SessionType` of the `SessionInfo.Sessions` entry whose `SessionNum` field matches (not by array position), with the recorder's strict reading: exactly `Race`. In a heat-race event, pressing inside a heat goes to that heat's green.
2. **Otherwise the last race session in `SessionInfo.Sessions`.** In a single-race event, this is the race whether the cursor is in practice, qualifying or a warmup. In a heat-race event it is the feature, which is the one a reviewer is most likely after. No `sessionUniqueId` is known for it, so the lookup goes by `sessionNum`.
3. **No race session at all**: the press does nothing, and the key shows Replay Markers' unavailable look (grey colour slots, faded artwork). This is the only case that greys the key, and it is judged from the session list alone, so the key redraws when a replay of another session is opened. With no session list at all (not connected, the YAML not parsed yet) the key keeps its plain look: grey whenever the sim is closed would read as broken. A missing record does not grey it either, because the fallback can still land. From the car the key does not grey either: it does nothing, as every other Replay Control jump does there, and a key that is grey all race would read as broken.

The other press guards are the ones Jump to Fastest Lap uses, in the same order. `SessionNum < 0` (the post-seek transient) ignores the press. A press while a fastest-lap walk, a record jump or another green jump is in flight is ignored before any command is sent, through the same in-flight slot. Outside a replay nothing is sent, because iRacing ignores replay commands from the car; like Jump to Fastest Lap since #1324, that reads the controller's debounced replay state, not the raw `IsReplayPlaying`, which is false for about 300 ms after every seek.

**Session restarts in an offline AI event** (maintainer, 2026-10-12). An AI event can be restarted as a whole, so one recording can run Practice, Qualifying, Race and then Practice, Qualifying or the start of the Race again; hosted and official sessions never do. Each pass is its own `SessionUniqueID` under a `SessionNum` seen before, which is why the record is keyed by the pair: three race passes are three starts, and none replaces another. A press inside one pass goes to that pass's green (rule 1 carries the pair). A press from outside any race (rule 2) knows no unique id, and takes the latest recorded start for that `SessionNum`, the one with the highest frame.

## The jump

**On a record hit:** the waited jump from `shared/replay-seek.ts` (#1275) to `max(0, frame − GREEN_APPROACH_FRAMES)`, holding the replay-cursor claim, then the playback step. `GREEN_APPROACH_FRAMES` is 300 (five seconds at 60 frames per second):

- **Standing start:** about the moment the gantry reaches Set, so the lights going out is in view.
- **Rolling start or restart:** the field bunched and approaching the line or restart zone.
- **Saved file:** the frame lag makes it about six seconds, which is still before the green.

Fastest Lap's 60 frames show a car crossing a line. Here the run-up is the point, which a single second does not show. It is a constant, not a setting. If the manual test shows the run-up is too short or too long, it is changed in one commit.

**Playback** is whatever the shared jump sequence applies. #1276 merges first (agreed between the two sessions, 2026-10-12), so the mode is written against its sequence: the press starts a replay jump before any command, the jump lands through the shared pause, seek and landing wait, and the **Replay jump playback** setting decides whether the replay then keeps the speed it had, plays at 1× or stays paused. The mode never sends a play command of its own, and the setting's list of covered jumps gains this one.

**On a miss:** the log line `Jump to green: record MISS (<reason>); falling back to lap 1 of the pole car`, and then the fallback.

## The fallback: the pole car's lap 1

- **The car:** the pole sitter from `QualifyResultsInfo.Results` (`Position` 0), read through `grid-utils`. iRacing publishes that grid for standing and rolling starts alike, including race-only events (#974). With no grid, the fallback uses the car the camera is on.
- **The frame:** `laps.findLapStart` for that car's lap 1 in the target session. Replays recorded by 3.4 and 3.5 have a lap record and no green record, so this is a single jump for them. With no lap record, it runs the #1203 walk to lap 1 of that car in the target session. That requires the walk to take a target session other than the cursor's. As built, the walk is told which instance of the `SessionNum` to work in: the one the cursor is in (Jump to Fastest Lap, and a green press made inside the race), or the latest one in its session map (a green press from practice or qualifying, which knows no `SessionUniqueID` for the race). A converged walk records its landing in the lap record, as it already does, so the next press is a lookup.
- **A walk that could not verify its landing** still anchors the jump when the landed sample shows the car in the target session, on the lap before lap 1, at 99 % of that lap or later. The walk's own bound for what it writes into the lap record is 99.9 %, and iRacing's Next Lap has been measured landing 11 frames before the line, at 99.78 % of an 85-second lap, where the walk reports "did not converge" and records nothing (#1432). Eleven frames are noise against a 300-frame run-up, so the green fallback judges the landing by its own, looser bound and leaves the record's bound to #1432. A landing further off than that, or in another lap or session, leaves the replay paused where the walk stopped. Until #1432 lands, a press on such a tape searches again every time and the walk logs its warning.
- **The landing:** `lapStart − GREEN_APPROACH_FRAMES`, through the same waited jump. A walk lands two ticks before the line, so it gets one further waited jump back by the rest of the lead-in.
- **The camera:** the walk's lap search is camera-relative, so it switches the camera to the pole car (#1203). After the landing, and before playback, the camera goes back to the car the user was watching at the press. A record-path fallback switches no camera. When the pole car is already on screen nothing is switched at all. The camera also goes back when the walk gives up, but not when another replay command cancelled the press: that command's own effect stands, and the camera is left on the pole car. A switch back that iRacing refuses is a warning, not a stop: the replay is where the press asked, and the playback step still runs.

Why this lands close: the pole car takes the line first. In a standing start its grid slot sits just behind the line, and in a rolling start the green flies at the line. So its lap-1 start trails the green by a second or two at most, and the five-second lead-in absorbs that. It is approximate in two named cases, both accepted. A pole sitter who started from the pit lane crosses late, so the fallback lands after the green. A track whose standing-start grid sits well behind the timing line lands correspondingly early.

**The capture that confirms the model:** on a rolling start, the record holds both the green frame and the pole car's lap-1 start. Their difference must be small and positive. That also shows whether pace-lap crossings move `CarIdxLapCompleted` off −1 (#307 says it stays −1 until the first S/F crossing, but no capture of a pace lap is cited for it). If a parade crossing does open lap 1, the fallback is wrong on rolling starts. It then targets the first lap start after the race session's last `ParadeLaps` tick, and this spec is amended.

**Rejected: land at the race session's start (Next / Previous Session).** It is cheap and always works, but for a rolling start it lands minutes before the green, behind gridding and every pace lap. That does not do what the mode's name says.

**Rejected: grey the key out with no record.** Every replay that exists today has no green record, so the key would be dead on all of them, including the requester's. Replay Markers greys out because a marker is the user's own and there is nothing to approximate.

**Rejected: search the replay for the `SessionState` change.** A bisection on `SessionState` across the race session would find the green itself. But whether session-level telemetry follows the replay cursor is not established (see #1324 above), and the lap walk is already built, repaired and tested.

## Restarts

The extension, not part of this issue: a **Next Green / Previous Green** pair over every green of the cursor's race session instance, start included. It walks relative to the cursor with Replay Markers' next/previous windows, and the key greys at the ends like a marker key, since restarts have no approximation. It joins `DIRECTIONAL_PAIRS`, which also makes it a natural rotation mode for #801's dial. Because the record already holds restarts, adding the pair is action code only. It is not offered to the requester now (maintainer, 2026-10-05); it stays here as the design the stored data was shaped for, and a later issue builds it.

## The dial

Replay Control has no dial surface on `master`: both the Elgato and the Mirabox manifest declare it `["Keypad"]`, and the `onDialDown` / `onDialRotate` handlers are unreachable (#640). #1316 builds the surface (Speed Control, with Frame Jog in #1317), and #801 adds stepping modes to it.

- **No rotation mode.** A single moment has no direction to turn in. If the restart pair is adopted, "greens" becomes one more stepping target in #801's list, not part of this issue.
- **A gesture-slot option: yes.** #1316's slots (Press, Long Press, Tap Display, Long Touch) offer Play/Pause, Set 1x and play, Reverse direction, Jump to Live and None. Jump to Live already makes a jump a slot option, and Jump to Green Flag is its counterpart at the other end of a race review. A Mirabox knob reaches only Press, so without the option a knob user needs a separate key for the one jump a review starts from. The cost is one enum value under `.catch`, one PI option, one comms entry and a hold-preview caption (`GREEN`). The preview's `false` cases are the press guards above. This lands after #1316. If #1316 is not merged when this is implemented, the option goes into #1316's slot list instead.

## Out of scope

- The Next Green / Previous Green pair (see _Restarts_).
- A green record for practice, qualifying or any non-race session.
- Restarts after a red flag that raise no caution bit, local yellows, and the checkered flag ("jump to finish"). Each would be another `kind` or section, and the store carries unknown kinds through.
- A per-key or global setting for the lead-in.
- Recording a green seen during a replay tick, live-field or otherwise. The gate is the lap record's, and it changes with it if #1324 or a later issue changes it.
- Renaming the _Fastest Lap Search Delay_ setting, although the fallback walk now uses it too. The website paragraph says so.
- The legacy Replay Navigation action (#1334). It gains nothing.
- Ulanzi dials: none are declared until verified (`encoders-and-touchscreen.md` rule 7).

## Testing

Unit tests, in the package that owns each piece:

- **`diff/replay-greens.ts`** (sim-events-iracing):
  - The first tick seeds and emits nothing.
  - `Warmup → Racing` on a standing race session and `ParadeLaps → Racing` on a rolling one each emit one `start` with the tick's `ReplayFrameNumEnd` and `SessionTime`.
  - `Invalid → Racing` emits nothing, and neither does a practice or qualifying session going `Racing`.
  - A re-entry into `ParadeLaps` followed by `Racing` emits a second `start`.
  - A caution episode (Caution set, then `Green` rising with both caution bits clear) emits one `restart`. A green rising with `Caution` still set (yellow-checkered) emits nothing.
  - An ineligible window (in a replay by the debounced state, `SessionNum` −1, no readable frame) unseeds the recorder. The first eligible tick after it emits nothing even across a `Racing` flip, and it also clears the caution latch.
  - A session change re-seeds.
  - The diff's position in `handleTick` is pinned as pre-guard (the #1127 ordering-test shape), and the shared eligibility function has its own tests, used by both diffs.
  - A fixture cut from a capture of a race start, one standing and one rolling, replays through the diff and asserts the exact event, and the caution fixtures assert each restart. They are the `local/` captures of 2026-09-17 and 2026-09-18; a frame is made from the session time, since the watch recorded none.
  - One start per race pass across an AI session restart: Practice, Qualifying, Race, a restart back to Practice and through to the Race, and a restart straight to the Race give three starts under three unique ids.
- **The store's `greens` section** (replay-store):
  - A record and lookup by pair.
  - The `sessionNum` fallback picks the unique session, and of several the latest.
  - The latest start replaces an earlier one for the same instance.
  - Restarts are de-duplicated within the window and stay in frame order.
  - A newer `version` is carried through: `record` returns false and the lookup reports `newer format`.
  - An unknown `kind` and unknown fields survive a write, and `markers`, `laps` and an unknown section survive a `greens` write.
  - The unreadable-file merge keeps the file's start and unions the restarts.
  - The offline record takes greens and never reaches disk.
- **Replay Control** (iracing-actions, under the deck-core mock with a mocked store):
  - Target resolution: the cursor's race session, and the last race session from practice. The last race session wins in a multi-race session list.
  - With no race session, nothing is sent and the key renders unavailable.
  - A hit sends one waited `setPlayPosition(Begin, frame − 300)` (clamped at 0), then the playback step, and no camera switch.
  - A miss with a lap record jumps to the pole car's lap-1 start minus 300 with no camera switch. With no grid, it uses the viewed car.
  - A miss with no lap record walks lap 1 in the target session, takes the extra waited jump back, and switches the camera back to the press-time car before playback.
  - A walk that did not converge still lands when it left the car within 1 % of a lap of the line, and leaves the replay where it stopped when it did not. The camera goes back when the walk gives up, and stays when another command cancelled it.
  - From the car, nothing is sent. A press during an in-flight walk or jump is ignored, of this mode or of Jump to Fastest Lap, and does not cancel the one in flight. `SessionNum` −1 ignores the press.
  - The key redraws on the tick the session list gains or loses its race, and keeps its plain look with no session list.
  - The comms catalog entry exists, `docs/reference/actions.json` lists the mode, and the icon preview's freshness test passes.
- **Dial** (with #1316's surface): the slot option dispatches the same path, and the hold preview returns `false` with no race session or outside a replay.
- **Catalog and harness**: the event compiles in `event-names.ts`.

Manual, in the sim (the gate before the PR), with the plugin's log open at debug. The steps use offline AI races, and restarting the event between steps adds a race pass to the same recording: a press inside a pass goes to that pass's green, a press from practice or qualifying to the latest one.

1. **Standing start, record path.** Run an AI race with a standing start (offline, held in memory). Afterwards, in the in-session replay, press from well after the start (more than 20000 frames away, the #1275 lesson). Expect `Jump to green: record HIT (matchedBy pair)`, a landing five seconds before lights out with the gantry in view, and the replay left as **Replay jump playback** says (paused stays paused under the default; set Play to see the start run). Read the edge tick's debug line: the `StartGo` rising edge must be within a few ticks of the `Racing` flip.
2. **Rolling start.** The same on a rolling start: the field approaching the line, and the `Green` bit within a few ticks of the flip. In the same session, compare the recorded green frame with the pole car's lap-1 start in the lap record. The difference must be small and positive. **This is the capture for the fallback model.**
3. **Restart.** An oval AI race with a caution. Check the file or the debug log for a `restart` entry on the tick `caution.restarted` fires.
4. **Saved file.** In a hosted or official session (offline sessions write no file), save the replay, reopen the `.rpy` and press. Expect a record hit landing about six seconds before the green.
5. **Fallback, lap record.** Remove the `greens` section from a session file (or open a replay recorded by 3.5) and restart the deck host. Expect `Jump to green: record MISS (no green); falling back to lap 1 of the pole car`, then `Jump to green: lap record HIT (matchedBy …)`, a landing before the green from the pole car's lap 1, and the camera unchanged.
6. **Fallback, walk.** Move the session file away and restart the deck host. Expect `record MISS (no file)`, then `Jump to green: lap record MISS (no file); searching the replay`, the walk to lap 1 of the pole car, a landing before the green, and the camera back on the car that was viewed at the press. Note whether the walk logs `walk converged` or `walk did not converge` (#1432): the press must land either way, and only a converged walk makes the next press a single jump. This is also the first time the walk is asked for lap 1, so it shows whether iRacing's lap search stops at the line that opens it.
7. **Multi-session replay.** Press from practice and from qualifying in a replay that holds both before the race: it lands at the race's green. Press inside the race: same.
8. **No race.** A practice-only or test-drive replay: the key is grey and a press does nothing.
9. **Session-level telemetry in replay.** In an in-session replay, scrub from before the green to after it with a Telemetry Display key showing `{{telemetry.SessionState}}`, and record in this spec whether it changes with the cursor. It no longer gates the feature, since the recorder's gate ignores replay ticks either way, but it decides whether a later "search the replay for the green" is possible. Then leave the replay for the car and check the log: no `Replay green` line may appear from the scrubbing or from the return.
10. **Dial** (once #1316 is in), on a Stream Deck+ and the Mirabox knob: a slot set to Jump to Green Flag behaves like the key.
