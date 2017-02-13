---
layout: post
title: 'Java: Save InputStream Into File'
date: '2011-11-14T12:15:00.001+02:00'
author: Juri Timošin
tags:
- Java
modified_time: '2011-11-14T12:44:25.701+02:00'
blogger_id: tag:blogger.com,1999:blog-360329120074358364.post-8217883902575003198
blogger_orig_url: http://dracoater.blogspot.com/2011/11/java-save-inputstream-into-file.html
---

Imagine we need to save an InputStream into file. That can happen when requesting some url, or when
just copying the file from one place to another on hard disk. There are a lot of answers provided
by google on request *java save inputstream to file*. I have  checked the first result page and
everywhere almost 1 and the same solution is provided, which includes the following loop:

{% highlight java %}
int read = 0;
byte[] bytes = new byte[1024];

while ((read = inputStream.read(bytes)) != -1) {
	out.write(bytes, 0, read);
}
{% endhighlight java %}

Seriously, guys, don't you think there is something wrong here? Even in C++ people do not copy
streams by operating bytes anymore! There should be a lot better way :) (I am not considering now
usage of any additional libraries that require some additional jar files).

{% highlight java %}
import sun.misc.IOUtils;

new FileOutputStream("tmp.txt").write(IOUtils.readFully(inputStream, -1, false));
{% endhighlight java %}
