# Technical Writer Orchestrator (Claude Code plugin)

A Claude Code plugin that packages a **technical-writing agent team** as one
installable unit. An orchestrator coordinates a writer, up to six parallel review
axes, and an editor in a bounded **generate → review → revise → edit** loop that
produces a single polished technical draft.

This is the Claude Code **plugin** equivalent of the AIM
`technical-writer-orchestrator` agent-spec set. The procedure (the SOP content) is
embedded directly in each subagent's system prompt, so the plugin is fully
self-contained — no external MCP SOP server is required.

## What it adds

Four subagents (`agents/*.md`), loaded as subagents Claude can delegate to:

| Subagent | Role | Writes |
|---|---|---|
| `technical-writer-orchestrator` | Owns the brief, style spec, and loop control; fans out review; decides when to stop. Delegates to the three below. | run log, merged worklist |
| `technical-writer` | Writes/revises the draft. Enforces ASD-STE100. No self-review. | `draft.md` |
| `technical-reviewer` | Judges ONE axis; judgment-only, never edits the draft. One invocation per axis, run in parallel. | `findings/<axis>.json` |
| `technical-editor` | Final line-level polish, runs once. | `final.md` |

Review axes (default six): `technical-accuracy`, `continuity`, `voice-consistency`,
`narrative-pacing`, `inclusive-language`, `terminology-consistency`.

## Layout

```
technical-writer-orchestrator-plugin/
├── .claude-plugin/
│   ├── plugin.json          # plugin manifest (name, version, agents)
│   └── marketplace.json     # marketplace entry (install this repo as a marketplace)
├── agents/
│   ├── technical-writer-orchestrator.md
│   ├── technical-writer.md
│   ├── technical-reviewer.md
│   └── technical-editor.md
├── LICENSE
└── README.md
```

## Install

### From a private git repo (recommended)

Push this folder to your private repo, then in a Claude Code session:

```
/plugin marketplace add <owner>/<repo>
/plugin install technical-writer-orchestrator@technical-writer-orchestrator-marketplace
```

For a private GitHub repo, authenticate first (`gh auth login`, or an SSH remote).
You can also point at an explicit git URL:

```
/plugin marketplace add https://github.com/<owner>/<repo>.git
```

### From a local folder (for testing before you push)

```
claude --plugin-dir /local/home/ansraju/.kiro/crew/workspace/technical-writer-orchestrator-plugin
```

Validate the manifest at any time:

```
claude plugin validate /local/home/ansraju/.kiro/crew/workspace/technical-writer-orchestrator-plugin
```

## Use

You do not "launch into" the orchestrator — Claude Code subagents are delegated to.
Start a normal session and ask Claude to use the orchestrator:

```
Use the technical-writer-orchestrator agent to run the writing loop.
Brief is at .agents/brief.md, style spec at .agents/style.md, max_iterations 3.
```

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
`.agents/draft/final.md`.

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
