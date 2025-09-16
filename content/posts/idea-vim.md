---
title: "IDEA Vim"
date: 2025-09-16T11:27:02+01:00
draft: false
toc: false
images:
- /logo.png
categories:
- Software Development
tags:
  - Android
  - Kotlin
  - Vim
  - IDEA
  - Productivity
---

I've been using the [IdeaVim plugin](https://www.jetbrains.com/help/idea/using-product-as-the-vim-editor.html) in Android Studio for a while. I only ever learned the basics with [Practical Vim](https://www.goodreads.com/book/show/13607232-practical-vim), but that's been enough.

The best part is that even this small amount of Vim carries over everywhere: I can use the same workflow in [Android Studio](https://developer.android.com/studio), [VS Code](https://code.visualstudio.com), [iTerm](https://iterm2.com), or [Obsidian](https://obsidian.md). Most editor shortcuts I used to know are gone, and I don’t miss them.

## IDEA Vim config

Here’s my `.ideavimrc`:

```vim
" Global .ideavimrc Settings
source ~/.vimrc 

" General Settings
set nooldundo

" Idea Vim Plugins
Plug 'machakann/vim-highlightedyank'
Plug 'tpope/vim-commentary'

" IdeaVim Plugins: Enabling
set surround
set highlightedyank

" Key Mappings
map Q gq
```

## Global Vim config

And here’s the global `.vimrc` (sourced in the first line above):

```vim
let mapleader = ','

" General Settings
set scrolloff=10
set number
set relativenumber
set showmode
set showcmd
set showmatch
set visualbell

" Search Improvements
set ignorecase
set smartcase
set incsearch
set hlsearch

" Other Configurations
set nocompatible
set cursorline
set cursorcolumn

filetype on
filetype plugin on
filetype indent on

syntax on

set wildmenu
set wildmode=list:longest
set wildignore=*.docx,*.jpg,*.png,*.gif,*.pdf,*.pyc,*.exe,*.flv,*.img,*.xlsx

nnoremap <esc> :noh<return><esc>
```

That’s it. Nothing fancy, but enough to keep me productive across tools.

> ℹ️ If you enjoyed the article you might enjoy following me on [Bluesky](https://bsky.app/profile/marcellogalhardo.dev). ℹ️
