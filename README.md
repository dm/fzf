fzf - Fuzzy finder for your shell
=================================

fzf is a general-purpose fuzzy finder for your shell.

![](https://raw.github.com/junegunn/i/master/fzf.gif)

It was heavily inspired by [ctrlp.vim](https://github.com/kien/ctrlp.vim) and
the likes.

Requirements
------------

fzf requires Ruby (>= 1.8.5).

Installation
------------

Clone this repository and run
[install](https://github.com/junegunn/fzf/blob/master/install) script.

```sh
git clone https://github.com/junegunn/fzf.git ~/.fzf
~/.fzf/install
```

The script will setup:

- `fzf` executable
- Key bindings (`CTRL-T`, `CTRL-R`, and `ALT-C`) for bash and zsh
- Fuzzy auto-completion for bash

If you don't use bash or zsh, you have to manually place fzf executable in a
directory included in `$PATH`. Key bindings are not yet supported.

### Install as Vim plugin

Once you have cloned the repository, add the following line to your .vimrc.

```vim
set rtp+=~/.fzf
```

Or you may use any Vim plugin manager, such as
[vim-plug](https://github.com/junegunn/vim-plug).

Usage
-----

```
usage: fzf [options]

  Options
    -m, --multi          Enable multi-select
    -x, --extended       Extended-search mode
    -e, --extended-exact Extended-search mode (exact match)
    -q, --query=STR      Initial query
    -f, --filter=STR     Filter mode. Do not start interactive finder.
    -n, --nth=[-]N       Match only in the N-th token of the item
    -d, --delimiter=STR  Field delimiter regex for --nth (default: AWK-style)
    -s, --sort=MAX       Maximum number of matched items to sort (default: 1000)
    +s, --no-sort        Do not sort the result. Keep the sequence unchanged.
    -i                   Case-insensitive match (default: smart-case match)
    +i                   Case-sensitive match
    +c, --no-color       Disable colors
    +2, --no-256         Disable 256-color
        --black          Use black background
        --no-mouse       Disable mouse

  Environment variables
    FZF_DEFAULT_COMMAND  Default command to use when input is tty
    FZF_DEFAULT_OPTS     Defaults options. (e.g. "-x -m --sort 10000")
```

fzf will launch curses-based finder, read the list from STDIN, and write the
selected item to STDOUT.

```sh
find * -type f | fzf > selected
```

Without STDIN pipe, fzf will use find command to fetch the list of
files excluding hidden ones. (You can override the default command with
`FZF_DEFAULT_COMMAND`)

```sh
vim $(fzf)
```

If you want to preserve the exact sequence of the input, provide `--no-sort` (or
`+s`) option.

```sh
history | fzf +s
```

### Keys

Use CTRL-J and CTRL-K (or CTRL-N and CTRL-P) to change the selection, press
enter key to select the item. CTRL-C, CTRL-G, or ESC will terminate the finder.

The following readline key bindings should also work as expected.

- CTRL-A / CTRL-E
- CTRL-B / CTRL-F
- CTRL-W / CTRL-U / CTRL-Y
- ALT-B / ALT-F

If you enable multi-select mode with `-m` option, you can select multiple items
with TAB or Shift-TAB key.

You can also use mouse. Double-click on an item to select it or shift-click (or
ctrl-click) to select multiple items. Use mouse wheel to move the cursor up and
down.

### Extended-search mode

With `-x` or `--extended` option, fzf will start in "extended-search mode".

In this mode, you can specify multiple patterns delimited by spaces,
such as: `^music .mp3$ sbtrkt !rmx`

| Token    | Description                      | Match type           |
| -------- | -------------------------------- | -------------------- |
| `^music` | Items that start with `music`    | prefix-exact-match   |
| `.mp3$`  | Items that end with `.mp3`       | suffix-exact-match   |
| `sbtrkt` | Items that match `sbtrkt`        | fuzzy-match          |
| `!rmx`   | Items that do not match `rmx`    | inverse-fuzzy-match  |
| `'wild`  | Items that include `wild`        | exact-match (quoted) |
| `!'fire` | Items that do not include `fire` | inverse-exact-match  |

If you don't need fuzzy matching and do not wish to "quote" every word, start
fzf with `-e` or `--extended-exact` option.

Useful examples
---------------

```sh
# vimf - Open selected file in Vim
vimf() {
  local file
  file=$(fzf --query="$1") && vim "$file"
}

# fd - cd to selected directory
fd() {
  local dir
  dir=$(find ${1:-*} -path '*/\.*' -prune \
                  -o -type d -print 2> /dev/null | fzf +m) &&
  cd "$dir"
}

# fda - including hidden directories
fda() {
  local dir
  dir=$(find ${1:-.} -type d 2> /dev/null | fzf +m) && cd "$dir"
}

# fh - repeat history
fh() {
  eval $(history | fzf +s | sed 's/ *[0-9]* *//')
}

# fkill - kill process
fkill() {
  ps -ef | sed 1d | fzf -m | awk '{print $2}' | xargs kill -${1:-9}
}

# fbr - checkout git branch
fbr() {
  local branches branch
  branches=$(git branch) &&
  branch=$(echo "$branches" | fzf +s +m) &&
  git checkout $(echo "$branch" | sed "s/.* //")
}

# fco - checkout git commit
fco() {
  local commits commit
  commits=$(git log --pretty=oneline --abbrev-commit --reverse) &&
  commit=$(echo "$commits" | fzf +s +m -e) &&
  git checkout $(echo "$commit" | sed "s/ .*//")
}

# ftags - search ctags
ftags() {
  local line
  [ -e tags ] &&
    line=$(grep -v "^!" tags | cut -f1-3 | cut -c1-80 | fzf --nth=1) &&
    $EDITOR $(cut -f2 <<< "$line")
}

# fq1 [QUERY]
# - Immediately select the file when there's only one match.
#   If not, start the fuzzy finder as usual.
fq1() {
  local lines
  lines=$(fzf --filter="$1" --no-sort)
  if [ -z "$lines" ]; then
    return 1
  elif [ $(wc -l <<< "$lines") -eq 1 ]; then
    echo "$lines"
  else
    echo "$lines" | fzf --query="$1"
  fi
}

# fe [QUERY]
# - Open the selected file with the default editor
#   (Bypass fuzzy finder when there's only one match)
fe() {
  local file
  file=$(fq1 "$1") && ${EDITOR:-vim} "$file"
}
```

Key bindings for command line
-----------------------------

The install script will setup the following key bindings.

### bash/zsh

- `CTRL-T` - Paste the selected file path(s) into the command line
- `CTRL-R` - Paste the selected command from history into the command line
- `ALT-C` - cd into the selected directory

If you're on a tmux session, `CTRL-T` will launch fzf in a new split-window. You
may disable this tmux integration by setting `FZF_TMUX` to 0, or change the
height of the window with `FZF_TMUX_HEIGHT` (e.g. `20`, `50%`).

The source code can be found in `~/.fzf.bash` and in `~/.fzf.zsh`.

Auto-completion
---------------

Disclaimer: *Auto-completion feature is currently experimental, it can change
over time*

### bash

#### Files and directories

Fuzzy completion for files and directories can be triggered if the word before
the cursor ends with the trigger sequence which is by default `**`.

- `COMMAND [DIRECTORY/][FUZZY_PATTERN]**<TAB>`

```sh
# Files under current directory
# - You can select multiple items with TAB key
vim **<TAB>

# Files under parent directory
vim ../**<TAB>

# Files under parent directory that match `fzf`
vim ../fzf**<TAB>

# Files under your home directory
vim ~/**<TAB>


# Directories under current directory (single-selection)
cd **<TAB>

# Directories under ~/github that match `fzf`
cd ~/github/fzf**<TAB>
```

#### Process IDs

Fuzzy completion for PIDs is provided for kill command. In this case
there is no trigger sequence, just press tab key after kill command.

```sh
# Can select multiple processes with <TAB> or <Shift-TAB> keys
kill -9 <TAB>
```

#### Host names

For ssh and telnet commands, fuzzy completion for host names is provided. The
names are extracted from /etc/hosts and ~/.ssh/config.

```sh
ssh **<TAB>
telnet **<TAB>
```

#### Settings

```sh
# Use ~~ as the trigger sequence instead of the default **
export FZF_COMPLETION_TRIGGER='~~'

# Options to fzf command
export FZF_COMPLETION_OPTS='+c -x'
```

### zsh

TODO :smiley:

(Pull requests are appreciated.)

Usage as Vim plugin
-------------------

(fzf is a command-line utility, naturally it is only accessible in terminal Vim)

### `:FZF[!]`

If you have set up fzf for Vim, `:FZF` command will be added.

```vim
" Look for files under current directory
:FZF

" Look for files under your home directory
:FZF ~

" With options
:FZF --no-sort -m /tmp
```

Note that the environment variables `FZF_DEFAULT_COMMAND` and `FZF_DEFAULT_OPTS`
also apply here.

If you're on a tmux session, `:FZF` will launch fzf in a new split-window whose
height can be adjusted with `g:fzf_tmux_height` (default: '40%'). However, the
bang version (`:FZF!`) will always start in fullscreen.

### `fzf#run([options])`

For more advanced uses, you can call `fzf#run()` function which returns the list
of the selected items.

`fzf#run()` may take an options-dictionary:

| Option name | Type          | Description                                                         |
| ----------- | ------------- | ------------------------------------------------------------------- |
| `source`    | string        | External command to generate input to fzf (e.g. `find .`)           |
| `source`    | list          | Vim list as input to fzf                                            |
| `sink`      | string        | Vim command to handle the selected item (e.g. `e`, `tabe`)          |
| `sink`      | funcref       | Reference to function to process each selected item                 |
| `options`   | string        | Options to fzf                                                      |
| `dir`       | string        | Working directory                                                   |
| `tmux`      | number/string | Use tmux split if possible with the given height (e.g. `20`, `50%`) |

#### Examples

If `sink` option is not given, `fzf#run` will simply return the list.

```vim
let items = fzf#run({ 'options': '-m +c', 'dir': '~', 'source': 'ls' })
```

But if `sink` is given as a string, the command will be executed for each
selected item.

```vim
" Each selected item will be opened in a new tab
let items = fzf#run({ 'sink': 'tabe', 'options': '-m +c', 'dir': '~', 'source': 'ls' })
```

We can also use a Vim list as the source as follows:

```vim
" Choose a color scheme with fzf
nnoremap <silent> <Leader>C :call fzf#run({
\   'source':
\     map(split(globpath(&rtp, "colors/*.vim"), "\n"),
\         "substitute(fnamemodify(v:val, ':t'), '\\..\\{-}$', '', '')"),
\   'sink':    'colo',
\   'options': '+m',
\   'tmux':    15
\ })<CR>
```

`sink` option can be a function reference. The following example creates a
handy mapping that selects an open buffer.

```vim
" List of buffers
function! g:buflist()
  redir => ls
  silent ls
  redir END
  return split(ls, '\n')
endfunction

function! g:bufopen(e)
  execute 'buffer '. matchstr(a:e, '^[ 0-9]*')
endfunction

nnoremap <silent> <Leader><Enter> :call fzf#run({
\   'source':  g:buflist(),
\   'sink':    function('g:bufopen'),
\   'options': '+m +s',
\   'tmux':    15
\ })<CR>
```

Tips
----

### Rendering issues

If you have any rendering issues, check the followings:

1. Make sure `$TERM` is correctly set. fzf will use 256-color only if it
  contains `256` (e.g. `xterm-256color`)
2. If you're on screen or tmux, `$TERM` should be either `screen` or
  `screen-256color`
3. Some terminal emulators (e.g. mintty) have problem displaying default
  background color and make some text unable to read. In that case, try `--black`
  option. And if it solves your problem, I recommend including it in
  `FZF_DEFAULT_OPTS` for further convenience.
4. If you still have problem, try `--no-256` option or even `--no-color`.
5. Ruby 1.9 or above is required for correctly displaying unicode characters.

### Ranking algorithm

fzf sorts the result first by the length of the matched substring, then by the
length of the whole string. However it only does so when the number of matches
is less than the limit which is by default 1000, in order to avoid the cost of
sorting a large list and limit the response time of the query.

This limit can be adjusted with `-s` option, or with the environment variable
`FZF_DEFAULT_OPTS`.

```sh
export FZF_DEFAULT_OPTS="--sort 20000"
```

License
-------

MIT

Author
------

Junegunn Choi

