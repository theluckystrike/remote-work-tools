---
layout: default
title: "Remote Team Neovim Setup and Config Sharing"
description: "Share Neovim configs across a distributed team using a dotfiles repo with LSP, Treesitter, and lazy.nvim plugin lock files for reproducible setups across all machines"
date: 2026-03-22
author: theluckystrike
permalink: /remote-team-neovim-setup-config-sharing/
categories: [guides]
reviewed: true
score: 7
intent-checked: true
voice-checked: true
tags: [remote-work-tools, remote-work]
---

{% raw %}
## Remote Team Neovim Setup and Config Sharing

Getting a new engineer's editor configured correctly used to mean a half-day pairing session going through their dotfiles. With a shared Neovim config repo and a one-command bootstrap, a new team member has a working setup in under ten minutes — the same setup everyone else uses, with the same LSP servers, formatters, and keybindings.

---

## Repo Structure

```
nvim-config/
├── init.lua
├── lua/
│   ├── config/
│   │   ├── options.lua
│   │   ├── keymaps.lua
│   │   └── autocmds.lua
│   └── plugins/
│       ├── init.lua
│       ├── lsp.lua
│       ├── treesitter.lua
│       ├── completion.lua
│       ├── ui.lua
│       └── git.lua
├── lazy-lock.json
└── .nvimrc.local.example
```

The key decision: commit `lazy-lock.json`. This pins every plugin to the exact version everyone else is running.

---

## Bootstrap Script

**`install.sh`**

```bash
#!/bin/bash
set -euo pipefail

CONFIG_REPO="${1:-git@github.com:yourorg/nvim-config.git}"
NVIM_CONFIG_DIR="${XDG_CONFIG_HOME:-$HOME/.config}/nvim"

if [[ -d "$NVIM_CONFIG_DIR" ]]; then
  mv "$NVIM_CONFIG_DIR" "${NVIM_CONFIG_DIR}.bak.$(date +%s)"
fi

git clone "$CONFIG_REPO" "$NVIM_CONFIG_DIR"

if ! command -v nvim &>/dev/null; then
  if [[ "$(uname)" == "Darwin" ]]; then
    brew install neovim
  else
    curl -LO https://github.com/neovim/neovim/releases/latest/download/nvim-linux-x86_64.appimage
    chmod +x nvim-linux-x86_64.appimage
    sudo mv nvim-linux-x86_64.appimage /usr/local/bin/nvim
  fi
fi

nvim --headless "+Lazy! restore" +qa
echo "Neovim setup complete. Run: nvim"
```

---

## Core Config

**`init.lua`**

```lua
require("config.options")
require("config.keymaps")
require("config.autocmds")
require("plugins")

local local_config = vim.fn.stdpath("config") .. "/.nvimrc.local"
if vim.loop.fs_stat(local_config) then
  dofile(local_config)
end
```

**`lua/config/options.lua`**

```lua
local opt = vim.opt

opt.number = true
opt.relativenumber = true
opt.expandtab = true
opt.shiftwidth = 2
opt.tabstop = 2
opt.smartindent = true
opt.termguicolors = true
opt.signcolumn = "yes"
opt.updatetime = 250
opt.timeoutlen = 300
opt.undofile = true
opt.ignorecase = true
opt.smartcase = true
opt.splitright = true
opt.splitbelow = true
opt.scrolloff = 8
opt.clipboard = "unnamedplus"
```

---

## Plugin Setup with lazy.nvim

**`lua/plugins/init.lua`**

```lua
local lazypath = vim.fn.stdpath("data") .. "/lazy/lazy.nvim"
if not vim.loop.fs_stat(lazypath) then
  vim.fn.system({
    "git", "clone", "--filter=blob:none",
    "https://github.com/folke/lazy.nvim.git",
    "--branch=stable",
    lazypath,
  })
end
vim.opt.rtp:prepend(lazypath)

require("lazy").setup("plugins", {
  lockfile = vim.fn.stdpath("config") .. "/lazy-lock.json",
  checker = { enabled = false },
  change_detection = { enabled = false },
})
```

**`lua/plugins/lsp.lua`**

```lua
return {
  {
    "neovim/nvim-lspconfig",
    dependencies = {
      "williamboman/mason.nvim",
      "williamboman/mason-lspconfig.nvim",
    },
    config = function()
      require("mason").setup()
      require("mason-lspconfig").setup({
        ensure_installed = {
          "lua_ls",
          "pyright",
          "tsserver",
          "gopls",
          "rust_analyzer",
          "bashls",
        },
        automatic_installation = true,
      })

      local lspconfig = require("lspconfig")
      local on_attach = function(_, bufnr)
        local map = function(keys, func, desc)
          vim.keymap.set("n", keys, func, { buffer = bufnr, desc = desc })
        end
        map("gd", vim.lsp.buf.definition, "Go to definition")
        map("gr", vim.lsp.buf.references, "References")
        map("K", vim.lsp.buf.hover, "Hover documentation")
        map("<leader>rn", vim.lsp.buf.rename, "Rename")
        map("<leader>ca", vim.lsp.buf.code_action, "Code action")
        map("<leader>f", function()
          vim.lsp.buf.format({ async = true })
        end, "Format")
      end

      require("mason-lspconfig").setup_handlers({
        function(server_name)
          lspconfig[server_name].setup({ on_attach = on_attach })
        end,
      })
    end,
  },
}
```

---

## Managing Plugin Updates as a Team

```bash
# On a branch, update all plugins
nvim --headless "+Lazy! update" +qa

# lazy-lock.json is now modified
git diff lazy-lock.json

# Open PR for team review
git add lazy-lock.json
git commit -m "chore: update nvim plugins $(date +%Y-%m-%d)"
gh pr create --title "Neovim plugin updates $(date +%Y-%m-%d)" \
  --body "Weekly plugin update. Test: git checkout <branch> && nvim +Lazy! restore"
```

---

## Per-Developer Overrides

**`.nvimrc.local.example`**

```lua
-- Copy to .nvimrc.local and customize (NOT committed to repo)

-- Personal colorscheme override
-- vim.cmd.colorscheme("catppuccin-mocha")

-- Personal keybindings
-- vim.keymap.set("n", "<leader>e", ":NvimTreeToggle<CR>")

-- Workspace-specific LSP settings
-- require("lspconfig").pyright.setup({
--   settings = { python = { pythonPath = "/usr/local/bin/python3" } }
-- })
```

---

## Shared Snippets

```lua
-- In options.lua
require("luasnip.loaders.from_vscode").lazy_load({
  paths = {
    vim.fn.stdpath("config") .. "/snippets/shared",
    vim.fn.stdpath("config") .. "/snippets/local",
  }
})
```

---

## Related Reading

- [Remote Team tmux Config Sharing Guide](/remote-work-tools/remote-team-tmux-config-sharing/)
- [Remote Team fish Shell Setup Guide](/remote-work-tools/remote-team-fish-shell-setup/)
- [How to Set Up Teleport for Secure Access](/remote-work-tools/teleport-secure-access-setup/)

---

Built by theluckystrike — More at [zovo.one](https://zovo.one)
{% endraw %}
