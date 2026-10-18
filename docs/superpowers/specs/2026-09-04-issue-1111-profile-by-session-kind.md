> **Issue:** [#1111](https://github.com/niklam/iracedeck/issues/1111) · **Supersedes:** _none_ · **Superseded by:** _none_
>
> Point-in-time design record. The code and `.claude/rules/` are the truth; this is not documentation.

# Stream Deck: switch to a bundled profile automatically by session kind

Revised 2026-10-18 by the maintainer's word, before any of it shipped: five session kinds instead of seven (no garage row, one qualifying row), every row defaulting to "Do not change the profile" instead of a pre-filled mapping, a starter profile for each kind, and the settings card's name. The 2026-09-04 version is in git history.

## The problem

lexoduss asked on Discord for iRaceDeck profiles to be activated automatically by session type, listing eight kinds: Open Practice, Test Drive, Race, Spotting, Open Qualifying, Solo Qualifying, Garage/Replay, Race Control. Peter's reply — "limited to what telemetry options are available to set the profile" — is the right frame, and there is a second limit he did not name that shapes the feature more.

## The constraint that decides the shape

**A plugin can only switch to profiles it ships.** The Elgato SDK's own typing says so in its doc comment ("plugins may only switch to profiles distributed with the plugin, as defined within the manifest, and cannot access user-defined profiles") and `.claude/rules/profiles-and-devices.md` records it as policy. There is no API that activates a user's own profile, and no API that reports which profile is active.

So the request as written — point the feature at _my_ practice profile — is impossible. What is possible is one step removed: iRaceDeck ships a profile per session kind, the user edits them in the app (bundled profiles install with `Readonly: false`), and the plugin switches between those.

Profiles are Elgato-only. Mirabox and Ulanzi have no profile system; `switchToProfile` is a no-op there and the whole feature is gated on the `profiles` platform flag.

## What ships

- A card **Automatically switch to profile** on the settings window's Profiles tab: a master toggle, then one dropdown per session kind — **Test**, **Practice**, **Qualifying**, **Race**, **Replay** — each listing every bundled profile plus **Do not change the profile**.
- A bundled profile for each kind: **iRaceDeck Test**, **iRaceDeck Qualifying** and **iRaceDeck Race** are new here; **iRaceDeck Practice** comes from #1456 and **iRaceDeck Replay** exists.
- A reactor that switches every connected profile-capable deck when the session kind changes, and returns it to the profile it was on before the first automatic switch when iRacing exits.

## Decisions

### 1. A starter profile per kind, which may start nearly empty

Mapping only onto the existing templates would let a driver route replay to Replay and little else; the point of the request is a different deck per session, which needs a profile per kind the user can shape. Test, Qualifying and Race are new templates. Each needs only one thing to ship: a Switch Profile key back to Default carrying the host-profile marker, so Back to previous and the history stack know where the deck is (#762). Anything beyond that is the maintainer's call while authoring; empty is acceptable, because the user is expected to fill them.

Files for the Stream Deck, the XL and the + XL (`SD`, `XL`, `Plus XL`), nine in all, `AutoInstall: true` and `DontAutoSwitchWhenInstalled: true` like Replay. The Stream Deck + variants come with #1457, which covers every template of the release. Each gets a Switch Profile icon keyed by display name, as #1456 does for Practice. The device-suffix scheme (#753) and `switchToBundledProfile` need no change.

### 2. Five kinds, from what the sim tells apart

| Row        | Signal                                                         |
| ---------- | -------------------------------------------------------------- |
| Test       | `SessionType` `Offline Testing`                                |
| Practice   | `Practice`, `Lone Practice`, `Warmup`                          |
| Qualifying | `Open Qualify`, `Lone Qualify`                                 |
| Race       | `Race`                                                         |
| Replay     | `WeekendInfo.SimMode === "replay"`, which wins over the type   |

A `SessionType` string not in the table switches nothing and is logged at debug, so an unknown kind never moves the deck.

**Replay means a replay session**, a saved replay opened from the sim, not watching the replay in the middle of a live session. Pausing a race to review an incident would otherwise swap the deck under the driver and swap it back when they return to the car, twice in a few seconds; the Replay Control keys on the race profile are there for that.

**Not offered:** an in-garage row (dropped 2026-10-18), separate open and solo qualifying rows (folded into one), and Spotting and Race Control, which have no signal: there is no spectator or spotter flag in telemetry, and race control is a role a human holds, not a state the sim reports.

A new `classifySessionKind()` lives beside the existing three-bucket `classifySessionType()`, which is untouched because every Race Engineer callout depends on its exact buckets.

### 3. Off by default, and every row starts as "Do not change the profile"

`profileAutoSwitchEnabled`, default **off**: switching a user's deck unasked moves the buttons under their hand, so this is not a feature that defaults on. The rows are `profileForSessionKindTest`, `…Practice`, `…Qualifying`, `…Race`, `…Replay`, each a bundled profile's display name, or empty for **Do not change the profile**, which is every row's default. Turning the toggle on therefore does nothing until the user picks a profile for at least one kind; the card's supporting text says so. These are `GlobalSettingsSchema` fields with `.catch` defaults, a published contract once shipped.

The rows stay visible and editable while the toggle is off, so the mapping can be set up before it is switched on.

### 4. When it switches, and how it gets back

Triggers: the first snapshot after connect (the sim may already be in a session), and `session.changed` (its payload carries only session numbers; the kind is read from the translator at that moment, the way the callout engine does). A switch is made only when the kind differs from the last one acted on, so a manual switch mid-session is never fought; the next session change applies the mapping again.

On iRacing exit — the app monitor's terminate hook — the reactor returns each deck to the profile it was on before the **first** automatic switch, through the per-device history stack Back to previous already keeps (#762); with nothing to pop it falls back the way Back does. Exit is a no-op for a deck the reactor never switched.

Every connected profile-capable deck is switched. A deck with no variant of the mapped profile (the Mini, the Neo, the Galleon, and the Stream Deck + until #1457) is skipped and logged at debug. The mapping is plugin-wide, not per deck; per-deck tables are not worth their settings surface until someone asks.

### 5. Where it lives

deck-core is sim-neutral and may not import an iRacing package (#1351), so the pieces split along that seam: `classifySessionKind()` in `sim-events-iracing` with a translator accessor for the current kind; a pure `resolveProfileForKind(kind, mapping)` with no sim import; and a thin reactor that subscribes to the kind and calls `switchToBundledProfile` / `requestProfileSwitchBack`. The reactor is started from the Stream Deck `PluginExtension` (`elgato-extension.ts`), the one place Elgato-only behaviour is wired since #1349, and is gated on `profiles`.

## Alternatives rejected

**Decline.** Considered, because the literal request cannot be built. Rejected because the one-step-removed shape delivers the outcome the user described through editable profiles.

**Map onto the existing templates only.** Decision 1.

**A pre-filled mapping.** The 2026-09-04 version pre-filled every row so the toggle worked at once. The maintainer chose "Do not change the profile" for every row: the toggle is then safe to try, and nothing switches to a profile the user has not chosen.

**Treat in-session replay viewing as Replay.** Decision 2.

**Default on.** Decision 3.

## Out of scope

- Switching to a user's own profiles: the SDK does not allow it.
- Garage, Spotting and Race Control rows; separate open and solo qualifying rows.
- A per-deck mapping.
- Mirabox and Ulanzi, which have no profile system.
- What the Test, Qualifying and Race profiles contain beyond the marker key.
- The Stream Deck + files (#1457) and the Practice profile itself (#1456).

## Open questions

- **Do user edits survive a plugin update that re-ships the profile file?** This matters more here than anywhere, since these profiles exist to be edited. Bundled profiles do not reliably auto-install on updates, which suggests edits persist, but nothing has measured a _changed_ file against an edited install. Task: edit an installed bundled profile, ship a build with a modified file, observe. If edits are lost, the fallback is `AutoInstall: false` for these profiles plus a documented export-and-reimport step.

## Verification

Classifier tests over every known `SessionType` string, an unknown one, and replay mode. Resolver tests over the mapping, empty rows included. Reactor tests for the trigger matrix: connect mid-session, a kind change, a session change within the same kind (no switch), toggle off, a deck without a variant, and return on exit only for decks it switched. Manual, on hardware: a test drive; a practice → qualifying → race weekend with three different profiles mapped; a row left at "Do not change the profile"; a manual switch mid-practice surviving until qualifying; a saved replay; closing the sim.
