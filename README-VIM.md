FZF Vim integration
===================

Summary
-------

The Vim plugin of fzf provides two core functions, and `:FZF` command which is
the basic file selector command built on top of them.

1. **`fzf#run([spec dict])`**
    - Starts fzf inside Vim with the given spec
    - `:call fzf#run({'source': 'ls'})`
2. **`fzf#wrap([spec dict]) -> (dict)`**
    - Takes a spec for `fzf#run` and returns an extended version of it with
      additional options for addressing global preferences (`g:fzf_xxx`)
        - `:echo fzf#wrap({'source': 'ls'})`
    - We usually *wrap* a spec with `fzf#wrap` before passing it to `fzf#run`
        - `:call fzf#run(fzf#wrap({'source': 'ls'}))`
3. **`:FZF [fzf_options string] [path string]`**
    - Basic fuzzy file selector
    - A reference implementation for those who don't want to write VimScript
      to implement custom commands
    - If you're looking for more such commands, check out [fzf.vim](https://github.com/junegunn/fzf.vim) project.

The most important of all is `fzf#run`, but it would be easier to understand
the whole if we start off with `:FZF` command.

`:FZF[!]`
---------

```vim
" Look for files under current directory
:FZF

" Look for files under your home directory
:FZF ~

" With fzf command-line options
:FZF --reverse --info=inline /tmp

" Bang version starts fzf in fullscreen mode
:FZF!
```

