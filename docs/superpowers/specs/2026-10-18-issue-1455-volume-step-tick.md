# Volume step tick

> **Issue:** [#1455](https://github.com/niklam/iracedeck/issues/1455) · **Supersedes:** _none_ · **Superseded by:** _none_
>
> Point-in-time design record. The code and `.claude/rules/` are the truth; this is not documentation.

## The decision

Every volume step made from the deck plays one short tick: higher for up, lower for down.

- **What counts as a step.** A tap of an iRacing volume binding (Spotter Louder / Quieter, Master and Voice Chat volume up / down) from a key or a dial detent, and a change of iRaceDeck's own `raceEngineerVolume` or `radarVolume` from a key or a dial detent. Mute / Unmute, Mute a Driver, Skip Spotter Call and Push to Talk are not steps and stay silent (below).
- **What.** Two new clips, `packages/audio-assets/sfx/IRD-volume-up.wav` and `IRD-volume-down.wav`: one short dry tick each, under 40 ms, the up tick pitched clearly above the down tick.
- **Where it plays.** A new mixer channel, `AudioChannel.Feedback`, which belongs to no bus. Each tick is played at a level the caller passes.
- **How loud.** A Race Engineer or Radar step ticks at that level's new value; an iRacing step ticks at the Radar Volume level. Both are floored at `VOLUME_TICK_FLOOR` so the tick never drops below audibility.
- **No switch.** Decided by the maintainer: no setting, no checkbox. It plays whatever the Race Engineer, radar or any callout toggle says.

It is not a Race Engineer callout: there is no bus event, no contract, no voice-pack involvement and no harness shortcut. It is deck feedback, and it lives in `iracing-actions` beside the volume helpers it reports on.

## The sound

Two clips, one per direction, each a single tick.

- **Why two.** The requester asked to count steps; a direction cue comes free with the count and catches the commonest dial mistake, turning the wrong way, on the first detent instead of the fifth.
- **Why a single tick, not a two-note figure.** #1161's exit chirp rises and #1272's limiter beep falls, and those shapes already mean "limit off, go" and "limiter on". A rising/falling pair here would borrow those meanings. A two-note figure is also two to three times longer, too long to repeat per detent on a fast spin.
- **Why pitch rather than timbre for direction.** Higher-for-louder is the mapping drivers already expect from every OS volume control, so it needs no learning.
- **Distinct from the existing tones.** It must not be mistaken for `IRD-pit-speed-warning.wav`, the repeating "too fast" tick, or the radar proximity tones in `sfx/radar/`. It is shorter and softer than the speeding tick and has no left/right placement. The final pair is picked by ear from candidates, as #912's tick was, with its gain baked into the asset.
- **WAV, not MP3.** An MP3 frame carries encoder delay and padding of the same order as the whole tick, which would make the tick late and its spacing uneven. `sfx/` is copied into the plugins unchanged (`audio-assets` `CLAUDE.md`), so the WAV ships as authored. `IRD-pit-speed-warning.wav` is the precedent.
- **One pair for every level.** The key or dial the driver touched already says which level moved. Six clips, one pair per level, would be sounds to learn for no information.

## How loud it plays

**Race Engineer and Radar steps tick at the new level.** The tick is then a sample of the result: a step up is heard a little louder than the step before, and the driver knows roughly where the level sits without waiting for the next callout or radar alert. The level read is the persisted value (`raceEngineerVolume` / `radarVolume`), not the bus's effective volume, so a Race Engineer step taken while the Race Engineer is off still ticks at the value it set, while `applyRaceEngineerAudio()` keeps the Voice bus muted as today.

**iRacing steps tick at the Radar Volume level.** iRaceDeck cannot follow iRacing's master, voice chat or spotter level: none is in telemetry. Radar Volume is the slider every code-owned tone follows (#912's speeding tick, #1161's exit chirp, #1272's limiter beep), so a driver who wants all iRaceDeck tones quieter has one slider for it. Rejected:

- **The Race Engineer volume.** It is the engineer's voice level; a driver who turns the engineer down to hear the spotter better would lose the tick that helps adjust the spotter.
- **The Background volume.** Users turn it down for the ambience, often for audio-processing sensitivities (#471); the tick would vanish with it. #1272 rejected it for the same reason.
- **A fixed level.** With no toggle, the slider is the only control the driver has over the tick; a fixed level would leave none.

**The floor.** The service's perceptual curve (exponent 2.5, #531) puts 10 % at about −50 dB and 20 % at about −35 dB, so a tick played at a low level is inaudible in a car, which is the range where counting matters most when turning something down. Below `VOLUME_TICK_FLOOR` (20 on the 0–100 scale, tuned by ear in the manual test) the tick plays at the floor. Above it, it tracks the level. The cost is that the preview stops being literal below the floor, where a literal preview could not be heard anyway. The same floor means Radar Volume 0, which silences the radar, does not silence the tick, as "always plays" requires.

## A channel of its own

**Decision: `AudioChannel.Feedback`, the next free channel index, routed to no bus, played at a level passed per call.**

Why not an existing channel:

- **Radar.** The proximity radar ticks every 180–250 ms with a car alongside and #912's speeding tick every 300 ms, and `playOnChannel` replaces what a channel is playing, so either would cut the volume tick or be cut by it. #1272 measured the same exposure for its beep.
- **Voice.** It goes through the interpreter's queue, and a tick played on it would cut the engineer mid-word, for a reason the driver did not intend.
- **SFX / Ambient.** They frame callouts and carry the pit bed, both on the Background slider (rejected above as a level source).
- **`Cue`** (planned by #1161 / #1272, routed to the Alerts bus). It would solve the cutting but not the level: a Race Engineer step must tick at the Race Engineer level, which the Alerts bus does not carry. Sharing it would also let a fast dial spin cut a pit exit beep that iRacing's exit line triggers on its own timing.

Why no bus: the tick's level depends on which level was stepped, so it changes per tick. Routing the channel to a bus and overriding the level per play would leave a later `setBusVolume` on that bus rewriting a tick mid-play to the wrong level. A channel no bus owns is the honest shape: `CHANNEL_BUS_MAP` gains an explicit "no bus" entry for it (not a missing key, so the exhaustiveness the table relies on stays), `setBusVolume` skips it, and a new service method plays a clip on a channel at an explicit 0–1 level through the same perceptual curve the bus setters use. This is the one exception to "every channel routes to exactly one bus", and the header comment in `audio-service.ts` says so.

Rejected: **one channel per stepped bus** (a Voice-bus feedback channel and an Alerts-bus one). It doubles the channel cost to avoid a per-play level, which the native `playOnChannel` already takes as an argument.

What the channel touches is what #1272's spec lists for `Cue`: `IRD_MAX_CHANNELS` in `addon.cc` (sizing `g_channels`, `g_completionTSFN`, `g_tsfnRegistered`), the enum in `audio-native` and `audio-service`, the mock, and in `audio-service` the end-callback registration loop, the `stopAllChannels` loop and the four-element `channelVolumes`, `channelCallbacks` and `channelActive` arrays. The end callback is load-bearing: the #849 lifecycle releases the device only when every channel is idle, and a channel without one would hold the device open and block PC sleep. Whichever of #1161, #1272 and this issue lands first replaces those hardcoded four-element shapes with ones sized from the channel count, so each later channel is an enum entry and a constant bump. The index is whichever is next free when this lands (4 before `Cue`, 5 after).

**A stale binary degrades to silence.** Every native entry point rejects `channel >= IRD_MAX_CHANNELS`, so a `.node` built before this change refuses the new channel: the play returns false, the service never marks it active, and the tick is silent rather than a crash or a stuck device. `pnpm build:ts`, or a rebuild while a deck host holds the DLL, produces exactly that binary (the build notes in `packages/audio-native/CLAUDE.md`), so the manual test needs a full native rebuild with the deck host closed.

## When it ticks, and when it does not

**iRaceDeck's own levels: one tick per step actually applied.** The tick lives in `stepRaceEngineerVolumeBy` / `stepRadarVolumeBy` in `iracing-actions/src/audio/audio-volume.ts`, the single path every surface steps through: the Audio Controls keys and dial (through `INTERNAL_AUDIO_BUSES`) and the Pit Crew key's hidden Radar Volume mode (`stepRadarVolume`). Putting it there rather than in each surface means a surface added later ticks without remembering to. The number of ticks is `|next − current| / VOLUME_STEP`, so a press at 100 going up, already a no-op that persists nothing, plays nothing. Silence at the end is information the iRacing side cannot give: twenty ticks down from 100, then nothing, means 0.

**iRacing levels: one tick per tap sent while the sim is connected.** The keys and the dial already know whether a tap went out: `tapBinding` returns false for an unset binding, a failed key send or a SimHub role that did not start. A tap that returned false ticks nothing; a key with no binding sends nothing and is silent. A tap that returned true ticks only when `getSimConnection().isConnected()`, because the keystroke is sent whether iRacing runs or not (`keyboard-service.ts` focuses iRacing if it can and sends regardless), and a tick for a keystroke that reached no simulator would be a false confirmation. The tick cannot know whether iRacing acted on the key, nor whether its level was already at the top or bottom; iRacing exposes neither. So at iRacing's limit every press still ticks. That is the documented limitation, and the website says so beside the existing "no level to display" note.

The paths: Audio Controls' Voice Chat and Master `volume-up` / `volume-down` keys (`executeControl`), the dial's Voice Chat, Master and Spotter rotation (one tick per tap in the existing loop, capped at `MAX_TAPS_PER_EVENT` like the taps themselves), and AI Spotter Controls' `louder` / `quieter` controls. AI Spotter Controls' other controls (Damage Report, Weather Report, Toggle Report Laps, Announce Leader, Skip Spotter Call) are not steps.

**Not steps, so silent:**

- **Mute / Unmute.** A toggle, not a step. For the iRacing categories iRaceDeck cannot see the mute state, so a tick could say neither on nor off; for Race Engineer and Radar it flips a feature gate whose state the key and dial strip already show, and the Race Engineer's speaks its own acknowledgement.
- **Mute a Driver** and **Skip Spotter Call.** Blind one-shot commands; there is no amount to count.
- **Push to Talk.** A hold with no amount, and a tick at the moment the driver opens the microphone is the sound most likely to be transmitted to the other drivers.
- **The settings window and Property Inspector sliders.** A drag passes through many values and would rattle; each slider already has a Test button that plays the real sound at the chosen level.

**The Race Engineer and radar switches do not gate it.** "No toggle" means the tick is deck feedback, not part of the Race Engineer: a driver with the engineer switched off who adjusts the spotter hears the tick, and one who steps the Race Engineer volume while the engineer is off hears the step at the value it stored. The `Feedback` channel is on no bus, so the master gate's muting of the Voice and Background buses never reaches it.

## Dial spins and repeated presses

A Stream Deck+ dial or Mirabox knob coalesces a fast spin into one rotate event carrying several detents. One tick per event would undercount exactly when the driver turns quickly, and several ticks played at once on one channel would replace each other and be heard as one. So the tick module keeps a short queue: the first tick plays at once, each later one waits until `VOLUME_TICK_SPACING_MS` (about 70 ms, longer than a tick) after the previous, and a backlog beyond `VOLUME_TICK_MAX_QUEUED` (5, matching `MAX_TAPS_PER_EVENT`) is dropped. Order is kept, so a reversal mid-spin is heard as the reversal. A drop only happens above roughly 14 steps a second, where nobody is counting; a single press is never delayed.

Keys have no hold-to-repeat: one key down is one step and one tick, and none is added. Mashing a key goes through the same queue.

Rejected: **one tick per rotate event** (the undercount above), and **one tick per event with a pitch or length scaled to the detent count** (a sound to decode instead of a count).

## The playback device

A tick reaches an idle-stopped device (#849) like any other sound: `playOnChannel` starts the stream on demand and the idle release re-arms `IDLE_STOP_DELAY_MS` after the last channel ends. A dial spin keeps it warm through the 5 s debounce. While iRacing runs, the #1347 session hold keeps the device open anyway, so every iRacing step and every in-session step ticks without a device start. The only ticks that pay a device start are Race Engineer or Radar steps taken with iRacing closed; the manual test checks that a single press after the idle window still ticks whole. Rejected: holding the device while an Audio Controls or AI Spotter Controls key is visible, which would block PC sleep on a deck left on the desk, the exact failure #849 removed.

## Relation to #269

#269 asks for a configurable sound on any button press, with a global switch and a per-action override. This is narrower: volume steps only, fixed, always on, and directional. It does not wait on #269. If #269 lands later, it can reuse the `Feedback` channel and the per-play level, and a volume step plays this tick in place of #269's press sound rather than both.

## Out of scope

- Any sound for non-volume controls (mute toggles, Push to Talk, spotter reports) or for any other action; that is #269.
- A setting, a separate volume, or a per-action switch for the tick.
- A distinct "limit reached" sound at 0 or 100.
- Ticks for the settings window and Property Inspector volume sliders.
- Reading or estimating iRacing's own master, voice chat or spotter levels.
- Moving #912's speeding tick or the radar tones to another channel.

## Testing

Suite:

- Tick module: up and down pick their clips; the level passed is the given level above `VOLUME_TICK_FLOOR` and the floor below it, 0 included; the first tick plays at once, later ones at the spacing; the queue drops beyond `VOLUME_TICK_MAX_QUEUED` and keeps order across a reversal.
- Step helpers: `stepRaceEngineerVolumeBy(+1)` from 50 ticks once at 55; `stepRadarVolumeBy(+3)` from 95 ticks once (one applied step); a step at the clamp ticks nothing; a Race Engineer step with the engineer off ticks at the stored value; the Pit Crew hidden Radar Volume mode ticks through `stepRadarVolume`.
- Audio Controls: Voice Chat / Master volume keys tick on a sent tap while connected, not on an unset binding, not on a false tap, not while disconnected; the dial's Voice Chat / Master / Spotter rotation ticks once per tap sent, capped at five; Race Engineer / Radar rotation ticks per applied detent; Mute / Unmute, Mute a Driver, Skip Spotter Call and Push to Talk, on keys and dial, never tick.
- AI Spotter Controls: Spotter Louder / Quieter tick on a sent tap while connected; every other control never ticks.
- Channel: `Feedback` is in no bus, and `setBusVolume` on each bus leaves it alone; the per-play level goes through the perceptual curve; an end callback is registered for it, so the device-idle release still fires after a tick; `stopAllChannels` clears it; the mock accepts it.

By hand, on a full native rebuild with the deck host closed:

- Each path ticks once per press, higher going up and lower going down: Spotter Louder / Quieter keys, Master and Voice Chat keys, the dial in Spotter, Master, Voice Chat, Race Engineer and Radar modes.
- Mute / Unmute, Mute a Driver, Skip Spotter Call and Push to Talk make no tick.
- Race Engineer and Radar ticks grow louder step by step above the floor, stay at the floor below it, and stop at 0 and 100; with Radar Volume at 0, an iRacing step still ticks at the floor.
- With the Race Engineer switched off, a spotter step and a Race Engineer step both tick.
- A key with no binding, and an iRacing key with iRacing closed, make no tick.
- A fast dial spin plays a clean train of separate ticks, not a smear; five fast detents on Master from a noted level move iRacing's master slider in its sound options by five steps, which checks that iRacing takes every tap of a burst the ticks claim landed.
- A tick plays whole over a Race Engineer callout, with the radar ticking alongside, and over the pit-road speeding tick.
- With iRacing closed, a single Race Engineer step after more than five idle seconds ticks whole (device start).
- The pair reads as distinct from the speeding tick and the radar tones, and from #1161's and #1272's cues once they exist; the floor value is set by ear in a car at race volume.
