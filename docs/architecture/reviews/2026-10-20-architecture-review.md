# Architecture review — 2026-10-20

> **Previous:** [2026-10-09](2026-10-09-architecture-review.md) · **Follow-ups:** [#1465](https://github.com/niklam/iracedeck/issues/1465) · [#1466](https://github.com/niklam/iracedeck/issues/1466) · [#1467](https://github.com/niklam/iracedeck/issues/1467) · [#1468](https://github.com/niklam/iracedeck/issues/1468) · **Still open from the previous review:** [#1373](https://github.com/niklam/iracedeck/issues/1373) · [#1374](https://github.com/niklam/iracedeck/issues/1374) · [#1375](https://github.com/niklam/iracedeck/issues/1375) · [#1376](https://github.com/niklam/iracedeck/issues/1376) · [#1377](https://github.com/niklam/iracedeck/issues/1377) · [#1378](https://github.com/niklam/iracedeck/issues/1378) · **Related:** [#714](https://github.com/niklam/iracedeck/issues/714), [#643](https://github.com/niklam/iracedeck/issues/643)
>
> Point-in-time review of `master` at `41f815395` (v3.7.0-dev.0; the code is identical to `1980eae7f`, the one commit between them is a spec). The code and the developer [Architecture page](../../../packages/website/src/content/docs/docs/development/architecture.md) are the truth; this is not documentation. It is frozen once committed: a later review is a new dated file, never an edit to this one.

## Scope and method

Eleven days and 195 commits after the 2026-10-09 review (`3d8cc28b9..1980eae7f`, one of them a merge). None of that review's six follow-ups has merged; all six are open. The window was feature work instead:

- **Replay.** Ten squash-merged PRs: the Replay Control dial (#1316, #801), replay jump playback (#1276), Jump to Race Start (#1354), one gate for every replay command (#1430), lap and session keys from the replay record (#1426), and the busy mark (#1437).
- **Race Engineer.** The rolling-start formation and caution catch-up (#1167/#1168), and quiet gap calls under caution (#1171).
- **Audio device.** The device held open while iRacing runs (#1347), and two default-device fixes (#1411, #1414).
- **Diagnostics.** Plugin state in the Telemetry Snapshot (#1387).
- **Process.** Changelog fragments (#1386) and the second read (#1396).

The questions were the same as last time: what is good, what is bad, how extendable the structure is, whether package boundaries are respected, how ready it is for a second sim, and what moved since the previous review.

Five read-only reviewers took one seam each:

1. sim and telemetry
2. deck platform
3. audio and the Race Engineer
4. cross-cutting: build, settings and process
5. **new:** the replay stack, end to end, because that is where the growth went.

Each re-ran the previous review's counts on both `3d8cc28b9` and HEAD with an explicit, repeatable method, recorded in the tables below. The coordinator then re-measured the figures the conclusions rest on and settled the two places where reviewers disagreed. Figures marked **(verified)** were re-measured by the coordinator; the rest come from one reviewer's measurement. Judgements are marked as such.

Line counts are physical lines of non-test `.ts` under `src/` unless stated.

## Verdict

**The deck seam held completely; everything above it grew where the last review said it would not.**

- **What held.** Across 195 commits, `deck-core` gained 54 lines (verified). The adapters, the plugin shells and `plugin-build` changed only version lines. `SimConnection`, `deck-iracing/src` and the sim-boundary lint rule have zero diff. The package graph is still acyclic, and only two edges were added.
- **What grew.** The previous review declined to schedule the god modules because none was "getting worse fast enough". Since then:
  - `replay-control.ts` grew 47% (2,611 → 3,835, verified).
  - `translator.ts` grew 21% (2,705 → 3,274, verified).
  - `diff/caution.ts` more than doubled (611 → 1,356, verified).
  - `scenario-shortcuts.ts` grew 22% (3,358 → 4,100, verified).
- **The replay stack.** It almost tripled, to about 21k non-test lines. Two-thirds of that is a stateful replay engine that lives inside `iracing-actions` (item 1).
- **The process layer is now the fastest-growing part of the repo:**
  - 159 of the 195 commits are spec edits (verified).
  - 82% of master's CI runs were triggered by them (verified).
  - The always-loaded instructions are 1.1k characters short of the repo's own budget test going red (verified, item 2).

Nothing here is a regression in behaviour. The ranking below follows two kinds of cost:

- **Concentration:** about 6.7k lines of process-wide replay state that nothing structural owns.
- **A budget about to run out:** a red `master` the next time someone adds a few lines to an always-loaded rule.

## What moved since 2026-10-09

| #   | Previous finding                            | Status                         | Now                                                                                                                                                                                                                               |
| --- | ------------------------------------------- | ------------------------------ | --------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| 1   | Sim seam above the deck layer, iRacing-typed wiring | **Mixed**              | `rolling-start.ts` stopped importing `getSessionType` (catalog files 15 → 14, casts 12 → 11). But `SimRuntime` is 17 → 19 `typeof` getters (verified), `plugin-runtime` files knowing the sim are 7 → 8 (verified), and actions now read translator getters directly (item 4) |
| 2   | Silent native mock fallback                 | **Unchanged**                  | Both `catch {}` blocks are still there (`iracing-native/src/index.ts:35`, `audio-native/src/index.ts:56`). #1373 is open                                                                                                         |
| 3   | God modules                                 | **Worse**                      | Every file in the previous table grew except `replay-control`'s neighbours (item 3). `initSettings` 458 → 472 lines, and the replay store's wiring is now inside it                                                                |
| 4   | Boundary enforcement lags the split         | **Unchanged, slightly worse**  | `eslint.config.js` is byte-identical. Test files reaching a sibling's `src` went 16 → 20. The aliases and vestigial dependencies are unchanged                                                                                    |
| 5   | Wiring and settings drift                   | **Unchanged, one part worse**  | The harness still passes 0 of 6 translator options (verified). The risk the review called hypothetical has happened: three persisted callout keys were added outside the frozen baseline guard (item 6)                          |
| 6   | Hand-kept family lists                      | **Unchanged**                  | The `SCENARIO_ID_TO_*` and `*_CLIP_SOURCES` counts are identical. A new live dependency (`getCautionGap`) took 10 sites in 3 packages                                                                                            |
| 7   | Singletons and init naming                  | **Worse**                      | `_reset*` hooks 33 → 40 and module-level nullable `let`s 59 → 78, almost all of the growth in the replay modules                                                                                                                  |
| 8   | Documentation drift                         | **Unchanged**                  | Every line-cited claim from the previous review is still in place. #1374 is open                                                                                                                                                  |
| 9   | Smaller items                               | **Unchanged**                  | All still present; `app-constants` gained a third iRacing-flavoured file (`replay-jump-playback.ts`)                                                                                                                             |

## Baseline figures

The numbers a later review should re-measure. Both columns use the same method; where the "3d8cc28b9" column differs from the previous review's own figure, see *Corrections* below.

| Measure | 3d8cc28b9 | HEAD | Method |
| --- | --- | --- | --- |
| Packages / declared `@iracedeck` edges (dep + dev) / longest chain | 38 / 178 / 7 | 38 / 180 / 7 | `packages/*/package.json`, deduplicated dep ∪ dev, DFS |
| Package cycles | 0 | 0 | DFS, with a positive control (`logger → deck-core` produces 2) |
| `deck-core` lines / files / barrel | 6,555 / 32 / 343 | 6,579 / 32 / 347 | files from git objects; the barrel is `src/index.ts` |
| `deck-core` files importing a sim package | 0 | 0 | `from` / `import()`, comments stripped |
| `plugin.ts`, Stream Deck / Mirabox / Ulanzi | 22 / 23 / 24 | 22 / 23 / 24 | `wc -l` |
| `plugin-runtime` lines / files; files importing a sim package | 2,236 / 18; 7 | 2,421 / 19; 8 (verified) | + `state-sections.ts`, which also imports the `audio-scenarios` catalog (verified) |
| `initSettings` / `initVoicePacks` | 458 / 452 | 472 / 452 | `^export function X` to the first column-0 `}` |
| `iracing-actions` non-test lines / files | 39,683 / 103 | 54,012 / 146 | git objects, minus `*.test.ts` |
| Replay stack non-test lines / test-to-source ratio | 7,566 / 1.40 | 21,338 / 1.71 | the replay reviewer's file set (item 1); test = `*.test.ts`, `*.test-support.ts`, `test-support/`, `__fixtures__/` |
| `replay-control.ts` / longest method | 2,611 / `runFastestLapWalk` 556 | 3,835 / `walkToLapStart` 803 (verified) | `wc -l`; TS compiler API spans |
| `shared/replay-command-gate.ts` | — | 2,591 lines, 27 exports (verified) | `wc -l` |
| `sim-events-iracing/src/translator.ts` / `handleTick` | 2,705 / 644 | 3,274 (verified) / 734 | `wc -l`; `function handleTick(` to the next line that is exactly `}` |
| `TranslatorState` top-level members / `wipeStateForReplay` | 220 / 119 | 261 / 207 | two-space-indent members of the type |
| `diff/caution.ts` | 611 | 1,356 (verified) | `wc -l` |
| `audio-scenarios/src/interpreter.ts` / `ScenarioEngine` span | 3,128 / 2,304 | 3,300 (verified) / 2,337 | `wc -l`; AST class span |
| `pit-crew/index.ts` / `registerPitCrew` span | 1,907 / 779 | 1,971 / 792 | `wc -l`; AST span |
| `PitCrewDeps` keys / `SimRuntime` getters | 28 / 17 | 30 / 19 (verified) | AST members; `readonly … typeof` lines |
| `calloutEnabled*` keys / keys in the frozen baseline fixture | 100 / 100 | 103 / 100 | registry families vs `settings/src/__fixtures__/callout-settings-baseline.json` |
| Catalog contracts / `CATALOG_FLOOR` | 165 / 159 | 168 / 159 (verified) | `pack-reference.json` `callouts.length`; `bundled-scripts.test.ts:123` |
| `scenario-harness/src/scenario-shortcuts.ts` / shortcut entries | 3,358 / 223 | 4,100 (verified) / 226 | `wc -l`; AST array elements |
| `audio-scenarios` non-test files importing a sim package / `getLatestTelemetry` readers outside the translator | 15 / 8 | 14 / 9 | import regex, comments stripped; the ninth reader is `telemetry-control.ts:510` |
| Unchecked `e.telemetry as TelemetryData` casts | 12 sites in 8 files | 11 in 7 | exact string |
| Action files: `SimIRacingAction` subclasses / `getCommands()` / `event-bus` | 26 / 14 / 2 | 26 / 11 / 3 | comments stripped |
| `iracing-actions` files importing `iracing-sdk` / `sim-events-iracing` | 40 / 4 | 60 / 6 | import regex; 18 of the 20 new `iracing-sdk` importers are replay files |
| Catalog events / events keyed by `subSessionId` | 84 / 2 | 90 / 5 | `event-catalog.ts` |
| Test files importing a sibling package's `src` by relative path | 16 | 20 | resolve each relative specifier |
| `_reset*` hooks / module-level nullable `let`s (repo) | 33 / 59 | 40 / 78 | `^export (function\|const) _reset`; column-0 `let` with `null` |
| Untyped `getGlobalSettings() as Record<string, unknown>` reads, `plugin-runtime` / `race-engineer-wiring` | 13 / 4 | 13 / 4 | exact string, per line |
| `.claude/rules` total / largest file | 806,235 B / 248,216 B | 876,640 B / 267,954 B | `wc -c`; largest is `race-engineer-callout-examples.md` |
| Always-loaded rules + `.claude/CLAUDE.md` | 116.8k chars | **128.9k chars** (verified) of the 130k `REPO_BUDGET` | the 9 rules with no `paths:`, plus `.claude/CLAUDE.md` |
| `CLAUDE.md` files / bytes | 30 / 474,014 | 30 / 565,932 | `git ls-files` |
| `scripts/claude-hooks` non-test / test lines; the merge gate | 2,456 / 2,326; ~353 | 2,899 / 3,393; ~770 | `wc -l`; the gate is the `gh pr merge` rule + `coderabbit-summary.mjs` + its `lib.mjs` helpers |
| Commits / `docs(specs)` commits | — | 195 / 159 (verified) | `git log --format=%s 3d8cc28b9..1980eae7f` |
| Master push CI runs since 2026-10-09 / on a `docs(specs)` head | — | 676 / 552 (verified) | `gh run list --branch master --event push` |
| Local turbo cache in the `master` checkout | 87 GB | 94 GB (verified) | `du -sh .turbo` |

### Corrections to the 2026-10-09 baseline

These figures in the previous review do not reproduce on `3d8cc28b9` with an explicit method:

- **"`audio-scenarios/CLAUDE.md` lists 8 catalog files that do not exist."** Only one does: `position-range.ts` (`:28`). `pools.ts` and the six deleted families are named in explicit "deleted in #1065" notes (verified). One of the two reviewers this round repeated the old claim; the coordinator checked and it does not hold.
- **"`ScenarioEngine` ~2,570 lines."** The AST span is 2,304; the method count (~70) was right.
- **"14 untyped settings casts in `plugin-runtime`."** 13 by exact string; 14 only if the listener cast at `settings.ts:283` is counted.
- **"15 `getCommands()` action files."** 14 with comments stripped; the fifteenth (`comms-catalog.ts`) only mentions it in a comment.
- **"Nullable `let`s 75 → 61."** 59 at the baseline by the method above; the earlier count is not reproducible.
- **"README lists 28 of 38 packages."** 29. The missing nine are `audio-native`, `audio-scenarios`, `audio-service`, `callout-script`, `event-bus`, `rasterizer`, `scenario-harness`, `sim-events-iracing` and `track-data`.
- **"Full coverage of 159 contracts."** 159 is `CATALOG_FLOOR`; the catalog had 165 contracts at the baseline and has 168 now.
- **"24 vitest aliases."** 24 entries covering 23 packages; one is the `audio-scenarios/pit-crew` subpath.
- **Rule and `CLAUDE.md` sizes were bytes.** They reproduce exactly in bytes; the always-loaded limit is stated in characters, which was 116.8k at the baseline, not 117k.
- **"194 commits."** 195 in `3d8cc28b9..1980eae7f`, one of them a merge.
- **Off-by-one inclusive spans:** `initSettings` is 458, not 457; `runFastestLapWalk` was 556, not 557.

## What is good

- **The deck layer absorbed a large feature burst without changing.**
  - The diff in 195 commits is 54 lines in `deck-core`, and none in the adapters, the shells or `plugin-build` beyond version lines (verified).
  - A dial added to an existing action (`d251b7020`) touched `iracing-actions`, two manifests and `pi-components`, plus docs, and nothing else.
- **The sim boundary held without being touched.** `SimConnection`, `deck-iracing/src` and the lint rule with its positive-control test have zero diff, and `deck-core` still has no sim import.
- **The replay stack has one sender, and a test proves it.**
  - `replay-command-gate.ts:189` is the only code that reaches `getCommands().replay` (verified).
  - `replay-command-gate.single-sender.test.ts` scans every production source for three ways around it.
  - The gate's model is a documented decision table (`:12-19`) with an injectable env and a state seam that only reports (`subscribeReplayGate`). The busy indicator reads that seam and sends nothing.
- **Clean small seams around the replay engine.**
  - `diff/replay-record-gate.ts` is 147 lines of pure resolvers that all four recorders ask and none loosens.
  - The debounced replay state lives once, on `SDKController` (#1324).
  - Replay Markers and Replay Control run the same jump sequence (`beginReplayJump` / `runReplayJump`).
  - The dial-mode files are small (34–120 lines).
  - `replay-store` has no sim import and preserves sections it cannot read.
- **#1347 is a model seam.**
  - The mechanism (a set of hold reasons) is in `audio-service` (`audio-service.ts:584-690`).
  - The policy is a fully injected controller with a structural `AudioHoldTarget`.
  - The composition order is pinned by `start-plugin.test.ts`.
  - It added no singleton, and it reused `isRaceEngineerEnabled` instead of a fourth copy.
- **The diagnostics registry adds no edge.**
  - `diagnostics` imports only `app-constants`, `logger` and `node:` modules.
  - Each owning package exports a reader of its own state (`readSimState` with a degraded fallback, `readRaceEngineerState`, 13 family `read*DebugState` readers).
  - A failing section does not cost the others (`diagnostics/src/state-sections.ts:18-24`).
  - A directory-scan test fails on a stateful family with no reader.
- **The typed wiring caught real changes.** The `getCautionLineup` → `getPaceLineup` rename and two new live dependencies flowed through `Required<PitCrewDeps>` and `SimRuntime`, so the harness followed with no drift on the engine side.
- **Reuse over duplication.** Examples:
  - `named-car.ts` was extracted, not copied.
  - `session-type.ts` is one leaf for `SessionType`.
  - `isRacingSurface` replaced three private copies.
  - #1329 hoisted `NOOP_HOLD_PREVIEW` (6 copies) and the EJS `resolveController` (17 copies).
  - The busy mark is generic, in `icon-composer` behind `assembleIcon`'s `busy` option.
- **The changelog-fragment pipeline is what it claims.** Every reader goes through `scripts/lib/changelog-fragments.mjs`, two releases have folded through it (3.6.0: 25 fragments; 3.6.1: one), and it has 113 tests across fragments, fold and release hooks.
- **The guards keep failing closed and keep their positive controls:**
  - the CodeRabbit summary reader refuses on any wording drift (`coderabbit-summary.mjs:16-19`);
  - `website-workspace-deps.test.mjs` was written from the real #1458 incident;
  - `dial-sim-agnostic.test.ts` now also refuses relative imports from outside `dial-*`;
  - nine new meta-tests were added under `scripts/`.
- **Tests grew faster than code.** `iracing-actions` test lines went from 54k to 80k, and the replay stack's test-to-source ratio rose from 1.40 to 1.71. Nine new `sim-events-iracing` tests run against recorded telemetry.

## What is bad, ranked

### 1. A replay engine lives inside `iracing-actions`, with one sender but no single owner — high

**Size.**
- The replay stack went from 7,566 to 21,338 non-test lines.
- `actions/replay-control/` went from one file to 22 (9,295 lines).
- `shared/replay-*` went from 329 to 4,939 lines.
- `replay-command-gate.ts` alone is 2,591 lines (verified) with nine module-level nullable `let`s (`:401-436`).

About 6.7k lines of this have no deck-host import at all, measured with `grep deck-core`:
- **In `shared/`:** the gate, cursor, position state, jump playback, seek, frame move, jump speed, timing and record walk.
- **In `replay-control/`:** the search queue, speed state, incident observer, record step, green-flag target, lap-walk probes and lap count.

They depend only on `deck-iracing` (`getCommands` / `getController`), `iracing-sdk`, `replay-store`, `logger` and two `settings` reads. About 1.1k more lines of the same kind sit inside the action class: `walkToLapStart` (803 lines, `replay-control.ts:2321-3123`, verified), `searchGreenFromPoleCar`, `putReplayBack` and the jump helpers.

**Ownership.** The gate centralises the *order* commands are sent in. Lifecycle and state are not centralised:

- **State about the one replay sits in nine places:**
  - the gate (search, takeover, slot, held, owed, `speedSent`, witness, timer);
  - the cursor (`replay-cursor.ts:79-95`);
  - the pending target (`replay-position-state.ts:194-196`);
  - the search runner (`replay-search-queue.ts:279-281`);
  - the dial's speed state;
  - the busy indicator;
  - the dial's camera waiter (`replay-control-track-order-mode.ts:124-130`);
  - `ReplayControl.activeFastestLapPress` (`replay-control.ts:1031`);
  - `SDKController`'s debounced state.
- **Four records of the replay speed:**
  - the key's per-context map, set optimistically even when the gate held or dropped the press (`replay-control.ts:1001-1004`, `:1398`);
  - the dial's speed state;
  - the gate's `speedSent`;
  - the cursor's `jumpSpeed`.
- **Three ways to read telemetry:** the gate's `env.readTelemetry`, the search queue's env, and the walk reading `this.sdkController` with its own `Date.now()` deadlines (`replay-control.ts:2369-2405`).
- **Leaving the replay is handled per driver.**
  - The gate's `leftReplay()` ends its search and slot but not the cursor claim (`gate:1199-1205`).
  - The walk and the search queue each notice for themselves (`replay-control.ts:2666,2742`, `replay-search-queue.ts:505`).
- **The key and the dial treat a camera switch differently.**
  - The dial's Track Order and My Car switches wait behind a jump in a newest-wins slot of their own, which is a second copy of the gate's slot outside the gate.
  - The keypad's Jump to My Car and car-number switches send `camera.switchNum` at once (`replay-control.ts:1465`, `:3688`).
- **The dial depends on the key class.**
  - Its `jumpToLive` and `jumpToRaceStart` call the key's 468-line `executeMode` (`replay-control.ts:1052-1056`).
  - The loggers of three process-wide singletons (gate, search queue, busy indicator) are set only in `ReplayControl`'s constructor (`:1095-1100`), so Replay Markers and the legacy replay actions depend on that construction implicitly.
- **Small duplicates.**
  - `ReplaySpeed` is declared twice with identical fields (`replay-control-dial-mode.ts:32`, `shared/replay-jump-speed.ts:13`, verified).
  - `HOLD_GAP_MS` is defined twice (`replay-search-queue.ts:153`, `replay-command-gate.ts:234`, verified).
  - Four separate loops poll until the replay has settled.
- **Module state across the stack:** 6 → 36 module-level `let`s, 0 → 16 mutable containers, 3 → 11 `_reset*`/`_configure*` hooks, and 7 → 18 timer call sites.

Judgement: this is the same case that moved `replay-store` into its own package in #1367. It is a process-wide, iRacing-specific service with its own state and lifecycle, sitting in a package whose unit is the action. The design was settled by experiment against the sim (the #801 / #1426 / #1430 / #1437 specs took 33 / 19 / 14 / 10 commits), and its measured facts are spread across seven module headers. A `@iracedeck/replay-engine` beside `replay-store`, with the walks pulled out of `ReplayControl` and one owner for the cursor, speed and leave-replay lifecycle, would give the next replay feature a home other than `replay-control.ts`.

### 2. The always-loaded instructions are 1.1k characters from turning `master` red — high (process)

`scripts/claude-rules-frontmatter.test.mjs:14` sets `REPO_BUDGET = 130_000` characters for the always-loaded rules plus `.claude/CLAUDE.md`. They are at 128.9k (verified), up 12.1k in eleven days. At that rate the test goes red in about a day of ordinary rule edits, and that would turn `master` red in the middle of an unrelated PR.

The growth is per feature, not per package; no package was added:
- `.claude/CLAUDE.md`'s package list grew from 13.1k to 14.8k bytes. Its `sim-events-iracing` bullet alone doubled (977 → 1,983 bytes), accumulating issue numbers and module names.
- `changelog.md` grew from 6.0k to 9.7k.
- `code-review.md` grew from 7.6k to 9.5k.
- `hooks.md`'s single `gh pr merge` table cell grew from 949 to 2,532 bytes, and `issue-workflow.md` restates it.

The 150k limit the budget protects is shared with the user's global rules and auto-memory index; with those, the total is about 147.7k (unverified that they count against the same limit).

Judgement: the package list in `.claude/CLAUDE.md` is an index and should read like one, a line per package pointing at the package's own `CLAUDE.md`, which is path-scoped and already holds the detail. The merge-gate mechanics belong in a path-scoped section, not an always-loaded table cell.

### 3. The god modules keep absorbing features — medium

| File | 3d8cc28b9 → HEAD | What drove it |
| --- | --- | --- |
| `iracing-actions/.../replay-control.ts` | 2,611 → 3,835 | the replay features (item 1); `walkToLapStart` is 803 lines, `executeMode` 468 |
| `sim-events-iracing/src/translator.ts` | 2,705 → 3,274 | #1167 +202, #1387 +119, #254 +87, #1426 +62, #1354 +30; `handleTick` 644 → 734, 31 → 35 diff calls |
| `sim-events-iracing/src/state.ts` | 1,591 → 2,026; 220 → 261 members | 18 `cautionCatchUp*` and 13 `replay*` members added to the flat bag |
| `sim-events-iracing/src/diff/caution.ts` | 611 → 1,356 | #1167/#1168 |
| `scenario-harness/src/scenario-shortcuts.ts` | 3,358 → 4,100 | #1167 added 738 lines for 3 shortcuts; the growth is in the helper section (1,506 → 2,241 lines before the array) |
| `audio-scenarios/src/interpreter.ts` | 3,128 → 3,300 | #1387's `describeState`, mostly outside the class |
| `plugin-runtime/src/phases/settings.ts` (`initSettings`) | 458 → 472 | the replay store's five bus subscriptions live in it (`settings.ts:221-249`) |
| `replay-store/src/replay-session-store.ts` (`createReplaySessionStore`) | 747 → 1,050 | one closure; each new section is hand-written into the record fields, `recordOf` and `serialize` (~100 lines per section) |

**Two specific misplacements:**

- **The translator now has a second job.** The four replay recorders (`translator.ts:2432-2499`) feed `replay-store`, not the Race Engineer. Each re-reads `getSessionInfo()` with a cast, adds state members, and is deliberately inside or outside `wipeStateForReplay`'s wiped set. Where they run relative to the replay guard is load-bearing, and only about 50 lines of comments (`:2420-2476`) say so. They already share their own eligibility gate and need only telemetry, session info and `inReplay`.

  Judgement: a sibling controller subscriber is the cleanest first extraction from the translator.
- **The replay store's wiring is in the settings phase.** Its subscriptions sit in `initSettings`, which #1377 already proposes to split.

### 4. The sim seam above the deck layer: one leak closed, three opened — medium for a second sim

- **Closed.** `catalog/pit-crew/rolling-start.ts` now takes injected `RollingStartContractDeps` instead of importing `getSessionType`. That is the shape #1378 asks for, and it brings the catalog from 15 to 14 files.
- **A third access path from actions to the sim.**
  - Actions used to reach the sim through the SDK controller and, in two files, the bus. They now also call translator getters directly:
    - `telemetry-control.ts:510` (`getLatestTelemetry`);
    - `session-info.ts:54,1077,1536` (`getLastStopTireTemps`, verified).
  - `iracing-actions` files importing `sim-events-iracing` went from 4 to 6, and `event-bus` importers from 2 to 3.
- **`plugin-runtime` knows the sim in 8 files.** The new `state-sections.ts:19-29` imports `deck-iracing`, `sim-events-iracing` and the `audio-scenarios` pit-crew catalog (verified). `architecture.md:309` still says "`initSim` is the only phase that knows which sim is running".
- **`SimRuntime` is 19 `typeof` iRacing getters (verified).** The new ones are `carIdx`-shaped (`PaceLineup.followCarIdx`, `RollingStartFormation.firstFollowCarIdx`).
- **Replay events keyed on iRacing identities.** `replay.greenFlag`, `replay.sessionStarted` and `replay.incident` carry `subSessionId`, `sessionUniqueId`, `carIdx` and `carNumberRaw` (`event-catalog.ts:1167-1243`). Events keyed by `subSessionId` went from 2 to 5.

  The three new Race Engineer events (`caution.gapOpened`, `caution.gapStillOpen`, `rollingStart.lineup.changed`) are sim-neutral by construction: empty payloads, with the value read live.
- **Unchanged:**
  - `"Lone Qualify"` is still compared at `radar-engine.ts:162` and `spotter-engine.ts:365`;
  - `pitService.statusChanged` still carries raw `irsdk_PitSvStatus` (`event-catalog.ts:683`);
  - `phases/sim.ts:5` still imports the catalog's `OPPONENT_PENALTY_FLAG_TO_CALLOUT_ID`;
  - the module-level `getSDK()` calls are still at `car-control.ts:176` and `tire-service.ts:216`.

### 5. Spec churn now dominates `master` — medium (measured; the cost is judgement)

- **Commit share.** 159 of 195 commits are `docs(specs)` (verified); 37 landed on 2026-10-12 alone.
- **Size.** 30 spec files were added and 10 amended, and spec bytes written (920k of churn) equal the non-test source bytes written (940k).
- **Per issue:** #801 took 33 spec commits (58 KB), #1426 19 (43 KB), #1430 14 (68 KB), #1437 10 (39 KB).
- **As-built notes.** Many amendments are as-built notes, such as "Frame Jog as built" and five follow-ups to #1316 in one afternoon. That is an implementation diary, not the decision record `specs-and-plans.md` describes.
- **CI cost.** Each spec push runs all four CI workflows, which have no `paths` filter. That came to 552 of 676 master push runs since 2026-10-09 (82%, verified).

The freeze rule held: the only post-ship edit, to the #1253 spec, was its `Superseded by:` pointer.

Judgement:
- A `paths-ignore` for `docs/superpowers/specs/**` on the four CI workflows is a one-line fix per workflow.
- Keeping as-built detail in the PR body, or batching amendments, would cut most of the rest. Neither change touches the decision record itself.

### 6. Drift no compiler sees has started to happen — medium-low

- **Persisted keys outside the guard.** Three keys have no rename or default guard: `calloutEnabledCautionCatchUp`, `calloutEnabledCautionCatchUpReminder` and `calloutEnabledRollingStartLineupChanged`.
  - The registry holds 103 keys; the frozen baseline fixture holds 100.
  - The family tests pin each new key's name and label (`caution.test.ts:3226-3235`, `rolling-start.test.ts:291-295`), but nothing central does.
  - The previous review called this hypothetical; it is now real.
- **`CATALOG_FLOOR` is still 159 against 168 contracts** (verified), although its comment says to bump it when a callout is added.
- **The harness's translator still gets none of the six options the plugin passes** (`scenario-harness/src/main.ts:66` vs `plugin-runtime/src/phases/sim.ts:27-61`, verified).
- **Settings ↔ translator constants are still mirrored by comment only** (`settings/src/global-settings.ts:423,438`).
- **The 17 untyped settings casts are unchanged.**
- **The harness lists are complete only by convention.** All 78 bus events have an `EVENT_TEMPLATES` entry today, but no test compares `EVENT_TEMPLATES` with `SimEventMap`, or the shortcut list with the contracts.

All of this is in #1376's scope except the floor and the harness lists, which belong with it.

### 7. A new callout family still costs the same — medium-low

- **#1167** (3 callouts, 2 new dependencies, a rename) touched 141 files in 10 packages (verified):
  - 35 hand-written source files;
  - 45 `audio-assets` files across both voices;
  - 17 harness files.
- **The live dependency `getCautionGap`** took 10 sites in 3 packages.
- **Quiet conditions are threaded by hand.** #1171 added no `PitCrewDeps` key, but threaded `getUnderFullCourseCaution` positionally into 5 builders (`index.ts:1620-1711`). Each family re-checks it at event time and again at speak time (`gaps.ts` 12 references, `race-status.ts` 7).

  Judgement: "stay quiet under caution / after the finish" is a contract-level declaration, not per-family code.
- **The registry held.** Each of the 3 new opt-in keys is one line in its family file, and `registerPitCrew` grew by only 13 lines for 3 callouts.

### 8. The dial family has no base — medium-low

- **The copied wrapper.** #1329 enforced its late-event rule by pasting a ~22-line `inputContext` wrapper into all 18 dial surfaces (verified), each with its own `ensureContext`.
- **Repetition.** About 14% of the family's lines repeat verbatim (1,808 occurrences of 120 shared lines in 13,178).
- **No family-wide check.** No test checks that every surface follows the rule.
- **Relative imports.** Test files reaching `deck-core/src/dial-gesture.js` by relative path went from 10 to 13, because each new surface's test copies the import.
- **Two text-width models in the dial kit.** A 0.6 em average (`dial-fit.ts:12`, repeated inline at `dial-caption-mark.ts:121,133` and `fuel-dial-surface.ts:763,774`) sits beside a measured Arial Bold table (`dial-text-width.ts`).

Judgement: a shared surface base or composite would make the rule structural; its host-generic half (context lifecycle, gestures) fits beside `dial-gesture` in `deck-core`.

### 9. The native addons still fall back silently to fake data — medium

Unchanged. The only commit to either `index.ts` (#1411) changed a comment. #1373 is still the cheapest fix in either review and is still open.

### 10. The merge gate doubled and now parses an undocumented third-party format — medium-low

- **Size.** The gate went from about 353 to about 770 lines.
- **Factoring.** It is well factored: `coderabbit-summary.mjs` is pure, with 585 test lines.
- **The dependency.** It reads CodeRabbit's hidden HTML markers (`change_assessment_commit`, `final_review_risk_coverage`, `coderabbit-summary.mjs:33,50-56`), which CodeRabbit can change without notice.
- **The risk.** It fails closed by design, so the risk is merges that stop being accepted, not a false acceptance.
- **The prose.** Its description is the largest single cell in an always-loaded file (item 2).

### 11. Documentation and hygiene — low

- **Every documentation claim the previous review cited by line is still in place:**
  - `architecture.md:6` ("one new adapter … nothing else changes");
  - `architecture.md:309` (`initSim`, #1351);
  - the Mermaid graph without `audio-scenarios → iracing-sdk`;
  - "only package allowed" at `translator.ts:6-7` and `phases/sim.ts:21-23`;
  - `sim-runtime.ts:6-7` and `race-engineer-wiring/CLAUDE.md` pointing at #1351;
  - `.claude/CLAUDE.md`'s "ten startup phases" (l.47) and "three consumers" (l.20);
  - the README's 29 of 38 packages, with both "33" and "32" actions, while `SHARED_ACTIONS` registers 35 classes.
- **New documentation drift:**
  - `architecture.md:475` says 15 catalog files import a sim package; it is now 14.
  - `audio-scenarios/CLAUDE.md` does not name `driver-name-clip.ts`, `opponent-flags.ts`, `pit-speeding-engine.ts` or `tire-wear.ts`.
  - `icon-composer/src/busy-mark.ts:2` and `index.ts:20` call a generic API "Replay busy marks" (verified).
- **The commit-trailer rule has no enforcement.** `hooks.md` says it "belongs to husky", but `.husky/` has only `pre-commit`. Three commits since the baseline carry a `Claude-Session:` trailer (verified):
  - `6054a605f`, a squash-merged PR;
  - `efa689289` and `073040128`, pushed straight to `master`.
- **Misfiled runtime code.**
  - The audio session hold is a runtime service that lives in `iracing-actions/src/audio/` and reaches `startServices` through a re-export in `plugin-runtime/src/actions.ts` (verified).
  - `//#generate:changelog-data` is a root task that writes into `iracing-actions/src/actions/data/`.
- **The website's engine reach grew.** It now has 10 hand-listed `$TURBO_ROOT$` inputs, up from 8. Since #1458, two dependencies are declared only to get turbo's `^build` ordering; the website imports neither by name.
- **Test-support files.** `iracing-actions` names test-support files three ways (`*.test-support.ts`, `shared/test-support/`, `*.fixture-cases.ts`), 786 lines that count as source.
- **Still present from the previous review:**
  - the dead `iracing-plugin-stream-deck/src/shared/index.ts`;
  - the near-identical Mirabox and Ulanzi file loggers (they differ only in their log prefixes);
  - the `switchToProfile` no-ops nobody calls through the interface;
  - `fake-sim-connection.ts` shipping in `dist`;
  - `pi-components`' three copied helper sets;
  - `turbo.json`'s three identical plugin `build` blocks (#1386 had to edit all three);
  - `iracing-sdk`'s generic helpers;
  - the sim-boundary lint not covering `settings` and `voice-packs`.
- **The local turbo cache** is 94 GB and unpruned (operational).

## Extensibility

| Change | 2026-10-09 | Now | Why |
| --- | --- | --- | --- |
| New first-party voice pack | Low-moderate | Same | A parity-guarded `FIRST_PARTY_SCRIPTS` row and a hand-written `CLIP_SOURCE_FLOOR` entry, plus coverage of 168 contracts (not 159) |
| Third-party voice pack | Low, by design | Same | Unchanged |
| New action | Moderate, linear | Same | About 8 touch points; no action was added in this window |
| Dial on an existing action | — | Low-moderate | Contained to `iracing-actions`, the manifests and `pi-components`, but each surface still copies about 100 lines of lifecycle code (item 8) |
| New callout in an existing family | Low-moderate | Same | One registry line and a small `registerPitCrew` delta, plus about 240 harness lines, both voices' clips and catalog bumps, and a floor nobody bumps |
| New callout family | Moderate-high | Same, slightly higher | A live dependency is about 10 sites in 3 packages; a stateful family now also owes a debug-state reader (guarded); quiet conditions are threaded by hand |
| New replay mode | — | Moderate-high | Lands in `replay-control.ts` or `shared/`, touching the gate, cursor, speed state and leave-replay handling, with no owner to keep them consistent (item 1) |
| New deck host | Medium | Same | Nothing in the adapter seam moved: a copied logger, a triplicated turbo block and a hand-written manifest |
| Second sim | High | High, slightly worse in mass | The deck layer still carries over. Against it: the translator is 21% larger, now including replay recording; `SimRuntime` has two more iRacing getters; three more events are keyed on iRacing identities; and almost all of the 21k-line replay stack is iRacing's (its broadcast commands, 2048-frame jump timing, `SessionNum` and `CarIdx` arrays) |

The previous review's conclusion holds, with one addition: a second sim would get the Race Engineer events the catalog expresses neutrally, and no replay features at all. Only `replay-store`'s persistence, the busy indicator and the shape of the gate's rules are not iRacing-specific. Making the replay engine generic should wait for a real second SDK, but packaging it (item 1) would make that boundary visible.

## The process layer

Judgement, not a defect.

- **Rules and `CLAUDE.md` files.** With no new packages, the rules grew by 70k (to 877k bytes) and the `CLAUDE.md` files by 92k (to 566k). The largest per-file growth:
  - `iracing-actions/CLAUDE.md`, +23k
  - `race-engineer-callout-examples.md`, +20k
  - `scenario-harness/CLAUDE.md`, +19k
  - `encoders-and-touchscreen.md`, +18k
  - `audio-native/CLAUDE.md`, +11k
- **Docs outpace the fix.** The documentation layer is growing faster than #1374, which was meant to correct it, and none of #1374's items moved.
- **Process outweighs product on `master`.** By commit count (159 of 195) and by CI runs (82%), process now outweighs product changes on `master`.
- **The second read earns its place.** Across 18 PR-body tallies (#1399–#1459), about 72 findings were raised and about 70 held; #1443 alone had 14 raised and 14 held. That makes its "ten runs with none held" removal clause moot for now. Caveat: the reviewer grades "held" itself.

The always-loaded budget (item 2) is the one process measure with a hard edge, and it is the closest to it.

## Follow-ups

Filed or updated with this review, in the order it would take them:

1. [#1465](https://github.com/niklam/iracedeck/issues/1465): **keep the always-loaded instructions inside the budget** (item 2). Cut `.claude/CLAUDE.md`'s package list to one line per package that points at the package's own `CLAUDE.md`, and move the merge-gate mechanics out of `hooks.md`'s always-loaded table into a path-scoped section. Small, and urgent because the next rule edit can turn `master` red.
2. [#1373](https://github.com/niklam/iracedeck/issues/1373) as it stands: the silent native mock fallback (item 9). Still the cheapest fix on the list.
3. [#1466](https://github.com/niklam/iracedeck/issues/1466): **skip CI for spec-only pushes** (item 5): `paths-ignore: docs/superpowers/specs/**` on the four `ci-*.yml` workflows. One line each.
4. [#1467](https://github.com/niklam/iracedeck/issues/1467): **extract `@iracedeck/replay-engine`** (item 1). Move in the gate, cursor, position state, jump playback, seek, frame move, jump speed, timing, record walk, search queue, speed state, incident observer, record step and green-flag target. Inject the two settings reads and set the loggers at the composition root. Pull the walks out of `ReplayControl`, with one telemetry path and one clock. Keep one speed model and one camera-switch policy. Move the store's bus wiring out of `initSettings`. A refactor with no behaviour change, best done before the next replay feature.
5. [#1376](https://github.com/niklam/iracedeck/issues/1376), widened (item 6) with the three unguarded callout keys, a central guard for keys added after the baseline, `CATALOG_FLOOR`, and harness-list completeness tests (`EVENT_TEMPLATES` ↔ `SimEventMap`, a shortcut per contract).
6. [#1468](https://github.com/niklam/iracedeck/issues/1468): **move the replay recorders out of the translator** (item 3) into a sibling controller subscriber that owns its own state. This is a first cut at `translator.ts` that a second sim would not have to reproduce.
7. [#1374](https://github.com/niklam/iracedeck/issues/1374), [#1375](https://github.com/niklam/iracedeck/issues/1375), [#1377](https://github.com/niklam/iracedeck/issues/1377) and [#1378](https://github.com/niklam/iracedeck/issues/1378) as they stand, with this review's additions recorded on each as a comment: the new doc drift (item 11), the 16 → 20 test reaches and the eslint coverage (item 4), `initSettings` hosting the replay store, and the new translator-getter readers in actions.

Not proposed: a dial-surface base (item 8), contract-level quiet conditions (item 7), the remaining god modules (`interpreter.ts`, `scenario-shortcuts.ts`, `diff/caution.ts`) and the singleton naming. Each is worth doing, but the replay-engine extraction (#1467) and the recorder cut (#1468) are where growth is concentrated now.
