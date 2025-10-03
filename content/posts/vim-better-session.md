---
title       : "Better sessions with VIM"
description : Vim sessions are great. Can we make it better?
date        : 2025-07-23
template    : post.html
---

Vim sessions are great! They give me power to jump back into whatever I was
working on earlier with ease. Among other things, all my open buffers are
restored, and the `.viminfo` file keeps track of my `:marks`, `:jumps` and
`:registers` already. So it feels like I am actually continuing from where I
left off.

But, I like to work on multiple things. Small side-projects, or POCs, or an idea
I've been thinking about implementing for so long. They all coexist in my system
as partially completed or just started code. Also, I don't keep notes about code
project ideas. I start it off as a project itself.

Now this is where Vim needs a custom flow for me. I can easily create and
maintain different sessions for my different projects. But the `.viminfo` file
is shared among them. So the `:marks` are shared which is a pain. To help me with this, I created my custom Session management code and that's
what BetterSessions is!

BetterSessions is not anything new. It just wraps vim sessions along with
custom viminfo together. Here's how it happens

For storing sessions,
1. Store session normally with `:mksesion! <session_name>.vim`
2. Write the current `.viminfo` with `:wviminfo! <session_name>.viminfo`

For loading sessions, we just do the same thing in reverse
1. Load session normally with `:source <session_name>.vim`
2. Read `viminfo` with `:rviminfo! <session_name>.viminfo`

Voila! Now you have separate viminfo for each session you store, and thus your
marks, registers, jumps etc are restored when you load the session

Cleaning up, I used `$HOME/.local/state/vim/better-sessions` to store the
session and viminfo files. I also added commands `:SessionLoad <session_name>`
and `:SessionStore <session_name>` to help me add keybinds for this.
Just for better usability I also added completion options for the commands, so
typing `:SessionLoad <TAB>` will show you the list of sessions available under
better-sessions.

You can find my code for reference at
[better-session](https://github.com/riturajborpujari/better-session.vim)

Cheers
