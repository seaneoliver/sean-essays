---
name: add-essay-to-repo
description: Mirror a Substack or Typeshare essay into the sean-essays repo at `~/Projects/sean-essays/`. Two source modes - firecrawl (URL) and local (path or slug pointing at a markdown file). Copies article + images, generates the article-level CLAUDE.md, cascades index updates across root CLAUDE.md, AGENTS.md, and the series-level CLAUDE.md, then opens a clean PR. Use whenever Sean says "add this essay to sean-essays", "ingest [URL]", "mirror [URL] into the essays repo", or pastes a Substack URL (https://seanoliver.substack.com/p/...) or Typeshare URL (https://typeshare.co/seanoliver/...) and asks to bring it into the repo.
---

# Add Essay To Repo

End-to-end skill for getting a Substack or Typeshare essay into
`~/Projects/sean-essays/`. Covers source discovery through PR.

## When the skill triggers

Sean just published an essay (or wants to ingest a backlog item) and needs to
mirror it into the repo so the AI tutor can teach from it. Two patterns:

- He pastes a URL: `add this essay: https://seanoliver.substack.com/p/teach-your-edge`
  → **firecrawl mode**.
- He passes a local path or slug:
  `add the local file ~/vaults/SOS/essays/Recents/foo.md`
  → **local mode**.

## Mode selection (Stage 0)

Inspect the input:

- Matches `^https?://` → **firecrawl mode**. Derive slug from the URL:
  - Substack: last `/p/<slug>` segment.
  - Typeshare: last `/seanoliver/<slug>` segment.
- Otherwise → **local mode**. Treat input as either an absolute/relative path
  to a `.md` file, or a slug to look up under `~/vaults/SOS/essays/Recents/`.

Sean can override: "use firecrawl on <url>" forces firecrawl, "use local
<path>" forces local.

Determine series from the URL host (or ask Sean if local mode and ambiguous):

- `seanoliver.substack.com` → `series: "Substack"`, target folder `substack/`
- `typeshare.co` → `series: "Typeshare"`, target folder `typeshare/`

Write `/tmp/add-essay-<slug>.json`:

```json
{"state": "mode_selected", "mode": "firecrawl|local", "input": "<input>", "slug": "<slug>", "series": "Substack|Typeshare"}
```

## Stage 1: Source fetch

### firecrawl mode

Use the Firecrawl MCP (`mcp__firecrawl__firecrawl_scrape`) to fetch the URL as
markdown. If Firecrawl is unavailable or the page returns thin content, fall
back to `mcp__supadata__supadata_scrape`. Save raw output to
`/tmp/add-essay-<slug>/raw.md`.

For Substack posts, also fetch the JSON metadata for canonical title, subtitle,
and `post_id` (used to construct anchor URLs):

```bash
curl -s "https://seanoliver.substack.com/api/v1/posts/<slug>" \
  | python3 -c "import json,sys; d=json.load(sys.stdin); print(json.dumps({k:d.get(k) for k in ['id','title','subtitle','post_date','canonical_url']}, indent=2))" \
  > /tmp/add-essay-<slug>/meta.json
```

For Typeshare, scrape the page and parse title from the `<h1>`, date from the
visible byline, and use `original_url` as the canonical link. There is no JSON
API; record `post_id: null`.

### local mode

```bash
cp <input-path> /tmp/add-essay-<slug>/raw.md
```

Read the file and infer title from the first `^# ` heading and `published`
from any frontmatter or filename date prefix.

Update temp file:
`{"state": "source_fetched", "title": "...", "subtitle": "...", "published": "YYYY-MM-DD", "post_id": <id|null>, "canonical_url": "..."}`.

## Stage 2: Sanitize

Clean the raw markdown:

1. Strip Substack chrome: subscribe widgets, "Share" buttons, "Like this post"
   blocks, footer subscribe CTAs, paywall markers, comment threads.
2. **Replace every em dash (`—`, U+2014) with " - " (space-hyphen-space).**
   Also replace en dashes (`–`, U+2013) inside prose with the same. Sean's
   voice rule. Numeric ranges like "9-5" stay as hyphens.
3. Convert smart quotes to straight quotes only inside YAML frontmatter
   strings; preserve smart quotes in body prose.
4. Image refs: collect every `![alt](url)` with an http(s) URL into a download
   list. Defer download to Stage 4.
5. Strip Substack image proxy wrappers (`https://substackcdn.com/image/fetch/...`)
   and decode the inner URL.
6. Drop any HTML comments and editor-only artifacts.

Write `/tmp/add-essay-<slug>/sanitized.md` and
`/tmp/add-essay-<slug>/images.json` (list of `{src_url, target_filename}`
pairs). Pick filenames as the slugified original filename, fallback to
`image-NN.<ext>`.

Update temp file: `{"state": "sanitized", "image_count": N}`.

## Stage 3: Determine target paths

Read the live repo. Source of truth is the series folder structure.

Compute next NN within the series:

```bash
cd ~/Projects/sean-essays/<series-folder>
ls -d [0-9][0-9]-* 2>/dev/null | sed 's/-.*//' | sort -n | tail -1
# +1, zero-padded to 2 digits
```

If the series folder is empty (no `NN-` folders yet), start at `01`. If a
folder for this slug already exists at any NN, abort with: "Already ingested
at <path>. Use `update-essay-in-repo` instead."

Compute:

- `target_folder = <series-folder>/<NN>-<slug>/`
- `target_article = <target_folder>article.md`
- `target_claude_md = <target_folder>CLAUDE.md`
- `target_images = <target_folder>images/`

## Stage 4: Branch + copy + image download

Sequential. Branch must exist before any writes.

```bash
cd ~/Projects/sean-essays
git pull origin main 2>/dev/null || true   # tolerate first-commit case
git checkout -b add/<slug>
mkdir -p <target_folder>/images
```

Build the final `article.md`:

1. Prepend the YAML frontmatter block (see `.claude/article-frontmatter-schema.md`).
   Required fields: `title`, `subtitle`, `authors[]`, `published`, `series`,
   `series_number`, `original_url`, `license`. Omit `media:` unless real media
   exists.
2. Append the sanitized body.
3. Rewrite every image `![alt](src_url)` to `![alt](images/<target_filename>)`.

Download every image:

```bash
for img in <images.json entries>; do
  curl -sL "$src_url" -o "<target_images>/$target_filename"
done
```

If any download returns non-2xx, log the failure but proceed; the warning
surfaces in Stage 7.

Update temp file:
`{"state": "files_copied", "branch": "add/<slug>", "target_folder": "..."}`.

## Stage 5: Generate article-level CLAUDE.md

Format MUST match the existing repo pattern. Reference example once one
exists; until then, follow this spec.

Required structure:

```markdown
# <Article Title>

**Author:** Sean Oliver | **Published:** YYYY-MM-DD

## Article Map

| Section | Line | Coverage | Source |
|---------|------|----------|--------|
| Title & subtitle | <line> | <one-phrase coverage> | [Link](<original_url>) |
| <heading text> | <line> | <one-phrase coverage> | [Link](<anchor URL>) |
| ... | ... | ... | ... |
```

Generation algorithm:

1. Walk `article.md`. For every line matching `^#{1,4} ` (after the
   frontmatter), record `(line_number, heading_text)`.
2. For each heading, write a one-phrase coverage summary (10-20 words). Pull
   the first 1-2 sentences after the heading and compress.
3. Build the source URL:
   - **Substack:** `https://seanoliver.substack.com/i/<post_id>/<heading-slug>`
     where `<heading-slug>` is `heading_text.lower().replace(" ", "-")` with
     punctuation stripped. If `post_id` is null, fall back to `[Link](<original_url>)`.
   - **Typeshare:** Typeshare doesn't expose deep section anchors. Use `—` for
     every row except Title (which gets `[Link](<original_url>)`).
4. Sanitize the same way `article.md` is sanitized: no em dashes anywhere in
   the table.

Write to `<target_folder>/CLAUDE.md`.

## Stage 6: Cascade index updates

Three files get updated. All edits use Edit tool, not full rewrites.

### 6a. Series-level `<series-folder>/CLAUDE.md`

Append a row to the `## Articles` table:

```
| NN | YYYY-MM-DD | <Title> | `NN-slug/` | <key topics, comma-separated, 5-8 max> |
```

If the table header is the placeholder one (no rows yet), the row goes
immediately under the divider.

### 6b. Root `CLAUDE.md`

Append a row to the `## Article index` table with:

```
| NN | YYYY-MM-DD | <Title> | <Series> | <series-folder>/<NN-slug>/article.md | <key topics> |
```

For each key topic in the article, also add a row to the `## Topic quick lookup`
table:

```
| <Topic> | <Series> #<NN>: <Title> | <Section heading or "Full article"> |
```

Pick at most 5 topics per essay. Skip topics already well-covered in the table
unless this essay is the canonical reference.

### 6c. `AGENTS.md`

Re-copy from `CLAUDE.md`:

```bash
cp ~/Projects/sean-essays/CLAUDE.md ~/Projects/sean-essays/AGENTS.md
```

Update temp file: `{"state": "indexes_updated"}`.

## Stage 7: Verify

Run these checks. Surface failures in the PR body.

1. **No em dashes:** `grep -rn '—\|–' <target_folder>/` returns zero matches.
2. **All images present:** every `![alt](images/<f>)` ref in `article.md` has a
   corresponding file in `<target_folder>/images/`.
3. **Frontmatter parses:** `python3 -c "import yaml,sys; yaml.safe_load(open('<target_article>').read().split('---')[1])"`.
4. **Indexes updated:** grep for the new slug in `~/Projects/sean-essays/CLAUDE.md`
   and `<series-folder>/CLAUDE.md` returns at least one match each.

## Stage 8: Commit, push, PR

```bash
cd ~/Projects/sean-essays
git add <target_folder> CLAUDE.md AGENTS.md <series-folder>/CLAUDE.md
git commit -m "Add <Series> #<NN>: <Title>"
git push -u origin add/<slug>
gh pr create --title "Add <Series> #<NN>: <Title>" --body "$(cat <<'EOF'
## Summary

- Ingests <Series> #<NN>: <Title> from <original_url>
- Adds article + section CLAUDE.md + N image(s) under <target_folder>
- Updates root and series indexes

## Verification

- [ ] No em dashes in new files
- [ ] All N images downloaded
- [ ] Frontmatter parses
- [ ] Indexes show the new entry

EOF
)"
```

If `gh` is not authenticated or the remote doesn't exist yet, stop after the
commit and tell Sean to set up the remote.

Update temp file: `{"state": "done", "pr_url": "<url>"}`.

## Failure recovery

The temp file (`/tmp/add-essay-<slug>.json`) records the last completed stage.
On re-invocation with the same slug, resume from the next stage. Don't repeat
work.

Common failures:

- **Firecrawl thin content:** fall back to supadata. If both thin, drop into
  local mode and ask Sean to paste the article body.
- **Image download 403:** Substack sometimes hotlinks images via signed CDN
  URLs. Retry with `User-Agent: Mozilla/5.0`. If still failing, leave the
  remote URL in the body and note the broken refs in Stage 7 verify.
- **NN collision:** another ingest is running. Wait, then re-derive NN.

## Style enforcement (always-on)

- No em dashes anywhere in any output: titles, subtitles, body, table cells,
  PR body, commit messages.
- No title-stacking on Sean. Default to "Sean" or "Sean Oliver" in any
  generated prose.
- Match Sean's voice rules from the SOS vault when summarizing: short
  sentences, opinionated, no hedging.
