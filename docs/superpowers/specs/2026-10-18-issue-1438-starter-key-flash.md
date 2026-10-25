# Car Control Starter: flash while starting

> **Issue:** [#1438](https://github.com/niklam/iracedeck/issues/1438) · **Supersedes:** _none_ · **Superseded by:** _none_
>
> Point-in-time design record. The code and `.claude/rules/` are the truth; this is not documentation.

The Starter key shows whether the engine is running (#561): the red START badge at full strength with a green status bar when it is, dimmed with a red bar when it is not. This adds a flash while the starter is cranking, so the key visibly reacts the way the starter sounds. The issue carries the request; this records how.

## Behaviour

- **Every Starter key flashes; there is no setting.** The first design made it an opt-in checkbox, off by default, because the requester asked for an option. Niklas decided on hardware that it should not be an option (2026-10-25). Car Control is a keypad action in all three manifests.
- **Holding the key cranks, and the badge cranks with it.** The badge alternates between its dimmed and full-strength look, beginning lit, slow first and then faster, like a starter spinning up: frames of 450, 380, 320, 270, 230, 190, 160, 140 and 120 ms, and then 120 ms for as long as the key is held. The fastest frame stays under the 10 Hz cap the key's other updates keep. The status bar and border keep showing the real state from telemetry throughout, so the flash never claims the engine is running.
- **Releasing the key ends the flash at once** and the key shows the real state. Nothing is cranking when the key is not pressed, so there is no tail after a release (Niklas, 2026-10-25). An earlier version kept flashing for 3 seconds after a tap; that is gone.
- **The engine catching always ends the flash**, and the key goes green while it is still held. "Caught" is not the stall bit: on hardware iRacing cleared `EngineStalled` about 45 ms into every hold, as soon as the starter turned the engine over, so cranking reads as running. A cranking engine turns well below idle, so caught means the stall bit clear **and** RPM at or above the 500 RPM floor #561's `isEngineRunning` already falls back on (stopped engines read about 300 RPM in captures, the lowest idle near 900). Without RPM, the stall bit is all there is. iRacing's cranking RPM has not been measured; the end-of-flash debug line logs the highest RPM a flash saw, so the floor can be checked on hardware.
- **A press on a running engine gives one "chuk":** a single 200 ms dimmed blink over the green ON look, then the ON look again, however long the key is held — the one blip of a starter tried on a running engine.
- **No flash** for a press with no Starter binding set: it sent nothing, so the key must not claim to be cranking. The press must itself be a Starter press as well as the stored Mode, so a Mode change landing during another control's send does not flash. A press while a flash runs restarts it.
- **It is display only.** The press-and-hold behaviour is unchanged: the binding is held on press and released on release exactly as before. The flash starts with the press, while the hold is still being sent: a SimHub-role hold is an HTTP round trip, and starting after it resolved held the flash back for the whole round trip (found on hardware). The first frame waits a macrotask, so the hold's request still leaves first and the flash never delays the key.

## Mechanics

- **One flash per key context**, in a map beside `autoHoldTimers`: its kind (`crank` or `chuk`), the one pending timeout (the next crank frame, or the end of the chuk), the frame index into the ramp, the current frame, and the highest RPM seen. Frames are a chain of timeouts rather than an interval, since each is shorter than the last. Key up and dial up end a crank. It is cleared on `onWillDisappear` and on `onDidReceiveSettings`, so a Mode change during the hold's await stops it.
- **The flash owns the image while it runs.** `updateDisplayFromTelemetry` skips the render while a flash runs, so the throttle and the flash never draw over each other, except that a tick reading the engine caught ends a crank. Each crank frame reads the same telemetry and ends it too, so ending does not depend on a subscriber tick arriving. When a flash ends, the context's `lastState` entry is cleared and the real state is drawn at once.
- **The current frame is part of the resolved state.** While a flash runs, the Starter's resolved telemetry state carries the frame's lit flag, so the regenerate callback a global-settings change runs draws the frame and not the bare state.
- **Frames go straight to `updateKeyImage`**, as Session Info's flashes do, not through `IconUpdateThrottle`.
- **Artwork:** `starterArtwork` gains a separate "lit" input next to the state, so a frame can draw the full-strength badge over an OFF status bar, or the dimmed one over an ON bar for the chuk. The existing two-argument output stays identical, so the icon previews do not change.
- **No settings key.** `CarControlSettings` is unchanged, so no stored-settings contract moves. A key that stored the short-lived `starterFlash` during development keeps an ignored extra key.
- **Debug lines** record when a flash starts and its kind, why it ends (`released`, `engine running`, `chuk done`) with the highest RPM it saw, and why a press does not start one.

## Out of scope

- Flashing any other Car Control mode. If someone asks, the same timer serves them.
- A configurable ramp or rate, or turning the flash off.
- A sound on start. The Race Engineer is a separate system.
- Any change to what the Starter key's ON/OFF bar shows. That stays #561's `isEngineRunning`; only the flash's end uses the stricter catch.

## Artifacts

- `packages/iracing-actions/src/actions/car-control/` (action and test) for all three plugins, through the shared action list. The `.ejs` PI does not change.
- Website: the Car Control page's Starter section documents the flash; a changelog fragment under Features.
- No manifest or UUID change.

## Testing

- **Unit tests with fake timers** in `car-control.test.ts`:
  - each crank frame lasts its ramp length, the ramp only shortens, the fastest frame repeats while held, and frames alternate lit and dimmed over the real OFF bar;
  - a cranking engine (stall bit clear, RPM below the floor) keeps the flash running; a caught engine ends it at once on the ON artwork, from a tick or from a frame step;
  - releasing the key ends it at once with no tail;
  - a press on a running engine gives one dimmed chuk of 200 ms, then the ON look, held or released;
  - `hasEngineCaught` for cranking, idle, stalled, no-RPM and N/A telemetry;
  - it flashes while the hold's request is still pending, and the first frame waits for the request to go out;
  - no flash with no Starter binding, or for a press that sent another control; a second press restarts the ramp; `onWillDisappear` and a settings change clear it;
  - the hold and release calls are the same whether it cranks or chuks;
  - `starterArtwork` with two arguments returns exactly the previous output.
- **Manual:** on a Stream Deck, stall the car and hold Starter. The key should flash slowly, then faster, and go green the moment the engine catches, while still held. Release before it catches: the flash should stop at once. Press it with the engine running: one short blink. Check the end-of-flash debug line's max RPM against the 500 RPM floor. Also check a key with no Starter binding: it shows the warning and does not flash.
