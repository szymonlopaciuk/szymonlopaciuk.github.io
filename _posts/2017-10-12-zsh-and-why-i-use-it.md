---
layout: post
title: "ZSH Tricks and Why I Use It"
date: 2017-10-13 23:26
---

Why use zsh in the first place, you might ask? That is a totally reasonable question. After all, bash gets preinstalled on (almost?) all Linux distributions and beyond, everyone knows it, it's tested and of established quality. Why change?

There are of course multitudes of technical and usability reasons to use it. But let's get the first obvious one out of the way. It's cool. It's what the cool kids use, and the reason it's cool, is because it's not bash. Bash is what boring kids go for. In a way it's like the old holy war between Emacs and Vim users. The whole war exists not because one is objectively better than the other. Let me elaborate:

If there is any reason why people use Vim/Emacs, it's because at some point, when they let someone use their computer, they get to nonchalantly, seemingly in passing mention "oh, you don't know how to use Vim?" while watching that someone struggle to exit it. Everyone likes feeling superior.

When I have to use a command line text editor, I use nano. Why? It gets the job done quickly and painlessly and is a low strain on my memory. I'm not a masochist like those Vim and Emacs people... ... See what I just did? Yes, I was being condescending. Like I said: everyone likes to feel superior in some way.

That is not to say that bash and zsh are the same. They are not, I really do believe that one is better than the other. Below are some reasons/excuses as to why.

## 1. Shared history

The reason for the switch for me was something seemingly unimportant, but something that always drove me crazy. Okay, maybe not crazy, but made me slightly annoyed. You know when you have several bash windows open and you try to find a command you used previously, either by pressing up arrow key or Ctrl-R? And you have to go through all the windows, because you don't exactly remeber which one the command was originally issued in? Well at some point, I decided that I deserve better than that and decided to look around. Turned out zsh had what I needed: in my `~/.zshrc` file I could add:

```
setopt share_history
setopt inc_append_history
export HISTSIZE=1000
export SAVEHIST=50000
export HISTFILE=~/.zsh_history
```

What do those lines do? The first one is the most crucial one, it enables sharing history between terminal sessions, this is what I wanted. However to make it functional in the described scenario, we need the second line which says that a command should be added to the history file immediately after execution, instead of after the session is closed. In the third line, `HISTSIZE` specifies the maximum amount of lines to be stored in the history of one session (by default only set to 30!) In the fourth `SAVEHIST` is the maximum amount of lines in the history file, location of which is specified in `HISTFILE` on line 5.

## 2. Autocompletion

Bash has autocompletion... a pretty dumb one in comparison to zsh. This is pretty much all that needs to be said on this topic. In zsh, when you install the proper modules (or have them installed for you, we'll get to this further down), you get proper autocompletion with tools like git, which can autocomplete not only commands, but also options, branch names, everything. Or with package managers, don't remember the full name of the package, or are too lazy to type? No matter if you use dnf, apt-get, pacman, zsh has you covered.

Also, when typing a partial filename and following it by tab press, by default, zsh will also try to match files in a case insensitive manner. A small thing, but can help you if you made a mistake. This (like most other features) can be disabled though if you don't like it.

## 3. Handling of lack of newline at the end of output

If a command exits without newline at the end of it's output, bash will print your prompt where the command left off, making your terminal session messy and hard to read. zsh improves on this, by printing a mark, a highlighted `%` sign, if the output doesn't end with a newline, while appending one anyway.

## 4. Extensibility

If you cannot be bothered setting everything up by yourself and going through the intricacies of configuration options, then [oh-my-zsh](http://ohmyz.sh)! This website has hundreds of themes and modules you can choose from, featuring the craziest options. From simple things like showing the current git branch you're working on in your prompt or CPU usage to things like the weather.

Personally, my config is very simple though. Apart from the lines you've seen above, which configure shared history, I only modified my command prompt a little bit, to have it show the return value of a command (if non-zero) and if a file ended in a newline of not:

```
export promptchars=%#
export PS1="%B%(?..↝ %?
)%n@%m:%~%#%b "
```

In `promptchars` the first entry specifies the prompt character for a regular user, and the second one for root. `PS1` is a pattern used to display a command prompt. It looks complicated, but only until you understand what it does:

- `%B` means start bold font;
- `%(?..↝ %?␊)` is a conditional expression, those come in the form of `%(x.true-text.false-text)` where `.` can be any character that is then also used to separate `true-text` from `false-text`. The condition `x` is a character from a [predefined list](http://zsh.sourceforge.net/Doc/Release/Prompt-Expansion.html#Conditional-Substrings-in-Prompts), in our case `?` means: true if exit code was equal to zero. True text is empty in such a case, meaning that if a command exits properly, nothing is written out. If, however the the command exits with a non-zero status, an arrow will we printed, followed by a space, and exit code (`%?`) and a line feed;
- `%n` means current user and `%m` means hostname, so `%n@%m` gives user@hostname;
- `:` is just a colon for style;
- `%~` current working directory, can be in relation to user's home, if lower in the file system hierarchy;
- `%#` the suitable prompt character;
- `%b ` ends the bold font, and adds a space

Simple sample session with the above configuration:

```
szymonlopaciuk@tatooine:~/Desktop% pwd
/home/szymonlopaciuk/Desktop
szymonlopaciuk@tatooine:~/Desktop% rm /
rm: /: is a directory
↝ 1
szymonlopaciuk@tatooine:~/Desktop% 
```

