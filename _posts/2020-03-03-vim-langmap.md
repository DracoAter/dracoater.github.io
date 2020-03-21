---
layout: post
title: Vim langmap
author: Juri Timošin
tags:
- vim
---

Sometimes I need to write in russian using vim. The problem that arise is that normal mode in vim
does not work with cyrillic, so I need to switch back and forth between US and RU layouts.

<!--more-->

I don't like switching layouts back and forth all the time. So eventually I found some kind of
solution to this:

```
:help langmap
```

When in Normal mode the 'langmap' option takes care of translating a character in different
layout to the original meaning of the key. So I added the following line into my `.vimrc`.

```
set langmap=йЙцЦуУкКеЕнНгГшШщЩзЗхХъЪфФыЫвВаАпПрРоОлЛдДжЖэЭ\/яЯчЧсСмМиИтТьЬбБюЮ.\,;qQwWeErRtTyYuUiIoOpP[{]}aAsSdDfFgGhHjJkKlL\;:'"\\|zZxXcCvVbBnNmM\,<.>/?
```

Yes, you need to map both small and capital letters. Now I don't have to switch from RU to US in
Normal mode for the shortcuts to work..
