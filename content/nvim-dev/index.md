---
title: "Nvim Dev"
date: 2025-03-17T11:27:33+03:00
draft: true
---

Here is a quick list of my Neovim setup for dev work.
This includes plugins, LSPs and some handy keymaps.

## Plugins
1. [Telescope](https://github.com/nvim-telescope/telescope.nvim) - Fuzzy find files quickly. No need for filetree with this one.
2. [Harpoon](https://github.com/ThePrimeagen/harpoon) - Mark frequently visited files and get to them blazingly fast.
3. [Mason](https://github.com/williamboman/mason) - This helps in installing language servers. Make sure you have [mason-lspconfig.nvim](https://github.com/williamboman/mason-lspconfig.nvim) and [nvim-lspconfig](https://github.com/neovim/nvim-lspconfig).
4. [nvim-cmp](https://github.com/hrsh7th/nvim-cmp) - For completions. Make sure to also get [Luasnip](https://github.com/L3MON4D3/LuaSnip), [cmp_luasnip](https://github.com/saadparwaiz1/cmp_luasnip), [friendly-snippets](https://github.com/rafamadriz/friendly-snippets) and [nvim-cmp-lsp](https://github.com/hrsh7th/cmp-nvim-lsp).
5. [Neogit](https://github.com/NeogitOrg/neogit) - Git integration.
6. [Copilot](https://github.com/zbirenbaum/copilot.lua) - Get copilot in neovim. Do not forget to get [copilot-cmp](https://github.com/zbirenbaum/copilot-cmp) as well.
7. [Treesitter](https://github.com/nvim-treesitter/nvim-treesitter) - For some sweet syntax highlighting.
8. [Oil](https://github.com/stevearc/oil.nvim) - Create directories and files. (You can also opt for Netrw)

## LSPs
This section starts with must have LSPs. The optional section is just catered to my taste and programming languages I use.
1. [Snyk](https://github.com/snyk/snyk-ls) - Static code security checks in neovim as you code. **I'll write about how to get this working in Neovim as documentation might be a bit lacking**.
2. [Lua-ls](https://github.com/LuaLS/lua-language-server) - How can you use neovim without it?

### Optional
3. [zls](https://github.com/zigtools/zls) - Because I write zig.
4. [pylyzer](https://github.com/mtshiba/pylyzer) - I write python as well. **Fast and slow :D**

## Keymaps
1. For document symbols. View and jump around code symbols
```lua
vim.keymap.set('n', '<leader>ds',
    ':Telescope lsp_document_symbols<CR>',
    {noremap = true, silent=true})
```
2. For code actions
```lua
vim.keymap.set({ 'n', 'v' },
    '<leader>ca',
    vim.lsp.buf.code_action, {})
```
3. Creating new file/directory
```lua
vim.keymap.set("n", "<C-n>",
    ":lua require('oil.actions').new()<CR>",
    { desc = "Create new file/directory in Oil" })
```
4. Search keymaps
```lua
vim.keymap.set('n', '<leader>sk',
    require("telescope.builtin").keymaps,
    { desc = '[S]earch Keymaps ' })
```

> [!Note]
>
> Part 2 which will be more cybersecurity focused
