---
layout: post
title: How To Split Audio CD Image (.flac) Into Several Tracks In Ubuntu
author: Juri Timošin
tags:
- audio
---

This is still working

```bash
$ sudo apt-get install cuetools shntool flac #install needed tools
$ cuebreakpoints sample.cue | shnsplit -o flac sample.flac #read breakpoints from cue and give them to splitter
$ cuetag sample.cue split-track*.flac #add tags to newly created files
```
