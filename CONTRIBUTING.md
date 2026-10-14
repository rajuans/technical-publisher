# Contributing to Technical Publisher

Thanks for improving the plugin. This guide covers how it is structured, how to make a
change safely, and how to validate it before you push.

## How the plugin is put together

```
technical-publisher-plugin/
├── .claude-plugin/
│   ├── plugin.json          # manifest: name, version, and the agents[] list
│   └── marketplace.json     # one-plugin marketplace entry for install
├── agents/                  # one markdown file per subagent (the actual behavior)
│   ├── technical-publisher.md        # orchestrator — owns the loop, delegates
│   ├── technical-writer.md
│   ├── technical-reviewer.md
│   ├── technical-editor.md
│   └── technical-publisher-export.md # pandoc export
├── CONTRIBUTING.md
├── LICENSE
└── README.md
```

Each subagent is a markdown file: **YAML frontmatter** (`name`, `description`,
`tools`, optional `model`) followed by the **system prompt** in the body. There is no
code and no external SOP server — the whole procedure lives in the prompt bodies, so
the plugin is self-contained.

Two invariants hold the design together; keep them when you edit:

- **The orchestrator delegates; it never writes or judges.** Loop control only.
- **Reviewers are judgment-only and context-isolated.** A reviewer writes only its
  findings file, never the draft, and sees only the draft plus its own axis spec.

## Local setup

```bash
git clone <your-fork-or-repo-url>
cd technical-publisher-plugin
```

You need the `claude` CLI on `PATH` to validate and test (`claude plugin validate`,
`claude --plugin-dir`). The export agent needs `pandoc` on `PATH` only if you test
binary output.

## Making a change

### Edit an existing subagent
Change the body (the prompt) or the frontmatter in `agents/<name>.md`. If you change a
subagent's `name`, update it in **three** places or install breaks:
1. the frontmatter `name:`,
2. the file name (`git mv`),
3. the `agents[]` path in `.claude-plugin/plugin.json`.

### Add a new subagent
1. Create `agents/<new-name>.md` with frontmatter and a prompt body. Copy an existing
   file as the template.
2. Grant the **minimum** tools in `tools:`. A judgment/read role should not have
   `Write` or `Edit`; a conversion role should not have `Edit`.
3. Add `"./agents/<new-name>.md"` to the `agents[]` array in `plugin.json`.
4. If the orchestrator should delegate to it, describe when in the orchestrator's body
   and reference it by `name`.

### Add or change a review axis
Axes are not separate files — `technical-reviewer.md` is axis-parameterized and the
orchestrator fans one invocation out per axis. To add an axis, document it in the
reviewer body and add it to the axis list the orchestrator fans out over (in
`technical-publisher.md`) and to the README's axis list.

### Bump the version
Update `version` in `plugin.json` using semantic versioning (MAJOR for a breaking
change to agent names/behavior, MINOR for a new agent or axis, PATCH for prompt fixes).

## Validate before you push

Both JSON manifests must parse and the plugin must validate clean:

```bash
# manifests parse
python3 -m json.tool .claude-plugin/plugin.json  >/dev/null && echo "plugin.json OK"
python3 -m json.tool .claude-plugin/marketplace.json >/dev/null && echo "marketplace.json OK"

# plugin validates
claude plugin validate .

# smoke-test locally without installing
claude --plugin-dir "$PWD"
```

A real behavior test: set up a tiny `.agents/brief.md`, delegate to the
`technical-publisher` agent, and confirm the loop runs end to end and (if you touched
the export path) that `final.docx` lands on disk.

## Commit and PR conventions

- One logical change per commit; present-tense imperative subject (`Add pdf-engine
  option to export agent`).
- State in the body **what changed and why**, and note if you bumped the version.
- Keep run artifacts out of git — `.agents/` is git-ignored; do not commit drafts,
  findings, or generated `.docx`/`.pdf` files.
- Open a PR describing the change, how you validated it, and any new prerequisite
  (e.g. a pandoc flag or a new tool grant).

## Style

Write prompt bodies the way the plugin asks its own writer to: short sentences, active
voice, one instruction per sentence. State each rule as a MUST/NEVER where behavior
depends on it, and say *why* a constraint exists — the prompts are the product.
