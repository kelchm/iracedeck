# Car Control Wipers and Visor: one key for the visor or the wipers

> **Issue:** [#1463](https://github.com/niklam/iracedeck/issues/1463) · **Supersedes:** _none_ · **Superseded by:** _none_
>
> Point-in-time design record. The code and `.claude/rules/` are the truth; this is not documentation.

Today clearing your view takes up to three keys: Tear Off Visor (Car Control) for open-cockpit cars, and Trigger Wipers and Toggle Wipers (Cockpit Misc) for cars with a windshield. A driver who switches between car classes keeps all three on the deck, though only one or two ever do anything in a given car. This adds one Car Control mode that becomes whichever of them the current car has. The issue carries the request; this records how.

## Behaviour

- **A new Car Control mode, Wipers and Visor** (`wipers-and-visor`). It reads which kind of car you are in from telemetry, through #637's helpers `hasVisor` and `hasWipers` in `@iracedeck/iracing-sdk`, which detect the car from whether iRacing exposes `dcTearOffVisor` or a wiper variable (`dcToggleWindshieldWipers` / `dcTriggerWindshieldWipers`) for it.
- **Visor car** (`hasVisor` true): a press tears off the visor **on key-down**, at once, exactly as the Tear Off Visor mode does. There is no long press: holding the key does nothing more, and the release does nothing.
- **Windshield car** (`hasWipers` true): the press is classified **on release** with deck-core's dual-press tracker, against the global `dualPressThresholdMs` the setup keys already use, so one threshold governs every tap/hold key. By default a **tap triggers the wipers** and a **long press toggles them**. The classification happens once, at release, so a press never fires both.
- **Swap option:** a per-key setting **Wipers on tap**, shown only when the Mode is Wipers and Visor, with the options **Trigger** (default) and **Toggle**. Toggle swaps the two: a tap toggles the wipers and a long press triggers them. It does not affect a visor car. The PI says beside it that a long press does the other one. (Added during implementation, at the maintainer's request.) It is a new key in a stored settings schema, so it is a contract change: a key stored without it reads as Trigger.
- **A car with only one wiper variable** (only `dcTriggerWindshieldWipers`, or only `dcToggleWindshieldWipers`): both the tap and the long press send the binding for the one the car has, so the key is never dead on a wiper car. Whether such cars exist is unverified; the rule costs nothing if none do.
- **Unknown car** — anything but a driver live in their own car, decided by the same `isLiveOnTrack` gate the Ignition and Starter keys use: the menus, the garage before getting in, spectating, a replay, the sim not running. A replay publishes a full telemetry frame, so without the gate the key would read the car being watched rather than the user's. The key shows the combined Wipers and Visor icon (below), and a press sends nothing and logs at debug level. Sending a guess could tear off a visor on a car that turns out to have wipers, or the reverse.
- **A car that has neither** (live in own car, both helpers false): the key shows the combined icon in the repo's **unavailable look** (`shared/unavailable-look.ts`: greyed colours and border, faded art — the look Replay Markers and Replay Control use for a press that would do nothing), and a press sends nothing. Whether such cars exist is unverified (#637 calls the two "mutually exclusive", not exhaustive); this state exists so the key never lies if one does.
- **The detection follows the car live.** Switching cars, or getting into the car, re-renders the key on the next telemetry tick. The render is keyed on the resolved kind (`visor` / `wipers` / `unknown` / `neither`) plus which wiper action a tap sends (the swap setting, or the one variable a one-variable car has), so the key is redrawn only when one of those changes.
- **Keys only.** Car Control is a Keypad-only action in all three manifests, so there is no dial or knob surface, and this adds none.

## Architecture

- **The mode's logic lives in its own module, `car-control/wipers-and-visor.ts`**, not in `car-control.ts`, which is already 1,286 lines over 14 modes. The module holds the pure parts, each exported for its own tests in `wipers-and-visor.test.ts`: the car-kind resolver (`visor` / `wipers` / `unknown` / `neither` from a telemetry frame), the choice of binding for a kind, a press kind (tap / long) and the swap setting, the icon and title for a kind and the swap, and the missing-binding rule. `car-control.ts` only wires it in: the mode enum and settings key, a `DualPressTracker` for this mode's key-down / key-up, and a call into the module per render and per press. This keeps the 2,100-line `car-control.test.ts` to the wiring tests.
- **Detection stays where the other capability-aware Car Control keys read it.** Car Control is a `SimIRacingAction` and already reads `hasPitLimiter` and `isLiveOnTrack` straight from the SDK controller; this mode reads `hasVisor` / `hasWipers` the same way. That is one of the known, measured places an action reads iRacing telemetry directly rather than through the bus (#1351's "seams & where the abstraction leaks"), and adding one more call there is consistent with it rather than a new leak. A sim-neutral car-capability signal is not built here (Out of scope).
- **Two seams it crosses, both accepted.** The wiper bindings keep their `cockpitMisc*` setting names while Car Control uses them too; renaming them would be a one-shot settings migration (an xhigh contract change) for a cosmetic gain, and the `carControl` binding list already duplicates other actions' keys. The visor-car and wiper-car art is imported from `icons/car-control/` and `icons/cockpit-misc/` respectively, rather than copied, so the existing keys and this one cannot drift apart.

## Bindings

No new key bindings. Wipers and Visor reuses the three existing global bindings, so a user who has already bound them binds nothing new:

| Car | Press | Binding (global setting) |
| --- | --- | --- |
| Visor | key-down | `carControlTearOffVisor` |
| Wipers, tap (default) | release | `cockpitMiscTriggerWipers` |
| Wipers, long press (default) | release | `cockpitMiscToggleWipers` |

Car Control's PI has one binding-status line under Mode and one Related Key Bindings accordion fed from `key-bindings.json`'s `carControl` list, and the status line reads from the accordion's inputs. So the two `cockpitMisc*Wipers` bindings are added to the `carControl` list (the same duplication the car-cycle bindings already use), bound to the same global setting keys, so binding them in either action binds both.

**The missing-binding warning must not nag.** A user who only drives one kind of car never binds the other side, and must not see a permanent warning for it. On the key, the warning follows the resolved kind: a visor car checks `carControlTearOffVisor`; a wiper car checks the tap binding (and the long-press binding only for its own missing-binding hint, if the warning machinery supports a partial state — otherwise the tap binding alone); unknown and neither show none. In the PI, which cannot know the car, the status line warns only when no side is usable at all: the visor binding **and** both wiper bindings unbound. How that maps onto `keybindKeys` / `setActiveBinding` is the builder's call within those two constraints. A wiper car missing only the long-press binding still fires its tap.

## Icons

- **Known car: the existing artwork, unchanged.** A visor car shows `icons/car-control/tear-off-visor.svg`. A wiper car shows what a **tap** does: `icons/cockpit-misc/trigger-wipers.svg` by default, `toggle-wipers.svg` when Wipers on tap is Toggle. The default title follows the same choice (`VISOR\nTEAR OFF`, `TRIGGER\nWIPERS`, `TOGGLE\nWIPERS`).
- **Unknown or neither: a new combined icon**, `icons/car-control/wipers-and-visor.svg`. The face is split diagonally from top-right to bottom-left: a small wipers glyph in the top-left half, a small helmet glyph (from the Tear Off Visor art) in the bottom-right half. Default title `WIPERS\nVISOR`. It follows `icons.md` (colour slots and locks, the distinctiveness check) and gets its preview regenerated. The neither state is the same art through `unavailableLook`.
- **Not built here:** a wiping animation when the wipers fire. Listed under Out of scope.

## Out of scope

- **A wiping animation on press.** Wanted, but a separate piece of work (frame timing, who owns the image while it runs, which wiper states it applies to). If it is built later, #1438's Starter flash is the pattern: a per-context timer that owns the image while it runs.
- **Showing whether the wipers are on, or a tap that turns running wipers off.** Neither is possible from telemetry: measured on 2026-10-25 in a car with wipers, `dcToggleWindshieldWipers` stayed false for the whole time the wipers ran, so it marks only that the car has the control. A tap-turns-off rule was built on it during implementation and dropped once the probe showed this; tracking the state inside the plugin was rejected, since a wheel button or iRacing's own binding would leave it wrong.
- **Removing or deprecating Tear Off Visor, Trigger Wipers or Toggle Wipers.** They stay, so no existing profile changes and a user can still pin one behaviour to a key.
- **A long press on a visor car.** It tears off on key-down; there is no second action.
- **Per-car overrides** (forcing visor or wipers for a specific car). The telemetry detection is the only source.
- **Renaming the `cockpitMisc*Wipers` setting keys**, or a sim-neutral car-capability signal on the bus. See Architecture.
- **A dial or knob surface.** Car Control is Keypad-only; adding dials is its own change (manifests on two hosts, layouts, the dial gesture path).

## Artifacts

- `packages/iracing-actions/src/actions/car-control/` — `wipers-and-visor.ts` and its test (the mode's rules), and in `car-control.ts` the mode, its settings key, the dual-press tracker wiring, tests, and the `.ejs` PI (Mode option, Wipers on tap dropdown). All three plugins get it through the shared action list; no manifest or UUID change.
- `key-bindings.json`: the two wiper bindings added to the `carControl` list.
- `packages/icons/car-control/wipers-and-visor.svg` and its regenerated preview (`node scripts/generate-icon-previews.mjs`).
- `comms-catalog.ts` and the generated `action-comms.json`: Wipers and Visor's communication method (keyboard, three bindings).
- Website: a Wipers and Visor section on the Car Control page (`docs/actions/driving/car-control.md`) in the `website-action-docs.md` format, with its keyboard and icon-state tables; any mode count that names Car Control's 14 modes; a changelog fragment under Features.
- The `iracedeck-actions` skill: Car Control's mode list and count.

## Testing

- **Unit tests**, the pure rules in `wipers-and-visor.test.ts` and the press / render wiring in `car-control.test.ts`:
  - visor car: key-down taps `carControlTearOffVisor` at once; key-up sends nothing; a long hold sends nothing more;
  - wiper car, default: a release under the threshold taps `cockpitMiscTriggerWipers`, a release at or over it taps `cockpitMiscToggleWipers`, and key-down sends nothing;
  - wiper car, Wipers on tap = Toggle: the two swap;
  - the threshold is read live from global settings (a changed `dualPressThresholdMs` applies to the next press);
  - a car with only one wiper variable: tap and long press both send that one's binding;
  - unknown (null telemetry, and a full frame that is not live in own car, e.g. a replay) and neither: no keystroke on press or release;
  - the icon and title per state (visor, wipers trigger, wipers toggle, unknown, neither through `unavailableLook`), and a re-render when telemetry switches the car kind, and none when it does not;
  - the missing-binding warning: a visor-only user with no wiper bindings sees none on a visor car, and the PI line warns only when every binding is unbound;
  - a settings key stored without the swap reads as Trigger.
- **Manual:** with one key set to Wipers and Visor, drive a windshield car (e.g. a GT3): a tap wipes once, a long press toggles the wipers on and off, the key shows the Trigger Wipers art. Set Wipers on tap to Toggle and check the swap and the Toggle Wipers art. Switch to an open-cockpit car (e.g. a Formula car): the key shows the Tear Off Visor art and tears off on the press, without waiting for release. In the menus with no car, and while watching a replay, it shows the combined icon and does nothing. Check the key on Mirabox and Ulanzi too.
