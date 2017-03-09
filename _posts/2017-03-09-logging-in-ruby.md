---
layout: post
title: Logging In Ruby
date: '2017-03-09'
author: Juri Timošin
tags:
- ruby
---

[1]: https://ruby-doc.org/stdlib/libdoc/logger/rdoc/Logger.html
[2]: https://en.wikipedia.org/wiki/Singleton_pattern

My work involvs writing many small command line applications (scripts) in ruby. One of the common
things among them is logging. The application should produce adequate log messages, so that one
could follow the application flow. Ruby has a very good [logger][1] in standard library. You can
use it like that:

<!--more-->

{% highlight ruby linenos %}
require 'logger'

# initialize
logger = Logger.new(STDOUT)
logger.level = Logger::WARN

# write messtages
logger.debug("Created logger")
logger.info("Program started")
logger.warn("Nothing to do!")

{% endhighlight ruby %}

But the applications most of the time are consisting from several classes. But at the same time we
do want to use the same logger in every part of our application, we do not want to initialize the
logger inside every class. That means we have either to pass the logger to the object, when it's
created, or user global variables (`$logger`). Either approach seemed not so comfortable for me.

In fact this is a perfect place for the [singleton pattern][2]. I have a following class now in my
applications.

{% highlight ruby linenos %}
require 'forwardable'
require 'logger'

class Log
	extend SingleForwardable

	def self.init( log: $stdout, verbose: 0 )
		@logger = Logger.new log
		@logger.level = Logger::ERROR - verbose
		@logger.formatter = proc do |severity, datetime, progname, msg|
			if log == $stdout
				"#{msg}\n"
			else
				"[#{datetime.strftime '%FT%T%:z'}] #{severity} #{msg}\n"
			end
		end
		@logger
	end

	def_delegators :logger, :debug, :info, :warn, :error, :fatal, :unknown, :level

	private

	def self.logger
		@logger ||= self.init
	end
end
{% endhighlight ruby %}

Now logger can be called from any place in the application and it doesn't have to be initialized
beforehand.

{% highlight ruby linenos %}
Log.debug("Created logger")
Log.info("Program started")
Log.warn("Nothing to do!")
{% endhighlight ruby %}
