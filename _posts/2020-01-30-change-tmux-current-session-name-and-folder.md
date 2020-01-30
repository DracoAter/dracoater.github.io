---
layout: post
title: Change Tmux Current Session Name and Default Directory
author: Juri Timošin
tags:
- tmux
---

Run these commands inside the tmux session you want to change.

```bash
:rename-session <new-name>
:attach-session -t . -c <new-default-path>
```
