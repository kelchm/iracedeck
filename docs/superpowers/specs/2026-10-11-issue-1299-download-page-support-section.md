# A Support Section on the Download Page

> **Issue:** [#1299](https://github.com/niklam/iracedeck/issues/1299) · **Supersedes:** _none_ · **Superseded by:** _none_
>
> Point-in-time design record. The code and `.claude/rules/` are the truth; this is not documentation.

## Problem

The homepage ends with a line asking for support and a Buy Me a Coffee button. The Download page, where many visitors land directly from a store listing, a Discord link or a search result, has nothing like it. The homepage's version is also a `<script>` tag typed into `index.mdx`, so a second page would mean a second copy of the slug, the button and the wording.

Since the issue was filed there is an iRaceDeck Supporter membership on Buy Me a Coffee, which neither page mentions. #1303 adds the same link to the Settings window and takes its link target and wording from here.

## Decisions

**One component and one data module.** `SupportSection.astro` renders the whole section and both pages use it. The URLs live in `src/data/support.ts`, beside `brands.ts`, as three string constants: the support page, the membership page and the button image. A data module rather than the component's frontmatter because a test can import it, which is how `website-device-list.test.mjs` reads `brands.ts`.

**The button goes to the general page, and a second line names the membership.** The button links to `https://buymeacoffee.com/niklaslampen`, where a one-off and the membership are both one click away. Below it a quieter line links straight to `https://buymeacoffee.com/niklaslampen/membership`. Pointing the button itself at the membership was rejected: a visitor who wants to give once would have to find their way back to the general page.

**The wording.** In order:

1. "iRaceDeck is free — servers and Red Bull aren't. Any support is genuinely appreciated." This is the homepage's line, unchanged.
2. The button, with the text "Support iRaceDeck".
3. "Or become an iRaceDeck Supporter for a monthly amount and get the Supporter role on Discord." The words "iRaceDeck Supporter" are the membership link.

The copy names no tier and no price. The membership has three tiers today, and a line that listed them would be wrong the day one changes.

**The button is Buy Me a Coffee's image, inside a plain link.** The homepage loads `cdnjs.buymeacoffee.com/1.0.0/button.prod.min.js`, which writes the button into the page. The component instead renders an `<img>` from `img.buymeacoffee.com/button-api/` inside an `<a>`, with the parameters the script was given: slug `niklaslampen`, text "Support iRaceDeck", the 🏎️ emoji, the Cookie font, `FFDD00` for the button, `000000` for the text and the outline, `ffffff` for the cup. Three reasons:

- No third-party script runs on iracedeck.com for it.
- The link's target is ours to set. The script only ever links to the slug's page.
- It is what #1303 uses in the Settings window, where a script is not acceptable, so both surfaces draw the same button.

Measured on 2026-10-11, that URL returns a 21,442-byte `image/svg+xml` with a `viewBox` of `0 0 253 50`, the font embedded and no `<script>`. The `<img>` carries `width="253"` and `height="50"` so the page does not shift when it loads, and the alt text "Support iRaceDeck on Buy Me a Coffee". The two numbers live beside the URL in `support.ts`, because the image's width follows its `text` parameter.

**The image is not a copy of the script's button, and that is accepted.** Rendering it showed three differences: it is 253 × 50 where the script's button is about 233 × 60, the emoji is drawn at 16px beside 24px text, and a darker panel on the right carries a heart and the page's current number of supporters (20 on the day). It is the number Buy Me a Coffee had when it drew the image, not a live one: the response is cached and may be kept for a year (`Cache-Control: max-age=31536000`), so a returning visitor can see an old count. No parameter was found that leaves it out. Two alternatives were put to the maintainer with that in view: a button of our own in CSS, matching the script's look with no count and no request to Buy Me a Coffee until it is clicked, and keeping the script. He chose the image, count included.

Committing a copy of the SVG to the site was considered and left out. The site makes no promise of third-party-free pages (it loads Google Analytics and Google Fonts), the hotlinked image is the one request that replaces the script's, and a copy would be a file to refresh by hand whose supporter count stops on the day it was fetched. The plugin is a different case, and #1303 decides it for itself.

**Both links open in a new tab,** as the script's button did. On the Download page the visitor may be about to click a download button, and should not lose the page.

**Placement.** The first build put the section last on both pages, as the issue asked, and the maintainer moved it up: at the bottom it was the least important thing on either page. On the Download page it sits directly under the three download cards, before the installation-guide line and without a divider, so it reads as part of the download block while every download button still comes first. Above the cards was rejected, since a visitor would meet the ask before the download. On the homepage it is its own block between "Works with" and the category cards, with a `section-divider` rule on each side, and the block that used to end the page is gone.

**The class is renamed with the markup.** `.donate-section` in `custom.css` becomes `.support-section`, matching the component, and gains the rules the second line and the image link need. The styles stay in `custom.css`, where every other component's are.

**What #1303 takes from here.** The two link targets, the three lines of wording and the button's parameters. The website imports nothing from the plugin and the plugin nothing from the website, so the plugin holds its own copy. Whichever of the two changes merges second adds a test that reads both and fails when they differ.

## Out of scope

- The Settings window's Support tab (#1303).
- An analytics event for clicks on the button or the membership link.
- The support link on the developer docs page (`docs/development/index.md`), which already points at the same URL and stays a plain Markdown link. The test below holds it to that URL.
- A support section on any other page, or in the docs sidebar or footer.
- Tier names, prices or benefits beyond the Discord role.

## Testing

- `scripts/website-support-section.test.mjs`:
  - `index.mdx` and `downloads.mdx` each import and render `SupportSection` once. On the Download page it comes after the last download link and before the installation-guide line; on the homepage, between "Works with" and the category cards.
  - No website source loads the `cdnjs.buymeacoffee.com` script.
  - Every `buymeacoffee.com` URL in the website's sources is one of the three constants, so a hand-typed copy that drifts fails.
  - The three constants are `https`, the membership URL is the support URL plus `/membership`, and the image URL carries the same slug.
- `pnpm --filter @iracedeck/website build` and `pnpm typecheck`.
- Manual, on the built site:
  - `/` and `/downloads/` both show the section in its place, in the dark and the light theme and at phone width.
  - The button opens the general page and the "iRaceDeck Supporter" link the membership page, each in a new tab.
  - On `/downloads/` the section is under the download cards and the page does not jump when the image loads.
