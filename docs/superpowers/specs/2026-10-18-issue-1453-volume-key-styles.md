# Paired key styles on the volume keys

> **Issue:** [#1453](https://github.com/niklam/iracedeck/issues/1453) · **Supersedes:** _none_ · **Superseded by:** _none_
>
> Point-in-time design record. The code and `.claude/rules/` are the truth; this is not documentation.

Extends #810 (`2026-07-07-paired-adjust-key-styles-design.md`). That spec's model is kept: no group setting, pairs come from where the user places the keys, the style is a per-key setting, Legacy for existing keys and Split for new ones, hold-to-repeat on every style except Legacy. This record decides only what differs for volume keys.

## Which keys

| Action | Keys | Level source |
| --- | --- | --- |
| Audio Controls | Master Volume Up / Down, Voice Chat Volume Up / Down | none: iRacing exposes no volume |
| AI Spotter Controls | Spotter Louder / Spotter Quieter | none: iRacing exposes no volume |
| Audio Controls | Race Engineer Volume Up / Down, Radar Volume Up / Down | the 0–100 global setting behind `INTERNAL_AUDIO_BUSES` |

Not pairs, and untouched: Push to Talk, Voice Chat Mute, Mute a Driver, Skip Spotter Call, the four spotter report keys. Each is a single press with no opposite, so a +/− style has nothing to say about it. A key whose stored mode is one of these keeps its `keyStyle` value in its settings and ignores it, so switching back to a volume mode restores the look.

The Audio Controls dial is unchanged. Audio Controls' keypad has no Spotter mode, and this does not add one: spotter keys stay in AI Spotter Controls.

## Styles per key

The Key Style list is the same on every volume key: Legacy, Split, Edge chevrons, Joined pill, Big +/−, Big chevrons, Pill end, with Position in Pair for the position-aware ones, exactly as on a setup action's adjustment mode. The View-key styles (Pill middle) are not offered, because no volume has a View key.

**On a key with no level, a value style is drawn in its no-value form:**

| Stored style | Drawn on an iRacing volume key as |
| --- | --- |
| Split | Big +/− |
| Edge chevrons | Big chevrons (same Position in Pair) |
| Joined pill | Pill end (same Position in Pair) |

The mapping is a render rule only. The stored value is never rewritten, and the Property Inspector shows the stored style with a note on iRacing volume modes: "iRacing doesn't report its volume, so value styles show without a value: Split as Big +/−, Edge chevrons as Big chevrons, Joined pill as Pill end."

Why render-only and one list. A fresh Audio Controls key is seeded with Split when it is placed, before the user has picked a Mode, so the seed cannot know whether the key will be Master or Race Engineer. Rejected alternatives:

- **A separate no-value list on iRacing modes**, the way View modes get their own Display Style select. The seeded Split is not in that list, so every newly placed Master key would open with a blank Key Style. The #810 partial accepts a blank select as cosmetic after a mode-category switch; here it would be the default path.
- **The PI writing the no-value form when the mode changes.** Switching a key from Race Engineer to Master and back would turn the user's Split into Big +/− for good. A written fallback cannot be told from a choice afterwards; the standing rule is to render a fallback and persist nothing.
- **Falling back to Legacy**, as #810 does for a setup mode with no value. New Master, Voice Chat and Spotter keys would then look exactly as today, which is the opposite of the request. #810 could afford Legacy because its valueless modes are rare; here half the keys are valueless.
- **New "label instead of value" variants of Split, Edge chevrons and Joined pill.** Three new layouts to draw, test and document, for a look the existing no-value styles already give once they carry a label (below).

**The label shows by default on every volume key's no-value style.** #810 hides the title on Big +/−, Big chevrons and Pill end because in a 3-key group the View key in the middle carries the label. A volume pair has no middle key, and a deck with Master, Voice Chat and Spotter pairs would otherwise be six identical + and − keys. The labels are one word, since the glyph already gives the direction: MASTER, VOICE, SPOTTER, ENGINEER, RADAR. They go through the title system like every other key's title, so the global title defaults and the per-key Title Overrides (text, size, position, hide) apply. The renderer gains one optional input for this; the setup actions do not pass it and render exactly as before.

**Colours.** Background and text come from the action's existing volume icon (`colorSourceSvg`), the +/− and pill accent from the Graphic 1 slot with #810's yellow default, and pill styles lock the normal border off, all as in #810.

## Defaults

Unchanged from #810: the schema default is Legacy with `.catch("legacy")` (and `pairPosition` `"auto"` with `.catch("auto")`), and a key that appears with no stored settings at all is seeded with Split.

- **Existing keys stay Legacy.** Every key that has appeared since `addedWithVersion` stamping exists carries at least that field, so in practice no existing key is seeded. #810's caveat about never-configured old keys picking up Split no longer has a population.
- **A new key gets Split**, which shows the level on Race Engineer and Radar and is drawn as Big +/− with its label on Master, Voice Chat and Spotter.
- **The seed is written before `super.onWillAppear`.** The base class stamps `addedWithVersion` by writing `{...payload, addedWithVersion}`. Audio Controls already runs its #1015 migration before `super` and hands the base the migrated payload, so the stamp lands last and keeps the change; the seed joins that step (a seed and the migration never both apply, since an empty payload has nothing to migrate). AI Spotter Controls gets the same pre-`super` step. The setup actions seed after `super`, so their seed write appears to replace the stamp, which the next appearance re-adds; that is not changed here.

## The level on Race Engineer and Radar keys

- **Format.** The integer level without a unit, `55`, the same text the Audio Controls dial shows. Units are dropped as in #810, and percent is the only unit these levels have.
- **Feature off.** While the Race Engineer or the Radar is switched off, the value reads **OFF**, as the dial does. Pressing still steps the level, as it does today on the key and the dial, and the number returns when the feature is switched on. Rejected: showing the number while off, which reads as audible; showing both, which does not fit at Split's value size.
- **Refresh.** The level is a global setting, not telemetry, so these keys re-render on a global-settings change rather than on SDK ticks. deck-core's base action already does exactly that: it subscribes to `onGlobalSettingsChange` once and re-runs every context's regenerate callback, which reads the level live. That covers every writer: the key, its partner, the dial, the Settings window sliders, the Pit Crew toggles. No second listener is added (amended during implementation): a memoized one could not stop the base's re-render and would only send the same image twice. Keys with no level read nothing global beyond what every key does.

## Hold to repeat

On for every volume key in any style but Legacy, through `RepeatController` with #810's constants: the plugin-wide long-press threshold before the first repeat, about 150 ms between steps, and the 15 s safety cap. Legacy keys keep their single step per press.

- **iRacing volumes.** Each step is another `tapBinding` of the same key binding, about seven per second. That is the same thing a dial does on a fast turn, and the setup actions already repeat key bindings this way, so it adds no new kind of input. The key binding, window focus mode and #612 missing-binding overlay behave exactly as for a tap.
- **Race Engineer and Radar.** Each step is one 5 % `stepBy(±1)`: 0 to 100 in about three seconds. The step persists through the settings store's debounced write, and a step at 0 or 100 is a no-op that writes nothing.

## One implementation for two actions

`adjust-styles.ts` stays the owner of the style catalog, the settings fields, the seed, the renderer and the repeat constants. Its value lookup (`renderPairedIconOrNull`, `telemetryMemoValue`) is tied to the setup actions' `VIEW_DEFS` telemetry registry, so the volume keys do not go through it. A sibling `shared/volume-key-styles.ts` holds what is volume-specific:

- a volume key description: the label, the direction, and either a level reader or none;
- the stored-style resolution (the table above, applied only when there is no level reader);
- the render entry, which returns the styled icon or `null` for Legacy, so each action falls back to its existing renderer exactly as the setup actions do.

Each action maps its own settings to a description: Audio Controls from `{category, action}`, reading the level through `INTERNAL_AUDIO_BUSES`; AI Spotter Controls from `louder` / `quieter`. The shared module imports from neither action folder: the level reader is handed in. Each action keeps its own `RepeatController`, as the setup actions do. Two additions to `adjust-styles.ts` are generic catalog knowledge: the value-style → no-value-style mapping and the label-on-no-value-styles render option.

Rejected: putting the volume keys through `VIEW_DEFS` with synthetic entries (they read no telemetry field, and the registry's memo and subscription assume one), and generalising `renderPairedIconOrNull` into a value-source abstraction now (below).

## Property Inspector

`adjust-style.ejs` is reused with two generalisations, both optional so the setup actions' includes stay as they are:

- **A mode made of several selects.** Audio Controls' pairing depends on Mode and Action together. The partial accepts a list of select ids and matches on their values joined with `-` (`master-volume-up`), the same keys `AUDIO_CONTROLS_GLOBAL_KEYS` uses. AI Spotter Controls passes its single Mode select.
- **`noValueModes`.** The modes that show the no-value note above. Audio Controls lists the Master and Voice Chat volume pairs, AI Spotter Controls `louder` and `quieter`.

The Key Style, Position in Pair and help rows show only on volume modes. The binding status line, key bindings section and colour, title and border overrides are unchanged.

## Settings schemas

Both actions' keypad schemas spread `adjustStyleSettingsFields`; both fields carry `.catch`, so a value from a newer build degrades that field instead of failing the whole parse. AI Spotter Controls' `control` has no `.catch` today, so an unknown stored control fails the parse and resets every field to its default, the new ones included. It gains `.catch("damage-report")`: an unknown control still lands on Damage Report, as now, but the key's style survives. Audio Controls' fields already carry `.catch`. No global setting, action UUID or manifest entry changes.

## Out of scope

- **Fuel Service fuel-to-add and Replay Speed** (#810's later phase). The volume module is not free generalisation for them: Fuel Service's value lives in action state with its own unit handling, and Replay Speed has its own mode semantics and dial. They get their own decision when picked up; the volume module's description-with-a-level-reader shape is what that work would generalise from.
- **A level View key for Race Engineer or Radar**, and so 3-key groups with a value in the middle for volumes. Both keys of a pair already show the level.
- **A Spotter mode on Audio Controls' keypad.** Spotter volume keys stay in AI Spotter Controls.
- **The Audio Controls dial**, and dial surfaces generally.
- **The setup actions' seed ordering** relative to the `addedWithVersion` stamp (noted under Defaults).
- **A tick sound on every volume step**, a separate idea filed in parallel. No dependency either way; if both land, a held key ticks once per repeat step, the same as a tap.

## Testing

Suite:

- **`volume-key-styles`.** Style resolution: each value style maps to its no-value form only on a key without a level reader, Legacy returns `null`, the no-value styles pass through. The label is shown by default on no-value styles and a title override still hides or changes it. The level renders as `55`, `0`, `100` and `OFF`.
- **`adjust-styles`.** The new render option is off by default: every existing setup-action expectation passes unchanged. The mapping covers all three value styles.
- **Audio Controls.** A fresh key is seeded with Split and the `addedWithVersion` stamp written after it still carries it; a configured key is not seeded; the #1015 migration still runs. Each of the eight volume pairs renders through the shared module and every other pair through the existing renderer; a #1196 unavailable pair keeps its warning icon. A global-settings change re-renders a Race Engineer key with its new level, or OFF. Repeat with mocked timers: a quick tap steps once, a hold repeats, release stops it, for a Master key (repeated `tapBinding`) and a Radar key (repeated `stepBy`). A missing Master binding shows the #612 overlay.
- **AI Spotter Controls.** Seeding as above, Louder / Quieter render and repeat, the report and Skip Spotter Call keys unchanged, an unknown stored `control` keeps `keyStyle`.
- `pnpm build` (the PI partial compiles into all three plugins), `pnpm typecheck`, `lint`, `format`, `test`.

Manual, on hardware:

- Stream Deck: Master Up / Down as a horizontal Joined pill pair and as a vertical Pill end pair; Voice Chat and Spotter pairs beside it, each identifiable by its label; a newly placed key opens with Split in the PI and draws Big +/− on Master.
- In iRacing: hold Master Volume Up and confirm iRacing steps repeatedly and stops on release; same for Spotter Louder.
- Race Engineer pair: the level on both keys follows a press, the partner key, the dial and the Settings window slider; switching the Race Engineer off shows OFF on both keys, and on again shows the level.
- An existing Audio Controls and AI Spotter Controls key from before the upgrade still shows its old icon.
- Mirabox and Ulanzi: the pair styles render as on Stream Deck (QT-safe SVG, as in #810).
