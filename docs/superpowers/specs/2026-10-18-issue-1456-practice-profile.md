> **Issue:** [#1456](https://github.com/niklam/iracedeck/issues/1456) · **Supersedes:** _none_ · **Superseded by:** _none_
>
> Point-in-time design record. The code and `.claude/rules/` are the truth; this is not documentation.

# An iRaceDeck Practice profile for every deck with bundled profiles

## The problem

The bundled set is Default, Replay, Car Selector and Race Admin Per Car. None is a practice layout, so a driver who wants one builds it from nothing. Automatic profile switching (#1111) needs a profile behind its Practice row too, and a plugin can only switch to profiles it ships (`.claude/rules/profiles-and-devices.md`).

## Decisions

### 1. One template, one file per target device

`iRaceDeck Practice`, shipped as `iRaceDeck Practice SD`, `iRaceDeck Practice XL` and `iRaceDeck Practice Plus XL` under the #753 scheme. The Stream Deck + file comes with #1457, which covers every template shipping in the same release. No code change is needed to resolve the names: `deviceProfileName` and `switchToBundledProfile` already handle any registered template, and the settings window's Profiles card gets its row from the generated `profiles.json`.

### 2. Installing it never moves the user

`AutoInstall: true`, `DontAutoSwitchWhenInstalled: true`, the Replay pattern. Only Default switches on install; a second profile that did so would take a returning user off the profile they were on at every update that installs it.

### 3. Default is not re-exported to link to it

A Switch Profile key on Default pointing at Practice would mean re-shipping Default on three devices. Whether a re-shipped file keeps a user's edits to their installed copy is unmeasured (#1111's open question), so touching Default risks every user's own changes for one convenience key. Practice is reached from the settings window's Profiles card, a Switch Profile key the user places, or #1111.

### 4. What is on it

Authored by the maintainer in the app. The starting point is Default with its race-only keys replaced by practice ones: setup and in-car adjustments, splits and reference, tyre and fuel information, lap timing black boxes. The authoring workflow's rules apply unchanged, in particular the host-profile marker on every Switch Profile key, without which Back to previous cannot learn that Practice is the active profile (#762).

### 5. Switch Profile shows it

A `packages/icons/switch-profile/practice.svg` icon keyed `iRaceDeck Practice` in `PROFILE_ICONS`, so a Switch Profile key targeting it is not drawn with the fallback artwork.

## Out of scope

- The Stream Deck +, Mini, Neo and Galleon files: #1457 adds the Stream Deck +; the rest stay `candidate`.
- Test, Qualifying and Race profiles, which #1111 adds.
- Any change to Default, Replay, Car Selector or Race Admin Per Car.

## Testing

Nothing here is unit-testable beyond what exists: `profiles.json` regenerates from the manifest under its freshness test, and the icon previews under theirs. Manually, on each of the three devices: Switch in the settings window prompts the install and lands on page 0; a Switch Profile key targeting Practice switches and shows the new icon; Back to previous from Practice returns to the profile before it.
