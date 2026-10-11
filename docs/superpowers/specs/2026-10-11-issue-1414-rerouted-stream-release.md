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

At the line where `ma_device_reinit__wasapi` drops its old playback client, the vendored `miniaudio.h` hands the pointer to the addon first, and the addon releases it later. That is the release miniaudio's comment says is missing, done where it is safe.

This does not break the rule the #1411 spec set, that our threads never call into miniaudio's audio client: the client is no longer miniaudio's. No thread of miniaudio's has the pointer any more, so nothing can be using it or waiting on it.

### 2. A hook in the vendored header, not a guess from outside

The vendored `miniaudio.h` carries one local change, marked as ours: a macro call right before the pointer is set to NULL, which `addon.cc` defines as a function of its own. It is the first change this repo makes to that file.

The first implementation avoided it and worked out which client had been abandoned by watching the device's pointer from outside. The code review showed what that costs: a reroute that lands inside `ma_engine_init`, before the addon's first look, is missed (one client per engine creation, in the same window a sound-triggered open hits), and a failed init could leave a remembered pointer that was later released a second time. With the hook there is nothing to remember and nothing to miss, and the client a reroute abandons when it then fails half-way is caught too.

### 3. On the helper thread, never in the callback and never on the JS thread

The release runs on the helper thread #1411 added for naming the session. The hook runs on Windows' notification thread, inside miniaudio's reroute, so it only queues the pointer under the helper's mutex. No COM call is made there.

The helper releases each client one second after it was abandoned. That is twice the rename's delay, so the rename for the same reroute is served first and never queues behind a release that has to wait on Windows. Each client has a due time of its own: a default device that keeps changing (a headset in a reconnect loop) cannot hold the older ones back, as one shared, restarted delay would. If a notification is in flight when a release runs, the release waits for it while holding nothing any other thread needs, the same reasoning as the rename.

The JS thread never releases a client. A release that has to wait on Windows must not be able to hold up a key press.

### 4. The helper thread no longer depends on a session identity

Since #1411 the helper thread starts when a session identity is first set, because naming was its only job. A process that plays audio without an identity (the scenario harness, a script) would have nobody to release for it. The thread therefore starts when the audio context is created on WASAPI (`initAudioEngine`), which every engine needs first. While no helper runs, the hook queues nothing, and the client stays abandoned as before.

A pending rename is cancelled when the engine is torn down. A queued release is not: the abandoned client is independent of the engine and stays abandoned.

Stopping the helper differs by who asks. `destroyAudioEngine` stops it after the engine is gone, when no reroute can be in flight, and the helper releases what is still queued on its way out. The environment cleanup hook can run with an engine alive and a reroute in flight, and the JS thread waits for the helper there, so it drops the queue: the process is ending.

### 5. A miniaudio upgrade cannot lose or double the release silently

An upgrade replaces the header and with it the hook. `addon.cc` checks at compile time that the hook is there and refuses to build without it, with a message pointing at `ma_device_reinit__wasapi`. Whoever upgrades re-reads that function: if it still drops the client without releasing it, the hook goes back in; if it releases the client itself, the hook and the release are deleted, because ours would then be a second release of freed memory.

### 6. No new log line

The JS side learns nothing new: `Audio device rerouted` (#1330) already records the reroute, and a release has no outcome worth a line. Failures cannot be observed from here, and there is no fallback to report.

## Alternatives considered

- **Take the reroute over from miniaudio:** turn its automatic stream routing off, watch for default changes ourselves, and tear the engine down and recreate it from the JS thread. Nothing would be abandoned, and miniaudio would stop doing its stop / reroute / start on Windows' notification thread, where both #1411 lock-ups came from. It costs much more: `ma_engine_config` exposes no WASAPI options, so the addon would own the `ma_device` and feed the engine itself, and a sound playing across the change would be cut or would have to be carried over by hand, where today it continues. Worth it if miniaudio's reroute causes a third problem; not for this one.
- **Patch the vendored miniaudio to release the client itself:** it has the same problem it describes, nowhere safe to do it, so it would need a thread of its own inside the header. The hook is the small version of this: miniaudio hands the pointer over and the addon's existing thread does the rest.
- **Work out the abandoned client from outside, with no change to the header:** the first implementation. Rejected after review, for the reasons in decision 2.
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

- **The measurement that found it, repeated after the fix:** with `powercfg /requests` sampled from an elevated terminal, open the device on a quiet output (the monitor's), switch the default to a Sonar device, release the device and keep the process alive. Before the fix the headset's audio request stays until the process exits (three runs of three). After it, the request is gone within a few seconds of the release (the release of the clients runs one second after the switch, so a device released sooner than that keeps the request for the remainder). The controls (no switch; monitor to headphones; headphones to monitor) stay clean. Measured on 2026-10-11, the two addons back to back: held for the 24 s until the process exited before, gone at the first sample after the release with the fix.
- **The probe gains the mode this needs:** release the device and stay alive, so the measurement is repeatable from `packages/audio-native/scripts/`.
- **No new lock-up:** the #1411 probe runs, `--busy 15` and `--busy 15 --cycle 150`, through a hundred or more scripted default changes each, with no stall. The release is a new wait on Windows from the helper thread, and this is the check that it never reaches the JS thread.
- **Many reroutes, one process:** the process's handle count across a hundred default changes (404 reroutes) with the device held open, measured on the addon before and after. Measured on 2026-10-11: 274 to 3910 before, about nine handles per reroute, and 274 to 275 with the fix.
- **In the plugin:** a Settings **Test** sound, a switch of the Windows default to a Sonar device within the 5 s the device stays open, then ten seconds of silence: `powercfg /requests` shows no audio request once nothing else is playing, without restarting the deck host.
- **On the #1347 branch, rebased over this:** the same with the device held, switching the default several times during a session and checking after iRacing has exited.
