# tmux & Herdr

Terminal multiplexers keep sessions alive after you disconnect and split one terminal into windows and panes.

I used to use **tmux**, but have since switched to **Herdr**. It's much better at managing several AI coding agents side by side, and at working across remote servers from one window. The tmux setup is still [further down](#tmux) for posterity, but I don't use it anymore.

## Herdr

Link: [herdr.dev](https://herdr.dev)

Herdr describes itself as an "agent multiplexer that lives in your terminal": a workspace manager for running coding agents like Claude Code in parallel panes. It has named sessions (`herdr --session <name>`, `herdr session attach <name>`) and can attach to a session on another machine over SSH with `herdr --remote <ssh-target>`.

```bash
brew install herdr
```

My config just tones the UI down and makes it use the terminal's colours, so it inherits Ghostty's Catppuccin palette:

```toml title="~/.config/herdr/config.toml"
onboarding = false

[ui]
status_indicators = "symbols"
show_agent_labels_on_pane_borders = false

[theme]
name = "terminal"
auto_switch = false
```

### Remote machines

The Mac runs the Herdr client, and my two Linux servers are saved as **SSH machines**. All three appear in one sidebar: local workspaces plus each server's workspaces. The agent list covers every machine, so I can see at a glance which Claude Code session, wherever it's running, is waiting for an answer.

| Machine | What it is | Hardware |
|---|---|---|
| Local | MacBook, where the Herdr client runs | |
| **prometheus** | Main Linux workhorse, and where I drive my other servers from over SSH | 24 cores, 62 GB RAM |
| **apollo** | Linux box with a GPU | 8 cores, 125 GB RAM, RTX 4060 Ti |

Each server runs its own Herdr server (the `default` session), so agents keep running there when the laptop sleeps or the connection drops. Herdr reconnects by itself, and a dropped connection to one machine doesn't affect the others.

#### Networking and SSH

Both servers are on my [Tailscale](https://tailscale.com) network, so they're reachable from anywhere without opening ports. Herdr uses plain OpenSSH underneath, so everything starts with a normal `~/.ssh/config` entry:

```text title="~/.ssh/config"
Host prometheus
    HostName <tailscale IP or MagicDNS name>
    ForwardAgent yes
    AddKeysToAgent yes
```

- `ForwardAgent yes` lets `git push` over SSH and commit signing inside remote panes use the keys on the Mac. Herdr doesn't turn on forwarding by itself. Only enable it for hosts you trust.
- Herdr's background connections can't answer prompts. If the key has a passphrase, load it with `ssh-add` first, which `AddKeysToAgent` handles after the first use.

Check plain `ssh prometheus` works before involving Herdr.

#### Adding and using machines

```bash
herdr machine add prometheus   # installs/starts Herdr on the server if needed (asks first)
herdr machine add apollo
herdr machine list             # saved profiles and their IDs
herdr machine status           # non-interactive health check: "reachable" or not
herdr machine reconnect apollo # redo SSH auth in this terminal if a machine shows "! auth"
```

- **Switching:** click a machine or workspace in the sidebar, or press `prefix` + `w` and then use the arrow keys to move through workspaces across all machines, with `Enter` to jump.
- **Scripting:** `herdr --machine apollo agent list` runs API commands against a saved machine without opening the UI.
- **One-off attach** without saving a profile: `herdr --remote prometheus`.

!!! tip "Agent status in the sidebar"
    For Herdr to show whether a Claude Code session is working, idle or waiting for input, install its Claude integration **on each machine where Claude Code runs**, not just on the Mac. It adds a small hook to `~/.claude/`:

    ```bash
    herdr integration install claude
    herdr integration status        # check it's installed and up to date
    ```

## tmux

!!! info "No longer used"
    This is my old tmux setup, kept for posterity. I've switched to [Herdr](#herdr).

!!! warning "Older computing clusters"
    TPM and many of its plugins need a fairly recent tmux. On some older computing clusters, the system tmux is far too old, and installing a new enough version yourself (without root, with old compilers and libraries) can be a real challenge.

Config lives at `~/.config/tmux/tmux.conf`. Like most of my config, it lives under `~/.config` rather than as `~/.tmux.conf`, and note there's no dot in front of `tmux.conf`.

### Plugins

Plugins are managed by [TPM](https://github.com/tmux-plugins/tpm). Install it once, then press `prefix` + `I` inside tmux to fetch everything listed in the config:

```bash
git clone https://github.com/tmux-plugins/tpm ~/.config/tmux/plugins/tpm
```

| Plugin | What it does |
|---|---|
| [tmux-sensible](https://github.com/tmux-plugins/tmux-sensible) | Sensible defaults everyone agrees on. |
| [tmux-cpu](https://github.com/tmux-plugins/tmux-cpu) | CPU / RAM usage for the status bar. |
| [tmux-resurrect](https://github.com/tmux-plugins/tmux-resurrect) | Save and restore sessions, windows and panes across reboots. |
| [tmux-continuum](https://github.com/tmux-plugins/tmux-continuum) | Saves automatically, and with `@continuum-restore 'on'` restores when tmux starts. |
| [tmux-better-mouse-mode](https://github.com/nhdaly/tmux-better-mouse-mode) | Mouse scrolling that behaves. |
| [tmux-menus](https://github.com/jaclu/tmux-menus) | Popup menus for tmux actions, handy when you forget a keybinding. |
| [tmux-fzf](https://github.com/sainnhe/tmux-fzf) | Fuzzy-find sessions, windows, panes and commands. |
| [tmux-fzf-url](https://github.com/wfxr/tmux-fzf-url) | Fuzzy-pick any URL visible in the pane and open it. |

The theme is [catppuccin/tmux](https://github.com/catppuccin/tmux). Unlike the others it's cloned manually and loaded with a `run` line:

```bash
git clone https://github.com/catppuccin/tmux.git ~/.config/tmux/plugins/catppuccin/tmux
```

### Settings worth copying

```bash
set -g base-index 1              # start indexing windows at 1 instead of 0
set -g history-limit 1000000     # increase history size (from 2,000)
set -g renumber-windows on       # renumber all windows when any window is closed
set -g update-environment -r     # pick up the newest environment variables
set-option -g mouse on
set-option -g default-shell $SHELL
set -g set-clipboard on
```

### Linux-only bits

My tmux config is a [chezmoi](terminal-setup.md#chezmoi) template, and a few lines only apply on Linux servers:

```bash
{{ if ne .chezmoi.os "darwin" }}
set-option -g prefix C-g
{{ end }}

{{ if ne .chezmoi.os "darwin" }}
set -as terminal-features ',ghostty*:clipboard'
set -as terminal-overrides ',ghostty*:Ms=\E]52;%p1%s;%p2%s\007'
{{ end }}

{{ if ne .chezmoi.os "darwin" }}
set -g default-terminal "tmux-256color"
set -ag terminal-overrides ",ghostty*:RGB"
{{ end }}
```

- **Prefix `Ctrl+G` on servers.** It stays `Ctrl+B` on the Mac. When a server's tmux is running inside the Mac's tmux, each one has its own prefix, so there's no pressing `Ctrl+B` twice.
- **OSC 52 clipboard.** Tells tmux that Ghostty supports OSC 52, so copying in tmux on a server lands in the Mac's clipboard. This is the same mechanism as the [Neovim clipboard](neovim.md#clipboard-over-ssh-osc-52).
- **True colour.** `tmux-256color` plus the `RGB` override, so the Catppuccin colours render properly instead of being approximated.

??? note "Full tmux.conf (chezmoi template)"

    ```bash
    # List of plugins
    set -g @plugin 'tmux-plugins/tpm'
    set -g @plugin 'tmux-plugins/tmux-sensible'
    set -g @plugin 'tmux-plugins/tmux-cpu'
    set -g @plugin 'tmux-plugins/tmux-resurrect'
    set -g @plugin 'tmux-plugins/tmux-continuum'
    set -g @plugin 'nhdaly/tmux-better-mouse-mode'
    set -g @plugin 'jaclu/tmux-menus'
    set -g @plugin 'sainnhe/tmux-fzf'
    set -g @plugin 'wfxr/tmux-fzf-url'

    set -g update-environment -r     # Update the tmux environment to use the newest environmental variables
    set -g base-index 1              # start indexing windows at 1 instead of 0
    set -g history-limit 1000000     # increase history size (from 2,000)
    set -g renumber-windows on       # renumber all windows when any window is closed

    # some general status bar options
    # set-option -g status-position top
    # set-option -g status-style bg=default

    # Catpuccin options
    set -g @catppuccin_flavor "macchiato" # latte, frappe, macchiato, mocha

    set -g @catppuccin_window_status_style "rounded"
    set -g @catppuccin_window_number_position "right"

    set -g @catppuccin_window_number "#I"
    set -g @catppuccin_window_text "#W"
    set -g @catppuccin_window_number_color "#{@thm_lavender}"

    set -g @catppuccin_window_current_number "#I"
    set -g @catppuccin_window_current_text "#W"
    #set -g @catppuccin_window_current_background "#{@thm_lavendar}"

    set -g @catppuccin_status_left_separator  " "
    set -g @catppuccin_status_right_separator ""
    set -g @catppuccin_status_fill "icon"
    set -g @catppuccin_status_connect_separator "no"

    # Load catppuccin
    set -g status-right-length 80
    set -g status-left ""
    set -g status-right "#{E:@catppuccin_status_application}"
    set -ag status-right "#{E:@catppuccin_status_directory}"
    set -ag status-right "#{E:@catppuccin_status_user}"
    set -ag status-right "#{E:@catppuccin_status_host}"

    # Load Catppuccin
    run ~/.config/tmux/plugins/catppuccin/tmux/catppuccin.tmux

    # Enable automatic restore
    set -g @continuum-restore 'on'

    # Mouse options
    set-option -g mouse on

    # Use configured shell as default shell in tmux
    set-option -g default-shell $SHELL

    # Change the prefix key if not on a mac
    {{ if ne .chezmoi.os "darwin" }}
    set-option -g prefix C-g
    {{ end }}

    set -g set-clipboard on
    # Set some options for OSC 52 clipboard
    {{ if ne .chezmoi.os "darwin" }}
    set -as terminal-features ',ghostty*:clipboard'
    set -as terminal-overrides ',ghostty*:Ms=\E]52;%p1%s;%p2%s\007'
    {{ end }}

    # Then set some tmux terminal colours for macos ghostty
    {{ if ne .chezmoi.os "darwin" }}
    set -g default-terminal "tmux-256color"
    set -ag terminal-overrides ",ghostty*:RGB"
    {{ end }}

    # Initialize TMUX plugin manager (keep this line at the very bottom of tmux.conf)
    run '$XDG_CONFIG_HOME/tmux/plugins/tpm/tpm'
    ```
