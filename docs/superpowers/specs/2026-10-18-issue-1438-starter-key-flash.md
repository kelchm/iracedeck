# Car Control Starter: flash while starting

> **Issue:** [#1438](https://github.com/niklam/iracedeck/issues/1438) · **Supersedes:** _none_ · **Superseded by:** _none_
>
> Point-in-time design record. The code and `.claude/rules/` are the truth; this is not documentation.

The Starter key shows whether the engine is running (#561): the red START badge at full strength with a green status bar when it is, dimmed with a red bar when it is not. This adds a flash after a press, so the key visibly reacts while the engine is cranking. The issue carries the request; this records how.

## Behaviour

- **Every Starter key flashes; there is no setting.** The first design made it an opt-in checkbox, off by default, because the requester asked for an option. Niklas decided on hardware that it should not be an option (2026-10-25), so it is always on. Car Control is a keypad action in all three manifests.
- **A press starts the flash** unless the engine is already confirmed running. The badge alternates between its dimmed and full-strength look every **250 ms**, Session Info's flash rhythm (`FLASH_INTERVAL_MS`). The status bar and border keep showing the real state from telemetry throughout, so the flash never claims the engine is running.
- **It runs for as long as the key is held, whatever the engine reads.** The first design stopped it the first time `starterToggleState` read `on`. On hardware that ended every held flash about 45 ms into the press, because iRacing clears `EngineStalled` as soon as the starter turns the engine over, so cranking reads as running. While held, nothing but teardown ends the flash.
- **After release it stops when the engine reads running**, drawing the green ON look, **or 3 seconds after the press**, drawing whatever telemetry says, normally the dimmed OFF look. A release that comes after those 3 seconds ends it at once. So a short tap still flashes for up to 3 seconds, which is the feedback the request asked for.
- **A press while a flash runs restarts it.** A press while the engine is already running starts no flash, and neither does one with no Starter binding set: that press sent nothing, so the key must not claim to be cranking. The press must itself be a Starter press as well as the stored Mode, so a Mode change landing during another control's send does not flash.
- **It is display only.** The press-and-hold behaviour is unchanged: the binding is held on press and released on release exactly as before. The flash starts with the press, while the hold is still being sent: a SimHub-role hold is an HTTP round trip, and starting after it resolved held the flash back for the whole round trip (found on hardware). The first frame waits a macrotask, so the hold's request still leaves first and the flash never delays the key.

## Mechanics

- **One flash per key context**, in a map beside `autoHoldTimers`: an interval that steps the frames, the first frame's timeout, the frames left before the 3-second deadline, the current frame, and whether the key is still held. Key up and dial up clear `held`. It is cleared on `onWillDisappear` and on `onDidReceiveSettings`, as the auto-hold timers are, so a Mode change during the hold's await stops it.
- **The flash owns the image while it runs.** `updateDisplayFromTelemetry` skips the render while a flash runs, so the throttle and the flash never draw over each other. After release, a tick that reads the engine running ends the flash, and each frame step and the release itself read the same telemetry and end it too, so ending does not depend on a subscriber tick arriving. When the flash ends, the context's `lastState` entry is cleared and the real state is drawn at once, so a key whose telemetry has stopped never sticks on a flash frame.
- **The current frame is part of the resolved state.** While a flash runs, the Starter's resolved telemetry state carries the frame's lit flag, so the regenerate callback a global-settings change runs draws the frame and not the bare state.
- **Frames go straight to `updateKeyImage`** at 4 Hz, as Session Info's flashes do. They do not go through `IconUpdateThrottle`, whose 10 Hz cap is above the flash rate anyway.
- **Artwork:** `starterArtwork` gains a separate "lit" input next to the state, so a frame can draw the full-strength badge over an OFF status bar. The existing two-argument output stays identical, so the icon previews do not change.
- **No settings key.** `CarControlSettings` is unchanged, so no stored-settings contract moves. A key that stored the short-lived `starterFlash` during development keeps an ignored extra key.
- **Debug lines** record when a flash starts, why it ends (`engine running` or `time elapsed`) and why a press does not start one.

## Out of scope

- Flashing any other Car Control mode. If someone asks, the same timer serves them.
- A configurable duration or rate, or turning the flash off. 3 seconds and 250 ms are fixed; the request asked for "about 2–3 seconds".
- A sound on start. The Race Engineer is a separate system.
- Any change to what counts as "engine running". That stays #561's `isEngineRunning`; this only stops trusting it while the key is held.

## Artifacts

- `packages/iracing-actions/src/actions/car-control/` (action and test) for all three plugins, through the shared action list. The `.ejs` PI does not change.
- Website: the Car Control page's Starter section documents the flash; a changelog fragment under Features.
- No manifest or UUID change.

## Testing

- **Unit tests with fake timers** in `car-control.test.ts`:
  - a tap alternates the badge between lit and dimmed every 250 ms and stops at 3 seconds, ending on the OFF artwork;
  - a hold keeps flashing past 3 seconds and while telemetry reads the engine running, and its release ends it on the real state;
  - after release, telemetry or a frame step reading the engine running ends it at once on the ON artwork;
  - it flashes while the hold's request is still pending, and the first frame waits for the request to go out;
  - no flash with the engine already running, with no Starter binding, or for a press that sent another control;
  - a second press restarts it; `onWillDisappear` and a settings change clear it, and no frame is drawn after;
  - the hold and release calls are the same whether or not it flashes;
  - `starterArtwork` with two arguments returns exactly the previous output.
- **Manual:** on a Stream Deck, stall the car and hold Starter. The key should flash for the whole hold and go green after release once the engine runs. Tap it briefly instead: it should flash for up to 3 seconds and return to OFF. Also check a key with no Starter binding: it shows the warning and does not flash.
