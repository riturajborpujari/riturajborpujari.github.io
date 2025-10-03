---
title               : "Terminal Tools, Shell Aliases, and directory bookmarks"
description         : >
                      Using Unix tools in the terminal to build simple 
                      directory bookmarks. Allows simple navigation above cd
                      program without hurting your ability to work in a machine
                      without
date                : 2024-04-28
template            : post.html
---

I like using terminal tools and programs. They are so handy that you can't ignore once you get used to them. There are many of them at your disposal, each doing something which is unique in their own.

They do their own thing best, and they work with each other. Take a look at [Unix Philosophy](https://en.wikipedia.org/wiki/Unix_philosophy) to understand the guiding principles that these tools follow to make them so easy to work with!

As I begin using these tools and the many ways in which they allow [composability](https://en.wikipedia.org/wiki/Function_composition_(computer_science)), I found myself using such composed commands and even scripts. Here's one that allows me to move to frequently used directories without typing the path everytime, from any directory I'm in!

## Directory bookmarks
We use `cd` a lot in terminal. It allows us to change to the directory we want to work on. As I started using [fzf](https://wiki.archlinux.org/title/fzf), finding a specific subdirectory became much more easier than before. I can type in `cd $(fzf)` and it will open up a search window where I can [fuzzy-search](https://en.wikipedia.org/wiki/Approximate_string_matching) for a specific directory. I can write a shell alias like `alias cdf="cd $(fzf)"` and then I can just type in `cdf` and press `return` to activate the command. Its great!

But still, when I'm deep in a particular directory and I wanted to move to another directory which is not a sub dir this gives me a problem. Because the required directory that I want to move into is not a subdirectory, I can't just `cdf` any more (atleast not from the current dir). I have to move up the directory structure untill the required directory becomes a subdir to the present working directory. This is inadequate!

What if I can store certain directories that I frequently `cd` into, somewhere in a file, and then use `fzf` to match and `cd` into whichever one I want. That would be awesome!

That's what I built with **`dmk` (directory bookmark)**. Its a set of two shell aliases to help me navigate to any directory with ease. From anywhere!!!

### Marking directories as bookmarks
First things first, I need a way to mark certain directories as important so that I can come back to them when needed. For this, I just need to save the directory path into a file. 

If I run the command `pwd >> ~/.dmks`, the present working directory path gets appended to a file called `.dmks` in my home directory `~`. Here, `pwd` command prints the present working directory to terminal, and `>>` causes that output to get appended to the filename provided after.

For simpler frequent usage, I created an alias `alias dmk="pwd >> ~/.dmks"`. I added this command into my shell init rc file `~/.bashrc` and the alias is available from any shell window I open next.

### Moving into a bookmarked directory
Okay, now that we have a way of marking directories by saving their path into a file, its time to use `fzf` to move into one.

For this I run the command `cd $(cat ~/.dmks | fzf)`. This moves into the directory which is given by the [sub-shell command](https://en.wikiversity.org/wiki/Bash_programming/Subshells) `cat ~/.dmks | fzf`. Whatever is the output of this sub-shell command gets used by `cd` and it will cause the change in working directory. It's key to note that, `cat ~/.dmks` prints the content of the file `~/.dmks` into the shell and because of the pipe operator `|` it gets passed on to `fzf`. This file has all of our marked directories which we marked with `dmk` and this list is passed on to FZF. FZF then shows an interactive search window, from which we can search the required directory from the list of marked directories and select the one we need.

Again, to help simplify running this command regularly, I have created a shell alias for this too! `alias cdmk="cd \$(cat ~/.dmks | fzf)"` creates an alias and I write this command in the shell rc file `~/.bashrc` to make it available in any new shell window as well.

Note the escape character `\` is required before `$` when writing into the shell rc file! Otherwise, the shell will execute the command itself when initializing the alias! We don't want that! We want us to be able to use the alias, and when we do that, we need the command to run!

That is that, and I get directory bookmark system available in any of my shell window!!! `dmk` and `cdmk`!!!

P.S. here is the [link to the gist containing the aliases](https://gist.github.com/riturajborpujari/096d8b704157a7ad3eb5f8237ba05b52).

Cheers!
