---
layout: post
title: Dynamic Bash Prompt
author: Juri Timošin
tags:
- bash
---

[1]: https://ss64.com/bash/syntax-prompt.html

I have a problem. I keep constantly committing (and pushing) to wrong branches and then keep
searching for how to move those commits to the right place. So I decided that bash prompt should
remind me, what branch I am currently on. I need a scm tool name (because I use several) and
a branch.

<!--more-->

Bash has some [special environment variables][1], which are responsible for the bash prompt.
Those are PS1, PS2, PS3, PS4 and PROMPT\_COMMAND. We can set our prompt by changing the value of
the PS1 environment variable, for example like that:

{% highlight console %}
$ export PS1="My new prompt> "
{% endhighlight %}

But this only assigns the prompt statically. What we actually need is, assigning dynamically,
based on what our current working directory is. For that the PROMPT\_COMMAND variable is
responsible. Its value is interpreted as a command to execute before the printing of each primary
prompt ($PS1). PS1 variable is set in your ~/.bashrc, so what I did is added some code there:

{% highlight diff linenos %}
+PROMPT_COMMAND=scm_prompt
+
+scm_prompt(){
+  if [ -d ".git" ]; then
+    scm=" git(`git symbolic-ref --short HEAD`)"
+  elif [ -d ".hg" ]; then
+    scm=" hg(`hg branch`)"
+  else
+    scm=""
+  fi
+}
+
 if [ "$color_prompt" = yes ]; then
   PS1='${debian_chroot:+($debian_chroot)}\[\033[01;32m\]\u@\h\[\033[00m\]:\[\033[01;34m\]\w\[\033[00m\]\$ '
 else
-  PS1='${debian_chroot:+($debian_chroot)}\u@\h:\w\$ '
+  PS1='${debian_chroot:+($debian_chroot)}\u@\h:\w${scm}\$ '
 fi
 unset color_prompt force_color_prompt
{% endhighlight diff %}

Now, if current directory contains `.git` or `.hg` directories, then I will see the current scm
and branch I am on in bash prompt.

{% highlight console %}
jt@ratchet:~/projects/dotfiles hg(default)$
{% endhighlight %}
