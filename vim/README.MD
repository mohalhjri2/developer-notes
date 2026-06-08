# Vim — The Complete Field Guide

> "Vim is not an editor with modal editing. It's a language for editing text."

---

## Table of Contents

1. [The Vim Language](#the-vim-language)
2. [Motions](#motions)
3. [Operators](#operators)
4. [Text Objects](#text-objects)
5. [Registers](#registers)
6. [Macros](#macros)
7. [Search & Replace](#search--replace)
8. [Windows, Tabs & Splits](#windows-tabs--splits)
9. [Marks & Jumps](#marks--jumps)
10. [Visual Mode](#visual-mode)
11. [Command-line Mode](#command-line-mode)
12. [Buffers](#buffers)
13. [Folding](#folding)
14. [The vimrc](#the-vimrc)
15. [Plugins Worth Knowing](#plugins-worth-knowing)
16. [Pro Tips & Gotchas](#pro-tips--gotchas)

---

## The Vim Language

Vim commands compose like a language: `[count] [operator] [motion/text-object]`

```
d2w     → delete 2 words
ci"     → change inside quotes
y3j     → yank 3 lines down
>ap     → indent a paragraph
gUiw    → uppercase the current word
```

**Modes:**
| Mode | Key | Purpose |
|------|-----|---------|
| Normal | `Esc` / `Ctrl-[` | Navigate and edit |
| Insert | `i`, `a`, `o`, `I`, `A`, `O` | Type text |
| Visual | `v`, `V`, `Ctrl-v` | Select text |
| Command | `:` | Execute commands |
| Replace | `R` | Overwrite text |
| Ex | `Q` | Multi-line commands |

---

## Motions

### Character & Word
```
h j k l         ← ↓ ↑ →
w  W            next word (W = WORD, skips punctuation)
b  B            back word
e  E            end of word
ge gE           end of previous word
0  ^  $         start of line / first non-blank / end of line
g0 g^ g$        same but for wrapped lines
```

### Paragraph & Document
```
{  }            previous / next empty line (paragraph)
gg  G           top / bottom of file
50G  :50        go to line 50
H  M  L         top / middle / bottom of screen
```

### Search-based Motion
```
f{c}  F{c}      find char forward / backward (inclusive)
t{c}  T{c}      to char forward / backward (exclusive)
;  ,             repeat f/t forward / backward
*  #             search word under cursor forward / backward
/pattern        search forward
?pattern        search backward
n  N             next / previous match
```

### Code Navigation
```
%               jump to matching bracket/paren/brace
[{  ]}          jump to enclosing { / }
[(  ])          jump to enclosing ( / )
[m  ]m          jump to method start / end (Java/C++)
gd  gD          go to local / global definition
gf              go to file under cursor
```

---

## Operators

Operators wait for a motion or text object:

```
d    delete (cut)
y    yank (copy)
c    change (delete + insert)
>    indent right
<    indent left
=    auto-indent
gU   uppercase
gu   lowercase
g~   toggle case
!    filter through external command
```

**Double the operator to act on the current line:**
```
dd   delete line
yy   yank line
cc   change line
>>   indent line
==   auto-indent line
gUU  uppercase line
```

---

## Text Objects

Format: `[i|a][object]` — inner vs around (includes surrounding whitespace/delimiters)

```
iw  aw    inner/around word
iW  aW    inner/around WORD
is  as    inner/around sentence
ip  ap    inner/around paragraph
i"  a"    inner/around double quotes
i'  a'    inner/around single quotes
i`  a`    inner/around backticks
i(  a(    inner/around parentheses (also ib ab)
i[  a[    inner/around brackets
i{  a{    inner/around braces (also iB aB)
i<  a<    inner/around angle brackets
it  at    inner/around HTML/XML tag
```

**Power combos:**
```
ci"     change text inside double quotes
da{     delete a block including braces
yi(     yank what's inside parentheses
=ip     auto-indent paragraph
```

---

## Registers

```
"ayy         yank line into register a
"ap          paste from register a
"+y          yank to system clipboard (+ register)
"+p          paste from system clipboard
"0p          paste last yanked text (not deleted)
"_dd         delete to black hole register (no overwrite)
```

**Special registers:**
| Register | Contents |
|----------|----------|
| `"` | Default (last d/y/c) |
| `0` | Last yank |
| `1`–`9` | Delete history (rolling) |
| `+` | System clipboard |
| `*` | X11 selection / macOS clipboard |
| `_` | Black hole |
| `/` | Last search pattern |
| `:` | Last command |
| `.` | Last inserted text |
| `%` | Current filename |

**View all registers:**
```
:reg
:reg a b c      view specific registers
```

---

## Macros

```
qq              start recording into register q
[commands]      do your editing
q               stop recording

@q              replay macro q
@@              replay last macro
5@q             replay macro q 5 times
```

**Edit a macro:**
```
"qp             paste macro q as text
[edit the text]
"qyy            yank back into register q
```

**Recursive macro (runs until it fails):**
```
qq              start recording
[commands + @q at end]
q               stop
@q              runs until it hits an error (use for bulk edits!)
```

---

## Search & Replace

### Basic
```
/pattern        search forward
?pattern        search backward
n / N           next / previous
*               search word under cursor
:noh            clear highlights
```

### Substitution
```
:s/old/new/         replace first on current line
:s/old/new/g        replace all on current line
:%s/old/new/g       replace all in file
:%s/old/new/gc      replace all with confirmation
:5,10s/old/new/g    replace in lines 5–10
```

**Flags:**
```
g    all occurrences on line
i    case-insensitive
I    case-sensitive (override ignorecase)
c    confirm each
e    don't error if no match
n    count matches (don't replace)
```

### Advanced Patterns
```
:%s/\bfoo\b/bar/g          whole-word match
:%s/\(foo\)/[\1]/g         backreference
:%s/^/    /                indent every line
:%s/\s\+$//                strip trailing whitespace
:%s/\n/,/                  join lines with comma
:%g/pattern/d              delete lines matching pattern
:%g!/pattern/d             delete lines NOT matching pattern
```

---

## Windows, Tabs & Splits

### Splits
```
:sp  / Ctrl-w s     horizontal split
:vsp / Ctrl-w v     vertical split
Ctrl-w w            cycle windows
Ctrl-w h/j/k/l      navigate splits
Ctrl-w H/J/K/L      move split to edge
Ctrl-w =            equalize sizes
Ctrl-w _            maximize height
Ctrl-w |            maximize width
Ctrl-w +/-          resize height
Ctrl-w </> / Ctrl-w ,/.   resize width (Neovim)
:q                  close current window
:only               close all other windows
```

### Tabs
```
:tabnew [file]      new tab
:tabn / gt          next tab
:tabp / gT          previous tab
:tabclose           close tab
:tabonly            close other tabs
[N]gt               go to tab N
```

---

## Marks & Jumps

```
ma              set mark 'a' (lowercase = file-local)
mA              set mark 'A' (uppercase = global, across files)
`a              jump to exact position of mark a
'a              jump to line of mark a
``              jump to last jump position
`.              jump to last change position
'^              jump to last insert position
Ctrl-o          jump back in jump list
Ctrl-i          jump forward in jump list
:marks          view all marks
```

---

## Visual Mode

```
v               character-wise visual
V               line-wise visual
Ctrl-v          block visual (column editing)

In visual mode:
o               move to other end of selection
O               move to other corner (block mode)

Block editing:
Ctrl-v → select column → I → type → Esc    insert before all lines
Ctrl-v → select column → A → type → Esc    append after all lines
Ctrl-v → select column → c → type → Esc    change all
Ctrl-v → select column → r{char}            replace all with char
```

---

## Command-line Mode

```
:!command           run shell command
:r !command         insert output of command
:w !sudo tee %      save file with sudo
:.!date             replace current line with date output
:10,20!sort         sort lines 10–20

:earlier 5m         undo to 5 minutes ago
:later 1h           redo to 1 hour ahead

:set list           show hidden chars (tabs, EOL)
:set paste          disable auto-indent for pasting
:retab              convert tabs to spaces (or vice versa)
:sort               sort lines
:sort!              sort lines reversed
:sort u             sort and remove duplicates
```

---

## Buffers

```
:ls / :buffers      list all buffers
:b N                switch to buffer N
:b name             switch to buffer by partial name (tab-completes)
:bn / :bp           next / previous buffer
:bd                 delete (close) buffer
:bufdo %s/foo/bar/g run command on all buffers
:args **/*.js       load all JS files into args list
:argdo %s/foo/bar/gc run on all arg files
```

---

## Folding

```
zf{motion}      create fold (manual)
zo / zO         open fold / open all folds recursively
zc / zC         close fold / close all folds recursively
za / zA         toggle fold / toggle recursively
zr / zR         reduce folding (open one level / all)
zm / zM         more folding (close one level / all)
zd              delete fold
[z / ]z         move to start / end of current fold
zj / zk         move to next / previous fold
```

**In vimrc:**
```vim
set foldmethod=indent   " fold by indentation
set foldmethod=syntax   " fold by syntax (language-aware)
set foldmethod=marker   " fold between {{{ and }}}
set foldlevelstart=99   " open all folds on file open
```

---

## The vimrc

A battle-tested starting point:

```vim
" ============================================
" CORE SETTINGS
" ============================================
set nocompatible                " Disable Vi compatibility
filetype plugin indent on       " Enable filetype detection
syntax enable

" Encoding
set encoding=utf-8
set fileencoding=utf-8

" Display
set number relativenumber       " Hybrid line numbers
set cursorline                  " Highlight current line
set scrolloff=8                 " Keep 8 lines above/below cursor
set sidescrolloff=8
set signcolumn=yes              " Always show sign column
set colorcolumn=80,120          " Column rulers
set wrap linebreak              " Soft wrap at word boundaries
set showmatch                   " Highlight matching brackets
set laststatus=2                " Always show status line
set showcmd                     " Show partial commands
set wildmenu                    " Better command-line completion
set wildmode=longest:full,full

" Search
set hlsearch incsearch          " Highlight + incremental search
set ignorecase smartcase        " Smart case sensitivity

" Indentation
set expandtab                   " Tabs to spaces
set tabstop=4 shiftwidth=4      " 4-space tabs
set autoindent smartindent

" Performance
set lazyredraw                  " Don't redraw during macros
set updatetime=250              " Faster CursorHold (for plugins)
set timeoutlen=500              " Faster key sequence timeout

" Files
set nobackup nowritebackup      " No backup files (use version control)
set noswapfile                  " No swap files
set undofile                    " Persistent undo
set undodir=~/.vim/undodir      " Undo directory

" ============================================
" KEY MAPPINGS
" ============================================
let mapleader = " "             " Space as leader

" Escape alternatives
inoremap jk <Esc>
inoremap kj <Esc>

" Save and quit
nnoremap <leader>w :w<CR>
nnoremap <leader>q :q<CR>
nnoremap <leader>x :x<CR>

" Clear search highlight
nnoremap <leader>h :noh<CR>

" Navigate splits
nnoremap <C-h> <C-w>h
nnoremap <C-j> <C-w>j
nnoremap <C-k> <C-w>k
nnoremap <C-l> <C-w>l

" Resize splits
nnoremap <M-Up>    :resize +2<CR>
nnoremap <M-Down>  :resize -2<CR>
nnoremap <M-Left>  :vertical resize -2<CR>
nnoremap <M-Right> :vertical resize +2<CR>

" Buffer navigation
nnoremap <leader>bn :bnext<CR>
nnoremap <leader>bp :bprev<CR>
nnoremap <leader>bd :bd<CR>

" Move lines up/down
nnoremap <A-j> :m .+1<CR>==
nnoremap <A-k> :m .-2<CR>==
vnoremap <A-j> :m '>+1<CR>gv=gv
vnoremap <A-k> :m '<-2<CR>gv=gv

" Keep cursor centered when searching
nnoremap n nzzzv
nnoremap N Nzzzv

" Keep cursor in place when joining lines
nnoremap J mzJ`z

" Better undo breakpoints in insert mode
inoremap , ,<C-g>u
inoremap . .<C-g>u
inoremap ! !<C-g>u
inoremap ? ?<C-g>u

" Indent and stay in visual mode
vnoremap < <gv
vnoremap > >gv

" Yank to end of line (consistent with D, C)
nnoremap Y y$

" System clipboard
nnoremap <leader>y "+y
vnoremap <leader>y "+y
nnoremap <leader>Y "+Y
nnoremap <leader>p "+p
nnoremap <leader>P "+P

" Open file explorer
nnoremap <leader>e :Explore<CR>

" ============================================
" AUTOCMDS
" ============================================
augroup general
  autocmd!
  " Remove trailing whitespace on save
  autocmd BufWritePre * :%s/\s\+$//e
  " Return to last cursor position
  autocmd BufReadPost * if line("'\"") > 1 && line("'\"") <= line("$") | exe "normal! g'\"" | endif
  " Auto-reload files changed outside vim
  autocmd FocusGained,BufEnter * checktime
augroup END

" Language-specific
augroup filetypes
  autocmd!
  autocmd FileType yaml,json,javascript,typescript,html,css,scss
    \ setlocal tabstop=2 shiftwidth=2
  autocmd FileType go setlocal noexpandtab tabstop=4 shiftwidth=4
  autocmd FileType make setlocal noexpandtab
augroup END
```

---

## Plugins Worth Knowing

### Essential
| Plugin | Purpose |
|--------|---------|
| [vim-plug](https://github.com/junegunn/vim-plug) | Plugin manager |
| [fzf.vim](https://github.com/junegunn/fzf.vim) | Fuzzy file/buffer/grep search |
| [vim-fugitive](https://github.com/tpope/vim-fugitive) | Git integration |
| [vim-surround](https://github.com/tpope/vim-surround) | Change surrounding chars |
| [vim-commentary](https://github.com/tpope/vim-commentary) | Toggle comments (`gcc`) |
| [vim-repeat](https://github.com/tpope/vim-repeat) | Make `.` work with plugins |
| [vim-unimpaired](https://github.com/tpope/vim-unimpaired) | Paired `[` `]` mappings |
| [coc.nvim](https://github.com/neoclide/coc.nvim) | LSP + completion |
| [ale](https://github.com/dense-analysis/ale) | Async linting/fixing |

### Navigation
| Plugin | Purpose |
|--------|---------|
| [NERDTree](https://github.com/preservim/nerdtree) | File explorer |
| [vim-vinegar](https://github.com/tpope/vim-vinegar) | Better netrw (`-`) |
| [vim-easymotion](https://github.com/easymotion/vim-easymotion) | Hop anywhere |
| [vim-sneak](https://github.com/justinmk/vim-sneak) | 2-char search motion |

### vim-surround Cheatsheet
```
cs"'            change " to '  around cursor
cs'<q>          change ' to <q> tag
cst"            change tag to "
ds"             delete surrounding "
ysiw)           wrap word in ()
yss)            wrap entire line in ()
S)              wrap selection in () (visual mode)
```

---

## Pro Tips & Gotchas

### Performance
```vim
" Syntax highlighting is slow on large files
set synmaxcol=200       " only highlight first 200 chars per line
set regexpengine=1      " use old engine (faster for some patterns)
```

### The `.` command
The repeat command `.` is one of Vim's most powerful features. Design your edits with it in mind:
- `ciw` + type + `Esc` → move to next word → `.` repeats the change
- `A;<Esc>` on first line → `j.` to add semicolons to subsequent lines

### Global command patterns
```
:g/pattern/d            delete all matching lines
:g/pattern/normal @q    run macro on all matching lines
:g/^$/d                 delete blank lines
:g/TODO/copy $          copy all TODO lines to end of file
:v/pattern/d            delete all NON-matching lines (inverse global)
```

### Quickfix workflow
```
:grep -r 'pattern' .    search files (populates quickfix)
:copen                  open quickfix window
:cn / :cp               next / previous result
:cdo s/old/new/g        run substitution on all quickfix entries
```

### Spell checking
```
:set spell spelllang=en_us
]s / [s             next / previous misspelling
z=                  suggest corrections
zg                  add word to dictionary
zw                  mark word as wrong
```
