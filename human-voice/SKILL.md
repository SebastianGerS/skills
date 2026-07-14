---
name: human-voice
description: Use when drafting or composing any external-facing message or document for a person to read — a Slack message, an email, a Linear issue/ticket/comment, or a Notion page/doc — before writing the draft. Triggers whenever output is meant to be read by colleagues rather than run as code or shown as internal reasoning.
---

# human-voice

Drafts meant for people should read like the user dashed them off, not like an assistant produced them. The first tell to kill is **length**: say less.

## Do this, in order

**1. Sample the user's real voice first.** Before drafting, pull a few of their own recent messages on the target surface and copy how they actually write — length, sentence shape, capitalization, punctuation, whether they greet or sign off, whether they use bullets or emoji.
- Slack → `slack_search_public_and_private` with `from:<@U0314GC3YRY>` (add `in:#channel` when known); read ~5–8 recent.
- Linear → their recently authored issue descriptions and comments.
- Notion → `notion-search` (query_type internal) for pages they created; read 1–2.
- Email → no source available; mirror their Slack voice.

If sampling isn't possible, fall back to the contract below.

**2. Write to this contract:**
- First sentence carries the point or the ask. No warm-up.
- Length tracks their samples; when unsure, go shorter, then cut ~30% more.
- **Shorter ≠ telegraphic.** Cut whole ideas and warm-up, not the grammar. Write complete plain sentences, not compressed fragments. A clean sentence that's slightly longer beats a clipped one studded with assistant tells.
- No em-dashes. Also avoid the compression shorthands that read as machine output: `→` and `/` between values, `+` for "and", `X→Y` for ranges, stacked parentheticals. Spell them out — "och", "från X till Y", a second sentence.
- Their register: their casing, their punctuation, their sign-off habit (usually none on Slack).
- Prose by default. Use a list only when there are 3+ genuinely parallel items AND their samples use lists. Bold at most one key term, and only if they do.
- One idea per short paragraph.

**3. Per surface** (budgets are the default; if their samples run longer, match the samples):
- *Slack* — no greeting or sign-off, lead with the point. Budget: 4 sentences.
- *Email* — greeting/sign-off only if their samples have them; the subject line states the point. Budget: 6 sentences of body.
- *Linear* — title `type(scope): short description NEX-ID`; body is 1–3 plain paragraphs (what's broken, why it matters, what to do). Headed sections only for a genuinely large ticket.
- *Notion* — headings are fine; keep the body prose, not bullet scaffolding.

**4. Read it back, then cut once.** Would a busy colleague assume a human wrote this in a minute? If it has a tidy assistant shape they wouldn't use (intro → three bullets → wrap-up, a closing "let me know if…", a recap of what you just said), flatten it. Then do one deliberate cut pass: delete every sentence the reader doesn't need in order to act or get the point — background, restated context, and hedges go first. If the draft is over the surface budget, cut whole ideas until it fits, not words.

## Examples

The trap is compression that reads as machine output. Both versions below are short; the rewrite is even a touch longer, but it's clean sentences instead of clipped fragments.

> ✗ Aikido-fixar: NEX-1709 + NEX-1712 (openssl/libssl3t64 på tv4-metadata-repository) — en PR på metadata-api-varnish (varnish base 8.0.1→8.0.2) täckte båda, godkänd av Erik och merge:ad

> ✓ Aikido-fixar: NEX-1709 och NEX-1712 (openssl/libssl3t64 på tv4-metadata-repository). En PR på metadata-api-varnish där Varnish-basen uppdaterades från 8.0.1 till 8.0.2 täckte båda. PR:en godkändes av Erik och är nu mergad.

What changed: `+` → "och", `8.0.1→8.0.2` → "från 8.0.1 till 8.0.2", the em-dash splice became two sentences, "merge:ad" became "är nu mergad". One idea per sentence, each ending in a period.

> ✗ Mercury/Comet: flaggade att wonProductCode inte alltid innehåller ett WON-id (mer ett generiskt external reference) — påverkar Comet-migrationen efter sommaren

> ✓ Mercury/Comet: Flaggade att wonProductCode inte alltid innehåller ett WON-id, utan ibland verkar vara mer av en generisk extern referens. Det påverkar antagandet inför Comet-migrationen efter sommaren.

The parenthetical aside and the em-dash both became their own clauses or sentences; the bullet opens with a capital.

## When not to use
Code, commit messages, internal reasoning, or when the user explicitly asks for a formal or long-form document.
