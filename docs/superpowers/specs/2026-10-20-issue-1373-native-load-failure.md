# A failed native addon load is loud

> **Issue:** [#1373](https://github.com/niklam/iracedeck/issues/1373) · **Supersedes:** _none_ · **Superseded by:** _none_
>
> Point-in-time design record. The code and `.claude/rules/` are the truth; this is not documentation.

The issue carries the problem: on Windows a `.node` that fails to load is swallowed by a bare `catch {}`, and `iracing-native` then serves the mock's rotating telemetry as if iRacing were running. It left one choice open, whether the user sees the failure. Decided with Niklas on 2026-10-20: **logged, shown as a Property Inspector banner, and the iRacing side reports "not connected" instead of fake data.**

## Decision 1: each package says how it loaded

Both `@iracedeck/iracing-native` and `@iracedeck/audio-native` export a status read once at module load:

```typescript
// shape, not final names
export type NativeLoadStatus =
  | { kind: "native" }
  | { kind: "mock"; reason: "platform" | "forced" }
  | { kind: "load-failed"; error: string };

export function getNativeLoadStatus(): NativeLoadStatus;
```

- **`forced`** is `IRACEDECK_MOCK` or a `.mock` file, exactly as today; **`platform`** is any non-Windows host. Neither path changes, so the suite's forced mock (#1084) and macOS development behave as before.
- **`load-failed`** keeps the caught error's message. It is the only new state.
- The type is declared in each package rather than shared: both are zero-dependency leaves, and a shared home would give a native addon its first workspace dependency for one union. `plugin-runtime` reads the two structurally.
- The load decision moves into a pure function (platform, force flag, loader in; addon and status out), so the failed path is testable without a `.node` and without un-mocking the suite.

## Decision 2: a failed iRacing load reports "not connected"

When `iracing_native.node` fails to load, `IRacingNative.startup()` returns `false` and never starts the mock's rotation, so `isConnected()` is `false`, the header and data are `null`, and every key shows its no-iRacing state. The input-side methods (scan keys, clipboard, focus, pointer) keep delegating to the mock's no-ops, and the elevation probe keeps its "no mismatch" answer. Fake telemetry is the harm the issue names; a user with a broken install should see iRacing as unavailable, not see believable numbers.

The forced and platform mocks still serve the rotating telemetry. A fresh clone that skipped `pnpm build:native` now reads "not connected" on Windows rather than fake data; a developer who wants the mock there sets `.mock` or `IRACEDECK_MOCK`, which is what those levers are for.

**Audio keeps its mock on a failed load.** `AudioNativeMock` produces silence and no data, so nothing it does misleads beyond the silence itself, which the banner now explains. Making `initAudioEngine()` fail instead would push a new failure path through audio-service for no user-visible gain.

## Decision 3: logged once per addon, at error

At startup `plugin-runtime` logs one line per addon whose status is `load-failed`, at `error`, naming the addon and carrying the error message (which names the `.node` path and the OS reason, the two things a support log needs). A `native` or `mock` status logs nothing new at `info`; the forced mock already announces itself by being asked for.

## Decision 4: a banner per failed addon

Each failed addon raises an `error`-level record in the run-scoped `_warnings` store (#610, #1014), with its own id (`native-addon-iracing`, `native-addon-audio`), so it shows at the top of every Property Inspector and in the settings window on all three hosts.

- **Wording** says what does not work and the two remedies a user can act on, and points at the log for the rest. Roughly: *"iRaceDeck could not load its iRacing component, so keys cannot read or control iRacing. Reinstall the plugin and check that your antivirus has not quarantined it. The plugin log has the details."* The audio one names the Race Engineer and the radar instead. No file path in the banner: it sits under the Windows profile.
- **Re-asserted on every settings arrival**, beside the voice-script banner (#1064). The condition is fixed for the run, but `_warnings` is run-scoped and asks every producer to re-assert within the run; a set that skips when the record is already identical makes that free. A `native` or `mock` status never raises the record and clears none: nothing else writes these ids.

## Decision 5: the Telemetry Snapshot records it

The snapshot's `environment` section (#1387) gains the two statuses' `kind` (and `reason` for a mock), so a snapshot from an affected user says why its telemetry is empty. The error text is left out: it carries the `.node` path, and that section never holds a path.

## Out of scope

- Retrying a failed load during the run. The causes (quarantine, a missing runtime, a locked file) do not clear by themselves, and a retry loop would need a reload path through every consumer of the addon.
- A key-icon state for the failure. The banner and the "not connected" icons already say it, on every key at once.
- Explicit `IRACEDECK_MOCK` value parsing (#394), in the same modules but a separate change.
- The scenario harness, which constructs `AudioNative` itself and has its own console.

## Testing

- **Unit:** the pure load function for each package returns `native` on a loader that succeeds, `load-failed` with the message on one that throws, `forced` under the force flag without calling the loader, and `platform` off Windows without calling it. `IRacingNative` over a `load-failed` state returns `false` from `startup()` and `null` from `getHeader()`, and still answers the input methods. The plugin-runtime step logs one error line and raises one banner per failed addon, and raises nothing for `native` or `mock`. The snapshot's environment section carries the kinds.
- **Manual:** on Windows, rename `iracing_native.node` in a linked plugin's `bin` tree and restart the deck host: the log has the error line, every PI shows the iRacing banner, keys show no-iRacing with iRacing running. Repeat for `audio_native.node`. Restore both and restart: no banner, real telemetry.
