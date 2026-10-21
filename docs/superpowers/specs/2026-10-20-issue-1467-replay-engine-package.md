# The replay engine is its own package

> **Issue:** [#1467](https://github.com/niklam/iracedeck/issues/1467) · **Supersedes:** _none_ · **Superseded by:** _none_
>
> Point-in-time design record. The code and `.claude/rules/` are the truth; this is not documentation.

The issue carries the problem: since 2026-10-09 about 8k lines of replay engine (the command gate, the cursor, the position state, jump playback, seek, the search queue, the speed state, the lap walks) have grown inside `@iracedeck/iracing-actions`, much of it inside the `ReplayControl` class. That code has no deck-host import and no single owner for the replay's state. The issue left three choices open: where the package boundary runs, how speed is modelled, and what policy camera switches follow. The issue asked for no behaviour change. Niklas decided on 2026-10-20 to accept four small changes that come with giving each of these one owner; they are listed under *Behaviour changes* below.

## Decision 1: `@iracedeck/replay-engine`, injected, beside `replay-store`

A new package `packages/replay-engine`, iRacing-specific, beside `replay-store`. It depends on `@iracedeck/iracing-sdk` (types, `ReplayPosMode`, `replaySpeedFromTelemetry`, `REPLAY_EXIT_GRACE_MS`, the grid helpers), `@iracedeck/replay-store` (types, constants and the session-store singleton the walks and record step read), `@iracedeck/logger` and `@iracedeck/app-constants` (`parseReplayJumpPlayback`), and on nothing else in the workspace.

It does **not** depend on `deck-iracing`, `deck-core`, `settings` or `icons`. `deck-iracing` is the deck seam's sim adapter, not a domain, and the engine has no deck concept. What those packages supply today is injected once, at the composition root:

```typescript
configureReplayEngine({
  controller,            // SDKController (iracing-sdk type): telemetry, session info, getReplayState(), subscribe
  replay: () => getCommands().replay,   // the replay broadcast surface (only the gate may use it)
  camera: () => getCommands().camera,   // camera.switchNum
  now: () => Date.now(),
  logger,                // the engine scopes its own children with createScope
  settings: {
    replayJumpPlayback: () => getGlobalSettings().replayJumpPlayback,
    fastestLapSearchDelayMs: () => getGlobalSettings().fastestLapSearchDelayMs,
  },
});
```

The settings are closures read at the time of use, the shape `initializeSimEventsIracing` already takes in `phases/sim.ts`, so a change in the Settings window takes effect without re-wiring. The gate's existing `ReplayGateEnv` and the search queue's `ReplaySearchQueueEnv` collapse into this one env; the test hooks (`_configureReplayCommandGate`, `_configureReplaySearchQueue`) become one `_configureReplayEngine` override.

**One clock.** Every engine module reads time through `env.now()`, not `Date.now()`: the search queue, seek, jump playback and the walks all take their own time today. Sleeping stays on a real `setTimeout` behind one engine `sleep` helper, so tests can fake both together.

**Boundary guard.** A new `eslint.config.js` block refuses `@iracedeck/deck-core`, `@iracedeck/deck-iracing`, `@iracedeck/settings` and `@iracedeck/icons` in `packages/replay-engine/src`, and the package joins `scripts/deck-core-sim-boundary.test.mjs` (or a sibling test in the same shape) so removing the lint block turns a test red.

## Decision 2: what moves, and what stays

**Moves into the engine:**

- From `iracing-actions/src/shared/`: `replay-command-gate`, `replay-cursor`, `replay-position-state`, `replay-jump-playback`, `replay-seek`, `replay-frame-move`, `replay-jump-speed`, `replay-timing`, `replay-record-walk`.
- From `actions/replay-control/`: `replay-search-queue`, `replay-speed-state`, `replay-incident-observer`, `replay-record-step`, `green-flag-target`, `lap-walk-probes`, `replay-lap-count`, `replay-speed-format`.
- Out of the `ReplayControl` class and its module scope: `walkToLapStart`, `searchGreenFromPoleCar`, `putReplayBack`, the jump helpers (`runJumpToFrame`, `holdFastestLapSlot` and their kin), the walk's module-level support (the fastest-lap session-map cache, `FastestLapPressCancelled`, the owner names, `readSubSessionId` / `readUserId`, `findFastestLapForCar`, `computeFastestLapBisectionProbe`) and the press's single-flight state (`activeFastestLapPress`).

The extracted walks read telemetry, session info and replay state through the env, not `this.sdkController`, and log through the engine's logger. Where they call back into the deck today, they take no port. `setLocalSpeed` goes away under Decision 4. Redraws come from the speed state's change subscription, and the action keeps `updateAllTelemetryDisplays`. The press's single-flight guard reports an ignored press through `reportReplayPressIgnored`, which lives in the busy indicator. The engine cannot import that, so the guard returns an `ignored` result and the action calls the busy indicator on it.

**Stays in `iracing-actions`:** the busy indicator (`shared/replay-busy-indicator.ts`), every dial-mode file, `replay-speed-ladder`, `replay-car-arrays`, the legacy replay actions (`replay-navigation`, `replay-speed`, `replay-transport`), `replay-markers` and its ops. They import from `@iracedeck/replay-engine` instead of relative paths. Also unchanged: the recorders and record gate in `sim-events-iracing`, and `replay-store`'s section modules.

**Duplicates collapse to one declaration in the engine:** `ReplaySpeed` and `isSameSpeed` (today in both `replay-jump-speed.ts` and the dial files; the dial files re-export the engine's, which is the minimal edit to files the issue otherwise leaves alone), and `HOLD_GAP_MS` (gate and search queue). The settle-polling loops (the gate's `waitForReplaySettled`, the search queue's `holdReading`, the walk's two settle loops, seek's `waitForReplay`) share one engine poll helper where their exit conditions are the same. Where they differ, the difference is kept and the helper takes it as a predicate. This is a cleanup within the move, not a redesign of any loop.

## Decision 3: one camera policy, a shared waiter

Today the dial's Track Order and My Car switches wait behind a running jump (`replay-control-track-order-mode.ts`, a newest-wins waiter over `waitForReplaySettled({ jumpsOnly: true })`), and the keypad's switches send at once. **Both wait behind the jump** (Niklas, 2026-10-20).

The waiter moves into the engine as its own module, `requestCameraSwitch(send, { stillValid })`, and every camera switch the key or dial makes goes through it: `switchToMyCar`, Next / Prev Car, and the car-number switch. While nothing is jumping it sends at once; otherwise it parks the newest request and sends it when jumps settle, dropping it if the wait ends some other way or `stillValid()` is false. `send` is a callback because the keypad's Next / Prev Car are keystrokes, not `camera.switchNum`.

**Not a lane in the gate.** The gate's one newest-wins slot holds a speed, search or pause; a camera switch parked there would displace one of those, so it would need a second slot. The keystroke sends would also need a deck port that the gate does not have. The waiter needs only the gate's public waits. Camera sends stay outside the gate's single-sender rule, as they are today.

**What a camera press cancels.** The dial's switch cancels only the two lap walks (`cancelReplayCursorOwnerIf`). The key's cancels any cursor owner, including a marker or record jump. That contradicts waiting for that jump, so the key now cancels only the walks, like the dial.

## Decision 4: one speed model

Replay speed is recorded in four places today: the key's per-context maps (`replaySpeed` / `replaySlowMotion`, set optimistically even when the gate held or dropped the press), `replay-speed-state`, the gate's `speedSent`, and the cursor's `jumpSpeed`.

`replay-speed-state` moves into the engine and is **the** owner:

- The gate's dispatch feeds it every speed it actually sends, whichever surface asked (key, dial, legacy actions, jump playback). This replaces `noteDialSpeedSent` and the slot listener.
- Telemetry feeds it through `noteTelemetrySpeed` (see the last point of this list for who calls it).
- Every reader (the key's icons, the dial, `getCurrentSpeed`) uses the one read the dial already uses: the waiting speed if one is held, otherwise the resolved live speed.
- The speed state exposes one change subscription (`onReplaySpeedChanged`). It fires when a send changes the speed. That covers sends the engine makes on its own: a slot dispatch, a search's speed and a walk pausing the replay. The keys and the dial redraw from it, because a paused replay may deliver no tick. It replaces `onReplaySpeedSentFromSlot`, `onReplaySearchSpeedSent` and the walk's `onPaused` → `updateAllTelemetryDisplays` call.
- `remembered` and `provisional` are the speed the dial's Play / Reverse resumes. The sends that update them today keep doing so, and no others: the dial's own sends, and any speed the gate's slot dispatches later (today through the slot listener), whichever surface pressed it. A speed a key or legacy action sends directly updates only `pending`, the speed that is shown. So a key's Fast Forward sent at once does not change what the dial resumes, as today.
- `noteTelemetrySpeed` is still called by `ReplayControl` on its ticks, so it runs only while a Replay Control context is visible, as today. Moving it to an engine-owned subscription would update the dial's resume speed while nothing is on screen, which is a behaviour change nobody asked for.

The key's per-context maps and `setLocalSpeed` are deleted. A press the gate holds shows as the waiting speed; a dropped press never shows.

The cursor's `jumpSpeed` stays. It is the speed captured to restore after a jump, not a live speed, so it is not a fifth record of the current one. The gate's `speedSent` stays too. It is internal to the gate's dispatch decisions (`currentSpeed`, `startingToMove`, with their 200 ms `SPEED_SENT_HOLD_MS`), and folding it into the speed state's 500 ms hold would change how the gate orders commands, which is out of scope. Nothing outside the gate reads it once its listener is gone.

## Decision 5: leaving the replay is one engine event

No module owns this today. The gate's `leftReplay()` runs only while the gate is busy and does not end the cursor claim. The walk checks `inReplay` only when it starts. The search queue polls its own `replayOnScreen()`. `activeFastestLapPress` is cleared only in a `finally`.

The engine subscribes to the controller and detects the edge of `getReplayState().inReplay` going false. That reading is the existing debounced state behind `REPLAY_EXIT_GRACE_MS`, so a replay blip does not count as leaving. A disconnect counts too: it arrives as a null tick, not a `SessionTick` advance. On that edge the engine emits one internal `replayLeft { reason: "left" | "disconnected" }`, and every holder of replay state honours it:

- the gate runs its `leftReplay()` on `left` and its existing `disconnect()` on `disconnected`, unchanged in what each clears and in the reason each reports to the busy indicator;
- the search queue ends its search as `gave-up`;
- a running lap walk (fastest-lap or green-flag) stops, its cursor claim ends with a `replay-left` reason, and its press resolves as cancelled.

**The edge is held through the plugin's own seek** (added 2026-10-21, once Task 7 read the code). In a live session the debounced state gives the replay up about a second into any long seek the plugin makes itself (the search queue's `replayOnScreen()`, and the #801 spec's in-replay guard). A walk's first probe on a long tape is such a seek, so the bare edge would cancel the walk that made it, which would be a fifth behaviour change. So while the debounced state reads false and a move of the plugin's own still stands, `left` is not emitted yet:

- a gate position target that has not read back holds it for that target's own wait (`replayTargetGiveUpMs`, the larger of `REPLAY_TARGET_GIVE_UP_MS` and twice the jump's length at iRacing's search rate), counted from the edge, or from the target's send when it went out after the edge, so a walk's next probe gets its whole wait;
- a seek that shows without a target holds it for `REPLAY_TARGET_GIVE_UP_MS`.

`left` is emitted when the state still reads false once nothing holds it. If the state reads true again first, nothing is emitted. The hold is bounded on purpose, for the reason #801 gives: an unbounded hold would never let the event fire during a walk, because the walk always holds a target. The cost is that behaviour change 4 can land up to one jump's wait after the replay is really left, never before. A disconnect is not held.

The cursor does **not** end every claim on `replayLeft`. Two holders must keep theirs. A jump whose position target the gate deliberately keeps through `leftReplay()` still owns that target and waits out its own wait. A Jump to Live leaves the replay on purpose (`env.leftForLive()`). Those claims end as they do today. Only the lap walks' claims, which today run until their timeout, end on the event.

`replayLeft` is engine-internal. It is not added to the `@iracedeck/event-bus` sim-event catalog, which is a published contract and has no consumer for it. The search queue's own `replayOnScreen()` poll stays as a second check inside its read loop, because it also covers the gate still holding a target.

## Decision 6: the composition root

A new startup phase `initReplay(core)` runs right after `initSettings` and before `registerActions`. It does two things:

1. `configureReplayEngine(...)` as above, with the engine's logger created here. The gate and search-queue loggers are no longer set from `ReplayControl`'s constructor. The busy indicator stays in `iracing-actions`, which has no registration entry point of its own. Its logger setter is exported from the package barrel and called in `registerActions`, just before the actions are constructed.
2. The replay store's wiring moves out of `initSettings` into this phase: `initializeReplaySessionStore`, its exit flush and its five `core.bus` subscriptions (`replay.lapStarted`, `lapTimed`, `greenFlag`, `sessionStarted`, `incident`).

`start-plugin.test.ts`'s recorded order is updated to the new phase, and `plugin-structure.md`'s startup-phase list says where the replay engine and store are wired.

## Behaviour changes

Accepted by Niklas on 2026-10-20. Nothing else changes behaviour. Where a change of owner could have changed one, Decision 4 keeps the existing behaviour on purpose (the dial's resume speed, when telemetry speed is noted), and the tests prove it unchanged.

1. A camera switch on the key waits for a running jump to finish, like the dial's (Decision 3).
2. A camera switch on the key no longer cancels a marker or record jump; it cancels only a lap walk (Decision 3).
3. The key's icon no longer shows a speed the gate dropped. A press the gate holds shows as the waiting speed until it is sent, as the dial's display already does (Decision 4).
4. A fastest-lap or green-flag walk stops when you leave the replay, instead of running until it times out (Decision 5).

These four get one changelog fragment, under `Improvements`, about the replay controls being more consistent.

## Out of scope

- The busy indicator, and the dial-mode files beyond two edits: re-pointing a duplicated type or function at the engine, and the camera waiter moving out of `replay-control-track-order-mode.ts` into the engine (Decision 3). Also out of scope: the legacy replay actions, `replay-markers`' surfaces, the recorders and record gate in `sim-events-iracing`, and `replay-store`'s section modules.
- Any change to how the gate orders replay commands, its slot semantics, or its timing constants.
- Adding a replay event to the `event-bus` catalog.
- Redesigning the walks' search algorithms. They move and change their reads, not what they search for.
- Supporting a second sim. The engine is iRacing-specific by name and by dependency.

## Testing

- **Unit tests move with their modules**, about 7k lines. Where they mock `@iracedeck/settings` or `@iracedeck/deck-iracing`, they configure the engine env instead. The fastest-lap and race-start walk tests (`replay-control.test.ts` `jump-to-fastest-lap`, `replay-control-race-start.test.ts`) are split. The walk's own behaviour is tested in the engine against a fake controller and clock. The action keeps a thin test that the press calls the engine and redraws.
- **The single-sender guard** (`replay-command-gate.single-sender.test.ts`) moves to the engine. It scans both `replay-engine/src` and `iracing-actions/src`, and also refuses any engine module other than the gate that reads the injected `replay` surface. Its positive control (a planted violation fails it) is kept.
- **The boundary guard** from Decision 1 has its own positive control.
- **New tests for each behaviour change:** key camera switch parked behind a jump and sent after it settles (and dropped when the wait fails); key camera press does not cancel a marker jump's claim; a key speed press the gate drops never shows as the icon's speed, and one it holds shows as the waiting speed; `replayLeft` on the debounced exit edge and on a disconnect, ending the cursor claim, the search and a running walk; a replay blip inside the grace window does not emit it; an edge during the plugin's own long seek is held, emitting nothing when the target reads back and `left` once the target's wait lapses with the state still false. The two kept behaviours are pinned too: a key's speed press sent at once updates the shown speed but not the dial's resume speed, while one the gate's slot dispatches later updates both, and `onReplaySpeedChanged` fires for a slot dispatch, a search's speed and a walk's pause, so the icons redraw with no tick.
- **`start-plugin.test.ts`** records the new phase and the store wiring's new position.
- **The full gate:** `install`, `build`, `typecheck`, `format`, `lint`, `test`.
- **Manual, in iRacing (Niklas):** fastest-lap and race-start jumps from the key and the dial; a camera switch pressed on the key during a jump (it lands after the jump); speed keys while a jump runs; leaving the replay mid-walk; markers and record step still working.

## Staging

One issue, one worktree (`../ir-1467`), one PR, committed in logical steps. Two steps are seams that later steps build on, and each gets a task-scoped review before the next one starts. The first seam is the package skeleton, the env and the leaf modules, together with the `initReplay` phase. The second is the gate, jump playback, search queue and incident observer on that env. Speed, the camera waiter, `replayLeft` and the walk extraction follow, in that order, because the walk extraction needs `setLocalSpeed` gone first. Moving the store's wiring is independent of all of these. Docs come last: the package `CLAUDE.md`, the `iracing-actions` `CLAUDE.md`, `.claude/CLAUDE.md`'s package list, `plugin-structure.md`, the Architecture page and the README.
