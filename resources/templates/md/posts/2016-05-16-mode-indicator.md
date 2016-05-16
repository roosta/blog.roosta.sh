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

## Zsh:
Starting off I assume vi mode is being used ``` bindkey -v ```. If you use emacs mode ```bindkey -e``` feel free to skip this part.

What I ended up opting for here was changing the symbol on the prompt depending on mode rather than the cursor shape. I did figure out how to change cursor shape as well, but since the shell is built with a block cursor in mind there were to many cases of the cursor shape was more confusing than not. I will nevertheless include examples of both methods.

I use the excellent [pure prompt](https://github.com/sindresorhus/pure) and modified it to include a mode indicator.
