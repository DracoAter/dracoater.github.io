---
layout: post
title: Testing Chef Cookbooks. Part 2.5. Speeding Up ChefSpec Run
date: '2013-12-14T15:47:00.000+02:00'
author: Juri Timošin
tags:
- chef
- chefspec
- rspec
- ruby
modified_time: '2014-01-04T13:27:58.271+02:00'
blogger_id: tag:blogger.com,1999:blog-360329120074358364.post-1232515208708780825
blogger_orig_url: http://dracoater.blogspot.com/2013/12/testing-chef-cookbooks-part-25-speeding.html
---

[1]: https://github.com/sethvargo/chefspec#faster-specs
[2]: {% post_url 2013-09-23-testing-chef-cookbooks-part-2-chefspec %}
[3]: https://www.relishapp.com/rspec/rspec-core/v/3-0/docs/helper-methods/define-helper-methods-in-a-module

**Disclaimer**: as of Jan 4 2014, this is already [implemented inside][1] ChefSpec
(>= 3.1.2), so you don't have to do anything. The post just describes the problem and solution with
more details.

Last time we were speaking about testing Chef recipes, [I introduced to you ChefSpec][2]
as a very good tool for running unit tests on your cookbooks. But lately I encountered a problem
with it. I have over 800 unit tests (aka *examples* in RSpec world) and now it takes about 20
minutes to run. **20 minutes, Karl!!!** That's a extremely long time for this kind of task. So I
decided to delve, what exactly is responsible for taking so much time.

<!--more-->

My examples look like that (many recipes have similar example groups for *windows* and *mac_os_x*):

{% highlight ruby linenos %}
describe "example::default" do
	context 'ubuntu' do
		subject { ChefSpec::ChefRunner.new( :platform => 'ubuntu', :version => '12.04' ).converge described_recipe }
		let( :node ) { subject.node }

		it { should do_something }
		it 'does some other thing' do
			should do_another_thing
		end
		it { should do_much_more }
	end
end
{% endhighlight ruby %}

I put some printouts inside *describe*, *context*, *subject* and *let* blocks, as well as read RSpec
documentation about *let* and *subject*. Turned out, that *subject* and *let* blocks are called for
every test, i.e. they are cached when accessed inside 1 test (*it* block), but not across tests
inside test group (in our case *ubuntu* context). So for these tests *subject* is actually
calculated 3 times. That is not a problem for ordinary RSpec tests, where subject most of the time
is an object returned by constructor, e.g. <code>User.new</code>. But in ChefSpec case we have a
*converge* operation as Subject under Test (SuT), which is more costly and takes more time to
calculate. Another difference is that, opposing to ordinary RSpec tests we do not change the SuT
in ChefSpec, but just make sure that it has right resources with right actions. So running
*converge* for every example is a huge overhead.

How can we fix that? Well, obviously we should somehow save the value across the examples. I tried
different approaches, some of them worked partially, some didn't at all. The simplest thing was to
use *before :all* block.

{% highlight ruby linenos %}
describe "example::default" do
  context 'ubuntu' do
    before :all { @chef_run = ChefSpec::ChefRunner.new( :platform => 'ubuntu', :version => '12.04' ).converge described_recipe }
    subject { @chef_run }
    [...]
  end
end
{% endhighlight ruby %}

It does not require any more than small change in spec files, but the drawback of this approach is
**no mocking is supported in _before :all_ block**. So if you have to mock for example file
existence, **it would not work**:

{% highlight ruby linenos %}
describe "example::default" do
  context 'ubuntu' do
    before :all do
      ::File.stub( :exists? ).with( '/some/path/' ).and_return false
      @chef_run = ChefSpec::ChefRunner.new( :platform =&gt; 'ubuntu', :version =&gt; '12.04' ).converge described_recipe
    end
    subject { @chef_run }
    [...]
  end
end
{% endhighlight ruby %}

[RSpec allows to extend modules with your own methods][3] and the idea was to write
method similar to *let*, but which will cache the results across examples too. Create a
*spec_helper.rb* file somewhere in your Chef project and add the following lines there:

{% highlight ruby linenos %}
module SpecHelper
  @@cache = {}
  FINALIZER = lambda {|id| @@cache.delete id }

  def shared( name, &amp;block )
    location = ancestors.first.metadata[:example_group][:location]
    define_method( name ) do
      unless @@cache.has_key? Thread.current.object_id
        ObjectSpace.define_finalizer Thread.current, FINALIZER
      end
      @@cache[Thread.current.object_id] ||= {}
      @@cache[Thread.current.object_id][location + name.to_s] ||= instance_eval( &amp;block )
    end
  end

  def shared!( name, &amp;block )
    shared name, &amp;block
    before { __send__ name }
  end
end

RSpec.configure do |config|
  config.extend SpecHelper
end
{% endhighlight ruby %}

Values from *@@cache* are never deleted, and you can use same names with this block, so I also use
*location* of the usage, which looks like that: "./cookbooks/my_cookbook/spec/default_spec.rb:3".
Now change *subject* into *shared( :subject )* in your specs:

{% highlight ruby linenos %}
describe "example::default" do
  context 'ubuntu' do
    shared( :subject ) { ChefSpec::ChefRunner.new( :platform => 'ubuntu', :version => '12.04' ).converge described_recipe }
    [...]
  end
end
{% endhighlight ruby %}

And when running the tests you will now have to include the spec_helper.rb too:

{% highlight bash %}
rspec --include ./relative/path/spec_helper.rb cookbooks/*/spec/*_spec.rb
{% endhighlight bash %}

If you use the rake task I introduced in [previous post][chefspec-post], add the following line to
it.

{% highlight ruby linenos %}
desc 'Runs specs with chefspec.'
RSpec::Core::RakeTask.new :spec, [:cookbook, :recipe, :output_file] do |t, args|
  [...]
  t.rspec_opts += ' --require ./relative/path/spec_helper.rb'
  [...]
end
{% endhighlight ruby %}

And that's all! Now tests run in 2 minutes. 10 times faster!
