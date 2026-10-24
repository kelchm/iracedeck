# Pit Speed Limiter units: follow iRacing, or a unit chosen per key

> **Issue:** [#1476](https://github.com/niklam/iracedeck/issues/1476) · **Supersedes:** _none_ · **Superseded by:** _none_
>
> Point-in-time design record. The code and `.claude/rules/` are the truth; this is not documentation.

The Pit Speed Limiter key draws the track's pit speed limit inside a circle, and it has always drawn it in km/h: `parsePitSpeedLimit` in `car-control.ts` takes the leading digits of `WeekendInfo.TrackPitSpeedLimit` (`"80.00 kph"` → `80`) and nothing reads the driver's units. This makes the key follow iRacing's display units by default and adds a per-key choice. The issue carries the request; this records how.

## Behaviour

- **A new per-key setting, Unit** (`pitSpeedUnit`), shown in the PI only when the Mode is Pit Speed Limiter, with three options: **Auto (from iRacing)** (`auto`, the default), **km/h** (`kmh`) and **mph** (`mph`). The values match Session Info's `windSpeedUnit` spelling (`kmh`, `mph`). The label and the first option are Fuel Service's, word for word: its **Unit** dropdown opens with **Auto (from iRacing)** and its page describes that option as following the iRacing display units, so the two keys that follow iRacing's units say it the same way. Setup Chassis' **Units** / **Auto (follow iRacing)** is the one dropdown that says it differently, and it is left as it is here.
- **Auto (from iRacing)** reads `DisplayUnits` from the live telemetry: `0` (English) shows mph, `1` (Metric) shows km/h. A missing `DisplayUnits`, or no telemetry, counts as metric, which is what Session Info and the translator already do and what the key shows today.
- **The key draws a whole number only, in either unit**, as it does today. The decimals iRacing publishes are used for the conversion and never drawn. A limit iRacing posts in mph reaches us converted (`45.00 mph` is published as `"72.42 kph"`): in km/h the key shows `72`, and in mph the conversion starts from `72.42`, not from `72`, and the result is rounded to the nearest whole number, so the key shows the posted `45` (the mph value is 44.9997, which truncation would draw as `44`). The km/h figure is rounded the same way. That matches today's truncated figure for any limit whose decimals are below .5 (`72.42` → `72`, `80.00` → `80`), and differs only for one at .5 or above, which would now draw one higher.
- **The placeholder follows the unit too.** With no session the key shows its default 80 km/h, which becomes `50` in mph, so a deck shows a plausible number in the driver's unit before a session loads.
- **Nothing else about the key changes**: the number alone, no unit label; the same circle, status bar, border and tri-state. A key stored without `pitSpeedUnit` reads `auto`, so an imperial driver's existing key switches to mph on upgrade. That is the fix the issue asks for; it is called out in the changelog.
- **The key redraws when the unit changes**: when the setting changes, and when `DisplayUnits` changes mid-session under `auto`. The render cache key carries the displayed figure (it already carries the speed), so a changed unit yields a changed key and a redraw, and an unchanged one does not.

## Architecture

- **One parser for `TrackPitSpeedLimit`.** Two exist today: the action's leading-digits read, and the translator's `resolvePitSpeedLimit`, which handles `kph` / `mph` / `kmh` / `km/h` and the posted decimals for the pit-speeding threshold (#1059). The action's copy is the one that is wrong, so rather than fix it in place, the string is parsed once by a pure `parseTrackPitSpeedLimit(raw)` in `@iracedeck/iracing-sdk`, returning the value, its unit and how many decimals were published, or `null` when the string does not parse. The translator keeps its half-unit correction on top of that result — it is a threshold concern (a car held at the posted limit must not read as speeding), not a display one — and the key converts the parsed value without it. A string with no unit is read as km/h, which is what iRacing publishes.
- **The conversion lives with the key**, in a pure helper beside `pitLimiterSpeedGraphic` in `car-control.ts`, exported for its tests: parsed limit + unit setting + `DisplayUnits` → the whole number to draw. It does not reuse the translator's `getSessionStartConditions().pitSpeedLimit`, which is rounded into the units iRacing is set to and so cannot honour a per-key override.
- **No sim-neutral seam is added.** Car Control is a `SimIRacingAction` and already reads session info and telemetry through its controller for this mode; reading `DisplayUnits` there is the same read, one of the known places an action reads iRacing directly (#1351).
- **Shared with #1463.** Wipers and Visor, open while this is written, also edits `car-control.ts`, its test and `car-control.ejs`. Neither change touches the other's mode, so whichever lands second rebases; the settings schema gains one key from each.

## Settings contract

`pitSpeedUnit` is a new key in a stored action settings schema, which makes it a contract change: `z.enum(["auto", "kmh", "mph"]).default("auto")`. A missing key reads `auto`. An unknown stored value (from a later version that adds a unit) must not reset the key's other settings, so the field is parsed with `.catch("auto")` rather than failing the schema.

## Out of scope

- A unit label on the icon (rejected in the issue: the sign shows the number alone, and the large circle has no room).
- A plugin-wide unit setting shared by every key.
- Units anywhere else: Session Info's wind speed and temperatures, Telemetry Display templates, and the Race Engineer's spoken pit speed, which already follows `DisplayUnits`.
- The pit-speeding cue and the limiter callouts: their threshold logic is unchanged apart from reading the shared parser.

## Artifacts

- `packages/iracing-sdk/src/` — `parseTrackPitSpeedLimit` and its test, exported from the package index.
- `packages/sim-events-iracing/src/translator.ts` — `resolvePitSpeedLimit` reads the shared parser; its half-unit correction and its tests stay.
- `packages/iracing-actions/src/actions/car-control/` — the `pitSpeedUnit` field, the conversion helper, the render cache key, tests, and the Unit dropdown in `car-control.ejs` (gated on the Pit Speed Limiter mode). All three plugins get it through the shared action list and the shared PI template; no manifest or UUID change.
- Website: the Pit Speed Limiter section of `docs/actions/driving/car-control.md` — the Settings list (today "No additional settings"), which describes the default as Fuel Service's page does ("**Auto (from iRacing)** (default) — km/h or mph, following your iRacing display units"), and the icon description, which should say the number is in the unit chosen.
- A changelog fragment under Improvements, naming the upgrade change for imperial drivers.

## Testing

- **Parser:** `"80.00 kph"` → 80 km/h, 2 decimals; `"72.42 kph"` → 72.42 km/h, 2 decimals (the parsed value; the key never draws it); `"45 mph"` → 45 mph; `"60"` → 60 km/h; `""`, `undefined` and `"abc"` → `null`. The translator's existing #1059 tests still pass unchanged.
- **Conversion:** the result is always an integer; `"72.42 kph"` shows `72` in km/h and `45` in mph; `"80.00 kph"` shows `50` in mph; `auto` follows `DisplayUnits` `0` → mph, `1` → km/h, missing → km/h; `kmh` and `mph` ignore `DisplayUnits`.
- **Action:** a stored key without `pitSpeedUnit` parses as `auto`; an unknown value parses as `auto` and keeps the other settings; a `DisplayUnits` change under `auto` changes the render key and redraws, and under `kmh` does not.
- **By hand, in iRacing:** at a track whose limit is posted in mph, with iRacing in imperial, the key shows the posted mph figure; switch iRacing to metric and it shows km/h without a plugin restart; set the key to km/h and back with iRacing in imperial; repeat on a road course posted in km/h. Check the dropdown appears only in the Pit Speed Limiter mode, on Stream Deck and on Mirabox.
