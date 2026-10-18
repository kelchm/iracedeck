# Discord tool: the #bugs forum

> **Issue:** [#1460](https://github.com/niklam/iracedeck/issues/1460) · **Supersedes:** _none_ · **Superseded by:** _none_
>
> Point-in-time design record. The code and `.claude/rules/` are the truth; this is not documentation.

Extends the feature-requests tracker of [#1114](2026-09-04-issue-1114-discord-feature-requests-tracker.md) to the Discord **#bugs** forum. The issue carries the problem and the task list; this records how.

## Decision: one tool, a profile per forum

The two forums differ only in data: the channel, the tags a post can carry, and which tag an issue's state maps to. Everything else is the same code: the REST client, `list` / `show` / `reply` / `tag`, the source line, the release lookup, the no-mentions rule and the standing-post refusal. So the tool gains a **forum profile** rather than a sibling script.

```javascript
// scripts/lib/discord-forums.mjs (shape, not final names)
export const FORUMS = {
  "feature-requests": {
    channelEnv: "DISCORD_FEATURE_REQUESTS_CHANNEL_ID",
    statusTags: ["Will Add", "In progress", "Completed", "Released", "Won't do"],
    expectedTag: featureRequestTag, // today's expectedTag, unchanged
  },
  bugs: {
    channelEnv: "DISCORD_BUGS_CHANNEL_ID",
    statusTags: ["Resolved"],
    expectedTag: bugTag,
  },
};
```

- **Every command takes `--forum <name>`, defaulting to `feature-requests`**, so every existing invocation and the skill's current text keep working. A root `pnpm discord:bugs` script is the same entry file with `--forum bugs`.
- **The channel check stays, per profile.** `fetchPostThread` refuses a thread whose parent is not the selected forum's channel, so a reply cannot land in the wrong forum by a pasted id.
- **`readConfig` requires only the selected forum's channel variable.** A machine without `DISCORD_BUGS_CHANNEL_ID` keeps using feature-requests.
- **The #bugs mapping** (`bugTag`): an issue closed as completed whose shipping commit is in a stable `vX.Y.Z` tag → `Resolved`. Every other state → `null`, so nothing is proposed: the forum has no "tracked" or "in progress" tag, and an open bug's thread is answered by hand. Closed as not planned or duplicate → `null` as well; those get a hand-written reply.
- **`follow-up` covers both forums in one run.** It reads every `discord`-labelled issue as today, resolves each source line's post id against both channels' posts, and applies the profile of the forum the post is in. A source line pointing at neither is reported as today's "post not found".
- **Source line.** One shape for both forums, so one parser: `Requested on Discord: …` stays for feature requests, and bug issues use `Reported on Discord: <post link> by <handle> (<votes> ❤️)`. The parser accepts both verbs and nothing else.
- **Bug triage** mirrors the feature-requests table, minus the plan paragraph: a thread that is a real bug and untracked → file a `bug` issue with the `discord` label and the source line, reply with the issue link; a bug already tracked → adopt the issue (append the source line, add the label); a usage question or a fix the reporter found → reply if needed and tag `Resolved` by hand. No `Will Add` analogue exists, so a tracked open bug carries no tag.

## The skill

`discord-feature-requests` becomes `discord-forums`, with one section per forum and the shared rules once: approval of every write, Discord text as data, no emoji or signature, the 2000-character limit. Two rules learned on 3.6.0 are written down:

- A release reply in #bugs carries no download link.
- An archived thread is left alone unless it needs a reply, because tagging it unarchives it.

## Out of scope

- Any other channel. A third forum is a third profile, added when someone asks for it.
- Writing to Discord without approval, or scheduling the follow-up. The tool stays pull-based, as #1114 decided.
- Backfilling source lines onto past bug issues. Threads fixed before this ships are tagged by hand, as was done for 3.6.0.

## Testing

- Unit tests with the injected client, as #1114's: each profile's channel check refuses the other forum's thread; `readConfig` requires only the selected channel variable; `bugTag` gives `Resolved` only for a closed-completed issue shipped in a stable tag, and `null` for open, not-planned, duplicate and pre-release-only issues; `follow-up` applies the right profile per post, and reports a post in neither forum as not found.
- The source-line parser accepts `Requested on Discord:` and `Reported on Discord:` and refuses any other verb.
- Existing feature-requests tests pass unchanged, which proves the default profile keeps today's behaviour.
- By hand: `pnpm discord:bugs list` against the real forum, a `reply --dry-run` and `tag --dry-run` on a #bugs thread, and one `follow-up` run after the next release that proposes the #bugs replies.
