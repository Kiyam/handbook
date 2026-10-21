# Skills

[Skills](https://docs.anthropic.com/en/docs/claude-code/skills) are folders with a `SKILL.md` that Claude loads when a task matches its description. They live in `~/.claude/skills/`, one folder per skill.

## My own

Written for research work:

| Skill | What it does |
|---|---|
| `paper-summariser` | Turns an arXiv paper or PDF into a structured note: method, assumptions, contribution, relevance. It appraises the paper rather than just paraphrasing it. |
| `arxiv-rag` | Reads a `.bib` file, downloads the LaTeX source of every entry with an arXiv ID, and builds a corpus ready for RAG. Python scripts sit next to the `SKILL.md`. |
| `jax-review` | Reviews and writes idiomatic JAX: dtypes, PRNG keys, `jit`/`vmap`/`scan`, NaN gradients, recompiles, GPU memory. |

They live in my [dotfiles](../terminal/terminal-setup.md#chezmoi), so `chezmoi update` puts them on every machine where I run Claude Code. Skill scripts declare their Python dependencies inline ([PEP 723](https://peps.python.org/pep-0723/)) and run with `uv run`, so nothing has to be installed globally on each machine.

## From the skills CLI

Installed with the [skills CLI](https://github.com/vercel-labs/skills), which puts them in `~/.agents/skills/` and symlinks them into each agent's skills folder, so other coding agents can use them too:

```bash
npx skills add vercel-labs/skills      # find-skills: search for and install more skills
npx skills add ogulcancelik/herdr      # herdr: lets Claude drive Herdr panes and agents
```

## Hunk

[Hunk](../terminal/hunk.md) ships two skills of its own in `~/.hunk/skills/`. They're updated with Hunk itself (`hunk update`), so I symlink them rather than copying them:

| Skill | What it does |
|---|---|
| `hunk-review` | Drives a live Hunk window through `hunk session …`: reads where you are, jumps to a file, hunk or line, leaves inline review notes and highlights exact character ranges. |
| `hunk-extensions` | A map of Hunk's TypeScript extension API, for when you want Claude to write an extension (a command, pane, theme or key binding). |

```bash
ln -s ~/.hunk/skills/hunk-review ~/.claude/skills/hunk-review
ln -s ~/.hunk/skills/hunk-extensions ~/.claude/skills/hunk-extensions   # optional
```

A one-line `/hunk` command makes the review skill quick to start:

```markdown title="~/.claude/commands/hunk.md"
---
description: Walk through the changes in the open Hunk session using the hunk-review skill
---

Use the hunk-review skill to walk me through the changes in my open Hunk session. $ARGUMENTS
```

With `hunk diff --watch` open in another pane, `/hunk` (or `/hunk focus on the error handling`) has Claude walk through its own changes inside that window. The skill controls an open Hunk window but never starts one, because the viewer is interactive. Open Hunk first. The full workflow is on [Hunk → Reviewing Claude's changes](../terminal/hunk.md#reviewing-claudes-changes).

The symlink and the command are in my dotfiles too. chezmoi stores the skill as a `symlink_` entry, so on a new machine install Hunk *before* running `chezmoi update`, or the link points at nothing.

## Writing good skills

The description in a skill's frontmatter decides when it gets used. Make it list the actual phrases you'd type ("my gradient is NaN", "TL;DR of this PDF"), not just a summary of the topic.
