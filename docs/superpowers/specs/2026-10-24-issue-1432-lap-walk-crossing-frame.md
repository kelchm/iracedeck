# The lap walk ends on the frame the lap counter flips, and records that frame

> **Issue:** [#1432](https://github.com/niklam/iracedeck/issues/1432) · **Supersedes:** _none_ · **Superseded by:** _none_
>
> Point-in-time design record. The code and `.claude/rules/` are the truth; this is not documentation.

The issue names `replay-control.ts`; since #1467 the walk is `packages/replay-engine/src/lap-walk.ts` (`walkToLapStart`, phases 5–7), shared by Jump to Fastest Lap and Jump to Race Start.

## Context

Today the walk decides "at the end of lap N−1" from a distance (`CarIdxLapDistPct >= FASTEST_LAP_DIST_THRESHOLD`, 0.999), then jumps `FASTEST_LAP_FINAL_BACKSTEP_TICKS` (2) back and records the frame before that back-step as the lap start. Two things are wrong with that. The distance encodes a landing offset iRacing does not keep: Next Lap rests 11 frames short of the line on the issue's tape (0.9974–0.9983 on four cars), 99.82 % for Next Lap and 99.90 % for Previous Lap on lap 2 of the #1354 Motegi tape, and about 100 frames short for lap 1 of a rolling start there. And even when the check passes, the recorded frame is where iRacing's search rested, not where the lap starts, so the record holds a frame a few frames early that every later record jump (`target.frame − LAP_START_APPROACH_FRAMES`) and record step reads as the truth.

The issue's own measurement shows what the walk can verify instead: parked frame by frame, the car read lap 1 on 27339 and lap 2 on 27341. A lap start is a frame where `CarIdxLap` flips, and the walk can find it.

Niklas confirmed the calls below on 2026-10-24: the 120-frame bound, the final landing at the record jump's frame (`C − LAP_START_APPROACH_FRAMES`), Jump to Race Start in scope through the shared walk, and `CarIdxLap` as the lap field.

## Decision 1: the convergence rule

**The walk has converged when it holds two frames at most `LAP_CROSSING_MAX_FRAMES` (120) apart, the earlier reading the target car on lap N−1 and the later on lap N, both in the target `(SessionNum, SessionUniqueID)`; the crossing phase (Decision 2) then narrows them to adjacent frames.** Nothing about the distance enters the rule. The bracket is built from where a lap search left the cursor: forward from a landing that reads N−1 (the issue's case), backward from one that reads N (a search that lands on or past the line, which is what the test buffers model today).

**Why 120 frames (2 s).** The three models the issue leaves open reduce to two: the replay records 60 frames a second whatever the sim's speed, so a fixed time is a fixed number of frames. Under that model 11 frames is the whole story, and any bound above it works. Under the third, iRacing rests a fixed distance before the line, so the offset in frames grows as the car's speed at the line falls. A fastest lap starts from a flying lap, so its line is crossed at racing speed; a bound eleven times the measured offset covers a car taking the line at a tenth of the measured car's speed, below a pit-lane limiter for any racing car. It also covers the one larger offset measured, the rolling start's ~100 frames, so Jump to Race Start converges there too (Decision 6). The bound's cost is logarithmic (each doubling costs one probe), so it is set generously rather than tight. Its risk is only that a bracket probe 2 s past the landing finds the car in the pits, out of the world, or in the next session, and each of those is a plain "did not converge" (Decision 3), never a wrong record, because the rule demands a frame seen on each side.

**The capture that would settle the model**, so the bound can be revisited with numbers rather than guessed: the issue's frame-by-frame park around Next Lap's landing, repeated for the same lap on two cars that take the line at clearly different speeds (a racing lap and a lap under the pace car or a slow-zone, on any tape), recording the landing frame, the flip frame and the offset between them. It is not needed to ship: the walk logs that offset on every walk (Decision 4), so every walked lap from now on is a sample.

## Decision 2: how the flip is found

**A bisection by absolute jumps, with a near bracket first.** From a landing L on lap N−1, the first probe is `L + LAP_CROSSING_NEAR_FRAMES` (16, the measured 11 with half again and a power of two). If it reads N the flip is inside 16 frames and four more probes find it. If it reads N−1 the next probe is `L + 120`, and if that reads N seven probes bisect the 104 between. A probe frame is clamped to the target session's end frame. Backward from a landing that reads N is the mirror image. Every probe is the walk's existing `jumpTo`: an absolute jump sent through the gate as the claim holder (`sendOwnedReplayCommand`), waited for until `ReplayFrameNum` reads the frame sent (`seekReplayFrame`), so nothing is ever sent into an unlanded jump or a search on its way, and no clock decides a landing.

| Flip after the landing | Probes | Today |
| --- | --- | --- |
| ≤ 16 frames (the issue's 11) | 5 | 1 (the back-step), and nothing recorded |
| 17–120 frames | 9 | 1, and nothing recorded |
| > 120 frames | 2, then Decision 3 | 1, and nothing recorded |

Each probe is a short jump: within an update or two on a quiet replay, but iRacing acts on a jump sent soon after another only some 400 ms after the earlier one landed (2026-10-12, `replay-position-state.ts`). So the issue's case costs between a fraction of a second and about 2.5 s more than today, once per lap: the frame is recorded, and the next press is a single record jump.

Rejected:

- **Frame steps from the landing.** The cost is linear in the very offset being bounded: 11 probes on the issue's tape, 120 at the bound. And it buys nothing in the gate: a step sent while a target stands goes out as an absolute jump anyway, and a step sent into a search on its way cancels the search (#1430's captures).
- **Interpolating from the distance** (the frame where `CarIdxLapDistPct` would reach 1 at the landing's rate). It needs a second sample for the rate and assumes a constant speed through the line; the bisection's worst case is already 9 probes and assumes nothing.
- **One bracket probe at L+120 and a plain bisection.** 8 probes always, 3 more than the near bracket in the measured case.

## Decision 3: what is recorded, and what a miss does

**The recorded lap start is the first frame on which the car reads lap N.** That is what the live recorder writes: `diff/replay-laps.ts` records the first live tick on which `CarIdxLapCompleted` has risen by one, the tick the car starts lap `completed + 1`, which `CarIdxLap` reads (#1203). The walk decides on `CarIdxLap`, not `CarIdxLapCompleted`: it is the field every other phase of the walk bisects and steps on, the one the issue measured flipping, and #1281's snapshots show `CarIdxLapCompleted` reading the live field in an in-session replay. The walk measures in the replay's own frames, so in a saved replay a walked lap is the crossing there, while a live-recorded one carries #1162's lag uncorrected; that is #1162's decision and is unchanged.

**The walk then leaves the cursor on `max(0, C − LAP_START_APPROACH_FRAMES)`**, where a record jump to the same lap lands, so the first press of a lap (the walk) and every later one (the record) look the same. The two-tick back-step and `FASTEST_LAP_FINAL_BACKSTEP_TICKS` go: it existed because the walk could not know the crossing (#1203 said so), and two frames before the line is the car already on it. `lapStartFrame` becomes C, `landedFrame` the approach frame, and `priorLapDist` is read on the lap search's landing rather than on the back-stepped frame, which is what Jump to Race Start's fallback means by it.

**A flip not found within the bound records nothing, as today.** The walk warns `walk did not converge`, returns the unverified landing (`lapStartFrame` = the lap search's landing, `converged: false`) and the press goes on as it does now: Jump to Fastest Lap leaves the replay 60 frames before that landing under the playback setting, Jump to Race Start takes it only through `acceptsGreenFallbackLanding`. Recording the landing anyway is rejected for the reason #1203's hardening gives: a recorded frame is trusted by every later press without walking, so an unverified one is a permanent wrong jump.

## Decision 4: the nudge, and `FASTEST_LAP_DIST_THRESHOLD`

**The threshold goes, at the nudge too.** The nudge exists to bring the cursor near the line when the lap steps did not; whether it is near is now asked of the bracket probes, which answer it exactly. The rule, for the sample the lap-step phase leaves on lap N−1:

- **Put there by a lap search:** bracket from it. Only when its far probe still reads N−1 is the Next Lap nudge sent, once, and the bracket tried from the nudge's landing.
- **A bisection probe** (no lap search moved the cursor since): the nudge first, since a probe is anywhere in the lap; then the bracket from the nudge's landing, and only if that does not bracket, from the probe's own frame (a probe that happened to fall in the last 120 frames). This is #1354's lap-0 rule ("a probe that landed on lap 0 without a lap search is nudged whatever it reads") for every lap, so the special case goes with the threshold.
- **A nudge that lands on lap N** (today's overshoot) is bracketed backward from where it landed, instead of the `prevLap` recovery search, which lands at an offset nobody has measured. One search fewer, and exact.

At most one nudge, as today, and at most two bracket attempts. A grid car sitting on lap 0 within metres of the line (#1354) reads lap 0 at its far probe and is nudged, as now.

## Decision 5: cancel and interrupt

The crossing phase adds no wait of its own: it is `jumpTo` calls and at most one `search`, each of which calls `checkpoint()` before it sends and observes the claim at every poll. So a replay command from any key cancels it (`FastestLapPressCancelled`, nothing more sent) exactly as it cancels a bisection probe, and #1428/#1435's interrupt (a camera key, `LapWalkInterrupted` from the same `checkpoint`) takes the put-back path from any of its probes. Nothing is recorded until after the last probe and the final `checkpoint()`, so neither leaves a frame in the record. The final approach jump to `C − LAP_START_APPROACH_FRAMES` is not a search phase: once the walk has recorded its frame it is a plain jump, which a camera key waits for and does not interrupt (Niklas, 2026-10-24; the #1428/#1435 spec's *The search ends where the answer is found*). The #1428/#1435 spec's walk-interrupt tests name "the nudge, the back-step" as phases; with this change those are "the nudge, a crossing probe", and the two specs ship together on one branch.

## Decision 6: Jump to Race Start is in scope, through the shared walk

Its lap-1 search runs the same phases and has the same flaw, worse: on a rolling start iRacing's lap search rests about 100 frames before the pole car takes the line (Motegi, #1354), so today it never converges, never records, and lands by the 90 % fallback with a 6.1 s run-up. Under this change the flip at ~100 frames is inside the bound, the walk converges, records the pole car's real lap-1 start and lands `GREEN_APPROACH_FRAMES` before it, as Jump to Race Start's lap-record path already does. On a standing start a landing inside the grid wait either brackets the real crossing or does not converge, so #1354's open concern (a grid-wait frame recorded as the lap-1 start at 99.9 %) is closed by the same rule. `GREEN_FALLBACK_MIN_LAP_DIST` and `acceptsGreenFallbackLanding` stay, for a flip the walk did not find; their doc comment loses the "the lap search does not put the start of a lap on the line" reasoning only where it described the recorded frame.

## Log lines

- `walk did not converge (…)` keeps its wording; its reasons change. Gone: `not at the end of lap N before the back-step (…)`, `back-step landed elsewhere (…)`, `nudge recovery (prevLap) …`. New: `lap N not reached within 120 frames of the lap search's landing (frame L, …)`, `a crossing probe on frame F read <lap/session/out of the world>`.
- Debug, new, on every converged walk: `<label>: lap N starts at frame C; the lap search landed D frames before it (P crossing probes)` (or `after it`, for a backward bracket). This is the measurement Decision 1 wants.
- Debug, changed: the nudge line names why (`lap N−1 still read on frame L+120`, or `the lap steps ended on a bisection probe`); the converged line drops `+ 2-tick back-step` for `+ P crossing probes` and names `lap start frame=C, landed frame=C−60`; the `nextLap overshot … recovering via prevLap` line becomes `nextLap landed on lap N; bracketing back from frame L`.
- Info lines unchanged: `walk converged (lap start recorded)`, `replay put back where the press found it`.

## Out of scope

- **#1436**, finding moments by bisecting telemetry over the whole session instead of iRacing's lap search. This change keeps the lap search to get near and bisects only the last ≤ 120 frames; #1436 may replace it altogether.
- **The lap-step phase's own landings.** A Next Lap from lap N−2 that rests short of the line reads N−2, not N−1, and the loop then sends another; what iRacing does with a Next Lap sent 11 frames short of a line is unmeasured. The crossing phase starts from whatever sample the loop ends on and is not changed by it; the manual test watches for it.
- **The live recorder** and #1162's saved-file lag: unchanged, and not reconciled with walked frames.
- **Re-walking laps already recorded by a walk** under the old rule. Today's rule recorded only landings at ≥ 0.999, within about five frames of the line on the issue's tape; those stand.
- **Changing the bound by speed or lap length.** One constant until the capture in Decision 1 says otherwise.
- **The camera-key interrupt itself**: #1428/#1435's spec.

## Testing

**Unit** (`lap-walk.test.ts`, `fastest-lap-press.test.ts`, `race-start-press.test.ts`, against the fake controller and clock). The synthetic buffer (`lap-walk-buffer.test-support.ts`) gains a lap-search lead: Next Lap and Previous Lap rest `lead` frames before the line, `lead: 0` keeping today's model of a landing on the line.

- `lead` 11 (the issue): converges; records exactly the first frame of lap N; sends 5 crossing probes, every one an absolute jump that read back before the next went out; leaves the cursor on C − 60; the next press is a record hit landing on the same frame.
- `lead` 0: brackets backward from the landing on lap N; records C; sends no `prevLap`.
- `lead` 16, 17, 120 and 121: the two bracket tiers at their edges; 121 does not converge, records nothing, warns with the new reason, and lands 60 frames before the lap search's landing under the playback setting.
- The lap steps ending on a bisection probe mid-lap: one nudge, then the bracket; a probe in the last 120 frames whose nudge fails to bracket: brackets from the probe.
- A crossing probe reading another session, the car out of the world, or lap N+1: does not converge, records nothing.
- A bracket probe past the session's end frame is clamped to it.
- A replay key during a crossing probe: cancelled, no further probe, nothing recorded. A camera key during one (with #1428/#1435's interrupt): put back, nothing recorded.
- Jump to Race Start, `lead` 100 on lap 1: converges, records the pole car's lap-1 start, lands `GREEN_APPROACH_FRAMES` before it; `lead` 130: falls back through `acceptsGreenFallbackLanding` as today; a car on the grid on lap 0: nudged.
- The new debug line carries the offset; `FASTEST_LAP_DIST_THRESHOLD` and `FASTEST_LAP_FINAL_BACKSTEP_TICKS` no longer exist.

**The full gate:** `install`, `build`, `typecheck`, `format`, `lint`, `test`.

**Manual, in iRacing (Niklas)**, on the saved replay of sub-session 89265205 with its lap record moved aside and the deck host restarted, debug logging on:

1. Car #1, Jump to Fastest Lap (lap 2): `walk converged (lap start recorded)`, no warning, the debug line naming the lap-2 start (27340 or 27341) and an offset of about 11 frames. With Frame Backward / Frame Forward from there, the car reads lap 1 one frame before the recorded frame and lap 2 on it.
2. Press again: `record HIT`, the same landing about a second before the line, no search.
3. The other three cars of the issue: each converges; note each offset from the debug line.
4. The time of each walk against today's (the issue's 1.3 s cached, 4.3 s uncached).
5. If a rolling-start replay with no record is at hand (the Motegi tape): Jump to Race Start converges and lands five seconds before the pole car takes the line.

## Affected artifacts

- `packages/replay-engine/src/lap-walk.ts`: the crossing phase, the nudge rule, the final approach jump; `LAP_CROSSING_NEAR_FRAMES`, `LAP_CROSSING_MAX_FRAMES`; `FASTEST_LAP_DIST_THRESHOLD` and `FASTEST_LAP_FINAL_BACKSTEP_TICKS` removed; the `FastestLapWalkLanding` field docs; the header's phase list.
- `fastest-lap-press.ts` (the header's steps 3 and 4), `race-start-press.ts` (its debug line), `green-flag-target.ts` (the `GREEN_FALLBACK_MIN_LAP_DIST` comment), `lap-walk-buffer.test-support.ts` and the three test files above; `index.ts` only if a removed constant is exported (it is not today).
- `packages/replay-engine/CLAUDE.md`: the `lap-walk.ts` line says the walk records the frame the lap counter flips.
- Website `packages/website/src/content/docs/docs/actions/view-camera/replay-control.md`: Jump to Fastest Lap's search paragraph ("steps lap by lap to the line" gains "and finds the frame the car crosses it"); Jump to Race Start's "When the lap has to be searched for, iRacing's own lap search decides where lap 1 starts …" sentence, which no longer holds when the crossing is found.
- `changelog.d/1432-fastest-lap-search-remembers-lap.md`, `Bug Fixes`, weight 40: Jump to Fastest Lap now remembers a lap it had to search the replay for on every replay, so the next press for that lap is a single jump, and Jump to Race Start's search lands five seconds before the pole car takes the line on a rolling start.
- The issue's own list names `packages/iracing-actions/…/replay-control.ts` and `packages/iracing-actions/CLAUDE.md`; both are stale since #1467 and need no change for this.
