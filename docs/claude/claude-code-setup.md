# Claude Code setup

How the rest of my [Claude Code](https://docs.anthropic.com/en/docs/claude-code) is set up. The [plugins](plugins.md) (caveman, ponytail and others) and [skills](skills.md) have their own pages.

## A `CLAUDE.md` per project

Projects I work on with Claude have a `CLAUDE.md` at their root. Claude reads it at the start of each session. Keep it to what Claude can't work out from the code:

- the commands to build, run and deploy;
- the layout and *why* it's that way;
- the gotchas that already cost you time once.

For example, the `CLAUDE.md` for my [homelab Docker stack](../homelab/docker.md) says to always restart services from the repo root and never with `-f compose/<file>.yml`, because the included files can't see the root networks. The `claude-md-management` [plugin](plugins.md#other-plugins) helps keep these files current.

## Herdr integration

On every machine where I run Claude inside [Herdr](../terminal/tmux-herdr.md#herdr), `herdr integration install claude` adds a `SessionStart` hook, so Herdr knows which Claude session lives in which pane. See [Remote machines](../terminal/tmux-herdr.md#remote-machines).

## Safety hook: confirm recursive `rm`

A `PreToolUse` hook stops any Bash command containing a recursive `rm` and asks me on the terminal before it runs. Claude can't approve it on my behalf.

=== "~/.claude/hooks/rm_confirmation.sh"

    ```bash
    #!/usr/bin/env bash
    # Hook: intercept rm -r / rm -rf commands and require explicit user confirmation.
    # Receives tool input JSON on stdin; exits 0 to allow, exits 2 to block.

    set -euo pipefail

    input=$(cat)
    command=$(echo "$input" | python3 -c "import sys,json; d=json.load(sys.stdin); print(d.get('command',''))" 2>/dev/null || true)

    # Match rm with recursive flags anywhere in the command
    if echo "$command" | grep -qE '(^|[;&|`(]|\bsudo\b)\s*rm\s+[^;]*-[a-zA-Z]*r'; then
      # Write prompt directly to the terminal, bypassing stdout (which Claude reads)
      exec 3>/dev/tty
      echo "" >&3
      echo "⚠️  Claude wants to run a recursive rm command:" >&3
      echo "   $command" >&3
      echo "" >&3
      printf "Allow this? [y/N] " >&3
      exec 3>&-

      # Read the answer from the terminal
      read -r answer </dev/tty

      if [[ "$answer" =~ ^[Yy]$ ]]; then
        exit 0
      else
        echo "User denied recursive rm. Command was: $command"
        exit 2
      fi
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
