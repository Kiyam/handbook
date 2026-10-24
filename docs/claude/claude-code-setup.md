# Claude Code setup

How the rest of my [Claude Code](https://docs.anthropic.com/en/docs/claude-code) is set up. [Plugins](plugins.md), [skills](skills.md) and the [research wiki](research.md) have their own pages.

## One config, three machines

I run Claude Code on the Mac and on two servers, the servers reached through [Herdr](../terminal/tmux-herdr.md#remote-machines). Everything under `~/.claude/` that I care about is in my [chezmoi](../terminal/terminal-setup.md#chezmoi) dotfiles, so all three behave the same:

| File | What it holds |
|---|---|
| `~/.claude/settings.json` | Model, hooks, enabled plugins, statusline, skill overrides, permission mode. It's a chezmoi **template**, so paths use each machine's home directory. |
| `~/.claude/CLAUDE.md` | My global instructions (see [Shared memory](#shared-memory-across-machines)). |
| `~/.claude/hooks/`, `statusline-combined.sh` | The [rm hook](#safety-hook-confirm-recursive-rm) and the [statusline](#statusline). |
| `~/.claude/skills/`, `commands/` | My [skills](skills.md) and slash commands. |

`.chezmoiignore` keeps all of this on exactly those three machines. Clusters and other servers where I don't run Claude get none of it:

```text title=".chezmoiignore (excerpt)"
{{- if not (has .chezmoi.hostname (list "<mac>" "<server-1>" "<server-2>")) }}
.claude/settings.json
.claude/statusline-combined.sh
.claude/hooks
{{- end }}
```

Plugins listed in `settings.json` also need installing. A `run_onchange` script does that, and reruns whenever its plugin list changes:

```bash title="run_onchange_after_claude-install-plugins.sh"
#!/bin/sh
PATH="$HOME/.local/bin:/opt/homebrew/bin:$PATH"
command -v claude >/dev/null || exit 0
for p in claude-md-management pyright-lsp session-report; do
  claude plugin install "$p@claude-plugins-official" >/dev/null
done
```

Other settings worth knowing about:

- `"permissions": {"defaultMode": "auto"}` starts every session in auto mode. A classifier checks each action instead of asking me about every command. Recursive `rm` still always asks, because of the [hook](#safety-hook-confirm-recursive-rm).
- `"skillOverrides"` turns off skills I don't want Claude to pick automatically, such as the bundled Office/PDF skills and a couple of my older ones (see [Skills](skills.md)).
- `"theme": "dark-ansi"` uses the terminal's own ANSI colours, like everything else.

## Shared memory across machines

Claude Code's auto-memory is per project and per machine, so something learned on the Mac (a host's quirks, a rule about the homelab) is unknown on a server. I keep facts that should hold everywhere in a separate **private git repo**, cloned at `~/.claude/shared-memory`, with one Markdown file per fact and an `index.md`.

A `SessionStart` hook pulls the repo and prints the index, so every session on every machine starts with it in context:

```json title="~/.claude/settings.json (excerpt)"
{
  "hooks": {
    "SessionStart": [{
      "matcher": "^(startup|resume|clear|compact|fork)$",
      "hooks": [{
        "type": "command",
        "command": "m=\"$HOME/.claude/shared-memory\"; GIT_SSH_COMMAND='ssh -o ConnectTimeout=3 -o BatchMode=yes' git -C \"$m\" pull -q --ff-only >/dev/null 2>&1; [ -f \"$m/index.md\" ] && { echo \"Shared memory index ($m):\"; cat \"$m/index.md\"; }; true",
        "timeout": 8
      }]
    }]
  }
}
```

- **The pull can't hang a session.** `ConnectTimeout=3` and `BatchMode=yes` mean an unreachable git server just skips the pull, and the trailing `true` means the hook never fails.
- **Only the index is loaded.** Claude reads individual files when one looks relevant.

The global `~/.claude/CLAUDE.md` tells Claude how to *write* to it:

```markdown title="~/.claude/CLAUDE.md"
# Shared memory

Facts that hold on every machine live in the private repo `~/.claude/shared-memory`,
same file format as auto-memory. A SessionStart hook pulls it and prints its `index.md`,
so the index is already in context; read individual files on demand.

Save a memory there instead of per-project memory when it applies across machines or
projects. Run `git -C ~/.claude/shared-memory pull --ff-only` first, add the file and
its index line, then commit and push. If the push is rejected, `pull --rebase` and
push again. If the repo is missing, clone it from <private git URL>.
```

To set it up:

```bash
# 1. a private repo on your git host, then on each machine:
git clone <private git URL> ~/.claude/shared-memory
# 2. start the index
printf '%s\n' '- [Example fact](example-fact.md) — one-line summary Claude reads at session start' > ~/.claude/shared-memory/index.md
```

3. Add the `SessionStart` hook above to `~/.claude/settings.json`, and the instructions above to `~/.claude/CLAUDE.md`.
4. Start a new session. The index should appear in the hook output, and from then on Claude adds memories to the repo itself.

Each machine needs an SSH key that can push to the repo. The hook's `BatchMode=yes` means a key with a passphrase only works if it's already loaded in the agent.

The repo itself is private: it holds hostnames and details about my network, which is exactly why it isn't in the public dotfiles.

## A `CLAUDE.md` per project

Projects I work on with Claude have a `CLAUDE.md` at their root. Claude reads it at the start of each session. Keep it to what Claude can't work out from the code:

- the commands to build, run and deploy;
- the layout and *why* it's that way;
- the gotchas that already cost you time once.

For example, the `CLAUDE.md` for my [homelab Docker stack](../homelab/docker.md) says to always restart services from the repo root and never with `-f compose/<file>.yml`, because the included files can't see the root networks. The `claude-md-management` [plugin](plugins.md#other-plugins) helps keep these files current.

## Herdr integration

On every machine where I run Claude inside [Herdr](../terminal/tmux-herdr.md#herdr), `herdr integration install claude` adds a `SessionStart` hook script, so Herdr knows which Claude session lives in which pane. See [Remote machines](../terminal/tmux-herdr.md#remote-machines).

The hook entry in `settings.json` is added by the chezmoi template **only if the script exists**, so a machine without Herdr doesn't get an error at every session start:

```text title="private_settings.json.tmpl (excerpt)"
{{ if stat (joinPath .chezmoi.homeDir ".claude/hooks/herdr-agent-state.sh") }}
  ,{"type":"command","command":"bash '{{ .chezmoi.homeDir }}/.claude/hooks/herdr-agent-state.sh' session","timeout":10}
{{ end }}
```

## Safety hook: confirm recursive `rm`

A `PreToolUse` hook catches any Bash command containing a recursive `rm` and makes Claude Code ask me before it runs, **even in auto mode**. Claude can't approve it on my behalf.

The hook doesn't prompt by itself. It replies with `permissionDecision: "ask"`, so Claude Code shows its normal permission prompt. An earlier version read the answer from `/dev/tty`, which fought with Claude Code's own terminal UI. The pattern catches `rm` anywhere in the command line: after `sudo`, `xargs`, `find -exec`, `then`/`do`, inside quotes, or as `/bin/rm`.

=== "~/.claude/hooks/rm_confirmation.sh"

    ```bash
    #!/usr/bin/env bash
    # Hook: intercept rm -r / rm -rf commands and require explicit user confirmation.
    # Receives the PreToolUse JSON on stdin; the command is at .tool_input.command.
    # Replies with permissionDecision "ask", so Claude Code shows its own prompt
    # (also in auto mode) instead of this script reading /dev/tty under the TUI.

    set -euo pipefail

    input=$(cat)
    command=$(echo "$input" | python3 -c "import sys,json; d=json.load(sys.stdin); print(d.get('tool_input',{}).get('command',''))" 2>/dev/null || true)

    # Match rm as a standalone word anywhere (inside quotes, after xargs/find -exec/
    # env/then/do, /bin/rm, ...) with a recursive flag before the next ; & or |
    if echo "$command" | grep -qE '(^|[^[:alnum:]_.-])rm[[:space:]]([^;&|]*[[:space:]])?(-[a-zA-Z]*[rR]|--recursive)'; then
      python3 -c 'import json,sys; print(json.dumps({"hookSpecificOutput": {"hookEventName": "PreToolUse", "permissionDecision": "ask", "permissionDecisionReason": "Recursive rm: confirm before running:\n" + sys.argv[1]}}))' "$command"
    fi

    exit 0
    ```

=== "~/.claude/settings.json (excerpt)"

    ```json
    {
      "hooks": {
        "PreToolUse": [
          {
            "matcher": "Bash",
            "hooks": [
              {
                "type": "command",
                "command": "bash ~/.claude/hooks/rm_confirmation.sh",
                "timeout": 30
              }
            ]
          }
        ]
      }
    }
    ```

## Statusline

The statusline shows the [caveman and ponytail](plugins.md) mode badges plus my plan usage on one line:

<pre class="terminal-preview"><span class="c172">[CAVEMAN]</span> <span class="c108">[PONYTAIL]</span>  <span class="dim">·</span>  <span class="green">session 42%</span><span class="dim"> · 2h10m</span> · <span class="green">weekly 18%</span> · <span class="dim">$3.2 session</span></pre>

- **Caveman badge:** `[CAVEMAN]`, or `[CAVEMAN:ULTRA]` etc. at levels other than `full`. If `/caveman-stats` has run at least once, the lifetime tokens saved follow the badge. Set `CAVEMAN_STATUSLINE_SAVINGS=0` to hide that.
- **Ponytail badge:** `[PONYTAIL]` in green, or `[PONYTAIL:LITE]` etc. at other levels. `ULTRA` shows in amber. It comes from `hooks/ponytail-statusline.sh`, which ships with the plugin. It doesn't use the JSON from Claude Code, so the wrapper calls it without piping anything in.
- **Usage:** how much of the 5-hour session quota is used and when it resets, the weekly quota, and this session's cost. Percentages turn yellow at 75% and red at 90%.

Claude Code only accepts one `statusLine` command, so a small wrapper script builds the line. It's the same on macOS and Linux. The usage numbers come straight from the JSON that Claude Code sends to the statusline command on stdin (`rate_limits` and `cost`), and [`jq`](https://jqlang.org/) formats them. `jq` is the only extra dependency: it's preinstalled on recent macOS, and on Linux it's `apt install jq` or equivalent.

```bash title="~/.claude/statusline-combined.sh"
#!/bin/bash
# Combined statusline: caveman + ponytail badges + usage from Claude Code's own rate_limits JSON.
INPUT=$(cat)

CAVEMAN=$(printf '%s' "$INPUT" | bash "$HOME/.claude/plugins/marketplaces/caveman/src/hooks/caveman-statusline.sh" 2>/dev/null)
PONYTAIL=$(bash "$HOME/.claude/plugins/marketplaces/ponytail/hooks/ponytail-statusline.sh" 2>/dev/null)
BADGES="$CAVEMAN${CAVEMAN:+${PONYTAIL:+ }}$PONYTAIL"

USAGE=$(printf '%s' "$INPUT" | jq -r '
  def col(p): if p >= 90 then "\u001b[31m" elif p >= 75 then "\u001b[33m" else "\u001b[32m" end;
  def eta(t): (t - now | floor) as $s
    | if $s < 60 then "\($s)s" elif $s < 3600 then "\($s/60|floor)m"
      elif $s < 86400 then "\($s/3600|floor)h\(($s%3600)/60|floor)m" else "\($s/86400|floor)d" end;
  [ (.rate_limits.five_hour // empty | col(.used_percentage) + "session \(.used_percentage|round)%\u001b[0m\u001b[2m · \(eta(.resets_at))\u001b[0m"),
    (.rate_limits.seven_day // empty | col(.used_percentage) + "weekly \(.used_percentage|round)%\u001b[0m"),
    (.cost.total_cost_usd // empty | "\u001b[2m$\(. * 100 | round / 100) session\u001b[0m")
  ] | join(" · ")' 2>/dev/null)

if [ -n "$BADGES" ] && [ -n "$USAGE" ]; then
  printf '%s  \033[2m·\033[0m  %s' "$BADGES" "$USAGE"
elif [ -n "$BADGES" ]; then
  printf '%s' "$BADGES"
else
  printf '%s' "$USAGE"
fi
```

Make the wrapper executable with `chmod +x ~/.claude/statusline-combined.sh`, then point Claude Code at it:

```json title="~/.claude/settings.json (excerpt)"
{
  "statusLine": {
    "type": "command",
    "command": "~/.claude/statusline-combined.sh"
  }
}
```

!!! note
    The caveman and ponytail script paths point to each plugin's marketplace checkout. If a plugin moves after an update, its badge just disappears without an error. Check that path first if that happens.
