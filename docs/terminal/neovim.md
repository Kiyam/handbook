# Neovim

I use [Neovim](https://neovim.io) with the [LazyVim](https://www.lazyvim.org) distribution. LazyVim provides a complete IDE-like setup (LSP, completion, telescope-style pickers, git signs, file tree) out of the box, so my own config is just a few small overrides on top.

The config lives in my dotfiles under [`private_dot_config/nvim`](https://github.com/Kiyam/dotfiles/tree/master/private_dot_config/nvim) and is applied by chezmoi.

## Install

```bash
brew install neovim   # or your distro's package; LazyVim needs a recent version
```

I set it as the editor everywhere:

```bash title="~/.config/zsh/.zshrc"
export EDITOR='nvim'
export VISUAL='nvim'
```

## Layout

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

## Overrides

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

## Clipboard over SSH (OSC 52)

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
