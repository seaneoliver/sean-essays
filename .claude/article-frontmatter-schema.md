# Article Frontmatter Schema

Canonical YAML frontmatter for every `article.md` in this repo. Every article
must use this shape exactly. The `add-essay-to-repo` skill emits this shape
automatically; this doc is the source of truth when the skill or a human is
unsure.

## Canonical block

```yaml
---
title: "<Title>"
subtitle: "<Subtitle>"
authors:
  - name: "Sean Oliver"
    publication:
      name: "Sean's Newsletter"
      url: "https://seanoliver.substack.com/"
  - name: "<Co-author>"
    publication:
      name: "<Their Pub>"
      url: "<https://...>"
published: YYYY-MM-DD
series: "Substack"          # OR "Typeshare". OMIT for any future standalones.
series_number: 12           # zero-padded sequence within the series. OMIT for standalones.
original_url: "https://..."  # canonical Substack or Typeshare URL of the published post
license: "CC BY-NC 4.0"
media:                       # OMIT unless real podcast/video companion exists
  spotify: "https://..."
  apple:   "https://..."
  youtube: "https://..."
---
```

## Field rules

- **`title`**: full essay title as published. Smart quotes are fine in titles
  (preserve what Substack/Typeshare published).
- **`subtitle`**: one line. No surrounding asterisks or italics. Markdown
  belongs in the body, not the frontmatter.
- **`authors`**: ordered list. First entry is the lead author (the person
  whose publication owns the original post). Each entry has `name` and
  `publication`.
- **`authors[].publication`**: object with `name` (display label) and `url`
  (publication homepage). Use the canonical URL per author and publication
  pair (see table below). Don't add `links:`, `role:`, or any other sub-keys.
- **`published`**: ISO date `YYYY-MM-DD`. The date the post went live.
- **`series`**: string. Currently `"Substack"` or `"Typeshare"`. Omit for any
  future standalones. Don't write `series: null`.
- **`series_number`**: integer. Same rule as `series`: omit for standalones.
- **`original_url`**: the canonical URL of the post. For posts whose original
  source has been deleted, use empty string `""`.
- **`license`**: always `"CC BY-NC 4.0"` unless explicitly overridden.
- **`media`**: optional object. Only include when the article has a real
  companion podcast/video. Flat keys: `spotify`, `apple`, `youtube`. Omit any
  platform that doesn't have a published companion. Omit the entire `media:`
  block if no media exists.

## Canonical author → publication URL mapping

One canonical URL per (author, publication) pair. If the same author writes for
a different publication, use that publication's URL there.

| Author | Publication | URL |
|---|---|---|
| Sean Oliver | Sean's Newsletter | https://seanoliver.substack.com/ |
| Sean Oliver | Typeshare | https://typeshare.co/seanoliver |

Add new rows when co-authored essays are ingested.

## Why no `links` block, no `role` field

Earlier patterns (carried over from the reference In the Weeds repo) embedded
`authors[].links.{substack,linkedin,youtube,github,website}` and
`authors[].role`. Both stay omitted here for the same reasons:

- **`links` is dead metadata.** Bio links live in
  [`CONTRIBUTORS.md`](../CONTRIBUTORS.md), the actual source of truth.
  Duplicating them in every article frontmatter creates drift risk.
- **`role` strings are inconsistent and unused.** The order of `authors[]`
  already conveys lead vs. co-author.

If a future need surfaces (e.g., generating an authors page), reach for
`CONTRIBUTORS.md` first. Don't reintroduce `links` here.

## Voice rule (overrides anything else)

**Never use em dashes** in titles, subtitles, body text, or any field. Use
commas, colons, periods, or " - ". The ingest skill enforces this on copy.

## Who writes this

The [`add-essay-to-repo`](skills/add-essay-to-repo/SKILL.md) skill emits this
shape automatically when mirroring a Substack or Typeshare post into the repo
(Stage 4 sanitize). When that skill changes, update this doc and the skill in
the same commit.
