---
layout: post
title: Git aliases
date: 2024-10-23
author: Juri Timošin
tags:
- git
- bash
- powershell
---

[1]: https://learn.microsoft.com/en-us/powershell/module/microsoft.powershell.core/about/about_aliases?view=powershell-7.4

Here are a bunch of aliases I use to work with git in command line.

<!--more-->

## Bash

For the bash it is simple, just copy and paste them into your `~/.bash_aliases`

```bash
alias gfmp='git fetch -p && git checkout master && git pull'
alias ga='git add'
alias gd='git diff'
alias gdi='git diff --ignore-cr-at-eol'
alias gc='git commit'
alias gs='git status'
alias gpl='git pull'
alias gps='git push --all'
alias gl='git log --oneline --decorate --graph --all'
alias gld='git log --format="%C(auto)%h %d %ad %<(10,trunc)%al %s" --graph --all --date="format:%Y-%m-%d %H:%M"'
```

And you can use `gfmp` to git fetch, checkout master and pull, and some others like `gl` for git
log.

## Powershell

With powershell it is a bit harder to configure. [Powershell aliases][1] do not support passing
parameters.

> You cannot assign an alias to a command and its parameters. For example, you can assign an alias
> to the Get-Eventlog cmdlet, but you cannot assign an alias to the Get-Eventlog -LogName System
> command.

As a workaround we can use functions. Let's create a profile which is automatically run in every
new powershell window and add there our new functions:

```ps
PS> New-Item -Type file -Path $PROFILE -Force
PS> notepad.exe $PROFILE
```

Add the following lines into the editor and save:

```ps
function gfmp{git fetch -p; git checkout master; git pull}
# @args you can pass multi arguments for example
# ga fileName1 fileName2 
function ga{git add @args}
function gd{git diff}
function gdi{git diff --ignore-cr-at-eol}

function gc{git commit}
function gs{git status}
function gpl{git pull}
function gps{git push --all}
function gl{git log --oneline --decorate --graph --all}
function gld{git log --format="%C(auto)%h %d %ad %<(10,trunc)%al %s" --graph --all --date="format:%Y-%m-%d %H:%M"}
```

Now when you start the new powershell window, you will be able to use the new aliases for git.

### Except...

Because I got used to those aliases (in bash), I don't want to change them. But Powershell already
has default aliases defined for some of them, like `gc -> Get-Content` and `gl -> Get-Location`.
And they prevail on my custom functions.

Btw, to list all existing aliases you can run `Get-Alias`.

Removing those default aliases with `Remove-Alias` does not work, as they are "constant or
read-only". But there is a way:

```ps
PS> Remove-Item -Force Alias:gl
PS> Remove-Item -Force Alias:gc
```

And voila!
