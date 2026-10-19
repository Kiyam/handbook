# Neovim

## What is Neovim?

[Neovim](https://neovim.io) is a text editor that runs inside the terminal. It's a modernised fork of **Vim**, which in turn goes back to `vi` from the 1970s. Neovim keeps Vim's keybindings and adds a built-in LSP client (go-to-definition, errors as you type), Tree-sitter syntax highlighting, and configuration and plugins in Lua instead of Vimscript.

The big idea is **modal editing**. Instead of always typing text, you switch between modes:

- **Normal mode** (where you start): keys are commands, e.g. `dd` deletes a line and `w` jumps a word.
- **Insert mode**: keys type text, like a normal editor. Press `i` to enter it and `Esc` to go back.
- **Command mode**: press `:` in Normal mode to type a command such as save or quit.

It's awkward for a week, and then it's very hard to go back. Because it runs in a terminal, it's also always there over SSH on a server.

### How do I quit?

Vim is famous for people not knowing how to quit it. People have got stuck in it by accident (often via `git commit`) and ended up closing the whole terminal. So, first things first. Press `Esc` to make sure you're in Normal mode, then:

| Command | What it does |
|---|---|
| `:q` | Quit (refuses if there are unsaved changes) |
| `:q!` | Quit and throw away changes |
| `:w` | Save ("write") |
| `:wq` or `:x` | Save and quit |
| `:qa` | Quit all open files/windows |

A few more basics to survive:

| Keys | What they do |
|---|---|
| `i` / `a` | Insert before / after the cursor |
| `o` | Open a new line below and insert |
| `u` / `Ctrl+R` | Undo / redo |
| `dd` / `yy` / `p` | Delete (cut) line / copy ("yank") line / paste |
| `/text` then `n` | Search for `text`, then go to the next match |
| `gg` / `G` | Jump to the top / bottom of the file |

Run `vimtutor` in a terminal for a 30-minute interactive tutorial. In Neovim, `:Tutor` does the same.

### With LazyVim: just press space

You don't have to memorise much with LazyVim. **Press `Space`** (the "leader" key) in Normal mode and a popup ([which-key](https://github.com/folke/which-key.nvim)) lists every shortcut that follows, grouped by topic. For example, `Space` `f` for files, `Space` `s` for search and `Space` `g` for git. Keep pressing keys and it narrows down, so you can explore the shortcuts as you go. `Space` `Space` finds files, and `Space` `/` searches across the project.

## My setup

I use [Neovim](https://neovim.io) with the [LazyVim](https://www.lazyvim.org) distribution. LazyVim provides a complete IDE-like setup (LSP, completion, telescope-style pickers, git signs, file tree) out of the box, so my own config is just a few small overrides on top.

The config lives in my dotfiles under [`private_dot_config/nvim`](https://github.com/Kiyam/dotfiles/tree/master/private_dot_config/nvim) and is applied by [chezmoi](terminal-setup.md#chezmoi).

### Install

```bash
brew install neovim   # or your distro's package; LazyVim needs a recent version
```

I set it as the editor everywhere:

```bash title="~/.config/zsh/.zshrc"
export EDITOR='nvim'
export VISUAL='nvim'
```

### Layout

```text
~/.config/nvim
├── init.lua                 # require("config.lazy")
└── lua
    ├── config
    │   ├── lazy.lua         # bootstraps lazy.nvim + LazyVim (stock LazyVim starter)
    │   └── options.lua      # clipboard over OSC 52
    └── plugins              # every file here is loaded automatically
        ├── theme.lua
        ├── cmp.lua
        ├── vimtex.lua
        ├── flutter.lua
        └── diffview.lua
```

### Overrides

=== "Theme"

    Catppuccin Macchiato, to [match everything else](terminal-setup.md#one-colour-scheme-everywhere):

    ```lua title="lua/plugins/theme.lua"
    return {
      {
        "LazyVim/LazyVim",
        opts = {
          colorscheme = "catppuccin-macchiato",
        },
      },
    }
    ```

=== "Completion"

    LazyVim uses [blink.cmp](https://github.com/saghen/blink.cmp). I switch it to the "super-tab" preset, so `Tab` accepts a completion. It also steps through snippet placeholders and accepts AI suggestions before falling back to a normal tab:

    ```lua title="lua/plugins/cmp.lua"
    return {
      {
        "saghen/blink.cmp",
        opts = {
          keymap = {
            preset = "super-tab",
            ["<Tab>"] = {
              require("blink.cmp.keymap.presets").get("super-tab")["<Tab>"][1],
              require("lazyvim.util.cmp").map({ "snippet_forward", "ai_accept" }),
              "fallback",
            },
          },
        },
      },
    }
    ```

=== "LaTeX"

    [VimTeX](https://github.com/lervag/vimtex) for writing LaTeX, with [Skim](https://skim-app.sourceforge.io) as the PDF viewer on macOS (it supports jumping between source and PDF in both directions):

    ```lua title="lua/plugins/vimtex.lua"
    return {
      {
        "lervag/vimtex",
        lazy = false, -- VimTeX shouldn't be lazy-loaded
        init = function()
          vim.g.vimtex_view_method = "skim"
        end,
      },
    }
    ```

=== "Flutter"

    [flutter-tools.nvim](https://github.com/nvim-flutter/flutter-tools.nvim): run, hot-reload and pick devices for Flutter apps from inside Neovim.

    ```lua title="lua/plugins/flutter.lua"
    return {
      {
        "nvim-flutter/flutter-tools.nvim",
        lazy = false,
        dependencies = {
          "nvim-lua/plenary.nvim",
          "stevearc/dressing.nvim", -- optional for vim.ui.select
        },
        config = true,
      },
    }
    ```

=== "Diffs"

    [diffview.nvim](https://github.com/sindrets/diffview.nvim): `:DiffviewOpen` shows a side-by-side review of all changed files, and `:DiffviewFileHistory` shows a file's history.

    ```lua title="lua/plugins/diffview.lua"
    return {
      {
        "sindrets/diffview.nvim",
        lazy = true,
      },
    }
    ```

### Clipboard over SSH (OSC 52)

Yanking on a remote server normally can't reach the clipboard on your laptop. OSC 52 is a terminal escape sequence that sends the yanked text *through the terminal* to the local clipboard. It works over SSH and inside tmux, as long as the terminal allows it, which Ghostty does with [`clipboard-write = allow`](terminal-setup.md#terminal-emulator-ghostty).

```lua title="lua/config/options.lua"
vim.opt.clipboard = "unnamedplus"
vim.g.clipboard = {
  name = "OSC 52",
  copy = {
    ["+"] = require("vim.ui.clipboard.osc52").copy("+"),
    ["*"] = require("vim.ui.clipboard.osc52").copy("*"),
  },
  paste = {
    ["+"] = function()
      return {}
    end,
    ["*"] = function()
      return {}
    end,
  },
}
```

With `unnamedplus`, every yank goes to the system clipboard. Reading from the system clipboard is stubbed out, so to paste something from the laptop, use the terminal's normal paste (`Cmd+V`).
