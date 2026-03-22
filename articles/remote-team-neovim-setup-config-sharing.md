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

This guide walks through the full setup: repo structure, bootstrap script, core config, LSP, Treesitter, completion, per-developer overrides, shared snippets, plugin update workflow, and a few practical patterns learned from running this across teams of different sizes.

---

## Why a Shared Neovim Config

The argument for shared editor configs is the same as the argument for shared linter configs: consistency reduces noise in code review and debugging. When two engineers look at the same file in Neovim, they should see the same diagnostics, the same formatting, and the same go-to-definition behavior.

There are also practical onboarding benefits:

- A new hire who uses Neovim gets a fully working setup from the team's config repo in minutes, not hours
- Engineers switching between machines (laptop, dev server, CI container with an embedded editor) get the same experience everywhere
- When a language server is upgraded, the team upgrades together via a PR to the lock file rather than each person independently discovering the change

The tradeoff is that some engineers want personal config control. The approach here handles that with a `.nvimrc.local` override file that loads after the shared config and is never committed.

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

The key decision: commit `lazy-lock.json`. This pins every plugin to the exact version everyone else is running. Without the lock file, `Lazy! restore` resolves to the latest-stable of each plugin, which can differ between installs by weeks.

The `snippets/` directory is optional but valuable — shared code snippets for the languages your team writes every day.

---

## Bootstrap Script

The install script handles the full setup in a single command: backup any existing config, clone the repo, install Neovim if missing, and restore pinned plugins.

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

Run it on a fresh machine:

```bash
bash <(curl -s https://raw.githubusercontent.com/yourorg/nvim-config/main/install.sh)
```

The `Lazy! restore` command reads `lazy-lock.json` and installs exactly the pinned commit of every plugin. The `--headless` flag runs Neovim without a UI, so it works over SSH, in Docker, or as part of an automated dotfiles provisioner like Ansible.

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

**`lua/config/keymaps.lua`** — keep team keymaps in one place so every engineer has the same muscle memory:

```lua
local map = vim.keymap.set

-- Better window navigation
map("n", "<C-h>", "<C-w>h", { desc = "Move to left window" })
map("n", "<C-j>", "<C-w>j", { desc = "Move to lower window" })
map("n", "<C-k>", "<C-w>k", { desc = "Move to upper window" })
map("n", "<C-l>", "<C-w>l", { desc = "Move to right window" })

-- Quick buffer navigation
map("n", "<S-h>", "<cmd>bprevious<cr>", { desc = "Prev buffer" })
map("n", "<S-l>", "<cmd>bnext<cr>", { desc = "Next buffer" })

-- Clear search highlights
map("n", "<leader>h", "<cmd>nohlsearch<cr>", { desc = "Clear highlights" })

-- Quick save
map({ "n", "i" }, "<C-s>", "<cmd>w<cr>", { desc = "Save file" })

-- Diagnostics navigation
map("n", "[d", vim.diagnostic.goto_prev, { desc = "Prev diagnostic" })
map("n", "]d", vim.diagnostic.goto_next, { desc = "Next diagnostic" })
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

Disabling `checker` and `change_detection` prevents Neovim from automatically checking for plugin updates. Updates happen explicitly via PR, not automatically on each developer's machine.

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

## Treesitter and Completion

**`lua/plugins/treesitter.lua`** — install parsers for all languages the team uses:

```lua
return {
  {
    "nvim-treesitter/nvim-treesitter",
    build = ":TSUpdate",
    config = function()
      require("nvim-treesitter.configs").setup({
        ensure_installed = {
          "lua", "python", "typescript", "javascript",
          "go", "rust", "bash", "json", "yaml", "toml",
          "markdown", "markdown_inline", "dockerfile",
        },
        highlight = { enable = true },
        indent = { enable = true },
        auto_install = false,
      })
    end,
  },
}
```

**`lua/plugins/completion.lua`** — nvim-cmp with LSP, buffer, and snippet sources:

```lua
return {
  {
    "hrsh7th/nvim-cmp",
    dependencies = {
      "hrsh7th/cmp-nvim-lsp",
      "hrsh7th/cmp-buffer",
      "hrsh7th/cmp-path",
      "L3MON4D3/LuaSnip",
      "saadparwaiz1/cmp_luasnip",
    },
    config = function()
      local cmp = require("cmp")
      local luasnip = require("luasnip")

      cmp.setup({
        snippet = {
          expand = function(args)
            luasnip.lsp_expand(args.body)
          end,
        },
        mapping = cmp.mapping.preset.insert({
          ["<C-b>"] = cmp.mapping.scroll_docs(-4),
          ["<C-f>"] = cmp.mapping.scroll_docs(4),
          ["<C-Space>"] = cmp.mapping.complete(),
          ["<C-e>"] = cmp.mapping.abort(),
          ["<CR>"] = cmp.mapping.confirm({ select = true }),
        }),
        sources = cmp.config.sources({
          { name = "nvim_lsp" },
          { name = "luasnip" },
        }, {
          { name = "buffer" },
          { name = "path" },
        }),
      })
    end,
  },
}
```

---

## Managing Plugin Updates as a Team

Plugin updates go through a PR so everyone can review the diff before updating:

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

The PR body should note any plugins with major version bumps. A reviewer who uses that plugin regularly should test before approving.

To roll back a plugin update that broke something:

```bash
git checkout main -- lazy-lock.json
nvim --headless "+Lazy! restore" +qa
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

Add `.nvimrc.local` to the repo's `.gitignore` so it can never accidentally be committed. The `init.lua` loads it only if the file exists, so engineers who prefer no overrides don't need to do anything.

---

## Shared Snippets

Snippets for common patterns — API route skeletons, test function stubs, docstring templates — save time and keep the team's code consistent:

```lua
-- In options.lua
require("luasnip.loaders.from_vscode").lazy_load({
  paths = {
    vim.fn.stdpath("config") .. "/snippets/shared",
    vim.fn.stdpath("config") .. "/snippets/local",
  }
})
```

Store snippets in VSCode-compatible JSON format in `snippets/shared/<language>.json`. LuaSnip loads them on startup. Engineers can add personal snippets in `snippets/local/` (gitignored).

---

## Frequently Asked Questions

**Can engineers use a different plugin manager?** Not with this setup — the lock file and bootstrap script assume lazy.nvim. Engineers who strongly prefer Packer or vim-plug should maintain a personal fork and sync shared LSP configs manually.

**What if an engineer doesn't use Neovim at all?** This config doesn't affect them. It's opt-in. The benefit accumulates when most of the team shares the same setup; a mixed team still benefits from shared LSP configurations (which you can export and import into other editors).

**How do you handle projects that need different Python paths or different LSP settings?** The `.nvimrc.local` file is the right place for workspace-specific overrides. Alternatively, use `vim.env` in a project-local `.nvim.lua` file (Neovim 0.10+ loads these automatically when `exrc` is set).

**Does this work on Windows?** Partially. The bootstrap script is Bash and assumes a Unix filesystem layout. On Windows, the XDG paths differ. Engineers using Windows can clone the repo to `%LOCALAPPDATA%\nvim` manually and run `Lazy! restore` from within Neovim.

---

## Related Reading

- [Remote Team tmux Config Sharing Guide](/remote-work-tools/remote-team-tmux-config-sharing/)
- [Remote Team fish Shell Setup Guide](/remote-work-tools/remote-team-fish-shell-setup/)
- [How to Set Up Teleport for Secure Access](/remote-work-tools/teleport-secure-access-setup/)

---

Built by theluckystrike — More at [zovo.one](https://zovo.one)
{% endraw %}
