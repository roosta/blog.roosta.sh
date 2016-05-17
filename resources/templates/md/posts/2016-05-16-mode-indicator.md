{:title "Improving mode visibility in vim/vimode"
 :layout :post
 :tags  ["zsh" "vim" "terminal" "tmux"]
 :toc false}

## Introduction:
What I mean with the headline is, using various methods to improve mode visibility (normal, insert, replace) using terminal vim. Like gvim does by default by changing the cursor from line to block depending on mode is a good way of doing it and I'd like the same for terminal vim and/or zsh.

## Prerequisites

* [rxvt-unicode](http://software.schmorp.de/pkg/rxvt-unicode.html)
* [vim](http://www.vim.org/)  
* [zsh](http://www.zsh.org/)
* optional:
  * [tmux](https://tmux.github.io/)
  * [Pure Prompt](https://github.com/sindresorhus/pure)

All of these snippets are spesifcially for VTE compatible terminals (urxvt, st, xterm, gnome terminal ++). The same technique should apply for other terminal emulators, but you'll have to find other escape sequences. A good place to start would be [here.](http://vim.wikia.com/wiki/Change_cursor_shape_in_different_modes)

## Vim

Using vim control sequences we can change the cursor shape by passing along commands to the terminal.
```viml
let &t_SI = "\<Esc>[6 q"  " enter insert mode
let &t_SR = "\<Esc>[4 q"  " enter replace mode
let &t_EI = "\<Esc>[2 q"  " exit insert mode
```
This works fine until Tmux is introduced into the mix. Now how all these tools interact and how they handle escape sequences is somewhat convoluted but after a fair bit of searching I found a  [blessed article](http://blog.yjl.im/2014/12/passing-escape-codes-for-changing-font.html) detailing proper escape sequences for tmux, allowing our original sequences to pass through:

```viml
let &t_SI = "\<Esc>Ptmux;\<Esc>\<Esc>[6 q\<Esc>\\"
let &t_SR = "\<Esc>Ptmux;\<Esc>\<Esc>[4 q\<Esc>\\"
let &t_EI = "\<Esc>Ptmux;\<Esc>\<Esc>[2 q\<Esc>\\"
```

lastly, lets do a check in vim, and see if tmux is running.

```viml
" 1 or 0 -> blinking block
" 3 -> blinking underscore
" Recent versions of xterm (282 or above) also support
" 5 -> blinking vertical bar
" 6 -> solid vertical bar
" change cursor depending on mode (VTE compatible terminals running tmux)
if exists('$TMUX')
  let &t_SI = "\<Esc>Ptmux;\<Esc>\<Esc>[6 q\<Esc>\\"
  let &t_SR = "\<Esc>Ptmux;\<Esc>\<Esc>[4 q\<Esc>\\"
  let &t_EI = "\<Esc>Ptmux;\<Esc>\<Esc>[2 q\<Esc>\\"
else
" change cursor depending on mode (VTE compatible terminals)
  let &t_SI = "\<Esc>[6 q"
  let &t_SR = "\<Esc>[4 q"
  let &t_EI = "\<Esc>[2 q"
endif
```
Resulting cursor should look like this.
<hr>
![vim-result](/img/vim-result.gif)
<hr>

## Zsh:
Starting off I assume vi mode is being used (`bindkey -v`). If you use emacs mode (`bindkey -e`) feel free to skip this part.

What I ended up opting for here was changing the symbol on the prompt depending on mode rather than the cursor shape. Same method applies to cursor shape as well, but since the shell is built with a block cursor in mind there were to many cases of the cursor shape was more confusing than not. I will nevertheless include examples of both methods.

I use the excellent [pure prompt](https://github.com/sindresorhus/pure) and modified it to include a mode indicator:
<hr>
![zsh-result](/img/zsh-result.gif)
<hr>

First thing is we create this function which checks which keymap is currently active and assigns values to the `PROMPT` variable defined over:
```zsh
# Define mode prompts. Both turn red on non-zero exit code
PROMPT_SYMBOL_VIINS="%(?.%F{white}.%F{red})%f%F{magenta}%f "
PROMPT_SYMBOL_VICMD="%(?.%F{white}.%F{red})%f%F{magenta}%f "

# enable colors before setting prompt variable
autoload -U colors && colors
PROMPT=$PROMPT_SYMBOL_VIINS
RPROMPT="%(?.%j.%F{red}%?%f %j%f"

function zle-line-init () {
  # Make sure the terminal is in application mode, when zle is
  # active. Only then are the values from $terminfo valid.
  if (( ${+terminfo[smkx]} )); then
    printf '%s' "${terminfo[smkx]}"
  fi
  prompt_mode
}
function zle-line-finish () {
  # Make sure the terminal is in application mode, when zle is
  # active. Only then are the values from $terminfo valid.
  if (( ${+terminfo[rmkx]} )); then
    printf '%s' "${terminfo[rmkx]}"
  fi

  # return to block on command
  if [ -z ${TMUX+x} ]; then
     print -n -- "\E[2 q"
  else
     print -n -- "\EPtmux;\E\E[2 q\E\\"
  fi

}
function zle-keymap-select () {
  prompt_mode
}

# change cursor and or prompt based on prompt mode.
# big thanks to: http://blog.yjl.im/2014/12/passing-escape-codes-for-changing-font.html
function prompt_mode() {
  # change prompt in VTE compatible terminals
  case $KEYMAP in
    vicmd)

      # change to block cursor
      if [ -z ${TMUX+x} ]; then
        print -n -- "\E[2 q"
      else
        print -n -- "\EPtmux;\E\E[2 q\E\\"
      fi
      PROMPT=$PROMPT_SYMBOL_VICMD
      ;;
    viins|main)

      # change to line cursor
      if [ -z ${TMUX+x} ]; then
        print -n -- "\E[6 q"
      else
        print -n -- "\EPtmux;\E\E[6 q\E\\"
      fi
      PROMPT=$PROMPT_SYMBOL_VIINS
      ;;
  esac
  zle reset-prompt
  zle -R
}

zle -N zle-line-init
zle -N zle-line-finish
zle -N zle-keymap-select
```
