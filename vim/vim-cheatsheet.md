# Vim Cheat Sheet

## Modes

- `i` → insert
- `Esc` → normal mode
- `v` → visual mode
- `V` → visual line
- `Ctrl-v` → block visual

---

## File Ops

- `vim file`
- `:w` save
- `:q` quit
- `:wq` save & quit
- `:q!` force quit

---

## Movement

- `h j k l` → move
- `w` next word
- `b` back word
- `e` end word
- `0` start line
- `$` end line
- `gg` top
- `G` bottom

---

## Editing

- `x` delete char
- `dd` delete line
- `dw` delete word
- `D` delete to end
- `u` undo
- `Ctrl-r` redo

---

## Copy/Paste

- `yy` copy line
- `yw` copy word
- `p` paste below
- `P` paste above

---

## Search

- `/text` search
- `n` next
- `N` previous
- `:noh` clear highlight

---

## Replace

- `:%s/old/new/g` replace all
- `:%s/old/new/gc` confirm replace

---

## Power commands

- `ci"` change inside quotes
- `ci(` change inside parentheses
- `ggVG` select all
- `ddp` move line down
- `yyP` duplicate line
