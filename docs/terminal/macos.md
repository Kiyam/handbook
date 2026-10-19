# macOS extras

Not terminal tools as such, but they make the Mac nicer to work on. All of them install with Homebrew.

```bash
brew install --cask nikitabobko/tap/aerospace linearmouse boring-notch
brew install mole
```

## AeroSpace: tiling windows

[AeroSpace](https://github.com/nikitabobko/AeroSpace) is an i3-style tiling window manager. Windows tile automatically instead of overlapping, and everything is on the keyboard with the same `h`/`j`/`k`/`l` keys as Neovim. It's configured in `~/.config/aerospace/aerospace.toml`, managed by [chezmoi](terminal-setup.md#chezmoi).

| Keys | Action |
|---|---|
| `alt` + `h`/`j`/`k`/`l` | Focus the window left / down / up / right |
| `alt` + `ctrl` + `h`/`j`/`k`/`l` | Move the window |
| `alt` + `6`–`0`, `y`–`p` | Switch workspace |
| `alt` + `ctrl` + workspace key | Send the window to that workspace |
| `alt` + `tab` | Back to the previous workspace |
| `alt` + `/` or `alt` + `,` | Tiles or accordion layout |
| `alt` + `ctrl` + `.` | Service mode: `f` float/tile, `g` fullscreen, `r` reset layout, `esc` reload config |

Some details worth copying:

- The default layout is **accordion**, which stacks windows with a small offset rather than squashing them side by side. This works better on a laptop screen.
- Gaps are set per monitor: bigger on the external display, tight on the laptop.
- `on-window-detected` rules make small utility apps (Finder, authenticator apps, the VPN client) float instead of tiling.
- [JankyBorders](https://github.com/FelixKratz/JankyBorders) is started from `after-startup-command` and draws a border around the focused window, so you can see which one has focus.
- `ff` in the shell fuzzy-finds any open window and jumps to it:

    ```bash
    fuzzy_aerospace() {
      aerospace list-windows --all | fzf --bind 'enter:execute(bash -c "aerospace focus --window-id {1}")+abort'
    }
    alias ff=fuzzy_aerospace
    ```

## LinearMouse

[LinearMouse](https://linearmouse.app) sets mouse settings per device. Mine turns off pointer acceleration and reverses vertical scrolling for the mouse only, so the trackpad keeps natural scrolling. It also makes the side buttons work as back/forward in every app.

## Boring Notch

[Boring Notch](https://github.com/TheBoredTeam/boring.notch) turns the MacBook notch into a small dynamic island. Hovering over it shows media controls, a calendar and a file shelf, so the notch is useful rather than just in the way.

## Mole

[Mole](https://mole.fit) (`mo`) is a command-line Mac cleaner. Running `mo` opens a menu; the subcommands I use most:

```bash
mo clean --dry-run   # preview what would be deleted (caches, logs, leftovers)
mo clean             # then actually free the space
mo uninstall         # remove an app along with its support files
mo analyze           # explore what's using the disk
mo purge             # remove old project build artifacts
```
