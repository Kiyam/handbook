# Plugins

The two [Claude Code](https://docs.anthropic.com/en/docs/claude-code) plugins that change day-to-day behaviour the most are **caveman** and **ponytail**. They complement each other:

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
