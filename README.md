# Technical Publisher (Claude Code plugin)

A Claude Code plugin that packages a **technical-writing agent team** as one
installable unit. An orchestrator coordinates a writer, up to six parallel review
axes, and an editor in a bounded **generate → review → revise → edit** loop that
produces a single polished technical draft, with an **optional export step** that
renders a `.docx`, `.pdf`, or `.html` deliverable via pandoc.

This is the Claude Code **plugin** equivalent of the AIM
`technical-writer-orchestrator` agent-spec set. The procedure (the SOP content) is
embedded directly in each subagent's system prompt, so the plugin is fully
self-contained — no external MCP SOP server is required.

## What it adds

Five subagents (`agents/*.md`), loaded as subagents Claude can delegate to:

| Subagent | Role | Writes |
|---|---|---|
| `technical-publisher` | Owns the brief, style spec, and loop control; fans out review; decides when to stop. Delegates to the four below. | run log, merged worklist |
| `technical-writer` | Writes/revises the draft. Enforces ASD-STE100. No self-review. | `draft.md` |
| `technical-reviewer` | Judges ONE axis; judgment-only, never edits the draft. One invocation per axis, run in parallel. | `findings/<axis>.json` |
| `technical-editor` | Final line-level polish, runs once. | `final.md` |
| `technical-publisher-export` | Optional. Converts `final.md` to `.docx`/`.pdf`/`.html` via pandoc. Format conversion only — never edits content. | `final.docx` etc. |

Review axes (default six): `technical-accuracy`, `continuity`, `voice-consistency`,
`narrative-pacing`, `inclusive-language`, `terminology-consistency`.

## Layout

```
technical-publisher-plugin/
├── .claude-plugin/
│   ├── plugin.json          # plugin manifest (name, version, agents)
│   └── marketplace.json     # marketplace entry (install this repo as a marketplace)
├── agents/
│   ├── technical-publisher.md
│   ├── technical-writer.md
│   ├── technical-reviewer.md
│   ├── technical-editor.md
│   └── technical-publisher-export.md
├── CONTRIBUTING.md
├── LICENSE
└── README.md
```

## Install

### To distribute it (install from a git repo)

**1. Push this folder to a git repo.**

```bash
git remote add origin git@github.com:rajuans/technical-publisher-plugin.git
git push -u origin master
```

**2. In a Claude Code session, run these two commands** — replace only the repo on the
first line; copy the second line exactly:

```
/plugin marketplace add rajuans/technical-publisher-plugin
/plugin install technical-publisher@technical-publisher-marketplace
```

Use `owner/repo` for GitHub, or a full URL for any host (e.g.
`/plugin marketplace add https://github.com/rajuans/technical-publisher-plugin.git`).

**3. Restart Claude Code** so it loads the five subagents. Run `/plugin` to confirm
`technical-publisher` is listed and enabled.

To pull in later updates you push to the repo, run
`/plugin marketplace update technical-publisher-marketplace`.

> Why two commands? Claude Code installs plugins from a *marketplace* (a plugin
> catalog), not from a bare repo — so the first command registers this repo as a
> catalog and the second installs the plugin from it. The names in the second command
> (`technical-publisher` and `technical-publisher-marketplace`) are fixed by this
> repo's JSON manifests, which is why you copy it as-is.

### To use it locally (no repo)

```
claude --plugin-dir /local/home/ansraju/.kiro/crew/workspace/technical-publisher-plugin
```

Validate the manifest at any time:

```
claude plugin validate /local/home/ansraju/.kiro/crew/workspace/technical-publisher-plugin
```

## Use

You do not "launch into" the orchestrator — Claude Code subagents are delegated to.
Start a normal session and ask Claude to use the orchestrator:

```
Use the technical-publisher agent to run the writing loop.
Brief is at .agents/brief.md, style spec at .agents/style.md, max_iterations 3.
Export the final result as docx.
```

Drop the last line to keep the deliverable as Markdown only.

Set up the inputs first in the directory you run from:

```
.agents/
├── brief.md        # REQUIRED — topic, audience, scope, outline, length target
├── style.md        # optional — voice/tone/POV/tense/forbidden words
├── glossary.md     # optional — term base (enables terminology-consistency axis)
└── inclusive.md    # optional — non-inclusive→preferred term map
```

The orchestrator then: writes `.agents/draft/draft.md`, fans out one reviewer per
axis (each writing `.agents/review/findings/<axis>.json`), merges them into
`.agents/review/worklist.json`, has the writer revise against the worklist, loops
until clean or `max_iterations`, then runs the editor once to produce
`.agents/draft/final.md`. When you ask for a binary format, it then runs the export
step to render `.agents/draft/final.docx` (and/or `.pdf`/`.html`).

## Binary output (.docx / .pdf / .html)

The pipeline is Markdown end to end — reviewable, diffable, git-friendly. Binary
deliverables are produced by the `technical-publisher-export` subagent as a final
conversion step; the content is never changed, only rendered.

**Prerequisite: pandoc.** The export step shells out to [pandoc](https://pandoc.org),
which must be on `PATH`. Install it once:

```bash
sudo yum install -y pandoc        # Amazon Linux / RHEL
sudo apt-get install -y pandoc    # Debian / Ubuntu
brew install pandoc               # macOS
```

`docx` and `html` need only pandoc. `pdf` additionally needs a LaTeX engine
(`sudo yum install -y texlive`, or `brew install --cask basictex`). If pandoc is
missing, the export agent stops and prints the install command rather than failing
the run — the Markdown deliverable is still produced.

**Request it** by naming the format(s) in your delegation prompt ("export as docx",
"also give me a pdf and html"). The orchestrator passes `output_formats` to the
export agent. To match a corporate Word template, point it at a reference doc: *"export
as docx using .agents/template.docx as the reference doc"* (pandoc copies that file's
styles).

**Convert manually** at any time without the agent:

```bash
pandoc .agents/draft/final.md --from gfm --to docx -o final.docx
```

## Design invariants

- **The orchestrator delegates, never writes or judges.** It owns the loop only.
- **Reviewers are judgment-only and context-isolated.** Each sees only the draft and
  its own spec — never the writer's reasoning — so it cannot rubber-stamp. The
  prompt forbids editing the draft; a reviewer writes only its findings file.
- **The writer revises, it does not regenerate.** Preserving unflagged prose is what
  lets the loop converge.
- **Non-convergence escalates.** If blocking+major counts stop decreasing, the
  orchestrator stops and asks the human rather than looping to the cap.

## Model note

Subagent `.md` files here do **not** pin a `model`, so each inherits the main
Claude Code session's model. If you want a specific model per role (e.g. a cheaper
model for reviewers), add a `model:` line to that agent's frontmatter.

## Contributing

Want to add a review axis, a new subagent, or an export format? See
[`CONTRIBUTING.md`](CONTRIBUTING.md) for the structure, how to edit a subagent safely,
and how to validate before you push.

## License

MIT — see `LICENSE`.
