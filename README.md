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
├── LICENSE
└── README.md
```

## Install

### From a private git repo (recommended)

Push this folder to your private repo, then install it in a Claude Code session.

**Why two commands?** Claude Code installs plugins from a *marketplace* — a catalog
that lists one or more plugins — not from a bare repo. This repo already *is* a
one-plugin marketplace (that is what `.claude-plugin/marketplace.json` declares). So
you (1) register the catalog, then (2) install the plugin from it:

```
/plugin marketplace add <owner>/<repo>
/plugin install technical-publisher@technical-publisher-marketplace
```

- Replace `<owner>/<repo>` with YOUR repo — e.g. `ansraju/technical-publisher-plugin`
  for GitHub, or a full URL for any host:
  `/plugin marketplace add https://github.com/ansraju/technical-publisher-plugin.git`
- The second command is fixed — do NOT change it. `technical-publisher` is the plugin
  name and `technical-publisher-marketplace` is the catalog name; both come straight
  from the JSON manifests in this repo. The `plugin@marketplace` form means "install
  plugin `technical-publisher` from the catalog you just added."

For a **private** repo, authenticate first (`gh auth login` for GitHub, or use an SSH
remote like `git@github.com:ansraju/technical-publisher-plugin.git`) so Claude Code
can clone it.

After installing, run `/plugin` to confirm `technical-publisher` is listed and
enabled, then **restart Claude Code** so it loads the five subagents. To pick up later
updates you push to the repo, run `/plugin marketplace update technical-publisher-marketplace`.

### From a local folder (for testing before you push)

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

## License

MIT — see `LICENSE`.
