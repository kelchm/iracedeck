> **Issue:** [#1457](https://github.com/niklam/iracedeck/issues/1457) · **Supersedes:** _none_ · **Superseded by:** _none_
>
> Point-in-time design record. The code and `.claude/rules/` are the truth; this is not documentation.

# Bundled iRaceDeck profiles for the Stream Deck +

## The problem

#736 shipped profile templates for three target devices: the Stream Deck (0), the XL (2) and the + XL (13). The Stream Deck + (7: 8 keys in 4×2, four dials, the touch strip) was left a `candidate`. Nothing tells its owners. In the settings window's Stream Deck Profiles card the single deck is auto-selected, Switch asks the app for `iRaceDeck Default Plus`, the manifest has no such profile, and the press does nothing visible.

## Decisions

### 1. Every template of the release, not a subset

The Stream Deck + gets a `Plus` file for every template shipping in 3.7.0: Default, Replay, Car Selector, Race Admin Per Car, Practice (#1456), and Test, Qualifying and Race (#1111). Eight files. A partial set would recreate the same dead button one row down, and the Switch Profile key, the Race Admin selector and #1111's mapping all assume a template exists on every target device. The work is authored last of the three issues for that reason.

### 2. Graduate it to `target`

`DEVICE_SUPPORT[StreamDeckPlus].profileTemplates` becomes `target` and the device joins `PROFILE_TARGET_DEVICES`, together, as the rule requires and the test enforces. The suffix `Plus` already exists, so names resolve with no other change.

### 3. The dials are part of the layout

The four dials get real dial actions where a template has a natural one: Default with the dial surfaces drivers reach for in a race (Fuel Service, Audio Controls, a setup dial), Replay with Replay Control and Replay Markers. The Car Selector and Race Admin Per Car are key layouts and may leave the dials empty. The selector needs no change for 8 keys: slots are ordinals among the visible select-car keys with learned page counts (#754), never a grid assumption.

### 4. A deck without a variant says so

The Profiles card knows the chosen deck's type (`_deckDevices` carries it) and each template's device types (`profiles.json`). When the chosen deck has no variant of any template — after this, the Mini, the Neo and the Galleon — the card shows one line saying iRaceDeck ships no profiles for that deck yet, in place of the Switch buttons. A row whose template lacks a variant for the chosen deck is hidden. This is the failure that hid the Stream Deck + gap, and it would hide the next one the same way.

## Out of scope

- The Mini, the Neo and the Galleon: still `candidate`.
- The Stream Deck Studio and Mobile: deliberately `excluded` (#736).
- New templates: the eight are #736's four plus #1456's and #1111's.

## Testing

Unit: the `device-profiles.ts` invariant test (target list against `DEVICE_SUPPORT`); the Profiles card's variant check against a deck type with and without variants. Manual, on a Stream Deck +, which is required because the Virtual Stream Deck is a separate device type: each of the eight profiles installs from the settings window's Switch and from a Switch Profile key; Back to previous returns; the dials on Default and Replay act; the Car Selector fills and pages its 8 keys in a session with more than 8 cars. On a deck with no variant, the card shows the note and no Switch buttons.
