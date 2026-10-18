# Camera Controls dial: the replay dial's look

> **Issue:** [#1449](https://github.com/niklam/iracedeck/issues/1449) · **Supersedes:** _none_ · **Superseded by:** _none_
>
> Point-in-time design record. The code and `.claude/rules/` are the truth; this is not documentation.

Camera Controls' dial draws its own carousel strips (`camera-controls/camera-dial-surface.ts`: `renderCameraCarousel`, `renderSubCameraCarousel`, `renderCarCarousel`, `renderRacePositionCarousel`, and `renderKnobCarousel` for the Mirabox knob). The Replay Control dial draws the shared dash box (`shared/dial-box.ts`) that most of the plugin's dials use: a label, a value, a caption, side marks, and, from #801, the neighbours beside the value. With #801 (`2026-10-12-issue-801-replay-control-dial-stepping.md`) and #1448 (`2026-10-18-issue-1448-replay-dial-camera-car-modes.md`), Track Order, Car # and Position exist on both dials, drawn two ways. The maintainer prefers the replay dial's look (2026-10-18: "This look is better than what Camera Controls has.") and chose to restyle Camera Controls' dial in an issue of its own, after #1448 (2026-10-18). It is built after #801, #1448 and #1437: it changes `shared/dial-box.ts`, which #801 (the neighbour slot) and #1437 (the busy look) change too, so it lands after both and rebases onto them.

## The decision

1. **Every Camera Controls dial mode is drawn in the shared dash box**, through `renderDialBox`, which draws the Stream Deck+ strip and the Mirabox knob alike. The carousel renderers and their tests are deleted.
2. **What a mode does is unchanged.** The targets, the sends, the number-primary rotation (a clockwise detent lowers the number in Car # and Position, #884 / #973), `reverseRotation`, the hold preview's outcomes (Focus My Car only, #1120), the #612 missing-binding warning on Sub-camera and Track Order, and the stored settings: no value is added, renamed or removed. So nothing in the settings changes, and no migration is needed.
3. **The camera-car modes look exactly as on the replay dial**: Car # and Position as #1448 draws them, Track Order as #801 draws it, the neighbours beside the value included. One mode, one look, on either dial.

   | Mode | Label | Value | Neighbours | Caption | Side marks |
   | --- | --- | --- | --- | --- | --- |
   | Camera | `CAMERA` | the group's icon and name | the dimmed neighbouring groups of the enabled subset, one detent either way | none | both lit |
   | Sub-camera | `SUB CAM` | the camera name | the dimmed neighbouring cameras from the group's `Cameras[]`, a guide as today (#852) | none | both lit |
   | Car # | `CAR #` | `#12` | the car numbers one detent either way | none | both lit |
   | Position | `POSITION` | `P4` | the positions one detent either way | `#12` | lit where a detent that way has a car |
   | Track Order | `TRK ORD` | `#12` | as #801: the cars ahead and behind, where the positions can be trusted | none | both lit |
   | Driving Cam | `DRIVING` | the group's icon and name | none, as today: iRacing resolves `group ± 1` itself | none | both lit |

   Every mode keeps its side marks, and a box label too long for them is shortened instead (maintainer, 2026-10-18: "I want the small arrows! If the name is too long, call it TRK ORDER or TRACK ORD, or something like that."). The rule is a margin: a box label leaves at least `INCIDENT`'s margin between itself and the side marks on each side, about 10 px on the Mirabox knob, where the room is tightest (the triangles' inner edges at ±0.3 of its 176 px width leave about 96 px for a label in Arimo Bold at 16 px; the Stream Deck+ strip leaves about 110 px at 15 px). Touching is not enough: #801's measurements found `TRK ORDER` and `TRACK ORD` both touching on the knob. The width test #801 adds for its own labels pins the margin, and these labels are added to it, measured with the renderer's own text width on the strip and on the knob. By Arial Bold advance widths (Arimo's metrics; no kerning, so the test's measurement decides), each side on the knob: `TRK ORD` about 11.6 px, `SUB CAM` and `DRV CAM` about 10.7 px, `DRIVING` about 14.7 px, `CAMERA` about 12.9 px, `CAR #` well over, and `POSITION` about 9.8 px against `INCIDENT`'s 10.2. So Track Order is `TRK ORD` and Driving Cam `DRIVING` (the maintainer's picks, 2026-10-18; `DRIVE CAM` left about 3 px), Sub-camera `SUB CAM`; `CAMERA` and `CAR #` fit as they are, and `POSITION` is #1448's. Only the box is shortened: the Property Inspector's Mode option and the website keep the full names (amended 2026-10-18; the earlier rule that a long label took no side marks is gone). Position here keeps Camera Controls' order as it is (the canonical order, the official one as fallback); the replay dial's `P--` rule for a replay scrubbed back in a live session (#1448) is that dial's, and Camera Controls is not changed in what it does.
4. **The camera modes keep their group icons and their neighbour names** (maintainer, 2026-10-18, chosen over dropping them): the shared box gains the slots for them (below).
5. **The per-mode accents stay** (`MODE_COLOR`) as the box's default accent per mode, and the user's colour overrides (`dialAppearanceFields`) apply as they do today.
6. **Out of a session** each mode shows the box dimmed with its label, as the replay dial does, in place of today's plain identity label.

## The box's new slots

The shared dash box (`shared/dial-box.ts`, drawn by `dial-strip-box.ts` and `dial-knob-box.ts`) gains two optional arguments, on the strip and on the knob:

- **An icon** beside the value: Camera Controls' group icons, drawn as the carousel draws them today, the value fitted into what is left.
- **Neighbours that take a name**, not only a number: #801's neighbour slot widened to fit a short text either side, dimmed, fitted by the box like the value and cut with an ellipsis where it cannot fit.

**Every dial that does not pass them is drawn byte for byte as before.** Fourteen dials and more draw this box (the seven Setup dials, Camera Editor Adjustments, Cockpit Misc, Force Feedback, Replay Control, Replay Markers, Splits & Reference, View Adjustment, Black Box Selector, and Replay Control's new modes). A test pins it: each of those dials' boxes, rendered with representative arguments on the strip and on the knob, is compared with its SVG as rendered on `master` before this change, captured once as fixtures in the same change and refused if it differs by a byte.

## Review

`/code-review high`: a shared renderer that fourteen dials use is still ordinary domain code. It holds no persisted data, decides no stored setting and is no published contract, so it is not the `xhigh` row. The byte-identity test is what carries the risk of a change to every dial. No second read: the diff is not a gate, a build or publish path, or a state machine.

## The maintainer's decision (2026-10-18)

**The camera modes keep their icons and neighbours**: the shared box gains an icon slot and wider neighbour slots, and every dial that does not use them stays byte-identical, pinned by tests.

**Every mode keeps its side marks**, a long box label shortened instead: Track Order `TRK ORD`, Driving Cam `DRIVING`, Sub-camera `SUB CAM`; Position as #1448 decides it.

## Out of scope

- **Any change to what a mode does**: its targets, sends, rotation mapping and settings.
- **The replay dial's rules on Camera Controls' dial**: the `P--` rule for Position in a replay scrubbed back, and the wait for a jump under way before a camera switch (#1435 covers a camera key during a jump).
- **The keypad**, and Camera Controls' Property Inspector beyond any help text that describes the strip.
- **Ulanzi dials**, which are not declared.

## Testing

**Unit:**

- Each mode's box: label, value, icon, neighbours, caption and side marks, on the strip and on the knob; a long group or camera name fitted by the box, and a long neighbour name cut with an ellipsis; out of a session, the dimmed box.
- **Byte identity**: every other dial's box, on the strip and on the knob, renders byte for byte as the fixtures captured on `master` before the change; the test fails on a planted one-byte change to the renderer.
- The camera-car modes' boxes are the same as the replay dial's for the same camera car and field (one helper, or the same expectations in both suites).
- The #612 warning drawn on Sub-camera and Track Order with a binding missing, and on no other mode.
- The hold preview: Focus My Car's car number in the value slot with the pending bar, the side marks unchanged.
- Every existing dispatch and rotation test passes unchanged: the behaviour did not move.
- The per-mode accent and a user colour override reach the box.

**Manual, on a Stream Deck+ and on the Mirabox knob:**

1. Each of the six modes in a live session: the box shows what the table says, and a turn does what it did before.
2. Car #, Position and Track Order side by side with a Replay Control dial in the same mode: the two boxes look the same.
3. Hold the dial past the threshold with Focus My Car on Long Press: your car number shows with the pending bar, and the release focuses it.
4. Sub-camera and Track Order with their iRacing binding unset: the warning shows.
5. Out of a session: each mode's box is dimmed with its label.
