---
layout: post
title: OptionParser With Commands
date: '2017-03-16'
author: Juri Timošin
tags:
- ruby
---

[1]: http://ruby-doc.org/stdlib/libdoc/optparse/rdoc/OptionParser.html
[2]: https://github.com/defunkt/choice
[3]: https://github.com/commander-rb/commander
[4]: http://stoneship.org/software/cri/
[5]: http://rubyworks.github.io/executable/
[6]: https://github.com/ahoward/main
[7]: https://github.com/davetron5000/methadone
[8]: https://rubygems.org/gems/mixlib-cli
[9]: https://rubygems.org/gems/optitron
[10]: http://github.com/leejarvis/slop
[11]: http://whatisthor.com/
[12]: http://manageiq.github.io/trollop/
[13]: https://bitbucket.org/DracoAter/jenkins2/src/tip/lib/jenkins2/cmdparse.rb
[14]: https://bitbucket.org/DracoAter/jenkins2/src/tip/test/unit/command_line_test.rb

Ruby standard library provides a good solution for parsing command line options passed to the
script or application. It's called [OptionParser][1]. It's powerful and very easy to use. The code
you have to write is easy to read and modify. It also supports different kind of options: short
and long flags, required or optional arguments, also different types of arguments including
dates, lists and so on.

<!--more-->

The only thing it cannot do, is parsing commands - that is a _word_ without any flag before it. For
example, it will not pick `do_something` from this command line:

{% highlight shell %}
$ my_program do_something -a attr1 --verbose --log=/path/to/file
{% endhighlight %}

Soon we will learn how to improve OptionParser, so that it is able to do that. Of course there are
many other different libraries that do command line argument parsing, including:

- [choice][2]
- [commander][3]
- [cri][4]
- [executable][5]
- [main][6]
- [methadone][7]
- [mixlib-cli][8]
- [optitron][9]
- [slop][10]
- [thor][11]
- [trollop][12]

They were created for different purposes, so not all of them support commands. Definitely you could
pick one of them to do the job, but I personally do not like to be dependant on 3rd party libraries,
that are not included in standard ruby installation. In this case also because as we will see, it
is very easy to add needed functionality to the OptionParser.

We are creating a parser, that will allow having different commands in cli, these commands can have
their own attributes and there are also some global attributes, that can be provided with any
command.

{% highlight shell %}
$ my-program [global-options] <command> [command-options]
{% endhighlight %}

We want to use the new option parser similar to existing one. At the same time we should be able to
have separate command options for every command, like that:

{% highlight ruby linenos %}
@global_options = OptionParser::OptionMap.new
@command_options = OptionParser::OptionMap.new
parser = CommandParser.new 'Usage: my_program [global-options] <command> [command-options]' do |opts|
	opts.separator "Global options (accepted by all commands):"
	opts.on '-h', '--help', 'Show help' do
		@global_options[:help] = true
	end
	opts.command 'my-command', 'It surely does something' do |cmd|
		cmd.on '-a', '--attribute INT', Integer, 'Some integer attribute for the command.' do |opt|
			@command_options[:attribute] = opt
		end
	end
end
{% endhighlight %}

We can achieve that by creating our `OptionParserWithCommands` that inherits from `OptionParser` and
giving it a new method called `command`, which just should create a `OptionParser::Switch` and add
it to an instance variable `commands`.

{% highlight ruby linenos %}
class OptionParserWithCommands < OptionParser
	def command( key, desc, &block )
		sw = OptionParser::Switch::NoArgument.new( key, nil, [key], nil, nil, [desc],
			Proc.new{ OptionParser.new( &block ) } ), [], [key]
		commands[key.to_s] = sw[0]
	end

	def commands
		@commands ||= {}
	end
end
{% endhighlight %}

Still the most important part is missing: parsing. We have to rewrite the `parse!` method.

{% highlight ruby linenos %}
def parse!( argv=default_argv )
	# Find, if there is a command in argv
	@command_name = argv.detect{|c| commands.has_key? c }
	if command_name
		#create a temporary parser with option definitions from both - global and particular command -
		# and parse all the options
		OptionParser.new do |parser|
			parser.instance_variable_set(:@stack,
				commands[command_name.to_s].block.call.instance_variable_get(:@stack) + @stack)
		end.parse! argv
	else
		# we do not have any (right) command provided, fallback to just parsing the options.
		super( argv )
	end
end
{% endhighlight %}

Perfect! There is just 1 thing missing - help message. Remember, OptionParser generates help
automatically based on options' definitions. We want our extended parser to do that too. But
providing all the options for all the commands may be too much. So let's do it this way. The global
options are always listed in help. And then, if a command is not provided, we list all the available
commands with descriptions (without their personal options) and, if a command is provided, we list
all the personal options for this command together with command's description.

{% highlight ruby linenos %}
attr_reader :command_name

private
def summarize(to = [], width = @summary_width, max = width - 1, indent = @summary_indent, &blk)
	super(to, width, max, indent, &blk) # display global options
	# command is provided
	if command_name and commands.has_key?( command_name )
		to << "Command:\n"
		# display command description
		commands[command_name].summarize( {}, {}, width, max, indent ) do |l|
			to << (l.index($/, -1) ? l : l + $/)
		end
		to << "Command options:\n"
		# display command options
		commands[command_name].block.call.summarize( to, width, max, indent, &blk )
	else
		to << "Commands:\n"
		# display available commands
		commands.each do |name, command|
			command.summarize( {}, {}, width, max, indent ) do |l|
				to << (l.index($/, -1) ? l : l + $/)
			end
		end
	end
	to
end
{% endhighlight %}

I have it implemented in a ruby gem [_jenkins2_][13]. There are also [some tests][14], which show
the supported options and commands.

So now if you would like to parse commands with OptionParser, all you need is just copy the code
from [this file][13] to your project.

This approach does not support nested commands, but it should be faily easy to add it, by replacing
`OptionParser` with `OptionParserWithCommands` inside `parse!` method and may be adding some checks.
