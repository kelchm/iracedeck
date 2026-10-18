# Car Control Starter: flash while starting

> **Issue:** [#1438](https://github.com/niklam/iracedeck/issues/1438) · **Supersedes:** _none_ · **Superseded by:** _none_
>
> Point-in-time design record. The code and `.claude/rules/` are the truth; this is not documentation.

The Starter key shows whether the engine is running (#561): the red START badge at full strength with a green status bar when it is, dimmed with a red bar when it is not. This adds an optional flash after a press, so the key visibly reacts while the engine is cranking. The issue carries the request; this records how.

## Behaviour

- **A new setting on the Starter key: Flash while starting**, a checkbox shown only when the Mode is Starter, on keys and on dials. **Off by default**: the requester asked for an option, and a key that starts blinking after an update changes a look people already know.
- **A press starts the flash** when the setting is on and the engine is not already confirmed running. The badge alternates between its dimmed and full-strength look every **250 ms**, Session Info's flash rhythm (`FLASH_INTERVAL_MS`), for at most **3 seconds** from the press. The status bar and border keep showing the real state from telemetry throughout, so the flash never claims the engine is running.
- **It stops early when the engine is running.** The first time `starterToggleState` reads `on`, the flash ends and the key shows the normal green ON look.
- **If the engine has not started after 3 seconds**, the flash ends and the key shows whatever telemetry says, normally the dimmed OFF look.
- **A press while the flash runs restarts the 3 seconds.** A press while the engine is already running starts no flash.
- **It is display only.** The press-and-hold behaviour is unchanged: the binding is held on press and released on release exactly as today, and the flash is started after the hold is sent, so it never delays the key.
- **A short tap still flashes.** Releasing the key does not stop the flash. A tap too short to start the engine flashes for 3 seconds and then shows OFF, which is the feedback the request asks for.

## Mechanics

- **One timer per key context**, in a map beside `autoHoldTimers`. An interval steps the frames and stops itself at the 3-second deadline. It is cleared on `onWillDisappear` and on `onDidReceiveSettings`, as the auto-hold timers are.
- **The flash owns the image while it runs.** `updateDisplayFromTelemetry` checks the context's flash state each tick: when the engine reads running, it ends the flash; otherwise it skips the render, so the throttle and the flash never draw over each other. When the flash ends, the context's `lastState` entry is cleared, so the next telemetry tick draws the real state.
- **Frames go straight to `updateKeyImage`** at 4 Hz, as Session Info's flashes do. They do not go through `IconUpdateThrottle`, whose 10 Hz cap is above the flash rate anyway.
- **Artwork:** `starterArtwork` gains a separate "lit" input next to the state, so a frame can draw the full-strength badge over an OFF status bar. The existing two-argument output stays identical, so the icon previews do not change.
- **The setting** is `starterFlash`, a boolean in `CarControlSettings` that defaults to `false`, coerced from the string form the PI writes, like `autoHold`. It is a new key in a stored settings schema, so adding it is a contract change: keys stored without it read as off.

## Out of scope

- Flashing any other Car Control mode. If someone asks, the same timer serves them.
- A configurable duration or rate. 3 seconds and 250 ms are fixed; the request asked for "about 2–3 seconds".
- A sound on start. The Race Engineer is a separate system.
- Any change to what counts as "engine running". That stays #561's `isEngineRunning`.

## Artifacts

- `packages/iracing-actions/src/actions/car-control/` (action, test, `.ejs` PI) for all three plugins, through the shared action list.
- Website: the Car Control page's Starter section documents the setting; a changelog fragment under Features.
- No manifest or UUID change.

## Testing

- **Unit tests with fake timers** in `car-control.test.ts`:
  - with the setting on and the engine off, a key-down alternates the badge between lit and dimmed every 250 ms and stops at 3 seconds, ending on the OFF artwork;
  - telemetry reporting the engine running mid-flash stops it at once and draws the ON artwork;
  - no flash with the setting off, or with the engine already running;
  - a second press restarts the window;
  - `onWillDisappear` and a settings change clear the timer, and no frame is drawn after it;
  - the hold and release calls are the same as without the setting;
  - `starterArtwork` with two arguments returns exactly today's output.
- **Manual:** on a Stream Deck, set a Starter key's Flash while starting, stall the car and press Starter. The key should flash and go green as the engine catches. Tap it briefly instead: it should flash for 3 seconds and return to OFF. Also check a Stream Deck+ dial set to Starter, and a Mirabox knob.
