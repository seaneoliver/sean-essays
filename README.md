# Sean Oliver: Essays

Welcome. This is the agent-readable archive of Sean Oliver's published essays.
Clone this repo, open it in your IDE, and any AI coding agent that reads
`CLAUDE.md` or `AGENTS.md` (Claude Code, Cursor, Codex, etc.) becomes a tutor on
this body of work. It answers your questions, cites the specific essay and
section it's drawing from, and helps you learn at your own pace.

This is also a way to access the writing where you actually work, so the ideas
can be useful even when Sean is not in the room.

## What's in here

**Substack — Sean's Newsletter** (https://seanoliver.substack.com/): essays on
decision-making with data, AI collaboration, career visibility, meetings,
writing, and how individual contributors stay legible inside large
organizations.

**Typeshare — Atomic essays** (https://typeshare.co/seanoliver): shorter
single-idea essays. Ingestion in progress.

The full article index, topic lookup, and per-essay section maps live in
[`CLAUDE.md`](./CLAUDE.md) at the root of this repo.

## Getting started

### With Claude Code (recommended)

1. Clone this repo:

   ```
   git clone https://github.com/seanoliver/sean-essays.git
   ```

2. Open the folder in your IDE (Cursor, VS Code, or anything that hosts a
   terminal).

3. Start Claude Code:

   ```
   claude
   ```

4. Ask your first question:

   > "I'm new here. What should I read first?"

Claude reads the `CLAUDE.md` files in this repo and uses them as a teaching
index. It knows every essay, every section, and exactly where to point you.

### With Cursor, Codex, or other AI tools

These tools read `AGENTS.md`, which mirrors `CLAUDE.md`. Clone the repo, open it
in your tool, and ask away.

### Without any AI tool

Read the essays directly. Each lives at `<series>/<NN>-<slug>/article.md`. The
section maps in each folder's `CLAUDE.md` make it easy to skim.

## Repository layout

```
sean-essays/
├── CLAUDE.md              # instructor persona, article index, topic lookup
├── AGENTS.md              # mirror of CLAUDE.md for non-Claude tools
├── README.md              # you are here
├── CONTRIBUTORS.md        # bios + canonical author URLs
├── LICENSE                # CC BY-NC 4.0
├── .claude/               # frontmatter schema, settings, ingest skill
├── substack/              # Substack essays, numbered chronologically
│   └── NN-slug/
│       ├── article.md
│       ├── CLAUDE.md      # section-level navigation map
│       └── images/
└── typeshare/             # Typeshare atomic essays, numbered chronologically
    └── NN-slug/
        ├── article.md
        ├── CLAUDE.md
        └── images/
```

## License

CC BY-NC 4.0. See [LICENSE](LICENSE). Per-article overrides, when they exist,
live in the article's frontmatter.

## Subscribe

New essays land at https://seanoliver.substack.com/. This repo updates after
each post via the `add-essay-to-repo` skill in [`.claude/skills/`](./.claude/skills/).
