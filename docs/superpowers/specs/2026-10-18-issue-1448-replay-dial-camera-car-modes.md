# Replay Control dial: Camera Controls' camera-car modes, in the replay dial's look

> **Issue:** [#1448](https://github.com/niklam/iracedeck/issues/1448) · **Supersedes:** _none_ · **Superseded by:** _none_
>
> Point-in-time design record. The code and `.claude/rules/` are the truth; this is not documentation.

#801 (`2026-10-12-issue-801-replay-control-dial-stepping.md`) gives the Replay Control dial a **Track Order** mode: Camera Controls' Track Order rotation, iRacing's Next / Previous Car binding, drawn in the replay dial's box. This issue brings Camera Controls' other camera-car modes over the same way. It is built after #801 (maintainer, 2026-10-18: "a new issue, after #801") and on #1430's replay command gate (`2026-10-12-issue-1430-replay-command-gate.md`). The maintainer's reason (2026-10-18): "We could also copy other similar modes as "Track Order" from Camera Controls to Replay Control, and make them look and feel what the Replay Control Dial now looks and feels like. This look is better than what Camera Controls has."

## What Camera Controls' dial has

Read on `master` at `c4509d379`, `camera-controls/camera-dial-surface.ts`. Six rotation modes, in two families:

| Mode (stored value) | What a detent does | How it is sent | What the strip shows |
| --- | --- | --- | --- |
| Camera (`camera`) | the next / previous camera group of the enabled subset | the keypad's cycle dispatch, an SDK camera command (`cycle-camera`) | the group's icon and name, the neighbouring groups dimmed either side |
| Sub-camera (`sub-camera`) | the next / previous camera within the group | iRacing's Next / Previous Sub Camera **binding** (#852) | the camera's name, its neighbours from `Cameras[]` beside it, as a guide only |
| **Car #** (`car-number`) | the next / previous car by number, every car the session has had (#1281) | `camera.switchNum` to the computed car (`computeCarNumberTarget` over `getAllCarNumbers`) | the car number large, the neighbouring numbers beside it |
| **Position** (`race-position`) | the next / previous car in the race order; a car with no position re-enters at the end (#803) | `camera.switchNum` to the car NUMBER at the target position, resolved from the canonical order (`getLiveRacePositions`, official `CarIdxPosition` as the fallback, `race-positions.md`), never `switchPos` | `P<pos>` large with the car number beneath, the neighbouring positions beside it |
| Track Order (`track-order`) | the car ahead / behind on track | iRacing's Next / Previous Car **binding** (#1277) | the car number alone |
| Driving Cam (`driving`) | the next / previous camera group of the driving cycle | the keypad's cycle dispatch, `group ± 1` handed to iRacing | the group's icon and name, no neighbours |

Car # and Position are number-primary: a clockwise detent makes the number on the strip go down (`#94 → #77`, `P4 → P3`, #884 / #973), and a `reverseRotation` setting inverts every mode's mapping. Car # and Position are the two camera-car modes besides Track Order. Camera, Sub-camera and Driving Cam step the camera, not the car.

## The decision

1. **The Replay Control dial's Mode select gains Car # and Position.** Stored values `car-number` and `race-position`, the same as Camera Controls', labels on the box `CAR #` and `POSITION`, Mode select labels Cars by Number and Cars by Position. The rest of the surface is #801's and is unchanged: the press and gesture slots, the in-replay guard, the throttle and the dash box.
2. **Each does what it does in Camera Controls.** The same target: the car number walk over every car the session has had, and the race-order walk with its re-entry at the end. The same send: `camera.switchNum` to a car number. The target computation is shared, not copied: `computeCarNumberTarget` is in `shared/car-cycling.ts` already, and Position's `computeRacePositionTarget` and `carNumberRawAtPosition` move from the Camera Controls surface to `shared/` with their tests, Camera Controls importing them from there.
3. **Clockwise follows Camera Controls' rule**: a clockwise detent makes the number on the box go down, as there. One rule for every dial that shows a car number or a position, rather than a second mapping to learn; Track Order keeps clockwise = the car ahead, as in both dials. The Replay Control dial has no `reverseRotation` setting and gets none here.
4. **The look is #801's Track Order box.**

   | | Car # | Position |
   | --- | --- | --- |
   | Label | `CAR #` | `POSITION` |
   | Value | the camera car, `#12` | the camera car's position, `P4` |
   | Neighbours | the car numbers one detent either way, as Track Order's are drawn | the positions one detent either way |
   | Caption | none | the camera car's number, `#12` |
   | Side marks | both lit | lit where a detent that way has a car to go to |

   Every mode keeps its side marks, and a box label too long for them is shortened instead (maintainer, 2026-10-18: "I want the small arrows! If the name is too long, call it TRK ORDER or TRACK ORD, or something like that."). The rule is a margin: a box label leaves at least `INCIDENT`'s margin between itself and the side marks on each side, about 10 px on the Mirabox knob, where the room is tightest (the triangles' inner edges at ±0.3 of its 176 px width leave about 96 px for a label in Arimo Bold at 16 px; the Stream Deck+ strip leaves about 110 px at 15 px). Touching is not enough: #801's measurements found `TRK ORDER` and `TRACK ORD` both touching on the knob. The width test #801 adds for its own labels pins the margin, and these labels are added to it, measured with the renderer's own text width on the strip and on the knob. By Arial Bold advance widths (Arimo's metrics; no kerning, so the test's measurement decides), each side on the knob: `TRK ORD` about 11.6 px, `SUB CAM` and `DRV CAM` about 10.7 px, `DRIVING` about 14.7 px, `CAMERA` about 12.9 px, `CAR #` well over, and `POSITION` about 9.8 px against `INCIDENT`'s 10.2. `CAR #` fits as it is. `POSITION` comes out a hair under the margin by those widths, within their error, so the test's measurement decides; if it fails, the box label is open for the maintainer (below). Only the box is shortened: the Property Inspector's Mode option (Cars by Number, Cars by Position) and the website keep the full names (amended 2026-10-18; the earlier rule that a long label took no side marks is gone).

   Car # needs no trust rule: the car list is the session's, and a camera switch to any car works in a replay (#1281). Position does: the race order is read from the per-car arrays, which are the replay's own only in a saved replay and at the live edge of an in-session replay (#1281's capture, #801's Track Order rule). Where they are not, the box shows the car number with `P--` and no neighbours, and a detent sends nothing and logs it at debug: a position stepped from the live field would focus a car the replay on screen does not place there (maintainer, 2026-10-18, chosen over stepping by car number there).
5. **The camera switch goes out as #801's Track Order does.** iRacing drops a camera change that arrives during a position jump's seek (#1430's capture: 9 of 9 and 3 of 3), so a detent waits until no jump is under way: `waitForReplaySettled({ jumpsOnly: true })` with no claim, bounded by `REPLAY_JUMP_MAX_WAIT_MS`. A search on its way does not hold it, since a camera change during a search is acted on and harmless (8 of 8). The detent cancels a cursor owner only when it is one of the two camera-riding walks (Jump to Fastest Lap's and Jump to Race Start's), checked by owner as #801's Car tap does it, and leaves a marker jump or a record jump's run standing. A newer detent replaces one still waiting, and the target is worked out when the switch goes out, from the camera car then.
6. **Long Press's Mode default** (#801's `mode-default`) runs Jump to My Car in Car # and Position too, as it does in Track Order.

