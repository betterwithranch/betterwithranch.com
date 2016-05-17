---
title: Opening a new tmux window in the current directory
date: 2016-05-17 18:51 UTC
tags: tmux
---

If you are using [tmux](https://tmux.github.io/), you know about splitting windows and opening new windows.
I do that all the time to get a new terminal to do things like run a dependency service or tail a log.  Until recently, this
involved having the terminal open in the initial directory where the session was started and then
having to navigate to the directory of the project where I am working.

Well, it turns out that tmux supports opening a window where the terminal's directory
is the same as the directory of the current pane and it's easy to do!  Simply use
the `-c` option to set the working directory and the `{pane_current_path}` variable to use
the current panes path for the value.

For example, to open a new terminal in a vertical split:

```bash
split-window -c '#{pane_current_path}'
```

I've configured my [`~/.tmux.conf`](https://github.com/betterwithranch/dotfiles/blob/master/.tmux.conf)
file to change the split-window and new-window aliases to use this because I almost always want this
behavior.

```bash
# In ~/.tmux.conf

# Note that the binding to `"` needs to be escaped.
bind '"' split-window -c '#{pane_current_path}'
bind % split-window -h -c '#{pane_current_path}'
bind c new-window -c '#{pane_current_path}'
```

Remember, if you change your tmux configuration and want to reload the current server to use the
configuration run the following command:

```bash
source-file ~/.tmux.conf
```
