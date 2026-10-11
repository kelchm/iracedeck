# A Support iRaceDeck Tab in the Settings Window

> **Issue:** [#1303](https://github.com/niklam/iracedeck/issues/1303) · **Supersedes:** _none_ · **Superseded by:** _none_
>
> Point-in-time design record. The code and `.claude/rules/` are the truth; this is not documentation.

## Problem

iRaceDeck asks for support in one place, the bottom of the website's homepage, and #1299 adds the Download page. Plenty of users never go back to the website after installing. The Settings window is the part of iRaceDeck they spend time in, and it says nothing about the project being free, who pays for it, or how to chip in.

The issue settles the shape: a tab of its own, written properly, silent for everyone who does not open it. This spec decides what the issue left open: the copy, how the button image is loaded, whether the free ways to help are listed, and what keeps the window and the website saying the same thing.

## Decisions

**A tab of its own, and the last one.** A `support` pane with the nav button "Support iRaceDeck", after Diagnostics, so it never sits between the user and a setting. It carries no badge, nothing links or jumps to it from another tab, and no startup path opens the window on it. Whoever goes looking finds it; nobody else is shown it.

**The pane is a partial, laid out as prose.** `packages/pi-components/partials/settings-window-support.ejs`, included by `settings-window.ejs` like the Getting Started and What's New panes. Its styles live with the rest of the window's CSS in `settings-window.ejs` under an `.sw-support-` prefix, the `.sw-gs-` and `.sw-cl-` precedent. It binds no setting and has no script.

**The copy.** In order, exactly:

1. Heading: "Support iRaceDeck".
2. Subtitle: "iRaceDeck is free — servers and Red Bull aren't. Any support is genuinely appreciated." This is the website's lead line, unchanged (#1299).
3. "There is no paid version and nothing is locked: every action, every Race Engineer callout and every voice is yours whether or not you ever give anything."
4. "iRaceDeck is built and maintained in spare time. Beyond that time, the main costs are AI subscriptions and the servers behind the website. If iRaceDeck has earned a place on your rig and you would like to chip in, this is where. Nothing is ever expected."
5. The button, with the text "Support iRaceDeck".
6. "Or become an iRaceDeck Supporter for a monthly amount and get the Supporter role on Discord." The words "iRaceDeck Supporter" are the membership link. This is #1299's line, unchanged.
7. Heading: "Other ways to help", then "None of these cost anything, and they help just as much." and three links: "Join the Discord", "Report a bug or suggest a feature", "Star iRaceDeck on GitHub".

The costs named are the real ones (Niklas, 2026-10-11): time, AI subscriptions, and the website's servers. Test hardware is not named, although the issue lists it, because the decks were given to the project and many users know that. The copy names no tier and no price, for #1299's reason: a line that listed them would be wrong the day one changes.

**The free ways to help are listed, without a store rating.** The issue left the list to this spec, and it is in (Niklas, 2026-10-11): the tab is the one place a user who cannot or will not pay is told what else helps. A "rate iRaceDeck in the store" row is out, because neither the Elgato Marketplace nor Mirabox Space has ratings. That also means the template needs no knowledge of which deck the build is for.

**The link targets are #1299's.** The button goes to `https://buymeacoffee.com/niklaslampen` and the membership link to `https://buymeacoffee.com/niklaslampen/membership`, as decided in `docs/superpowers/specs/2026-10-11-issue-1299-download-page-support-section.md`. The other three are the ones the website already uses: `https://discord.gg/c6nRYywpah`, `https://github.com/niklam/iracedeck/issues` and `https://github.com/niklam/iracedeck`.

**The button is Buy Me a Coffee's image, loaded the way the website loads it.** An `<img>` from `img.buymeacoffee.com/button-api/` inside the link, with the URL and parameters #1299 uses, so both surfaces draw the same button from the same address: `https://img.buymeacoffee.com/button-api/?text=Support%20iRaceDeck&emoji=%F0%9F%8F%8E%EF%B8%8F&slug=niklaslampen&button_colour=FFDD00&font_colour=000000&font_family=Cookie&outline_colour=000000&coffee_colour=ffffff`. Never the `cdnjs.buymeacoffee.com` widget script, which the issue rules out: the image looks the same and runs no code, and the script would run inside a page that holds a write connection to the user's settings. Measured on 2026-10-11, the image URL returns a 21,442-byte `image/svg+xml` with a `viewBox` of `0 0 253 50`, the Cookie font embedded and no `<script>`; an SVG drawn by an `<img>` runs no script and follows no link in any case.

Shipping a committed copy of that SVG in the plugin was considered and declined (Niklas, 2026-10-11): the window does it the same way the website does. The copy would have kept the window free of third-party requests and working offline on this tab, at the price of a file to refresh by hand and a font licence entry. What the hotlink costs is handled by three attributes and one accepted limit:

- `loading="lazy"`. The pane is `hidden` until its tab is chosen, and a lazy image with no box is not fetched, so the request goes out when a user opens the tab and not every time the window opens. The manual test checks this in the browser, because it is the behaviour the tab's "silent for everyone else" rests on.
- `referrerpolicy="no-referrer"`, so the request tells Buy Me a Coffee nothing about the page that made it.
- `width="253"`, `height="50"` and the alt text "Support iRaceDeck on Buy Me a Coffee", as on the website, so the layout does not shift and the link still reads as what it is when the image cannot load.
- Offline, or with the image blocked, the button is its alt text inside the link. The link works exactly the same. No fallback artwork is drawn.

**Every link is a plain anchor.** `<a href="https://…" target="_blank" rel="noopener noreferrer">`, like the header links and the What's New tab. The window's shared external-link handler already sends every `http(s)` anchor click through `openUrl` to the plugin, which opens the user's own browser. No new code, no new command.

**The URLs live in both places, and a test holds them together.** The website imports nothing from the plugin and the plugin nothing from the website. The partial declares the three values under the names the website's `src/data/support.ts` exports: `SUPPORT_URL`, `MEMBERSHIP_URL` and `SUPPORT_BUTTON_IMAGE_URL`. Each side carries a comment naming the other. `scripts/support-link-parity.test.mjs` imports the website module, reads the partial as text and fails when any of the three differ. Whichever of #1299 and #1303 merges second adds it.

**The window runs no code from another site, and that becomes a rule.** The page holds a write connection to the user's settings. Until now nothing in it came from a third party by habit; with a hotlinked image in the page, the line has to be written down. The rule in `.claude/rules/settings-window.md`: no `<script>`, stylesheet, font, frame or media element in the window names another origin; an image may, only when it is on a short allow-list that today holds the one button image URL; and an anchor is the only other element that may carry an absolute URL. A test enforces it in two places: over the source of `settings-window.ejs` and every partial, which always runs, and over each plugin's built `ui/settings-window.html` when the build is present. The matcher has a positive control, a fixture holding a remote script and an unlisted remote image that it must flag.

**The screenshot of this tab needs the network.** `pnpm capture:settings` reads nothing from the network today. The Support tab's shot shows the hotlinked button, so the capture waits for the pane's images to finish loading, bounded by a timeout, before it takes that shot, and `.claude/rules/website-screenshots.md` records that this one screenshot depends on Buy Me a Coffee answering. A capture taken offline shows the alt text and is not committed.

**The artifacts that follow a new tab.**

- `scripts/lib/settings-window-capture/tabs.mjs`: `{ pane: "support", label: "Support iRaceDeck", file: "support.png", elgatoOnly: false }`, last. `tabs.test.mjs` fails until the list matches the built page.
- `pnpm capture:settings` for `support.png`.
- A `### Support iRaceDeck` section on `docs/getting-started/settings.md`, after Diagnostics. It says what the tab is and that its button image is loaded from Buy Me a Coffee when the tab is opened, the only thing the window loads from another site.
- `.claude/rules/settings-window.md` (the architecture row, the rule above), the partial list in `.claude/rules/pi-templates.md`, and `.claude/rules/website-screenshots.md`.
- A `Features` fragment in `changelog.d/`, weight 30.

## Out of scope

- A banner, badge, toast, first-run mention or any other nudge towards the tab, in the window or in a Property Inspector.
- Counting or reporting clicks on any of the links.
- A setting that hides the tab.
- A `Content-Security-Policy` header on the settings server. It would enforce the new rule in the browser rather than in a test, but it is a change to the request path that is reviewed at `max`, and it gets an issue of its own.
- A committed copy of the button image, or fallback artwork for when it cannot load.
- The website's support section (#1299).
- Tier names, prices or benefits beyond the Discord role.
- A store-rating link, until a marketplace has ratings.

## Testing

- `packages/pi-components/src/build/settings-window-support-partial.test.ts` renders the real partial:
  - The copy above is on the page, line for line.
  - Every anchor is `https`, has `target="_blank"` and `rel="noopener noreferrer"`, and `resolveExternalUrl` returns its URL, which is the condition for the click to go through `openUrl`.
  - The button's anchor points at `SUPPORT_URL` and wraps an `<img>` whose `src` is `SUPPORT_BUTTON_IMAGE_URL`, with `loading="lazy"`, `referrerpolicy="no-referrer"` and the width, height and alt text above.
  - The partial contains no `<script>` and no element that stores a setting.
- The no-third-party-code test described under Decisions, including its positive control.
- `tabs.test.mjs` passes with the new entry, and `support.png` is committed.
- The capture harness's tests cover the wait for a pane's images and its timeout.
- `pnpm test`, `pnpm typecheck`, `pnpm build`, and `pnpm --filter @iracedeck/website build`.
- Manual, in the built plugin:
  - The tab is last in the sidebar, and no other tab mentions it.
  - The button opens `https://buymeacoffee.com/niklaslampen` in the default browser, and the "iRaceDeck Supporter" link the membership page. The three other links open their pages the same way.
  - With DevTools open on the window, reloading it shows no request to any host but `127.0.0.1`; opening the Support tab then shows exactly one, to `img.buymeacoffee.com`, with no `Referer` header.
  - With the network disconnected, the tab renders with the alt text in place of the button, and the link still opens the browser.
  - The window at its narrowest width does not scroll sideways on the tab.