Similarly to [ctrlp.vim](https://github.com/kien/ctrlp.vim), use enter key,
`CTRL-T`, `CTRL-X` or `CTRL-V` to open selected files in the current window,
in new tabs, in horizontal splits, or in vertical splits respectively.

Note that the environment variables `FZF_DEFAULT_COMMAND` and
`FZF_DEFAULT_OPTS` also apply here.

### Configuration

- `g:fzf_action`
    - Customizable extra key bindings for opening selected files in different ways
- `g:fzf_layout`
    - Determines the size and position of fzf window
- `g:fzf_colors`
    - Customizes fzf colors to match the current color scheme
- `g:fzf_history_dir`
    - Enables history feature
- `g:fzf_new_tab`
    - Opens files in new tab instead of current tab

#### Examples

```vim
" This is the default extra key bindings
let g:fzf_action = {
  \ 'ctrl-t': 'tab split',
  \ 'ctrl-x': 'split',
  \ 'ctrl-v': 'vsplit' }

" An action can be a reference to a function that processes selected lines
function! s:build_quickfix_list(lines)
  call setqflist(map(copy(a:lines), '{ "filename": v:val }'))
  copen
  cc
endfunction

let g:fzf_action = {
  \ 'ctrl-q': function('s:build_quickfix_list'),
  \ 'ctrl-t': 'tab split',
  \ 'ctrl-x': 'split',
  \ 'ctrl-v': 'vsplit' }

" Default fzf layout
" - down / up / left / right
let g:fzf_layout = { 'down': '~40%' }

" You can set up fzf window using a Vim command (Neovim or latest Vim 8 required)
let g:fzf_layout = { 'window': 'enew' }
let g:fzf_layout = { 'window': '-tabnew' }
let g:fzf_layout = { 'window': '10new' }

" Customize fzf colors to match your color scheme
" - fzf#wrap translates this to a set of `--color` options
let g:fzf_colors =
\ { 'fg':      ['fg', 'Normal'],
  \ 'bg':      ['bg', 'Normal'],
  \ 'hl':      ['fg', 'Comment'],
  \ 'fg+':     ['fg', 'CursorLine', 'CursorColumn', 'Normal'],
  \ 'bg+':     ['bg', 'CursorLine', 'CursorColumn'],
  \ 'hl+':     ['fg', 'Statement'],
  \ 'info':    ['fg', 'PreProc'],
  \ 'border':  ['fg', 'Ignore'],
  \ 'prompt':  ['fg', 'Conditional'],
  \ 'pointer': ['fg', 'Exception'],
  \ 'marker':  ['fg', 'Keyword'],
  \ 'spinner': ['fg', 'Label'],
  \ 'header':  ['fg', 'Comment'] }

" Enable per-command history
" - History files will be stored in the specified directory
" - When set, CTRL-N and CTRL-P will be bound to 'next-history' and
"   'previous-history' instead of 'down' and 'up'.
let g:fzf_history_dir = '~/.local/share/fzf-history'

" Open in new tab.
let g:fzf_new_tab = 1
```

`fzf#run`
---------

`fzf#run()` function is the core of Vim integration. It takes a single
dictionary argument, *a spec*, and starts fzf process accordingly. At the very
least, specify `sink` option to tell what it should do with the selected
entry.

```vim
call fzf#run({'sink': 'e'})
```

We haven't specified the `source`, so this is equivalent to starting fzf on
command line without standard input pipe; fzf will use find command (or
`$FZF_DEFAULT_COMMAND` if defined) to list the files under the current
directory. When you select one, it will open it with the sink, `:e` command.
If you want to open it in a new tab, you can pass `:tabedit` command instead
as the sink.

```vim
call fzf#run({'sink': 'tabedit'})
```

Instead of using the default find command, you can use any shell command as
the source. The following example will list the files managed by git. It's
equivalent to running `git ls-files | fzf` on shell.

```vim
call fzf#run({'source': 'git ls-files', 'sink': 'e'})
```

fzf options can be specified as `options` entry in spec dictionary.

```vim
call fzf#run({'sink': 'tabedit', 'options': '--multi --reverse'})
```

You can also pass a layout option if you don't want fzf window to take up the
entire screen.

```vim
" up / down / left / right / window are allowed
call fzf#run({'source': 'git ls-files', 'sink': 'e', 'left': '40%'})
call fzf#run({'source': 'git ls-files', 'sink': 'e', 'window': '30vnew'})
```

`source` doesn't have to be an external shell command, you can pass a Vim
array as the source. In the next example, we pass the names of color
schemes as the source to implement a color scheme selector.

```vim
call fzf#run({'source': map(split(globpath(&rtp, 'colors/*.vim')),
            \               'fnamemodify(v:val, ":t:r")'),
            \ 'sink': 'colo', 'left': '25%'})
```

The following table summarizes the available options.

| Option name                | Type          | Description                                                           |
| -------------------------- | ------------- | ----------------------------------------------------------------      |
| `source`                   | string        | External command to generate input to fzf (e.g. `find .`)             |
| `source`                   | list          | Vim list as input to fzf                                              |
| `sink`                     | string        | Vim command to handle the selected item (e.g. `e`, `tabe`)            |
| `sink`                     | funcref       | Reference to function to process each selected item                   |
| `sink*`                    | funcref       | Similar to `sink`, but takes the list of output lines at once         |
| `options`                  | string/list   | Options to fzf                                                        |
| `dir`                      | string        | Working directory                                                     |
| `up`/`down`/`left`/`right` | number/string | (Layout) Window position and size (e.g. `20`, `50%`)                  |
| `window` (Vim 8 / Neovim)  | string        | (Layout) Command to open fzf window (e.g. `vertical aboveleft 30new`) |

`options` entry can be either a string or a list. For simple cases, string
should suffice, but prefer to use list type to avoid escaping issues.

```vim
call fzf#run({'options': '--reverse --prompt "C:\\Program Files\\"'})
call fzf#run({'options': ['--reverse', '--prompt', 'C:\Program Files\']})
```

`fzf#wrap`
----------

We have seen that several aspects of `:FZF` command can be configured with
a set of global option variables; different ways to open files
(`g:fzf_action`), window position and size (`g:fzf_layout`), color palette
(`g:fzf_colors`), etc.

So how can we make our custom `fzf#run` calls also respect those variables?
Simply by *"wrapping"* the spec dictionary with `fzf#wrap` before passing it
to `fzf#run`.

- **`fzf#wrap([name string], [spec dict], [fullscreen bool]) -> (dict)`**
    - All arguments are optional. Usually we only need to pass a spec dictionary.
    - `name` is for managing history files. It is ignored if
      `g:fzf_history_dir` is not defined.
    - `fullscreen` can be either `0` or `1` (default: 0).

`fzf#wrap` takes a spec and returns an extended version of it (also
a dictionary) with additional options for addressing global preferences. You
can examine the return value of it like so:

```vim
echo fzf#wrap({'source': 'ls'})
```

After we *"wrap"* our spec, we pass it to `fzf#run`.

```vim
call fzf#run(fzf#wrap({'source': 'ls'}))
```

Now it supports `CTRL-T`, `CTRL-V`, and `CTRL-X` key bindings and it opens fzf
window according to `g:fzf_layout` setting.

To make it easier to use, let's define `LS` command.

```vim
command! LS call fzf#run(fzf#wrap({'source': 'ls'}))
```

Type `:LS` and see how it works.

We would like to make `:LS!` (bang version) open fzf in fullscreen, just like
`:FZF!`. Add `-bang` to command definition, and use `<bang>` value to set
the last `fullscreen` argument of `fzf#wrap` (see `:help <bang>`).

```vim
" On :LS!, <bang> evaluates to '!', and '!0' becomes 1
command! -bang LS call fzf#run(fzf#wrap({'source': 'ls'}, <bang>0))
```

Our `:LS` command will be much more useful if we can pass a directory argument
to it, so that something like `:LS /tmp` is possible.

```vim
command! -bang -complete=dir -nargs=* LS
    \ call fzf#run(fzf#wrap({'source': 'ls', 'dir': <q-args>}, <bang>0))
```

Lastly, if you have enabled `g:fzf_history_dir`, you might want to assign
a unique name to our command and pass it as the first argument to `fzf#wrap`.

```vim
" The query history for this command will be stored as 'ls' inside g:fzf_history_dir.
" The name is ignored if g:fzf_history_dir is not defined.
command! -bang -complete=dir -nargs=* LS
    \ call fzf#run(fzf#wrap('ls', {'source': 'ls', 'dir': <q-args>}, <bang>0))
```

Tips
----

### fzf inside terminal buffer

The latest versions of Vim and Neovim include builtin terminal emulator
(`:terminal`) and fzf will start in a terminal buffer in the following cases:

- On Neovim
- On GVim
- On Terminal Vim with a non-default layout
    - `call fzf#run({'left': '30%'})` or `let g:fzf_layout = {'left': '30%'}`

#### Starting fzf in Neovim floating window

```vim
" Using floating windows of Neovim to start fzf
if has('nvim')
  let $FZF_DEFAULT_OPTS .= ' --border --margin=0,2'

  function! FloatingFZF()
    let width = float2nr(&columns * 0.9)
    let height = float2nr(&lines * 0.6)
    let opts = { 'relative': 'editor',
               \ 'row': (&lines - height) / 2,
               \ 'col': (&columns - width) / 2,
               \ 'width': width,
               \ 'height': height }

    let win = nvim_open_win(nvim_create_buf(v:false, v:true), v:true, opts)
    call setwinvar(win, '&winhighlight', 'NormalFloat:Normal')
  endfunction

  let g:fzf_layout = { 'window': 'call FloatingFZF()' }
endif

```

#### Hide statusline

When fzf starts in a terminal buffer, the file type of the buffer is set to
`fzf`. So you can set up `FileType fzf` autocmd to customize the settings of
the window.

For example, if you use the default layout (`{'down': '~40%'}`) on Neovim, you
might want to temporarily disable the statusline for a cleaner look.

```vim
if has('nvim') && !exists('g:fzf_layout')
  autocmd! FileType fzf
  autocmd  FileType fzf set laststatus=0 noshowmode noruler
    \| autocmd BufLeave <buffer> set laststatus=2 showmode ruler
endif
```

[License](LICENSE)
------------------

The MIT License (MIT)

Copyright (c) 2019 Junegunn Choi
