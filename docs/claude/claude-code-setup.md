# Claude Code setup

How my [Claude Code](https://docs.anthropic.com/en/docs/claude-code) is set up. The two plugins that change day-to-day behaviour the most are **caveman** and **ponytail**. They complement each other:

- **Caveman** shrinks what the agent *says*.
- **Ponytail** shrinks what the agent *builds*.

Caveman doesn't touch code, and ponytail doesn't touch prose, so they don't overlap. Together you get terse explanations of minimal code.

## Caveman

Repo: [JuliusBrussee/caveman](https://github.com/JuliusBrussee/caveman)

Makes Claude drop filler, pleasantries and hedging and reply in short fragments ("Bug in auth middleware. Token expiry check use `<` not `<=`. Fix:"). Code blocks, commands, error messages and commit messages stay exact and are written normally. The README reports about 65% fewer output tokens on average across its benchmarks. Thinking tokens aren't affected, so the main win is replies that are faster to read.

### Install

Run each command as a separate prompt inside Claude Code:

```text
/plugin marketplace add JuliusBrussee/caveman
```

```text
/plugin install caveman@caveman
```

Or from a shell:

```bash
claude plugin marketplace add JuliusBrussee/caveman && claude plugin install caveman@caveman
```

On Claude Code it turns itself on at the start of every session through a hook, so you don't need to type `/caveman`.

### Usage

| Command | What it does |
|---|---|
| `/caveman [lite\|full\|ultra\|wenyan]` | Set the compression level (default `full`). The level lasts until the session ends. |
| `normal mode` / `stop caveman` | Turn it off for the session. |
| `/caveman-commit` | Conventional Commit message with a subject of 50 characters or fewer. |
| `/caveman-review` | Code review as one line per issue: `L42: 🔴 bug: user null. Add guard.` |
| `/caveman-compress <file>` | Rewrite a memory file (e.g. `CLAUDE.md`) in caveman style to save input tokens every session. Keeps a `.original.md` backup. |
| `/caveman-stats` | Real token usage and savings for the session. |
| `/caveman-help` | Quick reference. |

It also ships `cavecrew-*` subagents (investigator / builder / reviewer) whose output comes back compressed, so the main context lasts longer.

!!! tip
    Caveman switches itself back to normal prose for security warnings, confirmations of irreversible actions, and step-by-step instructions where fragments could be misread. If something needs a careful explanation, ask it to clarify and it will.

## Ponytail

Repo: [DietrichGebert/ponytail](https://github.com/DietrichGebert/ponytail)

Puts a "lazy senior developer" inside the agent: someone who replaces fifty lines with one. Ask for a date picker and instead of installing a library and writing a wrapper component, you get:

```html
<!-- ponytail: browser has one -->
<input type="date">
```

### The ladder

Once the agent has read and understood the code involved, it stops at the first rung that works:

```text
1. Does this need to exist?   → no: skip it (YAGNI)
2. Already in this codebase?  → reuse it, don't rewrite
3. Stdlib does it?            → use it
4. Native platform feature?   → use it
5. Installed dependency?      → use it
6. One line?                  → one line
7. Only then: the minimum that works
```

It never cuts input validation, error handling that prevents data loss, security, or accessibility. Deliberate shortcuts get a `ponytail:` comment that names their limit and how to upgrade later. That way they can be tracked instead of forgotten.

### Install

Two separate prompts inside Claude Code:

```text
/plugin marketplace add DietrichGebert/ponytail
```

```text
/plugin install ponytail@ponytail
```

It needs `node` on the PATH of non-interactive shells for its hooks. Without it, the skills still work, but the always-on activation stays off.

### Usage

| Command | What it does |
|---|---|
| `/ponytail [lite\|full\|ultra\|off]` | Set the intensity, or turn it off. With no argument it reports the current level. |
| `/ponytail-review` | Review the current diff for over-engineering and return a list of things to delete. |
| `/ponytail-audit` | The same check across the whole repo. |
| `/ponytail-debt` | Collect every `ponytail:` shortcut into a ledger. |
| `/ponytail-gain` | Benchmark scoreboard. |
| `/ponytail-help` | Quick reference. |

??? note "Changing the default level"
    Set `PONYTAIL_DEFAULT_MODE` (`lite`/`full`/`ultra`/`off`), or add a `defaultMode` field to `~/.config/ponytail/config.json`. The default is `full`. By default the rules are also injected into subagents. To limit which agent types get them, set `PONYTAIL_SUBAGENT_MATCHER` to a regex.

## Other plugins

From the official `claude-plugins-official` marketplace, which is available by default:

| Plugin | Why |
|---|---|
| `superpowers` | Process skills: brainstorming, writing plans, TDD, systematic debugging, checking work before calling it done. |
| `code-review` | `/code-review` on a PR or the local diff. |
| `github` | GitHub MCP server (issues, PRs, code search). |
| `claude-md-management` | Audit and update `CLAUDE.md` files. |
| `skill-creator` | Build and test new skills. |
| `frontend-design` | Design guidance for UI work. |

Plus [obsidian-cli](https://github.com/pablo-mano/Obsidian-CLI-skill) for reading and writing my Obsidian vault.

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

The statusline shows caveman's mode badge plus my plan usage on one line:

```text
[CAVEMAN]  ·  session 42% · 2h10m · weekly 18% · $3.2 session
```

- **Caveman badge:** `[CAVEMAN]`, or `[CAVEMAN:ULTRA]` etc. at levels other than `full`. If `/caveman-stats` has run at least once, the lifetime tokens saved follow the badge. Set `CAVEMAN_STATUSLINE_SAVINGS=0` to hide that.
- **Usage:** how much of the 5-hour session quota is used and when it resets, the weekly quota, and this session's cost. Percentages turn yellow at 75% and red at 90%.

Claude Code only accepts one `statusLine` command, so a small wrapper script builds the line. Where the usage numbers come from depends on the OS:

=== "Linux"

    The usage numbers come straight from the JSON that Claude Code sends to the statusline command on stdin (`rate_limits` and `cost`). [`jq`](https://jqlang.org/) formats them, so `jq` is the only extra dependency.

    ```bash title="~/.claude/statusline-combined.sh"
    #!/bin/bash
    # Combined statusline: caveman badge + usage from Claude Code's own rate_limits JSON.
    INPUT=$(cat)

    CAVEMAN=$(printf '%s' "$INPUT" | bash "$HOME/.claude/plugins/marketplaces/caveman/src/hooks/caveman-statusline.sh" 2>/dev/null)

    USAGE=$(printf '%s' "$INPUT" | jq -r '
      def col(p): if p >= 90 then "\u001b[31m" elif p >= 75 then "\u001b[33m" else "\u001b[32m" end;
      def eta(t): (t - now | floor) as $s
        | if $s < 60 then "\($s)s" elif $s < 3600 then "\($s/60|floor)m"
          elif $s < 86400 then "\($s/3600|floor)h\(($s%3600)/60|floor)m" else "\($s/86400|floor)d" end;
      [ (.rate_limits.five_hour // empty | col(.used_percentage) + "session \(.used_percentage|round)%\u001b[0m\u001b[2m · \(eta(.resets_at))\u001b[0m"),
        (.rate_limits.seven_day // empty | col(.used_percentage) + "weekly \(.used_percentage|round)%\u001b[0m"),
        (.cost.total_cost_usd // empty | "\u001b[2m$\(. * 100 | round / 100) session\u001b[0m")
      ] | join(" · ")' 2>/dev/null)

    if [ -n "$CAVEMAN" ] && [ -n "$USAGE" ]; then
      printf '%s  \033[2m·\033[0m  %s' "$CAVEMAN" "$USAGE"
    elif [ -n "$CAVEMAN" ]; then
      printf '%s' "$CAVEMAN"
    else
      printf '%s' "$USAGE"
    fi
    ```

=== "macOS"

    On the Mac I use the **Claude God** app. It puts the same usage figures in the macOS menu bar, so they're visible even when no terminal is open. It also exports them to `~/.claude-god/usage.json` (turn on JSON export in its Settings) and generates `~/.claude-god/statusline.sh` to print them. That script also shows today's spend and adds `(stale Nm)` if the app has stopped updating.

    So the wrapper calls Claude God's script for the usage part instead of using `jq`:

    ```bash title="~/.claude/statusline-combined.sh"
    #!/bin/bash
    # Combined statusline: caveman badge + claude-god usage stats, one line.
    INPUT=$(cat)

    CAVEMAN=$(printf '%s' "$INPUT" | bash "$HOME/.claude/plugins/marketplaces/caveman/src/hooks/caveman-statusline.sh" 2>/dev/null)
    USAGE=$(printf '%s' "$INPUT" | "$HOME/.claude-god/statusline.sh" 2>/dev/null)

    if [ -n "$CAVEMAN" ] && [ -n "$USAGE" ]; then
      printf '%s  \033[2m·\033[0m  %s' "$CAVEMAN" "$USAGE"
    elif [ -n "$CAVEMAN" ]; then
      printf '%s' "$CAVEMAN"
    else
      printf '%s' "$USAGE"
    fi
    ```

    !!! tip
        The Linux script works on macOS too (`brew install jq`). Claude God is only needed for the menu bar.

Both versions use the same settings entry. Remember to make the wrapper executable with `chmod +x ~/.claude/statusline-combined.sh`.

```json title="~/.claude/settings.json (excerpt)"
{
  "statusLine": {
    "type": "command",
    "command": "~/.claude/statusline-combined.sh"
  }
}
```

!!! note
    The caveman script path points to the plugin's marketplace checkout. If the plugin moves after an update, the badge just disappears without an error. Check that path first if that happens.
