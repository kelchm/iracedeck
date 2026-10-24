# Pause Sim greys out when the driver is not alone in the session

> **Issue:** [#1477](https://github.com/niklam/iracedeck/issues/1477) · **Supersedes:** _none_ · **Superseded by:** _none_
>
> Point-in-time design record. The code and `.claude/rules/` are the truth; this is not documentation.

iRacing pauses only when the driver is the only human in the session; AI cars don't count. Car Control's Pause Sim key sends its binding (default `Shift+P`) and draws the same static icon in every session, so in an online race it looks exactly as usable as in an offline test drive. This makes the key look unavailable whenever another human driver is in the session. The issue carries the request; this records how.

## Behaviour

- **Alone, the key looks as it does today.** That covers an offline test drive, an AI race, and a hosted session with no other human driver in it.
- **With another human driver, the key draws the repo's unavailable look** (`shared/unavailable-look.ts`: every colour but the background in grey, the border with them, the art and title faded through `assembleIcon`'s `dimmed`), the look Replay Markers and Replay Control use for a press that would do nothing. A customised key stays recognisable, only switched off. No new icon art is needed.
- **A press still sends the binding in either state.** When iRacing ignores the press nothing happens, and a press the detection got wrong still pauses. This differs on purpose from Wipers and Visor's "neither" state (#1463), where a press could do the wrong thing.
- **Unknown reads as alone.** With no session info, no `DriverInfo`, or no `DriverCarIdx`, the key keeps its normal look. A missing read must not grey out a key that works (the #574 rule). Disconnected, the key falls under the plugin's usual offline handling, as every key does.
- **Who counts as another human:** a `DriverInfo.Drivers` entry that is not the player's car (`CarIdx !== DriverCarIdx`), not AI (`CarIsAI !== 1`), not the pace car (`CarIsPaceCar !== 1`) and not a spectator (`IsSpectator !== 1`). Leaving spectators out is the one judgement here that is not measured: a spectator in a hosted session does not drive, but whether iRacing refuses to pause while one is connected has not been tested. The manual test settles it; if a spectator does block the pause, they count and the rule changes in one line.
- **The key follows the session live.** A driver joining or leaving updates `DriverInfo`, and the key redraws on the next telemetry tick. It redraws only when the answer changes, not on every tick.

## Architecture

- **The rule is a pure helper in `@iracedeck/iracing-sdk`'s `session-utils.ts`**, beside `classifyCarNumberTarget`, which already reads `CarIsAI` and `CarIsPaceCar` from the same entries: `isOnlyHumanDriver(sessionInfo): boolean | null`, `null` when the session info can't tell. It counts drivers, so it is not a race order and `race-positions.md` doesn't apply. The SDK is where the session-YAML readers live, and the helper may be wanted elsewhere (a Telemetry Display variable, say), which is a reason not to bury it in the action.
- **Car Control reads it for the Pause Sim mode only.** `pause-sim` joins `TELEMETRY_AWARE_CONTROLS`; `getTelemetryState` adds a `pauseAvailable` field from `isOnlyHumanDriver(this.sdkController.getSessionInfo())` (with `null` → `true`); `buildStateKey` gains a `pause-sim|<available>` branch so the key redraws only when the answer changes; and the icon path spreads `unavailableLook(colors, border)` into `assembleIcon` when it is `false`. Car Control is a `SimIRacingAction` and already reads session info this way for the Enter / Exit / Tow and Pit Speed Limiter keys, so this adds no new direct read (#1351's measured leaks).
- **Per tick, the cost is one pass over at most 64 driver entries** of the session info the controller already holds parsed, and only for keys in the Pause Sim mode. No caching is needed.
- **Shared with #1463 and #1476**, both open, both editing `car-control.ts`, its test and its website page. None touches the Pause Sim mode, so whichever lands later rebases.

## Out of scope

- Disabling the press, or replacing the binding with an SDK command. `docs/keyboard-shortcuts.md` lists Pause Sim as keyboard-only, so the key stays a keybind.
- Showing whether the sim is paused right now (a paused / running state on the key). Which telemetry, if any, reports that has not been looked into; it would be a separate request.
- A replay. Pausing a replay is Replay Control's job, not this key's; in a saved replay the driver list is the original session's, and the key simply shows whatever that list says.
- Team events where several drivers share a car: `DriverInfo.Drivers` has one entry per car, so a teammate waiting in the same car is invisible to this rule. Whether iRacing pauses then is untested, and it is left as it falls.

## Artifacts

- `packages/iracing-sdk/src/session-utils.ts`: `isOnlyHumanDriver` and its tests, exported from the package index.
- `packages/iracing-actions/src/actions/car-control/car-control.ts`: the telemetry-aware Pause Sim path, state key and unavailable look, with tests in `car-control.test.ts`. No PI, setting, binding, comms-catalog or icon change; all three plugins get it through the shared action list.
- Website: the Pause Sim section of `docs/actions/driving/car-control.md`. "Telemetry-aware icon: No" becomes a line saying the key greys out while another human driver is in the session, since pausing only works when you are alone (AI cars don't count), and that a press still sends the binding.
- A changelog fragment under Improvements.
- The `iracedeck-actions` skill's telemetry-aware row names Car Control's pit limiter; it gains Pause Sim.

## Testing

- **Helper:** only the player → `true`; the player and AI cars → `true`; the player and the pace car → `true`; the player and a spectator → `true`; the player and one other human → `false`; no `DriverInfo`, no `Drivers` or no `DriverCarIdx` → `null`.
- **Action:** Pause Sim draws the normal icon when alone or unknown, and the unavailable look with another human; the state key changes only when the answer does; a press sends `carControlPauseSim` in both states; other modes are untouched (their state keys and icons are unchanged).
- **By hand, in iRacing:** an offline test drive and an AI race show the normal key, and the press pauses; a hosted or official session with other drivers shows the greyed key, and the press does nothing. A hosted session you are alone in shows the normal key and pauses. The same hosted session with a second account joined as a spectator shows whether a spectator blocks the pause; record the result and adjust the rule if it does. Check the key on Stream Deck and Mirabox.
