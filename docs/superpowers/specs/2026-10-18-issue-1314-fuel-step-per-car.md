# Fuel Service: a press always moves the fuel request, whatever the car's fuel step

> **Issue:** [#1314](https://github.com/niklam/iracedeck/issues/1314) · **Supersedes:** _none_ · **Superseded by:** _none_
>
> Point-in-time design record. The code and `.claude/rules/` are the truth; this is not documentation.

A bug, but the fix involves a design choice, so it gets a spec. The issue carries the report and the logs; this records how it is fixed.

## The model

iRacing banks a requested fuel amount on a car-specific grid. The MX-5 banks whole US gallons (3.785 L); the Porsche 911 GT3 banks whole liters. `pit.fuel` takes whole liters, and iRacing rounds what it is sent to the nearest point on the car's grid. Both are read off the issue's logs: every MX-5 value lands on a gallon multiple, and `pit.fuel(10)` from 11.35 L (3 gal) banks 3 gal again.

Fuel Service's Add Fuel and Reduce Fuel keys send `round(PitSvFuel ± amount)`. When the amount is less than half the car's step, the request rounds back to where it was, and the press does nothing.

The grid is not published anywhere iRaceDeck can read: no session-info or telemetry field holds it. The issue also cites an unconfirmed claim that the step can depend on the series (a 1.7 gal minimum on ovals). So the fix must not depend on knowing the step.

#726 met the same rounding on the Fuel Dial in 2.0 and listed this exact fix as its option B ("a 1 L request on a gallons car does nothing, so try 2 L, 3 L… until the value actually moves"). Only its option A (the readout follows `PitSvFuel`) was built, so this is new work, not a regression.

## Decision: check that the press landed, and step further when it did not

- **Send, then read back.** After an Add Fuel or Reduce Fuel press sends its request, Fuel Service watches `PitSvFuel` for up to **300 ms**. Telemetry runs at 60 Hz and iRacing reflects a pit command within a tick or two (#726 measured Fuel Service following the black box within one tick), so 300 ms is generous.
- **If the amount moved in the press's direction, the press is done.**
- **If it did not move, send the next whole liter further in the same direction and watch again.** Repeat until the amount moves, or until the request is **8 L past the first one**, a little over two gallons. That covers the MX-5 (one or two extra liters) and leaves room for a coarser step if the oval claim turns out true.
- **The press always moves the amount by at least one step.** That is the issue's expected behaviour ("at least to the next step the car accepts"). A press whose amount is larger than the step behaves as today, because its first request already lands.
- **A head start per car.** The extra liters that last made a press land are remembered per car (the player's `CarPath`) and direction, for the plugin's lifetime, so the next press starts there and lands on the first try. Changing car resets nothing: the memory is keyed by car. A head start that overshoots by more than a step is not possible with the bounds above, because the head start is always below half of any step it learned.
- **Presses during a check queue one deep.** A press, or a held key's repeat, that arrives while a check is running is held. The newest one runs, from the new baseline, once the running check ends. A held key on the MX-5 therefore steps one gallon per check instead of restarting a check that never finishes.
- **When nothing moves after 8 L**, the check stops and logs at info: the tank is full, the request is at zero, or the car is not in a pit-service state. It does not retry past that and shows nothing new on the key.
- **Set Fuel Amount is unchanged.** It sets an absolute amount, and banking the grid point nearest to it is correct.

## The dial

The Fuel Service dial's **Add Amount** rotation has the same problem when turned one detent at a time on a car whose step is larger than the dial's step size: each detent's request rounds back, and the readout settles where it was. Its rotation sends through the same check: when the throttled send lands unchanged, the next whole liter in the rotation direction is sent, with the same bound and the same per-car head start.

**Fill To** mode is not changed by this spec. It already rounds its add up to the next whole display unit, and on a gallon car that rounding can still be undone by iRacing's nearest-gallon rounding. Whether that leaves a stop below target is checked during the manual test. If it does, that is a separate issue.

## Where it lives

A new module beside `fuel-pipeline.ts`, used by both surfaces: `sendFuelStep(direction, firstLiters)`. It sends through `FuelPipeline.fuel`, reads `PitSvFuel` through the SDK controller, and owns the queue and the per-car head starts. `executeSdkFuelAdjust` and the dial's Add Amount send call it instead of `pipeline.fuel` directly. The preserve-off clear that follows a send today is sent once, after the check ends.

## Out of scope

- A per-car table of fuel steps. The check makes one unnecessary, and a table would need every car tested and kept up to date.
- Showing the car's step on the key or in the Property Inspector.
- Persisting the head starts across plugin restarts. One extra liter on the first press of a session costs nothing.
- Fill To mode (see above), Set Fuel Amount, and the autofuel paths.

## Testing

- **Unit tests** for the new module with a fake pipeline and fake telemetry, on fake timers:
  - a request that lands on the first send stops at once (Porsche-like grid);
  - on a gallon grid, 1 L Reduce and 1 L Add each move exactly one gallon, sending the extra liters the issue's log predicts;
  - the head start is remembered per car and direction, so the second press lands on its first send;
  - the check stops at 8 L past the first request and logs;
  - a press during a check is queued one deep, and the newest wins;
  - the preserve-off clear is sent once, after the check.
- **Existing Fuel Service tests** pass unchanged for cars that bank whole liters.
- **Manual, in the sim:** in the Global MX-5 Cup with metric units, Reduce Fuel and Add Fuel with Amount 1 L move the request one gallon per press, and a held key keeps stepping. The same keys in the Porsche 911 GT3 still move 1 L per press. Turn the Fuel Service dial one detent at a time in Add Amount mode on the MX-5 and check it moves. In Fill To mode on the MX-5, check whether a stop finishes below the target.
