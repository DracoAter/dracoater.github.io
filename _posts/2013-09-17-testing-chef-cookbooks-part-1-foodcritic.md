---
layout: post
title: Testing Chef Cookbooks. Part 1. Foodcritic.
date: '2013-09-17T11:41:00.000+03:00'
author: Juri Timošin
tags:
- chef
- foodcritic
- ruby
modified_time: '2013-09-23T14:50:13.134+03:00'
blogger_id: tag:blogger.com,1999:blog-360329120074358364.post-3595062972362814103
blogger_orig_url: http://dracoater.blogspot.com/2013/09/testing-chef-cookbooks-part-1-foodcritic.html
---

[1]: http://rubygems.org/gems/rake/
[2]: http://acrmp.github.com/foodcritic/
[3]: https://github.com/opscode/chef-repo/blob/master/Rakefile
[4]: http://acrmp.github.io/foodcritic/#FC001
[5]: https://github.com/etsy/foodcritic-rules
[6]: {% post_url 2013-09-23-testing-chef-cookbooks-part-2-chefspec %}
[7]: https://github.com/acrmp/chefspec

This post have been in draft for almost a year. At last I have made myself to turn back to my blog
and continue writing.

A couple of years ago we have adopted automating our server configuration through Chef recipes. In
the beginning we didn't have many cookbooks and all the recipes were more or less simple. But as
time passed new applications had to be installed and configured, including some not so simple
scenarios. Turned out that "hit and miss" method wasn't too good. We came across a lot of errors,
such as:

<!--more-->

- typo: simple as that. Although knife makes a syntax check on uploading cookbooks to server, but
many times it was a wrong path or something like that, which could be revealed only after we tried
to provision a node.
- missed dependencies: we ran our scripts on ec2 instances, and to save time on bootstrapping the
new instance, we didn't do it every time we changed something in our scripts, but only the first
time. When we finally got the recipe working on this node, sometimes it turned out that the recipe
will not be working on another clean node, because we forgot some dependencies, that were somehow
already installed on our test node.
- interfering applications: some recipes were used for configuring several applications. Although
they were very similar, they were not identical, and sometimes changing the recipe for installing
one application broke installation of the other one.

At last we came to the idea, that infrastructure code as any other code should be tested. Currently
we have built a pipeline using Jenkins that first runs syntax check, then coding conventions tests,
then unit tests and finally integration tests on cookbooks. And only if all the tests are passing,
Jenkins runs `knife cookbook upload`, publishing the cookbooks on chef-server.

In my following posts I will share with you how we established this testing architecture for our
Chef recipes. There will 3 parts in the tutorial, we will start from the easiest checks that only
make sure that your ruby code can be parsed and follows some code conventions.

First of all make sure you are running ruby 1.9.2 or newer. (We use Ubuntu on our linux servers, so
all the code I provide is tested in Ubuntu 12.04 LTS.)

{% highlight bash %}
sudo aptitude install ruby1.9.3
{% endhighlight bash %}

Now we need [rake][1] for building our project and [foodcritic][2] for testing.

{% highlight bash %}
sudo gem install rake foodcritic --no-ri --no-rdoc
{% endhighlight bash %}

Chef cookbook repository has a [Rakefile][3] in it. Now when we have rake installed, we can try and
run rake inside the folder Rakefile is in.

{% highlight bash %}
rake
{% endhighlight bash %}

This should run syntax check on your recipes, templates and files inside cookbooks. Next we can try
to run foodcritic tests on your cookbooks. Type

{% highlight bash %}
foodcritic cookbooks
{% endhighlight bash %}

and hit enter (assuming that "cookbooks" is the folder your cookbooks are in). If it founds some
warnings, it will print them out, otherwise an empty string will be printed.

Of course you may not agree with some of the rules and would like to ignore them. This could be
achieved by providing additional options to `foodcritic` command. You can also write your own rules
and add them into checks. For example, there was a [FC001][4] rule, which stated "Use strings in
preference to symbols to access node attributes". I actually preferred vice versa using symbols to
strings. So I created a new rule:

{% highlight ruby linenos %}
rule "JT001", "Use symbols in preference to strings to access node attributes" do
	tags %w{style attributes jt}
		recipe do |ast|
		attribute_access( ast, :type => :string )
	end
end
{% endhighlight ruby %}

and saved it into `foodcritic-rules.rb` file. Then it was easy to disable the existing FC001 rule
and enable mine with:

{% highlight bash %}
foodcritic cookbooks --include foodcritic-rules.rb --tags ~FC001
{% endhighlight bash %}

There are also some [3<sup>rd</sup> party rules][5] available, so you have something to start with.

Now we will join rake and foodcritic together by creating a rake task that runs foodcritic tests.
Add a new rake task similar to:

{% highlight ruby linenos %}
desc "Runs foodcritic linter"
task :foodcritic do
  if Gem::Version.new("1.9.2") <= Gem::Version.new(RUBY_VERSION.dup)
    sh "foodcritic cookbooks --epic-fail correctness"
  else
    puts "WARN: foodcritic run is skipped as Ruby #{RUBY_VERSION} is < 1.9.2."
  end
end
task :default => 'foodcritic'
{% endhighlight ruby %}

Now when you run `rake` it should run both syntax and code conventions tests.
[Next post will be about unit tests][6] using [chefspec][7].