## How Camera Controls' own dial relates

Unchanged by this issue. The maintainer chose (2026-10-18) to restyle it to the replay dial's look later, in an issue of its own: #1449 (`2026-10-18-issue-1449-camera-controls-dial-replay-look.md`). Until then the two dials draw the same camera-car modes in two looks.

## Open label

**Position's box label, if `POSITION` fails the margin** when measured (about 9.8 px each side on the knob by advance widths, against `INCIDENT`'s 10.2): `POS` (about 31 px each side) or `POSN` (about 25 px). Which?

## The maintainer's decisions (2026-10-18)

1. **Camera Controls' own dial** is restyled later, in #1449; this issue does not touch it.
2. **The camera-angle modes** (Camera, Sub-camera, Driving Cam) do not come to the replay dial: Car # and Position only.
3. **Position after scrubbing back in an in-session replay** shows `P--`, and a detent sends nothing, as decided above.

## Out of scope

- **Changing Camera Controls' dial or keypad.** Only the shared target helpers move to `shared/`, with no change in behaviour.
- **Camera, Sub-camera and Driving Cam on the replay dial** (the maintainer's decision 2).
- **Restyling Camera Controls' dial** (#1449).
- **A `reverseRotation` setting** for the Replay Control dial.
- **A car or camera key pressed during a jump** (#1435) and Camera Controls' own sends in a replay: the waits here are the replay dial's.
- **Ulanzi dials**, which are not declared.

## Testing

**Unit:**

- The moved target helpers pass Camera Controls' existing tests unchanged, from `shared/`.
- Each mode's target from a camera car: Car # over a field with gaps in the numbers, a departed car and the pace car excluded; Position with the leader, last place and a car with no position re-entering at the end.
- Clockwise lowers the number in both modes; Track Order's direction is unchanged.
- The view: label, value, neighbours, caption and side marks per mode; Position in a saved replay, at the live edge, and scrubbed back in an in-session replay (`P--`, no neighbours, and a detent sends nothing).
- The send: on a quiet replay a detent sends `switchNum` at once; with a marker jump under way it sends after the landing and the jump lands on its frame; with a search on its way it sends at once; it cancels a fastest-lap walk and leaves a marker jump's run standing; a second detent while one waits replaces it.
- Long Press's Mode default runs Jump to My Car in both modes.
- The settings enum takes the two new values, and a stored value from before reads as before.

**Manual, in an iRacing replay, on a Stream Deck+ and on the Mirabox knob:**

1. Car #: turn either way; the camera steps by car number, and the box shows the car with its neighbours. Clockwise goes to the lower number.
2. Position: the same in a saved replay of a race, at several points in it; the position shown is the one the replay's standings show at that moment.
3. Position scrubbed back in an in-session replay: `P--`, and a turn does nothing.
4. Press Next Marker and turn either mode straight after: the replay lands on the marker, and the camera then moves to the car the turn asked for.
5. Long press in either mode: the camera goes to your own car.
