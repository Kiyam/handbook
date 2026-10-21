# Hunk

[Hunk](https://hunk.dev) is a terminal diff viewer built for reviewing changes, especially ones an agent like Claude Code just made. The whole changeset reads as one stream, top to bottom, with a file list on the side. It has real syntax highlighting, split or unified layout, inline notes, and a live-session API that lets Claude drive the review while you watch.

## Install

```bash
curl -fsSL https://hunk.dev/install.sh | sh
```

The installer puts a single binary in `~/.hunk/bin/hunk`. Hunk's bundled skills go in `~/.hunk/skills/`. It's also on Homebrew (`brew install hunk`), npm and mise. Add it to the `PATH`, which in my setup lives in the [chezmoi](terminal-setup.md#chezmoi)-managed `.zshrc`:

```bash
export PATH=$HOME/.hunk/bin:$PATH
```

`hunk update` updates it with whichever method installed it.

## Everyday use

```bash
hunk diff                  # working tree changes (including untracked files)
hunk diff --staged         # what you're about to commit
hunk diff main             # this branch against main
hunk diff HEAD~3 HEAD      # two revisions
hunk show                  # the last commit
hunk show <sha>            # a specific commit
hunk log                   # browse history
hunk diff --watch          # reload automatically as files change, e.g. while Claude is editing
hunk diff --files a.py b.py
```

`hunk status` lists the changes in this checkout and any sibling git worktrees.

## Keys

Press `?` inside Hunk for the full list. The ones to learn first:

| Keys | Action |
|---|---|
| `j` / `k`, `↑` / `↓` | Move the current line |
| `]` / `[` | Next / previous hunk |
| `.` / `,` | Next / previous file |
| `Space` / `b`, `d` / `u` | Page down / up, half page down / up |
| `g` / `G` | Start / end of the review |
| `/`, then `n` / `N` | Search the diff, next / previous match |
| `0` / `1` / `2` | Auto / unified / split layout |
| `s` | Toggle the file list |
| `w`, `l` | Toggle line wrapping, line numbers |
| `z` | Show the unchanged context around the selected hunk |
| `e` | Open the file in `$EDITOR` (Neovim) |
| `c` | Add a note on the current line (`Ctrl+S` saves it) |
| `v`, then `y` | Select lines and copy them |
| `a` | Show or hide agent notes |
| `q` | Quit |

The mouse works too: click a file in the sidebar to jump to it, scroll with the wheel, and drag across rows to select. Copying uses OSC 52, the same terminal clipboard protocol as [Neovim over SSH](neovim.md), so it also works on remote machines.

## Config

```toml title="~/.config/hunk/config.toml"
theme = "catppuccin-macchiato"   # matches everything else
mode = "auto"                    # split when the terminal is wide, unified when narrow
line_numbers = true
wrap_lines = false
hunk_headers = true
menu_bar = true
agent_notes = false
cursor_line = "row"
```

Press `t` to try other themes live. When you quit after changing the view, Hunk offers to save the change to this file. A repository can also have its own `.hunk/config.toml`.

## Reviewing Claude's changes

This is the main reason to use Hunk. Hunk runs a small local daemon, and a running Hunk window can be inspected and steered from the command line with `hunk session …`. The bundled `hunk-review` skill teaches Claude Code those commands. With it, Claude can:

- read which file and hunk you're looking at;
- jump the view to a specific file, hunk or line;
- leave inline review notes, with a summary and rationale, right above the code they're about;
- highlight the exact characters it's talking about.

Setup:

```bash
ln -s ~/.hunk/skills/hunk-review ~/.claude/skills/hunk-review   # make the skill visible to Claude Code
```

```markdown title="~/.claude/commands/hunk.md"
---
description: Walk through the changes in the open Hunk session using the hunk-review skill
---

Use the hunk-review skill to walk me through the changes in my open Hunk session. $ARGUMENTS
```

Both live in my dotfiles. chezmoi manages the skill as a symlink, so `chezmoi update` sets this up on every machine.

The workflow:

1. Claude makes changes.
2. In another pane, run `hunk diff --watch`.
3. In Claude Code, type `/hunk`, or `/hunk focus on the error handling`.
4. Claude walks through the diff in your Hunk window: it moves the view, leaves notes on what changed and why, and highlights the risky lines. Step through them with `{` / `}` (previous / next annotated hunk).
5. Reply to a note in Hunk (`c`, or `R` on a note), then ask Claude to read the notes with `hunk session comment list` and act on them.

[Herdr](tmux-herdr.md#herdr) makes the side-by-side layout easy: Claude in one pane, `hunk diff --watch` in the next.

!!! tip "The skill only controls Hunk, it doesn't open it"
    The skill tells Claude not to start `hunk diff` itself, because that's an interactive program for you. If Claude says there's no session, open Hunk in a terminal first.

## Extras

- **Git pager.** `hunk pager` wraps Git's pager and switches to Hunk's view whenever the output is a diff. For example, `git config --global core.pager "hunk pager"` uses it for `git diff`, `git show` and `git log -p`.
- **Patches from anywhere.** `gh pr diff 123 | hunk patch`, or `hunk gh pr <number>` directly.
- **Extensions.** Plain TypeScript files in `~/.config/hunk/extensions/` can add commands, panes, themes and more. The bundled `hunk-extensions` skill (`hunk skill show hunk-extensions`) explains the API to Claude.
