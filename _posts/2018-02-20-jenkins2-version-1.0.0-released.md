---
layout: post
title: Jenkins2 Version 1.0.0 Released
author: Juri Timošin
tags:
- ruby
- jenkins
---

[1]: https://rubygems.org/gems/jenkins2
[2]: https://www.goodreads.com/book/show/21824181-metaprogramming-ruby-2
[3]: https://github.com/huboard/ghee
[4]: https://bitbucket.org/DracoAter/jenkins2/src/e9c986eefcedc96401b0535bea7dfbff9397bd83/lib/jenkins2/resource_proxy.rb
[5]: https://bitbucket.org/DracoAter/jenkins2/src/e9c986eefcedc96401b0535bea7dfbff9397bd83/lib/jenkins2/cli.rb
[6]: https://bitbucket.org/DracoAter/jenkins2/addon/pipelines/home
[7]: https://bitbucket.org/DracoAter/jenkins2/src/e9c986eefcedc96401b0535bea7dfbff9397bd83/bitbucket-pipelines.yml
[8]: https://bitbucket.org/DracoAter/jenkins2/src/e9c986eefcedc96401b0535bea7dfbff9397bd83/lib/jenkins2/cli/plugins.rb
[9]: https://bitbucket.org/DracoAter/jenkins2/issues

Yesterday I have released version 1.0.0 of my [jenkins2][1] gem. It's a API client and a
command line application at the same time, that allows you to manage your Jenkins.
And to be honest I am proud of the code I have pushed. Here is why:

<!--more-->

## Ghost Methods

I have read a book [Metaprogramming Ruby 2][2], and there was a good example of API client
([ghee][3]), which uses some metaprogramming magic to map Github's API to objects. I also managed
to do something similar in my own code. Any API class implements methods that require
specific code, and at the same time forwards methods that just read data to [ResourceProxy][4]
class, which is essentially a wrapper for the json objects returned by Jenkins. This way the gem
adapts automatically to changes in Jenkins API, and at the same time the code is kept compact.

## Command Line Parser

Another thing I am proud of, is my personal command line argument parser, which is at the same time
powerful and compact. It can parse not just command line arguments, but also commands, although it
doesn't use any third party gem, only the OptionParser, which is included in standard library.
Again all the classes that provide commands derive from one [CLI][5] class, which does most of the
common work like parsing global arguments and command itself, and delegates parsing the command
arguments to particular derivative class. Adding new command is very simple, all you need is to
create a new class, derive it from CLI and overwrite some methods. (Check out [ShowPlugin][8]
class for example.)

## High Test Coverage

And the last, but not least thing is, of course, test coverage, which is currently at 98.74%. I
am testing the gem with unit and integration tests, which run in [bitbucket pipelines][6].
Integration tests are run against Jenkins server inside docker container.
Look into [bitbucket-pipelines.yml][7] for details.

The work, of course, is not finished yet. I am planning to add support for more API endpoints and
plugins as well as provide more commands in command line client. If you feel like lacking some
functionality, please [create an issue][9], or if you have some free time, make yourself
comfortable and create a pull request.
