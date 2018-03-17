---
layout: post
title: Integration Tests for Test Kitchen Driver
author: Juri Timošin
tags:
- lxd
- ruby
- test-kitchen
---

[1]: https://kitchen.ci
[2]: https://github.com/DracoAter/kitchen-lxd

Imagine you write a driver for [Test Kitchen][1]. It's reasonable, if you want to check, if your
driver is working before publishing it. But which approach to take? Building the gem and
installing it locally, then manually trying to launch Test Kitchen with your driver, does not
really feel the way to do it.

<!--more-->

I write a [Test Kitchen driver for Lxd][2] and here how I solved the problem of testing. Obviously
we start by adding a `.kitchen.yml` to our driver project, where we make Kitchen use our driver.
My driver name is `lxd`, so my `.kitchen.yml` looks like that:

{% highlight yaml linenos %}
---
driver:
  name: lxd

transport:
  name: lxd
{% endhighlight %}

Now when we try to run Kitchen we get the error, telling us that there is no such driver found.

{% highlight console %}
$ kitchen list
>>>>>> ------Exception-------
>>>>>> Class: Kitchen::ClientError
>>>>>> Message: Could not load the 'lxd' driver from the load path. Please ensure that your driver
is installed as a gem or included in your Gemfile if using Bundler.
>>>>>> ----------------------
>>>>>> Please see .kitchen/logs/kitchen.log for more details
>>>>>> Also try running `kitchen diagnose --all` for configuration
{% endhighlight %}

Looking further into `.kitchen/logs/kitchen.log` reveals the exact error:

{% highlight console %}
ERROR -- Kitchen: Message: cannot load such file -- kitchen/driver/lxd
{% endhighlight %}

Well, Kitchen cannot find the file, obviously, because it does not know where to look for. This
file is in the `lib` directory of our driver repository. What we need to do now, is to tell Kitchen,
that it should look also in `lib` directory. This can be done by adding `lib` directory to
`$LOAD_PATH` global ruby variable. Thankfully Kitchen parses `.kitchen.yml` as an `erb` template,
so we can just add these instructions at the top of the file:

{% highlight yaml linenos %}
# Make sure the local copy of the driver is loaded
<% lib = File.expand_path('lib', __dir__) %>
<% $LOAD_PATH.unshift(lib) unless $LOAD_PATH.include?(lib) %>
---
driver:
  name: lxd

transport:
  name: lxd
{% endhighlight %}

And that's it! Now when you run `kitchen list` again, you will see no error, but the list of
kitchen instances.

Now you can continue with adding different tests to `test/integration` folder as usual, as if you
are testing some completely random project, not connected to Test Kitchen, with Test Kitchen.
