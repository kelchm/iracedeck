# Race Admin Car Selector dial: browse the field, press to select

> **Issue:** [#1478](https://github.com/niklam/iracedeck/issues/1478) · **Supersedes:** _none_ · **Superseded by:** _none_
>
> Point-in-time design record. The code and `.claude/rules/` are the truth; this is not documentation.

Race Admin picks its target car on the keypad with Select Car keys: each key is a slot in a grid over the session's car numbers (#732, #754), and a press stores the car as the admin target (`_selectedCar`) and switches to the Race Admin profile. Race Admin has no dial today; all three manifests declare it `Keypad` only. This adds one dial surface, Car Selector, which walks the same car list on the dial and commits on the press. The issue carries the request; this records how.

## Behaviour

- **The car list is the Select Car keys' list.** `getAllCarNumbers(sessionInfo, true, true)` from `@iracedeck/iracing-sdk`: every car in `DriverInfo.Drivers` except the pace car and spectators, sorted by car number. That is what `resolveSlotCar` reads for the keys, so a car the dial offers is a car the keys offer. There is no world-presence filter: a car in the garage or one that has left is still a car an admin may want to act on (the #1281 rule).
- **Rotation browses and sends nothing.** A detent moves a cursor through the list. It doesn't write the selection, move the camera or send anything to iRacing. Clockwise lowers the car number (`#94 → #77`), the number-primary rule in `encoders-and-touchscreen.md` (#884, #973); a rotation is scaled by its `ticks`. The cursor wraps from one end of the list to the other, because a field has no natural first car.
- **The cursor follows a car, not an index.** When the roster changes (a driver joins or leaves), the cursor stays on the same `carIdx`; if that car is gone, it moves to the car with the next number, or wraps to the first. On appear, and whenever `_selectedCar` changes from elsewhere (a Select Car key press), the cursor moves to the selected car if it is in the list. Otherwise it starts at the first car.
- **What the screen shows,** drawn by the shared `renderDialBox` on both canvases (the Stream Deck+ strip and the Stream Dock knob):
  - label `CAR`;
  - value: the cursor's car number, `#12`;
  - caption: the driver's last name, uppercased. It is the same text the Select Car key shows (`splitDriverName(userName).lastName`), so a car reads the same on the key and on the dial;
  - `neighbours`: the car numbers either side, dimmed, oriented so the number a clockwise detent goes to is on the clockwise side (the #801 Track Order precedent).

  With no session, or an empty list, the box shows `CAR` and `—` and the press does nothing.
- **The press selects the car shown, exactly as a Select Car key does**: `_selectedCar` is written with the car's `carIdx` and number, and the deck switches to the dial's Target Profile (`requestProfileSwitch(deviceId, profile, 0)`, the profile name resolved for the device as the key resolves it). It doesn't move the camera. Where the switch can't happen, the target is still set and the skip is logged, as the key does: on a device with no bundled Race Admin Per Car profile (the regular Stream Deck+ until another issue ships one), and on Mirabox, where profiles don't exist (Elgato only, `profiles-and-devices.md`). A press with an empty list does nothing.
- **Gestures.** Press is the only action. A touch tap on the Stream Deck+ strip does the same as a press, and there is no long press or push+turn. With nothing behind a hold there is no hold preview, so the surface uses deck-core's `NOOP_HOLD_PREVIEW` on every host. A press on the Mirabox knob is atomic (`dialDown` only), which a press-only surface handles as is.

## Architecture

- **A surface beside the action, `race-admin/race-admin-dial-surface.ts`,** shaped like the others (`fuel-service/fuel-dial-surface.ts` is the reference): per-context state created only by `willAppear` / `didReceiveSettings` through `ensureContext`; input events through `inputContext` and `hasDialInputContext`, so an event after `willDisappear` is dropped (#1329); a press classified at release by `classifyDialReleaseForHost`; pushes throttled to 10 a second. `race-admin.ts` routes every dial event to it, and every handler branches on `isDial()`.
- **One selection path for the key and the dial.** The body of `executeSelect` — write `_selectedCar`, resolve the target profile for the device, request the switch or log the skip — moves into a function both call with a car and a device. The dial must not get a copy: the device-name resolution and the no-profile fallback are where the two would drift.
- **The dial's settings** are their own schema under `dial`: `mode` (an enum with the one value `car-selector`, so a second dial mode later is additive) and `selectorTargetProfile` (default `iRaceDeck Race Admin Per Car`, as the keys'). `_deviceProfiles` is pushed for a dial context too, so the Target Profile dropdown lists the device's profiles. These are new keys in a stored settings schema, so a contract change; a stored key without `dial` parses to the defaults.
- **Manifests.** Race Admin's `Controllers` gains `Encoder` on Elgato, with a `layouts/race-admin.json` full-canvas layout and `TriggerDescription` (Rotate "Browse cars", Push "Select car", Touch "Select car"), and `Knob` on Mirabox, with no config block. Ulanzi stays `Keypad` until its dials are verified. `scripts/manifest-actions-order.test.mjs` keeps the Encoder and Knob sets equal. The action UUID is unchanged.
- **The PI** includes the shared `dial-controller` partial and calls `irdResolveController()` / `irdApplyDialView()` at the end of `initialize()`. The keypad settings go in `#keypad-settings`, and the dial view shows the Mode dropdown (Car Selector, labelled "Mode" per the terminology rule) and Target Profile.
- **No comms-catalog entry for the rotation.** It sends nothing to iRacing; the press writes a setting and switches a profile, neither of which is a way the action talks to iRacing (the Replay Markers Add / Delete precedent).

## Out of scope

- Bundled Race Admin profiles for dial decks that have none yet, the regular Stream Deck+ first. Another issue ships them; the Target Profile dropdown picks them up when they exist.
- Moving the camera to the car, on rotation or on press. The keypad's focus-camera intent (Camera Controls' Focus Select Car) stays keypad-only.
- Ordering by race position or track order. The list is car-number order, matching the keys.
- A per-car admin command on the dial (black flag, DQ, and so on). Those stay Race Admin key modes acting on the selected car.
- Ulanzi dials.

## Artifacts

- `packages/iracing-actions/src/actions/race-admin/`: `race-admin-dial-surface.ts` and its test; `race-admin.ts` (dial routing, the shared selection function, `_deviceProfiles` for a dial); `race-admin.ejs` (the dial view).
- Elgato and Mirabox `manifest.json` (Race Admin's controllers), `layouts/race-admin.json` on Elgato.
- Website: a dial section on `docs/actions/communication/race-admin.md` in the `website-action-docs.md` dial form, with the Target Profile setting and the Elgato-only note for the switch; the per-mode "Dial: No rotation support" lines become a single statement that the dial runs Car Selector.
- A changelog fragment under Features.
- The `iracedeck-actions` skill's dial listing.

## Testing

- **Car list and cursor:** pace car and spectators excluded; sorted by number; clockwise lowers the number; `ticks: 3` moves three cars; wrap at both ends; the cursor stays on its car when another car joins or leaves, and moves to the next number when its own car leaves; it starts on `_selectedCar` and follows a selection made by a key.
- **Render:** the label, `#<number>`, the last name in the caption, the dimmed neighbours on the right sides, and `CAR` / `—` with no session, on the strip and the knob canvas.
- **Press:** writes `_selectedCar` and requests the switch with the device-resolved profile; with no bundled profile, writes the target and logs the skip; with an empty list, does nothing; never sends a camera command. A dial event after `willDisappear` is dropped.
- **Shared selection:** the Select Car key's existing tests pass unchanged against the extracted function.
- **By hand:** on a Stream Deck + XL in a hosted session, rotate through the field and check the names against the iRacing roster, press, and confirm the deck lands on the Race Admin Per Car profile with that car as the target (a per-driver command acts on it). On a regular Stream Deck+, the press sets the target and stays on the current profile. On a Mirabox knob, rotate and press. Confirm the camera never moves.
