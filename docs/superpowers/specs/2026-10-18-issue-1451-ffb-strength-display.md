# FFB Strength on the Force Feedback dial

> **Issue:** [#1451](https://github.com/niklam/iracedeck/issues/1451) · **Supersedes:** _none_ · **Superseded by:** _none_
>
> Point-in-time design record. The code and `.claude/rules/` are the truth; this is not documentation.

Related: #802 (the dial), #1268 (the View FFB Force key, open).

## Context

The Force Feedback dial shows the live FFB max force as `XX.X Nm` from `SteeringWheelMaxForceNm` (`formatDialValue` in `force-feedback-dial-surface.ts`). iRacing also shows the same setting as a **Strength** value, and some users tune by that number. Telemetry has no Strength variable. The SDK's own description of `SteeringWheelMaxForceNm` is "Value of strength or max force slider in Nm for FFB", which suggests one slider shown two ways. That is the SDK's wording, not something we have verified.

### What the data says about the conversion

The request proposes `Strength = 340 / SteeringWheelMaxForceNm`. Other suggested constants are 340.23 (another user) and 338.87 (the maintainer, from 72.1 × 4.7). The pairs the maintainer read off iRacing (Nm → Strength) are 10.1 → 33.5, 32.4 → 10.5, 5.7 → 60.0, 72.1 → 4.7, 629.6 → 0.5 and 8500.0 → 0.0.

iRacing shows both numbers to one decimal, so no single pair gives the constant. It only bounds `K = Nm × Strength` to an interval. Assuming both displays round to the nearest tenth:

| Pair (Nm → Strength) | K lies in |
| --- | --- |
| 10.1 → 33.5 | [10.05 × 33.45, 10.15 × 33.55] ≈ [336.2, 340.5] |
| 32.4 → 10.5 | ≈ [338.1, 342.4] |
| 5.7 → 60.0 | ≈ [338.7, 345.3] |
| 72.1 → 4.7 | ≈ [335.0, 342.7] |
| 629.6 → 0.5 | ≈ [283.3, 346.3] |
| 8500.0 → 0.0 | below ≈ 425 |

The intersection is about **[338.7, 340.5]**. One constant explains every pair, and 340, 340.23 and 338.87 all fit. The data cannot tell the three apart. 10.1 → 33.5 only looks wrong for 340 (340 / 10.1 = 33.66) because the true max force can be anywhere from 10.05 to 10.15 Nm. At 10.15, 340 gives 33.50.

Two caveats limit how far this goes:

- **The rounding assumption matters.** If iRacing truncates its displays instead of rounding, the same pairs give K in [342.0, 342.7]. None of the three candidates fits that range, and 338.87 is then the lowest value the 72.1 → 4.7 pair allows, not a fit. The capture below tells the two apart.
- **A wheelbase term is neither shown nor ruled out.** All the pairs may come from one wheelbase at one torque setting. If Strength were defined relative to the wheelbase's peak torque, these pairs could not show it.

### The live reading (2026-10-18)

Telemetry, read from a running sim at full precision, gave `SteeringWheelMaxForceNm = 32.25806427001953`, with `SteeringWheelUseLinear = 1` and `SteeringWheelPeakForceNm = -1` (apparently unset). That float is exactly `Math.fround(340 / 10.54)`. For the other candidates, 340.23 gives a Strength of 10.547130 and 338.87 one of 10.504970. A random value lands on a clean two-decimal Strength about one time in a hundred.

The model this suggests:
- iRacing stores Strength in 0.01 steps.
- It computes the max force as exactly `340 / Strength`.
- It shows both numbers rounded to one decimal.

That explains why `340 / displayed Nm` misses some displayed Strengths: 33.50 gives a true 10.149 Nm, shown as 10.1. 340.23 and 338.87 read as constants fitted to the rounded on-screen values. The same reading's `app.ini` has `displayLinearInNm=0` ("Display the force level in peak Nm when using the linear mode"). So the Strength number is likely the linear-mode display, and the peak torque mapping is the candidate for a wheel term.

## Decision

### The model

`Strength = FFB_STRENGTH_K / SteeringWheelMaxForceNm`, where `FFB_STRENGTH_K` is a named constant in Nm. It is **340**, decided by the maintainer on 2026-10-18 on the live reading above. The capture below confirms it before the PR opens. Implementation does not wait for the capture, because at most the constant's value would change.

If the capture shows K depending on `SteeringWheelPeakForceNm` or `SteeringWheelUseLinear`, this spec is amended on `master` before the PR, replacing the constant with a function of that variable. Until then the model is a constant.

### The capture that sets K

The maintainer runs it in the sim. At each slider position it reads two things:

1. `SteeringWheelMaxForceNm` at full precision, from a Telemetry Display key showing the raw value or from a Telemetry Control snapshot.
2. iRacing's displayed Strength and Nm, read at **eight or more slider positions**. These must include the strongest end of the slider (the smallest Nm) and a few mid-range positions. With exact Nm, each reading bounds K to `Nm × [S − 0.05, S + 0.05]`, an interval only `0.1 × Nm` wide. A 1 Nm reading therefore pins K to within 0.1, while a 30 Nm one only to within 3.

Record `SteeringWheelPeakForceNm` and `SteeringWheelUseLinear` alongside them. `SteeringWheelUseLinear` is not typed on `TelemetryData`; the snapshot carries it raw. Then repeat three or four positions at a **second wheelbase-torque setting**, or on a second wheelbase.

Outcomes:

- **`340 / Nm` lands on a two-decimal Strength at every position and at both torque settings.** The constant is exactly 340 and stays.
- **The display's rounding mode.** A position whose stored Strength ends in 5 (for example 10.55) shows whether iRacing rounds or truncates its one-decimal display. The readout copies whichever it does.
- **K moves with peak torque or linear mode.** The constant model is refuted, and the spec is amended as above.

The capture also records one more thing: which way the Nm value moves when the FFB Force **Increase** binding is pressed. The website page needs this to describe the turn honestly (see Rotation).

### The setting

`dial.ffbReadout`: `z.enum(["max-force", "strength"])`. It defaults to `max-force` and is `.catch`-guarded per field. A missing value keeps today's Nm readout. An unknown value, such as one written by a newer version, falls back to Nm without hitting the whole `dial` object's `.catch`, which would also reset the user's gestures and colours. It is a dial-only field under the `dial` root, like every other dial setting.

In the PI it is a **Show as** select with the options *Max force (Nm)* and *Strength*. It goes in the dial view below the Mode select and is shown only while `dial.setting` is `ffb-force`, using the conditional-visibility pattern in `stream-deck-actions.md`. It appears on both the Stream Deck+ and the Mirabox dial PI, since both draw the readout. The supporting text that says the strip shows Nm is updated to name both readouts. The LFE settings have no readout, so the select stays hidden for them.

### The readout

- **One formatter.** A new `shared/ffb-readout.ts` holds `FFB_STRENGTH_K`, `ffbStrengthFromNm(nm): number | null` and `formatFfbReadout(nm, readout): string`. `formatDialValue` calls it for `ffb-force`. It lives in `shared/` rather than in the action folder so that `shared/setup-view.ts` (#1268's key) can import it without reaching into an action.
- **Max force:** unchanged, `11.5 Nm`.
- **Strength:** one decimal with no unit, `33.5`, as iRacing shows it. The missing `Nm` is what tells the two readouts apart.
- **The label stays `FFB`**, in the same accent colour. The label names the setting the dial adjusts, which has not changed. The trigger description stays "Adjust FFB Force" for the same reason.
- **Bad or extreme values:**
  - Missing, non-finite, zero or negative Nm shows `---` in both readouts. Strength is computed only for `nm > 0`, so there is never a division by zero, an `Infinity` or a negative Strength.
  - Very large Nm gives a Strength that rounds to `0.0` (8500 Nm → 0.04 → `0.0`), which is what iRacing shows. It is not special-cased.
  - Very small positive Nm gives a large Strength. It is not clamped, since iRacing's slider bounds the range. A three- or four-digit Strength (`340.0`) is still shorter than the `99.9 Nm` the box already fits.
  - Anything non-finite after the division shows `---`.
- **Change detection** needs nothing new. The displayed value string is already part of the render signature, so switching the readout or a moving value re-renders under the existing throttle. `didReceiveSettings` already busts the memo.

### Rotation

A turn still taps `cockpitMiscFfbForceIncrease` / `cockpitMiscFfbForceDecrease`, clockwise = increase, scaled by ticks and capped at 5 taps per event. The readout choice never changes what a turn sends.

Strength is the inverse of max force, so whichever readout is shown, one of the two numbers moves against the turn. Which one depends on what iRacing's Increase binding does to Nm. That is not verified here, and the capture records it. Flipping the rotation when Strength is shown was rejected: a user who changed the readout would find their knob reversed. The rotation follows the binding names, the value-rotation case of rule 4 in `encoders-and-touchscreen.md`. The website page states which way the shown number moves.

### Relation to #1268

The two issues are **independent**: neither needs the other to ship. They share the formatter, and the rule for the order is: **whichever lands second wires the Show as choice into the other's surface.**

- **This lands first.** #1268's spec is amended on `master` to say its View FFB Force key gets a root-level `ffbReadout` field with the same enum, default and `.catch`, shown only for that mode. Its `formatForceNm` becomes a call into `formatFfbReadout`. Its label follows the readout (`FFB FORCE` / `FFB STRENGTH`), sized on the icon preview.
- **#1268 lands first.** This issue adds that same key field and moves `formatForceNm` into `shared/ffb-readout.ts`.

The key's field sits at the root and the dial's under `dial`. That follows the existing split that keeps the two surfaces' keys from colliding, and a placement is only ever one of the two.

## Alternatives rejected

- **A plugin-wide preference.** A user's choice of readout is arguably a personal setting, not a per-key one. But it would add a field to `GlobalSettingsSchema`, a persisted contract, for one readout, and every other dial readout option in the plugin is per instance. A user with one FFB dial sets it once.
- **Auto-detecting which readout the user has in iRacing.** We know of no telemetry that says which way iRacing is showing the slider.
- **Showing both values** (`33.5 / 10.1 Nm`). Too much for the strip and the knob screen.
- **340.23 or 338.87.** Both fit the rounded on-screen pairs, but neither gives a clean Strength for the exact telemetry value, which 340 does (10.540000).
- **Writing the maintainer's pairs into tests as truth.** Their Nm values are rounded, so `340 / 10.1` legitimately fails to give 33.5. Fixtures use the capture's full-precision Nm.

## Out of scope

- A Strength readout on the keypad FFB Force increase / decrease keys. A live value on a key is #1268's View mode.
- The Cockpit Misc `ffb-max-force` legacy alias, and the LFE settings, which have no readback.
- A `reverseRotation` setting for the Force Feedback dial. File one if someone wants the turn to follow the shown number.
- A wheelbase or linear-mode term in the formula, unless the capture shows one. In that case this spec is amended, not extended later.
- Setting Strength to an absolute value. iRacing's bindings only step.

## Testing

Suite:

- `ffb-readout.test.ts`:
  - `ffbStrengthFromNm` with captured full-precision Nm values against the Strength iRacing showed for them.
  - 8500 Nm → `0.0`.
  - `0`, a negative value, `NaN`, `Infinity` and `undefined` → `null`.
  - `formatFfbReadout`: `11.5 Nm` and `---` for max force; `33.5` and `---` for strength.
  - If the capture picks truncation, a value just under a tenth boundary.
- `force-feedback-dial-surface.test.ts`:
  - The existing `11.5 Nm` / `---` output is unchanged with no `ffbReadout`.
  - `ffbReadout: "strength"` shows the Strength string.
  - The schema defaults `ffbReadout` to `max-force` and parses an unknown string to `max-force` while keeping a sibling such as `pressAction: "auto-ffb"`.
  - A settings change from `max-force` to `strength` re-renders.
  - With Strength shown, a clockwise turn still taps `cockpitMiscFfbForceIncrease` and a counter-clockwise turn `cockpitMiscFfbForceDecrease`, with the same tap cap.
  - The trigger description is unchanged.
- If #1268 has merged: its View key's tests for both readouts and the root-level field's default and downgrade.

Manual, in iRacing:

- The capture above, before the PR. It sets `FFB_STRENGTH_K` and the rounding mode.
- On a Stream Deck+ and on a Mirabox knob: with Show as Strength, the dial shows the same Strength as iRacing's own display at several slider positions, including both ends. Switching back to Nm shows today's readout.
- The Show as select appears only for FFB Force, and the Nm default holds on an existing dial after the update.
- What the dial shows in the garage, in a replay and with iRacing closed. Expected: `---` without a value. Confirm it in the sim rather than assuming.
- Which way the shown number moves on a clockwise turn in each readout, so the website page can say so.
