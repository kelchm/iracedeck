# Releasing the audio stream miniaudio abandons at a reroute

> **Issue:** [#1414](https://github.com/niklam/iracedeck/issues/1414) · **Supersedes:** _none_ · **Superseded by:** _none_
>
> Point-in-time design record. The code and `.claude/rules/` are the truth; this is not documentation.

## Problem

When miniaudio follows a Windows default-device change, it reinitialises its device on the new endpoint and drops its pointer to the old `IAudioClient` without releasing it. Its own comment in `ma_device_reinit__wasapi` calls this a known leak: releasing the client inside Windows' notification callback deadlocks, and it has nowhere else to do it. Nothing can reach that client afterwards. `ma_device_uninit` releases only the current one.

One change is several reroutes (four on the machine measured: two roles, each reported twice), so clients are abandoned on the device the stream moves to as well as on the one it left.

Measured on 2026-10-11 with `powercfg /requests` sampled every 2 s (the table is in the issue): on plain hardware devices an abandoned client holds nothing. When the stream is rerouted **onto a SteelSeries Sonar device**, the audio request for the headset Sonar plays through stays until our process exits. The PC is then held awake until the deck host is restarted, which is the thing #849 tore the engine down to prevent, and nothing in the plugin or its log says so.

It needs the plugin's device to be open when the default changes: during a sound and the 5 s after it today, and for the whole iRacing session once #1347 holds the device.

## Decision

### 1. The addon releases what miniaudio abandons

The addon keeps track of the playback `IAudioClient` the device holds. When miniaudio reports a reroute, the client it held before is the abandoned one, and the addon releases it. That is the release miniaudio's comment says is missing, done where it is safe.

This is the one exception to the rule the #1411 spec set, that our threads never call into miniaudio's audio client. It holds because the client is no longer miniaudio's: no thread of miniaudio's has the pointer any more, so nothing can be using it or waiting on it.

### 2. On the helper thread, never in the callback and never on the JS thread

The release runs on the helper thread #1411 added for naming the session. miniaudio's notification callback only hands the pointer over: it reads the device's new client, swaps it with the tracked one under the helper's mutex, and queues the old one. No COM call is made there.

The helper releases queued clients when it serves the rename for the same reroute, about half a second after the last round, so it normally runs with no notification in flight. If one is, the release waits for it while holding nothing any other thread needs, the same reasoning as the rename.

The JS thread never releases a client. A release that has to wait on Windows must not be able to hold up a key press.

### 3. The helper thread no longer depends on a session identity

Since #1411 the helper thread starts when a session identity is first set, because naming was its only job. A process that plays audio without an identity (the scenario harness, a script) would have nobody to release for it. The thread therefore starts when the audio context is created on WASAPI (`initAudioEngine`), which every engine needs first. `destroyAudioEngine` and the environment cleanup hook still stop and join it, and release whatever is still queued first.

A pending rename is cancelled when the engine is torn down. A queued release is not: the abandoned client is independent of the engine and stays abandoned.

### 4. Never the client the device holds now

Releasing a client miniaudio still owns would crash the plugin, so the rule is explicit: a client is queued only when it differs from the one the device holds at that moment. The tracked pointer is written on the JS thread at engine creation and on the notification thread at a reroute, and a reroute can land between miniaudio creating the device and the addon reading its client. The comparison makes that order harmless: at worst one abandoned client goes untracked, which is today's behaviour for that one.

### 5. Tied to the miniaudio version that leaks

If a later miniaudio releases the old client itself, ours becomes a second release of freed memory. The addon therefore fails to compile against any miniaudio version other than the one this was read against (0.11.25), with a message pointing at `ma_device_reinit__wasapi`. An upgrade has to re-read that function and either keep, change or delete this code.

### 6. No new log line

The JS side learns nothing new: `Audio device rerouted` (#1330) already records the reroute, and a release has no outcome worth a line. Failures cannot be observed from here, and there is no fallback to report.

## Alternatives considered

- **Take the reroute over from miniaudio:** turn its automatic stream routing off, watch for default changes ourselves, and tear the engine down and recreate it from the JS thread. Nothing would be abandoned, and miniaudio would stop doing its stop / reroute / start on Windows' notification thread, where both #1411 lock-ups came from. It costs much more: `ma_engine_config` exposes no WASAPI options, so the addon would own the `ma_device` and feed the engine itself, and a sound playing across the change would be cut or would have to be carried over by hand, where today it continues. Worth it if miniaudio's reroute causes a third problem; not for this one.
- **Patch the vendored miniaudio to release the client:** it has the same problem it describes, nowhere safe to do it, so it would need a thread of its own, and the patch would have to be carried through every upgrade.
- **Tear the engine down after a reroute:** it releases the current client, not the abandoned ones, which is where the request is held.
- **Leave it:** it needs a Sonar-style device and a default change at the wrong moment, but the result is a PC that silently no longer sleeps, and #1347 makes the wrong moment the whole session.

## Out of scope

- **Virtual audio devices other than Sonar:** not measured. The fix does not depend on the device: every abandoned client is released.
- **Why Sonar keeps the headset's stream open:** read as "while any client exists on its device", not measured, and not ours to change.
- **A default change while the plugin's device is closed:** nothing is rerouted, so nothing is abandoned.
- **miniaudio doing the reroute on Windows' notification thread:** see the first alternative.
- **Capture and loopback devices:** the addon opens playback only.

## Testing

No unit test can reach this: it is COM on a live device, and the suite runs against the native mock. The TypeScript side does not change.

- **The measurement that found it, repeated after the fix:** with `powercfg /requests` sampled from an elevated terminal, open the device on a quiet output (the monitor's), switch the default to a Sonar device, release the device and keep the process alive. Before the fix the headset's audio request stays until the process exits (three runs of three). After it, the request is gone within a few seconds of the release. The controls (no switch; monitor to headphones; headphones to monitor) stay clean. Measured on 2026-10-11, the two addons back to back: held for the 24 s until the process exited before, gone at the first sample after the release with the fix.
- **The probe gains the mode this needs:** release the device and stay alive, so the measurement is repeatable from `packages/audio-native/scripts/`.
- **No new lock-up:** the #1411 probe runs, `--busy 15` and `--busy 15 --cycle 150`, through a hundred or more scripted default changes each, with no stall. The release is a new wait on Windows from the helper thread, and this is the check that it never reaches the JS thread.
- **Many reroutes, one process:** the process's handle count across a hundred default changes (404 reroutes) with the device held open, measured on the addon before and after. Measured on 2026-10-11: 274 to 3910 before, about nine handles per reroute, and 274 to 275 with the fix.
- **In the plugin:** a Settings **Test** sound, a switch of the Windows default to a Sonar device within the 5 s the device stays open, then ten seconds of silence: `powercfg /requests` shows no audio request once nothing else is playing, without restarting the deck host.
- **On the #1347 branch, rebased over this:** the same with the device held, switching the default several times during a session and checking after iRacing has exited.
